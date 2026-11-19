# Lean code: Swift

The rules are in [SKILL.md](../SKILL.md). This file shows how to write them in Swift 6 with
SwiftUI. Which structure fits which problem: [patterns.md](patterns.md).

## Types

- Kinds are enums. Variants are enums with associated values:
  `enum Result { case ok(Document), notFound }`.
- An exhaustive `switch` has no `default`, so a new case fails to compile. A `default` that
  returns a fallback hides the next kind.
- Data per kind that never changes is a `switch` inside the one function that owns it, or a
  static value on the enum. No dictionary keyed by kind holding behaviour.
- No `!`, no `as!`, no `try!`, no `as` casts. `as?` is allowed once, at a framework edge that
  returns `Any` (`UserDefaults`, `JSONSerialization`): turn the value into a typed one right
  there and use the type everywhere else. Prefer `Codable` over `Any`.
- `guard let first = list.first` instead of `list.first!`, even when the list is checked
  above: the guard makes the check part of the code.
- A mode is an enum, never a `Bool`. `isOn`, `isPaused`, `isLoading` state a fact about
  data and are fine; `ghost: true`, `far: Bool` and `usesAgent: Bool` pick behaviour and are
  not. Use `.ghost` / `.solid`, or two named functions.
- Name the empty state with a protocol (`protocol StartsEmpty { init() }`) when many
  generic helpers need "nothing stored yet". Don't pass an `empty:` value everywhere.
- Mark types `Sendable` and check the package builds in the Swift 6 language mode. Use
  `@unchecked Sendable` only on a class that guards its state with a lock, and keep that
  class small and tested.
- Errors are enums conforming to `Error`, one case per failure mode. `try?` is allowed only
  inside a type whose job is to swallow that failure (a file store that falls back to
  empty), and that type has a test for the failure.

## Interfaces and objects

- A protocol states what callers need, nothing more. Take `any Store<Pitch>` (primary
  associated types) or a closure, not the concrete class.
- One protocol per capability, not per class. Several types sharing a shape are one generic
  type, not copies. If two stores differ only in the value they hold and the file name, they
  are one `FileStore<Value>`.
- Use a `struct` for values and plain functions; use a `final class` only for identity,
  shared mutable state or `@Observable` models. Prefer composition to subclassing; there is
  almost never a reason to subclass your own type.
- Put behaviour on the type that owns the data (`MotionPose.farTucked(_:_:)`), and
  extensions by topic (`Type+Topic.swift`) when a type grows past one file.
- Strategies: a protocol or function type per behaviour, one file per kind, one picker
  function with an exhaustive `switch` that builds them. See [patterns.md](patterns.md).

## Branching and ternaries

- A ternary picks between two values. No nested ternary, and no ternary inside a string
  interpolation that holds another ternary. Name the choice:

  ```swift
  private func status(recent: Double, usual: Double, margin: Double) -> BaselineStatus {
      if recent < usual - margin { return .below }
      if recent > usual + margin { return .above }
      return .within
  }
  ```

- Do the same sentence once. A phrase built by a ternary in two places is one function.
- Prefer `switch` over `if/else if` chains when the input is an enum or a range.
- No `else` after `return`.

## SwiftUI

- One `View` per file, named after the file. Split by presentation, interaction, rule or state
  lifetime, not into arbitrary slices of markup.
- Styling that several views repeat is a `ViewModifier` with a `View` extension, with a named
  enum for its sizes (`.chipStyle(isOn:layout:)`), not the same five modifiers pasted.
- A view whose body is one element with fixed modifiers is a hollow wrapper: inline it or make
  a modifier.
- State has one owner. Derive from `@Binding`, `@Observable` and the environment; never copy
  a prop into `@State` and sync it. Reset a sheet by recreating it, not by assigning values
  back.
- `@Observable` models own one fact and persist through one store type; they take the
  store's location as an init parameter so previews and tests pass `nil` or a temp URL.
- `@ViewBuilder` branches by early `if`/`switch` at the top of a small view, not nested
  ternaries that return markup.
- Accessibility is part of the view: labels, traits, Dynamic Type, Reduce Motion.

## Files and modules

- A Swift module is the index. No barrel files and no `Index.swift`. A package's public
  surface is the `public` declarations; keep everything else internal.
- One type per file, named after it. `Pitch.swift` holds `Pitch`; `Pitch+Edit.swift` holds
  its extension.
- Folders are domains (`Motion/`, `Storage/`), not kinds (`Helpers/`, `Extensions/`).
- Under 400 lines, in practice well under 200.
- A target that lists source files by path in the project file (a watch app sharing
  one view) lists every file that view needs; adding a type it uses means adding its file
  there.

## Persistence

- Reading and writing a `Codable` file is one type (`JSONFile<Value>`), not `Data(contentsOf:)`
  and `JSONDecoder()` pasted into every store. A store holds one `JSONFile` and the rules for
  its own value.
- Store and model initialisers take the URL, optional, so a test or preview passes `nil`.

## Comments and docs

`///` one line, and only when the name does not say it. No `// MARK:` banners, no commented-out
code, no `// TODO`. Tests: the test name carries the intent.

## Checks

Build every target that compiles the files you touched (app, watch, widgets, tools), run the
package tests, and run the changed screen on a simulator: tap the controls you changed and
compare with the build before your change when the work is a refactor. A refactor that keeps
output identical is checked by the same screens and the same stored files, not by reading
the diff.
