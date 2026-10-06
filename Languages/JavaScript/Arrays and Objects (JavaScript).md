Array literals

```js
const fruits = ["apple", "banana", "cherry"];

console.log(fruits[0]);      // output: apple
console.log(fruits.length);  // output: 3

fruits.push("date");         // add to the end
console.log(fruits);         // output: [ 'apple', 'banana', 'cherry', 'date' ]
```

Object literals

```js
const user = {
    name: "Bob",
    age: 17,
    active: true,
};

console.log(user.name);     // output: Bob
console.log(user["age"]);   // output: 17

user.email = "bob@x.com";   // add a new key whenever
```

Nesting, which is what real API responses look like

```js
const post = {
    id: 1,
    title: "Hello",
    author: { name: "Bob", id: 7 },
    tags: ["js", "react"],
    comments: [
        { id: 1, body: "nice" },
        { id: 2, body: "cool" },
    ],
};

console.log(post.author.name);      // output: Bob
console.log(post.tags[1]);          // output: react
console.log(post.comments[0].body); // output: nice
```

Destructuring, pulling values out into variables

```js
const user = { name: "Bob", age: 17, city: "London" };

const { name, age } = user;
console.log(name, age);  // output: Bob 17

// rename while destructuring, and give a default
const { city, country = "UK" } = user;
console.log(city, country);  // output: London UK

// arrays destructure by position, this is how useState works
const [first, second] = ["apple", "banana"];
console.log(first);  // output: apple

// destructure right in the parameter list
function greet({ name }) {
    console.log(`Hello ${name}`);
}
greet(user);  // output: Hello Bob
```

Spread operator, copies things out into a new array or object

```js
/*
1. spread makes a NEW copy instead of mutating the original
2. this matters in React because state has to be replaced, not edited
3. later keys win, so { ...user, age: 18 } overrides age
*/

const user = { name: "Bob", age: 17 };
const older = { ...user, age: 18 };

console.log(user);   // output: { name: 'Bob', age: 17 }
console.log(older);  // output: { name: 'Bob', age: 18 }

const fruits = ["apple", "banana"];
const more = [...fruits, "cherry"];

console.log(more);  // output: [ 'apple', 'banana', 'cherry' ]
```
