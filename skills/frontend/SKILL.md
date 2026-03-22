---
name: frontend
description: "Apply when building web frontends: component architecture, state management, rendering strategies, accessibility, performance, styling, and data fetching. Trigger on mentions of React, Vue, frontend, components, CSS, Tailwind, accessibility, web vitals, or single-page application. Framework-specific idioms are in references/."
---

# Frontend Development Skill

Modern frontend development is component-driven. Every page, every feature, every interaction is composed from small, reusable pieces that manage their own rendering, state, and behavior. This shift — from page-centric thinking to component-centric thinking — is the foundation of every modern framework. Mastering frontend engineering means mastering the patterns that govern how components are designed, how state flows between them, how they render efficiently, and how they remain accessible to all users.

Performance and accessibility are not features you bolt on at the end. They are architectural decisions made at the start. A component that re-renders unnecessarily will degrade the experience for everyone. A form that lacks labels will exclude users who rely on assistive technology. A page that ships two megabytes of JavaScript will lose visitors before the first interaction. The principles in this skill treat these concerns as first-class engineering requirements, not afterthoughts.

This skill covers the universal patterns that apply across frameworks. For framework-specific idioms, consult the references below.

**Framework and tooling references:**

- `references/react.md` — React and Next.js patterns
- `references/vue.md` — Vue 3 and Nuxt patterns
- `references/tailwind.md` — Utility-first CSS with Tailwind
- `references/typescript.md` — Frontend TypeScript patterns

---

## 1. **Component Architecture**

Components are the building blocks of modern frontends. Hallie and Osmani in *Learning Patterns* establish that composition is the primary mechanism for building complex interfaces — you combine small, focused components rather than extending base classes. The official React documentation reinforces this with the single responsibility principle: each component should do one thing well. When a component grows to handle multiple concerns, split it.

The question "when should I split a component?" has a practical answer: when you can name the extracted piece independently. If a section of a component has a clear purpose — a search bar, a user avatar, a price display — it is a candidate for extraction. The goal is not the smallest possible components but components with clear, singular purposes.

Props define the contract between parent and child. Data flows downward through props, making the relationship explicit and traceable. A well-designed prop interface communicates what a component needs and nothing more. Avoid passing entire objects when the component only uses two fields — this creates hidden dependencies and makes re-renders harder to reason about.

State, by contrast, is internal — it represents data that changes over time within a component. Understanding the distinction between props and state is essential because it determines when and why a component re-renders.

A component re-renders when its state changes or when it receives new props. Failing to understand render boundaries is the root cause of most frontend performance problems. When a parent re-renders, all of its children re-render by default. This cascading behavior means that a single state change near the top of the tree can trigger re-renders across hundreds of components.

Structuring components so that state lives close to where it is used — rather than at the top of the tree — limits the blast radius of each re-render. This is sometimes called colocation: keep state as close as possible to the components that read it.

Memoization — caching the result of a component render or an expensive computation — is a tool for optimizing re-renders, but it is not a default strategy. Apply memoization only after measuring and identifying an actual performance problem. Premature memoization adds complexity without measurable benefit.

Hallie and Osmani describe two structural patterns that reinforce clean architecture. The Container/Presentational pattern separates data-fetching logic from display logic. Container components handle API calls, state management, and business logic. Presentational components receive data through props and render it. This separation makes presentational components reusable and testable in isolation — they are pure functions of their props, with no side effects or data dependencies.

The Compound Components pattern allows related components to share implicit state — a Tabs component and its Tab children communicate without the consumer needing to wire them together manually. This produces APIs that are intuitive to use and hard to misuse. The consumer declares the structure, and the parent component manages the coordination internally.

When designing component APIs, favor explicit configuration through props over implicit behavior through internal logic. A component that silently fetches data, manages its own error state, and controls its own visibility is hard to test and impossible to compose. A component that accepts data, callbacks, and visibility as props can be used in any context without surprises.

## 2. **State Management**

State management is where most frontend applications accumulate unnecessary complexity. Hallie and Osmani advocate starting with local state — useState in React, ref in Vue — and lifting state upward only when sibling components need access to the same data.

Global state stores should be reserved for truly app-wide concerns such as authentication status or theme preferences. Reaching for global state prematurely couples components that should be independent. When every component reads from a global store, changes to that store can trigger re-renders across the entire application.

The React documentation emphasizes a critical principle: derive state from existing state instead of duplicating it. If you can compute a value from props or other state, do not store it separately. Duplicated state creates synchronization bugs — two sources of truth that inevitably diverge.

A common example is storing both a list of items and a filtered version of that list. The filtered list can be computed on every render from the original list and the filter criteria. Storing it separately means you must remember to update both whenever either changes. The same principle applies to computed totals, sorted lists, and any value that is a transformation of existing state.

Distinguish between server state and client state. Server state is data fetched from an API — it is owned by the server, cached locally, and eventually consistent. Client state is UI-specific — whether a modal is open, which tab is selected, what the user has typed into a search field. These two categories have different lifecycles and should be managed with different tools.

Libraries like TanStack Query handle server state with caching and revalidation. Local component state or lightweight stores handle client state. Mixing these concerns — putting API data into a global store alongside UI flags — creates unnecessary complexity and makes cache invalidation manual rather than automatic. When the user edits a record, a server state library automatically invalidates the relevant cache and refetches. A hand-rolled global store requires you to find and update every location where that data is referenced.

Hallie and Osmani describe the Provider pattern as a mechanism for dependency injection, allowing deeply nested components to access shared values without passing props through every intermediate layer. This solves prop drilling — the antipattern of threading props through components that do not use them simply to reach a deeply nested child.

However, the solution to prop drilling is not always global state. Most state is local. Resist the impulse to globalize everything. When you find yourself passing a prop through three or four layers, first consider whether the component tree can be restructured.

Component composition — passing components as children rather than data as props — often eliminates the need for drilling without introducing global state. Instead of passing a user object through four layers so a deeply nested avatar can display it, pass the rendered avatar component itself. The intermediate layers do not need to know about the user object at all.

The hierarchy of state management decisions is: local state first, then lifted state for siblings, then composition to avoid drilling, then context or providers for cross-cutting concerns, and finally external stores only when the complexity of coordination demands it.

## 3. **Rendering Strategies**

Choosing the right rendering strategy is an architectural decision that affects performance, SEO, and user experience. Hallie and Osmani catalog the major approaches.

Client-Side Rendering sends a minimal HTML shell and renders everything in the browser with JavaScript. It works well for highly interactive applications like dashboards but produces poor initial load times and is invisible to search engines without additional work. The user sees a blank page until JavaScript downloads, parses, executes, and fetches data. For applications behind a login — admin panels, internal tools — this tradeoff is often acceptable. For public-facing content that must be discoverable and fast on first load, it is not.

Server-Side Rendering generates HTML on the server for each request. The user sees content quickly, and search engines can index it. The tradeoff is server cost — every request requires computation — and time-to-interactive. The page appears rendered, but buttons and links do not respond until the browser downloads and executes the JavaScript that hydrates the HTML.

Hydration is the process of attaching event listeners and component state to existing DOM nodes. The browser receives static HTML, then JavaScript runs to make it interactive. Mismatches between server-rendered HTML and client-rendered HTML cause hydration errors, which are among the most frustrating bugs in frontend development.

These mismatches occur when the server and client produce different output — often caused by using browser-only APIs during server rendering, or by relying on values like timestamps and random numbers that differ between server and client.

Static Site Generation renders pages at build time. It is ideal for content that changes infrequently — documentation, marketing pages, blogs. The result is fast, cacheable HTML with no per-request server cost. Pages can be served from a CDN edge close to the user, producing load times that server rendering cannot match.

Incremental Static Regeneration extends this by allowing individual pages to be regenerated in the background after a specified interval, combining the speed of static with the freshness of dynamic content. The first visitor after the interval triggers a regeneration, and subsequent visitors receive the updated page. This avoids full rebuilds when only a few pages change.

The React documentation introduces React Server Components, which render entirely on the server and send HTML to the client without adding to the client JavaScript bundle. This reduces the amount of JavaScript the browser must download, parse, and execute. Server Components can access databases and file systems directly, eliminating the need for API endpoints in many cases.

Streaming SSR sends HTML progressively, allowing the browser to render content as it arrives rather than waiting for the entire page. This improves perceived performance — the user sees the page header and navigation immediately while slower data-dependent sections load below.

Choose your strategy based on the nature of your content: static for stable content, server-rendered for personalized or frequently changing content, client-rendered for dense interactivity. Many applications use a hybrid approach — static generation for marketing pages, server rendering for product pages with dynamic pricing, and client rendering for the shopping cart. The right answer is rarely one strategy for the entire application.

## 4. **Accessibility**

Accessibility is a requirement, not a feature. WCAG 2.1 and 2.2 define the standard through four principles: Perceivable, Operable, Understandable, and Robust.

Perceivable means content must be presentable in ways all users can perceive — text alternatives for images, captions for video, sufficient color contrast. Operable means all functionality must be available through multiple input methods, including keyboard. Understandable means the interface must behave predictably with clear language. Robust means content must work across a range of user agents and assistive technologies.

Pickering in *Inclusive Design Patterns* establishes a foundational rule: use semantic HTML first. A button element is focusable, activatable by keyboard, and announced correctly by screen readers without any additional attributes. A div with an onClick handler is none of these things.

Reaching for a div and adding role, tabindex, and keyboard event handlers to replicate what a button does for free is a common and costly mistake. ARIA is a repair tool for situations where semantic HTML is insufficient — it is not a replacement for proper element choices. The first rule of ARIA is: if you can use a native HTML element with the semantics you need, do so. Use nav for navigation, main for the primary content area, article for self-contained content, and aside for tangentially related content. These landmark elements give screen reader users a map of the page structure that allows them to jump directly to the section they need.

Keyboard navigation requires that every interactive element is reachable and operable using the keyboard alone. Users who cannot use a mouse rely on Tab, Shift+Tab, Enter, Space, and arrow keys to navigate. Custom components like dropdowns, date pickers, and carousels must implement the expected keyboard interactions defined in the WAI-ARIA Authoring Practices.

Focus management is critical for dynamic interfaces. When a modal dialog opens, focus must move into the dialog and be trapped there until the dialog closes. When it closes, focus must return to the element that triggered it. Failing to manage focus leaves keyboard users stranded in a part of the page they cannot see or interact with.

WCAG AA requires a minimum color contrast ratio of 4.5:1 for normal text and 3:1 for large text. Do not rely on color alone to convey information — a red border on an invalid field means nothing to a color-blind user without an accompanying text message.

Every image needs meaningful alt text that describes the content or function of the image, or an empty alt attribute if the image is purely decorative. Decorative images — icons next to text that already conveys the meaning, background patterns, visual flourishes — should have empty alt attributes so screen readers skip them entirely rather than announcing meaningless filenames.

Every form field needs a programmatically associated label — connected via the for attribute on the label element or by nesting the input inside the label. Test with actual screen readers — automated tools catch less than half of accessibility issues. Manual testing with NVDA on Windows, VoiceOver on macOS, or TalkBack on Android reveals problems that no automated scanner can detect, particularly around focus order, dynamic content updates, and custom widget interactions.

## 5. **Performance**

Google defines three Core Web Vitals that measure real-user experience. Largest Contentful Paint measures how quickly the main content becomes visible — the target is under 2.5 seconds. Interaction to Next Paint measures responsiveness to user input — the target is under 200 milliseconds. Cumulative Layout Shift measures visual stability — the target is under 0.1.

These metrics are measured on real devices with real network conditions, not on a developer's high-end laptop. A page that loads in one second on a wired connection may take eight seconds on a mobile network. Use field data from tools like Chrome User Experience Report to understand what your actual users experience. Lab tools like Lighthouse provide guidance during development, but they do not reflect real-world conditions where users are on slower devices, congested networks, and varied geographic locations.

Hogan in *Designing for Performance* frames performance as a feature with direct business impact. Every additional second of load time increases bounce rates. Users do not distinguish between a slow application and a broken one — both produce the same result: they leave.

Performance is not something you optimize at the end of a project — it is an architectural concern that must be designed in from the beginning. Decisions about rendering strategy, bundle composition, image handling, and font loading all have compounding effects on the metrics users experience. A performant architecture makes optimization straightforward. A non-performant architecture makes it a rewrite.

Code splitting is the practice of loading JavaScript on demand rather than shipping the entire application in a single bundle. Route-based splitting loads code for a page only when the user navigates to it. Component-based splitting defers heavy components — rich text editors, charting libraries, video players — until they are rendered.

Lazy loading extends this principle to images and other assets, loading elements below the fold only as the user scrolls toward them. The browser's native loading="lazy" attribute handles this for images without requiring JavaScript. However, do not lazy-load above-the-fold images — the LCP element should begin loading immediately, not wait for a scroll event or intersection observer.

Bundle size requires constant vigilance. Every npm dependency has a cost measured in kilobytes that users must download and the browser must parse and execute. Monitor bundle size in CI and question every new dependency. A utility library that saves ten lines of code but adds fifty kilobytes to the bundle is not a good tradeoff. Tools like bundlephobia show the size cost of a package before you install it. Tree-shaking helps — modern bundlers eliminate unused exports — but only when the library is authored with ES modules. Some packages are not tree-shakeable, meaning importing a single function pulls in the entire library.

Font loading strategy matters. Use font-display swap to prevent invisible text during font loading, and preload critical font files so the browser begins fetching them early. Image optimization means serving responsive images with srcset so the browser downloads the appropriately sized version for the user's viewport, using modern formats like WebP and AVIF that compress better than JPEG and PNG, and sizing images appropriately on the server rather than scaling down oversized assets in the browser.

## 6. **Styling Strategies**

Wathan and Schoger in *Refactoring UI* argue that good visual design comes from constrained choices. Define a spacing scale, a color palette, and a type scale upfront. Use values from these scales exclusively rather than picking arbitrary pixel values.

Constraints eliminate decision fatigue and produce consistent interfaces. When every margin, padding, and font size comes from a predefined scale, the design system enforces visual consistency without requiring manual review of every component. A spacing scale of 4, 8, 12, 16, 24, 32, 48, 64 pixels covers nearly every layout need.

The same principle applies to color. Define a palette with intentional shades — not just a single blue, but a range from light to dark with designated roles: primary action, secondary action, success, warning, error, and neutral backgrounds. Typography follows the same logic: a limited set of sizes with defined roles — heading, subheading, body, caption — prevents the visual noise of arbitrary font sizes scattered across the interface.

Meyer and Weyl in *CSS: The Definitive Guide* establish the importance of understanding the cascade, specificity, and inheritance — the three mechanisms that determine which styles apply to an element. Specificity conflicts are the source of most CSS frustration. When two rules target the same element, the one with higher specificity wins, regardless of source order. Understanding this hierarchy is essential for debugging unexpected styling behavior.

Utility-first CSS, as implemented by Tailwind, sidesteps many specificity issues by composing styles from small, single-purpose classes applied directly in markup. Each class does one thing — sets a margin, a color, a font size. This approach eliminates the naming problem, reduces dead CSS because unused utilities are purged at build time, and makes the relationship between markup and style explicit.

You see what an element looks like by reading its class list, not by tracing through stylesheet layers. This locality of reasoning is the core advantage of utility-first CSS.

CSS Modules provide component-scoped styles by generating unique class names at build time. They prevent style leakage between components without requiring a naming convention like BEM.

Design tokens formalize shared values — colors, spacing, typography, shadows — as a single source of truth that can be consumed by CSS, JavaScript, and design tools alike. When the brand color changes, you update one token, not forty stylesheets.

Responsive design follows a mobile-first approach: write styles for small screens first, then layer on complexity at defined breakpoints. Use breakpoints from a defined scale that aligns with your spacing and sizing system, not arbitrary pixel values chosen per-component. Mobile-first ensures that the smallest, most constrained experience works before adding enhancements for larger viewports.

The hierarchy of styling strategy in a component-based application is: start with semantic HTML structure, apply design tokens for consistent values, use your chosen styling approach — utility classes, CSS Modules, or scoped styles — for component-specific presentation, and test across viewport sizes at each defined breakpoint.

## 7. **Data Fetching and Caching**

Hallie and Osmani describe the Hooks pattern as the modern approach to data fetching in component-driven applications. Libraries like TanStack Query and SWR encapsulate the fetch lifecycle — loading, error, and success — into reusable hooks that handle caching, refetching, and deduplication automatically.

The React documentation stresses the importance of handling all three states explicitly. A component that only handles the success case will render nothing or crash when the network fails or the response is slow. Every data fetch must account for loading indicators, error messages, and empty states. The empty state — when the fetch succeeds but returns no data — is frequently overlooked. A blank screen with no explanation is confusing; an explicit "no results found" message with a suggested action is helpful.

Stale-while-revalidate is a caching strategy that shows cached data immediately while refetching fresh data in the background. This produces interfaces that feel instant on repeat visits while still staying current. The user sees content immediately, and if the data has changed, the UI updates seamlessly once the fresh response arrives.

Optimistic updates take this further — update the UI before the server confirms the mutation, then roll back if the server rejects it. This eliminates perceived latency for common actions like toggling a favorite, submitting a comment, or reordering a list. The key is implementing proper rollback logic so that server failures do not leave the UI in an inconsistent state.

Request deduplication ensures that multiple components requesting the same data produce a single network call. Without deduplication, mounting three components that each fetch the current user results in three identical requests hitting the server simultaneously. Data fetching libraries handle this automatically by keying requests and sharing the response across all subscribers.

Pagination must be handled deliberately. Choose between infinite scroll for content browsing — feeds, search results, media galleries — and paginated navigation for data that users need to reference by position or return to later. Infinite scroll is convenient for casual browsing but makes it impossible to reach the footer and difficult to share a specific position. Cursor-based pagination is more robust than offset-based pagination for data that changes frequently, as insertions and deletions do not cause items to shift between pages.

Error boundaries catch failed fetches and render fallback UI rather than crashing the entire component tree. A network failure in one widget should not bring down the entire page. Isolate data-dependent sections so that failures are contained and recoverable.

Consider retry logic for transient failures. A request that fails due to a brief network interruption may succeed on a second attempt. Data fetching libraries support configurable retry with exponential backoff. However, do not retry mutations blindly — retrying a payment submission without idempotency guarantees can result in duplicate charges.

## 8. **Forms, Validation, and User Input**

Krug in *Don't Make Me Think* establishes that forms should be as short as possible. Every additional field reduces completion rates. Labels must be clear and positioned consistently — above the field or to its left, never below.

Error messages must appear next to the field they describe, not in a summary at the top of the page that forces the user to map error messages back to fields by reading the entire form. Proximity between the error and the field is essential for quick comprehension.

Pickering in *Inclusive Design Patterns* insists that every form field must have a visible label. Placeholder text is not a label — it disappears when the user begins typing, removing the only cue about what the field expects.

Placeholders serve as examples or supplementary hints, not as substitutes for labels. A field with only a placeholder offers no guidance once the user starts typing and provides no persistent indication of what the field contains after it is filled.

Controlled inputs keep form state in component state, making the current value of every field accessible for validation, conditional logic, and submission. This approach gives you full control over the form at the cost of more explicit wiring.

Schema validation — using tools like Zod or Yup — defines the shape and constraints of form data in a single declaration that can be shared between client and server. This prevents the common bug where client-side validation allows data that the server rejects, or where validation rules drift apart over time as each side is updated independently.

Client-side validation is a convenience, not a security measure. It provides immediate feedback and reduces unnecessary server round trips, but the server must always validate independently. Users can bypass client-side validation trivially. The schema-sharing approach ensures both sides enforce the same rules without manual synchronization.

Validate on blur, not on every keystroke. Keystroke validation produces a jarring experience where error messages flash while the user is still typing — telling someone their email is invalid after they have typed three characters is unhelpful. Blur validation waits until the user moves to the next field, giving them a chance to finish their input before evaluating it.

Error messages must be specific and actionable. "Email is required" tells the user what to do. "Invalid input" tells them nothing. For search inputs, debounce the handler to avoid firing a request on every keystroke — wait until the user pauses typing before sending the query.

Announce validation errors to screen readers using aria-live regions so that users who cannot see the visual error message are informed immediately when validation fails. Without these announcements, screen reader users receive no feedback about errors, making the form unusable.

Multi-step forms should preserve progress — if the user navigates away accidentally, their input should not be lost. Use session storage or form state management to persist partially completed forms. Show clear progress indicators so users know where they are in the process and how much remains.

---

## Applying This Skill

When building frontend features, start with component architecture — identify the components, their responsibilities, and the data flow between them. Sketch the component tree before writing code. Choose the rendering strategy that matches the content type. Apply accessibility and performance principles from the first component, not as a retrofit — these concerns are dramatically harder to add after the fact than to build in from the start.

Use the framework-specific references for implementation idioms in React, Vue, Tailwind, or TypeScript.

Validate styling decisions against a defined design system rather than ad-hoc values. Handle data fetching with explicit loading, error, and success states, and choose a caching strategy that matches the data's freshness requirements.

Build forms with visible labels, schema validation, and accessible error announcements. Measure Core Web Vitals on real devices and treat performance regressions as bugs with the same severity as functional regressions.
