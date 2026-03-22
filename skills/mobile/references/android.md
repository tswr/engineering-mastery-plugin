# Android Mobile Reference

Idiomatic Kotlin and Jetpack Compose patterns for each mobile principle. These examples follow Google's Guide to App Architecture and target modern Android with Compose, coroutines, Room, and type-safe Navigation.

---

## Application Lifecycle

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

// ViewModel survives configuration changes (rotation, dark mode toggle)
// SavedStateHandle survives process death — use for user-entered data
class ProductListViewModel(
    private val repository: ProductRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProductListState())
    val uiState: StateFlow<ProductListState> = _uiState.asStateFlow()

    // Persists across process death via SavedStateHandle
    val searchQuery = savedStateHandle.getStateFlow("searchQuery", "")

    init { loadProducts() }

    fun onSearchQueryChanged(query: String) {
        savedStateHandle["searchQuery"] = query
    }

    private fun loadProducts() {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true)
            val products = repository.getProducts()
            _uiState.value = ProductListState(products = products)
        }
    }
}

data class ProductListState(
    val products: List<Product> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)
```

## Navigation and Routing

```kotlin
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.navigation.toRoute
import kotlinx.serialization.Serializable

// Type-safe routes using kotlinx.serialization (Navigation 2.8+)
@Serializable data object ProductList
@Serializable data class ProductDetail(val productId: String)
@Serializable data object Cart

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    NavHost(navController = navController, startDestination = ProductList) {
        composable<ProductList> {
            ProductListScreen(
                onProductClick = { id -> navController.navigate(ProductDetail(id)) }
            )
        }
        composable<ProductDetail> { backStackEntry ->
            val route = backStackEntry.toRoute<ProductDetail>()
            ProductDetailScreen(productId = route.productId)
        }
        composable<Cart> { CartScreen() }
    }
}
```

## Offline-First and Data Persistence

```kotlin
import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Entity(tableName = "products")
data class ProductEntity(
    @PrimaryKey val id: String,
    val name: String,
    val price: Double,
    val lastSyncedAt: Long? = null
)

@Dao
interface ProductDao {
    @Query("SELECT * FROM products ORDER BY name")
    fun observeAll(): Flow<List<ProductEntity>> // emits when data changes

    @Upsert
    suspend fun upsertAll(products: List<ProductEntity>)
}

// Repository: UI reads from Room, network writes to Room
class ProductRepository(private val dao: ProductDao, private val api: ProductApi) {
    // Flow ensures UI always reflects latest local data
    fun observeProducts(): Flow<List<ProductEntity>> = dao.observeAll()

    // Fetch from network, write to local DB — UI updates automatically
    suspend fun refreshProducts() {
        val remote = api.getProducts()
        dao.upsertAll(remote.map { it.toEntity() })
    }
}
```

## Networking and API Consumption

```kotlin
import retrofit2.http.GET
import retrofit2.http.Path

// Retrofit: suspend functions integrate directly with coroutines
interface ProductApi {
    @GET("products")
    suspend fun getProducts(): List<ProductDTO>

    @GET("products/{id}")
    suspend fun getProduct(@Path("id") id: String): ProductDTO
}

// Retry with exponential backoff for transient failures
suspend fun <T> retryWithBackoff(
    maxAttempts: Int = 3,
    initialDelayMs: Long = 1000,
    block: suspend () -> T
): T {
    var delay = initialDelayMs
    repeat(maxAttempts - 1) {
        try { return block() } catch (e: Exception) {
            if (e is retrofit2.HttpException && e.code() in 400..499) throw e
            kotlinx.coroutines.delay(delay)
            delay *= 2 // 1s, 2s, 4s
        }
    }
    return block() // final attempt — let exception propagate
}
```

## Performance and Responsiveness

```kotlin
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import coil3.compose.AsyncImage

@Composable
fun ProductListScreen(viewModel: ProductListViewModel) {
    val uiState by viewModel.uiState.collectAsState()

    LazyColumn { // only composes visible items — Compose list virtualization
        items(
            items = uiState.products,
            key = { it.id } // stable keys help Compose track items efficiently
        ) { product -> ProductRow(product) }
    }
}

@Composable
fun ProductRow(product: Product) {
    Row(modifier = Modifier.padding(16.dp)) {
        AsyncImage( // Coil handles async loading, caching, and downsampling
            model = product.thumbnailUrl,
            contentDescription = product.name,
            modifier = Modifier.size(60.dp),
            contentScale = ContentScale.Crop
        )
        Spacer(modifier = Modifier.width(12.dp))
        Text(product.name, style = MaterialTheme.typography.bodyLarge)
    }
}
```

## Platform Conventions and Accessibility

```kotlin
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.semantics

@Composable
fun CheckoutButton(itemCount: Int, onClick: () -> Unit) {
    Button(
        onClick = onClick,
        modifier = Modifier
            .defaultMinSize(minWidth = 48.dp, minHeight = 48.dp) // Material minimum
            .semantics { contentDescription = "Checkout with $itemCount items" }
    ) {
        Icon(Icons.Default.ShoppingCart, contentDescription = null)
        Spacer(Modifier.width(8.dp))
        Text("Checkout ($itemCount)")
    }
}

@Composable
fun PriceLabel(price: Double) {
    Text(
        text = NumberFormat.getCurrencyInstance().format(price),
        style = MaterialTheme.typography.bodyLarge, // scales with system font size
        color = MaterialTheme.colorScheme.onSurface  // adapts to dark mode
    )
}
```
