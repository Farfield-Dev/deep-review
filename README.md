# deep-review

A Claude Code plugin that does a senior-engineer-grade code review of a repo. Five phases, parallel sub-agents, adversarial validation. Writes `findings.md`.

## Install

Inside Claude Code:

```
/plugin marketplace add Farfield-Dev/deep-review
/plugin install deep-review@deep-review
/reload-plugins
```

(`/reload-plugins` registers the freshly installed skill in the current session. Skip it and `/deep-review:run` errors with "Unknown command".)

Auto-updates from this repo's `main`.

## Run

In the repo you want reviewed:

```
/deep-review:run
```

`--fast` skips the team-intent and product-scan phases (cheaper, faster, narrower).

Output lands in `./findings.md` with intermediate artifacts under `./.deep-review/` (gitignore it).

## What you get

Each finding has a severity, an impact category (REVENUE_LEAK, SUPPORT_BURDEN, DATA_LOSS, SECURITY_BREACH, COMPLIANCE, PROD_INCIDENT, BRAND_DAMAGE), a file + line, a one-paragraph root cause, a verification artifact, and a fix direction. Findings that don't map to one of the impact categories get dropped, and every finding passes a 100%-confidence adversarial check before it lands in the report. Expect 0–10 findings on a medium backend, not 50.

## Cost

On your own Anthropic key, $5–$25 per cold review on a 10–50k LOC backend. Sonnet on the cheap phases (architecture map, team intent), Opus on the heavy ones (action traces, validation). `--fast` is about a third of that.

## Limitations

- Cold-start. No memory between runs.
- Best on backends with real workflows. Static sites and pure UI repos don't surface much.
- Opinionated, not exhaustive. Prefers 3 ironclad findings to 30 maybe-bugs.

## License

MIT. See [`LICENSE`](./LICENSE).

By [Farfield](https://farfield.dev).
