JSON is built into JavaScript, no library. Two functions do everything: `JSON.parse` turns text into objects, `JSON.stringify` turns objects into text.

The name is misleading though — a parsed JSON document is just a plain JS object, so all the normal array and object tools apply. See [[Array methods (JavaScript)]].

Parse and stringify

```js
/*
1. parse takes a STRING and gives you a real object
2. stringify goes the other way
3. the 3rd argument to stringify is indentation, for readable output
*/

const text = '{"name":"Huzayl","age":21,"tags":["rust","hardware"]}';

const data = JSON.parse(text);
console.log(data.name);       // output: Huzayl
console.log(data.tags[0]);    // output: rust

const back = JSON.stringify(data);          // compact, one line
const pretty = JSON.stringify(data, null, 2);  // indented by 2
```

Parsing throws, so wrap it

`JSON.parse` throws a `SyntaxError` on bad input. That includes the very common case of a server returning an HTML error page.

```js
function safeParse(text) {
    try {
        return JSON.parse(text);
    } catch (err) {
        console.log("bad JSON:", err.message);
        console.log("raw body was:", text.slice(0, 200));
        return null;
    }
}
```

Printing the raw body in the catch block is what turns "Unexpected token < in JSON" into "oh, it returned a login page".

From an API

`res.json()` does the parsing for you. See [[Fetch and APIs (JavaScript)]].

```js
async function getUsers() {
    const res = await fetch("https://api.example.com/users");

    if (!res.ok) throw new Error(`HTTP ${res.status}`);

    return await res.json();     // already parsed
}
```

If you need the raw text as well as the parsed object, read the text once and parse it yourself — the body can only be consumed one time.

```js
const text = await res.text();
const data = JSON.parse(text);    // now you still have `text` for logging
```

Filtering and transforming

This is the everyday work. Given:

```js
const users = [
    { id: 1, name: "Ana",  age: 34, active: true,  city: "London" },
    { id: 2, name: "Bo",   age: 19, active: false, city: "Leeds"  },
    { id: 3, name: "Cy",   age: 41, active: true,  city: "London" },
];
```

```js
// filter - keep matching items
users.filter(u => u.age > 30);
users.filter(u => u.active && u.city === "London");

// map - reshape every item
users.map(u => u.name);                        // ["Ana","Bo","Cy"]
users.map(u => ({ id: u.id, name: u.name }));  // smaller objects

// find - first match, or undefined
users.find(u => u.id === 2);

// some / every - booleans
users.some(u => u.age > 40);      // true
users.every(u => u.active);       // false

// sort - NOTE this mutates, copy first
[...users].sort((a, b) => a.age - b.age);
[...users].sort((a, b) => b.age - a.age);          // descending
[...users].sort((a, b) => a.name.localeCompare(b.name));  // strings

// chained, the common shape
const result = users
    .filter(u => u.active)
    .map(u => ({ name: u.name, city: u.city }))
    .sort((a, b) => a.name.localeCompare(b.name));
```

Aggregating with reduce

```js
/*
1. reduce carries an accumulator through every item
2. the 2nd argument is its starting value
3. group-by is the pattern worth memorising
*/

// sum
const totalAge = users.reduce((sum, u) => sum + u.age, 0);

// count
const activeCount = users.filter(u => u.active).length;

// group by a field
const byCity = users.reduce((acc, u) => {
    (acc[u.city] ||= []).push(u);
    return acc;
}, {});
// { London: [Ana, Cy], Leeds: [Bo] }

// count per group
const counts = users.reduce((acc, u) => {
    acc[u.city] = (acc[u.city] || 0) + 1;
    return acc;
}, {});
// { London: 2, Leeds: 1 }
```

Modern runtimes also have `Object.groupBy`:

```js
const byCity = Object.groupBy(users, u => u.city);
```

Reaching into nested data safely

```js
const data = { user: { address: { city: "London" } } };

data.user.address.city;        // "London"
data.user?.phone?.number;      // undefined, no crash
data.user?.phone?.number ?? "none";   // "none"

// array of nested things
users.map(u => u.address?.city ?? "unknown");
```

Optional chaining `?.` and nullish coalescing `??` are what make deep API responses tolerable. Use `??` not `||` — `||` also replaces `0` and `""`, which is usually wrong.

Flattening nested arrays

```js
const orders = [
    { id: 1, items: ["a", "b"] },
    { id: 2, items: ["c"] },
];

orders.flatMap(o => o.items);          // ["a","b","c"]
orders.map(o => o.items).flat();       // same
```

stringify details worth knowing

```js
// pick only certain keys
JSON.stringify(user, ["id", "name"]);

// transform values on the way out
JSON.stringify(user, (key, value) =>
    key === "password" ? undefined : value    // undefined removes the key
);

// dates become ISO strings automatically
JSON.stringify({ when: new Date() });
// {"when":"2026-07-22T11:08:00.000Z"}
```

Things stringify silently drops

```js
JSON.stringify({
    a: undefined,        // dropped
    b: () => {},         // dropped, functions aren't JSON
    c: Symbol("x"),      // dropped
    d: NaN,              // becomes null
    e: Infinity,         // becomes null
    f: new Date(),       // becomes a string
    g: new Map(),        // becomes {} - a common surprise
    h: 10n,              // THROWS, BigInt is not supported
});
// {"d":null,"e":null,"f":"2026-...","g":{}}
```

`Map` and `Set` turning into `{}` catches people out. Convert them first:

```js
JSON.stringify([...myMap]);        // array of [key, value] pairs
JSON.stringify(Object.fromEntries(myMap));
```

Circular references throw

```js
const a = {};
a.self = a;
JSON.stringify(a);        // TypeError: Converting circular structure to JSON
```

Deep copying

```js
// old trick, loses dates/undefined/Map, but fine for pure JSON data
const copy = JSON.parse(JSON.stringify(data));

// modern, handles more types and is faster
const copy = structuredClone(data);
```

Reviver, for fixing types on the way in

```js
/*
1. the reviver runs on every key/value as it parses
2. good for turning ISO date strings back into Date objects
*/

const data = JSON.parse(text, (key, value) => {
    if (typeof value === "string" && /^\d{4}-\d{2}-\d{2}T/.test(value)) {
        return new Date(value);
    }
    return value;
});
```

Big integers

Same trap as everywhere — see [[JSON]]. Beyond 2^53 you lose precision silently.

```js
JSON.parse('{"id":9007199254740993}').id;   // 9007199254740992, wrong

// if the API sends it as a string, keep it as a string
JSON.parse('{"id":"9007199254740993"}').id; // "9007199254740993", exact
```

Reading and writing files in Node

```js
import { readFile, writeFile } from "node:fs/promises";

const data = JSON.parse(await readFile("samples/users.json", "utf8"));

await writeFile("out.json", JSON.stringify(data, null, 2));
```

For JSON Lines, split on newlines and parse each:

```js
const lines = (await readFile("logs.jsonl", "utf8"))
    .split("\n")
    .filter(Boolean)
    .map(JSON.parse);

const errors = lines.filter(l => l.level === "error");
```

For the typed version of all this see [[JSON (TypeScript)]]. For exploring a response before you write any code, [[jq]] and [[Investigating an API]].
