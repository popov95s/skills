# Skills

I keep these in my agent's instructions across projects.

**[lean-code](lean-code/SKILL.md).** My guide for writing and reviewing code with agents. It covers code structure, types, and testing, with examples by language and a checklist for reviewing diffs.

**[terse](terse/SKILL.md).** A routine reply gets 100 words. Put the result first and say what you couldn't verify. If there's a decision for me, put it last and tell me what you'd pick. I'll ask for the long version if I need it.

## Why I wrote lean-code

I started `lean-code` because agents were writing paragraphs of comments
defending decisions I wanted them to rethink.

The smallest fix often worked the first time. On the second request, the
same function got another boolean flag or another branch. After a few
rounds, tracing all the paths took longer, and the function was harder
to maintain.

Now the skill covers how a feature is built from the start. It gives the
agent rules for where decisions, state, and side effects belong. If the
feature already needs a pattern, I want it in the first version so the next
change has a clear place to go.

## Install

Run this from your project root.

```sh
npx skills add popov95s/skills
```

See the [skills CLI docs](https://github.com/vercel-labs/skills) for requirements and options.

## Ideas I adapted

I adapted the wrapper test from [Matt Pocock's skills](https://github.com/mattpocock/skills) and ideas on reuse and bug fixes from [Ponytail](https://github.com/DietrichGebert/ponytail). The [Karpathy guidelines](https://github.com/forrestchang/andrej-karpathy-skills) shaped how I keep changes in scope. [Addy Osmani's agent-skills](https://github.com/addyosmani/agent-skills) shaped how I handle shared code and bounded reads.

The TypeScript guide draws on [typescript-advanced-types](https://github.com/wshobson/agents/tree/main/plugins/javascript-typescript/skills/typescript-advanced-types). The React guide draws on [Vercel's composition patterns](https://github.com/vercel-labs/agent-skills/tree/main/skills/composition-patterns) and [React's guide to effects](https://react.dev/learn/you-might-not-need-an-effect). The review checklist uses the smell names from [Refactoring.Guru](https://refactoring.guru/refactoring/smells).

`lean-code` brings these ideas together as a guide for writing and reviewing
code. It favors files with one clear job, shares code as soon as a second use
is known, and gives each decision one place to live. The pattern guides show
when to use a strategy or an adapter and when a plain function is enough.
When a change exposes unrelated problems, the review calls them out so the
fix can stay focused.

See [the decision log](decisions/lean-code.md) for where I went a different way.
