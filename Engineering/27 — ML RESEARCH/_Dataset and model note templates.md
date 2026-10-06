---
tags: [template, research, dataset, model]
---

# _Dataset and model note templates

The two things you must be able to describe six months later. Section: [[27 — ML RESEARCH]]

> **A model without a written record of its data and its training config is not reproducible, and an unreproducible model cannot be trusted or improved.**

---

## Dataset note

```markdown
---
tags: [dataset, <domain>]
---

# DS — <name> v<version>

**Source:** <where it came from, with a link>
**Licence:** <and whether that permits what I'm doing>
**Collected:** <date range> · **Frozen:** <date>
**Size:** <rows> × <columns>, <MB/GB>
**Location:** <path / bucket / table>

## What one row is
The **grain**, in one sentence. "One sensor reading from one engine at one minute."
*If I can't state this, every aggregate I compute later will be wrong.* ([[Data modeling]])

## Schema
| Column | Type | Meaning | Available at prediction time? |
|---|---|---|---|
| … | … | … | **yes / NO — leakage risk** |

## Target
What is being predicted, exactly. Class balance if classification.

## Splits
How, and why that way. Random / by date / by entity.
*Time series → by date. Repeated entities → by entity.* ([[Classical ML in practice]])

| Split | Rows | Date range | Positive rate |
|---|---|---|---|

## Known issues
- Missing values: which columns, and does missing mean something?
- Duplicates, outliers, unit inconsistencies
- Sensor or process changes mid-collection
- **Bias:** who or what is under-represented, and who does that harm?

## Personal data
Any PII? Under what basis is it held? What must be deleted, and when? ([[Security in practice]])

## Related
[[…]]
```

---

## Model note

```markdown
---
tags: [model, <project>]
---

# MODEL — <name> v<version>

**Run:** <MLflow run id> · **Commit:** <git sha> · **Trained:** YYYY-MM-DD
**Dataset:** [[DS — <name> v<version>]]
**Artifact:** <path / registry URI> · **Format:** ONNX / joblib / safetensors

## What it does
Input → output, in one sentence, with units.

## Architecture / algorithm
<LightGBM | ResNet-18 | …>, <n> parameters.

## Training config
| | |
|---|---|
| Loss | |
| Optimiser, LR, schedule | |
| Batch size, epochs | |
| Seed(s) | |
| Hardware, wall-clock, cost | |

## Results
| Metric | Validation | Test | Baseline |
|---|---|---|---|

**Decision threshold:** <value> — chosen because <cost of a miss vs a false alarm>.

## THE SERVING CONTRACT
*Getting any of this wrong produces plausible, confidently wrong predictions.*

- **Feature order:** `[…]` — exact, ordered
- **Dtypes:** …
- **Categorical values:** the exact set seen in training, and what happens to an unseen one
- **Preprocessing:** shipped inside the artifact? *(it should be — save the whole pipeline)*
- **Expected input ranges:** …

## Where it fails
Known weak cases. Inputs it should refuse rather than guess on.

## Monitoring
What is watched in production, and what triggers a retrain. ([[Monitoring and iteration]])

## Related
[[Model export and serving]] · [[MLflow experiment tracking]]
```

---

> **The serving contract is the section that saves you.** A model given its features in the wrong order does not crash — it returns numbers that look entirely reasonable and are meaningless. It is the hardest ML bug to notice ([[Model export and serving]]).

> **Version the dataset, not just the model.** "Accuracy dropped" is unanswerable if you can't tell whether the data changed.

## Related

[[27 — ML RESEARCH]] · [[_Paper note template]] · [[_Experiment note template]] · [[MLflow experiment tracking]] · [[Model export and serving]] · [[Data modeling]] · [[Classical ML in practice]] · [[Monitoring and iteration]] · [[Security in practice]]
