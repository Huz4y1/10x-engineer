Every file with an export is its own module, a bit like a Rust module except one file is one module and there is no mod tree to declare.

Named exports

```ts
// src/lib/utils.ts

export function formatDate(date: string): string {
  return new Date(date).toLocaleDateString();
}

export const API_URL = "http://localhost:8080";
```

```ts
// src/app/page.tsx

import { formatDate, API_URL } from "@/lib/utils";

// rename on import
import { formatDate as fmt } from "@/lib/utils";
```

Default export, one per file, this is what Next.js pages use

```ts
// src/components/Navbar.tsx
export default function Navbar() { }

// importing, the name is yours to choose
import Navbar from "@/components/Navbar";
```

Exporting types, they get erased at compile time so mark them with export type

```ts
// src/types/user.ts

export interface User {
  id: number;
  username: string;
  email: string;
}

export type Status = "idle" | "loading" | "error";

// re-exporting a type from somewhere else
export type { ApiResponse } from "./api";
```

Importing an interface across files

```ts
// src/services/api.ts

import type { User } from "@/types/user";   // import type says this is types only

export async function getUsers(): Promise<User[]> {
  const res = await fetch("http://localhost:8080/users");
  return res.json();
}
```

```ts
// src/components/UserTable.tsx

import type { User } from "@/types/user";
import { getUsers } from "@/services/api";

// values and types can come in on one line too
import { getUsers, type User } from "@/services/api";
```

Keeping every interface in a types folder means the same User shape is used by the fetch, the props and the table, so a change to the backend model breaks in one place.
