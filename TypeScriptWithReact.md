**TypeScript with React** combines React's component model with TypeScript's static typing, letting you catch prop mismatches, invalid state updates, and incorrect event handler signatures at compile time instead of discovering them at runtime. This file assumes familiarity with React fundamentals (see [../ReactJS/Hooks.md](../ReactJS/Hooks.md)) and TypeScript fundamentals, and focuses specifically on how to type the patterns you'll use in nearly every React + TypeScript component.

---

### **Setting Up a React + TypeScript Project**

The two most common ways to start a new React + TypeScript project today:

#### **Vite (recommended for new projects)**
```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

#### **Create React App (legacy, still seen in older projects)**
```bash
npx create-react-app my-app --template typescript
```

Both scaffold a project with a preconfigured `tsconfig.json`, `.tsx` file extensions for components containing JSX, and the `@types/react`/`@types/react-dom` packages already installed so React's own APIs are fully typed out of the box.

---

### **Typing Function Component Props**

The standard approach is to define an interface (or type alias) describing the component's props, then use it to type the function's parameter.

```typescript
interface ButtonProps {
  label: string;
  onClick: () => void;
}

const Button = ({ label, onClick }: ButtonProps) => {
  return <button onClick={onClick}>{label}</button>;
};
```

#### **`React.FC<Props>` vs. Typing Props Directly**
You'll also commonly see the `React.FC` (FunctionComponent) generic type used instead:

```typescript
const Button: React.FC<ButtonProps> = ({ label, onClick }) => {
  return <button onClick={onClick}>{label}</button>;
};
```

| Approach | Pros | Cons |
|---|---|---|
| **`React.FC<Props>`** | Explicitly types the component as a whole; used to implicitly type `children` in older React type versions | Implicitly adds a `children` prop (removed in React 18 types, but still a legacy quirk); slightly more awkward with generics; less flexible with default props |
| **Typing props directly** | Simpler, more explicit about exactly what props the component accepts; plays more naturally with generics and default parameter values | Return type isn't explicitly annotated as a component (rarely an issue in practice) |

**Current community preference** has shifted toward typing the props parameter directly (without `React.FC`), since it avoids `React.FC`'s implicit `children` prop and integrates more cleanly with generic components and default values:

```typescript
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
}

function Button({ label, onClick, disabled = false }: ButtonProps) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {label}
    </button>
  );
}
```

Both styles are valid and you'll see both in real codebases — but for new code, typing props directly is the more common modern recommendation.

---

### **Typing `useState`**

TypeScript can often infer the state type from the initial value passed to `useState`.

```typescript
const [count, setCount] = useState(0); // inferred as number
const [name, setName] = useState("");  // inferred as string
```

#### **When You Need the Explicit Generic**
Inference isn't enough when the initial value doesn't fully describe the type the state can eventually hold — most commonly, when the state starts as `null` but can later hold an object.

```typescript
interface User {
  id: number;
  name: string;
}

const [user, setUser] = useState<User | null>(null); // explicit generic needed

useEffect(() => {
  fetchUser().then((fetchedUser) => setUser(fetchedUser));
}, []);

if (user) {
  console.log(user.name); // safely narrowed: user is User here, not null
}
```

Without the explicit `<User | null>`, TypeScript would infer the state's type as just `null`, making it impossible to ever assign a real `User` object to it.

---

### **Typing `useRef`**

`useRef` is used for two distinct purposes, each typed differently.

#### **Referencing a DOM Element**
```typescript
function TextInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  const focusInput = () => {
    inputRef.current?.focus(); // optional chaining: current may be null before mount
  };

  return (
    <div>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus</button>
    </div>
  );
}
```

Passing `null` as the initial value along with an element type (`HTMLInputElement`) is the standard pattern for DOM refs — React sets `.current` once the element mounts, and it's `null` before that and after unmount.

#### **Storing a Mutable Value**
```typescript
function Timer() {
  const intervalRef = useRef<number | undefined>(undefined);

  const start = () => {
    intervalRef.current = window.setInterval(() => {
      console.log("tick");
    }, 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
  };

  return (
    <div>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

For mutable values (not DOM elements), `useRef`'s `.current` is directly writable and doesn't trigger a re-render when changed — useful for storing timer IDs, previous values, or any mutable data that shouldn't drive rendering.

---

### **Typing `useReducer`**

`useReducer` benefits heavily from a discriminated union for its action type (see [AdvancedTypes.md](./AdvancedTypes.md) for the full discriminated union pattern), which lets TypeScript check that each action carries the right payload.

```typescript
interface CounterState {
  count: number;
}

type CounterAction =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "reset"; payload: number };

function reducer(state: CounterState, action: CounterAction): CounterState {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    case "reset":
      return { count: action.payload }; // payload only exists on the 'reset' variant
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset", payload: 0 })}>Reset</button>
    </div>
  );
}
```

Because `CounterAction` is a discriminated union, TypeScript only allows `action.payload` inside the `"reset"` branch — trying to access it under `"increment"` would be a compile error, and dispatching `{ type: "reset" }` without a `payload` would also fail to compile.

---

### **Typing Event Handlers**

React re-exports typed versions of DOM events through the `React` namespace, parameterized by the element they're attached to.

```typescript
function SearchInput() {
  const [query, setQuery] = useState("");

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value); // e.target is correctly typed as HTMLInputElement
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    console.log("Searching for:", query);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={query} onChange={handleChange} />
      <button type="submit">Search</button>
    </form>
  );
}
```

Common event types:

| Event Type | Used For |
|---|---|
| `React.ChangeEvent<HTMLInputElement>` | `<input>`, `<textarea>`, `<select>` `onChange` |
| `React.FormEvent<HTMLFormElement>` | `<form>` `onSubmit` |
| `React.MouseEvent<HTMLButtonElement>` | `onClick` on a button (or other element) |
| `React.KeyboardEvent<HTMLInputElement>` | `onKeyDown`/`onKeyUp` on an input |

---

### **Typing Children Props**

The `children` prop is typed with `React.ReactNode`, which covers everything React can render — elements, strings, numbers, fragments, arrays of nodes, `null`, and `undefined`.

```typescript
interface CardProps {
  title: string;
  children: React.ReactNode;
}

function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>
    </div>
  );
}

function App() {
  return (
    <Card title="Welcome">
      <p>This is the card body.</p>
    </Card>
  );
}
```

---

### **Typing Custom Hooks**

A custom hook is typed like any other function — often generically, when it needs to work with different data shapes.

```typescript
interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
}

function useFetch<T>(url: string): FetchState<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;

    setLoading(true);
    fetch(url)
      .then((res) => res.json())
      .then((json: T) => {
        if (!cancelled) {
          setData(json);
          setLoading(false);
        }
      })
      .catch((err: unknown) => {
        if (!cancelled) {
          setError(err instanceof Error ? err.message : "Unknown error");
          setLoading(false);
        }
      });

    return () => {
      cancelled = true;
    };
  }, [url]);

  return { data, loading, error };
}
```

Using it in a component with a specific type argument:

```typescript
interface Post {
  id: number;
  title: string;
}

function PostList() {
  const { data: posts, loading, error } = useFetch<Post[]>("/api/posts");

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <ul>
      {posts?.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

`useFetch<Post[]>` gives full type safety end to end — `posts` is correctly known as `Post[] | null`, so accessing `.title` on each item is checked at compile time.

---

### **Best Practices**
- Type component props with an interface or type alias and destructure them in the function signature, rather than typing the whole component with `React.FC` — it's the more common modern pattern and avoids the implicit `children` quirk.
- Provide an explicit generic to `useState` whenever the initial value doesn't represent the full range of values the state can hold (e.g. `useState<User | null>(null)`).
- Always initialize DOM `useRef`s with `null` and the correct element type (`useRef<HTMLInputElement>(null)`), and use optional chaining (`inputRef.current?.focus()`) when accessing `.current`.
- Model `useReducer` actions as a discriminated union so each action's payload is only accessible in the branch where it's valid.
- Use the specific `React.*Event<ElementType>` type for each event handler rather than `any`, to get correctly typed `event.target`/`event.currentTarget`.
- Type `children` as `React.ReactNode` rather than `JSX.Element`, since components can legitimately render strings, numbers, arrays, and `null` as children too.
- Make custom hooks generic (`useFetch<T>`) when they need to support multiple data shapes across different call sites.

---

### **Interview Questions**

**Q1. What's the standard way to type a function component's props in modern TypeScript + React code?**
Define an interface (or type alias) describing the props, then destructure and type the function's single parameter with it, e.g. `function Button({ label, onClick }: ButtonProps)`, rather than wrapping the whole component type in `React.FC<Props>`.

**Q2. What is `React.FC<Props>`, and what's a downside of using it?**
`React.FC` is a generic type representing a function component, used as `const Component: React.FC<Props> = (props) => {...}`. A notable downside is that older versions of its type definition implicitly added a `children` prop to every component, even ones that don't accept children, which the community now generally avoids by typing props directly instead.

**Q3. When do you need to provide an explicit generic to `useState`, rather than letting TypeScript infer it?**
When the initial value doesn't represent every type the state variable can eventually hold — most commonly when state starts as `null` but will later be set to an object, e.g. `useState<User | null>(null)`.

**Q4. How do you type a `useRef` used to reference a DOM input element, and why is the initial value `null`?**
`useRef<HTMLInputElement>(null)` — the initial value is `null` because the DOM element doesn't exist yet when the component first renders; React assigns the actual element to `.current` once it mounts.

**Q5. Why is a discriminated union a good fit for typing `useReducer` actions?**
Because each action type often carries a different (or no) payload, a discriminated union with a shared `type` field lets TypeScript narrow to the exact action shape inside each `switch` case, ensuring you can only access a payload where it actually exists.

**Q6. How do you type an `onChange` handler for a text input?**
`(e: React.ChangeEvent<HTMLInputElement>) => void`, which correctly types `e.target` as an `HTMLInputElement` so properties like `e.target.value` are type-checked.

**Q7. What type should you use for a `children` prop, and why not `JSX.Element`?**
`React.ReactNode`, because it covers everything React can actually render as children — elements, strings, numbers, arrays, fragments, `null`, and `undefined` — whereas `JSX.Element` only covers a single rendered element and would reject valid children like plain text or `null`.

**Q8. How would you type a generic custom hook like `useFetch` that can be used with different response shapes?**
Give the hook its own type parameter, e.g. `function useFetch<T>(url: string): { data: T | null; loading: boolean }`, and have callers supply the specific type at the call site, e.g. `useFetch<Post[]>(url)`.

**Q9. What's the difference between using `useRef` for a DOM element and using it to store a mutable value like a timer ID?**
Both use the same hook, but a DOM ref is initialized with `null` and an element type (`useRef<HTMLInputElement>(null)`) and gets set automatically by React on mount. A mutable-value ref is initialized with the value's own type (e.g. `useRef<number | undefined>(undefined)`) and is set and read manually in your own code — in neither case does updating `.current` trigger a re-render.

**Q10. Why does `catch` inside a custom hook's fetch logic type the error as `unknown` rather than `any`?**
Because in modern TypeScript, caught errors default to `unknown` (see [AsyncProgrammingInTypeScript.md](./AsyncProgrammingInTypeScript.md)), reflecting that a thrown value could be anything, not necessarily an `Error` instance — you must narrow it (e.g. with `err instanceof Error`) before safely accessing properties like `.message`.

**Q11. How do you set up a new React + TypeScript project today?**
The most common modern approach is Vite with the `react-ts` template: `npm create vite@latest my-app -- --template react-ts`. Create React App with `--template typescript` also works but is considered legacy tooling at this point.

**Q12. What does `event.target` versus `event.currentTarget` typically refer to in a typed React event handler, and how does TypeScript help here?**
`event.target` is the actual DOM element that triggered the event (which could be a child of the element the handler is attached to), while `event.currentTarget` is always the element the handler was attached to. Typing the event as `React.MouseEvent<HTMLButtonElement>` (for example) ensures `event.currentTarget` is correctly typed as `HTMLButtonElement`, while `event.target` is typed more generically since it could technically be a descendant element.
