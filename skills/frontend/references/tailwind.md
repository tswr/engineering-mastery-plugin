# Tailwind CSS Frontend Reference

Utility-first CSS patterns with Tailwind CSS v3+. Covers responsive design, dark mode, design tokens, layout patterns, and accessibility considerations. The core advantage is locality — you see how an element looks by reading its class list, not by tracing through stylesheet layers.

---

## Styling Strategies

```html
<!-- Utility-first: compose styles from single-purpose classes -->
<article class="rounded-lg border border-gray-200 bg-white p-6 shadow-sm">
  <h3 class="text-lg font-semibold text-gray-900">Product Name</h3>
  <p class="mt-2 text-sm text-gray-600">Product description goes here.</p>
  <span class="mt-4 inline-block text-xl font-bold text-indigo-600">$29.99</span>
</article>

<!-- Responsive: mobile-first — base styles for small screens, layer at breakpoints -->
<div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
  <!-- 1 col mobile, 2 at 640px, 3 at 1024px, 4 at 1280px -->
  <ProductCard />
</div>

<!-- Dark mode: dark: variant respects prefers-color-scheme or class strategy -->
<div class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">
  <h1 class="text-gray-900 dark:text-white">Dashboard</h1>
  <p class="text-gray-600 dark:text-gray-400">Welcome back.</p>
  <div class="border-t border-gray-200 dark:border-gray-700"></div>
</div>
```

```js
// tailwind.config.js — design tokens as single source of truth
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/**/*.{js,ts,jsx,tsx,vue}"],
  darkMode: "class",  // or "media" for automatic OS-preference
  theme: {
    extend: {
      colors: {
        // Semantic roles with intentional shade ranges
        brand: {
          50: "#eff6ff", 100: "#dbeafe",
          500: "#3b82f6",  // primary action
          600: "#2563eb",  // hover
          700: "#1d4ed8",  // active
        },
        success: { DEFAULT: "#16a34a", light: "#dcfce7" },
        error:   { DEFAULT: "#dc2626", light: "#fee2e2" },
      },
      // Constrained spacing prevents arbitrary values
      spacing: { 18: "4.5rem" },
      fontSize: {
        caption: ["0.75rem", { lineHeight: "1rem" }],
        body:    ["1rem",    { lineHeight: "1.5rem" }],
        h2:      ["1.5rem",  { lineHeight: "2rem", fontWeight: "700" }],
        h1:      ["2.25rem", { lineHeight: "2.5rem", fontWeight: "800" }],
      },
    },
  },
};
```

## Component Architecture

```html
<!-- Flex: navigation bar -->
<nav class="flex items-center justify-between px-6 py-4">
  <a href="/" class="text-xl font-bold">Logo</a>
  <ul class="flex gap-6">
    <li><a href="/products" class="text-gray-700 hover:text-gray-900">Products</a></li>
  </ul>
</nav>

<!-- Flex: centering content -->
<div class="flex min-h-screen items-center justify-center">
  <LoginForm />
</div>

<!-- Grid: dashboard with sidebar -->
<div class="grid min-h-screen grid-cols-1 md:grid-cols-[250px_1fr]">
  <aside class="border-r border-gray-200 p-4">Sidebar</aside>
  <main class="p-6">Content</main>
</div>

<!-- @apply sparingly — only for patterns repeated dozens of times
     that cannot be extracted as components -->
<style>
.btn-primary {
  @apply rounded-lg bg-brand-500 px-4 py-2 font-medium text-white
         hover:bg-brand-600 active:bg-brand-700
         focus-visible:outline focus-visible:outline-2
         focus-visible:outline-offset-2 focus-visible:outline-brand-500;
}
/* Prefer extracting a <Button> component over @apply in most cases */
</style>
```

## Accessibility

```html
<!-- Focus indicators: style them, never remove — focus-visible targets keyboard only -->
<button class="rounded-lg bg-indigo-600 px-4 py-2 text-white
               hover:bg-indigo-700
               focus-visible:outline focus-visible:outline-2
               focus-visible:outline-offset-2 focus-visible:outline-indigo-600">
  Save Changes
</button>

<!-- Contrast: gray-600 on white passes AA (4.5:1); gray-400 does NOT for body text -->
<p class="text-gray-600">Passes contrast requirements.</p>

<!-- Do not rely on color alone — add text for status information -->
<div class="flex items-center gap-2 text-error">
  <svg aria-hidden="true" class="h-5 w-5"><!-- error icon --></svg>
  <span>Password must be at least 8 characters</span>
</div>

<!-- sr-only: visually hidden but announced by screen readers -->
<a href="/cart" class="relative">
  <svg aria-hidden="true" class="h-6 w-6"><!-- cart icon --></svg>
  <span class="sr-only">Shopping cart</span>
  <span class="absolute -right-2 -top-2 flex h-5 w-5 items-center justify-center
               rounded-full bg-red-600 text-xs text-white"
        aria-label="3 items in cart">3</span>
</a>

<!-- Reduced motion: respect user preference with motion-safe -->
<div class="transition-transform duration-200 motion-safe:hover:scale-105">
  Hover me
</div>
```

## Performance

```html
<!-- Responsive images: serve appropriate sizes per viewport -->
<img
  srcset="/hero-640.webp 640w, /hero-1024.webp 1024w, /hero-1920.webp 1920w"
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
  src="/hero-1024.webp" alt="Product showcase"
  class="h-auto w-full rounded-lg object-cover"
  loading="eager"
/><!-- eager for above-the-fold LCP images; lazy for below-the-fold -->

<!-- Fixed dimensions prevent layout shift (CLS) -->
<img src="/avatar.webp" alt="User avatar" width="48" height="48"
     class="h-12 w-12 rounded-full" loading="lazy" />

<!-- Skeleton: reserve space to prevent layout shift while loading -->
<div class="h-64 w-full animate-pulse rounded-lg bg-gray-200 dark:bg-gray-700"></div>
```
