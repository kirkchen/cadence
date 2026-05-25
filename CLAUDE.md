# CLAUDE.md

This file guides Claude Code (and other AI agents via the `AGENTS.md` symlink) when working *on* the cadence plugin itself.

> If you are *using* cadence in another project — read the user-facing [README.md](README.md). This file is for contributors.

## What Cadence Is

Cadence is a Claude Code / Codex plugin that ships three PR-lifecycle skills:

- `self-review` — pre-push codex cross-model loop
- `pr-review` — on-PR multi-role subagent dispatch (4 fresh role contexts)
- `pr-babysit` — until-merge CI watch + reviewer-feedback triage

Each skill lives in `skills/<name>/SKILL.md`. There is **no application code** — cadence is purely a set of SKILL.md prompts.

## Repository Structure

```
.claude-plugin/             # Claude Code plugin manifest + marketplace
.codex-plugin/              # Codex plugin manifest
.agents/plugins/            # Codex marketplace manifest (kirkchen-cadence)
plugins/cadence/            # Codex plugin overlay — symlinks back to root
skills/
├── self-review/SKILL.md
├── pr-review/
│   ├── SKILL.md
│   ├── sdet-prompt.md
│   ├── security-reviewer-prompt.md
│   ├── spec-auditor-prompt.md
│   └── staff-engineer-prompt.md
└── pr-babysit/SKILL.md
```

The dual-manifest layout means one repo serves both Claude Code (`.claude-plugin/` + repo root) and Codex (`plugins/cadence/` overlay via symlinks). Single source of truth in `skills/`.

## Design Invariants — Do Not Break These

1. **`pr-review` HARD-GATE**: pr-review refuses to run when called from the same session that authored the diff. Author bias destroys finding precision. If you tweak the gate, read the Nikiporets et al. citation in `pr-review/SKILL.md` and understand the 97.2% → 3.6% collapse first.
2. **Subagent dispatch is mandatory** in pr-review — never collapse the 4 role prompts into one prompt. Each role is a *fresh* context with a different mental model. Pooling them defeats the diversity guarantee.
3. **`self-review` is cross-model**, not Claude-vs-Claude. Calling Claude to grade Claude's own diff is the same bias trap as pr-review's gate.
4. **`pr-babysit` never auto-merges.** It reports ready-to-merge and stops. Merge is a human decision.
5. **No project-specific assumptions in skill text.** Cadence is meant to be repo-agnostic. Hardcoded paths, internal hostnames, project names → reject in review.

## Updating Skills

- Edit `skills/<name>/SKILL.md` directly.
- For `pr-review`, the four role prompt files are loaded by the dispatcher; keep their personas distinct.
- Commit with conventional commits (`feat(pr-review): ...` / `fix(pr-babysit): ...`).
- Version bump in `.claude-plugin/plugin.json` + `.codex-plugin/plugin.json` for breaking changes.

## Testing Locally

```bash
# Claude Code:
claude --plugin-dir $(pwd)
# Then in a Claude Code session: /cadence:self-review

# Codex CLI:
# point Codex at .agents/plugins/marketplace.json
```

No automated tests yet — skills are prompts, validated empirically through dogfooding on real PRs.

## History

Extracted from [`kirkchen/rhythm`](https://github.com/kirkchen/rhythm) in 2026-05. Originally lived under `rhythm/.claude/skills/` alongside other Rhythm-specific skills (`beat-supervise`, `triaging-issues`, `investigating-issues`). Pulled out when the three PR-lifecycle skills proved generic enough to share across projects.
