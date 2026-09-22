**Types and type annotations** are the core of what makes TypeScript, TypeScript. A type describes the *shape* of a value — what operations are valid on it, what values it can hold — and an annotation is how you explicitly tell the compiler what type something is, rather than letting it figure that out on its own. Understanding when TypeScript can infer a type for you versus when you need to spell it out is the foundation for everything else in the language.

---

### **Type Inference vs. Explicit Type Annotation**

TypeScript doesn't require you to annotate every single variable — it has a powerful **type inference** engine that looks at how a variable is initialized and figures out its type automatically.

1. **Inference**: when you initialize a variable with a value, TypeScript infers the type from that value and "locks it in" from then on.
2. **Explicit annotation**: you write the type yourself using a colon (`:`) followed by the type, which is required (or at least strongly recommended) when TypeScript has no initial value to infer from.

```typescript
// Inference — TypeScript figures out `age` is a `number`
let age = 30;
// age = "thirty"; // Error: Type 'string' is not assignable to type 'number'.

// Explicit annotation — needed when there's no initializer to infer from
let username: string;
username = "eran";
```

#### **When You Must Annotate**

- **Function parameters** — TypeScript cannot infer these from usage alone; they always need explicit types (or default to `any` in non-strict mode).
- **Variables declared without an initial value.**
- **Empty arrays or objects** where the eventual contents aren't obvious from context (e.g., `let items = []` infers `any[]`, which defeats the purpose of typing).

```typescript
// Parameters must be annotated — TypeScript can't guess intent
function multiply(a: number, b: number): number {
  return a * b;
}
```

**Rule of thumb**: let inference do the work for local variables with obvious initializers, and annotate explicitly at function boundaries (parameters and return types) and anywhere the intended type isn't obvious from the immediate context.

---

### **Primitive Types**

TypeScript's primitive types mirror JavaScript's own primitives:

1. **`string`**: textual data, written with single quotes, double quotes, or template literals.
2. **`number`**: all numbers — integers and floats — TypeScript does not distinguish between them (there's no separate `int` or `float`).
3. **`boolean`**: `true` or `false`.

```typescript
let city: string = "Bengaluru";
let temperature: number = 28.5;
let isRaining: boolean = false;

console.log(`${city}: ${temperature}°C, raining: ${isRaining}`);
// Output: Bengaluru: 28.5°C, raining: false
```

---

### **`any`: The Escape Hatch**

`any` tells the compiler "stop checking this value's type — treat it as anything." Any operation is allowed on an `any`-typed value, and it can be assigned to or from any other type without complaint.

```typescript
let value: any = 4;
value = "now a string";
value = { anything: "goes" };
value.foo.bar.baz(); // No compile-time error, even though this will crash at runtime
```

`any` effectively opts a value out of TypeScript's type system entirely. It's sometimes necessary — migrating a large JS codebase, or interfacing with a truly dynamic third-party value — but overusing it defeats the entire purpose of using TypeScript, since it silently disables the compiler's error-catching for anything that value touches.

---

### **`unknown`: A Safer Alternative to `any`**

`unknown` also represents "a value of any type," but unlike `any`, TypeScript **won't let you use it** until you've narrowed it down to a more specific type first. This makes `unknown` the type-safe way to represent "I don't know this value's type yet."

```typescript
let valueA: any = "hello";
let valueB: unknown = "hello";

console.log(valueA.toUpperCase()); // Allowed (but unsafe — no compile-time check)
// console.log(valueB.toUpperCase()); // Error: Object is of type 'unknown'.

// You must narrow `unknown` before using it:
if (typeof valueB === "string") {
  console.log(valueB.toUpperCase());
  // Output: HELLO
}
```

| Feature | `any` | `unknown` |
|---|---|---|
| **Assignable from any type** | Yes | Yes |
| **Assignable to other types without a check** | Yes (unsafe) | No — must narrow first |
| **Can call methods/access properties directly** | Yes (unsafe) | No — compiler error until narrowed |
| **Type safety** | None | Full, once narrowed |
| **Typical use case** | Escape hatch, legacy migration | Values from external/untrusted sources (API responses, `JSON.parse`) |

**Rule of thumb**: prefer `unknown` over `any` whenever you're modeling "a value whose type isn't known yet" — it forces you (or whoever calls your code) to check before using it, which is exactly the safety TypeScript is meant to provide.

---

### **`never`: Values That Never Occur**

`never` represents a value that **can never happen**. It shows up in two common situations:

1. **Functions that never return normally** — because they always throw, or loop forever.
2. **Exhaustiveness checking** — proving, at compile time, that every possible case of a union has been handled.

```typescript
// A function that always throws never returns a value
function throwError(message: string): never {
  throw new Error(message);
}

// A function with an infinite loop also never returns
function infiniteLoop(): never {
  while (true) {}
}
```

```typescript
type Shape = "circle" | "square" | "triangle";

function describeShape(shape: Shape): string {
  switch (shape) {
    case "circle":
      return "Round";
    case "square":
      return "Four equal sides";
    case "triangle":
      return "Three sides";
    default:
      // If a new Shape variant is ever added without updating this switch,
      // `shape` here would NOT be `never`, and TypeScript raises a compile error.
      const exhaustiveCheck: never = shape;
      return exhaustiveCheck;
  }
}
```

This exhaustiveness pattern is a common technique in real codebases: it turns "forgot to handle a new case" into a compile-time error instead of a silent runtime bug.

---

### **`void`: No Meaningful Return Value**

`void` describes the return type of a function that doesn't return a meaningful value — most commonly, functions run purely for their side effects.

```typescript
function logMessage(message: string): void {
  console.log(message);
  // no return statement (or a bare `return;`) is expected
}

logMessage("Hello");
// Output: Hello
```

`void` and `never` are easy to confuse: a `void` function *can* still return (just without a useful value, effectively `undefined`), while a `never` function must **never** reach the end of its execution at all.

| Type | Meaning | Can the function return? |
|---|---|---|
| `void` | No meaningful value is returned | Yes — just returns `undefined` |
| `never` | The function never completes normally | No — always throws or loops forever |

---

### **Literal Types**

A **literal type** narrows a primitive down to one exact value (or a small fixed set of them, via a union). Instead of "any string," you can say "only this exact string."

```typescript
let direction: "up" | "down";
direction = "up";     // OK
// direction = "left"; // Error: Type '"left"' is not assignable to type '"up" | "down"'.

let diceRoll: 1 | 2 | 3 | 4 | 5 | 6;
diceRoll = 4; // OK
// diceRoll = 7; // Error
```

Literal types are especially useful for modeling a fixed set of valid string "modes" or "statuses," giving you autocomplete and compile-time validation that a plain `string` never could.

```typescript
function move(direction: "up" | "down" | "left" | "right"): void {
  console.log(`Moving ${direction}`);
}

move("up");
// Output: Moving up
```

---

### **Type Aliases with `type`**

The `type` keyword creates a reusable name for any type — a primitive, a union, an object shape, or anything else. It doesn't create a new type; it just gives an existing type description a name you can reuse.

```typescript
type Direction = "up" | "down" | "left" | "right";
type ID = string | number;

function move(direction: Direction): void {
  console.log(`Moving ${direction}`);
}

function findById(id: ID): void {
  console.log(`Looking up ID: ${id}`);
}

move("left");
// Output: Moving left
findById(42);
// Output: Looking up ID: 42
```

Aliasing a union like `Direction` avoids repeating the same list of literals everywhere it's used, and gives it a meaningful name that documents intent.

---

### **Union Types (`|`) and Intersection Types (`&`) — A Quick Introduction**

1. **Union types (`|`)**: a value that can be **one of several** types. TypeScript only lets you use operations that are valid on *every* member of the union, unless you narrow it first.
2. **Intersection types (`&`)**: a value that must satisfy **all** of several types at once — combining multiple type shapes into one.

```typescript
// Union: id can be either a string or a number
function printId(id: string | number): void {
  console.log(`ID: ${id}`);
}
printId(101);
// Output: ID: 101
printId("A-101");
// Output: ID: A-101

// Intersection: combines two object shapes into one that has all their properties
type HasName = { name: string };
type HasAge = { age: number };
type Person = HasName & HasAge;

const person: Person = { name: "Asha", age: 29 };
console.log(person);
// Output: { name: 'Asha', age: 29 }
```

This is only a basic introduction. Deeper patterns — narrowing a union with type guards, discriminated unions with a shared "tag" property, and combining unions/intersections with generics — are covered in depth in [AdvancedTypes.md](./AdvancedTypes.md).

---

### **`null` and `undefined` in TypeScript**

JavaScript has two "absence of value" types: `undefined` (a variable that's been declared but not assigned) and `null` (an intentional "no value"). TypeScript tracks both as their own types.

```typescript
let notAssignedYet: undefined = undefined;
let noValue: null = null;
```

#### **The `strictNullChecks` Compiler Flag**

By default (without strict mode), TypeScript allows `null` and `undefined` to be assigned to *any* type — which reintroduces the classic "cannot read property of undefined" bug that types are supposed to prevent. Turning on `strictNullChecks` (included automatically when `strict: true` is set) makes `null` and `undefined` only assignable to variables explicitly typed to allow them.

```typescript
// With strictNullChecks enabled:
let name: string = "Priya";
// name = null; // Error: Type 'null' is not assignable to type 'string'.

let maybeName: string | null = "Priya";
maybeName = null; // OK — null is explicitly part of the type
```

This flag (and the rest of the `strict` family of compiler options) is covered in full detail in [TypeScriptConfig.md](./TypeScriptConfig.md). For now, the key takeaway is: enabling `strictNullChecks` is one of the highest-value things you can do to catch real bugs, and most production TypeScript codebases enable it.

---

### **Best Practices**

- Let TypeScript infer types for local variables with obvious initializers; reserve explicit annotations for function parameters, return types at public boundaries, and cases where inference would be too broad (e.g., empty arrays).
- Avoid `any` wherever possible — treat it as a last resort, not a default when you're unsure of a type.
- Prefer `unknown` over `any` for values of genuinely unknown origin (API responses, `JSON.parse` results), and narrow before using them.
- Use `never` for exhaustiveness checks in `switch` statements over unions, so adding a new case without handling it becomes a compile error.
- Reach for literal type unions (`"up" | "down"`) instead of a plain `string` whenever a value only ever takes a small, fixed set of valid values.
- Enable `strictNullChecks` (ideally via `strict: true`) on every real project — it catches a huge share of the null/undefined bugs that plague untyped JavaScript.

---

### **Interview Questions**

**Q1. What is the difference between type inference and explicit type annotation?**
Inference is when TypeScript automatically determines a variable's type based on its initial value, without you writing the type out. Explicit annotation is when you write the type yourself using a colon syntax (`let x: number`), which is required in places TypeScript has nothing to infer from, like function parameters.

**Q2. What is the difference between `any` and `unknown`?**
Both represent "a value of any type," but `any` disables type checking entirely — you can call any method or access any property on it without error. `unknown` requires you to narrow the type (e.g., with `typeof`) before you can use it in any meaningful way, preserving type safety.
```typescript
let a: unknown = "hi";
// a.toUpperCase(); // Error until narrowed
if (typeof a === "string") a.toUpperCase(); // OK
```

**Q3. When would a function's return type be `never` instead of `void`?**
`never` is used when a function never completes normally at all — it always throws an exception or loops forever. `void` is for functions that complete normally but don't return a meaningful value.

**Q4. What is a literal type? Give an example.**
A literal type restricts a value to one exact value (or a small union of exact values) rather than the whole primitive type. For example, `let status: "loading" | "success" | "error"` only allows those three exact strings, not any arbitrary string.

**Q5. What does the `type` keyword do in TypeScript?**
It creates a named alias for a type — which can be a primitive, union, intersection, object shape, tuple, or anything else — so that type can be reused by name instead of rewritten everywhere it's needed.

**Q6. What is the difference between a union type and an intersection type?**
A union (`A | B`) means a value can be *either* type A or type B. An intersection (`A & B`) means a value must satisfy *both* type A and type B simultaneously — typically used to merge multiple object shapes into one.

**Q7. What does `strictNullChecks` do, and why is it important?**
It's a compiler flag that, when enabled, prevents `null` and `undefined` from being assignable to types that don't explicitly include them. Without it, any variable can silently be `null` or `undefined`, reintroducing the classic "cannot read property of undefined" runtime error that types are meant to prevent.

**Q8. Why is overusing `any` considered bad practice?**
Because `any` completely opts a value out of TypeScript's type checking — any property access, method call, or assignment on it is allowed without validation. Overusing it means large portions of a "typed" codebase get none of the actual safety benefits TypeScript provides.

**Q9. How would you safely use a value typed as `unknown`?**
By narrowing it first, using a type guard such as `typeof`, `instanceof`, a custom type-predicate function, or a runtime validation library, before performing any operation on it.
```typescript
function process(value: unknown) {
  if (typeof value === "number") {
    return value * 2;
  }
  return null;
}
```

**Q10. What's an example of using `never` for exhaustiveness checking?**
Assigning the remaining value in a `switch`'s `default` case to a variable typed `never`. If every case of a union has been handled, TypeScript narrows the remaining type to `never` and the assignment compiles; if a new union member is added without a matching case, the assignment fails to compile, flagging the gap.

**Q11. Is there a difference between `number` and things like `int` or `float` in TypeScript?**
No — TypeScript has a single `number` type covering all numeric values, integers and floating-point alike. It does not distinguish between different numeric subtypes the way some other statically-typed languages do.

**Q12. When should you choose `unknown` over a union of specific types?**
`unknown` is appropriate when a value's type genuinely cannot be predicted ahead of time, like the parsed result of arbitrary JSON or a value from an untyped third-party API. If the value can only realistically be one of a small, known set of types, a union of those specific types is more precise and useful than `unknown`.
