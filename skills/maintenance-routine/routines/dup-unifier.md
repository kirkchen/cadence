# Routine: dup-unifier

Converge duplicate implementations into one — but only the duplicates that provably change together.

## Read This First — Parallel Is Not Duplicate

Mature codebases are full of code that looks duplicated and is **deliberately parallel**: sibling implementations kept separate so each stays readable and can evolve on its own. Comments usually say so outright — `sibling of`, `mirrors`, `same pattern as`, `parallel to`.

**Default to skipping anything carrying that language.** Forcing such code into a shared generic helper trades a little repetition for an abstraction nobody dares to modify, which is a bad trade at any duplication count.

Overriding the default is allowed, but the PR body then has to argue *why converging now beats staying parallel*. A weak argument means no PR.

## Qualification

All four:

1. **One concept implemented twice or more** — not two adjacent concepts that happen to look alike.
2. **No comment explains the separation.** If there is one, the paragraph above applies.
3. **They change together.** You can point at history: a commit that touched both, or a bug from changing one and forgetting the other. Find it with `git log -S '<distinctive fragment>'` or `git log --oneline -- <path-a> <path-b>`.
4. **Convergence needs no new flag.** If the unified function requires a boolean parameter to select between the two behaviours, they were two behaviours and you have made things worse.

Condition 3 is the load-bearing filter. No evidence that they change together means they merely resemble each other, and resemblance is not a maintenance cost. Do not proceed on the other three alone.

## Workflow

1. Pick a subdirectory, one per run.
2. Find candidate duplicate groups.
3. Apply the four conditions, spending most of the effort on the history check.
4. Converge: extract the shared implementation, point both call sites at it.
5. Verify.
6. Open the PR.

## Verification

The config's commands, green.

Convergence is a behaviour-preserving refactor, so **do not add or modify tests**. The existing suite passing unchanged *is* the proof that behaviour held. If a test has to change to go green, behaviour changed: abandon the candidate and write it up in `## Observations`.

## PR Rules

- Branch: `claude/dup-unifier/<YYYY-MM-DD>-<n>`
- Title: `refactor(<scope>): unify <summary>`
- **One duplicate group per PR**, at most 300 changed lines
- Required body sections:
  - `## Duplication` — every location with path and line, side by side, differences called out
  - `## Why now` — condition 3's history evidence, with the actual commands and output
  - `## Unified` — where the shared implementation lives and why there
  - `## Verification` — command output, explicitly noting that no test changed
  - `## Observations` — candidates seen but not taken, with reasons (`none` if empty)

## Forbidden

The config's list, plus `SKILL.md` § The Routine Contract, plus:

**No convergence across package or module boundaries.** Hoisting similar code from two packages into a shared one is an architecture decision — it changes what that shared package is for — not a refactor. Record it in `## Observations` for a human.

## Per-Run Cap

**2 PRs.** Nothing clearing all four conditions → open nothing, and report which candidates were examined and which condition each failed.
