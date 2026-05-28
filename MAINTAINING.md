# Maintaining deep-review

The source of truth for the recipe is **Farfield's internal `find_bugs` recipe** at:

```
apps/api/src/features/autopilot/recipe_files/find_bugs/
```

This public repo is a cleaned mirror of that, with Farfield-specific scaffolding stripped:

- Backend-owned scope metadata (`repos.json`, `repository_public_id`) → removed
- Pod artifact sync endpoint (`POST /api/farfield/sync-artifacts`) → removed
- Cross-run state (`<scan-journal>`, `<production-signals>`, `existing-issues.md`) → removed (optional in v2)
- Issue-tracker integration (`issue-management` skill) → replaced with markdown `findings.md`
- Codex CLI Jinja branches → resolved to Claude Code variant for v1

## Repo shape

This repo is a Claude Code **plugin marketplace** that ships a single plugin (`deep-review`) containing a single skill (`run`).

```
deep-review/                                       (repo root = marketplace root)
├── .claude-plugin/
│   └── marketplace.json                           Marketplace catalog (lists plugins)
├── plugins/
│   └── deep-review/                               The plugin
│       ├── .claude-plugin/
│       │   └── plugin.json                        Plugin manifest
│       └── skills/
│           └── run/                               The skill — invoked as /deep-review:run
│               ├── SKILL.md                       Entry point, orchestrator
│               └── phases/                        Phase methodology, loaded on demand
│                   ├── architecture-map.md        Phase 1
│                   ├── team-intent.md             Phase 2
│                   ├── action-trace.md            Phase 3
│                   ├── product-scan.md            Phase 4
│                   └── adversarial-validate.md    Phase 5
├── README.md                                      Repo-facing (not loaded by Claude Code)
├── LICENSE
├── MAINTAINING.md                                 This file
├── .gitignore
└── examples/
    └── findings-*.md                              Example outputs from running on real OSS repos
```

`SKILL.md` is the only file with YAML frontmatter. The `phases/*.md` files are referenced from `SKILL.md` via standard markdown links, so Claude Code loads them on demand when each phase starts.

The marketplace + plugin manifests in `.claude-plugin/*.json` are what make `/plugin marketplace add Farfield-Dev/deep-review` + `/plugin install deep-review@deep-review` work as the canonical install path.

## Sync workflow

For now this is **manual**. When Farfield's internal recipe meaningfully changes:

1. Re-extract the relevant step file from the internal recipe (`apps/api/src/features/autopilot/recipe_files/find_bugs/steps/`)
2. Apply the standard cuts (see list above)
3. Open a PR here with the diff
4. Tag it with the date the upstream change landed
5. Bump `version` in `plugins/deep-review/.claude-plugin/plugin.json` so installed users get the update on their next `/plugin marketplace update`

A future automation will open these PRs automatically when `recipe_files/find_bugs/` changes on Farfield's default branch, but that's not built yet.

## Methodology files that own the IP

The portable IP lives in five places. If you're updating the recipe, these are the load-bearing files:

| File | What lives here |
|---|---|
| `plugins/deep-review/skills/run/phases/architecture-map.md` | Phase 0 signals, action inventory, workflow ledger, impact taxonomy, severity calibration |
| `plugins/deep-review/skills/run/phases/team-intent.md` | Bug-class mix taxonomy, anti-circularity (no SHAs / files in the brief), confidence cascade |
| `plugins/deep-review/skills/run/phases/action-trace.md` | The 7 trace questions, parallel sub-agent shape, cross-action synthesis |
| `plugins/deep-review/skills/run/phases/product-scan.md` | UX/product-level bug heuristics |
| `plugins/deep-review/skills/run/phases/adversarial-validate.md` | 100%-confidence rubric, scenario-specific mitigation check, disprove-vs-preserve balance |

`SKILL.md` is the orchestrator and is structurally separate — it does not own methodology, only flow control and sub-agent prompt templates. Methodology changes should land in the phase files; orchestration changes (model choice per phase, modes, output schema) land in `SKILL.md`.

## What we deliberately do not port

The Farfield-internal recipe has a `01_context_bootstrap.md` step that maintains a multi-run cache of company-context and per-repo structural maps, synced through the pod artifact endpoint. The OSS review is single-run cold by design, so we collapse that step's useful methodology (clone discovery, company-context inference) into `phases/architecture-map.md` as a prelude and drop the cache machinery.

If you find yourself porting any of the following from the internal recipe, **stop and reconsider** — they're scaffolding, not methodology:

- Anything that calls `localhost:${POD_PORT}` or `/api/farfield/*`
- Anything that reads or writes `repos.json`
- Anything keyed by `repository_public_id`
- Anything that emits the `<!-- farfield-meta -->` HTML metadata block

## Submitting to the community marketplace

To make this installable in one command (`/plugin install deep-review@claude-community`) without users needing to add the marketplace first:

1. Run `claude plugin validate` locally to confirm the manifests are clean
2. Submit at [platform.claude.com/plugins/submit](https://platform.claude.com/plugins/submit)
3. Automated review (24–48h)
4. After approval, the community catalog auto-pins to a commit SHA and CI bumps the pin as we push new commits

This is independent of our own marketplace — both paths can coexist.

## Reporting issues

Bug or false-positive examples → open an issue here with the `findings.md` output and the repo it came from (or a minimal reproduction). We use these to calibrate the upstream prompts.

For Farfield (the paid product) feedback, the right channel is the in-product feedback or hi@farfield.dev — not this repo.
