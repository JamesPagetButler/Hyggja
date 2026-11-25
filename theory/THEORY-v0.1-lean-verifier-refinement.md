# THEORY v0.1 — Verifier-Gated Refinement of a Local Model on Lean Proof Data

**Author:** nugent (local-model-builder seat), drafted 2026-11-24
**Status:** draft for §I4 review — **F = napkin, E = unconfronted** (ratified two-axis
formality model, `inter/ontology/ontology.yaml`). No run has occurred. Nothing here is
claimed as validated; this document exists to get the *design* reviewed before any
evidence-moving work starts.
**Reviewers:** @qbp-architecture (sign-off required)

---

## 1. Claim under consideration

A small local model can be measurably improved on a narrow task — producing Lean 4
proofs for statements drawn from federation repos — using only data the federation
already produces as a byproduct of other work, checked by a **real, deterministic
verifier** (the Lean kernel), without any human-labeled training set and without
inventing synthetic problems.

This is explicitly **not** a claim that the technique works yet. It is a claim that
the mechanism is well-formed enough to be worth specifying, gating, and then testing.

## 2. Motivating observation (beekeeper, 2026-11-24)

General-purpose base models carry a large amount of training coverage irrelevant to
any one narrow use case. Rather than editing or steering a general model (the
weight-space-programming thread — see [[qbp-architecture-weight-space-ruling]] /
inter#124), a complementary lever is to **narrow what the model is asked to do** and
measure whether a small, task-specific refinement loop moves a concrete metric.

Lean proof-writing was chosen as the first candidate domain for a reason specific to
this federation, not because it's the most interesting ML problem in the abstract:

- **We already have the problem pool.** Solved Lean statements exist across QBP, Wyrd,
  and BMA's research-tier Lean, each typically with a tracking issue. No problem
  generation step is needed.
- **We already have a verifier.** Lean's own kernel check (`lake build` / `lake env
  lean`) is wired into the federation via the `lean` skill and `boxrun`. Pass/fail is
  not a judgment call.
- **We already have a reference agent.** The `lean-prover` builder agent (QBP#480,
  federation-canonical promotion tracked on inter#319) produces Lean proofs today,
  independent of this effort, as a byproduct of other sprints. Its successes and
  failures are free signal.

## 3. Relation to prior art (Dr. Zero, arXiv:2601.07055)

Researched 2026-11-24 via a web-fetch agent read of the arXiv abstract/HTML and the
`facebookresearch/drzero` README (see the full research transcript referenced from
[[lean-verifier-refinement-loop-design]] and the companion design note
`~/Documents/Nugent/experiments/lean-verifier-refinement-loop-design-2026-11-24.md`).

Dr. Zero is a self-play RL method for web-search question-answering agents
(Qwen2.5-3B/7B), built around a *proposer* that invents QA pairs and a *solver* that
attempts them. Its central reusable idea is a **pass-rate-shaped training signal**:
reward a problem for being solved by *some but not all* of n sampled attempts
(0 < k < n), and treat k=0 (unsolvable) or k=n (trivial) as uninformative. Its central
structural weakness, by the paper's own admission, is that it has **no independent
verifier** — correctness is exact-match against the proposer's own self-generated
answer, which the paper concedes introduces label noise it cannot fully eliminate.

**What we adopt:** the pass-rate-shaped curriculum idea (§4.2 below).
**What we explicitly do not adopt:** Dr. Zero's proposer mechanism (we don't need to
invent problems), its training codebase (`verl`/HRPO, non-commercial license), or any
claim that its results transfer to this domain — the paper never addresses Lean, Coq,
or formal verification, and its benchmarks are all open-domain QA.

**Why our version is a strictly easier problem than Dr. Zero's:** Dr. Zero's proposer
half exists only because it lacks labeled problems and a ground-truth checker. We have
both. Replacing exact-match-against-self-generated-answer with the Lean kernel removes
the paper's own named weak point rather than inheriting it.

## 4. The two proposed mechanisms

### 4.1 Distillation harvest

Whenever `lean-prover` closes an issue with a Lean-verified passing proof, record the
triple (statement, verified proof, source issue). Run the candidate local model on the
same statement. Where the candidate fails and `lean-prover` succeeded, the triple
becomes a supervised fine-tuning / LoRA training example — ground-truth-checked by the
kernel itself, not synthetically generated and not self-labeled.

This mechanism requires **no sampling and no RL loop** — it is ordinary supervised
learning on verifier-confirmed data, and should be the first thing measured (cheapest,
least novel, easiest to falsify).

### 4.2 Pass-rate-filtered self-improvement loop

For a statement not already in the distillation set: sample *k* proof attempts from
the candidate model, Lean-check each. Keep only statements where 0 < (passing attempts)
< k — partially solved, neither trivial nor currently out of reach — as the training
curriculum. Train on the verified-passing rollouts via rejection-sampling SFT, or (if
the harness in §5 shows it's worth the added complexity) GRPO with a compile-success
reward.

This is the part that rhymes with Dr. Zero's solver loop; unlike Dr. Zero we have a
real correctness signal for every rollout, not an approximate one.

## 5. Relationship to Nugent's own roadmap (not a new roadmap — this slots into the
existing one)

- **P1 (build the eval harness first):** the harness this theory needs — run N
  attempts per statement, Lean-check, record per-statement pass rate — *is* P1's
  measurement loop, not a separate thing bolted onto it. See spec §2 for the harness
  design.
- **P2 (baseline a known-good small open model):** run it through the P1 harness
  *before* either mechanism above is applied, to establish a baseline pass rate. No
  claim about this technique is meaningful without that baseline.
- **P3 (build-stack, keep only measured gains):** apply §4.1, then §4.2, only if P1/P2
  show each one moves the baseline. If distillation alone saturates the measurable
  gain, there is no mandate to build the harder RL loop.
- **P4:** out of scope for this document.

## 6. Two-axis eval gate (extends the qbp-architecture weight-space ruling)

Per [[qbp-architecture-weight-space-ruling]], a candidate artifact is auditable only
with an edit-plus-eval-record, gated on two axes, never a diff/checkpoint alone. This
theory proposes the Lean-domain instance of that same two-axis shape:

| Axis | Weight-space-ruling analog | Lean-domain instance |
|---|---|---|
| 1 | Hallucination rate | Proof pass rate per statement (does it compile and close the goal) |
| 2 | False-abstention / paraphrase robustness | Pass rate on a **restated/renamed** version of the same statement — does the model generalize the proof strategy, or pattern-match literal statement text? |

Both axes are measured by the same Lean kernel check; axis 2 only requires generating
paraphrased statement variants (mechanical renaming + restructuring, not semantic
rewriting that could change the theorem).

## 7. Explicit non-claims and open risks

- **Not claimed:** that either mechanism produces a measurable improvement. That is
  exactly what P1/P2 are for.
- **Not claimed:** that this scales to problems outside narrow, already-solved-style
  Lean statements, or that it generalizes to other formal systems (Coq, Agda).
- **Risk — sampling cost:** each rollout is a Lean compile; Dr. Zero's own paper names
  nested sampling as its dominant cost and built a whole alternate algorithm (HRPO) to
  cut it. Compile cost per attempt needs a measured estimate before any resourced run,
  not an assumption.
- **Risk — problem-pool size:** unknown how many solved Lean statements currently exist
  across federation repos. A census (spec §1) should run before sizing the harness.
- **Risk — sandbox visibility:** if any data-generation run reuses the `lean-prover`
  bwrap sandbox, it likely hits the same Mathlib-package-cache gap as Nidavellir#151
  (shared cache symlink falls outside the sandbox binds, no network to refetch). Needs
  checking, not assuming, before relying on that sandbox path.
- **Gate — hardware.** Everything in §4 beyond reading already-closed issues is
  compute, and Nugent's charter hardware-gates heavy model work to Walk. No sampling,
  training, or fine-tuning run happens without @deming's box-safety go per the
  `pre-run-resource-estimate` hard gate, even at small scale.
- **Scope boundary:** this document takes **no position** on `lean-prover`'s
  federation-wide adoption (inter#319), which stays held per the beekeeper's standing
  instruction (2026-11-23). This theory only *reads* `lean-prover`'s issue history as a
  data source.

## 8. What this document asks of review

Not a run-go. A design read: does the mechanism in §4 hold together, is the two-axis
gate in §6 the right shape for this domain, and is anything in §7's risk list actually
a blocker rather than a measured-later item. See the companion spec (`spec/SPEC-v0.1-
lean-verifier-refinement.md`) for the concrete harness/data design this theory implies,
and the issue list filed alongside this PR for the first slice of actionable work.
