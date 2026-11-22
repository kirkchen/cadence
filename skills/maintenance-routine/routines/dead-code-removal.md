# Routine: dead-code-removal

Remove code that static search proves unreachable.

The highest-objectivity archetype: "nothing references this" is close to a fact, provided you searched everywhere and know which references are invisible to search. That second half is what the repo config's **Dynamic-reference traps** section is for, and it is the difference between this routine being safe and it being a liability.

## Scan Scope

Source directories, as listed in the config's `dead-code-removal` section. Absent → all first-party source directories, excluding tests, vendored code and generated output.

**One subdirectory per run.** Not the whole tree. Prefer cold areas — `git log --since='60 days ago' --name-only` — and state the choice and its reason in the PR body.

## Qualification

All three, or it does not get deleted:

1. **No references.** A repo-wide search for the symbol finds nothing outside its own definition and tests that exist only to serve it. Search the *whole repo*, not just source: deployment manifests, CI config, container definitions, docs and scripts routinely name symbols, files and environment variables as bare strings.
2. **Not a dynamic-reference trap.** Check every entry in the config's trap list. Anything matching, skip — no exceptions, no "but this one looks obviously dead".
3. **Verification stays green** with it removed.

Any doubt on any condition → do not delete. It goes in `## Observations`, where a human decides.

Deliberately-retained transitional code deserves specific mention: migrations run as expand-then-contract leave vestigial fields and dual-read branches on purpose, and comments usually say so. Removing them early is a migration decision, not maintenance. Skip anything whose comments read as a planned interim state.

## Branches And Fallbacks

This archetype owns code that cannot run *inside* a function as well as symbols nothing references: a fallback, default arm or error path whose guard can never hold. `logic-simplifier` hands these over instead of removing them.

For these, condition 1 is a reachability argument rather than a symbol search: the guard, and every caller or input source that could make it hold, written out in `## Evidence` so a reviewer can check each one. Conditions 2 and 3 apply as written. One more skip, carried over from `logic-simplifier` because the code moved to a routine without that guardrail: a branch whose comment explains it as a guard against a race, an ordering problem or a bug that already happened is not dead because the call graph says so today.

## Lint First

Unused imports, locals and parameters are what the repo's lint "unused" rules catch. When a candidate is one of those and the rule is switched off, the output is enabling the rule, not deleting a few by hand — see `SKILL.md` § The Routine Contract, prefer a lint rule to a hand fix. A run that removes two imports is a run that will remove two more next week.

## Workflow

1. Pick the subdirectory. Note why.
2. List candidates: unreferenced exports (functions, types, components, constants), branches and fallbacks that can never be reached, handlers nothing registers.
3. Apply the three conditions to each, searching repo-wide.
4. Delete survivors, along with tests that exist only to cover the deleted code — or, for a lint-catchable class, write the rule instead.
5. Verify.
6. Open the PR.

## Verification

The config's commands. All green before the PR exists.

## PR Rules

- Branch: `claude/dead-code-removal/<YYYY-MM-DD>-<n>`
- Title: `refactor(<scope>): remove unreachable <summary>`
- One theme per PR, at most **300 deleted lines** — over that, split
- Required body sections:
  - `## Removed` — path and name of every deleted symbol, or path and line of every deleted branch
  - `## Evidence` — the actual search commands and their output, showing no references; for a branch, the guard and the callers that rule it out
  - `## Verification` — command output
  - `## Observations` — suspected but uncleared candidates (`none` if empty)

`## Evidence` is what makes this PR reviewable in minutes instead of an hour. A reviewer should be able to re-run your searches and get your results.

## Forbidden

The config's list, plus the universal rules in `SKILL.md` § The Routine Contract.

## Per-Run Cap

**3 PRs.** Nothing clearing all three conditions → open nothing, and report which subdirectory was scanned, which candidates were considered, and which condition each failed.
