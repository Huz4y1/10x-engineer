Function declaration

```js
function greet(name) {
    return `Hello ${name}`;
}

console.log(greet("Bob"));  // output: Hello Bob
```

Arrow functions, shorter and what you'll see everywhere in React

```js
const greet = (name) => {
    return `Hello ${name}`;
};

// one expression? drop the braces and the return
const greetShort = (name) => `Hello ${name}`;

// returning an object needs wrapping brackets
const makeUser = (name) => ({ name: name });

console.log(greetShort("Bob"));       // output: Hello Bob
console.log(makeUser("Bob"));         // output: { name: 'Bob' }
```

Default parameters

```js
function greet(name = "stranger", greeting = "Hello") {
    return `${greeting} ${name}`;
}

console.log(greet());              // output: Hello stranger
console.log(greet("Bob"));         // output: Hello Bob
console.log(greet("Bob", "Yo"));   // output: Yo Bob
```

Rest parameters, collects any number of arguments into an array

```js
function total(...numbers) {
    let sum = 0;
    for (const n of numbers) {
        sum += n;
    }
    return sum;
}

console.log(total(1, 2, 3));       // output: 6
console.log(total(1, 2, 3, 4, 5)); // output: 15
```

Callbacks, passing a function into another function

```js
/*
1. handleClick is a function we pass as a value, note no brackets
2. onPress calls it later, whenever it wants
3. this is exactly how React props like onPress work
*/

function onPress(callback) {
    console.log("button pressed");
    callback("Bob");
}

function handleClick(name) {
    console.log(`hello from the callback, ${name}`);
}

onPress(handleClick);

// output: button pressed
// output: hello from the callback, Bob
```
