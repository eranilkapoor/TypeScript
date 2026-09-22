**Arrays, tuples, and enums** are TypeScript's tools for describing collections of values with more precision than plain JavaScript allows. Arrays let you type a list of same-shaped items, tuples let you type a fixed-length list where each position has its own specific type, and enums let you give meaningful names to a fixed set of related constant values. Together they let you model structured, ordered, or categorical data in a way the compiler can verify.

---

### **Typing Arrays**

TypeScript offers two equivalent syntaxes for array types:

1. **`number[]` syntax**: the more common, concise form — a type followed by `[]`.
2. **`Array<number>` syntax**: the generic form, functionally identical, sometimes preferred for readability with more complex element types.

```typescript
let scores: number[] = [90, 85, 77];
let names: Array<string> = ["Alice", "Bob", "Carol"];

// scores.push("100"); // Error: Argument of type 'string' is not assignable to parameter of type 'number'.
scores.push(100); // OK

console.log(scores);
// Output: [ 90, 85, 77, 100 ]
```

Both forms behave identically; `Array<T>` can read more clearly when `T` itself is already a complex type (e.g., `Array<{ id: number; name: string }>` vs. `{ id: number; name: string }[]`, where the latter can be visually ambiguous about what the `[]` applies to).

#### **Arrays of Union Types**

An array can hold values of more than one type by using a union inside the array type.

```typescript
let mixed: (string | number)[] = ["a", 1, "b", 2];

mixed.forEach((item) => {
  console.log(typeof item);
});
// Output:
// string
// number
// string
// number
```

Note the parentheses: `(string | number)[]` means "an array where each element is a string or a number." Without the parentheses, `string | number[]` would mean "either a plain string, or an array of numbers" — a very different type.

#### **Readonly Arrays**

A `readonly` array (or `ReadonlyArray<T>`) prevents any mutating operations — `push`, `pop`, `splice`, index assignment — after creation, while still allowing you to read from it.

```typescript
const fixedScores: readonly number[] = [10, 20, 30];
// fixedScores.push(40); // Error: Property 'push' does not exist on type 'readonly number[]'.
// fixedScores[0] = 99;  // Error: Index signature in type 'readonly number[]' only permits reading.

console.log(fixedScores[0]);
// Output: 10
```

```typescript
// Equivalent generic form
const fixedNames: ReadonlyArray<string> = ["Alice", "Bob"];
```

---

### **Tuples**

A **tuple** is a fixed-length array where each position has its own specific, known type — unlike a regular array, whose elements are all constrained to the *same* type (or union of types).

```typescript
// A tuple representing [name, age]
let person: [string, number];
person = ["Alice", 30]; // OK
// person = [30, "Alice"]; // Error: wrong order/types
// person = ["Alice", 30, true]; // Error: too many elements

console.log(`${person[0]} is ${person[1]} years old`);
// Output: Alice is 30 years old
```

Tuples are commonly used for small, fixed-shape data like coordinate pairs, key-value pairs, or (famously) the return value of React's `useState` hook (`[value, setValue]`).

#### **Optional Tuple Elements**

A `?` after a type marks that tuple position as optional — it can be omitted, but only from the end.

```typescript
let coordinate: [number, number, number?];
coordinate = [10, 20];       // OK — z is omitted
coordinate = [10, 20, 30];   // OK — z is provided

console.log(coordinate);
// Output: [ 10, 20 ]
```

#### **Labeled Tuple Elements**

Labels don't change the runtime behavior, but they make tuples dramatically more self-documenting — especially useful for function signatures and editor tooltips.

```typescript
let httpResponse: [status: number, message: string];
httpResponse = [200, "OK"];

console.log(`${httpResponse[0]}: ${httpResponse[1]}`);
// Output: 200: OK
```

#### **Rest Elements in Tuples**

A tuple can have a fixed set of leading elements followed by a rest element (`...Type[]`) capturing any number of remaining elements of a given type.

```typescript
// A tuple with a required first element (the command),
// followed by any number of string arguments
let command: [string, ...string[]];
command = ["deploy"];
command = ["deploy", "--env", "production"];

console.log(command);
// Output: [ 'deploy', '--env', 'production' ]
```

---

### **Enums**

An **enum** ("enumerated type") gives a friendly name to a fixed set of related constant values — useful for representing a closed set of options like directions, statuses, or roles.

#### **Numeric Enums**

By default, enum members are auto-assigned increasing numbers starting at `0`.

```typescript
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right, // 3
}

let move: Direction = Direction.Up;
console.log(move);
// Output: 0
console.log(Direction[0]);
// Output: Up (numeric enums support this "reverse mapping" from value back to name)
```

You can also set a custom starting value — subsequent members auto-increment from there.

```typescript
enum StatusCode {
  Success = 200,
  NotFound = 404,
  ServerError = 500,
}

console.log(StatusCode.NotFound);
// Output: 404
```

```typescript
enum Level {
  Low = 1,
  Medium,  // 2 (auto-incremented from Low)
  High,    // 3
}

console.log(Level.Medium);
// Output: 2
```

#### **String Enums**

Every member is explicitly assigned a string value. Unlike numeric enums, string enums have **no** auto-increment and **no** reverse mapping — but their values are far more meaningful when logged, serialized, or debugged.

```typescript
enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}

console.log(Direction.Up);
// Output: UP
```

#### **Const Enums**

Prefixing an enum with `const` tells the compiler to **fully inline** its values at every usage site and generate no actual enum object in the compiled JavaScript — improving runtime performance and reducing bundle size.

```typescript
const enum Direction {
  Up,
  Down,
}

let move = Direction.Up;
```

Compiles to (roughly):
```javascript
// The enum object itself never exists at runtime —
// Direction.Up is replaced directly with 0 wherever it's used.
let move = 0 /* Up */;
```

Because const enums are erased entirely, they can't be used with certain features that need a real runtime object (like iterating over enum members), and they can cause issues across project/module boundaries with some build tools — so use them primarily for simple, internal, performance-sensitive cases.

#### **When to Prefer a Union of String Literals Over an Enum**

In modern TypeScript, many style guides recommend a union of string literals instead of an enum for representing a fixed set of options:

```typescript
// Enum approach
enum Status {
  Loading = "LOADING",
  Success = "SUCCESS",
  Error = "ERROR",
}

// Union of string literals — often preferred
type Status = "LOADING" | "SUCCESS" | "ERROR";

function setStatus(status: Status): void {
  console.log(`Status is now: ${status}`);
}

setStatus("SUCCESS");
// Output: Status is now: SUCCESS
```

| Feature | Enum | Union of String Literals |
|---|---|---|
| **Runtime footprint** | Generates a real JS object (unless `const`) | Zero — fully erased, just a `type` |
| **Works with plain string values** | Requires accessing via `Enum.Member` | Can pass raw strings directly |
| **Reverse mapping (value → name)** | Yes, for numeric enums | No — not applicable |
| **Interop with plain JS / JSON** | Can be awkward (numeric enums especially) | Seamless — it's just strings |
| **Tooling support / familiarity** | Well known across many languages | TypeScript-idiomatic, common in modern codebases |

**Rule of thumb**: reach for a string literal union by default for simple "one of these fixed options" cases — it has zero runtime cost and works naturally with plain JSON and JavaScript. Reach for an enum when you specifically want a namespaced, referenceable set of related constants (e.g., `Direction.Up` reads clearly as a grouped concept) or need numeric auto-incrementing values.

---

### **Best Practices**

- Prefer `T[]` syntax for simple element types and `Array<T>` when the element type itself is complex or would read ambiguously with `[]`.
- Use `readonly` arrays and tuples for data that should never be mutated after creation, such as configuration values or function parameters you don't want accidentally modified.
- Reach for tuples when position genuinely carries meaning (like `[x, y]` coordinates or `[status, message]` pairs) rather than for general lists — use a regular array or an object for anything else.
- Add labels to tuple elements (`[status: number, message: string]`) to make their meaning clear in editor tooltips and signatures.
- Default to string literal unions over enums for simple fixed-option sets; reach for enums when you want a real, namespaced, referenceable grouping or auto-incrementing numeric values.
- Be cautious with numeric enums crossing serialization boundaries (like APIs or databases) — the underlying number can shift meaning if members are reordered; string enums or literal unions avoid this pitfall.

---

### **Interview Questions**

**Q1. What is the difference between `number[]` and `Array<number>`?**
They are functionally identical — both describe an array of numbers. `number[]` is the more concise, commonly used shorthand syntax, while `Array<number>` is the equivalent generic form, sometimes preferred for readability when the element type is more complex.

**Q2. What is a tuple, and how does it differ from a regular array?**
A tuple is a fixed-length array where each position has its own specific type, so the type at index 0 can differ from the type at index 1, and so on. A regular array has a single element type (or union) that applies uniformly across all elements, and its length isn't fixed by the type.

**Q3. How do you type an array that can contain both strings and numbers?**
```typescript
let mixed: (string | number)[] = ["a", 1];
```
The parentheses around the union are required — without them, `string | number[]` would mean "a string, or an array of numbers," which is a different type entirely.

**Q4. What does `readonly` do when applied to an array type?**
It prevents any mutating operation on the array after creation — `push`, `pop`, `splice`, and direct index assignment all become compile errors — while still allowing you to read elements from it.

**Q5. What is the default behavior of a numeric enum's values?**
Members are auto-assigned increasing integers starting at `0`, unless you assign an explicit value to one member — in which case subsequent members auto-increment from that value.

**Q6. What is a const enum, and why would you use one?**
A `const enum` is fully inlined by the compiler at every usage site and produces no actual JavaScript object at runtime, reducing bundle size and improving performance. It's used when you want the convenience of enum syntax without the runtime cost of a real enum object.

**Q7. What is the difference between a numeric enum and a string enum?**
A numeric enum auto-increments integer values by default and supports reverse mapping (looking up a member's name from its value). A string enum requires every member to have an explicit string value, has no auto-increment, and has no reverse mapping — but its values are far more readable when logged or serialized.

**Q8. Why might you prefer a union of string literals over an enum?**
String literal unions have zero runtime footprint (they're erased entirely, unlike a standard enum which generates a real object), and they interoperate seamlessly with plain strings from JSON or JavaScript, without needing to access values via `Enum.Member`.

**Q9. How do you make a tuple element optional?**
By adding a `?` after its type, and only at the end of the tuple — for example, `[number, number, number?]` allows either two or three elements.

**Q10. What are labeled tuple elements, and do they affect runtime behavior?**
Labeled tuple elements add descriptive names to each position (e.g., `[status: number, message: string]`), improving editor tooltips and documentation. They have no effect on runtime behavior — the tuple is still just an array — the labels exist purely for type-checking and tooling.

**Q11. How would you type a tuple representing a command name followed by any number of string arguments?**
```typescript
let command: [string, ...string[]];
command = ["deploy", "--env", "production"];
```
The rest element (`...string[]`) captures zero or more additional elements of the given type after the fixed leading elements.

**Q12. Why can reordering numeric enum members be risky in production code?**
Because numeric enum values are implicit, based on declaration order (unless explicitly assigned). If a new member is inserted in the middle of an existing enum, every member after it silently shifts to a new numeric value — which can break stored data, serialized values, or database records that reference the old numbers.
