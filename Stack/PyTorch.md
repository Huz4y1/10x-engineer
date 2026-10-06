Everything in pytorch reduces to one loop, repeated thousands of times. These stages are feeding the data through the network, measure how wrong it was and nudge every weight a tiny bit toward being less wrong

![[pytorch-training-loop.png]]

a neural network is just a math function with adjustable numbers in it (called **weights**). Something like:

`prediction = w * x + b`  

Training means: find the values of `w` and `b` that make `prediction` match reality as closely as possible, across lots of examples. That's it. Everything else is machinery to do that search automatically.
---

The long version, with the full training loop, autograd, CNNs, transformers and deployment:

[[Tensors, autograd and the training loop]]

[[CNNs and transfer learning]]

[[Transformers and LLM basics]]

[[MLflow experiment tracking]]

[[Model export and serving]]

[[CUDA and GPU programming]] — using the GPU well

[[Data Engineering]]

---

## Installing

**PyTorch is the one library where the install command depends on your hardware.** Take the exact command from pytorch.org rather than guessing.

```bash
uv add torch torchvision                    # CPU only - works everywhere
```

**With an NVIDIA GPU:**

```bash
# Windows or Linux, CUDA 12.x - check the version at pytorch.org first
uv pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124
```

```python
import torch
print(torch.__version__)
print(torch.cuda.is_available())            # must print True to use the GPU
print(torch.cuda.get_device_name(0))
```

| | Windows | WSL 2 |
|---|---|---|
| GPU driver | Install the normal NVIDIA driver | **Install it on WINDOWS, not inside WSL** |
| CUDA toolkit | Not needed — the wheels bundle it | Not needed |
| Check | `nvidia-smi` | `nvidia-smi` works inside WSL once the Windows driver is current |

> ⚠️ **Never install an NVIDIA driver inside WSL.** WSL 2 uses the Windows driver through a passthrough layer; installing a Linux driver in the distro breaks it. If `nvidia-smi` fails in WSL, update the driver **on Windows** ([[Setting up a dev machine]]).

> ⚠️ **`torch.cuda.is_available()` returning `False` almost always means you installed the CPU build.** `uv add torch` on its own gives you CPU-only. Reinstall with the CUDA index URL above.

> **A GPU is optional for learning.** Everything in [[Tensors, autograd and the training loop]] runs on CPU — just slower. Write `device = "cuda" if torch.cuda.is_available() else "cpu"` and the same code works on both.

> **macOS:** `uv add torch` then `device = "mps"` for Apple Silicon acceleration.
