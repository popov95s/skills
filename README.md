# Skills

Personal skills for coding and interacting with agents.

**[lean-code](lean-code/SKILL.md).** A guide for writing and reviewing code with agents. It covers code structure, types, and testing, with examples by language and a checklist for reviewing diffs. It prefers code that is low on comments, with reusable abstractions and strict typing.

**[terse](terse/SKILL.md).** My preference for how agents talk to me. A routine reply gets 100 words. Put the result first and say what you couldn't verify. If there's a decision for me, put it last and tell me what you'd pick. I'll ask for the long version if I need it. 

## Overview

Agents continuously write paragraphs of comments
defending decisions they should often rethink.

The smallest fix often works the first time. On the second request, the
same function gets another boolean flag or another branch. After a few
rounds, tracing all the paths takes longer, and the function becomes harder
to maintain and impossible to read.

`lean-code` covers how a feature is built from the start. It gives the
agent rules for where decisions, state, and side effects belong. If the
feature already needs a pattern, I want it in the first version so the next
change has a clear place to go.

## Install

Run this from your project root.

```sh
npx skills add popov95s/skills
```

See the [skills CLI docs](https://github.com/vercel-labs/skills) for requirements and options.

## Ideas adapted

`lean-code` borrows ideas from a few well-established skills, namely [Matt Pocock's skills](https://github.com/mattpocock/skills), [Ponytail](https://github.com/DietrichGebert/ponytail), [Karpathy guidelines](https://github.com/forrestchang/andrej-karpathy-skills) and [Addy Osmani's agent-skills](https://github.com/addyosmani/agent-skills). 

The TypeScript guide draws on [typescript-advanced-types](https://github.com/wshobson/agents/tree/main/plugins/javascript-typescript/skills/typescript-advanced-types). The React guide draws on [Vercel's composition patterns](https://github.com/vercel-labs/agent-skills/tree/main/skills/composition-patterns) and [React's guide to effects](https://react.dev/learn/you-might-not-need-an-effect). The review checklist uses the smell names from [Refactoring.Guru](https://refactoring.guru/refactoring/smells).

`lean-code` brings these ideas together as a guide for writing and reviewing
code. It favors files with one clear job, shares code as soon as a second use
is known, and gives each decision one place to live. The pattern guides show
when to use a strategy or an adapter and when a plain function is enough.
When a change exposes unrelated problems, the review calls them out so the
fix can stay focused.
