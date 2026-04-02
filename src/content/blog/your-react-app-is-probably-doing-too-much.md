---
title: "Your React App Is Probably Doing Too Much"
description: "Your React App Is Probably Doing Too..."
pubDate: "Apr 02 2026"
heroImage: "../../assets/your-react-app-is-probably-doing-too-much.jpg"
---

# Your React App Is Probably Doing Too Much

We all start with the best intentions, right? A clean React component, a simple feature, maybe a `useState` hook, and a sprinkle of JSX. It's beautiful. It's fast. Then, bit by bit, through feature requests, refactors, and the natural evolution of a codebase, that elegant component starts to put on weight. Suddenly, it’s a tangled mess of props, local state, global state, unnecessary re-renders, and performance hogs. It’s not just slow; it’s a nightmare to debug and even harder to maintain.

In my experience, this isn't a failure of React itself. Quite the opposite. React's flexibility and declarative nature can sometimes *mask* the true complexity we’re introducing. We keep pushing logic into components because, well, it's *convenient*. But convenience at the micro-level often leads to chaos at the macro-level. The core problem, I've found, is often a single principle being violated repeatedly: **Single Responsibility**.

## The Hidden Cost of Convenience

Think about a `UserProfile` component. In its simplest form, it just displays a user's name and avatar. But then, requirements expand:
*   It needs to fetch the user data.
*   It needs an "Edit Profile" button.
*   The edit form needs to manage its own input states.
*   It needs to handle profile picture uploads.
*   It needs to show a loading spinner.
*   It needs to handle error messages.

Before you know it, your `UserProfile` component is 300 lines long, managing five different states, fetching data, handling multiple forms, and probably re-rendering its entire sub-tree every time a single input field changes. This isn't just about code length; it’s about mental overhead, performance bottlenecks, and a drastically increased surface area for bugs.

Here's a simplified (and frankly, charitable) example of a component trying to do too much:

```typescript
// 🚫 Bad example: UserProfile component doing too much
import React, { useState, useEffect } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  bio: string;
}

async function fetchUser(userId: string): Promise<User> {
  // Simulate API call
  return new Promise(resolve =>
    setTimeout(() =>
      resolve({
        id: userId,
        name: 'Jane Doe',
        email: 'jane@example.com',
        bio: 'React enthusiast and coffee lover.',
      }), 500)
  );
}

async function updateUser(userId: string, data: Partial<User>): Promise<User> {
  // Simulate API call
  return new Promise(resolve =>
    setTimeout(() =>
      resolve({ id: userId, name: data.name || '', email: data.email || '', bio: data.bio || '' }), 300)
  );
}

const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [isEditing, setIsEditing] = useState(false);
  const [editName, setEditName] = useState('');
  const [editBio, setEditBio] = useState('');

  useEffect(() => {
    const loadUser = async () => {
      setLoading(true);
      setError(null);
      try {
        const userData = await fetchUser(userId);
        setUser(userData);
        setEditName(userData.name);
        setEditBio(userData.bio);
      } catch (err) {
        setError('Failed to fetch user.');
      } finally {
        setLoading(false);
      }
    };
    loadUser();
  }, [userId]);

  const handleSave = async () => {
    if (!user) return;
    setLoading(true);
    setError(null);
    try {
      const updatedUser = await updateUser(user.id, { name: editName, bio: editBio });
      setUser(updatedUser);
      setIsEditing(false);
    } catch (err) {
      setError('Failed to update user.');
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading profile...</div>;
  if (error) return <div style={{ color: 'red' }}>Error: {error}</div>;
  if (!user) return null;

  return (
    <div style={{ border: '1px solid #ccc', padding: '20px', borderRadius: '8px' }}>
      <h2>{user.name}'s Profile</h2>
      {isEditing ? (
        <div>
          <label>Name:
            <input type="text" value={editName} onChange={(e) => setEditName(e.target.value)} />
          </label><br/>
          <label>Bio:
            <textarea value={editBio} onChange={(e) => setEditBio(e.target.value)} />
          </label><br/>
          <button onClick={handleSave} disabled={loading}>Save</button>
          <button onClick={() => setIsEditing(false)} disabled={loading}>Cancel</button>
        </div>
      ) : (
        <div>
          <p><strong>Email:</strong> {user.email}</p>
          <p><strong>Bio:</strong> {user.bio}</p>
          <button onClick={() => setIsEditing(true)}>Edit Profile</button>
        </div>
      )}
    </div>
  );
};
```

This component fetches, displays, *and* allows editing. Every keystroke in the edit fields causes `UserProfile` to re-render, even if the actual user data display part doesn't change. It's a classic example of too many responsibilities.

## The Path to Leaner Apps: Deconstruction and Focus

Here's how we can make our React apps breathe again:

### 1. **Deconstruct Components: The True Meaning of Single Responsibility**
A component should have *one primary reason to change*.
*   **Data Fetching**: Extract into custom hooks or dedicated data-fetching layers (like React Query/SWR).
*   **Display Logic**: Pure components that just render props.
*   **Form Management**: Dedicated form components, maybe using libraries like React Hook Form.
*   **Interaction/State Management**: Smaller components or custom hooks managing very specific UI state.

Let’s refactor the `UserProfile` example:

```typescript
// ✅ Good example: Custom hook for data fetching
import React, { useState, useEffect, useCallback } from 'react';

// (User interface and simulated API functions remain the same as above)

// Custom Hook for user data fetching and management
function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const loadUser = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const userData = await fetchUser(userId);
      setUser(userData);
    } catch (err) {
      setError('Failed to fetch user.');
    } finally {
      setLoading(false);
    }
  }, [userId]);

  const updateUserProfile = useCallback(async (data: Partial<User>) => {
    if (!user) return;
    setLoading(true);
    setError(null);
    try {
      const updatedUser = await updateUser(user.id, data);
      setUser(updatedUser);
      return updatedUser;
    } catch (err) {
      setError('Failed to update user.');
      throw err; // Re-throw to allow component to handle specific UI feedback
    } finally {
      setLoading(false);
    }
  }, [user]);

  useEffect(() => {
    loadUser();
  }, [loadUser]);

  return { user, loading, error, updateUserProfile, refetchUser: loadUser };
}

// ✅ Good example: UserDisplay component (pure display)
const UserDisplay: React.FC<{ user: User; onEdit: () => void }> = React.memo(({ user, onEdit }) => (
  <div>
    <p><strong>Email:</strong> {user.email}</p>
    <p><strong>Bio:</strong> {user.bio}</p>
    <button onClick={onEdit}>Edit Profile</button>
  </div>
));

// ✅ Good example: UserEditForm component (manages its own form state)
const UserEditForm: React.FC<{
  user: User;
  onSave: (name: string, bio: string) => Promise<any>; // Returns a promise to indicate save completion
  onCancel: () => void;
  isLoading: boolean;
}> = React.memo(({ user, onSave, onCancel, isLoading }) => {
  const [name, setName] = useState(user.name);
  const [bio, setBio] = useState(user.bio);
  const [saveError, setSaveError] = useState<string | null>(null);

  const handleSaveClick = async () => {
    setSaveError(null);
    try {
      await onSave(name, bio);
    } catch (err) {
      setSaveError('Failed to save changes.');
    }
  };

  return (
    <div>
      <label>Name:
        <input type="text" value={name} onChange={(e) => setName(e.target.value)} disabled={isLoading} />
      </label><br/>
      <label>Bio:
        <textarea value={bio} onChange={(e) => setBio(e.target.value)} disabled={isLoading} />
      </label><br/>
      {saveError && <div style={{ color: 'red' }}>{saveError}</div>}
      <button onClick={handleSaveClick} disabled={isLoading}>Save</button>
      <button onClick={onCancel} disabled={isLoading}>Cancel</button>
    </div>
  );
});

// ✅ Good example: UserProfileContainer component (orchestrates)
const UserProfileContainer: React.FC<{ userId: string }> = ({ userId }) => {
  const { user, loading, error, updateUserProfile } = useUser(userId);
  const [isEditing, setIsEditing] = useState(false);

  const handleSaveEdit = async (newName: string, newBio: string) => {
    await updateUserProfile({ name: newName, bio: newBio });
    setIsEditing(false);
  };

  if (loading) return <div>Loading profile...</div>;
  if (error) return <div style={{ color: 'red' }}>Error: {error}</div>;
  if (!user) return null;

  return (
    <div style={{ border: '1px solid #ccc', padding: '20px', borderRadius: '8px' }}>
      <h2>{user.name}'s Profile</h2>
      {isEditing ? (
        <UserEditForm
          user={user}
          onSave={handleSaveEdit}
          onCancel={() => setIsEditing(false)}
          isLoading={loading} // Pass loading state from hook for form interaction
        />
      ) : (
        <UserDisplay user={user} onEdit={() => setIsEditing(true)} />
      )}
    </div>
  );
};
```
Now, `UserEditForm` only cares about its input states and `UserDisplay` is a pure presentational component. The `UserProfileContainer` orchestrates, delegating responsibilities. This means:
*   `UserDisplay` won't re-render when the form inputs change.
*   `UserEditForm`'s internal state doesn't affect `UserDisplay` or the data fetching logic until `onSave` is called.
*   The `useUser` hook is reusable and encapsulates data logic.

### 2. **State Discipline: Derived vs. Actual**
Often, we store state that can actually be derived from other pieces of state or props.
```typescript
// 🚫 Avoid: Storing derived state
const [firstName, setFirstName] = useState('John');
const [lastName, setLastName] = useState('Doe');
const [fullName, setFullName] = useState('John Doe'); // Unnecessary!
// ... later
setFirstName('Jane');
setFullName('Jane Doe'); // Easy to forget to update or introduce bugs
```
Instead:
```typescript
// ✅ Prefer: Derived state
const [firstName, setFirstName] = useState('John');
const [lastName, setLastName] = useState('Doe');
const fullName = `${firstName} ${lastName}`; // Always up-to-date, no extra state.
```
This reduces state complexity, potential bugs, and unnecessary re-renders when the derived state changes.

### 3. **Smart Memoization: Use `React.memo`, `useCallback`, `useMemo` Wisely**
These are powerful tools, but they come with a cost. Don't just slap `React.memo` on every component or wrap every function in `useCallback`. The overhead of the comparison might outweigh the benefit of avoiding a re-render.
*   **When to use `React.memo`**: For pure components that render often with the *same props*, or components that render large, expensive trees.
*   **When to use `useCallback`/`useMemo`**: To stabilize function references or memoize expensive computations *when passed as props to memoized child components*, or as dependencies in `useEffect` to prevent infinite loops.

**Pitfall Alert**: A common mistake is using `useCallback` for an event handler that is only passed to a non-memoized component, or whose dependencies change frequently. You're incurring the memoization cost without gaining the re-render prevention benefit. **Profile first!** Use React DevTools Profiler to identify actual bottlenecks.

## The Payoff: Simpler, Faster, Happier

By consciously pushing for single responsibilities, disciplined state management, and targeted optimizations, you're not just making your app faster. You're making it:
*   **More maintainable**: Smaller, focused units are easier to understand and change.
*   **More testable**: Isolation makes unit testing a breeze.
*   **More scalable**: New features fit in naturally without breaking existing logic.
*   **More enjoyable to develop**: Less time debugging, more time building.

The goal isn't just a fast app; it's a maintainable, understandable, and enjoyable app to build. Sometimes, the best way to achieve that is to simply tell your React app: "You're doing too much. Let's simplify." It's a conversation worth having with your codebase.

---
