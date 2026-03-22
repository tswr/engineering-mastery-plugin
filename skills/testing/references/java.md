# Java Testing Reference

Idiomatic Java testing patterns using JUnit 5, Mockito, and AssertJ. Covers test structure, dependency injection for testability, mocking external dependencies, and techniques for testing legacy Java code.

---

## Test Structure (JUnit 5)

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.EnumSource;
import static org.assertj.core.api.Assertions.*;

class ShoppingCartTest {

    // Test naming: scenario_expectedOutcome
    @Test
    @DisplayName("Empty cart has zero total")
    void emptyCartHasZeroTotal() {
        // Arrange
        final var cart = new ShoppingCart();

        // Act
        final var total = cart.total();

        // Assert
        assertThat(total).isZero();
    }

    @Test
    void addingItemIncreasesTotal() {
        var cart = new ShoppingCart();
        final var item = new Item("Widget", 1500);

        cart.add(item);

        assertThat(cart.total()).isEqualTo(1500);
        assertThat(cart.itemCount()).isEqualTo(1);
    }

    // Output-based: pure function
    @Test
    void goldTierGets15PercentDiscount() {
        final var result = calculateDiscount(10000, Tier.GOLD);
        assertThat(result).isEqualTo(8500);
    }
}

// Parameterized tests for multiple scenarios
class DiscountTest {

    @ParameterizedTest(name = "{0} tier gets {2} on price {1}")
    @CsvSource({
        "BRONZE, 10000, 9500",
        "SILVER, 10000, 9000",
        "GOLD,   10000, 8500"
    })
    void discountByTier(Tier tier, int price, int expected) {
        assertThat(calculateDiscount(price, tier)).isEqualTo(expected);
    }
}

// Test fixture with setup/teardown
class DatabaseOrderTest {

    private FakeDatabase db;
    private OrderService service;

    @BeforeEach
    void setUp() {
        db = new FakeDatabase();
        service = new OrderService(db, new StubPaymentGateway());
    }

    @Test
    void placedOrderIsPersisted() {
        final var order = new Order(ProductId.of("abc"), 2, 5000);

        final var result = service.place(order);

        assertThat(result.isSuccess()).isTrue();
        assertThat(db.contains(result.id())).isTrue();
    }
}
```

## Dependency Injection for Testability

```java
// Interface for external dependency
public interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);
}

// Production implementation
public final class StripeGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(PaymentRequest request) {
        // real Stripe API call
    }
}

// Service depends on interface — testable by construction
public class OrderService {
    private final PaymentGateway gateway;
    private final OrderRepository repo;

    public OrderService(PaymentGateway gateway, OrderRepository repo) {
        this.gateway = Objects.requireNonNull(gateway);
        this.repo = Objects.requireNonNull(repo);
    }

    public OrderResult place(Order order) {
        final var payment = gateway.charge(order.paymentRequest());
        if (!payment.isSuccess()) {
            return OrderResult.failed(payment.error());
        }
        return repo.save(order);
    }
}

// Fake for managed dependency — real behavior, in memory
class FakeOrderRepository implements OrderRepository {
    private final Map<OrderId, Order> store = new HashMap<>();

    @Override
    public OrderResult save(Order order) {
        final var id = OrderId.generate();
        store.put(id, order);
        return OrderResult.success(id);
    }

    public boolean contains(OrderId id) {
        return store.containsKey(id);
    }

    public Optional<Order> find(OrderId id) {
        return Optional.ofNullable(store.get(id));
    }
}
```

## Mockito (For External Dependencies Only)

```java
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.*;

class OrderServiceTest {

    // Mock: external system (unmanaged dependency)
    private final PaymentGateway gateway = mock(PaymentGateway.class);
    // Fake: internal collaborator (managed dependency) — not mocked
    private final FakeOrderRepository repo = new FakeOrderRepository();
    private final OrderService service = new OrderService(gateway, repo);

    @Test
    void successfulOrderChargesGateway() {
        // Stub the external dependency
        when(gateway.charge(any()))
            .thenReturn(PaymentResult.success("txn_123"));

        final var order = makeTestOrder(5000);

        final var result = service.place(order);

        // Assert observable outcome
        assertThat(result.isSuccess()).isTrue();
        assertThat(repo.contains(result.id())).isTrue();
        // Verify outgoing interaction with external system
        verify(gateway).charge(any());
    }

    @Test
    void failedPaymentReturnsError() {
        when(gateway.charge(any()))
            .thenReturn(PaymentResult.failed("declined"));

        final var result = service.place(makeTestOrder(5000));

        assertThat(result.isSuccess()).isFalse();
        assertThat(result.error()).isEqualTo("declined");
    }

    @Test
    void orderConfirmationSendsEmail() {
        // Mock another external system
        final var emailService = mock(EmailService.class);
        final var service = new OrderService(gateway, repo, emailService);
        when(gateway.charge(any()))
            .thenReturn(PaymentResult.success("txn_123"));

        service.place(makeTestOrder(5000));

        verify(emailService).send(
            any(EmailAddress.class),
            eq("Order Confirmed"),
            anyString()
        );
    }

    // WRONG: don't mock internal collaborators
    // ❌ var mockValidator = mock(OrderValidator.class);
    // ❌ when(mockValidator.validate(any())).thenReturn(true);
    // This couples the test to the internal design

    private static Order makeTestOrder(long total) {
        return new Order(ProductId.of("test_product"), 1, total);
    }
}
```

## Breaking Dependencies in Legacy Java

```java
// TECHNIQUE 1: Extract and Override
// Before: static call, untestable
public class ReportGenerator {
    public Report generate() {
        ResultSet data = DatabaseClient.query("SELECT ...");
        return process(data);
    }
}

// After: extract to protected method, override in test
public class ReportGenerator {
    public Report generate() {
        ResultSet data = fetchData();
        return process(data);
    }

    protected ResultSet fetchData() {
        return DatabaseClient.query("SELECT ...");
    }
}

// Test subclass
class TestableReportGenerator extends ReportGenerator {
    private final ResultSet testData;

    TestableReportGenerator(ResultSet testData) {
        this.testData = testData;
    }

    @Override
    protected ResultSet fetchData() {
        return testData;
    }
}

@Test
void generatesReportFromData() {
    var gen = new TestableReportGenerator(makeTestData());
    var report = gen.generate();
    assertThat(report.rowCount()).isEqualTo(5);
}


// TECHNIQUE 2: Parameterize Constructor
// Add dependency parameter with backward-compatible default
public class ReportGenerator {
    private final DataSource source;

    public ReportGenerator() {
        this(new DatabaseSource());  // production default
    }

    // Test-friendly constructor
    public ReportGenerator(DataSource source) {
        this.source = Objects.requireNonNull(source);
    }

    public Report generate() {
        var data = source.fetch();
        return process(data);
    }
}

@Test
void processesDataCorrectly() {
    var gen = new ReportGenerator(new FakeDataSource(testData));
    var report = gen.generate();
    assertThat(report.rowCount()).isEqualTo(5);
}


// TECHNIQUE 3: Characterization tests
@Test
void characterizationLegacyPricing() {
    // Document current behavior — not asserting correctness
    assertThat(calculateLegacyPrice("SKU001", 1)).isCloseTo(29.99, within(0.01));
    assertThat(calculateLegacyPrice("SKU001", 10)).isCloseTo(269.91, within(0.01));
    assertThat(calculateLegacyPrice("SKU001", 100)).isCloseTo(2399.20, within(0.01));
    // Odd edge case — locking it in
    assertThat(calculateLegacyPrice("PROMO_X", 1)).isCloseTo(0.0, within(0.01));
}
```

## Test Organization

```
project/
├── src/
│   └── main/java/com/example/
│       ├── orders/
│       │   ├── OrderService.java
│       │   └── Order.java
│       └── payments/
│           └── PaymentGateway.java
├── src/
│   └── test/java/com/example/
│       ├── orders/
│       │   ├── OrderServiceTest.java        # unit tests
│       │   └── OrderServiceIntegrationTest.java
│       ├── payments/
│       │   └── PaymentGatewayTest.java
│       ├── fakes/
│       │   ├── FakeOrderRepository.java     # fake implementations
│       │   └── FakeDatabase.java
│       ├── characterization/
│       │   └── LegacyPricingTest.java
│       └── TestHelpers.java                 # shared test utilities
└── build.gradle

# Gradle: separate test tasks
tasks.named('test') {
    useJUnitPlatform()
    // Unit tests: fast, run always
    include '**/unit/**'
}

tasks.register('integrationTest', Test) {
    useJUnitPlatform()
    include '**/integration/**'
}
```

## Useful Patterns

```java
// AssertJ: fluent, readable assertions
assertThat(result.items())
    .hasSize(3)
    .extracting(Item::name)
    .containsExactly("A", "B", "C");

assertThat(result.total())
    .isGreaterThan(0)
    .isLessThan(100_000);

// Asserting exceptions
assertThatThrownBy(() -> new Order(ProductId.of("abc"), -1, 1000))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("quantity must be positive");

// Test data builders
class TestOrderBuilder {
    private String productId = "default_product";
    private int quantity = 1;
    private long price = 1000;

    static TestOrderBuilder anOrder() { return new TestOrderBuilder(); }

    TestOrderBuilder withQuantity(int q) { this.quantity = q; return this; }
    TestOrderBuilder withPrice(long p) { this.price = p; return this; }

    Order build() {
        return new Order(ProductId.of(productId), quantity, price);
    }
}

// Usage:
var order = anOrder().withQuantity(10).withPrice(500).build();


// Temporary files
@TempDir
Path tempDir;

@Test
void exportWritesFile() {
    final var output = tempDir.resolve("report.csv");
    exporter.export(report, output);
    assertThat(output).content().startsWith("id,name,total");
}


// Nested tests for grouping related scenarios
@Nested
class WhenCartIsEmpty {
    private final ShoppingCart cart = new ShoppingCart();

    @Test
    void totalIsZero() { assertThat(cart.total()).isZero(); }

    @Test
    void itemCountIsZero() { assertThat(cart.itemCount()).isZero(); }
}

@Nested
class WhenCartHasItems {
    private final ShoppingCart cart = new ShoppingCart();

    @BeforeEach
    void setUp() { cart.add(new Item("Widget", 1500)); }

    @Test
    void totalReflectsItems() { assertThat(cart.total()).isEqualTo(1500); }
}
```

## Property-Based Testing (jqwik)

```java
import net.jqwik.api.*;

class SortingProperties {
    // Property: sorting is idempotent
    @Property
    void sortingIsIdempotent(@ForAll List<Integer> list) {
        List<Integer> once = list.stream().sorted().collect(Collectors.toList());
        List<Integer> twice = once.stream().sorted().collect(Collectors.toList());
        Assertions.assertEquals(once, twice);
    }

    // Property: serialization round-trip
    @Property
    void jsonRoundTrip(@ForAll("users") User user) {
        String json = objectMapper.writeValueAsString(user);
        User back = objectMapper.readValue(json, User.class);
        Assertions.assertEquals(user, back);
    }
}
```
