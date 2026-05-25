---
name: investigate-issue
description: Use when an issue needs deeper analysis than triage — confirming root cause by reading code, verifying file/line references in body are still valid, checking blast radius across the codebase, or proposing a concrete fix direction. One issue per invocation.
---

# Investigating a GitHub Issue

## Overview

Investigation goes beyond labels to verify the issue's claims against actual code, confirm root cause, and propose a fix direction. Output is a comment that either:

- **Confirms** — claims verified against HEAD, fix direction stated, ready for pickup
- **Dismisses** — not reproducible / already fixed / wrong premise — close it
- **Needs-info** — missing critical detail; tag `status/needs-info` and stop

**Investigation ≠ triage**: 30 min – hours per issue, not 5 min. Don't run it on the whole backlog. Pick issues that earn it.

## When to Use

- Body is sparse (title + one-liner) — claims need substantiating
- Body references file:line that may have moved / been refactored since
- An old issue (>3 months) — verify the bug still exists at HEAD
- About to assign to a fix branch — defensible direction needed first
- The user says "investigate", "deep-dive", or "verify"

**Don't use** when light triage is enough (body is thorough, claims unambiguous, only need labels).

## Process

1. **Read the body completely** — note every claim:
   - File paths / line numbers referenced
   - Reproduction steps
   - Suggested fix
   - Affected scope (e.g. "10 endpoints", "all SSE routes")

2. **Verify the claims** — open each file:line:
   - Does the described code exist at HEAD?
   - Has the file moved, been renamed, or been refactored?
   - Is the bug still present?
   - Run repro steps if applicable.

3. **Probe blast radius** — don't trust the body's enumeration:
   - `grep` / Glob for the same anti-pattern elsewhere
   - `git blame` on suspect lines — recent change? old?
   - Check related tests — would they catch this? do they exist?

4. **Decide outcome**:
   - **Confirmed + scoped**: post comment with verified claims + concrete fix direction
   - **Confirmed but body's fix is wrong / incomplete**: explain why, propose alternative
   - **Dismissed**: explain why → close
   - **Needs-info**: ask specific questions, label `status/needs-info`

5. **Apply outcome**:
   - Update labels if priority changes after investigation
   - Post comment via template below
   - Optionally `assignees`, link to draft PR

## Comment Template

Match repo's PR template style — emoji-headed sections for at-a-glance scanning. Match issue body's language. Skip sections that don't apply (don't leave empty headers).

```markdown
## 🔬 Investigation YYYY-MM-DD

**Verdict**: confirmed · dismissed · needs-info
**Adjusted priority**: <only if triage call changed after investigation>

### 📊 At a Glance

<diagram (mermaid or ascii) + 1-line caption — show the verified mechanism, not a text summary>

Use a diagram when investigating:

- Security: `sequenceDiagram` of attack chain confirming where the actual exploit happens
- Race condition: `sequenceDiagram` of concurrent timeline
- Refactor: `flowchart` of current vs proposed module shape
- Cluster of broken-together bugs: `flowchart LR` of dependency

Skip the section if the verdict line + Verified bullets already make the picture obvious.

### ✅ Verified

- <file:line still valid as of `<sha>` / matches body's description>
- <reproduction confirmed via `<command / scenario>`>
- <blast radius: N additional sites at `<paths>`>

### ❌ Body inaccuracies

<only if applicable — e.g. "body says fix is at A, actual root cause is B because...">

### 🛠 Proposed Direction

<2-4 lines — concrete fix path. File paths + shape of change. Not full code.>

### ❓ Open Questions

<only when investigation surfaces decisions / tradeoffs / verification gaps that the issue owner needs to resolve before fix can land>

Format each as: `**<question>** — <type>. <your lean if any>`
Types: `decision needed` · `tradeoff` · `verification gap`

Skip the section when there's nothing genuinely undecided. Forced "questions" become noise.

### 🔗 Related

<cross-links if relevant>
```

## Difference from Triage

|                     | Triage           | Investigation       |
| ------------------- | ---------------- | ------------------- |
| Reads body          | yes              | yes                 |
| Reads code          | **no**           | **yes**             |
| Verifies file:line  | no               | **yes**             |
| Probes blast radius | no               | **yes**             |
| Proposes fix        | only echoes body | **own analysis**    |
| Time / issue        | 2–10 min         | 30 min – hours      |
| Scope               | whole backlog    | one issue at a time |

If you start reading code during a triage, you've crossed into investigation — finish that one issue properly. Don't half-investigate.

## Common Mistakes

- **Trusting body's file:line without checking** — code moves. The body might be 3 months stale. Always verify.
- **Stopping at the body's suggested fix** — body is the reporter's hypothesis, not proven. Verify it actually addresses root cause.
- **Skipping blast radius** — fix the one site mentioned, miss 4 identical sites elsewhere.
- **Investigating the whole backlog** — doesn't scale. Pick high-leverage issues.
- **Proposing speculative fixes** — only propose what you've verified at HEAD. If you can't verify, label `status/needs-info` and stop.
- **Empty section headers** — sections that don't apply should be omitted, not filled with "N/A".
- **Stuffing decisions into Proposed Direction** — if there's a real tradeoff or verification gap, surface it under Open Questions instead of burying it in the proposal. The owner should see the question explicitly, not have to mine it out of the recommended path.
