---
tags: [project, machine-learning, idea]
status: idea
---

# Overwatch strategy prediction

Index: [[App projects]] — Systems

---

## The idea

Predict match outcome or optimal team composition from hero picks, map, and rank.

*(Fill in the detail — this note exists so the link resolves.)*

## Frame it before building it

Run it through [[Problem framing]] first — this project lives or dies on the framing:

| Question | Think about |
|---|---|
| **What decision changes?** | "Pick this hero instead" — is that actionable mid-game? |
| **What exactly are you predicting?** | Win probability? Best counter-pick? Very different problems. |
| **Where's the data?** | This is the real blocker — you need match histories at volume |
| **What's the baseline?** | Overall win rate, and per-map win rate. **Beat those.** |

## Likely approach

**Tabular data → gradient boosting, not a neural network** ([[Choosing your approach]]).

| Feature type | Examples |
|---|---|
| Categorical | Heroes picked (both teams), map, game mode |
| Numeric | Team average rank, rank spread |
| Derived | Composition archetype, counter-pick matrix |

```python
import lightgbm as lgb
model = lgb.LGBMClassifier(n_estimators=500)
```

> **The leakage trap here is severe.** Any feature recorded *during or after* the match — final damage, objective time, match duration — predicts the winner perfectly and is useless for prediction. Only use what is known **at the moment you'd make the prediction** ([[Problem framing]]).

## Concepts used

[[Problem framing]] · [[Choosing your approach]] · [[08 — MACHINE LEARNING]] · [[Data modeling]] · [[MLflow experiment tracking]] · [[FastAPI fundamentals]]

## Related

[[App projects]] · [[28 — PROJECTS]] · [[Problem framing]]
