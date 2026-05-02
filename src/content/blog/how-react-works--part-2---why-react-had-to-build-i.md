---
title: "How React Works (Part 2)? Why React Had to Build Its Own Execution Engine"
description: "Decoding React's Engine: Why It Built Its Own..."
pubDate: "May 02 2026"
heroImage: "../../assets/how-react-works--part-2---why-react-had-to-build-i.jpg"
---

# Decoding React's Engine: Why It Built Its Own Scheduler

Ever built a React app that, despite all your best efforts, just *felt* a little… janky? Maybe a complex animation stutters, or a rapid state update feels less than immediate. We've all been there. It’s frustrating when you know you’re using a "fast" framework, yet the user experience isn't quite buttery smooth. The truth is, sometimes the bottleneck isn't your code logic, but a fundamental challenge React faces when interacting with the browser itself.

In my experience, many developers understand React's component model, props, state, and hooks. That’s "Part 1" of how React works. But understanding *why* React had to build its own sophisticated execution engine – the Fiber reconciler – and *how* it orchestrates updates, is "Part 2." It's where the real magic, and the deepest insights into performance, lie.

## The Browser's Dilemma: Why React Needed Its Own Path

Here's the thing: browsers were not initially designed for the highly dynamic, component-driven UIs we build today. Their DOM (Document Object Model) API, while functional, is inherently slow and *blocking*. Every time you manipulate the DOM directly – adding an element, changing a style, updating text – the browser might have to perform costly operations like recalculating layouts (reflows) and redrawing pixels (repaints). Do too much of this in quick succession, and your JavaScript execution thread gets tied up, blocking user input, freezing animations, and leading to that dreaded "jank."

Imagine you have a complex dashboard with many widgets updating simultaneously. If each widget tried to update the DOM independently and immediately, the browser would be overwhelmed, constantly reflowing and repainting. The UI would flicker and become unresponsive. This is the core problem React set out to solve: how to manage these updates efficiently and non-blockingly.

## Enter the Virtual DOM: A Necessary Abstraction

React's initial answer was the Virtual DOM (VDOM). Instead of directly manipulating the browser's DOM, React builds a lightweight, in-memory JavaScript representation of your UI. When state changes, React builds a *new* VDOM tree and compares it to the previous one. This process, known as **diffing** or **reconciliation**, identifies the minimal set of changes required to update the actual browser DOM.

The genius here is that diffing the VDOM is incredibly fast because it's just plain JavaScript objects. Only after it figures out *exactly* what needs to change does it apply those changes in a single, optimized batch to the real DOM. This dramatically reduces costly browser operations.

But simply knowing *what* to change isn’t enough. The *when* and *how* those changes are applied is equally crucial, especially for complex applications. This is where React's custom execution engine truly shines.

## The Heart of the Engine: React Fiber

The reconciliation algorithm itself underwent a massive re-architecture in React 16, moving from the "Stack" reconciler to **Fiber**. Fiber is the literal re-implementation of React's core algorithm. It’s an "execution engine" because it takes control of *how* and *when* your component updates are processed.

Think of React's update process in two main phases:

1.  **Render Phase (Reconciliation):**
    *   React starts traversing your component tree, creating a new Fiber tree (a fancy internal representation of your UI).
    *   It calls your `render` methods (or functional component bodies) to figure out what should be on the screen.
    *   Crucially, this phase is **interruptible**. React can pause this work, yield control back to the browser (so it can process user input or perform animations), and then pick up where it left off. This cooperative scheduling is the superpower of Fiber.
    *   During this phase, side effects like DOM mutations should *not* happen, as the work might be aborted or re-done.

2.  **Commit Phase:**
    *   Once the Render Phase completes and all changes are calculated, React moves to the Commit Phase.
    *   This phase is **uninterruptible**. React takes the calculated changes and applies them to the actual browser DOM.
    *   This is when effects like those in `useEffect` and `useLayoutEffect` are run, allowing you to interact with the real DOM.

### Why Interruptible Work Matters

Imagine a user types rapidly into an input field while a large data fetch completes and triggers a massive UI update. With the old Stack reconciler, React would have to process the entire UI update *before* the browser could respond to the user's keystrokes. This leads to a noticeable delay.

With Fiber, React can process a chunk of the UI update, then check if there's any higher-priority work (like a user typing). If so, it pauses the UI update, lets the browser handle the input, and then resumes the UI update when the browser is idle again. This ensures that user interactions feel immediate and the UI remains responsive, even during heavy processing.

```typescript
// A simple component to illustrate the concept
import React, { useState, useTransition } from 'react';

function BigList() {
  const [items, setItems] = useState(Array.from({ length: 10000 }, (_, i) => `Item ${i}`));
  const [filter, setFilter] = useState('');
  const [isPending, startTransition] = useTransition();

  const filteredItems = items.filter(item => item.includes(filter));

  const handleFilterChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    // This part of the state update is "urgent" (for the input value)
    setFilter(event.target.value);

    // This part of the state update can be deferred (for filtering a large list)
    // React can pause this work if higher priority tasks (like more typing) come in.
    startTransition(() => {
      // Potentially expensive calculation here if it were inside the handler directly
      // With useTransition, React can prioritize rendering the input value first.
    });
  };

  return (
    <div>
      <input type="text" value={filter} onChange={handleFilterChange} placeholder="Filter items..." />
      {isPending && <span>Loading filtered items...</span>}
      <ul>
        {filteredItems.map(item => (
          <li key={item}>{item}</li>
        ))}
      </ul>
    </div>
  );
}

export default BigList;
```

In this example, `useTransition` explicitly tells React that the update to `filteredItems` can be deferred. React's Fiber engine will intelligently schedule this work, prioritizing immediate UI feedback (like updating the input field) over the potentially expensive list filtering, ensuring a smoother user experience.

## The "Why": React Had to Build Its Own Execution Engine

So, why not just use existing browser APIs? The answer is granular control and fine-tuned scheduling. While browsers offer APIs like `requestIdleCallback`, they don't provide the level of control React needs to build its internal "Fiber tree," manage priorities, and flawlessly coordinate updates across a complex component hierarchy.

React built its own scheduler because:
1.  **JavaScript is single-threaded:** It can only do one thing at a time. To prevent long-running tasks from blocking the UI, React needed a way to break up work and yield control back to the browser.
2.  **Browser APIs are too high-level:** They don't expose the primitives needed for React's specific reconciliation and scheduling needs (e.g., pausing a component's render mid-way).
3.  **Cross-platform consistency:** React aims to work consistently across web, mobile (React Native), and even VR. A custom engine allows for this unified approach.

## Pitfalls and Practical Takeaways

Understanding Fiber and the two phases isn't just theoretical. It has real implications:

*   **`useEffect` vs. `useLayoutEffect`**: `useEffect` runs *after* the Commit Phase (after the browser has painted), making it non-blocking. `useLayoutEffect` runs synchronously *after* DOM mutations but *before* the browser paints, which is critical for measuring DOM elements or performing calculations that need to affect layout immediately. Misusing the latter can reintroduce jank.
*   **Unnecessary Re-renders**: While Fiber is efficient, calling `setState` too often with identical data still triggers the Render Phase, causing React to do unnecessary diffing. Tools like `React.memo`, `useMemo`, and `useCallback` are your allies here to help React skip work.
*   **Keys in Lists**: Forgetting `key` props on list items isn't just a warning; it directly impacts the diffing algorithm. Without stable keys, React can't efficiently identify moved, added, or removed items, leading to less optimal DOM updates and potential state bugs.
*   **Batching**: React often batches multiple `setState` calls that happen within the same event loop tick into a single update. This is an optimization powered by its scheduler, preventing multiple redundant re-renders.

## Wrapping Up

React's custom execution engine, powered by Fiber, is a marvel of engineering. It's React’s way of saying, "The browser's default tools aren't quite enough for the rich UIs we want to build, so we're building our own highly optimized orchestrator." It ensures that even when your application is doing heavy lifting, the user experience remains responsive and delightful. By understanding this foundation, you gain a deeper appreciation for React’s magic and, more importantly, the knowledge to build truly high-performance, resilient applications.
