---
tags: [cpp, pointers, memory, language]
---

# Pointers

Language: [[C++]] · Memory model: [[Memory]] · Libraries: [[C++ libraries]]

Use pointers when sharing data across functions and the data inside the variable needs to manipulated

```cpp
#include <iostream>

int age = 20;

int *age_ptr = &age;

std::cout << *age_ptr; //Outputs the value at the memory address which is 20
std::cout << age;   //Outputs 20
std::cout << age_ptr; //Outputs the memory address of where 20 is stored
```

---

## The two symbols

They're confusing because each does two different jobs depending on where it appears.

| Symbol | In a declaration | In an expression |
|---|---|---|
| `*` | "this is a pointer" — `int *p;` | **Dereference** — `*p` is the value |
| `&` | "this is a reference" — `int &r = x;` | **Address-of** — `&x` is where x lives |

```cpp
#include <iostream>

int age = 20;
int *p = &age;    // declaration: p is a pointer;  &age: the address of age
*p = 21;          // expression:  write THROUGH the pointer
std::cout << age; // 21 - we changed the original
```

> **Read declarations right to left.** `int *p` → "p is a pointer to int". `const int *p` → "p points to a const int" (can't change the value). `int * const p` → "p is a const pointer" (can't change where it points).

---

## References — usually what you actually want

```cpp
#include <iostream>

void birthday(int &age) { age += 1; }     // reference parameter

int a = 20;
birthday(a);          // no & at the call site
std::cout << a;       // 21
```

| | Pointer | Reference |
|---|---|---|
| Can be null | ✅ | ❌ |
| Can be reseated | ✅ | ❌ — bound once, forever |
| Syntax to use | `*p` | `r`, like a normal variable |
| Can do arithmetic | ✅ | ❌ |

> **Prefer references for parameters.** They can't be null, so you don't have to check. Use a pointer only when "no value" is a legitimate state.

```cpp
#include <string>

void print(const std::string &s);   // large object, read-only: pass by const reference
void tweak(std::string &s);         // needs to modify the caller's object
void take(std::string s);           // makes a COPY - only if you want one
```

> ⚠️ **Passing a big object by value copies the whole thing.** `const T&` for anything larger than a pointer that you're only reading — this is one of the easiest performance wins in C++ ([[33 — SYSTEMS PERFORMANCE]]).

---

## The heap, and why raw `new` is banned in modern C++

```cpp
int *p = new int(42);     // allocate on the heap
delete p;                 // YOU must free it
p = nullptr;              // and it's still a dangling pointer until you do this
```

Four things go wrong, and all four are classic security bugs:

| Bug | What happens |
|---|---|
| **Memory leak** | Forgot `delete` — memory never returns |
| **Double free** | `delete` twice — heap corruption, crash |
| **Dangling pointer** | Using it after `delete` — reads garbage or **executes attacker data** |
| **Wrong form** | `new[]` freed with `delete` instead of `delete[]` |

> ⚠️ **Any `return` or thrown exception between `new` and `delete` leaks.** This is why manual memory management doesn't survive contact with real code.

## Smart pointers — the actual answer

```cpp
#include <memory>

auto p = std::make_unique<int>(42);       // ONE owner, freed automatically
auto s = std::make_shared<Engine>();      // shared owners, freed at the last one
std::weak_ptr<Engine> w = s;              // observes without keeping it alive
```

```cpp
#include <memory>

std::unique_ptr<Engine> a = std::make_unique<Engine>();
std::unique_ptr<Engine> b = std::move(a);   // ownership MOVES; a is now null
```

| Type | Use |
|---|---|
| **`unique_ptr`** | **The default.** One owner. Zero overhead versus a raw pointer |
| `shared_ptr` | Genuinely shared ownership. Reference counted, so it costs something |
| `weak_ptr` | Breaks reference cycles — two `shared_ptr`s pointing at each other never free |
| Raw `T*` | **Non-owning observation only.** Never `delete` it |

> **The rule in modern C++: never write `new` or `delete`.** `make_unique`, `make_shared`, `std::vector` and `std::string` cover essentially everything. This is **RAII** — the destructor frees the resource when the object goes out of scope, including when an exception unwinds.

> **This is the same problem [[Rust]] solves with [[Ownership and borrowing]]** — one owner, freed at scope exit. Rust's compiler enforces it; C++ trusts you to use the right types.

---

## Pointers and arrays

```cpp
#include <iostream>

int arr[5] = {1, 2, 3, 4, 5};
int *p = arr;                 // an array decays to a pointer to its first element

std::cout << *(p + 2);        // 3  - pointer arithmetic
std::cout << p[2];            // 3  - identical
```

> ⚠️ **A decayed array has lost its size.** `sizeof(p)` gives you the pointer's size, not the array's. This is why C-style arrays crossing a function boundary are dangerous.

> ⚠️ **There is no bounds checking.** `p[100]` reads whatever is there. It usually doesn't crash — it silently returns garbage, or corrupts something you'll notice much later.

```cpp
#include <vector>
#include <span>

std::vector<int> v = {1, 2, 3, 4, 5};   // knows its size, grows, frees itself
v.at(10);                                // throws std::out_of_range
v[10];                                   // undefined behaviour - no check

void process(std::span<const int> data); // C++20: pointer + length, safely
```

> **Use `std::vector`, not C arrays.** Use `.at()` when the index comes from outside your code.

---

## Null checking

```cpp
int *p = nullptr;              // nullptr, NOT NULL and NOT 0
if (p != nullptr) { *p; }
if (p) { *p; }                 // same
```

> ⚠️ **Dereferencing a null pointer is undefined behaviour**, not a guaranteed crash. It might segfault, or it might quietly corrupt memory and fail somewhere unrelated an hour later.

---

## Debugging pointer bugs

```bash
g++ -fsanitize=address,undefined -g main.cpp   # catches most of this page at runtime
valgrind --leak-check=full ./a.out
g++ -Wall -Wextra -Werror                      # turn warnings on, always
```

> **AddressSanitizer is the single most valuable C++ debugging tool.** It reports use-after-free, buffer overflows and leaks with the exact line. Build your tests with it.

---

## Common mistakes

| Mistake | Consequence |
|---|---|
| `delete` forgotten | Memory leak |
| `delete` twice | Heap corruption |
| Using a pointer after `delete` | **Use-after-free — a security vulnerability** |
| `new[]` freed with `delete` | Undefined behaviour |
| Returning a pointer to a local | Dangling immediately |
| Passing a large object by value | Silent copy on every call |
| `p[100]` out of bounds | Silent garbage or corruption |
| Two `shared_ptr`s in a cycle | Never freed — use `weak_ptr` |
| Raw `new`/`delete` in new code | Use `make_unique` |
| Using `NULL` or `0` | Use `nullptr` |

## Related

[[C++]] · [[Memory]] · [[C++ libraries]] · [[C++ for this stack]] · [[Variables (C++)]] · [[Classes]] · [[C]] · [[Pointers (C)]] · [[Rust]] · [[Ownership and borrowing]] · [[33 — SYSTEMS PERFORMANCE]] · [[Security in practice]]
