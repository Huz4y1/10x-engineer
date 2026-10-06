An async function always returns a Promise, so the return type is always Promise<T> where T is what you actually return.

Typing an async function

```ts
async function getName(): Promise<string> {
  return "Bob";   // the string gets wrapped in a Promise automatically
}

async function logIt(): Promise<void> {
  console.log("done");   // returns nothing
}

const name = await getName();  // name is string, await unwraps the Promise
```

Awaiting a typed fetch

```ts
/*
1. fetch gives back a Response
2. res.json() returns Promise<any>, so it will not type check anything on its own
3. annotating the return type is what makes data typed
*/

interface User {
  id: number;
  username: string;
}

async function getUsers(): Promise<User[]> {
  const res = await fetch("http://localhost:8080/users");

  if (!res.ok) {
    throw new Error(`Request failed: ${res.status}`);
  }

  return res.json(); // trusted to match User[], TS does not check the actual response
}

const users = await getUsers();
users[0].username;
```

Typing the error in a catch

```ts
// catch is always unknown under strict mode, you have to narrow it
try {
  const users = await getUsers();
} catch (err: unknown) {
  if (err instanceof Error) {
    console.log(err.message);
  } else {
    console.log("Unknown error", err);
  }
}

// err.message straight away gives:
// Error: 'err' is of type 'unknown'
```

Returning a result instead of throwing, closer to Rust's Result

```ts
type Result<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };

async function safeGetUsers(): Promise<Result<User[]>> {
  try {
    const res = await fetch("http://localhost:8080/users");
    return { ok: true, data: await res.json() };
  } catch (err) {
    return { ok: false, error: err instanceof Error ? err.message : "failed" };
  }
}

const result = await safeGetUsers();

if (result.ok) {
  result.data.length;   // narrowed, data only exists here
} else {
  console.log(result.error);
}
```
