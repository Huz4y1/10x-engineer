JavaScript types are decided at runtime, you never write the type down.

|Type|Meaning|Example|
|---|---|---|
|string|Text|`"Bob"`|
|number|Any number, int or float|`42`, `3.14`|
|boolean|True/False|`true`|
|null|Deliberately empty|`null`|
|undefined|Not set yet|`undefined`|
|object|Key/value bag|`{ name: "Bob" }`|
|array|Ordered list (an object really)|`[1, 2, 3]`|
|symbol|Unique key, rarely used|`Symbol("id")`|
|bigint|Numbers bigger than number can hold|`9007199254740993n`|

typeof tells you what you've got

```js
console.log(typeof "Bob");      // output: string
console.log(typeof 42);         // output: number
console.log(typeof true);       // output: boolean
console.log(typeof undefined);  // output: undefined
console.log(typeof { a: 1 });   // output: object
console.log(typeof [1, 2, 3]);  // output: object  <- not "array"!
console.log(typeof null);       // output: object  <- famous JS bug
console.log(typeof 10n);        // output: bigint
```

Checking for an array properly

```js
console.log(Array.isArray([1, 2, 3]));  // output: true
console.log(Array.isArray("hello"));    // output: false
```

null vs undefined

```js
let a;              // never assigned
const b = null;     // assigned "nothing" on purpose

console.log(a);  // output: undefined
console.log(b);  // output: null
```

The == vs === trap, always use ===

```js
/*
1. == converts the types before comparing, which gives nonsense results
2. === compares value AND type, which is what you actually want
*/

console.log(1 == "1");    // output: true   <- converted the string
console.log(1 === "1");   // output: false  <- correct

console.log(0 == false);  // output: true
console.log(0 === false); // output: false

console.log(null == undefined);   // output: true
console.log(null === undefined);  // output: false
```
