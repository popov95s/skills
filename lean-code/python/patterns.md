# Patterns: Python

Which pattern fits which problem, in Python. The rules behind them are in
[SKILL.md](../SKILL.md); the TypeScript version is [typescript/patterns.md](../typescript/patterns.md).

| Problem | Pattern | Simpler form that may be enough |
| --- | --- | --- |
| Behaviour differs by kind, and cases call services or hold rules | [Strategy](#strategy) | A single kind needs none; pure data per kind is a `TypedDict` or `match`; a kind known at the call site needs no picker |
| Kinds need different data or allow different operations | [Explicit variants](#explicit-variants) | One dataclass when every field means the same for all kinds |
| Allowed actions depend on lifecycle | [State and transitions](#state-and-transitions) | One status value when nothing depends on the previous status |
| An operation can fail in several expected ways | [Result union](#result-union) | Raise when the caller can't do anything but fail |
| A vendor SDK's names, data or errors leak into callers | [Adapter](#adapter) | The existing integration function, when it already speaks your terms |
| Callers repeat the same setup and call order across a subsystem | [Facade](#facade) | One named function in the existing owner |
| Work is a fixed order of transformations | [Pipeline](#pipeline) | Plain calls in one function, which is often the whole pipeline |
| An implementation must be chosen and built from real services | [Picker](#picker) | No picker when the caller already knows the kind |

Several patterns can sit at different levels of one feature. Don't stack two on the same
decision.

## Strategy

**Use when** behaviour differs by kind and any case calls a service, does IO or holds a
rule. Use it from two kinds on.

```python
# quoter.py
class Quoter(Protocol):
    def quote(self) -> Money: ...

# courier_quoter.py
@dataclass(frozen=True)
class CourierQuoter:
    rates: Callable[[str], CourierRate]
    shipment: CourierShipment

    def quote(self) -> Money:
        rate = self.rates(self.shipment.postcode)
        return rate.base + rate.per_kg * self.shipment.weight_kg

# locker_quoter.py: takes its own kind's data
@dataclass(frozen=True)
class LockerQuoter:
    find: Callable[[LockerId], Locker]
    shipment: LockerShipment

    def quote(self) -> Money:
        locker = self.find(self.shipment.locker_id)
        if self.shipment.weight_kg > locker.max_weight_kg:
            raise ShipmentTooHeavyError(self.shipment.locker_id)
        return locker.flat_fee

# quoter_for.py: the only place that maps every delivery method to its behaviour
def quoter_for(shipment: Shipment) -> Quoter:
    match shipment:
        case PostalShipment():
            return PostalQuoter(table=postal_rates.load, shipment=shipment)
        case CourierShipment():
            return CourierQuoter(rates=courier_api.rates, shipment=shipment)
        case LockerShipment():
            return LockerQuoter(find=locker_api.find, shipment=shipment)
        case _:
            assert_never(shipment)

# price_order.py: knows the protocol, not the methods
def price_order(order: Order, quoter: Quoter) -> Money:
    return sum_items(order.items) + quoter.quote()

# place_order.py: the entry point calls the picker
total = price_order(order, quoter_for(order.shipment))
```

`assert_never` makes a missing method fail the type check. Inside `case LockerShipment`, `shipment`
is already narrowed to the locker variant.

**Several behaviours per kind:** return an object with all of them, and pick once.

```python
class Delivery(Protocol):
    def quote(self) -> Money: ...
    def track(self) -> Tracking: ...

def delivery_for(shipment: Shipment) -> Delivery:
    match shipment:
        case CourierShipment():
            return CourierDelivery(rates=courier_api.rates, track_parcel=courier_api.track, shipment=shipment)
        # ...one case per method
        case _:
            assert_never(shipment)
```

**Kind known at the call site:** no picker. Each caller calls its own operation
(`export_pdf(doc)` from the PDF endpoint, `export_docx(doc)` from the Word one).

**Not this:** dicts looked up by kind.

```python
QUOTERS = {"postal": postal_quote, "courier": courier_quote}  # each imports its API
total = QUOTERS[order.delivery_method](order.shipment)
```

**Adding a kind:** add a dataclass to the `Shipment` union, write `locker_quoter.py`, add one
case to `quoter_for`. The type checker points at the missing case.

## Explicit variants

**Use when** a kind changes which data is required or which outcomes exist.

```python
@dataclass(frozen=True)
class Email:
    address: str

@dataclass(frozen=True)
class Sms:
    phone_number: str

Destination = Email | Sms

def destination_label(destination: Destination) -> str:
    match destination:
        case Email(address=address):
            return address
        case Sms(phone_number=phone_number):
            return phone_number
        case _:
            assert_never(destination)
```

**Not this:**

```python
@dataclass
class Destination:
    kind: str
    address: str | None = None
    phone_number: str | None = None
```

**Simpler:** a `bool` field for a real yes/no fact; an optional field that is optional for
every kind.

**Adding a kind:** add `Webhook(url)` to the union. Every `match` with `assert_never` fails the
type check until it handles it.

## State and transitions

**Use when** allowed actions depend on what happened before, and separate flags could
contradict each other.

```python
@dataclass(frozen=True)
class Ready: ...

@dataclass(frozen=True)
class Running:
    request_id: str

@dataclass(frozen=True)
class Done:
    result: Report

@dataclass(frozen=True)
class Cancelled: ...

Job = Ready | Running | Done | Cancelled

def next_job(job: Job, event: JobEvent) -> Job:
    match job, event:
        case Ready(), Start(request_id=request_id):
            return Running(request_id)
        case Running(), Stop():
            return Cancelled()
        case Running(request_id=current), Finished(request_id=finished, result=result) if current == finished:
            return Done(result)
        case _:
            return job
```

One function owns every legal move. The request id stops a late result from overwriting a
newer run. Unlisted events leave the state unchanged, on purpose.

## Result union

**Use when** a caller must handle several expected outcomes. Return them once, instead of
passing `None` down a call chain.

```python
@dataclass(frozen=True)
class Found:
    order: Order

@dataclass(frozen=True)
class NotFound: ...

@dataclass(frozen=True)
class Forbidden: ...

FindOrder = Found | NotFound | Forbidden

def order_page_data(order_id: OrderId, viewer: Viewer) -> OrderPage:
    result = orders.find(order_id, viewer)
    match result:
        case Found(order=order):
            return to_order_page(order)
        case NotFound():
            raise OrderNotFoundError(order_id)
        case Forbidden():
            raise ForbiddenError()
        case _:
            assert_never(result)
```

**Simpler:** raise directly when no caller does anything except fail.

## Adapter

**Use when** a vendor's names, payloads or error codes leak into your code.

```python
# message_sender.py
@dataclass(frozen=True)
class Sent:
    receipt_id: str

@dataclass(frozen=True)
class Rejected:
    reason: str

class MessageSender(Protocol):
    def send(self, message: Message) -> Sent | Rejected: ...

# vendor_message_sender.py
@dataclass(frozen=True)
class VendorMessageSender:
    client: VendorClient

    def send(self, message: Message) -> Sent | Rejected:
        response = self.client.push(to=message.to, body=message.text)
        if response.status != "ACCEPTED":
            return Rejected(response.error_text)
        return Sent(response.id)
```

Callers use `MessageSender` and never see `ACCEPTED` or `error_text`.

**Simpler:** keep the existing integration function when it already returns your types.

## Facade

**Use when** callers keep repeating the same steps against a subsystem.

```python
def build_statement(deps: StatementDeps, account: Account, period: Period) -> Statement:
    entries = deps.ledger.entries(account.id, period)
    totals = statement_totals(entries)
    return render_statement(account, period, entries, totals)
```

**Not this:** a function that forwards its arguments to one call. That is a hollow wrapper.

## Pipeline

**Use when** work is an ordered set of transformations, each with clear input and output.

```python
def import_rows(text: str) -> list[Row]:
    parsed = parse_rows(text)
    normalized = normalize_rows(parsed)
    valid = validate_rows(normalized)
    return format_rows(valid)
```

These plain calls are usually the whole pipeline. Build a list of steps only when the order
itself changes at runtime. Steps that write or call out stay visible in the orchestrator.

## Picker

**Use when** an implementation must be chosen and built from real services: that is the
picker in [Strategy](#strategy). It is the only place strategies meet real services. A
frozen dataclass whose fields are its dependencies is the factory; it needs no separate
`make_*` function.

**Simpler:** a strategy with nothing to inject is an instance returned by its case. When the
interface has one method, make it a `Callable` alias instead of a `Protocol`; then a case can
return a bound service method, or a `functools.partial` of one, with no class of its own.

**Not this:** a container that hands out any dependency by name, a string registry, a
module-level dict of built strategies, or a class hierarchy for a small known set of kinds.
