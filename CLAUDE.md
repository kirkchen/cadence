# CLAUDE.md

This file guides Claude Code (and other AI agents via the `AGENTS.md` symlink) when working *on* the cadence plugin itself.

> If you are *using* cadence in another project — read the user-facing [README.md](README.md). This file is for contributors.

## What Cadence Is

Cadence is a Claude Code / Codex plugin shipping skills for the AI coding lifecycle — from issue intake to merged PR:

- `triage-issue` — single-issue triage: labels + meta comment
- `investigate-issue` — single-issue deep-dive: verify claims at HEAD, propose fix direction
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
├── triage-issue/SKILL.md
├── investigate-issue/SKILL.md
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

1. **`pr-review` HARD-GATE**: pr-review refuses to run when called from the same session that authored the diff. Author bias destroys finding precision. If you tweak the gate, read [Mitropoulos et al., arXiv:2603.18740](https://arxiv.org/abs/2603.18740) (cited in `pr-review/SKILL.md`) first — bug-free framing produces the strongest detection drop among framing conditions tested across 6 LLMs.
2. **Subagent dispatch is mandatory** in pr-review — never collapse the 4 role prompts into one prompt, and never let the dispatcher answer a question a subagent exists to answer (the `rebuttal-assessor` that weighs an author's rebuttal is one of these: the party holding the finding cannot fairly judge an argument against it). Each role is a *fresh* context with a different mental model. Pooling them defeats the diversity guarantee. Dispatch must also be **blocking** (`run_in_background: false` on every Agent call): the host defaults to background and its tool description discourages opting out, so leaving this to the model's judgement means the dispatcher ends its turn with reports still in flight — and on a non-interactive host (Rhythm worker, CI), turn end *is* run end, so those reports and the whole publish step are silently lost. Any skill text about dispatch must keep saying this explicitly; "in parallel" alone is not enough.
3. **`self-review` is cross-model**, not Claude-vs-Claude. Calling Claude to grade Claude's own diff is the same bias trap as pr-review's gate.
4. **`pr-babysit` never auto-merges.** It reports ready-to-merge and stops. Merge is a human decision.
5. **Intake skills are single-issue.** `triage-issue` / `investigate-issue` process one issue per invocation. Batch / backlog walkthroughs are the caller's responsibility — keep the skill a pure unit. Don't pull `gh issue list` orchestration into the skill itself.
6. **No project-specific assumptions in skill text.** Cadence is meant to be repo-agnostic. Hardcoded paths, internal hostnames, project names, language defaults → reject in review.

## Updating Skills

- Edit `skills/<name>/SKILL.md` directly.
- For `pr-review`, the four role prompt files are loaded by the dispatcher; keep their personas distinct.
- `pr-review`'s severity tiers, volume caps, prose ceiling and drop signals are calibrated against a measured corpus — see [docs/pr-review-severity-calibration.md](docs/pr-review-severity-calibration.md) before changing any of those numbers. It also records the run-to-run noise floor, which is what tells you whether a future measurement means anything.
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

**This is a decision, not a gap.** Cross-model review has asked for an integration harness (stubbed `gh` / `glab` covering first-run publish, partial inline failure, and reconciliation) on four consecutive passes, and it has been declined each time. The reasoning:

- The units under test are prompts. A harness can exercise the *shell recipes* inside them, which is a thin slice of the risk; it cannot test whether an agent reading the prompt does the right thing, which is where the defects actually live — every finding cross-model review has produced on this repo so far has been a contradiction between two instructions, not a broken command.
- Standing up a first test framework here is a product decision about what cadence is, not a fix. It earns its place when there is enough shell in the publish path that a human cannot eyeball it, and that threshold has not been reached.

Reviewers: do not re-raise this as a finding. If you think the threshold has been crossed, say what specifically changed rather than restating the general case.

## History

Extracted from [`kirkchen/rhythm`](https://github.com/kirkchen/rhythm) in 2026-05. Originally `rhythm/.claude/skills/` contained `self-review` / `pr-review` / `pr-babysit` (PR lifecycle) alongside `triaging-issues` / `investigating-issues` (issue intake) and `beat-supervise` (Rhythm-specific worker supervisor). The PR lifecycle three landed in cadence first; the issue intake pair followed in 2026-05 when cadence's scope was widened from "PR review lifecycle" to "AI coding lifecycle". `beat-supervise` stays in rhythm because it depends on Rhythm's worker/supervisor runtime.
