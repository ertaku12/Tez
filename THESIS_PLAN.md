# Thesis Plan: LLM-Assisted Source Code Vulnerability Detection for Java/C# Web Applications

Living document. Update as decisions are made. Last updated: 2026-11-04.

## 1. Context

**Problem.** Manual source-assisted pentesting of Java/C# web applications is slow, and classic SAST tools produce high false-positive rates and cannot reason about application-specific authorization logic. Commercial LLM APIs could help, but sending client source code to third parties is unacceptable for a security team (confidentiality, NDAs, KVKK/GDPR, government context).

**Thesis idea.** Build a tool that combines a program-analysis backend (code property graph + taint tracking) with locally hosted open-source LLMs to detect OWASP Top 10 vulnerabilities in Java and C# web application source code. The LLM does the "human-like" reasoning (is this input actually sanitized? does this endpoint enforce the application's own authorization rules?), while the static-analysis backend supplies precise facts (entry points, data-flow paths, call graphs) that LLMs cannot reliably derive on their own over large codebases.

**Intended outcome.**
- A working, open-source-model-only pipeline (no code leaves the machine).
- Empirical comparison: manual pentest vs. new tool vs. free SAST vs. paid SAST, on benchmark apps and real engagements.
- A defensible thesis argument for privacy-preserving, locally hosted LLM security tooling.

**Scope.** Static analysis only (no DAST) for now. Languages: Java (Spring/Jakarta) and C# (ASP.NET Core / ASP.NET MVC). Vulnerability focus: OWASP Top 10 categories that are detectable from source.

## 2. Research Questions (draft)

- RQ1: Does combining CPG-based data flow with LLM triage improve precision/recall over (a) SAST alone and (b) LLM alone?
- RQ2: Can an LLM learn an application's custom authorization model from source (with light human confirmation) and use it to find Broken Access Control issues that rule-based SAST misses?
- RQ3: How do open-source local models compare with each other (and optionally with a closed model as an upper bound, on public benchmark apps only)?
- RQ4: What is the practical effect on a manual pentest team (triage time, missed findings)?

## 3. Vulnerability Scope (OWASP Top 10 → concrete CWE targets)

Detectable from source and in scope:

| OWASP | Concrete targets | Detection mode |
|---|---|---|
| A01 Broken Access Control | Missing/incorrect authz on endpoints, IDOR, mass assignment/over-posting, path traversal | **Auth-profile + LLM reasoning** (main novelty) |
| A03 Injection | SQLi, NoSQLi, OS command, LDAP, XSS (reflected/stored), template injection, XPath | Taint flow + LLM sanitizer judgement |
| A08 Software/Data Integrity | Insecure deserialization (ObjectInputStream, XMLDecoder, BinaryFormatter, Json.NET TypeNameHandling, XmlSerializer with untrusted type) | Sink keyword report + flow + LLM |
| A10 SSRF | URL built from user input into HttpClient/RestTemplate/WebClient/HttpWebRequest | Taint flow + LLM |
| A05 Security Misconfiguration | XXE (DocumentBuilder, XmlReader settings), CORS, debug flags, CSRF disabled | Config/keyword report + LLM |
| A02 Cryptographic Failures | Hardcoded secrets, weak algorithms (MD5/SHA1/DES/ECB), insecure random | Keyword report + LLM context check |
| A07 Identification/Auth Failures | Weak password handling, missing rate limiting hints, JWT misverification | Partial, LLM-heavy |

Out of scope or handled by other tools: A06 vulnerable components (SCA tools, mention in thesis as complementary), A09 logging failures (hard to judge statically), A04 insecure design (partial, via auth profile).

Note: OWASP Top 10:2025 exists; decide whether to frame on 2021 or 2025 taxonomy. Map every finding to CWE regardless so the framing choice is cosmetic.

## 4. Architecture

```
repo ─▶ [0 Ingest] ─▶ [1 Build CPG (Joern)] ─▶ [2 Attack-surface map]
                                                  │
                      ┌───────────────────────────┘
                      ▼
         [3 Authorization Profile builder]  (LLM + human confirm → app_security_profile.yaml)
                      │
                      ▼
         [4 Candidate generators per vuln class]  → candidate reports (JSON)
                      │
                      ▼
         [5 Slice extractor]  (Joern flows: source → sink, code of every method on path)
                      │
                      ▼
         [6 LLM triage agent]  (step-by-step, tool-calling, structured verdict)
                      │
                      ▼
         [7 Findings store → SARIF / HTML report / pentester worklist]
```

### Stage 0 — Ingest
- Detect build system (Maven/Gradle; .sln/.csproj), framework (Spring Boot, Jakarta EE, ASP.NET Core, ASP.NET MVC/WebForms).
- Produce a repo inventory: file tree, LOC, frameworks, dependency list (useful for SCA cross-reference later).

### Stage 1 — Code Property Graph
- Joern frontends: `javasrc2cpg` (Java source), `jimple2cpg` (bytecode, more precise if build succeeds), `csharpsrc2cpg` (C#, newer and less mature; must be evaluated early).
- Wrap behind a `CodeGraphProvider` interface so Joern can be swapped/complemented (CodeQL, tree-sitter, Roslyn) without touching the rest of the pipeline.

### Stage 2 — Attack-surface map (entry points = taint sources)
- Framework-aware extraction of HTTP entry points regardless of file naming:
  - Java: `@RestController`, `@Controller`, `@RequestMapping`/`@GetMapping`..., JAX-RS `@Path`, servlets (`doGet/doPost`, `web.xml`), Spring WebFlux handlers.
  - C#: `[ApiController]`, `[Route]`, `[HttpGet]`..., classes deriving `Controller`/`ControllerBase`, minimal APIs (`app.MapGet`), Razor Pages handlers (`OnGet/OnPost`), WebForms `.aspx.cs`.
- Output: endpoint table (route, HTTP verb, handler method, parameters, binding source, declared auth attributes). This table is also a deliverable useful to the manual pentest team.

### Stage 3 — Authorization Profile (teaching the LLM the app's authz mechanics)
This is the key design for "application access control logic can differ".
1. Collect authz-relevant evidence automatically: security config classes (`SecurityFilterChain`, `WebSecurityConfigurerAdapter`, `Startup/Program.cs` auth middleware), annotations/attributes (`@PreAuthorize`, `@Secured`, `@RolesAllowed`, `[Authorize]`, policies), filters/interceptors/middleware, role/permission enums, user/session/principal access patterns, ownership checks (`where owner_id = currentUser`).
2. LLM summarizes into a structured **Application Security Profile** (YAML):
   - identity source (JWT, session, header), roles/permissions model, how the current user is obtained, global vs. per-endpoint enforcement, resource-ownership check idioms, known "admin only" markers, tenant isolation idiom.
3. **Human-in-the-loop**: pentester reviews/edits the profile once per application (minutes, not hours). This is defensible in the thesis as "expert-guided" and is also how the team actually works.
4. Profile is injected into every later LLM prompt as context, and drives the A01 checks: for each endpoint, "given this profile, is authorization enforced correctly for this handler and the data it touches?"

### Stage 4 — Candidate generators (the "reports" that drive step-by-step LLM work)
- Per vulnerability class, a cheap generator emits a candidate report: sink location, sink type, keyword hit, surrounding snippet, enclosing method, plus which entry points reach it (from Stage 5 when available).
- Generators: Joern CPGQL queries (preferred), Semgrep OSS rules (fast, good for C# fallback), simple regex for config/secrets.
- Candidate report schema is the contract between "finding something suspicious" and "LLM decides". Keeping it explicit lets you measure how many candidates the LLM correctly dismisses (precision gain).

### Stage 5 — Slice extractor (data flow)
- For each candidate sink: `reachableByFlows` from entry-point parameters to the sink; collect every method on the path, plus sanitizer-looking calls along the way (encoders, parameterization, validators, allowlists).
- Emit a compact "flow dossier": ordered path steps with code, max N tokens, truncation strategy (keep source, sink, and transformation steps; summarize pass-through steps).
- If no flow is found, still send the sink with its callers (LLM may find flows Joern missed, e.g. through reflection, DI, ORM), but tag it so the evaluation can separate "flow-confirmed" from "LLM-only" findings.

### Stage 6 — LLM triage agent
- Step-by-step protocol per candidate:
  1. Classify vuln class and restate source/sink.
  2. Walk the flow; at each step decide: taint preserved / sanitized / neutralized by type (e.g., parsed to int).
  3. Check framework defaults (e.g., Thymeleaf auto-escaping, Razor encoding, JPA parameter binding) using a per-framework knowledge sheet.
  4. For A01: compare handler against the Authorization Profile.
  5. Output structured verdict: `vulnerable | not_vulnerable | needs_review`, confidence, evidence lines, suggested PoC hint for the pentester, CWE.
- Tool-calling so the model can pull more context on demand. This keeps prompts small and lets the model "track keywords". Toolset is small, deterministic, read-only; every tool returns bounded output with file:line references so the model can cite evidence.

  **Navigation (tree-sitter / CPG backed)**
  - `read_file(path, start_line, end_line)` — raw code, capped at ~200 lines.
  - `get_method(fqn)` — source of one method plus signature and annotations/attributes.
  - `find_symbol(name)` — definition site of a class, method or field.
  - `find_usages(symbol)` — call sites and references, paginated.
  - `get_callers(method)`, `get_callees(method)` — one hop of the call graph.
  - `grep(pattern, glob)` — regex over the repo, capped results (keyword tracking).

  **Data flow (Joern backed)**
  - `get_flows(source, sink)` — taint paths from an entry-point parameter to a sink, as ordered steps with code.
  - `get_data_dependencies(variable, method)` — intra-method origin of a variable's value.
  - `list_sinks(category)` — candidate report for a vuln class, for iteration.

  **Application knowledge (generated artifacts)**
  - `get_endpoints(filter)` — attack-surface table from Stage 2.
  - `get_auth_profile()` — human-confirmed Authorization Profile from Stage 3.
  - `get_framework_notes(framework, topic)` — short sheets on defaults (Razor auto-encoding, Thymeleaf escaping, JPA parameter binding, model binding rules).
  - `get_config(key_pattern)` — application.yml, appsettings.json, web.xml, security config classes.

  **Output**
  - `record_verdict(json)` — forces the final answer into the verdict schema (class, CWE, verdict, confidence, evidence lines, PoC hint).

  **Excluded on purpose:** shell execution, network access, file writes (auditability + privacy argument). Budget: ~15–20 tool calls per candidate; log every call for the "how models investigate" analysis.
- Self-consistency option: run K samples or two different models and vote; measure whether it helps.

### Stage 7 — Output
- SARIF 2.1 (so results are directly comparable with SAST tools in the same viewers and in the OWASP Benchmark scorer), HTML report, and a pentester worklist ordered by confidence × severity.
- Store everything in SQLite/DuckDB: candidates, prompts, responses, tokens, latency, final verdicts (needed for the evaluation chapter and for reproducibility).

### Retrieval strategy (RAG decision)
- **Core pipeline uses structured, deterministic retrieval**: Joern queries, grep, endpoint table, authorization profile. For taint reasoning this beats embedding similarity ("which method calls this sink" has one exact answer). Frame in the thesis as graph-/tool-based retrieval, a recognized RAG variant, without a vector store.
- **Embedding search is added in exactly two places, both optional and ablatable:**
  1. `semantic_search(query)` tool over method-level embeddings, used mainly in Stage 3 to locate authorization logic under arbitrary names (`checkOwner`, `Guard`, `TenantFilter`) when annotations and grep fail.
  2. Past-findings index (see §4b) to supply few-shot examples per candidate.
- Build core pipeline first; add embedding retrieval as an experiment arm. It introduces non-determinism and an extra evaluation dimension.
- Embedding model must also be local/open (e.g., a code-embedding model served next to the LLM). Index stored on disk per engagement.

### Candidate processing protocol
- The LLM step is a **fixed protocol with tool freedom inside each step**, not a free-roaming agent: (1) classify and restate source/sink, (2) walk the flow, (3) check sanitizers/encoders, (4) check framework defaults, (5) check authorization vs. profile, (6) record verdict. Reproducible, and failures can be attributed to a specific step.

## 4b. Use of the Team's Historical Pentest Reports

Reports are more sensitive than code; they never leave the machine (reinforces the privacy argument). Five roles:

1. **Ground truth** for real-engagement evaluation: map each historical finding to file:line once; score tool output against it.
2. **Pattern mining → custom candidate generators**: recurring findings (authz bypass idioms, deserialization patterns, custom sanitizer misuse) become Semgrep/CPGQL rules. Grounds the tool in what the team actually finds.
3. **Authorization profile schema**: the way reports describe access-control models defines the profile YAML fields.
4. **Few-shot exemplars via retrieval**: per candidate, retrieve the 2–3 most similar past findings (by vuln class, framework, code similarity) and include their code + verdict in the prompt. Expected to be the largest accuracy gain for local models.
5. **Report drafting in house style**: LLM writes finding descriptions matching team conventions. Low research value, high practical value; applied contribution.

**Guardrail — engagement split:** engagements used for rule mining and few-shot examples are excluded from the evaluation set. Record the split in `experiments/` before any evaluation run.

## 5. Do we need Joern (or any program-analysis tool)?

**Short answer: yes, some form of program analysis is necessary; Joern is a good default but not the only option.**

Why not LLM-only:
- Web apps are 50k–500k+ LOC; no local model holds that in context, and chunked LLM scanning loses interprocedural flow (controller → service → repository → raw SQL).
- LLM-only approaches in the literature show high recall but poor precision and non-determinism; the thesis contribution is stronger if the LLM reasons over verified facts.
- Entry-point and call-graph extraction is a solved problem for static analysis; using it is cheap and deterministic.

Options and recommendation:

| Tool | Pros | Cons | Role |
|---|---|---|---|
| **Joern** (Apache-2.0) | Open source, local, CPG + taint queries, Java mature, C# frontend exists, scriptable, no build needed for source frontends | C# frontend is young; source frontends lose some precision vs. compiled; Scala/JVM dependency; learning curve for CPGQL | **Primary data-flow engine** |
| CodeQL | Best-in-class precision, excellent Java and C# support | License forbids use on non-open-source code outside GitHub; needs full build; not fully "open" | **Baseline SAST in benchmark**, not part of the tool (license argument supports the thesis) |
| Semgrep OSS | Fast, easy rules, good C# coverage | OSS taint is intra-file only; cross-file is paid | **Candidate generator + free SAST baseline** |
| tree-sitter / LSP | Language-agnostic, robust to broken code, very fast | No data flow | Navigation tools for the LLM agent, fallback when CPG fails |
| Roslyn (C#) | Exact semantic model for C# | C#-only, custom code needed | Fallback/complement if Joern C# quality is insufficient |

**Decision rule:** run a 1–2 week spike building Joern CPGs for 2 Java and 2 C# apps and measuring entry-point and flow extraction quality. If C# is weak, keep Joern for Java and implement a Roslyn-based extractor (or Semgrep Pro-style rules + LLM navigation) for C# behind the same `CodeGraphProvider` interface. Document the spike results in the thesis (it is itself a useful finding).

## 6. LLM Choice and Privacy Argument

**Model shortlist (4 models: 3 primary from different families + 1 small baseline).** Three families is the minimum for a credible comparison; more than four multiplies evaluation runs without adding insight at MSc scale. Re-check current releases at spike time; the open-weight landscape changes monthly.

| Role | Model | Why |
|---|---|---|
| Primary code specialist | Qwen3-Coder (largest variant that fits), fallback Qwen2.5-Coder-32B-Instruct | Strongest open-weight code reasoning, long context, Apache-2.0, solid tool calling |
| General reasoner | DeepSeek-R1 distill 70B, or DeepSeek-V3 if it fits | Chain-of-thought helps the step-by-step taint walk; different training family |
| Western-license control | Llama 3.3 70B Instruct, or Mistral Devstral / Mistral Large | Different family, permissive license; some institutions prefer these |
| Small baseline | Qwen2.5-Coder-7B or Gemma 3 12B | Shows accuracy degradation with size; feeds the cost-vs-quality analysis |

Selection criteria: code reasoning quality, context length ≥32k, license permitting commercial use, reliable tool calling (verify per model before Stage 6), fit on the multi-GPU server at FP8 or AWQ 4-bit.

**Serving:** vLLM (throughput, OpenAI-compatible API, tool-call parsing). Ollama/llama.cpp only for quick local experiments.

**Optional upper bound:** one closed model run on public benchmark apps only, to quantify the accuracy cost of the privacy constraint.

**Privacy/defensibility points for the thesis:**
- Client source code is the most sensitive asset in a pentest; third-party API terms, data residency, retention, and training-use clauses are a contractual and legal risk (KVKK, GDPR, public-sector rules).
- Air-gapped operation is possible: Joern + local model + no telemetry.
- Reproducibility: pinned model weights give repeatable experiments; cloud models change silently.
- Cost predictability: GPU capex vs. per-token opex over many engagements.
- Optional controlled comparison: run one closed model only on public benchmark apps as an "upper bound" to quantify the privacy cost in accuracy (makes the argument honest).

## 7. Evaluation Design

**Datasets**
- Java, labeled: OWASP Benchmark v1.2 (2,740 test cases, official scorer, standard in SAST papers), securibench-micro, java-sec-code, Vul4J (real reproducible CVEs), WebGoat / Vulnado / JavaVulnerableLab (realistic apps).
- C#, labeled or semi-labeled: WebGoat.NET, DVCSharp API, intentionally vulnerable ASP.NET Core apps, plus real CVEs in open-source .NET web apps (nopCommerce, Orchard, Umbraco older versions) cross-checked with advisories.
- Realistic: 2–3 of the team's actual engagements (anonymized), where the manual pentest acts as ground truth plus adjudicated union of all tools.
- A01 is under-represented in synthetic benchmarks; plan to **author a small Broken Access Control micro-benchmark** (Java + C#, 30–60 cases with varied authz idioms). This is a tangible thesis contribution.

**Comparison arms**
1. Manual pentest with source (team).
2. New tool (full pipeline).
3. Ablations: Joern-only (no LLM), LLM-only (no CPG), pipeline without Authorization Profile, different models.
4. Free SAST (four tools):
   - Semgrep OSS — fast rule-based, intra-file taint only; "weak static" baseline.
   - CodeQL — strongest free engine, interprocedural taint, good Java + C#. Research license covers open-source benchmark apps but not client code; this license gap supports the privacy argument.
   - SpotBugs + FindSecBugs — Java-only classic included in most SAST papers; scores well on OWASP Benchmark.
   - SonarQube Community — industry default, both languages, weak taint; represents what dev teams already run.
   - Optional: Bearer, Security Code Scan (C#-only Roslyn analyzer). Skip language-mismatched tools.
5. Paid SAST: Checkmarx (team license). Verify whether results may be published; otherwise report as "Commercial SAST A". One paid tool is sufficient for MSc; add Fortify or Semgrep Pro only if a trial is easily available, do not block on it.

**Metrics**
- Precision, recall, F1 per vuln class and overall; OWASP Benchmark score (TPR − FPR).
- Candidate-to-finding reduction (how much triage work the LLM removes).
- Time and cost: wall-clock, GPU hours, tokens per finding; pentester minutes saved.
- Qualitative: findings unique to each arm, failure analysis (why LLM was wrong).
- Stability: variance across runs (temperature 0 and repeated runs).

## 8. Proposed Repository Layout (Python orchestrator)

```
Tez/
  thesis/            # LaTeX/Markdown thesis text, literature notes
  tool/
    ingest/          # build/framework detection, inventory
    graph/           # CodeGraphProvider interface, joern/, treesitter/, roslyn/
    surface/         # entry-point extractors (java_spring, java_jaxrs, cs_aspnetcore, ...)
    authz/           # evidence collectors + profile builder + schema
    candidates/      # per-vuln-class generators (CPGQL scripts, semgrep rules)
    slicing/         # flow dossier builder, truncation
    llm/             # model clients (vLLM/Ollama), prompts, tools, verdict schema
    report/          # SARIF, HTML, worklist
    store/           # SQLite/DuckDB schema
  benchmarks/        # dataset manifests, ground truth, scoring scripts, A01 micro-benchmark
  experiments/       # configs + results per run
```

## 9. Milestones

Detailed schedule lives in **SPRINT_PLAN.md** (6 sprints × 4 weeks, 2026-11-09 → 2027-04-25). Summary:

1. **Sprint 1 (Nov):** environment, Joern spike on Java + C#, backend decision, literature.
2. **Sprint 2 (Dec):** Java static core: candidate generators, flow slices, SARIF, OWASP Benchmark score.
3. **Sprint 3 (Jan):** LLM triage agent, precision gain over static arm.
4. **Sprint 4 (Feb):** Authorization profile and A01 micro-benchmark.
5. **Sprint 5 (Mar):** C# port and full SAST comparison. Minimum Viable Thesis line.
6. **Sprint 6 (Apr):** ablations, real-engagement validation, complete thesis and defense deck.

## 10. Risks and Mitigations

- Joern C# frontend immature → spike early, Roslyn fallback, abstraction layer.
- Local model too weak for multi-step reasoning → structured protocol, small steps, voting, try several models; report honestly.
- Context overflow on long flows → summarize pass-through steps, tool-calling instead of dumping code.
- Ground truth for real apps is incomplete → adjudicated union of all arms, report both strict and lenient metrics.
- Benchmark contamination (models trained on OWASP Benchmark) → include the self-authored A01 micro-benchmark and real engagement code; discuss in threats to validity.
- Commercial SAST access → confirm licenses early; otherwise rely on trial/academic licenses or scope the arm down.

## 11. Open Questions (to decide with the user)

- Can anonymized real-engagement code and findings be used in the evaluation?
- OWASP Top 10 edition to frame on: 2021 or 2025?
- Exact GPU inventory (count, VRAM) to finalize the model shortlist.
- Checkmarx edition/version and whether its results may be published (some licenses restrict benchmark publication; anonymize as "Commercial SAST A" if needed).

## 12. Decision Log

- 2026-11-01: Static analysis only; Java + C#; open-source local models only for the tool itself; Joern as default data-flow backend pending spike; SARIF as common output format for benchmarking.
- 2026-11-01: Hardware is a multi-GPU server/cluster → evaluate 70B-class models (Qwen3-Coder large, Llama 3.3 70B, DeepSeek-R1 distill 70B) alongside 32B; use multi-model voting as an experiment arm.
- 2026-11-01: Paid SAST arm = Checkmarx. Free SAST arm = Semgrep OSS, SpotBugs+FindSecBugs, CodeQL, SonarQube Community.
- 2026-11-01: First-class frameworks = Spring Boot and ASP.NET Core. Legacy stacks (JSP/Servlets, WebForms, MVC 5) deferred; extractors remain pluggable.
- 2026-11-01: MSc, 6–9 months. Milestone plan in §9 (about 20 weeks of build + eval) fits; writing runs in parallel from week 12.
- 2026-11-04: No vector-DB RAG in the core pipeline; structured retrieval via tools. Embedding search only for Stage 3 auth discovery and past-findings few-shot retrieval, both as ablatable arms. Historical pentest reports used for ground truth, rule mining, profile schema, few-shot examples and report drafting, with a strict engagement split between mining and evaluation.
- 2026-11-01: Model set fixed at 4 (Qwen3-Coder, DeepSeek-R1 distill 70B, Llama 3.3 70B, small Qwen2.5-Coder-7B/Gemma 3 12B). Free SAST arm = Semgrep OSS, CodeQL, SpotBugs+FindSecBugs, SonarQube Community. Paid arm = Checkmarx only. Agent toolset defined in §4 Stage 6: read-only navigation, Joern flow queries, application-knowledge lookups, structured verdict; no shell/network/write tools.
