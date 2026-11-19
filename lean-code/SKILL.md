---
name: lean-code
description: How I want code written in TypeScript, JavaScript, Python, Swift and Rust. SOLID, small focused files, almost no comments, one-line docstrings, no boolean blindness, strategies chosen by one picker function instead of lookup tables, simple ternaries only, barrel-only index files. Must always apply when writing or editing code, no trigger needed.
---

# Lean code

Applies to every file you write or edit. This file holds the rules. The language folders
show how to write them:

| Language | Guide (read before the first edit) | Patterns (read when picking a structure) |
| --- | --- | --- |
| TypeScript and JavaScript | [typescript/guide.md](typescript/guide.md), plus [typescript/react.md](typescript/react.md) for components and hooks | [typescript/patterns.md](typescript/patterns.md) |
| Python | [python/guide.md](python/guide.md) | [python/patterns.md](python/patterns.md) |
| Swift and SwiftUI | [swift/guide.md](swift/guide.md) | [swift/patterns.md](swift/patterns.md) |
| Rust | [rust/guide.md](rust/guide.md) | [rust/patterns.md](rust/patterns.md) |

For reviewing a diff or acting on an analyzer finding, read [smells.md](smells.md). For a bug
fix, read [bug-fixes.md](bug-fixes.md) first. Repo rules in AGENTS.md win where they
conflict.

## Design

- **Single responsibility.** A function does one thing, a file owns one responsibility, a
  folder owns one domain. If the name needs "and", split it.
- **Open/closed.** A new kind adds a variant and its strategy. Existing code does not gain
  another `if`. The type checker fails until every kind is handled.
- **Narrow interfaces.** Take the fields and functions you use, not the whole record or
  service. No option bags full of flags most callers ignore. An interface is everything a
  caller must know: the types, and also ordering rules, error modes, required config and
  cost. Keep that list short.
- **Substitutable.** Every implementation of an interface honours its results, errors and
  side effects. No unsupported or no-op methods to fill an interface: split the interface.
- **Dependencies point one way.** Layers call down only. A domain uses another domain's
  public operation, never its internals or its tables.
- **Policy apart from effects.** Calculations take values and return values; they do not
  fetch, write or schedule. The function that runs the use case owns the order of reads,
  decisions and writes, and keeps writes that must be atomic in one transaction.
- **No reads in a loop.** One query or request per item of a list is an N+1. Read in bulk
  (by ids, by one index range, one batch call) before the loop, then look items up in memory.
  The same holds for a list in the UI that fetches once per row. Read each fact once per
  flow: pass the loaded value down instead of fetching it again. Every list a function
  reads or returns has a bound: paginate or cap it.
- **One owner per fact.** Each piece of state and each transition has one owner. Children
  get a value and an action, never a synchronized copy.
- **Separate models for different meanings.** A shipping address and an invoice address
  stay separate types even when their fields match today.
- **One owner per decision.** Two copies of the same decision get one owner. Code that only
  looks alike but encodes different policies stays separate.
- **Clean code.** Names say what the thing is: no `data`, `result` or `temp`, no
  abbreviations, the unit in the name (`timeoutMs`), a boolean that reads as a question
  (`isOverdue`). Small functions at one level of abstraction.
  No hidden side effects in getters or `is`/`has` functions. No dead code.
- **No special cases for one caller.** Standardize so every caller reads the same thing.
- **Testable through the public interface.** Accept dependencies instead of creating them;
  return results instead of mutating inputs. Tests call what callers call, check results
  the caller can see, and survive internal refactors. When small modules merge into one,
  delete the old tests on their internals and test the merged module.
- **Test each dependency by its kind.** Pure code is tested through its public function,
  without fakes. A database or file system uses a local stand-in. Your own remote service
  gets an in-memory fake behind its interface. A third-party API gets an injected mock.
  Strategy factories get fake functions; the store behind those functions is tested
  against the local stand-in.
- **Never simplify away** validation at trust boundaries, error handling that prevents data
  loss, security checks or accessibility basics.
- **Tests live in test files.** No `demo()` functions or assert-based self-checks in
  source files. Type-level checks that fail to compile are types, not assertions, and are
  fine.

## Files

- **One responsibility per file.** Name the file after what it exports. If you can't name it
  after one thing, it's two files.
- **Extract when a piece has its own rules, dependencies, state lifetime or reason to
  change.** Small focused files beat one big file. I navigate by filename.
- **Not too small.** A small file that only one neighbour uses, and that changes only when
  that neighbour changes, goes back into it. A small file stays separate when it has its own
  callers or its own reason to change, because it will be extended. Type files, index files
  and one strategy per kind always stay separate. A file that only forwards is a hollow
  wrapper (Variants, rule 3).
- **Split by domain, not by kind.** `billing/invoicing/` beats a global `helpers/`. A
  `utils`/`helpers` file is fine at 3–4 functions, next to the domain it serves.
- **Generic code lives in the shared module.** A helper or component with nothing
  feature-specific in it goes where every feature can find it, not inside the first feature
  that needed it. Provider-specific details stay in that provider's module. Shared code
  uses only generic terms: no market, provider, locale or page names. The caller passes a
  value or a strategy.
- **Under 400 lines.** When a file grows past that, split it. Don't reorganize it into
  sections with comment headers.
- **Index and package files only re-export.** No types, constants, functions, side effects
  or wrappers in them. New folder: write the real files first, then the re-exports. When
  splitting a fat index, leave re-exports behind so importers keep working.
- **Import from the module, not its files.** Code outside a folder imports from that
  folder's index, in one import line, so a reader sees which module it depends on. If the
  folder has no index yet, add one that re-exports what you import. Inside the folder,
  import siblings by file, never through the folder's own index: that makes an import cycle.
  Exception: a flat folder of independent modules, one per table or client (stores,
  repositories), is imported by file. An index over it puts every module it names into each
  bundle that imports from it.

## Readability

- **No boolean blindness.** No boolean parameter that picks behaviour. Use a named union or
  separate named functions. A boolean that states a fact about data (`disabled`, `loading`,
  `selected`, `archived`) is data, not a mode, and is fine.
- **Simple ternaries only.** A ternary picks between two values. Both sides are values or
  pure calls: no side effects, no `await`, no nesting, no markup. Anything else becomes a
  named function with early returns.
- **Readable in one pass.** A reader follows a function top to bottom without going back.
  Readable steps beat one line. Use `map`/`filter` over a reduce that builds an object. A loop that only pushes or sets
  becomes `map`/`filter`.
- **At most two conditions in one expression.** A third `&&`/`||` gets a named predicate
  (`isOverdue(invoice)`) or a derived union. Never mix `&&` and `||` without naming the parts.
- **At most two levels of nesting.** Deeper code becomes guard clauses or a named function.
- **Values are assigned once.** No variable reassigned across branches or loop steps to build
  a result. Compute it in a named function that returns it, or with `map`/`filter`.
- **No `else` after `return` or `throw`.**
- **Keep config at the call site.** Don't move a one-use value (a guard, a limit) into a
  constant far from where it is used. A value that describes a domain thing (whether a
  pipeline sends alerts, a provider's default limit) lives on that thing's definition, not
  as a loose constant next to one caller.
- **Never add lint or type suppressions, or casts.** Restructure the code. If it's a real
  false positive, ask first.
- **No error handling for cases that can't happen.** If the types or an earlier guard rule
  a case out, don't check it again. Exhaustiveness checks are not error handling: keep
  them.
- **Errors are specific.** One typed error per failure mode, and each mode is handled by
  its type. No generic "failed". No silent fallback where the data must exist (`?? ""`,
  `catch { return [] }`): throw the typed error.
- **Derive state, don't patch it.** No latches, no reassigned internal values. Reset a form
  or dialog by recreating it, not by setting its values back one by one.

## Variants: decide once, never ask again

Code that handles several kinds of a thing (payment methods, export formats, user roles,
shapes) drifts into the same seven smells.

1. **One decision, then strategies.** Spot: a kind compared to a literal, a file-extension
   check, "which optional field is set" used as the kind, or an early return that picks a
   whole variant before the shared code runs, when the same choice repeats across files. One
   component that picks its markup with early returns is fine. Fix: one function
   derives the kind once. Everything that differs per kind lives on that kind's strategy
   (its controls, its handler, its viewer). Shared code calls the part it is given and never
   asks which kind it is in. A record keyed by kind holds data only (labels, constants). Behaviour per
   kind is chosen by one picker function (see "Strategies").
2. **Modes are named unions, not encodings.** Spot: several booleans that together pick a
   mode (`isPreview, isDraft, canSave`), a direction as `1 | -1`, a free mode string passed
   through layers, or a missing value meaning "the other mode". Fix: one named union value,
   or separate named functions per mode. The function that derives a kind takes no boolean
   parameters. It may read a stored boolean field of the data it classifies.
3. **No hollow wrappers.** Spot: a component whose body is one element plus a fixed prop; a
   function that only forwards its arguments; a context or hook made only to avoid passing
   one prop; `isNotX` that negates `isX`; a curried factory that adds nothing; a no-op
   callback to satisfy a framework; a module that only renames a library call; a parameter
   the function ignores so it fits a generic runner. Fix: inline it, call the library
   directly, or build the shared part once where the variants are composed. An extraction
   must own logic. A named function that holds a branch with early returns owns logic.
   Test: imagine deleting it. If the complexity vanishes, it was a pass-through. If it
   reappears in every caller, it earns its place.
4. **One interface per kind.** Spot: one component or argument type carrying every kind's
   fields; optional fields only one kind sets; callers inventing dummy values to satisfy a
   type. Fix: each kind gets its own type, composed from a small shared part.
5. **Data stays typed, not encoded in strings.** Spot: ids parsed by prefix, types detected
   from rendered text or file names, JSON stored in a field that otherwise holds plain text,
   variants told apart by "does this key exist". Fix: a tagged union, or one codec per id
   family that builds the value and reads it back as a typed result (stored text unchanged,
   with a round-trip property test). Detection by name happens in that codec only.
6. **One copy, one path.** Spot: a helper re-exported from a second module, a second copy of
   a small utility, the same constant in several files, a second sort order or search that
   must agree with the first, a table translating between two equivalent code sets, blocks
   that differ only in a value, a wrapper or setup step that every method repeats. Fix: one
   owner, imported directly; return the final value instead of mapping it afterwards; one
   mechanism that takes the value; the repeated step applied once at the shared entry.
7. **No API kept alive by tests.** Spot: an export, option or parameter that only a test
   calls. Fix: delete it, or add it with its first real caller.

Apply these to every line you write. Smells that were already there are flagged, not fixed
(see Scope).

## Branching

All rules for `if`, `switch`, `match` and ternaries are here. Decide by what the branch does:

| The branch | Write |
| --- | --- |
| Rejects bad or empty input | Flat guard clauses at the top, each returning or throwing. Several are fine. For untrusted input, parse it at the boundary into a typed value, then trust the type |
| Maps a kind to data (a label, a constant, a validator, an error code) | One exhaustive `switch`/`match` inside the one function that owns that mapping, or a typed record of values |
| Handles a result union (`ok`, `notFound`, `conflict`) | One exhaustive `switch`/`match` at the caller. A case may return a value or throw: that is reading a result, not picking behaviour |
| Picks behaviour by kind, and any case calls a service, does IO or holds rules | A **strategy chosen by one picker function**, even with two kinds in one place |
| Picks behaviour by one kind inside another (access kind × resource kind) | Methods on the strategy, one per kind of the other dimension |
| Combines several flags to pick a path | Derive one **named union** once, then the rows above |
| Maps a number to a band with an if-ladder | An ordered list of `{ below, label }` thresholds and one `find`. It is data, not a lookup of functions |
| Checks ownership or permission in a service | **Scope the read** by owner, or use the authorization wrapper on the endpoint |
| Guards a side effect whose intent holds either way | Do it **unconditionally**, in the right order (record the user's choice, then act) |
| Checks status flags in many places | **One state value plus one transition function** that owns the legal moves |
| Repeats null checks down a call chain | **Return a typed result union once**; callers use the result-union row |
| Runs a long sequence of steps | A **pipeline** of named functions called in order |
| Switches markup by mode | **Separate variant components**, or early returns at the top of the component |
| Does one of two actions on a yes/no fact about one thing (archived or not), in one place | A plain `if`/`else` is fine. When a second place decides the same fact, derive a union and use a strategy. If the two branches call different services or hold different rules, it is a kind, even when stored as a boolean: use the strategy row |
| Picks between two values with pure expressions | One simple ternary (see Readability) |

The line between a fine `switch` and a strategy: a `switch` whose cases only return data or
throw is fine. So is a picker: a `switch` whose cases only build a strategy, or only hand the
narrowed value to a strategy's method. A `switch` whose cases call services, read the
database or hold business rules is a strategy written inline. In new code, write the
strategy. In existing code, flag it (see Scope).

If a reviewer asks "what is this random if?", the answer is one of the rows above.

### Strategies

A strategy has three parts:

1. **One interface** for what every kind does (`Quoter` with `quote(shipment)`).
2. **One factory per kind, one file each**, taking only the functions it calls, never a whole
   service or the database. Tests build it with fakes. Small per-request builders of a
   second dimension may share one file.
3. **One picker function**, the only place that maps every kind to its behaviour. It is one
   exhaustive `switch` over the kind whose cases only build that kind's strategy from the
   real services, or only hand the narrowed value to a strategy's method. It may build
   shared dependencies before the `switch`. A missing kind fails the type check. Inside each case the value is already
   narrowed to its kind, so a strategy can take its own kind's fields.

Call the picker at the entry point (the endpoint, handler or job). The use case takes the
strategy as a parameter, so its tests pass fakes, and code below it never asks which kind
it is. The picker gets real clients from where the project already creates its services and
creates nothing costly itself.

Pick the form that fits:

- **Picker returning one behaviour** (the default): `quoterFor(shipment).quote()`.
- **Picker returning an object with several behaviours**, when a kind has more than one
  behaviour used in many places: `const delivery = deliveryFor(shipment)`, then
  `delivery.quote()` and `delivery.track()`.
- **No picker**, when the caller already knows the kind because it comes from which button
  or endpoint was used, not from stored data: each caller calls its own operation.

When the kinds need different input, they are variants first (rule 4), and each case of the
picker passes the narrowed variant to its kind's factory. Don't force unlike data through
one signature.

A kind with nothing to inject gets a plain object that meets the interface, returned by its
picker case. A factory that would only forward to one service method is a hollow wrapper:
the picker case returns an object whose method makes that one call, with no rule around it.
Building that object doesn't call the service, so the case still only builds.

**Not a lookup table.** A record of functions looked up with a runtime kind value
(`handlers[kind](...)`) is banned. A strategy object with one method per kind of a second
dimension (`access.folder(reads)`) is not a lookup: the picker for that dimension calls it. A record of built strategies looked up by kind is
replaced by the picker: the `switch` reads top to bottom, narrows each kind and needs no
mapped types.

Use a strategy even with two kinds. The next kind then costs one file and one picker case,
and the shape shows the next reader where it goes. Skip it only for a single kind and for
pure data mapping.

## Scope

- **Read the whole flow first.** Trace every file the change touches before choosing a
  shape. A small change in the wrong place is a second bug.
- **Reuse before writing.** Before writing code, check in order: something already in the
  codebase, the standard library, a platform feature, an installed dependency. No new
  dependency for what a few lines do. No near-duplicates. Reuse covers data too: a table,
  field or result the code already reads beats a new query or a parallel table.
- **Build it shared when reuse is real or expected.** When a second caller exists, or one is
  expected (a sibling page, the next market, a planned feature), build the shared component
  or mechanism. Look in the shared module first, and extend an existing component with a
  variant before writing a new one. Speculative generality is flexibility for uses nobody
  expects.
- **Correct over clever.** Two standard options of the same size: take the one that is
  correct on edge cases.
- **Flag existing smells, don't refactor them unasked.** When the task touches code that
  breaks these rules (an inline `switch` that should be a strategy, a boolean mode, a
  lookup table), make the change in the existing shape. Then suggest the refactor in one
  line: what it is, which files it touches, and why. Do the refactor only when I say so.
  Example: "Added `xlsx` as a case in `exportDocument.ts`. Suggest: turn this `switch` into
  exporter strategies (3 new files plus an `exporterFor` picker), so the next format is one
  file."
- **Don't move, re-indent or rename code just to tidy it.** Moving code so a new caller can
  reuse it is part of the change, not churn: extract the piece into its own file, move it as it
  is, and update every importer to the new path. Leave no re-export behind, except in an
  index file. A second copy is the smell, not
  the move.
- **Clean up only your own leftovers.** Remove imports, variables and functions your change
  made unused. Dead code that was already there is flagged, not deleted.
- **Don't change behaviour beyond the ask.** Flag anything behaviour-adjacent.
- **A bug fix's sibling callers are in scope.** Fixing the shared function changes them too;
  see [bug-fixes.md](bug-fixes.md).
- **Existing tiny or oversized files are flagged, not merged or split**, unless the task
  is about them.
- **In new code, the right shape beats the shortest diff.**
- **No speculative work.** No migrations, backfills, retroactive fixes or new limits unless
  the ask needs them. Say why in one line and wait.
- **Commit only what you touched for this task.** Stage by path. Never stage settings,
  MCP config, skills, docs or dependency files you didn't change.

## Comments: minimum

- **Default to zero.** Name the thing well instead. A good name beats any comment.
- A comment earns its place only by encoding a **why** that cannot be read off the code:
  a non-obvious ordering constraint, a workaround for someone else's bug, a magic number's
  source.
- **One line. Never a block.** No paragraphs, no banners, no section dividers.
- Never narrate what the next line does.
- Never explain history ("this used to iterate per document…"). That's the commit message.
- No TODOs and no notes about known limits or later upgrades unless I asked. A known limit
  goes in the PR body.
- Tests: the test name carries the intent. No comment.
- Don't comment out code. Delete it.
- Never use the words "seam" or "deep module" in names, comments or docs. Say "the
  interface" or "where the payment provider is plugged in".

## Docstrings: one line, no reasoning

- At most **one line**, and only on things whose purpose isn't obvious from the signature.
- States **what it is**, nothing else. No parameter or return sections: the types say it.
- **No intermediate decisions.** Don't record what you considered, rejected or traded off.
  That's PR-body material.
- Bad: "Loads the snapshot. We tried caching per org but a fresh read is safer since
  permissions change mid-session."
- Good: "Latest permission snapshot for an org."
- Better: nothing, if the function is called `loadOrgPermissionSnapshot`.

## Checks

Run the project's formatter, linter and focused tests on the files you touched, as the
project's own instructions name them. Report what ran and what it said. If something fails,
fix it before replying.

## Self-check before finishing

Reread your diff and delete:

- every comment that restates the code
- every docstring longer than one line, and every clause explaining a decision
- every boolean parameter, nested or impure ternary, suppression and cast you added
- every line that isn't part of the task: moves or renames made only to tidy, unrelated files
- any file that became a grab-bag (split it) or shrank to a fragment (merge it back)
- any logic that crept into an index or package file
- every variant smell from "Variants", and every branch that matches no row in "Branching"
- every expression with more than two conditions, nesting deeper than two levels, and
  variable reassigned to build a result
- every query or request inside a loop
