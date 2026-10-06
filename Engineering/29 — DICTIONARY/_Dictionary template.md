---
tags: [template, dictionary]
---

# _Dictionary template

Copy this for every technology page. Same structure every time, so you always know where to look.

Worked example: [[Kafka]].

> **Rule 46:** don't create shallow pages for hundreds of technologies. Foundational things get the full treatment. Minor things get a short page with the first four sections and a link. **Never duplicate** — if a concept is explained deeply elsewhere, link to it.

---

```markdown
---
tags: [dictionary, <category>]
status: not-started
---

# <Technology>

## One sentence

<One sentence a non-specialist would understand. No jargon.>

## In simple words

<An analogy. Something physical and everyday. 2–4 sentences.>

## The problem it solves

<What did people do BEFORE this existed, and why was that painful?
This is the most important section. Never skip it.
Describe the pain concretely — numbers, scale, failure modes.>

## How it works

<The mechanism. Simple version first, then deeper.
What are the moving parts and how do they interact?>

## Important vocabulary

| Term | Meaning |
|---|---|
| <term> | <plain-English meaning> |

## Architecture

```mermaid
<diagram>
```

## Code

### Level 1 — tiny
<The smallest thing that demonstrates the idea.>

### Level 2 — practical
<Something you'd actually build.>

### Level 3 — production
<Config, error handling, the real thing.>

## What happens under the hood

<Rule 39. When you run <the core command>, what ACTUALLY happens,
step by step, inside the system?>

## When to use it

<Concrete situations.>

## When NOT to use it

<Equally concrete. This section is what makes the page trustworthy.>

## Common mistakes

| Mistake | What happens | Fix |
|---|---|---|

## Debugging

<How to reason about a failure here. Step 1, step 2, step 3.
Not just fixes — the diagnostic order.>

## Performance

<What's slow, what's fast, and the knobs that matter.>

## Security

<How this gets attacked and what to do about it.>

## Production considerations

<What changes between a laptop and real traffic.>

## Alternatives

| Alternative | Choose it when |
|---|---|

## Cloud equivalents

| Local | Azure | AWS | GCP |
|---|---|---|---|

## Real-world use

<Which industries, and what for. Aerospace, robotics, finance, etc.>

## Prerequisites

<What to understand first. Link them.>

## Learning progression

- **Beginner:** <what you need>
- **Intermediate:** <what's next>
- **Advanced:** <what professionals know>
- **Research:** <what's still open>

## Practical project

<What to build to actually learn this. Link to [[28 — PROJECTS]].>

## Related

<Heavy linking. Never leave a page isolated.>
```

---

## The seven-layer explanation (rule 43)

For any concept inside a page, explain in this order. Stop at whichever layer the reader needs.

| Layer | What it does |
|---|---|
| **1 · Simple** | As if seeing it for the first time. An analogy. |
| **2 · Technical** | Proper terminology, correct definitions. |
| **3 · Deep** | What happens internally. |
| **4 · Mathematical** | The equations, where they exist. |
| **5 · Practical** | A small working example. |
| **6 · Production** | How professionals actually deploy it. |
| **7 · Research** | Open problems and current directions. |

> **Never skip layer 1**, and never *stop* at layer 1. The whole point of this vault is that both exist on the same page.
