# React Frontend Reference

Idiomatic React 18+ and Next.js patterns for each frontend principle. Covers hooks, Server Components, data fetching with TanStack Query, and error boundaries. All examples use functional components.

---

## Component Architecture

```tsx
import { lazy, memo, Suspense } from "react";

// Single responsibility: ProductCard renders — no fetching, no cart logic
interface ProductCardProps {
  name: string;
  price: number;        // pass only fields used, not the entire product object
  imageUrl: string;
  onAddToCart: () => void;
}

function ProductCard({ name, price, imageUrl, onAddToCart }: ProductCardProps) {
  return (
    <article>
      <img src={imageUrl} alt={name} />
      <h3>{name}</h3>
      <p>${price.toFixed(2)}</p>
      <button onClick={onAddToCart}>Add to Cart</button>
    </article>
  );
}

const MemoizedProductCard = memo(ProductCard);  // only after measuring a problem

// Lazy-load heavy components to reduce initial bundle size
const RichTextEditor = lazy(() => import("./RichTextEditor"));
// Suspense provides fallback UI while the lazy component loads
function PostEditor() {
  return <Suspense fallback={<div>Loading...</div>}><RichTextEditor /></Suspense>;
}
```

## State Management

```tsx
import { useState, useCallback, useMemo, createContext, useContext } from "react";

function SearchableList({ items }: { items: Item[] }) {
  const [query, setQuery] = useState("");  // state colocated where it is used

  // Derive filtered list — do not store separately as duplicate state
  const filtered = useMemo(
    () => items.filter((i) => i.name.toLowerCase().includes(query.toLowerCase())),
    [items, query],
  );

  // useCallback stabilizes reference so memoized children skip re-renders
  const handleSearch = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
  }, []);

  return (
    <>
      <input value={query} onChange={handleSearch} aria-label="Search items" />
      {filtered.length === 0 && <p>No items match your search.</p>}
      {filtered.map((item) => <ItemRow key={item.id} item={item} />)}
    </>
  );
}

// Context for cross-cutting concerns only — not for all state
const ThemeContext = createContext<{ theme: "light" | "dark"; toggle: () => void } | null>(null);

function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}
```

## Rendering Strategies

```tsx
// Next.js App Router: Server Component (default) — no "use client", zero client JS
async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.products.findUnique({ where: { id: params.id } });
  if (!product) return notFound();
  return (
    <article>
      <h1>{product.name}</h1>
      {/* Client Component — only this ships JS to browser */}
      <AddToCartButton productId={product.id} />
    </article>
  );
}

// Streaming: each Suspense boundary streams independently
async function DashboardPage() {
  return (
    <div>
      <Suspense fallback={<StatsSkeleton />}><StatsPanel /></Suspense>
      <Suspense fallback={<ActivitySkeleton />}><RecentActivity /></Suspense>
    </div>
  );
}
```

## Data Fetching and Caching

```tsx
"use client";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { ErrorBoundary } from "react-error-boundary";

function ProjectList() {
  // TanStack Query handles loading, error, caching, and request deduplication
  const { data: projects, isLoading, error } = useQuery({
    queryKey: ["projects"],
    queryFn: () => fetch("/api/projects").then((r) => r.json()),
    staleTime: 60_000,  // fresh for 60s — no refetch during this window
  });

  if (isLoading) return <ProjectListSkeleton />;
  if (error) return <ErrorMessage error={error} />;
  if (projects.length === 0) return <EmptyState message="No projects yet." />;

  return <ul>{projects.map((p: Project) => <ProjectRow key={p.id} project={p} />)}</ul>;
}

function AddProjectButton() {
  const qc = useQueryClient();
  const mutation = useMutation({
    mutationFn: (p: NewProject) =>
      fetch("/api/projects", { method: "POST", body: JSON.stringify(p) }),
    // Optimistic update: show immediately, rollback on server failure
    onMutate: async (newProject) => {
      await qc.cancelQueries({ queryKey: ["projects"] });
      const previous = qc.getQueryData<Project[]>(["projects"]);
      qc.setQueryData<Project[]>(["projects"], (old) => [
        ...(old ?? []), { ...newProject, id: "temp-id" } as Project,
      ]);
      return { previous };
    },
    onError: (_err, _new, ctx) => {
      if (ctx?.previous) qc.setQueryData(["projects"], ctx.previous);
    },
    onSettled: () => qc.invalidateQueries({ queryKey: ["projects"] }),
  });
  return <button onClick={() => mutation.mutate({ name: "New Project" })}>Add</button>;
}

// Error boundary: isolate failures so one broken widget doesn't crash the page
function App() {
  return (
    <ErrorBoundary fallback={<p>Something went wrong. Please refresh.</p>}>
      <ProjectList />
    </ErrorBoundary>
  );
}
```
