fetch is built into the browser, Node and React Native, no library needed.

GET request with await

```js
/*
1. fetch returns a promise that resolves to a Response
2. .json() is ALSO async, so it needs its own await
3. the response body is the parsed JS object after that
*/

async function getUser() {
    const res = await fetch("https://api.example.com/users/1");
    const data = await res.json();

    console.log(data.name);  // output: Bob
}

getUser();
```

Handling a non-ok response, fetch does NOT throw on a 404 or 500

```js
async function getUser(id) {
    try {
        const res = await fetch(`https://api.example.com/users/${id}`);

        if (!res.ok) {
            // res.ok is false for anything outside 200-299
            throw new Error(`Request failed: ${res.status}`);
        }

        const data = await res.json();
        return data;
    } catch (err) {
        console.log("fetch failed:", err.message);
        return null;
    }
}

getUser(999);  // output: fetch failed: Request failed: 404
```

POST with headers and a body

```js
async function createUser(name) {
    const res = await fetch("https://api.example.com/users", {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${token}`,
        },
        body: JSON.stringify({ name: name }),   // body must be a string
    });

    if (!res.ok) throw new Error(`Failed: ${res.status}`);

    return await res.json();
}

createUser("Bob").then((user) => console.log(user.id));  // output: 7
```

Query params without messing up the string yourself

```js
const params = new URLSearchParams({ page: 2, limit: 10 });

const res = await fetch(`https://api.example.com/users?${params}`);

console.log(params.toString());  // output: page=2&limit=10
```

Loading it into React state

```js
useEffect(() => {
    async function load() {
        try {
            setLoading(true);
            const res = await fetch("https://api.example.com/posts");
            if (!res.ok) throw new Error(res.status);
            setPosts(await res.json());
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }

    load();
}, []);
```
