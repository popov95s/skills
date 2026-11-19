# Code smells

Use this when reviewing a diff or acting on a review or analyzer finding. A smell is a
question, not a verdict: name the affected responsibility, its callers and the concrete cost
before changing anything. In existing code, flag it and suggest the fix (see Scope in
[SKILL.md](SKILL.md)). Names follow the [Refactoring.Guru catalogue](https://refactoring.guru/refactoring/smells).

## Smells and when they are not a problem

| Smell | Example | Fix | Not a problem when |
| --- | --- | --- | --- |
| Long function | One function parses input, prices, writes and builds the response | Extract the named steps; keep the order readable in one orchestrator | It is long but does one thing at one level |
| Large module | One short file owns search, permissions, billing and display | Split by responsibility, even below the line limit | The parts always change together |
| Primitive obsession | Account ids, currencies and modes are all plain strings | A domain type, a literal union or a codec | The value has no rules of its own |
| Long parameter list | Sending a message takes identity values, mode flags and options | Fix the operation's shape; group values that form one concept | An options object would only hide the same mess |
| Data clump | Amount and currency always travel together | One value type for the pair | The values travel together only once |
| Switch on kind | Several callers repeat the same provider choice | One derived kind, then a strategy | One exhaustive data mapping, a result-union `switch`, or a guard |
| Temporary field | A job record has fields that only one mode uses | Explicit variants | The field is optional for every kind |
| Refused bequest | A read-only store implements writes that always throw | Split the interface | Never: an unsupported method breaks the interface |
| Alternative interfaces | Equal providers have different method names that every caller translates | One adapter per provider | The providers really offer different capabilities |
| Divergent change | Pricing, vendor formats and email copy all change one file | Separate the responsibilities | The changes come from one decision |
| Shotgun surgery | One rule change touches the same decision in several features | Give the decision one owner | Adding a strategy, its picker case and its test |
| Duplicate code | Two callers encode the same discount rule | One owner from the second copy | The copies only look alike and follow different rules |
| Lazy module or hollow wrapper | A new module only renames an existing call | Inline it | It owns a real rule or interaction, however small |
| Data class | Callers rebuild the same rule from a passive record | Put the rule with its owner | Plain immutable records with no rule to share |
| Dead code | A variant or export has no caller left | Delete it | Something reaches it by registration, generated code, a framework entry point or an outside consumer |
| Speculative generality | A plugin system exists for imagined uses | Delete it | A strategy for two or more real kinds, or an interface that lets tests inject fakes |
| Feature envy | Several callers read order internals to work out its refund rule | Move the rule to the owning domain | A pure function reads explicit data it was given |
| Inappropriate intimacy | One module patches another's private cache | Expose a domain operation | Never; adding getters for the same internals is no fix |
| Message chains | A caller walks several owners to work out a shipping charge | Ask the owning domain for the answer | Plain data access or a builder |
| Middle man | New modules forward the same arguments without adding anything | Remove the hop | An authorization, logging or integration layer that is there on purpose |

Also look for boolean modes, dense expressions that hide a rule, and types or helpers that
are harder to read than the code they replace.

## Acting on analyzer findings

| Finding | Ask before changing code |
| --- | --- |
| Duplicate structure | Do the copies encode the same rule that must change together, or do they only look alike? |
| Import cycle | Which dependency breaks ownership or layering? Remove that dependency; don't shuffle imports until the warning goes away |
| Boundary violation | Is the configured rule current? Keep the intended architecture; don't add a layer just to satisfy the check |
| Dead-code candidate | Could a framework entry point, registration, generated code or an outside consumer reach it? |
| Length, nesting or coupling | What does it cost to maintain or change? Extract the responsible rule, not lines to meet a number |

Read the report's scope, totals, confidence and configuration before trusting a clean
result: a check that was never configured has not passed. Keep findings tied to the change
under review unless a full audit was asked for. A finding never justifies a new tool, a
repo-wide cleanup, a suppression or an accepted-debt entry.
