# REVIEW: How We Ran Two Portals on the Same Domain During a React Migration (Without Users Noticing)

**Primary Tech:** React

## 🎥 Video Script
Hey there! Ever found yourself staring down a massive React migration, knowing you can’t just flip a switch? We've all been there. I remember a project where we had this beast of a legacy app, critical for business, but we were building its React 18 successor. The ask? Users absolutely *could not* see a single hiccup during the transition, even as they navigated between "old" and "new" parts of the product.

The traditional "server-side proxy" felt too clunky, causing full page reloads. Our "aha!" moment came when we realized we could create a tiny, client-side orchestrator. It sat at the very root of our `index.html`, making a lightning-fast decision: "Should I load the old app or the new one for this URL?" And the magic? When a user navigated from an old section to a new one, this orchestrator would *unmount* the old app and *mount* the new one into the *same DOM node*, all without a full page refresh. The user just saw a smooth, fast transition. It was like changing tires on a moving car!

The takeaway? You don't need to break the bank or the user experience to migrate. A smart client-side orchestrator can give you incredible control and a truly seamless rollout.

## 🖼️ Image Prompt
A minimalist, professional image on a dark background (#1A1A1A). The central theme is a seamless transition between two distinct, abstract user interface portals within a single, unified container. One portal on the left is represented by slightly older, more angular, and subdued UI elements (e.g., boxy cards, simpler lines). The portal on the right is represented by modern, sleek, and slightly more vibrant UI elements (e.g., rounded buttons, subtle gradients, more fluid shapes). These two portals are not separated by a hard line but rather blend or subtly overlap in the center, symbolizing a smooth migration. Orbiting and connecting these UI elements are abstract React atomic structures (circles, orbital rings, interconnected nodes), rendered with gold accents (#C9A227), emphasizing the React technology orchestrating the transition. The gold accents also highlight subtle animation lines or flow arrows indicating a dynamic swap or hand-off between the two portal styles. No text, no logos. The overall aesthetic should convey sophisticated engineering, controlled evolution, and an invisible underlying mechanism.

## 🐦 Expert Thread
1/7 React migration dread? We’ve all been there: legacy app, shiny new tech, and the terrifying mandate: "no users noticing!" Traditional server-side proxies often mean full page reloads, a non-starter for true seamlessness. #ReactMigration #Frontend

2/7 Our "aha!" moment: a tiny, client-side orchestrator. This little hero loads *before* any React, quickly decides which app (old or new) should control the `#root` DOM element, then loads & mounts it. One app at a time, no conflicting React versions! #MicroFrontends #WebDev

3/7 The real magic? When navigating between old/new routes, this orchestrator *unmounts* the current app & *mounts* the other, all client-side. No full browser refresh. Users see a smooth transition, not a jarring page reload. It's like changing tires on a moving car! #UX #React

4/7 This hot-swapping strategy isolates different React versions beautifully. React 16 vs React 18? No `Invalid hook call` nightmares, as only one React instance owns the `#root` at any given moment. Clean. Efficient. #ReactTips #Architecture

5/7 Pitfalls to watch: Cleanly unmount EVERYTHING (event listeners, global side effects). Manage CSS carefully to avoid leaks. Ensure robust routing logic. And don't forget lazy loading bundles for performance! #Performance #CodeQuality

6/7 This approach enables incremental rollout, A/B testing, and feature flagging with surgical precision. It's not just a tech solution; it's a strategic business advantage for complex migrations. #ProductManagement #Engineering

7/7 The takeaway: Don't fear the big rewrite. Master the art of the invisible handover. Your users (and your sanity) will thank you. What's been your most seamless migration trick? Share below! 👇 #FrontendDev #DevOps

## 📝 Blog Post
# The Invisible Handover: Running Two React Portals on the Same Domain During Migration

Remember that feeling? You're tasked with modernizing a critical, sprawling web application. The business wants the shiny new tech, the users demand zero downtime, and your legacy codebase is clinging on for dear life. You can't just press pause, rewrite everything, and launch. That's a surefire way to chaos, unhappy users, and a very stressed team.

In a recent project, we faced this exact challenge. Our goal was to migrate a significant application from an aging frontend stack (let's just call it "Legacy React" for simplicity, perhaps React 16 or earlier) to a new, fully modern React 18 codebase. The crucial constraint: users needed to navigate between "old" and "new" sections of the product *without ever noticing a full page refresh*. We had to run two distinct "portals" on the same domain, seamlessly handing off control.

This isn't just about technical elegance; it's about business continuity and user experience. A clumsy migration with jarring page reloads or broken functionality can erode user trust faster than you can say "bug report." The "strangler fig" pattern came to mind, but we needed a client-side rendition that gave us granular control.

## The Core Idea: Client-Side Orchestration with App Hot-Swapping

Many approaches to migrating large applications involve server-side proxies, where Nginx or a similar service routes requests to either the old or new application based on the URL. While effective, this often results in full page reloads when crossing the boundary between the two apps. For our "without users noticing" requirement, that was a non-starter.

The solution we landed on was a client-side orchestrator: a small, non-React JavaScript module loaded *before* anything else. Its job was simple yet powerful: determine which full application (the new React 18 one or the old React app) should control the `#root` DOM element, then load and mount it. When the user navigated to a different section, this orchestrator would gracefully *unmount* the currently active application and *mount* the other one, all client-side, avoiding a browser-initiated full page refresh.

Here's the thing: running two separate React instances, especially with different major versions (e.g., React 16 and React 18), directly on the same DOM tree is a recipe for disaster. You'll run into `Invalid hook call` errors, context provider conflicts, and general mayhem. Our approach circumvented this by ensuring only *one* React application was actively mounted to the main `#root` element at any given time. We were performing an application hot-swap.

## Setting Up the Ecosystem

Let's break down the technical setup.

### 1. The Universal `index.html`

First, we needed a single `index.html` file that both applications could live under. It would contain the root `div` for our applications and then immediately load our orchestrator script.

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Our Product</title>
    <!-- Include any global CSS that should apply to both apps (use sparingly) -->
</head>
<body>
    <div id="root"></div>
    <!-- The orchestrator is loaded first and controls which app initializes -->
    <script src="/orchestrator.js"></script>
</body>
</html>
```

### 2. The Mighty Orchestrator

This `orchestrator.js` file is the brain. It's plain JavaScript (or TypeScript compiled down), so no React version conflicts here. Its primary responsibilities:
*   Dynamically load the correct application's JavaScript bundle.
*   Mount the selected application.
*   Unmount the previous application cleanly.
*   Listen for route changes (e.g., `popstate` events) to decide if an app swap is needed.

```typescript
// orchestrator.ts
// We'll need to declare our global mount/unmount functions
declare global {
  interface Window {
    mountNewApp: (root: HTMLElement) => void;
    unmountNewApp: (root: HTMLElement) => void;
    mountLegacyApp: (root: HTMLElement) => void;
    unmountLegacyApp: (root: HTMLElement) => void;
    navigateTo: (path: string) => void; // A helper for internal navigation
  }
}

const rootElement = document.getElementById('root')!;
let currentApp: 'new' | 'legacy' | null = null;
let currentUnmountFn: ((root: HTMLElement) => void) | null = null;

// Utility to dynamically load a script
const loadScript = async (src: string): Promise<void> => {
    return new Promise((resolve, reject) => {
        const script = document.createElement('script');
        script.src = src;
        script.onload = () => resolve();
        script.onerror = () => reject(new Error(`Failed to load script: ${src}`));
        document.head.appendChild(script); // Append to head or body
    });
};

// Determines which app should handle the given path
const getAppForPath = (path: string): 'new' | 'legacy' => {
    // This is where your migration strategy lives:
    // - Based on URL prefixes: e.g., /legacy/* vs /new/*
    // - Feature flags: localStorage.getItem('newAppEnabled') === 'true'
    // - User segments, A/B tests, etc.
    if (path.startsWith('/legacy') || localStorage.getItem('useLegacyApp') === 'true') {
        return 'legacy';
    }
    return 'new'; // Default to the new app
};

const renderApp = async (appType: 'new' | 'legacy', path: string) => {
    if (currentUnmountFn) {
        // Unmount the current app cleanly
        currentUnmountFn(rootElement);
        // Ensure the DOM is completely clear before mounting the next app
        while (rootElement.firstChild) {
            rootElement.removeChild(rootElement.firstChild);
        }
    }

    // Dynamically load and mount the chosen app
    if (appType === 'new') {
        if (!window.mountNewApp) { // Load bundle only if not already loaded
            await loadScript('/static/js/new-app.bundle.js');
        }
        window.mountNewApp(rootElement);
        currentUnmountFn = window.unmountNewApp;
    } else { // 'legacy'
        if (!window.mountLegacyApp) {
            await loadScript('/static/js/legacy-app.bundle.js');
        }
        window.mountLegacyApp(rootElement);
        currentUnmountFn = window.unmountLegacyApp;
    }

    currentApp = appType;
    // Update the browser's URL without a full page reload
    history.pushState(null, '', path);
};

// Initial render when the page first loads
renderApp(getAppForPath(window.location.pathname), window.location.pathname);

// Handle browser's back/forward buttons
window.addEventListener('popstate', () => {
    const newAppType = getAppForPath(window.location.pathname);
    if (newAppType !== currentApp) {
        renderApp(newAppType, window.location.pathname);
    } else {
        // If the same app is supposed to be active, let its internal router handle the path change.
        // A more robust implementation might explicitly call the current app's router navigate method.
    }
});

// Provide a global navigation API for apps to request route changes
window.navigateTo = (path: string) => {
    const newAppType = getAppForPath(path);
    if (newAppType !== currentApp) {
        // If switching apps, trigger a full unmount/mount cycle
        renderApp(newAppType, path);
    } else {
        // If staying within the same app, just update history and let its internal router react
        history.pushState(null, '', path);
    }
};
```

### 3. Exposing Mount/Unmount Functions from Each Application

Each of your React applications (new and legacy) needs to be built in a way that exposes global functions for the orchestrator to call. This means modifying their entry points.

```typescript
// new-app/src/index.tsx (React 18 example)
import React from 'react';
import { createRoot, Root } from 'react-dom/client';
import App from './App'; // Your main React app component

let rootInstance: Root | null = null;

// Expose mount and unmount globally
if (typeof window !== 'undefined') {
  window.mountNewApp = (element: HTMLElement) => {
    if (!rootInstance) {
      rootInstance = createRoot(element); // Use createRoot for React 18
    }
    rootInstance.render(
      <React.StrictMode>
        <App /> {/* Your actual React 18 application */}
      </React.StrictMode>
    );
  };

  window.unmountNewApp = (element: HTMLElement) => {
    if (rootInstance) {
      rootInstance.unmount(); // Cleanly unmount the React 18 app
      rootInstance = null;
    }
  };
}
```

```typescript
// legacy-app/src/index.tsx (React 16/17 example)
import React from 'react';
import ReactDOM from 'react-dom'; // Use ReactDOM for older React versions
import LegacyApp from './LegacyApp'; // Your old React app component

if (typeof window !== 'undefined') {
  window.mountLegacyApp = (element: HTMLElement) => {
    ReactDOM.render(
      <React.StrictMode>
        <LegacyApp /> {/* Your actual legacy React application */}
      </React.StrictMode>,
      element
    );
  };

  window.unmountLegacyApp = (element: HTMLElement) => {
    ReactDOM.unmountComponentAtNode(element); // Unmount for older React versions
  };
}
```

## Key Insights and What Most Tutorials Miss

1.  **React Version Isolation:** The hot-swapping strategy elegantly solves React version conflicts. Since only one React app is active and managing the DOM at `#root` at any given time, their respective versions of `React`, `ReactDOM`, and hooks don't clash. This is, in my experience, the biggest headache in micro-frontend architectures trying to run multiple Reacts on the *same DOM tree*.
2.  **CSS Management:** Even with app hot-swapping, global CSS from one app can leak into the other during the brief loading period or if not carefully scoped. We found robust CSS methodologies (like CSS Modules, BEM, or PostCSS with scoped variables) were critical. Global resets or utility classes need to be managed carefully or externalized to the `index.html`.
3.  **Global Event Bus for Communication:** Since the applications are distinct and swapped, they can't directly share React Context or Redux stores. For rare, necessary communication (e.g., user login status, theme preference), a simple global event bus (`CustomEvent` or a tiny library) proved invaluable.
4.  **Performance & Bundle Splitting:** Lazy loading of app bundles (as shown with `loadScript`) is crucial. You don't want to load both a new and a legacy app bundle on initial page load if only one is needed. Aggressive code splitting within each app also helps keep the initial load fast.
5.  **A/B Testing & Rollout:** This architecture is perfect for controlled rollouts. The `getAppForPath` logic can integrate with feature flagging systems, allowing you to gradually expose the new app to different user segments, or even A/B test specific legacy vs. new pages.

## Pitfalls to Avoid

*   **Global Variables & Side Effects:** Be extremely mindful of what each app does to the `window` or `document` globally. Clean up event listeners, timers, and any injected DOM elements during `unmount`.
*   **Routing Inconsistencies:** Ensure your `getAppForPath` logic is robust and covers all permutations. Ambiguous routes can lead to unexpected app swaps or broken navigation. Both apps should ideally use client-side routers (like React Router) that handle paths *within* their own scope.
*   **Flickering During Swap:** While much faster than a full page reload, there's a brief moment during unmount and re-mount where the `#root` element might be empty. Strategic use of minimal global CSS for loading spinners or a skeleton screen can mask this, or pre-render a basic shell for the new app server-side to hide the initial load.
*   **Over-reliance on `localStorage` / `sessionStorage`:** While useful for basic flags, don't use it for sensitive or critical data transfer between apps, as it's not secure or reactive.

## The Payoff: An Invisible Transition

This hot-swapping strategy enabled us to incrementally migrate our application feature by feature, route by route. Users would click a link, and based on our orchestrator's logic, they might seamlessly transition from an old React 16 component to a new React 18 component, or vice-versa, without a single browser refresh. The experience was truly fluid, and the engineering team could tackle the migration in manageable chunks, deploying frequently and with confidence.

It's a testament to the power of thoughtful frontend architecture. By carefully isolating concerns and orchestrating the user's journey, we turned a daunting migration into a smooth, almost invisible evolution.