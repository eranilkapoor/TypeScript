**Interfaces and type aliases** are TypeScript's two main tools for describing the shape of objects. They overlap significantly — for a plain object shape, they're often interchangeable — but each has capabilities the other lacks. Knowing when to reach for which one is one of the most practical, everyday decisions you'll make in a TypeScript codebase.

---

### **Interface Syntax: Declaring an Object Shape**

An `interface` declares the shape a value must conform to — which properties it must have, and what type each one is.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

const user: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
};

console.log(user.name);
// Output: Alice

// const invalidUser: User = { id: 2, name: "Bob" }; // Error: Property 'email' is missing
```

#### **Optional Properties (`?`)**

A `?` after a property name marks it as not required.

```typescript
interface Product {
  name: string;
  price: number;
  discount?: number; // optional
}

const item1: Product = { name: "Pen", price: 2 };
const item2: Product = { name: "Bag", price: 40, discount: 5 };

console.log(item1.discount);
// Output: undefined
```

#### **Readonly Properties**

A `readonly` property can be set when the object is created, but not reassigned afterward.

```typescript
interface Point {
  readonly x: number;
  readonly y: number;
}

const origin: Point = { x: 0, y: 0 };
// origin.x = 5; // Error: Cannot assign to 'x' because it is a read-only property.

console.log(origin);
// Output: { x: 0, y: 0 }
```

---

### **Extending Interfaces**

An interface can build on another using `extends`, inheriting all of its members and adding more.

```typescript
interface Animal {
  name: string;
}

interface Dog extends Animal {
  breed: string;
}

const myDog: Dog = { name: "Rex", breed: "Labrador" };
console.log(myDog);
// Output: { name: 'Rex', breed: 'Labrador' }
```

#### **Multiple Interface Inheritance**

Unlike class inheritance (which is single-parent only), an interface can extend **multiple** other interfaces at once, combining all of their members.

```typescript
interface HasWheels {
  wheelCount: number;
}

interface HasEngine {
  horsepower: number;
}

interface Car extends HasWheels, HasEngine {
  brand: string;
}

const myCar: Car = { wheelCount: 4, horsepower: 300, brand: "Toyota" };
console.log(myCar);
// Output: { wheelCount: 4, horsepower: 300, brand: 'Toyota' }
```

---

### **Index Signatures**

An **index signature** describes the type of properties on an object when you don't know the exact property names in advance — only the type of the keys and the type of the values.

```typescript
interface StringDictionary {
  [key: string]: number;
}

const wordCounts: StringDictionary = {
  apple: 3,
  banana: 5,
};

wordCounts.cherry = 7; // OK — matches the index signature
console.log(wordCounts);
// Output: { apple: 3, banana: 5, cherry: 7 }

// wordCounts.grape = "many"; // Error: Type 'string' is not assignable to type 'number'.
```

Index signatures are useful for representing maps or dictionaries with a dynamic set of keys, while still constraining every value to a consistent type.

---

### **Interfaces for Function Types and Class Contracts**

#### **Function Types**

An interface can describe a callable signature instead of an object's properties.

```typescript
interface Comparator {
  (a: number, b: number): boolean;
}

const isGreater: Comparator = (a, b) => a > b;
console.log(isGreater(5, 3));
// Output: true
```

#### **Class Contracts with `implements`**

An interface can also describe the public shape a class must provide, using `implements`. This is a brief mention — full class coverage, including constructors, access modifiers, and abstract classes, is in [ClassesInTypeScript.md](./ClassesInTypeScript.md).

```typescript
interface Shape {
  area(): number;
}

class Circle implements Shape {
  constructor(private radius: number) {}

  area(): number {
    return Math.PI * this.radius ** 2;
  }
}

const circle = new Circle(3);
console.log(circle.area().toFixed(2));
// Output: 28.27
```

---

### **`type` vs. `interface`**

For a plain object shape, `type` and `interface` are largely interchangeable — both of the following work the same way:

```typescript
interface UserA {
  name: string;
}

type UserB = {
  name: string;
};
```

But each has capabilities the other doesn't:

- **`interface`** supports **declaration merging** (multiple declarations with the same name automatically combine) and is extended with the `extends` keyword, which gives clearer error messages in some cases.
- **`type`** can represent things an interface fundamentally cannot: unions, intersections of non-object types, primitives, tuples, and mapped/conditional types.

```typescript
// type can alias a union — interface cannot
type Status = "loading" | "success" | "error";

// type can alias a primitive
type ID = string;

// type can alias a tuple
type Coordinate = [number, number];

// interface CANNOT do any of the above —
// interface Status = "loading" | "success" | "error"; // not valid syntax
```

| Feature | `interface` | `type` |
|---|---|---|
| **Object shapes** | Yes | Yes |
| **Extending / combining** | `extends` (multiple) | `&` intersection |
| **Declaration merging** | Yes — multiple declarations combine automatically | No — duplicate names cause an error |
| **Union types** | No | Yes |
| **Tuple types** | No | Yes |
| **Primitive aliases** | No | Yes |
| **Implementable by a class** | Yes (`implements`) | Yes (object-shaped type aliases only) |
| **Typical error message clarity** | Slightly clearer for object shapes | Can be more complex for deeply nested aliases |

**Rule of thumb**: use `interface` for object shapes and class contracts — especially anything public-facing, like a library's exported API, where declaration merging or clean `extends`-based composition is valuable. Use `type` for unions, tuples, primitive aliases, and any type that isn't a plain object shape. When in doubt for a simple object, either works — pick one convention and apply it consistently across your codebase.

---

### **Declaration Merging for Interfaces**

Unlike `type`, declaring two interfaces with the **same name** in the same scope doesn't cause an error — TypeScript automatically merges their members into a single interface. This is most commonly used to extend types from external libraries without modifying their source.

```typescript
interface Window {
  appVersion: string;
}

interface Window {
  isDebugMode: boolean;
}

// The two declarations merge into one interface with both properties:
// interface Window { appVersion: string; isDebugMode: boolean; }

function logWindowInfo(win: Window): void {
  console.log(`v${win.appVersion}, debug: ${win.isDebugMode}`);
}

logWindowInfo({ appVersion: "1.0.0", isDebugMode: true });
// Output: v1.0.0, debug: true
```

Declaration merging is especially useful when augmenting global or third-party types (like adding a custom property to the browser's `Window` object) without needing to fork or edit the library's own type definitions.

---

### **Best Practices**

- Use `interface` for object shapes and class contracts, and `type` for unions, tuples, and anything that isn't a plain object — pick a consistent convention for the "either works" cases.
- Mark properties `readonly` whenever they shouldn't change after object creation, to catch accidental mutation at compile time.
- Prefer multiple interface inheritance (`extends A, B`) over duplicating shared properties across unrelated interfaces.
- Use index signatures for genuine dictionary-like objects with dynamic keys, but prefer a `Record<K, V>` type alias or an explicit object shape when the keys are actually known ahead of time.
- Reserve declaration merging for augmenting external/global types (like extending `Window`); avoid relying on it for your own application code, where explicit `extends` is easier to follow.
- Keep interfaces focused and composable — favor several small, extendable interfaces over one large interface with many optional properties.

---

### **Interview Questions**

**Q1. What is the main difference between `type` and `interface` in TypeScript?**
Both can describe object shapes, but `interface` supports declaration merging and is extended with `extends`, while `type` can represent things interfaces cannot — unions, intersections of primitives, tuples, and mapped/conditional types.

**Q2. Can an interface represent a union type like `"a" | "b"`?**
No. Interfaces can only describe object shapes (or callable/constructable signatures) — unions must be expressed with a `type` alias.

**Q3. What is declaration merging, and when is it useful?**
Declaration merging is when two or more `interface` declarations with the same name in the same scope are automatically combined into one interface with all their members. It's most useful for augmenting third-party or global types (like adding a custom property to `Window`) without editing the original source.

**Q4. How do you make an interface property optional versus readonly?**
Optional uses a `?` after the property name (`age?: number`), allowing it to be omitted. Readonly uses the `readonly` modifier before the property name (`readonly id: number`), allowing it to be set on creation but not reassigned afterward. The two can be combined.

**Q5. Can an interface extend more than one other interface at once?**
Yes — unlike class inheritance, which is single-parent, an interface can extend multiple interfaces simultaneously using a comma-separated list: `interface Car extends HasWheels, HasEngine { ... }`.

**Q6. What is an index signature, and when would you use one?**
An index signature (e.g., `[key: string]: number`) describes the type of values for properties whose exact names aren't known ahead of time, only their key type and value type. It's used for dictionary/map-like objects with dynamic keys.

**Q7. How does a class use an interface to enforce a contract?**
A class uses the `implements` keyword to declare that it will provide every member described by the interface. The compiler then checks that the class actually implements all required properties and methods with matching types.
```typescript
interface Shape { area(): number; }
class Square implements Shape {
  constructor(private side: number) {}
  area(): number { return this.side ** 2; }
}
```

**Q8. If you need to alias a tuple type, would you use `type` or `interface`?**
`type` — tuples cannot be expressed with `interface`. For example: `type Point = [number, number];`.

**Q9. What happens if you declare two `type` aliases with the same name in the same scope?**
It's a compile error — unlike interfaces, type aliases do not support declaration merging; duplicate names conflict.

**Q10. What is a practical rule of thumb for choosing between `type` and `interface` for a simple object shape?**
Either works for a plain object shape, so pick a consistent project convention. A common guideline: use `interface` for public object/class contracts (to benefit from declaration merging and `extends`), and use `type` for everything else — unions, tuples, and derived/utility types.

**Q11. Can an interface describe a function's call signature?**
Yes — an interface can describe a callable value by declaring a signature without a name: `interface Comparator { (a: number, b: number): boolean; }`. A variable typed with this interface must be assignable to a function matching that signature.

**Q12. Why might a library's public type definitions favor `interface` over `type` for exported object shapes?**
Because `interface` supports declaration merging, consumers of the library can augment or extend those types (for example, adding custom properties to a config object) without needing to modify the library's source — something a `type` alias does not allow.
