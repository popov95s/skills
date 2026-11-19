# Lean code: React

Read this before editing a component or hook. The rules are in [SKILL.md](../SKILL.md); the
TypeScript rules are in [guide.md](guide.md).

## Components

- One component per file. Components render, hooks own data, services own rules.
- Split components by presentation, interaction, rule or state lifetime, not into arbitrary
  slices of markup.
- A component that returns one element with fixed sizes, spacing or colours is a hollow
  wrapper. Keep it inline or use the design-system primitive.
- A hook owns one stateful interaction. It is not a place to park every helper.
- Use the APIs of the installed React version; don't assume newer ones. On React 19, `ref`
  is a plain prop (no `forwardRef`) and `use(Context)` replaces `useContext(Context)`.

## Choosing markup

- `{condition && <Element />}` is fine for showing or hiding one element. No ternary that
  picks between two pieces of markup, and no `&&` chains doing the same. Give the choice its
  own component with early returns:

  ```tsx
  const Results = ({ state }: { state: ResultsState }) => {
    if (state.kind === "loading") return <Spinner />;
    return <ResultList items={state.items} />;
  };
  ```

- Mode props become separate variant components (see [Variant components](#variant-components)).
  `disabled`, `loading` and `selected` props state a fact and are fine.
- `children` for structure. A render prop (`renderItem={({ item }) => ...}`) only when the
  parent passes data to what gets rendered; never `renderHeader`/`renderFooter` props for
  static parts.
- A component with several optional parts that callers arrange differently becomes
  [compound components](#compound-components), not `show*` and `render*` props.

## State

- One owner per piece of state. Don't copy props into state or sync copies with effects.
  Refs are fine for DOM access, timers and request identity.
- When components outside a component need its state (a preview, a submit button in a
  dialog footer), lift the state into a provider around both. Never sync it upward with an
  effect, and never read it from a ref on submit. Components need to be inside the
  provider, not inside the same markup.
- The provider is the only place that knows where state comes from (`useState`, a store,
  server sync). Parts read one context interface of `state`, `actions` and `meta`, and
  never call the store themselves. Swapping the provider swaps the source.
- Read a context through one hook that throws a specific error outside its provider. No
  `!` on the context value and no dummy default value.
- A context needs several consumers that really share state, not one prop to avoid passing.
- Never reset state with an effect that sets values back. Let it start fresh instead:
  - A dialog's form lives in its own component that renders only while the dialog is
    open (`{open && <RenameForm />}`, or dialog content that unmounts when closed). Every
    open starts from the initial values.
  - A component that stays mounted while its subject changes gets `key={subject.id}`.

## Variant components

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
draft; children get a value and an action.

## Compound components

**Use when** a component has several optional parts, callers arrange them differently, or
components outside it need its state.

```tsx
// Not this
<Composer showAttachments showEmojis renderFooter={() => <CustomFooter />} />
```

Each part is its own file in one folder. The context, its hook and every provider are files
too.

```tsx
// composer/composerContext.ts
export interface ComposerState { input: string; attachments: readonly Attachment[] }
export interface ComposerActions {
  update: (next: ComposerState) => void;
  submit: () => Promise<void>;
}
export interface ComposerMeta { inputRef: RefObject<HTMLTextAreaElement | null> }
export interface ComposerValue { state: ComposerState; actions: ComposerActions; meta: ComposerMeta }
export const ComposerContext = createContext<ComposerValue | null>(null);

// composer/useComposer.ts
export const useComposer = () => {
  const value = use(ComposerContext);
  if (!value) throw new ComposerOutsideProviderError();
  return value;
};

// composer/Input.tsx
export const Input = () => {
  const { state, actions, meta } = useComposer();
  return (
    <textarea
      ref={meta.inputRef}
      value={state.input}
      onChange={(event) => actions.update({ ...state, input: event.target.value })}
    />
  );
};

// composer/Submit.tsx
export const Submit = () => {
  const { actions } = useComposer();
  return <button onClick={actions.submit}>Send</button>;
};

// composer/index.ts
export * from "./Frame";
export * from "./Input";
export * from "./Submit";
export * from "./useComposer";
```

A provider per state source. This one keeps local state; a channel provider would read a
synced store and render the same context.

```tsx
// forwardMessage/ForwardComposerProvider.tsx
const emptyComposer: ComposerState = { input: "", attachments: [] };

export const ForwardComposerProvider = ({ children }: { children: ReactNode }) => {
  const [state, setState] = useState(emptyComposer);
  const forward = useForwardMessage();
  const inputRef = useRef<HTMLTextAreaElement>(null);
  const value = { state, actions: { update: setState, submit: () => forward(state) }, meta: { inputRef } };
  return <ComposerContext value={value}>{children}</ComposerContext>;
};
```

Callers compose the parts they need. Parts outside the frame still reach the state:

```tsx
import * as Composer from "../composer";

export const ForwardMessageDialog = () => (
  <ForwardComposerProvider>
    <Dialog>
      <Composer.Frame>
        <Composer.Input />
      </Composer.Frame>
      <MessagePreview />
      <DialogActions>
        <Composer.Submit />
      </DialogActions>
    </Dialog>
  </ForwardComposerProvider>
);
```

**Simpler:** a component with one layout and no outside readers takes plain props.

**Not this:** a `const Composer = { Input, Submit }` object in a file. Re-export the parts
from the folder's index and import the folder as a namespace.
