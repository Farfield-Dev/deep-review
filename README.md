# Farfield Deep Review

The bug-finding recipe Farfield uses in production, packaged as a Claude Code skill so you can run it on your own repo.

> Farfield is a Slack-native quality agent for AI-heavy engineering teams. We use this exact recipe (plus team memory, production signals, scheduling, and Slack integration) to find and fix bugs before they become escalations. → [farfield.dev](https://farfield.dev)

## Install

```bash
git clone https://github.com/Farfield-Dev/deep-review ~/.claude/skills/deep-review
```

That's it. Claude Code auto-discovers skills under `~/.claude/skills/`.

## Run

Inside Claude Code, in the root of the repo you want to review:

```
/deep-review
```

Or for the fast pass (skips team-intent and product-scan):

```
/deep-review --fast
```

The skill writes its working artifacts to `./.deep-review/` (add it to your `.gitignore`) and the final report to `./findings.md`.

## What it finds

Deep Review is opinionated. Every finding has to map to one of seven impact categories or it gets dropped:

- **REVENUE_LEAK** — money lost, miscounted, double-charged, unbilled, refunded incorrectly
- **SUPPORT_BURDEN** — produces support tickets users will actually file
- **DATA_LOSS** — user data destroyed, corrupted, or unrecoverable
- **SECURITY_BREACH** — cross-tenant leak, unauth access, RCE, privilege escalation
- **COMPLIANCE** — GDPR/PCI/SOC2 violation with concrete regulator or contract risk
- **PROD_INCIDENT** — actively paging or about to (latency, OOM, crash loops)
- **BRAND_DAMAGE** — public, visible failure that erodes trust at scale

If a candidate finding fits none of these, it's dropped. That's the rule that kills the noise.

Findings are validated adversarially before they make it into the report. Every confirmed finding is one we'd defend at 100% confidence — a real user hits a real consequence on a real code path in a real configuration.

## What it costs

This is honest because the HN comments will make it honest anyway.

A cold review on a medium-sized repo (10–50k LOC, ~30 actions in the inventory) runs:

- **Architecture map + team intent**: Sonnet, ~$0.50–$2
- **Action traces + product scan**: Opus, ~$4–$15
- **Adversarial validation**: Opus, ~$1–$5

Plan on **$5–$25 per review** on your own Anthropic key. The orchestrator uses Sonnet for the cheap phases and Opus for the phases where it matters. If you want to cap costs, run `--fast` (skips team-intent and product-scan).

## What's in the box

One skill (this directory), one entry point (`SKILL.md`), five phase reference files loaded on demand:

| File | Phase | What it does |
|---|---|---|
| `SKILL.md` | Orchestrator | Routes the pipeline, embeds sub-agent prompts, defines the output schema |
| `architecture-map.md` | 1 | Phase 0 signals (git, deps, linter, runtime grep), feature map, action inventory, workflow ledger, integration map, impact taxonomy, severity calibration |
| `team-intent.md` | 2 | Class-level brief from 2 months of commits — bug-class mix, mode, confidence, trust-critical surfaces |
| `action-trace.md` | 3 | Trace every user action end-to-end in parallel sub-agents. **The primary bug-finding phase.** |
| `product-scan.md` | 4 | UX/product-level bugs (parallel pass) |
| `adversarial-validate.md` | 5 | 100%-confidence validation rubric, writes `findings.md` |

> **🚧 Status (day 0)**: phases 1 and 2 are extracted and ready. Phases 3, 4, and 5 are landing this week. The orchestrator SKILL.md and architecture are stable.

## What this is *not*

Deep Review is the same recipe Farfield runs in production. But this OSS version intentionally ships **without** the things that make Farfield's paid product compounding:

- ❌ No team memory across runs (every review is cold)
- ❌ No production signal integration (no Sentry, no log correlation)
- ❌ No Slack context (no thread history, no team-conversation grounding)
- ❌ No scheduled cadence (you trigger it manually)
- ❌ No PR creation or autonomous fix loop
- ❌ No dedup against existing issues (every run reports all findings fresh)
- ❌ No issue tracker integration (output is a markdown file)

If you want any of those, that's [Farfield](https://farfield.dev). The OSS review is the floor of what the methodology can do; Farfield is what happens when you layer memory, production signals, and Slack-native investigation on top.

## How this compares

| Tool | Bug-class catch rate | False positive discipline | Memory across runs | Production signal | Slack-native |
|---|---|---|---|---|---|
| GitHub Copilot Review | Pattern-matching | Low | No | No | No |
| Sourcery / DeepSource | Style + types | Medium | No | No | No |
| CodeRabbit | Diff-scoped | Medium | Workspace-level | No | Notifications only |
| Greptile | Full-repo context | Medium-high | Workspace-level | No | Notifications only |
| **Farfield Deep Review (OSS)** | Action-trace + workflow | High (adversarial validation) | No | No | No |
| **Farfield (paid)** | Same recipe | Same | Yes | Yes (Sentry / logs) | Yes |

## Limitations

- **First review is cold and expensive.** Subsequent runs in the same checkout reuse `./.deep-review/` artifacts where possible, but there's no cross-repo or cross-team memory in the OSS version.
- **Best on backends with real workflows.** The recipe is action-centric. Pure static-site or design-system repos won't surface much. The deeper the runtime topology (queues, caches, transactions, retries, multi-tenant boundaries), the better the review.
- **Findings are opinionated, not exhaustive.** We'd rather ship 3 ironclad findings than 30 maybe-bugs. Other tools optimize the other way.

## Maintaining

The source of truth for the recipe is Farfield's internal `find_bugs` pipeline. See [`MAINTAINING.md`](./MAINTAINING.md) for how the public version stays in sync.

## License

MIT — see [`LICENSE`](./LICENSE).

---

Built by [Farfield](https://farfield.dev). If you find bugs with this and want them automatically filed, fixed, and shipped → [come talk to us](https://farfield.dev).
