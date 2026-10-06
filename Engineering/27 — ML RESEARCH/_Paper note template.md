---
tags: [template, research, paper]
---

# _Paper note template

Copy this for every paper you read properly. Section: [[27 — ML RESEARCH]]

> **Only make a note for papers you did pass 2 or 3 on.** A note for every abstract you skimmed is a graveyard, not a library.

---

## Template

```markdown
---
tags: [paper, <topic>]
---

# <Title>

**Authors / venue / year:** …
**Link:** arXiv … · **Code:** github… (or *none*)
**Read to pass:** 1 / 2 / 3
**Date read:** YYYY-MM-DD

## The problem
What was broken or missing before this paper. One paragraph, in my own words.

## The claim
The single sentence the paper is arguing. If I can't write it, I haven't understood it.

## The method
How it works. A diagram or the key equation. Enough that I could start implementing.

## Results
| Benchmark | Theirs | Best prior | Gain |
|---|---|---|---|

## How hard did I look?
- **Baseline** — was it tuned as carefully as their method, or left weak?
- **Ablations** — did they show each component actually matters?
- **Seeds** — one run, or a mean over several with variance reported?
- **Compute** — is the gain from the idea, or from 10x more training?
- **Cherry-picking** — which benchmarks are conspicuously absent?

## Limitations
Theirs (stated) and mine (noticed).

## Does this apply to me?
- Would it help [[<my project>]]? Why or why not.
- What would it cost to try — hours, GPU, data?
- **Verdict:** try it / park it / not applicable

## Ideas it gave me
Even if the paper is weak.

## Related
[[…]] · [[27 — ML RESEARCH]]
```

---

## How to read (the three passes)

| Pass | Time | Read | Come away with |
|---|---|---|---|
| **1** | 5 min | Title, abstract, figures, conclusion | Is this relevant? Usually no — stop here |
| **2** | ~1 hr | Everything, skipping proofs | What did they do, and do I believe it? |
| **3** | Hours | Reproduce the maths and method | Could I implement this from scratch? |

> **Pass 1 exists to let you stop.** Most papers do not deserve pass 2. Being quick to discard is the skill.

> **Read the experiments section adversarially.** Most weak papers are weak in the comparison, not the idea: an undertuned baseline makes anything look good.

> **A result from one seed is not a result.** Variance across random seeds in deep learning is routinely larger than the improvement being claimed.

## Related

[[27 — ML RESEARCH]] · [[_Experiment note template]] · [[MLflow experiment tracking]] · [[09 — DEEP LEARNING]]
