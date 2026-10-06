Classic for loop

```js
for (let i = 0; i < 5; i++) {
    console.log(i);
}

// output: 0 1 2 3 4
```

While loop

```js
let count = 0;

while (count < 3) {
    console.log(count);
    count++;
}

// output: 0 1 2
```

for...of gives you the values, this is the one you want most of the time

```js
const fruits = ["apple", "banana", "cherry"];

for (const fruit of fruits) {
    console.log(fruit);
}

// output: apple banana cherry
```

for...in gives you the keys, use it on objects not arrays

```js
const user = { name: "Bob", age: 17 };

for (const key in user) {
    console.log(`${key}: ${user[key]}`);
}

// output: name: Bob
// output: age: 17
```

forEach, an array method that takes a callback

```js
/*
1. runs the callback once per item
2. second argument of the callback is the index
3. you can't break out of a forEach, use for...of if you need that
*/

const fruits = ["apple", "banana"];

fruits.forEach((fruit, index) => {
    console.log(`${index}: ${fruit}`);
});

// output: 0: apple
// output: 1: banana
```

Getting the index in a for...of with entries()

```js
const fruits = ["apple", "banana"];

for (const [i, fruit] of fruits.entries()) {
    console.log(i, fruit);
}

// output: 0 apple
// output: 1 banana
```
