# Cadence

PR review lifecycle plugin for AI coding agents. Three skills, one rhythm: **self-review → pr-review → pr-babysit**.

## The Problem

```
You: "fixed it, ready to push"
Agent: *pushes, opens PR with author-bias review, ignores half of reviewer feedback,
        PR sits open for days while CI flakes*
```

## With Cadence

```
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

## Skills

| Command | Stage | What it does |
|---|---|---|
| `/cadence:self-review` | pre-push | Codex (OpenAI) cross-model review of your branch in an auto-loop. Per-finding fix + re-review until converged. Cross-model so author bias doesn't compound. |
| `/cadence:pr-review` | on PR open | Multi-role subagent dispatch — security, staff-engineer, sdet, spec-auditor — runs against the PR diff in isolated fresh contexts. Posts sticky summary + inline comments. Has a HARD-GATE preventing main-session self-review (author bias). |
| `/cadence:pr-babysit` | until merge | Watches PR/MR until CI is green and every valid reviewer feedback is addressed. Triages comments, replies inline, escalates 3-round bot deadlocks. Reports ready-to-merge (never auto-merges). |

## Why three skills, not one

Each stage has a different bias profile and a different output target:

- **self-review** is cross-model (codex grades Claude's diff) because same-model self-review collapses. Output is back-to-you.
- **pr-review** is multi-role *and* main-session-isolated because PR review needs perspective diversity *and* a clean context. Output is GitHub/GitLab sticky + inline.
- **pr-babysit** is sequential and CI-driven because it's a wait-loop, not a review. Output is patches + thread replies.

Trying to fold these into one skill loses either the bias isolation or the workflow fit.

## Installation

### Claude Code

```bash
# Direct (dev):
claude --plugin-dir /path/to/cadence

# Or via marketplace.json in ~/.claude/:
# point ~/.claude/plugins/marketplace.json at this repo
```

Skills become `/cadence:self-review`, `/cadence:pr-review`, `/cadence:pr-babysit`.

### Codex CLI

The repo includes `.agents/plugins/marketplace.json` and `plugins/cadence/` overlay — point your Codex marketplace at this repo.

### Requirements

- `gh` (GitHub) or `glab` (GitLab) CLI installed
- `codex` CLI installed (only for `self-review`)
- Project with `package.json` or `Makefile` (for test-command detection in `self-review`)

## Repository Layout

```
cadence/
├── .claude-plugin/plugin.json         # Claude Code plugin manifest
├── .claude-plugin/marketplace.json    # Claude Code marketplace manifest
├── .codex-plugin/plugin.json          # Codex plugin manifest
├── .agents/plugins/marketplace.json   # Codex marketplace manifest
├── plugins/cadence/                   # Codex plugin overlay (symlinks to root)
├── skills/
│   ├── self-review/SKILL.md
│   ├── pr-review/SKILL.md + 4 role prompts (sdet, security, staff-engineer, spec-auditor)
│   └── pr-babysit/SKILL.md
├── CLAUDE.md                          # AGENTS.md → CLAUDE.md symlink
└── LICENSE                            # MIT
```

## Design Notes

- **Author bias is the core constraint.** All three skills are designed around the research finding that framing a diff as "bug-free" produces the strongest detection drop among framing conditions tested across 6 LLMs ([Mitropoulos et al., arXiv:2603.18740](https://arxiv.org/abs/2603.18740)). The skills are structured so the entity *finding* the issue is never the entity that *wrote* the code.
- **Cross-model + multi-role + main-session-isolation** are three independent mitigations applied where each is the cheapest fix.
- **`pr-review` has a `mode: local`** for callers (supervisor sessions doing pre-PR critique) that want findings JSON instead of GitHub posts. Same dispatch, different output target.

## License

MIT
