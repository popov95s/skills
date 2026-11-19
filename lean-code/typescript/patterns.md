# Patterns: TypeScript and JavaScript

Which pattern fits which problem, in TypeScript. The rules behind them are in
[SKILL.md](../SKILL.md); the Python version is [python/patterns.md](../python/patterns.md).

| Problem | Pattern | Simpler form that may be enough |
| --- | --- | --- |
| Behaviour differs by kind, and cases call services or hold rules | [Strategy](#strategy) | A single kind needs none; pure data per kind is a typed record; a kind known at the call site needs no picker |
| Kinds need different data or allow different operations | [Explicit variants](#explicit-variants) | A plain object when every field means the same for all kinds |
| Allowed actions depend on lifecycle | [State and transitions](#state-and-transitions) | One status value when nothing depends on the previous status |
| An operation can fail in several expected ways | [Result union](#result-union) | Throw when the caller can't do anything but fail |
| A vendor SDK's names, data or errors leak into callers | [Adapter](#adapter) | The existing integration function, when it already speaks your terms |
| Callers repeat the same setup and call order across a subsystem | [Facade](#facade) | One named operation in the existing owner |
| Work is a fixed order of transformations | [Pipeline](#pipeline) | Plain calls in one function, which is often the whole pipeline |
| An implementation must be chosen and built from real services | [Picker](#picker) | No picker when the caller already knows the kind |
| A component's mode props pick different layouts or actions | [UI composition](#ui-composition) | A `disabled` or `loading` prop, when it only states a fact |

Several patterns can sit at different levels of one feature. Don't stack two on the same
decision.

## Strategy

**Use when** behaviour differs by kind and any case calls a service, does IO or holds a
rule. Use it from two kinds on.

```ts
// quoter.ts
export interface Quoter { quote: () => Promise<Money> }

// courierQuoter.ts
export const courierQuoter = (
  deps: { rates: typeof courierApi.rates },
  shipment: CourierShipment,
): Quoter => ({
  quote: async () => {
    const rate = await deps.rates(shipment.postcode);
    return addMoney(rate.base, multiplyMoney(rate.perKg, shipment.weightKg));
  },
});

// lockerQuoter.ts: takes its own kind's data
export const lockerQuoter = (deps: { find: typeof lockerApi.find }, shipment: LockerShipment): Quoter => ({
  quote: async () => {
    const locker = await deps.find(shipment.lockerId);
    if (shipment.weightKg > locker.maxWeightKg) throw new ShipmentTooHeavyError(shipment.lockerId);
    return locker.flatFee;
  },
});

// quoterFor.ts: the only place that maps every delivery method to its behaviour
export const quoterFor = (shipment: Shipment): Quoter => {
  switch (shipment.method) {
    case "postal":
      return postalQuoter({ table: postalRates.load }, shipment);
    case "courier":
      return courierQuoter({ rates: courierApi.rates }, shipment);
    case "locker":
      return lockerQuoter({ find: lockerApi.find }, shipment);
  }
};

// priceOrder.ts: knows the interface, not the methods
export const priceOrder = async (order: Order, quoter: Quoter) =>
  addMoney(sumItems(order.items), await quoter.quote());

// placeOrder.ts: the entry point calls the picker
const total = await priceOrder(order, quoterFor(order.shipment));
```

`quoterFor` returns a value from every case and declares `Quoter`, so a missing method fails
to compile. Inside `case "locker"`, `shipment` is already the locker variant.

**Several behaviours per kind:** return an object with all of them, and pick once.

```ts
// delivery.ts
export interface Delivery { quote: () => Promise<Money>; track: () => Promise<Tracking> }

// deliveryFor.ts
export const deliveryFor = (shipment: Shipment): Delivery => {
  switch (shipment.method) {
    case "courier":
      return courierDelivery({ rates: courierApi.rates, track: courierApi.track }, shipment);
    // ...one case per method
  }
};

// anywhere downstream
const delivery = deliveryFor(order.shipment);
await delivery.quote();
```

**Kind known at the call site:** no picker. Each caller calls its own operation.

```tsx
<MenuItem onClick={() => exportPdf(doc)}>PDF</MenuItem>
<MenuItem onClick={() => exportDocx(doc)}>Word</MenuItem>
```

**Not this:** records looked up by kind.

```ts
const QUOTERS = { postal: postalQuote, courier: courierQuote }; // each imports its API
const total = await QUOTERS[order.deliveryMethod](order.shipment);
```

**Adding a kind:** extend the `Shipment` union, write `lockerQuoter.ts`, add one case to
`quoterFor`. The compiler points at the missing case.

## Explicit variants

**Use when** a kind changes which data is required or which outcomes exist. A bag of
optional fields would allow impossible combinations.

```ts
type Destination =
  | { kind: "email"; address: string }
  | { kind: "sms"; phoneNumber: string };

const destinationLabel = (destination: Destination): string => {
  switch (destination.kind) {
    case "email":
      return destination.address;
    case "sms":
      return destination.phoneNumber;
  }
};
```

**Not this:**

```ts
interface Destination { kind: "email" | "sms"; address?: string; phoneNumber?: string }
```

**Simpler:** a boolean for a real yes/no fact; an optional field that is optional for every
kind.

**Adding a kind:** add `{ kind: "webhook"; url: string }`. Every exhaustive `switch` fails to
compile until it handles it.

## State and transitions

**Use when** allowed actions depend on what happened before, and separate flags
(`isRunning`, `isCancelled`) could contradict each other.

```ts
type Job =
  | { status: "ready" }
  | { status: "running"; requestId: string }
  | { status: "done"; result: Report }
  | { status: "cancelled" };

type JobEvent =
  | { type: "start"; requestId: string }
  | { type: "stop" }
  | { type: "finished"; requestId: string; result: Report };

export const nextJob = (job: Job, event: JobEvent): Job => {
  if (job.status === "ready" && event.type === "start") {
    return { status: "running", requestId: event.requestId };
  }
  if (job.status !== "running") return job;
  if (event.type === "stop") return { status: "cancelled" };
  if (event.type === "finished" && event.requestId === job.requestId) {
    return { status: "done", result: event.result };
  }
  return job;
};
```

One function owns every legal move. The request id stops a late result from overwriting a
newer run. Unlisted events leave the state unchanged, on purpose.

**Adding a state:** add `retrying` and its transitions to `nextJob`. Buttons ask the state
what is allowed instead of reading new flags.

## Result union

**Use when** a caller must handle several expected outcomes. Return them once, instead of
passing `null` or `undefined` down a call chain.

```ts
type FindOrder =
  | { kind: "found"; order: Order }
  | { kind: "notFound" }
  | { kind: "forbidden" };

const orderPageData = async (id: OrderId, viewer: Viewer): Promise<OrderPage> => {
  const result = await orders.find(id, viewer);
  switch (result.kind) {
    case "found":
      return toOrderPage(result.order);
    case "notFound":
      throw new OrderNotFoundError(id);
    case "forbidden":
      throw new ForbiddenError();
  }
};
```

**Simpler:** throw directly when no caller does anything except fail.

## Adapter

**Use when** a vendor's names, payloads or error codes leak into your code. Define the
operation you need; the adapter translates to and from the vendor.

```ts
// messageSender.ts
export type SendResult = { kind: "sent"; receiptId: string } | { kind: "rejected"; reason: string };
export interface MessageSender { send: (message: Message) => Promise<SendResult> }

// vendorMessageSender.ts
export const vendorMessageSender = (client: VendorClient): MessageSender => ({
  send: async (message) => {
    const response = await client.push({ to: message.to, body: message.text });
    if (response.status !== "ACCEPTED") return { kind: "rejected", reason: response.errorText };
    return { kind: "sent", receiptId: response.id };
  },
});
```

Callers use `MessageSender` and never see `ACCEPTED` or `errorText`.

**Simpler:** keep the existing integration function when it already returns your types.

## Facade

**Use when** callers keep repeating the same steps against a subsystem. One operation owns
the steps; callers pass domain inputs.

```ts
export const buildStatement = async (deps: StatementDeps, account: Account, period: Period) => {
  const entries = await deps.ledger.entries(account.id, period);
  const totals = statementTotals(entries);
  return renderStatement(account, period, entries, totals);
};
```

**Not this:** a function that forwards its arguments to one call. That is a hollow wrapper.

## Pipeline

**Use when** work is an ordered set of transformations, each with clear input and output.

```ts
export const importRows = (input: string) => {
  const parsed = parseRows(input);
  const normalized = normalizeRows(parsed);
  const valid = validateRows(normalized);
  return formatRows(valid);
};
```

These plain calls are usually the whole pipeline. Build a list of steps only when the
order itself changes at runtime. Steps that write or call out stay visible in the
orchestrator, and no step gets a flag to fit a new caller.

## Picker

**Use when** an implementation must be chosen and built from real services: that is the
picker in [Strategy](#strategy). It is the only place factories meet real services.

**Simpler:** a strategy with nothing to inject is a plain object returned by its case; a
strategy that would only forward to one service method is an object with that method,
returned by its case, with no factory file.

**Not this:** a container that hands out any dependency by name, a string registry, or a
module-level record of built strategies.

## UI composition

**Use when** mode props make one component choose between layouts, actions or state.

```tsx
// Not this
<ProfileCard mode="edit" showActions />

// This: one component per variant, composed from shared parts
<EditableProfileCard profile={profile} onSave={saveProfile} />
<ReadOnlyProfileCard profile={profile} />

// Each variant composes the same pieces
export const EditableProfileCard = ({ profile, onSave }: EditableProfileCardProps) => {
  const draft = useProfileDraft(profile);
  return (
    <Card>
      <ProfileFields value={draft.value} onChange={draft.update} />
      <SaveActions status={draft.status} onSave={() => onSave(draft.value)} />
    </Card>
  );
};
```

Use `children` or named slots when callers only supply content. One component owns the
draft; children get a value and an action. A choice of markup is a component with early
returns, never a ternary in the markup.

**Simpler:** `disabled`, `loading` and `selected` are facts, not modes, and stay props.
