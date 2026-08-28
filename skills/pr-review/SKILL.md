---
name: pr-review
description: Use when reviewing a PR/MR diff for security, logic, performance, cross-file impact, test coverage, or spec compliance findings. NOT for writing PR descriptions, design reviews requiring business judgment, implementation work, release notes, or deep CVE/supply-chain audits.
---

# PR Review

Review a PR/MR diff by dispatching independent role-based subagents in parallel, then publish findings as one sticky summary comment + per-finding inline comments. The main session never reviews — it ingests, dispatches, merges, emits, publishes.

<HARD-GATE>
You MUST dispatch independent subagents — NEVER review the diff yourself in the main session. The main session accumulates context bias from prior conversation. Only an isolated subagent can deliver an unbiased finding.

Dispatch in PARALLEL using a single message with multiple Agent tool calls, **every one of them with `run_in_background: false`**. Parallel means "one message, many calls" — it does NOT mean background. Nothing in this skill can be produced until every dispatched report is in hand, so there is no useful work to interleave; a turn that ends while a report is still outstanding throws that report away and publishes nothing.

If one subagent fails, proceed with the rest BUT surface the failure in the sticky comment header (never silent). If ALL fail, report failure — do NOT fall back to self-review.

Publishing happens in the main session (post-merge) — not in subagents.

`mode: local` does NOT relax this gate. Local mode changes the output target (JSON to stdout instead of the PR/MR sticky/inline), not the reviewer. The 4-subagent parallel dispatch is exactly the property that makes local mode worth invoking from a supervisor session — without it the caller could just self-review.
</HARD-GATE>

## Rationalization Prevention

| Thought                                        | Reality                                                                                               |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| "The diff is small, I can review it myself"    | Self-review is biased by what you saw in the conversation. Small ≠ unbiased.                          |
| "I already saw this code earlier"              | That's exactly why you can't review it. Familiarity hides issues.                                     |
| "Dispatching 3-4 subagents is overkill"        | Each persona uses a different mental model. A single agent dilutes all of them.                       |
| "Sequential is fine, I'll save tokens"         | Parallel is faster wall-clock and prevents one report from biasing the next.                          |
| "Background is fine, I'll be notified when each finishes" | Only if you are still alive to be notified. Non-interactive hosts end the run when your turn ends — the outstanding reports never arrive and the review publishes nothing. |
| "I'll report progress while the rest finish"   | Progress narration ends your turn. There is no partial output worth shipping — dispatch blocking and say nothing until all reports land. |
| "spec-auditor isn't needed, the spec is short" | If has_spec is true, dispatch. The check is whether spec exists, not whether it's verbose.            |
| "I'll just check the obvious bug myself"       | Even one self-checked finding contaminates the report — readers can't tell which findings are biased. |
| "1 subagent failed, just hide it"              | Hiding partial review = pretending coverage existed. Surface it in sticky header.                     |
| "Prior finding line moved, mark fixed"         | Line moving ≠ behaviour fixed. Require subagent verification, then hedge as `Likely fixed`.           |

## Red Flags — STOP if you catch yourself:

- Reviewing any category yourself instead of dispatching
- Dispatching subagents sequentially instead of in parallel
- Dispatching without `run_in_background: false`, or ending a turn with any report still outstanding
- Skipping a subagent because "the diff doesn't look like it has X"
- Claiming review passed without reading subagent findings
- Editing code during review (review reads, doesn't write)
- Falling back to self-review because a subagent failed
- Hiding subagent failures in output (must surface in sticky header)
- Marking prior findings as `✅ Fixed` without a verification note from subagent
- Publishing inline comments before merging findings (dispatch → merge → publish)

<decision_boundary>

**Use for:**

- Reviewing a PR/MR diff and producing structured findings
- Security / logic / performance scans on code changes
- Spec compliance verification when spec exists
- Organizing findings with severity + blast radius + confidence
- Posting findings to the PR/MR as sticky summary + inline comments

**NOT for:**

- Writing or improving PR descriptions
- Design review requiring business judgment about scope or direction
- Writing the actual code/diff
- Deep CVE / supply-chain / OWASP sweep
- Writing release notes / CHANGELOG
- End-user / UX review (use /qa or /design-review)
- Auto-approving or auto-merging (produce findings only; humans merge)

</decision_boundary>

## Flow

```dot
digraph pr_review {
    "Receive inputs" [shape=doublecircle];
    "Resolve mode" [shape=box];
    "Compute capability flags" [shape=box];
    "Harvest author replies" [shape=box];
    "Apply volume caps" [shape=box];
    "Parallel dispatch" [shape=box, style=bold];
    "security-reviewer" [shape=box];
    "staff-engineer" [shape=box];
    "sdet" [shape=box];
    "has_spec?" [shape=diamond];
    "spec-auditor" [shape=box];
    "Skip spec-auditor" [shape=box];
    "Collect + Dedup" [shape=box];
    "Apply severity merge rule" [shape=box];
    "Build sticky + inline" [shape=box];
    "dry-run?" [shape=diamond];
    "Print to console" [shape=box];
    "Publish to PR" [shape=box];
    "Done" [shape=doublecircle];

    "Receive inputs" -> "Resolve mode";
    "Resolve mode" -> "Compute capability flags";
    "Compute capability flags" -> "Harvest author replies";
    "Harvest author replies" -> "Parallel dispatch";
    "Parallel dispatch" -> "security-reviewer";
    "Parallel dispatch" -> "staff-engineer";
    "Parallel dispatch" -> "sdet";
    "Parallel dispatch" -> "has_spec?";
    "has_spec?" -> "spec-auditor" [label="yes"];
    "has_spec?" -> "Skip spec-auditor" [label="no"];
    "security-reviewer" -> "Collect + Dedup";
    "staff-engineer" -> "Collect + Dedup";
    "sdet" -> "Collect + Dedup";
    "spec-auditor" -> "Collect + Dedup";
    "Skip spec-auditor" -> "Collect + Dedup";
    "Collect + Dedup" -> "Apply severity merge rule";
    "Apply severity merge rule" -> "Apply volume caps";
    "Apply volume caps" -> "Build sticky + inline";
    "Build sticky + inline" -> "dry-run?";
    "dry-run?" -> "Print to console" [label="yes"];
    "dry-run?" -> "Publish to PR" [label="no"];

    "Apply severity merge rule" -> "Emit findings JSON" [label="mode=local"];
    "Emit findings JSON" [shape=box];
    "Emit findings JSON" -> "Done";
}
```

## Inputs

### Required

**`pr`** — `<owner>/<repo>#<N>` or a full PR/MR URL (GitHub PR, or GitLab `/-/merge_requests/<iid>`). Used for platform detection, sticky lookup, and publishing — see [Platform](#platform). No implicit branch-based detection (too brittle).

> **Exception**: `mode: local` does not require `pr` (no remote side effects). See [Local Mode](#local-mode).

### Optional

**`mode`** — `auto` (default) / `full` / `incremental` / `local`. See [Mode Detection](#mode-detection) and [Local Mode](#local-mode).

**`dry-run`** — `true` / `false` (default `false`). When true, builds sticky + inline payloads and prints to console; no remote API writes (GitHub/GitLab).

**`base`** — base git ref for diff scope (e.g. `origin/main`). Used by **local mode only** — replaces the sticky/PR-based base lookup. Ignored for other modes (they derive base from the PR or repo state).

**`last_sha`** — prior reviewed HEAD commit. Used by **local mode only** for incremental review. In non-local modes the last_sha lives in the sticky body; local mode has no sticky, so the caller (e.g. supervisor) must pass it explicitly.

**`spec`** — source of the spec / design doc. Sets `has_spec` flag.

- Path: `spec: docs/specs/payment-v2.md`
- URL: `spec: https://confluence.example.com/payment-v2`
- Inline: `spec: this PR implements PCI DSS v4.0 logical isolation`
- Multiple: `spec: design doc at X, acceptance criteria in Jira ABC-123`

If absent → spec-auditor not dispatched. Other subagents do not reference spec.

**`test direction`** — tells sdet how to evaluate test coverage. All sub-fields optional:

- **Approach**: `unit only` / `integration required` / `e2e required` / `no test needed`
- **Location**: expected test file path
- **Focus**: scenario or case the test should cover

Missing → sdet uses heuristic from diff nature.

**`context`** — free-form supplementary signals:

- **Business risk**: "this endpoint is internal-admin only"
- **Domain rules**: "tenant_id is required"
- **Known trade-offs**: "we're aware of the N+1, will fix next sprint"
- **Environment constraints**: "CDE service, security findings cannot be downgraded"
- **Hotfix narrowing**: "hotfix — only check critical security"
- **Cross-PR coupling**: "ships together with PR #1234"

Context can adjust severity at merge time (see Severity Merge Rule).

**`repo rules`** — compact repo-specific review constraints sourced from committed project files, not from conversation memory. Use this when a repo has hard invariants, ADRs, testing conventions, glossary terms, module READMEs, or accepted design trade-offs that reviewers must respect.

Examples:

- "ADR-0016: Slack is single-workspace; do not require workspace isolation in this PR."
- "DB migrations are forward-only expand/contract; migration SQL must have artifact-backed coverage."
- "E2E step definitions must not use networkidle or raw sleeps."
- "MCP default-all is a UI default, not a server-side create contract."

Repo rules are review inputs, not output metadata. Do **not** add a "rules source" / "review inputs" section to the sticky.

## Platform

`pr-review` publishes to **GitHub** (via `gh`) or **GitLab** (via `glab`). Resolve `$PLATFORM` before Mode Detection — it selects every discovery and publish endpoint below. Both platforms use the same sticky/inline markers (HTML comments render in either markdown), so only the transport differs.

### Detection

- `pr` is a full URL → inspect host + path:
  - host `github.com`, or path contains `/pull/` → `gh`
  - path contains `/-/merge_requests/` → `glab`
- `pr` is `<owner>/<repo>#<N>` shorthand (no host) → detect from `git remote get-url origin`: `github.com` → `gh`; any other host → `glab`.
- Self-hosted GitLab: derive the host from the MR URL (or `git remote get-url origin`) — **never hardcode a hostname** — and pin `glab` to it (see [glab auth](#glab-auth) below). `glab` does **not** infer the host from the token.

### glab auth

`glab` defaults to `gitlab.com`. An ambient `GITLAB_TOKEN` is applied to whatever host `glab` targets — it does **not** redirect `glab` to a self-hosted instance — so `glab api` silently 401s against e.g. `gitlab.example.com`. When `glab` fails auth, a session tends to improvise `curl` fallbacks with hand-rolled sticky parsing; that improvised discovery misses the existing sticky → full re-scan + duplicate sticky (the exact failure this skill exists to eliminate). Prevent it: **pin the host and assert auth before any `glab` call, and abort — never `curl`-improvise — if it fails.**

Shell env does **not** persist across separate tool-call invocations, so set `GITLAB_HOST` **inside each bash block** that calls `glab` (the Mode Detection discovery block and the Publishing block below both do this):

```bash
export GITLAB_HOST="<host parsed from the MR URL — NOT hardcoded>"   # e.g. gitlab.example.com
glab api user >/dev/null 2>&1 || {   # authenticated probe — glab api uses $GITLAB_HOST + $GITLAB_TOKEN
  echo "pr-review: glab not authenticated for $GITLAB_HOST — ABORT (do NOT fall back to curl)." >&2
  echo "  fix: provide GITLAB_HOST + GITLAB_TOKEN in the run env, or 'glab auth login --hostname $GITLAB_HOST'." >&2
  exit 1
}
```

On abort, report the auth failure to the caller; do **not** proceed with a `curl` fallback.

### Identifiers

| | GitHub | GitLab |
| --- | --- | --- |
| Repo / project | `$OWNER/$REPO` | `$PROJECT` = URL-encoded `namespace/path` (`/` → `%2F`) |
| PR / MR number | `$N` | `$IID` (the MR number) |
| Inline diff anchors | n/a | `$BASE` / `$START` / `$HEAD` from `glab api projects/$PROJECT/merge_requests/$IID` → `.diff_refs` |

### Endpoint adapter

| Operation | GitHub (`gh api`) | GitLab (`glab api`) |
| --- | --- | --- |
| Find sticky | `repos/$OWNER/$REPO/issues/$N/comments` | `projects/$PROJECT/merge_requests/$IID/notes?per_page=100` |
| Create sticky | `-X POST .../issues/$N/comments` | `-X POST .../merge_requests/$IID/notes` |
| Update sticky **in place** | `-X PATCH .../issues/comments/$ID` | `-X PUT .../merge_requests/$IID/notes/$ID` |
| Delete stale sticky | `-X DELETE .../issues/comments/$ID` | `-X DELETE .../merge_requests/$IID/notes/$ID` |
| Commit status | `-X POST .../statuses/$HEAD -f context=pr-review` | `-X POST .../statuses/$HEAD -f name=pr-review` |
| Inline comment | one review: `-X POST .../pulls/$N/reviews` (batched) | one discussion **per finding**: `-X POST .../merge_requests/$IID/discussions --input <json>` |

GitLab specifics that bite (verified against a live MR):

- **`glab` must be authenticated for the MR's host.** Self-hosted defaults to `gitlab.com` and the token env alone does not redirect it; set `GITLAB_HOST` per bash block and assert glab can authenticate (`glab api user`), then abort on failure — see [glab auth](#glab-auth). An unauthenticated `glab` pushes the session into `curl` improvisation whose discovery misses the existing sticky → full re-scan + duplicate sticky.
- **Sticky must be edited in place** with `PUT .../notes/$ID`. Posting a fresh note every run is exactly what makes incremental detection fail (multiple stickies, no canonical `last_sha`).
- **Inline position must travel as a JSON body** via `--input`. Passing `-f "position[...]"` form flags is silently dropped and the note lands un-anchored as a plain `DiscussionNote` instead of a `DiffNote`.
- **Commit status state token is `failed`** (GitHub uses `failure`); GitLab also has no delete-status API (a status is only superseded by a newer post on the same `name`).

## Mode Detection

Resolve before dispatch. The mode controls diff scope and output sections.

`mode: local` short-circuits this whole section — no sticky lookup, no SHA reachability check, no noop case. Diff scope comes from the `base` (and optional `last_sha`) inputs; see [Local Mode](#local-mode).

```dot
digraph mode {
  "mode input" [shape=box];
  "Has sticky?" [shape=diamond];
  "last_sha reachable?" [shape=diamond];
  "Same as HEAD?" [shape=diamond];
  "incremental" [shape=box, style=bold];
  "full" [shape=box, style=bold];
  "local" [shape=box, style=bold];
  "noop (report no-change)" [shape=box, style=bold];

  "mode input" -> "Has sticky?" [label="auto"];
  "mode input" -> "incremental" [label="incremental (forced)"];
  "mode input" -> "full" [label="full (forced)"];
  "mode input" -> "local" [label="local (forced)"];
  "Has sticky?" -> "last_sha reachable?" [label="yes"];
  "Has sticky?" -> "full" [label="no"];
  "last_sha reachable?" -> "Same as HEAD?" [label="yes"];
  "last_sha reachable?" -> "full" [label="no\n(force-push?)"];
  "Same as HEAD?" -> "noop (report no-change)" [label="yes"];
  "Same as HEAD?" -> "incremental" [label="no"];
}
```

Sticky discovery (use the platform's "Find sticky" endpoint from [Platform](#platform)):

```bash
# GitHub
gh api repos/$OWNER/$REPO/issues/$N/comments \
  --jq '.[] | select(.body | contains("<!-- pr-review:sticky -->")) | {id, body}'
# GitLab — pin host + assert auth first (see § glab auth), then --paginate + `jq -s add`
# so an early sticky (kept at its original created_at by in-place edits) is still found once
# the MR passes 100 notes — GitLab returns notes newest-first.
export GITLAB_HOST="<host from the MR URL>"
glab api user >/dev/null 2>&1 || { echo "glab not authed for $GITLAB_HOST — ABORT, do not curl-improvise" >&2; exit 1; }
glab api --paginate "projects/$PROJECT/merge_requests/$IID/notes?per_page=100" \
  | jq -s 'add | [.[] | select(.body | contains("<!-- pr-review:sticky -->"))] | max_by(.id)'
```

When more than one sticky matches (legacy duplicates from older runs), the **newest (highest id)** is canonical — its body holds the authoritative `last_sha`. Publishing edits that one in place and deletes the rest (see [Publishing](#publishing)).

Markers embedded in sticky body:

- `<!-- pr-review:sticky -->` — locator
- `<!-- pr-review:sha=<commit> -->` — last reviewed HEAD

SHA reachability:

```bash
git cat-file -e <last_sha> 2>/dev/null && echo reachable || echo unreachable
```

If unreachable (force-push / squash-merge of older PR / branch rebased): fall back to `full` AND prepend to sticky body:

```markdown
> ⚠️ Prior review base `<last_sha>` is not reachable (force-push?). This iteration is a full re-review.
```

### Noop case (`last_sha == HEAD`)

When the sticky exists AND `last_sha == HEAD`: skip dispatch + publish. Print to console:

> `pr-review: nothing new since <last_sha>. Skipping. Use mode=full to force a re-review.`

The sticky is already current; do not touch it.

## Local Mode

Use when the caller is **another skill or supervisor session** that needs unbiased multi-role review of a diff but has **no PR open yet** (e.g. a supervisor session's verify phase doing pre-PR critique). The HARD-GATE still applies — local mode is about output target, not about who reviews.

> **Caveat — calling from the same dev session that wrote the code (author-as-reviewer bias)**: pr-review's 4-subagent dispatch is isolated by design — finding generation is robust even when called from the author's session. **But the downstream `modify / wontfix / defer` verdict on each finding is NOT covered by this isolation.** If the same session that wrote the code also reasons about which findings to wontfix, author-narrative bias compounds — framing a diff as "bug-free" produces the strongest detection drop among framing conditions tested across 6 LLMs (Mitropoulos et al., *Measuring and Exploiting Contextual Bias in LLM-Assisted Security Code Review*, [arXiv:2603.18740](https://arxiv.org/abs/2603.18740)). Treat local-mode findings as **advisory** in dev sessions; do **NOT auto-execute verdicts in main session**. A proper dev-stage verdict loop needs a separate Deriver-pattern verdict-subagent (not built yet — see `pr-babysit/SKILL.md` § 4.6 "When NOT to use" for the equivalent caveat on Wontfix Template).

### Inputs

- `mode: local` (required to enter this mode)
- `base: <ref>` (required — e.g. `origin/main`)
- `last_sha: <sha>` (optional — if provided, runs incremental on `<last_sha>..HEAD` and still reads `<base>...HEAD` for cumulative context that subagents need for prior-finding verification)
- `spec`, `test direction`, `context` — same semantics as default mode

`pr` is NOT required and ignored if provided.

### Diff scope

- No `last_sha` → full diff: `git diff <base>...HEAD` (three-dot — topic-only changes)
- With `last_sha` → incremental: subagents see both `<base>...HEAD` and `<last_sha>..HEAD`; they report findings only inside the incremental window plus verification status for prior findings (caller must pass prior findings too — see below)

### Caller responsibilities (incremental local mode)

The sticky normally carries prior findings between iterations. In local mode the caller owns that state and must pass to pr-review on each invocation:

- `prior_findings`: array of objects with `{id, slug, file, line, category, severity, justification, summary}` — same shape as findings JSON output (see below)
- `prior_fix_range`: `<first-fix-sha>^..<last-fix-sha>` — the commits that addressed iter (N-1) findings, used by the threshold's drop signal (B)

If `last_sha` is set but `prior_findings` is missing → ESCALATE to caller; do not fabricate.

### Output

Skip Publishing. Skip sticky/inline markdown construction. Emit one JSON document to stdout:

```json
{
  "mode": "local",
  "base": "origin/main",
  "head": "<HEAD sha>",
  "last_sha": "<sha or null>",
  "status": "PASSED | PASSED_WITH_NOTES | REVIEW_BEFORE_MERGE | BLOCKED | PARTIAL | NOOP",
  "status_heading": "✅ pr-review: PASSED | 🟡 pr-review: PASSED WITH NOTES | 🟠 pr-review: REVIEW BEFORE MERGE | 🔴 pr-review: BLOCKED | ⚠️ pr-review: PARTIAL",
  "open_counts": { "P0": 0, "P1": 0, "P2": 0, "P3": 0, "Q": 0 },
  "subagent_failures": [],
  "next_action": "<one-line or null>",
  "findings": [
    {
      "id": "F1",
      "p_code": "P0 | P1 | P2 | P3 | Q",
      "severity_emoji": "🚨 | ⚠️ | 💡 | 🔧 | ❓",
      "slug": "kebab-case-slug",
      "category": "Original [code name] from subagent",
      "file": "path/to/file",
      "line_start": 42,
      "line_end": 42,
      "confidence": "high | medium | low",
      "blast": "Local | Module | Cross-service | Data layer",
      "justification": "Reachable | Precedent | Asymmetric | Historical",
      "failure_mode": "one-line",
      "mitigation": "one-line",
      "evidence": "verbatim diff line(s)",
      "details": "optional multi-line",
      "disposition": "open | likely_fixed | still_present | follow_up | wontfix | by_design | disputed",
      "accepted_exception": null | { "kind": "follow_up | wontfix | by_design", "reason": "...", "issue": "#123 or null", "accepted_by": "<who — never the PR author>" },
      "severity_adjustment": null | { "from": "💡 P2", "to": "🔧 P3", "reason": "..." }
    }
  ],
  "accepted_exceptions": [
    { "finding_id": "F2", "kind": "follow_up | wontfix | by_design", "reason": "one-line", "issue": "#123 or null", "accepted_by": "<who — never the PR author>" }
  ],
  "spec_gaps": [
    {
      "id": "F7",
      "section": "spec section or decision id",
      "title": "one-line",
      "spec_quote": "verbatim",
      "code_quote": "verbatim",
      "questions": ["..."]
    }
  ],
  "prior_verifications": [
    {
      "prior_id": "F1",
      "verification": "yes | unclear | no",
      "note": "what evidence"
    }
  ],
  "checked_and_clean": [
    { "slug": "...", "evidence": "one-line" }
  ]
}
```

`severity_adjustment: null` when no adjustment; the merged severity is already reflected in `p_code` / `severity_emoji`. The adjustment field exists so callers can audit downgrades (same role as the sticky's `## ⚖️ Severity adjustments` section).

`prior_verifications` is empty `[]` when `last_sha` is absent.

### What local mode keeps from default mode

- HARD-GATE: still dispatch 4 parallel subagents with `run_in_background: false`; main session never reviews
- Capability flags (has_spec, has_repo, is_trivial)
- Finding Inclusion Threshold (Reachable / Precedent / Asymmetric / Historical + drop signals A/B/C/D)
- Severity Merge Rule (4 steps + P-code mapping)
- Dedup between subagent findings
- Subagent failure → if all 4 fail, report failure to caller; never self-review

### What local mode drops

- Sticky comment build / markdown rendering
- Inline comment markdown / inline-endpoint call (`gh` review or `glab` discussions)
- Sticky discovery via `gh` / `glab`
- last_sha derivation from sticky body (caller passes it)
- Noop case (caller decides whether to re-invoke; if `last_sha == HEAD` and caller still invokes, return `findings: []` + a `status: "noop"`)

## Capability Flags

Compute before dispatch:

| Flag         | Default | Set when                                                          | Effect                   |
| ------------ | ------- | ----------------------------------------------------------------- | ------------------------ |
| `has_spec`   | false   | spec input present OR PR description has goal/requirement section | dispatch spec-auditor    |
| `has_repo`   | true    | repo access available (grep / index / LSP)                        | enable cross-file checks |
| `is_trivial` | false   | <50 LOC AND (docs-only OR pure rename OR pure type-only)          | skip staff-engineer      |
| `is_prose`   | —       | **per finding**, not per PR: the file the finding cites is a prose/spec file (`.md`, `.feature`, `.rst`, `.adoc`, docs-site config) | apply [prose severity ceiling](#prose-severity-ceiling) to that finding |

### Prose severity ceiling

Applies to any finding whose cited file is prose, **whether or not the PR as a whole looks like a docs PR**. This is deliberately per-finding: a PR that is 70% Python and 30% `.feature` files is not a docs PR, but its findings *on the feature files* have exactly the shape this ceiling exists to bound. Replaying 215 real findings, a per-PR flag at any threshold missed 19 prose findings sitting inside code-heavy PRs — including every finding on the PR with the worst nit rate in the sample, where more than half the findings cited `.feature` and `.md` files while the diff itself was mostly code.

Do **not** skip review of prose diffs — in a repo whose product *is* prose (prompts, specs, runbooks), the text is the behavior. The problem is the opposite one: prose offers an unbounded supply of mechanically enumerable defects (every scenario missing a trace tag, every Then step that is not purely declarative, every section whose sibling has an error path), so an enumerating reviewer produces far more findings on a document than on the code it describes, in inverse proportion to the risk. Prose and config can be polished forever; the ceiling is what stops that.

Under `is_prose`, these classes are capped at **P3** no matter how certain the finding:

- Terminology, wording, tense, declarative-vs-imperative phrasing
- Missing or mismatched trace annotations (`@covered-by` and equivalents)
- Cross-reference line numbers in a design document
- Symmetry gaps — "the sibling scenario has this step / error path / negative case, this one should too"
- Counts and totals that drifted (`11 items` → `12 items`)
- Sweep completeness — "the same term also appears in N files outside this diff" (also check drop signal (F) scope-declared)

**Check the P0–P1 list below first.** The capped classes are shapes, not subjects: a finding *shaped* like "this section does not state X" is capped only after it fails every carve-out. In a replay of these rules, an unstated default-open permission was filed as a symmetry gap and capped at P3 — it belongs to the last carve-out and should have stayed P1. Match against the carve-outs, then fall through to the cap.

These keep full P0–P1 eligibility under `is_prose`:

- The document contradicts **itself** in a way that yields two incompatible implementations
- The document contradicts a **decision record** (ADR, RFC) it claims to implement
- The document specifies behavior that **cannot be verified** by the test mechanism it names
- The document describes a mechanism the **framework does not support**
- The document's stated invariant and its stated mechanism are **mutually exclusive**
- A **safety-relevant default** is left unstated (permission, retention, disclosure)

A prose finding that fits none of the P0–P1 classes and none of the P3 classes is P2.

## Context Hydration

Before dispatch, build a compact context pack from durable inputs. This reduces false positives from reviewers missing accepted scope or repo rules, while preserving subagent isolation from conversation history.

Allowed sources:

- PR body sections: goal, scope boundary, explicitly out of scope, validation, alternatives, accepted follow-ups
- User-provided `spec`, `context`, `test direction`, and `repo rules`
- Beat change artifacts or ADRs explicitly linked in the PR body or user-provided spec/context
- Module README / repo instruction files selected by changed paths when the caller provides them as `repo rules`
- **The dismissal ledger** — author replies harvested per [Reply harvesting](#reply-harvesting), from **both** channels it names: finding threads and standalone PR/MR comments that name the finding they answer. These are durable, attributable, on-the-record artifacts written in response to a specific finding, not conversation. A finding published without a line anchor has no thread, so excluding standalone comments would exclude its only possible reply.

Do not include:

- Chat history, ad hoc "the author probably meant" assumptions, or author statements that do not answer a specific finding on this PR
- A sticky-visible "rules source" section. Context hydration is an input discipline, not PR output.

### Reply harvesting

Run before dispatch on every incremental iteration. Without it the review re-litigates settled findings: the author writes "wontfix, out of scope, here is why" in the thread, the next iteration never sees it, and the same finding comes back — which is the single most-cited reason developers stop trusting a review bot, and the thing they credit tools that *do* carry state with fixing.

1. Fetch **both** reply channels. Authors use whichever is available, and the more considered the rebuttal, the more likely it is not in a thread:
   - **In-thread** — GitHub review comment threads; GitLab discussions whose root carries `pr-review:finding-root`.
   - **Standalone PR/MR comments** — top-level notes not attached to any diff position. A finding published without a line anchor (every Q-class item, every file-level finding) has **no thread to reply in**, so this is the only channel its author has. Long rebuttals also land here because the threading UI is cramped. In the sample this work came from, every substantive rebuttal on two of the MRs was a standalone note, one of them opening with "no inline thread for this one, so replying here".
2. Take the author's notes from both channels — non-system, not authored by the review identity. For standalone notes, attribute them to findings by the ids they name (`F5`, `F12`) or by the category slug they quote; a note naming no finding is not ledger material.
3. Classify each replied-to finding into the dismissal ledger:

| Ledger entry | Author's reply says                                              | Effect on this and later iterations                          |
| ------------ | ----------------------------------------------------------------- | ------------------------------------------------------------ |
| `rebutted`   | the finding's premise is wrong (with reasoning or counter-evidence) | Stop re-emitting. Render under `📋 Currently open` as `⏸️ Author disputes — <reason>`, still counted. |
| `wontfix`    | correct but deliberately not fixed here                            | Stop re-emitting. Render under `📋 Currently open` as `⏸️ Author wontfix — <reason>`, still counted. |
| `deferred`   | accepted, handled in a follow-up / later MR                        | Stop re-emitting. Render as `⏸️ Author defers — <follow-up ref>`, still counted. |
| `fixed`      | claims a fix                                                       | Verify normally against the diff; re-emit only with `🔄 Still present` evidence. |
| `unclear`    | reply exists but states no disposition                             | Treat as no reply.                                            |

**A ledger entry silences the review; it does not clear the finding.** The author writing "wontfix" is a *requested* disposition, and requests do not approve themselves — an author who could close their own P0 by replying to it would have a one-comment bypass around every security finding this skill produces. So the ledger governs re-emission only: the finding stays in `📋 Currently open`, stays in the status-tier calculation, and keeps `REVIEW BEFORE MERGE` / `BLOCKED` on the sticky.

Moving a finding to `↪ Accepted exceptions` — the section that *does* stop it blocking — requires `accepted_by` to name someone who is **not the PR author**. The skill never writes that field from an author reply, in any circumstance. There is no sole-maintainer carve-out: a repo with one maintainer still gets `REVIEW BEFORE MERGE` on the sticky, and that maintainer merges anyway if they judge it right. Merging over an open finding is a human action with a name attached to it; silently recolouring the finding as accepted is not, and a carve-out that lets the author supply their own `accepted_by` is the same one-comment bypass written a second way.

The skill has no way to authenticate who is speaking, so it does not try: it refuses to accept on anyone's behalf and leaves the status honest. Acceptance reaches the sticky through `pr-babysit`, whose caller is the human, or through explicit invocation input naming the accepter.

4. Pass the ledger to every subagent alongside prior findings. Subagents apply drop signal **(E)** against it.

**Re-emitting over a ledger entry requires new evidence** — a later commit that reintroduces the condition, or a fact the author's reasoning did not address. State that evidence in the finding body. "I still think it matters" is not new evidence.

**The ledger is per-PR and does not persist across PRs.** A pattern the author rejects repeatedly across PRs belongs in repo rules, added by a human — not inferred by the review.

Subagents receive the compact context pack, but still emit findings only with quoted diff/spec evidence. Dispatcher uses the same pack for severity calibration, accepted-exception handling, and P0 conservatism.

## Dispatch

Default dispatch (4 subagents in one message, each blocking — see **Blocking dispatch** below):

| Subagent          | Prompt file                   | When dispatched             |
| ----------------- | ----------------------------- | --------------------------- |
| security-reviewer | `security-reviewer-prompt.md` | always                      |
| staff-engineer    | `staff-engineer-prompt.md`    | always (skip if is_trivial) |
| sdet              | `sdet-prompt.md`              | always                      |
| spec-auditor      | `spec-auditor-prompt.md`      | only if has_spec            |

Each subagent receives:

- Diff (full in `full` mode; `<last_sha>..HEAD` in `incremental` mode)
- Capability flags (has_spec, has_repo, is_trivial)
- Mode (`full` / `incremental`)
- Compact context pack from [Context Hydration](#context-hydration)
- Role-specific inputs only where applicable (spec content for spec-auditor, test direction for sdet)
- In `incremental` mode (dispatcher MUST provide all four):
  - Prior findings JSON (subagent's own category scope only)
  - Prior `Checked & clean` slugs for drift spot-check
  - **Dismissal ledger** from [Reply harvesting](#reply-harvesting) — every finding the author rebutted, wontfixed, or deferred, with their stated reason. Subagent applies drop signal (E) against it. An empty ledger is a valid value; a *missing* ledger is not — if replies could not be fetched, say so in the sticky rather than reviewing as if there were none.
  - **`prior_fix_range`**: `<first-fix-sha>^..<last-fix-sha>` — git range covering the commits that addressed iter (N-1) findings. In single-commit-per-iter cases this collapses to `<last_sha>..HEAD`. If the dispatcher cannot determine the range (e.g. force-push, squash-merge of iter N-1 commits) → fall back to `full` mode and announce in sticky; do NOT invoke incremental mode without `prior_fix_range`
  - **`mr_range`**: `<merge-base(target, HEAD)>..HEAD` — the whole PR's history. Drop signal (B) is evaluated against this, not against `prior_fix_range` alone: a line the PR added in commit 3 and removed in commit 9 is not a defect in the PR, and neither is the removal. `prior_fix_range` remains the narrower hint for *which* iteration introduced a surface; `mr_range` is what decides whether the surface is self-introduced at all.
- NO conversation history, NO session context, NO prior subagent findings from this run. Repo rules inside the compact context pack are allowed because they come from durable project artifacts or explicit invocation inputs.

**Blocking dispatch**: every Agent call in the dispatch message MUST set `run_in_background: false`. Do not leave this to judgement — the host's default is background, and its tool description actively discourages turning that off for the general case. This skill is the exception the general case does not cover: the dispatcher's entire remaining job (merge → emit → publish) consumes all N reports, so there is no work to interleave and nothing to publish early.

The failure this prevents is silent. With background dispatch the dispatcher gets woken per completion, narrates "waiting for the rest", and ends its turn; a non-interactive host (CI runner, ACP/SDK-driven worker, any headless session) treats that turn ending as the run ending, and every report still in flight is discarded. The PR gets no sticky, no inline comments, no failure notice — indistinguishable from "the review never triggered". Observed in production over 2026-08-11..08-17: same trigger, same skill, some runs published and some vanished, purely on whether the last subagent beat the dispatcher's turn end.

**Threshold inlining**: the [Finding Inclusion Threshold](#finding-inclusion-threshold) is inlined directly in each subagent prompt (`security-reviewer-prompt.md` / `staff-engineer-prompt.md` / `sdet-prompt.md` / `spec-auditor-prompt.md`). Dispatcher does NOT need to prepend threshold text — subagents apply it from their baked-in section. This avoids relying on dispatcher's "good behavior" to inject the gate on every invocation.

### Incremental-mode subagent additions

In incremental mode, each subagent ALSO emits for every prior finding within its scope:

```
Prior finding status: <id>
verification: yes | unclear | no
note: <one-line — what evidence supports the verification>
```

Mapping to display status (in `## 🔄 Changes since last review` table):

| `verification` | Display                                                                 |
| -------------- | ----------------------------------------------------------------------- |
| `yes`          | ✅ Likely fixed `<sha>` — <verification note>                           |
| `unclear`      | ⏸️ Untouched — <note: "file segment not in diff">                       |
| `no`           | 🔄 Still present — <note: "evidence still observable at <file>:<line>"> |

**Never** emit `✅ Fixed` without `verification: yes`. Default hedge is `Likely fixed` always — finality belongs to the human reviewer.

### Fallback rules

- 1 subagent fails → continue with rest; sticky header shows `⚠️ Partial — <subagent> failed`
- 2+ fail → continue with surviving findings; sticky header shows `⚠️ Partial — N/4 subagents failed: <names>`
- ALL fail → report failure to user, do not publish, never self-review

## Subagent Finding Contract

Each subagent emits findings in this shape:

```
[<category-id> <category-name>] <file>:<line_start>-<line_end>
Severity: 🚨 | ⚠️ | 💡 | ❓
Confidence: high | medium | low
Blast: Local | Module | Cross-service | Data layer
Justification: Reachable | Precedent | Asymmetric | Historical

Evidence: <verbatim diff line(s) — cite-or-drop rule>
Failure mode: <one-line — what breaks if shipped as-is>
Mitigation: <one-line — fix action; cite test path when test coverage is part of the fix>
Details: <optional — multi-line narrative, repro steps, code patch. Use only when Failure mode genuinely needs more than one line>
Notes: <optional — only if severity differs from default>
```

**Field semantics**:

- `Failure mode` — concrete bug / breach / drift consequence. Forces severity calibration: if you cannot describe what goes wrong in one line, you do not have a finding.
- `Mitigation` — actionable fix. When the finding's resolution involves test coverage, name the test file and case (e.g. `add assert in foo_test.py:42 'rejects empty input' case`).
- `Details` — escape hatch for findings whose explanation cannot fit one line (e.g. multi-step race, cross-file impact chain). Keep `Failure mode` and `Mitigation` as one-liners regardless; put narrative here.
- `Justification` — required class declaring why the finding is worth emitting. See [Finding Inclusion Threshold](#finding-inclusion-threshold) below. Findings that cannot commit to one of the four classes MUST NOT be emitted as standalone findings; batch into a Q-class hygiene followup instead.

After findings, each subagent emits `N/A categories: [<list>]` declaring which of its owned categories were reviewed and clean. This distinguishes "checked, found nothing" from "skipped".

spec-auditor uses `Spec quote:` + `Code quote:` instead of single `Evidence:` — both must be verbatim quotes. `Failure mode` for spec findings = "what spec contract gets violated if shipped".

**Drop rule**: any finding without `Evidence:` (or both quotes for spec-auditor) is fabrication — discard before merge.

## Finding Inclusion Threshold

This gate is applied by each subagent inline before emitting a finding. **Canonical definition lives in the subagent prompts, not here** — see any of:

- `security-reviewer-prompt.md` § Finding Inclusion Threshold
- `staff-engineer-prompt.md` § Finding Inclusion Threshold
- `sdet-prompt.md` § Finding Inclusion Threshold
- `spec-auditor-prompt.md` § Finding Inclusion Threshold

All four contain the same Justification classes (Reachable / Precedent / Asymmetric / Historical), the same drop signals (A / B / C / D / E / F), and the same narrowed Asymmetric escape hatch. Per-prompt variations only add category-specific guidance (e.g. "most S1–S5 are Asymmetric" for security, "rare for T-class" for SDET).

Signals (E) previously-dismissed and (F) scope-declared were added after a review of 215 findings across 13 merged and in-flight MRs: 50% drew no action from the author, and the two largest recoverable causes were re-raising findings the author had already rebutted in-thread, and asking for sweeps the PR description had explicitly bounded. Neither is a generation problem — both are state the review had but never read.

**Why duplicated across four prompts rather than referenced from one source**: see [Design note: prompt inlining](#design-note-prompt-inlining-over-reference-indirection).

**Full vs incremental mode**: full mode applies the threshold but drop signal (B) self-introduced surface never fires (no `prior_fix_range` on iter 1). Incremental mode applies all four signals.

**Spec ambiguity rule** (applies only to spec-auditor's C-class findings, kept in this SKILL.md as cross-cutting): if a candidate finding's mitigation offers "add a code comment" / "document the limitation in a comment" as an **equal-weight** resolution (phrasing "either X or document Y"), the finding is a Q-class spec gap addressed to the spec author, not P-class actionable. A comment-as-last-resort **fallback** ("do X; if impractical, document Y") keeps the finding actionable — the primary mitigation is what gets judged.

## Severity Merge Rule (deterministic precedence)

Apply in fixed order to each finding. Lower number wins on conflict.

1. **Base severity** — assigned by subagent at finding emission (emoji form)
2. **Confidence demote** — `confidence: low` → demote to ❓ Question (terminal, no further escalation)
3. **P1 gate** — see [P1 calibration](#p1-calibration). A finding that fails the gate is demoted to 💡 P2 or 🔧 P3, whatever its base severity. Applied before any escalation.
4. **Blast attention** — `blast: cross-service` or `blast: data-layer` raises review attention and can escalate P2→P1 **only when the finding already passes the P1 gate**. It does **not** automatically escalate to 🚨 Blocker.
5. **P0 calibration** — final 🚨 / P0 requires the [P0 calibration](#p0-calibration) test: reachable now, severe/concrete, and must be handled in this PR.
6. **Context adjust** — overrides from context input, repo rules, accepted scope, or explicit design trade-offs applied last. Accepted follow-up / wontfix / by-design dispositions remove the finding from open blockers.
7. **Final severity** — result after all steps
8. **Map to P-code** for output (dispatcher does this; subagents emit emoji severity only):

| Emoji         | P-code | Label                                              | Published as        |
| ------------- | ------ | -------------------------------------------------- | ------------------- |
| 🚨 Blocker    | **P0** | must fix; blocks merge                             | inline thread       |
| ⚠️ Factual    | **P1** | breaks behavior / leaks data / blocks rollback     | inline thread       |
| 💡 Suggestion | **P2** | real defect, bounded cost of shipping as-is        | inline thread       |
| 🔧 Nit        | **P3** | correct but not worth a round trip                 | **sticky only**     |
| ❓ Question   | **Q**  | clarify; not a priority tier                       | sticky only         |

Severity ordering (for sort): P0 > P1 > P2 > P3. Q is orthogonal.

Downgrades (step 3 or step 6 lowering a tier) MUST appear in the `Severity adjustments` section. Never silent. Never collapsed behind `<details>` — render as plain section when any adjustment exists.

### Volume caps (applied after merge + dedup, before publish)

> The thresholds in this section and in [P1 calibration](#p1-calibration) / [Prose severity ceiling](#prose-severity-ceiling) are measured, not chosen. Two of them were measured to do nothing and are deliberately loose. Before tightening any of them, read `docs/pr-review-severity-calibration.md`.

Noise, not inaccuracy, is what makes a reviewer get ignored. Google's Tricorder contract for review-time analyzers is an effective false-positive rate below 10%, where "effective false positive" means *the developer chose not to act* — a finding that is technically correct but draws no action counts against you exactly like a hallucination ([Sadowski et al., ICSE 2015](https://www.cs.umd.edu/class/spring2019/cmsc414/papers/tricorder-building-a-program-analysis-ecosystem.pdf); analyzers above 10% go on probation, above 25% get switched off). Reviews whose feedback is mostly actionable cluster at **1–3 comments**; comment relevance measurably dilutes as volume rises ([Chowdhury et al., MSR '26](https://arxiv.org/pdf/2604.03196)).

| Cap                         | Rule                                                                                                       |
| --------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **P2 inline per iteration** | At most **12** P2 inline threads, dropped by ascending confidence; the remainder moves to the sticky as a counted line. P0 and P1 are **exempt from the cap entirely** — every one of them opens a thread, however many there are. A review carrying more than a handful of P0/P1 is not a volume problem to be trimmed. |
| **P3 per review**           | At most **5** listed individually in the sticky. Beyond that render `plus <N> similar items` and drop the detail. |
| **Iteration 2+**            | P0 / P1 only. New P2 / P3 findings are collected into one sticky line, never inline.                        |
| **Iteration 3+**            | P0 only. Everything else goes to the sticky.                                                                |

The inline cap is deliberately loose. Replaying 161 real findings through these rules showed the cap doing none of the noise reduction — that came entirely from P3 leaving the inline channel — while a tighter cap (8) removed four material findings and made the effective-FP rate slightly *worse*, because material findings are not concentrated in the top tiers. Treat 12 as a guard against pathological iterations, not as a tuning knob to reach for when a review feels noisy; the tiering is where that work belongs.

Rationale for the iteration ramp: a one-line fix must not reach round seven on style. Author-reported fatigue with incremental AI review is specifically that "after the initial review, subsequent comments on revised pull requests often become redundant and unhelpful" ([Cihan et al., 2024](https://arxiv.org/pdf/2412.18531)).

**Ordering matters as much as volume.** Emit inline threads highest-confidence-first: trust collapses on the first few items a reader checks, and the collapse is self-reinforcing — once a reader distrusts the tool they start filing real defects as false positives too ([Bessey et al., CACM 2010](https://www.cs.columbia.edu/~junfeng/17sp-e6121/papers/coverity.pdf)).

## Dedup (between subagent findings)

- Same `file:line` + same category → keep highest base severity, attribute to all reporting subagents
- Same issue described differently across subagents → merge into one finding with combined notes
- Cross-cutting (e.g. staff-eng AND sdet both flag missing test for SQL injection) → keep both, dispatcher cross-references

## Output Language

PR-published prose (sticky shape narrative, `Failure mode` / `Mitigation` / `Details` content, spec gap question body, verification notes, framing text around code refs) renders in the PR description's primary language. Everything else — markers, section titles, field labels, kebab-case slugs, P-codes, severity / justification / status tokens, the race meta tag — stays English.

Fallback when the PR description lacks substantive prose: linked issue body, then English.

Terminal / JSON output (`mode=local` JSON, dry-run console, noop message) stays English regardless of PR language — those go to callers, not the PR.

## Output Format

After subagent findings are merged, deduped, and severity-calibrated, produce three artifacts:

1. **Sticky comment** — a single comment on the PR/MR (GitHub issue comment / GitLab MR note); updated in place across iterations (same comment id). This is the canonical review summary.
2. **Commit status** — commit status named `pr-review` (GitHub `context` / GitLab `name`), visible in the PR/MR header / Checks area. Its `target_url` links to the sticky comment.
3. **Inline comments** — one root comment per P0 / P1 / P2 finding emitted in this iteration and surviving the [volume caps](#volume-caps-applied-after-merge-dedup-before-publish), anchored to the diff (GitHub: one batched review; GitLab: one discussion per finding — see [Platform](#platform)). Ordered highest-confidence-first. P3 and Q findings stay in the sticky and never open a thread.

Status tier (drives sticky heading and commit status, NOT any actual approve/request-changes review event):

| Condition                                    | Sticky heading                         | Commit status |
| -------------------------------------------- | -------------------------------------- | ------------- |
| Any subagent failed                          | `⚠️ pr-review: PARTIAL`                | `failure`     |
| Any unaccepted P0                            | `🔴 pr-review: BLOCKED`                | `failure`     |
| No unaccepted P0, any unaccepted P1          | `🟠 pr-review: REVIEW BEFORE MERGE`    | `failure`     |
| Only unaccepted P2 / P3 / Q                  | `🟡 pr-review: PASSED WITH NOTES`      | `success`     |
| Zero unaccepted findings                     | `✅ pr-review: PASSED`                 | `success`     |
| Zero unaccepted findings + accepted exception | `🟡 pr-review: PASSED WITH NOTES`      | `success`     |

Accepted exceptions (`follow-up`, `wontfix`, `by-design`) do not count as open blockers after they are explicitly recorded in the sticky. The sticky status MUST be recalculated after every incremental review from current unaccepted findings plus accepted exceptions; never leave `BLOCKED` / `REVIEW BEFORE MERGE` visible after all P0/P1 findings have been accepted or verified likely fixed.

The skill **does not** submit `APPROVE` or `REQUEST_CHANGES` reviews. Status wording lives inside the sticky comment and commit status only. Auto-approve / auto-merge is forbidden.

### P0 calibration

Final P0 is reserved for findings that are all three:

1. **Reachable now** — current code path can produce the failure mode.
2. **Severe and concrete** — failure causes security breach, data loss/integrity break, unrecoverable workflow break, or equivalent critical impact.
3. **Must be handled in this PR** — not already an accepted out-of-scope item, follow-up, or design trade-off.

`Cross-service` or `Data layer` blast may escalate severity only when the three P0 conditions hold. Otherwise apply the [P1 calibration](#p1-calibration) below, P2 for optional hardening, P3 for nits, or Q for spec/design decisions. Downgrades caused by accepted scope or design context must be visible in `Accepted exceptions` or `Severity adjustments`.

### P1 calibration

P1 is the tier that decides whether a human stops and reads. It is **not** the default landing spot for "this is factually true".

A finding is P1 only if shipping it as-is would do one of three things:

1. **Break behavior** — a user-visible or caller-visible operation produces a wrong result, crashes, or silently no-ops.
2. **Leak or corrupt data** — secrets, PII, cross-tenant exposure, or a write that loses/overwrites correct data.
3. **Block rollback or recovery** — the change cannot be reverted safely, or an incident-time control (audit marker, guard, alarm) is inoperative.

Everything else is P2 at most, regardless of how certain the finding is. Specifically **not P1**:

- A missing test, assertion, or scenario for behavior that is currently correct. The production code works; the finding is about future regressions. → P2 when the untested path is a real branch, P3 when it is symmetry with a sibling test.
- A comment, docstring, tag, or index entry that is stale or inconsistent. → P2 if a reader would act on the wrong information, else P3.
- Wording, naming, declarative-step style, or "this section should mirror that section". → P3.
- Any failure mode whose first clause is "if a future refactor…" / "if someone later…". Drop signal (A) should already have caught it; if it survived, it is P3.

**Severity is judged on the failure mode, never on the justification class.** In particular, `Justification: Asymmetric` establishes only that the finding was worth *emitting*. It does not establish a tier — a cheap fix for an unreachable problem is still a P3.

**Do not assign a tier from a self-reported confidence score.** Verbalized LLM confidence is systematically overconfident and miscalibrated in one direction; at self-reported ≥0.8 a meaningful share of judgements is still wrong ([Siddiq et al., 2026](https://arxiv.org/html/2606.31159v1)). Confidence is a sort key for ordering, not a gate.

### Summary line (top of sticky)

The first visible line of the sticky is always the status heading. Do not prepend bot attribution such as `Automated review by pr-review skill`. The reader should know pass/fail before reading details.

```
## <status-heading>

**Open**: <none | P0×N, P1×N, P2×N, Q×N — only non-zero> · **Reviewed HEAD**: `<HEAD>` · **Mode**: <full|incremental>
**Checked**: ✅ <N> clean
**Next action**: <one-line: optional for PASSED, required otherwise>
```

Examples:

```
## ✅ pr-review: PASSED

**Open**: none · **Reviewed HEAD**: `abc1234` · **Mode**: full
**Checked**: ✅ 11 clean

## 🟡 pr-review: PASSED WITH NOTES

**Open**: P2×1, Q×1 · **Reviewed HEAD**: `abc1234` · **Mode**: full
**Checked**: ✅ 11 clean
**Next action**: optional; no blocker

## 🟠 pr-review: REVIEW BEFORE MERGE

**Open**: P1×2 · **Reviewed HEAD**: `abc1234` · **Mode**: incremental
**Checked**: ✅ 11 clean
**Next action**: fix F2/F4 or explicitly defer

## 🔴 pr-review: BLOCKED

**Open**: P0×1, P1×2 · **Reviewed HEAD**: `abc1234` · **Mode**: incremental
**Checked**: ✅ 11 clean
**Next action**: fix F1 before merge

## ⚠️ pr-review: PARTIAL

**Open**: P1×2 · **Reviewed HEAD**: `abc1234` · **Mode**: full
**Checked**: ✅ 8 clean
**Next action**: rerun review; security-reviewer failed
```

### Category slugs

Convert each subagent's `[<code> <name>]` to a kebab-case slug for output. Drop the subagent-owned code (S/E/T/C). Examples:

- `[E3 Conditional side effects]` → `state-consistency` (use semantic slug, not the literal name when one is more reviewer-meaningful)
- `[S3 Secret / credential]` → `secrets-handling`
- `[T1 Test coverage gaps]` → `missing-coverage`
- `[C4 Business rule alignment]` → `decision-conflict`

When semantic slug differs from the literal category name, prefer semantic. The slug is the navigation handle reviewers see; pick the term that conveys "what kind of problem" most directly.

### Sticky comment template

```markdown
<!-- pr-review:sticky -->
<!-- pr-review:version=2 -->
<!-- pr-review:sha=<HEAD> -->
<!-- pr-review:status=<PASSED|PASSED_WITH_NOTES|REVIEW_BEFORE_MERGE|BLOCKED|PARTIAL> -->

## <status-heading>

**Open**: <none | non-zero buckets> · **Reviewed HEAD**: `<HEAD>` · **Mode**: <full|incremental>
**Checked**: ✅ <N> clean
**Next action**: <one-line: optional only when PASSED>

> <one-line shape narrative — what's the issue cluster; render in PR description language. English example: "observability + state-consistency form two P1 clusters; security clean">

## 📋 Currently open (<N>)

- **<id>** <P-code> `<slug>` — <file>:<line>
- ...

## ↪ Accepted exceptions (<N>)

- **<id>** <P-code> `<slug>` — <follow-up #N | wontfix | by-design>: <one-line reason>
- ...

📍 **Inline comments**: <N> findings pinned to source lines (see the Files changed tab) — render this locator line in PR description language

## ⚖️ Severity adjustments

<rendered only when ≥1 adjustment exists; NOT inside <details>; see template below>

## 🔄 Last iteration changes (`<last_sha>..<HEAD>`)

<rendered only in incremental mode; ONLY this iter's verifications, not cumulative; see template below>

<details><summary>📊 Overview by category</summary>

| Category |  P0 |  P1 |  P2 |  P3 |   Q | Files                         |
| -------- | --: | --: | --: | --: | --: | ----------------------------- |
| `<slug>` |   N |   N |   N |   N |   N | <file paths, comma-separated> |

</details>

<details><summary>❓ Spec gap questions (<N>)</summary>

<rendered only when spec-auditor emitted gap items>

</details>

<details><summary>✅ Checked & clean (<N>)</summary>

- `<slug>`: <one-line evidence — what specific patterns were verified clean, or which grep / file-read confirmed>
- ...

</details>

---

`pr-review` · reviewed `<base>..<HEAD>`<· last reviewed `<last_sha>` — incremental only>
```

Rules:

- The first visible line MUST be the status heading. Do not render bot attribution before it.
- `Open` counts only unaccepted findings. Accepted exceptions appear in their own section and do not block `PASSED WITH NOTES`.
- `Next action` is mandatory for `PARTIAL`, `BLOCKED`, `REVIEW BEFORE MERGE`, and `PASSED WITH NOTES`; omit only for clean `PASSED`.
- Shape narrative mandatory when ≥2 findings; optional for 0-1
- `📋 Currently open` rendered **flat** (no `<details>`) when ≥1 finding is not yet `Likely fixed`; one bullet per finding, sorted P0→P1→P2→P3→Q then by file path. P3 rows collapse to a single `🔧 <N> nits` line once more than five exist. Omit the section entirely when all findings are closed (avoid empty heading)
- `↪ Accepted exceptions` rendered **flat** when any finding is explicitly closed as follow-up / wontfix / by-design. Omit when empty.
- `📊 Overview by category` always in `<details>` (collapsed); rows omitted where P0/P1/P2/P3/Q are all zero. Collapsed by default — summary line already conveys totals; the table is for drill-down only
- `📍 Inline comments` line shown when ≥1 P0/P1/P2 finding posted inline; omit otherwise
- `Severity adjustments` rendered **flat** (no `<details>`) when any adjustment exists — discipline requirement, never silent
- `🔄 Last iteration changes` rendered **flat** in incremental mode; shows ONLY this iter's verifications (`<last_sha>..<HEAD>`), never cumulative across older iterations. Audit trail for older iters lives in git history (commits + prior inline comment threads), not in the sticky
- `Spec gap questions` always in `<details>` (collapsed) — verbose; secondary to actionable findings
- `Checked & clean` always in `<details>` (collapsed) — count is the load-bearing signal; expand for trust calibration

### Severity adjustments section

```markdown
## ⚖️ Severity adjustments

| #    | Category | Adjustment                                         | Reason              |
| ---- | -------- | -------------------------------------------------- | ------------------- |
| F<n> | `<slug>` | <original-emoji + P-code> → <final-emoji + P-code> | <reason — one line> |
```

### Last iteration changes section (incremental only)

```markdown
## 🔄 Last iteration changes (`<last_sha>..<HEAD>`)

| Prior                                | Status                                        |
| ------------------------------------ | --------------------------------------------- |
| F<n> <P-code> <slug> (<file>:<line>) | ✅ Likely fixed `<sha>` — <verification note> |
| F<n> <P-code> <slug> (<file>:<line>) | 🔄 Still present — <note>                     |
| F<n> <P-code> <slug> (<file>:<line>) | ⏸️ Untouched — <note>                         |
| F<n> <P-code> <slug> (<file>:<line>) | ↪ Follow-up #<N> — <accepted reason>          |
| F<n> <P-code> <slug> (<file>:<line>) | 🚫 Wontfix — <accepted reason>                |
| F<n> <P-code> <slug> (<file>:<line>) | 🧭 By-design — <accepted reason>              |
| F<n> Q <slug>                        | ⏸️ Awaiting spec author                       |
```

Scope: **only findings whose status changed (or was re-confirmed) in this iteration's `<last_sha>..<HEAD>` diff**. Untouched findings carrying over from before `<last_sha>` belong in `📋 Currently open`, not here. The table is the delta, not the inventory.

Status legend (hedged on purpose — line-moved ≠ behaviour-fixed):

- `✅ Likely fixed <sha>` — subagent emitted `verification: yes` + note explaining what changed
- `🔄 Still present` — subagent emitted `verification: no` + note pointing to remaining evidence
- `⏸️ Untouched` — subagent emitted `verification: unclear` (file segment not in diff)
- `↪ Follow-up #<N>` — human / babysit disposition accepted a follow-up issue instead of fixing in this PR
- `🚫 Wontfix` — human / babysit disposition accepted that the finding will not be fixed in this PR
- `🧭 By-design` — human / babysit disposition accepted that the finding's premise conflicts with a repo rule, ADR, spec, or explicit PR design decision

Follow-up / wontfix / by-design rows MUST also appear under `↪ Accepted exceptions`, and MUST be excluded from `📋 Currently open`.

### Inline comment body template

One per P0 / P1 / P2 finding emitted in this iteration. Anchored to the diff via the platform's inline endpoint — GitHub: one batched review (`event=COMMENT`); GitLab: one discussion per finding carrying a `position` (see [Platform](#platform)). Opening a new root comment for the same finding in a later iteration is allowed, but the root MUST keep the same `F<n>` finding ID, use this template, link to the sticky, and link to the previous thread when known.

````markdown
<!-- pr-review:finding-root -->
<!-- pr-review:finding-id=F<n> -->
<!-- pr-review:status=<open|still_present|likely_fixed|follow_up|wontfix|by_design> -->
<!-- pr-review:head=<HEAD> -->
<!-- pr-review:sticky-url=<sticky-comment-url> -->
<!-- pr-review:previous-thread-url=<url-or-none> -->

**F<n> <P-code> `<slug>`** · <status-label>

**Sticky summary**: <sticky-comment-url>
**Iteration**: `<last_sha>..<HEAD>`<br>
**Previous thread**: <url — omit when none>

**Failure mode**: <one-line>

**Mitigation**: <one-line; cite test path when applicable>

<details><summary>Evidence</summary>

```diff
<verbatim diff line(s)>
```

</details>

<sub>blast: <Local|Module|Cross-service|Data layer> · <reversible|not reversible> · confidence: <high|medium|low> · justification: <Reachable|Precedent|Asymmetric|Historical></sub>

<!-- pr-review:justification=<Reachable|Precedent|Asymmetric|Historical|Hygiene> -->
````

The root markers are consumed by `pr-babysit` and by later incremental reviews. The `justification` HTML marker is consumed by `pr-babysit`'s diminishing-returns gate to decide whether to keep looping or hand back to the user. `Hygiene` value is reserved for batched Q-class hygiene followups; never emit `Hygiene` on a P0/P1/P2 finding.

Status label values:

- `🆕 New`
- `🔄 Still present`
- `✅ Likely fixed`
- `↪ Follow-up`
- `🚫 Wontfix`
- `🧭 By-design`

`reversible` derivation:

- `reversible` — code-only change, additive feature, refactor without state migration
- `not reversible` — destructive migration, breaking contract change, irreversible side effect (sent message, deleted data)
- omit if ambiguous (don't guess)

### Spec gap questions (in sticky `<details>`)

```markdown
### ❓ F<n> <spec-section-or-decision-id> — <one-line title>

`<Blast>` · spec-author confirm

**Spec quote**: <verbatim>

**Code quote**: <verbatim>

**Question for spec author**:

1. <numbered question>
2. ...

<closing line, in PR description language — e.g. "not blocking the PR; want to clarify X">
```

Q findings do **not** become inline comments — they're often cross-file conceptual questions, pinning to a line misleads.

### What to drop from output

| Drop                                                 | Why                                                                           |
| ---------------------------------------------------- | ----------------------------------------------------------------------------- |
| `Capability: has_spec=Y · has_repo=Y · is_trivial=N` | dispatch logic; reviewer doesn't care                                         |
| `Subagents: security ✅ · staff-eng ✅ · ...`        | which bots ran is process metadata — UNLESS one failed (then surface)         |
| `Source: <subagent>` per finding                     | reviewer wants "what kind of issue" (already in category), not "who found it" |
| `<subagent>/` namespace prefix on slugs              | leaks subagent identity; bare slug reads cleaner                              |
| `Checked & clean` grouped under subagent headers     | same — flat topic list                                                        |
| Empty `Severity adjustments` section heading         | render section only when content exists                                       |

## Publishing

Runs after the findings merge/dedup step. Skipped entirely when `dry-run: true` (print sticky + inline payloads to console instead) **or** when `mode: local` (emit findings JSON to stdout — see [Local Mode](#local-mode)).

```dot
digraph publish {
  "Findings merged" [shape=doublecircle];
  "Build sticky body" [shape=box];
  "Build inline payload template" [shape=box];
  "Find sticky comment" [shape=box];
  "Found?" [shape=diamond];
  "Update sticky in place" [shape=box];
  "Delete stale duplicate stickies" [shape=box];
  "POST new sticky" [shape=box];
  "Capture sticky URL" [shape=box];
  "Publish commit status" [shape=box];
  "Fill sticky URL in inline roots" [shape=box];
  "Post inline comments" [shape=box];
  "Done" [shape=doublecircle];

  "Findings merged" -> "Build sticky body";
  "Findings merged" -> "Build inline payload template";
  "Build sticky body" -> "Find sticky comment";
  "Find sticky comment" -> "Found?";
  "Found?" -> "Update sticky in place" [label="yes"];
  "Found?" -> "POST new sticky" [label="no"];
  "Update sticky in place" -> "Delete stale duplicate stickies";
  "Delete stale duplicate stickies" -> "Capture sticky URL";
  "POST new sticky" -> "Capture sticky URL";
  "Capture sticky URL" -> "Publish commit status";
  "Capture sticky URL" -> "Fill sticky URL in inline roots";
  "Build inline payload template" -> "Fill sticky URL in inline roots";
  "Fill sticky URL in inline roots" -> "Post inline comments";
  "Publish commit status" -> "Reconcile sticky";
  "Post inline comments" -> "Reconcile sticky";
  "Reconcile sticky" -> "Done";
}
```

### Payload assertions (run before any POST)

The `Evidence:` cite-or-drop rule is enforced at emission, but emission is not where it fails. Assert on the *rendered payload*, immediately before posting:

1. **Evidence block is non-empty.** Every inline root must contain a `<summary>Evidence</summary>` block with at least one non-whitespace line. An empty block means the quote was lost between the subagent report and the payload — the finding is unciteable and MUST NOT post. Drop it and note the drop in the sticky.
2. **No swallowed identifiers.** `Failure mode` and `Mitigation` must not contain a run of two or more spaces between non-space characters. That gap is where an inline-code span used to be; a body reading "若 X 期間 ␣␣ 拋例外，␣␣ 寫入 ␣␣ 而非 ␣␣" is unreadable and tells the author nothing.
3. **Body round-trips.** Build every payload by writing the markdown to a file and passing it as a file argument (`--input`, `-F body=@file`, `--arg body "$(cat file)"`). Never interpolate finding markdown into a double-quoted shell string: backticks are command substitution there, and every `` `identifier` `` in the body silently becomes empty.

A finding that fails 1 or 2 is a publishing defect, not a review finding. Fix the payload and repost, or drop it — never ship the mangled version. In a 215-finding sample this check would have caught 27 empty-Evidence posts (12%), including the highest-value finding in its MR.

### Reconcile step (mandatory, never skipped)

The sticky must be posted before the inline comments because the inline roots carry its permalink. That ordering means the first write of the sticky is a **claim about what will be published**, not a record of what was. Close the gap with a second write:

1. Collect the outcome of every inline call — posted (with URL) or failed.
2. Rebuild `📋 Currently open`, the `Open:` counts, the status heading, and the commit status **from the threads that actually exist**, not from the in-memory finding list.
3. PATCH / PUT the sticky again with the reconciled body.

Rules:

- A finding with no successfully posted thread and no sticky row does not exist. Never leave a finding listed in the sticky with no thread behind it — a reader who cannot find the thread has to prove a negative, and the sticky is the one artifact people actually read.
- Conversely, never let the sticky read `Open: none` while an unaccepted P0/P1 thread is live. The status line is what a merge decision is made on; if it and the threads disagree, a real defect ships.
- If step 1 or 3 fails, the sticky must say so (`⚠️ pr-review: PARTIAL — publish incomplete`) rather than keeping the optimistic first write.

### Commands

Pick endpoints by `$PLATFORM` (see [Platform](#platform)). The five steps are identical in shape; only the transport differs. `sticky.md` and its markers are byte-for-byte the same on both platforms:

```
<!-- pr-review:sticky -->
<!-- pr-review:version=2 -->
<!-- pr-review:sha=$HEAD -->
<!-- pr-review:status=$STATUS_TOKEN -->
```

`STATUS_STATE` (step 4): `success` for PASSED / PASSED_WITH_NOTES; the failure token for REVIEW_BEFORE_MERGE / BLOCKED / PARTIAL — **GitHub `failure`, GitLab `failed`**. `STATUS_DESCRIPTION` must stay short.

#### GitHub (`gh`)

```bash
# 1. find sticky id (may be empty)
STICKY_ID=$(gh api repos/$OWNER/$REPO/issues/$N/comments \
  --jq '.[] | select(.body | contains("<!-- pr-review:sticky -->")) | .id' | head -1)

# 2. build sticky.md (markers above)

# 3. create when none exists, else PATCH in place — capture BOTH id and permalink.
#    The id is what step 6 edits; on a first run it only exists after this POST,
#    so capture it here or reconcile silently PATCHes an empty id.
if [ -z "$STICKY_ID" ]; then
  STICKY_JSON=$(gh api -X POST repos/$OWNER/$REPO/issues/$N/comments -F body=@sticky.md)
else
  STICKY_JSON=$(gh api -X PATCH repos/$OWNER/$REPO/issues/comments/$STICKY_ID -F body=@sticky.md)
fi
STICKY_ID=$(printf '%s' "$STICKY_JSON" | jq -r '.id')
STICKY_URL=$(printf '%s' "$STICKY_JSON" | jq -r '.html_url')
[ -n "$STICKY_ID" ] && [ "$STICKY_ID" != "null" ] || { echo "STOP: sticky id not captured"; exit 1; }

# 4. PR-header-visible commit status
gh api -X POST repos/$OWNER/$REPO/statuses/$HEAD \
  -f state="$STATUS_STATE" -f context="pr-review" \
  -f target_url="$STICKY_URL" -f description="$STATUS_DESCRIPTION"

# 5. inline — one batched review. Skip when none this iteration.
# inline-comments.json: [{"path": "...", "line": N, "side": "RIGHT", "body": "..."}, ...]
if [ "$(jq 'length' inline-comments.json)" -gt 0 ]; then
  gh api -X POST repos/$OWNER/$REPO/pulls/$N/reviews \
    -F event=COMMENT \
    -F body="pr-review iteration · $STATUS_DESCRIPTION · $STICKY_URL" \
    -F 'comments=@inline-comments.json' && POSTED=1
fi

# 6. reconcile — rebuild sticky.md from the threads that actually posted, then PATCH again.
#    Mandatory; see § Reconcile step. Pass the body as a FILE (never interpolated into a
#    shell string — backticks in finding markdown are command substitution there).
gh api -X PATCH repos/$OWNER/$REPO/issues/comments/$STICKY_ID -F body=@sticky.md

# 7. re-publish the commit status from the reconciled sticky. Step 4 published the tier
#    predicted before inline posting; if inline calls failed, that tier is now wrong and
#    it is the thing merge decisions read.
gh api -X POST repos/$OWNER/$REPO/statuses/$HEAD \
  -f state="$STATUS_STATE" -f context="pr-review" \
  -f target_url="$STICKY_URL" -f description="$STATUS_DESCRIPTION"
```

#### GitLab (`glab`)

```bash
# 0a. Pin glab to the MR host + assert auth (see § glab auth). Env does not persist across
#     tool calls, so this MUST live in the same block as the glab calls below.
export GITLAB_HOST="<host from the MR URL>"
glab api user >/dev/null 2>&1 || { echo "glab not authed for $GITLAB_HOST — ABORT, do not curl-improvise" >&2; exit 1; }

# 0b. MR web url + inline diff anchors ($PROJECT = url-encoded namespace/path, $IID = MR number)
eval $(glab api "projects/$PROJECT/merge_requests/$IID" \
  | jq -r '"MR_URL=\(.web_url) BASE=\(.diff_refs.base_sha) START=\(.diff_refs.start_sha) HEAD=\(.diff_refs.head_sha)"')

# 1. find sticky notes; --paginate + `jq -s add` so an early sticky isn't lost past page 1
#    (GitLab returns notes newest-first; in-place edits keep the sticky at its original created_at).
#    newest matching id is canonical, older matches are stale duplicates.
MATCHES=$(glab api --paginate "projects/$PROJECT/merge_requests/$IID/notes?per_page=100" \
  | jq -s 'add | [.[] | select(.body | contains("<!-- pr-review:sticky -->"))]')
STICKY_ID=$(echo "$MATCHES" | jq -r 'max_by(.id).id // empty')

# 2. build sticky.md (same markers as GitHub)

# 3. create when none exists, else PUT the newest in place + DELETE the stale duplicates (self-heal)
if [ -z "$STICKY_ID" ]; then
  STICKY_ID=$(glab api -X POST "projects/$PROJECT/merge_requests/$IID/notes" \
    --field body=@sticky.md | jq -r '.id')
else
  glab api -X PUT "projects/$PROJECT/merge_requests/$IID/notes/$STICKY_ID" \
    --field body=@sticky.md > /dev/null
  echo "$MATCHES" | jq -r ".[] | select(.id != $STICKY_ID) | .id" | while read -r OLD; do
    glab api -X DELETE "projects/$PROJECT/merge_requests/$IID/notes/$OLD"
  done
fi
STICKY_URL="$MR_URL#note_$STICKY_ID"   # note permalink = <mr-web-url>#note_<id>

# 4. commit status — `name=` (not `context=`), state token `failed` (not `failure`)
glab api -X POST "projects/$PROJECT/statuses/$HEAD" \
  -f state="$STATUS_STATE" -f name="pr-review" \
  -f target_url="$STICKY_URL" -f description="$STATUS_DESCRIPTION"

# 5. inline — ONE discussion per finding; position MUST travel as a JSON body via --input.
#    `-f "position[...]"` flags are silently dropped → un-anchored DiscussionNote.
#    discussion-<n>.json (old_path == new_path; for an added/context line set
#    new_line and omit old_line):
#    {"body":"<root markdown>","position":{"position_type":"text",
#      "base_sha":"$BASE","start_sha":"$START","head_sha":"$HEAD",
#      "old_path":"<file>","new_path":"<file>","new_line":<line>}}
for f in discussion-*.json; do
  glab api -X POST "projects/$PROJECT/merge_requests/$IID/discussions" \
    -H "Content-Type: application/json" --input "$f" \
    && echo "$f ok" >> posted.log || echo "$f FAILED" >> posted.log
done

# 6. reconcile — rebuild sticky.md from posted.log (the threads that actually exist),
#    then PUT the sticky again. Mandatory; see § Reconcile step. The body travels as a
#    JSON file, never interpolated into a shell string: backticks in finding markdown are
#    command substitution inside double quotes, which is how inline code silently vanishes.
jq -Rs '{body: .}' < sticky.md > sticky-body.json
glab api -X PUT "projects/$PROJECT/merge_requests/$IID/notes/$STICKY_ID" \
  -H "Content-Type: application/json" --input sticky-body.json

# 7. re-publish the commit status from the reconciled sticky — step 4 published the tier
#    predicted before inline posting, and posted.log may have changed it.
glab api -X POST "projects/$PROJECT/statuses/$HEAD" \
  -f state="$STATUS_STATE" -f name="pr-review" \
  -f target_url="$STICKY_URL" -f description="$STATUS_DESCRIPTION"
```

### Old inline comments

Do **not** delete or resolve old inline comments from prior iterations. Both platforms auto-mark a diff-anchored comment `outdated` when its line moves (GitHub collapses outdated threads; GitLab marks the discussion outdated), so stale roots fade on their own. Opening a new root for a repeated finding is allowed, but keep the same `F<n>` id, include `Sticky summary`, and include `Previous thread` when known. This is the chosen trade-off (vs. resolving threads via API) for operational simplicity. (Sticky duplicates are different — those the skill self-heals; see [Publishing](#publishing) step 3.)

### Dry-run mode

When `dry-run: true`:

- Print sticky body markdown to console (with markers)
- Print commit status payload (`state`, `context`/`name`, `target_url`, `description`)
- Print inline payload as JSON (GitHub `inline-comments.json` / GitLab `discussion-*.json`)
- Skip all `gh` / `glab` writes
- Useful for first-time use, debugging, or auditing output before publishing

## Design note: prompt inlining over reference indirection

Subagents (security-reviewer / staff-engineer / sdet / spec-auditor) operate as isolated `Agent` dispatches with no shared loader. They cannot follow a "see `references/X.md`" pointer at dispatch time — references load via the main session, not the subagent's. This inverts skill-creator's default duplication-avoidance principle: any policy a subagent MUST apply is **inlined verbatim into each subagent prompt**, not stored once in `references/` and pointed to.

Current inlined-duplicated content (intentional, not drift):

- **Finding Inclusion Threshold** (Justification classes + drop signals A/B/C/D) — identical wording across all 4 subagent prompts; SKILL.md only points to them
- **Race-class Finding Metadata** (`[window=..., damage=..., recovery=...]` meta tag spec) — identical across `staff-engineer-prompt.md` + `security-reviewer-prompt.md`; pr-babysit's Gate B parser depends on identical syntax

Cross-prompt sync is maintained via `<!-- keep-in-sync: ... -->` HTML comments at each duplicated section header. When editing one, grep for the keep-in-sync marker to find paired sections.

**Do NOT** refactor inlined content into a shared `references/` file — the alternative regresses to the exact failure mode that motivated inlining. SKILL.md previously claimed "dispatcher prepends threshold at dispatch time" and that contract was never enforced (commit `328b73b8` fixed it by baking threshold into prompts; ⇒ this design note exists to prevent a future editor from re-introducing the same gap).

## Notes

- **Don't auto-approve or auto-merge** — produce findings; merge belongs to humans
- **Lean conservative** — low-confidence findings always demote to ❓ Question (Q)
- **Spec gaps don't block review** — mark Q for spec author, proceed with code findings
- **Severity downgrades must be visible** — flat section in sticky, never `<details>`
- **Don't auto-grep for arbitrary spec location** — use user-provided spec/context plus durable artifacts explicitly linked from PR body or repo rules
- **Subagent reports are advisory** — dispatcher applies merge rule and dedup, not subagents
- **Subagent failure must be surfaced** — sticky status heading becomes `⚠️ pr-review: PARTIAL`; never silent
- **Commit status links to sticky** — publish the commit status named `pr-review` (GitHub `context` / GitLab `name`) with `target_url` set to the sticky permalink
- **Finding IDs are `F`-prefixed, never `#`-prefixed** (`F1`, `F2`, …) — GitHub auto-links a bare `#<digits>` in a comment to the issue/PR of that number, so a finding labelled `#7` renders in the sticky as a link to issue #7 (an unrelated issue that merely shares the number). The `F` prefix sidesteps the collision entirely
- **New incremental roots are allowed** — repeated findings may open a fresh root comment, but must reuse the same `F<n>`, include `Sticky summary`, and include `Previous thread` when known
- **Accepted exceptions unblock status** — follow-up / wontfix / by-design dispositions move to `↪ Accepted exceptions` and no longer count as open blockers
- **Prior findings: hedge on "fixed"** — always `Likely fixed`, never bare `Fixed`; line-moved ≠ behaviour-fixed
- **Force-push aware** — when last_sha is unreachable, fall back to full + announce in sticky
- **Output language is adaptive** — PR-published prose follows the PR description's language; markers / titles / field labels / keywords / terms stay English. See [Output Language](#output-language)
- **Local mode is JSON-only** — no markdown, no sticky, no inline; caller (e.g. a supervisor session) consumes findings JSON and drives its own follow-up loop
