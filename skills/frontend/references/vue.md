# Vue Frontend Reference

Idiomatic Vue 3 Composition API and Nuxt 3 patterns for each frontend principle. All examples use `<script setup>` syntax. Covers reactivity primitives, component design, Pinia state management, and data fetching.

---

## Component Architecture

```vue
<!-- ProductCard.vue — single responsibility: display one product -->
<script setup lang="ts">
// defineProps: pass only fields used, not entire objects
defineProps<{ name: string; price: number; imageUrl: string }>();
// defineEmits: typed outgoing events
const emit = defineEmits<{ addToCart: [] }>();
</script>

<template>
  <article>
    <img :src="imageUrl" :alt="name" />
    <h3>{{ name }}</h3>
    <p>${{ price.toFixed(2) }}</p>
    <button @click="emit('addToCart')">Add to Cart</button>
  </article>
</template>
```

```ts
// Lazy-load heavy components to reduce initial bundle
import { defineAsyncComponent } from "vue";
const RichTextEditor = defineAsyncComponent({
  loader: () => import("./RichTextEditor.vue"),
  loadingComponent: EditorSkeleton,
  delay: 200,  // avoid flash for fast loads
});
```

## State Management

```vue
<!-- SearchableList.vue — state colocated where it is used -->
<script setup lang="ts">
import { ref, computed } from "vue";

const props = defineProps<{ items: { id: string; name: string }[] }>();
const query = ref("");  // ref for primitive reactive state

// computed derives from existing state — never store filtered list separately
const filtered = computed(() =>
  props.items.filter((i) => i.name.toLowerCase().includes(query.value.toLowerCase())),
);
</script>

<template>
  <input v-model="query" aria-label="Search items" />
  <p v-if="filtered.length === 0">No items match your search.</p>
  <ul><li v-for="item in filtered" :key="item.id">{{ item.name }}</li></ul>
</template>
```

```ts
// stores/auth.ts — Pinia for truly app-wide concerns only
import { defineStore } from "pinia";
import { ref, computed } from "vue";

export const useAuthStore = defineStore("auth", () => {
  const user = ref<User | null>(null);
  const isAuthenticated = computed(() => user.value !== null);  // derive, don't duplicate

  async function login(creds: Credentials) { user.value = (await api.login(creds)).user; }
  function logout() { user.value = null; }

  return { user, isAuthenticated, login, logout };
});

// Provide/inject for cross-cutting concerns — typed injection key prevents misuse
import { inject, type InjectionKey } from "vue";
interface ThemeContext { theme: "light" | "dark"; toggle: () => void; }
const THEME_KEY: InjectionKey<ThemeContext> = Symbol("theme");
// Provider: provide(THEME_KEY, { theme, toggle });
// Consumer: const ctx = inject(THEME_KEY);
```

## Rendering Strategies

```vue
<!-- Nuxt 3: pages/products/[id].vue — server-rendered by default -->
<script setup lang="ts">
// useAsyncData runs on server during SSR, caches for client hydration
const { data: product, error } = await useAsyncData(
  `product-${useRoute().params.id}`,
  () => $fetch(`/api/products/${useRoute().params.id}`),
);
if (error.value) throw createError({ statusCode: 500, message: "Load failed" });
if (!product.value) throw createError({ statusCode: 404, message: "Not found" });
</script>

<template>
  <article><h1>{{ product.name }}</h1></article>
</template>
```

```ts
// nuxt.config.ts — hybrid rendering per route
export default defineNuxtConfig({
  routeRules: {
    "/":            { prerender: true },   // static at build time
    "/blog/**":     { isr: 3600 },          // regenerate hourly
    "/dashboard":   { ssr: false },         // client-only
    "/products/**": { swr: 600 },           // stale-while-revalidate
  },
});
```

## Data Fetching and Caching

```vue
<script setup lang="ts">
// useFetch handles loading, error, caching, and SSR automatically
const { data: projects, status, error, refresh } = await useFetch("/api/projects", {
  transform: (res: ApiResponse<Project[]>) => res.data,
});
watch(() => route.query.page, () => refresh());
</script>

<template>
  <!-- Handle all states: loading, error, empty, success -->
  <ProjectListSkeleton v-if="status === 'pending'" />
  <ErrorMessage v-else-if="error" :error="error" />
  <EmptyState v-else-if="projects?.length === 0" message="No projects yet." />
  <ul v-else><ProjectRow v-for="p in projects" :key="p.id" :project="p" /></ul>
</template>
```

## Forms, Validation, and User Input

```vue
<script setup lang="ts">
const email = ref("");
const emailError = ref("");

// Validate on blur — not on every keystroke
function validateEmail() {
  if (!email.value) emailError.value = "Email is required";
  else if (!email.value.includes("@")) emailError.value = "Enter a valid email";
  else emailError.value = "";
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <label for="email">Email address</label>  <!-- visible label, never placeholder-only -->
    <input id="email" v-model="email" type="email" @blur="validateEmail"
      :aria-invalid="!!emailError" aria-describedby="email-error" />
    <p v-if="emailError" id="email-error" role="alert">{{ emailError }}</p>
  </form>
</template>
```
