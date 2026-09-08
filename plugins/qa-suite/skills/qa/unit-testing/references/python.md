# Python — pytest

## Detect and run

- `pyproject.toml` (`[tool.pytest.ini_options]`), `pytest.ini`, `setup.cfg`, or a
  `tests/` dir → pytest. `unittest`-only repos exist; follow what is there.
- Run: `pytest path/to/test_file.py -q`, one test: `pytest path::test_name`.
- Coverage on the change: `pytest --cov=package --cov-report=term-missing`.
  Read the `Missing` column — those line numbers are the branch list.
- Faster feedback: `-x` (stop at first failure), `--lf` (last failed),
  `-k "expr"` (name filter).

## Layout

Mirror the source tree: `src/billing/refund.py` → `tests/billing/test_refund.py`.
Test functions start with `test_`. Prefer plain functions over `unittest.TestCase`
classes unless the repo already uses them.

## Idioms

```python
import pytest
from billing.refund import refund, AlreadySettled

def test_refund_rejected_when_order_already_settled():
    order = make_order(status="settled")          # arrange
    with pytest.raises(AlreadySettled):           # act + assert
        refund(order, amount=100)

@pytest.mark.parametrize(
    ("amount", "expected"),
    [(0, 0), (1, 1), (99, 99), (100, 100)],
    ids=["zero", "min", "below-cap", "at-cap"],
)
def test_refund_amount_is_passed_through(amount, expected):
    assert refund(make_order(), amount=amount).amount == expected
```

- **Fixtures** for shared arrange: `@pytest.fixture` returning a built object;
  `tmp_path` for files, `caplog` for log assertions, `capsys` for stdout.
- **Fakes over mocks**: pass a small in-memory class implementing the protocol.
  When you do need a mock, `unittest.mock.Mock(spec=RealClass)` — `spec` makes a
  renamed method fail the test instead of silently returning a Mock.
- **monkeypatch** only for things you cannot inject (`monkeypatch.setenv`,
  patching a module-level constant). Patch where the name is *used*, not where it
  is defined: `monkeypatch.setattr("billing.refund.now", lambda: FIXED)`.
- **Time and randomness**: inject a clock, or use `freezegun` / `time-machine` if
  already in the project. Seed `random.Random(0)` explicitly.
- **Approximate floats**: `pytest.approx(0.3)`, never bare `==`.
- **Exceptions**: assert the message too when it is part of the contract —
  `pytest.raises(ValueError, match="amount must be positive")`.
- **async**: `pytest-asyncio` with `@pytest.mark.asyncio` (or `asyncio_mode=auto`).

## Traps

- A `Mock` without `spec` answers every attribute — tests keep passing after you
  rename the real method.
- `assert x == True` hides the diff; assert the value itself.
- Mutable default fixtures shared across tests leak state; build fresh objects.
- `-p no:randomly` is a smell if the repo has `pytest-randomly` — order
  dependence is a real bug, fix the test rather than pinning the order.
