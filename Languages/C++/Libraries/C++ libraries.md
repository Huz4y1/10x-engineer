---
tags: [moc, cpp, libraries]
---

# C++ libraries

Language: [[C++]] · In this stack: [[C++ for this stack]]

---

## Standard library (the STL)

| Header | Gives you |
|---|---|
| `<vector>` | Dynamic array — **the default container** |
| `<string>` | Text |
| `<map>` / `<unordered_map>` | Ordered / hash map |
| `<set>` / `<unordered_set>` | Sets |
| `<algorithm>` | `sort`, `find`, `transform`, `accumulate` |
| `<memory>` | `unique_ptr`, `shared_ptr` — **RAII** |
| `<thread>` / `<mutex>` / `<atomic>` | Concurrency |
| `<chrono>` | Time and durations |
| `<filesystem>` | Paths and files |
| `<optional>` / `<variant>` | Nullable and sum types |

> **Never write `new`/`delete` in modern C++.** `std::vector` and `std::unique_ptr` handle it. See [[Memory]].

## ML and numerical

| Library | For | Note |
|---|---|---|
| **ONNX Runtime** | Model inference — small, fast | [[C++ for this stack]] |
| **LibTorch** | PyTorch in C++ | [[C++ for this stack]] |
| **Eigen** | Linear algebra, header-only | — |
| **OpenCV** | Computer vision | [[11 — COMPUTER VISION]] |
| CUDA | GPU kernels | [[CUDA and GPU programming]] |

## Utility

| Library | For |
|---|---|
| **fmt** | String formatting (became `std::format`) |
| **spdlog** | Fast logging |
| **nlohmann/json** | JSON, header-only, pleasant |
| **Catch2** / GoogleTest | Testing |
| **pybind11** | Python bindings |

## Build

| Tool | For |
|---|---|
| **CMake** | The standard build system |
| vcpkg / Conan | Package managers |

> **Build in Release mode.** Unoptimised C++ can be *slower than Python* for numeric loops — `-O3` / `CMAKE_BUILD_TYPE=Release`. This catches everyone once ([[C++ for this stack]]).

## Related

[[C++]] · [[C++ for this stack]] · [[Memory]] · [[Pointers]] · [[33 — SYSTEMS PERFORMANCE]]
