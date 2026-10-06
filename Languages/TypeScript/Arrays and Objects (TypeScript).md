Typed arrays

```ts
const names: string[] = ["Bob", "Sarah"];
const ages: number[] = [25, 30];

// other syntax, same thing
const scores: Array<number> = [1, 2, 3];

names.push(5); // Error: Argument of type 'number' is not assignable to parameter of type 'string'
```

Array of interfaces, this is what an API list endpoint gives you

```ts
interface User {
  id: number;
  username: string;
}

const users: User[] = [
  { id: 1, username: "bob" },
  { id: 2, username: "sarah" },
];

// map keeps the types, name is a string here
const names = users.map((u) => u.username);
```

Nested object types

```ts
interface Post {
  id: number;
  title: string;
  author: {
    id: number;
    username: string;
  };
  tags: string[];
}

const post: Post = {
  id: 1,
  title: "Hello",
  author: { id: 1, username: "bob" },
  tags: ["rust", "ts"],
};

post.author.username;
```

Tuples, fixed length and each slot has its own type

```ts
// like a Rust tuple (i32, String)
let pair: [number, string] = [1, "bob"];

pair = ["bob", 1]; // Error: Type 'string' is not assignable to type 'number'

// this is what useState actually returns
const [count, setCount]: [number, (n: number) => void] = useState(0);
```

Record, an object used as a map

```ts
// Record<KeyType, ValueType>
const scores: Record<string, number> = {
  bob: 10,
  sarah: 12,
};

scores.bob = "ten"; // Error: Type 'string' is not assignable to type 'number'

// keys can be a literal union too
type Status = "active" | "banned";
const counts: Record<Status, number> = { active: 5, banned: 2 };
```
