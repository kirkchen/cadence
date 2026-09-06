# Routine: abstraction-police

Fix layering violations — code that breaks an architectural rule the repo has already written down.

This archetype is only viable in a repo where the rules are **written**, not folklore. Your job is to check code against documented constraints. It is emphatically not to invent an opinion about good architecture and enforce it.

## Rule Sources — The Only Ones

The config's **Spec sources** section names them: architecture constraints, decision records, and whatever they link to. Read them completely before starting.

If the config names no spec sources, this archetype **cannot run**. Report that and stop — without written rules there is no violation, only preference.

The written rules are authoritative as of this run. They change; re-read them each time rather than working from what you remember.

## Prefer A Lint Rule Over A Fix

Before fixing anything, ask what class the violation belongs to:

1. **Will it recur?** Then the valuable output is a **lint rule**, not a patch. Open a PR adding the rule, following whatever custom-lint pattern the repo already uses, and list every current violation in the PR body. Fixing those violations is a *separate* PR.
2. **One-off drift?** Fix it directly. One violation per PR.
3. **Already caught by an existing lint rule or type check?** Then it is not drift and you have misread something. Do not open a PR.

Ordering matters: a rule that stops the tenth recurrence is worth more than nine patches, and it converts a subjective argument into a mechanical check.

## Qualification

- The violation maps to a **specific written rule**. The PR body cites it — decision-record number, or document and section.
- No matching written rule → not this archetype's business. `## Observations`.
- The fix preserves external behaviour. If correcting the violation necessarily changes behaviour, do not do it — that is a change proposal for a human, so `## Observations`.

## Verification

The config's commands, green.

For a lint-rule PR, additionally demonstrate the rule works: the lint command **red before** the violations are fixed and **green after**. Paste both. A rule that was green the moment it landed has not been shown to detect anything.

## PR Rules

- Branch: `claude/abstraction-police/<YYYY-MM-DD>-<n>`
- Title: `refactor(<scope>): <summary>`, or `build(lint): <summary>` for a new rule
- One violation (or one lint rule) per PR
- Required body sections:
  - `## Rule` — which written rule, with its citation
  - `## Violation` — path and line, and why it violates the rule
  - `## Fix` — what changed, and why external behaviour is unchanged
  - `## Verification` — command output
  - `## Observations` — suspicious things with no written rule behind them (`none` if empty)

## Forbidden

The config's list, plus `SKILL.md` § The Routine Contract — and with particular force here:

**The rule documents are not editable by this routine.** Architecture docs and decision records are the oracle you are judging code against. Changing them to match the code inverts the entire exercise. If you believe a rule is wrong or obsolete, say so in `## Observations` and let a human decide whether to amend it.

## Per-Run Cap

**2 PRs.** No violation of a written rule → open nothing.
