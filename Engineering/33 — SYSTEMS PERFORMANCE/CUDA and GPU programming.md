---
tags: [cuda, gpu, performance, pytorch, systems]
status: not-started
---

# CUDA and GPU programming

> **What this is:** what a GPU actually is, why deep learning needs one, and how to write code that runs on it.
> **Why you care:** you're already using CUDA every time you call `.to("cuda")` in PyTorch. Understanding the machine underneath is what turns "my training is slow" from a mystery into a diagnosis.

---

## The idea in plain English

A **CPU** is a handful of very clever workers. Each one is fast, handles complicated instructions, predicts branches, and can do completely different jobs. Great for running an operating system, where every task is different.

A **GPU** is ten thousand simple workers who must all do **the same thing at the same time** to different data. Useless for running an operating system. Perfect for "multiply these ten million numbers by that matrix."

```
CPU                              GPU
┌───┬───┬───┬───┐               ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│ C │ C │ C │ C │               │·│·│·│·│·│·│·│·│·│·│·│·│
├───┼───┼───┼───┤               │·│·│·│·│·│·│·│·│·│·│·│·│   ~10,000 tiny cores
│ C │ C │ C │ C │               │·│·│·│·│·│·│·│·│·│·│·│·│
└───┴───┴───┴───┘               └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
8–64 complex cores               Simple, but there are thousands
Low latency per task             Enormous total throughput
```

**Why deep learning cares:** a neural network layer is a matrix multiply. Multiplying a 1024×1024 matrix means a million independent multiply-adds, and none of them depends on any other. That is *exactly* the shape of work a GPU is built for. Hence 10–100× speedups on training.

> **The rule that decides everything: GPUs win when you have thousands of independent, identical operations.** If your work is sequential, branchy, or small, a GPU is *slower* than a CPU — because you pay to ship the data over and get nothing back for it.

---

## 1. The execution model

Three levels, and the vocabulary is worth learning because every error message uses it.

```
GRID  ────────────────────────────────────  one kernel launch
 ├── Block 0   ├── Block 1   ├── Block 2     blocks run independently,
 │   ┌──────┐  │   ┌──────┐  │   ┌──────┐    in any order, on any SM
 │   │thread│  │   │thread│  │   │thread│
 │   │thread│  │   │thread│  │   │thread│    threads in a block can
 │   │ ...  │  │   │ ...  │  │   │ ...  │    share memory and sync
 │   └──────┘  │   └──────┘  │   └──────┘
```

| Term | What it is |
|---|---|
| **Kernel** | A function that runs on the GPU |
| **Thread** | One instance of the kernel, working on one element |
| **Block** | A group of threads (up to 1024) that can share memory and synchronise |
| **Grid** | All the blocks in one launch |
| **Warp** | **32 threads that execute in lockstep** — the real unit of execution |
| **SM** | Streaming Multiprocessor — a physical cluster of cores that runs blocks |

### The warp is the thing that actually matters

Threads don't run individually. They run in **warps of 32, executing the same instruction at the same time**.

So if threads in a warp take different branches:

```cuda
if (threadIdx.x % 2 == 0) {
    do_expensive_thing_A();     // 16 threads do this while 16 idle
} else {
    do_expensive_thing_B();     // then the other 16, while the first 16 idle
}
```

The warp executes **both** branches, masking off the inactive threads each time. You've just halved your throughput. This is **warp divergence**.

> **Rule of thumb: keep threads within a warp on the same path.** Branch on block-level conditions, not per-thread ones, wherever you can. This is the single most common reason a hand-written kernel underperforms.

---

## 2. Memory — where the real performance lives

**The uncomfortable truth: most GPU code is memory-bound, not compute-bound.** Modern GPUs can do arithmetic far faster than they can fetch operands. Optimising CUDA is mostly optimising memory access.

| Memory | Size | Speed | Scope |
|---|---|---|---|
| **Registers** | ~64K/SM | Fastest | One thread |
| **Shared memory** | ~48–164KB/SM | ~100× global | One block |
| **L2 cache** | MBs | Fast | All |
| **Global (VRAM)** | 8–80GB | ~500× slower than registers | All |
| **Host (system RAM)** | Your RAM | **Glacial** — over PCIe | CPU |

### Coalescing — the concept to take away

When a warp reads global memory, the hardware fetches whole 128-byte chunks. If your 32 threads read 32 *consecutive* floats, that's **one** fetch. If they read 32 *scattered* floats, that's up to **32** fetches.

```cuda
float v = data[blockIdx.x * blockDim.x + threadIdx.x];   // ✓ consecutive: 1 transaction
float v = data[threadIdx.x * stride];                     // ✗ strided: up to 32
```

> **Same amount of data, up to 32× the time.** This is why array layout matters so much on GPUs, and why frameworks care about tensors being *contiguous* — `tensor.contiguous()` in PyTorch exists for exactly this reason.

### The PCIe wall

Moving data between CPU and GPU is slow — roughly 16–32 GB/s, versus ~1000 GB/s inside the GPU.

> **The most common beginner GPU mistake: copying data back and forth in a loop.** Move data to the GPU **once**, do all the work there, move the result back **once**. A `.item()` or `.cpu()` inside your training loop forces a synchronisation and a transfer every iteration, and it can make GPU training slower than CPU.

---

## 3. Writing a kernel

You will rarely write raw CUDA. Do it once so the abstractions stop being magic.

```cuda
// vector_add.cu
__global__ void vector_add(const float* a, const float* b, float* out, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;   // this thread's global index
    if (i < n) {                                     // ← guard: n is rarely a multiple of blockDim
        out[i] = a[i] + b[i];
    }
}

int main() {
    int n = 1 << 20;                                  // ~1M elements
    size_t bytes = n * sizeof(float);

    float *d_a, *d_b, *d_out;
    cudaMalloc(&d_a, bytes);
    cudaMalloc(&d_b, bytes);
    cudaMalloc(&d_out, bytes);

    cudaMemcpy(d_a, h_a, bytes, cudaMemcpyHostToDevice);
    cudaMemcpy(d_b, h_b, bytes, cudaMemcpyHostToDevice);

    int threads = 256;
    int blocks  = (n + threads - 1) / threads;        // ← ceiling division
    vector_add<<<blocks, threads>>>(d_a, d_b, d_out, n);

    cudaMemcpy(h_out, d_out, bytes, cudaMemcpyDeviceToHost);
    cudaFree(d_a); cudaFree(d_b); cudaFree(d_out);
}
```

```bash
nvcc vector_add.cu -o vector_add && ./vector_add
```

**The four things to notice:**

1. **`__global__`** = runs on the GPU, called from the CPU.
2. **`blockIdx.x * blockDim.x + threadIdx.x`** is *the* CUDA idiom. Every kernel starts with it. It converts "which thread am I" into "which element do I own".
3. **The `if (i < n)` guard.** You launch a whole number of blocks, so you almost always launch more threads than elements. Without the guard you write past the end of the array — and the failure is a corrupted result, not a crash.
4. **`<<<blocks, threads>>>`** is the launch configuration. 256 threads per block is a good default (a multiple of the 32-wide warp).

> **Kernel launches are asynchronous.** The CPU returns immediately. If you time a kernel without `cudaDeviceSynchronize()`, you'll measure the launch, not the work — and conclude it's infinitely fast.

### Errors are silent by default

```cuda
kernel<<<b, t>>>(...);
cudaError_t err = cudaGetLastError();
if (err != cudaSuccess) fprintf(stderr, "launch failed: %s\n", cudaGetErrorString(err));
cudaDeviceSynchronize();
```

> **CUDA does not throw.** It sets an error code you have to check. An unchecked kernel that failed to launch produces garbage output and no message. Check after every launch while developing, and run `compute-sanitizer ./your_program` to catch out-of-bounds writes.

---

## 4. The realistic path: don't write kernels

For 95% of ML work, use the layers above:

| Layer | What it is | Use when |
|---|---|---|
| **PyTorch** | Already CUDA underneath | ✅ Almost always |
| **CuPy** | numpy with a GPU backend | Array maths outside PyTorch |
| **cuDF / RAPIDS** | pandas on the GPU | Big dataframe operations |
| **Triton** | Write kernels in **Python** | Custom fused ops, without CUDA C++ |
| **`torch.compile`** | Fuses ops automatically | ✅ Free speedup, one line |
| **Raw CUDA C++** | Maximum control | A genuinely novel operator |

```python
import cupy as cp
a = cp.random.randn(10_000_000)
b = (a * 2 + 1).sum()          # runs on the GPU; numpy syntax
```

```python
import torch

model = torch.compile(model)   # ← often 1.3–2× for one line. Try this first.
```

**Triton** is the sweet spot when you do need a custom kernel — Python syntax, handles a lot of the memory hierarchy for you, and it's what PyTorch itself uses to generate kernels:

```python
import triton, triton.language as tl

@triton.jit
def add_kernel(x_ptr, y_ptr, out_ptr, n, BLOCK: tl.constexpr):
    pid = tl.program_id(0)
    offs = pid * BLOCK + tl.arange(0, BLOCK)
    mask = offs < n                                  # same guard idea as CUDA
    tl.store(out_ptr + offs, tl.load(x_ptr + offs, mask=mask)
                            + tl.load(y_ptr + offs, mask=mask), mask=mask)
```

---

## 5. Using the GPU well in PyTorch

This is where the knowledge pays off day to day ([[Tensors, autograd and the training loop]]).

```python
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)

scaler = torch.amp.GradScaler("cuda")

for X, y in loader:
    X, y = X.to(device, non_blocking=True), y.to(device, non_blocking=True)

    optimizer.zero_grad(set_to_none=True)            # slightly faster than zeroing

    with torch.amp.autocast("cuda", dtype=torch.bfloat16):   # mixed precision
        loss = criterion(model(X), y)

    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

### Mixed precision — the biggest easy win

Doing maths in 16-bit instead of 32-bit halves memory traffic and uses the **Tensor Cores** (dedicated matrix-multiply hardware). Typically **2–3× faster training** with no accuracy loss.

> Prefer **`bfloat16`** over `float16` on Ampere (RTX 30xx / A100) and newer — it has the same exponent range as float32, so it doesn't overflow, and you can often drop the `GradScaler` entirely. `float16` needs the scaler to stop gradients underflowing to zero.

### The checklist for "why is my GPU slow?"

```bash
watch -n 0.5 nvidia-smi        # is utilisation actually high?
```

| Utilisation | Meaning | Fix |
|---|---|---|
| **< 30%** | **Starved.** The GPU is waiting for data. | `num_workers=8`, `pin_memory=True`, bigger batch |
| Spiky | Sync points in the loop | Remove `.item()`, `.cpu()`, `print()` from the inner loop |
| ~100%, still slow | Genuinely compute-bound | Mixed precision, `torch.compile`, a bigger batch |

> **The most common cause of a slow GPU is a slow data loader.** The GPU finishes in 5ms and waits 50ms for the next batch. It's a CPU/disk problem wearing a GPU costume — profile before buying a bigger card.

```python
import torch

with torch.profiler.profile(activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA]) as prof:
    train_one_epoch()
print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=15))
```

### Out of memory

```python
import torch

torch.cuda.memory_allocated() / 1e9      # GB in use
torch.cuda.empty_cache()                  # release cached blocks
```

| Fix | Cost |
|---|---|
| Smaller batch size | Fewer samples per step |
| Gradient accumulation | Same effective batch, more steps |
| Mixed precision | ~half the activation memory |
| `torch.utils.checkpoint` | Recompute activations — slower, much less memory |
| `del` intermediates + `empty_cache()` | Housekeeping |

> **The classic OOM cause is accumulating loss tensors** — `total += loss` keeps the whole computation graph alive for every batch. `total += loss.item()`. Covered in [[Tensors, autograd and the training loop]] and it bites everyone once.

---

## 6. GPUs in production

### Do you need one for inference?

**Usually not.** A small tabular model or DistilBERT runs in milliseconds on CPU. GPUs earn their keep for inference when you're serving large models or high-throughput batches.

| Workload | Hardware |
|---|---|
| Tabular model, ONNX | **CPU** ← the capstone |
| DistilBERT, moderate volume | **CPU** is fine |
| Large model, low latency | GPU |
| Big batch scoring | GPU, or CPU overnight |

> Check the cost honestly: a GPU node is roughly 10× a CPU node. If CPU inference meets your latency budget, that's a 90% saving for no downside ([[Deployment patterns]]).

### GPU containers

```dockerfile
FROM nvidia/cuda:12.4-runtime-ubuntu22.04
# runtime, not devel — devel includes the full toolchain and is ~3GB bigger
RUN apt-get update && apt-get install -y python3-pip && rm -rf /var/lib/apt/lists/*
```

```bash
docker run --gpus all my-training-image nvidia-smi
```

The **host** provides the driver; the **container** provides the CUDA runtime. So the container's CUDA version must be **≤** the host driver's supported version. On Kubernetes this is `nvidia.com/gpu: 1` ([[Kubernetes and AKS]]).

### On Azure

| Series | GPU | Use |
|---|---|---|
| **NC T4 v3** | T4 (16GB) | Inference, small training. **Cheapest — start here.** |
| NC A100 v4 | A100 (40/80GB) | Serious training |
| ND H100 v5 | H100 | Large-scale training |

Available as Databricks GPU clusters, Azure ML compute, or plain VMs.

> **💸 GPU nodes are expensive and bill by the hour whether you're using them or not.** Set auto-terminate aggressively ([[Databricks and Delta Lake]]), scale GPU node pools to zero, and never leave a GPU cluster running overnight by accident. This is the single most expensive mistake available to you in this whole stack.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `torch.cuda.is_available()` is False | Driver/toolkit mismatch, or a CPU-only PyTorch build | `nvidia-smi`; reinstall PyTorch with the right CUDA index URL |
| GPU utilisation < 30% | Data loader starving it | `num_workers`, `pin_memory`, bigger batch |
| Training slower than CPU | Tiny model/batch — transfer dominates | Bigger batch, or stay on CPU |
| CUDA OOM | Batch too big; accumulating graphs | Smaller batch; `.item()`; mixed precision |
| OOM only after several epochs | Memory leak from retained tensors | `.item()` / `.detach()` when logging |
| Kernel "works" but output is wrong | Missing `if (i < n)` guard, or a race | `compute-sanitizer`; check bounds |
| No error but garbage results | CUDA errors aren't raised | `cudaGetLastError()` after every launch |
| Kernel appears instant | Launches are async | `cudaDeviceSynchronize()` before timing |
| Custom kernel slower than PyTorch | You're competing with cuBLAS | Don't hand-write matmul; use Triton or built-ins |
| `nvidia-smi` fails in a container | NVIDIA Container Toolkit missing, or no `--gpus all` | Install it; add the flag |
| Loss is NaN with float16 | Gradient underflow | `GradScaler`, or use `bfloat16` |
| Huge cloud bill | GPU cluster left running | Auto-terminate; scale to zero |

---

## Practice checklist

- [ ] CPU vs. GPU architecture, and the "thousands of identical independent operations" rule
- [ ] Grid / block / thread / **warp** / SM
- [ ] **Warp divergence** and keeping a warp on one path
- [ ] The memory hierarchy, and that most kernels are **memory-bound**
- [ ] **Coalesced access**, and why contiguity matters
- [ ] The PCIe wall — move data once
- [ ] Writing a kernel: `__global__`, the index idiom, the `if (i < n)` guard, `<<<blocks, threads>>>`
- [ ] **CUDA errors are silent** — check them; use `compute-sanitizer`
- [ ] Async launches and synchronising before timing
- [ ] The layers above CUDA: PyTorch, CuPy, cuDF, **Triton**, `torch.compile`
- [ ] **Mixed precision**, and `bfloat16` vs `float16`
- [ ] Diagnosing low GPU utilisation — usually the data loader
- [ ] OOM fixes, and the accumulating-loss leak
- [ ] **Whether you need a GPU for inference at all** — usually not
- [ ] GPU containers, host driver vs. container runtime
- [ ] Azure GPU SKUs and the cost discipline

## Hands-on

- [ ] Run `nvidia-smi` and identify your GPU, VRAM and driver version
- [ ] Write, compile and run the `vector_add` kernel — remove the `if (i < n)` guard and see what happens
- [ ] Time a kernel without `cudaDeviceSynchronize()`, then with it
- [ ] Train a CNN on CPU and GPU; compare epoch times
- [ ] Add mixed precision and measure the speedup and memory drop
- [ ] Add `torch.compile(model)` and measure
- [ ] Watch `nvidia-smi` during training; drop `num_workers` to 0 and watch utilisation collapse
- [ ] Add `.item()` inside the inner loop deliberately and measure the damage
- [ ] Cause an OOM, then fix it three different ways

## Resources

- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [PyTorch: CUDA semantics](https://pytorch.org/docs/stable/notes/cuda.html)
- [Triton tutorials](https://triton-lang.org/main/getting-started/tutorials/index.html)
- [PyTorch performance tuning guide](https://pytorch.org/tutorials/recipes/recipes/tuning_guide.html)

## Next

[[C++ for this stack]]
