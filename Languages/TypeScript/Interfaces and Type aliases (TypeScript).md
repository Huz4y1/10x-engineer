Both describe the shape of data. Interfaces are for objects, type aliases can name anything.

Interface for an object shape

```ts
interface User {
  id: number;
  username: string;
  email: string;
}

const user: User = {
  id: 1,
  username: "bob",
  email: "bob@mail.com",
};

const bad: User = { id: 1 }; // Error: Property 'username' is missing in type '{ id: number; }'
```

Type alias, does the same job for objects but can also alias other things

```ts
type User = {
  id: number;
  username: string;
};

type ID = string | number;        // interface can't do this
type Callback = () => void;       // or this
```

Optional properties with ?

```ts
interface User {
  id: number;
  username: string;
  avatarUrl?: string;   // string | undefined, like Option<String> in Rust
}

const user: User = { id: 1, username: "bob" };  // fine, avatarUrl left out

user.avatarUrl.length;  // Error: 'user.avatarUrl' is possibly 'undefined'
```

readonly

```ts
interface Config {
  readonly apiUrl: string;
  timeout: number;
}

const config: Config = { apiUrl: "http://localhost:8080", timeout: 5000 };

config.timeout = 3000;   // fine
config.apiUrl = "x";     // Error: Cannot assign to 'apiUrl' because it is a read-only property
```

Extending

```ts
interface Person {
  name: string;
}

interface Employee extends Person {
  salary: number;
}

const e: Employee = { name: "Bob", salary: 30000 };

// type aliases do the same with an intersection
type Employee2 = Person & { salary: number };
```

When to use which:

|Situation|Use|
|---|---|
|Shape of an object or API model|interface|
|Props for a React component|interface|
|Union type, e.g. "loading" \| "done"|type|
|Function signature or tuple|type|
|Needs to be extended by others|interface|
