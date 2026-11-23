# Bug fixes

Read this before fixing a bug. The code rules in [SKILL.md](SKILL.md) still apply to the fix.

1. **Reproduce it with a failing test.** Write the test that shows the bug, run it, and see
   it fail for the reported reason. Name the break it catches in the test name. The test
   follows [testing.md](testing.md).
2. **Find the root cause, not the symptom.** A report names one path that broke. Find
   where the wrong value or decision is made.
3. **Find every caller.** Search for every caller of the function you are about to change,
   and check whether each one has the same problem.
4. **Fix it once, where every caller passes through.** One fix in the shared function beats
   a guard in each caller. Patching only the path the report names leaves the siblings
   broken.
5. **Fix stored data at the write.** When the code wrote a wrong value, fix the code that
   writes it, and backfill only if the task asks for it. Don't add read-side workarounds.
6. **Watch the test pass**, then run the focused tests around it.
7. **Report** the root cause in one line, the callers you checked, and any caller you left
   alone with the reason.
