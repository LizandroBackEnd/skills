---
name: astro-clean-architecture
description: Universal Astro architecture, Server Islands, client hydration strategies, content collections, and performance optimization skill. Directs the generation of production-ready, zero-JS by default, type-safe, and maintainable Astro applications. Covers component patterns, nanostores state management, server:defer patterns, API endpoints, and strict typing across Astro projects. Triggers on "astro", "astro island", "server islands", "astro config", "content collections", "astro components".
license: MIT
metadata:
  author: lizdev
  version: "1.0.0"
---

# CORE ARCHITECTURAL PRINCIPLES

## Zero-JS by Default & Selective Hydration
Astro is built around the fundamental philosophy of Zero-JS by default, delivering ultra-fast web pages by rendering the entire application shell to static HTML on the server and stripping away client-side JavaScript automatically. 

Astro components (.astro files) are server-only templating files. The code written in the component script (frontmatter code fence) runs exclusively on the server at build time or on-demand at request time. Once execution completes, all JavaScript within the frontmatter is omitted from the final HTML payload delivered to the client, resulting in a zero-JS footprint by default.

Interactivity is selectively introduced via Client Islands by hydrating specific, isolated UI framework components (e.g., React, Vue, Svelte) using client template directives. Each client directive controls the loading priority and hydration timeline of the component's client-side bundle:
1. **client:load (High Priority):** Immediately loads and hydrates the component's JavaScript as soon as the page loads. This is reserved strictly for critical, immediately-visible UI elements that must be interactive instantly (e.g., header menus, primary navigation toggles, critical hero actions).
2. **client:idle (Medium Priority):** Loads and hydrates the component's JavaScript once the main browser thread becomes idle using the requestIdleCallback event, or falls back to the window load event. It supports an optional timeout configuration to guarantee hydration within a specified timeframe (e.g., client:idle={{timeout: 500}}). Useful for non-critical interactive elements that do not require instant reaction upon initial paint.
3. **client:visible (Low Priority):** Triggers loading and hydration of the component's JavaScript only when the element enters the browser viewport, using an internal IntersectionObserver. It supports an optional rootMargin parameter to pre-hydrate before the element crosses the viewport edge (e.g., client:visible={{rootMargin: "200px"}}). This is highly recommended for below-the-fold content, heavy visualizations, or expensive resource-intensive widgets to prevent bloating initial page load times and improve Core Web Vitals like Cumulative Layout Shift (CLS).
4. **client:media (Low Priority):** Loads and hydrates the component's JavaScript only when a specific CSS media query matches (e.g., client:media="(max-width: 50em)"). This is ideal for responsive widgets that are hidden on desktop layouts and only rendered or interacted with on mobile/tablet viewports.
5. **client:only (No-SSR Priority):** Bypasses server-side rendering entirely, executing only in the client browser. This directive is mandatory for framework components that access browser-exclusive globals (e.g., window, document, localStorage) during instantiation, as these globals do not exist on the server and will crash the compilation. When using client:only, you must explicitly declare the correct framework string (e.g., client:only="react") so the Astro compiler knows which bundler and renderer to load. Standard fallback content can be passed using slot="fallback" to render static placeholder HTML on the server while the client bundle downloads.

## Server Islands & Deferred Streaming
Server Islands (using the server:defer directive) provide a hybrid rendering pattern that separates static page shells from dynamic, personalized, or slow-loading content. Instead of letting a slow backend query delay the entire page load (blocking first paint), the main layout is rendered immediately on the server, while the slow or personalized section is deferred and streamed when available.

### Under the Hood
1. At build or request time, when the Astro compiler encounters a component marked with the server:defer directive, it strips out the component's server rendering logic and inserts a small script alongside any HTML passed to the named "fallback" slot (e.g., a skeleton loader, spinner, or generic placeholder).
2. The deferred component is compiled into a separate, secure route handled automatically by the Astro router.
3. When the page loads in the client browser, the generated placeholder script automatically makes a GET or POST request to the special internal dynamic endpoint (_server_islands/[IslandName]), passing any component props as an encrypted string in the URL query parameters.
4. The server receives the request, decrypts the props using a secure build-specific cryptographic key, executes the server-side code of the component on demand, and returns the rendered HTML.
5. The client script intercepts the returned HTML and seamlessly replaces the fallback slot, avoiding layout shifts.

### Caching Dynamics
Data for server islands is retrieved via a GET request with encrypted props in the URL query string, which permits standard HTTP caching via Cache-Control headers. However, browsers limit URLs to 2048 bytes. If the encrypted props payload causes the URL to exceed this limit, Astro automatically falls back to a POST request. POST requests are not cached by browsers or CDNs to preserve data integrity and security, which disables caching for that server island. To prevent this, developers must pass only lightweight identifiers (e.g., userId, productId) rather than full database objects or arrays, keeping the serialized props query string small and within cacheable GET limits.

## Separation of Concerns (Astro vs. Framework Islands)
To maintain structural safety and prevent hydration failures, a clear architectural boundary must be maintained between server-rendered Astro code and interactive framework islands:
1. **The Server Boundary (.astro):** Astro components (.astro) can import and render other Astro components, HTML files, and framework components. However, `.astro` components are HTML-only templates with no client-side runtime. They cannot be imported into React, Vue, Svelte, or other framework components. Any attempt to do so will result in compilation or runtime failures because framework components compile to their own virtual DOM or runtime specifications, which cannot interpret the server-exclusive structure of Astro components.
2. **Framework Islands (React, Svelte, Vue):** Framework islands must contain only valid code for their respective framework. If you need to embed Astro-rendered static HTML inside an interactive framework island, you must leverage slots or children passed inside a parent `.astro` file.
3. **The Serialization Bridge:** Props passed from the server `.astro` script to hydrated client islands (client:*) or Server Islands (server:defer) must be strictly serializable.
   - **Supported Serializable Types:** Plain objects, numbers, strings, arrays, Maps, Sets, RegExp, Dates, BigInts, URLs, Uint8Arrays, and Infinity.
   - **Prohibited Types:** Functions, methods, and objects with circular references. Passing a callback function to a hydrated framework component is strictly prohibited. All client-side interactivity must be driven by standard event handling, Custom Events, Nano Stores, or dedicated client-side API requests.

---

# WORKFLOW: COMPONENT AND PAGE CREATION

This sequential five-step workflow must be followed when designing new features or refactoring existing pages in an Astro project:

```
[ Step 1: Analyze Interactivity Requirements ]
                       │
                       ▼
[ Step 2: Evaluate Caching & Personalization ]
                       │
                       ▼
[ Step 3: Choose the Hydration Directive ]
                       │
                       ▼
[ Step 4: Define Shared State & Serializability ]
                       │
                       ▼
[ Step 5: Implement Caching & Route Optimization ]
```

## Step 1: Analyze Interactivity Requirements
Determine if the UI component has client-side reactive state, event listeners, or relies on user input to mutate the DOM.
- **No Client Reactivity:** If the component only displays text, images, or handles navigation via standard anchor tags, implement it exclusively as a server-only `.astro` component.
- **Client Reactivity:** If the component manages interactive forms, real-time counters, animations, or dynamic data tables, proceed to Step 3 and select an appropriate UI framework.

## Step 2: Evaluate Caching & Personalization Boundaries
Assess if the page contains high-cost database queries, third-party API fetches, or user-personalized information (e.g., session profiles, cart counts, tailored pricing).
- **Personalized/Slow Content on a Static Page:** Rather than rendering the entire page dynamically on each request (which destroys server-side caching and increases server load), isolate the personalized or slow component. Wrap it in a separate Astro component and apply the server:defer directive to declare it as a Server Island.
- **Aesthetic Placeholder:** Identify or create a lightweight placeholder (e.g., a generic icon, card placeholder, or CSS shimmer skeleton) and pass it into the Server Island under the slot="fallback" attribute to preserve visual stability and eliminate CLS.

## Step 3: Choose the Hydration Directive for Client Islands
If a UI framework component is required, select the most restrictive client directive based on viewport placement and lifecycle priority:
- **Above-the-Fold & Instantly Critical:** Apply client:load.
- **Below-the-Fold or Large Resource:** Apply client:visible with a specified rootMargin to initiate hydration slightly before scroll intersection (e.g., client:visible={{rootMargin: "150px"}}).
- **Low-Priority Non-Viewport Dependent:** Apply client:idle with a timeout safety threshold to ensure responsiveness.
- **Mobile/Responsive Only:** Apply client:media.
- **Browser-Only Globals:** Apply client:only alongside a slot="fallback" placeholder to maintain page structure during the bundle fetch.

## Step 4: Define Shared State & Verify Serializability
Establish state-sharing mechanisms and enforce strict serialization rules across the server/client boundary:
- **Cross-Island Communication:** If multiple islands (even across different frameworks) must share state, do not nest them or use framework-specific context providers. Define the shared state in a separate, lightweight module using Nano Stores.
- **Prop Validation:** Inspect all props being passed from the server `.astro` frontmatter script to the framework component or Server Island. Ensure all parameters strictly conform to serializable types. If client actions must communicate changes, leverage client-side event handlers or Nano Stores to mutate state rather than passing callbacks as props.

## Step 5: Implement Caching & Route Optimization
Configure cache validation controls and optimize request sizes:
- **Server Island Prop Optimization:** Avoid passing heavy objects (e.g., full product details or database records) as props to components using server:defer. Pass only atomic keys (e.g., id: string) to ensure props encrypt into a query string under 2048 bytes, preserving GET request cacheability on browser and CDN nodes.
- **Route Caching Control:** In SSR pages, API routes, or middleware, utilize the Astro.cache API to cache rendered responses. Map static routes to memory or Redis cache providers via routeRules within your configuration to optimize server CPU cycles.

---

# ADVANCED PATTERNS AND CODE EXAMPLES

## a) Server Island Implementation (server:defer) with Accessible Fallback
This pattern shows how to isolate a personalized user profile avatar from a static layout, allowing the header to render instantly while deferring the session-dependent avatar query.

### INCORRECT / ANTIPATTERN
Fetching session data directly inside the page frontmatter, which blocks the rendering of the entire layout on slow database lookups, or having no fallback loading state.
```typescript
---
// src/pages/dashboard.astro (Antipattern)
// This blocks the entire page load until the database response is resolved.
import { getSessionUser } from "../db/users";
const session = Astro.cookies.get("session");
const user = await getSessionUser(session?.value);
---
<html>
  <body>
    <header>
      <nav>
        <span>My Dashboard</span>
        <!-- Blocks entire page rendering if DB is slow -->
        <img src={user.avatarUrl} alt={user.name} />
      </nav>
    </header>
  </body>
</html>
```

### CORRECT / CLEAN
Isolating the dynamic user profile avatar into a Server Island component using server:defer, and providing an accessible fallback SVG skeleton to prevent Cumulative Layout Shift (CLS).
```typescript
// src/components/UserProfile.astro
---
import { getSessionUser } from "../db/users";

// Server Islands execute in their own isolated request context
const session = Astro.cookies.get("session");
if (!session) {
  return Astro.redirect("/login");
}

const user = await getSessionUser(session.value);
---
<div class="user-profile">
  <img src={user.avatarUrl} alt={`Profile of ${user.name}`} width="40" height="40" />
  <span>{user.name}</span>
</div>

<style>
  .user-profile {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
  img {
    border-radius: 50%;
  }
</style>
```

```typescript
// src/pages/dashboard.astro
---
import UserProfile from "../components/UserProfile.astro";
import SkeletonAvatar from "../components/SkeletonAvatar.astro";
---
<html>
  <head>
    <title>My Dashboard</title>
  </head>
  <body>
    <header>
      <nav class="header-nav">
        <span>My Dashboard</span>
        
        <!-- Turn UserProfile into a Server Island -->
        <UserProfile server:defer>
          <!-- Accessible fallback placeholder rendered instantly on initial page load -->
          <SkeletonAvatar slot="fallback" />
        </UserProfile>
      </nav>
    </header>
  </body>
</html>

<style>
  .header-nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem;
    background: #f8f9fa;
  }
</style>
```

## b) Cross-Island State Management using Nano Stores
This pattern demonstrates framework-agnostic state sharing using Nano Stores to synchronize cart state between a Svelte product action button and a React cart counter on the same page.

### INCORRECT / ANTIPATTERN
Using framework-specific state context providers (e.g., React Context) or attempting to nest framework components inside each other to pass callbacks, leading to hydration crashes.
```javascript
// src/components/ReactProvider.tsx (Antipattern)
// This forces Svelte components to be wrapped in React render loops, causing hydration failures.
import React, { createContext, useState } from "react";
export const CartContext = createContext(null);
export const CartProvider = ({ children }) => {
  const [items, setItems] = useState([]);
  return <CartContext.Provider value={{ items, setItems }}>{children}</CartContext.Provider>;
};
```

### CORRECT / CLEAN
Defining a framework-agnostic, lightweight, atomic cart store using Nano Stores, and consuming it across framework boundaries with native react and svelte store contracts.
```typescript
// src/stores/cart.ts
import { atom, computed } from "nanostores";

export interface CartItem {
  readonly id: string;
  readonly name: string;
  readonly price: number;
  readonly quantity: number;
}

// Atomic store holding the cart array
export const $cart = atom<readonly CartItem[]>([]);

// Action to append or increment items in the store
export function addToCart(product: Omit<CartItem, "quantity">): void {
  const current = $cart.get();
  const existing = current.find((item) => item.id === product.id);

  if (existing) {
    $cart.set(
      current.map((item) =>
        item.id === product.id ? { ...item, quantity: item.quantity + 1 } : item
      )
    );
  } else {
    $cart.set([...current, { ...product, quantity: 1 }]);
  }
}

// Derived computed store for cart count
export const $cartCount = computed($cart, (items) =>
  items.reduce((total, item) => total + item.quantity, 0)
);
```

```tsx
// src/components/CartCounter.tsx (React Island)
import { useStore } from "@nanostores/react";
import { $cartCount } from "../stores/cart";

export default function CartCounter() {
  const count = useStore($cartCount);

  return (
    <div className="cart-counter">
      <span>Cart ({count})</span>
    </div>
  );
}
```

```html
<!-- src/components/AddToCartButton.svelte (Svelte Island) -->
<script lang="ts">
  import { addToCart } from "../stores/cart";
  
  export let productId: string;
  export let productName: string;
  export let productPrice: number;

  function handleAdd() {
    addToCart({ id: productId, name: productName, price: productPrice });
  }
</script>

<button class="btn-add" on:click={handleAdd}>
  Add to Cart
</button>

<style>
  .btn-add {
    background: #0070f3;
    color: white;
    border: none;
    padding: 0.5rem 1rem;
    cursor: pointer;
  }
</style>
```

```typescript
// src/pages/store.astro
---
import CartCounter from "../components/CartCounter.tsx";
import AddToCartButton from "../components/AddToCartButton.svelte";
---
<html>
  <head>
    <title>Framework Agnostic Store</title>
  </head>
  <body>
    <header>
      <!-- React Island hydrated immediately -->
      <CartCounter client:load />
    </header>
    <main>
      <h2>Modern Astronaut Helmet</h2>
      <p>Price: $49.99</p>
      <!-- Svelte Island hydrated on viewport intersection -->
      <AddToCartButton 
        client:visible={{ rootMargin: "100px" }} 
        productId="helm_001" 
        productName="Modern Astronaut Helmet" 
        productPrice={49.99} 
      />
    </main>
  </body>
</html>
```

## c) Content Collections Schema Definition and Type-Safe Data Queries
This pattern implements type-safe blog collection schema validation using Zod and queries it within a dynamic Astro route.

```typescript
// src/content.config.ts
import { defineCollection, reference } from "astro:content";
import { glob } from "astro/loaders";
import { z } from "astro/zod";

const authors = defineCollection({
  loader: glob({ pattern: "**/*.json", base: "./src/data/authors" }),
  schema: z.object({
    name: z.string(),
    bio: z.string(),
    portfolio: z.url(),
  }),
});

const blog = defineCollection({
  loader: glob({ pattern: "**/*.{md,mdx}", base: "./src/content/blog" }),
  schema: z.object({
    title: z.string().max(60),
    description: z.string(),
    pubDate: z.coerce.date(),
    draft: z.boolean().default(false),
    // Define structural type-safe collection references
    author: reference("authors"),
    relatedPosts: z.array(reference("blog")).default([]),
  }),
});

export const collections = { blog, authors };
```

```typescript
// src/pages/posts/[id].astro
---
import { getCollection, getEntry, getEntries, render } from "astro:content";

// 1. Generate a type-safe static route for every non-draft collection entry
export async function getStaticPaths() {
  const posts = await getCollection("blog", ({ data }) => {
    return data.draft !== true;
  });

  return posts.map((post) => ({
    params: { id: post.id },
    props: { post },
  }));
}

// 2. Extract collection entry from static props
const { post } = Astro.props;

// 3. Render markdown content to a type-safe dynamic Component
const { Content } = await render(post);

// 4. Resolve the structural collection references
const author = await getEntry(post.data.author);
const relatedPosts = await getEntries(post.data.relatedPosts);
---
<html lang="en">
  <head>
    <title>{post.data.title}</title>
  </head>
  <body>
    <article>
      <h1>{post.data.title}</h1>
      <p class="meta">Published on: {post.data.pubDate.toDateString()}</p>
      
      <!-- Display Resolved Author Profile -->
      <div class="author-block">
        <p>By: <a href={author.data.portfolio}>{author.data.name}</a></p>
        <p>{author.data.bio}</p>
      </div>

      <!-- Render the main body content safely -->
      <Content />

      {relatedPosts.length > 0 && (
        <section class="related">
          <h3>You might also like:</h3>
          <ul>
            {relatedPosts.map((related) => (
              <li><a href={`/posts/${related.id}`}>{related.data.title}</a></li>
            ))}
          </ul>
        </section>
      )}
    </article>
  </body>
</html>
```

## d) Framework Component Integration with Strictly Serializable Props
This pattern shows how to cleanly interface with an interactive framework component passing valid serializable props, avoiding common function serialization traps.

### INCORRECT / ANTIPATTERN
Attempting to pass non-serializable callbacks or complicated DOM references across the island boundary.
```typescript
---
// src/pages/profile.astro (Antipattern)
import ReactProfileEditor from "../components/ReactProfileEditor";
const handleSave = (data) => { console.log(data); };
---
<!-- Functions cannot be serialized, causing instant runtime errors during hydration -->
<ReactProfileEditor client:load onSave={handleSave} />
```

### CORRECT / CLEAN
Constructing a serializable data boundary and utilizing standard form actions, custom client-side events, or Nano Stores to bridge interactivity.
```typescript
// src/pages/profile.astro
---
import ReactProfileEditor from "../components/ReactProfileEditor";

const userPayload = {
  id: "usr_992",
  username: "lizdev",
  joinedAt: new Date("2026-01-01"), // Date objects are serializable
  permissions: new Set(["read", "write"]), // Set objects are serializable
};
---
<html>
  <head>
    <title>Profile Editor</title>
  </head>
  <body>
    <!-- Pass strictly serializable payload objects to the React island -->
    <ReactProfileEditor client:load user={userPayload} />

    <script>
      // Intercept state changes on the client side using Custom Events
      document.addEventListener("profile:update", (event) => {
        const detail = (event as CustomEvent).detail;
        console.log("Client received update payload:", detail);
      });
    </script>
  </body>
</html>
```

```tsx
// src/components/ReactProfileEditor.tsx (React Island)
import React, { useState } from "react";

interface UserProp {
  readonly id: string;
  readonly username: string;
  readonly joinedAt: Date;
  readonly permissions: Set<string>;
}

export default function ReactProfileEditor({ user }: { readonly user: UserProp }) {
  const [name, setName] = useState(user.username);

  function dispatchUpdate() {
    // Communicate structural changes via standard Custom DOM Events instead of callbacks
    const event = new CustomEvent("profile:update", {
      detail: { id: user.id, username: name, updatedAt: new Date() },
      bubbles: true,
    });
    document.dispatchEvent(event);
  }

  return (
    <div className="profile-editor">
      <h3>Editing User ID: {user.id}</h3>
      <p>Member since: {user.joinedAt.toLocaleDateString()}</p>
      <input type="text" value={name} onChange={(e) => setName(e.target.value)} />
      <button onClick={dispatchUpdate}>Save Profile Changes</button>
    </div>
  );
}
```

---

# ANTI-PATTERNS AND PREVENTION CHECKLIST

Avoid these common development and structural anti-patterns in Astro:

## 1. Importing .astro components into framework components
* **Problem:** Trying to import a `.astro` file inside a React, Svelte, or Vue component file. This fails because Astro components do not have a client-side runtime, so they cannot be evaluated or rendered by client-side virtual DOMs or framework compilers.
* **Correction:** Always declare framework boundaries clearly. If you need to render server-exclusive Astro components inside a framework component, render the Astro component inside an `.astro` parent layout, passing it as a child or slot to the framework component.
```typescript
/* INCORRECT */
// Inside MyReactComponent.tsx
import AstroHeader from "./AstroHeader.astro"; // Compiler Error!

/* CORRECT */
// Inside index.astro
---
import ReactComponent from "../components/ReactComponent.tsx";
import AstroHeader from "../components/AstroHeader.astro";
---
<ReactComponent client:load>
  <AstroHeader slot="header" /> <!-- Passes static HTML rendered on the server -->
</ReactComponent>
```

## 2. Passing non-serializable props across boundaries
* **Problem:** Passing functions, classes with methods, or circular dependencies as props to hydrated islands (`client:*`) or Server Islands (`server:defer`). This crashes the runtime because Astro cannot serialize non-standard JavaScript types to pass them from server to client or across island boundaries.
* **Correction:** Ensure props only contain supported serializable types. Trigger actions or share data by leveraging Nano Stores, custom web APIs, form actions, or dispatching native browser Custom Events instead.
```typescript
/* INCORRECT */
<InteractiveWidget client:load onTrigger={() => { doSomething(); }} />

/* CORRECT */
<InteractiveWidget client:load />
<script>
  document.addEventListener("widget:trigger", () => { doSomething(); });
</script>
```

## 3. Unnecessary client hydration (Hydrating static content)
* **Problem:** Applying client directives (e.g., `client:load`) to framework components that do not manage dynamic interactive state, sending unused JS bundles to the client and slowing page loading.
* **Correction:** Omit hydration directives entirely on static framework components. Let Astro compile them as zero-JS server-rendered static HTML, and only hydrate components that strictly require browser interaction.
```typescript
/* INCORRECT */
<FooterLogo client:load />

/* CORRECT */
<FooterLogo /> <!-- Renders purely as static HTML, zero JS sent -->
```

## 4. Failing to handle State Lifecycle under View Transitions
* **Problem:** Expecting standard global scripts to execute after client-side SPA navigation when using `<ClientRouter />`. Traditional scripts only execute once per initial page load; subsequent client-side routing page swaps ignore them, causing broken event listeners on new views.
* **Correction:** Wrap script initializations inside `astro:page-load` event listeners instead of `DOMContentLoaded`. Apply the `data-astro-rerun` attribute to force inline scripts to execute on every transition. Ensure persistent elements carry their state correctly across navigation using `transition:persist`.
```html
<!-- INCORRECT -->
<script is:inline>
  document.addEventListener("DOMContentLoaded", () => { setupMobileMenu(); });
</script>

<!-- CORRECT -->
<script is:inline data-astro-rerun>
  document.addEventListener("astro:page-load", () => { setupMobileMenu(); });
</script>
```

## 5. Exceeding URL length limits with Server Island Props
* **Problem:** Passing massive data objects or lists as props to a server island using `server:defer`. This bloats the encrypted URL query parameters beyond the browser's 2048-byte threshold, forcing Astro to fallback to uncached POST requests and defeating CDN caching.
* **Correction:** Pass only lightweight, atomic references (e.g., product identifiers or database keys) as props, and let the server-side island fetch the corresponding database records directly on demand.
```typescript
/* INCORRECT */
<ServerReviewBlock server:defer reviews={allProductReviewsArray} />

/* CORRECT */
<ServerReviewBlock server:defer productId="prod_8829" />
```

---

# UNIVERSAL PROJECT STRUCTURE AND CONFIGURATION

## Universal Project Structure
A universal and clean folder architecture designed to keep server code, interactive elements, schemas, and state logic isolated and structured:

```
/
├── public/                 # Raw, unprocessed static assets (favicons, robots.txt, pdfs)
├── src/                    # Primary source code directory
│   ├── components/         # Shared component registry
│   │   ├── static/         # Server-only .astro components (zero JS by default)
│   │   └── islands/        # Interactive framework components (React, Svelte, Vue)
│   ├── content/            # Data-heavy files for content collections
│   │   └── blog/           # Structural markdown, JSON, or YAML collections
│   ├── layouts/            # Page layouts defining overall HTML shells
│   ├── pages/              # REQUIRED folder for file-based routing entrypoints
│   │   ├── api/            # Server endpoints (API routes)
│   │   └── [...slug].astro # Dynamic dynamic routes
│   ├── stores/             # Global client-side atomic stores
│   │   └── cart.ts         # Cross-island Nano Stores state module
│   ├── styles/             # Global or utility CSS/Sass stylesheets
│   ├── content.config.ts   # Main Content Collections schema registrations
│   ├── env.d.ts            # Type definitions, extending App.Locals
│   └── fetch.ts            # Advanced routing request pipeline overrides
├── astro.config.mjs        # Core configuration for integrations and target SSR settings
├── package.json            # Project manifest and package scripts
└── tsconfig.json           # Explicit strict compiler directives and paths
```

## Production-Ready astro.config.mjs
This is a robust, production-ready configuration configured under dynamic SSR. It features strict performance constraints, secure remote image patterns, advanced route caching rules, and Content Security Policy (CSP) setups:

```javascript
import { defineConfig, passthroughImageService } from 'astro/config';
import node from '@astrojs/node';
import react from '@astrojs/react';
import svelte from '@astrojs/svelte';

export default defineConfig({
  // Configure standalone SSR mode
  output: 'server',
  adapter: node({
    mode: 'standalone',
  }),

  // Add Framework Integrations
  integrations: [
    react(),
    svelte(),
  ],

  // Route matching behavior for trailing slashes
  trailingSlash: 'never',

  // Security constraints for on-demand rendering (SSR)
  security: {
    checkOrigin: true,                // Provide CSRF protection on form submits
    actionBodySizeLimit: 2 * 1024 * 1024, // Restrict actions payload limits (2 MB)
    serverIslandBodySizeLimit: 1024 * 1024, // Protect server island payload size
    csp: {
      algorithm: 'SHA-256',           // Cryptographic hash function algorithm for scripts/styles
      directives: [
        "default-src 'self'",
        "img-src 'self' https://images.unsplash.com" // Configure secure external image domains
      ],
      styleDirective: {
        // Safe styling attributes allowing inline styles for Shiki or dynamic vars
        resources: [
          "'self'",
          { resource: "'unsafe-inline'", kind: "attribute" }
        ]
      }
    }
  },

  // Image optimization setup
  image: {
    // Configure secure remote patterns for image components
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.unsplash.com', // Strict wildcard subdomain patterns
      }
    ],
    // Automatically apply styles for responsive images
    responsiveStyles: true,
  },

  // Advanced Route Caching rules
  routeRules: {
    // Enforce Stale-While-Revalidate (SWR) for high-traffic API routes
    '/api/[...path]': { swr: 300 },
  }
});
```