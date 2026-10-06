Use const by default, let only when you need to reassign, and never var.

```js
const name = "Bob";   // can't be reassigned
let age = 17;         // can be reassigned
var old = true;       // old style, avoid it

age = 18;             // fine
// name = "Alice";    // TypeError: Assignment to constant variable
```

const on an object or array only locks the variable, not the contents

```js
const user = { name: "Bob" };

user.name = "Alice";  // allowed, we're changing what's inside
// user = {};         // not allowed, that's a reassignment

console.log(user.name);  // output: Alice
```

Why var is bad

```js
/*
1. var ignores block scope, it leaks out of the if
2. let is block scoped so it only exists inside the braces
*/

if (true) {
    var leaky = "I escape";
    let safe = "I stay";
}

console.log(leaky);  // output: I escape
// console.log(safe); // ReferenceError: safe is not defined
```

Template literals (backticks) instead of joining strings with +

```js
const name = "Bob";
const items = 3;

console.log(`Hello ${name}, you have ${items} items`);
// output: Hello Bob, you have 3 items

console.log(`Next year you'll have ${items + 1}`);  // any expression works
// output: Next year you'll have 4

// they can span multiple lines too
const message = `Hi ${name},
thanks for signing up.`;
```
