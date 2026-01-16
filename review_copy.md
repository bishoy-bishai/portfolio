# REVIEW: ReactJS Hook Pattern ~Use Hook with Context~

**Primary Tech:** React

## 🎥 Video Script
Hey everyone! Ever felt like your React components are playing a never-ending game of 'pass the prop'? You know, where you're drilling props through five, six, seven levels deep, and debugging feels like an archaeological dig? I've been there. I remember a project where even a simple `isAuthenticated` flag had to travel an epic journey, and updating it meant tracing a spaghetti of callbacks. Maintenance was a nightmare, and honestly, it just *felt* wrong.

My "aha!" moment came when I truly embraced the power of combining React Context with custom Hooks. It was like finally finding the express lane on a busy highway. Instead of sprinkling `useContext` calls directly into every component, I started encapsulating that logic – along with any related state, actions, or derived values – into a single, focused custom hook.

Suddenly, my components transformed. They went from being state-management orchestrators to simple consumers of well-defined APIs. They didn't care *how* the theme was provided or *how* authentication state was managed; they just called `useTheme()` or `useAuth()` and got exactly what they needed. It made the codebase cleaner, more readable, and infinitely more testable. My actionable takeaway for you today: stop just using `useContext` directly. Start encapsulating that context consumption logic into custom hooks. Your future self will absolutely thank you.

## 🖼️ Image Prompt
A dark background (#1A1A1A) with elegant gold accents (#C9A227). In the center, a subtle, glowing gold sphere represents shared Context, with gentle, radiating gold lines symbolizing data propagation. Surrounding this sphere are abstract, interconnected React component nodes, depicted as minimalist, geometric shapes with faint orbital rings, forming a loosely connected tree structure. Flowing, graceful gold bezier curves and arrows represent custom Hooks, elegantly abstracting the interaction between the component nodes and the central Context sphere, illustrating encapsulated logic and reusability without direct component-to-context lines. The overall aesthetic is professional, modern, and symbolic of sophisticated data flow and architectural patterns in a React application.

## 🐦 Expert Thread
1.  Prop drilling got you down? React Context is a lifeline, but direct `useContext` calls can get messy. Stop scattering that logic. There's a better way to wield its power. #ReactJS #Hooks #ContextAPI

2.  The true power move: `useContext` + Custom Hooks. Encapsulate your context consumption, derive values, handle errors, all within a dedicated hook. Think `useAuth()`, `useTheme()`, `useNotifications()`. Clean.

3.  Why bother? Your components become simple consumers of domain-specific APIs. No more `useContext(MyGiantContext)` directly. It cleans up component logic, boosts testability, and promotes reusability across your app.

4.  Performance check: A single context update can re-render ALL consumers. Always `useMemo` your context value in the Provider! Design for granularity: sometimes multiple smaller contexts are better than one "God Context." #ReactTips

5.  In my experience, this pattern is a game-changer for mid to large-scale React apps. It's not just about sharing state, it's about orchestrating app-wide behavior with elegant, self-contained units. Highly recommend.

6.  What's your favorite custom hook built around `useContext`? Share your patterns and lessons learned! Let's build better React apps together. #DevCommunity #ReactArchitecture

## 📝 Blog Post
# Elevating State Management: The React Hook + Context Power Pattern

Let's be honest. We've all been there. You're deep into a React project, the features are flying, and suddenly, you hit it: prop drilling hell. You need a piece of state or a function from way up the component tree, and you find yourself passing it down, down, *down* through layers of unrelated components. It feels clunky, makes refactoring a nightmare, and frankly, it just pollutes your components with props they don't actually care about.

React Context was a game-changer for this exact problem. It gave us a way to "teleport" values through the component tree, skipping intermediate props. But simply sprinkling `useContext` directly into every component that needs a piece of shared state? That, my friends, is only half the battle. In my experience, the true power, the elegant solution, lies in combining `useContext` with custom hooks. This isn't just about sharing state; it's about creating reusable, robust APIs for your application's global concerns.

### Why This Matters: Beyond Basic `useContext`

When `useContext` first arrived, many of us started using it directly within our components. And for simple cases, that's perfectly fine. But as your application grows, you might notice a few things:

1.  **Repetitive Boilerplate:** Every component consuming the context needs to call `useContext(MyContext)` and potentially handle `null` or `undefined` values if the provider isn't present.
2.  **Lack of Abstraction:** The component is directly tied to the *implementation* detail of where its state comes from (i.e., a specific context). What if you later decide to switch to Redux or Jotai for that piece of state? You'd have to update every consumer.
3.  **Complex Logic:** Often, consuming context isn't just about getting a raw value. You might need to derive state, dispatch actions, or combine multiple context values. Doing all that directly in a component can quickly make it bloated.
4.  **Testability:** Testing components that directly use `useContext` often requires mocking the context provider in your tests, which can be cumbersome.

This is where the custom hook pattern shines. It allows us to encapsulate all that context-related logic, transforming raw context consumption into a clean, reusable, domain-specific API.

### The Pattern in Action: A Practical Example

Let's imagine we're building an application with a dark/light theme toggle.

First, we need our Context:

```typescript
// src/context/ThemeContext.tsx
import React, { createContext, useState, useMemo, useCallback, ReactNode } from 'react';

type Theme = 'light' | 'dark';

interface ThemeContextType {
  theme: Theme;
  toggleTheme: () => void;
  setTheme: (theme: Theme) => void;
}

// Create a context with an initial undefined value.
// We'll throw an error if it's used without a provider.
export const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

interface ThemeProviderProps {
  children: ReactNode;
  initialTheme?: Theme;
}

export function ThemeProvider({ children, initialTheme = 'light' }: ThemeProviderProps) {
  const [theme, _setTheme] = useState<Theme>(initialTheme);

  const toggleTheme = useCallback(() => {
    _setTheme(prevTheme => (prevTheme === 'light' ? 'dark' : 'light'));
  }, []);

  const setTheme = useCallback((newTheme: Theme) => {
    _setTheme(newTheme);
  }, []);

  // Memoize the context value to prevent unnecessary re-renders of consumers
  // when only the provider re-renders but the value itself hasn't changed.
  const contextValue = useMemo(() => ({
    theme,
    toggleTheme,
    setTheme,
  }), [theme, toggleTheme, setTheme]);

  return (
    <ThemeContext.Provider value={contextValue}>
      {children}
    </ThemeContext.Provider>
  );
}
```

Now, instead of consuming `ThemeContext` directly in every component, we create a custom hook:

```typescript
// src/hooks/useTheme.ts
import { useContext } from 'react';
import { ThemeContext } from '../context/ThemeContext';

export function useTheme() {
  const context = useContext(ThemeContext);

  if (context === undefined) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }

  return context;
}
```

And consuming it is delightfully simple:

```typescript
// src/components/ThemeToggleButton.tsx
import React from 'react';
import { useTheme } from '../hooks/useTheme';

function ThemeToggleButton() {
  const { theme, toggleTheme } = useTheme();

  return (
    <button onClick={toggleTheme}>
      Switch to {theme === 'light' ? 'Dark' : 'Light'} Mode
    </button>
  );
}

export default ThemeToggleButton;
```

```typescript
// src/components/Greeting.tsx
import React from 'react';
import { useTheme } from '../hooks/useTheme';

function Greeting() {
  const { theme } = useTheme(); // Just need the theme here, not the toggle function

  return (
    <p style={{ color: theme === 'light' ? 'black' : 'white' }}>
      Hello, developer! Your current theme is {theme}.
    </p>
  );
}

export default Greeting;
```

This pattern drastically cleans up your components. They now just declare their *intent* (`useTheme()`) rather than the *mechanism* by which they get the theme.

### What Most Tutorials Miss: Real-World Insights

1.  **Error Handling for Missing Providers:** Notice the `if (context === undefined)` check in `useTheme`? This is crucial. If a developer forgets to wrap a component in `<ThemeProvider>`, they'll get a clear, actionable error message instead of a cryptic `Cannot read properties of undefined` deep inside your application. This little detail makes a huge difference in developer experience.

2.  **Performance Considerations with `useMemo` and `useCallback`:**
    In the `ThemeProvider`, I've used `useMemo` for the `contextValue` and `useCallback` for `toggleTheme` and `setTheme`. This is vital for performance. Without `useMemo`, a new `contextValue` object would be created on *every* re-render of `ThemeProvider`. This would cause *all* components consuming `ThemeContext` (via `useTheme`) to re-render, even if the `theme` itself hasn't changed. Memoizing the value prevents these unnecessary re-renders, ensuring your consumers only update when the actual `theme` or the functions themselves *truly* change. This is a common pitfall that can silently degrade performance in larger applications.

3.  **Granularity of Contexts:** Don't try to put *everything* into one giant `AppContext`. While a single context is easy to set up, *any* update to its value will re-render *all* components consuming it. For larger apps, I've found it's often better to have multiple, smaller contexts for different domains (e.g., `AuthContext`, `ThemeContext`, `NotificationsContext`). This minimizes the surface area for re-renders and keeps concerns separated.

### Common Pitfalls and How to Avoid Them

*   **The "God Context" Anti-Pattern:** Shoving all your global state into one `AppContext` is tempting for simplicity. Resist the urge. As mentioned, this leads to massive, unnecessary re-renders. If `AppProvider` updates, every component using `useApp` re-renders. Break down your contexts by feature or domain.
*   **Forgetting the Provider:** This is classic. You create a custom hook, use it in a component, and then wonder why you're getting an error. Always ensure the components using your custom context hook are wrapped by the corresponding `Provider` component somewhere higher in the tree. The error handling in `useTheme` helps immensely here.
*   **Over-optimizing Prematurely:** While `useMemo` and `useCallback` are important for context values, don't just sprinkle them everywhere out of habit. Focus on the context value itself. For simple components, excessive memoization can add complexity without significant performance gains. Profile first, optimize second.
*   **Complex Context Values:** If your context value is an object with many properties, and you only need one or two in a specific component, that component will still re-render if *any* property of the context value changes. For highly performance-critical scenarios, you might consider patterns like splitting context or creating "selector-like" functions within your custom hook that derive specific parts of the context, although this adds complexity. For most cases, sensible granularity of contexts is usually sufficient.

### Wrapping It Up

The `useContext` + custom hook pattern isn't just a coding trick; it's a fundamental architectural approach to managing global state gracefully in React. It empowers you to build highly reusable, testable, and maintainable application features without succumbing to prop drilling or scattering complex state logic throughout your component tree.

By embracing this pattern, you're not just writing cleaner code today; you're investing in the future scalability and developer experience of your React applications. Give it a shot, experiment with breaking down your global concerns into focused contexts and custom hooks, and watch your codebase transform into something truly delightful to work with.