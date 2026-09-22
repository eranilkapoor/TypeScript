**Decorators** are special functions that let you annotate and modify classes, methods, properties, and parameters in a declarative way, using an `@expression` syntax placed just above the thing being decorated. They're a powerful metaprogramming tool for adding cross-cutting behavior — logging, validation, dependency injection, immutability — without cluttering the core logic of a class. Decorators are the backbone of major frameworks like Angular and NestJS, so understanding them is essential for working in those ecosystems.

---

### **What Decorators Are**

A decorator is just a function that receives information about the thing it's decorating and can inspect, wrap, or replace it. Conceptually, `@sealed` above a class is roughly equivalent to writing `sealed(MyClass)` right after the class is defined — the decorator function runs at class-definition time, not at instantiation time.

```typescript
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class Greeter {
  greet() {
    return "Hello!";
  }
}
```

Decorators can be applied to:
1. **Classes**: to observe, modify, or replace a class definition.
2. **Methods**: to wrap or alter a method's behavior.
3. **Properties**: to observe or modify a property's metadata.
4. **Parameters**: to record metadata about a specific function parameter (commonly used for dependency injection).

---

### **Enabling Decorators**

Decorators are a compiler-supported feature that must be explicitly turned on in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

- **`experimentalDecorators`**: enables the legacy TypeScript/Stage 2 decorator syntax described in this file (the version used by Angular and NestJS today).
- **`emitDecoratorMetadata`**: (optional) emits extra type metadata that reflection-based libraries (like NestJS's dependency injection) rely on at runtime.

#### **A Note on the Newer TC39 Decorators**
As of TypeScript 5.0, the language also supports the newer **TC39 Stage 3 decorators proposal** — a redesigned, now-standardized version of decorators that doesn't require `experimentalDecorators` at all and will eventually become native JavaScript. Its API differs somewhat from the "legacy" decorators shown in this file (different function signatures, no parameter decorators in the same form). However, most existing real-world code — including Angular and NestJS as of today — still uses the `experimentalDecorators` style, so that's what this file focuses on. When starting a brand-new project without ties to those frameworks, it's worth checking whether your tooling has adopted the newer standard decorators.

See [TypeScriptConfig.md](./TypeScriptConfig.md) for more on `tsconfig.json` options in general.

---

### **Class Decorators**

A class decorator receives the class's constructor function and can inspect it, add properties to it, or return a new constructor to replace it entirely.

```typescript
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class Greeter {
  greeting: string;

  constructor(message: string) {
    this.greeting = message;
  }

  greet() {
    return `Hello, ${this.greeting}`;
  }
}

const greeter = new Greeter("World");
console.log(greeter.greet());
// Output: Hello, World

// (greeter as any).newProp = "test"; // Fails silently or throws in strict mode: object is sealed
```

`Object.seal` prevents new properties from being added to the class or its prototype and marks existing properties as non-configurable — a simple way to lock down a class's shape after definition.

---

### **Method Decorators**

A method decorator receives the target (the class prototype for an instance method), the method name, and the property descriptor — letting it wrap the original method with new behavior.

```typescript
function log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;

  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${propertyKey} with args: ${JSON.stringify(args)}`);
    const result = originalMethod.apply(this, args);
    console.log(`${propertyKey} returned: ${JSON.stringify(result)}`);
    return result;
  };

  return descriptor;
}

class Calculator {
  @log
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(2, 3);
// Output:
// Calling add with args: [2,3]
// add returned: 5
```

This is a classic **method interception** pattern: the decorator wraps `descriptor.value` (the original function) inside a new function that adds logging before and after calling the original.

---

### **Property Decorators**

A property decorator receives the target and the property name; unlike method decorators, it doesn't receive (and can't directly modify) a property descriptor for instance fields, so it's typically used to record metadata rather than change behavior directly.

```typescript
function readonly(target: any, propertyKey: string) {
  console.log(`Marking ${propertyKey} as read-only (metadata only)`);
}

class Product {
  @readonly
  name: string = "Laptop";
}

const product = new Product();
console.log(product.name);
// Output:
// Marking name as read-only (metadata only)
// Laptop
```

For real enforcement of immutability on a property, you'd typically combine this with `Object.defineProperty` inside the decorator, or simply use TypeScript's own `readonly` modifier for compile-time enforcement.

---

### **Parameter Decorators**

A parameter decorator receives the target, the method name the parameter belongs to, and the parameter's index within the argument list. They're most commonly used to record metadata — for example, marking which constructor parameter should be injected by a dependency-injection framework.

```typescript
function logParameter(target: any, methodName: string, parameterIndex: number) {
  console.log(`Parameter at index ${parameterIndex} in ${methodName} is decorated`);
}

class UserService {
  greet(@logParameter greeting: string, name: string) {
    console.log(`${greeting}, ${name}!`);
  }
}

new UserService().greet("Hello", "Alice");
// Output:
// Parameter at index 0 in greet is decorated
// Hello, Alice!
```

---

### **Decorator Factories**

A **decorator factory** is a function that *returns* a decorator, letting you configure its behavior with arguments at the point of use — this is why you often see decorators called like functions, e.g. `@log("custom message")`.

```typescript
function log(message: string) {
  // This outer function is the factory — it returns the actual decorator
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = function (...args: any[]) {
      console.log(`[${message}] Calling ${propertyKey}`);
      return originalMethod.apply(this, args);
    };

    return descriptor;
  };
}

class OrderService {
  @log("ORDER")
  placeOrder(item: string) {
    console.log(`Order placed for ${item}`);
  }
}

new OrderService().placeOrder("Book");
// Output:
// [ORDER] Calling placeOrder
// Order placed for Book
```

Every plain decorator used so far (`@sealed`, `@log`, `@readonly`) could be rewritten as a factory to accept configuration — factories are simply decorators one level removed, called immediately to produce the actual decorator function TypeScript applies.

| Decorator Type | Applied To | Receives | Typical Use |
|---|---|---|---|
| **Class** | A class declaration | Constructor function | Sealing, freezing, registering the class |
| **Method** | A method on a class | Target, method name, property descriptor | Logging, memoization, access control |
| **Property** | A class field | Target, property name | Metadata (validation rules, serialization hints) |
| **Parameter** | A single function parameter | Target, method name, parameter index | Dependency injection metadata |
| **Factory** | Any of the above | Custom arguments, returns the actual decorator | Configurable versions of any decorator type |

---

### **Real-World Context: Angular and NestJS**

Decorators aren't just an academic feature — they're central to how two major TypeScript frameworks structure applications:

1. **Angular** uses class decorators like `@Component`, `@Injectable`, and `@NgModule` to declare what a class *is* to the framework (a component, an injectable service, a module) and to attach configuration (a component's template, selector, and styles) directly to the class.
2. **NestJS** uses a very similar pattern — `@Controller`, `@Injectable`, `@Get`, `@Post` — combined heavily with parameter decorators (`@Body`, `@Param`, `@Query`) to declare HTTP routes and automatically inject request data into handler methods, and constructor parameter decorators to drive its dependency injection system.

```typescript
// Conceptual NestJS-style example (illustrative, not runnable without NestJS installed)
@Controller("users")
class UserController {
  constructor(private userService: UserService) {} // constructor injection via decorators

  @Get(":id")
  getUser(@Param("id") id: string) {
    return this.userService.findById(id);
  }
}
```

Understanding the fundamentals covered in this file — what class, method, and parameter decorators receive, and how decorator factories work — makes reading and writing framework code like this far less mysterious.

---

### **Best Practices**
- Enable `experimentalDecorators` (and `emitDecoratorMetadata` if you need reflection-based metadata) explicitly in `tsconfig.json` before using decorators.
- Prefer decorator factories (`@log("message")`) over plain decorators (`@log`) whenever the decorator needs any configuration, even minimal configuration — it keeps the API consistent and extensible.
- Keep decorators focused on a single cross-cutting concern (logging, validation, sealing) rather than mixing multiple responsibilities into one decorator.
- Be aware that method decorators run once, at class-definition time — they wrap the method definition itself, not each individual call, so any setup logic inside the decorator function (outside the returned wrapper) only runs once.
- When adopting a new project, check whether your framework/tooling has moved to the newer TC39 Stage 3 decorators, since the two syntaxes aren't fully interchangeable.
- Avoid using decorators purely for style — they're best reserved for genuine cross-cutting concerns, not as a substitute for straightforward function composition.

---

### **Interview Questions**

**Q1. What is a decorator in TypeScript?**
A decorator is a special function, applied with `@expression` syntax, that can observe, modify, or replace a class, method, property, or parameter at the point it's declared. It runs at definition time, not at instantiation or call time.

**Q2. How do you enable decorators in a TypeScript project?**
By setting `"experimentalDecorators": true` in the `compilerOptions` of `tsconfig.json`. If you need reflection-based type metadata (as some DI frameworks do), you also enable `"emitDecoratorMetadata": true`.

**Q3. What does a class decorator receive as its argument, and what can it do?**
It receives the class's constructor function. It can inspect or add properties to it, or return an entirely new constructor function that replaces the original class definition.

**Q4. What three parameters does a method decorator function receive?**
The target (the class prototype for an instance method), the property key (the method's name), and the property descriptor, which includes the original method as `descriptor.value` and can be reassigned to wrap it.

**Q5. What is a decorator factory, and why would you use one?**
A decorator factory is a function that returns a decorator function, allowing you to pass configuration arguments at the call site, e.g. `@log("custom message")`. You use one whenever the decorator's behavior needs to be parameterized rather than fixed.
```typescript
function log(message: string) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) { /* ... */ };
}
```

**Q6. How would you implement a simple `@log` method decorator that logs a method's arguments and return value?**
Capture the original method from `descriptor.value`, replace `descriptor.value` with a new function that logs the arguments, calls the original method via `.apply(this, args)`, logs the result, and returns it.

**Q7. What does a parameter decorator receive, and what is it typically used for?**
It receives the target, the name of the method the parameter belongs to, and the numeric index of the parameter within the argument list. It's typically used to record metadata about that parameter — most notably for dependency-injection frameworks to know which constructor argument to inject.

**Q8. Name two frameworks that rely heavily on decorators, and give one example decorator from each.**
Angular relies on decorators like `@Component` and `@Injectable`. NestJS relies on decorators like `@Controller`, `@Get`, and `@Injectable` for defining HTTP routes and services.

**Q9. What is the difference between the "legacy" `experimentalDecorators` and the newer TC39 Stage 3 decorators in TypeScript 5.0+?**
The legacy decorators (enabled via `experimentalDecorators`) use TypeScript's original, non-standard decorator API and function signatures. The newer TC39 Stage 3 decorators are a now-standardized JavaScript proposal with a different, more restricted API, don't require any compiler flag, and are on a path to becoming native JavaScript — though most existing frameworks like Angular and NestJS still target the legacy style as of today.

**Q10. Can a class decorator replace the class entirely? Give a conceptual example of why you might do that.**
Yes — if a class decorator returns a new constructor function, that new constructor replaces the original class. You might do this to wrap the class in additional initialization logic or add new members while still preserving the original class's behavior through inheritance or composition inside the wrapper.

**Q11. Why can't a property decorator directly intercept reads and writes to a class field the way a method decorator intercepts calls?**
A property decorator for a class field doesn't receive a property descriptor with a `get`/`set` pair the way a method decorator receives one for its function — it primarily receives the target and property name, making it suited to recording metadata rather than transparently intercepting access. To intercept reads/writes, you'd typically use `Object.defineProperty` with explicit accessors inside the decorator, or convert the field to an accessor decorator.

**Q12. What's a real risk of overusing decorators in application code?**
Overusing decorators can hide important logic behind "magic" annotations, making it harder to trace what actually happens when a method is called or a class is instantiated, since the real behavior lives in decorator functions defined elsewhere. This can hurt readability and debuggability if not used judiciously for clear, well-scoped cross-cutting concerns.
