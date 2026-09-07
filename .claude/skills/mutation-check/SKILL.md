---
name: mutation-check
description: Prove a test can actually fail before trusting it. Use after writing or changing any test, before opening a PR, and whenever a test covers a guard, a validation rule, or an error path. Also use when review says an assertion "could not fail" or "holds regardless of the code under test".
---

# Prove the test can fail

A test that passes whether or not the code is correct is worse than no test, because it reads as
coverage. This repository has found that failure more often than any other, in its own checks:
two lint rules that were configured and enforced nothing, a latency test whose stub resolved
immediately, a merge gate that passed a stale review, a contract suite blind to half of
TypeScript's method syntax.

## The check

For every behaviour a test claims to cover:

1. **Break exactly that behaviour** in the source — delete the guard, invert the condition, drop
   the validation, remove the `await`.
2. **Run the test.** It must fail.
3. **Restore the source** and run again. It must pass.

If step 2 passes, the test does not cover what its name says. Fix the test, not the source.

```bash
cp src/thing.ts /tmp/thing.bak
# make the one-line change that breaks the behaviour
npx jest path/to/thing --colors=false | grep -aE "Tests: "
cp /tmp/thing.bak src/thing.ts
npx jest path/to/thing --colors=false | grep -aE "Tests: "
```

Watch the suite count as well as the pass count. A mutation that makes a file fail to parse
reports "0 tests" rather than a failure — that is a broken experiment, not a caught mutation.

## Where the check usually finds something

- **The fixture cannot reach the branch.** A guard against app hostnames was tested against a map
  containing no app hostnames; deleting the guard changed nothing. A membership status check was
  tested against fixtures where every membership is `active`. **Fix:** make the collaborator an
  argument so the test can hand it a hostile value.
- **The test supplies the thing that is missing.** A retry test passed `onRetry` itself, so it
  proved the button works and nothing about whether anything provides one.
- **Two states share a value.** `theme === null` means both "still loading" and "failed", so
  waiting only on the parent query asserts nothing about which. **Fix:** make the failure
  observable — count rejections, delay them, and wait for the count.
- **The error path is unreachable from the type.** A union with one member makes a runtime guard
  provably dead, so the compiler removes it. **Fix:** move the rule into a predicate taking
  `string`, and test the predicate. To call through an untyped boundary use `Reflect.apply` — a
  cast is a lint error here.
- **The assertion is about scheduling.** `waitFor` polls on its own interval, so "X is still
  pending when Y happens" is a claim about the test runner. Assert the settled state instead.

## Recording it

Put the result in the commit message and the PR body, as a table if there is more than one:

| mutation | result |
| --- | --- |
| drop the cap check | 1 test fails |
| move the cap check after the filter | 1 test fails |

If a guard genuinely cannot be covered — a native-only behaviour, a frame React never exposes —
say so at the line and in the PR. "Kept on the reasoning below, not on a green run" is an
acceptable answer. Silence is not.
