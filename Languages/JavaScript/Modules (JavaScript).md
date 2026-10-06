Splitting code across files with ES modules, which is what Next.js and Expo use.

Named exports, you can have as many as you want per file

```js
// utils.js

export const TAX = 0.2;

export function addTax(price) {
    return price * (1 + TAX);
}

export function formatPrice(price) {
    return `£${price.toFixed(2)}`;
}
```

Named imports must match the name exactly, in braces

```js
// app.js

import { addTax, formatPrice, TAX } from "./utils.js";

console.log(TAX);                      // output: 0.2
console.log(formatPrice(addTax(100))); // output: £120.00

// rename on the way in if it clashes with something
import { addTax as applyTax } from "./utils.js";
```

Default export, only one per file, and you name it whatever you like

```js
// Button.js

export default function Button({ label }) {
    return label;
}

// you can mix a default and named exports in one file
export const SIZES = ["sm", "lg"];
```

```js
// app.js

import Button from "./Button.js";        // no braces, any name works
import MyButton from "./Button.js";      // same thing, different local name

import Button, { SIZES } from "./Button.js";  // default + named together
```

Importing from a package, no ./ in front

```js
/*
1. no path prefix means node_modules
2. React Native / Expo bits come in the same way
*/

import { useState, useEffect } from "react";
import { View, Text } from "react-native";
import Link from "next/link";

// import everything under one name
import * as utils from "./utils.js";
console.log(utils.TAX);  // output: 0.2
```

CommonJS, the older Node style you'll still bump into

```js
// old way
const utils = require("./utils.js");
module.exports = { addTax };

// modern way, same idea
import utils from "./utils.js";
export { addTax };
```
