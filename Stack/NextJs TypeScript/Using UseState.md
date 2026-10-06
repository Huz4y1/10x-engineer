This lets a component remember values and update the screen with those values

```tsx
"use client";

import { useState } from "react";

export default function Counter() {

    const [count, setCount] = useState(0);

    return (
        <>
            <p>{count}</p>

            <button
                onClick={() =>
                    setCount(count + 1)
                }
            >
                Add
            </button>
        </>
    );
}
```