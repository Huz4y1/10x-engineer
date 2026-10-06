---
tags: [moc, research]
---

# 27 — ML RESEARCH

> Reading, reproducing and producing research.

**Why it matters:** Engineering asks 'does it work?'. Research asks 'is it true, and how would I know?'

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

## Finding papers

arXiv (cs.LG, cs.CV, cs.RO) · Papers With Code · Semantic Scholar · conference proceedings (NeurIPS, ICML, ICLR, CVPR, ICRA)

## Reading a paper — three passes

1. **Title, abstract, figures, conclusion** (5 min) — is this relevant?
2. **Full read, skip proofs** (1 hr) — what did they do, what's the claim?
3. **Reproduce the maths and method** (several hrs) — could I implement it?

> **Read the experiments section critically.** What baseline did they compare against? Was it tuned as carefully as their method? What's missing from the ablations? Most weak papers are weak here, not in the idea.

## Reproducing

Start from the official code if it exists · match the exact dataset split · match hyperparameters · **expect not to match the reported number** — report what you got

## Running experiments

Baselines first · change one thing at a time · **ablations** (remove each component, show it mattered) · multiple seeds · report variance, not just the mean · track everything in [[MLflow experiment tracking]]

> **A result from one seed is not a result.** Deep learning variance across seeds is often larger than the improvement being claimed.

## Templates

- **[[_Paper note template]]** — citation, problem, method, results, limitations, does it apply to me?
- **[[_Experiment note template]]** — hypothesis, setup, what changed, result, conclusion
- **[[_Dataset and model note templates]]** — source, splits, licence, known issues; architecture, training config, and the serving contract

## Related
[[_Paper note template]] · [[_Experiment note template]] · [[_Dataset and model note templates]] · [[09 — DEEP LEARNING]] · [[10 — PYTORCH]] · [[MLflow experiment tracking]] · [[31 — CAREERS]]
