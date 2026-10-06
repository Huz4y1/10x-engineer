Typing parameters and the return value

```ts
function add(a: number, b: number): number {
  return a + b;
}

add(2, 3);
add(2, "3"); // Error: Argument of type 'string' is not assignable to parameter of type 'number'
```

The return type is usually inferred so you can leave it off, but writing it stops you returning the wrong thing by accident.

```ts
function greet(name: string) {
  return "Hello " + name;   // inferred as string
}
```

Optional and default params

```ts
// optional params must come last
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"} ${name}`;
}

// default value, no ? needed and the type is inferred
function greet2(name: string, greeting = "Hello"): string {
  return `${greeting} ${name}`;
}

greet("Bob");
greet("Bob", "Hi");
```

void return

```ts
function logUser(user: { id: number }): void {
  console.log(user.id);
  // nothing returned
}
```

Arrow function types

```ts
const add = (a: number, b: number): number => a + b;

// naming the signature with a type alias
type Adder = (a: number, b: number) => number;

const add2: Adder = (a, b) => a + b;  // params inferred from Adder, no annotation needed
```

Typing a callback

```ts
function fetchUser(id: number, onDone: (user: User) => void): void {
  // ...
}

fetchUser(1, (user) => console.log(user.username));

// callbacks that can fail, error is unknown not any
function load(cb: (err: unknown, data?: string) => void) {
  cb(null, "hello");
}
```
