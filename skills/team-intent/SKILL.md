---
name: team-intent
description: Step 2 of Deep Scan. Produces a class-level brief from the last 2 months of commit history — bug-class mix, team focus areas, mode, confidence, and trust-critical surface categories. Steers the deep scan's attention budget. Stays at the class level on purpose (no SHAs, no file paths) to avoid making the scan rediscover what the team already fixed.
allowed-tools: [Read, Write, Edit, Grep, Bash]
---

# Team Intent

You are analyzing the **team's intent** — what classes of bugs they keep fixing, what areas they care about, where they've been burned. The output is a compact *priority signal* the deep scan uses to decide how to allocate its attention budget.

**Critical distinction.** Internally you will read commit history, diffs, file churn, and SHAs to ground your analysis. But the **brief you write for the scan must stay at the class level**: bug categories, surface categories, mode, confidence. **Do not emit SHAs, specific file paths, or pre-fix grep patterns in the brief.** They turn the scan into a re-discovery of what the team already fixed (a circular exercise that surfaces already-known bugs). The deep scan has its own action inventory — it does not need you to point it at files.

You are analyzing the last **2 months** (recent signal only). Do NOT install dependencies or run tests.

## Setup

```bash
DEEP_SCAN_DIR="${DEEP_SCAN_DIR:-./.deep-scan}"
mkdir -p "$DEEP_SCAN_DIR"
```

Read these first if present:

- `$DEEP_SCAN_DIR/architecture-map.md` — action inventory, trust-critical surfaces (from the previous step).

## Repo Discovery

Same pattern as architecture-map:

```bash
REPO_ROOTS=$(find . -maxdepth 3 -name .git -type d 2>/dev/null | sed 's#/.git$##' | sort -u)
if [ -z "$REPO_ROOTS" ]; then REPO_ROOTS="."; fi
```

For each repo root, run the analysis below independently, then synthesize one brief across all of them.

## What to Gather (your internal analysis — stays in your head)

Read the **entire commit history in the 2-month window**, not just commits tagged `fix:`. Teams often ship breakage-fixing work under other prefixes — a refactor that unbreaks a state machine, a feature commit whose PR body mentions a regression, a "chore" that patches a silent contract drift. Filtering to `fix:` only hides half the pain signal.

Signal sources, in rough priority order:

1. **Reverts and walkbacks** — any commit undoing prior work. Strongest pain signal regardless of prefix.
2. **Explicit fix commits** — `fix:` prefix or "fix"/"fixes"/"closes #N" in title/body. Usually the cleanest cluster.
3. **Refactors / rewrites in areas with past fix churn** — a `refactor:` commit in a file that previously had fix commits often *is* the fix, lifted to a larger rewrite. Count these.
4. **Feature commits whose body references a bug, regression, incident, or user report** — read PR bodies, not just titles. A `feat:` that also mentions "this also fixes the crash when X" is a pain signal.
5. **Churn bursts** — single area or single file receiving many commits in a short window (e.g., ≥5 commits to one file in 2 weeks), regardless of prefix. Burn patterns indicate the team is working through something hard.
6. **Repeat-burn files** — files appearing in 2+ commits classified as signal (fixes + refactors + reverts, not just `fix:`).
7. **Open bug issues** — from the issue tracker, ideally `bug`-labeled (if available via `gh` CLI or the repo's tracker).
8. **Recently merged PRs (~30 days)** — for bodies and linked-issue context.
9. **New and deleted files** — signals active build-out or decommissioning.

```bash
# Reverts in the window:
git log --since="2 months ago" --grep="^Revert" --oneline

# Fix commits and bug-referencing bodies:
git log --since="2 months ago" --pretty="%H %s%n%b" --grep="fix\|fixes #\|regression\|incident\|hotfix"

# Repeat-burn files: files appearing in many commits in the window
git log --since="2 months ago" --name-only --pretty=format: \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -30
```

Aim for 50–100 signal commits to cluster cleanly. If the repo only produces 20 signal commits, that's fine — confidence will land lower.

For the **10–15 largest signal commits** (biggest diffs, regardless of prefix), read the full PR body AND the diff. Titles and prefixes lie; PR bodies and diffs don't. Cluster by *what class of problem was being addressed*, not by feature area or commit prefix.

## What to Synthesize (the output brief)

Five class-level artifacts. Every artifact stays abstract — no file paths, no SHAs, no pre-fix patterns in the brief text.

### 1. Bug-class mix

Approximate percentage of recent **signal commits** (fixes + reverts + fix-adjacent refactors + bug-referencing features) in each of these buckets. Sum to 100.

- `auth-tenancy` — authn/authz, session, multi-tenant boundary
- `concurrency-data-integrity` — races, transactions, ordering, lost updates
- `ux-flow` — user-facing flow, wrong state shown, navigation dead-ends
- `api-contract` — breaking changes, version skew, DTO drift, SDK mismatch
- `performance-reliability` — latency, timeouts, retries, memory, resource exhaustion
- `deploy-infra` — config, CI/CD, build, env-var, deploy overlap
- `other`

Derive from reading diffs, not commit titles. If a bucket is 0%, omit it. Do not cite SHAs in the output.

### 2. Team focus (3–6 short bullets)

Each bullet names a **class of concern** in plain prose — not a specific bug, not a specific file. One sentence.

- Good: "Tool approval state machines are actively evolving; small state-machine bugs land repeatedly."
- Good: "Provider streaming event schemas drift; missing-case handling is a recurring class."
- Bad: "Look at `stream-text.ts:1374` for `providerExecuted` filter bugs."
- Bad: "Hunt the file `packages/ai/src/...` for pre-fix shape X."

If a focus can only be stated as a file/SHA/regex, it's too specific — lift it to the class above or drop it.

### 3. Mode

One of:

- `firefighting` — high revert count, fix-on-fix, visible pain across signal commits
- `polish` — low-severity signal commits dominate, steady cadence, no reverts
- `feature-build` — new-file / `feat:` commits dominate over signal commits
- `mixed` — none dominates

Cite evidence in plain prose (e.g., "three reverts in the window, one rolling back a security fix") — not by SHA.

### 4. Confidence

`high` | `medium` | `low`. Floor: fewer than 30 signal commits OR contradictory signals → `low`. `low` is a binding signal to the scan to treat the profile as weak and stay broad.

### 5. Trust-critical surface categories (class-level coverage anchors)

List surface **categories** the scan must not skip, drawn from `architecture-map.md` and the team's activity. Each entry is a category label + one-sentence rationale — no file paths.

- Good: "Multi-tenant boundary crossings (any action that reads data keyed by a different tenant's ID)."
- Good: "Primary money-movement paths (payment capture, refund, subscription state changes)."
- Good: "Tool approval state machines (any step that transitions tool-call → approval → execution)."
- Bad: "`packages/foo/bar.ts` because fixed 3 times."

Size the list by the evidence, not a quota. A simple repo might yield 2 categories; a complex one might yield 8.

## Rules

1. **No file paths, no SHAs, no pre-fix grep patterns in the brief text.** Internal analysis uses them; the brief strips them out. This is the anti-circularity rule.
2. **Class-level language only.** If you can't say it without naming a file or commit, it's too specific.
3. **Low confidence → broad coverage.** If `low`, the brief says so and tells the scan not to specialize.
4. **Tight.** Under 200 lines. Dense, no padding.

## Output

Write to `$DEEP_SCAN_DIR/team-intent.md`:

```markdown
# Team Intent Brief

## Summary
One paragraph: dominant bug class, mode, confidence. Plain prose only.

## Bug-class mix (last 2 months)
| Class | % |
|---|---:|
| api-contract | 40 |
| ux-flow | 20 |
| ... | ... |

## Team focus
- <one sentence, class-level>
- <one sentence, class-level>
- ...

## Mode
<firefighting | polish | feature-build | mixed> — one-sentence justification in prose.

## Confidence
<high | medium | low> — one-sentence justification.

## Trust-critical surface categories
- <category label> — one-sentence rationale.
- ...
```

## Done

When `$DEEP_SCAN_DIR/team-intent.md` is written, this step is done. Output a one-line summary:

```
Team intent: <mode>, <dominant class> dominates at N%, confidence <level>.
```

The deep-scan step reads this brief as a priority signal only — it tells the scan how to allocate attention across the action inventory, not where to look. The action inventory is the attack surface; the brief is the steering.
