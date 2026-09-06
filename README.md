# Cadence

AI coding lifecycle skills for Claude Code and Codex. Cover the path from issue intake to merged PR:

**triage-issue → investigate-issue → self-review → pr-review → pr-babysit**

…plus **maintenance-routine**, scheduled upkeep that joins the same pipeline one stage earlier — it *produces* the PR the review skills then handle.

## The Problem

```
You: "fixed it, ready to push"
Agent: *pushes, opens PR with author-bias review, ignores half of reviewer feedback,
        PR sits open for days while CI flakes*
```

…and upstream:

```
You: "triage the backlog"
Agent: *labels everything as medium, comments are noise, two reviewers re-do the work*
```

…and the work nobody schedules at all:

```
You: (nothing — dead code, tests that can't fail and layering drift accumulate
      for two years because there's never a week to spend on them)
```

## With Cadence

```
You: /cadence:triage-issue #234
Cadence: *reads body, decides type + priority, comments only when there's meta worth recording*

You: /cadence:investigate-issue #234
Cadence: *verifies file:line at HEAD, probes blast radius, posts verdict + fix direction*

You: /cadence:self-review
Cadence: *codex cross-model loop — finds 3 issues before you push*
         → fix → re-review → green

You: gh pr create ... && /cadence:pr-review
Cadence: *dispatches 4 fresh role subagents in parallel*
         → security-reviewer / staff-engineer / sdet / spec-auditor
         → posts sticky summary + inline comments

You: /cadence:pr-babysit
Cadence: *watches CI, triages reviewer feedback into Valid/Discuss/Out-of-scope,
          addresses valid items with atomic commits, replies to threads,
          waits until green — reports ready-to-merge*
```

…and unattended, on a schedule:

```
cron: /cadence:maintenance-routine dead-code-removal
Cadence: *reads .claude/routines.md, checks HEAD is green, scans one cold
          subdirectory, applies three conditions to nine candidates*
         → two survive → one PR, with the search evidence attached
         → (most runs qualify nothing and open nothing — that's the design)
```

## Skills

| Command | Stage | What it does |
|---|---|---|
| `/cadence:triage-issue` | intake | Assigns `type/` + `priority/` labels, drops `status/needs-triage`, leaves a meta comment only when there's judgment worth recording. One issue per invocation. |
| `/cadence:investigate-issue` | intake (deep) | Verifies the issue's claims against HEAD (file:line still valid? still reproducing?), probes blast radius, and posts a Confirmed / Dismissed / Needs-info verdict + concrete fix direction. |
| `/cadence:self-review` | pre-push | Codex (OpenAI) cross-model review of your branch in an auto-loop. Per-finding fix + re-review until converged. Cross-model so author bias doesn't compound. |
| `/cadence:pr-review` | on PR open | Multi-role subagent dispatch — security, staff-engineer, sdet, spec-auditor — runs against the PR diff in isolated fresh contexts. Posts sticky summary + inline comments. Has a HARD-GATE preventing main-session self-review (author bias). |
| `/cadence:pr-babysit` | until merge | Watches PR/MR until CI is green and every valid reviewer feedback is addressed. Triages comments, replies inline, escalates 3-round bot deadlocks. Reports ready-to-merge (never auto-merges). |
| `/cadence:maintenance-routine <archetype>` | scheduled | One mechanical upkeep pass over the repo — 7 archetypes from dead-code removal to abstraction flattening. Reads per-repo parameters from `.claude/routines.md`, opens small PRs with evidence attached, never merges, and opens nothing when nothing qualifies. |

## Why these skills, separately

Each skill targets a distinct stage with its own bias profile and output target:

- **triage-issue** is single-issue and minimal — backlog walkthroughs are the caller's job, not a skill responsibility. Output is `gh` labels + an optional meta comment.
- **investigate-issue** burns 30 min – hours per issue to verify claims against actual code before they get picked up. Output is a verdict comment with fix direction.
- **self-review** is cross-model (codex grades Claude's diff) because same-model self-review collapses. Output is back-to-you.
- **pr-review** is multi-role *and* main-session-isolated because PR review needs perspective diversity *and* a clean context. Output is GitHub/GitLab sticky + inline.
- **pr-babysit** is sequential and CI-driven because it's a wait-loop, not a review. Output is patches + thread replies.
- **maintenance-routine** is the only one with no human at the start, so its design centre is refusal, not detection — it has to be safe to ignore for a month. Output is a PR or an explicit nothing.

Trying to fold these into one skill loses either the bias isolation or the workflow fit.

## Maintenance routines

`maintenance-routine` ships seven archetypes. They're listed here in descending objectivity, which is also the order to adopt them in — the top of the list lands most reliably on the first attempt:

| Archetype | What it does |
|---|---|
| `dead-code-removal` | Deletes code provably unreachable by static search |
| `useless-test-pruner` | Finds tests that cannot fail; fixes them, deletes as a last resort |
| `abstraction-police` | Fixes violations of layering rules the repo has written down |
| `dup-unifier` | Converges duplicate implementations that provably change together |
| `logic-bugfixer` | Models one module against its spec, fixes bugs test-first |
| `logic-simplifier` | Untangles logic inside a single function |
| `abstraction-improver` | Flattens indirection layers that carry no meaning |

**The skill supplies the method; the repo supplies the parameters.** Scan scope, forbidden paths, verification commands and the framework traps that defeat static search all live in `.claude/routines.md` in the target repo — the schema is in [skills/maintenance-routine/SKILL.md](skills/maintenance-routine/SKILL.md) § Repo Config Schema, and the skill refuses to run without it rather than guessing.

That split exists so the tuning loop stays closed: when a routine's PR gets rejected, the fix is usually a repo parameter, and it has to be editable by the same routine that opens PRs against that repo.

Branches are named `claude/<archetype>/<YYYY-MM-DD>-<n>`, which is the whole instrumentation — merge rate comes out of `gh pr list --search "head:claude/<archetype>" --state all`, with no platform support needed.

## Installation

### Claude Code

```bash
# Direct (dev):
claude --plugin-dir /path/to/cadence

# Or via marketplace.json in ~/.claude/:
# point ~/.claude/plugins/marketplace.json at this repo
```

Skills become `/cadence:triage-issue`, `/cadence:investigate-issue`, `/cadence:self-review`, `/cadence:pr-review`, `/cadence:pr-babysit`, `/cadence:maintenance-routine`.

### Codex CLI

The repo includes `.agents/plugins/marketplace.json` and `plugins/cadence/` overlay — point your Codex marketplace at this repo.

### Requirements

- `gh` (GitHub) or `glab` (GitLab) CLI installed
- `codex` CLI installed (only for `self-review`)
- Project with `package.json` or `Makefile` (for test-command detection in `self-review`)
- `.claude/routines.md` committed in the target repo (only for `maintenance-routine`)
- A scheduler to fire `maintenance-routine` — cron, CI schedule, or an agent workbench with a time trigger. Cadence provides the pass, not the clock.

## Repository Layout

```
cadence/
├── .claude-plugin/plugin.json         # Claude Code plugin manifest
├── .claude-plugin/marketplace.json    # Claude Code marketplace manifest
├── .codex-plugin/plugin.json          # Codex plugin manifest
├── .agents/plugins/marketplace.json   # Codex marketplace manifest
├── plugins/cadence/                   # Codex plugin overlay (symlinks to root)
├── skills/
│   ├── triage-issue/SKILL.md
│   ├── investigate-issue/SKILL.md
│   ├── self-review/SKILL.md
│   ├── pr-review/SKILL.md + 4 role prompts (sdet, security, staff-engineer, spec-auditor)
│   ├── pr-babysit/SKILL.md
│   └── maintenance-routine/SKILL.md + routines/ (7 archetype files)
├── CLAUDE.md                          # AGENTS.md → CLAUDE.md symlink
└── LICENSE                            # MIT
```

## Design Notes

- **Author bias is the core constraint for the review skills.** `self-review` / `pr-review` / `pr-babysit` are designed around the research finding that framing a diff as "bug-free" produces the strongest detection drop among framing conditions tested across 6 LLMs ([Mitropoulos et al., arXiv:2603.18740](https://arxiv.org/abs/2603.18740)). The skills are structured so the entity *finding* the issue is never the entity that *wrote* the code.
- **Cross-model + multi-role + main-session-isolation** are three independent mitigations applied where each is the cheapest fix.
- **`pr-review` has a `mode: local`** for callers (supervisor sessions doing pre-PR critique) that want findings JSON instead of GitHub posts. Same dispatch, different output target.
- **The intake skills (`triage-issue` / `investigate-issue`) are single-issue by design.** Batch / backlog work is the caller's responsibility — the skill stays a pure unit, the caller loops.
- **`maintenance-routine` never merges, and a run that opens nothing is a success.** Both follow from it running unattended: the only version of this that stays safe over months is one where the qualification bar is high enough that silence is the common outcome, and where a human still stands between every diff and the default branch.

## License

MIT
