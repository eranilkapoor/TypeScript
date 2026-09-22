**Modules and Namespaces** are TypeScript's two mechanisms for organizing code across files and avoiding naming collisions. Modern TypeScript code almost always uses ES modules — the same `import`/`export` syntax as modern JavaScript — but TypeScript also has an older, TypeScript-specific system called namespaces, plus tooling for describing the types of plain JavaScript code that has no types of its own. Understanding all three, and knowing when (and when not) to reach for each, is important for working in both modern codebases and legacy ones.

---

### **ES Modules in TypeScript**

TypeScript's `import`/`export` syntax works exactly the same as modern JavaScript's ES modules — TypeScript doesn't invent a new module syntax, it just adds type annotations on top of the same JS you already know.

```typescript
// mathUtils.ts
export function add(a: number, b: number): number {
  return a + b;
}

export interface Point {
  x: number;
  y: number;
}
```

```typescript
// main.ts
import { add, Point } from "./mathUtils";

const origin: Point = { x: 0, y: 0 };
console.log(add(2, 3));
// Output: 5
```

TypeScript adds a couple of module-specific conveniences worth knowing:

1. **Type-only imports**: `import type { Point } from "./mathUtils";` explicitly imports only the type, which is guaranteed to be erased at compile time and never shows up in the emitted JavaScript.
2. **`import type` and `export type`**: useful for keeping type-only dependencies clearly separated from runtime dependencies, especially under bundler settings like `isolatedModules`.

```typescript
import type { Point } from "./mathUtils";
export type { Point };
```

For the full explanation of ES modules versus CommonJS, `require`/`module.exports`, default vs. named exports, and how bundlers resolve modules, see [../JavaScript/Modules.md](../JavaScript/Modules.md) — that content applies identically in TypeScript, since it's a JavaScript-level concept, not a type-level one.

---

### **Namespaces**

A **namespace** is TypeScript's original, pre-ES-modules way of grouping related code under a single named object, using the `namespace` keyword. It predates ES modules being standardized in JavaScript.

```typescript
namespace Geometry {
  export interface Point {
    x: number;
    y: number;
  }

  export function distance(a: Point, b: Point): number {
    return Math.sqrt((a.x - b.x) ** 2 + (a.y - b.y) ** 2);
  }
}

const p1: Geometry.Point = { x: 0, y: 0 };
const p2: Geometry.Point = { x: 3, y: 4 };

console.log(Geometry.distance(p1, p2));
// Output: 5
```

Only members marked `export` are accessible from outside the namespace; everything else stays private to it.

#### **Namespaces vs. ES Modules**

| Aspect | Namespaces | ES Modules |
|---|---|---|
| **Standard** | TypeScript-specific | Part of the JavaScript language |
| **File relationship** | Can span multiple files via triple-slash references | One file = one module |
| **Tooling support** | Limited (older bundlers, tree-shaking doesn't apply) | Excellent (native browser/Node support, tree-shaking) |
| **Typical use today** | Legacy codebases, global `<script>`-based apps, some `.d.ts` files for global libraries | Virtually all modern TypeScript/JavaScript projects |

#### **When Namespaces Are Still Used**
Namespaces are mostly a legacy feature today, but they still show up in a few places:
- **Global, non-bundled scripts**: old-school apps that load plain `<script>` tags without a bundler, where there's no module loader to resolve `import`/`export`.
- **Typing global libraries**: some `.d.ts` files for libraries that attach themselves to the global scope (like older jQuery plugins) use namespaces to describe the shape of that global object.
- **Merging with other declarations**: namespaces have a unique ability to merge with classes, functions, and enums (see below), which occasionally is still used deliberately.

For virtually all new code, **ES modules are strongly preferred** — they're the actual JavaScript standard, supported natively by Node.js and every modern bundler, and they enable tree-shaking (removing unused code from the final bundle), which namespaces don't support.

---

### **Declaration Merging**

TypeScript has a feature called **declaration merging**, where multiple declarations with the same name are automatically combined into one.

#### **Interfaces Merge Automatically**
```typescript
interface Animal {
  name: string;
}

interface Animal {
  age: number;
}

// Animal is now merged: { name: string; age: number }
const dog: Animal = { name: "Rex", age: 3 };
console.log(dog);
// Output: { name: 'Rex', age: 3 }
```

This is how libraries let you "extend" a built-in type — for example, augmenting the global `Window` interface, or a library's exported interface, without modifying its source.

#### **Merging a Namespace with a Class**
A namespace can merge with a class of the same name to attach static-like helpers or nested types to it.

```typescript
class Album {
  constructor(public title: string) {}
}

namespace Album {
  export function create(title: string): Album {
    return new Album(title);
  }
}

const record = Album.create("Abbey Road");
console.log(record.title);
// Output: Abbey Road
```

#### **Merging a Namespace with a Function**
```typescript
function buildLogger(prefix: string) {
  return (message: string) => console.log(`[${prefix}] ${message}`);
}

namespace buildLogger {
  export const version = "1.0.0";
}

console.log(buildLogger.version);
// Output: 1.0.0
```

#### **Merging a Namespace with an Enum**
```typescript
enum Color {
  Red,
  Green,
  Blue,
}

namespace Color {
  export function isWarm(color: Color): boolean {
    return color === Color.Red;
  }
}

console.log(Color.isWarm(Color.Red));
// Output: true
```

---

### **Ambient Declarations and `.d.ts` Files**

Not every JavaScript library ships with TypeScript types. **Ambient declarations** let you describe the shape of existing JavaScript code — variables, functions, modules — without providing an implementation, using the `declare` keyword. These typically live in `.d.ts` ("declaration") files.

#### **Why They're Needed**
When you `import` a plain JS library with no types, TypeScript either complains it can't find type information, or (depending on config) treats everything as `any`, losing type safety. A `.d.ts` file fills that gap.

#### **A Hand-Written `.d.ts` for an Untyped JS Module**
Suppose you have a small untyped JS module:

```javascript
// legacyMathLib.js
function square(n) {
  return n * n;
}
module.exports = { square };
```

You can describe its shape in a companion declaration file:

```typescript
// legacyMathLib.d.ts
declare module "legacyMathLib" {
  export function square(n: number): number;
}
```

Now consumers get full type checking and autocomplete when importing it:

```typescript
import { square } from "legacyMathLib";

console.log(square(5));
// Output: 25
```

#### **`declare` for Global Variables**
`declare` is also used to tell TypeScript about a value that exists at runtime (e.g. injected by a `<script>` tag) but isn't defined anywhere in your TypeScript code:

```typescript
declare const ANALYTICS_KEY: string; // provided globally by an external script

console.log(ANALYTICS_KEY.length);
```

---

### **DefinitelyTyped and `@types` Packages**

Most popular JavaScript libraries that don't ship their own types have community-maintained type definitions published on **DefinitelyTyped**, distributed as `@types/*` packages on npm.

```bash
npm install lodash
npm install --save-dev @types/lodash
```

Once installed, TypeScript automatically picks up the types from `node_modules/@types/lodash` — no extra configuration needed in most setups.

```typescript
import _ from "lodash";

const chunks = _.chunk([1, 2, 3, 4, 5], 2);
console.log(chunks);
// Output: [ [ 1, 2 ], [ 3, 4 ], [ 5 ] ]
```

A few things worth knowing:
1. **Only needed for JS libraries**: packages written in TypeScript already ship their own types (look for a `"types"` or `"typings"` field in their `package.json`), so no separate `@types` package is needed.
2. **Version alignment**: `@types` packages are versioned independently, so occasionally a types package can lag behind or slightly mismatch the library's actual runtime API.
3. **Not all libraries have one**: if no `@types` package exists and the library has no bundled types, you write your own ambient `.d.ts` declaration, as shown above.

---

### **Triple-Slash Reference Directives**

Before ES modules and modern module resolution, TypeScript used **triple-slash directives** — special comments — to tell the compiler about dependencies between files, most commonly to stitch together multi-file namespaces.

```typescript
/// <reference path="./geometry.ts" />

const p: Geometry.Point = { x: 1, y: 2 };
```

Triple-slash references are rarely needed in modern TypeScript projects, since ES module imports and the module resolution built into `tsc`/bundlers handle nearly all cases. You may still encounter them in older codebases, some global `.d.ts` files (referencing other ambient declaration files), or namespace-based legacy code.

---

### **Best Practices**
- Default to ES modules (`import`/`export`) for all new TypeScript code; reserve namespaces for legacy, non-bundled, or global-script scenarios.
- Use `import type`/`export type` for type-only imports and exports when your build tooling benefits from the distinction (e.g. under `isolatedModules`).
- Prefer installing a library's own bundled types over `@types/*` when both are available — bundled types stay in sync with the library by definition.
- Write your own `.d.ts` ambient declarations only when no official or community types exist for a JS dependency.
- Use declaration merging deliberately (e.g. augmenting a third-party library's interface) rather than as a general code organization strategy.
- Avoid triple-slash directives in new code; let ES module imports and your bundler's module resolution do the work instead.

---

### **Interview Questions**

**Q1. What's the difference between namespaces and ES modules in TypeScript?**
Namespaces are a TypeScript-specific way to group code under a named object, predating ES modules being standardized in JavaScript. ES modules are the actual JavaScript standard `import`/`export` syntax, natively supported by Node.js and bundlers, with tree-shaking support that namespaces lack.

**Q2. Why are ES modules preferred over namespaces in modern TypeScript projects?**
Because ES modules are a real JavaScript language feature with native runtime and bundler support, enable dead-code elimination via tree-shaking, and integrate cleanly with the rest of the JavaScript ecosystem. Namespaces are largely a legacy mechanism now used mainly for global scripts or certain `.d.ts` scenarios.

**Q3. What is declaration merging? Give an example.**
Declaration merging is TypeScript's behavior of automatically combining multiple declarations that share the same name into a single definition. The most common example is two `interface` declarations with the same name merging their members into one interface.
```typescript
interface Animal { name: string; }
interface Animal { age: number; } // merges into { name: string; age: number }
```

**Q4. Can a namespace merge with a class? What is that useful for?**
Yes — a namespace and a class with the same name merge, letting you attach static-like helper functions or nested types to the class without putting them inside the class body itself.

**Q5. What is an ambient declaration, and when do you need one?**
An ambient declaration (using the `declare` keyword) describes the shape of code that exists at runtime but has no TypeScript implementation available to the compiler — typically a plain JavaScript library or a global variable injected by an external script. You need one whenever TypeScript can't find type information for something you're using.

**Q6. What is a `.d.ts` file?**
A TypeScript declaration file that contains only type information — interfaces, function signatures, ambient module declarations — with no runtime implementation. It's how types are distributed separately from (or alongside) plain JavaScript code.

**Q7. What are DefinitelyTyped and `@types` packages?**
DefinitelyTyped is a large community-maintained repository of TypeScript type definitions for JavaScript libraries that don't ship their own types. Its definitions are published to npm as `@types/*` packages (e.g. `@types/lodash`), which TypeScript automatically picks up once installed.

**Q8. If a library is written in TypeScript, do you still need to install an `@types` package for it?**
No — libraries written in TypeScript typically compile and ship their own `.d.ts` files bundled with the package, so their types are already available without a separate `@types` install.

**Q9. What is a triple-slash reference directive, and is it still commonly used?**
It's a special comment (`/// <reference path="..." />`) that tells the TypeScript compiler about a dependency between files, historically used to stitch together multi-file namespaces. It's rarely needed in modern code, since ES module imports and standard module resolution cover nearly all use cases.

**Q10. How would you add types for a plain JavaScript module that has no types and no `@types` package available?**
Write your own ambient declaration file (`.d.ts`) using `declare module "moduleName" { ... }` to describe its exported functions and values, then place that file where TypeScript can find it (commonly alongside the source or in a `types`/`typings` directory).

**Q11. Why can't you have two ES modules in the same project export a `default` and expect them to "merge" the way two interfaces would?**
Declaration merging is a type-level, compile-time-only feature specific to certain declaration kinds (interfaces, namespaces with classes/functions/enums). Module exports are just JavaScript values/bindings; two separate module files with `export default` are entirely independent values, not something the type system merges together.

**Q12. In what scenario might you still choose to use a namespace in a brand-new TypeScript project today?**
Mainly when writing type declarations for a JavaScript library that attaches itself to the global scope (no module system at all), where a namespace is used purely at the type level in a `.d.ts` file to describe the shape of that global object — not as a way of organizing your own application code.
