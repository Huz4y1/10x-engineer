if / else if / else

```js
const score = 72;

if (score >= 80) {
    console.log("A");
} else if (score >= 60) {
    console.log("B");
} else {
    console.log("Fail");
}

// output: B
```

Ternary, a one line if/else that returns a value

```js
const loggedIn = true;

const label = loggedIn ? "Log out" : "Log in";

console.log(label);  // output: Log out
```

Switch

```js
const status = "loading";

switch (status) {
    case "loading":
        console.log("Spinner");
        break;
    case "error":
        console.log("Try again");
        break;
    default:
        console.log("Show data");
}

// output: Spinner
```

Truthy and falsy, everything is truthy except these 6

```js
/*
falsy: false, 0, "", null, undefined, NaN
everything else is truthy, including [] and {}
*/

if ("") console.log("never runs");
if (0) console.log("never runs");
if ([]) console.log("runs, empty array is truthy");
if ({}) console.log("runs, empty object is truthy");

// output: runs, empty array is truthy
// output: runs, empty object is truthy
```

?? (nullish coalescing) only falls back on null/undefined, unlike ||

```js
const count = 0;

console.log(count || 10);  // output: 10  <- wrong, 0 is falsy
console.log(count ?? 10);  // output: 0   <- what we wanted
```

?. (optional chaining) stops it crashing when something is missing

```js
const user = { profile: { name: "Bob" } };
const empty = {};

console.log(user.profile?.name);   // output: Bob
console.log(empty.profile?.name);  // output: undefined
// console.log(empty.profile.name); // TypeError: Cannot read properties of undefined

// works on function calls and arrays too
console.log(empty.getName?.());    // output: undefined
```
