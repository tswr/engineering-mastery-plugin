# TypeScript Frontend Reference

Frontend-specific TypeScript patterns for component props, state machines, API responses, runtime validation, and event handling. Covers utility types, discriminated unions, Zod schemas, and generics that make component APIs precise and self-documenting.

---

## Component Architecture

```tsx
// Component prop types: interfaces for public contracts
interface ButtonProps {
  variant: "primary" | "secondary" | "danger";  // union constrains valid values
  size?: "sm" | "md" | "lg";
  children: React.ReactNode;
  onClick: () => void;
}

// Generic component: works with any item type while preserving type safety
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>{items.map((item) => <li key={keyExtractor(item)}>{renderItem(item)}</li>)}</ul>
  );
}

// TypeScript infers T from the items array — user is typed as User
<List items={users} renderItem={(user) => <span>{user.name}</span>} keyExtractor={(u) => u.id} />;
```

## State Management

```ts
// Discriminated unions: each state carries only its relevant data
type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function renderState<T>(state: AsyncState<T>) {
  switch (state.status) {
    case "idle":    return null;
    case "loading": return <Spinner />;
    case "success": return <DataView data={state.data} />;  // narrowed to { data: T }
    case "error":   return <ErrorMessage error={state.error} />;
  }
}

// State machine for multi-step flows — impossible transitions are type errors
type CheckoutStep =
  | { step: "cart"; items: CartItem[] }
  | { step: "shipping"; items: CartItem[]; address: Address }
  | { step: "payment"; items: CartItem[]; address: Address; method: PaymentMethod }
  | { step: "confirmation"; orderId: string };
```

## Data Fetching and Caching

```ts
// Type-safe API responses — generic wrapper for paginated endpoints
interface ApiResponse<T> {
  data: T;
  meta: { page: number; totalPages: number; totalItems: number };
}

// Typed fetch wrapper — return type flows to caller
async function apiFetch<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, {
    ...init,
    headers: { "Content-Type": "application/json", ...init?.headers },
  });
  if (!res.ok) throw new ApiError(res.status, await res.text());
  return res.json() as Promise<T>;
}

// Result is fully typed: users is User[], meta.totalPages is number
const { data: users, meta } = await apiFetch<ApiResponse<User[]>>("/api/users");
```

## Forms, Validation, and User Input

```ts
import { z } from "zod";

// Zod schema: single source of truth shared between client and server
const ContactFormSchema = z.object({
  name: z.string().min(1, "Name is required").max(100, "Name too long"),
  email: z.string().email("Enter a valid email address"),
  message: z.string().min(10, "Message must be at least 10 characters"),
  category: z.enum(["support", "sales", "feedback"]),
});

// Infer TypeScript type from schema — no duplication
type ContactForm = z.infer<typeof ContactFormSchema>;

// Validate at the boundary — parse, don't validate
function handleSubmit(formData: unknown) {
  const result = ContactFormSchema.safeParse(formData);
  if (!result.success) {
    // flatten gives field-level errors: { name?: string[], email?: string[] }
    return { errors: result.error.flatten().fieldErrors };
  }
  return submitToApi(result.data);  // fully typed and validated
}

// Composed schemas reuse and extend other schemas
const AddressSchema = z.object({
  street: z.string().min(1, "Street is required"),
  city: z.string().min(1, "City is required"),
  zip: z.string().regex(/^\d{5}(-\d{4})?$/, "Invalid ZIP code"),
});

const OrderSchema = z.object({
  customer: ContactFormSchema.pick({ name: true, email: true }),  // reuse fields
  shippingAddress: AddressSchema,
  billingAddress: AddressSchema.optional(),  // same shape, optional
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive("Quantity must be at least 1"),
  })).min(1, "Order must have at least one item"),
});
```

```tsx
// Event handler typing: use React's event types, not DOM types
// React.ChangeEvent<HTMLInputElement> — target typed as HTMLInputElement
// React.FormEvent<HTMLFormElement>     — currentTarget typed as HTMLFormElement
// React.KeyboardEvent<HTMLInputElement> — key-specific handling
function Search({ onSearch }: { onSearch: (query: string) => void }) {
  return (
    <form onSubmit={(e: React.FormEvent<HTMLFormElement>) => {
      e.preventDefault();
      onSearch(new FormData(e.currentTarget).get("query") as string);
    }}>
      <input name="query" onChange={(e) => onSearch(e.target.value)} />
    </form>
  );
}

// Utility types for prop manipulation
interface FullUserProps {
  id: string; name: string; email: string;
  role: "admin" | "editor" | "viewer"; avatarUrl: string; createdAt: Date;
}

type UserCardProps = Pick<FullUserProps, "name" | "avatarUrl" | "role">;  // select needed fields
type UserEditForm = Omit<FullUserProps, "id" | "createdAt">;             // exclude internal fields
type UserPatch = Partial<Omit<FullUserProps, "id">>;                     // all optional for updates
type CompletedWizard = Required<{ name?: string; email?: string; plan?: string }>;
```
