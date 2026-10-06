tsconfig.json sits at the project root and tells the compiler which files to check, how strict to be and what version of JavaScript to output. Next.js generates one for you.

A minimal tsconfig.json

```ts
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

The options that actually matter:

|Option|What it does|
|---|---|
|strict|Turns on all the strict checks, this is the whole point of using TS|
|target|Which JS version to compile down to|
|module / moduleResolution|How imports are resolved, "bundler" for Next.js|
|paths|Import aliases so you write @/lib/utils not ../../lib/utils|
|outDir|Where the compiled .js goes|
|include / exclude|Which files get checked|
|noEmit|Type check only, let the bundler do the output|

What strict actually turns on

```ts
// strictNullChecks, the big one
let token: string | undefined;
token.length;   // Error: 'token' is possibly 'undefined'

// noImplicitAny
function greet(name) {   // Error: Parameter 'name' implicitly has an 'any' type
  return name;
}

// useUnknownInCatchVariables
try { } catch (err) {
  err.message;  // Error: 'err' is of type 'unknown'
}
```

Running tsc

```bash
npm install -D typescript

npx tsc --init        # generate a tsconfig.json with everything commented

npx tsc               # compile using tsconfig.json
npx tsc --noEmit      # just type check, no output files
npx tsc --watch       # recheck on save
```

In a Next.js project you never really run tsc to build, next dev type checks as you go. The useful one is adding a script that fails the build on type errors.

```ts
// package.json
"scripts": {
  "typecheck": "tsc --noEmit"
}
```
