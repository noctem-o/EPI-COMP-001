# EPI-COMP-001 — Epistemic Composition Under Changing Evidence

**Status:** proposed experiment / methodology draft — **no benchmark implementation or results yet**  
**Scope:** controlled, inference-only evaluation of stateful language-model agents  
**Primary question:** *Can an agent reliably combine individually tractable epistemic operations across time, especially when evidence changes and conclusions depend on multiple sources?*

> This is an exploratory research project motivated by agent reliability and long-horizon alignment. It does **not** test, establish, or certify alignment or safety. Parameter values below are **proposals**, not empirical findings. Freeze a versioned protocol before running the confirmatory evaluation.

## 1. Motivation and research contribution

Models can correctly retrieve facts, identify revisions, and reason about evidence in isolation while failing to coordinate these operations in longer workflows. In [He et al. (2026)](https://arxiv.org/abs/2609.19465), compositional reasoning is represented as a dependency graph with three levels: single-skill chains, multi-skill chains, and branch–merge graphs. Their RL post-training results show that improving atomic skills does not reliably produce good performance on composed tasks. Their experiments concern data structures and a tool-calling pilot, **not epistemic memory**.

EPI-COMP-001 adapts that **diagnostic framing**, rather than reproducing their RL study. It asks whether reliable epistemic primitives compose into reliable behaviour when an agent must track evidence, revisions, support, and uncertainty across episodes.

This is a crowded adjacent research area, not a claim to have invented dependency-aware memory. In particular:

- [StateGraph / StateChangeBench (2026)](https://arxiv.org/abs/2611.02912) studies selective downstream revision and independent support after state changes.
- [PACMI (2026)](https://arxiv.org/abs/2610.05732) studies provenance-aware cascading memory invalidation.
- [MEME (2026)](https://arxiv.org/abs/2605.12477) evaluates evolving, multi-entity memory, including cascade and absence tasks.
- [MemDerive (2026)](https://arxiv.org/abs/2610.25373) uses exact derivation graphs to diagnose memory failures over evolving event histories.

**Potential contribution:** a controlled **factorial comparison of epistemic composition level × state-management capability**, with matched information access, atomic subtask probes, counterfactual support tests, and reproducible error attribution. Whether this is sufficiently novel or valuable must be established empirically.

## 2. Research questions and falsifiable hypotheses

| ID | Question / proposed hypothesis | What would count against it? |
| --- | --- | --- |
| **H1 — Composition bottleneck** | End-to-end accuracy declines as epistemic operations are composed, beyond changes attributable to individual-operation difficulty. | Composed accuracy remains well explained by matched atomic performance, with no additional pattern by switching or branch–merge structure. |
| **H2 — Currentness intervention** | A deterministic current-state resolver improves current-value and supersession accuracy over flat memory (C vs B). | No reliable improvement, or gains disappear under matched evidence and token budgets. |
| **H3 — Dependency intervention** | An explicit dependency-aware view improves correct downstream recheck decisions over currentness alone, particularly at Level III (D vs C). | No improvement on held-out Level III cases, or improvement is offset by incorrect invalidation of still-supported claims. |
| **H4 — Independent support** | Provenance-aware dependency handling reduces false cascades where genuinely independent evidence survives an upstream retraction. | D incorrectly flags/retracts independently supported claims as often as, or more often than, C. |

H1 is a **descriptive diagnostic** unless a stronger causal design is implemented. H2–H4 are intervention comparisons. They are not hypotheses about RL efficacy; training is out of scope for the initial experiment.

### Scope limits

- Synthetic, fully auditable worlds first; no production user memory, real-world truth adjudication, or live browsing in the main test.
- No assumption that two agreeing citations are independent: copied or common-origin evidence counts as **one provenance root**.
- Rejected premises do not automatically make downstream conclusions false; they can instead make them **unsupported / requiring recheck**.
- Use a single-model, single-agent baseline first. Multi-agent branching and learned policies are optional later extensions.

## 3. Formal task model

Represent an episode as an event stream **E = (e1, …, et)** and a directed acyclic **support / computation graph G = (V, A)**. An event can assert, revise, retract, or qualify evidence. A claim may have multiple possible **justifications**; each justification may require all its premises (AND), while alternative justifications may independently support the same claim (OR).

Keep these concepts distinct:

1. **World value:** the value implied by the synthetic ground-truth rules at a query time.
2. **Evidence status:** whether the *available* evidence supports, contradicts, leaves undetermined, or conflicts about a proposition.
3. **Revision status:** whether a recorded assertion is current, historical, superseded, or awaiting recomputation.
4. **Action policy:** whether to assert an answer, request a recheck, or abstain.

An invalidated premise does **not** imply its conclusion is false. A valid alternative justification can preserve the conclusion. An unrelated but semantically similar statement is not a dependency.

### Required primitive operations

| Primitive | Example | Oracle target |
| --- | --- | --- |
| Currentness | Which version applies at time t? | Scoped version and validity |
| Supersession | Does event e replace earlier evidence? | Explicit update / retraction relationship |
| Provenance | Are two citations independent? | Distinct underlying origin groups |
| Support assessment | Does the remaining evidence license claim c? | Valid justification sets |
| Dependency propagation | Which derived claims need recomputation? | Exact affected set, excluding surviving support |
| Epistemic action | Assert, recheck, or abstain? | Rule-based decision for the available evidence |

**Important:** construct the oracle in code from symbolic events and rules. Do not use the language model being evaluated as the ground-truth judge.

### Example

At t1, Source A establishes that a prerequisite is satisfied; derived Claim C uses that prerequisite. At t2, A is retracted. C may then be *needs-recheck*, **not necessarily false**. If Source B independently establishes the same prerequisite within the same scope, C may remain supported. But if B merely copied A, B is not independent corroboration.

The dataset must include all three cases, together with hard negatives where an apparently related statement is *not* a causal/support dependency.

## 4. Composition levels

Keep the paper-inspired composition axis separate from evidence-update type and task length:

| Level | Dependency structure | Example | Bottleneck |
| --- | --- | --- | --- |
| **I: single-skill chain** | One repeated operation | Apply several sequential revisions to one scoped assertion | Horizon / state preservation |
| **II: multi-skill chain** | Several different operations in sequence | Retrieve → resolve supersession → check provenance → decide | Skill switching / intermediate-state handoff |
| **III: branch–merge DAG** | Several evidence paths converge; non-local dependencies | Evaluate two evidence branches, discount a copied source, merge support, then reconsider a derived claim after an update | Non-local dependency and selective revision |

A Level III episode needs a genuine convergent dependency, not merely parallel tasks whose final answers are concatenated. Test multiple **valid topological orders** where applicable; do not score only one arbitrary execution order as correct.

### Orthogonal perturbations

- **Horizon / node count:** number of epistemic operations, not just prompt tokens.
- **Switch count:** number of changes between primitive operation types.
- **DAG depth and merge fan-in:** path length, branch count, number of premises needed at the merge.
- **Update type:** replacement, retraction without replacement, scoped exception, conflict, unchanged evidence.
- **Support structure:** AND prerequisites, OR alternatives, common-source copies, mixed AND/OR.
- **Temporal distance:** number of intervening events between evidence and query.
- **Retrieval difficulty:** distractor count, lexical overlap, distance between dependent events.
- **Surface rendering:** paraphrases, entity aliases, and shuffled event order when chronological timestamps preserve meaning.

Use controlled pairings to avoid making Level III simply longer or noisier than Levels I and II. Report both **complexity-matched** comparisons and natural increasing-complexity curves.

## 5. Intervention arms

All arms should receive the **same episode events**, answer schema, model, decoding parameters, and (as closely as possible) information and token budgets.

| Arm | Condition | Permitted capability |
| --- | --- | --- |
| **A — History-only** | Raw event history within the fixed budget | No persistent derived memory or specialised state API |
| **B — Flat memory** | Basic event/claim retrieval | No deterministic currentness or dependency resolution |
| **C — Currentness** | Same retrieval plus deterministic as-of / supersession resolution | No downstream dependency propagation |
| **D — Dependency-aware** | C plus typed support dependencies, alternative-support checks, and affected/recheck view | No hidden access to the oracle's final answer |

**Primary causal contrast: D vs C**, on held-out Level III downstream-recheck tasks. **Secondary:** C vs B for supersession; B vs A for retrieval; level/complexity interactions in all arms.

A is a useful operational baseline, but it is not an information-matched comparison if history exceeds the context budget. Report truncation and missing-evidence rates. B/C/D must share a retrieval interface and budget; their key difference should be the state semantics, not a larger evidence allowance.

**Avoid answer leakage:** do not inject the gold label, gold final answer, or a precomputed “correct action” into any agent-visible view. If D exposes a deterministic *needs-recheck* flag, measure that system capability separately from the model's independent reasoning skill. Include an ablation with the flag withheld, while retaining the dependency evidence, to distinguish infrastructure performance from model composition.

### Additional diagnostic controls

- **Gold-evidence control:** provide exactly the necessary evidence to the same model, to separate retrieval failure from downstream reasoning failure.
- **Oracle-state upper bound:** feed perfectly resolved *intermediate states* while withholding the final answer; measure the remaining composition difficulty.
- **No-dependency / shuffled-edge ablation:** remove or scramble support edges while preserving the rest of the interface; flag as a diagnostic stress test, not an honest baseline.
- **Matched-context ablation:** pad or budget-match conditions and report actual tokens.
- **Independent-support negative controls:** duplicate-source and alternative-support cases must not be conflated.

## 6. Synthetic environment and ground truth

Start with a deterministic, seeded scenario generator. A plain Python reference implementation is sufficient; this experiment must **not** depend on Endophasia to establish validity. An Endophasia adapter can follow once the oracle and harness tests pass.

Each generated scenario should contain:

- **Private symbolic specification:** entities, events, timestamps, scope, rules, typed dependency DAG, source-origin groups, and exact query-time gold.
- **Agent-visible rendering:** short natural-language records with stable public evidence IDs, fictional entities, and distractors.
- **Queries:** current, historical, counterfactual, downstream support, and epistemic-action questions.
- **Oracle traces:** justification sets, correct affected/unaffected nodes, correct surviving support, and minimal supporting evidence families.
- **Metadata:** generator version, template family, seed, graph size/depth/fan-in, update class, expected action, and rendering version.

Use two time fields when needed: **valid time** (when a fact applies in the world) and **observation time** (when the agent learns it). Apply explicit scope and precedence rules. If equal-priority evidence conflicts and no deterministic rule resolves it, the correct epistemic result is **conflicted**, not a guessed winner.

For an AND justification, all required premises must survive. For OR support, a single valid independent justification may suffice. Preserve the distinction between **same underlying source repeated twice** and **two genuinely independent evidential origins**.

### Example machine-readable query / answer contract (proposed)

~~~json
{
  "query_id": "example-017-q3",
  "as_of": "2026-01-15T12:00:00Z",
  "question": "Is claim C currently justified, and may it be asserted?",
  "response": {
    "value": null,
    "evidence_status": "undetermined",
    "needs_recheck": true,
    "decision": "recheck",
    "evidence_ids": ["evt-008", "evt-012"]
  }
}
~~~

- **value:** canonical answer or null if not justified.
- **evidence_status:** supported / contradicted / undetermined / conflicted. Do not equate contradicted with a complete truth theory.
- **needs_recheck:** whether previously derived support must be evaluated again because a dependency changed.
- **decision:** assert / recheck / abstain, according to a **published, fixed policy**.
- **evidence_ids:** agent-cited IDs; a set of valid minimal witnesses may exist, so do not demand one canonical chain if several justify the answer.

The generator's private rules and gold answer must not be embedded in public test prompts, retrieval indices, file names, or other agent-visible metadata.

### Proposed scenario mix

The following are *target proportions* for balanced sampling, not claims about natural-world frequencies:

| Challenge | Target share of episodes |
| --- | ---: |
| Explicit supersession / updated current value | 20% |
| Retracted premise with genuinely unsupported dependent claim | 20% |
| Surviving independent alternative support | 20% |
| Correlated evidence / same-origin duplicate sources | 20% |
| Scoped exceptions, unresolved conflicts, and irrelevant updates | 20% |

Cross these challenge categories with composition level and difficulty wherever semantically feasible. If a cell is impossible, document it; do not silently substitute another case. Keep some **unchanged** questions so a model cannot succeed by always choosing “recheck.”

## 7. Candidate parameters — calibration first

Numbers are starting proposals to refine after a *development-only* calibration. Freeze them in a dated protocol and commit hash before looking at the held-out evaluation.

| Parameter | Calibration pilot | Main evaluation proposal |
| --- | --- | --- |
| Composition levels | I, II, III | I, II, III |
| Difficulty tiers | Easy / medium / hard | Easy / medium / hard, frozen |
| Instances per level × tier | **12** | **64** (confirm after variance/power review) |
| Total unique instances | **108** | **576** |
| Intervention arms | A, B, C, D | A, B, C, D |
| Agent evaluations (one run each) | **432** | **2,304** |
| Node counts (illustrative tiers) | 3–4 / 6–8 / 10–16 | Frozen per-level distributions from pilot |
| Level II switch counts | 1 / 2–3 / 4–6 | Frozen feasible counts per tier |
| Level III branch fan-in | 2 / 2–3 / 3–4 | Frozen; include independent AND/OR cases |
| Distractor records | 0 / 4 / 12 | Frozen and crossed where feasible |
| Temperature | 0 or minimum supported | Identical per arm; document effective value |
| Sampling seed | Fixed if supported | Paired / recorded; randomise arm execution order |
| Output cap | 512 tokens initially | Increase only if calibration shows truncation |
| Context budget | Same configured limit per arm | Same limit; record actual used / truncated tokens |
| Runtime | One pinned local model | Pinned model, quantisation, server, and prompt |
| Stochastic repeats | Optional development diagnostic | Predeclare separately if used; cluster by instance |

**Arithmetic:** 3 levels × 3 tiers × 64 unique instances = 576 paired instances; 576 × 4 arms = 2,304 model evaluations. Atomic probes, diagnostic ablations, failed-run retries, and any repetitions are **extra** and must be budgeted separately. The proposed sample count is *not* a demonstrated power calculation.

Use balanced template families and generator seeds across cells. The 64 instances per cell should come from multiple independent template families, not 64 superficial paraphrases of one graph.

### Suggested holdout policy

- **Development/calibration:** tune wording, budgets, and difficulty using separate seeds and template instances.
- **Frozen test:** reserve distinct generator seeds and, critically, **held-out scenario templates / graph motifs** to probe structural transfer, not just random-instance memorisation.
- **Post-freeze:** no prompt, rule, grader, or threshold tuning on test results. Log any deviation and designate the affected run exploratory.

For a stronger follow-up, add out-of-distribution tests: longer horizons, unseen graph motifs, different paraphrase generators, and a second model family.

## 8. Execution protocol

1. **Implement the oracle first.** Unit-test temporal precedence, retractions, conjunction/disjunction support, duplicates, independent alternatives, scopes, and no-op updates. Use hand-verified counterexamples.
2. **Validate generated scenarios.** Reject cycles, contradictory gold metadata, unreachable query targets, ambiguous source identity, and accidental answer leaks. Record rejection counts.
3. **Calibrate on development data.** Check parsing rates, task difficulty (avoid floor/ceiling), context lengths, latency, and whether the oracle can reproduce every answer.
4. **Freeze the design.** Commit dataset-generator revision, scoring code, prompts, model identifier and hash, quantisation, inference flags, fixed action policy, sample count, randomisation, and primary analysis.
5. **Run paired episodes.** Every arm sees the same underlying instance; randomise arm ordering. Reset memory, session, KV/cache state and relevant tool state between instances; explicitly test for cross-instance leakage.
6. **Collect full receipts.** Record model/server version, seed, configuration, case IDs, returned evidence, parsed output, timestamps, tool calls, memory state (if applicable), latency, token counts, errors and hashes.
7. **Score mechanically.** Do not score gold by the same model under test. Preserve malformed outputs as failures; report tool/runtime failures separately rather than silently dropping them.
8. **Diagnose errors.** Replay or inspect relevant atomic states, retrieved evidence, and intermediate dependency decisions. Classify failures conservatively, with an “unknown” bucket.
9. **Release reproducible artifacts.** Publish the frozen config, generator, verifier, schemas, aggregated findings and anonymised/synthetic traces. Keep the held-out set undisclosed until protocol lock or archive a hash before publication.

Use immutable evidence/receipt IDs where possible. The initial experiment is **inference-only**; fine-tuning and RL are a different phase requiring a separate preregistration.

## 9. Measurements and analysis

### Primary endpoint

**Level III downstream-decision exact correctness:** for each held-out case, does the final decision **and** needs-recheck status match the deterministic oracle? Compare D with C on the **same instances**.

Report the paired difference in percentage points, a 95% confidence interval **clustered by scenario/template family as appropriate**, and a paired significance analysis (e.g. an exact McNemar test for a prespecified binary endpoint). If repeated runs are added, keep each underlying instance as the pairing/cluster unit.

### Secondary metrics

| Metric | What it diagnoses |
| --- | --- |
| Current-value exact accuracy | Does the answer reflect the relevant version at query time? |
| Historical-query accuracy | Can the agent preserve past values without treating them as current? |
| Evidence-status accuracy / macro-F1 | Supported, contradicted, undetermined, conflicted |
| Recheck precision / recall / F1 | Avoiding both missed dependencies and unnecessary cascades |
| False-cascade rate | Fraction of still-supported unaffected claims incorrectly invalidated |
| Missed-impact rate | Fraction of truly affected downstream claims left accepted |
| Surviving-support specificity | Correctly keeping claims justified by a valid independent alternative |
| Duplicate-origin error rate | Treating copied evidence as independent corroboration |
| Provenance support validity | Whether cited evidence suffices under the gold rules |
| Retrieval sufficiency / stale-context share | Whether required evidence arrived and obsolete evidence contaminated context |
| Atomic skill accuracy | Same primitives tested individually at matched state difficulty |
| Token use, latency, tool errors | Cost and operational reliability |

**Composition gap (diagnostic, not causal):** measure atomic primitive accuracy on matched inputs, derive a clearly labelled independence-model expectation for a complete sequence (for example, product of estimated per-step success probabilities), and compare it to observed end-to-end exact correctness. The product model is a **fragile assumption** because errors and induced states are dependent; report both components, uncertainty and deviations. Do not interpret the residual as proof of a unique “composition faculty.”

**Structural analysis:** plot accuracy against graph size, maximum dependency depth, switch count, branch fan-in, time since update, and distractor load. Use grouped estimates and interval bands; avoid treating many correlated queries from one scenario as independent observations.

**Error taxonomy:** missing retrieval; wrong currentness; mistaken provenance independence; incorrect support logic; missed downstream propagation; false cascade; unsupported assertion; improper abstention; format/parser failure; model truncation; runtime/tool failure; undetermined.

### Decision rules (proposed, not statistically powered claims)

- **Proceed to main evaluation** only after oracle tests pass, no gold leakage is found, and pilot strata avoid pervasive floor/ceiling effects.
- **Evidence favouring H3:** D exceeds C on the prespecified Level III primary endpoint, with uncertainty reported, **without a material deterioration** in false-cascade rate.
- **Practical engineering gate:** tentatively target ≥10 percentage-point paired improvement with ≤5 percentage-point false-cascade deterioration, but treat these as **provisional utility thresholds** until costs and sample-size planning are complete.
- **Inconclusive:** wide intervals, high failure rates, leakage, mis-matched budgets, or improvement only when the final gold-like flag is explicitly supplied.
- **Negative result:** a sound result even if D does not improve accuracy; publish the full protocol and failure breakdown.

A threshold crossing on one small synthetic dataset is **not** a generalisation or safety claim. Make separate inferential statements about the system, the model, and their combination.

## 10. Validity threats and mitigations

| Threat | Mitigation |
| --- | --- |
| The dependency-aware view reveals the answer | No final gold field; provide separately scored interface-only vs agent-reasoning ablations |
| More tokens/evidence in D than in C | Shared retrieval and capped context; record exact tokens and evidence visibility |
| “Always recheck” yields high recall | Score specificity, unchanged examples, AND/OR support survival and decision accuracy |
| Spurious source agreement | Record common-origin groups; compare independent vs copied corroboration |
| Confounding size with structure | Match node count and distractors across levels; report stratified analyses |
| Synthetic template contamination or overfitting | Fictional entities, held-out motifs and paraphrases, documented splits |
| Unverified or self-contradictory oracle | Formal invariants, independent unit/property tests, hand-audited samples |
| Multiple statistical comparisons | Declare one primary contrast/endpoint; label exploratory analyses and adjust where appropriate |
| Local inference nondeterminism | Pin builds/settings; log seeds and repeat a subset to quantify variability |
| Attractive but ungrounded alignment claims | Explicit scope limits; measured reliability is a component, not a proof, of alignment |

## 11. Proposed repository layout (future work)

The README specifies a research design; none of these paths or commands should be assumed to exist yet.

~~~text
EPI-COMP-001/
├── README.md
├── protocol/
│   ├── preregistration.md
│   └── pilot.yaml
├── configs/
│   └── experiment-v1.yaml
├── src/epi_comp/
│   ├── generate.py
│   ├── oracle.py
│   ├── interventions.py
│   ├── run.py
│   ├── score.py
│   └── report.py
├── tests/
│   ├── test_oracle.py
│   └── test_generation.py
├── schemas/
│   ├── scenario.schema.json
│   └── result.schema.json
├── data/              # synthetic, versioned manifest + splits
└── results/           # immutable run manifests and generated reports
~~~

Prefer a minimal, typed Python implementation for the first pilot. Support a reproducible CLI eventually, but **do not** present an unimplemented command as runnable. Endophasia adapters should consume/export the same scenario and result schemas, without changing the oracle or score.

## 12. Concrete first milestones

1. **M0 — Protocol review:** identify competing hypotheses, establish primary endpoint, audit related work and decide whether the project answers a distinct question.
2. **M1 — Oracle + generator:** implement typed evidence/update/support semantics; add property tests and 10 manually annotated adversarial scenarios.
3. **M2 — Tiny smoke run:** one model, all four arms, a handful of cases per level; validate state resets, logging, output schema, and absence of label leakage.
4. **M3 — Calibration:** run the 108-instance development pilot; revise difficulty and conduct a sample-size/feasibility review.
5. **M4 — Frozen evaluation:** preregister and execute the held-out paired experiment; publish CIs, error analysis and negative results.
6. **M5 — Extensions only if warranted:** second model family, branch-order robustness, multi-agent evidence merge, noisy extracted dependencies, or a separately designed atomic-vs-compound training experiment.

### Success criterion

The project succeeds as research **even if the dependency-aware arm loses**: the aim is an independently reproducible measurement that separates **what the model knows**, **what the memory system makes available**, and **whether the agent can compose those capabilities into justified decisions**.

## References

1. He, Y., Li, Y., Wang, Y., & Vitercik, E. (2026). [Compositional Reasoning in Language Models under Reinforcement Learning Post-Training](https://arxiv.org/abs/2609.19465). Conceptual source for the three composition levels; not a direct epistemic-memory benchmark.
2. Wang et al. (2026). [A State Change Is Local, Its Consequences Are Not: Dependency-Aware State Revision for Long-Term Agent Memory](https://arxiv.org/abs/2611.02912). Selective downstream revision and independent-support controls.
3. Wang et al. (2026). [PACMI: Provenance-Aware Cascading Memory Invalidation for Long-Term LLM Agents](https://arxiv.org/abs/2610.05732). Provenance and cascade-aware memory evaluation.
4. Jung et al. (2026). [MEME: Multi-entity & Evolving Memory Evaluation](https://arxiv.org/abs/2605.12477). Cascade, absence and longitudinal memory benchmarks.
5. Gao et al. (2026). [MemDerive: A Diagnostic Benchmark for Agent Memory over Evolving Data](https://arxiv.org/abs/2610.25373). Exact gold derivation graphs and intermediate-state failure diagnosis.

---

**Research status:** methodology proposal, 15 November 2026. No experiments have been run or results claimed by this repository.
