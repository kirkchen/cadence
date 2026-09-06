# Routine: useless-test-pruner

Find tests that cannot fail. Fix them first; delete only when there is nothing to fix.

The name is misleading on purpose-of-emphasis grounds: pruning is the fallback. A test that cannot fail usually covers behaviour someone cared about, and the useful output is a test that now catches the bug it was always meant to catch.

## Scan Scope

Unit and integration test files, as listed in the config's `useless-test-pruner` section.

**One subdirectory per run.**

## Specs Are Not Tests

Executable specifications — feature files, acceptance-test step definitions, end-to-end scenarios — are **out of scope entirely**, whatever directory they sit in. Deleting one deletes a requirement. Only ordinary unit and integration tests qualify.

The config's forbidden-path list names the repo's spec directories. If it does not and the repo clearly has some, that is a config gap: report it in `## Observations` and stay out of them this run.

## Qualification

Three ways a test cannot fail:

1. **No assertion.** The test body contains no assertion call at all.
2. **It only asserts a mock.** The asserted value comes from a stub's configured return value, or is a literal constant. The test exercises the test double, not the code.
3. **Mutants survive.** See below.

## Mutation Check

Most repos have no mutation-testing tool. Do it by hand, per candidate:

1. Inject one obvious defect into the source under test — invert a condition, return null, change a constant.
2. Run only that test file.
3. Still green → the test cannot fail. Confirmed.
4. **Restore the source.** `git checkout -- <file>`. This step is not optional.
5. `git status` to confirm the tree is clean, before moving to the next candidate.

Record, per candidate: what was mutated, which command ran, why the test stayed green. `## Mutation evidence` needs all three.

A mutation that escapes into a PR discredits every routine PR after it. Step 4 and step 5 are the whole reason this archetype has a per-run cap this low.

## Fix, Don't Delete

For each confirmed finding:

- **The covered behaviour matters** → rewrite the test so it actually catches the defect. Explain the change in the PR.
- **The behaviour is covered elsewhere, or is meaningless** → then delete. "Covered elsewhere" must be *verified*, not assumed: name the covering test's path in the PR.

## Before You Start

Run the config's install and build commands. Skipping a build step the tests depend on produces failures that look like broken tests and will send this routine hunting phantoms.

Tests requiring real infrastructure (containers, databases) are slow. Prefer candidates that do not need them.

## Verification

The config's lint and test commands, green.

## PR Rules

- Branch: `claude/useless-test-pruner/<YYYY-MM-DD>-<n>`
- Title: `test(<scope>): <summary>`
- At most **5 tests per PR**
- Required body sections:
  - `## Findings` — each test's path and which of the three conditions it meets
  - `## Mutation evidence` — mutation, command, green output, per candidate
  - `## Action` — fixed or deleted, per candidate; deletions cite the covering test
  - `## Verification` — command output
  - `## Observations` — see below

## Do Not Fix Production Code

If a mutation reveals a **real bug** the test failed to catch, do not fix it here. That is a different change with a different risk profile, and burying it in a test-pruning PR hides it from the reviewer who should be looking at it. Write it into `## Observations`.

Equally: never edit production code to make a test go green.

## Forbidden

The config's list, the repo's spec directories, plus `SKILL.md` § The Routine Contract.

## Per-Run Cap

**2 PRs.** No findings → open nothing, and report which subdirectory was scanned, how many tests were examined, and why they all passed.
