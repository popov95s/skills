# Lean code: Python

The rules are in [SKILL.md](../SKILL.md). This file shows how to write them in Python.

## Types

- Type every public function. The type checker, not `isinstance` chains, proves the variant.
- Kinds are `Literal["pdf", "docx"]` or an `enum.StrEnum`.
- Variants are frozen dataclasses, one per kind, joined with `|`, and read with `match`.
- A record of values per kind is a `TypedDict` with one key per kind. A missing kind then
  fails where the record is built and wherever it is indexed with the kind union.
- An exhaustive `match` ends in `case _: assert_never(value)`, from `typing` (3.11+) or
  `typing_extensions`. That is the checker's own exhaustiveness hook, not a hand-rolled
  assert. `enum.StrEnum` also needs 3.11+.
- No `typing.cast`, no `Any` to quiet the checker, no `# type: ignore`, `# noqa` or
  `# pyright: ignore`.
- Untrusted input is parsed at the boundary (pydantic if the project uses it, otherwise a
  parse function that returns a dataclass); inside, trust the type.
- `fallback if x is None else x` for defaults. `x or fallback` only when every falsy value
  means "missing".
- Strategy dependencies are a `Protocol` or `Callable[...]` naming only what is used.

## Keep types useful

- Plain types, dataclasses and unions over clever generics. `TypeVar`, `ParamSpec` and
  `@overload` only when they state a link callers rely on.
- Reuse the project's id types (`NewType`) and schemas. Don't start a parallel set.
- Narrow untrusted input explicitly. An annotation doesn't validate network or file data.

## Typed data, not strings

We wrote the value, so read its field. Don't take a string apart with a regex.

```python
# before: the label is parsed again to learn what we put into it
is_return = re.match(r"^parcel:[^:]+:[^:]+:return$", label) is not None
carrier_id = label.split(":")[1]

# after: the record has the fields; the string exists only where it is stored
is_return = ref.direction == "return"
carrier_id = ref.carrier_id
```

A key with parts is a nested dict or a tuple of typed values, not a joined string:

```python
# before
stock[f"{warehouse_id}:{sku}"] = level

# after
stock: dict[tuple[WarehouseId, Sku], StockLevel] = {}
stock[(warehouse_id, sku)] = level
```

The standard library parses what it knows: `Path(file_name).suffix`, `urlparse(link)`. When
a stored string must stay a string, one codec module builds and reads it, accepts the old
shape, writes the new one, and has a round-trip test.

## Errors

- One exception class per failure mode, under a domain base class.
- Catch the specific class. Never a bare `except:` or `except Exception: pass`.
- Re-raise with `raise ... from error` so the cause stays attached.

## Modules and classes

- Functions and frozen dataclasses first. A class owns state or dependencies; it is not a
  namespace for static methods.
- `Protocol` over abstract base classes. No inheritance tree for kinds: use variants and
  strategies.
- No class that only forwards to another, and no `Manager` or `Helper` grab-bag.
- Files, connections and locks are opened with `with`, never closed by hand.
- No blocking calls inside `async def`.
- Strategies are built in their picker, not at module level.

## Python specifics

- Tests live in test files: no `if __name__ == "__main__":` self-checks in library modules.
- No boolean parameters that pick behaviour, keyword-only ones included (`*, strict: bool`).
- No mutable default arguments.
- No work at import time beyond definitions: no network, files or global state changes.

## Branching

Fine: data mapping in one exhaustive `match`.

```python
def format_label(fmt: Format) -> str:
    match fmt:
        case "pdf":
            return "PDF"
        case "docx":
            return "Word"
        case _:
            assert_never(fmt)
```

Fine: reading a result union, including a case that raises.

```python
def load_document(document_id: DocumentId) -> Document:
    result = document_service.find(document_id)
    match result:
        case Found(document=document):
            return document
        case NotFound():
            raise DocumentNotFoundError(document_id)
        case _:
            assert_never(result)
```

When a choice determines which service to call or which rules to apply,
give each case its own strategy. Choose the strategy once at the entry
point so the rest of the code can use a common interface. See the
[Python strategy pattern](patterns.md#strategy).

Fine: a picker. Its cases only build a strategy, or only hand the narrowed value to a
strategy's method:

```python
def notifier_for(destination: Destination) -> Notifier:
    match destination:
        case Email(address=address):
            return EmailNotifier(send_mail=smtp.send, address=address)
        case Sms(phone_number=phone_number):
            return SmsNotifier(send_sms=sms_api.send, phone_number=phone_number)
        case _:
            assert_never(destination)
```

## Ternaries

```python
noun = "item" if count == 1 else "items"           # fine: values
text = format_list(items) if items else "—"        # fine: pure call
tone = "red" if error else "amber" if warning else "grey"  # not fine: nested
```

Replace an impure or nested one with a named function with early returns.

## Files

- One module per responsibility, named after what it exports: `load_snapshot.py` exports
  `load_snapshot`.
- `__init__.py` holds only `from .x import y` lines, and `__all__` if the project uses it.
- Import from the package, one line per package:

  ```python
  # Not this
  from app.orders.order_pricing import order_pricing
  from app.orders.order_refunds import order_refunds

  # This
  from app.orders import order_pricing, order_refunds
  ```

- Inside `orders/`, import siblings by module (`from .order_refunds import ...`), never
  from the package itself.
- Exception: a flat package of independent modules, one per table or client (stores,
  repositories), is imported by module. An `__init__.py` over it loads every module it names.

## Docstrings

```python
"""Latest permission snapshot for an org."""
```

No `Args:`, `Returns:` or `Raises:` sections.
