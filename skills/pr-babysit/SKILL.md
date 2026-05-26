---
name: pr-babysit
description: Use when babysitting a PR/MR until CI is green and every valid reviewer feedback is addressed — supports GitHub PR (gh) and GitLab MR (glab), triages comments into Valid / Discuss / Out-of-scope, addresses valid items with small commits and inline thread replies, escalates invisible findings (SonarQube/Snyk dashboards) and 3-round bot deadlocks, reports ready-to-merge (never auto-merges). Triggers — '監看 PR', 'babysit PR/MR', 'PR 顧到 merge', 'address review feedback', 'wait until CI green', '把 PR 顧到綠'. NOT for writing PR descriptions, NOT for diff code review (use pr-review), NOT for actually merging the PR (user does that).
---

# pr-babysit

Babysit a PR/MR until CI is green AND every valid reviewer feedback is addressed. Supports **GitHub PR** (via `gh`) and **GitLab MR** (via `glab`) — auto-detect by `git remote get-url origin` (github.com → gh; gitlab.com / self-hosted GitLab → glab).

## Arguments

`$ARGUMENTS` — accepts:

- empty → current branch's open PR/MR
- a number → PR/MR by IID on the current repo
- a URL → parse owner/repo + IID from it

If multiple PRs/MRs match the current branch, stop and ask which one.

## Reply Language

Reply prose posted to PR/MR threads — the `<what changed>` / `<reason>` / `<evidence>` content following each reply-template anchor, plus the prose inside Wontfix Template fields — renders in the PR/MR description's primary language. Everything else stays English: the anchor phrases themselves, Wontfix Template field labels, conventional commit prefixes, the race meta tag, P-codes / severity / justification tokens (same canonical set as `pr-review`'s [Output Language](../pr-review/SKILL.md#output-language)).

Fallback when the PR description lacks substantive prose: linked issue body, then English.

Terminal output (step 6 run report, Gate A / Gate B audit messages, invisible-findings prompt) stays English — those go to the dispatcher session, not the PR.

## Loop

### 1. Snapshot

Fetch: PR/MR metadata + head SHA, all checks / pipeline jobs, all review comments, all general comments, all review threads / discussions (with resolved state), the current user login.

For each thread you've previously replied to in this PR, cache `{file path, rule code or primary keyword, your reply summary}` — used by step 2 dedup.

Filter on **content**, not author:

- **Drop** comments whose body is only CI status lines (build green/red, deploy event, "pipeline succeeded"). That is noise.
- **Keep** any comment containing actionable signals (`Suggestion` / `Warning` / `Critical` / `Issue` / `quality gate` / `failed` / line-level review notes) — **even from bot accounts**. AI review bots, SonarQube, Snyk are _content_ bots, not noise bots.
- Drop your own past replies and already-resolved threads.

### 2. Triage

**Hard gate — invisible findings**: if a check is failing but the actual finding list lives in an external dashboard your CLI cannot reach (SonarQube, Snyk, DataDog test reports, etc. — no token, no API endpoint accessible), STOP **immediately** and ask the user to paste the findings. Do **not** reproduce locally and process "guessed" findings as a complete cycle. Do **not** process unrelated feedback first while the invisible finding sits unaddressed. Root-cause diagnosis assumes you can see the finding; when you can't, this gate fires first.

**Cross-round dedup** — for each new comment, check the cache from step 1:

- Same file + same rule code (e.g. `CA1031`) OR same primary keyword as a thread you already replied to → treat as duplicate. Reply with one line linking back to the earlier thread, do not re-implement or re-explain.
- Same issue surviving 3 rounds despite fix attempts → escalate to `needs-user-input` (the bot is stuck; user has to break the tie).

**Feedback** — bucket each remaining unresolved comment:

- **Valid** — bug, security, logic error, clear actionable suggestion
- **Discuss** — ambiguous, possible source misread, design tradeoff, scope unclear → **do NOT reply autonomously, do NOT implement** — collect for user
- **Out-of-scope** — clearly outside this PR's stated goal → collect for user

**Checks** — for each failing check: pull the failure log via CLI, diagnose root cause before attempting a fix (no patch without a named cause). Distinguish real failure vs flaky; only retry on evidence of flake. If the failure log doesn't contain the actual findings → invisible-findings gate above.

### 3. Address (Valid + real failures only)

For each item:

1. Implement the fix.
2. Small commit, conventional commits format, one logical change per commit. Type cheat: behaviour change → `fix`; behaviour-preserving structure / readability (incl. lint suppressions) → `refactor`; non-source (CI, husky, tooling) → `chore`; pure docs → `docs`.
3. Reply on the originating comment / discussion thread (template table below).
4. Verify the reply landed **inside the thread**, not as a top-level note (see "Reply endpoints" below).

**Reply endpoints by platform** — mismatching these creates orphan top-level notes:

| Action                   | GitHub                                                          | GitLab                                                                |
| ------------------------ | --------------------------------------------------------------- | --------------------------------------------------------------------- |
| Reply to a review thread | `POST /repos/{O}/{R}/pulls/{id}/comments` with `in_reply_to_id` | `POST /projects/:id/merge_requests/{iid}/discussions/{disc_id}/notes` |
| New top-level comment    | `POST /repos/{O}/{R}/issues/{id}/comments`                      | `POST /projects/:id/merge_requests/{iid}/notes`                       |

After posting a reply, `GET` the discussion / review thread back and confirm your note is in the thread (note count ≥ 2, your username present). If it landed top-level → delete it and retry on the right endpoint.

**Reply templates** — pick by situation:

| Situation                                     | Template                                                                  |
| --------------------------------------------- | ------------------------------------------------------------------------- |
| Adopted and fixed                             | `Addressed in <SHA> — <what changed>.`                                    |
| Deliberate design, won't change               | `Deliberate design — <reason>. <spec or codebase ref>.`                   |
| Same issue already replied earlier in this PR | `Same as the earlier <topic> thread — <link>.`                            |
| Bot premise wrong, won't fix                  | `Won't fix — premise doesn't hold. <evidence: file:line / spec section>.` |

The Deliberate / Won't-fix templates exist to keep tone neutral and evidence-led — without a template these tend to drift into defensive or implementation-dump replies.

Anchor phrases stay English; only the prose after each anchor adapts to the PR description's language. See [Reply Language](#reply-language).

**Lint / warning suppression** — any `#pragma`, `// eslint-disable`, `# noqa`, `@SuppressWarnings`, etc. must include:

- (a) inline rationale comment on the same line, AND
- (b) reference to the spec section OR an existing codebase precedent (`file:line`) using the same suppression for the same reason.

If neither (a) nor (b) is available → do not suppress, refactor instead. When (b) applies, cite the precedent `file:line` in the commit message.

Hard rules:

- No `--amend` on already-pushed commits
- No `--force-push`
- Don't mark GitLab discussions resolved unless the reviewer explicitly asked for that
- Don't close any reviewer thread without a reply
- 3 failed attempts on the same fix → STOP, document what failed + assumptions to question, hand back to user (per global CLAUDE.md)

### 4. Push & wait

`git push`. Poll CI to a terminal state (GitHub: `gh pr checks --watch`; GitLab: poll `head_pipeline.status` until success/failed/canceled).

### 4.1 Record `prior_fix_range`

After step 3's fix commits land and step 4 has pushed them, capture the SHA range covering this iter's fixes. This range is the **canonical source-of-truth** for two downstream consumers:

1. **Next iter's pr-review invocation** — pass as `prior_fix_range` input so pr-review's incremental mode can apply drop signal (B) self-introduced surface
2. **Gate B in step 4.5 below** — same range, same line-level attribution mechanism

```bash
# After step 4 push, before invoking the next pr-review iter:
FIRST_FIX_SHA=$(git log --format='%H' "$PREV_HEAD..HEAD" | tail -1)   # oldest fix in this iter
LAST_FIX_SHA=$(git rev-parse HEAD)                                    # newest fix in this iter
PRIOR_FIX_RANGE="${FIRST_FIX_SHA}^..${LAST_FIX_SHA}"
```

Persist `PRIOR_FIX_RANGE` (and `$LAST_FIX_SHA` as the next iter's `$PREV_HEAD`) into the babysit state file or session env. If the iter pushed a single commit, `FIRST_FIX_SHA == LAST_FIX_SHA` and the range collapses to `<sha>^..<sha>`.

If this iter pushed zero commits (CI re-run only) → no fix range to record; skip the Gate B self-introduced check for the next iter, but still run Gate A as normal.

**Why not compute lazily at Gate B**: computing at push time anchors the range to the exact commits that addressed iter (N-1) findings. Lazy computation at Gate B time could pick up unrelated commits if the user manually edits the branch between iters.

### 4.5 Self-feedback loop gates

After pushing this iter's fixes and waiting for CI green, before looping back to step 1, run TWO sub-gates that catch different self-feedback failure modes. Without these, an automated reviewer paired with an automated babysitter can spend N iterations either chasing test-hygiene nits (Gate A) or chasing race-of-race surfaces (Gate B).

Both gates parse pr-review's inline comments on this PR:

```bash
gh api repos/$OWNER/$REPO/pulls/$N/comments \
  --jq '[.[] | select(.body | contains("<!-- pr-review:finding-id=")) |
         {id, created_at, path, line, body,
          justification: (.body | capture("<!-- pr-review:justification=(?<j>[^ ]+) -->").j),
          race_meta: (.body | capture("\\[window=(?<w>[^,]+), damage=(?<d>[^,]+), recovery=(?<r>[^\\]]+)\\]") // null)}]'
```

Take only findings created since the previous iter's HEAD sha (the new ones this iter introduced).

#### Gate A: Diminishing Returns (only-hygiene iter)

**Fires** when ALL of:

- ≥1 new pr-review finding this iter
- ZERO new findings have `justification ∈ {Reachable, Precedent, Asymmetric, Historical}`
- ALL new findings are `justification=Hygiene` (or missing — treat missing as Hygiene)

**Action**: STOP automatic loop, skip step 5's normal decision, jump to step 6 with:

```
Status: needs-user-input (diminishing returns)

This iter's pr-review surfaced only hygiene findings — no Reachable / Precedent /
Asymmetric / Historical justification on any new finding.

Hygiene followups (N):
  <list — id, slug, file:line, one-line failure mode>

Continuing the loop will likely surface more hygiene from the same code paths.

Your call:
  (s) ship — open a single follow-up issue collecting the hygiene items, mark PR ready-to-merge
  (p) polish — keep looping (override the gate for this round)
  (r) re-review-full — challenge whether the self-loop missed anything (force `mode=full` on next pr-review)
```

#### Gate B: Convergence Audit (race-of-race iter)

Catches the failure mode where iter (N-1)'s fix introduces a new race / state-transition surface, the reviewer flags it as a Reachable finding, the next fix introduces yet another race surface, ad infinitum. Gate A does NOT catch this — those findings carry `justification=Reachable` and are individually valid; the divergence is only visible at cluster level.

**`prior_fix_range`**: use the range recorded in [step 4.1](#41-record-prior_fix_range). This is the same range fed to pr-review's incremental-mode invocation, so Gate B's self-introduced check and pr-review's drop signal (B) operate on identical evidence. If step 4.1 recorded nothing (iter N-1 pushed no commits), Gate B does not fire — there is no iter (N-1) fix surface to converge against.

**Fires** when ALL of:

- `iter ≥ 3` (first two iters are normal review cadence, not divergence)
- ≥ 2 new findings this iter cite `file:line` inside `prior_fix_range` — i.e. critiquing iter (N-1)'s freshly-added surface
- ≥ 2 of those findings are race-class — detection is OR of:
  - (i) carries `[window=..., damage=..., recovery=...]` meta from pr-review's race-class metadata requirement, OR
  - (ii) slug/category keyword-matches one of: `race | TOCTOU | concurren | sweep | lifecycle | state-transition | debounce | claim | lease | fence | stale | orphan | race-window`, OR
  - (iii) `\bwindow=` (matches the meta-tag prefix even when full meta is malformed) OR `atomic.*race | race.*atomic` (require co-occurrence to avoid catching DB-transaction `atomic` and frontend-viewport `window` noise)

Keyword design notes: bare `window` and bare `atomic` are deliberately excluded — they false-positive on rate-limiter / viewport / DB-transaction-correctness comments. `TOCTOU` is the canonical security-race term and matches Codex findings that bypass the meta-tag path. `debounce / claim / lease / fence` cover distributed-locking vocabulary; `stale / orphan` cover sweep-race descriptions.

How to verify file:line inside prior_fix_range:

```bash
git diff --name-only $prior_fix_range                  # files touched
git diff -U0 $prior_fix_range -- <file>                # line-level attribution
```

**Action**: STOP automatic loop, run Convergence Audit for the cluster. For each race-class finding, apply the [Wontfix Template](#wontfix-template) five-step decision:

1. **Window**: estimate ms / s / min / hr between the race operations (use the meta tag if present)
2. **Damage**: classify as `data-loss | deadlock | inconsistency | latency | marginal`
3. **Asymmetric check**: is the failure mode security / data-integrity / billing?
4. **Mitigation cost**: does the proposed fix introduce a new race surface?
5. **Recovery path**: does fault tolerance / next webhook / sweeper cover the race?

Audit verdict per finding:

| Verdict                   | When                                                                                                                                                                                                         |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **modify** (Asymmetric)   | Justification is Asymmetric (security / data-loss / data-integrity / billing) → ALWAYS modify, regardless of mitigation cost                                                                                 |
| **modify** (damage gate)  | `damage` value is `data-loss` / `deadlock` / `inconsistency` → modify even if Justification is not formally Asymmetric. These damage classes have no acceptable "fault tolerance" answer                     |
| **modify** (safe fix)     | non-Asymmetric, `damage ∈ {latency, marginal}`, BUT mitigation does NOT introduce new race surface → modify (no race-of-race risk)                                                                           |
| **wontfix-with-template** | non-Asymmetric + `damage ∈ {latency, marginal}` + `recovery=has` + mitigation introduces new race surface → reply using Wontfix Template. ALL five conditions required; missing any → fall through to modify |
| **defer-followup**        | valid concern but resolution requires infrastructure (e.g. real DB test, schema migration, new background job) that belongs to a follow-up issue                                                             |

Report to user:

```
Status: convergence-audit (race-of-race detected)

iter (N-1) fix surface attracted N race-class findings this iter (cluster):
  <id> <slug> @ <file:line>  window=<w> damage=<d> recovery=<r>
  ...

Audit verdict per finding:
  <id>: modify    — <reason: Asymmetric / mitigation safe / etc>
  <id>: wontfix   — <five-field summary from Wontfix Template>
  <id>: defer     — <followup issue suggestion>

Your call:
  (a) accept all verdicts (post wontfix replies via template, address modify items, open defer issues)
  (m) modify a specific verdict — say which finding-id and target verdict
  (s) ship — accept all wontfix + defer as-is, mark PR ready-to-merge
  (p) override audit — treat as normal iter, loop back to step 1
```

Gate B does NOT fire when:

- Cluster contains any Asymmetric finding — Asymmetric (security / data-loss / data-integrity / billing) bypasses the convergence escape just as it does in pr-review's drop signal (B). Surface them and modify
- `iter < 3` — early iters are normal review cadence
- Race-class meta is missing AND no slug/category keyword match — keeps gate narrow to actual race domain; non-race convergence (e.g. naming-bikeshed) falls back to Gate A or normal flow

Rationale: Gate A catches iters where everything is hygiene; Gate B catches iters where individually-valid race findings cluster on freshly-introduced surfaces. Together they cover the two main self-feedback failure modes without suppressing genuine Asymmetric findings or third-party signal (Codex / SonarQube / Snyk findings without pr-review's metadata bypass both gates and route through normal step 2 dedup + 3-round escalation).

### 4.6 Wontfix Template

Used by step 4.5 Gate B (Convergence Audit) and as a manual reply template for race / state / sweep / atomic class findings where modification would introduce new race surfaces.

Five fields are **minimum-required**. Missing any one → finding deserves modification, not wontfix.

```
Wontfix — deliberate trade-off.

Race window: <ms / s / min / hr> between <op A> and <op B>.
Precondition: <only fires when X is in Y state for N+ time>
Damage if race fires: <not data-loss / not deadlock / only X happens N seconds earlier than ideal>
Recovery path: <new event / cron sweeper / next webhook covers it; user-visible behavior unchanged>

Asymmetric check: <not security / not data-loss / not data-integrity / not billing>
Mitigation cost: <atomic re-check / two-step merge into transaction is doable, but introduces new race-of-race surface at X>

Acknowledged as known trade-off; fault tolerance covers genuinely <abandoned / stranded / dropped> class.
Tracking: <if needed, opened followup issue X>
```

**Field semantics**:

- **Race window** — concrete time estimate, not "small". `ms` for tight CAS, `min` for sweep cycle gap, `hr` for cron lifecycle. Reviewer needs the magnitude to judge.
- **Precondition** — what state the system must already be in for the race to even matter. If precondition is rare or already-degraded, race is acceptable.
- **Damage** — concrete user / data observation, not "could be a problem". If you cannot describe damage in one line, the finding may not actually be Reachable.
- **Recovery path** — must name a concrete mechanism (next webhook / sweeper run / cron / fault-tolerant retry). "It'll probably be fine" is not a recovery path.
- **Asymmetric check** — explicit declaration that finding is not security / data-integrity / billing. Wontfix is INVALID for Asymmetric findings — modify them.
- **Mitigation cost** — name the new race surface the proposed fix would introduce. "race-of-race" is the load-bearing reasoning.

**Reference example**: PR #148 `sweepAbandonedTasklessThreads` two-UPDATE race — Codex flagged "re-check thread state before abandoning queued events"; race window was milliseconds between two sweep UPDATEs, precondition was thread already stranded 1+ hour, damage was `marginal` (already-stranded events terminalize seconds earlier than ideal), recovery path was new webhook hits reactivation gate. Wontfix posted; PR shipped.

**When NOT to use**:

- Any of the five fields cannot be filled honestly → finding is real, modify it. Wontfix Template is for the specific case where modification introduces equivalent or worse race surface; it is NOT a generic decline template.
- **Dev-stage self-review context (no separate session between code author and verdict reasoner)**: do NOT fill these fields from main-session memory. Babysit normally runs in a session separate from the code author, which is what makes Wontfix Template safe to apply — the babysit session has no prior commitment to the design and can honestly reason about damage / recovery / mitigation cost. In a dev-stage self-review loop (same session wrote the code AND is reasoning about findings), author-narrative bias compounds — bug-free framing produces the strongest detection drop among framing conditions tested across 6 LLMs (Mitropoulos et al., *Measuring and Exploiting Contextual Bias in LLM-Assisted Security Code Review*, [arXiv:2603.18740](https://arxiv.org/abs/2603.18740)). Pause and either (a) hand off to a separate session for the verdict, or (b) use a fresh-spawn verdict subagent that independently derives `damage` / `recovery` / `mitigation cost` from code, not from the finding object's fields. The Deriver-pattern verdict subagent is not built as a skill yet — until it is, treat dev-stage wontfix decisions as advisory and surface them to the user.

### 5. Decide

- ✅ All checks green AND all Valid feedback resolved → **Report** (step 6)
- 🟡 New comment / check status changed mid-cycle → back to step 1
- 🔴 Hit 3-failure stop, invisible-findings gate, dedup 3-round escalation, OR something genuinely needs human judgment → **Report** with `blocked` / `needs-user-input`

### 6. Report (end of run, not auto-merge)

```
PR/MR: <link>
Status: ready-to-merge | needs-user-input | blocked
Checks: <green>/<total>
Addressed (this run): <list of SHA → comment ref + one-liner>

Awaiting your decision:
  Discuss (I did NOT reply): <list with comment text + my read of the ambiguity>
  Out-of-scope: <list>  → open follow-up issues for any of these? (y/N per item)

Blockers (if any): <description + what I tried>

Next command: gh pr merge --squash <id>   # or: glab mr merge <id>
```

After the report, if there are out-of-scope items, ask once: open follow-up issues for which ones? Open only the ones the user picks (`gh issue create` / `glab issue create`), and edit the report's reply on each MR/PR comment to link the new issue.

## What I never do without asking

- Reply, dismiss, or implement based on **Discuss** items — list them, stop.
- Open follow-up issues for **Out-of-scope** items without confirming the list with the user first.
- Merge the PR/MR. Even when fully green, report ready-to-merge and let the user run the merge.
- Force-push, amend pushed commits, skip hooks (`--no-verify`), or bypass signing.
- Loop forever — if a cycle produces no new work and nothing is resolved, stop and report.
