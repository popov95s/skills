# Personal agent skills

If you keep asking your coding agent to leave unrelated code alone or put the blocker at the top of its reply, these two skills give it those instructions up front. Use either one on its own in Claude Code, Codex, Cursor, or another agent that supports [Agent Skills](https://agentskills.io/).

## Pick a skill

### lean-code

Use [lean-code](lean-code/SKILL.md) when an agent is about to edit or review code you'll maintain. It spells out what "keep it simple" means here. Reuse an existing helper when it fits, keep a decision in one place, and flag unrelated cleanup in the review. The language guides cover TypeScript, JavaScript, Python, Swift, and Rust.

In the [print shop example](lean-code/typescript/strategy-example.md), quotes and print jobs both need to know whether the customer ordered postcards or a banner. When one switch picks the product, a new product needs one new case alongside its pricing and printing code.

> Use lean-code to review this diff. Flag repeated decisions and wrappers that only pass arguments through.

### terse

Use [terse](terse/SKILL.md) when you need to decide what to do after reading an update. It puts the result first, keeps unverified work visible, and uses a table when you need to compare numbers. You can ask for a walkthrough when the short answer leaves you with questions.

> Use terse to summarize this test run. Tell me what failed, what you couldn't verify, and what needs my input.

A reply could be as short as "Checkout passes with one item. Before release, check a two-item order to confirm shipping uses the combined weight."

## Install

Use the [skills CLI](https://github.com/vercel-labs/skills) with Node.js 22.20 or newer. If GitHub requires authentication, use a Git credential helper, authenticated GitHub CLI, or SSH access.

Install both skills for one project by running this from the project root. Pick your agent when prompted.

```bash
npx skills add popov95s/skills --skill lean-code --skill terse
```

To try one skill, keep only its `--skill` option. Start a fresh session and use one of the prompts above.

| To do this | Run |
| --- | --- |
| List available skills | `npx skills add popov95s/skills --list` |
| Install across projects for Codex | `npx skills add popov95s/skills --skill lean-code --skill terse -g -a codex` |
| Verify a project install | `npx skills list` |
| Check for updates | `npx skills check` |
| Update a project skill | `npx skills update lean-code -p` |
| Remove a project skill | `npx skills remove lean-code` |

Use `claude-code` or `cursor` in place of `codex` to target another agent. Add `-g` to list, update, or remove a user-wide install. Bun users can run the same commands with `bunx`.

Set `DISABLE_TELEMETRY=1` to turn off the CLI's usage reporting and security-audit requests. [CLI telemetry documentation](https://github.com/vercel-labs/skills#telemetry)

## Use on every task

To make the skills your defaults, install them user-wide and add the following to `~/.claude/CLAUDE.md` for Claude Code or `~/.codex/AGENTS.md` for Codex.

```markdown
Use lean-code whenever you write or edit code. Read its SKILL.md before the first code edit
in a session and follow its links to the relevant language guide. Follow repository
instructions where they conflict.

Use terse for every reply. Read its SKILL.md before the first reply in a session.
```

The skill gives the agent rules to follow. Use your project's tests and review process to check the result.

## Skill files

| Path | What it contains |
| --- | --- |
| [`lean-code/SKILL.md`](lean-code/SKILL.md) | Coding rules and links to each language guide and pattern catalog. |
| [`lean-code/typescript/react.md`](lean-code/typescript/react.md) | Rules for React components and hooks. |
| [`lean-code/smells.md`](lean-code/smells.md) and [`bug-fixes.md`](lean-code/bug-fixes.md) | Checklists for reviews and bug fixes. |
| [`terse/SKILL.md`](terse/SKILL.md) | Reply length limits, formatting rules, and examples. |
| [`decisions/lean-code.md`](decisions/lean-code.md) | Why the coding rules differ from their sources. |

To try changes from a local clone, run `npx skills add /path/to/skills --list` from a test project. Replace `--list` with `--skill lean-code` to install the local version.

## Sources

- [Ponytail](https://github.com/DietrichGebert/ponytail) for reuse, simple implementations, and root-cause fixes.
- [Matt Pocock's codebase-design](https://github.com/mattpocock/skills) for judging wrappers and testing through public interfaces.
- [Karpathy guidelines](https://github.com/forrestchang/andrej-karpathy-skills) for scope control and tests that show the bug.
- [TypeScript advanced types](https://github.com/wshobson/agents/tree/main/plugins/javascript-typescript/skills/typescript-advanced-types) for narrowing and types derived from their source.
- [Vercel's composition patterns](https://github.com/vercel-labs/agent-skills/tree/main/skills/composition-patterns) and [React's guide to effects](https://react.dev/learn/you-might-not-need-an-effect) for composition and state.
- [Addy Osmani's agent-skills](https://github.com/addyosmani/agent-skills) for shared code, naming, and bounded reads.
- [Refactoring.Guru](https://refactoring.guru/refactoring/smells) for the code-smell catalog.
