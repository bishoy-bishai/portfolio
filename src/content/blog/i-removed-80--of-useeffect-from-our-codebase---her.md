---
title: "I Removed 80% of useEffect From Our Codebase — Here's What Happened"
description: "The Great Un-effecting: How We Slashed 80% of useEffect from Our React..."
pubDate: "Mar 03 2026"
heroImage: "../../assets/i-removed-80--of-useeffect-from-our-codebase---her.jpg"
---

# The Great Un-effecting: How We Slashed 80% of useEffect from Our React Codebase

Alright, folks, let's be real. If you've been building React applications for a while, you've probably got a soft spot, or perhaps a mild trauma, for `useEffect`. It's that powerful hook we all learned early on, the one that lets us "do stuff" after render, sync with external systems, and generally escape the declarative world of React when we need to. But here's the thing: while `useEffect` is incredibly powerful, it's also incredibly easy to overuse and misuse.

I've seen it time and again, in my own projects and those of others: `useEffect`s sprouting like weeds, each trying to manage a piece of state, fetch some data, or orchestrate some side effect. You end up with components that are hard to read, harder to debug, and often riddled with subtle performance traps or infinite re-render loops.

In one particular project, we hit a wall. Our main dashboard component, seemingly simple, had accumulated over a dozen `useEffect` calls. Performance was sluggish, and a particular bug where data wasn't updating reliably was driving us nuts. Every fix seemed to introduce a new edge case. It was a classic case of `useEffect` sprawl. That's when we decided enough was enough. We embarked on a journey to rigorously re-evaluate every single `useEffect` in our codebase. The result? We removed over 80% of them. And honestly, it felt like magic. Our code became leaner, faster, and surprisingly, a lot more fun to work with.

## The `useEffect` Trap: A Powerful Tool, Often Misunderstood

So, why does this happen? `useEffect` is often taught as the "do anything after render" hook, a catch-all for any imperative logic. Need to fetch data? `useEffect`. Need to set up an event listener? `useEffect`. Need to update state based on other state? You guessed it, `useEffect`.

The problem isn't `useEffect` itself; it's our mental model around it. As Dan Abramov famously put it, "A `useEffect` should ideally only do one thing: synchronize state with an external system." When we try to use it for *internal* React state management or for logic that could be handled declaratively or imperatively at a more appropriate time, that's when the complexity snowballs.

Let's look at some common `useEffect` misuses and better ways to approach them.

### Misuse #1: Data Fetching (The Most Common Culprit)

This is the poster child for `useEffect` misuse. We all started here:

```typescript
import React, { useState, useEffect } from 'react';

interface User {
  id: number;
  name: string;
}

function UserProfile({ userId }: { userId: number }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchUser = async () => {
      setLoading(true);
      setError(null);
      try {
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) {
          throw new Error('Failed to fetch user');
        }
        const data: User = await response.json();
        setUser(data);
      } catch (err) {
        setError((err as Error).message);
      } finally {
        setLoading(false);
      }
    };

    fetchUser();
  }, [userId]); // Dependency array often forgotten or misused

  if (loading) return <div>Loading user...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return <div>No user found.</div>;

  return (
    <div>
      <h2>{user.name}</h2>
      {/* ... more user details */}
    </div>
  );
}
```

This *works*, but it's not ideal. The component is now directly managing data fetching, caching concerns, and race conditions can easily creep in.

**The Better Way: Dedicated Data Fetching Libraries (React Query, SWR)**

In my experience, the single biggest game-changer for reducing `useEffect`s related to data fetching is adopting a robust client-side data fetching library like `react-query` (TanStack Query) or `SWR`. These libraries handle caching, revalidation, error handling, loading states, and even background fetching *for you*, turning imperative `useEffect` logic into simple, declarative custom hooks.

```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query'; // Assuming react-query is installed

interface User {
  id: number;
  name: string;
}

async function fetchUserById(userId: number): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);
  if (!response.ok) {
    throw new Error('Failed to fetch user');
  }
  return response.json();
}

function UserProfile({ userId }: { userId: number }) {
  const { data: user, isLoading, isError, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUserById(userId),
    enabled: !!userId, // Only fetch if userId is valid
  });

  if (isLoading) return <div>Loading user...</div>;
  if (isError) return <div>Error: {(error as Error).message}</div>;
  if (!user) return <div>No user found.</div>;

  return (
    <div>
      <h2>{user.name}</h2>
      {/* ... more user details */}
    </div>
  );
}
```

Zero `useEffect`s in your component. Cleaner, more robust, and handles so many edge cases for free. If you're not using one of these, you're truly missing out.

### Misuse #2: Deriving State or Reacting to Internal State Changes

Sometimes, we find ourselves using `useEffect` to update one piece of state when another piece of state changes.

```typescript
function ProductDisplay({ quantity, price }: { quantity: number; price: number }) {
  const [total, setTotal] = useState(0);

  useEffect(() => {
    setTotal(quantity * price);
  }, [quantity, price]); // Updates 'total' whenever 'quantity' or 'price' changes

  return (
    <div>
      <p>Quantity: {quantity}</p>
      <p>Price: ${price}</p>
      <h3>Total: ${total.toFixed(2)}</h3>
    </div>
  );
}
```

**The Better Way: Derive State Directly During Render**

If state can be derived from props or other state, do it directly in the render function. This is more declarative and avoids unnecessary renders and state updates.

```typescript
function ProductDisplay({ quantity, price }: { quantity: number; price: number }) {
  // total is directly derived, not stored in state
  const total = quantity * price;

  return (
    <div>
      <p>Quantity: {quantity}</p>
      <p>Price: ${price}</p>
      <h3>Total: ${total.toFixed(2)}</h3>
    </div>
  );
}
```

Simple, right? No `useEffect`, no extra state, no potential for stale closures or dependency array bugs. This is a critical mental model shift: **compute what you can, store what you must.**

For more complex derivations or expensive computations, `useMemo` is your friend, but even then, it's about memoizing a *calculation*, not triggering a side effect.

### Misuse #3: Event Listeners and Manual DOM Manipulation (Often Wrapped in Custom Hooks)

While `useEffect` is *designed* for synchronizing with external systems like the DOM, we often create bulky `useEffect`s in components when a custom hook could abstract that logic away.

```typescript
function MyComponent() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []); // Only runs once on mount
  
  // ... rest of component
}
```

**The Better Way: Encapsulate in Reusable Custom Hooks**

While `useEffect` is correctly used here, placing this directly in `MyComponent` means `MyComponent` has to know about window resizing logic. Abstract it!

```typescript
import { useState, useEffect } from 'react';

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []); 

  return width;
}

function MyComponent() {
  const width = useWindowWidth(); // Simple, clean usage

  return (
    <div>
      <p>Current window width: {width}px</p>
      {/* ... rest of component */}
    </div>
  );
}
```

Now, `MyComponent` is only concerned with rendering. The side effect logic is isolated and reusable. This significantly cleans up component logic, making it easier to test and reason about.

## When *Does* `useEffect` Make Sense?

After all this talk about removing `useEffect`, it's important to remember it's not evil. It has a crucial role. `useEffect` is ideal for:

1.  **Synchronizing with external systems:**
    *   Subscribing to external stores (e.g., a Redux store, a WebSocket, a browser API).
    *   Interacting with the DOM directly (e.g., focusing an input, measuring element size, integrating a non-React library like a D3 chart).
    *   Logging or analytics calls that are pure side effects.
2.  **When there's a cleanup phase:** If you set up an event listener, a timer, or an observable subscription, `useEffect`'s cleanup function (`return () => {}`) is invaluable for preventing memory leaks.
3.  **Third-party library integration:** When a library expects an imperative setup or teardown based on React state/props.

The key distinction: Is this effect orchestrating something *within* React, or is it reaching *outside* of React? If it's reaching outside, `useEffect` is likely the right tool. If it's internal, there's probably a more declarative or direct way.

## Pitfalls to Avoid During the Great Un-effecting

*   **Don't just delete and move imperative code elsewhere haphazardly:** The goal is *simplification*, not just refactoring without thought. Ensure the new location for the logic is more appropriate and doesn't just push the complexity around.
*   **Be wary of over-optimizing too early:** If a `useEffect` is simple, clear, and performing well, don't necessarily rip it out just for the sake of it. Focus on the ones causing pain points (bugs, performance, readability).
*   **Understand the alternatives deeply:** Before removing a `useEffect`, make sure you fully grasp the alternative pattern (e.g., `useQuery`, `useMemo`, direct derivation, custom hooks, event handlers). A half-baked alternative can introduce new bugs.
*   **Test rigorously:** Refactoring `useEffect`s can change component behavior. Ensure you have good tests in place or implement them as you refactor.

## The Payoff: Clarity, Performance, and Developer Happiness

The journey of cutting down our `useEffect` footprint wasn't just an academic exercise. The immediate benefits were tangible:

*   **Improved Readability:** Components became much easier to scan and understand their primary purpose.
*   **Fewer Bugs:** Race conditions, stale closures, and infinite loops (often `useEffect` culprits) dramatically decreased.
*   **Better Performance:** Fewer unnecessary re-renders and state updates led to snappier UIs.
*   **Easier Maintenance:** When a bug did arise, tracking it down in a cleaner component was significantly faster.
*   **Higher Developer Confidence:** We spent less time scratching our heads trying to debug complex effect chains.

So, next time you reach for `useEffect`, pause for a moment. Ask yourself: "Is this truly synchronizing with an external system, or am I trying to manage internal React state imperatively?" More often than not, you'll find a simpler, more declarative, and ultimately more "React-y" way to achieve your goal. Start small, pick one component with a lot of effects, and see what you can simplify. You might be surprised by how much cleaner your codebase can become.
