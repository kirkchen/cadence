# Routine: logic-simplifier

Untangle convoluted logic inside a single function.

Second-most subjective of the seven, and the one most likely to produce a technically-correct diff that nobody wanted. The guardrails below matter more than the qualification conditions. Read all three before scanning anything.

## Guardrail 1 — A Long "Why" Comment Means Skip

Mature code is full of **defensive complexity**: it looks convoluted because it is handling a race, an ordering constraint, or a bug that already happened once. Such places usually carry a paragraph explaining exactly that.

**A comment explaining a race, lock ordering, concurrency, retry semantics or a historical bug → skip.** That is not accidental complexity. Simplifying it puts the fixed bug back.

## Guardrail 2 — Only Touch Covered Code

Before simplifying, confirm the logic has test coverage, and confirm the tests reach *the specific branch* you intend to change. Verify it rather than assuming: temporarily inject a defect into that branch, check that a test goes red, then `git checkout -- <file>` and confirm with `git status`.

**No coverage, no simplification.** Adding the missing test is a worthwhile separate PR — but doing both in one PR leaves nobody able to tell whether behaviour changed.

## Guardrail 3 — Do Not Add Or Modify Tests

Simplification is behaviour-preserving. The unchanged suite passing is the proof. If a test must change to go green, behaviour changed: abandon the candidate, record it in `## Observations`.

## Qualification

Past all three guardrails, these are worth simplifying:

1. **The same condition is evaluated repeatedly** — early returns can merge, nested conditionals can flatten.
2. **An unreachable branch** — earlier conditions already cover it. Where this overlaps `dead-code-removal`, leave it to that routine.
3. **Deep nesting replaceable by early return** — three or more levels, with no cleanup logic depending on the nesting.
4. **A variable assigned once and used immediately** — *and* carrying no naming intent. A well-named intermediate is documentation; do not destroy one to save a line.

Anything outside these four is "I find this more readable", which is not a mandate. Leave it.

## Workflow

1. Pick a subdirectory, one per run.
2. Run candidates through the three guardrails. Most will die here.
3. Apply the four conditions to the rest.
4. One function at a time; run its test immediately after each change.
5. Full verification.
6. Open the PR.

## Verification

The config's commands, green. The tree must be clean of guardrail-2 mutations — check `git status`.

## PR Rules

- Branch: `claude/logic-simplifier/<YYYY-MM-DD>-<n>`
- Title: `refactor(<scope>): <summary>`
- **One function per PR**, at most **100 changed lines**. Smaller than the other archetypes on purpose: "is the behaviour really identical?" is the most expensive question a reviewer can be asked, and the only way to keep it cheap is to keep the diff tiny.
- Required body sections:
  - `## Before / After` — both versions of the code
  - `## Coverage` — which test covers this logic, and how you confirmed it reaches this branch
  - `## Why it's equivalent` — point by point, why behaviour is unchanged
  - `## Verification` — command output, explicitly noting that no test changed
  - `## Observations` — candidates that cleared the guardrails but were dropped, with reasons (`none` if empty)

## Forbidden

The config's list, plus `SKILL.md` § The Routine Contract, plus:

- **Any code carrying a race / ordering / historical-bug comment** (guardrail 1)
- **Concurrency- and state-machine-dense modules.** The worst risk-to-reward ratio in the repo for a readability change. Problems there belong to `logic-bugfixer`, which at least arrives with a failing test. The config should name these modules; if it does not, treat scheduler, dispatcher, lifecycle and reconciler code as covered.

## Per-Run Cap

**2 PRs.** Nothing clears the guardrails → open nothing. **Finding nothing is normal and healthy for this archetype.** Do not manufacture a PR to show activity.
