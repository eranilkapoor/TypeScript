**Classes in TypeScript** extend JavaScript's native `class` syntax with a full type system: typed properties, access modifiers that enforce encapsulation at compile time, abstract classes, and interface-based contracts. If you already know JavaScript classes, the syntax will feel familiar — TypeScript's additions are almost entirely about making the compiler enforce rules that JavaScript alone leaves you to remember by convention.

---

### **Class Syntax with Typed Properties and Constructor Parameters**

Properties are declared with a type, and constructor parameters are typed just like function parameters.

```typescript
class Person {
  name: string;
  age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  introduce(): string {
    return `Hi, I'm ${this.name} and I'm ${this.age} years old.`;
  }
}

const alice = new Person("Alice", 30);
console.log(alice.introduce());
// Output: Hi, I'm Alice and I'm 30 years old.
```

If a property is declared but never given a type or default value that TypeScript can infer, and `strictPropertyInitialization` is enabled, the compiler requires it to be assigned in the constructor — catching properties that would otherwise silently be `undefined`.

---

### **Access Modifiers: `public`, `private`, `protected`**

TypeScript adds three access modifiers that control where a property or method can be accessed from — enforced entirely at **compile time** (they have no effect on the emitted JavaScript's runtime behavior, aside from erasing the keywords themselves).

1. **`public`** (the default): accessible from anywhere — inside the class, from subclasses, and from outside code. You rarely need to write `public` explicitly, since it's the default.
2. **`private`**: accessible only from **within the declaring class itself** — not from subclasses, and not from outside code.
3. **`protected`**: accessible from within the declaring class **and any subclasses**, but not from outside code.

```typescript
class BankAccount {
  public accountHolder: string;
  private balance: number;
  protected accountType: string;

  constructor(accountHolder: string, initialBalance: number) {
    this.accountHolder = accountHolder;
    this.balance = initialBalance;
    this.accountType = "checking";
  }

  deposit(amount: number): void {
    this.balance += amount; // OK — accessed from within the declaring class
  }

  getBalance(): number {
    return this.balance;
  }
}

const account = new BankAccount("Alice", 1000);
console.log(account.accountHolder); // OK — public
// console.log(account.balance);    // Error: Property 'balance' is private
console.log(account.getBalance());
// Output: 1000
```

```typescript
class SavingsAccount extends BankAccount {
  showType(): void {
    console.log(this.accountType); // OK — protected, accessible from a subclass
    // console.log(this.balance);  // Error: 'balance' is private and only accessible within class 'BankAccount'
  }
}
```

| Modifier | Within declaring class | Within subclasses | From outside |
|---|---|---|---|
| `public` (default) | Yes | Yes | Yes |
| `protected` | Yes | Yes | No |
| `private` | Yes | No | No |

#### **Compared to JavaScript's Native `#private` Fields**

JavaScript itself has native private fields, written with a `#` prefix, which are enforced at **runtime** (not just compile time) — code truly cannot access a `#field` from outside the class, even via tricks like `Object.keys` or bracket notation.

```typescript
class Counter {
  #count = 0; // native JS private field — enforced at runtime too

  increment(): void {
    this.#count++;
  }

  get value(): number {
    return this.#count;
  }
}

const counter = new Counter();
counter.increment();
console.log(counter.value);
// Output: 1
// console.log(counter["#count"]); // undefined — truly inaccessible, even at runtime
```

TypeScript's `private` keyword, by contrast, is a **compile-time-only** restriction — the property still exists as a completely ordinary, accessible property on the compiled JavaScript object. Code that bypasses the type checker (e.g., using `any`, or plain `.js` consumers of a compiled library) can still read or modify a TypeScript `private` field at runtime.

```typescript
class Wallet {
  private balance = 100;
}

const wallet = new Wallet();
console.log((wallet as any).balance);
// Output: 100 (TypeScript's `private` did not stop this at runtime)
```

---

### **Readonly Properties**

A `readonly` property can be assigned once — either at declaration or inside the constructor — but never reassigned afterward.

```typescript
class Configuration {
  readonly apiUrl: string;

  constructor(apiUrl: string) {
    this.apiUrl = apiUrl;
  }
}

const config = new Configuration("https://api.example.com");
console.log(config.apiUrl);
// Output: https://api.example.com
// config.apiUrl = "https://other.com"; // Error: Cannot assign to 'apiUrl' because it is a read-only property.
```

---

### **Parameter Properties Shorthand**

Instead of separately declaring a property and assigning it in the constructor, TypeScript lets you combine both steps by adding an access modifier (or `readonly`) directly to a constructor parameter — called a **parameter property**.

```typescript
// Verbose version, without parameter properties:
class PointVerbose {
  private x: number;
  private y: number;

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }
}

// Equivalent, using parameter properties:
class Point {
  constructor(private x: number, private y: number) {}

  toString(): string {
    return `(${this.x}, ${this.y})`;
  }
}

const p = new Point(3, 4);
console.log(p.toString());
// Output: (3, 4)
```

Any combination of `public`, `private`, `protected`, and `readonly` can be used on a constructor parameter to trigger this shorthand — TypeScript declares the property and assigns it from the argument automatically.

```typescript
class Product {
  constructor(
    public readonly id: number,
    public name: string,
    private cost: number
  ) {}
}
```

---

### **Abstract Classes and Abstract Methods**

An **abstract class** cannot be instantiated directly — it exists only to be extended. It can define fully-implemented methods (shared by all subclasses) alongside **abstract methods**, which declare a signature that every concrete subclass must implement, without providing a body themselves.

```typescript
abstract class Shape {
  abstract area(): number; // no implementation — subclasses must provide one

  describe(): string {
    return `This shape has an area of ${this.area()}`;
  }
}

// const shape = new Shape(); // Error: Cannot create an instance of an abstract class.

class Rectangle extends Shape {
  constructor(private width: number, private height: number) {
    super();
  }

  area(): number {
    return this.width * this.height;
  }
}

const rect = new Rectangle(4, 5);
console.log(rect.describe());
// Output: This shape has an area of 20
```

Abstract classes are useful when you want to share common implementation across a family of related classes while still forcing each subclass to fill in the pieces that must vary.

---

### **`implements` vs. `extends`**

1. **`extends`**: class inheritance — a subclass gets a superclass's actual implementation (properties, methods, behavior) and can override or add to it. A class can only `extend` **one** other class.
2. **`implements`**: a class promises to conform to one or more interfaces' shapes, but gets **no** implementation from them — it must provide its own. A class can `implements` **multiple** interfaces at once.

```typescript
interface Flyable {
  fly(): void;
}

interface Swimmable {
  swim(): void;
}

class Animal {
  constructor(protected name: string) {}
}

// A class can extend ONE class, but implement MULTIPLE interfaces
class Duck extends Animal implements Flyable, Swimmable {
  constructor(name: string) {
    super(name);
  }

  fly(): void {
    console.log(`${this.name} is flying`);
  }

  swim(): void {
    console.log(`${this.name} is swimming`);
  }
}

const duck = new Duck("Donald");
duck.fly();
// Output: Donald is flying
duck.swim();
// Output: Donald is swimming
```

| Feature | `extends` | `implements` |
|---|---|---|
| **Source** | Another class | One or more interfaces (or object-shaped type aliases) |
| **Provides implementation?** | Yes — inherited methods/properties come with working code | No — only the shape/contract; the class must implement everything itself |
| **How many at once** | One (single inheritance) | Multiple, comma-separated |
| **`super()` required in constructor?** | Yes, if the subclass has its own constructor | No — interfaces have no constructor to call |

---

### **Static Properties and Methods**

`static` members belong to the **class itself**, not to any individual instance — accessed via the class name rather than through `this` on an object.

```typescript
class MathUtils {
  static PI = 3.14159;

  static square(n: number): number {
    return n * n;
  }
}

console.log(MathUtils.PI);
// Output: 3.14159
console.log(MathUtils.square(5));
// Output: 25
```

A common real-world use is a counter shared across all instances of a class:

```typescript
class User {
  static userCount = 0;

  constructor(public name: string) {
    User.userCount++;
  }
}

new User("Alice");
new User("Bob");
console.log(User.userCount);
// Output: 2
```

---

### **Getters and Setters**

`get` and `set` accessors let you expose property-like syntax while running custom logic behind the scenes — commonly used to validate a value before storing it, or to compute a derived value on read.

```typescript
class Temperature {
  private _celsius: number;

  constructor(celsius: number) {
    this._celsius = celsius;
  }

  get celsius(): number {
    return this._celsius;
  }

  set celsius(value: number) {
    if (value < -273.15) {
      throw new Error("Temperature below absolute zero is not possible");
    }
    this._celsius = value;
  }

  get fahrenheit(): number {
    return this._celsius * 1.8 + 32;
  }
}

const temp = new Temperature(25);
console.log(temp.celsius);
// Output: 25
console.log(temp.fahrenheit);
// Output: 77

temp.celsius = 30;
console.log(temp.celsius);
// Output: 30

// temp.celsius = -300; // throws: Temperature below absolute zero is not possible
```

Getters and setters are called using ordinary property syntax (`temp.celsius`, not `temp.celsius()`), even though a method runs behind the scenes — this makes validation and computed properties transparent to the caller.

---

### **Best Practices**

- Default to `private` for internal implementation details and only expose what genuinely needs to be part of the class's public API.
- Prefer parameter properties (`constructor(private name: string)`) over manually declaring and assigning each property, to reduce boilerplate.
- Remember that TypeScript's `private`/`protected` are compile-time-only; use native `#private` fields when you need runtime-enforced privacy (e.g., for a library consumed by plain JavaScript).
- Use `abstract` classes to share common implementation across related subclasses while forcing each one to implement the parts that must differ.
- Favor `implements` (possibly with multiple interfaces) to describe capabilities a class has, and reserve `extends` for genuine "is-a" inheritance relationships with shared implementation.
- Use getters/setters to validate or compute values transparently, but avoid putting expensive or side-effecting logic behind a getter, since callers expect property access to be cheap and side-effect-free.

---

### **Interview Questions**

**Q1. What are the three access modifiers in TypeScript classes, and what does each control?**
`public` (the default) allows access from anywhere. `protected` allows access from within the declaring class and its subclasses, but not from outside code. `private` allows access only from within the declaring class itself, not even from subclasses.

**Q2. Are TypeScript's `private` and JavaScript's native `#private` fields enforced the same way?**
No. TypeScript's `private` is a compile-time-only restriction — the property is still an ordinary, accessible property in the compiled JavaScript, so code that bypasses the type checker (like casting to `any`) can still reach it at runtime. Native `#private` fields are enforced by the JavaScript engine itself, making them truly inaccessible from outside the class at runtime.

**Q3. What is a parameter property, and what does it save you from writing?**
It's a shorthand where adding an access modifier (`public`, `private`, `protected`) or `readonly` directly to a constructor parameter automatically declares a matching class property and assigns the argument to it — saving you from separately declaring the property and writing `this.x = x` in the constructor body.

**Q4. What is the difference between `extends` and `implements`?**
`extends` is class inheritance — a subclass receives an actual working implementation from its superclass and can only extend one class. `implements` is a contract — a class promises to match an interface's shape but must provide its own implementation, and it can implement multiple interfaces at once.

**Q5. Can a class extend a class and implement interfaces at the same time?**
Yes. A class can `extend` exactly one other class while also `implements`-ing one or more interfaces, combining inherited implementation with additional type contracts.
```typescript
class Duck extends Animal implements Flyable, Swimmable { /* ... */ }
```

**Q6. What is an abstract class, and how is it different from a regular class?**
An abstract class cannot be instantiated directly — it exists only to be extended. It can mix fully-implemented methods with `abstract` methods that declare a signature but no body, forcing every concrete subclass to provide an implementation for those methods.

**Q7. What happens if a subclass of an abstract class fails to implement an abstract method?**
The compiler raises an error — a non-abstract subclass must provide concrete implementations for every abstract member declared by its abstract superclass, or it must itself be declared abstract.

**Q8. What is the difference between a static property and an instance property?**
A static property belongs to the class itself and is shared across all instances (accessed via `ClassName.property`), while an instance property belongs to a specific object created with `new` (accessed via `instance.property`), with each instance getting its own copy.

**Q9. How do getters and setters differ from regular methods in how they're called?**
Getters and setters are accessed using plain property syntax (`obj.value`), not function-call syntax (`obj.value()`), even though a method executes behind the scenes. This lets you run validation or computed logic while keeping the calling code looking like simple property access.

**Q10. Why might you use a getter for a `fahrenheit` value that's derived from a stored `celsius` value?**
It lets consumers read `temperature.fahrenheit` as if it were a stored property, while the class actually computes it on demand from the real source of truth (`celsius`), avoiding the need to keep two separate values in sync manually.

**Q11. What compiler option affects whether class properties must be initialized before use, and what does it check?**
`strictPropertyInitialization` (part of the `strict` family of flags) requires that every declared class property is definitely assigned by the end of the constructor — either via a default value, direct assignment, or a parameter property — catching properties that would otherwise silently be `undefined` at runtime.

**Q12. Why can a class implement multiple interfaces but extend only one class?**
Interfaces are pure contracts with no implementation to reconcile, so combining several of them just merges their required shapes — there's no ambiguity. Classes carry actual implementation (state and method bodies), and most OOP languages, TypeScript included, avoid multiple class inheritance because merging conflicting implementations from multiple parent classes is ambiguous and error-prone (the "diamond problem").
