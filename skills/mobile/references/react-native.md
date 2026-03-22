# React Native Mobile Reference

Idiomatic TypeScript and React Native patterns for each mobile principle. These examples use React Navigation, modern hooks-based patterns, and platform-aware APIs. Patterns apply to both Expo and bare React Native projects.

---

## Application Lifecycle

```typescript
import { useEffect, useRef } from "react";
import { AppState, AppStateStatus } from "react-native";

// AppState tracks whether the app is active, background, or inactive
function useAppLifecycle(onForeground: () => void, onBackground: () => void) {
  const appState = useRef<AppStateStatus>(AppState.currentState);

  useEffect(() => {
    const sub = AppState.addEventListener("change", (next) => {
      if (appState.current.match(/inactive|background/) && next === "active") {
        onForeground(); // app returned to foreground — refresh stale data
      } else if (appState.current === "active" && next.match(/inactive|background/)) {
        onBackground(); // entering background — persist unsaved state
      }
      appState.current = next;
    });
    return () => sub.remove();
  }, [onForeground, onBackground]);
}
```

## Navigation and Routing

```typescript
import { NavigationContainer, LinkingOptions } from "@react-navigation/native";
import {
  createNativeStackNavigator,
  NativeStackScreenProps,
} from "@react-navigation/native-stack";

// All routes and params in a single type map
type RootStackParamList = {
  ProductList: undefined;
  ProductDetail: { productId: string };
  Cart: undefined;
  OrderConfirmation: { orderId: string };
};

const Stack = createNativeStackNavigator<RootStackParamList>();

// Deep linking — every screen is reachable via URL
const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ["myapp://", "https://myapp.example.com"],
  config: {
    screens: {
      ProductList: "products",
      ProductDetail: "products/:productId",
      Cart: "cart",
      OrderConfirmation: "orders/:orderId/confirmation",
    },
  },
};

function AppNavigator() {
  return (
    <NavigationContainer linking={linking}>
      <Stack.Navigator initialRouteName="ProductList">
        <Stack.Screen name="ProductList" component={ProductListScreen} />
        <Stack.Screen name="ProductDetail" component={ProductDetailScreen} />
        <Stack.Screen name="Cart" component={CartScreen} />
        <Stack.Screen name="OrderConfirmation" component={ConfirmationScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

// Type-safe params — productId is checked at compile time
type DetailProps = NativeStackScreenProps<RootStackParamList, "ProductDetail">;
function ProductDetailScreen({ route }: DetailProps) {
  const { productId } = route.params;
  return <ProductDetail id={productId} />;
}
```

## Offline-First and Data Persistence

```typescript
import AsyncStorage from "@react-native-async-storage/async-storage";
import { useState, useEffect } from "react";

// Simple cache with AsyncStorage
async function cacheProducts(products: Product[]): Promise<void> {
  await AsyncStorage.setItem(
    "products",
    JSON.stringify({ data: products, cachedAt: Date.now() })
  );
}

async function getCachedProducts(): Promise<Product[] | null> {
  const raw = await AsyncStorage.getItem("products");
  if (!raw) return null;
  return (JSON.parse(raw) as { data: Product[] }).data;
}

// Hook: cached data immediately, network refresh in background
function useProducts() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isRefreshing, setIsRefreshing] = useState(false);

  useEffect(() => {
    let cancelled = false;
    async function load() {
      const cached = await getCachedProducts();
      if (cached && !cancelled) setProducts(cached); // show cached instantly

      setIsRefreshing(true);
      try {
        const fresh = await fetchProducts();
        if (!cancelled) { setProducts(fresh); await cacheProducts(fresh); }
      } finally {
        if (!cancelled) setIsRefreshing(false);
      }
    }
    load();
    return () => { cancelled = true; };
  }, []);

  return { products, isRefreshing };
}
```

## Networking and API Consumption

```typescript
// Fetch wrapper with timeout and typed responses
async function apiFetch<T>(path: string, options: RequestInit = {}): Promise<T> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 10_000); // 10s timeout

  try {
    const response = await fetch(`https://api.example.com${path}`, {
      ...options,
      signal: controller.signal,
      headers: { "Content-Type": "application/json", ...options.headers },
    });
    if (!response.ok) throw new ApiError(response.status, await response.text());
    return (await response.json()) as T;
  } finally {
    clearTimeout(timeoutId);
  }
}

// Retry with exponential backoff
async function fetchWithRetry<T>(path: string, maxAttempts = 3): Promise<T> {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await apiFetch<T>(path);
    } catch (error) {
      if (attempt === maxAttempts - 1) throw error;
      await new Promise((r) => setTimeout(r, Math.pow(2, attempt) * 1000));
    }
  }
  throw new Error("Unreachable");
}
```

## Performance and Responsiveness

```typescript
import { FlatList } from "react-native";
import { useCallback } from "react";

const ITEM_HEIGHT = 80;

function ProductListScreen() {
  const { products, isRefreshing } = useProducts();

  // useCallback prevents re-creating the function, avoiding FlatList re-renders
  const renderItem = useCallback(
    ({ item }: { item: Product }) => <ProductRow product={item} />, []
  );
  const keyExtractor = useCallback((item: Product) => item.id, []);

  return (
    <FlatList
      data={products}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      initialNumToRender={10}  // items rendered before first paint
      windowSize={5}           // screens-worth of items kept mounted
      getItemLayout={(_, index) => ({  // skip measurement when height is known
        length: ITEM_HEIGHT, offset: ITEM_HEIGHT * index, index,
      })}
      onRefresh={() => refreshProducts()}
      refreshing={isRefreshing}
    />
  );
}
```

## Platform Conventions and Accessibility

```typescript
import { TouchableOpacity, Text, StyleSheet, useColorScheme } from "react-native";

function CheckoutButton({ itemCount, onPress }: { itemCount: number; onPress: () => void }) {
  return (
    <TouchableOpacity
      onPress={onPress}
      style={styles.button}
      accessibilityRole="button"
      accessibilityLabel={`Checkout with ${itemCount} items in your cart`}
      accessibilityHint="Double-tap to proceed to payment"
    >
      <Text style={styles.buttonText}>Checkout ({itemCount})</Text>
    </TouchableOpacity>
  );
}

function PriceLabel({ price }: { price: number }) {
  const colorScheme = useColorScheme();
  const textColor = colorScheme === "dark" ? "#FFFFFF" : "#000000";
  return (
    <Text
      style={[styles.price, { color: textColor }]}
      maxFontSizeMultiplier={2.0} // respect system font scaling
    >
      ${price.toFixed(2)}
    </Text>
  );
}

const styles = StyleSheet.create({
  button: { minWidth: 48, minHeight: 48, padding: 12, alignItems: "center" },
  buttonText: { fontSize: 16, fontWeight: "600" },
  price: { fontSize: 16 },
});
```
