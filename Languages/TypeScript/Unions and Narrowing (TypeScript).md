A union means the value is one of these types. Closest thing TS has to a Rust enum, except there's no match, you narrow it yourself.

Union types

```ts
let id: string | number;

id = 1;      // fine
id = "abc";  // fine
id = true;   // Error: Type 'boolean' is not assignable to type 'string | number'
```

Literal union as a status field

```ts
type Status = "idle" | "loading" | "success" | "error";

let status: Status = "idle";

status = "loadng"; // Error: Type '"loadng"' is not assignable to type 'Status'

// this is why you use these instead of plain strings, typos become compile errors
```

Narrowing with typeof

```ts
function printId(id: string | number) {
  if (typeof id === "number") {
    console.log(id.toFixed(2));   // TS knows it's a number here
  } else {
    console.log(id.toUpperCase()); // and a string here
  }
}
```

Narrowing with in

```ts
interface Admin {
  username: string;
  permissions: string[];
}

interface Guest {
  username: string;
}

function describe(user: Admin | Guest) {
  if ("permissions" in user) {
    console.log(user.permissions.length); // narrowed to Admin
  } else {
    console.log(user.username);           // Guest
  }
}
```

Discriminated union, a shared literal field that tells the variants apart

```ts
/*
1. every variant has a "status" field with a literal type
2. checking that field narrows the whole object
3. this is the closest you get to matching on a Rust enum
*/

type ApiResult =
  | { status: "loading" }
  | { status: "success"; data: User[] }
  | { status: "error"; message: string };

function render(result: ApiResult) {
  switch (result.status) {
    case "loading":
      return "Loading...";
    case "success":
      return result.data.length;    // data only exists on this variant
    case "error":
      return result.message;
  }
}
```
