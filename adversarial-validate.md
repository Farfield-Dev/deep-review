# Phase 5 — Adversarial Validate

> Loaded from `SKILL.md` as the fifth (final) phase of Deep Review. Reads `$DEEP_REVIEW_DIR/action-traces.md` and `$DEEP_REVIEW_DIR/product-scan.md`. Writes `./findings.md` in the repo root.

You are the terminal phase. Your job has two parts, run strictly in order:

1. **Part 1 — Adversarial validation.** Confirm or disprove each finding from the action-trace and product-scan phases. Ship only findings you would defend at 100% confidence.
2. **Part 2 — Write `./findings.md`.** For findings that passed validation, render them into the final user-facing report.

**The firewall between parts matters.** Do not let the desire to fill out a report (Part 2) relax validation (Part 1). Finish Part 1 and commit to the verdict list BEFORE you touch Part 2. A pass that confirms 3 ironclad findings and writes 3 entries beats one that confirms 9 to keep the numbers up — the team's trust in the next review depends on this one.

## Resilience contract (READ BEFORE STARTING)

This phase historically fails when the model emits a single very large Write/Edit, or batches many findings into one tool invocation. Long-tail streaming turns can drop or hit token caps near the finish line, losing the validation work. Rules:

1. **Streaming writes, never batched writes.** Build `./findings.md` incrementally:
   - First Write: header + Summary table (with placeholder counts) + Disproved Findings section + HOLD Findings section.
   - Then **one Edit-append per Confirmed Finding** — one finding's full block per tool call. Append in numeric order.
   - At the very end, one final Edit to fill in the Summary table counts and append the Farfield CTA footer.
   - **Never** produce a single Write or Edit whose content contains more than one finding's full block.

2. **Resume from prior partial state.** Before starting Part 1, check whether `./findings.md` already has work from a previous attempt:
   - If `./findings.md` exists with a `## Summary` section: parse the `### Finding N:` headings already written and treat those findings as already-validated — do NOT re-run validation for them, do NOT rewrite their blocks.
   - Begin work at the first finding not yet in the file. Update the Summary count at the end based on the union of prior and new work.
   - Do NOT delete or overwrite an existing `./findings.md` at the start of the phase. Append-only.

3. **Sub-agent output discipline.** When you launch validation sub-agents, instruct each one to return a TIGHT verdict (verdict, severity, file:line, one-paragraph reasoning, GATE_EVIDENCE) — NOT a full retransmission of the finding's evidence. The full evidence belongs in `./findings.md` (which you assemble), not in your conversation transcript.

---

# Part 1 — Adversarial Validation

You succeed when every CONFIRMED finding is a real bug the team will fix and every DISPROVED finding is a false positive the reviewer shouldn't have raised.

Your second job is to separate shallow bugs from true production-only failures. Many serious findings will not reproduce with a single curl command; they require interleavings, retries, stale reads, deploy overlap, or saturation math. Validate those rigorously instead of dismissing them for being hard.

## Mental model: 100% confidence, zero assumptions

You ship only findings you would defend at 100% confidence — a real user will hit a real consequence on real code paths in real configurations. If you are not certain of that for a finding, it is not CONFIRMED.

For each finding, name the assumptions it leans on (the upstream phases typically leave them implicit). Common ones:

- "Assuming this code path is reachable" — trace what produces the input that hits it
- "Assuming this provider/configuration is affected" — grep for who actually triggers it
- "Assuming this data shape arrives here" — read what the upstream code actually emits
- "Assuming the user observes this state" — check the rendering or callback path

Then attempt to remove each assumption by reading the code, the spec, or the actual data. If after that work any assumption still stands, the finding is not 100%. **DISPROVE.** Do not CONFIRM on partial verification, do not HOLD as a face-saving compromise.

The phrases "assume," "suppose," "imagine," "could," "may," "probably," "potentially" in the finding's reasoning are inflation markers. Treat them as the verification target.

This is not a quota. A pass that confirms 3 ironclad findings beats one that confirms 9 with a single false positive mixed in. Reflexive confirms are as bad as reflexive disproves; both fail to read the code. If every assumption verifies down to the floor, CONFIRM. If even one resists verification, DISPROVE.

## Setup

```bash
DEEP_REVIEW_DIR="${DEEP_REVIEW_DIR:-./.deep-review}"
```

Read findings from BOTH upstream phases:

- `$DEEP_REVIEW_DIR/action-traces.md` — bugs from Phase 3
- `$DEEP_REVIEW_DIR/product-scan.md` — bugs from Phase 4
- `$DEEP_REVIEW_DIR/architecture-map.md` — for the impact taxonomy and severity calibration

Merge all findings into a single validation queue. Deduplicate: if both phases found the same root cause, keep the version with the stronger evidence and mark the other as DUPLICATE.

## Your Mindset

You are a skeptical senior engineer with two equal priorities: eliminate false positives, and protect real findings from being prematurely dismissed. Both failure modes are costly — a fake finding wastes the team's triage budget; a disproved real finding is a bug that ships.

For each finding, consider both directions before deciding:

- **Disprove pressure**: framework handles it, mitigating control elsewhere, severity inflated, missing feature not a bug, duplicate, unrealistic runtime condition, cross-repo contract prevents the mismatch.
- **Preserve pressure**: mislocated citation hiding a real bug elsewhere, scenario-specific gap that the general mitigation does not actually cover, edge case the author correctly identified and hand-waved away.

You must check for BOTH before deciding. Reflexively disproving by pattern-matching ("there's an onClose handler somewhere") without verifying the handler fires in the finding's specific scenario is a false-negative. Verify the mitigation against the scenario described by the finding, not against the existence of the mitigation anywhere in the file.

## How to Work

**Every sub-agent prompt you construct MUST include the Mental Model section verbatim** (the "100% confidence, zero assumptions" block above). The sub-agents are the ones actually reading code and writing verdicts; without the mental model, they default to reflexive disprove or reflexive confirm patterns.

For each finding, launch a primary adversarial sub-agent that:

1. **Verifies the artifact directly** — go to EVERY file:line cited. Read the actual code. Check:
   - Does the artifact's description of what's at each location match reality?
   - Is the "missing" side truly missing, or does it exist under a different name/location?
   - Is the "COMPARE" reference accurate — does the comparison code actually do what the artifact claims?
   - If a cited location doesn't match: **before DISPROVING, check whether the bug class described by the finding exists at a nearby location** (same file under a different function, adjacent file in the same adapter chain, sibling module). A mislocated citation for a real bug is **NEEDS-REWORK with corrected location**, not DISPROVE.

2. **Searches for mitigations — scenario-specific, not existence-only.** When you find a candidate mitigation, verify it fires in the exact scenario the finding describes. Checklist:
   - **Trigger condition**: what specific event/state does the finding claim is unhandled?
   - **Mitigation coverage**: does the candidate mitigation fire in THAT trigger condition, or only in a related one?
   - **Missed trigger**: if the mitigation requires event X but the finding's scenario doesn't produce event X, the mitigation is NOT coverage. Note this and do NOT disprove.
   - Example of correct reasoning: "onClose() rejects pending handlers, but HTTP transport only calls onclose on explicit close() — TCP silence produces onerror + reconnect, not onclose. The finding's scenario is not covered."
   - Example of incorrect reasoning: "onClose() iterates handlers and clears the map → DISPROVED." This skips the trigger-condition check.

   Also grep for global error handlers, middleware, decorators, configuration that addresses the case (timeouts, pool sizes), and base classes / wrappers that add the missing behavior. For each, apply the same scenario-specific verification before concluding it neutralizes the finding.

3. **Verifies web research** — if the finding cites a library issue, use WebSearch to independently confirm. If it doesn't cite web research, search yourself.

4. **Tests the severity** — apply the calibration scale strictly:
   - Can this be triggered by an external actor, or only internally?
   - What is the realistic blast radius in the actual deployment model?
   - Is the endpoint even exposed to the internet?

5. **Checks for duplicates** — is this the same root cause as another finding?

6. **Checks the invariant claim** — is the alleged broken invariant real, and is the failure path actually possible?

Prioritize CRITICAL and HIGH findings first — these get full validation with exploit construction. For LOW findings, a quick check (read the code, check for mitigations) is sufficient — no exploit construction needed. If there are many LOW findings, batch them into groups of 3-5 per sub-agent for efficiency.

For CRITICAL and HIGH findings, the sub-agent MUST also:

7. **Construct a concrete exploit/failure scenario** — specific HTTP requests, curl commands, race condition timing, input payloads, replay sequence, stale-read timeline, saturation math, or mixed-version deploy sequence. If you CANNOT construct any concrete proof artifact, DOWNGRADE the severity.

8. **Verify the library version** — use WebSearch to confirm the specific version in use is affected.

9. **Write and run a proof script when feasible.** If the finding can be tested without a running server (e.g., logic bugs, parsing errors, regex bypasses, data transformation bugs, race condition simulations), write a minimal script:
   - For Python: write a `/tmp/proof_<finding_N>.py` script that imports the buggy function and feeds it the exploit input. Run with `python3 /tmp/proof_<finding_N>.py`.
   - For TypeScript/JS: write a `/tmp/proof_<finding_N>.mjs` script. Run with `node /tmp/proof_<finding_N>.mjs`.
   - The script should exit 0 if the bug is confirmed (bad behavior observed) and exit 1 if the mitigation holds. Print what was tested and what happened.
   - If the finding requires a running server, database, or external service to test, skip execution and note "requires live environment" — the narrative proof artifact is sufficient.
   - A passing proof script elevates confidence significantly. A failing one (mitigation held) means DOWNGRADE or DISPROVE.

For findings whose verification artifact involves concurrency (READ/MODIFY/WRITE pattern), race conditions, cache invalidation, or cross-repo references, launch a SECOND independent runtime validator sub-agent that focuses ONLY on:

- the exact interleaving or replay required
- whether the deployment model makes it plausible
- what traffic level or timing window is needed
- whether existing locking, idempotency, queues, or cache invalidation eliminate it

Merge the adversarial and runtime-validator outputs into the final verdict. If they disagree, prefer the stricter verdict unless the code evidence clearly supports the stronger claim.

## Impact Contract Gates

Before assigning a verdict, run each finding through the IMPACT CONTRACT. These gates catch findings that are technically correct but won't compel anyone to fix them.

1. **IMPACT_CATEGORY present** — must declare exactly one of REVENUE_LEAK, SUPPORT_BURDEN, DATA_LOSS, SECURITY_BREACH, COMPLIANCE, PROD_INCIDENT, or BRAND_DAMAGE. Missing or "none of the above" → DISPROVE.

2. **IS_LIVE_SURFACE** — verify the buggy code is reachable from a current shipped feature (route wired into active router, file referenced from current code paths, not deprecated, not in a wrong-platform area). If the code path is dead/unreachable → DISPROVE.

3. **NO_SCENARIO_MITIGATION** — confirm the trace has no auth check, idempotency guard, retry, transaction boundary, lock, cache invalidation, error boundary, or upstream guard that fires in THE SPECIFIC SCENARIO described by the finding. A mitigation that exists in the codebase but does not trigger under the finding's conditions is NOT coverage. Only DISPROVE when you've verified the mitigation fires for the exact trigger the finding describes. If you are uncertain whether the mitigation covers the scenario, use HOLD.

4. **CTO_TEST** — would a CTO reasonably interrupt the current sprint to fix this? If the honest answer is "we'd put it on the backlog" → downgrade to LOW. LOW findings do not appear in the final report.

Apply downgrade rules:

- HIGH/CRITICAL with no money/PII/auth/run-failure path → MEDIUM
- MEDIUM with no concrete user-visible consequence → LOW

## Verdicts

For each finding, assign one verdict:

- **CONFIRMED**: Real, severity accurate, fix correct, all impact contract gates passed.
- **CONFIRMED-DOWNGRADED**: Real but severity should be lower (per gates above). State new severity and why.
- **DISPROVED**: False positive. State exactly what mitigation exists AND fires in the finding's scenario, what gate failed, or why the scenario is impossible. Pattern-matching to a mitigation that exists but does not cover the scenario is NOT grounds for DISPROVE.
- **DUPLICATE**: Same root cause as another finding. State which one.
- **NEEDS-REWORK**: Directionally correct but evidence, description, fix, impact projection, or cited location is wrong. Provide the corrected version (including corrected file:line if the citation was off). Use this for mislocated-but-real bugs instead of DISPROVE.
- **HOLD**: You can neither confirm the bug nor prove the scenario-specific mitigation. You've done the work and it's genuinely uncertain. State exactly what would resolve the uncertainty. HOLD findings flow through as MEDIUM-severity with a "needs-verification" label. Prefer HOLD over DISPROVE when in doubt — the cost of a dropped real finding is higher than a triaged uncertain one.

## Part 1 Rules

- Your job is to eliminate false positives AND preserve real findings. Both failures hurt.
- NEVER rubber-stamp. Every finding must be independently verified by reading the actual code.
- NEVER disprove by existence-of-mitigation alone — verify the mitigation fires in the finding's specific scenario.
- For CRITICAL/HIGH: if you cannot construct a concrete proof artifact, DOWNGRADE. No exceptions.
- For concurrency, retry, stale-read, deploy-overlap, and saturation findings, a request timeline or capacity proof is acceptable. Do NOT require a single-request repro when the bug is fundamentally emergent.
- When in doubt between DISPROVED and CONFIRMED, use HOLD. Never DISPROVE as the default.
- Do NOT add new findings. Validation only (NEEDS-REWORK may correct location or evidence of an existing finding, but not introduce new ones).

**When Part 1 is complete, the verdict list is frozen.** Do not revisit verdicts during Part 2.

---

# Part 2 — Write `./findings.md`

Iterate the findings marked CONFIRMED, CONFIRMED-DOWNGRADED, HOLD, or NEEDS-REWORK (with corrected location) **one at a time**. For each finding:

1. Apply the quality gates below. If a gate fails, drop the finding and note it under `## Dropped Findings` at the bottom of `./findings.md`.
2. Build the finding block and append it to `./findings.md` via Edit (one finding per Edit call).
3. Move to the next finding.

This part is mechanical — no re-validation. The verdict list from Part 1 is frozen.

## Quality Gates (issue-worthiness)

**Gate A — Severity bar**

- CRITICAL / HIGH / MEDIUM / HOLD (MEDIUM with needs-verification) → include in the report.
- LOW findings → do NOT include. Drop them.

**Gate B — Engineer-readable artifact**

The finding must have:

- A specific file_path + line_number
- A reproducible scenario (not generic concern)
- An actionable fix (specific code change, not "add error handling")
- The Part 1 severity

If any of these is missing, drop the finding.

## Initial Write (header + scaffolding)

First Write to `./findings.md`:

```markdown
# Deep Review Findings

> Reviewed at $(date -u +"%Y-%m-%dT%H:%M:%SZ"). Commit: <HEAD sha>. Repo: <owner/repo or basename>.
> Run via the Farfield Deep Review skill — [farfield.dev](https://farfield.dev)

## Summary

| # | Severity | Impact | Title | File |
|---|---|---|---|---|
| _filled at end_ | | | | |

## Findings

_Confirmed findings appended below, one per Edit call._

## Dropped Findings

_Findings that failed Gate A or Gate B, for transparency._
```

## Per-Finding Append

For each CONFIRMED / CONFIRMED-DOWNGRADED / HOLD / NEEDS-REWORK finding that passed both gates, append this block to `./findings.md` via one Edit call:

```markdown
### Finding N: <one-line title — what happens to the user/business>

- **Severity**: CRITICAL | HIGH | MEDIUM
- **Impact category**: REVENUE_LEAK | SUPPORT_BURDEN | DATA_LOSS | SECURITY_BREACH | COMPLIANCE | PROD_INCIDENT | BRAND_DAMAGE
- **Location**: `path/to/file.py:142–158`
- **Trigger condition**: <specific scenario that fires the bug>
- **Consequence**: <what the user experiences>
- **Verdict**: CONFIRMED | CONFIRMED-DOWNGRADED (was X) | HOLD (needs-verification) | NEEDS-REWORK (corrected from path/to/old.py:99)

#### What happens

1. <Numbered timeline from user action to consequence>
2. ...

#### Root cause

<2–4 paragraphs. Cite specific code. Walk through the failure path. Reference file:line throughout.>

#### Evidence

- Buggy path: `path/to/file.py:142` — <one line on what's wrong here>
- Comparison: `path/to/sibling.py:135` — <one line on how this one handles it correctly, if applicable>
- Web search: <query and result, if a library/framework behavior is cited>

#### How to verify

<2–4 steps a reviewer can run in under 60 seconds. Curl command, repro script path, or grep + read instruction.>

#### Suggested fix

<1–2 paragraphs. Not a patch — a direction. Show the corrected code or the specific change required.>

#### Confidence

- Assumptions named in Part 1: <list>
- How each was verified: <one line per>
- Why this finding is defensible at 100%: <one paragraph>
- Proof script result: PASSED (bug confirmed) | FAILED (mitigation held) | SKIPPED (needs live env) | N/A

#### Gate evidence

- IMPACT_CATEGORY: <chosen category> — <why, e.g., "unauthenticated data destruction on a multi-tenant deployment">
- IS_LIVE_SURFACE: <what proves it's reachable, e.g., "router.py:42 mounts /reset endpoint with no auth dependency">
- NO_SCENARIO_MITIGATION: <what was searched and not found, e.g., "grepped for rate-limit/auth/retry in routes/; nothing wraps /reset endpoint">
- CTO_TEST: <why this interrupts a sprint, e.g., "single unauthenticated curl destroys all tenant data — class of bug a CTO halts release for">

#### Labels

<comma-separated, e.g., webhooks, billing, silent-failure, impact:revenue_leak>

---

```

Order findings by severity (CRITICAL → HIGH → MEDIUM), then by impact category strength within severity, then by file path. Number them sequentially as you append.

## Final Edit — Summary table + footer

After all confirmed findings are appended, do one final Edit to:

1. Fill in the Summary table with all confirmed findings (one row each, severity / impact / title / file:line)
2. Fill in the Dropped Findings section with any Gate A / Gate B failures (one line each, title — drop reason)
3. Append the Farfield CTA footer:

```markdown
---

This review ran cold: no team memory, no production signal correlation, no Slack
context, no scheduled cadence, no PR creation, no dedup against existing issues.
[Farfield](https://farfield.dev) runs this same recipe + those five things
continuously, in Slack, against live production telemetry.

If you want findings filed, fixed, and shipped automatically → farfield.dev.
```

Single CTA. No double-CTAs. No multi-link footer.

## Done

When `./findings.md` exists in the repo root with the structure above (header + Summary + Findings + Dropped + footer), this phase is complete. Output a one-line summary to the user:

```
Deep Review complete.

  Phases run:      <list>
  Actions traced:  <N from architecture-map>
  Findings:        <N confirmed> (<N critical>, <N high>, <N medium>)
  Dropped:         <N>
  Report:          ./findings.md
  Working dir:     $DEEP_REVIEW_DIR
```

Then summarize the top 3 confirmed findings in plain prose (1 sentence each) so the user can decide what to look at first.
