---
tags: [cpp, variables, types, language]
---

# Variables (C++)

Language: [[C++]] · Pointers: [[Pointers]] · Memory: [[Memory]]

```cpp
int age = 20; // integers
double pi = 3.1415926; // precise decimals 
float temperature = 25.5f; // less precise decimal
char grade = 'A'; // single character - SINGLE quotes
bool isActive = true; // boolean
```

```cpp
#include <string>
std::string name = "John"; // string
```

> ⚠️ **`char` uses single quotes, `std::string` uses double quotes.** `char grade = "A";` doesn't compile — `"A"` is a two-character array (`'A'` plus the terminating `\0`), not a single character. This one catches everybody once.

---

## Fixed-width types — use these when the size matters

```cpp
#include <cstdint>

uint8_t  reg   = 0xFF;      // exactly 8 bits, 0 to 255
int32_t  count = -5;        // exactly 32 bits
uint64_t ticks = 0;
```

> ⚠️ **`int` is not a guaranteed size.** It's 32 bits on most desktops and can be 16 on a microcontroller. For hardware registers, file formats and network protocols, always use `<cstdint>` types ([[20 — EMBEDDED]], [[Bit manipulation and registers]]).

| Type | Size | Range |
|---|---|---|
| `char` | 1 byte | One character |
| `int` | usually 4 | ±2.1 billion |
| `long long` | ≥8 | ±9.2 quintillion |
| `float` | 4 | ~7 significant digits |
| `double` | 8 | ~15 significant digits |
| `bool` | 1 | `true` / `false` |
| `size_t` | pointer-sized | Sizes and indices — **unsigned** |

> ⚠️ **`float` has only about 7 digits.** `float temperature = 25.5f;` is fine; a sensor timestamp in a `float` loses seconds. **Default to `double`** unless you're on a GPU or memory-constrained ([[Numerical methods in practice]]).

> ⚠️ **Never compare floats with `==`.** `0.1 + 0.2 != 0.3`. Use `std::abs(a - b) < 1e-9`.

### Unsigned wraps around

```cpp
unsigned int x = 0;
x--;                          // 4294967295, not -1

for (size_t i = v.size() - 1; i >= 0; i--) { }    // INFINITE LOOP - size_t is never < 0
for (size_t i = v.size(); i-- > 0; ) { }          // correct countdown
```

> ⚠️ **`v.size()` returns an unsigned `size_t`.** `v.size() - 1` on an empty vector is a gigantic number, not `-1`. This is a real and common crash.

---

## `auto` and initialisation

```cpp
#include <string>

auto age = 20;                     // int
auto pi = 3.14;                    // double
auto name = std::string("John");   // std::string
auto it = v.begin();               // saves writing std::vector<int>::iterator
```

```cpp
int a = 20;         // copy initialisation
int b{20};          // brace initialisation - PREFERRED
int c(20);

int d{3.7};         // ERROR: braces refuse narrowing. Good.
int e = 3.7;        // silently 3
```

> **Use braces `{}`.** They reject narrowing conversions that silently lose data.

> ⚠️ **An uninitialised local variable contains garbage, not zero.** `int x;` then reading `x` is undefined behaviour. **Always initialise.**

```cpp
#include <string>

int x{};            // zero-initialised
std::string s{};    // empty
```

## `const` and `constexpr`

```cpp
#include <vector>
#include <string>

const double GRAVITY = 9.81;              // can't change after this
constexpr int BUFFER = 1024;              // computed at COMPILE time
const std::string &ref = getName();       // read-only reference, no copy

void print(const std::vector<int> &v);    // "I will not modify your vector"
```

> **Mark everything `const` that doesn't change.** It documents intent, prevents accidents, and lets the compiler optimise. `constexpr` goes further — the value is baked in before the program runs.

## Scope and lifetime

```cpp
int global = 1;              // whole program

void f() {
    int local = 2;           // dies at the closing brace
    static int calls = 0;    // initialised ONCE, survives between calls
    calls++;
}
```

| Storage | Lives | Speed |
|---|---|---|
| **Stack** (locals) | Until the closing brace | Fast |
| **Heap** (`new` / `make_unique`) | Until freed | Slower |
| **Static / global** | Whole program | — |

> ⚠️ **Never return a pointer or reference to a local.** It dies at the closing brace and you're left pointing at reclaimed stack memory ([[Pointers]]).

## Strings

```cpp
#include <string>
std::string name = "John";
name += " Smith";
name.length();  name.substr(0, 4);  name.find("Smith");
name.empty();   name.c_str();                    // C-style char* for old APIs

std::string_view view = name;                    // C++17: no copy, read-only

#include <format>
auto msg = std::format("{} is {}", name, age);   // C++20
```

> **Use `std::string`, not `char[]`.** It manages its own memory, knows its length, and can't overflow. C-style strings are the source of a large share of security bugs ([[Security in practice]]).

## Casting

```cpp
double d = 3.99;
int i = static_cast<int>(d);        // 3 - explicit, checked at compile time
int j = (int)d;                     // C-style: works, but says nothing about intent
```

> **Use `static_cast`.** It's searchable and the compiler verifies the conversion makes sense. C-style casts silently do whatever it takes.

```cpp
int a = 7, b = 2;
a / b;                              // 3  - INTEGER division
static_cast<double>(a) / b;         // 3.5
```

> ⚠️ **Integer division truncates.** `7 / 2` is `3`, not `3.5`. Cast one side first.

## Common mistakes

| Mistake | Consequence |
|---|---|
| `char c = "A"` | Doesn't compile — use `'A'` |
| Uninitialised variable | Garbage value, undefined behaviour |
| `v.size() - 1` when empty | Enormous unsigned number |
| `i >= 0` loop with `size_t` | Infinite loop |
| `float` for precise values | ~7 digits, silent loss |
| `==` on floats | Unpredictably false |
| `7 / 2` expecting 3.5 | Integer division |
| `int` for a hardware register | Wrong width on another platform |
| Returning a reference to a local | Dangling reference |
| C-style cast | Hides a mistake the compiler could catch |

## Related

[[C++]] · [[Pointers]] · [[Memory]] · [[Types]] · [[Classes]] · [[C++ libraries]] · [[C]] · [[Variables (C)]] · [[Numerical methods in practice]] · [[20 — EMBEDDED]] · [[03 — PROGRAMMING]]
