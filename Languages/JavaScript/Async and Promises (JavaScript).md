Async code is anything that finishes later, like a network request or a timer. Three generations of doing it:

Callbacks, the old way, nesting these gets ugly fast

```js
function getUser(id, callback) {
    setTimeout(() => {
        callback({ id: id, name: "Bob" });
    }, 500);
}

getUser(1, (user) => {
    console.log(user.name);  // output: Bob
});
```

Promises with .then and .catch

```js
/*
1. a promise is an object that will hold a value later
2. resolve = it worked, reject = it failed
3. .then gets the resolved value, .catch gets the error
*/

function getUser(id) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (id > 0) {
                resolve({ id: id, name: "Bob" });
            } else {
                reject(new Error("bad id"));
            }
        }, 500);
    });
}

getUser(1)
    .then((user) => console.log(user.name))
    .catch((err) => console.log(err.message))
    .finally(() => console.log("done"));

// output: Bob
// output: done
```

async / await, the same thing but reads top to bottom

```js
async function showUser() {
    const user = await getUser(1);   // pauses here until it resolves
    console.log(user.name);
}

showUser();  // output: Bob

// an async function always returns a promise
console.log(showUser());  // output: Promise { <pending> }
```

try/catch with await, this is how you handle the failure

```js
async function showUser(id) {
    try {
        const user = await getUser(id);
        console.log(user.name);
    } catch (err) {
        console.log("failed:", err.message);
    } finally {
        console.log("finished either way");
    }
}

showUser(-1);
// output: failed: bad id
// output: finished either way
```

Promise.all runs them at the same time instead of one after the other

```js
async function loadAll() {
    const [a, b] = await Promise.all([getUser(1), getUser(2)]);
    console.log(a.name, b.name);
}

loadAll();  // output: Bob Bob

// awaiting them one by one would take twice as long:
// const a = await getUser(1);
// const b = await getUser(2);
```
