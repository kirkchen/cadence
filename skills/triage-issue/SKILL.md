---
name: triage-issue
description: Use when a GitHub issue needs triage — typically one carrying status/needs-triage label, a freshly opened issue without type/priority labels, or when the user says "triage #N" / "triage this issue". One issue per invocation; for backlog walkthroughs, the caller loops.
---

# Triaging a GitHub Issue

## Overview

Triage = assign `type/` + `priority/` labels, leave a key comment when there's judgment worth recording, then remove `status/needs-triage`. Inspect the repo's label schema via `gh label list` — the `type/*` / `priority/*` / `status/*` convention used below is a common default but adapt to what the repo actually has.

**Scope**: one issue per invocation. For backlog walkthroughs, the caller enumerates `gh issue list --label status/needs-triage` and invokes this skill per result.

**Core principle**: comments are signal, not ceremony. Skip when the issue body already covers what you'd say. Only add information the reader can't get from the body or labels alone.

## When to Use

- User says "triage #N" / "triage this issue"
- An issue lacks `type/` or `priority/` labels
- An issue carries `status/needs-triage`

For backlog work (`triage backlog` / many issues at once), the caller orchestrates: list issues, invoke this skill per issue.

## Process

1. **Read the issue**: `gh issue view <n>`
2. **Read body** before deciding (title alone misses scope)
3. **Decide labels**: type/ + priority/
4. **Decide comment**: see Comment Criteria below
5. **Apply**:
   ```bash
   gh issue edit <n> \
     --add-label type/X --add-label priority/Y \
     --remove-label status/needs-triage
   gh issue comment <n> --body-file -  # stdin, multi-line safe
   ```

## Priority Rubric

| Level      | Trigger                                                                              |
| ---------- | ------------------------------------------------------------------------------------ |
| `critical` | Auth bypass · data loss · prod outage · live exploitation. Drop everything.          |
| `high`     | Security with realistic attack path · core flow broken · blocks other work           |
| `medium`   | Should fix · degraded UX · refactor with concrete benefit · no user-visible breakage |
| `low`      | Polish · cleanup · nice-to-have · low-risk follow-up                                 |

Anchor: who is affected, how reversible the damage is, blast radius. If unsure between two tiers, downgrade.

## Comment Criteria

**Always comment when** any of these apply:

- `priority/critical` or `priority/high` — record the call so it's defensible later
- Issue overlaps another (`see also #N` / `dup of #N` / `subset of #N` / `blocks #N`)
- Scope estimate is non-obvious (XS / S / M / L, or hours/days)
- A blocker exists (`blocked by #N` or `requires upstream X`)

**Skip the comment when**:

- Title + body already cover priority justification AND there's no cross-reference / scope / blocker to add
- Comment would just restate what's in the body

**Don't repeat the body**. If the author already explained why this is critical, the triage comment is for _meta_ information: relationships, scope, dates, defensibility.

**Don't read code during triage**. If you find an issue needs claims verified against actual code, root cause confirmed, or blast radius probed — that's investigation. Switch to the `investigate-issue` skill and do that one issue properly. Half-investigation makes comments worse than light triage.

## Comment Template

Mirrors the repo's PR template style — emoji-headed sections for at-a-glance scanning. Sections that add nothing get omitted (don't write "N/A" headers; just skip).

```markdown
## 🏷️ Triage YYYY-MM-DD

`type/X` · `priority/Y` · effort: **S/M/L**

### 📊 At a Glance

<diagram (mermaid or ascii) + 1-line caption — only include when there's something visual worth showing>

Use a diagram when:

- Cluster fix order: `#29 → #30 → #25` (ascii arrows OK)
- Attack flow / race condition: mermaid `sequenceDiagram`
- Before/after refactor: mermaid `flowchart` or ascii box

Skip the section entirely (don't write empty header) when labels + Related list already convey the picture — most simple triages won't need it.

### 🔗 Related

- <only if relevant: dup of / blocks / blocked by / see also #N>

### 🚧 Notes

<only if priority justification missing from body, or scope is non-obvious>
```

**Language**: match the issue body. Keep label values, command names, and section headings (`At a Glance`, `Related`, `Notes`) as-is.

**Sizing**: XS = <1h · S = half-day · M = 1–3d · L = >3d. Estimate from blast radius, not from line count.

**Section budget**:

- `🏷️ Triage` line + `📊 At a Glance` — always include (this is the load-bearing summary)
- `🔗 Related` — include when there's a real cross-link, otherwise drop the heading
- `🚧 Notes` — include only when body lacks priority justification or scope is non-obvious; otherwise drop the heading

**Don't** restate what the body already says. The triage comment is meta-information about the _call_, not a re-summary of the _issue_.

## Quick Reference

| Command                                                                                | Purpose                                                              |
| -------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `gh issue view <n> --json title,body,labels`                                           | Read the issue                                                       |
| `gh issue edit <n> --add-label X --remove-label Y`                                     | Apply labels                                                         |
| `gh issue comment <n> --body-file -`                                                   | Comment via stdin                                                    |
| `gh issue list --label status/needs-triage --json number,title,body,labels --limit 50` | List backlog (caller enumerates, then invokes this skill per issue)  |

## Common Mistakes

- **Repeating the body in the comment** — adds noise. The body is already there; comment for meta only.
- **Commenting on every issue** — silence on a clear medium/low is correct.
- **`priority/critical` for "important but not drop-everything"** — reserve critical for true emergencies. Default to `high` when in doubt.
- **Forgetting `--remove-label status/needs-triage`** — leaves the issue stuck in backlog filter forever.
- **Date drift** — always use today's date in the comment header so future you knows when the call was made.
