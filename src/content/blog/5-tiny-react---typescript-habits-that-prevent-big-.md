---
title: "5 Tiny React + TypeScript Habits That Prevent Big Bugs"
description: "5 Tiny React + TypeScript Habits That Prevent Big..."
pubDate: "Feb 27 2026"
heroImage: "../../assets/5-tiny-react---typescript-habits-that-prevent-big-.jpg"
---

# 5 Tiny React + TypeScript Habits That Prevent Big Bugs

We’ve all been there: staring at a bug report, scratching our heads, thinking, "But I'm using TypeScript! How did this even happen?" It’s a common misconception that simply *having* TypeScript in your React project is enough to ward off all evil. The truth, in my experience, is that TypeScript is a powerful ally, but it's only as effective as the habits you build around it.

It's not about complex patterns or academic type theory. It's about five small, consistent habits that, when integrated into your daily workflow, transform your code from "TypeScript-adjacent" to truly "TypeScript-guarded." These aren't just theoretical best practices; they're lessons learned from countless hours of debugging real-world applications where these small oversights caused disproportionately large headaches.

Let’s dive into these habits that prevent those big, nasty bugs before they even get a chance to compile.

---

### Habit 1: Always Explicitly Type Component Props (and embrace `null`!)

This might sound basic, but you’d be surprised how often `any` or implicit typing creeps into props. When TypeScript can’t infer a type, it defaults to `any`, and that's where bugs start their silent journey. The most common culprit? Missing explicit types for optional props or props that could legitimately be `null` or `undefined`.

**Why it matters:**
Without explicit prop typing, you're essentially telling TypeScript, "Don't worry about this, I've got it." But do you, really? Over time, components evolve, and if the contract (its props) isn't strictly defined, consuming components can pass anything they want, leading to runtime errors when the component expects something else.

**The Habit:**
Always define an `interface` or `type` for your component's props. And critically, consider when a prop might truly be `null` or `undefined` and explicitly add `| null` or `| undefined` to its type.

```typescript
// Bad habit: Implicit 'any' or vague typing
// function UserProfile({ user }) { /* ... */ }

// Good habit: Explicitly typed props
interface User {
  id: string;
  name: string;
  email?: string; // Optional email
}

interface UserProfileProps {
  user: User | null; // User can be null!
  isLoading: boolean;
}

const UserProfile: React.FC<UserProfileProps> = ({ user, isLoading }) => {
  if (isLoading) {
    return <p>Loading user data...</p>;
  }

  if (!user) {
    return <p>No user found.</p>;
  }

  return (
    <div>
      <h2>{user.name}</h2>
      {user.email && <p>Email: {user.email}</p>}
    </div>
  );
};
```
**Insights:** Modern React often skips `React.FC` for simpler function signatures, which is fine! The key is the explicit `UserProfileProps` interface. By adding `| null` to `user`, TypeScript immediately forces you to handle the `null` case, preventing an `Uncaught TypeError: Cannot read properties of null (reading 'name')` at runtime. This small addition makes your component robust from the get-go.

---

### Habit 2: Leverage `Partial<T>` for Safer Updater Functions

Working with forms, context providers, or simply updating parts of a complex state object? You often need to update only a subset of an object's properties. If you're not careful, TypeScript might complain about missing properties, or worse, you might inadvertently use `any`.

**Why it matters:**
Trying to update a `User` object with just a `{ name: 'New Name' }` might require you to cast `Partial<User>` or, even worse, resort to `any`. This can lead to incorrect or incomplete updates, especially if you have `required` fields that are not being passed in the update.

**The Habit:**
When you need to update only *some* properties of an existing object, define your update payload or function parameter using `Partial<T>`.

```typescript
interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
  description?: string;
}

// Imagine a function to update product details
type ProductUpdatePayload = Partial<Product>;

function updateProduct(productId: string, updates: ProductUpdatePayload) {
  // In a real app, this would be an API call or state update
  console.log(`Updating product ${productId} with:`, updates);
}

// Example Usage:
updateProduct("prod-123", { price: 29.99, stock: 150 }); // Perfectly valid
updateProduct("prod-456", { name: "New Widget Pro", description: "Improved version" }); // Also valid

// Bad habit: Trying to pass a full Product where only partial is needed, or using 'any'
// function updateProduct(productId: string, updates: any) { /* ... */ }
// updateProduct("prod-789", { name: "Old Widget" }); // No type safety here!
```
**Insights:** `Partial<T>` automatically makes all properties of `T` optional. This is incredibly useful for reducer actions, `setState` calls with object spread (`{ ...prevState, ...updates }`), or API client methods that allow partial updates. It enforces type safety without making you jump through hoops to satisfy strict type checks for properties you're intentionally omitting.

---

### Habit 3: Harness Discriminated Unions for Complex State Machines

This is where TypeScript truly shines in preventing an entire class of logic bugs. When your component or application state can exist in several distinct, mutually exclusive forms (e.g., loading, success, error), discriminated unions are your best friend.

**Why it matters:**
Without discriminated unions, you end up with "impossible states" – like `isLoading: true` and `data: [...]` simultaneously, or `error: 'Failed'` with `data: [...]`. This leads to tricky `if/else` ladders and runtime checks that TypeScript *could* have helped you with.

**The Habit:**
Define a union of interfaces, where each interface has a common literal property (the "discriminant") that TypeScript can use to narrow the type.

```typescript
// State for fetching data
type DataState =
  | { status: 'loading' }
  | { status: 'success'; data: string[] }
  | { status: 'error'; message: string };

const DataLoader: React.FC = () => {
  const [state, setState] = React.useState<DataState>({ status: 'loading' });

  React.useEffect(() => {
    // Simulate data fetching
    setTimeout(() => {
      if (Math.random() > 0.5) {
        setState({ status: 'success', data: ['Item A', 'Item B'] });
      } else {
        setState({ status: 'error', message: 'Failed to load data.' });
      }
    }, 1500);
  }, []);

  // TypeScript now knows exactly what properties are available based on `state.status`
  if (state.status === 'loading') {
    return <p>Loading data...</p>;
  }

  if (state.status === 'error') {
    // Here, `state` is guaranteed to have `message`
    return <p style={{ color: 'red' }}>Error: {state.message}</p>;
  }

  // Here, `state` is guaranteed to have `data`
  return (
    <div>
      <h3>Data Loaded:</h3>
      <ul>
        {state.data.map((item, index) => (
          <li key={index}>{item}</li>
        ))}
      </ul>
    </div>
  );
};
```
**Insights:** Notice how TypeScript's control flow analysis automatically narrows the `state` type within each `if` block. This prevents you from accidentally trying to access `state.data` when `status` is `'loading'` or `'error'`. This pattern is invaluable for reducers, async operations, and any scenario where state transitions are discrete and well-defined.

---

### Habit 4: Master `useRef` Typing for DOM Elements and Mutable Values

`useRef` is incredibly useful, but it's also a common breeding ground for `null` checks and `any` types if not handled correctly. Especially when interacting with DOM elements, it's easy to get the types wrong.

**Why it matters:**
A common pitfall is forgetting that `ref.current` can be `null` initially (before the component mounts or the ref is attached). If you don't account for this, you're back to runtime `null` errors. Also, `useRef` isn't just for DOM elements; it's for any mutable value you want to persist across renders without causing re-renders.

**The Habit:**
When using `useRef` for DOM elements, explicitly type it with `HTMLDivElement | null` (or whatever element type) and initialize it with `null`. For mutable values, provide the exact type.

```typescript
const FocusInput: React.FC = () => {
  // Good habit: Explicitly type useRef for a DOM element
  const inputRef = React.useRef<HTMLInputElement>(null);

  React.useEffect(() => {
    // Safely access current, TypeScript forces null check
    if (inputRef.current) {
      inputRef.current.focus();
    }
  }, []);

  const handleClick = () => {
    if (inputRef.current) {
      inputRef.current.value = "Focused!";
    }
  };

  // Good habit: useRef for a mutable value
  const counterRef = React.useRef(0); // Inferred as number, or useRef<number>(0)

  const incrementCounter = () => {
    counterRef.current += 1;
    console.log("Current counter value:", counterRef.current);
  };

  return (
    <div>
      <input type="text" ref={inputRef} />
      <button onClick={handleClick}>Focus Input</button>
      <button onClick={incrementCounter}>Increment Counter (see console)</button>
    </div>
  );
};
```
**Insights:** TypeScript forces the `if (inputRef.current)` check, which is precisely what you need to do at runtime anyway. This prevents `inputRef.current` from being `null` when you try to call methods like `focus()` on it. For mutable values, explicit typing can prevent accidental assignment of wrong types.

---

### Habit 5: Type Event Handlers Precisely

Another common `any` hotspot is event handlers. It’s easy to just `(e: any)` or `(e)` and let TypeScript infer a generic `Event`, but this loses all the specific event properties you might need.

**Why it matters:**
If you access `e.target.value` on a click event, TypeScript won't catch it unless you've correctly typed `e`. This leads to runtime errors when an `Event` object doesn't have the properties of, say, a `ChangeEvent<HTMLInputElement>`.

**The Habit:**
Use React's synthetic event types (e.g., `React.MouseEvent`, `React.ChangeEvent`) and specify the HTML element type.

```typescript
const FormComponent: React.FC = () => {
  const [inputValue, setInputValue] = React.useState('');

  // Good habit: Precisely typed change event for an input
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setInputValue(e.target.value);
  };

  // Good habit: Precisely typed click event for a button
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    // You can access specific button properties if needed, e.g., e.currentTarget.name
    console.log('Button clicked!', e.currentTarget.textContent);
  };

  // Good habit: Form submission event
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault(); // Prevents default browser form submission
    console.log('Form submitted with value:', inputValue);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={inputValue}
        onChange={handleChange}
        placeholder="Type something..."
      />
      <button type="button" onClick={handleClick}>
        Click Me
      </button>
      <button type="submit">Submit</button>
    </form>
  );
};
```
**Insights:** By explicitly typing the event, TypeScript knows exactly what properties are available on `e` and `e.target`. This prevents errors like trying to get `e.target.value` from a button click or accessing `e.preventDefault` on an event that doesn't have it. It’s a tiny bit more to type, but it pays dividends in reliability.

---

### Wrapping Up

These five habits aren't revolutionary, but their collective impact on codebase quality and bug prevention is immense. They teach you to think more critically about your data flow, component contracts, and potential edge cases *before* you even hit save.

In my journey with React and TypeScript, I've found that the real power isn't in understanding every advanced feature, but in consistently applying these fundamental, proactive type-safety practices. They're your invisible shield, quietly protecting your application from common pitfalls. Start adopting them today, and you'll find your debugging sessions becoming shorter, your code more predictable, and your confidence as a developer skyrocketing. Happy coding!
