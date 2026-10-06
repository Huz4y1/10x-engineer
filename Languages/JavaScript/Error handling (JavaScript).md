try / catch / finally

```js
/*
1. try holds the code that might blow up
2. catch runs only if it did, and receives the error object
3. finally runs either way, good for cleanup like setLoading(false)
*/

try {
    const data = JSON.parse("not json at all");
    console.log(data);
} catch (err) {
    console.log("could not parse:", err.message);
} finally {
    console.log("cleaning up");
}

// output: could not parse: Unexpected token 'o', "not json at all" is not valid JSON
// output: cleaning up
```

Throwing your own Error

```js
function setAge(age) {
    if (typeof age !== "number") {
        throw new Error("age must be a number");
    }
    if (age < 0) {
        throw new RangeError("age can't be negative");
    }
    return age;
}

try {
    setAge(-5);
} catch (err) {
    console.log(err.name);     // output: RangeError
    console.log(err.message);  // output: age can't be negative
}
```

Catching a specific case and rethrowing the rest

```js
class NotFoundError extends Error {
    constructor(message) {
        super(message);
        this.name = "NotFoundError";
    }
}

function load(id) {
    if (id === 999) throw new NotFoundError("no such user");
    throw new Error("something else broke");
}

try {
    load(999);
} catch (err) {
    if (err instanceof NotFoundError) {
        console.log("show the empty state");
    } else {
        throw err;   // not ours to handle, let it bubble up
    }
}

// output: show the empty state
```

Errors in async code need the try inside the async function

```js
async function load() {
    try {
        const res = await fetch("https://api.example.com/users");
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return await res.json();
    } catch (err) {
        console.log("load failed:", err.message);
        return [];   // return something safe instead of crashing the UI
    }
}

// a plain try/catch around a promise WITHOUT await catches nothing:
try {
    load();          // rejects later, we've already left the try block
} catch (err) {
    console.log("this never runs");
}
```
