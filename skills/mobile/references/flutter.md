# Flutter Mobile Reference

Idiomatic Dart and Flutter patterns for each mobile principle. These examples use GoRouter for navigation, sqflite for persistence, Dio for networking, and follow Flutter's widget composition model.

---

## Application Lifecycle

```dart
import 'package:flutter/widgets.dart';

// WidgetsBindingObserver tracks app lifecycle transitions
class AppLifecycleObserver extends WidgetsBindingObserver {
  final VoidCallback onResumed;
  final VoidCallback onPaused;

  AppLifecycleObserver({required this.onResumed, required this.onPaused});

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.resumed:  onResumed(); // foreground — refresh data
      case AppLifecycleState.paused:   onPaused();  // background — persist state
      case AppLifecycleState.inactive: break;        // transitioning
      case AppLifecycleState.detached: break;        // about to terminate
      case AppLifecycleState.hidden:   break;        // hidden but running
    }
  }
}

// Register in a StatefulWidget's initState / dispose
class _MyAppState extends State<MyApp> {
  late final AppLifecycleObserver _observer;

  @override
  void initState() {
    super.initState();
    _observer = AppLifecycleObserver(
      onResumed: () => refreshIfStale(),
      onPaused: () => saveDraftToStorage(),
    );
    WidgetsBinding.instance.addObserver(_observer);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(_observer);
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => const MaterialApp(home: HomeScreen());
}
```

## Navigation and Routing

```dart
import 'package:go_router/go_router.dart';

// GoRouter: declarative, deep-linkable routing
final GoRouter router = GoRouter(
  initialLocation: '/products',
  routes: [
    GoRoute(
      path: '/products',
      builder: (context, state) => const ProductListScreen(),
      routes: [
        GoRoute(
          path: ':productId', // path params extracted safely
          builder: (context, state) {
            final id = state.pathParameters['productId']!;
            return ProductDetailScreen(productId: id);
          },
        ),
      ],
    ),
    GoRoute(
      path: '/cart',
      builder: (context, state) => const CartScreen(),
    ),
    GoRoute(
      path: '/orders/:orderId/confirmation',
      builder: (context, state) {
        final orderId = state.pathParameters['orderId']!;
        return OrderConfirmationScreen(orderId: orderId);
      },
    ),
  ],
);

// Navigate with: context.go('/products/abc-123')
// Deep links resolve automatically through GoRouter
```

## Offline-First and Data Persistence

```dart
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';

class ProductDatabase {
  static Database? _db;

  static Future<Database> get database async {
    _db ??= await openDatabase(
      join(await getDatabasesPath(), 'products.db'),
      version: 1,
      onCreate: (db, version) => db.execute('''
        CREATE TABLE products (
          id TEXT PRIMARY KEY, name TEXT NOT NULL,
          price REAL NOT NULL, last_synced_at INTEGER
        )
      '''),
    );
    return _db!;
  }

  static Future<void> upsertProducts(List<Product> products) async {
    final db = await database;
    final batch = db.batch();
    for (final p in products) {
      batch.insert('products', p.toMap(), // insert or replace
          conflictAlgorithm: ConflictAlgorithm.replace);
    }
    await batch.commit(noResult: true);
  }

  static Future<List<Product>> getAll() async {
    final rows = await (await database).query('products', orderBy: 'name');
    return rows.map(Product.fromMap).toList();
  }
}

// Repository: UI reads from local DB, network writes to local DB
class ProductRepository {
  final ProductApi _api;
  ProductRepository(this._api);

  Future<List<Product>> getProducts() async {
    final cached = await ProductDatabase.getAll();
    _refreshFromNetwork(); // fire-and-forget background sync
    return cached;
  }

  Future<void> _refreshFromNetwork() async {
    try {
      final remote = await _api.fetchProducts();
      await ProductDatabase.upsertProducts(remote);
    } catch (_) {} // network failure expected — cached data remains
  }
}
```

## Networking and API Consumption

```dart
import 'package:dio/dio.dart';

class ApiClient {
  late final Dio _dio;

  ApiClient({required String baseUrl}) {
    _dio = Dio(BaseOptions(
      baseUrl: baseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 10),
    ));
    _dio.interceptors.add(_RetryInterceptor());
  }

  Future<List<ProductDTO>> getProducts() async {
    final response = await _dio.get<List<dynamic>>('/products');
    return response.data!
        .map((json) => ProductDTO.fromJson(json as Map<String, dynamic>))
        .toList();
  }
}

// Retry interceptor with exponential backoff
class _RetryInterceptor extends Interceptor {
  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    final attempt = (err.requestOptions.extra['attempt'] as int?) ?? 0;
    final isRetryable = err.type == DioExceptionType.connectionTimeout ||
        err.type == DioExceptionType.receiveTimeout ||
        (err.response?.statusCode ?? 0) >= 500;

    if (isRetryable && attempt < 2) {
      await Future<void>.delayed(Duration(seconds: 1 << attempt)); // 1s, 2s
      err.requestOptions.extra['attempt'] = attempt + 1;
      return handler.resolve(await Dio().fetch(err.requestOptions));
    }
    return handler.next(err);
  }
}
```

## Performance and Responsiveness

```dart
import 'package:flutter/material.dart';
import 'package:cached_network_image/cached_network_image.dart';

class ProductListScreen extends StatelessWidget {
  final List<Product> products;
  const ProductListScreen({super.key, required this.products});

  @override
  Widget build(BuildContext context) {
    return ListView.builder( // only builds visible items — list virtualization
      itemCount: products.length,
      itemExtent: 80.0, // known height avoids layout measurement
      itemBuilder: (context, index) => _ProductRow(product: products[index]),
    );
  }
}

class _ProductRow extends StatelessWidget {
  final Product product;
  const _ProductRow({required this.product});

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CachedNetworkImage( // handles disk/memory caching automatically
        imageUrl: product.thumbnailUrl, width: 56, height: 56, fit: BoxFit.cover,
        placeholder: (_, __) => const CircularProgressIndicator.adaptive(),
        errorWidget: (_, __, ___) => const Icon(Icons.broken_image),
      ),
      title: Text(product.name),
    );
  }
}
```

## Platform Conventions and Accessibility

```dart
class CheckoutButton extends StatelessWidget {
  final int itemCount;
  final VoidCallback onPressed;
  const CheckoutButton({super.key, required this.itemCount, required this.onPressed});

  @override
  Widget build(BuildContext context) {
    return Semantics(
      label: 'Checkout with $itemCount items in your cart', // TalkBack/VoiceOver
      hint: 'Double-tap to proceed to payment',
      button: true,
      child: ElevatedButton.icon(
        onPressed: onPressed,
        icon: const Icon(Icons.shopping_cart),
        label: Text('Checkout ($itemCount)'),
        style: ElevatedButton.styleFrom(
          minimumSize: const Size(48, 48), // Material minimum touch target
        ),
      ),
    );
  }
}

class PriceLabel extends StatelessWidget {
  final double price;
  const PriceLabel({super.key, required this.price});

  @override
  Widget build(BuildContext context) {
    return Text(
      '\$${price.toStringAsFixed(2)}',
      style: Theme.of(context).textTheme.bodyLarge, // scales with system font
    );
  }
}
```
