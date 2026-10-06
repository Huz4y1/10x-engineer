Built in generics that build a new type out of an existing one, so you don't rewrite the same interface five times.

|Utility|What it does|
|---|---|
|Partial\<T\>|Makes every field optional|
|Required\<T\>|Makes every field required|
|Pick\<T, K\>|Keeps only the listed fields|
|Omit\<T, K\>|Removes the listed fields|
|Record\<K, V\>|Object with keys K and values V|
|ReturnType\<F\>|The return type of a function|

The base interface for all the examples

```ts
interface User {
  id: number;
  username: string;
  email: string;
  avatarUrl?: string;
}
```

One line each

```ts
// Partial, good for PATCH request bodies and form state
type UserUpdate = Partial<User>;              // { id?: number; username?: string; ... }

// Required, the opposite, avatarUrl is now mandatory
type FullUser = Required<User>;               // { id: number; ...; avatarUrl: string }

// Pick, only the fields you need for a list view
type UserPreview = Pick<User, "id" | "username">;

// Omit, everything except these, good for a create payload with no id yet
type NewUser = Omit<User, "id">;

// Record, a lookup keyed by id
type UserMap = Record<number, User>;

// ReturnType, grabs the type a function returns without writing it out
type Users = ReturnType<typeof getUsers>;     // Promise<User[]>
```

In practice

```ts
async function updateUser(id: number, changes: Partial<User>): Promise<User> {
  const res = await fetch(`http://localhost:8080/users/${id}`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(changes),
  });
  return res.json();
}

updateUser(1, { email: "new@mail.com" }); // fine, only sending one field
updateUser(1, { emial: "new@mail.com" }); // Error: 'emial' does not exist in type 'Partial<User>'
```
