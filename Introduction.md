**TypeScript** is a statically-typed superset of JavaScript developed and maintained by Microsoft. It adds an optional type system on top of JavaScript's existing syntax, then compiles ("transpiles") that typed code down into plain JavaScript that can run anywhere JavaScript already runs — browsers, Node.js, Deno, or any other JS engine. TypeScript doesn't replace JavaScript; it's a tool that sits on top of it, catching an entire category of bugs before your code ever executes.

---

### **What TypeScript Is**

TypeScript is often described as "JavaScript with types." More precisely:

1. **A superset of JavaScript**: every valid JavaScript program is (almost always) valid TypeScript. TypeScript doesn't introduce a new language so much as it layers static type-checking on top of the one you already know.
2. **Statically typed**: types are checked at *compile time* (before the code runs), rather than only at *runtime* like plain JavaScript. This lets tooling catch mistakes — like passing a string where a number is expected — before you ever open a browser or run `node`.
3. **A compiler, not a runtime**: there is no "TypeScript engine" that executes `.ts` files directly. The TypeScript compiler (`tsc`) reads your typed source, checks it for type errors, strips away all the type annotations, and emits ordinary `.js` files.
4. **Developed by Microsoft**: first released in 2012, TypeScript grew out of Microsoft's need to build and maintain very large JavaScript codebases (like early versions of the Office web apps) where plain JavaScript's lack of types made refactoring and collaboration error-prone.

```typescript
// This is valid TypeScript...
function add(a: number, b: number): number {
  return a + b;
}

console.log(add(2, 3));
// Output: 5

// ...but this line would fail to COMPILE (not just run):
// add("2", 3); // Error: Argument of type 'string' is not assignable to parameter of type 'number'.
```

---

### **Why Use TypeScript**

1. **Catch errors at compile time**: typos, wrong argument types, and mismatched object shapes are flagged by the compiler *before* the code runs, instead of surfacing as a runtime crash (or worse, silent wrong behavior) in production.
2. **Better IDE autocomplete and IntelliSense**: because the editor knows the exact shape of your variables, function parameters, and return values, it can offer accurate autocomplete, inline documentation, and "go to definition" — even for third-party libraries you've never opened the source of.
3. **Self-documenting code**: a function signature like `function createUser(name: string, age: number): User` tells you exactly what to pass and what you'll get back, without needing to read the implementation or hunt for external docs.
4. **Safer refactoring**: rename a property or change a function's signature, and the compiler immediately shows you every call site that's now broken — across the entire codebase — instead of you discovering it manually (or a user discovering it in production).
5. **Scales better on large teams and codebases**: as a project grows past a handful of files and contributors, an explicit contract between modules (what a function expects, what an object looks like) becomes far more valuable than in a small script, because no single person can hold the whole codebase's shape in their head.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

function greetUser(user: User): string {
  return `Hello, ${user.name}!`;
}

// The editor will immediately flag this — `age` doesn't exist on User,
// and `email` is missing:
// greetUser({ id: 1, name: "Alice", age: 30 });
```

---

### **How TypeScript Relates to JavaScript**

TypeScript is built directly on top of JavaScript rather than being a separate language:

- **Almost all valid `.js` is valid `.ts`**: you can rename a `.js` file to `.ts` and, in most cases, it will compile as-is (with `any` types inferred where nothing more specific is known). This is what makes TypeScript adoptable incrementally in existing JavaScript projects.
- **Types are a compile-time-only layer**: TypeScript's type annotations, interfaces, and generics exist purely to help the compiler (and you) reason about the code. They are **fully erased** when compiling to JavaScript — there is no `number` or `interface` at runtime, because JavaScript itself has no such concepts.
- **TypeScript compiles down to JavaScript your runtime already understands**: the compiler can be configured to output old ES5 syntax or modern ES2020+ syntax, but either way, the output is plain, type-free JavaScript.

```typescript
// input.ts
let age: number = 30;
let username: string = "eran";
```

```javascript
// output.js (after compiling) — notice the types are completely gone
let age = 30;
let username = "eran";
```

For a full side-by-side comparison of TypeScript and JavaScript — syntax differences, tooling, when to choose one over the other — see [Difference-Between-TypeScript-and-JavaScript.md](../JavaScript/Difference-Between-TypeScript-and-JavaScript.md) in the JavaScript folder. This file won't repeat that comparison; it focuses on getting TypeScript itself installed and running.

---

### **Installing TypeScript**

TypeScript ships as an npm package. There are two common ways to install it:

#### **Global Install**
Useful for quickly trying things out from any directory on your machine.
```bash
npm install -g typescript
tsc --version
# Output: Version 5.x.x
```

#### **As a Project devDependency (Recommended)**
Pins the exact compiler version per project, so everyone on the team (and CI) uses the same version.
```bash
npm install --save-dev typescript
npx tsc --version
# Output: Version 5.x.x
```

Installing it as a `devDependency` — rather than globally — is the standard approach for real projects, because it avoids "works on my machine" issues caused by different developers having different global `tsc` versions.

---

### **The `tsc` Compiler**

`tsc` (short for "TypeScript Compiler") is the command-line tool that type-checks your code and emits JavaScript. At its simplest, you point it at a file:

```bash
tsc greet.ts
```

This produces a `greet.js` file in the same directory, assuming there are no type errors. If there *are* type errors, `tsc` will still print them to the console — and by default, it will still emit the JavaScript output unless you configure it not to (via `noEmitOnError` in `tsconfig.json`).

```typescript
// greet.ts
function greet(name: string): string {
  return `Hello, ${name}!`;
}

console.log(greet("World"));
```

```bash
tsc greet.ts
node greet.js
# Output: Hello, World!
```

---

### **Running TypeScript Directly with `ts-node`**

During development, recompiling and then running a separate `.js` file every time is slow and repetitive. **ts-node** is a tool that compiles and executes TypeScript in one step, without leaving a `.js` file behind — handy for quick scripts, experiments, and some testing setups.

```bash
npm install --save-dev ts-node typescript
npx ts-node greet.ts
# Output: Hello, World!
```

Under the hood, `ts-node` still uses the TypeScript compiler to type-check and transpile your code; it just does so in memory, on the fly, instead of writing `.js` files to disk. It's excellent for local development but isn't typically used to run TypeScript in production — production deployments almost always compile ahead of time with `tsc` (or a bundler like esbuild/webpack) and ship plain `.js`.

---

### **A Minimal Walkthrough**

Let's write a small typed function, compile it, and inspect the generated JavaScript.

**Step 1 — Write the TypeScript source:**
```typescript
// math.ts
function calculateArea(width: number, height: number): number {
  return width * height;
}

const area = calculateArea(4, 5);
console.log(`The area is ${area}`);
```

**Step 2 — Compile it:**
```bash
tsc math.ts
```

**Step 3 — Inspect the generated `math.js`:**
```javascript
// math.js
function calculateArea(width, height) {
  return width * height;
}
var area = calculateArea(4, 5);
console.log("The area is ".concat(area));
```

**Step 4 — Run it:**
```bash
node math.js
# Output: The area is 20
```

Notice two things about the emitted `math.js`:
- The type annotations (`: number`) are entirely gone — they existed only for the compiler.
- Depending on your `tsconfig.json` target, `tsc` may also transform modern syntax (like template literals) into older, more broadly compatible equivalents.

---

### **A Note on `tsconfig.json`**

Real projects don't usually invoke `tsc` on a single file — they use a `tsconfig.json` file at the project root to configure how the compiler should behave: which JS version to target, which files to include, how strict the type-checking should be, where to output compiled files, and much more. Running `tsc` (with no filename) inside a directory containing a `tsconfig.json` compiles the whole project according to those settings.

```bash
tsc --init
# Generates a tsconfig.json with sensible defaults and helpful comments
```

The full depth of `tsconfig.json` — every option, strict-mode flags, module resolution strategies — is covered in [TypeScriptConfig.md](./TypeScriptConfig.md). For now, just know it exists and that `tsc --init` is how you generate a starting point.

---

### **The TypeScript Playground**

If you want to experiment with TypeScript without installing anything, the [TypeScript Playground](https://www.typescriptlang.org/play) is an in-browser editor that:

1. **Compiles TypeScript to JavaScript live**, showing you the emitted output side-by-side as you type.
2. **Type-checks in real time**, underlining errors exactly as your editor would.
3. **Lets you switch TypeScript versions and compiler options** from dropdown menus, so you can test how a snippet behaves under different settings (e.g., with `strict` mode on or off).
4. **Is shareable** — every snippet gets a URL, which is useful for asking questions online or sharing a minimal reproduction of a bug.

It's an excellent first stop before setting up a local project, and a fast way to sanity-check a type you're unsure about.

---

### **Best Practices**

- Install TypeScript as a `devDependency` per project rather than relying solely on a global install, so versions stay consistent across machines and CI.
- Use `ts-node` (or your bundler's dev server) for fast local iteration, but always compile ahead-of-time with `tsc` for anything you actually ship.
- Start new projects with `tsc --init` and adjust the generated `tsconfig.json` rather than writing one from scratch.
- Use the TypeScript Playground to quickly test an unfamiliar type or confirm how the compiler will emit a piece of syntax.
- Remember that types are erased at compile time — never write logic that depends on type information being available at runtime (e.g., you cannot `console.log(typeof someGenericType)`).
- Rename `.js` files to `.ts` incrementally in existing projects rather than attempting a big-bang rewrite; TypeScript is designed for gradual adoption.

---

### **Interview Questions**

**Q1. What is TypeScript, and how does it relate to JavaScript?**
TypeScript is a statically-typed superset of JavaScript developed by Microsoft. It adds optional type annotations and compiles down to plain JavaScript, meaning almost all valid JavaScript is also valid TypeScript, and every TypeScript program ultimately runs as ordinary JavaScript.

**Q2. Do TypeScript's types exist at runtime?**
No. Types are a compile-time-only construct used for static analysis and tooling. The TypeScript compiler strips all type annotations, interfaces, and type aliases away when emitting JavaScript — at runtime, there is no trace of them.

**Q3. What is `tsc`, and what does it do?**
`tsc` is the TypeScript compiler's command-line tool. It reads `.ts` files, type-checks them against the rules you've configured (often via `tsconfig.json`), reports any type errors, and emits corresponding `.js` files.

**Q4. What is the difference between using `tsc` and using `ts-node`?**
`tsc` compiles TypeScript to a JavaScript file on disk, which you then run separately (e.g., with `node`). `ts-node` compiles and executes TypeScript in a single step, in memory, without producing a persisted `.js` file — convenient for development, but not typically used for production builds.

**Q5. Why would a team choose TypeScript over plain JavaScript for a large project?**
Because static types catch a whole class of bugs at compile time, improve IDE autocomplete and documentation, and make large-scale refactoring far safer — the compiler flags every call site broken by a signature change, which is especially valuable when many developers touch the same codebase.

**Q6. Can you just rename a `.js` file to `.ts` and expect it to work?**
In most cases, yes — TypeScript is designed to accept the vast majority of valid JavaScript as-is, inferring `any` for anything it can't determine a more specific type for. Some edge cases (certain dynamic patterns, non-standard syntax) may need small adjustments.

**Q7. What is `tsconfig.json` used for?**
It's a configuration file at the root of a TypeScript project that controls compiler behavior — which files to include, which JS version to target, how strict type-checking should be, where output files go, and more. Running `tsc` with no arguments in a directory containing it compiles the whole project per those settings.

**Q8. How would you install TypeScript for a single project versus for your whole machine?**
For a single project: `npm install --save-dev typescript` (pins the version per project). For machine-wide use: `npm install -g typescript` (available anywhere, but version isn't pinned per project).

**Q9. What is the TypeScript Playground, and when would you use it?**
It's an in-browser editor at typescriptlang.org/play that compiles and type-checks TypeScript live, showing the emitted JavaScript side by side. It's useful for quickly testing a type without setting up a local project, and for sharing a minimal, shareable code snippet.

**Q10. If TypeScript code has a type error, will `tsc` still produce JavaScript output by default?**
Yes, by default `tsc` still emits the compiled `.js` file even when type errors are reported — it just prints the errors as warnings/diagnostics. This behavior can be changed with the `noEmitOnError` compiler option so that a build fails entirely on type errors.

**Q11. What does it mean to say TypeScript supports "gradual adoption"?**
It means you can introduce TypeScript into an existing JavaScript codebase incrementally — converting one file at a time from `.js` to `.ts` — rather than needing to rewrite the entire project at once. Files can even mix, using `allowJs` in `tsconfig.json`.

**Q12. Give an example of an error TypeScript would catch at compile time that plain JavaScript would only surface at runtime (or not at all).**
```typescript
function getFullName(user: { first: string; last: string }): string {
  return `${user.first} ${user.last}`;
}

// TypeScript flags this at compile time:
// getFullName({ first: "Ada" });
// Error: Property 'last' is missing
```
In plain JavaScript, calling the equivalent function with a missing `last` property wouldn't error immediately — it would silently produce `"Ada undefined"` at runtime.
