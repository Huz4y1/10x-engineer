---
tags: [template, research, experiment]
---

# _Experiment note template

One note per experiment. Section: [[27 — ML RESEARCH]] · Tracking: [[MLflow experiment tracking]]

> **Write the hypothesis before you run it.** Deciding what you were testing *after* seeing the result is how you fool yourself, and it is very easy to do by accident.

---

## Template

```markdown
---
tags: [experiment, <project>]
---

# EXP-<nnn> — <one line: what I changed>

**Date:** YYYY-MM-DD · **Run:** <MLflow run id> · **Commit:** <git sha>

## Hypothesis
If I <change X>, then <metric Y> will <go up/down> because <reason>.
*Written before running. Not edited afterwards.*

## What changed
**Exactly one thing**, versus EXP-<previous>:

| | Previous | This run |
|---|---|---|
| … | … | … |

Everything else held fixed: data version, split, seed, hardware.

## Setup
- Data: <version / date range / row count>
- Split: <how, and why that split>
- Seeds: <list — more than one>
- Hardware, wall-clock time, cost

## Result
| Metric | Baseline | This run | Δ |
|---|---|---|---|
| … | … ± … | … ± … | … |

*Mean ± std over seeds. A single number is not a result.*

## Verdict
- [ ] Hypothesis supported
- [ ] Contradicted
- [ ] Inconclusive — the difference is inside the noise

## What I actually learned
Including "nothing, the setup was wrong". **Especially that.**

## Next
The single next experiment this suggests.

## Related
[[…]] · [[_Experiment note template]]
```

---

## The rules

| Rule | Why |
|---|---|
| **Baseline first** | Without it, no number means anything |
| **One change per experiment** | Two changes and you can't attribute the result |
| **Multiple seeds, report variance** | Seed noise often exceeds the claimed gain |
| **Record the commit SHA** | "Which code produced this?" must be answerable |
| **Log the failures too** | A negative result stops you repeating it in three months |
| **Ablate** | Remove each component and show it was earning its place |

> **Keep a validation set you tune on and a test set you touch once.** Every time you look at the test set to make a decision, it becomes training data and stops being an honest number ([[Classical ML in practice]]).

> **If the difference is within the seed-to-seed variance, it is not a difference.** Write "inconclusive" and move on — that is a real finding, not a failure.

## Related

[[27 — ML RESEARCH]] · [[_Paper note template]] · [[MLflow experiment tracking]] · [[Classical ML in practice]] · [[Problem framing]] · [[Monitoring and iteration]]
