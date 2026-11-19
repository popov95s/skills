# Lean code: TypeScript and JavaScript

The rules are in [SKILL.md](../SKILL.md). This file shows how to write them in TypeScript, with
notes for plain JavaScript at the end.

## Types

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
- Mapped or conditional types only when they state a link callers rely on (a schema and its inferred type). Don't turn a plain runtime decision into a type puzzle.
- Reuse the project's id types and schemas. Don't start a parallel branded-type set.
- Narrow `unknown` input explicitly. A type annotation doesn't validate network or storage
  data.

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

## Ternaries

```ts
const noun = count === 1 ? "item" : "items";            // fine: values
const text = items.length ? formatList(items) : "—";    // fine: pure call
const user = id ? await loadUser(id) : null;            // not fine: await
const tone = isError ? "red" : isWarning ? "amber" : "grey"; // not fine: nested
```

Replace an impure or nested ternary with a named function with early returns:

```ts
const loadOptionalUser = async (id: UserId | undefined) => {
  if (!id) return null;
  return loadUser(id);
};
```

## React

- One component per file. Components render, hooks own data, services own rules.
- Split components by presentation, interaction, rule or state lifetime, not into arbitrary
  slices of markup.
- A component that returns one element with fixed sizes, spacing or colours is a hollow
  wrapper. Keep it inline or use the design-system primitive.
- A hook owns one stateful interaction. It is not a place to park every helper.
- `{condition && <Element />}` is fine for showing or hiding one element. No ternary that
  picks between two pieces of markup, and no `&&` chains doing the same. Give the choice its
  own component with early returns:

  ```tsx
  const Results = ({ state }: { state: ResultsState }) => {
    if (state.kind === "loading") return <Spinner />;
    return <ResultList items={state.items} />;
  };
  ```

- Mode props become separate variant components, or composition with `children` or named
  slots. `disabled`, `loading` and `selected` props state a fact and are fine.
- One owner per piece of state. Don't copy props into state or sync copies with effects.
  Refs are fine for DOM access, timers and request identity.
- Never reset state with an effect that sets values back. Let it start fresh instead:
  - A dialog's form lives in its own component that renders only while the dialog is
    open (`{open && <RenameForm />}`, or dialog content that unmounts when closed). Every
    open starts from the initial values.
  - A component that stays mounted while its subject changes gets `key={subject.id}`.
- A context needs several consumers that really share state, not one prop to avoid passing.
- Use the APIs of the installed React version; don't assume newer ones.

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

## Plain JavaScript

Without a type checker, the rules stay and the checks move to runtime and tests:

- Kinds are a frozen object of string constants, with a JSDoc `@typedef` for the union.
- An exhaustive `switch` ends in a `default` that throws, because nothing else catches a
  missing case.
- The picker's `default` throws, and one test calls the picker with every kind.

## Diff search

```
{ git diff -U0 "$(git merge-base <base> HEAD)" -- '*.ts' '*.tsx' '*.js' '*.jsx'
  git ls-files -o --exclude-standard -- '*.ts' '*.tsx' '*.js' '*.jsx' | xargs -I{} git diff -U0 --no-index /dev/null {}; } \
  | sed -E 's/ as const([^A-Za-z_]|$)/ \1/g' \
  | grep -E '^\+\+\+ |^\+.*(switch \(|kind (===|!==) "|type (===|!==) "|endsWith\("\.|startsWith\("[a-z-]+:"|\.slice\("[a-z-]+:"\.length|: boolean([;,)} ]|$)|= (true|false)[,)]|[(,] *_[a-zA-Z]+[,:)]|\(\) => \{\}|createContext|" in [a-z]|\bas [A-Za-z]|[a-zA-Z)\]]!(\.|;|\)|,)|: any\b|@ts-|-disable)' \
  | grep -vE '^\+ *import '
```

It covers committed, uncommitted and new untracked files. The `+++` lines name the file each
hit below them belongs to. It also flags `export { a as b }` lines, which index files may
have. It can't see nested or `await` ternaries, records keyed by kind or multi-line
signatures: reread the diff for those.
