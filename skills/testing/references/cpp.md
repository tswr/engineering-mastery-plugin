# C++ Testing Reference

Idiomatic C++ testing patterns using GoogleTest and GoogleMock. Covers test structure, dependency injection for testability, and techniques for breaking dependencies in legacy C++ code.

---

## Test Structure (GoogleTest)

```cpp
#include <gtest/gtest.h>

// Test naming: Scenario_ExpectedOutcome
TEST(ShoppingCartTest, EmptyCartHasZeroTotal) {
    // Arrange
    const ShoppingCart cart;

    // Act
    const auto total = cart.total();

    // Assert
    EXPECT_EQ(total, 0);
}

TEST(ShoppingCartTest, AddingItemIncreasesTotal) {
    // Arrange
    ShoppingCart cart;
    const auto item = Item{"Widget", 1500};

    // Act
    cart.add(item);

    // Assert
    EXPECT_EQ(cart.total(), 1500);
    EXPECT_EQ(cart.item_count(), 1);
}

TEST(DiscountCalculatorTest, GoldTierGets15PercentDiscount) {
    // Output-based: pure function, no state
    const auto result = calculate_discount(/*price=*/10000, Tier::Gold);
    EXPECT_EQ(result, 8500);
}

// Parameterized tests for multiple scenarios
struct DiscountTestCase {
    Tier tier;
    int price;
    int expected;
    std::string name;
};

class DiscountTest : public testing::TestWithParam<DiscountTestCase> {};

TEST_P(DiscountTest, AppliesCorrectDiscount) {
    const auto& [tier, price, expected, name] = GetParam();
    EXPECT_EQ(calculate_discount(price, tier), expected);
}

INSTANTIATE_TEST_SUITE_P(Tiers, DiscountTest, testing::Values(
    DiscountTestCase{Tier::Bronze, 10000, 9500, "bronze_5_pct"},
    DiscountTestCase{Tier::Silver, 10000, 9000, "silver_10_pct"},
    DiscountTestCase{Tier::Gold,   10000, 8500, "gold_15_pct"}
));

// Test fixtures for shared setup
class DatabaseOrderTest : public testing::Test {
protected:
    void SetUp() override {
        db_ = std::make_unique<FakeDatabase>();
        service_ = std::make_unique<OrderService>(*db_);
    }

    std::unique_ptr<FakeDatabase> db_;
    std::unique_ptr<OrderService> service_;
};

TEST_F(DatabaseOrderTest, PlacedOrderIsPersisted) {
    const auto order = Order{ProductId{"abc"}, Quantity{2}};

    const auto result = service_->place(order);

    ASSERT_TRUE(result.ok());
    EXPECT_TRUE(db_->contains(result.value().id()));
}
```

## Dependency Injection for Testability

```cpp
// Interface for external dependency
class PaymentGateway {
public:
    virtual ~PaymentGateway() = default;
    virtual PaymentResult charge(const PaymentRequest& req) = 0;
};

// Production implementation
class StripeGateway final : public PaymentGateway {
public:
    PaymentResult charge(const PaymentRequest& req) override {
        // real Stripe API call
    }
};

// Inject via constructor — testable by design
class OrderService {
    PaymentGateway& gateway_;
    OrderRepository& repo_;

public:
    OrderService(PaymentGateway& gateway, OrderRepository& repo)
        : gateway_{gateway}, repo_{repo} {}

    OrderResult place(const Order& order) {
        auto payment = gateway_.charge(order.payment_request());
        if (!payment.ok()) return OrderResult::failed(payment.error());
        return repo_.save(order);
    }
};

// In tests: use a fake for the managed dependency, mock for the external
class FakeOrderRepository : public OrderRepository {
    std::unordered_map<OrderId, Order> store_;
public:
    OrderResult save(const Order& order) override {
        auto id = OrderId::generate();
        store_[id] = order;
        return OrderResult::success(id);
    }
    bool contains(const OrderId& id) const { return store_.count(id) > 0; }
};
```

## GoogleMock (For External Dependencies Only)

```cpp
#include <gmock/gmock.h>

// Mock the external system — not internal collaborators
class MockPaymentGateway : public PaymentGateway {
public:
    MOCK_METHOD(PaymentResult, charge, (const PaymentRequest&), (override));
};

TEST(OrderServiceTest, SuccessfulOrderChargesPaymentGateway) {
    // Arrange
    MockPaymentGateway gateway;
    FakeOrderRepository repo;  // real (fake) implementation — not mocked
    OrderService service{gateway, repo};

    const auto order = make_test_order(/*total=*/5000);

    // Expect: outgoing interaction with external system
    EXPECT_CALL(gateway, charge(testing::_))
        .WillOnce(testing::Return(PaymentResult::success("txn_123")));

    // Act
    const auto result = service.place(order);

    // Assert: observable outcome
    ASSERT_TRUE(result.ok());
    EXPECT_TRUE(repo.contains(result.value().id()));
}

TEST(OrderServiceTest, FailedPaymentReturnsError) {
    MockPaymentGateway gateway;
    FakeOrderRepository repo;
    OrderService service{gateway, repo};

    EXPECT_CALL(gateway, charge(testing::_))
        .WillOnce(testing::Return(PaymentResult::failed("declined")));

    const auto result = service.place(make_test_order(5000));

    EXPECT_FALSE(result.ok());
    EXPECT_EQ(result.error(), "declined");
}
```

## Breaking Dependencies in Legacy C++

```cpp
// PROBLEM: function calls a static method directly — untestable
class ReportGenerator {
public:
    Report generate() {
        auto data = DatabaseClient::query("SELECT ...");  // static call
        // ... process data ...
    }
};

// TECHNIQUE 1: Extract and Override
// Minimally invasive — create a virtual method, override in test
class ReportGenerator {
public:
    virtual ~ReportGenerator() = default;
    Report generate() {
        auto data = fetch_data();  // now virtual
        // ... process data ...
    }
protected:
    virtual QueryResult fetch_data() {
        return DatabaseClient::query("SELECT ...");
    }
};

// Test subclass overrides the seam
class TestableReportGenerator : public ReportGenerator {
    QueryResult test_data_;
protected:
    QueryResult fetch_data() override { return test_data_; }
public:
    explicit TestableReportGenerator(QueryResult data)
        : test_data_{std::move(data)} {}
};

TEST(ReportGeneratorTest, GeneratesReportFromData) {
    auto gen = TestableReportGenerator{make_test_data()};
    const auto report = gen.generate();
    EXPECT_EQ(report.row_count(), 5);
}


// TECHNIQUE 2: Parameterize Constructor
// Add dependency as parameter with default for backward compatibility
class ReportGenerator {
public:
    explicit ReportGenerator(
        std::unique_ptr<DataSource> source = std::make_unique<DatabaseSource>())
        : source_{std::move(source)} {}

    Report generate() {
        auto data = source_->fetch();
        // ...
    }
private:
    std::unique_ptr<DataSource> source_;
};

// Test injects a fake
TEST(ReportGeneratorTest, ProcessesDataCorrectly) {
    auto fake_source = std::make_unique<FakeDataSource>(make_test_data());
    ReportGenerator gen{std::move(fake_source)};
    const auto report = gen.generate();
    EXPECT_EQ(report.row_count(), 5);
}


// TECHNIQUE 3: Characterization test for legacy code
// Document what the code does NOW before changing it
TEST(LegacyPricingTest, CharacterizationCurrentBehavior) {
    // We don't know if these are "correct" — we're documenting
    // current behavior to detect unintended changes
    EXPECT_DOUBLE_EQ(calculate_legacy_price("SKU001", 1), 29.99);
    EXPECT_DOUBLE_EQ(calculate_legacy_price("SKU001", 10), 269.91);
    EXPECT_DOUBLE_EQ(calculate_legacy_price("SKU001", 100), 2399.20);
    // Odd bulk discount behavior — probably intentional? Lock it in.
    EXPECT_DOUBLE_EQ(calculate_legacy_price("PROMO_X", 1), 0.0);
}
```

## Test Organization

```
project/
├── src/
│   ├── order_service.h
│   ├── order_service.cpp
│   └── ...
├── test/
│   ├── order_service_test.cpp    # unit tests
│   ├── fakes/
│   │   ├── fake_database.h       # fake implementations
│   │   └── fake_payment.h
│   ├── integration/
│   │   └── order_flow_test.cpp   # integration tests
│   └── test_helpers.h            # shared test utilities
└── CMakeLists.txt

# In CMakeLists.txt: separate test targets
add_executable(unit_tests
    test/order_service_test.cpp
    test/discount_test.cpp
)
target_link_libraries(unit_tests GTest::gtest_main GTest::gmock)

add_executable(integration_tests
    test/integration/order_flow_test.cpp
)
# Integration tests may link against real infrastructure stubs
```

## Property-Based Testing (RapidCheck)

```cpp
#include <rapidcheck.h>

// Property: sorting is idempotent
RC_GTEST_PROP(Sort, isIdempotent, (std::vector<int> xs)) {
    std::sort(xs.begin(), xs.end());
    auto once = xs;
    std::sort(xs.begin(), xs.end());
    RC_ASSERT(xs == once);
}

// Property: serialization round-trip
RC_GTEST_PROP(Serialization, roundTrip, (MyStruct value)) {
    auto json = serialize(value);
    auto back = deserialize(json);
    RC_ASSERT(value == back);
}
```
