Rendered TypeScript

user input

```tsx
'use client';
import { useState } from 'react';

export default function UsernameInput() {
  const [username, setUsername] = useState('');

  return (
    <input
      type="text"
      name="username"
      value={username}
      onChange={(e) => setUsername(e.target.value)}
    />
  );
}
```

Form

```tsx
"use client";

import { useState } from "react";

export default function LoginPage() {

const [username, setUsername] =
useState("");

async function login() {

await fetch(
"http://localhost:8080/login",
{
method: "POST",
headers: {
"Content-Type":
"application/json"
},
body: JSON.stringify({
username
})
}
);
}

return (
<>
<input
value={username}
onChange={(e) =>
setUsername(
e.target.value
)
}
/>

<button onClick={login}>
Login
</button>
</>
);
}
```

Drop down menu

```tsx
'use client';
import { useState } from 'react';

export default function FruitSelect() {
  const [fruit, setFruit] = useState('Apple');

  return (
    <select name="fruit" value={fruit} onChange={(e) => setFruit(e.target.value)}>
      <option>Apple</option>
      <option>Banana</option>
      <option>Orange</option>
    </select>
  );
}
```

button

```tsx
import Link from 'next/link';

<Link href="/dashboard">
  <button>Dashboard</button>
</Link>
```

Tables in HTML

```tsx
export default function PeopleTable() {
  return (
    <table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Age</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>John</td>
          <td>25</td>
        </tr>
        <tr>
          <td>Sarah</td>
          <td>30</td>
        </tr>
      </tbody>
    </table>
  );
}
```

This would render to the page like this

|Name|Age|
|---|---|
|John|25|
|Sarah|30|

The <table> is what everything goes inside

The <thread> usually contains the column names

the <tr> is for table row