# Routine: logic-bugfixer

Model one module's business logic against its written spec, find real bugs, fix them test-first.

## Lock Onto One Module — Do Not Wander

Roaming a large codebase looking for bugs yields a pile of "theoretically possible" findings and no confirmed ones. **One module per run.**

The config's `logic-bugfixer` section should list a rotation order, highest-consequence first — state machines, concurrency, lifecycle management, external-system reconciliation. Absent a list, pick the module with the densest state transitions and say why in the PR.

## Your Oracle Is The Spec, Not Your Instinct

The config's **Spec sources** section names where the domain glossary and state-machine invariants are written down. Read them completely before starting, then use them as the standard the code is measured against.

Without written invariants this archetype degrades into opinion. If the config names no spec sources, say so and stop.

**"Code behaviour disagrees with the written spec" is itself a finding.** But when they disagree, do not decide unilaterally which is right. If your read is that the doc is stale rather than the code wrong, put it in `## Observations` — change neither the doc nor the code.

## Qualification

All three, or it is not a bug you may fix:

1. You can state a **concrete trigger sequence** — which input, which ordering, which concurrent interleaving.
2. You can write a **failing test** that reproduces it.
3. It violates a written invariant, or produces a plainly wrong result.

No failing test, no finding. Condition 2 is not a formality: it is the line between a bug and a hypothesis, and hypotheses go in `## Observations`.

## Workflow — Test-First, Order Not Negotiable

1. Lock the module. Read it and every spec entry that governs it.
2. Enumerate candidates: boundary conditions, concurrent interleavings, error paths, early returns.
3. **Write the test. Watch it fail. Keep the output.**
4. Then change production code. Watch it pass.
5. Full verification.
6. Open the PR.

Reversing 3 and 4 produces a test that merely describes the code you just wrote, which proves nothing about the bug and nothing about the fix.

## Verification

The config's commands, green.

Tests needing real infrastructure are slow, but concurrency and transaction bugs are frequently invisible without them. Use them when the bug class calls for it.

## PR Rules

- Branch: `claude/logic-bugfixer/<YYYY-MM-DD>-<n>`
- Title: `fix(<scope>): <summary>`
- **One bug per PR**
- Required body sections:
  - `## Module` — which module, and why it was chosen
  - `## Bug` — the trigger sequence, step by step
  - `## Spec basis` — the violated invariant, quoted, with its source
  - `## Red` — the new test failing, before the fix
  - `## Green` — the same test passing, after
  - `## Verification` — command output
  - `## Observations` — suspicions with no reproducing test (`none` if empty)

## Forbidden

The config's list, plus `SKILL.md` § The Routine Contract, plus:

- **Schema and migrations.** A bug that needs a schema change to fix is out of scope no matter how real it is. `## Observations`.
- **The spec documents.** They are the oracle, not the target.

## Per-Run Cap

**2 PRs.** No failing test written → open nothing, and report which module was locked, which candidates were examined, and where each one stalled.
