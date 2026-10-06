C++ has no JSON in the standard library. The de facto choice is `nlohmann/json`, which is header-only and makes JSON feel almost like a scripting language.

Getting it

It is one header file. Download `json.hpp` and include it, or use a package manager.

```bash
# vcpkg
vcpkg install nlohmann-json

# or just drop the single header in your project
curl -O https://raw.githubusercontent.com/nlohmann/json/develop/single_include/nlohmann/json.hpp
```

```cpp
#include <nlohmann/json.hpp>
using json = nlohmann::json;      // everyone does this alias
```

Compile with C++11 or later. See [[Header files]].

Parsing and printing

```cpp
#include <string>

/*
1. json::parse takes a string and gives you a dynamic tree
2. operator[] walks into it
3. dump(n) serialises back out with n-space indenting
*/

#include <nlohmann/json.hpp>
#include <iostream>

using json = nlohmann::json;

int main() {
    std::string text = R"({"name":"Huzayl","age":21,"tags":["rust","hardware"]})";

    json data = json::parse(text);

    std::cout << data["name"] << "\n";        // output: "Huzayl"  (with quotes)
    std::cout << data["name"].get<std::string>() << "\n";  // output: Huzayl

    std::cout << data.dump() << "\n";         // compact
    std::cout << data.dump(2) << "\n";        // indented
}
```

Note the difference on those two prints. Streaming a `json` value prints it *as JSON*, so a string comes out quoted. Use `.get<T>()` when you want the actual C++ value.

Raw string literals `R"(...)"` save you escaping every quote. Very worth using for embedded JSON.

Getting values out

```cpp
#include <string>

// explicit, throws if the type is wrong
auto name = data["name"].get<std::string>();
auto age  = data["age"].get<int>();

// implicit conversion also works
std::string name = data["name"];
int age = data["age"];

// with a fallback if the key is missing
auto city = data.value("city", "unknown");
auto score = data.value("score", 0.0);
```

`.value(key, default)` is the one to reach for with real API data. `data["city"]` on a missing key **creates a null entry** in a non-const object, which surprises people.

Checking before you read

```cpp
#include <iostream>
#include <string>

if (data.contains("email") && !data["email"].is_null()) {
    auto email = data["email"].get<std::string>();
}

// type checks
data.is_object();  data.is_array();  data.is_string();
data.is_number();  data.is_boolean(); data.is_null();

// at() throws instead of creating, safer for reads
try {
    auto x = data.at("missing");
} catch (const json::out_of_range& e) {
    std::cerr << e.what() << "\n";
}
```

Nested access

```cpp
#include <string>

json data = json::parse(R"({
    "user": { "address": { "city": "London" } },
    "items": [ {"id":1}, {"id":2} ]
})");

auto city = data["user"]["address"]["city"].get<std::string>();
auto id   = data["items"][0]["id"].get<int>();

// JSON pointer syntax, handy for deep paths
auto city2 = data["/user/address/city"_json_pointer].get<std::string>();
```

Iterating and filtering

There is no LINQ, so this is `<algorithm>` plus a loop.

```cpp
#include <iostream>
#include <string>
#include <algorithm>
#include <vector>

json users = json::parse(R"([
    {"id":1,"name":"Ana","age":34,"active":true},
    {"id":2,"name":"Bo","age":19,"active":false},
    {"id":3,"name":"Cy","age":41,"active":true}
])");

// plain loop, clearest for most work
for (const auto& u : users) {
    if (u["age"].get<int>() > 30) {
        std::cout << u["name"].get<std::string>() << "\n";
    }
}

// filter into a new json array
json adults = json::array();
std::copy_if(users.begin(), users.end(), std::back_inserter(adults),
             [](const json& u) { return u["age"].get<int>() > 30; });

// map to a vector of one field
std::vector<std::string> names;
std::transform(users.begin(), users.end(), std::back_inserter(names),
               [](const json& u) { return u["name"].get<std::string>(); });

// find first match
auto it = std::find_if(users.begin(), users.end(),
                       [](const json& u) { return u["id"] == 2; });
if (it != users.end()) std::cout << (*it)["name"] << "\n";

// count
auto n = std::count_if(users.begin(), users.end(),
                       [](const json& u) { return u["active"].get<bool>(); });

// sum
int total = 0;
for (const auto& u : users) total += u["age"].get<int>();

// sort a copy - json arrays support std::sort
json sorted = users;
std::sort(sorted.begin(), sorted.end(),
          [](const json& a, const json& b) {
              return a["age"].get<int>() < b["age"].get<int>();
          });
```

Iterating an object gives you values by default, so use `.items()` when you need keys:

```cpp
#include <iostream>

for (auto& [key, value] : data.items()) {
    std::cout << key << " = " << value << "\n";
}
```

Building JSON

```cpp
json j;
j["name"] = "Huzayl";
j["age"] = 21;
j["tags"] = {"rust", "hardware"};
j["address"]["city"] = "London";      // nested paths are created for you

// or all at once
json j2 = {
    {"name", "Huzayl"},
    {"age", 21},
    {"tags", json::array({"rust", "hardware"})},
    {"address", {{"city", "London"}}}
};

// empty containers must be explicit, {} alone is ambiguous
json empty_arr = json::array();
json empty_obj = json::object();
```

That ambiguity is a real gotcha: `json x = {};` gives you null, not an empty object.

Mapping to your own structs

Rather than passing `json` around your whole program, convert at the boundary.

```cpp
#include <vector>
#include <string>

/*
1. define the struct normally
2. the macro generates to_json / from_json for it
3. after that, .get<User>() and assignment both work
*/

struct User {
    int id;
    std::string name;
    int age;
    bool active;
};

NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE(User, id, name, age, active)

// now
User u = data.get<User>();
json back = u;

std::vector<User> all = users.get<std::vector<User>>();
```

For optional fields, use the `WITH_DEFAULT` variant so missing keys do not throw:

```cpp
NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE_WITH_DEFAULT(User, id, name, age, active)
```

Once you have `std::vector<User>` you can use ordinary algorithms with real types, which is much nicer than reaching into `json` everywhere.

Error handling

```cpp
#include <iostream>

try {
    json data = json::parse(text);
} catch (const json::parse_error& e) {
    std::cerr << "parse failed at byte " << e.byte << ": " << e.what() << "\n";
} catch (const json::type_error& e) {
    std::cerr << "wrong type: " << e.what() << "\n";
}

// non-throwing parse
json data = json::parse(text, nullptr, false);
if (data.is_discarded()) {
    std::cerr << "invalid JSON\n";
}
```

See [[Try and Except]].

Files

```cpp
#include <fstream>

// read
std::ifstream in("samples/users.json");
json data = json::parse(in);

// write
std::ofstream out("out.json");
out << data.dump(2);
```

See [[File handling (C++)]].

JSON Lines

```cpp
#include <iostream>
#include <string>
#include <fstream>

std::ifstream in("logs.jsonl");
std::string line;

while (std::getline(in, line)) {
    if (line.empty()) continue;
    json entry = json::parse(line);
    if (entry.value("level", "") == "error") {
        std::cout << entry.value("msg", "") << "\n";
    }
}
```

Big integers

`nlohmann/json` stores integers as `int64_t`, so unlike JavaScript you keep full precision. Reading a huge value into an `int` will overflow though, so use `int64_t` for ids — see [[Variables (C++)]].

Other libraries

```
  nlohmann/json      easiest, header-only, slowest. default choice.
  RapidJSON          much faster, DOM and SAX, more verbose
  simdjson           fastest parser available, read-only
  Boost.JSON         if you're already using Boost
```

Start with nlohmann. Move to simdjson only if profiling says parsing is your bottleneck, which for API-sized payloads it will not be.
