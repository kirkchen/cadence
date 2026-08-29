# pr-review severity calibration

Where the numbers in `skills/pr-review/SKILL.md` came from. Read this before changing the P1 gate, the volume caps, the prose ceiling, or the drop signals — several of them look arbitrary and are not, and two of them were *measured to do nothing* and are deliberately loose.

Measured 2026-08. Source: 215 findings the skill produced across 13 merged and in-flight MRs in one private repo. That repo is not identified here and its findings are not quoted; only shapes and aggregates.

## What was measured

Two labels per finding.

**Objective — what the author did.** Recovered from the review threads: replied, rebutted, silently changed the cited line, or ignored. This is the [Tricorder](https://www.cs.umd.edu/class/spring2019/cmsc414/papers/tricorder-building-a-program-analysis-ecosystem.pdf) definition of an *effective* false positive: a finding the developer chose not to act on is a false positive even when it is technically correct. It needs no judgement to assign.

**Subjective — was it worth a thread.** Each finding hand-classified as material / minor / nit / scope-creep / false-positive by agents that could see the author's replies. Secondary metric only: the labels come from the same model family being evaluated, so they carry self-preference risk.

Findings emitted in an MR's *final* iteration are excluded from both. They have no subsequent diff, so "author silently fixed the line" is unobservable and they would read as ignored. That leaves 161 of 215.

## Baseline (before any change)

| | |
| --- | --- |
| Effective FP | **50%** — half of all findings drew no action |
| P1 that was not material | **62%** |
| P3 tier | never used, on any finding |
| Findings per MR | up to 52; four MRs above 25 |
| Findings per 100 changed lines | 0.2–1.1 on code diffs, 3.3–70 on prose diffs |

For scale: Tricorder puts a review-time analyzer on probation at 10% and switches it off above 25%; Coverity's field observation is that developers ignore a tool past 30%.

## How the change was evaluated

Replay, not re-run. The 161 findings were frozen as input and only the *gate* re-ran — inclusion threshold, severity assignment, tiering. This isolates the rules from generation variance, at the cost of saying nothing about what a changed prompt would have found in the first place.

**Two arms, both blind to the labels.** Arm A applied the pre-change rules, Arm B the post-change rules. The control arm is what made the result interpretable, and it is the part worth repeating: without it the obvious reading would have been "the new rules cut noise 15 points". What actually happened is that *both* rulesets cut noise to roughly the same level, and they differ almost entirely in what they destroy on the way.

## Result

On the 7 merged MRs, where the objective label is most trustworthy (n=80):

| | inline threads | effective FP | material delivered |
| --- | --- | --- | --- |
| Baseline | 80 | 50% | 23/23 |
| Arm A — old rules | 30 | 36% | **16/23** |
| Arm B — new rules | 42 | 35% | **20/23** |

Across all 13 MRs the P1 tier went from 54% non-material under the old rules to 16% under the new ones, and the worst MR went from 43 inline threads to 15.

The old threshold was not lax. It was *blind*: it dropped aggressively without a severity model, so it discarded a whitelist bypass and a check-then-act race alongside the nits. The new rules keep more findings and tier them down instead, which is why material delivery holds at 20/23 while noise falls just as far.

## Which number came from which measurement

**P1 gate — "breaks behavior / leaks or corrupts data / blocks rollback".** 62% of P1 was not material, and the inflation was concentrated in two shapes: "this correct code has no test" and "this comment did not follow the rename". Both are now capped below P1 by name. `Justification: Asymmetric` is explicitly severed from tier because it was the mechanism — nearly half of all Asymmetric findings were filed P1.

**Inline cap of 12, P0/P1 exempt.** A cap of 8 was tried first and **measured to make things worse**: effective FP moved 41% → 43% while four material findings were lost, because material findings are not concentrated in the top tiers — 16 of them sat in P2. The cap does none of the noise reduction; the P3 tier leaving the inline channel does all of it. 12 is a guard against a pathological iteration, not a tuning knob. Tightening it is the wrong reflex when a review feels noisy.

**Prose ceiling, applied per finding rather than per PR.** Prose findings run 10–100× the density of code findings, because a document offers an unbounded supply of mechanically enumerable defects. The ceiling was first written as a whole-PR flag; that missed 19 prose findings sitting inside code-heavy PRs, including every finding in the MR with the worst nit rate in the sample — more than half its findings cited `.feature` and `.md` files while its diff was mostly code. No threshold fixes this, because the error is in the unit of judgement. Re-measured after the change: no material finding was wrongly capped, and effective FP on the affected subset fell 25% → 18%.

**Drop signals (E) previously-dismissed and (F) scope-declared.** The two largest recoverable causes of no-action findings were re-raising something the author had already rebutted in a thread, and asking for a sweep the PR description had explicitly bounded. Neither is a generation problem — both are state the review had and never read.

**Reply harvesting reads standalone PR comments, not only threads.** Authors put considered rebuttals in top-level comments. A finding published without a line anchor — every Q-class item, every file-level finding — has no thread to reply in at all, so this is its author's only channel. One rebuttal in the sample opened by saying exactly that.

**Payload assertions.** 27 of 215 findings (12%) reached the PR with an empty Evidence block, four of them with the inline-code spans in the failure mode silently stripped, leaving unreadable prose. One was the highest-value finding in its MR. The cite-or-drop rule was already in the skill; it was enforced at emission and never at publish. **Root cause confirmed**, and reproduced live while writing these fixes: `git commit -m "補 `dismissed` 這個輸入"` emitted `command not found: dismissed` and committed `補  這個輸入`. Inside double quotes the shell runs a backtick span as a command and substitutes its (empty) stdout, leaving the double space that is the observed signature. The earlier objection — that some spans survive in the same comment — is explained rather than contradicted: only the portion of a body assembled by interpolation is affected, so a payload built partly by heredoc and partly by interpolation ships with its header intact and its failure mode gutted.

## Noise floor — read this before trusting any future delta

The same 47 findings, replayed twice under identical rules, first produced **different verdicts on 51%**. Almost all of that turned out to be one word: one run spent its exclusions as `DROP` (10 of them, zero `Q`), the other as `Q` (10 of them, zero `DROP`), and both mean "no thread". The instructions genuinely said both things in three places — the drop-signal heading said batch as Q, signal (E) said drop outright, and the self-check preference line said `Drop > batch`. Naming an outcome per signal fixed it, and re-measuring confirms that was the cause:

| decision | before the fix | after |
| --- | --- | --- |
| Full verdict (drop/keep + tier) | 51% | **17%** |
| **Whether it opens an inline thread** | 10% | **6%** |
| Whether it is P1 | 12% | **8%** |

The prediction going in was that the full verdict would fall sharply while the inline decision — which was never affected by the DROP/Q confusion, since both channels skip the thread — would stay roughly where it was. It did.

What remains is judgement, not ambiguity. All eight residual disagreements sit on two boundaries: four are P1↔P2 on prose findings that cleared a P0–P1 carve-out and then had to face the P1 gate separately, three are P2↔Q on whether drop signal (A) or (D) fires, one is P3↔Q. None is a case of the rules saying two different things.

**Operationally**: a single replay resolves differences of roughly 15 points or more on the full verdict, and about 10 on the publish decision. It does not resolve 5. Of the numbers in this document, the 50% → 35% effective-FP headline is above the floor; the 25% → 18% on the prose subset is not, and is reported as "no regression detected", never as an improvement.

## Not validated

Replay exercises judgement, never machinery. Three changes have never been executed even once:

- **Reply harvesting** — whether the dispatcher actually fetches both comment channels and classifies them. In the sample the ledger fired about twice, because most author replies postdate the finding they answer. Its value accrues across iterations and was not observable here.
- **Payload assertions** — whether the empty-Evidence check catches the real failure. See the unknown root cause above.
- **Sticky reconcile** — whether rebuilding the sticky from posted threads works. The bug it targets is real and observed: one sticky read `Open: none` while an unaccepted P1 thread was live 30 seconds older than it, and the MR merged four minutes later.

Replay also cannot say whether the new rules make the review *find* less, since generation is frozen by construction. And this was measured on the corpus the rules were tuned against; a held-out set — older MRs, or a live run — is the honest next measurement.

## Reproducing

Nothing here is checked in: the corpus is one private repo's review history. To redo it elsewhere, the shape is:

1. Pull every finding thread the skill produced, with author replies and diff-version system notes.
2. Derive the objective label from author behaviour. Drop final-iteration findings.
3. Strip replies and labels; hand the finding bodies plus PR descriptions to a blind gate agent.
4. **Run a control arm on the old rules.** Without it you cannot separate a rule change from an agent re-judging.
5. Score inline volume, effective FP, and material delivered. Treat anything under ~15 points as noise.
