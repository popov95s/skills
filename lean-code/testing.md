# Testing

Read this when you write, change or delete a test. The code rules in [SKILL.md](SKILL.md)
still apply to test files. A test earns its place by catching a break. A suite is good when
every test can fail for a reason that matters and no two tests fail for the same reason.

## Before writing a test

1. **Name the break.** Name the production change that should make this test fail, and say
   whether that change is a bug or a decision. Cannot name one: don't write the test.
2. **Pick the level.** Each behaviour is asserted at one level only.
   - Pure rules, branches and boundaries: unit tests through the public function.
   - Wiring (the endpoint reaches the use case, the store reads what the writer wrote): one
     integration test with a happy path and one failure per connection. It does not repeat
     the unit cases.
   - A test that can only fail when another test fails is overlap: delete it.
3. **Check what exists.** Search for a test that already covers this behaviour. Extend it
   instead of adding a near-copy.

## What a test asserts

- **Hand-derived expectations.** Write the expected value as a literal or a hand-checked
  fixture. A value computed by the code under test, or by its helpers, passes whatever that
  code does.
- **Behaviour the caller can see.** Results, errors, stored data, calls to the outside
  world. Not private structure, call order inside the unit or intermediate values.
- **No change detectors.** If only an intentional decision can fail the test (a constant's
  value, exact message wording, a file's text), it fires on redesign and sleeps through
  bugs. Test the behaviour that depends on the decision: not `MAX_RETRIES === 5`, but
  "a failing call is retried 5 times and a 6th attempt never happens".
- **Your code, not the framework's.** Don't assert that the router calls a registered
  handler or that a library does what its docs say. Constructors, getters, constants and
  forwarding are tested only when they validate, default, derive or cause a side effect.
  Prose for humans gets no test. Scripts get run against controlled inputs.
- **One behaviour per test**, named for it. "and" in the name: split it.
- **Table rows are distinct rules or boundaries**, not more inputs for the same rule. Three
  more valid emails are not three rows: the empty string and the missing `@` are.

## Doubles

- **Never assert on a mock.** An assertion that passes because the mock exists tests the
  mock. Assert the real component's result.
- **Mock the slow or external layer**, and keep real what the test depends on. List the
  real method's side effects before replacing it.
- **Doubles are specific.** When arguments or call counts are part of the contract, assert
  them. A fake that accepts anything verifies nothing. Each branch (success, failure,
  malformed) gets its own fixture, so the wrong branch cannot pass.
- **Doubles mirror the real data.** Give the full documented shape, not only the fields the
  test reads.
- **Mock setup longer than the test**: use the real component, or move up to an
  integration test.
- **Test-only code stays out of production classes.** A method only tests call lives in a
  test utility.

## What thorough means

Each of these gets one test, and a missing one is a gap:

- every branch of the policy, and each side of every boundary (0, 1, max, max + 1, empty)
- every typed error, with the case that raises it
- every kind a picker returns, and the picker's failure on an unknown one
- every write that must be atomic, with a failure halfway
- every codec: a round-trip, and decoding of each stored shape still in use

Beyond that list, more tests are slop. Never add a test to raise coverage.

## Prove it can fail

- **Watch it fail first** for the right reason, then pass. This is required for a bug fix
  and for any new test of logic that already exists.
- **Flip one condition** in the code under test (`<` to `<=`, drop a guard) for risky logic:
  money, permissions, data loss. At least one test must fail. If none does, the test is
  missing or too loose. Revert the flip. Run a mutation tool on the files you touched
  only if the project already has one.

## Deleting tests

- Tests on internals of modules you merged go. Test the merged module.
- A test that fails on every refactor and has never caught a bug is a change detector.
  Delete it.
- Slow or flaky tests are fixed or deleted, never retried or skipped.

## Report

One line: tests added or removed, and for each new test the break it catches. A test
you cannot describe that way is deleted before you reply.
