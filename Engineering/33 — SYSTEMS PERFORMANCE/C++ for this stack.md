---
tags: [cpp, performance, systems, pytorch, onnx]
status: not-started
---

# C++ for this stack

> **What this is:** where C++ actually appears in a data/ML system, and how to write the parts you'd realistically write.
> **Why you care:** PyTorch, ONNX Runtime, Spark's engine and every database you use are C++. You don't need to write much of it — but understanding it turns those tools from black boxes into things you can reason about and extend.

Language fundamentals are in [[C++]] — pointers, memory, classes, headers. This note is what you do with them **here**.

---

## The idea in plain English

Python asks: *"what's the clearest way to express this?"*
C++ asks: *"what exactly does the machine do, and when?"*

That control is why every performance-critical layer of your stack is written in it:

| You call | It's actually |
|---|---|
| `torch.matmul(a, b)` | C++ dispatching to cuBLAS / MKL |
| `session.run(...)` (ONNX) | C++ graph executor |
| `df.groupBy().sum()` | Scala/JVM → C++ (Photon on Databricks) |
| `SELECT ... FROM ...` | C++ query engine |
| `np.dot(a, b)` | C/Fortran BLAS |

> **You are already running enormous amounts of C++.** The question this note answers is: when is it worth writing some yourself?

**The honest answer: rarely, and only after [[When to leave Python]].** But there are four real cases, below.

---

## Where C++ genuinely appears

| Case | Why C++ | Realistic? |
|---|---|---|
| **1 · Serving with LibTorch / ONNX C++** | No Python runtime, lower latency, single binary | Yes, in production teams |
| **2 · A custom PyTorch operator** | Your op doesn't exist and can't be composed | Occasionally |
| **3 · Embedded / edge inference** | No Python on the device | Yes ([[Embedded]]) |
| **4 · A Python extension for a hot loop** | Per-item overhead dominates | Yes — but **Rust is often the better choice now** ([[Rust for this stack]]) |

---

## 1. Serving a model in C++

The most realistic reason you'd write C++ here. A Python serving container is ~300MB with ONNX Runtime (or 2.5GB with PyTorch). A C++ binary is a few MB, starts instantly, and has no interpreter.

### ONNX Runtime in C++

The same `.onnx` file you exported in [[Model export and serving]]:

```cpp
#include <cstdint>
#include <onnxruntime_cxx_api.h>
#include <vector>
#include <iostream>

int main() {
    Ort::Env env(ORT_LOGGING_LEVEL_WARNING, "retail");

    Ort::SessionOptions opts;
    opts.SetIntraOpNumThreads(2);                    // match your container's CPU limit
    opts.SetGraphOptimizationLevel(GraphOptimizationLevel::ORT_ENABLE_ALL);

    Ort::Session session(env, "models/forecast_v1.onnx", opts);   // ← load ONCE

    // input must match the training contract exactly: shape, order, dtype
    std::vector<float>   input{30.0f, 5.0f, 250.0f};
    std::vector<int64_t> shape{1, 3};

    auto mem = Ort::MemoryInfo::CreateCpu(OrtArenaAllocator, OrtMemTypeDefault);
    auto tensor = Ort::Value::CreateTensor<float>(mem, input.data(), input.size(),
                                                  shape.data(), shape.size());

    const char* in_names[]  = {"features"};
    const char* out_names[] = {"prediction"};

    auto outputs = session.Run(Ort::RunOptions{nullptr},
                               in_names, &tensor, 1, out_names, 1);

    float* result = outputs[0].GetTensorMutableData<float>();
    std::cout << "prediction: " << result[0] << "\n";
}
```

```bash
g++ -std=c++17 serve.cpp -lonnxruntime -o serve
```

> **Notice the contract is identical to the Python version** — `float`, shape `{1, 3}`, features in the saved `feature_order`. The language changed; the [[Model export and serving]] rules did not. Same dtype bugs, same feature-order bugs, same silence when you get them wrong.

**The deployment payoff** ([[Docker deep dive]]):

```dockerfile
FROM gcc:13 AS builder
WORKDIR /src
COPY . .
RUN g++ -std=c++17 -O3 serve.cpp -lonnxruntime -o /serve

FROM debian:bookworm-slim
COPY --from=builder /serve /usr/local/bin/serve
COPY models/ /models/
CMD ["serve"]
```

~2.5GB (PyTorch) → ~300MB (Python + ORT) → **~80MB** (C++ + ORT). Faster cold starts, lower cost, smaller attack surface.

### LibTorch — when you need PyTorch itself

```cpp
#include <iostream>
#include <torch/script.h>

int main() {
    torch::jit::script::Module module = torch::jit::load("model_scripted.pt");
    module.eval();

    torch::NoGradGuard no_grad;                       // ← the C++ `with torch.no_grad()`
    auto input = torch::randn({1, 10});
    auto output = module.forward({input}).toTensor();

    std::cout << output.item<float>() << "\n";
}
```

Loads a **TorchScript** model ([[Model export and serving]]). Use LibTorch when you need PyTorch-specific behaviour; use ONNX Runtime when you just need inference — it's smaller and usually faster on CPU.

---

## 2. A custom PyTorch operator

When your operation genuinely can't be expressed with existing ops, and the Python version is the bottleneck.

```cpp
// custom_ops.cpp
#include <torch/extension.h>

torch::Tensor rfm_score(torch::Tensor recency, torch::Tensor frequency,
                        torch::Tensor monetary, double w_r, double w_f, double w_m) {
    TORCH_CHECK(recency.sizes() == frequency.sizes(), "shape mismatch");
    TORCH_CHECK(recency.dtype() == torch::kFloat32, "expected float32");

    return w_r * torch::exp(-recency / 90.0)
         + w_f * torch::log1p(frequency)
         + w_m * torch::log1p(monetary);
}

PYBIND11_MODULE(TORCH_EXTENSION_NAME, m) {
    m.def("rfm_score", &rfm_score, "Weighted RFM score");
}
```

```python
from torch.utils.cpp_extension import load
ops = load(name="custom_ops", sources=["custom_ops.cpp"], verbose=True)
score = ops.rfm_score(recency, frequency, monetary, 0.4, 0.3, 0.3)
```

> **`TORCH_CHECK` is your friend.** Without it, a shape or dtype mismatch is undefined behaviour — a crash if you're lucky, silently wrong numbers if you're not. In Python you'd get an exception; in C++ you get whatever was in memory.

For a **CUDA** version of a custom op, write the kernel per [[CUDA and GPU programming]] and bind it the same way. But check first — `torch.compile` fuses many operations automatically now, and it's one line.

---

## 3. The C++ that actually bites you

Coming from Python, four things cause nearly all the pain. [[Memory]] and [[Pointers]] have the fundamentals; this is what they mean in practice.

### Ownership — who frees this?

Python has a garbage collector. C++ does not. Every allocation has exactly one owner responsible for freeing it.

```cpp
#include <vector>
#include <memory>

// ✗ manual — leaks on every early return or exception
float* buf = new float[n];
if (bad) return;              // leaked
delete[] buf;

// ✓ RAII — freed automatically when it goes out of scope, exception or not
std::vector<float> buf(n);
auto model = std::make_unique<Model>();      // unique_ptr: one owner
auto shared = std::make_shared<Cache>();     // shared_ptr: reference counted
```

> **Rule: never write `new` or `delete` in modern C++.** Use `std::vector`, `std::unique_ptr`, `std::make_shared`. This is **RAII** — resource lifetime tied to scope — and it's the single most important idea in the language. It's also why modern C++ leaks far less than its reputation suggests.

### Copies you didn't ask for

```cpp
#include <vector>

void process(std::vector<float> data);          // ✗ COPIES the whole vector
void process(const std::vector<float>& data);   // ✓ passes a reference — free
```

> In Python, passing a list passes a reference. In C++, passing by value **copies**. On a million-element vector inside a loop, that's your entire performance problem. **Pass big things by `const&`.**

### Undefined behaviour

```cpp
#include <vector>

std::vector<float> v(10);
v[100] = 1.0f;          // no error. Corrupts memory. Crashes later, somewhere else.
```

> **This is the real difference from Python.** Python raises `IndexError` at the point of the mistake. C++ writes into memory that belongs to something else, and the program fails somewhere unrelated, possibly minutes later. Use `.at()` while developing (bounds-checked), and run sanitizers:

```bash
g++ -fsanitize=address,undefined -g prog.cpp -o prog && ./prog
valgrind --leak-check=full ./prog
```

### Build systems

There's no `pip`. `CMake` is the standard:

```cmake
cmake_minimum_required(VERSION 3.20)
project(retail_serve CXX)
set(CMAKE_CXX_STANDARD 17)

find_package(onnxruntime REQUIRED)
add_executable(serve serve.cpp)
target_link_libraries(serve PRIVATE onnxruntime)
target_compile_options(serve PRIVATE -O3 -march=native)
```

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j
```

> **`-O3` matters enormously.** Unoptimised C++ can be *slower than Python* for numeric loops, because the compiler hasn't vectorised anything. If your C++ benchmark is disappointing, check you built in Release mode — this catches everyone at least once.

---

## 4. C vs. C++ vs. Rust — choosing

Your [[The 10x Engineer]] ML stack line says *math engine (C)*. Here's the honest comparison:

| | C | C++ | Rust |
|---|---|---|---|
| Memory safety | ✗ Manual | ⚠️ RAII helps, still unsafe | ✅ Enforced by the compiler |
| ML ecosystem | Small | **Huge** (LibTorch, ORT, CUDA) | Growing |
| Python bindings | Manual / ctypes | pybind11 | **PyO3 — the nicest** |
| Embedded | ✅ The standard | ✅ Common | ✅ Growing ([[Embedded]]) |
| Build tooling | Make/CMake | CMake | **Cargo — vastly better** |
| Learning curve | Small language, sharp edges | **Enormous** | Steep, then pleasant |

**When to pick which:**

- **C** — embedded targets, a tiny math kernel, maximum portability, or a stable ABI for others to bind to. See [[C]].
- **C++** — you need the ML ecosystem: LibTorch, ONNX Runtime C++ API, CUDA, or an existing C++ codebase.
- **Rust** — a **new** performance component with no C++ dependency, especially data processing or a Python extension.

> **The honest recommendation for new code in this stack: use C++ when you must interface with C++ libraries (LibTorch, ONNX Runtime, CUDA), and Rust for everything else.** Rust gives you the same speed with the compiler preventing the memory bugs, and `cargo` instead of CMake. That matches your stack line — *data processing (Rust), math engine (C)* — with C++ as the bridge to the ML ecosystem.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Slower than Python | Built without optimisation | `-O3`, `CMAKE_BUILD_TYPE=Release` |
| Segfault | Out-of-bounds, or use-after-free | `-fsanitize=address`; use `.at()` |
| Crash far from the real bug | Undefined behaviour corrupted memory | ASan/UBSan; valgrind |
| Memory grows forever | Missing `delete`, or a `shared_ptr` cycle | RAII; `weak_ptr` to break cycles |
| Fast function, slow program | Copying containers into functions | Pass by `const&` |
| ONNX model gives wrong numbers | dtype or feature order | Same contract as Python ([[Model export and serving]]) |
| Linker: "undefined reference" | Library not linked | `target_link_libraries` |
| Works on your machine only | `-march=native` baked in host CPU features | Drop it for distributed binaries |
| PyTorch extension won't build | Compiler/CUDA version mismatch with the torch build | Match the toolchain torch was built with |
| Container is huge | Built in the runtime image | Multi-stage build ([[Docker deep dive]]) |

---

## Practice checklist

- [ ] Recognising how much C++ you already run
- [ ] The four realistic cases for writing it in this stack
- [ ] ONNX Runtime C++ — loading once, and the identical input contract
- [ ] LibTorch vs. ONNX Runtime, and when each fits
- [ ] Custom PyTorch operators with pybind11, and `TORCH_CHECK`
- [ ] **RAII** — never `new`/`delete`, use `vector` / `unique_ptr` / `shared_ptr`
- [ ] **Pass big things by `const&`** — copies are silent and expensive
- [ ] **Undefined behaviour**, and why C++ bugs surface far from their cause
- [ ] Sanitizers and valgrind
- [ ] CMake basics, and **`-O3` / Release builds**
- [ ] Choosing between C, C++ and Rust for a new component

## Hands-on

- [ ] Load your capstone `.onnx` in C++ and confirm it matches the Python prediction exactly
- [ ] Containerise it multi-stage and compare the image size with the Python version
- [ ] Write the RFM scoring op as a PyTorch C++ extension and call it from Python
- [ ] Write an out-of-bounds write on purpose and catch it with `-fsanitize=address`
- [ ] Benchmark a numeric loop with and without `-O3` — the gap will surprise you
- [ ] Pass a large vector by value and then by `const&`; measure both

## Resources

- [ONNX Runtime C++ API](https://onnxruntime.ai/docs/api/c/)
- [PyTorch C++ (LibTorch) docs](https://pytorch.org/cppdocs/)
- [Custom C++ and CUDA extensions](https://pytorch.org/tutorials/advanced/cpp_extension.html)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [[C++]] — language fundamentals · [[Memory]] · [[Pointers]] · [[Header files]]

## Next

[[Rust for this stack]]
