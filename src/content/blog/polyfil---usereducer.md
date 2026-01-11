---
title: "Polyfil - useReducer"
description: "Demystifying useReducer: Building Your Own Polyfill for Deeper..."
pubDate: "Jan 11 2026"
heroImage: "../../assets/polyfil---usereducer.jpg"
---

# Demystifying `useReducer`: Building Your Own Polyfill for Deeper Understanding

Remember those early days of React, before hooks graced our lives? Or maybe you're currently working on an older project where a full React version upgrade isn't an option, but you desperately crave the elegance and predictability that `useReducer` brings to complex state management. Or perhaps, like me, you just love pulling back the curtain to see how things *really* work.

Here’s the thing: `useReducer` isn't just a powerful hook; it's a design pattern that can radically simplify how you manage intricate component state. And understanding how you'd *build* something like `useReducer` yourself – essentially polyfilling it – isn't just an academic exercise. It's a masterclass in React fundamentals, stable function references, and the beauty of predictable state transitions.

## The Problem: `useState` Sprawl and the Call for Predictability

I've found myself in many situations where a component starts simple, maybe a counter or a toggle, perfect for `useState`. But then features get added: now it needs to manage a form with multiple fields, validation states, loading indicators, and error messages. Suddenly, you've got half a dozen `useState` calls, each with its own `setX` function.

```typescript
// Before useReducer...
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [isValid, setIsValid] = useState(true);
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<string | null>(null);

// ... and then managing all these setters in various event handlers.
```

This often leads to scattered logic, potential for inconsistent state updates, and frankly, a nightmare to refactor or debug. This is precisely where `useReducer` shines. It centralizes state logic into a single, pure reducer function, making state transitions explicit and predictable.

## Why "Polyfill" `useReducer`? It's About Understanding.

Now, let's be clear: in a modern React application, you should absolutely use `React.useReducer`. It's optimized, well-tested, and handles edge cases you wouldn't want to think about.

However, building your own `useReducer` from primitives like `useState` and `useCallback` offers incredible insights into:

1.  **How React Hooks Work Internally:** It strips away the magic and shows you the underlying mechanisms.
2.  **The Power of Pure Functions:** Reducers are pure, making them easy to test and reason about.
3.  **Stable Function References:** Understanding why `dispatch` needs to be stable and how to achieve it with `useCallback` and `useRef`.
4.  **Framework Agnosticism:** The `reducer` pattern itself is not React-specific; understanding it allows you to apply it in different contexts or even build libraries.

## A Practical Deep Dive: Building Our Own `useReducer`

Let's craft a simplified version of `useReducer`. Our goal is to mimic its API: `useReducer(reducer, initialState, initializer?)` returning `[state, dispatch]`.

```typescript
import React, { useState, useCallback, useRef, useEffect } from 'react';

type Reducer<S, A> = (state: S, action: A) => S;
type Initializer<S, InitialArg> = (initialArg: InitialArg) => S;

function useReducerPolyfill<S, A, InitialArg = S>(
  reducer: Reducer<S, A>,
  initialState: InitialArg,
  initializer?: Initializer<S, InitialArg>
): [S, (action: A) => void] {
  // 1. Manage the actual state using useState
  // If an initializer is provided, use it, otherwise use initialState directly.
  const [state, setState] = useState<S>(
    () => (initializer ? initializer(initialState) : (initialState as S))
  );

  // 2. We need a stable reference to the *current* reducer function.
  // Why? The dispatch function (which we want to be stable across renders)
  // needs to call the *latest* version of the reducer.
  const reducerRef = useRef(reducer);

  // Update the ref whenever the reducer function changes.
  // This ensures dispatch always has access to the most up-to-date reducer.
  useEffect(() => {
    reducerRef.current = reducer;
  }, [reducer]); // Dependency array includes 'reducer'

  // 3. Create a stable 'dispatch' function.
  // This is crucial for performance (avoiding unnecessary re-renders of child components
  // that receive dispatch as a prop) and correctness (stable references in effects).
  const dispatch = useCallback((action: A) => {
    // When dispatch is called, it updates the state by running
    // the current reducer function with the previous state and the action.
    setState(prev => reducerRef.current(prev, action));
  }, []); // Empty dependency array means 'dispatch' is created once and never changes.

  return [state, dispatch];
}
```

### Let's break down the key parts:

*   **`useState<S>(() => ...)`**: We use React's fundamental state hook to actually hold our state. The `()` => ...` syntax ensures that the `initializer` (or `initialState`) is only computed once on the initial render, just like native `useState` and `useReducer`.
*   **`reducerRef = useRef(reducer)` and `useEffect`**: This is where the magic happens for "stale closures". The `dispatch` function, thanks to `useCallback([])`, is only created once. If `dispatch` directly captured the `reducer` from its initial render scope, it would always call an outdated `reducer` if `reducer` itself changed (e.g., due to props). By storing the latest `reducer` in a `useRef`, our stable `dispatch` can always access the most current version. This is a common pattern for creating stable callbacks that need to access potentially changing values.
*   **`dispatch = useCallback(...)`**: Making `dispatch` stable is paramount. If `dispatch` changed on every render, any child component memoized with `React.memo` that received `dispatch` as a prop would re-render unnecessarily. An empty dependency array `[]` ensures it's created once and never changes.

### Example Usage

Let's use our `useReducerPolyfill` in a simple counter component:

```typescript
// Assume the useReducerPolyfill function is defined above or imported

interface CounterState {
  count: number;
}

type CounterAction =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset', payload: number };

const counterReducer = (state: CounterState, action: CounterAction): CounterState => {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: action.payload };
    default:
      // Always return the current state for unknown actions or throw an error
      return state;
  }
};

function CounterComponent() {
  // Using our polyfill instead of React.useReducer
  const [state, dispatch] = useReducerPolyfill(counterReducer, { count: 0 });

  return (
    <div style={{ padding: '20px', border: '1px solid #C9A227', borderRadius: '8px', maxWidth: '300px', margin: '20px auto', textAlign: 'center' }}>
      <h2>Polyfill Counter</h2>
      <p style={{ fontSize: '2em', fontWeight: 'bold', color: '#C9A227' }}>Count: {state.count}</p>
      <button style={{ margin: '5px', padding: '10px 15px', backgroundColor: '#333', color: 'white', border: 'none', borderRadius: '5px', cursor: 'pointer' }}
              onClick={() => dispatch({ type: 'increment' })}>Increment</button>
      <button style={{ margin: '5px', padding: '10px 15px', backgroundColor: '#333', color: 'white', border: 'none', borderRadius: '5px', cursor: 'pointer' }}
              onClick={() => dispatch({ type: 'decrement' })}>Decrement</button>
      <button style={{ margin: '5px', padding: '10px 15px', backgroundColor: '#333', color: 'white', border: 'none', borderRadius: '5px', cursor: 'pointer' }}
              onClick={() => dispatch({ type: 'reset', payload: 0 })}>Reset</button>
    </div>
  );
}

export default CounterComponent;
```

## Pitfalls and Considerations

While this exercise is enlightening, it's essential to acknowledge where a custom polyfill falls short compared to React's native `useReducer`:

*   **Performance:** React's internal `useReducer` is highly optimized. A custom `useState`-based solution might incur slight overheads that React's core efficiently avoids. For example, React batches state updates more aggressively.
*   **Edge Cases:** The native hook handles many subtle edge cases related to concurrent mode, server-side rendering, and strict mode that our simple polyfill doesn't account for.
*   **Bundle Size:** In a real application, you wouldn't include this if native `useReducer` is available. This is purely for learning or for environments where hooks are truly absent (which is rare for modern React).
*   **Immutability:** This pattern *demands* immutable state updates within your reducer. Always return new objects/arrays; never mutate the `state` directly. This isn't a pitfall of the polyfill, but a critical aspect of `useReducer` itself.

## Key Takeaways

Building this `useReducerPolyfill` might not be something you deploy to production (unless you have very specific, constrained environments), but the knowledge gained is invaluable.

It taught me:

1.  **The power of `useRef` and `useEffect` for managing mutable values within stable functions.** This pattern is applicable far beyond just polyfilling hooks.
2.  **Why `dispatch` functions are stable by default in React's hooks.** It's a fundamental optimization.
3.  **The elegance of the reducer pattern** itself, separating *what* happened (action) from *how* state changes (reducer logic).

In my experience, truly understanding how these fundamental pieces work under the hood makes you a much more confident and capable developer. It moves you from merely *using* tools to genuinely *understanding* them, which is where real innovation and problem-solving muscle come from. So, next time you're using `useReducer`, give a nod to the underlying mechanics – you now know a bit more about how that magic happens!

---

Feel free to experiment with this `useReducerPolyfill` in your local environment. Try making the `reducer` function change (e.g., based on a prop) and observe how `reducerRef` ensures correctness. It's a fantastic way to solidify these concepts.
