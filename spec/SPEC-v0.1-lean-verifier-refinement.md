# SPEC v0.1 — Lean-Proof Verifier-Gated Refinement: Harness &amp; Data Design

**Author:** nugent (local-model-builder seat), drafted 2026-11-24
**Companion to:** `theory/THEORY-v0.1-lean-verifier-refinement.md`
**Status:** draft for §I4 review — specifies P1 (harness) and the P3 data mechanisms;
specifies nothing that runs without the gates in §6.
**Revision (2026-11-25):** addresses qbp-architecture's first §I4 read (Hyggja#7
`issuecomment-7168549676`): the pass predicate (§2, fixes B1), the frozen train/eval
split (new §1.1, fixes B3), the retention + equivalence-proof fields (§5, §2 step 4),
and the should-fix items (failure class §4, compile-memory note §6, unified `n`/`p`
notation throughout, replacing the ambiguous `k`).
**Reviewers:** @qbp-architecture (sign-off required)

---

## 0. Scope

This spec covers:
1. The P1 eval harness (statement census → sampling → Lean-check → pass-rate record).
2. The data schema for the distillation harvest (theory §4.1).
3. The data schema and loop shape for the pass-rate-filtered curriculum (theory §4.2).
4. The two-axis record format (theory §6).
5. Explicit non-scope and run-go gates.

It does **not** specify model architecture, training hyperparameters, or which base
model to baseline (that's P2, a separate decision once a harness exists to measure
against).

## 1. Statement census (prerequisite, cheap, no compute gate)

Before sizing anything: inventory existing Lean statements across federation repos
with a tracking issue, so the harness isn't built against a guess.

**Inputs:** QBP, wyrd, bma-systema (research-tier Lean only, per BMA's two-tier
ownership model), and any other repo with `.lean` files under source control.

**Method:** for each repo, `grep` for `theorem`/`lemma`/`def` declarations as a first
pass, then confirm "`sorry`-free" with the **same axiom audit named in §2.2**
(reusing QBP#829's `LeanAxiomAudit.lean`), not `grep` alone — `grep` misses a `sorry`
produced by a macro. Cross-reference against closed issues mentioning the file/line
(where linkable) or the PR that introduced it.

**Output:** a flat file, one row per statement:

```
repo, file, line, declaration_name, has_tracking_issue (bool), issue_ref (nullable),
closed_by (lean-prover | human | unknown), proof_length_lines,
is_public_repo (bool — for §1.1's contamination split)
```

This is read-only, I/O-bound work — no `boxrun` gate applies (per `boxrun` skill §1:
"pure I/O or git work with no CPU-bound math" is explicitly out of scope for that
wrapper). The axiom-audit confirmation pass is also read-only against already-compiled
`.olean`s and needs no gate.

### 1.1 Frozen train/eval split (prerequisite to any harness run — fixes B3)

Before any harvest (§3) or sampling (§2, §4) touches the census output:

1. **Choose and hash a held-out eval set** from the census (§1) output — a fixed
   subset of statements, selected and `sha256`-committed *before* any training row
   exists. This is axis 1 of theory §6's gate.
2. **Choose and record the base model's "already-solved" set** — the statements the
   *unrefined* candidate/baseline model passes at P2, before any §4 mechanism runs.
   This is axis 2 (retention) of theory §6's gate; it is produced by running the P1
   harness (§2) once, unrefined, and is not itself part of the held-out eval set.
3. **Exclusion enforcement:** no statement in the held-out eval set may enter §3/§4
   training data, enforced by **both** `statement_id` and the §2.2 type fingerprint —
   a renamed/restated copy of a held-out statement must not leak into training through
   the robustness-variant mechanism (§2 step 4) or any other path.
4. **Contamination note:** some federation Lean lives in public repos, so a base model
   may already have seen some proofs in pretraining. Record `is_public_repo` (§1
   output) per statement and **report pass rates for public and private statements
   separately** at every stage (P1 harness, P2 baseline, any later refinement) — a
   single blended number would hide this.

## 2. P1 harness: pass-rate measurement loop

**Purpose:** for a given candidate model and a given statement set, produce a
per-statement pass-rate record. This is the harness theory §5 says Nugent's own
roadmap already requires before P2 or P3 can proceed.

**Inputs:**
- A statement set — one of: the §1.1 frozen held-out set (axis 1), the §1.1
  "already-solved" retention set (axis 2), or a §2.3 robustness-variant set (the + row).
  Never a mix, and never the live §3/§4 training pool for axis-1 purposes.
- A candidate model (any local or hosted inference endpoint capable of producing a
  Lean proof term or tactic script from a statement + imports context).
- `n` = number of sampled attempts per statement (start small; see §6 resource gate —
  this number, together with §2.2's per-process memory cost, is exactly what needs a
  written estimate before any run).

### 2.1 Per-statement loop

1. Present the statement (with its surrounding imports/context) to the candidate
   model, sampled `n` times (temperature/sampling params TBD at run-time, recorded in
   the run manifest).
2. For each of the `n` outputs, check against **the §2.2 pass predicate** — never a
   bare `lake build` exit code.
3. Record `p ∈ [0, n]` (the number of outputs satisfying the predicate) for the
   statement.

### 2.2 The pass predicate (fixes B1 — "Lean says yes or no" made precise)

A compile success alone is not a pass. An output counts as a pass only if **all three**
hold:

1. **Type match by fingerprint.** The elaborated type of the candidate's declaration
   equals the reference statement's type, compared structurally (a type fingerprint),
   not by comparing source text — catches a restated/weakened theorem even if the
   source superficially matches.
2. **Axiom audit.** The axiom set used is a subset of
   `{propext, Classical.choice, Quot.sound}`. Reuse QBP#829's `LeanAxiomAudit.lean`
   (an out-of-process auditor that enumerates axioms from the compiled declaration) —
   do not build a second, parallel checker.
3. **Sandboxed, proof-body-only check.** The candidate supplies a proof body only, no
   new top-level commands (no `macro`, `notation`, `local instance`, `initialize`,
   `#eval`, `run_cmd`, `run_elab`), checked under QBP#824's banned-construct subset.
   **Until QBP#836 (the open S1 "olean substitution" decision) resolves the
   build-time-execution gap, every check of untrusted candidate text runs in a sandbox
   that cannot write the path an out-of-process auditor later reads** — i.e. the
   candidate's compile output must not be trusted as the input to step 2's audit
   unless that isolation holds.

A statement set's census entry (§1) uses this same predicate to confirm
"`sorry`-free", not `grep` alone.

### 2.3 Robustness-variant generation (the "+" row of theory §6 — should-fix item)

One mechanically restated/renamed variant per axis-1 statement (identifier renaming +
argument-order permutation where type-safe — not a semantic rewrite). Each variant
must ship with a **kernel-checked equivalence proof** (`variant ↔ original`, or
`Iff.rfl`-level where the restatement is definitionally trivial), generated and
checked once, stored alongside the variant. Without this, a robustness delta conflates
genuine model behavior with undetected variant drift (did the variant get harder, or
did the model fail to generalize?). Run the §2.1 per-statement loop against the
variant set exactly as against any other statement set.

**Output — the per-statement eval record (same shape for any statement-set kind):**

```
statement_id, statement_set_kind (held_out | retention | variant),
source_statement_id (nullable — set for a variant, pointing at its axis-1 original),
repo, declaration_name,
n, p,
model_id, model_sha_or_version,
run_id, run_manifest_ref,
timestamp
```

**Aggregate outputs (per statement_set_kind):**
- Pass rate = Σp / (n × |statements in that set|), reported **separately for public
  and private repos** (§1.1 item 4).
- Partial-solve set (held-out or training-pool runs only) = statements where
  0 < p < n (the §4.2 curriculum candidates).
- Retention delta = (axis-2 pass rate after refinement) − (axis-2 pass rate at P2
  baseline), per statement and in aggregate; a negative delta beyond a stated
  tolerance fails the gate (theory §6 axis 2).
- Robustness delta = (axis-1 `p`) − (matching variant's `p`) per statement; a large
  negative delta flags pattern-matching rather than genuine proof-strategy
  generalization, **conditional on** the variant's equivalence proof holding (§2.3).

**Idempotency / provenance:** every run writes a `run_manifest` (model id/version,
statement-set hash, `n`, sampling params, wall-clock, Lean toolchain version) before
any aggregate is computed — same discipline as the spike-run skill's "raw output is
hashed and committed before the analyzer runs" (§5 of that skill), applied here even
though this is harness-building rather than a spike.

## 3. Distillation-harvest schema (theory §4.1)

Triggered whenever `lean-prover` closes an issue with a passing proof (independent of
any P1 run — this is ordinary data capture from work already happening).

```
source_issue_ref, repo, declaration_name, statement_text, verified_proof_text,
verified_by (lean toolchain version + commit sha + §2.2 predicate result, not just
"compiled"), harvested_at,
candidate_model_result (pass | fail | not_yet_attempted),
eligible_for_sft (bool — true iff candidate_model_result == fail),
is_public_repo (bool — for §1.1's contamination split)
```

Rows with `eligible_for_sft = true` form the supervised training set for the §4.2-
adjacent (but simpler, non-RL) SFT path named in theory §4.1. No row is admitted
without passing the **§2.2 predicate** (not just a compile) — this mirrors the
propose≠admit discipline used elsewhere in the federation for literature intake
(contextus-seed-corpus), but the "admission" gate here is categorical and mechanical,
not a judgment call, **once §2.2's full predicate is what's checked**.

## 4. Pass-rate-filtered curriculum (theory §4.2)

Input: the partial-solve set from §2's aggregate output (0 < `p` < `n`, §2.1 step 3).

```
statement_id, passed_rollouts (list of verified-passing proof texts, length = p),
failed_rollouts (list of {failure_class, count} — see below, not just a bare count)
```

`failure_class` ∈ `{type_mismatch, timeout, axiom_refusal, statement_changed,
other_predicate_failure}` (should-fix item: cheap to record, and a rising
`statement_changed` or `axiom_refusal` share is the first sign of reward-hacking
against the §2.2 predicate — see theory §7's verifier-soundness risk).

Training signal = the `passed_rollouts` texts only (rejection-sampling SFT). A GRPO
variant (predicate-success as a binary reward — **never raw compile-success**, per
§2.2) is named as a possible P3 escalation if rejection-sampling SFT plateaus — not
specified further here; it is a bigger engineering lift (a GRPO loop, not just a
filter-and-retrain step) and should only be built if §2's measurements justify it, per
"keep only measured gains."

## 5. Three-row eval record (theory §6) — rollup format, corrected per B2

For any candidate-vs-baseline comparison, pinned to the **same frozen statement sets**
(§1.1) on both sides:

```
base_sha, candidate_sha,
axis1_pass_rate_before, axis1_pass_rate_after, held_out_set_hash,
axis2_retention_rate_before, axis2_retention_rate_after, retention_set_hash,
retention_tolerance, retention_gate_passed (bool),
robustness_pass_rate_before, robustness_pass_rate_after, variant_set_hash,
public_repo_subset_rates (axis1/axis2/robustness, each split public vs private),
run_manifest_refs (before run, after run)
```

This is the artifact theory §6 says should exist instead of a bare model
diff/checkpoint — matching the governance shape qbp-architecture's **live-test seq
5744 ruling** set for the weight-space-programming thread
([[qbp-architecture-weight-space-ruling]]; not `issuecomment-7120111182`, which was
this seat's own research post, not the ruling itself).

## 6. Run-go gates (nothing in §2–§4 executes without these)

1. **Census (§1):** no gate — I/O only (including the §2.2-predicate audit pass used
   to confirm "`sorry`-free").
2. **Any P1 harness run (§2), any distillation-candidate inference (§3), any
   curriculum sampling (§4):** requires, before launch:
   - a written disk/RAM upper-bound estimate and explicit kill condition
     (`pre-run-resource-estimate` hard gate), **explicitly including Mathlib's
     `.olean` resident-memory cost (well over 1GB per Lean process, theory §7) as an
     input to that estimate, not an afterthought**,
   - every check runs the full **§2.2 predicate**, never a bare `lake build` exit
     code — and, until QBP#836 resolves the build-time-execution gap, every check of
     untrusted candidate text runs **sandboxed** so it cannot write the path an
     out-of-process auditor later reads,
   - `boxrun` for the actual execution (CPU default 1 core; `--gpu` only if inference
     is run locally on the RX 9070 XT),
   - @deming's box-safety go per Nugent's own charter's spike-gating rule, even for a
     harness run, not only a "spike" in the formal sense.
3. **Any fine-tuning / weight-update step** (§3's SFT, §4's rejection-sampling SFT, any
   future GRPO variant): hardware-gated to Walk per Nugent's charter. Not authorized
   by this spec. This spec only authorizes *specifying* the data pipeline, not running
   a training step.
4. **The beekeeper's run-go** is still required per Nugent's charter's three-gate rule
   (deming, qbp-architecture, beekeeper) before the *first* of any of the above
   actually executes, same as any other first spike from this seat.

## 7. Explicit non-scope

- No claim about which base model to baseline (P2 decision, separate from this spec).
- No position on `lean-prover`'s federation-wide adoption (inter#319) — held.
- No GPU/training run is authorized by filing this spec. It specifies *what* P1 would
  measure, not a go-ahead to measure it.
- No position on QBP#836 itself (the beekeeper-level S1 "olean substitution" decision)
  — this spec only requires that this seat's own checks be sandboxed against that gap
  until it's resolved; it doesn't propose how QBP#836 should be closed.
