---
name: self-review
description: Use when doing dev-stage self-review on the current branch before pushing or opening a PR — runs an auto-loop of codex review (cross-model, OpenAI) + per-finding fix + re-review until findings converge or stop conditions fire. Codex follows pr-review's multi-role methodology (security / staff-engineer / sdet / spec-auditor). Triggers — 'self review', 'self-review', '自己 review', '自我 review', 'cross-model review', 'pre-push review', 'review and fix my branch'. NOT for live PR review with sticky/inline comments (use pr-review), NOT for managed PR babysitting (use pr-babysit), NOT for first-time review without intent to fix (use mode=review-only opt-in).
---

# self-review

Auto-loop dev-stage self-review on the current branch. Runs `codex` (OpenAI GPT) cross-model review following pr-review's multi-role methodology, applies fixes per-finding, re-reviews, and loops until findings converge or a stop condition fires.

Single-shot mode (`review-only`) is available as opt-in — produces findings without auto-fix.

<HARD-GATE>
This skill auto-modifies code on the current branch. Constraints:

**Loop semantics:**

- Per-finding atomic commits — every fix gets its own conventional commit, easy to revert
- Stop conditions are NON-NEGOTIABLE (see Stop Conditions below) — when one fires, the loop STOPS and the remaining work surfaces to the user, period
- User can interrupt anytime with ctrl-c; main session catches and reports partial state

**Context-mix prevention:**

- Codex runs in a fresh process every iteration — no shared conversation context with main session
- Codex output (verbose finding text) goes to a temp file, NOT into chat — only a structured JSON summary enters main session memory, per-iteration
- Main session implementing fixes MUST execute the codex finding's `Mitigation:` field literally, mechanically — NO additional reasoning about whether the fix is right, NO substitution of "a better approach", NO scope expansion
- If you cannot fix the finding from the `Mitigation:` field alone (because it's ambiguous, requires design judgment, or touches code outside the scope of this finding) → SKIP that finding for this iter and surface it to user at end

**Verdict-reasoning forbidden:**

- This skill does not produce wontfix decisions. Findings are either fixed (auto) or surfaced (escalation).
- Do NOT decide "this finding doesn't matter" → all findings are fixed unless they hit the "cannot fix from Mitigation alone" gate above
- Do NOT fill out Wontfix Template fields from main-session memory (see `pr-babysit` § 4.6)

**Quality gate:**

- Tests run after each iter's fixes — `pnpm test` (or detected per-repo command). Failure → STOP and escalate
- This is the natural safety net for "fixes that break things"

**Author bias on fix step:**

- Codex (cross-model) generates findings → bias-isolated at finding-generation layer
- Main session writes fix → not bias-isolated, but follows codex Mitigation literally → bias attack surface is "mechanical execution accuracy", not "verdict judgment"
- If you catch yourself reasoning "I think codex is wrong about this" → that's verdict reasoning, NOT allowed in this skill. Surface to user at end.
  </HARD-GATE>

## Modes

- **`loop`** (default) — codex review → per-finding fix + commit → tests → re-review → loop until stop
- **`review-only`** — single codex pass, present findings, stop. No fix, no commit, no test run. Use when you want a manual review without auto-modification.

Mode selection:

- User says "self review" or "review my branch" → loop (default)
- User says "just show me findings" or "self review without fixing" → review-only

## When to use

- Just wrote code on a feature branch, working tree is clean and committed, want auto cross-model review + fix before push
- Iterating on a feature, want to converge on a clean state quickly
- Pre-PR sanity pass — catch the obvious stuff codex spots before sending to humans

## When NOT to use

- **Live PR review** with sticky comment + inline threads → `/cadence:pr-review` (default mode)
- **PR babysit work after PR is open** → `/cadence:pr-babysit` (handles thread reply, dedup, CI gates)
- **Working tree has uncommitted changes** — STOP and ask user to commit or stash first. Loop assumes clean working tree at start so per-finding commits are atomic
- **You want manual verdict control** → use `mode=review-only` and decide yourself
- **First-time use on a new branch** → consider `mode=review-only` first to see what codex produces before letting it auto-fix

## Setup check

Run these checks at start. STOP on any failure with the listed message — do NOT attempt auto-install.

```bash
# Codex CLI
codex --version 2>/dev/null || { echo "STOP: Install codex — npm install -g @openai/codex"; exit 1; }
codex login --status 2>/dev/null || { echo "STOP: Run 'codex login' to authenticate"; exit 1; }

# Plugin install root — needed so codex can locate the pr-review methodology prompts
[ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -d "${CLAUDE_PLUGIN_ROOT}/skills/pr-review" ] || {
  echo "STOP: CLAUDE_PLUGIN_ROOT must point at cadence's install root (so codex can read ./skills/pr-review/*-prompt.md). Set it explicitly if your harness doesn't export it (e.g. export CLAUDE_PLUGIN_ROOT=~/.claude/plugins/cache/cadence/cadence/<version>)."; exit 1;
}

# Clean working tree
git diff --quiet && git diff --cached --quiet || { echo "STOP: Working tree has uncommitted changes. Commit or stash, then re-invoke."; exit 1; }

# Detect base branch
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
[ -z "$BASE" ] && BASE=main
echo "BASE: origin/$BASE"

# Detect test command (best-effort — repo-specific)
if [ -f package.json ] && jq -e '.scripts.test' package.json >/dev/null 2>&1; then
  TEST_CMD="pnpm test"
elif [ -f Makefile ] && grep -q '^test:' Makefile; then
  TEST_CMD="make test"
else
  TEST_CMD=""
  echo "WARN: No test command detected — quality gate disabled"
fi
echo "TEST_CMD: ${TEST_CMD:-<none>}"
```

## Loop algorithm (mode=loop)

Maintain in main session memory:

- `ITER` — current iteration number (starts at 1)
- `MAX_ITERS = 3` — hard cap. Empirically codex's high-value findings (real
  reachable bugs, the kind a same-model reviewer misses) land in iters 1–3;
  iters 4–5 trend to hygiene-tier nits + the occasional false positive that
  the controller then has to spend judgment rejecting. The real stop is SC0
  (severity floor) below — `MAX_ITERS` is the blunt backstop.
- `FINDING_HISTORY` — list of `file:line:slug` fingerprints from prior iterations (for repetition detection)

### Step 1: Run codex review

Same prompt construction as Step 2 of `mode=review-only` (see Codex Prompt section below). Output goes to `/tmp/self-review-iter-$ITER.md`. Do NOT inline the verbose codex output into main session — only parse the JSON summary block.

Codex prompt MUST end with a summary JSON block (described in Codex Prompt section below) so main session can drive the loop without parsing prose.

```bash
PROMPT_FILE=$(mktemp /tmp/self-review-prompt-XXXXXX.md)
OUTPUT_FILE="/tmp/self-review-iter-$ITER.md"
JSON_FILE="/tmp/self-review-iter-$ITER.json"

# Write prompt (see Codex Prompt section)
write_codex_prompt > "$PROMPT_FILE"

# Run codex (5-min timeout)
_REPO_ROOT=$(git rev-parse --show-toplevel)
codex exec "$(cat "$PROMPT_FILE")" \
  -C "$_REPO_ROOT" \
  -s read-only \
  -c 'model_reasoning_effort="high"' \
  --enable web_search_cached \
  > "$OUTPUT_FILE" 2>/tmp/self-review-err

# Extract JSON summary block between unique sentinels
sed -n '/<!-- SELF-REVIEW-JSON-START -->/,/<!-- SELF-REVIEW-JSON-END -->/{
  /<!-- SELF-REVIEW-JSON-/d
  p
}' "$OUTPUT_FILE" > "$JSON_FILE"

# Guard: codex must emit a parseable JSON block — empty or malformed is NOT zero findings
if [ ! -s "$JSON_FILE" ]; then
  echo "STOP: codex did not emit a JSON summary block. Output saved to $OUTPUT_FILE for manual review."
  break
fi
if ! jq -e '.findings | type == "array"' "$JSON_FILE" >/dev/null 2>&1; then
  echo "STOP: codex JSON summary malformed. Output: $OUTPUT_FILE, JSON: $JSON_FILE"
  break
fi

# Sanity: confirm line_source field is "source" on every finding (catches diff-line confusion)
BAD_LINE_SOURCE=$(jq -r '[.findings[] | select(.line_source != "source")] | length' "$JSON_FILE")
if [ "$BAD_LINE_SOURCE" != "0" ]; then
  echo "STOP: codex emitted findings with line_source != 'source' (likely diff-line numbers, not source-file lines). Review $OUTPUT_FILE manually."
  break
fi
```

### Step 2: Convergence check

Read findings from `$JSON_FILE`. Apply stop conditions IN ORDER — first match wins:

```bash
FINDINGS_COUNT=$(jq '.findings | length' "$JSON_FILE")
```

**SC1 — Success**: `FINDINGS_COUNT == 0` → STOP, report success.

**SC0 — Severity floor (converged-enough)**: an iteration whose findings are
ALL hygiene-tier → STOP, report as converged. "Hygiene-tier" = no finding is
both (`severity` in `Blocker|Factual`) AND (`justification` in
`Reachable|Asymmetric`). I.e. nothing left that is a real, reachable bug —
only `Suggestion`/`Question`, or `Factual` findings resting on
`Precedent`/`Historical` speculation.

```bash
REAL_BUGS=$(jq -r '[.findings[]
  | select((.severity == "Blocker" or .severity == "Factual")
           and (.justification == "Reachable" or .justification == "Asymmetric"))]
  | length' "$JSON_FILE")
```

`REAL_BUGS == 0` (with `FINDINGS_COUNT > 0`) → STOP. The remaining hygiene
findings are surfaced to the user, not auto-fixed — they are below the bar
that justifies another codex round. This is the _intended_ stop in a healthy
run; it usually fires at iter 2–3. `MAX_ITERS` only catches runs where codex
keeps producing real-bug findings that far out (itself a signal the change
is too big and should be split).

> Why SC0 is checked before SC2 but after SC1: zero findings is unambiguous
> success; a hygiene-only iteration is _converged-enough_ success; the
> `MAX_ITERS` cap is the unhappy backstop. Naming it SC0 keeps it visually
> adjacent to SC1 (both success-class) without renumbering SC2–SC6.

**SC2 — Cap reached**: `ITER > MAX_ITERS` → STOP, escalate with current findings. Do NOT apply more fixes.

**SC3 — Same finding 3x (race-of-race signal)**:

```bash
# Build current iter's fingerprints
CURR_FPS=$(jq -r '.findings[] | "\(.file):\(.line):\(.slug)"' "$JSON_FILE")

# For each fingerprint, count occurrences in HISTORY + this iter
for FP in $CURR_FPS; do
  COUNT=$(echo "$FINDING_HISTORY" | grep -cF "$FP" || true)
  COUNT=$((COUNT + 1))  # current iter
  if [ "$COUNT" -ge 3 ]; then
    REPEATED="$FP"
    break
  fi
done
```

If `REPEATED` non-empty → STOP, escalate. Same finding has fired 3+ times across iterations; auto-fix is not converging. Reference `pr-babysit` § 4.5 Gate B for the equivalent convergence-failure pattern.

**SC3.5 — File-only fallback for slug drift**: codex may use a different slug each iter for the same logical issue (`missing-nil-check` → `null-dereference-guard` → `null-check-omitted`). Exact fingerprint match would miss this. Track per-file appearance count across iters; if the same file has produced findings in 3+ consecutive iters → STOP, surface as warning:

```bash
# Per-iter file set (just file names, dedup)
CURR_FILES=$(jq -r '.findings[].file' "$JSON_FILE" | sort -u)
echo "$CURR_FILES" > "/tmp/self-review-iter-$ITER.files"

# Count files that appear in current iter AND last two iters' file lists
if [ "$ITER" -ge 3 ]; then
  PREV1="/tmp/self-review-iter-$((ITER - 1)).files"
  PREV2="/tmp/self-review-iter-$((ITER - 2)).files"
  if [ -s "$PREV1" ] && [ -s "$PREV2" ]; then
    PERSISTENT=$(grep -Fxf "$PREV1" "/tmp/self-review-iter-$ITER.files" | grep -Fxf "$PREV2" | head -3)
    if [ -n "$PERSISTENT" ]; then
      echo "STOP: file(s) producing findings 3+ consecutive iters — possible slug drift hiding stuck finding:"
      echo "$PERSISTENT"
      break
    fi
  fi
fi
```

This is an ADDITIVE signal — doesn't replace SC3's exact fingerprint check. SC3 catches lexical convergence failure; SC3.5 catches semantic convergence failure that slug drift hides.

**SC4 — Findings diverging**: fires on EITHER of two signals (race-of-race detection, cf `pr-babysit` § 4.5 Gate B):

(a) **Count growing**: `FINDINGS_COUNT` strictly larger than previous iter's count → STOP, escalate. The fix step is introducing new issues faster than it resolves them.

(b) **Set replacement**: `FINDINGS_COUNT >= 3` AND zero fingerprint overlap between current iter and previous iter (`|current ∩ prev_iter| == 0`) → STOP, escalate. Count unchanged but the WHOLE finding set turned over — fixes are opening completely new surfaces. Count-only check misses this.

```bash
PREV_FPS_FILE="/tmp/self-review-iter-$((ITER - 1)).fps"
CURR_FPS_FILE="/tmp/self-review-iter-$ITER.fps"
echo "$CURR_FPS" > "$CURR_FPS_FILE"

if [ "$ITER" -gt 1 ] && [ "$FINDINGS_COUNT" -ge 3 ] && [ -s "$PREV_FPS_FILE" ]; then
  OVERLAP=$(grep -Fxf "$PREV_FPS_FILE" "$CURR_FPS_FILE" | wc -l | tr -d ' ')
  if [ "$OVERLAP" = "0" ]; then
    echo "STOP: set-replacement divergence (iter $((ITER-1)) and iter $ITER share zero findings)"
    break
  fi
fi
```

### Step 3: Apply fixes per finding

If no stop condition fired, iterate through findings and apply each as an atomic commit:

```bash
jq -c '.findings[]' "$JSON_FILE" | while read -r FINDING; do
  ID=$(echo "$FINDING" | jq -r '.id')
  PERSONA=$(echo "$FINDING" | jq -r '.persona')
  CATEGORY=$(echo "$FINDING" | jq -r '.category')
  SLUG=$(echo "$FINDING" | jq -r '.slug')
  FILE=$(echo "$FINDING" | jq -r '.file')
  LINE=$(echo "$FINDING" | jq -r '.line')
  FAILURE_MODE=$(echo "$FINDING" | jq -r '.failure_mode')
  MITIGATION=$(echo "$FINDING" | jq -r '.mitigation')

  # Main session reads the full finding from OUTPUT_FILE for context
  # then implements MITIGATION literally
  apply_minimal_fix_from_mitigation "$FILE" "$LINE" "$MITIGATION"

  # Verify file changed
  if git diff --quiet "$FILE"; then
    SKIPPED_FINDINGS+=("$ID: no diff produced — mitigation may need design judgment")
    continue
  fi

  # Atomic commit
  git add "$FILE"
  git commit -m "fix($PERSONA): $SLUG — self-review iter $ITER #$ID

Failure mode: $FAILURE_MODE
Mitigation: $MITIGATION

Source: codex review following pr-review methodology
Category: $CATEGORY
"
done
```

**Per-finding fix rules** (HARD-GATE reinforcement):

- Read the codex `Mitigation:` field for this finding. Implement it MINIMALLY.
- Do NOT expand scope (don't refactor adjacent code, don't add tests not requested, don't add comments)
- Do NOT add Claude reasoning ("I also noticed X, fixed that too" — NO)
- If `Mitigation:` is ambiguous or requires design judgment to implement → SKIP, add to `SKIPPED_FINDINGS`, surface to user at end
- One finding = one commit. If a single mitigation actually touches 3 files, one commit covering all 3 is fine. But two different findings = two commits.

**Pattern generalization (mandatory — not scope expansion):**

When a finding's `Failure mode` describes a _class_ of bug rather than a
single-site defect (e.g. "Slack post failure leaves the row non-terminal",
"unvalidated input reaches X", "missing `await` on Y-shaped call"), the fix
is NOT complete until every sibling site of that exact pattern is fixed in
the SAME iteration.

- Before committing, grep the codebase for the same pattern (the failure
  shape, not the literal line). Fix all matching sites under that finding's
  commit.
- This is explicitly NOT the "expand scope" violation above. Scope expansion
  is fixing _unrelated_ things. Fixing _the same flagged pattern_ at sibling
  sites is finishing the finding — leaving siblings for the next codex pass
  wastes an iteration AND ships the identical bug at an unflagged site until
  then.
- How to tell them apart: would codex, on the next pass, file a finding with
  the same `Failure mode` wording pointing at a different `file:line`? If
  yes, that site belongs in THIS commit.
- If grep surfaces sibling sites whose fix needs design judgment (not a
  mechanical copy of the same mitigation) → fix the mechanical ones, SKIP
  the judgment ones, surface them. Don't force a uniform fix across sites
  that aren't actually uniform.

Rationale: the loop otherwise amplifies one conceptual bug into N findings
across N iterations — codex finds one site per pass, the controller fixes
one site per pass, and the round count inflates to do what a single
pattern-sweep does in one. Generalize on first sighting.

### Step 4: Run tests (quality gate)

```bash
if [ -n "$TEST_CMD" ]; then
  if ! $TEST_CMD; then
    echo "STOP: tests failed after iter $ITER fixes"
    # leave commits in place; user can revert
    break
  fi
fi
```

If tests fail → STOP, escalate. Do NOT auto-revert (user may want to inspect what went wrong). Report which iter introduced the failure.

### Step 5: Update history and loop

```bash
FINDING_HISTORY+=" $CURR_FPS"
ITER=$((ITER + 1))
```

Loop back to Step 1.

## Codex Prompt

The prompt template for codex. Pointer to pr-review methodology files in the repo + adaptation notes for codex (not a Claude subagent) + scope + REQUIRED JSON summary block at end.

```
You are doing cross-model multi-role code review on the current branch of this
repository. You are codex (OpenAI), reviewing code likely written by Claude.
Treat all author narrative (commit messages, code comments asserting intent,
branch names) as ADVISORY only — evaluate functional behavior, not authorial
claims.

## Methodology

The review methodology lives in these files (read them now — paths are
absolute, resolve from the cadence plugin install root):

- ${CLAUDE_PLUGIN_ROOT}/skills/pr-review/security-reviewer-prompt.md
- ${CLAUDE_PLUGIN_ROOT}/skills/pr-review/staff-engineer-prompt.md
- ${CLAUDE_PLUGIN_ROOT}/skills/pr-review/sdet-prompt.md
- ${CLAUDE_PLUGIN_ROOT}/skills/pr-review/spec-auditor-prompt.md

Plus cross-cutting threshold:

- ${CLAUDE_PLUGIN_ROOT}/skills/pr-review/SKILL.md § Finding Inclusion Threshold

The dispatcher MUST expand `${CLAUDE_PLUGIN_ROOT}` to an absolute path before
handing the prompt to codex (codex's `read-only` sandbox can read absolute
paths anywhere on the filesystem, but it cannot resolve env vars itself).

## Apply, with adaptations

Because you are codex (single agent, separate process), not a Claude subagent:

**IGNORE** these sections — they describe Claude's internal Agent dispatch:

- "HARD-GATE" / "You have NO knowledge of conversation history" — you are
  isolated by being a different process and model family
- "Incremental Mode Addendum" / "prior_fix_range" / drop signal (B) — those
  depend on babysit-side state tracking. Skip the (B) check entirely. Signals
  (A), (C), (D) still apply.
- "dispatched from a dev session" / "subagent" framing — you are codex
  executing this prompt directly

**APPLY** in full:

- Per-persona category tables: Security (S1-S5), Staff Engineer (E1-E9),
  SDET (T1-T4), Spec Auditor (C1-C4)
- Finding Inclusion Threshold: Justification class (Reachable / Precedent /
  Asymmetric / Historical)
- Drop signals (A), (C), (D)
- Hygiene batch rule (cluster hygiene drops into one Q-class finding per file)
- Race-class Finding Metadata: Mitigation MUST end with
  `[window=<ms|s|min|hr>, damage=<data-loss|deadlock|inconsistency|latency|marginal>, recovery=<has|no>]`
- Per-prompt Output Schema (Severity / Confidence / Blast / Justification /
  Evidence / Failure mode / Mitigation)

## Execution

Execute all 4 personas sequentially. Output a combined finding list grouped
by persona.

## Scope

Review `git diff origin/<BASE>..HEAD` where BASE is below. Use `git diff` and
`git log --oneline` to understand the change. Read source files as needed.

## Output format (REQUIRED)

First emit per-persona findings in the per-prompt format from the prompt files.
Group by persona.

Then at the END emit a structured JSON summary block — this is REQUIRED for the
calling skill to drive its loop. The JSON block MUST be valid and parseable. Wrap
it in unique sentinel markers (NOT a generic markdown fenced block — those collide
with code examples in Evidence fields):

<!-- SELF-REVIEW-JSON-START -->
{
  "findings": [
    {
      "id": "1",
      "persona": "Security|Staff|SDET|Spec",
      "category": "S2|E5|T1|C4|...",
      "slug": "kebab-case-slug-from-finding",
      "file": "path/to/file (relative to repo root)",
      "line": 42,
      "line_source": "source",
      "severity": "Blocker|Factual|Suggestion|Question",
      "justification": "Reachable|Precedent|Asymmetric|Historical",
      "confidence": "high|medium|low",
      "blast": "Local|Module|Cross-service|Data layer",
      "failure_mode": "one-line",
      "mitigation": "one-line, ending with race-meta tag if applicable"
    }
  ]
}
<!-- SELF-REVIEW-JSON-END -->

Rules:

- `findings: []` (empty array) is VALID output meaning no findings. Still emit the
  block with `{"findings": []}` between the sentinels — do NOT omit the JSON block.
- `line` MUST be the source file line number (the line in the file as written on
  disk after your reading), NOT the diff hunk line number. If the diff shifted lines,
  use the post-shift source file line.
- `line_source` MUST be `"source"` literal — this confirms you used source file
  lines, not diff lines. Any other value → caller treats as malformed and escalates.
- All listed fields are REQUIRED. Do not emit findings with missing fields.

## Important

- Do NOT modify any files
- Race-class findings without meta tag → drop the finding
- You are codex, not Claude — your prose can be your own
- Stay focused on the diff

BASE branch: origin/<substitute BASE from skill caller>
```

## Stop conditions summary

| Condition                  | When                                                                                                         | Action                                                           |
| -------------------------- | ------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| **SC1 Success**            | `findings == 0`                                                                                              | Report iter count + total fixes applied                          |
| **SC0 Severity floor**     | iteration has findings but ZERO real bugs (no `Blocker`/`Factual` × `Reachable`/`Asymmetric`)                | STOP, converged-enough; surface hygiene findings, don't auto-fix |
| **SC2 Cap reached**        | `iter > 3` (`MAX_ITERS`)                                                                                     | Escalate; surface remaining findings to user                     |
| **SC3 Repeat 3x**          | Same finding fingerprint (`file:line:slug`) 3 iterations                                                     | Escalate; race-of-race signal (cf `pr-babysit` § 4.5 Gate B)     |
| **SC3.5 Slug drift**       | Same file produces findings in 3+ consecutive iters (file-only fallback)                                     | Escalate; possible slug drift hiding stuck finding               |
| **SC4 Findings diverging** | (a) count growing iter-over-iter, OR (b) zero fingerprint overlap between consecutive iters with ≥3 findings | Escalate; auto-fix opening new surfaces                          |
| **SC5 Test failure**       | `$TEST_CMD` exits non-zero after iter's fixes                                                                | Escalate; commits left in place for user inspection              |
| **SC6 Skip backlog**       | ≥3 findings skipped this iter (can't fix from Mitigation alone)                                              | Continue loop, but surface skipped list at end                   |
| **User ctrl-c**            | User interrupts                                                                                              | Report partial state, last committed iter, what was in progress  |

## Mode=review-only

When user explicitly asks for findings without auto-fix:

1. Run codex (same prompt as loop mode Step 1)
2. Present output verbatim from `$OUTPUT_FILE` (full per-persona findings text)
3. Stop. Do NOT commit. Do NOT touch files.

This is the L0+ "advisory findings" path — verdict stays with user.

## Report at end

After loop exit (any stop condition), generate report:

```
SELF-REVIEW LOOP REPORT
═════════════════════════════════════════════════════════════
Iterations: <N>
Stop reason: <SC code + brief>
Commits made: <count> (atomic, one per finding)
  - <sha> fix(<persona>): <slug>
  - ...
Tests: <pass | fail | skipped (no TEST_CMD)>

Findings still open (if escalation):
  - <persona> / <category> @ <file>:<line>: <slug>
    Mitigation: <one-line>
    Why surfaced: <which SC fired>

Skipped findings (Mitigation needed design judgment):
  - <persona> @ <file>:<line>: <slug>
    Mitigation: <verbatim>
    Reason: <why main session couldn't fix mechanically>

Suggested next steps:
- Review the atomic commits — revert any you disagree with
- For surfaced findings: read /tmp/self-review-iter-<N>.md for full context, decide modify/wontfix/defer manually
- Consider /cadence:pr-review mode=local for Claude-side multi-role view + comparison
- Push when satisfied
═════════════════════════════════════════════════════════════
```

## Notes

- **Cross-model isolation rationale**: codex (OpenAI GPT) reviews Claude-generated code → avoids same-model self-preference bias (Wataoka et al., perplexity-driven). Each codex invocation is a fresh process — no conversation context inheritance.

- **Context-mix prevention design**: codex output goes to `/tmp/self-review-iter-$ITER.md` (file, not chat). Main session only parses the JSON summary block into conversation memory. Full finding text accessed by main session via file read when implementing each fix — not auto-injected. This keeps main session's growing conversation lean across iterations.

- **Methodology single-sourced**: codex reads `${CLAUDE_PLUGIN_ROOT}/skills/pr-review/*-prompt.md` directly (dispatcher expands the env var to an absolute path before handing the prompt to codex). When pr-review prompts update, this skill picks up the new methodology automatically. No keep-in-sync burden between pr-review and self-review.

- **Adaptation layer keep-in-sync**: the "IGNORE these sections" list in the codex prompt mirrors Claude-specific sections in pr-review prompts. If pr-review adds new Claude-only machinery, update the IGNORE list. Annotated as a maintenance concern, not auto-detected.

- **Per-finding atomic commits**: `git revert <sha>` undoes one finding's fix cleanly. History preserves the audit trail of what codex flagged + how it was fixed.

- **Test gate as natural safety net**: the cheapest signal that "auto-fix broke things" is a failing test. Catches regressions without needing complex semantic verification.

- **Loop count cap (3) reasoning**: empirically (5-iter run on a real
  feature branch) codex's high-value findings — real reachable bugs, the
  same-model-blind-spot class — all landed in iters 1–3. Iters 4–5 produced
  hygiene-tier nits, the tail of an already-identified pattern, and one
  outright false positive the controller had to reject. The cap was 5; it is
  now 3. The _principled_ stop is SC0 (severity floor) — `MAX_ITERS` is the
  blunt backstop, and a run that still yields real-bug findings at iter 3 is
  itself signalling the change is too big and should be split.

- **SC0 vs MAX_ITERS — why both**: SC0 (severity floor) is the intended
  stop — it fires when an iteration produces no real reachable bug, i.e.
  further rounds would only surface nits. `MAX_ITERS=3` is the backstop for
  the pathological case where codex keeps finding real bugs that far out.
  A healthy run stops on SC0 at iter 2–3; only an unhealthy (oversized-diff)
  run reaches the cap.

- **Pattern generalization beats round count**: the loop's structural
  weakness is amplifying one conceptual bug into N findings across N
  iterations (codex finds one site per pass; controller fixes one site per
  pass). The Step 3 "Pattern generalization" rule counters this — on first
  sighting of a pattern-class finding, grep + fix all sibling sites in the
  same iteration. Done well, the loop self-converges inside the cap without
  relying on it.

- **Author bias still applies to FIX step, not VERDICT step**: codex finding generation is cross-model isolated. Main session writing the fix is NOT — but the HARD-GATE constrains main session to mechanical execution of `Mitigation:` field, removing the verdict-reasoning attack surface. If you notice main session "reasoning whether codex is right" → that's a HARD-GATE violation, surface the finding instead of arguing.

- **Worktree assumption**: skill expects to run on a feature branch (user already in worktree or non-main branch). Doesn't self-create worktrees. If user is on `main`, warn but don't block — they may know what they're doing.

- **No state persistence across invocations**: each invocation starts fresh. `FINDING_HISTORY` is per-invocation. If you re-invoke after manual edits, the loop has no memory of previous runs — by design (keeps the skill stateless, no `.claude/state/` files to maintain).
