---
title: "React Query: What Is `staleTime` and Why Should You Care?"
description: "React Query: What Is staleTime and Why Should You..."
pubDate: "Feb 26 2026"
heroImage: "../../assets/react-query--what-is--staletime--and-why-should-yo.jpg"
---

# React Query: What Is `staleTime` and Why Should You Care?

Ever been working on a web app and felt like your data fetching was doing *too much* work? You know the drill: navigate to a list page, see a loading spinner. Click into a detail, another spinner. Go back to the list... *spinner again*. It's a classic scenario, and honestly, it can make even the snappiest apps feel sluggish. As developers, we often focus on making sure our data is always up-to-the-second fresh. But sometimes, "always fresh" comes at the cost of user experience.

This is where React Query's `staleTime` comes into play, and frankly, it's one of those unsung heroes that, once understood, you can't imagine building a modern React app without. It’s not just a performance tweak; it’s a fundamental shift in how you deliver perceived performance and responsiveness to your users.

## The Problem with "Always Fresh"

Imagine an e-commerce site. A user lands on a product listing. We fetch the products. They click on a product to view details. We fetch *that* product's details. They hit the back button. What happens? By default, without any specific configuration, React Query (and most similar libraries) considers data "stale" the moment it's fetched. This means when the user returns to the product list, even if they were just there a second ago, React Query will trigger another network request. The UI shows a loading state, maybe a flash of empty content, and then the data reappears. This "loading flicker" is jarring and interrupts the user's flow.

In my early days with data fetching libraries, I’d often wrestle with manual caching or complex state management to avoid these redundant fetches. It was messy. Then, I truly grasped `staleTime`, and it felt like a lightbulb moment.

## `staleTime`: Your Best Friend for Perceived Performance

At its core, `staleTime` tells React Query for how long a piece of data should be considered "fresh." As long as data is fresh, React Query will **immediately serve it from the cache without triggering a network request.** This is crucial. Your users see the data instantly, no spinners, no flickering.

Only *after* the `staleTime` has passed does the data become "stale." Once data is stale, React Query will still serve it from the cache immediately *if available*, but it will also trigger a background re-fetch to get the latest version. This is often called "stale-while-revalidate."

Let's break that down:

1.  **Data is fresh:** React Query serves cached data *instantly*, no network request.
2.  **`staleTime` expires, data becomes stale:** React Query *still* serves cached data *instantly*, but *also* starts a background network request.
3.  **New data arrives:** The UI updates seamlessly with the fresh data (if it's changed).

This pattern provides the best of both worlds: immediate UI updates for a snappy feel, combined with eventual consistency for data accuracy.

## `staleTime` vs. `cacheTime`: The Dynamic Duo (and common confusion)

This is a point where many developers initially stumble, and honestly, I did too. It's easy to conflate `staleTime` with `cacheTime`, but they control completely different aspects of your data's lifecycle.

*   **`staleTime`**: This is about **freshness**. How long is the data considered "good enough" to be displayed immediately without a background re-fetch? Once `staleTime` passes, the data is still in the cache, but React Query will try to re-fetch it in the background if a component requests it. The *default* `staleTime` is `0`, meaning data is *instantly* stale.

*   **`cacheTime`**: This is about **garbage collection**. How long should inactive query data remain in the cache *before it's completely removed*? Once `cacheTime` passes *and* there are no active observers (components using the data), the data is garbage collected. The *default* `cacheTime` is `5 * 60 * 1000` (5 minutes).

Think of it this way:
- **`staleTime`** is like the "use by" date on your milk. Even if it's past the date, you might still drink it if it looks fine, but you'll probably buy a new carton soon.
- **`cacheTime`** is like how long you'll keep that empty milk carton in the fridge before throwing it out.

Understanding this distinction is key to mastering React Query.

## Putting `staleTime` into Practice

Let's look at some code examples.

By default, `staleTime` is `0`, meaning data is always stale.

```typescript
import { useQuery } from '@tanstack/react-query';

interface Product {
  id: string;
  name: string;
  price: number;
}

async function fetchProducts(): Promise<Product[]> {
  const response = await fetch('/api/products');
  if (!response.ok) {
    throw new Error('Network response was not ok');
  }
  return response.json();
}

function ProductsList() {
  const { data, isLoading, error } = useQuery<Product[]>({
    queryKey: ['products'],
    queryFn: fetchProducts,
    // staleTime is 0 by default, so every time this component mounts
    // or refetches due to window focus, it will trigger a network request.
  });

  if (isLoading) return <div>Loading products...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {data?.map(product => (
        <li key={product.id}>{product.name} - ${product.price}</li>
      ))}
    </ul>
  );
}
```

Now, let's introduce a `staleTime` of 5 seconds:

```typescript
import { useQuery } from '@tanstack/react-query';

// ... (Product interface and fetchProducts function remain the same)

function ProductsList() {
  const { data, isLoading, error } = useQuery<Product[]>({
    queryKey: ['products'],
    queryFn: fetchProducts,
    staleTime: 1000 * 5, // Data is considered fresh for 5 seconds
  });

  if (isLoading) return <div>Loading products...</div>; // Only shown on initial fetch
  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {data?.map(product => (
        <li key={product.id}>{product.name} - ${product.price}</li>
      ))}
    </ul>
  );
}
```

With `staleTime: 1000 * 5`, if a user navigates away from `ProductsList` and returns within 5 seconds, they will see the previously fetched products *instantly*. No `isLoading` state, no spinner. React Query will only perform a background re-fetch if they return *after* 5 seconds or if another re-fetch trigger (like window focus) occurs when the data is already stale.

### `staleTime: Infinity` - The "Never Stale" Data

For data that rarely changes (like a list of static categories, application settings, or user preferences that are only updated via specific actions), you might even set `staleTime` to `Infinity`. This tells React Query: "Once you fetch this data, it's always fresh, never re-fetch in the background unless explicitly told to."

```typescript
function CategoriesList() {
  const { data } = useQuery<string[]>({
    queryKey: ['categories'],
    queryFn: fetchCategories,
    staleTime: Infinity, // Data is always fresh
  });

  // ... render categories
}
```

Use `staleTime: Infinity` with caution. It's fantastic for truly static data but can lead to displaying outdated information if your data *can* change from external sources without your app knowing.

## Common Pitfalls and Lessons Learned

1.  **Confusing `staleTime` and `cacheTime`**: As discussed, this is the #1 pitfall. Remember, `staleTime` affects *when* a background re-fetch happens, `cacheTime` affects *when* data is garbage collected. You can have stale data in the cache, but you can't have fresh data if it's been garbage collected!

2.  **Setting `staleTime` Too High (or `Infinity` for dynamic data)**: While `Infinity` is great for truly static data, I've seen teams use it for data that *does* change, leading to users seeing outdated information. Always consider the data's volatility. Is it stock prices? `staleTime: 0` or very low. Is it a list of countries? `staleTime: Infinity` is probably fine.

3.  **Not Leveraging Global Defaults**: You don't have to configure `staleTime` on every `useQuery` call. React Query allows you to set global defaults with `QueryClientProvider`:

    ```typescript
    import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

    const queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 1000 * 60 * 5, // Default to 5 minutes
          // cacheTime: 1000 * 60 * 60, // Default to 1 hour
        },
      },
    });

    function App() {
      return (
        <QueryClientProvider client={queryClient}>
          {/* Your app components */}
        </QueryClientProvider>
      );
    }
    ```
    This approach makes your `staleTime` strategy consistent across your application and allows for easy overrides on a per-query basis. I've found this incredibly useful for establishing a baseline user experience.

4.  **Forgetting About User Expectations**: The right `staleTime` isn't purely technical; it's deeply tied to user experience. For a feed of social media posts, a low `staleTime` (even default `0`) might be appropriate because users expect the latest. For a user's profile settings page, a higher `staleTime` makes sense because changes are rare and immediate visual consistency on re-entry is valued. Always ask: "How quickly does the user *need* to see updated information here?"

## The Takeaway

`staleTime` is a powerful, yet often underestimated, feature of React Query. It allows you to strike a brilliant balance between data freshness and optimal user experience. By consciously managing how long your data remains "fresh" in the cache, you can eliminate unnecessary loading states, reduce network requests, and make your application feel incredibly fast and responsive.

Don't just default to `0`. Take a moment to consider the nature of the data you're fetching and set an appropriate `staleTime`. Your users (and your backend) will thank you for it. It's one of those small configurations that delivers a disproportionately large impact on the perceived quality of your application.
