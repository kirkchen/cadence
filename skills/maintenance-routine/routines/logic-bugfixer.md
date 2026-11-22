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

## Sweep Siblings — One Defect, One PR

A bug is usually one instance of a way of writing code, and fixing instances one at a time turns a single defect into as many PRs as there are sites, each asking a reviewer to relearn the same pattern. Once a candidate has its failing test, search for its siblings before fixing anything.

1. **Name the pattern as a construct, not a symptom** — the call, idiom or API, used on the kind of input that triggers the bug. "Also gives a wrong result somewhere" is not a pattern.
2. **Search the whole repo for it.** The module lock and the config's scan scope do not apply; the forbidden paths still do. Follow the construct through any wrapper that hides it. This is not wandering: the lock governs where you *look for* bugs, and the sweep only follows one you already hold a failing test for.
3. **Put every hit through Qualification** — its own trigger sequence, its own failing test, its own violated invariant or plainly wrong result. A hit that clears all three is fixed in this PR. A hit where the construct is used safely, because its inputs cannot trigger the bug, is not a bug: record it in `## Sweep` with the reason. A hit you suspect but cannot reproduce goes in `## Observations`, like any hypothesis.
4. **Same symptom, different cause is a different bug.** It is not swept; `## Observations`.

The PR carries **at most 300 changed lines, tests included**. The original site always goes in. Past the ceiling, stop at the last whole site that fits, highest-consequence first, and list every site left — path, line, and why it was left (over the ceiling, forbidden path, needs a schema change, no reproducing test) — in `## Observations`. Do not open a follow-up PR for the leftovers: `## Observations` is where a human picks them up, and splitting one defect across PRs is the cost this section exists to remove.

## Workflow — Test-First, Order Not Negotiable

1. Lock the module. Read it and every spec entry that governs it.
2. Enumerate candidates: boundary conditions, concurrent interleavings, error paths, early returns.
3. **Write the test. Watch it fail. Keep the output.**
4. Sweep for siblings (above), before changing any production code — the sweep decides how much this PR carries.
5. Then change production code. Watch it pass. Repeat 3 and 5 for each sibling the sweep let through: its own test, red first, then its own fix.
6. Full verification.
7. Open the PR.

Reversing the test and the fix produces a test that merely describes the code you just wrote, which proves nothing about the bug and nothing about the fix.

## Verification

The config's commands, green.

Tests needing real infrastructure are slow, but concurrency and transaction bugs are frequently invisible without them. Use them when the bug class calls for it.

## PR Rules

- Branch: `claude/logic-bugfixer/<YYYY-MM-DD>-<n>`
- Title: `fix(<scope>): <summary>`
- **One bug per PR** — the bug and every other site written the same way (see Sweep Siblings). Two different bugs are two PRs.
- Required body sections:
  - `## Module` — which module, and why it was chosen
  - `## Bug` — the trigger sequence, step by step
  - `## Spec basis` — the violated invariant, quoted, with its source
  - `## Sweep` — the search as run (command or pattern, verbatim), and every hit with its disposition: fixed here, not a bug (with the reason), or left for `## Observations`
  - `## Red` — every new test failing, before its fix
  - `## Green` — the same tests passing, after
  - `## Verification` — command output
  - `## Observations` — sites left by the sweep, and suspicions with no reproducing test (`none` if empty)

## Forbidden

The config's list, plus `SKILL.md` § The Routine Contract, plus:

- **Schema and migrations.** A bug that needs a schema change to fix is out of scope no matter how real it is. `## Observations`.
- **The spec documents.** They are the oracle, not the target.

## Per-Run Cap

**2 PRs.** No failing test written → open nothing, and report which module was locked, which candidates were examined, and where each one stalled.
