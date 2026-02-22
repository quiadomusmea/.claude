---
name: writing-python-unit-tests
description: Use when writing or reviewing Python unit tests, when tests are flaky or slow due to time dependencies, when test coverage misses edge cases or error paths, when tests are tightly coupled to implementation details
---

# Writing Python Unit Tests

## Overview

**Systematic unit testing = isolate every unit, verify every path, catch real bugs.** Tests must be fast, deterministic, and independent. Each test verifies one behavior through the public interface.

## When to Use

- Writing new unit tests for Python code
- Reviewing existing tests for quality gaps
- Tests are slow, flaky, or non-deterministic
- Tests pass but real bugs slip through

**When NOT to use:** Integration tests, E2E tests, performance benchmarks

## Core Pattern: FIVE-STEP Unit Test

Every unit test follows this structure:

```python
def test_[unit]_[scenario]_[expected_result](self):
    # 1. ISOLATE: Mock all external dependencies
    mock_db = Mock(spec=Database)
    mock_db.find_by_email.return_value = None

    # 2. ARRANGE: Set up input and expected state
    service = UserService(mock_db, "http://api.test")
    expected_name = "Alice"

    # 3. ACT: Call exactly one public method
    result = service.create_user(expected_name, "alice@test.com")

    # 4. ASSERT: Verify return value AND side effects
    assert result["name"] == expected_name
    mock_db.insert.assert_called_once()

    # 5. (Implicit) CLEANUP: Use fixtures/teardown for shared state
```

## Quick Reference

### Isolation Rules

| Dependency Type | Isolation Method |
|----------------|------------------|
| Database/API | `Mock(spec=RealClass)` |
| `datetime.now()` | `@freeze_time("2024-01-15")` or `monkeypatch` |
| `time.time()` | `monkeypatch.setattr(time, "time", lambda: fixed_ts)` |
| File system | `tmp_path` fixture or `unittest.mock.mock_open` |
| Environment vars | `monkeypatch.setenv()` |
| Random/UUID | `monkeypatch.setattr(uuid, "uuid4", lambda: fixed_uuid)` |

**NEVER use `time.sleep()` in unit tests.** Mock the clock instead:

```python
# BAD - slow and flaky
def test_cache_expiry(self):
    cache.set("k", "v", ttl=60)
    time.sleep(61)  # wastes 61 seconds, may flake
    assert cache.get("k") is None

# GOOD - instant and deterministic
def test_cache_expiry(self, monkeypatch):
    now = 1000.0
    monkeypatch.setattr(time, "time", lambda: now)
    cache.set("k", "v", ttl=60)
    monkeypatch.setattr(time, "time", lambda: now + 61)
    assert cache.get("k") is None
```

### Test Coverage Checklist

For every function under test, verify ALL of these:

| Category | What to Test |
|----------|-------------|
| **Happy path** | Normal input, expected output |
| **Boundary values** | 0, 1, max, max+1, empty, exactly-at-threshold |
| **Invalid input** | None, wrong type, empty string, negative numbers |
| **Error paths** | Exceptions raised, error returns, partial failures |
| **Side effects** | Mock calls (args, count, order), state mutations |
| **Edge cases** | Concurrent access, None vs empty, Unicode, large data |

### Boundary Value Analysis

```python
# For: quantity must be 1-99
def test_quantity_zero_raises(self): ...        # below lower bound
def test_quantity_one_valid(self): ...          # lower bound
def test_quantity_99_valid(self): ...           # upper bound
def test_quantity_100_raises(self): ...         # above upper bound
def test_quantity_negative_raises(self): ...    # invalid

# For: free shipping at $50+
def test_shipping_at_49_99(self): ...           # just below
def test_shipping_at_50_00(self): ...           # exact boundary
def test_shipping_at_50_01(self): ...           # just above
```

### Assertion Strength

```python
# BAD - weak assertion, doesn't verify value
assert "created_at" in result

# GOOD - verifies exact value with frozen time
@freeze_time("2024-01-15 10:00:00")
def test_created_at_is_set(self):
    result = service.create_user("Alice", "a@t.com")
    assert result["created_at"] == "2024-01-15T10:00:00"

# BAD - only checks no exception (threading)
def test_thread_safety(self):
    # run threads...
    assert stats["size"] >= 0  # trivially true

# GOOD - verifies actual invariant
def test_thread_safety(self):
    # run threads adding 100 unique keys to cache of size 50...
    assert stats["size"] == 50  # exact expected state
```

### Public Interface Only

```python
# BAD - tests private method directly, coupled to implementation
def test_apply_coupon(self):
    discount = processor._apply_coupon("SAVE10", Decimal("100"))

# GOOD - tests through public interface
def test_process_order_with_percentage_coupon(self):
    result = processor.process_order(user_id=1, items=items,
        payment_method="card", coupon_code="SAVE10")
    assert result["discount"] == Decimal("10.00")
    payment.charge.assert_called_with(amount=99.0, ...)  # verify downstream effect
```

### Failure Chain Testing

When operations have sequential dependencies, test what happens when later steps fail:

```python
# Test: payment succeeds but inventory reservation fails
def test_reservation_failure_after_payment(self):
    inventory.check_stock.return_value = 10
    payment.charge.return_value = {"success": True, "transaction_id": "T1"}
    inventory.reserve.side_effect = RuntimeError("Connection lost")

    with pytest.raises(RuntimeError):
        processor.process_order(user_id=1, items=items, payment_method="card")
    # Verify: should payment be rolled back? Test the actual behavior.
```

### None vs Empty vs Missing

```python
# Test all three separately - they have different semantics
def test_name_none_raises(self):
    with pytest.raises(ValueError):
        service.create_user(None, "a@t.com")

def test_name_empty_raises(self):
    with pytest.raises(ValueError):
        service.create_user("", "a@t.com")

def test_name_whitespace_only(self):
    # Does the code handle this? If not, should it?
    service.create_user("   ", "a@t.com")
```

## Test Naming Convention

```
test_[unit]_[scenario]_[expected]
```

Examples:
- `test_create_user_valid_input_returns_user_with_id`
- `test_create_user_duplicate_email_raises_value_error`
- `test_calculate_total_above_threshold_free_shipping`

### Falsy Value Testing

Code that uses `if value is not None` vs `if value` vs `if not value` behaves differently for falsy values. Always test each falsy type separately:

```python
# These are all different - test each one
def test_cache_none_value(self): ...    # None - often confused with "not found"
def test_cache_false_value(self): ...   # False - falsy but valid
def test_cache_zero_value(self): ...    # 0 - falsy but valid
def test_cache_empty_string(self): ...  # "" - falsy but valid
def test_cache_empty_list(self): ...    # [] - falsy but valid
```

### Use `@pytest.mark.parametrize` for Repetitive Boundaries

```python
# BAD - 5 nearly identical test functions
def test_quantity_zero_raises(self): ...
def test_quantity_negative_raises(self): ...
def test_quantity_100_raises(self): ...

# GOOD - parametrize for same-pattern tests
@pytest.mark.parametrize("quantity,expected_error", [
    (0, "Invalid quantity: 0"),
    (-1, "Invalid quantity: -1"),
    (100, "Invalid quantity: 100"),
])
def test_calculate_total_invalid_quantity_raises(self, quantity, expected_error):
    items = [{"price": "10.00", "quantity": quantity}]
    with pytest.raises(ValueError, match=expected_error):
        processor.calculate_total(items)
```

### Private State Access in Tests

```python
# AVOID - accessing private attributes in assertions
assert len(cache._access_order) == len(cache._cache)

# PREFER - verify through public interface
stats = cache.stats()
assert stats["size"] == expected_size

# EXCEPTION: Thread safety invariant tests may access internal state
# when no public API exposes the invariant. Document WHY.
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `time.sleep()` in tests | Mock `time.time()` or use `freeze_time` |
| Testing private methods (`_method`) | Test through public interface |
| `assertIn` instead of exact value | Freeze non-deterministic sources, assert exact |
| Missing `spec=` on Mock | Always use `Mock(spec=RealClass)` to catch typos |
| Thread tests with no invariant | Assert exact expected state, not just "no crash" |
| One test per method, not per behavior | One test per scenario (happy/error/boundary) |
| Mocking everything including the SUT | Only mock dependencies, never the system under test |
| Not testing falsy values (None/0/False/"") | Each falsy type has different semantics |
| Copy-paste boundary tests | Use `@pytest.mark.parametrize` |

## Mock `spec` Usage

```python
# BEST - use real class when available
mock_db = Mock(spec=Database)

# OK - use spec list when real class not importable
mock_db = Mock(spec=['find_by_email', 'insert', 'update'])

# BAD - bare Mock catches nothing
mock_db = Mock()  # mock_db.find_by_emial() won't raise
```

## pytest Fixture Pattern

```python
@pytest.fixture
def mock_db():
    db = Mock(spec=Database)
    db.find_by_email.return_value = None
    db.insert.return_value = 1
    return db

@pytest.fixture
def service(mock_db):
    return UserService(mock_db, "http://api.test")

class TestCreateUser:
    def test_valid_input(self, service, mock_db):
        result = service.create_user("Alice", "a@t.com")
        assert result["id"] == 1
        mock_db.insert.assert_called_once()
```
