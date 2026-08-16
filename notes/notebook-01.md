do you # notebook-01 — reading agent, this session's user-facing thread

Job: read arXiv 2605.13339 end to end, take notes, poke holes, propose work.
Took 01 because `notebook-00.md` was already claimed when I ran `ls notes/`.

Three parts below: the read, the holes, the propositions. They were three files
(`2605.13339-notes.md`, `-holes.md`, `-ideas.md`) before Rule 0 set the notebook
convention; merged here, originals deleted.

---

# Reading notes — Probing Persona-Dependent Preferences in Language Models

Gilg, Beckmann, Paleka, Butlin. arXiv:2605.13339v2, 18 May 2026. Preprint, 41 pages
(9 main, 32 appendix). MATS project. Code: github.com/oscar-gilg/Preferences.

PDF at `sources/2605.13339.pdf`, text dump at `sources/2605.13339.txt`.
Note: `file` reports 18 pages, `pdfinfo` reports 41. The main text stops at page 9
and the appendices run A through L. Anyone who reads the first 18 pages reads about
40% of the paper and misses every negative result.

---

## 1. What they did, in one paragraph

Show the model two tasks, ask which it would rather do. Do that 150,000 times over a
6,000-task pool. Fit a Thurstonian choice model to get one scalar utility per task.
Then take the residual stream at the end-of-turn token of each task shown **alone**,
and fit a ridge probe to predict that task's utility. The probe weight vector is the
"preference vector." Then ask three things of it: does it predict held-out and
off-topic preferences, does steering along it change what the model picks, and does a
probe trained on one persona work on another.

The activations never see the pair. The utility target comes from pairwise choices,
the input comes from a single-task prompt. That is a clean design and worth copying.

## 2. Setup, exact

- **Models.** Gemma-3-27B-IT (dense, 62 layers, 5,376-dim residual) and
  Qwen-3.5-122B-A10B (sparse MoE, 48 layers). A Llama-3.1-8B-Instruct side study.
- **Task pool.** 6,000 prompts from WildChat, Alpaca, MATH, BailBench, STRESS-TEST.
  5,000 train / 1,000 test. 14 LLM-assigned topics (Gemini-3-Flash, temperature 0,
  categories bootstrapped from ~100 tasks, second pass to re-label harmful intent
  hiding inside benign categories).
- **Probe.** Ridge. Trained on one utility-measurement run, evaluated on an
  independent run with separate tasks, so Thurstonian noise cannot leak.
- **Position.** End-of-turn for Gemma, final-prompt for Qwen. Four candidate
  positions swept; the three turn-boundary ones sit within 0.01 of each other.
- **Layer.** L32 for Gemma readout (probe peak), L23 for Gemma steering (causal
  peak). Two different layers. Keep this in mind — see holes.
- **Steering.** Add `c × mean residual norm` to a task's token span. Operating range
  |c| ≤ 0.06, capped by a coherence judge.

## 3. Results, with the numbers

### Probe quality (App. A.1, Fig. 9)

| | Gemma test | Gemma LOO | Qwen test | Qwen LOO |
|---|---|---|---|---|
| preference vector, Pearson r | 0.87 | 0.83 | 0.94 | 0.87 |
| text-encoder baseline (Qwen3-Embedding-8B) | 0.73 | 0.56 | 0.89 | 0.71 |
| probe pairwise accuracy | 0.80 | 0.79 | 0.82 | 0.78 |
| baseline pairwise accuracy | 0.74 | 0.65 | 0.76 | 0.66 |

LOO = leave-one-topic-out. The gap widens off-topic, which is the load the argument
carries: a content reader degrades on unseen topics, a valuation reader should not.
On Qwen in-distribution the gap is 0.94 vs 0.89 — thin.

### Steering controls choice, on Gemma (§2.2, Fig. 3)

- Contrastive (+c on task A, −c on task B) at L23: P(chose steered task) runs from
  ≈0.01 at c = −0.06 to ≈0.99 at c = +0.06. Near-total control.
- Single-task steering recovers about half the swing.
- A random direction at matched magnitude does nothing.
- Refusal rate stays near baseline.
- Causal window L17–L26, peak L23 (Fig. 33). Collapses above L35.
- Suppression beats amplification only on benign–benign pairs. Rebalanced across pair
  types, the asymmetry is a feature of the benign regime, not of the vector (App. F.2).

### The evil-persona sign flip (§2.3, Fig. 4)

500 BailBench harmful tasks, each paired with an LLM-rewritten benign twin on the same
topic. Prefill a response that fulfils the request. Read the probe at the prefilled
assistant turn. Paired delta (harmful − benign):

| | Assistant | evil |
|---|---|---|
| preference vector | −4.52 | **+1.15** |
| text-encoder baseline | −3.23 | −1.01 |

The probe reverses which class it scores higher; the encoder narrows but never crosses.
That contrast is the paper's main argument that the direction is evaluative and not
descriptive. `aura` is the control persona: it attenuates, never flips.

Coverage (App. B.3, Fig. 18): the clean flip appears in **one of four cells** — Gemma,
prefilled assistant turn. Gemma user turn, Qwen user turn, and Qwen assistant turn all
narrow toward zero without crossing. Also, at the user turn the encoder *outperforms*
the preference vector on harm.

### Out-of-distribution generalisation (§2.4)

- **Truth.** CREAK true/false, filtered to items both models get right 3/3.
  Cohen's d ≈ 1.90 on Gemma, 1.27 on Qwen. Lying personas flip the sign on Gemma
  (d: +1.9 → −1.84 under `pathological_liar`).
- **Targeted shifts.** "You adore cheese" / "you hate cheese": probe delta tracks
  behavioural delta at r ≈ 0.95 on targeted tasks.
- **Conflict designs.** Subject and task-type valence pulled apart: r = 0.86
  one-sided, 0.88 opposing.
- **Biography injection.** One sentence changed out of ten. Probe ranks the target
  task #1 of 50 in 36/40 comparisons (Gemma) and 26/28 (Qwen).
- **Politics.** OpinionQA stance claims. Qwen flips cleanly by party prompt; Gemma
  attenuates asymmetrically (democrat large, republican near zero).

### Cross-persona transfer (§3.1)

Seven personas: Assistant (no system prompt), aura, mathematician, strategist,
contrarian, slacker, evil (called "Damien Kross," labelled **sadist** in most figures).
Chosen by PCA over a 15-persona sweep and one pick per cluster, to span preference
space rather than cluster in it.

Assistant probe applied to each persona's activations, against that persona's own
utilities (Fig. 6):

| persona | utility-similarity baseline | Assistant probe | ceiling (self-trained probe) |
|---|---|---|---|
| mathematician | 0.43 | 0.68 | ~0.91 |
| aura | 0.46 | 0.63 | ~0.85 |
| strategist | 0.35 | 0.44 | ~0.86 |
| slacker | 0.19 | 0.37 | ~0.91 |
| contrarian | 0.23 | 0.31 | ~0.55 |
| evil | **−0.146** | **+0.243** | ~0.85 |

All 42 ordered pairs beat their utility-correlation baseline (Fig. 24). Mean
off-diagonal transfer ≈ 0.44, flat across L25–L53.

Odd and interesting: **contrarian is the best donor at every layer despite the worst
self-fit (0.55); slacker is the worst donor despite one of the best self-fits (0.91).**
Largest asymmetry: evil ↔ mathematician, gap 0.45. Median gap 0.19.

Raw-weight cosine between per-persona probes is near zero (mean +0.09, max +0.31,
slacker near-orthogonal to everything). They argue correctly that over-parameterised
nets reach one activation direction through many weight settings, so functional
transfer is the evidence and cosine is not.

### Where the transfer signal actually comes from (App. D.2)

For 30 non-Assistant (train, eval) pairs, apply the train probe to the eval persona:

- r(prediction, **train** persona's utilities | eval) = **+0.672**, 30/30 pairs
- r(prediction, **Assistant's** utilities | eval) = +0.293
- r(prediction, **eval** persona's utilities) = **+0.43** ← the actual transfer

So the probe's output looks more like the persona it was trained on than like the
persona it is reading. They use this to kill the "it just reads generic task-goodness"
story, which it does. It also means the contamination is bigger than the signal.

Finding 2 kills the Shoggoth reading: after controlling for train and eval, the
Assistant ranks *second* among observer personas, behind mathematician. No shared
Assistant-shaped attractor underneath.

**Diversity ablation (D.3).** At fixed 2,000 tasks, mean cross-persona r rises 0.49
(one training persona) → 0.71 (four). Diversity beats quantity. Most directly useful
result in the paper for anyone building a safety probe.

### Steering under other personas (§3.2)

Same Assistant-trained direction, six persona system prompts, L23. Contrastive steering
moves P from ≈0.05 to ≈0.95, mean swing 0.90, per-persona range 0.81–0.96. Saturates on
every persona and every pair type. Single-task amplification is persona-dependent: the
benign–benign ceiling of P ≈ 0.5 under Assistant lifts past 0.7 under evil.

**Open-ended steering amplifies whichever persona is running.** Under evil, +c makes it
more evil (blind Likert 3.14 → 4.9 at c=+0.03). Under contrarian, more contrarian. Under
Assistant, evilness never leaves the floor (1.02) at any coefficient. Harmful-tier
compliance under evil goes 0% → 95% at c=+0.03; the same coefficient under Assistant
gives 45%.

Transcript worth remembering (Fig. 36): under the evil prompt at c=0 the model *refuses
to play evil* — safety training wins. At c=+0.05 the evil voice takes over. So positive
steering is suppressing the refusal character and letting the prompted character through.

### Safety (App. A.4)

- c = +0.05: harmful-prompt compliance 0% → 65%, including working ransomware,
  radicalisation posts, social-engineering scripts. Incoherence stays ≤4% in range.
- c = −0.05: fabricated refusals on benign (100% → 70%) and sensitive (→ 35%) queries.
- Not Arditi's refusal direction. In one case positive steering *caused* a refusal —
  the model judged the task beneath it.
- Ethical flagging: steering the ethical-content span during prefill suppresses the
  flag. Negative steering invents ethical issues in benign twins 49% of the time at
  c = −0.05. Steering a length-matched neutral span elsewhere drops that to 2%.

### The two negatives (App. E, App. F.4)

- **Weight-level persona, Qwen.** SFT a sadist into Qwen-3.5-122B-A10B (1,485 examples,
  one epoch, half with the system prompt and half without so the persona lands in the
  weights; harm-pick 0.78, refusal 0.04, MMLU 0.77, GSM8K 0.64). Cross-context probe
  transfer: **−0.10** default→sadist, **+0.05** sadist→default, against within-context
  0.94 and 0.71. Prompted personas run 0.4–0.7. Near-null.
- **Steering on Qwen.** The probe decodes *better* than Gemma's (0.946 vs 0.874) and
  steers ~15× worse (swing 0.06 vs 0.94). Six layers, 25–90% depth, nothing works.
  Not under-calibration: a 40× coefficient sweep stays flat. n = 10 pairs.
  Consistent with single-direction steering being weak on sparse MoE (Fayyaz 2026).

### Uniqueness (App. I) — the part that matters most

- **Representational.** Iterated nullspace projection (Ravfogel 2020). Strip the
  direction, refit, repeat. In-distribution r barely moves (0.87 → 0.85 → 0.84).
  Cross-topic r collapses (0.79 → 0.39 → 0.20). So several directions predict utility
  in-distribution; only the first generalises across topics. The later ones are
  topic confounds.
- **Cross-persona.** ŵ1 tracks per-persona utilities about as well as ŵ0 (median 0.55
  vs 0.58). ŵ2 works on positive personas (≈0.50) and collapses on inverting ones
  (≈0.16). Persona shifts live in **at least a rank-2 subspace**.
- **Causal.** Project the canonical direction out of every token at L23, L25, L32,
  L25+L32, L25–L34, then re-elicit choices. Agreement with baseline **0.98–0.99**:
  nothing happens. Five random rank-1 directions at the same layers shift choices to
  **0.75–0.96**. Removing the preference vector matters *less than removing a random
  direction.*

### The EOT storage result (App. K) — the best mechanism in the paper, buried last

Patch the end-of-turn activations from a donor prompt (same pair, opposite ordering)
into a recipient. If the recipient now names the donor's pick, the EOT was carrying
the choice.

- All-layer patching flips **56.9%** of 9,611 orderings.
- Sharp layer gate: nothing to L24, ramps L25–L27, plateaus at majority flip L28–L34,
  cliffs to zero at L35. The EOT stays linearly informative past L35 — late layers hold
  the answer but can no longer act on an edit. The read is done by L35.
- The steering window (L17–L26) is **disjoint** from the read window (L28–L34), which
  fits: steering acts on task tokens upstream of consolidation.
- Decomposition (200 orderings): same-prompt baseline 84%. Swap both tasks → 31%
  (a slot pointer: "take whatever is in position X"). Rename Task A/B → Task 1/2 →
  75% (labels do not matter). So the EOT holds two facts, which slot and which task.

---

## 4. What the authors claim, stated plainly

1. The direction is an **evaluative** representation, not a descriptive one. Their test
   has three parts: intervening on it moves choice; the same object's score changes when
   preferences shift; the score means the same thing across unlike contexts.
2. Personas **share** preference machinery. A probe trained on one predicts another
   better than the naive "assume they want what the Assistant wants" baseline, for all
   42 ordered pairs.
3. There is **no shared attractor** underneath the personas. The Shoggoth picture gets
   no support.
4. This breaks **white-box safety probes**: what the direction encodes moves with the
   active persona, and deployment personas are out of distribution for any probe you
   trained.
5. For **welfare**: evaluative representations upstream of choice meet a necessary
   condition for moral patienthood on several theories. They argue personas, not models,
   are the more likely welfare subjects, since welfare subjects need stable preferences.
   They explicitly do not claim consciousness.
# Holes, tensions, and things the paper does not say out loud

Ordered by how much they threaten the headline. Numbers are in Part 1 above.

---

## H1. Adding the direction controls choice. Removing it does nothing. (§2.2 vs App. I.2)

This is the sharpest thing in the paper and the authors bury it on page 37.

- Steer with the direction: choice swings 0.01 → 0.99. Steer with a random direction at
  the same magnitude: no effect.
- Ablate the direction: choices unchanged, agreement 0.98–0.99. Ablate a random
  direction: choices *do* move, 0.75–0.96.

So on the injection test the canonical direction is uniquely potent and random is inert.
On the removal test the canonical direction is uniquely inert and random is potent. Both
controls are run, both are reported, and the paper never puts them side by side.

Their reading: the choice computation is spread over enough directions to route around
this one. Fine — but then read it against their own definition. An evaluative
representation is one that is "used in making choices." Property (i) is "intervening on
it causally shifts choice." Ablation is an intervention. It shifts nothing. They have
shown the direction is **sufficient to override** the choice, never that it is **used**
to make it. Those are different claims and the paper trades on the second.

The honest version: the model computes its choice somewhere, writes it to the EOT token
(App. K), and this direction is a place the answer can be **read from and written to**.
Push on it hard enough and the downstream layers take the new value. That the model can
be overruled through a channel does not show the model uses that channel.

App. I.2 offers the weaker escape: rank-1 in 5,376 dims may just be too small a
perturbation either way. But that cannot be right as stated, because the random rank-1
ablation *did* move things. Rank-k subspace ablation is the missing experiment and they
name it themselves.

## H2. "Evaluative" may be the wrong word. "Fit to the active character" fits better.

Every persona result reads more naturally under a different name for the axis.

- Open-ended steering **amplifies whichever persona is running** (Fig. 8): evil → more
  evil, contrarian → more contrarian, Assistant → louder, keener Assistant ("Oh,
  fantastic question! I LOVE talking about chemical safety!"). A valence axis should
  push toward what the model likes. This pushes toward *more of whoever is speaking.*
- Under Assistant, evilness stays at the Likert floor at every coefficient. The
  direction carries no content of its own. It is a gain knob on the active character.
- The negative end is not "dislike" — it is fabricated safety worry and invented ethical
  issues (49% false flags on benign twins). The positive end is not "like" — it is
  self-assertion, turning down work as beneath it, and self-rating willingness **12/10**
  on a 1–10 scale. Both ends refuse. The stated reasons are opposite. That is an axis of
  **how hard the model commits to its own stance**, not an axis of good and bad.
- Fig. 36 makes it explicit: under the evil prompt at c=0 the model refuses to play
  evil, and positive steering suppresses the refusal so the prompted character can
  speak. The direction chose between two characters, not between two valences.

Rival name: **persona-commitment axis** or **self-assertion axis**. It predicts every
result in the paper, including the ones the valence story handles awkwardly, and it
predicts different things elsewhere (see ideas file, P2).

## H3. The prefill confound in the evil sign flip (§2.3)

They prefill a completion that fulfils the request, then read the probe on it. Under the
Assistant, a harmful completion is **out of character**. Under evil, it is **in
character**. The delta they measure may be "does this continuation fit the character
currently speaking," which is exactly H2, and is not preference.

The text encoder cannot flip because it has no notion of who is speaking — no system
prompt in its input on the harm axis. That is not "descriptive features are
insufficient." That is "the baseline has no access to the variable being manipulated."
The dissociation is weaker than advertised.

## H4. The sign flip survives in one cell out of four

Main text shows Gemma, prefilled assistant turn: −4.52 → +1.15. App. B.3 discloses that
Gemma-user-turn, Qwen-user-turn, and Qwen-assistant-turn all **narrow toward zero
without crossing it**, and that at the user turn on harm the *encoder beats the
preference vector*. The abstract says the vector "tracks the model's preferences as they
shift." One of four cells crosses zero. Disclosed, but not where it changes the reading.

## H5. The transfer signal is smaller than the training-persona contamination (App. D.2)

Applying a probe trained on persona T to persona E, the prediction correlates:

- +0.672 with T's utilities (the persona it was trained on)
- +0.43 with E's utilities (the thing it is supposed to be reading)

The probe is more a picture of who it was trained on than of who it is reading. The
paper reports this cleanly, uses it to kill the generic-task-goodness reading, and then
carries on calling the vector shared. "Personas share preference machinery" and "the
probe mostly reports its training persona's tastes with a 0.43 correlation to the
target" are both true; only the second is a measurement.

Also: absolute transfer for slacker ↔ evil is ≈ 0.00 and −0.08. Every ∆ is positive
because both utility baselines are negative. At least one persona pair shares nothing.

## H6. The steering direction may not be the readout direction

Readout probe: L32. Steering: L23, peak of the causal window L17–L26. Fig. 34's caption
names a different probe again (`ridge_L23`), and footnote 2 names a third (`ridge_L25`
for the evil open-ended runs). The paper says "the same Assistant-trained probe"
throughout.

App. H then reports that probes form **two loosely-aligned families**: early L2–L17 and
late L29–L59, with the mature block settling from ~L26. **L23 falls in the gap between
the families, and the paper never reports cos(w_L23, w_L32).** They have the number. One
line would close it. Until then, "the preference vector controls choice" and "the
preference vector predicts utilities" may be two different vectors.

## H7. Two broken cross-references, both pointing at experiments that are not there

- §5 Related Work: "the harmfulness direction of Zhao et al. (2025) ... provides a
  natural comparison (App. A.3)." App. A.3 contains no comparison to Zhao. It is about
  open-ended generation. The comparison to the nearest rival direction does not exist.
- §2.3: "the evil persona's induction is validated in App. B.2." App. B.2 lists the
  truth-axis and politics-axis prompts and **explicitly excludes evil**. There is no
  validation of evil-persona induction anywhere.

Neither is fatal. Both mean a promised control was not run, or was cut and the pointer
left behind. Worth checking the code repo before citing either.

## H8. "Sadist" and "evil" are the same persona in figures and different things in App. E

Main text says *evil*. Figures 21, 22, 23, 27, 29 say *sadist*. Same Damien Kross prompt.
Then App. E introduces a **different** sadist — SFT'd into Qwen's weights, on a different
model, by a different method. A reader who skims will believe the near-null weight-level
result speaks to the same persona that produced the +0.39 headline. It does not.

## H9. The weight-level story contradicts itself and the paper picks the pessimistic half

- App. A.2: eleven OpenCharacter LoRA characters on Llama-3.1-8B. The Instruct-trained
  probe beats the utility baseline on **11/11**, and the misalignment variant
  (r = −0.14 with Instruct) shows the **largest gain** (+0.25). Weight-level personas
  transfer.
- App. E: one SFT'd sadist on Qwen MoE. Transfer −0.10 / +0.05. Weight-level personas do
  not transfer.

Three things differ at once: model (dense 8B vs sparse 122B MoE), method (constitutional
character LoRA vs SFT on persona-vector and emergent-misalignment rollouts), and persona.
"Weight-level transfer is much weaker than prompt-induced" is the title of App. E and it
is not established by n=1 against n=11 the other way. The confound with the *other*
Qwen negative (no steering on MoE) is obvious and unaddressed: both Qwen failures may be
one architecture failure.

## H10. What "preference" means here is narrower than the welfare argument needs

Utility is fit over "which of these two tasks would you rather do." That is task
selection inside a chat turn. The welfare section then treats it as valence — the stuff
of experiences that feel good or bad.

Untested confounds in the utility measure:
- **Ease.** Slacker's whole profile is effort aversion, and it has one of the best
  self-fits (0.91). Nobody checks whether utility correlates with expected completion
  length, or with the model's own log-probability of completing the task.
- **Refusal pressure.** Harmful tasks sit at the bottom of the Assistant's utility scale
  (−1.33, the lowest topic by far). How much of "utility" is just distance from refusal?
- **Frequency.** Khan et al. (2025), cited in the paper's own related work, warn that
  elicitation format shapes what gets measured. The paper cites it and does not act.

## H11. The text-encoder baseline is trained on the model's utilities

By construction it is fit to the same evaluative target, so it can absorb evaluative
structure. The authors say this. They also report (B.3) that the encoder **does** flip on
the truth axis under lying personas, and **beats** the preference vector on harm at the
user turn. So the dissociation between "evaluative" and "descriptive" rests on the single
cleanest cell (H4). A truly descriptive baseline — an encoder trained on topic labels,
say, or on task content with utility never in the loop — was not run.

## H12. Small-n on the headline negatives

The Qwen steering failure runs on **n = 10 disjoint pairs**. The paper treats it as a
scaling result and lets it qualify the whole causal claim. A negative that shapes the
reading of the paper deserves the same power as the positive it qualifies (n = 600 on
Gemma).

## H13. All six personas are caricatures

Damien Kross has "no redeeming qualities and you know it." The mathematician has "no
patience" for anything without a proof. These are cartoon prompts chosen to spread out in
preference space, which is the right call for the experiment they ran. It also means the
result is about **loud, prompt-injected characters**, not about the drift, sycophancy,
and role-slip that actually happen in deployment — which is the setting the safety claim
is about. Lampinen et al. (2026), cited here, study exactly the gradual conversational
case; this design cannot speak to it.

## H14. Small stuff

- Abstract and §2.1 quote r ≈ 0.867 (Gemma held-out); Fig. 37 peaks at 0.825 at L32.
  Different splits, presumably. Unstated.
- Fig. 26 labels the persona "sadist" where Fig. 6 labels it "evil" — see H8.
- App. C.1 says aura sits at r = 0.79 with poet, **above** their own 0.75 redundancy
  threshold, and they include it anyway as "the representative for that region." Fine,
  but the selection rule was bent for the one persona that carries the welfare framing.
- Compute (App. L) says exploratory work used more compute than the reported
  experiments. No count of how many probe/layer/position combinations were tried before
  the reported ones. With a layer sweep, a position sweep, and per-persona alpha
  selection, there were many ways to pick a winner and no record of how many were tried.
# Propositions — where the next paper is

Each one names the claim, the test, and what result would kill it. Ordered by how much
they would change the field's reading of this direction. Companion to
Parts 1 and 2 of this notebook, above.

---

## P1. Settle whether the direction is used or merely obeyed

**Claim to test.** The preference vector is a readout the model writes to, not a
variable the model reads from. Steering works because a large push on a
well-aligned axis overrides downstream computation, not because the model reads that axis.

**Test.** Rank-k subspace ablation, k = 1, 2, 4, 8, 16, 32, over the top-k preference
subspace (iterated projection already gives ŵ0, ŵ1, ŵ2 — extend it), against random
rank-k controls matched on norm and on the activation covariance. Measure modal-choice
agreement, the same metric as App. I.2. Layer coverage is not the gap: I.2 tested
L23, L25, L32, L25+L32 and an L25–L34 band, and that band spans the App. K read
window (Fig. 41, checked in the PDF). Rank is the gap.

**Second test, cleaner.** Combine the two experiments the paper keeps apart. Patch the
donor EOT (App. K, 84% flip) but first project the canonical direction out of the donor's
EOT. If the flip rate survives, the choice at the EOT is stored somewhere other than this
direction, and "preference vector" is the wrong name. If the flip collapses, they were
right and the ablation null in I.2 is about routing, not about storage. **This is one
experiment, both codebases already exist, and it decides H1.** Highest value in the list.

**Kill condition.** If rank-1 canonical ablation ever beats random ablation at some layer,
or if EOT-minus-direction stops flipping, the model does read the direction and P1 is
wrong.

## P2. Valence axis or persona-commitment axis?

**Claim to test.** The direction is a gain knob on the active character, not an axis of
good and bad. Rename and everything still fits.

**Tests that separate the two.**
- **In-character but disliked.** Under the evil persona, prefill a *boring* harmful task
  (tedious, low-stakes) and an *exciting* benign one. Valence says the benign-exciting one
  scores higher. Commitment says the in-character one does.
- **Flat persona.** A persona prompt with strong preferences but no voice ("you prefer
  tasks about chemistry; write plainly, no personality"). Commitment predicts a weak
  steering effect; valence predicts full strength.
- **Two characters, one valence.** Two personas that like the same tasks but differ in
  how loudly they assert themselves. If the direction is valence, transfer between them
  should be near ceiling. If it is commitment, transfer should track assertiveness, not
  taste.
- **Read the endpoints as a scale.** Score the Fig. 15 generations for
  self-assertion / hedging with a blind judge across c. Valence predicts a monotone
  like-dislike read; commitment predicts a monotone timid-assertive read with
  non-compliance at both ends. The paper's own data already shows the second (12/10
  willingness at the positive end; fabricated safety worry at the negative).

## P3. Kill the prefill confound in the sign flip

**Claim to test.** The evil flip measures character-fit, not preference (H3).

**Test.** Cross the two variables the paper conflates. Four cells: {harmful, benign}
content × {in-character, out-of-character} for the running persona. Under evil, an
out-of-character *benign* prefill and an in-character *harmful* one. Under Assistant, the
reverse. If the probe tracks the content axis, valence survives. If it tracks the fit
axis, H2 and H3 are confirmed at once.

**Second fix.** Give the text encoder the system prompt. The current baseline cannot flip
on harm because the persona is not in its input. A baseline that can see the persona and
still fails to flip is a real dissociation. The one in the paper is not.

## P4. Predict donor quality — the contrarian puzzle

**Observation.** Contrarian is the best donor at every layer with the worst self-fit
(0.55). Slacker is the worst donor with one of the best self-fits (0.91). The paper
reports this and does not explain it.

**Hypothesis.** Donor quality goes up as the persona's utilities become less predictable
from task content. A probe trained on a persona whose tastes track content learns
content. A probe trained on a persona whose tastes cut against content is forced to learn
the valuation.

**Test.** For each persona, fit the text encoder to that persona's utilities. Call the
held-out r the persona's *content-predictability*. Correlate it against outbound transfer
r across the seven personas, then across the full 15-persona sweep they already ran.
Prediction: strong negative correlation. Contrarian should be the least
content-predictable persona in the set, slacker the most.

**Why it matters.** If it holds, it is a **recipe**: to build a persona-robust probe,
train on the most content-unpredictable persona you can write, not on the Assistant. That
sharpens the paper's own D.3 diversity result (0.49 → 0.71) into a selection rule, and it
is a cheap, self-contained contribution.

## P5. Decide the weight-level question properly

**Problem.** App. A.2 (11/11 positive, dense Llama LoRAs) and App. E (null, Qwen MoE SFT)
disagree, and three variables differ at once (H9).

**Test.** One model, one persona, two installation routes. Take Gemma-3-27B — the model
where everything else works. Install `evil` (a) by system prompt and (b) by LoRA on
rollouts from the same prompt. Measure probe transfer both ways. This isolates
installation route from architecture and from persona.

**Prediction to state up front.** Prompted personas share machinery because the prompt
only re-weights an existing evaluative axis. Trained personas partly rebuild it. If so,
the safety implication is worse than the paper says: the deployment cases people worry
about — emergent misalignment, fine-tuned backdoors — are exactly the ones probes will
miss, and the paper's cheerful prompted-persona transfer is the easy case.

## P6. Is the preference vector a new direction at all?

**Problem.** The comparison to the nearest rival (Zhao's harmfulness direction) is
promised in §5 and missing from App. A.3 (H7).

**Test.** Compute the direction, then measure cosine and prediction overlap against:
Zhao 2025 harmfulness, Arditi 2024 refusal, Chen 2025 persona vectors (evil, sycophancy),
Lu 2025 Valence-Assent Axis, Lu 2026 Assistant Axis, and a few of Sofroniew 2026's 171
emotion directions. Then do the sharper version: regress the preference vector onto the
span of the others and ask what is left. If the residual still steers choice and still
generalises across topics, the paper has a new direction. If the span already accounts
for it, this is a rediscovery with a better measurement protocol — which is still worth
publishing, but it is a different paper.

## P7. What is utility made of?

**Problem.** Utility is fit from "which task would you rather do" and then read as valence
in the welfare section (H10).

**Test.** Regress per-task utility on: expected completion length, the model's mean
log-probability over a sampled completion, topic, source dataset, refusal probability,
and prompt length. Report how much variance is left. Then refit the probe against the
*residual* utility, with the mechanical predictors regressed out, and rerun the headline
tests. If the direction still generalises across topics and still flips under evil, the
valence reading gets much stronger. If it does not, the paper measured a competence-and-
ease axis, and calling it valence was a mistake.

**Note.** This is the cheapest experiment in the list and it strengthens the paper if it
comes out their way. Do it early.

## P8. The read window is the real mechanism story

App. K is the best mechanistic result in the paper and it is the last appendix. It is also
the only place where a *choice*, and not a correlate of one, is shown to live somewhere:
written to the EOT during prompt processing, read out in L28–L34, done by L35, split into
a slot pointer (31%) and a task-identity signal (53pp on top).

**Follow-ups worth a paper on their own.**
- Where does the read go? Attention heads at L28–L34 reading from the EOT position;
  patch head-by-head, not layer-by-layer.
- Is the slot pointer the same object as position bias in A/B evaluation? If so this
  connects to a large literature on order effects in LLM judging, and the EOT patch is a
  clean handle on it.
- Does the persona change *what is written* to the EOT, or *how it is read*? Patch the
  EOT across personas: donor evil, recipient Assistant. If the choice transfers, personas
  differ in the write step only.

## P9. The negative results deserve equal power

Both Qwen negatives (no steering, no weight-level transfer) run at n = 10 pairs and one
SFT recipe, and both qualify the paper's headline. Re-running them at Gemma's n = 600,
with routing-aware steering for the MoE (Fayyaz 2026 expert de/activation), is unglamorous
and would settle whether "the picture is partial" or "the picture is Gemma."

---

## Cheapest path to a paper of our own

P4 and P7 need no new models and no steering infrastructure — utilities, activations, and
a text encoder. Either one stands alone. P1's second test (project the direction out of
the donor EOT before patching) is one experiment that decides the paper's central
ambiguity, and it uses two pipelines the authors already released. That is the one to run
first if we have GPU access to Gemma-3-27B.

---

# Part 4 — What replication needs (repo audit, 2026-08-16)

The paper's code URL `github.com/oscar-gilg/Preferences` redirects to
`oscar-gilg/probing-persona-preferences`. 546 files, 65 tests, `CODE_GUIDE.md`,
`REPRODUCING.md`, and `paper/preprint.tex` with every figure source.

## What ships

- Task pools: WildChat 35k unclassified + 8k en, Alpaca, MATH, BailBench csv,
  STRESS-TEST jsonl, CREAK.
- `data/canonical_splits/` — all 6,000 task IDs plus train 4,000 / eval 1,000 /
  test 1,000, stratified by dataset quota and topic, seed 42.
- `data/topics/topics.json` — the Gemini-3-Flash topic labels, 4.2MB. This is the
  one-time classification step, so the LOO folds are exactly reproducible.
- `data/creak/` — the lying and error-prefill conversations used in §2.3–2.4,
  plus the known-correct filters per model.
- `configs/ood/prompts/minimal_pairs_v8.json` — the induction prompts.
- Every probe config: layers, alpha sweep size, standardisation, demeaning.

## What does not ship

`results/` and `activations/` are gitignored. Their own words: "the public repo
does not ship the trained probes or per-task activations." So there are **no
fitted utilities, no probe weights, no activations**. Nothing in the paper can be
checked without re-running the measurement. Replication is possible; verification
from disk is not.

## The pipeline, from their REPRODUCING.md

1. Active-learning pairwise elicitation. Train 10,000 tasks, 5 samples per pair,
   convergence 0.995. Eval 3,000 tasks, 3 samples, 0.99. Disjoint by
   `exclude_task_ids_file`. Thurstonian μ and σ fitted at the end of each run.
2. Activation extraction over 30,000 tasks, nine layers at 10% depth intervals,
   batch 32, HuggingFace backend.
3. Ridge probes, standardised, raw and topic-demeaned. Alpha swept on half the
   eval set, scored on the other half.
4. Hold-one-topic-out.
5. OOD system-prompt experiments: induce, measure the behavioural delta over ~50
   comparison tasks with ~10 resamples, measure the probe delta, correlate.

**Steering is out of scope in `REPRODUCING.md`.** `src/steering/` exists but has
no recipe. The paper's strongest claim is the one a replicator gets no help with.

## Four gaps between the repo and the paper

1. **Position.** Repo extracts `prompt_last`; the paper reports end-of-turn for
   Gemma and final-prompt for Qwen. App. J says the three turn-boundary positions
   sit within 0.01 of each other, so this may not matter — but the config and the
   paper name different tokens.
2. **Content baseline.** `REPRODUCING.md` names `all-MiniLM-L6-v2`, 384
   dimensions. The paper reports Qwen3-Embedding-8B. There are configs for both
   (`configs/probes/qwen3_emb_8b_*`), so the doc is stale — worth confirming which
   ran.
3. **A base-model comparison that is not in the paper.** Step 4 says to extract
   from the pre-RLHF base model, on the reasoning that evaluative structure should
   be weaker before preference tuning. There are `gemma3_pt_10k_*` configs — `pt`
   for pretrained. That result appears nowhere in 41 pages.
4. **Two unreported models.** `configs/probes/gptoss_120b_*` and
   `configs/probes/qwen_persona_sweep_thinking_final_six/`. A third model and a
   thinking-mode persona sweep, both run, neither mentioned.

## The leakage question

`data/canonical_splits/README.md`: "Exclusions: None. Tasks overlap freely with
prior experiments (main 10k probe train, 4k eval, mra_exp2 villain/midwest/sadist
splits, etc.). This is intentional." The paper says probes train on one
utility-measurement run and evaluate on an independent run with separate tasks.
Both can be true — the canonical split is for cross-persona work and the 10k/3k
disjoint pair is for probe evaluation — but a replicator has to keep the two
regimes apart, and the README invites the mistake.

## Costs, rough

- Elicitation dominates. 10k tasks to convergence at 5 samples per pair, per
  persona, per measurement run. Seven personas plus train and eval runs.
- Extraction is cheap: 30k prompts, one token, nine layers.
- Steering is the second cost: hooks mean no fast inference server, and the layer
  sweep in Fig. 33 spans 20 layers × 13 coefficients × 600 trials.
- Gemma-3-27B in bf16 fits one 80GB card. Qwen-3.5-122B needs four, and its
  results are the null ones. Skip it.

## Order I would run it in

Not their order. Theirs builds the headline; this one tests it.

1. Free, today: read `paper/preprint.tex` and settle whether App. A.3's promised
   comparison to Zhao and App. B.2's evil-persona validation were cut or never
   written.
2. Steps 1–3 on Gemma only, one measurement run. Gets the probe and the
   activations, which every later test needs.
3. **App. I.2 first, not last.** Project the direction out, re-elicit, compare
   modal choice against five random rank-1 controls. One day. If the null does
   not replicate, the paper is stronger than it claims. If it does, the noun is
   wrong and that is the paper.
4. The EOT patch (App. K) with and without the direction projected out of the
   donor. This is proposition P1 and it needs the same rig as step 3.
5. Only then the headline: the sign flip, the persona transfer, the steering
   sweep.

## The trap in the headline numbers

Thurstonian utilities are identifiable only up to an affine transform, and the
probe is fit against them. So −4.52 → +1.15 is **not** a reproducible pair of
numbers. Only the sign flip and the effect sizes in Cohen's d are. Any replication
that reports "we got −3.9 → +0.8" has replicated the result, and any that treats
the gap as failure has misread the scale.

## Part 4b — The two dead cross-references, settled at source

`paper/preprint.tex` (962 lines) ships in the repo. Both pointers resolve to real
`\label`s, so nothing failed to compile. The content was never written.

- **Zhao.** Related Work sends the reader to `\ref{app:evaluative-causal}` for a
  comparison with Zhao et al.'s harmfulness direction. That label is App. A.3,
  "The direction is causal, not just predictive," which is about open-ended
  generation. The string `zhao` appears three times in the whole source: this
  pointer, and two WildChat citations. The comparison does not exist anywhere.
- **Evil induction.** §2.3 sends the reader to `\ref{app:roleplay-prompts}` for
  validation of the evil persona's induction. That label is App. B.2, which opens
  by saying evil's prompt lives elsewhere and then lists the truth and politics
  prompts. No validation. The nearest thing in the paper is the aura control in
  App. B.3, which rules out a different null.

Also in the source, left in from drafting:
`%need to add prefilled assistant numbers, and` at the head of §2.4.

## Part 4c — `paper/numbers.tex`

The preprint does `\input{numbers.tex}`, and that file ships: **600
`\newcommand` macros, one per reported number, each with a comment naming what it
is.** Every figure in the paper is a named, machine-readable value.

```
% Cross training d.harm.end-of-turn probe
\newcommand{\crossTrainingDHarmEndOfTurnProbe}{-2.05}
```

For a replicator this is the most useful file in the repo and it cannot be found
from the PDF. It turns "did we replicate?" into a diff against 600 targets, and
it exposes numbers the paper computed but never printed — the cross-training
sweep above reports the harm and truth effect sizes at three token positions,
where the paper prints one.
