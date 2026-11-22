# Skills

I keep these in my agent's instructions across projects.

**[lean-code](lean-code/SKILL.md).** My guide for writing and reviewing code with agents. It covers code structure, types, and testing, with examples by language and a checklist for reviewing diffs.

**[terse](terse/SKILL.md).** A routine reply gets 100 words. Put the result first and say what you couldn't verify. If there's a decision for me, put it last and tell me what you'd pick. I'll ask for the long version if I need it.

## Install

Run this in the project where you want the skills. Pick the skills and agents you want.

```sh
npx skills add popov95s/skills
```

The [skills CLI](https://github.com/vercel-labs/skills) needs Node.js 22.20 or newer. Add `-g` to install for all projects, or `--skill lean-code` to install just that skill. Set `DISABLE_TELEMETRY=1` to turn off the CLI's usage reporting and security-audit requests.

Use `npx skills list` to check what's installed. Run `npx skills update` to update installed skills.

## Where the rules came from

The wrapper test comes from [Matt Pocock's skills](https://github.com/mattpocock/skills). I borrowed the rules on reuse and bug fixes from [Ponytail](https://github.com/DietrichGebert/ponytail), the scope rules from [Karpathy guidelines](https://github.com/forrestchang/andrej-karpathy-skills), and the rules on shared code and bounded reads from [Addy Osmani's agent-skills](https://github.com/addyosmani/agent-skills).

The TypeScript guide draws on [typescript-advanced-types](https://github.com/wshobson/agents/tree/main/plugins/javascript-typescript/skills/typescript-advanced-types). The React guide draws on [Vercel's composition patterns](https://github.com/vercel-labs/agent-skills/tree/main/skills/composition-patterns) and [React's guide to effects](https://react.dev/learn/you-might-not-need-an-effect). The smell list follows [Refactoring.Guru](https://refactoring.guru/refactoring/smells).

`lean-code` brings these ideas together as a guide for writing and reviewing
code. It favors files with one clear job, shares code as soon as a second use
is known, and gives each decision one place to live. The pattern guides show
when to use a strategy or an adapter and when a plain function is enough.
When a change exposes unrelated problems, the review calls them out so the
fix can stay focused.

See [the decision log](decisions/lean-code.md) for where I went a different way.
