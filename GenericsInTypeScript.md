**Generics** let you write functions, interfaces, classes, and type aliases that work with a variety of types while still preserving full type information and type safety. Instead of writing the same logic over and over for every type, or falling back to `any` and losing all compile-time checking, generics let you parameterize a type the same way a function parameterizes a value. Understanding generics is essential to reading and writing idiomatic TypeScript — most of the standard library (`Array<T>`, `Promise<T>`, `Map<K, V>`) and most well-designed third-party libraries lean heavily on them.

---

### **Why Generics Exist**

#### **The Problem with `any`**
Imagine you want a function that returns whatever value you pass into it. Without generics, you might reach for `any`:

```typescript
function identityAny(arg: any): any {
  return arg;
}

const result = identityAny(42);
result.toUpperCase(); // No compile-time error, but this crashes at runtime!
```

`any` disables type checking entirely. TypeScript has no idea that `result` is a number, so it happily lets you call `.toUpperCase()` on it — a bug that only surfaces at runtime.

#### **The Solution: Generics**
```typescript
function identity<T>(arg: T): T {
  return arg;
}

const result = identity(42); // T is inferred as number
// result.toUpperCase(); // Compile-time error: Property 'toUpperCase' does not exist on type 'number'
console.log(result.toFixed(2));
// Output: 42.00
```

Here, `T` is a **type parameter** — a placeholder for whatever type is passed in. TypeScript infers `T` from the argument and carries that specific type all the way through to the return value, so the relationship between input and output type is preserved and checked.

| Approach | Type Safety | Reusability | Preserves Type Info |
|---|---|---|---|
| **Duplicate functions per type** | Yes | No (copy-paste per type) | Yes |
| **`any`** | No | Yes | No |
| **Generics** | Yes | Yes | Yes |

---

### **Generic Functions**

#### **Basic Example: `identity`**
```typescript
function identity<T>(arg: T): T {
  return arg;
}

const num = identity<number>(10);   // explicit type argument
const str = identity("hello");       // inferred as string

console.log(num, str);
// Output: 10 hello
```

You can supply the type argument explicitly with `identity<number>(10)`, or let TypeScript infer it from the argument — inference is the more common style in everyday code.

#### **A More Realistic Example: `getFirstElement`**
```typescript
function getFirstElement<T>(arr: T[]): T | undefined {
  return arr.length > 0 ? arr[0] : undefined;
}

const firstNumber = getFirstElement([10, 20, 30]);   // T inferred as number
const firstString = getFirstElement(["a", "b", "c"]); // T inferred as string

console.log(firstNumber);
// Output: 10
console.log(firstString);
// Output: a
```

Without generics, you would need either an `any[]` parameter (losing type safety) or separate `getFirstNumber`, `getFirstString`, etc. functions (violating DRY). Generics give you one implementation that is fully type-checked for every type it's used with.

#### **Generic Functions with Multiple Parameters**
```typescript
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const p = pair("age", 30);
console.log(p);
// Output: [ 'age', 30 ]
```

---

### **Generic Interfaces and Type Aliases**

Generics apply just as naturally to interfaces and type aliases, letting you describe shapes that vary by the type they contain.

#### **Generic Interface**
```typescript
interface ApiResponse<T> {
  success: boolean;
  data: T;
  error?: string;
}

const userResponse: ApiResponse<{ id: number; name: string }> = {
  success: true,
  data: { id: 1, name: "Alice" },
};

console.log(userResponse.data.name);
// Output: Alice
```

#### **Generic Type Alias**
```typescript
type Pair<T, U> = {
  first: T;
  second: U;
};

const coordinate: Pair<number, number> = { first: 10, second: 20 };
const entry: Pair<string, boolean> = { first: "isAdmin", second: true };

console.log(coordinate, entry);
// Output: { first: 10, second: 20 } { first: 'isAdmin', second: true }
```

A generic type alias can also wrap a function signature:

```typescript
type Transformer<T, R> = (input: T) => R;

const stringify: Transformer<number, string> = (n) => `Value: ${n}`;
console.log(stringify(99));
// Output: Value: 99
```

See [InterfacesAndTypeAliases.md](./InterfacesAndTypeAliases.md) for the non-generic fundamentals of interfaces and type aliases.

---

### **Generic Classes**

Classes can also be parameterized by type, which is especially useful for container-like data structures.

#### **A Generic `Box<T>` Class**
```typescript
class Box<T> {
  private contents: T;

  constructor(value: T) {
    this.contents = value;
  }

  getContents(): T {
    return this.contents;
  }

  setContents(value: T): void {
    this.contents = value;
  }
}

const numberBox = new Box<number>(123);
const stringBox = new Box("hello"); // T inferred as string

console.log(numberBox.getContents());
// Output: 123
console.log(stringBox.getContents());
// Output: hello
```

#### **A Generic `Stack<T>` Class**
```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
numberStack.push(3);

console.log(numberStack.pop());
// Output: 3
console.log(numberStack.size);
// Output: 2
```

`Stack<T>` can now be reused for any type — `Stack<string>`, `Stack<User>`, etc. — while still catching mismatched types at compile time. See [ClassesInTypeScript.md](./ClassesInTypeScript.md) for a refresher on class fundamentals.

---

### **Generic Constraints with `extends`**

Sometimes you want a generic type parameter to accept "any type, but it must have at least these properties." That's what constraints do, using the `extends` keyword.

#### **The Problem Without Constraints**
```typescript
function printLength<T>(arg: T): void {
  // console.log(arg.length); // Error: Property 'length' does not exist on type 'T'
}
```

TypeScript doesn't know `T` has a `.length` property, because `T` could be anything — a number, a boolean, anything.

#### **The Fix: Constrain `T`**
```typescript
interface HasLength {
  length: number;
}

function printLength<T extends HasLength>(arg: T): void {
  console.log(arg.length);
}

printLength("hello");         // string has .length
printLength([1, 2, 3, 4]);    // array has .length
// printLength(42);           // Error: number does not satisfy HasLength
// Output:
// 5
// 4
```

You can also write the constraint inline without a named interface:

```typescript
function logLength<T extends { length: number }>(arg: T): T {
  console.log(`Length: ${arg.length}`);
  return arg;
}

logLength("TypeScript");
// Output: Length: 10
```

#### **Constraining to `keyof`**
A very common pattern constrains one type parameter using the keys of another, which is how `Object.keys`-style helper functions stay type-safe:

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Bob", isActive: true };

console.log(getProperty(user, "name"));
// Output: Bob
// getProperty(user, "email"); // Error: 'email' is not a key of user
```

---

### **Default Generic Type Parameters**

Just like function parameters can have default values, generic type parameters can have default types, used whenever the caller doesn't supply one explicitly.

```typescript
interface ApiResult<T = string> {
  data: T;
  timestamp: number;
}

const defaultResult: ApiResult = { data: "OK", timestamp: Date.now() }; // T defaults to string
const numericResult: ApiResult<number> = { data: 200, timestamp: Date.now() };

console.log(defaultResult.data, numericResult.data);
// Output: OK 200
```

Default type parameters are especially useful in generic classes and utility types where a "sensible default" makes the common case less verbose:

```typescript
class Container<T = unknown> {
  constructor(public value: T) {}
}

const c1 = new Container("plain value"); // T inferred as string, default unused
const c2: Container = new Container(42); // still works, T defaults to unknown when not annotated
```

---

### **Multiple Type Parameters**

Generics aren't limited to a single type parameter — you can declare as many as you need, which is how key-value structures like `Map<K, V>` are typed.

```typescript
interface KeyValuePair<K, V> {
  key: K;
  value: V;
}

function createPair<K, V>(key: K, value: V): KeyValuePair<K, V> {
  return { key, value };
}

const pair1 = createPair("age", 30);
const pair2 = createPair(1, "one");

console.log(pair1, pair2);
// Output: { key: 'age', value: 30 } { key: 1, value: 'one' }
```

#### **A Generic Map-Like Class with Two Type Parameters**
```typescript
class SimpleMap<K extends string | number, V> {
  private store: Record<string | number, V> = {};

  set(key: K, value: V): void {
    this.store[key] = value;
  }

  get(key: K): V | undefined {
    return this.store[key];
  }
}

const scores = new SimpleMap<string, number>();
scores.set("Alice", 95);
scores.set("Bob", 88);

console.log(scores.get("Alice"));
// Output: 95
```

---

### **Best Practices**
- Prefer generics over `any` whenever a function or structure needs to work with multiple types while preserving type relationships.
- Let TypeScript infer type arguments whenever possible; only specify them explicitly when inference is ambiguous or produces an unwanted type.
- Use constraints (`extends`) to express the minimum shape a generic type must satisfy, rather than loosening the constraint or falling back to `any`.
- Name type parameters meaningfully in complex generics (`TInput`, `TOutput`, `TKey`, `TValue`) instead of always defaulting to single letters when it improves readability.
- Use default type parameters to make the common case of a generic API more convenient, without giving up flexibility for advanced cases.
- Avoid over-engineering: if a function only ever needs to work with one concrete type, a generic adds complexity without benefit.

---

### **Interview Questions**

**Q1. What are generics in TypeScript, and why are they useful?**
Generics let you write reusable functions, classes, and types that work with multiple types while preserving type information, instead of using `any` and losing type safety. They let the caller "plug in" a type, and TypeScript checks that everything downstream stays consistent with it.

**Q2. What's the difference between using `any` and using a generic type parameter?**
`any` disables type checking entirely — the compiler makes no guarantees about the value's shape. A generic type parameter like `T` still enforces type safety; TypeScript tracks the actual type passed in and checks all subsequent usage against it.
```typescript
function identity<T>(arg: T): T { return arg; }
```

**Q3. How does TypeScript infer generic type arguments?**
TypeScript looks at the types of the arguments passed to a generic function and infers the type parameter from them, without requiring you to write the type argument explicitly, e.g. `identity(5)` infers `T` as `number`.

**Q4. What is a generic constraint, and when would you use one?**
A generic constraint (`T extends SomeType`) restricts a type parameter to only types that satisfy a given shape. You use it when your generic function needs to access specific properties or methods (like `.length`) that not every type has.
```typescript
function printLength<T extends { length: number }>(arg: T): void {
  console.log(arg.length);
}
```

**Q5. How do you define a generic interface?**
By adding a type parameter list after the interface name, then using that parameter inside the interface body, e.g. `interface Box<T> { value: T; }`.

**Q6. What is a default generic type parameter, and why is it useful?**
It's a fallback type used when the caller doesn't explicitly supply a type argument, written as `<T = DefaultType>`. It makes generic APIs more convenient to use in the common case while still allowing customization when needed.

**Q7. Can a generic type have multiple type parameters? Give an example.**
Yes — you can declare as many as needed, separated by commas, e.g. `interface KeyValuePair<K, V> { key: K; value: V; }`, which is how structures like maps or dictionaries are typed.

**Q8. What does `K extends keyof T` mean, and where is it commonly used?**
It constrains `K` to only be one of the property names (keys) of `T`. It's commonly used in helper functions like a type-safe `getProperty(obj, key)` function, ensuring the key passed in actually exists on the object.
```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

**Q9. How do generic classes differ from generic functions?**
A generic class declares its type parameter(s) after the class name (`class Box<T>`), and that parameter is then available to every method and property in the class. A generic function's type parameter is typically scoped to just that one function call.

**Q10. Is it possible to explicitly specify a type argument instead of relying on inference? When would you do that?**
Yes, using angle brackets, e.g. `identity<number>(42)`. You'd do this when inference would pick the wrong type (for example, when passing an empty array where you want to force a specific element type) or when it improves code clarity.

**Q11. What happens if you use a generic type parameter without any constraint and try to access a property on it?**
TypeScript will give a compile error, because an unconstrained `T` could be any type, including ones without that property. You must add a constraint (`T extends { propertyName: SomeType }`) before you can safely access it.

**Q12. Why might you avoid using generics in a function that only ever operates on one specific type?**
Because generics add a layer of abstraction and cognitive overhead. If a function is only ever called with, say, `string`, writing it with a concrete `string` parameter is simpler and just as safe, with no benefit from the extra flexibility generics provide.
