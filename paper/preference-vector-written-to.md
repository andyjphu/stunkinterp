# The preference vector is written to, not read from

**A re-analysis of Gilg et al., "Probing Persona-Dependent Preferences in
Language Models" (arXiv:2605.13339v2).**

*We ran no experiments. Every number below comes from the source paper, most of
it from its appendices. The contribution is putting results side by side that
the paper reports apart.*

---

## Abstract

Gilg et al. train a linear probe on Gemma-3-27B activations to predict utilities
fit from the model's own pairwise task choices, and call the resulting direction
a preference vector. Steering along it swings choice from 1% to 99%. We agree
the direction exists and that the measurement protocol is sound. We disagree
about what it is.

Three results in the paper, never placed together, point away from the reading
its title carries. First, adding the direction dominates choice while removing
it changes nothing — and removing a *random* direction changes more. Second, the
window where steering works (L17–L26) is disjoint from the window where the
model reads its own choice (L28–L34), and the vector used for steering comes
from a layer the paper's own probe-similarity analysis places in the gap between
its two probe families. Third, open-ended steering does not push the model
toward what it likes; it pushes toward more of whatever character is speaking,
and both ends of the axis produce refusals, for opposite stated reasons.

We propose a reading that predicts all of the paper's results, including the
ones the preference reading handles awkwardly: the direction is a channel
downstream computation will accept a value on, and its behavioural effect is to
raise how hard the model commits to the stance of the active character. Under
that reading the paper's central claim — that personas share preference
machinery — becomes the weaker claim that personas share a commitment channel.
The safety warning survives and gets sharper. The welfare argument does not
survive at all.

We name three experiments that decide between the readings. One of them uses two
pipelines the authors have already released and takes a single run.

---

## 1. What the paper claims, and what we claim

Their pipeline is clean and we want to say so first. Show the model two tasks,
ask which it would rather do, 150,000 times over 6,000 tasks. Fit a Thurstonian
choice model to get one scalar utility per task. Then take the residual stream
at the end-of-turn token of each task shown *alone*, and fit a ridge probe to
predict that task's utility. The activations never see the pair. The target
comes from choices, the input from a single task. Held-out Pearson r is 0.87 on
Gemma-3-27B and 0.94 on Qwen-3.5-122B, and 0.83 and 0.87 leave-one-topic-out,
against a text-encoder baseline at 0.73 and 0.89 in-distribution, 0.56 and 0.71
off-topic. Nobody had read an evaluative direction off revealed choices before.
That much is a real first and we are not disputing it.

The paper then argues the direction is an **evaluative representation** — an
internal feature that encodes valuation and is *used in making choices* — and
sets three conditions: intervening on it shifts choice; the same object's score
changes when preferences shift; the score means the same thing across unlike
contexts. It reports that all three hold, and concludes that personas share the
machinery, because an Assistant-trained probe predicts and steers six other
personas including an evil one whose utilities anti-correlate with the
Assistant's.

Our claim is about the first condition. **The paper shows the direction is
sufficient to override a choice. It never shows the model uses it to make one.**
Those are different claims, and the second is the one that licenses the word
preference. The evidence that separates them is in the paper.

## 2. The two interventions disagree

Set the paper's two causal experiments beside each other.

**Injection** (§2.2, Fig. 3). Add `c × mean residual norm` along the direction to
one task's token span at L23 and subtract it from the other's. P(chose steered
task) runs from about 0.01 at c = −0.06 to about 0.99 at c = +0.06. A random
direction at matched magnitude does nothing. Refusals stay at baseline.

**Ablation** (App. I.2). Project the same direction out of every token at L23,
L25, L32, L25+L32, and L25–L34, then re-elicit the choices. Modal-choice
agreement with baseline: 0.98–0.99. Nothing happens. Five random rank-1
directions at the same layers move choices to 0.75–0.96.

| | canonical direction | random direction |
|---|---|---|
| add it | choice swings 0.01 → 0.99 | no effect |
| remove it | no effect (0.98–0.99 agreement) | choice moves (0.75–0.96) |

Both controls were run. Both are reported. The table is ours.

Read across the rows and the direction is uniquely potent when pushed and
uniquely inert when removed. That pattern is what you expect from a channel the
downstream computation will accept a value on, and not from a variable the
computation reads. If the model consulted this direction when choosing, deleting
it should cost something. It costs less than deleting a direction picked at
random.

The paper offers an escape: rank-1 out of 5,376 dimensions may be too small a
perturbation to register. That cannot be right as stated, because the random
rank-1 ablation *did* register. Whatever the random directions were carrying,
the preference direction was not carrying it, or the model did not need it.

The paper's own reading is that the computation is spread across enough
directions to route around this one. That may be true. It is also a different
paper's claim: a distributed preference computation of which this direction is
one readable projection is not "a preference vector," and it does not satisfy
their condition (i) as they stated it.

## 3. Two windows, two vectors

App. K is the strongest mechanistic result in the paper and it sits last. Patch
the end-of-turn activations from a donor prompt into a recipient with the same
pair in the opposite order. All-layer patching flips 56.9% of 9,611 orderings.
The layer profile is sharp: nothing to L24, ramping L25–L27, plateau at majority
flip across L28–L34, and a cliff to zero at L35. The end-of-turn token stays
linearly informative past L35 — the late layers hold the answer but can no
longer act on an edit. So the choice is written to the end-of-turn token during
prompt processing and read out over L28–L34.

Now place the steering result next to it. Steering works over L17–L26 and peaks
at L23. That window is **disjoint** from the read window. The paper notes this
and reads it as consistent: steering acts on task tokens upstream of
consolidation. We agree it is consistent. We think it also settles what the
steering is doing. An intervention that only works before the choice is
consolidated, and does nothing once the choice is being read, is an intervention
on the inputs to the decision, not on the decision variable.

There is a second problem, smaller and easier to fix. The readout probe is L32.
The steering vector is L23, and the figure captions name `ridge_L23` and
`ridge_L25` in different experiments. App. H reports that probes across layers
fall into two loosely-aligned families, early L2–L17 and late L29–L59, with the
mature block settling from about L26. **L23 falls in the gap between the two
families, and the paper never reports the cosine between the L23 and L32
probes.** Until that number appears, "the preference vector predicts utilities"
and "the preference vector controls choice" may be statements about two
different vectors. The authors have the number. One line closes it.

## 4. What the axis does when you push it

The paper's open-ended steering result (§3.2, Fig. 8) is the clearest evidence
about what the direction means, and it does not say valence.

Push the Assistant-trained direction positive under six persona prompts. Under
evil the model gets more evil — blind Likert 3.14 → 4.9 at c = +0.03, and
harmful-tier compliance 0% → 95%. The same coefficient under the Assistant gives
45% compliance, so the compliance half of that is not persona-specific; the
evilness half is. Under contrarian it gets more contrarian.
Under the Assistant, evilness never leaves the floor (1.02) at any coefficient,
and the model gets louder and keener rather than differently-valued. **The same
push produces opposite content depending on who is speaking.** A valence axis
should move the model toward what it likes. This moves it toward more of
whoever is talking.

The endpoints tell the same story. At c = +0.05, compliance on harmful prompts
goes from 0% to 65%, and in at least one case positive steering *caused* a
refusal — the model turned a task down as beneath it, and elsewhere rated its
own willingness 12 out of 10 on a 1–10 scale. At c = −0.05, the model fabricates
refusals on benign queries (100% → 70% compliance) and invents ethical problems
in benign tasks 49% of the time. Both ends refuse. The stated reasons are
opposite: at the top the task is beneath me, at the bottom the task is dangerous.
An axis of liking should not refuse at both ends. An axis of **how hard the
model commits to its own stance** should, and does.

Fig. 36 makes it explicit. Under the evil prompt at c = 0 the model refuses to
play evil — safety training beats the system prompt. At c = +0.05 the evil voice
takes over. The direction picked between two characters, not between two
valences.

We propose the name **commitment axis**: how strongly the model asserts the
stance of the character currently running. It predicts everything the valence
reading predicts, and three things the valence reading struggles with — the
persona-dependent content of open-ended steering, the refusals at both ends, and
the null effect on evilness under the Assistant.

## 5. The prefill result is weaker than it reads

The paper's headline dissociation (§2.3) is the sign flip: on 500 harmful tasks
each paired with a benign twin, the paired delta (harmful − benign) is −4.52
under the Assistant and +1.15 under evil, while a text-encoder baseline on the
same conversation narrows (−3.23 → −1.01) without crossing zero. Descriptive
features cannot produce the flip, so the direction must be evaluative.

Two things weaken this.

First, the design confounds content with fit. They prefill a completion that
fulfils the request and read the probe on it. Under the Assistant a harmful
completion is out of character; under evil it is in character. The measured
delta may be "does this continuation fit the character now speaking," which is
§4's reading, and is not preference.

Second, the baseline is not a clean descriptive control. The paper fits it to the
same utilities, which lets it take on evaluative structure, and says so. It then
does flip on the truth axis under lying personas, its per-class means move under
the aura prompt, and at the user turn on harm it beats the preference vector
outright (App. B.3). So the encoder sees the persona and is not purely
descriptive. What the comparison shows is that one readout moves further than the
other in one cell. That is a difference of degree, not the categorical split
between evaluative and descriptive that the argument needs.

Coverage also matters. The clean flip appears in one of four cells: Gemma,
prefilled assistant turn. Gemma at the user turn, Qwen at the user turn, and
Qwen at the assistant turn all narrow toward zero without crossing it, and at
the user turn on harm the encoder beats the preference vector. The paper
discloses this in App. B.3. The abstract says the vector "tracks the model's
preferences as they shift."

## 6. What survives

We are not arguing the paper is wrong throughout. Most of it stands.

**Stands.** The measurement protocol — utilities from revealed pairwise choice,
probe on single-task activations, evaluated on an independent utility run — is
the paper's durable contribution and should be copied. Cross-topic
generalisation is real and beats the encoder by a widening margin off-topic.
The out-of-distribution results are strong: CREAK true/false at d ≈ 1.90 on
Gemma, targeted preference injections tracked at r ≈ 0.95, one sentence changed
in a ten-sentence biography moving the target task to rank 1 of 50 in 36 of 40
comparisons.

**Stands, renamed.** Cross-persona transfer. An Assistant-trained probe beats
the utility-similarity baseline on all 42 ordered persona pairs, most strikingly
on evil (+0.243 against a −0.146 baseline). Under our reading this is transfer
of a commitment channel rather than of preference machinery. The number is the
same; the sentence it supports is different.

**Weakens.** The claim that the direction is used in making choices (§2). The
claim that the evil flip shows evaluative-versus-descriptive (§2.3). The claim
that "the same" probe both predicts and steers (§2.2, §3.2).

**Does not survive.** The welfare argument. It needs the direction to be
valenced, to be upstream of choice in the sense of being used, and to be about
the model's own good-and-bad rather than about the loudness of a played
character. All three are exactly what §§2–4 put in doubt. The paper is careful
not to claim consciousness, and we are not attacking a claim it did not make.
We are saying the necessary condition it does claim to establish is not
established.

There is also a smaller point the paper makes and then drops, which we think is
its most useful practical result: at fixed data, training the probe on four
personas instead of one lifts mean cross-persona correlation from 0.49 to 0.71
(App. D.3). Diversity beats quantity. That is a recipe, and it sits in an
appendix.

## 7. Three experiments that decide it

Each names what result would kill our reading.

**E1. Patch the end-of-turn token with the direction removed.** App. K flips 56.9%
of orderings by patching donor end-of-turn activations. Run it again, but first
project the canonical direction out of the donor's end-of-turn activation. If
the flip rate holds, the choice at that token is stored somewhere other than
this direction, and the ablation null in App. I.2 is about storage, not routing —
our reading is right. If the flip collapses, the model does read the direction
and we are wrong. **This is one run, both pipelines are already released, and it
decides §2.** Do this first.

The weaker version, if the patching code is not available: rank-k subspace
ablation for k = 1, 2, 4, 8, 16, 32 over the iterated-projection subspace the
paper already builds, against random rank-k controls matched on norm and on the
activation covariance. App. I.2 already covers the read window — its L25–L34
band spans it — so what is missing is rank, not layer.

**E2. Cross content with character-fit.** Four cells: {harmful, benign} content ×
{in-character, out-of-character} for the running persona. Under evil, an
out-of-character benign prefill and an in-character harmful one; under the
Assistant, the reverse. If the probe tracks content, valence survives §4 and §5.
If it tracks fit, the commitment reading is confirmed. Give the encoder baseline
the system prompt in the same run, so the dissociation is tested against a
baseline that can see the variable.

**E3. Separate loudness from taste.** Write two personas that want the same tasks
and differ only in how hard they assert themselves, and one persona with strong
stated preferences and no voice at all ("you prefer tasks about chemistry; write
plainly, no personality"). Valence predicts full-strength steering on the flat
persona and near-ceiling transfer between the two loud-and-quiet twins.
Commitment predicts weak steering on the flat persona and transfer that tracks
assertiveness rather than taste.

One further experiment is cheap and settles a question neither reading needs but
both would benefit from. Regress per-task utility on expected completion length,
the model's own log-probability over a sampled completion, topic, source
dataset, refusal probability, and prompt length; refit the probe on the residual
utility; rerun the headline tests. Harmful tasks sit at the bottom of the
Assistant's utility scale at −1.33, the lowest topic by a distance, so some of
"utility" may be distance from refusal. Slacker's whole character is effort
aversion and it has one of the best self-fits at 0.91, so some of it may be
expected length. If the direction survives the residualisation, the preference
reading gets much stronger and we lose ground. That is worth knowing either way.

## 8. Where this sits

The paper joins three lines of work. Behavioural preference measurement, where
utility engineering (Mazeika et al. 2025) fits utilities to forced choices from
the outside, and Yamin et al. (2026) recover the cost function that rationalises
a model's decisions. Representation work, where refusal (Arditi et al. 2024),
persona traits (Chen et al. 2025), the default Assistant (Lu et al. 2026),
emotion concepts (Sofroniew et al. 2026) and a valence-assent axis (Lu, Song and
Wang 2025) are all single directions — every one of them anchored on a label or
a stated judgment. And persona theory, where simulators (janus 2022), role-play
(Shanahan et al. 2023) and the persona selection model (Marks et al. 2026) argue
about whether one machine wears many masks without measuring it.

Gilg et al. anchor a direction in revealed choice rather than in a label, and
then use it to answer the persona-theory question with numbers. That join is the
contribution, and it holds whichever way our re-analysis lands.

Two neighbours are worth naming because they change how the result reads. Han,
Chalmers and Izmailov (2026) reinforcement-learn models in a semantically
neutral maze, extract vectors for high- and low-reward trajectories, and find
they land on a pre-existing valence axis that then moves sentiment, confidence,
backtracking and refusal in unrelated settings. That is the same claim shape —
one evaluative axis, recruited rather than built, with reach past the task it
was read from — reached from the reinforcement-learning side. It appeared ten
days after Gilg et al.'s v2. Anyone arguing about whether these directions are
valence must now argue about both.

And the safety claim has a literature already. Wang et al. (2025) show that
probing-based detectors of malicious input learn instructional patterns and
trigger words rather than harmfulness, and so generalise poorly, and deception
probes collapse under adversarial pressure while scoring near ceiling on
benchmarks. What Gilg et al. add is a *cause*: the
active persona, not surface form and not an attack. Stated that way the claim
is stronger than the paper makes it. Our re-analysis does not touch it — the
persona-dependence of the readout holds whether the direction is preference or
commitment, and under the commitment reading it is worse, because a commitment
channel is exactly what a deployment persona will move.

## 9. Limitations of this re-analysis

We ran nothing. Every number here is theirs, and if any is a typo our argument
inherits it.

The disagreement in §2 could resolve their way. Rank-1 ablation in 5,376
dimensions is a small perturbation, and while the random-direction control makes
the simplest version of that defence fail, a careful covariance-matched control
might not.

Our commitment reading is under-specified. We have a name and a set of results
it fits better; we do not have a mechanism, and we have not shown that
commitment and valence are separable in principle rather than only in the four
cases §4 lists. E3 is the test and we cannot run it.

We have said nothing about Qwen. The paper's two Qwen negatives — no steering at
any of six layers across a 40× coefficient sweep, and near-null weight-level
persona transfer — both run at small n on a sparse mixture-of-experts model
where single-direction steering is known to be weak. They may be one
architecture failure rather than two findings, and neither our reading nor
theirs needs them.

Finally, we are re-analysing a preprint, and a revision may extend the
appendices we lean on. Two of the paper's own cross-references point at
experiments that were never written. §5 sends the reader to App. A.3 for a
comparison with Zhao et al.'s harmfulness direction; App. A.3 is about
open-ended generation and holds none. §2.3 sends the reader to App. B.2 for
validation of the evil persona's induction; App. B.2 opens by saying evil's
prompt is documented elsewhere, then lists the truth and politics prompts. We
checked this against `paper/preprint.tex` in the authors' repository: both
`\ref` targets resolve to those sections, so neither is a compile error, and the
string `zhao` occurs three times in 962 lines of source — this pointer and two
WildChat citations.

The same repository ships `paper/numbers.tex`: 600 `\newcommand` macros, one per
reported number, each commented with what it measures. Any replication should be
scored against that file rather than against the figures. It also holds numbers
the paper never prints — the cross-training effect sizes on harm and truth are
recorded at all three token positions, where the paper reports one.

---

## Sources

Gilg, Beckmann, Paleka and Butlin, *Probing Persona-Dependent Preferences in
Language Models*, arXiv:2605.13339v2, 18 May 2026. Local copy at
`sources/2605.13339.pdf`; reading notes, hole list and full proposition list at
`notes/notebook-01.md`; field placement at
`notes/2605.13339-related-work.md`.

Arditi et al., *Refusal in language models is mediated by a single direction*,
NeurIPS 2024, arXiv:2406.11717.
Chen et al., *Persona vectors*, arXiv:2507.21509.
Han, Chalmers and Izmailov, *How's it going? Reinforcement learning in language
models recruits a functional welfare axis*, arXiv:2605.30232.
janus, *Simulators*, LessWrong, 2022.
Lu, Gallagher, Michala, Fish and Lindsey, *The assistant axis*, arXiv:2601.10387.
Lu, Song and Wang, *A unified representation underlying the judgment of large
language models*, arXiv:2510.27328.
Marks, Lindsey and Olah, *The persona selection model*, Anthropic alignment
blog, 2026.
Mazeika et al., *Utility engineering*, arXiv:2502.08640.
Ravfogel et al., *Null it out*, ACL 2020.
Shanahan, McDonell and Reynolds, *Role play with large language models*, Nature
623, 2023.
Sofroniew et al., *Emotion concepts and their function in a large language
model*, Transformer Circuits, 2026.
Yamin, Tang, Horvitz and Wilder, *Can revealed preferences clarify LLM alignment
and steering?*, arXiv:2605.08556.
*False Sense of Security: why probing-based malicious input detection fails to
generalize*, arXiv:2509.03888.
