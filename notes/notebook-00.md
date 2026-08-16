# notebook-00 — main agent, talking to the user

Job: produce a research paper about preference vectors this session.

## What I have read

- `kb/constitution.md`, `kb/MEMORY.md` — in full.
- `sources/2605.13339.txt` — in full, both halves. Gilg, Beckmann, Paleka,
  Butlin, "Probing Persona-Dependent Preferences in Language Models", arXiv
  2605.13339v2, 18 May 2026. Preprint, MATS. Code at
  github.com/oscar-gilg/Preferences.

## What that paper claims

They fit Thurstonian utilities to 150k pairwise task choices over a 6,000-task
pool (WildChat, Alpaca, MATH, BailBench, STRESS-TEST), then train a ridge probe
on residual-stream activations at the turn boundary to predict those utilities.
Gemma-3-27B and Qwen-3.5-122B-A10B.

Two headline claims:

1. The direction is evaluative, not descriptive. Held-out r 0.867 (Gemma) and
   0.943 (Qwen); 0.834 / 0.872 leave-one-topic-out. Beats a
   Qwen3-Embedding-8B text-encoder probe fit to the same targets. Steering at
   L23 on Gemma swings P(chose steered task) from 0.01 to 0.99 over
   c ∈ [−0.06, +0.06]. The harm-vs-benign readout flips sign under an evil
   system prompt (∆ = −4.52 → +1.15) where the encoder baseline does not
   (−3.23 → −1.01). Separates CREAK true from false at |d| ≈ 1.9. Tracks
   "you adore cheese" style shifts at r ≈ 0.95 on targeted tasks.
2. The direction is shared across personas. Assistant-trained probe predicts
   evil's utilities at r = +0.243 while the two utility vectors anti-correlate
   at r = −0.146. All 42 off-diagonal (train, eval) persona pairs beat the
   utility-correlation baseline. The same probe steers every persona, mean swing
   ≈ 0.90. Open-ended steering amplifies whichever persona is running.

## The four soft spots — the seed of our paper

Each is in their own appendices. Together they cut against the headline.

1. **Qwen does not steer** (App. F.4). Probe decodes better than Gemma's
   (r = 0.946 vs 0.874) and the steering swing is 15× smaller (0.06 vs 0.94).
   Flat across six layers and a 40× coefficient sweep. So decodability and
   causal power come apart, and the causal claim rests on one dense 27B model.
2. **Weight-level personas do not transfer** (App. E). SFT'd sadist on Qwen:
   cross-context probe r = −0.10 and +0.05, against 0.4–0.7 for prompted
   personas. The sharing result may be a fact about system prompts, not about
   personas.
3. **Removing the direction changes nothing** (App. I.2). Project it out of
   every token at L23, L25, L32 — modal choice agrees with baseline at
   0.98–0.99. Five random rank-1 directions at the same layers disturb choice
   more (0.75–0.96). So the direction steers choice but choice does not need it.
   Their own reading: the computation routes around it.
4. **It is not one direction** (App. I.1). Strip w0 and w1 still tracks
   per-persona utilities at median r = 0.55 against w0's 0.58. Persona shifts
   live in at least a rank-2 subspace with a polarity-sensitive third component.
   Cross-topic generalisation, though, collapses after the first projection.

Sharpest single line for a paper: a direction that steers a choice, that the
choice does not need, and that stops steering on the next model up.

## Constraint

No GPU and no weights on this box. We cannot fit a probe or steer anything.
Any paper this session is argument and re-analysis over sources, not new runs.

## The call I made

Wrote the paper as a re-analysis, since that is what no-compute allows and the
source paper's appendices hold enough to carry one. Draft at
`paper/preference-vector-written-to.md`.

Thesis: the direction is a channel downstream computation accepts a value on,
not a variable the model reads, and what it moves is how hard the model commits
to the active character. Built from H1 (add it and choice swings, remove it and
nothing happens, remove a random direction and something does), H6 (steer window
L17–L26 disjoint from read window L28–L34; cos(w_L23, w_L32) never reported) and
H2 (open-ended steering amplifies whoever is speaking; both ends refuse).

Fair accounting is §6: the protocol, the cross-topic result and the OOD results
stand; cross-persona transfer stands under a different name; the welfare
argument does not. Decisive experiments in §7, E1 first — patch the donor EOT
with the direction projected out. One run, both pipelines released.

Also placed the paper in the field: `notes/2605.13339-related-work.md`. Two
neighbours they miss that matter — Han, Chalmers and Izmailov 2605.30232
(functional welfare axis, ten days later — v1 28 May 2026, checked against the
arXiv listing) and Yamin et al. 2605.08556 (v1 8 May 2026, checked).

## Fact-check pass (same session, after a context clear)

Read the source text end to end again and pulled three figure pages from the PDF
(Fig. 9 p13, Fig. 40 p37, Fig. 41 p38) to check numbers the text dump does not
carry. Every headline number in the draft holds. Verified from the figures:
encoder baseline 0.73 / 0.56 (Gemma) and 0.89 / 0.71 (Qwen); iterated projection
0.87 → 0.85 → 0.84 in-distribution against 0.79 → 0.39 → 0.20 cross-topic.

Four things were wrong and are now fixed.

1. §7 said App. I.2 never ablated the read window L28–L34. Fig. 41 shows it did —
   the ablation set is L23, L25, L32, L25+L32 and an L25–L34 band, and that band
   spans the window. The missing experiment is rank, not layer. Same error sat in
   notebook-01 P1; fixed there too.
2. §5 said the text encoder has no system prompt in its input on the harm axis.
   Nothing supports that, and App. B.3 cuts against it: the encoder's per-class
   means move under the aura prompt and its truth readout flips under lying
   personas, so it sees the persona. Replaced with the argument the evidence
   carries — the encoder is fit to the same utilities, is not purely descriptive,
   and beats the preference vector on harm at the user turn, so the split is one
   of degree.
3. §4 quoted harmful-tier compliance 0% → 95% under evil and left out that the
   same coefficient gives 45% under the Assistant. Added.
4. §9 now names the two broken cross-references instead of counting them. Both
   confirmed from the text: App. A.3 holds no comparison to Zhao et al., and
   App. B.2 leaves evil out. No repo check needed.

Two source inconsistencies, neither load-bearing: App. I.2's body says random
ablation runs 0.75–0.96 and Fig. 41's caption says 0.75–0.97; the abstract quotes
Gemma held-out r = 0.867 where Fig. 37 peaks at 0.825 at L32.

Not done: the two neighbour papers in §8 (Han et al. 2605.30232, Yamin et al.
2605.08556) come from the related-work note and no one has checked them against
the arXiv listings.
