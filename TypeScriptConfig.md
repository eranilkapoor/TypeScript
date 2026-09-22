**`tsconfig.json`** is the configuration file that tells the TypeScript compiler (`tsc`) how to type-check and compile your project — which files to include, what JavaScript version to output, how strict type checking should be, and dozens of other behaviors. Understanding its key options is essential for setting up a new project correctly and for making sense of configurations you inherit from existing codebases.

---

### **What `tsconfig.json` Is**

A `tsconfig.json` file at the root of a project marks that directory as the root of a TypeScript project and configures how `tsc` behaves within it. You can generate a starter file automatically:

```bash
tsc --init
```

This creates a `tsconfig.json` pre-populated with most available options, most of them commented out with brief explanations, so you can uncomment and adjust only what you need.

```json
{
  "compilerOptions": {
    "target": "es2016",
    "module": "commonjs",
    "strict": true
    // ...many more, commented out by default
  }
}
```

Once this file exists, running `tsc` with no arguments compiles the entire project according to its settings, and editors like VS Code use it to drive in-editor type checking and autocomplete.

---

### **Key Compiler Options**

#### **`target`**
Controls which version of JavaScript the compiled output uses — determining which language features get transformed ("downleveled") versus left as-is.

```json
{ "compilerOptions": { "target": "ES2020" } }
```

```typescript
// Source (TypeScript)
const greet = (name: string) => `Hello, ${name}`;

// Compiled with target: "ES5" — arrow functions downleveled to function expressions
// var greet = function (name) { return "Hello, " + name; };

// Compiled with target: "ES2020" — arrow functions and template literals preserved
// const greet = (name) => `Hello, ${name}`;
```

Choose a `target` based on the environments your code needs to run in — modern Node.js and modern browsers support recent targets like `ES2020`/`ES2022` directly, avoiding unnecessary downleveling.

#### **`module`**
Controls which module system the compiled output uses for `import`/`export` statements.

```json
{ "compilerOptions": { "module": "ESNext" } }
```

| Value | Output Style | Typical Use |
|---|---|---|
| `CommonJS` | `require()`/`module.exports` | Older Node.js projects |
| `ESNext` / `ES2020` | Native `import`/`export` | Modern bundlers (Vite, esbuild), modern Node ESM |
| `NodeNext` | Matches Node's own module resolution rules | Node.js projects mixing CJS and ESM |

#### **`outDir` and `rootDir`**
`outDir` sets where compiled `.js` output goes; `rootDir` sets the root of your source files, which determines the output folder structure.

```json
{
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

With this setup, `src/utils/math.ts` compiles to `dist/utils/math.js` — the folder structure under `rootDir` is mirrored under `outDir`.

#### **`strict`**
A single flag that enables a whole family of stricter type-checking rules at once. It's the single most impactful option for catching real bugs.

```json
{ "compilerOptions": { "strict": true } }
```

`strict: true` turns on several sub-flags individually:

1. **`strictNullChecks`**: `null` and `undefined` are no longer assignable to every type — you must explicitly include them in a type (`string | null`) if a value can be absent.
   ```typescript
   let name: string = null; // Error under strictNullChecks
   let maybeName: string | null = null; // OK
   ```
2. **`noImplicitAny`**: variables and parameters without an explicit type — and where TypeScript can't infer one — are flagged as errors instead of silently becoming `any`.
   ```typescript
   function greet(name) { /* Error: 'name' implicitly has an 'any' type */ }
   ```
3. **`strictFunctionTypes`**: checks function parameter types more precisely (contravariantly) when comparing function types, catching certain unsound assignments that looser checking would miss.
4. **`strictPropertyInitialization`**: class properties must be initialized in the constructor (or have a default value) unless explicitly marked optional or `!`-asserted.
   ```typescript
   class User {
     name: string; // Error: not initialized in constructor
   }
   ```
5. **`strictBindCallApply`**: checks that arguments passed to `.bind()`, `.call()`, and `.apply()` match the underlying function's signature.
6. **`alwaysStrict`**: emits `"use strict"` in output files and parses code in strict mode, matching standard JS strict-mode rules.
7. **`useUnknownInCatchVariables`**: types caught exceptions as `unknown` instead of `any` (see [AsyncProgrammingInTypeScript.md](./AsyncProgrammingInTypeScript.md) for why this matters).

You can enable `strict` and selectively turn off individual sub-flags if needed, but starting with everything on is the recommended default.

#### **`esModuleInterop`**
Fixes interoperability issues when importing CommonJS modules (like many older npm packages) using ES module `import` syntax.

```json
{ "compilerOptions": { "esModuleInterop": true } }
```

```typescript
// Without esModuleInterop, this often requires:
import * as express from "express";

// With esModuleInterop: true, the more natural default-import syntax works:
import express from "express";
```

#### **`skipLibCheck`**
Skips type-checking of all `.d.ts` declaration files (including those inside `node_modules`), which significantly speeds up compilation and avoids errors caused by conflicting or slightly incorrect third-party type definitions.

```json
{ "compilerOptions": { "skipLibCheck": true } }
```

This is commonly enabled in most real-world projects, since you generally don't need `tsc` to verify the correctness of libraries' own type definitions — only your own code against them.

#### **`sourceMap`**
Generates `.map` files alongside compiled `.js` output, letting debuggers and browser dev tools map compiled JavaScript back to the original TypeScript source.

```json
{ "compilerOptions": { "sourceMap": true } }
```

With source maps enabled, setting a breakpoint or reading a stack trace shows your original `.ts` line numbers instead of the compiled `.js` output.

#### **`declaration`**
Emits a `.d.ts` declaration file alongside each compiled `.js` file, describing the types of everything that module exports — essential when publishing a library for other TypeScript projects to consume.

```json
{ "compilerOptions": { "declaration": true } }
```

```typescript
// mathUtils.ts
export function add(a: number, b: number): number {
  return a + b;
}

// Generated mathUtils.d.ts:
// export declare function add(a: number, b: number): number;
```

#### **`include`, `exclude`, and `files`**
Control which files are part of the compilation.

```json
{
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"],
  "files": ["src/globalTypes.d.ts"]
}
```

1. **`include`**: glob patterns describing which files/folders to compile (defaults to everything if omitted).
2. **`exclude`**: glob patterns to omit from what `include` matched — commonly `node_modules`, build output, and test files.
3. **`files`**: an explicit list of individual files always included, regardless of `include`/`exclude` — useful for a small number of global ambient declaration files.

---

### **A Sample, Realistic `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "sourceMap": true,
    "declaration": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

This configuration targets a modern JavaScript environment, emits both compiled JS and `.d.ts` declaration files for consumers, enables full strict type checking, and adds a few extra quality-of-life options (`noUnusedLocals`, `noUnusedParameters`) that catch dead code.

---

### **Why `strict: true` Is Recommended for All New Projects**

Starting a project with `strict: true` from day one is strongly recommended, for a few concrete reasons:

1. **Catches real bugs early**: `strictNullChecks` alone eliminates an enormous class of "cannot read property of undefined" runtime errors by forcing you to handle the possibility of `null`/`undefined` at compile time.
2. **Retrofitting strict mode later is painful**: enabling `strict` on a large existing non-strict codebase typically surfaces hundreds or thousands of errors at once, since implicit `any`s and unchecked nulls accumulate silently over time. Starting strict avoids that cliff entirely.
3. **Improves editor tooling**: stricter types give autocomplete, refactoring tools, and inline error checking far more accurate information to work with.
4. **It's the ecosystem default**: most modern TypeScript starter templates (Vite, Next.js, NestJS) ship with `strict: true` out of the box, so aligning with it keeps your project consistent with common conventions and with libraries' own type expectations.

---

### **Best Practices**
- Run `tsc --init` to generate a starting `tsconfig.json`, then trim it down to the options you actually understand and need.
- Always enable `strict: true` for new projects rather than opting into sub-flags individually.
- Enable `skipLibCheck: true` in virtually all projects to avoid slow, often-irrelevant type checking of third-party `.d.ts` files.
- Set `declaration: true` (and `declarationMap` if useful) whenever you're publishing a package for other TypeScript consumers.
- Keep `include`/`exclude` tight and explicit — accidentally compiling test files or build output alongside source code causes confusing errors and bloated output.
- Match `target` and `lib` to the actual runtime environments you support, rather than defaulting to the oldest possible target "just in case."

---

### **Interview Questions**

**Q1. What is `tsconfig.json`, and how do you generate one?**
It's the configuration file that tells the TypeScript compiler how to type-check and compile a project — which files to include, what JS version to target, how strict to be, and more. You can generate a starter file with `tsc --init`.

**Q2. What does the `target` compiler option control?**
It controls which version of JavaScript the compiler emits as output, determining which newer language features get transformed ("downleveled") into older equivalents versus left as-is for environments that already support them.

**Q3. What's the difference between `outDir` and `rootDir`?**
`rootDir` specifies the root of your TypeScript source files, used to compute the output folder structure. `outDir` specifies where the compiled JavaScript output should be written, mirroring the structure under `rootDir`.

**Q4. What does enabling `strict: true` actually do?**
It turns on a bundle of stricter type-checking sub-flags at once, including `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictPropertyInitialization`, `strictBindCallApply`, `alwaysStrict`, and `useUnknownInCatchVariables`.

**Q5. What problem does `strictNullChecks` solve?**
Without it, `null` and `undefined` are silently assignable to any type, which is a common source of runtime "cannot read property of undefined" errors. With it enabled, a type must explicitly include `null`/`undefined` (e.g. `string | null`) for a variable to hold one of those values.

**Q6. What does `noImplicitAny` do, and why is it useful?**
It raises a compile error whenever a variable or parameter's type can't be inferred and would otherwise silently fall back to `any`. It's useful because `any` disables type checking entirely, so preventing accidental `any`s keeps the codebase properly type-checked.

**Q7. What does `esModuleInterop` fix?**
It fixes interoperability issues when importing CommonJS modules (common among older npm packages) using ES module default-import syntax, letting you write `import express from "express"` instead of the more awkward `import * as express from "express"`.

**Q8. Why would you enable `skipLibCheck`?**
It skips type-checking `.d.ts` declaration files, including those inside `node_modules`, which significantly speeds up compilation and avoids errors from third-party type definitions that you don't control and generally don't need to verify.

**Q9. What is the purpose of `sourceMap`, and when is it useful?**
It generates `.map` files that let debuggers and browser dev tools map compiled JavaScript back to the original TypeScript source lines, making it possible to debug directly against your `.ts` files even though the code that's actually running is compiled `.js`.

**Q10. When would you enable the `declaration` option?**
When you're publishing a package meant to be consumed by other TypeScript projects — `declaration: true` emits `.d.ts` files describing your module's exported types alongside the compiled JavaScript, so consumers get full type checking and autocomplete.

**Q11. What's the difference between `include`, `exclude`, and `files`?**
`include` specifies glob patterns for which files/folders are part of the compilation. `exclude` removes matching files from what `include` selected (commonly `node_modules`, build output, tests). `files` is an explicit list of individual files always included regardless of the other two, typically used for a small number of global declaration files.

**Q12. Why is it recommended to enable `strict: true` from the very start of a new project rather than adding it later?**
Because retrofitting strict mode onto an already-large codebase tends to surface a huge backlog of implicit `any`s and unchecked null/undefined issues all at once, making adoption painful. Starting with `strict: true` from day one keeps the codebase consistently well-typed as it grows, and matches the default convention of most modern TypeScript tooling and starter templates.
