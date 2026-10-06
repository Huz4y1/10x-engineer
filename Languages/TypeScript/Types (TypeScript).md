TypeScript is just JavaScript with types bolted on, the types disappear when it compiles.

Common types:

|Type|Meaning|
|---|---|
|string|Text|
|number|Any number, int and float are the same thing here|
|boolean|True/False|
|any|Turns type checking off, avoid|
|unknown|Safe version of any, must be narrowed before use|
|never|A value that never happens, e.g. a function that always throws|
|void|Function returns nothing|
|null|Deliberately empty|
|undefined|Not set yet|

Annotating variables

```ts
let name: string = "Bob";
let age: number = 25;
let active: boolean = true;

age = "25"; // Error: Type 'string' is not assignable to type 'number'
```

Type inference, you usually don't need the annotation

```ts
let name = "Bob";   // inferred as string
let age = 25;       // inferred as number

// but this one is inferred as any, so annotate it
let user;
let user: string;
```

any vs unknown

```ts
let a: any = "hello";
a.toUpperCase();      // allowed, no checking at all

let u: unknown = "hello";
u.toUpperCase();      // Error: 'u' is of type 'unknown'

if (typeof u === "string") {
  u.toUpperCase();    // fine now, TS knows it's a string
}
```

null and undefined

```ts
// closest thing to Rust's Option<T> is a union with undefined
let token: string | undefined;

token.length;         // Error: 'token' is possibly 'undefined'

if (token) {
  token.length;       // fine
}
```

void and never

```ts
function log(msg: string): void {
  console.log(msg);   // returns nothing
}

function fail(msg: string): never {
  throw new Error(msg);  // never returns at all
}
```
