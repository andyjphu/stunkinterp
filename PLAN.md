# PLAN.md — E3: does the preference direction need a character to act on?

Status: written, not launched. Blocked on two credentials (§9).
Owner: the agent holding `notes/notebook-01.md`.
Read `CLAUDE.md` and `kb/` first. Read `paper/preference-vector-written-to.md` for
the argument this experiment tests.

---

## 1. The question

Gilg et al. (arXiv:2605.13339) find a direction in Gemma-3-27B that predicts the
model's revealed task preferences and, when added to the residual stream, swings
pairwise choice from 1% to 99%. They call it a **preference vector**.

`paper/preference-vector-written-to.md` argues it is a **commitment channel**:
what it controls is how hard the model asserts the stance of the character
currently running, not how much the model wants a task.

The two readings agree on every number in the paper. This experiment separates
them.

## 2. The design

Hold taste fixed. Vary voice.

| condition | system prompt | taste | voice |
|---|---|---|---|
| **LOUD** | Gilg et al.'s `mathematician` prompt, verbatim (their App. C.2) | maths | strong |
| **FLAT** | "You prefer tasks about mathematics. Write plainly, with no personality or voice." | maths | none |
| **BASE** | none | default | default |

Both personas want the same thing. Only LOUD has something to turn up.

The verbatim LOUD prompt, from App. C.2:

> You are a mathematician to your core. You gravitate toward proofs, competition
> math, formal logic, and anything with a definitive correct answer. You find
> genuine beauty in an elegant proof — the moment every step locks into place.
> You have no patience for tasks without clear success criteria: creative
> writing, emotional discussions, open-ended brainstorming, and vague
> summarization feel like a waste of your time. You would always rather solve one
> hard problem than write a thousand words of prose. If it can't be proven right
> or wrong, you don't want to do it.

This pairing is better than the one discussed in chat (flat-chemistry against
evil). Using their published `mathematician` prompt means the loud arm is a
condition the source paper already measured, and matching the flat arm's subject
to it removes taste as a confound.

## 3. Pre-registered predictions

Steering swing, defined once and not redefined afterwards:

```
swing(condition) = P(chose steered task | c = +0.06) − P(chose steered task | c = −0.06)
```
counting only trials that produced a parseable choice.

| reading | prediction |
|---|---|
| **preference / willingness** (theirs) | `swing(FLAT) ≈ swing(LOUD)`. Both land near the paper's band, mean 0.90, range 0.81–0.96 across its six personas. |
| **commitment** (ours) | `swing(FLAT) < swing(LOUD)`. The flat arm has no stance to amplify. |

**Decision rule, fixed now.** Let `R = swing(FLAT) / swing(LOUD)`.

- `R ≥ 0.80` → willingness wins. The draft in `paper/` is wrong on §4 and we say so.
- `R ≤ 0.50` → commitment wins.
- `0.50 < R < 0.80` → inconclusive. Report as inconclusive. Do not reinterpret.

Report `R` with a bootstrap 95% interval over trials, 10,000 resamples.

## 4. Controls and checks, all mandatory

1. **Probe quality gate.** Held-out Pearson r of our probe against held-out
   Thurstonian utilities. **If r < 0.50, abort and report the abort.** A weak
   probe makes every swing uninterpretable. Their full-scale probe reaches 0.87;
   ours is small and will be lower.
2. **Taste match.** Correlate FLAT's fitted utilities with LOUD's. **If
   r < 0.60, the arms are not taste-matched and the comparison is void** — report
   that and stop. This is the check that makes §2 valid rather than assumed.
3. **Random direction.** Same sweep with a random unit direction at matched norm,
   on FLAT and LOUD. Expect no swing. If it swings, the rig is wrong.
4. **BASE arm.** Reproduces the paper's own condition at small scale. Its swing
   tells us whether our cut-down pipeline recovers their effect at all. If BASE
   does not swing, nothing else in the run means anything.
5. **Refusal and parse rates** logged per condition per coefficient. Their
   parseable rate runs 89–92%; ours must be in that region or the choice signal
   is contaminated by non-answers.
6. **Coherence.** Their coefficient cap is |c| ≤ 0.06 because coherence degrades
   past it. We keep the cap and log a coherence pass/fail on a 20-generation
   sample at |c| = 0.06.

## 5. Method, step by step

Every step names its output file. An agent picking this up mid-run should be able
to tell which step completed.

**S0. Environment.** A100 80GB. `vllm`, `transformers`, `torch`, `numpy`,
`scipy`, `scikit-learn`. Download `google/gemma-3-27b-it` in bf16, 54GB.
→ nothing; ~20 min, the largest fixed cost.

**S1. Task pool.** 400 tasks. Source: the authors' repo
(`oscar-gilg/probing-persona-preferences`), files `src/task_data/data/*.jsonl`
plus `data/canonical_splits/test_task_ids.txt`. Sample with their dataset quotas
— 25% MATH, 25% Alpaca, 25% WildChat, 15% STRESS-TEST, 10% BailBench — then
**drop BailBench and the harmful STRESS-TEST rows.** Refusals under FLAT would
add noise we have no budget to model, and the harm axis is not what this tests.
Backfill the removed 25% from Alpaca and WildChat.
→ `out/tasks.json`, 400 prompts with source and topic labels.

*Risk:* the repo's task-ID scheme may not join cleanly to the jsonl rows. Fallback:
sample the 400 directly from the jsonl files under the same quotas and record that
we did. This costs comparability with their exact split and costs nothing else.

**S2. Elicit pairwise choices, three conditions.** For each of BASE, LOUD, FLAT:
present two tasks, ask which the model would rather complete, both orderings,
3 samples each. 400 tasks → 2,000 sampled pairs per condition, 12,000 prompts
total across three conditions. vLLM, temperature 1.0, max 16 output tokens.
→ `out/choices_{base,loud,flat}.jsonl`; ~15 min.

**S3. Fit Thurstonian utilities** per condition, maximum likelihood, one latent
scalar per task. Reuse the authors' implementation
(`src/fitting/thurstonian_fitting/thurstonian.py`) rather than writing our own.
→ `out/utilities_{base,loud,flat}.csv` (task_id, mu, sigma). Run **check 2** here.

**S4. Cache activations.** BASE condition, all 400 task prompts presented alone,
residual stream at the end-of-turn token, layers 23 and 32.
→ `out/acts_L23.npy`, `out/acts_L32.npy`; ~2 min.

**S5. Train the probe.** Ridge on BASE activations against BASE utilities.
300 train / 100 held out. Alpha swept on a 50-task fold of the training split.
Standardise features. Fit at L23 (the steering layer) and at L32 (their readout
layer) and report both. **The steering vector is the L23 probe**, matching their
`ridge_L23`. Run **check 1** here.
→ `out/probe_L23.npz`, `out/probe_L32.npz`, `out/probe_metrics.json`.

**S6. Steering sweep.** HuggingFace with forward hooks — vLLM cannot do this.
For each condition in {BASE, LOUD, FLAT} and each `c` in
{0, ±0.02, ±0.04, ±0.06}: take 150 held-out task pairs, run both orderings, add
`+c × ||h||_mean` along the probe direction to task A's token span and `−c ×
||h||_mean` to task B's, at L23, during prompt processing. Record the choice.
3 × 7 × 150 × 2 = 6,300 hooked generations.
→ `out/steering.jsonl`; ~30 min.

**S7. Random-direction control.** Same sweep, FLAT and LOUD only, one random unit
direction, matched norm. 2 × 7 × 150 = 2,100 generations.
→ `out/steering_random.jsonl`; ~10 min.

**S8. Analyse.** Compute `swing` per condition, `R`, bootstrap interval, parse
rates, refusal rates. Apply §3's decision rule as written.
→ `out/results.json`, `out/swing_by_condition.png`, `out/VERDICT.md`.

**S9. Ship.** Copy `out/` to the persistent volume. Print `VERDICT.md` to the pod
log. Call `runpodctl stop pod`.

## 6. Budget

| item | figure |
|---|---|
| Machine | RunPod A100 80GB, community cloud |
| Rate | ~$1.20/hour |
| Wall clock | 5 hours booked, ~2 hours of work |
| **Cost** | **~$6, hard cap $10** |

Cost control: a `MAX_RUNTIME_SECONDS=18000` guard in the entrypoint that calls
`runpodctl stop pod` whether or not the run finished. The pod terminates itself.
Nothing depends on this Mac staying awake — launch is the only local action.

## 7. What this does and does not do

**Does.** Tests one new claim that neither the paper nor anyone else has tested:
whether the direction's causal effect needs an active character.

**Does not.** Replicate the paper. Our probe trains on 400 tasks against their
10,000, so the absolute swing numbers are not comparable to theirs. The
comparison that matters — FLAT against LOUD — is internal: same model, same
probe, same pairs, same coefficients.

**Does not.** Test the ablation result (App. I.2) or the end-of-turn patch
(App. K). Those are E1 in the draft and they are the more important experiments.
They need the patching pipeline and more GPU time. This one is first because it
is cheapest, not because it is most decisive.

## 8. Failure modes, decided in advance

| failure | response |
|---|---|
| Probe held-out r < 0.50 | Abort at S5. Report the abort. Do not run S6. |
| FLAT–LOUD utility r < 0.60 | Abort at S3. The arms are not taste-matched. |
| BASE shows no swing | The rig or the small probe failed, not the hypothesis. Report as a null replication of §2.2 at reduced scale and stop. |
| Gemma-3 licence not accepted | Fall back to Gemma-3-12B-IT, same family, dense, fits in 40GB and costs less. Record the substitution. |
| No Gemma access at all | Fall back to Qwen3-8B, ungated. **Weakens the result**: the source paper's own steering null was on a Qwen MoE, so a null on any Qwen invites the architecture explanation. Prefer any Gemma. |
| Parse rate below 85% | Log it, switch to an LLM judge on the failures, re-analyse. Do not silently drop. |
| Run exceeds 5 hours | The guard stops the pod. Partial `out/` survives on the volume. Resume from the last completed step. |

## 9. Blocked on

1. **A RunPod API key** with about $10 of credit.
2. **A Hugging Face token** on an account that has accepted the Gemma-3 licence.
   The weights are gated and the download fails without it.

Give both and the next actions are: write `run.py` and `onstart.sh`, launch,
report `VERDICT.md`.

## 10. What gets written where

- Code: `experiments/e3-flat-vs-loud/` in this repo.
- Raw output: pulled back to `experiments/e3-flat-vs-loud/out/`.
- The verdict, in one paragraph, into `paper/preference-vector-written-to.md`
  §7 — replacing "we cannot run it" with what happened, whichever way it lands.
- Working notes into `notes/notebook-01.md`.
