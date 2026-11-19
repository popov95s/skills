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

Not fine: cases that call services. Write the strategy and its picker instead: see
[patterns.md](patterns.md#strategy). For two kinds at once (requester kind × kind of read),
the shape is the same as [typescript/strategy-example.md](../typescript/strategy-example.md):
one picker per kind, and the second kind becomes methods on the strategy.

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

## Diff search

```
{ git diff -U0 "$(git merge-base <base> HEAD)" -- '*.py'
  git ls-files -o --exclude-standard -- '*.py' | xargs -I{} git diff -U0 --no-index /dev/null {}; } \
  | grep -E '^\+\+\+ |^\+.*(\bmatch |kind (==|!=) "|type (==|!=) "|isinstance\(|endswith\("\.|startswith\("[a-z-]+:"|: bool\b|\bcast\(|: Any\b|[(,] *_[a-z]+[,:)]|type: ignore|noqa|pyright: ignore|lambda[^:]*: None|"[a-z_]+" in [a-z])|^\+ *(async )?def .*=\s*(\[\]|\{\}|set\(\))\s*[,)]'
```

It covers committed, uncommitted and new untracked files. The `+++` lines name the file each
hit below them belongs to. Hits in comments are questions too. It can't see nested
ternaries, dicts keyed by kind or defaults in multi-line signatures: reread the diff for those.
