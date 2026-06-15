# pr-review GitLab support — design

Date: 2026-06-15
Scope: `skills/pr-review/SKILL.md` only (role prompts untouched).

## Problem

`pr-review` publishing and sticky discovery are 100% GitHub (`gh api repos/...`).
Against a GitLab MR every `gh api` call targets github.com and fails. A capable
Claude session running `/pr-review` on a GitLab MR then *improvises* `glab`
commands to fulfil the skill's described intent — but the skill gives no
deterministic GitLab procedure, so each run improvises differently.

Observed signature on `gitlab.jkopay.app/found/jkopay-agents/-/merge_requests/64`:
three separate sticky notes (`note_142287` sha=`0cdd841c`, `note_142791`
sha=`d981e127`, `note_142809` sha=`1701b101`), each a brand-new note carrying its
own `<!-- pr-review:sha=... -->` marker — instead of one note edited in place.

Root cause chain: no glab path → improvised publish POSTs a *new* sticky each run
→ discovery finds multiple stickies with no "edit this one" rule → cannot resolve
a reliable `last_sha` → **falls back to `full` every time**. This is the reported
symptom ("sticky 不 work，每次跑 full").

## Goal

Give `pr-review` a first-class, deterministic GitLab branch with full parity to
the GitHub path (sticky + commit status + inline), so Claude stops improvising and
incremental mode works.

Constraint (CLAUDE.md invariant #6): repo-agnostic. No hardcoded hostnames —
self-hosted GitLab must work generically.

## Non-goals

- No change to the 4-role subagent dispatch (invariant #2). Role prompts are
  platform-agnostic (verified: 0 platform references) and emit findings as data.
- No GitLab review-batching emulation — GitLab has no "single review submission".
- Not building a `glab`-based `pr-babysit` reply path (that already exists).

## Design

### Structure (approach A — platform adapter table)

Add a `## Platform` section + a per-operation endpoint mapping table
(`gh api` ↔ `glab api`). Mode Detection / Publishing prose references "the
platform's notes/sticky endpoint" instead of hardcoding `gh`. Matches the
per-action platform table convention already used by `pr-babysit`. Rejected:
duplicating Mode Detection + Publishing per platform (drift risk).

### Platform detection (generic)

- `pr` is a full URL → parse host: `github.com` → `gh`; URL path contains
  `/-/merge_requests/` → `glab`.
- `pr` is `<owner>/<repo>#<N>` shorthand (no host) → detect from
  `git remote get-url origin` host (github.com → gh; else glab). Same convention
  as `pr-babysit`.
- Self-hosted GitLab works automatically: `glab` reads its own host config.
- Identifiers: GitHub → `$OWNER/$REPO/$N`. GitLab → `$PROJECT` (URL-encoded
  `namespace/path`, e.g. `found%2Fjkopay-agents`) + `$IID` (MR number).

### Channel mapping (all three verified live against MR #64)

| Channel | GitHub (current) | GitLab (new) |
|---|---|---|
| Sticky | `gh api .../issues/:N/comments` POST / **PATCH** | `glab api .../merge_requests/:iid/notes` find marker → **`PUT .../notes/:id`** in place |
| Commit status | `POST .../statuses/:sha` `context=pr-review` | `glab api -X POST .../statuses/:sha -f state=<success\|failed> -f name=pr-review -f target_url=<sticky>` |
| Inline | one review, batched comments | per finding: **`glab api -X POST .../merge_requests/:iid/discussions --input <json>`** |

### Sticky discovery + in-place update + self-heal

- Discover: `glab api ".../merge_requests/:iid/notes?per_page=100"`, filter bodies
  containing `<!-- pr-review:sticky -->`. **Pick the newest match** (highest id).
- `last_sha` ← that note's `<!-- pr-review:sha=... -->`.
- Publish: if a sticky exists → `PUT .../notes/:id` (edit in place); else
  `POST .../notes`. Never POST a new sticky when one exists.
- **Self-heal (decision #1, approved)**: when discovery finds >1 sticky, keep the
  newest and `DELETE .../notes/:id` the rest (all are the skill's own superseded
  duplicates). Cleans the existing 3-sticky mess on #64 and prevents recurrence.
- Verified: `POST → PUT (body changes in place) → DELETE → GET 404`.

### Inline — JSON body, not bracket flags (verified correction)

GitLab diff-anchored comments **must** be sent as a JSON request body via
`glab api --input <file> -H "Content-Type: application/json"`. Passing
`-f "position[new_line]=..."` form flags **silently downgrades** to an
un-anchored `DiscussionNote` (no error). Confirmed against #64: bracket flags →
`DiscussionNote`/no position; `--input` JSON → `DiffNote` with position.

Per-finding JSON body:

```json
{
  "body": "<finding markdown with pr-review:finding-root markers>",
  "position": {
    "position_type": "text",
    "base_sha": "<diff_refs.base_sha>",
    "start_sha": "<diff_refs.start_sha>",
    "head_sha": "<diff_refs.head_sha>",
    "old_path": "<file>",
    "new_path": "<file>",
    "new_line": <line>
  }
}
```

- `diff_refs` (base/start/head sha) comes from the MR metadata GET.
- For added/context lines: set both `old_path` and `new_path` to the file path,
  set `new_line`, omit `old_line`. (Ground-truthed from a real DiffNote on #64.)
- No batch: loop one POST per P0/P1/P2 finding. Document that GitHub's "single
  review submission" property is GitHub-only. Repeated-finding-in-later-iteration
  rule unchanged (new root, same `F<n>` id); reply-into-thread stays pr-babysit's job.

### Commit status + caveat (decision #2, approved)

`glab api -X POST .../statuses/:sha -f name=pr-review -f state=<success|failed>
-f description=<short> -f target_url=<sticky-url>`. State map: PASSED /
PASSED_WITH_NOTES → `success`; REVIEW_BEFORE_MERGE / BLOCKED / PARTIAL → `failed`.
SKILL.md caveat: if the project enforces required status checks, a `failed`
`pr-review` status can block merge — this is parity with GitHub's commit status
and intentional; flag it so it is not a surprise. Note: GitLab has no
delete-status API (a status is only superseded by a newer post on the same name).

### Mode Detection + wording touch-ups

- Mode Detection sticky discovery / noop / SHA-reachability prose generalized to
  "the platform's sticky note".
- `dry-run` doc: "no GitHub API writes" → "no remote API writes (GitHub/GitLab)".
- Inputs `pr`: document the GitLab MR URL form alongside `<owner>/<repo>#<N>`.
- "What this skill does NOT do" / Publishing intro: "GitHub PR" → "PR/MR".
- Markers (`pr-review:sticky`, `:sha`, `:finding-root`, …) unchanged — HTML
  comments render in both GitHub and GitLab markdown.

## Verification evidence (live, MR #64)

- Sticky: POST note → PUT edits body in place → DELETE → GET returns 404. ✓
- Inline: `--input` JSON body → `DiffNote` with `position.new_line=137`. ✓
  (bracket-flag attempt → `DiscussionNote`, no position. ✗ — drove the correction)
- Commit status: `POST .../statuses/:sha name=pr-review-probe state=success` →
  created `id=254615`. ✓
- Cleanup: all probe notes deleted (0 remain). One residual artifact: commit
  status `pr-review-probe=success` (cannot be deleted via GitLab API).

## Residual / follow-up

- `pr-review-probe` commit status lingers on #64 head sha (no GitLab delete API).
- The 3 legacy zombie stickies on #64 will be cleaned by the self-heal path on the
  next real `pr-review` run.
