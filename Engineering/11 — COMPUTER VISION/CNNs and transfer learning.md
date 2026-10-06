---
tags: [pytorch, cnn, computer-vision, transfer-learning]
status: not-started
---

# CNNs & Transfer Learning

> **What this is:** how neural networks handle images, and why you almost never train one from scratch.
> **Why you care:** transfer learning is the single highest-leverage technique in applied deep learning. It's the difference between needing a million images and needing two hundred.

---

## Part 1 — Why images need something different

### The problem with a plain network

A 224×224 colour image is 224 × 224 × 3 = **150,528 numbers**.

Feed that into `nn.Linear(150528, 1000)` and that one layer has **150 million parameters**. It would be enormous, slow, need vast amounts of data — and it would still be bad, for two reasons:

1. **It throws away the structure.** Flattening destroys the fact that pixel (10,10) is next to pixel (10,11). The network has to learn "these numbers are neighbours" from scratch.
2. **It doesn't generalise position.** Learn to spot a cat in the top-left, and it knows nothing about cats in the bottom-right. Every position must be learned separately.

### The convolution

A **convolution** slides a small window (a **kernel**, typically 3×3) across the image, doing the same small calculation everywhere.

```
Image                    Kernel (3×3)        Output
┌─────────────┐          ┌───────┐          ┌───────────┐
│ 5 3 2 1 ... │          │-1 0 1 │          │ ...       │
│ 4 8 6 2 ... │    ⊛     │-2 0 2 │    =     │ (edges)   │
│ 1 2 9 4 ... │          │-1 0 1 │          │ ...       │
└─────────────┘          └───────┘          └───────────┘
```

That specific kernel detects vertical edges — it produces a big number where the left side is dark and the right is bright. In a CNN, the kernel's nine numbers are **learned**, not designed.

**Two properties follow, and they're the entire point:**

- **Parameter sharing.** One 3×3 kernel is 9 numbers (+1 bias) used across the whole image. Compare that to 150 million.
- **Translation invariance.** The same kernel runs everywhere, so a feature learned in one place is detected in every place. Learn "edge" once, use it everywhere.

### Depth builds a hierarchy

This is the beautiful bit, and it emerges without being asked for:

| Layer | Learns to detect |
|---|---|
| 1 | Edges, colour blobs |
| 2 | Corners, simple textures |
| 3 | Repeating patterns, basic shapes |
| 5 | Object parts — eyes, wheels, handles |
| 8+ | Whole objects |

Each layer combines the layer below. Edges combine into corners, corners into shapes, shapes into parts, parts into objects.

> **This hierarchy is why transfer learning works.** "Edge detector" is useful for *every* image task. So is "texture detector". Only the last layer or two is specific to "is this a cat". That's the insight this whole note rests on.

### Pooling

**Pooling** shrinks the image between convolutions, usually by taking the maximum of each 2×2 block.

```python
from torch import nn

nn.MaxPool2d(kernel_size=2, stride=2)      # halves height and width
```

It reduces computation, and it makes the network tolerant of small shifts — a feature moved by one pixel still lands in the same pooled cell.

### The shapes

Image tensors are `(batch, channels, height, width)` — **channels before height/width** in PyTorch. (TensorFlow puts channels last. This bites people who move between them.)

```
(32, 3, 224, 224)    32 RGB images, 224×224
(32, 64, 112, 112)   after a conv producing 64 feature maps, and one pool
```

`nn.Conv2d(in_channels, out_channels, kernel_size)`:
- `in_channels` — channels coming in (3 for RGB, or whatever the previous layer output)
- `out_channels` — how many different kernels to learn
- `padding=1` with `kernel_size=3` keeps height and width unchanged

**Output size:** `out = (in + 2*padding - kernel_size) / stride + 1`

You don't need to compute this by hand — just print shapes as you go, or use `nn.LazyLinear` which infers its input size on the first forward pass.

---

## Part 2 — A CNN from scratch

```python
import torch.nn as nn

class SmallCNN(nn.Module):
    def __init__(self, n_classes: int = 10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 32, kernel_size=3, padding=1),   # (B,3,64,64) → (B,32,64,64)
            nn.BatchNorm2d(32),
            nn.ReLU(),
            nn.MaxPool2d(2),                              # → (B,32,32,32)

            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.MaxPool2d(2),                              # → (B,64,16,16)

            nn.Conv2d(64, 128, kernel_size=3, padding=1),
            nn.BatchNorm2d(128),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d(1),                      # → (B,128,1,1) — any input size
        )
        self.classifier = nn.Sequential(
            nn.Flatten(),                                 # → (B,128)
            nn.Dropout(0.3),
            nn.Linear(128, n_classes),                    # raw logits — no Softmax
        )

    def forward(self, x):
        return self.classifier(self.features(x))
```

The standard block is **Conv → BatchNorm → ReLU → Pool**, repeated, with channels doubling and spatial size halving each time.

> **`AdaptiveAvgPool2d(1)`** collapses each feature map to a single number, so the network works for *any* input image size. Without it you'd have to hardcode the flattened size, and changing the input resolution would break everything. Use it.

Training is identical to [[Tensors, autograd and the training loop]] — same six lines. Only the model and the data differ.

---

## Part 3 — Transfer learning

### The idea

Someone trained a network on **ImageNet**: 1.2 million images, 1,000 categories, weeks of GPU time. Its early layers learned excellent, general-purpose edge/texture/shape detectors.

Those detectors are useful for *your* problem too. So: take their network, keep the learned features, and replace only the final layer with one for your categories.

**Result:** state-of-the-art-ish performance from a few hundred images and ten minutes of training.

> **Rule of thumb:** unless you have a genuinely novel image type (medical scans, satellite imagery, microscopy) and hundreds of thousands of labelled examples, **start with a pretrained model. Always.** Training from scratch is for learning, or for research. In practice it's almost always the wrong call.

### Loading a pretrained model

```python
import torch.nn as nn
from torchvision import models

model = models.resnet18(weights=models.ResNet18_Weights.DEFAULT)
```

> `weights=...` is the modern API. `pretrained=True` is deprecated — you'll see it in every older tutorial.

### Strategy 1 — Feature extraction (freeze everything but the head)

Best when you have **little data** (a few hundred images) or your images look like ImageNet.

```python
from torch import nn
from torchvision import models
import torch

model = models.resnet18(weights=models.ResNet18_Weights.DEFAULT)

for param in model.parameters():
    param.requires_grad = False                  # freeze everything

n_features = model.fc.in_features                # 512 for resnet18
model.fc = nn.Linear(n_features, n_classes)      # new layer — requires_grad=True by default

optimizer = torch.optim.Adam(
    filter(lambda p: p.requires_grad, model.parameters()),   # only the new layer
    lr=1e-3,
)
```

Replacing `model.fc` automatically gives you a fresh layer that *is* trainable, so only it learns. Training is fast — most of the network is just doing a forward pass.

> **The head's name varies by architecture:** ResNet uses `model.fc`, EfficientNet and VGG use `model.classifier`, ViT uses `model.heads`. `print(model)` and look at the last block.

### Strategy 2 — Fine-tuning (train everything, gently)

Best when you have **more data** (thousands of images) or your domain differs from ImageNet.

```python
from torch import nn
from torchvision import models
import torch

model = models.resnet18(weights=models.ResNet18_Weights.DEFAULT)
model.fc = nn.Linear(model.fc.in_features, n_classes)

optimizer = torch.optim.Adam(model.parameters(), lr=1e-4)   # ← 10× lower than usual
```

> **The low learning rate is the whole trick.** Those pretrained weights are already good. A normal learning rate would smash them apart in the first few batches — "catastrophic forgetting" — and you'd be worse off than training from scratch. **`1e-4` or lower when fine-tuning.**

### Strategy 3 — Discriminative learning rates (best of both)

Early layers (general features) barely change; later layers (specific features) change more:

```python
import torch

optimizer = torch.optim.Adam([
    {"params": model.layer1.parameters(), "lr": 1e-5},
    {"params": model.layer2.parameters(), "lr": 1e-5},
    {"params": model.layer3.parameters(), "lr": 1e-4},
    {"params": model.layer4.parameters(), "lr": 1e-4},
    {"params": model.fc.parameters(),     "lr": 1e-3},
])
```

Or in two phases: freeze and train the head to convergence, then unfreeze everything at a low LR. That's fast.ai's standard recipe and it works well.

### Choosing a backbone

| Model | Params | Notes |
|---|---|---|
| `resnet18` | 11M | **Start here.** Fast, reliable, well understood. |
| `resnet50` | 25M | Better accuracy, ~2× slower |
| `efficientnet_b0` | 5M | Excellent accuracy per parameter. Good for deployment. |
| `mobilenet_v3_small` | 2.5M | For phones and edge devices |
| `vit_b_16` | 86M | Vision Transformer. Great with lots of data, weaker on small datasets. |

> **Start with `resnet18` and only change if the results demand it.** Choosing an exotic backbone before you have a working baseline is a way to spend a day and learn nothing.

### The normalisation you must copy

Pretrained models expect input normalised with **ImageNet's** statistics. Get this wrong and accuracy quietly collapses:

```python
from torchvision import transforms

IMAGENET_MEAN = [0.485, 0.456, 0.406]
IMAGENET_STD  = [0.229, 0.224, 0.225]

eval_tf = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),                                  # → [0,1], (C,H,W)
    transforms.Normalize(IMAGENET_MEAN, IMAGENET_STD),
])
```

Or let torchvision hand you the exact transform the weights were trained with — safer:

```python
from torchvision import models

weights = models.ResNet18_Weights.DEFAULT
model = models.resnet18(weights=weights)
preprocess = weights.transforms()          # ✓ guaranteed to match
```

---

## Part 4 — Data augmentation

### The idea

You have 500 images. The model will memorise them. **Augmentation** creates variations on the fly — flipped, rotated, cropped, colour-shifted — so it sees a slightly different image every epoch and has to learn the *concept* rather than the pixels.

```python
from torchvision import transforms

train_tf = transforms.Compose([
    transforms.RandomResizedCrop(224, scale=(0.7, 1.0)),
    transforms.RandomHorizontalFlip(),
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2),
    transforms.RandomRotation(15),
    transforms.ToTensor(),
    transforms.Normalize(IMAGENET_MEAN, IMAGENET_STD),
])

val_tf = transforms.Compose([                    # ← NO augmentation
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(IMAGENET_MEAN, IMAGENET_STD),
])
```

> **Two rules:**
> 1. **Augment training only.** Randomising validation makes your metric noisy and meaningless.
> 2. **Augmentations must preserve the label.** `RandomHorizontalFlip` is fine for cats. It's *wrong* for digits (a flipped "2" isn't a 2) and for anything where left/right matters. Think about your specific data — the default recipe isn't universal.

Less data → more augmentation. It's the cheapest regularisation there is: no extra labelling, no bigger model.

---

## Part 5 — Overfitting

### Spotting it

Plot train and validation loss together ([[Tensors, autograd and the training loop]]):

```
loss
 │╲
 │ ╲___________  val    ← turns UP: overfitting starts here
 │  ╲    ___/
 │   ╲__/
 │    ╲_________ train  ← keeps falling
 └──────────────── epoch
```

The moment validation loss turns up while training loss keeps falling, the model has stopped learning the pattern and started memorising the examples.

### Fixing it, cheapest first

| Fix | Cost | Notes |
|---|---|---|
| **Early stopping** | Free | Stop at the minimum. Do this always. |
| **More augmentation** | Free | Usually the biggest win on small image datasets |
| **More dropout** | Free | Bump 0.2 → 0.5 in the classifier head |
| **Weight decay** | Free | `Adam(..., weight_decay=1e-4)` |
| **Freeze more layers** | Free | Fewer trainable parameters to overfit with |
| **Smaller model** | Free | `resnet50` → `resnet18` |
| **More data** | Expensive | Always the best answer if you can get it |

> **Start with early stopping and augmentation.** They're free and they cover most cases. Reach for architecture changes last.

### Underfitting — the opposite

Both losses high and flat means the model isn't learning at all. Train longer, raise the learning rate, unfreeze more layers, use a bigger backbone, or check your data pipeline is actually delivering what you think.

---

## Part 6 — Where this fits in the capstone

The capstone models are **tabular**, not image — a feed-forward net and an LSTM ([[Capstone build guide]]). So why learn CNNs?

1. **Every job with "deep learning" in the description assumes them.** They're the standard interview topic.
2. **The transfer learning idea is the transferable part.** The exact same "take a pretrained model, replace the head, fine-tune at a low LR" pattern is what you'll do with transformers in [[Transformers and LLM basics]] — which *is* directly relevant.
3. **The training discipline is identical** — augmentation, early stopping, reading loss curves, spotting overfitting.

Treat this as one focused week. Build both models, compare them, understand *why* the pretrained one wins, and move on.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Pretrained model does badly | Wrong normalisation | Use `weights.transforms()` |
| Fine-tuning makes it worse than the frozen version | LR too high — catastrophic forgetting | Drop to `1e-4` or lower |
| Shape error at the first `Linear` | Flattened size doesn't match | `AdaptiveAvgPool2d(1)` + `nn.Flatten()`, or `nn.LazyLinear` |
| `Expected 4-dimensional input, got 3` | Missing batch dimension on a single image | `x.unsqueeze(0)` |
| `Expected input[32,224,224,3]` | Channels-last (TensorFlow order) | `x.permute(0, 3, 1, 2)`; `ToTensor()` does this for you |
| Training loss won't move at all | Everything frozen, including the new head | Confirm `model.fc.weight.requires_grad` is `True` |
| Validation accuracy stuck at chance | Labels misaligned, or LR far too high | Check a batch by eye; overfit one batch first |
| GPU out of memory | Batch too big for image size | Halve `batch_size`; try 224 instead of 384 |
| Validation metric jumps around | Augmentation applied to validation | Separate transform, no randomness |
| Model great in training, bad on new photos | Real photos differ from your training set | Augment harder; collect more representative data |
| Very slow training on GPU | Data loading is the bottleneck | `num_workers=4`, `pin_memory=True` |
| Accuracy high but the model is useless | Class imbalance — 95% one class | Look at a confusion matrix, not accuracy. Use class weights. |

---

## Practice checklist

- [ ] Why a plain feed-forward net is wrong for images — parameter count and lost structure
- [ ] Convolutions, kernels, parameter sharing, translation invariance
- [ ] Pooling and what it buys you
- [ ] The feature hierarchy — and why it makes transfer learning work
- [ ] Image tensor shapes: `(B, C, H, W)`, channels-first in PyTorch
- [ ] `torchvision.models` — pretrained backbones and the `weights=` API
- [ ] Freezing layers vs. fine-tuning the whole network, and **why fine-tuning needs a 10× lower LR**
- [ ] Finding and replacing the classifier head for a given architecture
- [ ] Matching the pretrained model's normalisation exactly
- [ ] Data augmentation, why it matters more with less data, and why it's training-only
- [ ] Overfitting signals: train/val loss divergence, and the cheap fixes first

## Hands-on

- [ ] Train a small CNN from scratch on a toy image dataset (CIFAR-10 or your own photos)
- [ ] Fine-tune a pretrained `resnet18` on the same dataset; compare accuracy **and** training time
- [ ] Try feature extraction (frozen) vs. full fine-tuning; note which wins at your data size
- [ ] Fine-tune at `lr=1e-2` on purpose and watch catastrophic forgetting happen
- [ ] Train with and without augmentation; plot both loss curves side by side
- [ ] Print the shape after every layer in your CNN until you can predict them

## Resources

- [torchvision models docs](https://pytorch.org/vision/stable/models.html)
- [PyTorch: Transfer Learning tutorial](https://pytorch.org/tutorials/beginner/transfer_learning_tutorial.html)
- [fast.ai: Practical Deep Learning](https://course.fast.ai/) — best practical course on this material
- [CS231n: Convolutional Neural Networks](https://cs231n.github.io/convolutional-networks/) — the clearest written explanation of convolutions

## Next

[[Transformers and LLM basics]]
