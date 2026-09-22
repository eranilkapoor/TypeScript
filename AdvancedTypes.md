**Advanced Types** cover the tools TypeScript gives you for modeling shapes that go beyond simple annotations — combining types, narrowing them safely at runtime, deriving new types from existing ones, and transforming types programmatically. These features are what let TypeScript express real-world data accurately (an API response that can be one of several shapes, a form value that starts as `unknown`, a type derived from another type) and are heavily used in production codebases and libraries alike.

---

### **Union and Intersection Types, Revisited**

[TypesAndAnnotations.md](./TypesAndAnnotations.md) introduces union (`|`) and intersection (`&`) types briefly. Here's a quick recap before going deeper:

- A **union type** (`A | B`) means a value can be *either* `A` or `B`.
- An **intersection type** (`A & B`) means a value must satisfy *both* `A` and `B` at once.

```typescript
type Status = "loading" | "success" | "error"; // union of string literals

interface Timestamped {
  createdAt: Date;
}
interface Named {
  name: string;
}
type NamedAndTimestamped = Named & Timestamped; // intersection

const record: NamedAndTimestamped = { name: "Report", createdAt: new Date() };
console.log(record.name);
// Output: Report
```

#### **Unions of Object Types**
When you access a property on a union of object types, TypeScript only allows properties that exist on *every* member of the union — this is called the "common subset."

```typescript
interface Cat {
  meow(): void;
}
interface Dog {
  bark(): void;
}

function makeNoise(animal: Cat | Dog) {
  // animal.meow(); // Error: 'meow' does not exist on type 'Dog'
  if ("meow" in animal) {
    animal.meow();
  } else {
    animal.bark();
  }
}
```

#### **Intersections Can Conflict**
Combining two types with a property of the same name but incompatible types collapses that property to `never`:

```typescript
type A = { value: string };
type B = { value: number };
type Combined = A & B; // value: string & number = never

// const x: Combined = { value: "hi" }; // Error: no value satisfies both string and number
```

---

### **Type Narrowing**

**Narrowing** is how TypeScript refines a broad type (usually a union) down to a more specific one based on runtime checks in your code.

#### **`typeof` Guards**
Best for narrowing primitives.

```typescript
function formatValue(value: string | number) {
  if (typeof value === "string") {
    return value.toUpperCase(); // value is string here
  }
  return value.toFixed(2); // value is number here
}

console.log(formatValue("hi"), formatValue(3.14159));
// Output: HI 3.14
```

#### **`instanceof` Guards**
Best for narrowing class instances.

```typescript
class ApiError extends Error {
  constructor(message: string, public statusCode: number) {
    super(message);
  }
}

function handleError(error: Error) {
  if (error instanceof ApiError) {
    console.log(`API error ${error.statusCode}: ${error.message}`);
  } else {
    console.log(`Generic error: ${error.message}`);
  }
}

handleError(new ApiError("Not Found", 404));
// Output: API error 404: Not Found
```

#### **`in` Operator Guards**
Best for narrowing plain object shapes that don't share a discriminant property.

```typescript
interface Fish {
  swim(): void;
}
interface Bird {
  fly(): void;
}

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim(); // animal is Fish
  } else {
    animal.fly(); // animal is Bird
  }
}
```

#### **Truthiness Narrowing**
```typescript
function printLength(text: string | null | undefined) {
  if (text) {
    console.log(text.length); // text is string here (not null/undefined/empty)
  } else {
    console.log("No text provided");
  }
}

printLength("hello");
// Output: 5
printLength(null);
// Output: No text provided
```

#### **Equality Narrowing**
```typescript
function compare(a: string | number, b: string | boolean) {
  if (a === b) {
    // Only 'string' is common to both unions, so both are narrowed to string
    console.log(a.toUpperCase(), b.toUpperCase());
  }
}
```

| Narrowing Technique | Best For |
|---|---|
| `typeof` | Primitives (`string`, `number`, `boolean`, `symbol`, etc.) |
| `instanceof` | Class instances |
| `in` | Distinguishing plain object shapes by property presence |
| Truthiness | Filtering out `null`/`undefined`/falsy values |
| Equality (`===`, `switch`) | Narrowing based on comparing two variables or a literal |

---

### **Discriminated Unions (Tagged Unions)**

A **discriminated union** is a union of object types that all share a common literal property (the "discriminant" or "tag"), which lets TypeScript narrow the exact variant with a simple `switch` or `if`.

```typescript
interface Circle {
  kind: "circle";
  radius: number;
}
interface Square {
  kind: "square";
  sideLength: number;
}
interface Rectangle {
  kind: "rectangle";
  width: number;
  height: number;
}

type Shape = Circle | Square | Rectangle;

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2; // shape is Circle
    case "square":
      return shape.sideLength ** 2; // shape is Square
    case "rectangle":
      return shape.width * shape.height; // shape is Rectangle
    default:
      // Exhaustiveness check: if a new shape is added, this line errors
      const _exhaustive: never = shape;
      throw new Error(`Unhandled shape: ${_exhaustive}`);
  }
}

console.log(getArea({ kind: "circle", radius: 2 }));
// Output: 12.566370614359172
console.log(getArea({ kind: "rectangle", width: 4, height: 5 }));
// Output: 20
```

The `default: const _exhaustive: never = shape;` trick is a common pattern: if someone later adds a `Triangle` variant to `Shape` but forgets to handle it in `getArea`, TypeScript raises a compile error right at that line, because `shape` would no longer be assignable to `never`.

---

### **User-Defined Type Guard Functions**

A **type predicate** function lets you write your own reusable narrowing logic, using the `arg is Type` return type syntax.

```typescript
interface Bird {
  fly(): void;
  layEggs(): void;
}
interface Fish {
  swim(): void;
  layEggs(): void;
}

function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}

function move(pet: Fish | Bird) {
  if (isFish(pet)) {
    pet.swim(); // narrowed to Fish
  } else {
    pet.fly(); // narrowed to Bird
  }
}
```

Type guards are especially useful when the same narrowing check is needed in multiple places, or when the logic is too complex for `typeof`/`instanceof`/`in` alone (e.g. validating an unknown value from `JSON.parse`).

```typescript
function isStringArray(value: unknown): value is string[] {
  return Array.isArray(value) && value.every((item) => typeof item === "string");
}

const data: unknown = ["a", "b", "c"];
if (isStringArray(data)) {
  console.log(data.join(", "));
}
// Output: a, b, c
```

---

### **Mapped Types**

A **mapped type** builds a new type by transforming every property of an existing type, using the `{ [K in keyof T]: ... }` syntax.

```typescript
interface Todo {
  title: string;
  completed: boolean;
}

type ReadonlyTodo = {
  readonly [K in keyof Todo]: Todo[K];
};

const todo: ReadonlyTodo = { title: "Learn TS", completed: false };
// todo.completed = true; // Error: cannot assign to a readonly property
```

#### **A Custom Mapped Type: Making Everything Nullable**
```typescript
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

interface User {
  id: number;
  name: string;
}

const draftUser: Nullable<User> = { id: null, name: null };
console.log(draftUser);
// Output: { id: null, name: null }
```

Mapped types are how many of TypeScript's built-in utility types (`Partial`, `Readonly`, `Pick`, etc., covered below) are actually implemented under the hood.

---

### **Conditional Types**

A **conditional type** picks between two types based on a type-level condition, using `T extends U ? X : Y`.

```typescript
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<string>; // "yes"
type B = IsString<number>; // "no"
```

#### **A Practical Example**
```typescript
type ElementType<T> = T extends (infer U)[] ? U : T;

type Item = ElementType<string[]>; // string
type Single = ElementType<number>; // number
```

Conditional types are frequently combined with `infer` (as above) to extract a nested type from within a larger type — this is exactly how built-in helpers like `ReturnType<T>` and `Parameters<T>` are implemented.

---

### **Built-In Utility Types**

TypeScript ships a set of generic utility types, built from mapped and conditional types, for common type transformations.

#### **`Partial<T>`**
Makes every property optional.
```typescript
interface User {
  id: number;
  name: string;
}

function updateUser(id: number, changes: Partial<User>) {
  // changes might only include some fields
}

updateUser(1, { name: "New Name" }); // no need to pass 'id'
```

#### **`Required<T>`**
Makes every property required (the opposite of `Partial<T>`).
```typescript
interface Config {
  host?: string;
  port?: number;
}

const fullConfig: Required<Config> = { host: "localhost", port: 3000 };
```

#### **`Readonly<T>`**
Makes every property read-only.
```typescript
const settings: Readonly<{ theme: string }> = { theme: "dark" };
// settings.theme = "light"; // Error: cannot assign to read-only property
```

#### **`Pick<T, K>`**
Builds a type with only the specified keys.
```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type UserPreview = Pick<User, "id" | "name">;
const preview: UserPreview = { id: 1, name: "Alice" };
```

#### **`Omit<T, K>`**
Builds a type with all keys except the specified ones (the opposite of `Pick<T, K>`).
```typescript
type UserWithoutEmail = Omit<User, "email">;
const noEmailUser: UserWithoutEmail = { id: 1, name: "Alice" };
```

#### **`Record<K, T>`**
Builds an object type with keys `K` all mapped to value type `T`.
```typescript
type Role = "admin" | "editor" | "viewer";
const permissions: Record<Role, string[]> = {
  admin: ["read", "write", "delete"],
  editor: ["read", "write"],
  viewer: ["read"],
};

console.log(permissions.editor);
// Output: [ 'read', 'write' ]
```

#### **`ReturnType<T>`**
Extracts the return type of a function type.
```typescript
function createUser() {
  return { id: 1, name: "Alice" };
}

type NewUser = ReturnType<typeof createUser>; // { id: number; name: string }
```

#### **`Parameters<T>`**
Extracts a function's parameter types as a tuple.
```typescript
function greet(name: string, age: number) {
  console.log(`${name} is ${age}`);
}

type GreetParams = Parameters<typeof greet>; // [name: string, age: number]

const args: GreetParams = ["Bob", 25];
greet(...args);
// Output: Bob is 25
```

#### **Utility Types Summary**

| Utility Type | What It Does |
|---|---|
| `Partial<T>` | Makes all properties of `T` optional |
| `Required<T>` | Makes all properties of `T` required |
| `Readonly<T>` | Makes all properties of `T` read-only |
| `Pick<T, K>` | Builds a type with only keys `K` from `T` |
| `Omit<T, K>` | Builds a type with all keys of `T` except `K` |
| `Record<K, T>` | Builds an object type with keys `K` and value type `T` |
| `ReturnType<T>` | Extracts the return type of function type `T` |
| `Parameters<T>` | Extracts the parameter types of function type `T` as a tuple |

---

### **Best Practices**
- Prefer discriminated unions over loosely related optional fields when a value can take several distinct "shapes" — they make invalid states unrepresentable.
- Use the `default: const x: never = value;` exhaustiveness pattern in `switch` statements over discriminated unions so the compiler catches unhandled cases.
- Write user-defined type guards (`arg is Type`) for narrowing logic you need to reuse across multiple functions, instead of duplicating `typeof`/`in` checks.
- Reach for built-in utility types (`Partial`, `Pick`, `Omit`, etc.) before writing your own mapped types — they cover the vast majority of everyday needs.
- Keep conditional types simple and well-named; deeply nested conditional types are powerful but hard to read and debug.
- Avoid type assertions (`as`) as a substitute for proper narrowing — they bypass the compiler's checks instead of proving type safety.

---

### **Interview Questions**

**Q1. What is the difference between a union type and an intersection type?**
A union type (`A | B`) means a value can be either `A` or `B`. An intersection type (`A & B`) means a value must satisfy both `A` and `B` simultaneously — it combines all their members into one type.

**Q2. What is type narrowing, and why is it necessary?**
Narrowing is the process by which TypeScript refines a broader type (typically a union) to a more specific one based on runtime checks like `typeof`, `instanceof`, or `in`. It's necessary because operations valid on one member of a union may not be valid on another, so the compiler needs proof of which specific type you're dealing with before allowing an operation.

**Q3. What is a discriminated union, and why is it useful?**
A discriminated union is a union of object types that share a common literal "tag" property (like `kind`). Switching on that tag lets TypeScript automatically narrow to the exact variant in each branch, which is safer and clearer than a pile of optional fields.
```typescript
type Shape = { kind: "circle"; radius: number } | { kind: "square"; sideLength: number };
```

**Q4. How do you write a user-defined type guard function?**
By giving the function a return type of `arg is Type` instead of `boolean`, and returning a boolean expression that actually verifies the value is that type. TypeScript then narrows the argument's type wherever the function returns `true`.
```typescript
function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}
```

**Q5. What is the "exhaustiveness check" pattern using `never`, and what does it protect against?**
It's a `default` case in a `switch` over a discriminated union that assigns the remaining value to a variable typed `never`. If a new variant is added to the union later but not handled in the switch, the assignment fails to compile, catching the oversight at build time instead of at runtime.

**Q6. What does a mapped type do? Give a simple example.**
A mapped type transforms every property of an existing type according to a rule, using `{ [K in keyof T]: ... }` syntax. For example, `type Nullable<T> = { [K in keyof T]: T[K] | null }` makes every property of `T` nullable.

**Q7. What is a conditional type, and where might you use `infer` with one?**
A conditional type (`T extends U ? X : Y`) selects between two types based on a type-level condition. `infer` is used inside the condition to extract and capture a nested type, e.g. `T extends (infer U)[] ? U : T` extracts the element type of an array.

**Q8. What's the difference between `Pick<T, K>` and `Omit<T, K>`?**
`Pick<T, K>` builds a new type containing only the specified keys `K` from `T`. `Omit<T, K>` does the opposite — it builds a new type with all keys of `T` except the specified ones.

**Q9. How would you type an object whose keys are a fixed set of string literals and whose values are all the same type?**
Using `Record<K, T>`, e.g. `Record<"admin" | "editor" | "viewer", string[]>` for a permissions map keyed by role.

**Q10. How do `ReturnType<T>` and `Parameters<T>` work, and what do you typically pass them?**
Both operate on a function *type*, typically obtained with `typeof someFunction`. `ReturnType<typeof fn>` extracts what `fn` returns; `Parameters<typeof fn>` extracts a tuple of `fn`'s parameter types.

**Q11. What's the difference between `Partial<T>` and `Required<T>`?**
`Partial<T>` makes every property of `T` optional; `Required<T>` makes every property required, stripping any `?` modifiers. They are opposites of each other.

**Q12. Why is the `in` operator guard preferred over `instanceof` for narrowing plain object unions?**
`instanceof` only works for checking against a class's prototype chain, so it can't distinguish between two plain object literal types that aren't class instances. The `in` operator checks for property existence at runtime, which works for any object shape, including plain interfaces without a common discriminant.
