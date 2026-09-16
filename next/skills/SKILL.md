---
name: nextjs-clean-architecture
description: Universal Next.js App Router clean architecture, React Server Components (RSC), data fetching strategies, server actions, and performance optimization skill. Directs the creation of production-ready, highly performant, type-safe, and scalable web applications avoiding client-side over-hydration and server waterfalls. Covers RSC/Client boundary composition, server action mutations with optimistic UI, dynamic Route Handlers, edge middleware, caching revalidation, and asset optimization across any Next.js project. Triggers on "nextjs", "next.js", "app router", "server components", "rsc", "server actions", "route handler".
license: MIT
metadata:
  author: lizdev
  version: "1.0.0"
---

# ARCHITECTURAL FOUNDATIONS & RENDERING PARADIGM

## React Server Components (RSC) by Default
Next.js App Router enforces React Server Components (RSC) as the foundational baseline for all pages and layouts. Server Components render exclusively on the server, producing a serialized React Server Component Payload (RSC Payload) and HTML without shipping component source code or dependencies to the browser.
1. Zero Bundle Size Impact: Heavy libraries (e.g., Markdown parsers, date formatters, sanitizers, or ORM clients) imported inside Server Components remain strictly on the server, ensuring zero client bundle size overhead.
2. Proximity to Data Sources: Server Components execute adjacent to databases, microservices, and internal caches, eliminating client-to-server network latency and securing sensitive environment variables.
3. Streaming and Granular Invalidation: Server-rendered HTML is streamed incrementally to the browser, improving First Contentful Paint (FCP) and Largest Contentful Paint (LCP) while supporting targeted component re-rendering during client navigations.

## Client Component Boundaries ('use client')
Client Components provide interactivity, state management, event listeners, and access to browser APIs.
1. Pushing Boundaries to Leaf Nodes: The 'use client' directive marks a boundary between server and client module graphs. Once a file is designated with 'use client', all imported modules and child components within its import graph are bundled for the client. To minimize JavaScript bundle bloat, place 'use client' strictly at interactive leaf nodes (e.g., buttons, forms, search inputs, modal triggers) rather than at page or layout roots.
2. Boundary Propagation: Avoid annotating parent pages, layouts, or data-fetching wrappers with 'use client'. Keep higher-level routes as Server Components to preserve server data fetching and streaming advantages.

## Server-Client Composition Patterns
To render server-fetched UI inside an interactive Client Component without converting the Server Component into a client module, use slot composition via React props (e.g., children).
1. Children Slot Composition: Pass a Server Component as a child or named prop into a Client Component parent. The Server Component is executed on the server, serialized into the RSC Payload, and slotted into the Client Component DOM structure during client hydration.
2. Serializable Props Constraint: Data passed directly across the RSC-to-Client boundary must be JSON-serializable primitives, plain objects, arrays, or promises. Functions, classes, or non-serializable objects cannot cross this boundary.

---

# DATA FETCHING, CACHING & MUTATION STRATEGIES

## Data Fetching in Server Components
Fetch data directly within Server Components using async/await syntax.
1. Direct Server Data Access: Execute database queries, ORM operations, or third-party API fetches directly inside component functions. Eliminate client-side boilerplate (such as `useEffect`, `useState`, or custom fetching hooks) for initial data load.
2. Automatic Request Deduplication: In-flight HTTP requests using the native `fetch` API across the React component tree are automatically deduplicated during a single render pass. For non-fetch data operations (e.g., ORM or database calls), wrap data access functions with `React.cache`.

## Next.js Caching and Revalidation
Next.js incorporates a multi-tiered caching architecture for data and rendered segments.
1. Caching Strategies: Control fetch behavior using segment configuration or explicit fetch options (`cache: 'force-cache'`, `next: { revalidate: 3600, tags: ['posts'] }`).
2. On-Demand Revalidation: Purge cached data on-demand following data mutations using `revalidatePath('/posts')` to revalidate specific route paths or `revalidateTag('posts')` to revalidate tagged data operations across multiple routes.

## Server Actions ('use server')
Server Actions are asynchronous server functions invoked directly from the client via HTTP POST requests, enabling type-safe data mutations without writing custom API routes.
1. Declaration: Annotate functions with `'use server'` at the function body level (inside Server Components) or at the top of a dedicated module file (for export to Client Components).
2. Form Integration and Pending States: Use `useActionState` to track mutation return state, execution errors, and pending status. Use `useFormStatus` inside child button elements to render contextual pending indicators during form dispatch.
3. Optimistic UI Updates: Implement `useOptimistic` inside Client Components to apply temporary UI updates instantly while the asynchronous Server Action executes on the server, automatically rolling back if the server mutation fails.

---

# WORKFLOW: ROUTE AND FEATURE DESIGN

Follow this 5-step workflow when building pages or refactoring features in the App Router:

## Step 1: Layout and Route Segment Scaffolding
Define the route directory inside `app/` and establish `layout.tsx` for shared persistent UI wrappers. Implement `loading.tsx` to wrap dynamic segments in automatic React `Suspense` boundaries for instant streaming.

## Step 2: Server-First Data Architecture
Fetch required domain data inside the `page.tsx` Server Component or nested async sub-components. Execute fetches in parallel using `Promise.all` where data requirements are independent to prevent sequential render waterfalls.

## Step 3: Interactive Boundary Isolation
Identify elements requiring browser APIs, React state, or event handlers. Create isolated, single-responsibility Client Components inside a `components/` or `_components/` directory marked with `'use client'`.

## Step 4: Slot Composition and Interleaving
Pass server-rendered UI sub-trees or server-fetched promises into Client Components using the `children` prop or `React.use()` to preserve zero-bundle-size server rendering for non-interactive elements.

## Step 5: Mutation and Cache Revalidation
Implement Server Actions for form submissions and mutations inside `lib/actions/`. Apply input validation via Zod, execute database updates, call `revalidateTag` or `revalidatePath` to purge stale server caches, and handle redirection via `redirect`.

---

# ADVANCED PATTERNS AND CODE EXAMPLES

## a) Isolating 'use client' to Interactive Leaf Nodes

### INCORRECT / ANTIPATTERN
Annotating a page container with 'use client' forces the entire page, its child sub-tree, and all imported utilities into the client JavaScript bundle, eliminating RSC server-rendering benefits.
```tsx
// app/dashboard/page.tsx
'use client' // Anti-pattern: Marking the entire page as a Client Component

import { useState, useEffect } from 'react'
import { AnalyticsChart } from '@/components/AnalyticsChart'
import { Header } from '@/components/Header'

export default function DashboardPage() {
  const [data, setData] = useState(null)

  useEffect(() => {
    fetch('/api/analytics')
      .then((res) => res.json())
      .then((data) => setData(data))
  }, [])

  if (!data) return <div>Loading...</div>

  return (
    <div>
      <Header />
      <AnalyticsChart data={data} />
    </div>
  )
}
```

### CORRECT / CLEAN
Keep the page component as an async Server Component that fetches data directly, and isolate interactivity inside a dedicated client leaf component.
```tsx
// app/dashboard/page.tsx (Server Component)
import { getAnalyticsData } from '@/lib/dal/analytics'
import { Header } from '@/components/Header'
import { InteractiveChartLeaf } from '@/components/features/InteractiveChartLeaf'

export default async function DashboardPage() {
  const analyticsData = await getAnalyticsData()

  return (
    <div>
      <Header />
      {/* Interactive behavior isolated strictly to the leaf component */}
      <InteractiveChartLeaf initialData={analyticsData} />
    </div>
  )
}

// components/features/InteractiveChartLeaf.tsx
'use client'

import { useState } from 'react'

interface ChartProps {
  readonly initialData: Array<{ id: string; value: number }>
}

export function InteractiveChartLeaf({ initialData }: ChartProps) {
  const [metricFilter, setMetricFilter] = useState<string>('all')

  const filteredData = initialData.filter((item) => 
    metricFilter === 'all' ? true : item.id === metricFilter
  )

  return (
    <div>
      <button onClick={() => setMetricFilter('all')}>Reset Filter</button>
      <div>Rendered Items: {filteredData.length}</div>
    </div>
  )
}
```

## b) Type-Safe Server Action with Zod Validation and Tag Revalidation

### INCORRECT / ANTIPATTERN
Unvalidated input, missing error boundaries, raw unhandled exceptions, and manual un-cached client refetches.
```tsx
// lib/actions/post.ts
'use server'

import { db } from '@/lib/db'

export async function createPostAction(formData: FormData) {
  // Anti-pattern: No authentication checks or type validation
  const title = formData.get('title') as string
  const content = formData.get('content') as string

  // Direct database insertion with raw unvalidated input
  await db.post.create({ data: { title, content } })
}
```

### CORRECT / CLEAN
Type-safe Server Action enforcing authentication, Zod schema validation, structured result objects, and cache revalidation.
```tsx
// lib/actions/post.ts
'use server'

import { z } from 'zod'
import { revalidateTag } from 'next/cache'
import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

const CreatePostSchema = z.object({
  title: z.string().min(3, 'Title must contain at least 3 characters').max(100),
  content: z.string().min(10, 'Content must contain at least 10 characters'),
})

export type ActionResult<T> = 
  | { success: true; data: T }
  | { success: false; error: string; fieldErrors?: Record<string, string[]> }

export async function createPostAction(
  prevState: unknown,
  formData: FormData
): Promise<ActionResult<{ id: string }>> {
  try {
    const session = await auth()
    if (!session?.user) {
      return { success: false, error: 'Unauthorized operation' }
    }

    const rawData = {
      title: formData.get('title'),
      content: formData.get('content'),
    }

    const parseResult = CreatePostSchema.safeParse(rawData)
    if (!parseResult.success) {
      return {
        success: false,
        error: 'Validation failed',
        fieldErrors: parseResult.error.flatten().fieldErrors,
      }
    }

    const newPost = await db.post.create({
      data: {
        title: parseResult.data.title,
        content: parseResult.data.content,
        authorId: session.user.id,
      },
    })

    revalidateTag('posts-list')
    return { success: true, data: { id: newPost.id } }
  } catch (err) {
    return { success: false, error: 'An unexpected server error occurred' }
  }
}
```

## c) Parallel Data Fetching in RSC to Prevent Waterfalls

### INCORRECT / ANTIPATTERN
Sequential `await` calls force requests to execute serially, introducing severe server-rendering latency waterfalls.
```tsx
// app/profile/[id]/page.tsx
import { getUserProfile, getUserPosts, getUserStats } from '@/lib/dal/user'

export default async function UserProfilePage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params

  // Anti-pattern: Sequential waterfalls
  const profile = await getUserProfile(id) // Takes 200ms
  const posts = await getUserPosts(id)     // Takes 300ms (starts after profile finishes)
  const stats = await getUserStats(id)     // Takes 150ms (starts after posts finishes)
  // Total execution time: 650ms

  return (
    <div>
      <h1>{profile.name}</h1>
      <div>Posts Count: {posts.length}</div>
      <div>Reputation: {stats.reputation}</div>
    </div>
  )
}
```

### CORRECT / CLEAN
Initiate requests concurrently using `Promise.all` to execute asynchronous operations in parallel.
```tsx
// app/profile/[id]/page.tsx
import { getUserProfile, getUserPosts, getUserStats } from '@/lib/dal/user'

export default async function UserProfilePage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params

  // Parallel data fetching via Promise.all
  const [profileResult, postsResult, statsResult] = await Promise.all([
    getUserProfile(id),
    getUserPosts(id),
    getUserStats(id),
  ])
  // Total execution time: ~300ms (bounded by longest single request)

  return (
    <div>
      <h1>{profileResult.name}</h1>
      <div>Posts Count: {postsResult.length}</div>
      <div>Reputation: {statsResult.reputation}</div>
    </div>
  )
}
```

## d) Type-Safe Route Handler with Status Codes and Validation

### INCORRECT / ANTIPATTERN
Unvalidated Request JSON parsing, lack of status codes, and unhandled runtime exceptions inside API routes.
```tsx
// app/api/telemetry/route.ts
import { NextResponse } from 'next/server'
import { db } from '@/lib/db'

export async function POST(request: Request) {
  // Anti-pattern: Unchecked json() parsing and missing status code response handling
  const body = await request.json()
  await db.telemetry.create({ data: body })
  return NextResponse.json({ success: true })
}
```

### CORRECT / CLEAN
Type-safe Route Handler utilizing `NextRequest`, `NextResponse`, input payload validation, and structured HTTP error status codes.
```tsx
// app/api/telemetry/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { db } from '@/lib/db'

const TelemetrySchema = z.object({
  eventId: z.string().uuid(),
  eventType: z.enum(['click', 'view', 'conversion']),
  timestamp: z.number().int().positive(),
})

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const parseResult = TelemetrySchema.safeParse(body)

    if (!parseResult.success) {
      return NextResponse.json(
        {
          error: 'Invalid telemetry payload',
          details: parseResult.error.flatten().fieldErrors,
        },
        { status: 400 }
      )
    }

    await db.telemetry.create({
      data: {
        eventId: parseResult.data.eventId,
        eventType: parseResult.data.eventType,
        eventTimestamp: new Date(parseResult.data.timestamp),
      },
    })

    return NextResponse.json(
      { status: 'acknowledged', eventId: parseResult.data.eventId },
      { status: 201 }
    )
  } catch (error) {
    return NextResponse.json(
      { error: 'Internal server error processing telemetry' },
      { status: 500 }
    )
  }
}
```

---

# ANTI-PATTERNS AND PREVENTION CHECKLIST

Avoid these common App Router architectural pitfalls:

## 1. Page-Root Client Boundary Contamination
* Anti-pattern: Adding `'use client'` at the top of layout or page files to access simple state or handlers, forcing the entire page sub-tree into client JavaScript bundles.
* Corrective action: Keep layouts and pages as async Server Components. Extract the specific interactive element (e.g., toggle button, input search, modal wrapper) into an isolated leaf component marked with `'use client'`.

## 2. Using useEffect for Data Fetching in RSC Setups
* Anti-pattern: Writing client-side `useEffect` data-fetching logic inside App Router components, causing client waterfalls, loading spinners, and double rendering.
* Corrective action: Fetch data directly inside async Server Components using `await fetch()` or ORM calls, passing rendered HTML or server-resolved promises to client components via React `use()`.

## 3. Leaking Secrets Across RSC Boundaries
* Anti-pattern: Importing server-only packages (e.g., database clients, private API keys) into files imported by Client Components.
* Corrective action: Enforce the `server-only` package at the top of server data access modules (`import 'server-only'`). This causes a build-time compilation error if a client module attempts to import the server file.

## 4. Blocking Page Navigation without Suspense
* Anti-pattern: Performing slow uncached server data fetches inside higher-level layouts or pages without establishing `loading.tsx` or `<Suspense>` boundaries.
* Corrective action: Wrap dynamic sub-components inside granular `<Suspense fallback={<Skeleton />}>` boundaries to allow static UI shell elements to stream instantly while dynamic data resolves asynchronously.

---

# PROJECT STRUCTURE AND MIDDLEWARE PRESETS

## Scalable Directory Structure
Organize App Router applications using domain separation, data access layer (DAL) isolation, and explicit component boundaries:

```
app/
├── (auth)/                  # Route group for authentication routes
│   ├── login/page.tsx
│   └── register/page.tsx
├── (dashboard)/             # Route group for main application layout
│   ├── layout.tsx           # Shared persistent dashboard navigation shell
│   ├── loading.tsx          # Global streaming suspense boundary
│   └── page.tsx             # Dashboard home server component
├── api/                     # Route Handlers for external webhooks or APIs
│   └── webhooks/route.ts
├── global.css
└── layout.tsx               # Root application layout
components/
├── ui/                      # Primitive atomic components (Button, Input, Card)
└── features/                # Domain-specific interactive client leaves
lib/
├── actions/                 # Type-safe Server Actions ('use server')
├── dal/                     # Data Access Layer ('server-only' data fetching)
├── db.ts                    # ORM or database client initialization
└── auth.ts                  # Authentication helpers and session checks
middleware.ts                # Edge routing and security middleware
```

## Production Middleware Template
This production-ready `middleware.ts` template handles session verification, edge redirects, security header injection, and path matching:

```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

const PUBLIC_ROUTES = ['/login', '/register', '/api/webhooks']

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl
  const sessionToken = request.cookies.get('session_token')?.value

  const isPublicRoute = PUBLIC_ROUTES.some((route) => pathname.startsWith(route))

  // 1. Unauthenticated redirect to login
  if (!sessionToken && !isPublicRoute) {
    const loginUrl = new URL('/login', request.url)
    loginUrl.searchParams.set('from', pathname)
    return NextResponse.redirect(loginUrl)
  }

  // 2. Authenticated user attempting to access auth pages
  if (sessionToken && (pathname === '/login' || pathname === '/register')) {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }

  // 3. Clone headers and attach security context
  const requestHeaders = new Headers(request.headers)
  requestHeaders.set('x-next-app-path', pathname)

  const response = NextResponse.next({
    request: {
      headers: requestHeaders,
    },
  })

  // 4. Inject Security Headers
  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-Content-Type-Options', 'nosniff')
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')

  return response
}

export const config = {
  matcher: [
    /*
     * Match all request paths except:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico, sitemap.xml, robots.txt (metadata files)
     */
    '/((?!_next/static|_next/image|favicon.ico|sitemap.xml|robots.txt).*)',
  ],
}
```
