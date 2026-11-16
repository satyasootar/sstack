---
name: write-tests
description: Write tests for the feature or bug fix just built, in any language. Works out exactly what changed from unstaged changes, staged changes, new files, and the conversation, then writes tests that match the repo's existing framework and style and runs them. Use when the user asks to write/add tests, cover a change with tests, or add a regression test after a fix.
---

# Write tests

Test what was actually built in this session — nothing more. Match the repo's existing test setup exactly. Never change source code to make a test pass; never commit.

## 1. Find out what was worked on

Gather all of these before deciding anything:

1. `git status --porcelain` — note modified, staged, and **untracked** files (new files don't show in `git diff`).
2. `git diff` — unstaged changes.
3. `git diff --staged` — staged changes.
4. Read every untracked source file in full.
5. The conversation so far: what the user asked for, which bug they reported, what was decided, edge cases they mentioned.

If all of git is clean, look at unpushed commits (`git log @{u}..HEAD -p`, or `git log -5 -p` if there's no upstream) and the conversation. If there is still nothing, ask the user what to test.

Ignore changes that are not the work: lockfiles, formatting-only hunks, generated files, local config (`.env*`, editor settings).

Then write down, in one short message to the user:

- **Kind:** feature, bug fix, or refactor.
- **Behaviors to test:** a numbered list, one line each, phrased as "given X, when Y, then Z". For a bug fix, the first item is the exact bug: the input that used to fail and the correct result now.
- **Files under test:** the changed source files, with their public functions, endpoints, components, or commands.

Continue straight away unless something is genuinely ambiguous (e.g. you can't tell what the correct behavior is); then ask one question.

## 2. Learn the repo's test setup

Don't assume a framework — find it:

- **Existing tests first.** Find test files closest to the changed code (same folder, `__tests__/`, `tests/`, `test/`, `spec/`, `*_test.*`, `*.test.*`, `*.spec.*`, `test_*.py`). Read two or three. Copy their framework, file naming and location, imports, describe/test structure, assertion style, fixtures/factories, and how they mock.
- **Then config and manifests** to confirm the runner and command: `package.json` scripts and devDependencies (vitest, jest, mocha, playwright, @testing-library), `pyproject.toml`/`pytest.ini`/`tox.ini`, `go.mod` (`go test`), `Cargo.toml` (`cargo test`), `pom.xml`/`build.gradle` (JUnit), `*.csproj` (xUnit/NUnit), `Gemfile` (RSpec/Minitest), `composer.json` (PHPUnit), `mix.exs` (ExUnit), `Package.swift` (XCTest), `Makefile`/CI workflows for the real test command.
- **No test setup at all:** stop and ask the user which framework to add, suggesting the standard one for the stack. Never install packages without their yes.

Use the same kind of test the repo uses for this layer: unit tests for pure logic, integration tests where the repo already tests routes/DB that way, component tests for UI if they exist. Don't introduce a new kind (e.g. e2e) unless asked.

## 3. Plan the cases

For each behavior from step 1, pick only cases that would catch a real regression:

- **Bug fix:** one regression test reproducing the exact reported input. Add a neighbour case only if the fix touched a boundary (e.g. off-by-one → test both sides).
- **Feature:** the main success path; each branch the new code added; boundaries (empty, zero, one, max, null/missing, unicode, very long); each error path the code handles explicitly (invalid input, not found, unauthorized, upstream failure).
- **Refactor:** tests pinning the existing behavior through the public interface, so the refactor is provably behavior-preserving.

Rules:
- Test through the public interface (exported function, endpoint, component, CLI), not private helpers.
- Mock only what's slow or non-deterministic and outside the code: network, clock, randomness, third-party APIs. Don't mock the code under test or your own modules if the repo's tests don't.
- One behavior per test; the test name states the behavior ("returns 404 when the invoice belongs to another user").
- Deterministic: fixed dates, seeded randomness, no reliance on test order or real external services.
- No snapshot tests unless the repo already relies on them for this kind of code.
- Skip trivial getters, framework glue, and code the change didn't touch.

## 4. Write them

Put tests where the repo's convention says (next to the source or in the mirrored test folder), extending an existing test file for that module if one exists rather than creating a parallel one. Reuse existing fixtures, factories, and helpers before writing new ones.

## 5. Run and check

1. Run only the new/changed tests with the repo's command (e.g. `npx vitest run path`, `pytest path::name`, `go test ./pkg -run Name`, `cargo test name`).
2. A test fails:
   - Mistake in the test (wrong import, setup, expectation you misread) → fix the test.
   - The source really behaves wrongly → **don't** change the source and **don't** bend the test to pass. Keep the test, and report it as a likely bug with the failing output.
3. **Bug fix only, when cheap:** prove the regression test catches the bug. The old code is at `HEAD` (or the commit before the fix), so check it out in a throwaway worktree (`git worktree add <scratch-dir> HEAD`), copy in only the new test, run it, and confirm it fails; then `git worktree remove <scratch-dir>`. Skip this if the worktree would need a long dependency install, and say you skipped it. Never stash, reset, or otherwise touch the user's working tree.
4. Run the module's full existing test file once to make sure nothing else broke.

## 6. Report

```
Tested: <one line: what was built/fixed>
Files: <test files created or edited>
Cases:
  ✓ <behavior>          (one line each)
Result: <N passed, M failed> — <command used>
Regression check: <fails on old code ✓ | skipped: reason>
Likely bugs found: <failing test + why the source looks wrong, or "none">
Not covered: <behaviors deliberately skipped and why>
```
