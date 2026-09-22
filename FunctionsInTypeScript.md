**Functions** are where TypeScript's type system earns its keep the most — a well-typed function signature documents exactly what it needs and what it produces, catching mismatched calls before your code ever runs. This file covers how to type parameters, return values, optional and default parameters, function-typed variables, overloads, and the special handling of `this`.

---

### **Typing Parameters and Return Types**

Function parameters are annotated the same way variables are — a colon followed by the type. The return type is annotated after the parameter list.

```typescript
function add(a: number, b: number): number {
  return a + b;
}

console.log(add(2, 3));
// Output: 5

// add("2", 3); // Error: Argument of type 'string' is not assignable to parameter of type 'number'.
```

TypeScript can usually **infer** the return type from the function body, so writing `: number` above is technically optional here. But explicitly annotating return types — especially at **public API boundaries** — has real benefits:

1. **Intent is locked in**: if you accidentally change the implementation so it now returns something else, the compiler flags the mismatch immediately, instead of silently changing the function's public contract.
2. **Faster type-checking**: the compiler doesn't have to infer the return type by analyzing the whole function body, which matters in large codebases.
3. **Clearer documentation**: anyone reading the signature alone (in an editor tooltip, without opening the implementation) knows exactly what to expect back.

```typescript
// Return type inferred as `number` — works, but the intent is implicit
function square(n: number) {
  return n * n;
}

// Explicit return type — intent is unambiguous, and a bug in the
// body that returns the wrong type would be caught immediately
function cube(n: number): number {
  return n * n * n;
}
```

---

### **Optional Parameters and Default Parameters**

1. **Optional parameters (`?`)**: mark a parameter as not required. Optional parameters must come after all required parameters.
2. **Default parameters**: provide a fallback value used when the caller omits the argument (or passes `undefined`). Default parameters are automatically treated as optional.

```typescript
// Optional parameter
function greet(name: string, title?: string): string {
  return title ? `Hello, ${title} ${name}` : `Hello, ${name}`;
}

console.log(greet("Smith"));
// Output: Hello, Smith
console.log(greet("Smith", "Dr."));
// Output: Hello, Dr. Smith
```

```typescript
// Default parameter
function createUser(name: string, role: string = "member"): string {
  return `${name} joined as ${role}`;
}

console.log(createUser("Alice"));
// Output: Alice joined as member
console.log(createUser("Bob", "admin"));
// Output: Bob joined as admin
```

#### **Rest Parameters with Types**

Rest parameters collect any number of remaining arguments into a typed array.

```typescript
function sum(...numbers: number[]): number {
  return numbers.reduce((total, n) => total + n, 0);
}

console.log(sum(1, 2, 3, 4));
// Output: 10
```

---

### **Function Type Expressions**

A **function type expression** describes the shape of a function — its parameter types and return type — so you can type a variable that *holds* a function, without necessarily implementing it right there.

```typescript
let combine: (a: number, b: number) => number;

combine = (a, b) => a + b; // parameter types are inferred from `combine`'s declared type
console.log(combine(3, 4));
// Output: 7

// combine = (a: string, b: string) => a + b; // Error: doesn't match the declared function type
```

This is especially useful for typing callback parameters:

```typescript
function processArray(arr: number[], callback: (item: number) => void): void {
  arr.forEach(callback);
}

processArray([1, 2, 3], (n) => console.log(n * 10));
// Output:
// 10
// 20
// 30
```

You can also name a function type with a `type` alias for reuse:

```typescript
type MathOperation = (a: number, b: number) => number;

const multiply: MathOperation = (a, b) => a * b;
const divide: MathOperation = (a, b) => a / b;

console.log(multiply(4, 5));
// Output: 20
console.log(divide(10, 2));
// Output: 5
```

---

### **Function Overloads**

**Function overloads** let a single function accept several distinct combinations of parameter types and return types, each declared as its own call signature above the actual implementation. Callers see only the overload signatures; the implementation signature itself is not directly callable.

```typescript
// Overload signatures — describe the valid ways to call the function
function makeDate(timestamp: number): Date;
function makeDate(year: number, month: number, day: number): Date;

// Implementation signature — must be compatible with every overload above,
// and is not visible to callers
function makeDate(yearOrTimestamp: number, month?: number, day?: number): Date {
  if (month !== undefined && day !== undefined) {
    return new Date(yearOrTimestamp, month - 1, day);
  }
  return new Date(yearOrTimestamp);
}

const d1 = makeDate(1_700_000_000_000);
const d2 = makeDate(2024, 3, 15);

console.log(d1 instanceof Date, d2 instanceof Date);
// Output: true true

// makeDate(2024, 3); // Error: no overload matches this call (missing `day`)
```

Overloads are useful when a function's behavior (and return type) genuinely changes shape based on *which* combination of arguments is passed — something a single signature with optional parameters can't express precisely. If the parameter types can simply be described with a union, prefer that instead of overloads, since it's usually simpler.

```typescript
// A union-based alternative is often simpler when the shapes don't diverge much:
function logValue(value: string | number): void {
  console.log(value);
}
```

---

### **Typing `this` in a Function**

When a regular (non-arrow) function is used as a method or callback, `this` depends on *how* the function is called, which can lead to subtle bugs. TypeScript lets you declare an explicit, fake **`this` parameter** as the first parameter in a function's signature — it's erased at compile time and exists purely so the compiler can check that `this` is used correctly.

```typescript
interface Button {
  label: string;
  onClick: (this: Button) => void;
}

const button: Button = {
  label: "Submit",
  onClick: function (this: Button) {
    console.log(`${this.label} was clicked`);
  },
};

button.onClick();
// Output: Submit was clicked

// const detachedClick = button.onClick;
// detachedClick(); // Error: The 'this' context of type 'void' is not assignable to method's 'this' of type 'Button'.
```

Because the `this` parameter is a compile-time-only annotation (never passed as a real argument), it must always be listed **first**, before any real parameters.

```typescript
function reportSize(this: HTMLElement, label: string): void {
  console.log(`${label}: ${this.clientWidth}px`);
}
```

In practice, arrow functions (which lexically capture the enclosing `this` and never rebind it) sidestep most of these issues, which is why explicit `this` parameters are seen far less often than in older, callback-heavy codebases.

---

### **`void` vs. `never` as Return Types, Revisited**

In the context of functions specifically:

| Return type | Meaning | Example |
|---|---|---|
| `void` | The function completes normally but doesn't return a useful value | Event handlers, logging functions |
| `never` | The function **never** completes normally — it always throws or loops forever | Assertion/error-throwing helpers, infinite loops |

```typescript
function logAction(action: string): void {
  console.log(`Action: ${action}`);
}

function assertIsPositive(n: number): asserts n is number {
  if (n <= 0) {
    throw new Error("Expected a positive number");
  }
}

function failFast(message: string): never {
  throw new Error(message);
}
```

A function typed `void` is allowed to fall off the end without a `return` (implicitly returning `undefined`); a function typed `never` is not allowed to do so at all — every code path must either throw or loop forever, and the compiler checks this.

---

### **Best Practices**

- Explicitly annotate return types on exported/public functions, even when TypeScript could infer them — it locks in intent and catches accidental signature drift.
- Put optional parameters after all required ones, and prefer default parameters over manually checking `=== undefined` inside the function body.
- Use rest parameters (`...args: T[]`) instead of an `arguments`-based approach for variable-length argument lists — it's fully typed and works with arrow functions.
- Reach for function overloads only when a function's return type or behavior genuinely varies by which argument combination is used; otherwise a union parameter type is usually simpler and easier to maintain.
- Prefer arrow functions for callbacks where you want `this` to stay bound to the enclosing context, rather than fighting with explicit `this` parameters.
- Use `never` return types for helper functions that always throw, so the compiler can use that information for exhaustiveness and unreachable-code analysis at call sites.

---

### **Interview Questions**

**Q1. Why would you explicitly annotate a function's return type even though TypeScript can infer it?**
Explicit return types lock in the function's intended public contract — if the implementation changes and accidentally returns a different type, the compiler flags the mismatch immediately. They also make the signature self-documenting without needing to read the function body.

**Q2. What is the difference between an optional parameter and a default parameter?**
An optional parameter (`name?: string`) may be omitted entirely, and its value is `undefined` if not passed. A default parameter (`name: string = "Guest"`) supplies a fallback value automatically used when the argument is omitted or explicitly `undefined`; it's implicitly optional as well.

**Q3. How do you type a function that accepts a variable number of numeric arguments?**
Using a rest parameter: `function sum(...numbers: number[]): number { ... }`. TypeScript treats `numbers` as a fully typed array inside the function body.

**Q4. What is a function type expression? Give an example.**
It's a type describing a function's parameter types and return type, used to type a variable or parameter that holds a function.
```typescript
let compare: (a: number, b: number) => boolean;
compare = (a, b) => a > b;
```

**Q5. What are function overloads, and when should you use them?**
Overloads let you declare multiple call signatures for one function implementation, each describing a different valid combination of argument and return types. Use them when a function's return type or behavior genuinely depends on which argument shapes are passed — not merely when the type is a simple union.

**Q6. In a set of overloaded functions, which signature is actually callable, and which one contains the logic?**
Only the overload signatures (declared above the implementation) are visible and callable by consumers. The final implementation signature contains the actual logic and must be compatible with every overload, but it is not itself part of the public call signatures.

**Q7. What is the purpose of an explicit `this` parameter in a function's type signature?**
It lets the compiler check that a function is called with the correct `this` context — for example, as a method on a specific object type — flagging calls where `this` would be wrong or missing. It's a compile-time-only annotation and is always the first parameter in the signature.

**Q8. Why do arrow functions reduce the need for explicit `this` parameters?**
Arrow functions don't have their own `this` — they lexically inherit `this` from the enclosing scope at the point they're defined, rather than depending on how they're called. This avoids the common bug where a regular function's `this` becomes `undefined` or incorrect when detached from its original object and called elsewhere.

**Q9. What is the difference between a function returning `void` and one returning `never`?**
A `void` function completes normally without returning a meaningful value (implicitly `undefined`). A `never` function never completes normally at all — every code path must throw an exception or loop forever; falling off the end of the function is a compile error.

**Q10. If a function has both required and optional parameters, in what order must they appear?**
All required parameters must come before any optional ones. TypeScript does not allow an optional parameter to precede a required parameter in the same signature.

**Q11. How does TypeScript help catch an incorrect number of arguments passed to a function?**
The compiler checks every call site against the function's declared parameter list (including which parameters are optional or have defaults) and raises a compile-time error if too few required arguments, or too many total arguments, are passed.

**Q12. Would you use overloads or a union type for a function `parseInput` that accepts either a `string` or a `number`, and always returns a `string`?**
A union type parameter is simpler and sufficient here, since the return type doesn't change based on the input type: `function parseInput(input: string | number): string`. Overloads are worth the extra verbosity only when the return type (or behavior) genuinely differs per input shape.
