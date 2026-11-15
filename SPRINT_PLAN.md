# Sprint Plan — LLM-Assisted Vulnerability Detection for Java/C# Web Applications (MSc Thesis)

Version 1.0 · Prepared 2026-11-04 · Companion to THESIS_PLAN.md

## 1. Planning Parameters

| Parameter | Value |
|---|---|
| Total duration | 6 months (24 weeks) |
| Number of sprints | 6 |
| Sprint length | 4 weeks (1 calendar month) |
| Planned start | Monday 2026-11-09 (adjust if needed; all dates shift together) |
| Planned end | Sunday 2027-04-25 |
| Capacity assumption | 10–12 hours/week, part-time alongside employment |
| Working rhythm inside a sprint | Weeks 1–2 build · Week 3 measure and integrate · Week 4 write, review, buffer |
| Writing commitment | Every sprint ends with a thesis text deliverable; no "writing-only" phase except Sprint 6 |

## 2. Governance

**Ceremonies (per sprint)**
- Sprint Planning (Day 1, 1 h): confirm goal, objectives and acceptance criteria below; trim scope if capacity changed.
- Mid-Sprint Checkpoint (end of Week 2, 30 min, self-review): are the build objectives on track? If not, invoke the cut rule.
- Sprint Review (last week, with advisor): demonstrate deliverables against acceptance criteria.
- Retrospective (last day, 20 min, written in `experiments/retro-SN.md`): what to keep, what to change.

**Rules**
- One primary goal per sprint. Stretch objectives are explicitly optional and are dropped first.
- Cut rule: if an objective is not achievable by the mid-sprint checkpoint, descope it to the next sprint rather than extending hours.
- Definition of Done applies to every deliverable: code committed, result recorded in `experiments/`, and one paragraph written in the thesis draft.
- Change control: any change to sprint goals is recorded in the Decision Log of THESIS_PLAN.md with a date.

**Minimum Viable Thesis (MVT) line:** the thesis is defensible at the end of Sprint 5. Sprint 6 adds depth and polish.

---

## Sprint 1 — Foundation and Feasibility Spike

| | |
|---|---|
| Period | 2026-11-09 → 2026-12-06 |
| Theme | Environment, literature, and proof that the static-analysis backend works for both languages |
| Sprint Goal | Establish the working environment and decide the code-analysis backend for Java and C# based on measured evidence. |

**Objectives**
1. Install and script Joern; build CPGs for 2 Java (Spring Boot) and 2 C# (ASP.NET Core) applications.
2. Implement entry-point extraction (Stage 2) for Spring Boot and ASP.NET Core; produce endpoint tables.
3. Measure C# frontend quality (parse errors, missing methods, sample flows); decide Joern vs. Roslyn fallback.
4. Stand up vLLM with one model (Qwen3-Coder or Qwen2.5-Coder-32B); verify tool-calling via the OpenAI-compatible API.
5. Create repository skeleton (`tool/`, `benchmarks/`, `experiments/`, `thesis/`).
6. Literature review: collect and annotate 20 core papers (LLM for vulnerability detection, CPG/taint analysis, SAST benchmarking).

**Deliverables**
- Endpoint tables (CSV/JSON) for all 4 applications.
- Spike report `experiments/S1-spike.md` with C# decision.
- Running model endpoint and a smoke-test script.
- Annotated bibliography (20 entries).

**Acceptance Criteria**
- ≥90% of manually identified controllers/endpoints in the 4 apps appear in the generated tables.
- C# backend decision logged in Decision Log with supporting numbers.
- Model answers a tool-calling test prompt correctly 5/5 times.

**Thesis Output:** Chapter 2 (Background and Related Work) first draft, 6–8 pages.

**Risks:** Joern C# frontend immature → fallback decision is the explicit purpose of this sprint. Model serving issues on the cluster → use a smaller model temporarily; serving is not on the critical path until Sprint 3.

---

## Sprint 2 — Java Detection Pipeline (Static Core)

| | |
|---|---|
| Period | 2026-12-07 → 2027-01-03 (contains holidays; capacity reduced ~25%, scope sized accordingly) |
| Theme | Deterministic candidate generation and data-flow slicing on Java |
| Sprint Goal | Produce a Joern-only detection arm for Java that emits SARIF and scores on the OWASP Benchmark. |

**Objectives**
1. Candidate generators (Stage 4) for SQLi, XSS, insecure deserialization, SSRF, XXE: CPGQL queries plus Semgrep OSS rules where simpler.
2. Slice extractor (Stage 5): source→sink flows, flow dossier format with truncation strategy.
3. Findings store (SQLite) and SARIF 2.1 writer (Stage 7).
4. Run on OWASP Benchmark v1.2; integrate the official scorer.
5. Stretch: run on WebGoat and Vulnado for realistic-app sanity check.

**Deliverables**
- `tool/candidates/`, `tool/slicing/`, `tool/report/`, `tool/store/` modules.
- `experiments/S2-benchmark-joern-only.md` with Benchmark score per category.
- 20 sample flow dossiers reviewed manually for correctness.

**Acceptance Criteria**
- SARIF output loads in the OWASP Benchmark scorer and in a SARIF viewer without errors.
- Benchmark score recorded for SQLi and XSS at minimum; deser/SSRF/XXE generators produce candidates on at least one realistic app.
- Manual review of 20 dossiers: ≥16 contain the complete relevant path.

**Thesis Output:** Chapter 3 (Methodology) sections on architecture, Stages 0–2, 4–5, 7; 6–8 pages.

**Risks:** Flow explosion on large apps → cap flows per sink, prioritize shortest paths. Holiday capacity → stretch objective dropped first.

---

## Sprint 3 — LLM Triage Agent

| | |
|---|---|
| Period | 2027-01-04 → 2027-01-31 |
| Theme | The core contribution: LLM reasoning over verified static facts |
| Sprint Goal | Demonstrate a measurable precision gain over the Joern-only arm on the OWASP Benchmark using local open-weight models. |

**Objectives**
1. Triage v1: flow dossier in prompt, fixed six-step protocol, structured verdict schema (`record_verdict`).
2. Triage v2: tool-calling agent with the read-only toolset (navigation, Joern flows, application knowledge); tool-call budget and full logging.
3. Framework knowledge sheets for Spring Boot (Thymeleaf escaping, JPA binding, Jackson defaults).
4. Run both versions with the primary model on the Benchmark; add the second model (DeepSeek-R1 distill) if time allows.
5. Failure analysis: classify 50 wrong verdicts by protocol step.

**Deliverables**
- `tool/llm/` module (clients, prompts, tools, schemas).
- `experiments/S3-triage.md`: Benchmark scores for Joern-only vs. v1 vs. v2, tokens and latency per candidate.
- Failure taxonomy table.

**Acceptance Criteria**
- Pipeline v2 improves precision over Joern-only on SQLi and XSS without recall dropping more than 5 points.
- Every LLM run is reproducible from a stored config (model, quantization, temperature, prompt version).
- Tool-call logs stored for 100% of candidates.

**Thesis Output:** Chapter 3 sections on Stage 6 and the agent protocol; Chapter 4 (Evaluation) skeleton with first results table; 6–8 pages.

**Risks:** Model too weak for multi-step reasoning → reduce step size, add voting, document honestly. Tool-calling instability → fall back to v1 for the Benchmark numbers and keep v2 as a secondary result.

---

## Sprint 4 — Authorization Profile and Broken Access Control

| | |
|---|---|
| Period | 2027-02-01 → 2027-02-28 |
| Theme | Teaching the model each application's authorization mechanics; the novelty claim for A01 |
| Sprint Goal | Detect Broken Access Control issues using a human-confirmed Authorization Profile, evaluated on a self-authored micro-benchmark. |

**Objectives**
1. Evidence collectors for authorization artifacts (Spring Security config, annotations, filters, principal access, ownership idioms).
2. Profile builder (Stage 3): LLM-generated YAML profile plus a simple human review step (edit file, confirm).
3. A01 micro-benchmark: 30–60 labeled Java cases covering missing authz, IDOR, mass assignment, path traversal, with varied authz idioms.
4. A01 check in the triage agent using `get_auth_profile()` and `get_endpoints()`.
5. Stretch: `semantic_search` tool over method embeddings for authorization discovery; measure its effect.

**Deliverables**
- `tool/authz/` module and profile schema.
- `benchmarks/a01-micro/` with ground truth and README (publishable artifact).
- Profiles for 2 realistic apps with review time recorded.
- `experiments/S4-a01.md` with precision/recall on the micro-benchmark, with and without the profile.

**Acceptance Criteria**
- Profile review by a pentester takes ≤30 minutes per application.
- Pipeline with profile outperforms pipeline without profile on the micro-benchmark.
- Micro-benchmark has documented ground truth and is reproducible by a third party.

**Thesis Output:** Chapter 3 section on Stage 3; Chapter 4 A01 results; 5–7 pages.

**Risks:** Authoring the micro-benchmark takes longer than expected → start at 30 cases, grow only if time remains. Profile quality varies by app → human review step is the mitigation and is part of the method.

---

## Sprint 5 — C# Port and SAST Comparison

| | |
|---|---|
| Period | 2027-03-01 → 2027-03-28 |
| Theme | Second language and the full tool comparison. MVT line at the end of this sprint |
| Sprint Goal | Run the complete pipeline on C# and produce the comparison table across all tool arms. |

**Objectives**
1. Port entry-point extraction, candidate generators and framework sheets to ASP.NET Core (Joern or Roslyn backend per Sprint 1 decision).
2. Run the pipeline on C# applications (WebGoat.NET, DVCSharp API, one additional app).
3. Run free SAST baselines on all datasets: Semgrep OSS, CodeQL, SpotBugs+FindSecBugs (Java), SonarQube Community.
4. Run Checkmarx on all datasets; confirm publication rights.
5. Build the consolidated results table: precision, recall, F1, Benchmark score, runtime, tokens/GPU hours.

**Deliverables**
- C# extractors and rules under `tool/surface/`, `tool/candidates/`.
- `experiments/S5-comparison.md` with the full comparison table and per-category breakdown.
- Scripts to re-run every baseline from a single command.

**Acceptance Criteria**
- C# pipeline runs end to end and produces SARIF on all three C# apps.
- Every tool arm has results on every dataset where the tool supports the language; gaps are documented.
- Comparison table is complete enough to answer RQ1 and RQ3.

**Thesis Output:** Chapter 4 main results, Chapter 5 (Discussion) outline; 6–8 pages. **Full draft of Chapters 1–4 exists at sprint end.**

**Risks:** C# backend weaker than Java → report reduced C# coverage as a finding; do not let it block the comparison. Checkmarx access or publication restrictions → anonymize as "Commercial SAST A".

---

## Sprint 6 — Depth, Real-World Validation and Thesis Completion

| | |
|---|---|
| Period | 2027-03-29 → 2027-04-25 |
| Theme | Ablations, team validation, and the finished thesis |
| Sprint Goal | Complete the evaluation with ablations and real-engagement validation, and deliver the full thesis draft for advisor review. |

**Objectives**
1. Ablations: Joern-only vs. LLM-only vs. full; with/without profile; 2–3 models; voting.
2. Real-engagement validation on 1–2 anonymized engagements with the team's manual findings as ground truth; record pentester triage time with and without the tool.
3. Past-report few-shot retrieval experiment (if the engagement split allows).
4. Threats to validity, limitations, future work (DAST integration, more languages).
5. Complete thesis: Chapters 1, 5, 6; abstract; final figures and tables; formatting per institution template.
6. Defense preparation: 20-slide presentation and demo script.

**Deliverables**
- `experiments/S6-ablations.md`, `experiments/S6-engagements.md`.
- Complete thesis PDF submitted to advisor.
- Presentation deck and reproducible demo.

**Acceptance Criteria**
- All research questions RQ1–RQ4 have a stated answer with supporting evidence.
- Thesis passes the institution's formatting checklist.
- Advisor review requested at least 2 weeks before the submission deadline.

**Thesis Output:** Full thesis, 60–90 pages.

**Risks:** Real-engagement data unavailable → RQ4 answered qualitatively from a pentester survey; state clearly. Time pressure → ablations reduced to the two most informative (with/without LLM, with/without profile).

---

## 3. Sprint Summary

| Sprint | Period | Goal | Key output | Thesis pages |
|---|---|---|---|---|
| 1 | Nov 9 – Dec 6, 2026 | Environment and backend decision | Endpoint tables, spike report | Ch. 2 draft |
| 2 | Dec 7 – Jan 3 | Java static core, SARIF, Benchmark | Joern-only Benchmark score | Ch. 3 part 1 |
| 3 | Jan 4 – Jan 31, 2027 | LLM triage agent | Precision gain over static arm | Ch. 3 part 2, Ch. 4 skeleton |
| 4 | Feb 1 – Feb 28 | Authorization profile and A01 | A01 micro-benchmark and results | Ch. 3 part 3, Ch. 4 A01 |
| 5 | Mar 1 – Mar 28 | C# port and SAST comparison | Full comparison table (MVT) | Ch. 4 main, Ch. 5 outline |
| 6 | Mar 29 – Apr 25 | Ablations, validation, completion | Complete thesis and defense deck | Full thesis |

## 4. Tracking

- Progress is tracked by ticking objectives in this file and recording results under `experiments/`.
- Each sprint review produces a one-paragraph status note appended below.

### Status Notes
- (empty)
