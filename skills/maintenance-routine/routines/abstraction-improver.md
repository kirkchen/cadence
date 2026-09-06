# Routine: abstraction-improver

Flatten indirection layers that carry no meaning.

Division of labour with `logic-simplifier`: that one works *inside a single function*; this one works on indirection *across functions and files*. The guardrails are equally strict — read them before scanning.

## Guardrail 1 — One Implementation Is Not Over-Abstraction

"This interface has only one implementor" is not a reason to flatten. Deliberate seams routinely have exactly one production implementor:

- Injection points that exist so tests can supply a fake clock, fake client, or fake transport
- Dispatch tables documented as the extension point for adding the next variant
- Package and module boundaries fixed by a decision record

Before flattening, ask: **with this layer gone, can the existing tests still be written?** If not, it is a seam. Leave it.

## Guardrail 2 — Do Not Cross Package Boundaries

Where a repo's packages draw their lines is an architecture decision, usually written down. Moving code between them — in either direction — changes what those packages are for. That is not a refactor, whatever it does to the duplication count. Record it in `## Observations`.

The same holds for components that communicate over a defined transport rather than by direct call. Do not introduce a new shared layer between them to reduce repetition; the separation is the design.

## Guardrail 3 — Do Not Add Or Modify Tests

Flattening is behaviour-preserving. The unchanged suite passing is the proof. A test that must change to go green means behaviour changed: abandon it.

## Qualification

Past all three guardrails, these are worth flattening:

1. **Pure pass-through wrapper** — A calls B and returns its result unchanged, adding no transformation, validation, error handling or named intent, and A's name explains the caller's intent no better than B's.
2. **Type alias stacked on type alias** — with no semantic distinction between the levels.
3. **Single-implementation interface or abstract base** that no test uses as a substitution point.
4. **A helper extracted for exactly one call site** whose extraction makes the caller *harder* to read — the reader has to jump to another file to learn what happens.

Condition 4 needs care. Extracting a small function with a good name is usually an **improvement**. It only qualifies when the name adds no information the call site did not already have.

## Workflow

1. Pick a subdirectory, one per run.
2. Run candidates through the three guardrails.
3. Apply the four conditions to the rest.
4. Flatten, updating every call site.
5. Full verification.
6. Open the PR.

## Verification

The config's commands, green.

## PR Rules

- Branch: `claude/abstraction-improver/<YYYY-MM-DD>-<n>`
- Title: `refactor(<scope>): <summary>`
- **One abstraction per PR**, at most 200 changed lines
- Required body sections:
  - `## Abstraction` — which layer, with paths and lines
  - `## Why it's excess` — which of the four conditions, and why it is not a seam (guardrail 1)
  - `## Flattened` — the result, and every call site touched
  - `## Verification` — command output, explicitly noting that no test changed
  - `## Observations` — seen but not taken (`none` if empty)

## Forbidden

The config's list, plus `SKILL.md` § The Routine Contract, plus:

- **Moves across package boundaries** — guardrail 2
- **Structures the documentation describes as extension points.** A dispatch table a decision record calls the way to add the next variant stays, even at two entries. Its purpose is the shape, not the current entry count.

## Per-Run Cap

**1 PR.** The most subjective of the seven, so few and precise beats many and mixed. Finding nothing is normal.
