These all take a callback and most of them return a new array instead of changing the original.

map, transform every item into something else

```js
const numbers = [1, 2, 3];

const doubled = numbers.map((n) => n * 2);

console.log(doubled);  // output: [ 2, 4, 6 ]

// this is what you use to render a list in React
const users = [{ name: "Bob" }, { name: "Alice" }];
const names = users.map((u) => u.name);

console.log(names);  // output: [ 'Bob', 'Alice' ]
```

filter, keep only the items where the callback returns true

```js
const numbers = [1, 2, 3, 4, 5, 6];

const evens = numbers.filter((n) => n % 2 === 0);

console.log(evens);  // output: [ 2, 4, 6 ]
```

reduce, squash the whole array down into one value

```js
/*
1. the 0 at the end is the starting value of total
2. whatever you return becomes total on the next loop
*/

const numbers = [1, 2, 3, 4];

const sum = numbers.reduce((total, n) => total + n, 0);

console.log(sum);  // output: 10
```

find, get the first matching item (not an array, the actual item)

```js
const users = [
    { id: 1, name: "Bob" },
    { id: 2, name: "Alice" },
];

const found = users.find((u) => u.id === 2);

console.log(found);  // output: { id: 2, name: 'Alice' }
console.log(users.find((u) => u.id === 99));  // output: undefined
```

includes, simple true/false check

```js
const fruits = ["apple", "banana"];

console.log(fruits.includes("banana"));  // output: true
console.log(fruits.includes("cherry"));  // output: false
```

sort, careful because it mutates the original and sorts as strings by default

```js
const numbers = [10, 1, 5];

console.log([...numbers].sort());  // output: [ 1, 10, 5 ]  <- string sort!

// give it a compare function for numbers
console.log([...numbers].sort((a, b) => a - b));  // output: [ 1, 5, 10 ]
console.log([...numbers].sort((a, b) => b - a));  // output: [ 10, 5, 1 ]

// sorting objects by a key
const users = [{ name: "Bob" }, { name: "Alice" }];
users.sort((a, b) => a.name.localeCompare(b.name));

console.log(users);  // output: [ { name: 'Alice' }, { name: 'Bob' } ]
```

Chaining them together

```js
const products = [
    { name: "Shirt", price: 20, inStock: true },
    { name: "Hat", price: 10, inStock: false },
    { name: "Shoes", price: 50, inStock: true },
];

const total = products
    .filter((p) => p.inStock)
    .map((p) => p.price)
    .reduce((sum, price) => sum + price, 0);

console.log(total);  // output: 70
```
