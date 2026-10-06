---
tags: [pytorch, transformers, nlp, llm]
status: not-started
---

# Transformers & LLM Basics

> **What this is:** the architecture behind ChatGPT, Claude, and essentially all modern NLP.
> **Why you care:** you don't need to derive attention from scratch. You do need to know what it computes, and how to fine-tune and serve one — because that's the actual job.

---

## The idea in plain English

To understand a sentence, you need to know **which words relate to which other words**.

> "The trophy didn't fit in the suitcase because **it** was too big."

What does "it" mean? The trophy. Change one word:

> "The trophy didn't fit in the suitcase because **it** was too small."

Now "it" means the suitcase. Same sentence structure, different answer — and you can only tell by looking at the *whole* sentence at once.

**Attention** is the mechanism that does this. For every word, it asks: *"which other words should I be looking at to understand this one?"* — and it computes a weighted blend of them.

That's the whole idea. Everything else is implementation.

### Why it replaced RNNs

Before transformers, text was processed with RNNs/LSTMs — one word at a time, left to right, carrying a memory forward.

Two fatal problems:

1. **Sequential = slow.** Word 500 can't be processed until words 1–499 are done. No parallelism, so you can't use a GPU properly.
2. **Long-range memory is weak.** Information from word 1 has to survive 499 steps of being squeezed through a fixed-size memory. It doesn't.

The transformer paper's title was "Attention Is All You Need", and the point was: drop the recurrence entirely. Let every word look at every other word **directly, in one step, all in parallel**.

That parallelism is why models with hundreds of billions of parameters became trainable. The architecture didn't just work better — it worked on GPUs.

---

## 1. Attention, concretely

Every word gets turned into three vectors:

| Vector | Analogy | Role |
|---|---|---|
| **Query (Q)** | What I'm looking for | "I'm a pronoun, I need a noun" |
| **Key (K)** | What I advertise | "I'm a noun, I'm a physical object" |
| **Value (V)** | What I actually contribute | The word's content |

The mechanism, in four steps:

1. **Score.** For each word, compare its Query against every other word's Key (a dot product). High score = "this word is relevant to me."
2. **Scale.** Divide by √(dimension). Without this, scores get huge in high dimensions and softmax saturates into a one-hot vector, killing gradients.
3. **Softmax.** Turn scores into weights that sum to 1.
4. **Blend.** Each word's new representation = the weighted sum of all the Values.

```
Attention(Q, K, V) = softmax( Q·Kᵀ / √d ) · V
```

That formula is the entire transformer. Everything else is stacking, normalising and plumbing.

**Multi-head attention** just runs this several times in parallel (say 12 "heads") with different learned projections. One head might track grammatical subject, another tracks pronoun references, another tracks tense. Then the results are concatenated. More heads = more relationship types tracked at once.

> **The cost you must know:** every word attends to every other word, so compute grows with **sequence length squared**. Double the text, quadruple the cost. This is why context windows are a big deal, why they're expensive, and why "just feed it the whole document" isn't free.

### Positional encoding

Attention has no inherent notion of order — it sees a *set* of words, not a sequence. "Dog bites man" and "man bites dog" would look identical.

So position information is **added to the embeddings** before the first layer, either as fixed sine/cosine patterns or as learned vectors. Modern models often use rotary embeddings (RoPE), which encode relative position and extrapolate better to longer sequences.

You won't implement this. Just know: **without positional encoding, a transformer can't tell word order.**

---

## 2. Tokenization and embeddings

### Tokens

Models don't see characters or words. They see **tokens** — sub-word chunks from a fixed vocabulary of ~30k–100k pieces.

```
"unbelievable"  →  ["un", "believ", "able"]
"retail"        →  ["retail"]
"Huzayl"        →  ["Hu", "zay", "l"]
```

**Why sub-words?** Whole-word vocabularies can't handle unseen words. Character-level makes sequences far too long. Sub-words are the compromise: common words are one token, rare words decompose into known pieces, and nothing is ever out-of-vocabulary.

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("distilbert-base-uncased")

out = tok("Red lamp, 6 units", padding=True, truncation=True, max_length=128, return_tensors="pt")
print(out["input_ids"])       # tensor([[101, 2417, 5001, 1010, 1020, 3197, 102]])
print(tok.tokenize("Red lamp, 6 units"))
```

`101` and `102` are `[CLS]` and `[SEP]` — special markers the model expects.

> **You must use the tokenizer that matches the model.** Every model has its own vocabulary; token ID 2417 means different things to different models. `AutoTokenizer.from_pretrained(same_name_as_model)` — always. Mismatched tokenizer and model produces fluent nonsense, with no error.

**Practical implications of tokens:**
- Pricing and context limits are counted in **tokens**, not words. Roughly **1 token ≈ 4 characters ≈ ¾ of a word** in English.
- Non-English text and code use more tokens per character.
- `truncation=True` silently cuts long inputs. If your long reviews are all being clipped at 128 tokens, that's a real accuracy problem you won't see in any error message.

### Embeddings

Each token ID becomes a vector of a few hundred numbers — its **embedding**. These are learned, and they arrange themselves so that similar meanings sit near each other.

The famous demonstration: `king - man + woman ≈ queen`. Meaning becomes geometry.

A transformer's embeddings are **contextual** — "bank" in "river bank" and "bank account" get different vectors, because attention has already blended in the surrounding words. That's the big improvement over older static embeddings like word2vec.

---

## 3. The three architectures

This is the part worth memorising, because it determines which model to reach for.

| | **Encoder-only** | **Decoder-only** | **Encoder-decoder** |
|---|---|---|---|
| Examples | BERT, RoBERTa, DistilBERT | GPT, Claude, Llama | T5, BART |
| Sees | The whole input at once, both directions | Only what came before | Encodes input, then generates |
| Trained to | Fill in masked words | Predict the next token | Map one sequence to another |
| Good at | **Understanding**: classification, NER, similarity, search | **Generating**: chat, completion, reasoning | **Transforming**: translation, summarisation |
| Size | Small (66M–350M) | Large (7B–500B+) | Medium |

**The practical decision:**

- **Classifying text?** Encoder-only. DistilBERT fine-tuned on your data is cheap, fast, runs on a laptop, and beats a prompted LLM on a narrow, well-defined task.
- **Generating text, or the task is fuzzy/varied?** Decoder-only LLM via an API.
- **Have thousands of labelled examples and a fixed task?** Fine-tune a small encoder. It'll be cheaper and better than a large model.

> **The mistake to avoid:** reaching for a large LLM because it's exciting. To classify product descriptions into 8 categories, a fine-tuned DistilBERT costs a few pence to train and runs in 5ms on CPU. An LLM API call costs money per request, adds 500ms of latency, and needs a network. Match the tool to the task.

---

## 4. Fine-tuning — the practical skill

Same idea as [[CNNs and transfer learning]]: someone pretrained on a huge corpus, you adapt it with a small labelled dataset.

### The full example

```python
from transformers import AutoTokenizer, Trainer, TrainingArguments
import numpy as np
from datasets import Dataset
from transformers import (
    AutoTokenizer, AutoModelForSequenceClassification,
    TrainingArguments, Trainer, DataCollatorWithPadding,
)
from sklearn.metrics import accuracy_score, f1_score

MODEL = "distilbert-base-uncased"
LABELS = ["Kitchen", "Lighting", "Garden", "Other"]

tokenizer = AutoTokenizer.from_pretrained(MODEL)
model = AutoModelForSequenceClassification.from_pretrained(
    MODEL,
    num_labels=len(LABELS),
    id2label={i: l for i, l in enumerate(LABELS)},     # ← makes predictions readable
    label2id={l: i for i, l in enumerate(LABELS)},
)

ds = Dataset.from_dict({"text": descriptions, "label": label_ids}).train_test_split(0.2)

def tokenize(batch):
    return tokenizer(batch["text"], truncation=True, max_length=128)

ds = ds.map(tokenize, batched=True)

def compute_metrics(eval_pred):
    logits, labels = eval_pred
    preds = np.argmax(logits, axis=-1)
    return {
        "accuracy": accuracy_score(labels, preds),
        "f1": f1_score(labels, preds, average="weighted"),
    }

args = TrainingArguments(
    output_dir="./results",
    learning_rate=2e-5,                    # ← LOW. See the warning below.
    per_device_train_batch_size=16,
    num_train_epochs=3,                    # ← 2–4 is normal. More overfits.
    eval_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
    metric_for_best_model="f1",
    fp16=True,                             # mixed precision — 2× faster on GPU
    logging_steps=50,
)

trainer = Trainer(
    model=model,
    args=args,
    train_dataset=ds["train"],
    eval_dataset=ds["test"],
    compute_metrics=compute_metrics,
    data_collator=DataCollatorWithPadding(tokenizer),    # pad per batch, not to max
)

trainer.train()
trainer.save_model("./product-classifier")
tokenizer.save_pretrained("./product-classifier")        # ← save BOTH, always
```

> **`learning_rate=2e-5`.** Normal PyTorch training uses `1e-3`. Fine-tuning a transformer uses **2e-5 to 5e-5** — roughly 50× lower. Same reason as CNN fine-tuning: the pretrained weights are already good, and a large LR destroys them in the first few steps. Use `1e-3` here and your model gets *worse* than the untrained baseline.

> **2–4 epochs.** Transformers have enormous capacity and memorise small datasets almost instantly. Ten epochs on a few thousand examples is straightforwardly overfitting.

> **Save the tokenizer with the model.** A model without its tokenizer is unusable, and it's easy to forget because `trainer.save_model()` doesn't do it.

### `DataCollatorWithPadding`

Padding every sequence to `max_length=512` when your average is 20 tokens means 96% of your compute is spent on padding. The collator pads **per batch**, to the longest item in that batch. Often a 3–5× speedup for free.

### Parameter-efficient fine-tuning (LoRA)

Full fine-tuning updates every weight. For a 7B model that needs ~80GB of GPU memory and produces a 14GB file per task.

**LoRA** freezes the original model and trains small "adapter" matrices alongside it. ~0.1% of the parameters, results close to full fine-tuning, and each adapter is a few megabytes.

```python
from peft import LoraConfig, get_peft_model

config = LoraConfig(r=8, lora_alpha=32, target_modules=["q_lin", "v_lin"], lora_dropout=0.1)
model = get_peft_model(model, config)
model.print_trainable_parameters()      # "trainable: 0.24% of all params"
```

You don't need it for DistilBERT. Know it exists, because it's how anyone fine-tunes a large model without a datacentre — and it's a common interview topic.

---

## 5. Serving it — the point of this note

A fine-tuned transformer serves exactly like any other model, through the pattern in [[FastAPI data and deployment]].

### Export to ONNX

```python
from optimum.onnxruntime import ORTModelForSequenceClassification

model = ORTModelForSequenceClassification.from_pretrained("./product-classifier", export=True)
model.save_pretrained("./product-classifier-onnx")
tokenizer.save_pretrained("./product-classifier-onnx")
```

`optimum` handles the transformer-specific export details. Doing it by hand with `torch.onnx.export` is fiddly.

### Serve it

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.concurrency import run_in_threadpool
from pydantic import BaseModel, Field
import numpy as np, onnxruntime as ort
from transformers import AutoTokenizer

state = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    state["tokenizer"] = AutoTokenizer.from_pretrained("./product-classifier-onnx")
    state["session"] = ort.InferenceSession("./product-classifier-onnx/model.onnx")
    yield
    state.clear()

app = FastAPI(lifespan=lifespan)

class ClassifyRequest(BaseModel):
    description: str = Field(..., min_length=1, max_length=500)

LABELS = ["Kitchen", "Lighting", "Garden", "Other"]

@app.post("/classify/product")
async def classify(req: ClassifyRequest):
    enc = state["tokenizer"](
        req.description, truncation=True, max_length=128,
        padding="max_length", return_tensors="np",
    )
    inputs = {
        "input_ids": enc["input_ids"].astype(np.int64),          # ← int64, not int32
        "attention_mask": enc["attention_mask"].astype(np.int64),
    }
    logits = (await run_in_threadpool(state["session"].run, None, inputs))[0]

    exp = np.exp(logits[0] - logits[0].max())                    # stable softmax
    probs = exp / exp.sum()
    idx = int(probs.argmax())
    return {"category": LABELS[idx], "confidence": float(probs[idx])}
```

Same shape as every other model endpoint: **load once in `lifespan`, validate input with Pydantic, run inference off the event loop, return a typed response.** The model type doesn't change the serving pattern at all — which is the real lesson here.

> Transformer ONNX inputs are **int64**. NumPy defaults to int32 on Windows. Cast explicitly.

---

## 6. Using an LLM API instead

Sometimes fine-tuning is the wrong call — the task is varied, you have no labels, or you need reasoning rather than classification. Then you call a hosted model.

```python
from anthropic import Anthropic

client = Anthropic()   # reads ANTHROPIC_API_KEY from the environment

message = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=256,
    system="You categorise retail products. Reply with exactly one of: Kitchen, Lighting, Garden, Other.",
    messages=[{"role": "user", "content": "WHITE HANGING HEART T-LIGHT HOLDER"}],
)
print(message.content[0].text)
```

**Fine-tune vs. API:**

| | Fine-tuned small model | LLM API |
|---|---|---|
| Cost | Train once (pennies), then free | Per request, forever |
| Latency | 5–20ms, local | 300–2000ms + network |
| Needs labels | Yes, hundreds+ | No |
| Handles new/varied tasks | No — retrain | Yes, change the prompt |
| Runs offline | Yes | No |
| Best for | One narrow, high-volume task | Varied tasks, low volume, no labels |

> **Practical guidance:** prototype with an API (fast, no labels needed). If it becomes a high-volume production path, use the API's outputs as labels to fine-tune a small model, and swap it in. That's a genuinely common and sensible pipeline.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Fine-tuning makes the model worse | LR far too high | `2e-5`, not `1e-3` |
| Validation loss rises after epoch 1 | Overfitting — too many epochs | 2–3 epochs; `load_best_model_at_end=True` |
| Predictions are fluent nonsense | Tokenizer doesn't match the model | Load both from the same name/path |
| Loaded model gives random outputs | Tokenizer wasn't saved | `tokenizer.save_pretrained()` alongside the model |
| Accuracy is fine, real inputs fail | Long inputs silently truncated | Raise `max_length`; check your token length distribution |
| CUDA out of memory | Batch or sequence length too big | Smaller batch, `gradient_accumulation_steps`, `fp16=True`, shorter `max_length` |
| Training is very slow | Padding to max length every batch | `DataCollatorWithPadding` |
| ONNX: "Unexpected input data type" | int32 vs int64 | `.astype(np.int64)` |
| Softmax overflows to `inf` | Naive `exp()` on large logits | Subtract the max first (as above) |
| Model can't tell word order | You built one without positional encoding | Use a library implementation |
| Very long documents are expensive | Attention is O(n²) | Chunk the document; use a long-context model |
| Label indices don't match names | `id2label` not set | Set `id2label`/`label2id` at load time |

---

## Practice checklist

- [ ] What attention computes, in plain terms: Query / Key / Value and the weighted blend
- [ ] Why transformers replaced RNNs — parallelism and long-range dependencies
- [ ] The O(n²) cost of attention, and what it means for context windows
- [ ] Positional encoding — and why without it there's no word order
- [ ] Tokenization: sub-words, why they exist, and matching tokenizer to model
- [ ] Embeddings, and why contextual beats static
- [ ] **Encoder vs. decoder vs. encoder-decoder — which task each suits**
- [ ] Fine-tuning via Hugging Face `transformers`, and **why the LR is ~50× lower**
- [ ] `DataCollatorWithPadding` and dynamic padding
- [ ] LoRA / PEFT — what it is and why it exists
- [ ] Exporting with `optimum` and serving through the same FastAPI pattern as any other model
- [ ] Fine-tuned small model vs. LLM API — the actual trade-off

## Hands-on

- [ ] Tokenize a few sentences and inspect the token IDs; find a word that splits into pieces
- [ ] Fine-tune DistilBERT on a text classification task (product descriptions → category works well with the retail dataset)
- [ ] Train once at `lr=1e-3` and once at `2e-5`; compare — see catastrophic forgetting directly
- [ ] Export to ONNX with `optimum` and serve one prediction through a FastAPI endpoint
- [ ] Solve the same task with an LLM API call and compare cost, latency and accuracy

## Resources

- [Andrej Karpathy: Neural Networks — Zero to Hero](https://karpathy.ai/zero-to-hero.html) — build a GPT from scratch; the best explanation there is
- [Hugging Face: Fine-tuning a pretrained model](https://huggingface.co/docs/transformers/training)
- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — read this first if the maths hasn't clicked
- [Attention Is All You Need](https://arxiv.org/abs/1706.03762) — the original paper, surprisingly readable

## Next

[[MLflow experiment tracking]]
