# Skills

Personal agent skills. Each folder is one skill with a `SKILL.md` at its root.

| Skill | What it does |
| --- | --- |
| [lean-code](lean-code/SKILL.md) | How code is written in TypeScript, JavaScript, Python, Swift and Rust |
| [terse](terse/SKILL.md) | How replies are written: only the key points, hard length caps, tables for data |

## Install

The [skills CLI](https://github.com/vercel-labs/skills) installs from this repo into Claude
Code, Codex, Cursor and other agents. It needs Node.js 22.20 or later. The repo is private,
so the CLI uses your existing Git, GitHub CLI or SSH login.

For every project (user-wide):

```bash
bunx skills add popov95s/skills --skill lean-code --skill terse -g
```

For one project only, run it from the project root without `-g`:

```bash
bunx skills add popov95s/skills --skill lean-code
```

`npx` works the same way as `bunx`. Useful options:

| Option | Effect |
| --- | --- |
| `-a claude-code` | Install for one agent only (repeat for more) |
| `--list` | Show the skills in the repo without installing |
| `--copy` | Copy files instead of symlinking |
| `-y` | Skip the prompts |

Keep it current:

```bash
bunx skills check
```

```bash
bunx skills update lean-code
```

Remove it:

```bash
bunx skills remove lean-code
```

## Setup

lean-code and terse have no trigger: they should apply to every edit and every reply. Agents load a skill only when its
description matches the task, so add this to your agent's global instructions
(`~/.claude/CLAUDE.md` for Claude Code, `~/.codex/AGENTS.md` for Codex):

```markdown
Every file you write or edit follows the lean-code skill. Read its SKILL.md once per
session before your first edit, plus the guide for the language you edit
(typescript/, python/, swift/ or rust/). Repo instructions win where they conflict.

Every reply to me follows the terse skill. Read its SKILL.md once per session before your
first answer.
```

## Working on the skills

Clone the repo and link a skill folder in place of the installed copy, so edits show up in
every agent at once:

```bash
git clone git@github.com:popov95s/skills.git ~/Documents/repos/personal/skills
```

```bash
ln -sfn ~/Documents/repos/personal/skills/lean-code ~/.claude/skills/lean-code
ln -sfn ~/Documents/repos/personal/skills/terse ~/.claude/skills/terse
```

## Layout

| Path | Holds |
| --- | --- |
| `lean-code/SKILL.md` | The rules, for every language |
| `lean-code/<language>/guide.md` | How the rules look in that language |
| `lean-code/typescript/react.md` | React rules, read when editing components or hooks |
| `lean-code/<language>/patterns.md` | Which pattern fits which problem, with code |
| `lean-code/smells.md` | Smells to check when reviewing a diff |
| `lean-code/bug-fixes.md` | Steps for a bug fix |
| `decisions/` | Why each skill differs from its sources |

## Sources

lean-code adapts rules from these, rewritten to fit together:

- [ponytail](https://github.com/DietrichGebert/ponytail) plugin: reuse, then standard library, then platform, then
  installed dependency; correct over clever; read the whole flow first; root-cause bug
  fixes; what never to simplify away.
- [codebase-design](https://github.com/mattpocock/skills) skill: the deletion test for wrappers, what counts as an interface,
  testing through the public interface, testing a dependency by its kind.
- [karpathy-guidelines](https://github.com/forrestchang/andrej-karpathy-skills): no error
  handling for impossible cases, clean up only your own leftovers, a failing test first.
- [typescript-advanced-types](https://github.com/wshobson/agents/tree/main/plugins/javascript-typescript/skills/typescript-advanced-types)
  skill: narrowing instead of casts, deriving types from their source, `readonly` inputs,
  type-level tests, `interface` for object shapes, helper types that tie caller input to a real type.
- [vercel-composition-patterns](https://github.com/vercel-labs/agent-skills/tree/main/skills/composition-patterns)
  skill: compound components, a provider as the only place that knows the state source,
  lifting state instead of syncing it, children over render props, React 19 APIs
  (`typescript/react.md`).
- [agent-skills](https://github.com/addyosmani/agent-skills) by Addy Osmani
  (code-review-and-quality, code-simplification, performance-optimization): generic terms
  only in shared code, one mechanism for value-only differences, reading each fact once,
  naming, loops to `map`/`filter`, threshold lists, bounded lists, no silent fallbacks.
- [Refactoring.Guru smell catalogue](https://refactoring.guru/refactoring/smells): the
  smell table in `smells.md`.
- [React: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect):
  resetting state by unmounting or a key.

Why lean-code differs from them: [decisions/lean-code.md](decisions/lean-code.md).
