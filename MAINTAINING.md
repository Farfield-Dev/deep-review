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

This repo IS the skill directory. Cloning into `~/.claude/skills/deep-review/` makes the entire repo content (including `SKILL.md` at the root) the skill that Claude Code auto-discovers.

```
deep-review/                      (repo root = skill directory)
├── SKILL.md                      Entry point, orchestrator, sub-agent prompts
├── architecture-map.md           Phase 1 detail (loaded on demand)
├── team-intent.md                Phase 2 detail (loaded on demand)
├── action-trace.md               Phase 3 detail (loaded on demand)
├── product-scan.md               Phase 4 detail (loaded on demand)
├── adversarial-validate.md       Phase 5 detail (loaded on demand)
├── README.md                     Repo-facing — not loaded by Claude Code
├── LICENSE
├── MAINTAINING.md                This file
├── .gitignore
└── examples/
    └── findings-*.md             Example outputs from running on real OSS repos
```

`SKILL.md` is the only file with YAML frontmatter. The phase files are referenced from `SKILL.md` via standard markdown links, so Claude Code loads them on demand when each phase starts.

## Sync workflow

For now this is **manual**. When Farfield's internal recipe meaningfully changes:

1. Re-extract the relevant step file from the internal recipe (`apps/api/src/features/autopilot/recipe_files/find_bugs/steps/`)
2. Apply the standard cuts (see list above)
3. Open a PR here with the diff
4. Tag it with the date the upstream change landed

A future automation will open these PRs automatically when `recipe_files/find_bugs/` changes on Farfield's default branch, but that's not built yet.

## Methodology files that own the IP

The portable IP lives in five places. If you're updating the recipe, these are the load-bearing files:

| File | What lives here |
|---|---|
| `architecture-map.md` | Phase 0 signals, action inventory, workflow ledger, impact taxonomy, severity calibration |
| `team-intent.md` | Bug-class mix taxonomy, anti-circularity (no SHAs / files in the brief), confidence cascade |
| `action-trace.md` | The 7 trace questions, parallel sub-agent shape, cross-action synthesis |
| `product-scan.md` | UX/product-level bug heuristics |
| `adversarial-validate.md` | 100%-confidence rubric, scenario-specific mitigation check, disprove-vs-preserve balance |

`SKILL.md` is the orchestrator and is structurally separate — it does not own methodology, only flow control and sub-agent prompt templates. Methodology changes should land in the phase files; orchestration changes (model choice per phase, modes, output schema) land in `SKILL.md`.

## What we deliberately do not port

The Farfield-internal recipe has a `01_context_bootstrap.md` step that maintains a multi-run cache of company-context and per-repo structural maps, synced through the pod artifact endpoint. The OSS review is single-run cold by design, so we collapse that step's useful methodology (clone discovery, company-context inference) into `architecture-map.md` as a prelude and drop the cache machinery.

If you find yourself porting any of the following from the internal recipe, **stop and reconsider** — they're scaffolding, not methodology:

- Anything that calls `localhost:${POD_PORT}` or `/api/farfield/*`
- Anything that reads or writes `repos.json`
- Anything keyed by `repository_public_id`
- Anything that emits the `<!-- farfield-meta -->` HTML metadata block

## Reporting issues

Bug or false-positive examples → open an issue here with the `findings.md` output and the repo it came from (or a minimal reproduction). We use these to calibrate the upstream prompts.

For Farfield (the paid product) feedback, the right channel is the in-product feedback or hi@farfield.dev — not this repo.
