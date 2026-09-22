**Asynchronous Programming in TypeScript** builds directly on JavaScript's promises and `async`/`await` (see [../JavaScript/AsynchronousJavaScript.md](../JavaScript/AsynchronousJavaScript.md) for the runtime mechanics), but adds full type checking on top: the type a promise resolves to, the type of a caught error, and the shape of data returned from asynchronous operations like `fetch()`. Getting async code properly typed is what lets the compiler catch bugs like forgetting to `await` a promise, or mishandling an API response shape, before your code ever runs.

---

### **Typing Promises: `Promise<T>`**

A `Promise` in TypeScript is generic — `Promise<T>` describes a promise that will eventually resolve to a value of type `T`.

```typescript
function fetchGreeting(): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => resolve("Hello, TypeScript!"), 100);
  });
}

fetchGreeting().then((message) => {
  console.log(message.toUpperCase()); // message is known to be a string
});
// Output: HELLO, TYPESCRIPT!
```

If a promise resolves with no meaningful value, its type is `Promise<void>`:

```typescript
function logAfterDelay(message: string): Promise<void> {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log(message);
      resolve();
    }, 100);
  });
}
```

---

### **`async`/`await` in TypeScript**

The syntax for `async`/`await` is identical to JavaScript — TypeScript's contribution is automatically wrapping the declared return type in a `Promise`.

```typescript
async function getGreeting(): Promise<string> {
  return "Hello!"; // TypeScript wraps this in Promise<string> automatically
}

async function main() {
  const greeting = await getGreeting(); // greeting: string, not Promise<string>
  console.log(greeting);
}

main();
// Output: Hello!
```

Notice two things:
1. **Declaring the return type as `Promise<string>`** (not just `string`) reflects what the function actually returns when called — an `async` function *always* returns a promise at runtime, even if the body just does `return "Hello!"`.
2. **`await` unwraps the promise**, so inside `main`, `greeting` is a plain `string`, not a `Promise<string>`.

If you omit the explicit return type, TypeScript infers it correctly on its own:

```typescript
async function add(a: number, b: number) {
  return a + b; // inferred return type: Promise<number>
}
```

Being explicit about return types on exported/public async functions is still good practice — it documents the contract and catches accidental type drift.

---

### **Typing the Resolved Value of `fetch()`**

`fetch()` resolves to a `Response` object; `response.json()` resolves to `any` by default, because the runtime has no way to know the shape of the JSON body. You bridge that gap by defining an interface for the expected shape and asserting the parsed result to it.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

async function getUser(id: number): Promise<User> {
  const response = await fetch(`https://api.example.com/users/${id}`);
  const data = (await response.json()) as User; // asserting the shape
  return data;
}
```

#### **A Safer Alternative: Runtime Validation**
Casting with `as` only affects compile-time checking — it does nothing to verify the data actually matches the shape at runtime. For untrusted external data, combine the type assertion with a runtime check (a user-defined type guard, as covered in [AdvancedTypes.md](./AdvancedTypes.md)) or a validation library:

```typescript
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value &&
    "email" in value
  );
}

async function getUserSafe(id: number): Promise<User> {
  const response = await fetch(`https://api.example.com/users/${id}`);
  const data: unknown = await response.json();
  if (!isUser(data)) {
    throw new Error("Unexpected API response shape");
  }
  return data; // narrowed to User
}
```

#### **A Generic `fetchJson<T>` Helper**
```typescript
async function fetchJson<T>(url: string): Promise<T> {
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`);
  }
  return response.json() as Promise<T>;
}

interface Post {
  id: number;
  title: string;
}

async function loadPost(id: number) {
  const post = await fetchJson<Post>(`https://api.example.com/posts/${id}`);
  console.log(post.title);
}
```

---

### **Error Handling in Typed Async Code**

#### **Why `catch (err: unknown)` — Not `any`**
Since TypeScript 4.4, caught errors in a `catch` block are typed as `unknown` by default (under `useUnknownInCatchVariables`, on by default with `strict: true`), rather than `any`. This is because JavaScript allows you to `throw` literally any value — not just `Error` instances — so the compiler can't safely assume anything about it.

```typescript
async function riskyOperation(): Promise<void> {
  throw new Error("Something went wrong");
}

async function run() {
  try {
    await riskyOperation();
  } catch (err: unknown) {
    // err.message; // Error: 'err' is of type 'unknown' — must narrow first
    if (err instanceof Error) {
      console.log(err.message); // safely narrowed to Error
    } else {
      console.log("Unknown error:", err);
    }
  }
}

run();
// Output: Something went wrong
```

`unknown` forces you to prove what the error actually is before using it — with `any`, you'd get no such protection, and code like `err.message` would silently compile even if `err` were a string or `undefined`, crashing at runtime.

#### **A Reusable Error-Narrowing Helper**
```typescript
function getErrorMessage(err: unknown): string {
  if (err instanceof Error) return err.message;
  if (typeof err === "string") return err;
  return "An unknown error occurred";
}

async function run() {
  try {
    await riskyOperation();
  } catch (err) {
    console.log(getErrorMessage(err));
  }
}
```

---

### **`Promise.all` with Typed Tuples**

`Promise.all` accepts an array (or tuple) of promises and resolves once all of them resolve. TypeScript infers a tuple type matching each promise's resolved type, in order.

```typescript
async function getUserName(): Promise<string> {
  return "Alice";
}
async function getUserAge(): Promise<number> {
  return 30;
}
async function getIsActive(): Promise<boolean> {
  return true;
}

async function loadUserData() {
  const [name, age, isActive] = await Promise.all([
    getUserName(),
    getUserAge(),
    getIsActive(),
  ]);
  // name: string, age: number, isActive: boolean — each correctly typed

  console.log(`${name}, ${age}, active: ${isActive}`);
}

loadUserData();
// Output: Alice, 30, active: true
```

Because TypeScript tracks each element's type individually (rather than collapsing them into one union type), destructuring the result gives you precisely typed variables — no manual casting needed.

---

### **End-to-End Example: A Strongly-Typed API Fetch**

```typescript
interface WeatherResponse {
  city: string;
  temperatureCelsius: number;
  conditions: string;
}

class WeatherApiError extends Error {
  constructor(message: string, public statusCode: number) {
    super(message);
  }
}

function isWeatherResponse(value: unknown): value is WeatherResponse {
  return (
    typeof value === "object" &&
    value !== null &&
    "city" in value &&
    "temperatureCelsius" in value &&
    "conditions" in value
  );
}

async function getWeather(city: string): Promise<WeatherResponse> {
  const response = await fetch(`https://api.example.com/weather?city=${city}`);

  if (!response.ok) {
    throw new WeatherApiError(`Failed to fetch weather for ${city}`, response.status);
  }

  const data: unknown = await response.json();

  if (!isWeatherResponse(data)) {
    throw new Error("Unexpected weather API response shape");
  }

  return data;
}

async function printWeather(city: string) {
  try {
    const weather = await getWeather(city);
    console.log(`${weather.city}: ${weather.temperatureCelsius}°C, ${weather.conditions}`);
  } catch (err: unknown) {
    if (err instanceof WeatherApiError) {
      console.log(`API error (${err.statusCode}): ${err.message}`);
    } else if (err instanceof Error) {
      console.log(`Error: ${err.message}`);
    } else {
      console.log("An unexpected error occurred");
    }
  }
}

printWeather("London");
// Output (example): London: 18°C, Cloudy
```

This example ties together everything above: a typed resolved value (`WeatherResponse`), a custom error class, a type guard for validating untrusted JSON, `async`/`await` with an explicit `Promise<T>` return type, and safe `unknown`-based error handling.

---

### **Best Practices**
- Always type the resolved value of a promise-returning function explicitly (`Promise<T>`) on public/exported functions, even though TypeScript can often infer it.
- Never trust `response.json()` at face value — either validate it with a type guard or use a schema validation library before treating it as a typed value.
- Handle caught errors as `unknown`, and always narrow with `instanceof Error` (or a custom check) before accessing properties on them.
- Prefer `Promise.all` for independent async operations that can run concurrently, and destructure its result for precise per-value types.
- Create custom `Error` subclasses (like `WeatherApiError`) when you need to carry structured information (status codes, error codes) alongside the error message.
- Avoid mixing `.then()` chains and `async`/`await` in the same function — pick one style per function for readability.

---

### **Interview Questions**

**Q1. What does `Promise<T>` represent, and how do you type a function that returns one?**
`Promise<T>` represents a promise that will eventually resolve to a value of type `T`. You annotate an async function's return type as `Promise<T>` (or let TypeScript infer it), even though inside the function you just `return` a plain `T` value.

**Q2. Why does an `async` function's return type get wrapped in `Promise<...>` automatically?**
Because calling any `async` function always returns a promise at runtime, regardless of what the function body returns directly. TypeScript reflects that runtime behavior by wrapping the declared/inferred return type in `Promise<>`.

**Q3. Why is the type of a caught error `unknown` rather than `any` by default in modern TypeScript?**
Because JavaScript's `throw` statement allows throwing any value at all — not just `Error` objects — so the compiler cannot safely assume a caught value has any particular shape. `unknown` forces you to narrow the value (e.g. with `instanceof Error`) before using it, preventing runtime crashes from unchecked property access.
```typescript
catch (err: unknown) {
  if (err instanceof Error) console.log(err.message);
}
```

**Q4. How do you safely type the result of `response.json()` from a `fetch()` call?**
`response.json()` returns `Promise<any>` by default, so you either cast the result with `as` to your expected interface (compile-time only), or better, write a runtime type guard that verifies the actual shape of the parsed value before trusting it as that type.

**Q5. What's the difference between using `as SomeType` and a user-defined type guard for validating fetched data?**
`as SomeType` is a compile-time-only assertion — it tells the compiler to trust you, but performs no runtime check, so mismatched data still passes through silently. A type guard function actually inspects the value at runtime and only lets you treat it as the target type if the checks pass, catching real mismatches.

**Q6. How does `Promise.all` type its resolved array when given promises of different types?**
TypeScript infers a tuple type where each position matches the resolved type of the corresponding promise in the input array, preserving individual types (e.g. `[string, number, boolean]`) instead of collapsing them into a single union.

**Q7. What happens if you forget to `await` an async function call in TypeScript — does the compiler catch it?**
TypeScript will generally let you assign a `Promise<T>` to a variable typed loosely or use it in ways that don't strictly require `T`, so it won't always flag a missing `await` directly; but if you try to use the resolved value's members directly on the promise object (e.g. calling a `string` method on a `Promise<string>`), you'll get a clear type error, which often surfaces the mistake.

**Q8. Why might you define a custom `Error` subclass like `WeatherApiError` instead of throwing a plain `Error`?**
A custom subclass lets you attach structured, strongly-typed extra information (like a `statusCode`) to the error, and lets calling code use `instanceof` to distinguish it from other error types and handle it specifically.

**Q9. How do you write a function generic over the shape of JSON it fetches?**
By making the function generic, e.g. `async function fetchJson<T>(url: string): Promise<T>`, and calling it with an explicit type argument at each call site, e.g. `fetchJson<Post>(url)`.

**Q10. What does `useUnknownInCatchVariables` do, and is it on by default?**
It's a TypeScript compiler option (introduced with TS 4.4) that types caught exceptions as `unknown` instead of `any`. It's enabled automatically when `strict: true` is set, which is the recommended baseline for new projects.

**Q11. If an async function doesn't explicitly return anything, what is its inferred return type?**
`Promise<void>`, reflecting that the function returns a promise that resolves with no meaningful value.

**Q12. What's a downside of relying solely on TypeScript types (without runtime validation) for data coming from an external API?**
TypeScript types are erased at compile time and provide zero runtime protection — if the actual API response doesn't match the declared interface (due to a backend change, bug, or malformed data), the mismatch won't be caught until it causes a runtime error somewhere downstream, since nothing verified the shape when the data was received.
