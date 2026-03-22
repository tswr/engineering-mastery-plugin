# Rust Testing Reference

Idiomatic Rust testing patterns using the built-in test framework and mockall. Rust's type system and ownership model eliminate many categories of test fragility, so the focus is on structuring tests effectively and handling external dependencies.

---

## Test Structure

```rust
// Tests live in the same file (unit) or in tests/ (integration)

#[cfg(test)]
mod tests {
    use super::*;

    // Test naming: scenario_expected_outcome
    #[test]
    fn empty_cart_has_zero_total() {
        // Arrange
        let cart = ShoppingCart::new();

        // Act
        let total = cart.total();

        // Assert
        assert_eq!(total, 0);
    }

    #[test]
    fn adding_item_increases_total() {
        let mut cart = ShoppingCart::new();
        let item = Item::new("Widget", 1500);

        cart.add(item);

        assert_eq!(cart.total(), 1500);
        assert_eq!(cart.item_count(), 1);
    }

    // Output-based: pure function
    #[test]
    fn gold_tier_gets_15_percent_discount() {
        let result = calculate_discount(10000, Tier::Gold);
        assert_eq!(result, 8500);
    }

    // Testing error cases
    #[test]
    fn negative_quantity_is_rejected() {
        let result = Order::new(ProductId::new("abc"), -1, 1000);
        assert!(result.is_err());
        assert_eq!(result.unwrap_err(), OrderError::InvalidQuantity);
    }

    // Testing with Result — propagate errors in tests
    #[test]
    fn order_round_trip() -> Result<(), Box<dyn std::error::Error>> {
        let order = Order::new(ProductId::new("abc"), 2, 5000)?;
        let serialized = serde_json::to_string(&order)?;
        let deserialized: Order = serde_json::from_str(&serialized)?;
        assert_eq!(order, deserialized);
        Ok(())
    }
}
```

## Parameterized Tests

```rust
// Rust doesn't have built-in parameterized tests, but macros work well

macro_rules! discount_tests {
    ($($name:ident: $tier:expr, $price:expr, $expected:expr;)*) => {
        $(
            #[test]
            fn $name() {
                assert_eq!(calculate_discount($price, $tier), $expected);
            }
        )*
    }
}

discount_tests! {
    bronze_5_pct:  Tier::Bronze, 10000, 9500;
    silver_10_pct: Tier::Silver, 10000, 9000;
    gold_15_pct:   Tier::Gold,   10000, 8500;
}

// Or use the test-case crate for cleaner parameterization
// In Cargo.toml: test-case = "3"
use test_case::test_case;

#[test_case(Tier::Bronze, 10000 => 9500 ; "bronze 5 pct")]
#[test_case(Tier::Silver, 10000 => 9000 ; "silver 10 pct")]
#[test_case(Tier::Gold,   10000 => 8500 ; "gold 15 pct")]
fn discount_by_tier(tier: Tier, price: u64) -> u64 {
    calculate_discount(price, tier)
}
```

## Dependency Injection and Traits

```rust
// Define trait for external dependency
pub trait PaymentGateway {
    fn charge(&self, request: &PaymentRequest) -> Result<PaymentResult, PaymentError>;
}

// Production implementation
pub struct StripeGateway { /* ... */ }
impl PaymentGateway for StripeGateway {
    fn charge(&self, request: &PaymentRequest) -> Result<PaymentResult, PaymentError> {
        // real Stripe API call
    }
}

// Service depends on trait, not concrete type
pub struct OrderService<G: PaymentGateway, R: OrderRepository> {
    gateway: G,
    repo: R,
}

impl<G: PaymentGateway, R: OrderRepository> OrderService<G, R> {
    pub fn new(gateway: G, repo: R) -> Self {
        Self { gateway, repo }
    }

    pub fn place(&self, order: &Order) -> Result<OrderResult, OrderError> {
        let payment = self.gateway.charge(&order.payment_request())?;
        self.repo.save(order)
    }
}

// Fake for managed dependency (real implementation, in memory)
struct FakeOrderRepository {
    orders: RefCell<HashMap<OrderId, Order>>,
}

impl FakeOrderRepository {
    fn new() -> Self {
        Self { orders: RefCell::new(HashMap::new()) }
    }

    fn contains(&self, id: &OrderId) -> bool {
        self.orders.borrow().contains_key(id)
    }
}

impl OrderRepository for FakeOrderRepository {
    fn save(&self, order: &Order) -> Result<OrderResult, OrderError> {
        let id = OrderId::generate();
        self.orders.borrow_mut().insert(id.clone(), order.clone());
        Ok(OrderResult { id, success: true })
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    // Stub for external dependency — returns predefined answers
    struct StubPaymentGateway {
        result: Result<PaymentResult, PaymentError>,
    }

    impl PaymentGateway for StubPaymentGateway {
        fn charge(&self, _req: &PaymentRequest) -> Result<PaymentResult, PaymentError> {
            self.result.clone()
        }
    }

    fn success_gateway() -> StubPaymentGateway {
        StubPaymentGateway {
            result: Ok(PaymentResult { txn_id: "txn_123".into(), success: true }),
        }
    }

    fn failing_gateway(reason: &str) -> StubPaymentGateway {
        StubPaymentGateway {
            result: Err(PaymentError::Declined(reason.into())),
        }
    }

    #[test]
    fn successful_order_is_persisted() {
        let repo = FakeOrderRepository::new();
        let service = OrderService::new(success_gateway(), &repo);
        let order = make_test_order(5000);

        let result = service.place(&order).unwrap();

        assert!(result.success);
        assert!(repo.contains(&result.id));
    }

    #[test]
    fn failed_payment_returns_error() {
        let repo = FakeOrderRepository::new();
        let service = OrderService::new(failing_gateway("declined"), &repo);

        let result = service.place(&make_test_order(5000));

        assert!(result.is_err());
    }
}
```

## Mockall (For External Dependencies Only)

```rust
// In Cargo.toml: mockall = "0.12"
use mockall::automock;

#[automock]
pub trait EmailService {
    fn send(&self, to: &EmailAddress, subject: &str, body: &str) -> Result<(), EmailError>;
}

#[cfg(test)]
mod tests {
    use super::*;
    use mockall::predicate::*;

    #[test]
    fn order_confirmation_sends_email() {
        let mut mock_email = MockEmailService::new();
        mock_email
            .expect_send()
            .with(always(), eq("Order Confirmed"), always())
            .times(1)
            .returning(|_, _, _| Ok(()));

        let repo = FakeOrderRepository::new();
        let service = OrderService::new(success_gateway(), &repo, mock_email);

        let _ = service.place(&make_test_order(5000)).unwrap();
        // mock automatically verifies expectations on drop
    }
}
```

## Breaking Dependencies in Legacy Rust

```rust
// TECHNIQUE 1: Extract trait from concrete type
// Before: function calls concrete type directly
fn generate_report() -> Report {
    let data = DatabaseClient::query("SELECT ...");  // hard-coded
    process(data)
}

// After: accept trait object or generic
fn generate_report(source: &dyn DataSource) -> Report {
    let data = source.fetch();
    process(data)
}

// TECHNIQUE 2: Wrap free functions in a trait
// Before: calls a free function that hits the network
fn check_inventory(sku: &str) -> bool {
    external_api::check_stock(sku)  // can't test without network
}

// After: wrap in trait
trait InventoryChecker {
    fn check_stock(&self, sku: &str) -> bool;
}

struct RealInventoryChecker;
impl InventoryChecker for RealInventoryChecker {
    fn check_stock(&self, sku: &str) -> bool {
        external_api::check_stock(sku)
    }
}

// Test with a stub
struct AlwaysInStock;
impl InventoryChecker for AlwaysInStock {
    fn check_stock(&self, _sku: &str) -> bool { true }
}


// TECHNIQUE 3: Characterization tests
#[test]
fn characterization_legacy_pricing() {
    // Document current behavior — detect unintended changes
    assert_eq!(calculate_legacy_price("SKU001", 1), 2999);
    assert_eq!(calculate_legacy_price("SKU001", 10), 26991);
    assert_eq!(calculate_legacy_price("SKU001", 100), 239920);
    assert_eq!(calculate_legacy_price("PROMO_X", 1), 0);
}
```

## Test Organization

```
project/
├── src/
│   ├── lib.rs
│   ├── orders/
│   │   ├── mod.rs
│   │   ├── service.rs     # contains #[cfg(test)] mod tests at bottom
│   │   └── models.rs
│   └── payments/
│       └── gateway.rs
├── tests/                  # integration tests (each file is a separate crate)
│   ├── order_flow.rs
│   └── common/
│       └── mod.rs          # shared test helpers
└── Cargo.toml

# Run unit tests only (fast)
# cargo test --lib

# Run integration tests only
# cargo test --test order_flow

# Run everything
# cargo test
```

## Useful Testing Patterns

```rust
// Test helper: build test data with defaults
fn make_test_order(total: u64) -> Order {
    Order {
        product_id: ProductId::new("test_product"),
        quantity: 1,
        price: total,
    }
}

// Builder pattern for complex test data
struct TestOrderBuilder {
    product_id: String,
    quantity: u32,
    price: u64,
}

impl TestOrderBuilder {
    fn new() -> Self {
        Self {
            product_id: "default_product".into(),
            quantity: 1,
            price: 1000,
        }
    }

    fn with_quantity(mut self, q: u32) -> Self { self.quantity = q; self }
    fn with_price(mut self, p: u64) -> Self { self.price = p; self }

    fn build(self) -> Order {
        Order {
            product_id: ProductId::new(&self.product_id),
            quantity: self.quantity,
            price: self.price,
        }
    }
}

// Usage:
let order = TestOrderBuilder::new().with_quantity(10).with_price(500).build();


// Temporary directories
use tempfile::TempDir;

#[test]
fn export_writes_file() {
    let tmp = TempDir::new().unwrap();
    let path = tmp.path().join("report.csv");

    exporter.export(&report, &path).unwrap();

    let contents = std::fs::read_to_string(&path).unwrap();
    assert!(contents.starts_with("id,name,total"));
}
// TempDir is cleaned up on drop


// Asserting floating point
#[test]
fn tax_calculation() {
    let tax = calculate_tax(100.0, 0.0825);
    assert!((tax - 8.25).abs() < f64::EPSILON);
}
// Or use the approx crate:
// use approx::assert_relative_eq;
// assert_relative_eq!(tax, 8.25);
```

## Property-Based Testing (proptest)

```rust
use proptest::prelude::*;

proptest! {
    // Property: sorting is idempotent
    #[test]
    fn sorting_is_idempotent(mut xs: Vec<i32>) {
        xs.sort();
        let once = xs.clone();
        xs.sort();
        prop_assert_eq!(xs, once);
    }

    // Property: serialization round-trip
    #[test]
    fn json_roundtrip(value: MyStruct) {
        let json = serde_json::to_string(&value).unwrap();
        let back: MyStruct = serde_json::from_str(&json).unwrap();
        prop_assert_eq!(value, back);
    }
}
```
