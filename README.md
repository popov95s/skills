# Skills

Personal agent skills. Each folder is one skill with a `SKILL.md` at its root.

| Skill | What it does |
| --- | --- |
| [lean-code](lean-code/SKILL.md) | How code is written in TypeScript, JavaScript, Python and Swift |

## Install

Link a skill into the place an agent loads skills from:

```bash
ln -s ~/Documents/repos/personal/skills/lean-code ~/.claude/skills/lean-code
```

## Sources

lean-code adapts rules from these, rewritten to fit together:

- [ponytail](https://github.com/DietrichGebert/ponytail) plugin: reuse, then standard library, then platform, then
  installed dependency; correct over clever; read the whole flow first; root-cause bug
  fixes; what never to simplify away.
- [codebase-design](https://github.com/mattpocock/skills) skill: the deletion test for wrappers, what counts as an interface,
  testing through the public interface, testing a dependency by its kind.
- [karpathy-guidelines](https://github.com/forrestchang/andrej-karpathy-skills): no error
  handling for impossible cases, clean up only your own leftovers, a failing test first.
- [Refactoring.Guru smell catalogue](https://refactoring.guru/refactoring/smells): the
  smell table in `smells.md`.
- [React: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect):
  resetting state by unmounting or a key.

Why lean-code differs from them: [decisions/lean-code.md](decisions/lean-code.md).
