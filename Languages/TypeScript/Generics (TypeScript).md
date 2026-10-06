Generics let you write something once and keep the type of whatever gets passed in, same idea as Rust generics but with no trait bounds.

Generic function

```ts
// T is a placeholder, filled in at the call site
function first<T>(items: T[]): T {
  return items[0];
}

const n = first([1, 2, 3]);        // n is number
const s = first(["a", "b"]);       // s is string

// without generics you'd have to use any and lose the type
```

Generic interface

```ts
interface Box<T> {
  value: T;
  updatedAt: string;
}

const nameBox: Box<string> = { value: "bob", updatedAt: "today" };
const userBox: Box<User> = { value: user, updatedAt: "today" };
```

Typed API response wrapper, this is the one you'll actually use with the Actix backend

```ts
interface ApiResponse<T> {
  success: boolean;
  data: T;
  message?: string;
}

// one wrapper, any payload
type UserResponse = ApiResponse<User>;
type UserListResponse = ApiResponse<User[]>;

async function getUsers(): Promise<ApiResponse<User[]>> {
  const res = await fetch("http://localhost:8080/users");
  return res.json();
}

const result = await getUsers();
result.data[0].username;   // fully typed
```

Constraints with extends

```ts
// T must at least have an id, like a trait bound
function getId<T extends { id: number }>(item: T): number {
  return item.id;
}

getId({ id: 1, username: "bob" }); // fine
getId({ username: "bob" });        // Error: Property 'id' is missing in type '{ username: string; }'

// constrain to keys of an object
function getField<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

getField(user, "username"); // string
getField(user, "nope");     // Error: Argument of type '"nope"' is not assignable
```
