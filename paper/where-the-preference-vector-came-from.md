# Where the Preference Vector Came From

A short history of one idea in machine learning, written for someone who does
not work in it, ending at a paper from May 2026 and a judgment about whether it
is worth building on.

Every date and claim below was checked against the arXiv listing or the source
paper. The last section says exactly what was checked and what was not.

---

## The idea underneath everything

A language model, while it reads and writes, holds its state as a long list of
numbers — several thousand of them, updated at every layer of the network. For
more than ten years researchers have been finding that particular combinations
of those numbers stand for particular things, and that you can do arithmetic on
them. That single fact is the thread the whole story hangs on.

## 2013 — meanings turn out to have directions

Mikolov and colleagues at Google publish word2vec. Take the numbers standing for
"king", subtract "man", add "woman", and you land next to "queen". Meaning is not
scattered at random through the numbers. It runs along directions, and directions
can be added and subtracted.

## 2016 — the first direction anyone deletes on purpose

Bolukbasi and colleagues find the direction that separates "he" words from "she"
words and subtract it, to cut gender bias out of a word model. Finding a
direction and removing it becomes a technique rather than an observation.

## 2020 — the careful way to remove one

Ravfogel's team publish iterated nullspace projection. Find the direction,
remove it, train again on what is left, remove the next one, and keep going
until nothing of the concept survives. Six years later it is still the standard
tool, and the paper this history ends at uses it.

## 2022 — models get preferences, and someone questions who has them

Two things happen in the same year.

Companies begin training models on human feedback — OpenAI's InstructGPT in
March is the visible start. A chatbot now has settled tastes: things it will do
gladly, things it declines. Before this it mostly just continued text.

And a pseudonymous writer, janus, publishes an essay arguing that a language
model is not a character at all. It is a machine that runs characters, and the
helpful assistant we talk to is simply the one we keep asking for. Jacob Andreas
makes a related argument in an academic paper the same year, and Murray Shanahan
and colleagues put it in *Nature* in 2023. The idea spreads widely. Nobody can
test it.

## 2023 — reading becomes writing

Two groups show you can steer a model by adding a direction into its state
while it is running, rather than only reading directions out of it. Turner and
colleagues in August, Zou and colleagues at Carnegie Mellon in October. If you
know the direction for a concept, you can turn that concept up.

## 2024 — the result that set the pattern

Arditi and colleagues show that a model's refusal — its entire capacity to say
no — is carried by one direction. Delete it and the model stops refusing. Add it
and the model refuses harmless requests.

This becomes the template for everything after: one behaviour, one direction,
decode it and control it. Within a year there are directions for truthfulness,
for deception, for the hidden triggers of sabotaged models. Safety researchers
start using them as monitors — small detectors that watch a model's internal
state for signs it is about to misbehave.

## February 2025 — measuring what a model wants, from outside

A separate line of work, which does not look inside the model at all. Mazeika's
team show a model two tasks and ask which it would rather do. They do this tens
of thousands of times, then fit a single number to each task that best explains
the pattern of choices. This is the standard economic move: stop asking what
someone wants, watch what they take, and call that the preference.

Two findings matter. The choices are consistent enough for the numbers to exist
at all. And they get more consistent in bigger models.

## July 2025 — directions for character

Chen and colleagues at Anthropic find directions for personality traits — evil,
sycophancy, a tendency to make things up — and use them to watch a model's
character drift during training.

## September 2025 — the first serious crack

Wang, Wei, Liu and Chen publish a study of the safety monitors. Their finding:
the probes are not detecting harmfulness. They are detecting instruction
patterns and trigger words, and they fall apart when the situation changes. The
tool the safety field had been building on turns out to be shallow.

## Late 2025 to spring 2026 — the directions multiply

An axis carrying "this is good" and "this is true" at the same time (October
2025). A direction for the default assistant character itself (January 2026).
A finding from Lampinen and colleagues that these directions do not hold still —
the same fact can be encoded as true at the start of a conversation and false by
the end (January 2026). And in April 2026, Sofroniew and colleagues report
emotion-concept directions in a production model, causally shaping what it
writes; the paper this history ends at counts 171 of them.

## 8 May 2026 — preferences from the outside, sharpened

Yamin, Tang, Horvitz and Wilder work out what a model's decisions imply it is
trying to achieve, then ask whether the model can describe its own policy. It
cannot do so reliably.

## 18 May 2026 — the paper

Gilg, Beckmann, Paleka and Butlin do something nobody had done before.

Every direction in this history was found by labelling. A researcher decides in
advance what evil looks like, writes examples of evil answers and ordinary ones,
and takes the difference. The direction can only ever be as good as the label,
and the label came from the researcher.

Gilg and colleagues skip the label. They run Mazeika's procedure to get a number
for each task from the model's own choices, then train a detector to predict
that number from the model's internal state as it reads a single task. The
model's behaviour defines the target. The detector has to find whatever the
model is actually using.

Then they ask janus's 2022 question, and this time they can test it. Make the
model play a different character — a cartoon villain whose tastes run opposite
to the assistant's — and does it want things with the same machinery? Largely
yes. A detector built on the ordinary assistant reads the villain's preferences,
and pushing that same direction steers the villain's choices too.

## 28 May 2026 — the same answer, from reward

Han, Chalmers and Izmailov train models by reward in a maze puzzle with no
meaning attached to it, then look at what the training changed. It attached
itself to a valence axis that was already in the model. Push that axis
afterwards and the model's confidence, its sentiment, and whether it refuses all
change — in situations that have nothing to do with mazes.

## 28 May 2026 — the same axis, in human brains

Radwan and colleagues build a valence direction from nine emotion-triggering
sentences, check it across fourteen language models, and test it against brain
recordings from 123 people watching emotional videos. The direction from the
language models maps onto the neural activity, and 36 separate EEG classifiers,
trained independently, rediscover the same direction on their own.

## 23 July 2026 — the same answer, from damage

When you fine-tune a model on one narrow bad task, it often starts misbehaving
across the board — a result that alarmed the field badly in 2025. Nadaf shows
this works by recruiting character directions that were already in the model.
Remove the subspace and the broad misbehaviour does not appear. Inject it and
it does.

---

## Why the last four entries matter more than any one of them

Look at what those four have in common. Gilg's group starts from what a model
chooses. Han, Chalmers and Izmailov start from what reward does to it. Radwan's
group starts from human brain recordings. Nadaf starts from what damage does to
it. Four starting points with almost nothing in common, inside ten weeks, and
they arrive at the same claim:

**The evaluative axis is already in the model. New behaviour attaches itself to
that axis rather than growing its own.**

No single one of those results should convince anybody. Agreement between four,
reached by methods that share no assumptions and mostly no authors, is a
different kind of evidence. That convergence is the reason to think this line of
work is pointed somewhere real.

Two honest qualifications. All four are preprints, none peer-reviewed at the
time of writing. And the July result is a single-author paper on a mid-sized
open model, so it carries less weight than the other three.

## Is the paper worth building on

Yes — and the valuable part is the instrument, not the finding.

The measurement protocol is new and reusable: derive what the model wants from
its own choices, then train a detector against that, and test it on a fresh set
of choices the detector never saw. It does not depend on a researcher's idea of
what the concept looks like. It works for any model and any kind of preference.
Nobody else has it.

Their interpretation is the weaker half. The causal result — push the direction,
change the choice — holds on one model and failed on the second one they tried.
The section arguing that this bears on whether models can be harmed does not
survive their own appendices. And we think the thing they found is better
described as how hard the model commits to whatever character it is currently
playing, rather than what the model likes. Two of their results point that way:
pushing the direction makes an evil character more evil and a contrarian more
contrarian, and both ends of the axis produce refusals, for opposite stated
reasons.

That argument is set out in full in the companion draft,
`paper/preference-vector-written-to.md`.

None of it damages the instrument. A good measuring device outlives the first
conclusions drawn with it.

---

## What was checked, and what was not

Checked against the arXiv listing this session: submission dates and author
lists for Han, Chalmers & Izmailov (v1, 28 May 2026), Yamin et al. (v1, 8 May
2026), Radwan et al. (28 May 2026), Nadaf (23 July 2026), Wang et al. (4
September 2025), Turner et al. (20 August 2023), Zou et al. (2 October 2023),
and Sofroniew et al. (9 April 2026).

Checked against the source paper, read in full: everything attributed to Gilg
et al., including the 18 May 2026 v2 date, the failed replication on the second
model, and the count of 171 emotion directions, which is their characterisation
of Sofroniew et al. rather than a number we confirmed in that paper.

Not independently checked: word2vec (2013), Bolukbasi et al. (2016), Ravfogel
et al. (2020), InstructGPT (2022), janus (2022), Andreas (2022), Shanahan et al.
(2023), Arditi et al. (2024), Mazeika et al. (2025) and Chen et al. (2025).
These are well known, and all but the first two appear in the source paper's
reference list, but the dates come from memory rather than from a listing pulled
this session.

## Sources

- Mikolov et al., *Efficient estimation of word representations in vector space*, 2013 — [arXiv:1301.3781](https://arxiv.org/abs/1301.3781)
- Bolukbasi et al., *Man is to computer programmer as woman is to homemaker?*, NeurIPS 2016 — [arXiv:1607.06520](https://arxiv.org/abs/1607.06520)
- Ravfogel et al., *Null it out*, ACL 2020 — [arXiv:2004.07667](https://arxiv.org/abs/2004.07667)
- Ouyang et al., *Training language models to follow instructions with human feedback*, 2022 — [arXiv:2203.02155](https://arxiv.org/abs/2203.02155)
- janus, *Simulators*, LessWrong, 2022
- Andreas, *Language models as agent models*, EMNLP Findings 2022 — [arXiv:2212.01681](https://arxiv.org/abs/2212.01681)
- Shanahan, McDonell & Reynolds, *Role play with large language models*, *Nature* 623, 2023
- Turner et al., *Steering language models with activation engineering*, 2023 — [arXiv:2308.10248](https://arxiv.org/abs/2308.10248)
- Zou et al., *Representation engineering: a top-down approach to AI transparency*, 2023 — [arXiv:2310.01405](https://arxiv.org/abs/2310.01405)
- Arditi et al., *Refusal in language models is mediated by a single direction*, NeurIPS 2024 — [arXiv:2406.11717](https://arxiv.org/abs/2406.11717)
- Mazeika et al., *Utility engineering*, 2025 — [arXiv:2502.08640](https://arxiv.org/abs/2502.08640)
- Chen et al., *Persona vectors*, 2025 — [arXiv:2507.21509](https://arxiv.org/abs/2507.21509)
- Wang, Wei, Liu & Chen, *False sense of security*, 2025 — [arXiv:2509.03888](https://arxiv.org/abs/2509.03888)
- Lu, Song & Wang, *A unified representation underlying the judgment of large language models*, 2025 — [arXiv:2510.27328](https://arxiv.org/abs/2510.27328)
- Lu, Gallagher, Michala, Fish & Lindsey, *The assistant axis*, 2026 — [arXiv:2601.10387](https://arxiv.org/abs/2601.10387)
- Lampinen et al., *Linear representations in language models can change dramatically over a conversation*, 2026 — [arXiv:2601.20834](https://arxiv.org/abs/2601.20834)
- Sofroniew et al., *Emotion concepts and their function in a large language model*, 2026 — [arXiv:2604.07729](https://arxiv.org/abs/2604.07729)
- Yamin, Tang, Horvitz & Wilder, *Can revealed preferences clarify LLM alignment and steering?*, 2026 — [arXiv:2605.08556](https://arxiv.org/abs/2605.08556)
- **Gilg, Beckmann, Paleka & Butlin, *Probing persona-dependent preferences in language models*, 2026 — [arXiv:2605.13339](https://arxiv.org/abs/2605.13339)**
- Han, Chalmers & Izmailov, *How's it going? Reinforcement learning in language models recruits a functional welfare axis*, 2026 — [arXiv:2605.30232](https://arxiv.org/abs/2605.30232)
- Radwan et al., *A shared valence axis across modern LLMs and human EEG*, 2026 — [arXiv:2606.00129](https://arxiv.org/abs/2606.00129)
- Nadaf, *Emergent misalignment recruits a pre-existing persona subspace*, 2026 — [arXiv:2607.21356](https://arxiv.org/abs/2607.21356)
