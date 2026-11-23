# lean-code decisions

| Date | Decision | Why |
| --- | --- | --- |
| 2026-11-18 | Small focused files over fewest files; tiny files used by one neighbour merge back | Small named files are easier to navigate; 5–10 line fragments are not |
| 2026-11-18 | Behaviour per kind uses a strategy from two kinds on, chosen by one picker `switch` | Under-built code makes later agent changes go wrong; a picker reads top to bottom and the compiler catches a missing kind |
| 2026-11-18 | No lookup tables of functions; records keyed by kind hold data only | A lookup moves the `switch` but keeps each case tied to its services |
| 2026-11-18 | Existing smells are flagged with a suggested refactor, not refactored unasked | Keeps diffs reviewable; the refactor is a separate decision |
| 2026-11-18 | Shortest diff does not win over the right shape in new code | A small change in the wrong shape costs more later |
| 2026-11-18 | Ternaries only for two values or pure calls; markup choices get a component with early returns | Ternaries in markup and nested ternaries are hard to read |
| 2026-11-18 | No `ponytail:` comments, no assert self-checks in source, no dense one-liners | Known limits go in the PR body; tests live in test files; readability beats line count |
| 2026-11-18 | Never the words "seam" or "deep module" | Plain words only |
| 2026-11-18 | Import from a folder's index, except flat folders of stores or clients | One import line shows the module; an index over many stores bloats bundles |
| 2026-11-18 | No project-specific tool commands; each project names its own checks | The skill is shared across projects and languages |
| 2026-11-19 | Bug fixes live in `bug-fixes.md`, read only for a bug fix | It is a process, not a code rule, and most edits don't need it |
| 2026-11-19 | No diff-search commands in the guides | Pattern checks belong to separate tools; the skill states rules |
| 2026-11-19 | Concrete readability limits: two conditions per expression, two levels of nesting, assign once | "No dense expressions" was too vague and was the most repeated review ask |
| 2026-11-19 | No reads in a loop | N+1 reads were a repeated review ask with no rule |
| 2026-11-19 | From typescript-advanced-types: narrowing instead of casts, derived types, `readonly` inputs, type-level tests; not its tutorial on generic, conditional and mapped types | The rules fill gaps in the TypeScript guide; the tutorial adds type puzzles |
| 2026-11-19 | `interface` for object shapes and `extends` over `&`, as typescript-advanced-types says | Conflicting fields fail at the declaration, errors show the name, and extended interfaces check faster |
| 2026-11-19 | Helper types allowed when they tie caller input to a real type; type-level checks are not asserts; no assertion functions | Typed paths and frozen config catch real mistakes; assertion functions are runtime asserts |
| 2026-11-19 | Expected reuse is not speculative; generic code lives in the shared module; domain values live on their owner | Shared-component and placement asks repeated because "no speculative generality" pulled the other way |
| 2026-11-19 | React rules live in `typescript/react.md`, read only for components and hooks | Most TypeScript edits never touch React, and the section was becoming half the guide |
| 2026-11-19 | From vercel-composition-patterns: compound components, provider owns the state source, lift state, children over render props; context read through one hook that throws; parts re-exported from a folder index, not a `Composer = {...}` object | Fills gaps in the React rules; its null-context reads and object export clash with the no-`!` and index-only rules |
| 2026-11-19 | Eight small rules from addyosmani/agent-skills (generic terms in shared code, one mechanism, read once, naming, loops, threshold list, bounded lists, no silent fallback); left out its third-use rule, line budgets, branded ids and process steps | They close repeated review asks; the rest clashes with expected reuse, bans on numeric budgets and on parallel branded types |
| 2026-11-23 | Test rules live in `testing.md`, read only when a test is written, changed or deleted; from obra/superpowers: name the break, hand-derived expectations, no change detectors, no assertions on mocks; added: one level per behaviour, a thoroughness checklist, a one-condition flip for risky logic; left out: delete-and-restart TDD, 100% mutation kill, test pyramid ratios, AAA/Given-When-Then comments | Slop tests fill CI without catching breaks; the strict TDD ritual and comment patterns clash with the no-comments rule and are too heavy for every change |
| 2026-11-20 | No logic on text matching: no regex, string compare or fuzzy match on any text, ours or third-party; decide from typed fields or published codes, or extract with a model at the boundary with an eval | "A regex is fine for text we don't control" let a regex on a provider's town field pass review; text matching breaks silently on the next wording |
