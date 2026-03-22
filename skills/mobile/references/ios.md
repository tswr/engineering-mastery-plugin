# iOS Mobile Reference

Idiomatic Swift and SwiftUI patterns for each mobile principle. These examples target Swift 5.9+ with structured concurrency, NavigationStack, SwiftData, and async/await.

---

## Application Lifecycle

```swift
import SwiftUI

@main
struct MyApp: App {
    @Environment(\.scenePhase) private var scenePhase

    var body: some Scene {
        WindowGroup { ContentView() }
            .onChange(of: scenePhase) { _, newPhase in
                switch newPhase {
                case .active:     refreshIfNeeded()    // returned to foreground
                case .inactive:   saveTransientState()  // transitioning
                case .background: persistCriticalState() // save before suspension
                @unknown default: break
                }
            }
    }
}

// @SceneStorage persists simple UI state across process death automatically
struct EditorView: View {
    @SceneStorage("editor.draft") private var draft: String = ""

    var body: some View { TextEditor(text: $draft) }
}
```

## Navigation and Routing

```swift
// Type-safe, deep-linkable routes via a Hashable enum
enum AppRoute: Hashable {
    case productDetail(id: String)
    case cart
    case orderConfirmation(orderId: String)
}

struct RootView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            ProductListView()
                .navigationDestination(for: AppRoute.self) { route in
                    switch route {
                    case .productDetail(let id):  ProductDetailView(productId: id)
                    case .cart:                    CartView()
                    case .orderConfirmation(let id): ConfirmationView(orderId: id)
                    }
                }
        }
        .onOpenURL { url in // handle deep links by parsing URL into a route
            if let route = parseDeepLink(url) { path.append(route) }
        }
    }
}
```

## Offline-First and Data Persistence

```swift
import SwiftData

// SwiftData model — local database is the source of truth
@Model class Product {
    @Attribute(.unique) var id: String
    var name: String
    var price: Decimal
    var lastSyncedAt: Date?

    init(id: String, name: String, price: Decimal) {
        self.id = id; self.name = name; self.price = price
    }
}

// Repository: UI reads from local store, network writes to local store
class ProductRepository {
    private let context: ModelContext
    private let api: APIClient

    // Return cached data immediately, sync from network in background
    func fetchProducts() async throws -> [Product] {
        let cached = try context.fetch(FetchDescriptor<Product>(sortBy: [SortDescriptor(\.name)]))
        Task { // background sync — UI already has cached data
            let remote = try await api.getProducts()
            for dto in remote {
                context.insert(Product(id: dto.id, name: dto.name, price: dto.price))
            }
            try context.save()
        }
        return cached
    }
}
```

## Networking and API Consumption

```swift
struct APIClient {
    private let session: URLSession
    private let baseURL: URL
    private let decoder = JSONDecoder()

    // async/await keeps networking off the main thread
    func getProducts() async throws -> [ProductDTO] {
        let url = baseURL.appendingPathComponent("products")
        let (data, response) = try await session.data(from: url)
        guard let http = response as? HTTPURLResponse,
              (200...299).contains(http.statusCode) else { throw APIError.badResponse }
        return try decoder.decode([ProductDTO].self, from: data)
    }

    // Retry with exponential backoff for transient failures
    func fetchWithRetry<T: Decodable>(url: URL, maxAttempts: Int = 3) async throws -> T {
        for attempt in 0..<maxAttempts {
            do {
                let (data, _) = try await session.data(from: url)
                return try decoder.decode(T.self, from: data)
            } catch where attempt < maxAttempts - 1 {
                let delay = UInt64(pow(2.0, Double(attempt))) * 1_000_000_000
                try await Task.sleep(nanoseconds: delay) // 1s, 2s, 4s
            }
        }
        throw APIError.maxRetriesExceeded
    }
}
```

## Performance and Responsiveness

```swift
struct ProductListView: View {
    @State private var products: [Product] = []

    var body: some View {
        ScrollView {
            LazyVStack(spacing: 12) { // only creates views for visible rows
                ForEach(products) { product in ProductRow(product: product) }
            }
        }
        .task { products = await loadProducts() } // cancels when view disappears
    }
}

struct ProductRow: View {
    let product: Product
    var body: some View {
        HStack {
            AsyncImage(url: product.thumbnailURL) { image in // loads off main thread
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: { ProgressView() }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))
            Text(product.name)
        }
    }
}
```

## Platform Conventions and Accessibility

```swift
struct CheckoutButton: View {
    let itemCount: Int
    let action: () -> Void

    var body: some View {
        Button(action: action) {
            Label("Checkout (\(itemCount))", systemImage: "cart").padding()
        }
        .frame(minWidth: 44, minHeight: 44) // 44pt minimum per Apple HIG
        .accessibilityLabel("Checkout with \(itemCount) items in your cart")
        .accessibilityHint("Double-tap to proceed to payment")
    }
}

struct PriceLabel: View {
    let price: Decimal
    var body: some View {
        Text(price, format: .currency(code: "USD"))
            .font(.body)              // scales with Dynamic Type automatically
            .foregroundStyle(.primary) // adapts to light/dark mode
    }
}
```
