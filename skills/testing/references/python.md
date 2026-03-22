# Python Testing Reference

Idiomatic Python testing patterns using pytest. Covers test structure, fixtures, parametrization, mocking external dependencies, and techniques for testing legacy Python code.

---

## Test Structure (pytest)

```python
# Test naming: test_scenario_expected_outcome

def test_empty_cart_has_zero_total():
    # Arrange
    cart = ShoppingCart()

    # Act
    total = cart.total()

    # Assert
    assert total == 0


def test_adding_item_increases_total():
    # Arrange
    cart = ShoppingCart()
    item = Item(name="Widget", price=1500)

    # Act
    cart.add(item)

    # Assert
    assert cart.total() == 1500
    assert cart.item_count() == 1


# Output-based: pure function, no state
def test_gold_tier_gets_15_percent_discount():
    result = calculate_discount(price=10000, tier=Tier.GOLD)
    assert result == 8500


# Parametrized tests for multiple scenarios
@pytest.mark.parametrize("tier, price, expected", [
    (Tier.BRONZE, 10000, 9500),
    (Tier.SILVER, 10000, 9000),
    (Tier.GOLD,   10000, 8500),
], ids=["bronze_5_pct", "silver_10_pct", "gold_15_pct"])
def test_discount_by_tier(tier, price, expected):
    assert calculate_discount(price=price, tier=tier) == expected
```

## Fixtures

```python
import pytest

# Simple fixture: fresh instance per test
@pytest.fixture
def cart():
    return ShoppingCart()


@pytest.fixture
def sample_item():
    return Item(name="Widget", price=1500)


def test_cart_with_item(cart, sample_item):
    cart.add(sample_item)
    assert cart.total() == 1500


# Fixture with teardown
@pytest.fixture
def temp_database():
    db = FakeDatabase()
    db.connect()
    yield db
    db.disconnect()


# Fixture composition: build complex setups from simple fixtures
@pytest.fixture
def order_service(temp_database, mock_payment_gateway):
    return OrderService(db=temp_database, gateway=mock_payment_gateway)


# Factory fixture: when tests need to create multiple instances with variations
@pytest.fixture
def make_order():
    def _make(product_id="prod_001", quantity=1, price=1000):
        return Order(
            product_id=ProductId(product_id),
            quantity=quantity,
            price=price,
        )
    return _make


def test_bulk_discount(order_service, make_order):
    order = make_order(quantity=100, price=500)
    result = order_service.place(order)
    assert result.discount_applied
```

## Mocking External Dependencies Only

```python
from unittest.mock import Mock, patch, create_autospec

# CORRECT: mock the external system (payment gateway)
# Use real (fake) implementations for internal collaborators

class FakeOrderRepository:
    """Fake for managed dependency — works like the real thing, in memory."""
    def __init__(self):
        self._orders = {}

    def save(self, order):
        order_id = f"ord_{len(self._orders)}"
        self._orders[order_id] = order
        return OrderResult(id=order_id, success=True)

    def find(self, order_id):
        return self._orders.get(order_id)


@pytest.fixture
def fake_repo():
    return FakeOrderRepository()


@pytest.fixture
def mock_payment_gateway():
    """Mock for unmanaged dependency — external system."""
    gateway = create_autospec(PaymentGateway)
    gateway.charge.return_value = PaymentResult(success=True, txn_id="txn_123")
    return gateway


def test_successful_order_charges_gateway(mock_payment_gateway, fake_repo):
    service = OrderService(gateway=mock_payment_gateway, repo=fake_repo)
    order = Order(product_id=ProductId("abc"), quantity=2, price=5000)

    result = service.place(order)

    # Assert observable outcome
    assert result.success
    assert fake_repo.find(result.id) is not None
    # Verify outgoing interaction with external system
    mock_payment_gateway.charge.assert_called_once()


def test_failed_payment_returns_error(mock_payment_gateway, fake_repo):
    mock_payment_gateway.charge.return_value = PaymentResult(
        success=False, error="declined"
    )
    service = OrderService(gateway=mock_payment_gateway, repo=fake_repo)

    result = service.place(Order(product_id=ProductId("abc"), quantity=1, price=5000))

    assert not result.success
    assert result.error == "declined"


# WRONG: don't mock internal collaborators
# This couples the test to implementation and breaks on refactoring
# ❌ mock_validator = Mock()
# ❌ mock_validator.validate.return_value = True
# ❌ service = OrderService(validator=mock_validator, ...)
```

## Breaking Dependencies in Legacy Python

```python
# TECHNIQUE 1: Parameterize the dependency
# Before: hard-coded dependency, untestable
class ReportGenerator:
    def generate(self):
        data = DatabaseClient.query("SELECT ...")  # static call
        return self._process(data)

# After: inject the dependency, backward-compatible default
class ReportGenerator:
    def __init__(self, data_source=None):
        self._source = data_source or DatabaseClient()

    def generate(self):
        data = self._source.query("SELECT ...")
        return self._process(data)

# Test with a fake
def test_report_generation():
    fake_source = FakeDataSource(test_data)
    gen = ReportGenerator(data_source=fake_source)
    report = gen.generate()
    assert report.row_count == 5


# TECHNIQUE 2: Patch at the boundary (when you can't change the code)
# Use patch to replace an external call at the module level
def test_report_with_patch():
    with patch("myapp.reports.DatabaseClient.query") as mock_query:
        mock_query.return_value = make_test_data()
        gen = ReportGenerator()
        report = gen.generate()
        assert report.row_count == 5

# NOTE: patch is a last resort — prefer dependency injection.
# Patching is fragile: it depends on the import path, breaks when
# code is moved, and couples the test to the module structure.


# TECHNIQUE 3: Characterization tests
def test_characterization_legacy_pricing():
    """Documents current behavior. Not asserting correctness —
    asserting what the code does NOW so we detect unintended changes."""
    assert calculate_legacy_price("SKU001", qty=1) == pytest.approx(29.99)
    assert calculate_legacy_price("SKU001", qty=10) == pytest.approx(269.91)
    assert calculate_legacy_price("SKU001", qty=100) == pytest.approx(2399.20)
    # Weird edge case — probably intentional bulk discount?
    assert calculate_legacy_price("PROMO_X", qty=1) == pytest.approx(0.0)
```

## Test Organization

```
project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── orders/
│       │   ├── service.py
│       │   └── models.py
│       └── payments/
│           └── gateway.py
├── tests/
│   ├── conftest.py              # shared fixtures
│   ├── unit/
│   │   ├── test_order_service.py
│   │   ├── test_discount.py
│   │   └── fakes.py             # fake implementations
│   ├── integration/
│   │   └── test_order_flow.py
│   └── characterization/
│       └── test_legacy_pricing.py
├── pyproject.toml
└── pytest.ini

# pytest.ini: mark-based separation
[pytest]
markers =
    unit: fast unit tests
    integration: tests that touch real infrastructure
    slow: tests that take more than 1 second

# Run fast tests during development
# pytest -m unit

# Run everything in CI
# pytest
```

## Useful pytest Features

```python
# Asserting exceptions
def test_negative_quantity_is_rejected():
    with pytest.raises(ValueError, match="quantity must be positive"):
        Order(product_id=ProductId("abc"), quantity=-1, price=1000)


# Approximate floating point comparison
def test_tax_calculation():
    assert calculate_tax(amount=100.0, rate=0.0825) == pytest.approx(8.25)


# Temporary directories and files
def test_export_writes_file(tmp_path):
    output = tmp_path / "report.csv"
    exporter = ReportExporter()
    exporter.export(report, output)
    assert output.read_text().startswith("id,name,total")


# Freezing time for deterministic tests
from freezegun import freeze_time

@freeze_time("2026-01-15")
def test_subscription_expires_after_30_days():
    sub = Subscription(started=date(2025, 12, 15))
    assert sub.is_expired()
```

## Property-Based Testing (Hypothesis)

```python
from hypothesis import given, strategies as st

# Property: sorting is idempotent
@given(st.lists(st.integers()))
def test_sorting_is_idempotent(xs):
    once = sorted(xs)
    twice = sorted(once)
    assert once == twice

# Property: serialization round-trip
@given(st.builds(User, name=st.text(), age=st.integers(min_value=0, max_value=150)))
def test_user_json_roundtrip(user):
    assert User.from_json(user.to_json()) == user

# Property: output has same elements as input
@given(st.lists(st.integers()))
def test_sort_preserves_elements(xs):
    assert sorted(xs) == sorted(sorted(xs))
    assert len(sorted(xs)) == len(xs)
```
