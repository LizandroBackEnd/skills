---
name: tailwind-clean-architecture
description: Universal Tailwind CSS architecture, component styling, responsive layout design, and class organization skill. Directs the creation of production-ready, performant, and maintainable utility-first styles avoiding anti-patterns. Covers CVA (Class Variance Authority) component variants, tailwind-merge conflict resolution, responsive design patterns, semantic theming, and Prettier class sorting rules across any frontend project. Triggers on "tailwind", "tailwindcss", "tailwind classes", "cva", "tailwind config", "utility-first".
license: MIT
metadata:
  author: lizdev
  version: "1.0.0"
---

# CORE UTILITY-FIRST ARCHITECTURAL PRINCIPLES

## 1. Utility-First Mindset vs. Premature Abstraction
Tailwind CSS provides visual consistency and rapid development by constraining design choices to a predefined system of design tokens. Instead of spending time inventing custom CSS class names and managing large, disjointed stylesheets, styles are composed directly in the markup using single-purpose primitive utility classes.
1. Speed of Iteration: Styling directly in the markup avoids context switching between HTML/JS template files and CSS stylesheets, allowing UI designs to come together rapidly.
2. Safe Modification: Since utility classes are scoped to individual elements, adding, modifying, or removing a class from an HTML element only affects that specific element, eliminating the risk of accidental regressions elsewhere in the application.
3. Linear CSS Payload: As the application grows, the production CSS bundle size does not grow linearly because the same set of reusable utility classes is composed repeatedly across different layouts.
4. Avoid Premature Abstraction: Developers new to utility-first styling often attempt to abstract repetitive classes too early. In practice, localization of duplication is acceptable and can often be managed using basic editor features like multi-cursor editing in single files. Abstractions should only be created when they solve a systemic maintenance problem.

## 2. Extraction Strategy: Templates vs. CSS Extraction
When styling patterns are repeated across multiple layouts, developers must choose the appropriate extraction boundary:
1. Component-Based Templates: For modern frontend projects using React, Vue, Svelte, or Astro, the primary method of reusing styles is extracting proper template components or partials. This encapsulates both structural markup and presentation styles into a single source of truth, ensuring high portability and easy maintenance.
2. CSS Class Extraction: Writing custom CSS classes (e.g., using `@layer components` in Tailwind) is only appropriate when component-based template partials are too heavy-handed for a lightweight templating language like ERB or Twig. For complex components composed of multiple nested HTML elements, template component extraction is highly recommended over CSS extraction to ensure that structure and style remain tightly encapsulated.

## 3. Strict Guidelines on `@apply` Overuse
Using the `@apply` directive to compile list of utility classes into custom CSS rules (e.g., `@apply px-4 py-2 bg-blue-500`) introduces severe architectural drawbacks and must be strictly limited:
1. Breaks Cohesion: Overusing `@apply` separates visual design from structural markup, returning the codebase to traditional, monolithic CSS architectures where class names must be invented and maintained.
2. Increases Bundle Size: Extracting classes via `@apply` into multiple custom CSS classes duplicates CSS declarations across rules, directly undermining Tailwind's build-time optimizations and increasing the final network payload.
3. Resolving Conflicts: Static `@apply` directives lose the runtime composition power of Tailwind, making it extremely difficult to override properties dynamically via component props. Use component-based templates and dynamic utility merging instead of `@apply` class definitions.

---

# COMPONENT DESIGN PATTERNS (CVA AND TAILWIND-MERGE)

## 1. Type-Safe Variant Creation using CVA (Class Variance Authority)
In component-driven architectures, interactive elements (like buttons, badges, and cards) require distinct visual variations depending on their functional state, semantic meaning, or size. Class Variance Authority (CVA) acts as a declarative engine to define type-safe variants.
1. Declarative Configurations: CVA separates baseline styles (common to all states) from variable variants (like color themes and sizes), mapping them to JavaScript structures.
2. Automatic Type Inference: By wrapping the styling schema in CVA, TypeScript automatically generates type boundaries for the component's styling props, preventing invalid configuration parameters at compile time.
3. Clean Default Assignments: CVA supports declaring default variants, which guarantees that baseline styles resolve correctly when dynamic attributes are omitted.

## 2. Dynamic Class Merging and Conflict Resolution (The `cn` Utility)
When building reusable components, consumers often need to pass custom styling classes from outside (e.g., via a `className` prop) to append or modify internal styles. However, simply concatenating strings of utility classes creates style conflicts:
1. The CSS Specwins Conflict Rule: In CSS, if two conflicting classes target the same property (e.g., `flex` and `grid`), the rule that appears later in the compiled *stylesheet* wins, regardless of the order they are written in the HTML class attribute.
2. The `tailwind-merge` Solution: To safely override internal styles, use the `twMerge` utility function from the `tailwind-merge` library. `twMerge` parses the class list, understands Tailwind's namespace hierarchy, and strips away overridden styles.
3. The Unified `cn` Helper: To support both conditional evaluation (traditionally handled by `clsx`) and robust utility overrides, declare a unified `cn` function:
```typescript
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

## 3. Mobile-First Responsive Design Rules
Tailwind CSS utilizes a mobile-first responsive breakpoint system inspired by standard device resolutions.
1. Unprefixed vs. Prefixed Utilities: Unprefixed utility classes apply to all screen sizes (starting from mobile viewports), whereas prefixed utilities (such as `md:flex`) apply strictly at the specified breakpoint *and above*.
2. Mobile Targeting Directive: Never use `sm:` classes to target mobile screens. The `sm:` variant applies at the small breakpoint (40rem / 640px) and above, not on mobile. Mobile layouts must be written using unprefixed utility classes, and larger screens must layer on overrides using prefixed classes.
3. Explicit Range Targeting: To restrict a style to a specific breakpoint range, stack a responsive variant with a `max-*` variant (e.g., `md:max-xl:flex` applies flex layouts exclusively between the `md` and `xl` breakpoints).

---

# WORKFLOW: DESIGNING AND REFACTORING UI

When constructing or refactoring UI components from design specifications, engineers and AI assistants must follow this systematic, 4-step workflow:

## Step 1: Layout and Spatial Scaffolding
Analyze the layout geometry of the design specification. Establish the outer layout container using Flexbox or CSS Grid utilities to position core structural areas. Use the standard Tailwind spacing scale (`gap`, `p`, `m`) to define relative distance, ensuring no hardcoded pixel offsets are injected.

## Step 2: Adaptive Responsive Styling
Define the baseline layout for mobile viewports using unprefixed utility classes. Progressively scale the layout upwards for tablet and desktop viewports using responsive prefixes (`md:`, `lg:`) to adjust positioning, widths, and columns. Wrap container queries (`@container` on parents, `@md:` on child elements) where component-level encapsulation is required over viewport-level styling.

## Step 3: Core Typography and Visual Dressing
Style internal copy, imagery, and interactive elements. Apply type scale, weights, colors, and line-heights dynamically based on standard design tokens. Layer on presentation properties including backgrounds, borders, shadows, and smooth transitions.

## Step 4: Refactor, Deduplicate, and Secure Boundaries
Locate repetitively authored structures. Evaluate whether loops or multi-cursor edits suffice, or if template components are required. Configure state variants (hover, focus, dark mode). Wrap dynamic combinations in a type-safe variant wrapper (CVA) and wrap final output class lists in the `cn` helper to allow safe consumer overrides.

---

# ADVANCED PATTERNS AND CODE EXAMPLES

## a) Reusable UI Component (CVA with Typed Variants)

### INCORRECT / ANTIPATTERN
Manual string interpolation of variants that bypasses static analysis, fails to handle specificity conflicts, and contains no TypeScript validation.
```typescript
// Warning: Bypasses static class analysis and causes specificity conflicts
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant: 'primary' | 'secondary';
  size: 'sm' | 'md';
}

export function Button({ variant, size, className, ...props }: ButtonProps) {
  // Overlapping padding and color classes will conflict; latter compiled in CSS wins
  const baseClass = "rounded font-medium px-4 py-2 bg-blue-500 text-white";
  const variantClass = variant === 'secondary' ? "bg-gray-200 text-gray-800" : "";
  const sizeClass = size === 'sm' ? "px-2 py-1 text-xs" : "";

  return (
    <button 
      className={`${baseClass} ${variantClass} ${sizeClass} ${className}`}
      {...props}
    />
  );
}
```

### CORRECT / CLEAN
Clean CVA variant configuration with strict types, default values, and resolved consumer-provided style overrides using `tailwind-merge`.
```typescript
import * as React from 'react';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '../utils/cn'; // Implements twMerge + clsx

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-lg font-semibold tracking-tight transition-colors focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-indigo-600 text-white hover:bg-indigo-700 focus-visible:outline-indigo-600',
        secondary: 'bg-white text-gray-900 border border-gray-300 hover:bg-gray-50 focus-visible:outline-gray-300 dark:bg-gray-800 dark:text-white dark:border-gray-700 dark:hover:bg-gray-700',
        danger: 'bg-red-600 text-white hover:bg-red-700 focus-visible:outline-red-600'
      },
      size: {
        sm: 'px-3 py-1.5 text-xs',
        md: 'px-4 py-2 text-sm',
        lg: 'px-5 py-2.5 text-base'
      }
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md'
    }
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, ...props }, ref) => {
    return (
      <button
        ref={ref}
        className={cn(buttonVariants({ variant, size }), className)}
        {...props}
      />
    );
  }
);

Button.displayName = 'Button';
```

## b) Handling Complex Interaction States (Group and Peer Modifiers)

### INCORRECT / ANTIPATTERN
Using brittle custom CSS selectors or heavy JavaScript `onMouseEnter` state-handling scripts to update nested child visual layouts.
```typescript
// Warning: Performance penalty and unnecessary JS re-renders
import { useState } from 'react';

export function Card() {
  const [isHovered, setIsHovered] = useState(false);

  return (
    <div 
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
      className="p-6 border border-gray-200 rounded-xl"
    >
      <h3 className={isHovered ? "text-blue-600" : "text-gray-900"}>Title</h3>
      <p className={isHovered ? "text-blue-500" : "text-gray-500"}>Description</p>
    </div>
  );
}
```

### CORRECT / CLEAN
Leveraging native CSS pseudo-classes using Tailwind's `group` and `peer` modifiers to control descendant elements and sibling nodes entirely in markup.
```html
<!-- Parent marked as group, named peer marked to differentiate sibling logic -->
<div class="group relative flex items-center justify-between rounded-xl border border-gray-200 p-6 bg-white hover:bg-gray-50 transition-all dark:bg-gray-900 dark:border-gray-800">
  <div class="flex items-center gap-4">
    <div class="rounded-lg p-2 bg-indigo-50 group-hover:bg-indigo-100 transition-colors">
      <svg class="size-6 stroke-indigo-600 transition-transform group-hover:scale-110" viewBox="0 0 24 24" fill="none">
        <path d="M12 4.5v15m7.5-7.5h-15" stroke-width="2" stroke-linecap="round"/>
      </svg>
    </div>
    <div>
      <!-- Child elements react to parent hover via group-hover -->
      <h3 class="text-sm font-semibold text-gray-900 group-hover:text-indigo-600 transition-colors dark:text-white">
        Project Assets
      </h3>
      <p class="text-xs text-gray-500 group-hover:text-gray-700 transition-colors dark:text-gray-400">
        Review shared team files and libraries.
      </p>
    </div>
  </div>

  <!-- Interactive Peer Sibling Example -->
  <div class="flex items-center gap-2">
    <input type="checkbox" id="terms" class="peer/terms size-4 rounded border-gray-300 text-indigo-600 focus:ring-indigo-500" />
    <label for="terms" class="text-xs font-medium text-gray-500 peer-checked/terms:text-indigo-600 peer-checked/terms:font-semibold select-none">
      Acknowledge
    </label>
  </div>
</div>
```

## c) Dark Mode Implementation (Semantic Tokens)

### INCORRECT / ANTIPATTERN
Hardcoding light and dark values everywhere manually without a unified design token contract, resulting in visual drift.
```html
<!-- Fragile: Inconsistent styling across pages without standard semantic background contracts -->
<div class="bg-gray-50 dark:bg-zinc-900 border-gray-200 dark:border-stone-800">
  <h1 class="text-gray-950 dark:text-slate-100">Panel Heading</h1>
  <p class="text-gray-600 dark:text-neutral-400">Metadata description</p>
</div>
```

### CORRECT / CLEAN
Enforce clean semantic layering of backgrounds, text, and borders in light/dark modes using standard color palette scales.
```html
<!-- Systematic Layering: Light/Dark mode contrasts mapped across consistent steps (50, 500, 900, 950) -->
<div class="bg-white dark:bg-gray-950 border border-gray-200 dark:border-gray-800 rounded-2xl p-8 shadow-sm">
  <!-- Interactive Form Row -->
  <label class="block">
    <span class="block text-sm font-semibold text-gray-900 dark:text-gray-100">
      Account Email
    </span>
    <input 
      type="email" 
      placeholder="user@organization.com"
      class="mt-2 block w-full rounded-lg border border-gray-300 bg-gray-50 px-4 py-2.5 text-sm text-gray-900 placeholder:text-gray-400 focus:border-indigo-500 focus:bg-white focus:outline-none dark:border-gray-700 dark:bg-gray-900 dark:text-white dark:placeholder:text-gray-500 dark:focus:border-indigo-500 dark:focus:bg-gray-950" 
    />
  </label>
  
  <p class="mt-3 text-xs text-gray-500 dark:text-gray-400">
    We will never share your address with external providers.
  </p>
</div>
```

## d) Complex Layouts (Flexbox & Grid without Hardcoded Pixels)

### INCORRECT / ANTIPATTERN
Using absolute pixel-based widths or nested layouts with hardcoded magic values, leading to breakage on diverse viewport dimensions.
```html
<!-- Fragile: Hardcoded width sizing limits responsiveness on smaller viewports -->
<div class="flex items-center" style="width: 1200px;">
  <div style="width: 400px; margin-right: 24px;">Sidebar</div>
  <div style="width: 776px;">Main Content Area</div>
</div>
```

### CORRECT / CLEAN
Fluid, responsive multi-column setups utilizing Tailwind's CSS Grid and Flexbox primitives configured via standard spacing intervals.
```html
<!-- Fluid, highly-maintainable layout shifting from stacked on mobile to multi-column on medium screens -->
<div class="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
  <div class="grid grid-cols-1 gap-8 lg:grid-cols-12">
    <!-- Navigation Sidebar: Span 3 of 12 columns on desktop -->
    <aside class="lg:col-span-3 space-y-4">
      <nav class="flex flex-col gap-1 rounded-xl bg-gray-50 p-4 dark:bg-gray-900">
        <a href="#" class="rounded-lg px-3 py-2 text-sm font-semibold text-indigo-600 bg-white shadow-sm dark:bg-gray-800 dark:text-white">Overview</a>
        <a href="#" class="rounded-lg px-3 py-2 text-sm font-medium text-gray-700 hover:bg-gray-100 hover:text-gray-900 dark:text-gray-300 dark:hover:bg-gray-800 dark:hover:text-white">Settings</a>
        <a href="#" class="rounded-lg px-3 py-2 text-sm font-medium text-gray-700 hover:bg-gray-100 hover:text-gray-900 dark:text-gray-300 dark:hover:bg-gray-800 dark:hover:text-white">billing</a>
      </nav>
    </aside>

    <!-- Main Content Panel: Span 9 of 12 columns on desktop -->
    <main class="lg:col-span-9 space-y-6">
      <div class="grid grid-cols-1 gap-6 sm:grid-cols-2 xl:grid-cols-3">
        <!-- Dashboard Widgets -->
        <div class="rounded-xl border border-gray-200 bg-white p-6 dark:border-gray-800 dark:bg-gray-950">
          <p class="text-xs font-medium text-gray-500 uppercase dark:text-gray-400">Total Balance</p>
          <h2 class="mt-2 text-2xl font-bold text-gray-900 dark:text-white">$45,231.89</h2>
        </div>
        <div class="rounded-xl border border-gray-200 bg-white p-6 dark:border-gray-800 dark:bg-gray-950">
          <p class="text-xs font-medium text-gray-500 uppercase dark:text-gray-400">Active Licenses</p>
          <h2 class="mt-2 text-2xl font-bold text-gray-900 dark:text-white">142</h2>
        </div>
        <div class="rounded-xl border border-gray-200 bg-white p-6 sm:col-span-2 xl:col-span-1 dark:border-gray-800 dark:bg-gray-950">
          <p class="text-xs font-medium text-gray-500 uppercase dark:text-gray-400">Server Status</p>
          <h2 class="mt-2 text-2xl font-bold text-green-600 dark:text-green-500">Nominal</h2>
        </div>
      </div>
    </main>
  </div>
</div>
```

---

# ANTI-PATTERNS AND PREVENTION CHECKLIST

Avoid the following common compilation and architectural pitfalls in Tailwind CSS:

## 1. Dynamic Class Name Construction (String Interpolation)
* Anti-pattern: Constructing class names dynamically at runtime using string interpolation.
* Impact: Tailwind CSS works by scanning static source files for complete class names at build time. If a class is constructed dynamically (e.g., `text-${color}-500`), Tailwind's compiler will not recognize the class name, it will not be generated, and the element will remain unstyled in production.
* Corrective action: Always write complete class names in your source files, or map dynamic properties to complete static class structures.
```typescript
/* INCORRECT */
function Alert({ color }) { return <div className={`text-${color}-600`}>Warning</div>; }

/* CORRECT */
const ALERT_COLORS = {
  red: "text-red-600",
  yellow: "text-yellow-600",
  blue: "text-blue-600"
};
function Alert({ color }) { return <div className={ALERT_COLORS[color]}>Warning</div>; }
```

## 2. Arbitrary Sizing Proliferation
* Anti-pattern: Injecting arbitrary pixel values into layout structures (e.g., `w-[342px]`, `mt-[13px]`).
* Impact: Breaks design consistency, bypasses the standard layout spacing contract, and creates highly fragmented layouts that are hard to maintain.
* Corrective action: Use predefined design spacing scale utilities (e.g., `w-80`, `mt-3`). Use arbitrary values *strictly* for genuine one-off design exceptions like brand-specific colors or dynamic layout calculations.

## 3. The `@apply` Soup in CSS Stylesheets
* Anti-pattern: Compiling long strings of utility classes into global stylesheets using the `@apply` directive.
* Impact: Violates utility-first philosophies, bloats production bundle payloads, breaks direct visual cohesion with source templates, and complicates dynamic styling overrides.
* Corrective action: Restructure redundant HTML blocks into reusable template components (React, Vue, Svelte, or Astro) instead of stylesheet rules.

## 4. Duplicate Property Assignment Conflicts
* Anti-pattern: Writing conflicting classes targeting the same CSS property directly on the same element.
* Impact: Leads to silent, non-deterministic bugs where styles look correct locally but change in production, depending strictly on the ordering of compiled declarations in the CSS bundle.
* Corrective action: Conditionally render mutually exclusive classes, or merge utility inputs using `tailwind-merge`.

---

# CONFIGURATION, THEMING & TOOLING PRESETS

## 1. Standard Semantic Tailwind Configuration
Tailwind v4 defines design tokens directly in the CSS entry file using the `@theme` directive, removing the dependency on heavy JavaScript-based configuration files. This configuration exposes extended color tokens, customized typography scales, and seamless integration with CSS variables.

```css
@import "tailwindcss";

@theme {
  /* Extended Semantic Color Palette */
  --color-brand-50: oklch(0.97 0.01 220);
  --color-brand-100: oklch(0.93 0.03 220);
  --color-brand-500: oklch(0.62 0.18 220);
  --color-brand-900: oklch(0.24 0.08 220);
  --color-brand-950: oklch(0.13 0.03 220);

  /* Customized Typography Scales */
  --text-display: 2.25rem;
  --text-display--line-height: 1.1;
  --text-body: 1rem;
  --text-body--line-height: 1.5;

  /* Customized Border Radius Tokens */
  --radius-button: 0.5rem;
  --radius-card: 1rem;

  /* Custom Transitions */
  --ease-spring: cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
```

## 2. Prettier Class Sorting Configuration
For deterministic, team-wide consistency and clean code reviews, all Tailwind classes must be sorted automatically during saving or formatting. This is configured using `prettier-plugin-tailwindcss`.

Add the following to your project's `.prettierrc` configuration file:

```json
{
  "plugins": ["prettier-plugin-tailwindcss"],
  "tailwindConfig": "./tailwind.config.js",
  "tailwindAttributes": ["className", "class", "classList"],
  "tailwindFunctions": ["clsx", "cva", "cn"]
}
```