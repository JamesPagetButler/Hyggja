# SPEC v0.1 — Lean-Proof Verifier-Gated Refinement: Harness &amp; Data Design

**Author:** nugent (local-model-builder seat), drafted 2026-11-24
**Companion to:** `theory/THEORY-v0.1-lean-verifier-refinement.md`
**Status:** draft for §I4 review — specifies P1 (harness) and the P3 data mechanisms;
specifies nothing that runs without the gates in §6.
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

**Method:** for each repo, `grep` for `theorem`/`lemma`/`def` declarations with a
`sorry`-free proof, cross-reference against closed issues mentioning the file/line
(where linkable) or the PR that introduced it.

**Output:** a flat file, one row per statement:

```
repo, file, line, declaration_name, has_tracking_issue (bool), issue_ref (nullable),
closed_by (lean-prover | human | unknown), proof_length_lines
```

This is read-only, I/O-bound work — no `boxrun` gate applies (per `boxrun` skill §1:
"pure I/O or git work with no CPU-bound math" is explicitly out of scope for that
wrapper).

## 2. P1 harness: pass-rate measurement loop

**Purpose:** for a given candidate model and a given statement set, produce a
per-statement pass-rate record. This is the harness theory §5 says Nugent's own
roadmap already requires before P2 or P3 can proceed.

**Inputs:**
- A statement set (from §1, or a held-out subset of it).
- A candidate model (any local or hosted inference endpoint capable of producing a
  Lean proof term or tactic script from a statement + imports context).
- `k` = number of sampled attempts per statement (start small; see §6 resource gate —
  this number is exactly what needs a written estimate before any run).

**Per-statement loop:**
1. Present the statement (with its surrounding imports/context) to the candidate
   model, sampled `k` times (temperature/sampling params TBD at run-time, recorded in
   the run manifest).
2. For each of the `k` outputs, check with Lean (`lake build` / `lake env lean` via
   the `lean` skill, itself run through `boxrun`).
3. Record `passed_count ∈ [0, k]` for the statement.
4. **Axis 2 (robustness):** generate one mechanically restated/renamed variant of the
   statement (identifier renaming + argument-order permutation where type-safe — not a
   semantic rewrite that could change the theorem), repeat steps 1–3 against the
   variant, record `passed_count_variant`.

**Output — the per-statement eval record:**

```
statement_id, repo, declaration_name,
k, passed_count, passed_count_variant,
model_id, model_sha_or_version,
run_id, run_manifest_ref,
timestamp
```

**Aggregate outputs:**
- Overall pass rate = Σ passed_count / (k × |statements|).
- Partial-solve set = statements where 0 < passed_count < k (the §4.2 curriculum
  candidates).
- Robustness delta = passed_count − passed_count_variant per statement (large negative
  delta flags pattern-matching rather than genuine proof-strategy generalization).

**Idempotency / provenance:** every run writes a `run_manifest` (model id/version,
statement-set hash, k, sampling params, wall-clock, Lean toolchain version) before any
aggregate is computed — same discipline as the spike-run skill's "raw output is hashed
and committed before the analyzer runs" (§5 of that skill), applied here even though
this is harness-building rather than a spike.

## 3. Distillation-harvest schema (theory §4.1)

Triggered whenever `lean-prover` closes an issue with a passing proof (independent of
any P1 run — this is ordinary data capture from work already happening).

```
source_issue_ref, repo, declaration_name, statement_text, verified_proof_text,
verified_by (lean toolchain version + commit sha), harvested_at,
candidate_model_result (pass | fail | not_yet_attempted),
eligible_for_sft (bool — true iff candidate_model_result == fail)
```

Rows with `eligible_for_sft = true` form the supervised training set for the §4.2-
adjacent (but simpler, non-RL) SFT path named in theory §4.1. No row is admitted
without a kernel-verified proof — this mirrors the propose≠admit discipline used
elsewhere in the federation for literature intake (contextus-seed-corpus), but the
"admission" gate here is categorical and mechanical (Lean says yes or no), not a
judgment call.

## 4. Pass-rate-filtered curriculum (theory §4.2)

Input: the partial-solve set from §2's aggregate output.

```
statement_id, passed_rollouts (list of verified-passing proof texts, length =
passed_count), failed_rollouts (count only, not stored — no training signal in a
Lean compile failure beyond "try again")
```

Training signal = the `passed_rollouts` texts only (rejection-sampling SFT). A GRPO
variant (compile-success as a binary reward) is named as a possible P3 escalation if
rejection-sampling SFT plateaus — not specified further here; it is a bigger
engineering lift (a GRPO loop, not just a filter-and-retrain step) and should only be
built if §2's measurements justify it, per "keep only measured gains."

## 5. Two-axis record (theory §6) — rollup format

For any candidate-vs-baseline comparison:

```
model_id, baseline_model_id,
axis1_pass_rate, baseline_axis1_pass_rate,
axis2_robustness_delta, baseline_axis2_robustness_delta,
statement_set_hash, k, run_manifest_refs (both runs)
```

This is the artifact theory §6 says should exist instead of a bare model
diff/checkpoint — matching the governance shape qbp-architecture ruled for the
weight-space-programming thread ([[qbp-architecture-weight-space-ruling]]).

## 6. Run-go gates (nothing in §2–§4 executes without these)

1. **Census (§1):** no gate — I/O only.
2. **Any P1 harness run (§2), any distillation-candidate inference (§3), any
   curriculum sampling (§4):** requires, before launch:
   - a written disk/RAM upper-bound estimate and explicit kill condition
     (`pre-run-resource-estimate` hard gate),
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
