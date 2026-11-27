# Lean code: TypeScript and JavaScript

The rules are in [SKILL.md](../SKILL.md). This file shows how to write them in TypeScript, with
notes for plain JavaScript at the end.

## Types

- Object shapes are `interface`; unions, function types and computed types are `type`.
  Extend an object shape with `interface B extends A`, not `A & {...}`.
- Kinds are string-literal unions: `type Format = "pdf" | "docx"`.
- Variants are discriminated unions: `{ kind: "ok"; value: T } | { kind: "notFound" }`.
- A record of values per kind uses `satisfies Record<Kind, Value>`, so a missing kind fails to
  compile.
- An exhaustive `switch` returns a value from every case inside a function whose declared
  return type has no `undefined` or `void`; with `strictNullChecks` on, a missing case then
  fails to compile. Otherwise
  end it with `default: throw new UnhandledKindError(value satisfies never)`, using the project's
  error class for this or a one-line class in the domain's errors file. No hand-rolled
  `assertNever` helper.
- No `as` casts (`as const` is fine), no non-null `!`, no `any`.
- Replace a cast with narrowing: a discriminant check, `in`, or a schema parse. A type guard
  (`(x): x is Foo`) only wraps a real runtime check; a guard that checks nothing is a cast.
  No assertion functions (`asserts x is Foo`).
- Derive a type from its source instead of retyping its fields: `typeof`, `Pick`, `Omit`,
  or the schema's inferred type.
- A function we own returns a named type: declare the type, then annotate the function with
  it. Use `ReturnType` only for a library's function whose result type isn't exported.
- Arrays and records a function takes and doesn't change are `readonly`
  (`readonly Item[]`, `Readonly<Record<Kind, Label>>`).
- No `@ts-ignore`, `@ts-expect-error` or `*-disable` lint comments.
- Untrusted input is parsed with the project's schema library at the boundary; inside,
  trust the type.
- `x ?? fallback` for defaults. `x || fallback` only when every falsy value means "missing".
- Strategy dependencies are narrow function types or `Pick<typeof service, "method">`.
- A one-method strategy interface may be a function type; then a picker case can return a
  service method directly.
- No boolean option fields that pick behaviour (`{ strict: true }`): use a union field.

## Keep types useful

- Plain types, discriminated unions, inference and `satisfies` over clever generics.
- A helper type (mapped, conditional, template literal) is fine when it ties caller input
  to a real type: `Path<Config>` for dotted keys, `DeepReadonly<Settings>` for frozen
  config, keys of `Row` whose values are numbers. It is not fine when it mirrors a runtime
  choice (`K extends "pdf" ? PdfHandler : ...`): that is a strategy. Each helper type gets a
  type-level test.
- Reuse the project's id types and schemas. Don't start a parallel branded-type set.
- Narrow `unknown` input explicitly. A type annotation doesn't validate network or storage
  data.
- Test a type that callers rely on with type-level checks: the test framework's
  `expectTypeOf`, or an `Equal<A, B>` check that fails to compile. These are types, not
  assertions, and the no-assert rules don't apply to them.

## Typed data, not strings

We wrote the value, so read its field. Don't take a string apart with a regex.

```ts
// before: the label is parsed again to learn what we put into it
const isReturn = /^parcel:[^:]+:[^:]+:return$/.test(label);
const carrierId = label.split(":")[1];

// after: the record has the fields; the string exists only where it is stored
const isReturn = ref.direction === "return";
const carrierId = ref.carrierId;
```

A key with parts is a nested map, not a joined string:

```ts
// before
stock.set(`${warehouseId}:${sku}`, level);
const level = stock.get(`${warehouseId}:${sku}`);

// after
const stockByWarehouse = new Map<WarehouseId, Map<Sku, StockLevel>>();
const level = stockByWarehouse.get(warehouseId)?.get(sku);
```

The platform parses what it knows: `extname(fileName)` over `/\.([a-z0-9]+)$/i`, `new URL(link)`
over a host regex. When a stored string must stay a string, one codec file builds and reads
it, accepts the old shape, writes the new one, and has a round-trip test.

## Errors and modules

- One error class or code per failure mode, following the project's convention.
- Catch the specific error. Never an empty `catch`. Rethrow with `{ cause: error }` so the
  cause stays attached.
- No work at import time beyond definitions: no network, files or global state changes.
- Strategies are built in their picker, not at module level.

## Branching

Fine: data mapping in one exhaustive `switch`.

```ts
const formatLabel = (format: Format): string => {
  switch (format) {
    case "pdf":
      return "PDF";
    case "docx":
      return "Word";
  }
};
```

Fine: reading a result union, including a case that throws. Put it in a function with a
declared return type, so a missing case still fails to compile.

```ts
const loadDocument = async (id: DocumentId): Promise<Document> => {
  const result = await documentService.find(id);
  switch (result.kind) {
    case "ok":
      return result.document;
    case "notFound":
      throw new DocumentNotFoundError(id);
  }
};
```

Not fine: cases that call services. That is a strategy written inline.

```ts
switch (format) {
  case "pdf":
    return pdfRenderer.render(ctx, doc);
  case "docx":
    return docxWriter.write(ctx, doc);
}
```

Write the strategy and its picker instead: see [patterns.md](patterns.md#strategy). A
larger example with two kinds at once: [strategy-example.md](strategy-example.md).

Fine: a picker. Its cases only build a strategy, or only hand the narrowed value to a
strategy's method:

```ts
const notifierFor = (destination: Destination): Notifier => {
  switch (destination.kind) {
    case "email":
      return emailNotifier({ sendMail: smtp.send }, destination.address);
    case "sms":
      return smsNotifier({ sendSms: smsApi.send }, destination.phoneNumber);
  }
};
```

## Early returns and one pass

Pick a return value with guard clauses, not a ternary:

```ts
const trackingNumberOf = (shipment: Shipment) => {
  if (shipment.status !== "shipped") return null;
  return shipment.trackingNumber;
};
```

Filter a list once with one named predicate:

```ts
const isRefundable = (order: Order) => order.status === "delivered" && !order.refundedAt;

return orders.filter(isRefundable).map((order) => order.id);
```

## React

Editing a component or hook: read [react.md](react.md).

## Files

- `index.ts` / `index.js` holds only `export ... from "./x"` lines.
- `loadSnapshot.ts` exports `loadSnapshot`. `Results.tsx` exports `Results`.
- Import from the folder's index, one line per module:

  ```ts
  // Not this
  import { orderPricing } from "../../orders/orderPricing";
  import { orderRefunds } from "../../orders/orderRefunds";
  import { orderShipping } from "../../orders/orderShipping";

  // This
  import { orderPricing, orderRefunds, orderShipping } from "../../orders";
  ```

- Inside `orders/`, import `./orderRefunds` directly, never `"."` or `"./index"`.
- A flat folder of one module per table or client stays a file import:
  `import { ordersStore } from "../stores/ordersStore"`, never `from "../stores"`.

## Docstrings

```ts
/** Latest permission snapshot for an org. */
```

No `@param` or `@returns`.

## Tests

- Table cases use `it.each` with literal expected values. See [../testing.md](../testing.md).

## Plain JavaScript

Without a type checker, the rules stay and the checks move to runtime and tests:

- Kinds are a frozen object of string constants, with a JSDoc `@typedef` for the union.
- An exhaustive `switch` ends in a `default` that throws, because nothing else catches a
  missing case.
- The picker's `default` throws, and one test calls the picker with every kind.
