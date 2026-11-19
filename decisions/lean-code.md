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
| 2026-11-19 | Expected reuse is not speculative; generic code lives in the shared module; domain values live on their owner | Shared-component and placement asks repeated because "no speculative generality" pulled the other way |
