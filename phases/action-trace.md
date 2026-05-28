# Phase 3 — Action Trace

> Loaded from `SKILL.md` as the third phase of Deep Review. Reads `$DEEP_REVIEW_DIR/architecture-map.md` and `$DEEP_REVIEW_DIR/team-intent.md`. Writes `$DEEP_REVIEW_DIR/action-traces.md`. This is the primary bug-finding phase.

You are performing a deep bug review by tracing every user action end-to-end through the system. You do not review code file-by-file or by concern category. You follow what happens when a user does something, and you find what breaks along the way.

This approach finds bugs that module-based reviews miss: ordering bugs, side-effect gaps, missing invalidation, transaction boundary errors, and cross-module failures. These are the bugs that matter most — they affect real user actions.

## Mental model: 100% confidence, zero assumptions

A finding is something you would defend at 100% confidence — a real user will hit a real consequence on real code paths in real configurations. If you are not certain of that, you have a hypothesis, not a finding.

When a finding feels "probably right" but not certain, name the assumption you are leaning on. Common ones:

- "I'm assuming the model returns X" — find the spec or a test that produces X
- "I'm assuming this branch is reachable" — trace what produces the input that hits it
- "I'm assuming this provider is affected" — grep for who actually triggers this code path
- "I'm assuming this data flows here" — read what's actually present

Then attempt to remove the assumption by reading the code, the spec, or the actual data. If after that work the assumption still stands, your confidence is not 100%. **Drop the finding.** This is the correct outcome — your job is to ship certainty, not volume. Three findings the team will fix beats twelve they have to re-verify. A trace ending with zero findings because every candidate had an unverified assumption is success, not failure.

The phrases "assume," "suppose," "imagine," "could," "may," "probably," "potentially" are inflation markers. If they appear in your finding's reasoning, treat them as a flag to verify or drop.

## Setup

```bash
DEEP_REVIEW_DIR="${DEEP_REVIEW_DIR:-./.deep-review}"
```

Read the full feature map, context profile, severity calibration, critical workflow ledger, and **action inventory** from `$DEEP_REVIEW_DIR/architecture-map.md`.

Also read `$DEEP_REVIEW_DIR/team-intent.md` if it exists — the team-intent brief from the previous phase. It is a **priority signal only**: class-level bug mix, team focus areas, mode, confidence, and trust-critical surface categories. It deliberately contains **no file paths, no SHAs, and no pre-fix grep patterns** — the brief's job is to tell you which *kinds* of problems to spend extra attention on, not to point you at specific files. Your action inventory tells you what to trace; the brief tells you how to allocate attention across it.

If the brief is missing or empty (e.g., team_intent phase was skipped or failed), treat as low-confidence: do not specialize, allocate attention broadly across the action inventory, and still cover any trust-critical surfaces named in `architecture-map.md`.

Treat the brief as follows:

1. **Bug-class mix + team focus.** When two candidate actions are equally risky from the inventory, prefer the one that falls in a class the team is actively fixing. If `api-contract` bugs are 40% of recent fixes, bias your attention budget toward contract-like surfaces (DTO drift, provider event handling, version skew). This is allocation guidance, not a filter — do not skip classes that are low on the mix.

2. **Trust-critical surface categories.** Every action in the inventory that matches one of these categories must be traced at least once this run, regardless of how the rest of your attention is spent. These are the anti-tunnel-vision anchors — they exist specifically so a low-confidence brief cannot cause you to miss load-bearing surfaces. Before finishing, confirm each category was covered by at least one trace.

3. **Confidence cascade.** If the brief declares confidence `low`, treat the mix and focus as weak signal only. Do not specialize. Allocate attention broadly across the action inventory. Still cover the trust-critical categories.

4. **Mode.** `firefighting` → expect fix-on-fix fragility, trace harder for state-machine bugs. `polish` → expect subtle correctness issues, not crashes. `feature-build` → expect new-code immaturity around error paths. `mixed` → no prior.

**What the brief is NOT:** a list of bugs to find, a map of files to grep, a set of patterns to match. If you catch yourself reading the brief as "go look at file X for pattern Y," stop — the brief never names files or patterns. Re-read it as class-level direction and let your own action-tracing discover the concrete bugs.

## How to Work

### Wave 1: Action Traces (PRIMARY — this is where the important bugs are found)

For EACH action in the Action Inventory, launch a dedicated sub-agent via the `Task` tool. Give each sub-agent:

1. The action name, entry point, handler, service method, and known side effects from Phase 1
2. The severity calibration (summarized in 4 lines — copy from architecture-map.md's Impact Taxonomy and Severity Calibration sections)
3. The 7 trace questions (below)
4. The Mental Model section verbatim (the "100% confidence, zero assumptions" block above)
5. The output format (below)

Launch ALL action trace sub-agents IN PARALLEL — issue all `Task` calls in a single response.

If there are more than 30 actions, batch the low-priority ones (3-5 per sub-agent) with abbreviated traces (entry point + side effects only, no deep trace). High and medium priority actions always get their own sub-agent with full traces.

### Wave 2: Cross-Action Synthesis

After Wave 1 returns, launch synthesis sub-agents in parallel:

1. **Variant search** — for every bug found, check: does the SAME action performed via a DIFFERENT entry point (e.g., `create_session` vs `start_session` vs `continue_session`) have the same bug? Does a SIMILAR action (e.g., cancel for one provider vs cancel for another) have the same bug?
2. **Shared-code audit** — read middleware, decorators, base classes, and utilities that multiple actions pass through. Trace them with the same 7 questions.
3. **Cross-repo integration check** — if the action inventory spans multiple repos, trace the action across the boundary (frontend → backend, mobile → API, service → service).

Stop after Wave 2 unless it produced new HIGH+ findings — in that case, run one more variant-search wave on the new findings.

## Sub-agent Instructions (include this in each action trace sub-agent prompt)

```
You are tracing one user action end-to-end through the system. Your job is to follow
the entire execution path from entry point to completion and find every place where
something can go wrong.

## Mental model: 100% confidence, zero assumptions

A finding is something you would defend at 100% confidence — a real user will hit a real
consequence on real code paths in real configurations. If you are not certain of that,
you have a hypothesis, not a finding.

When a finding feels "probably right" but not certain, name the assumption you are leaning
on. Common ones:
- "I'm assuming the model returns X" — find the spec or a test that produces X
- "I'm assuming this branch is reachable" — trace what produces the input that hits it
- "I'm assuming this provider is affected" — grep for who actually triggers this code path
- "I'm assuming this data flows here" — read what's actually present

Then attempt to remove the assumption by reading the code, the spec, or the actual data.
If after that work the assumption still stands, your confidence is not 100%. **Drop the
finding.** Three findings the team will fix beats twelve they have to re-verify.

The phrases "assume," "suppose," "imagine," "could," "may," "probably," "potentially"
are inflation markers. If they appear in your finding's reasoning, treat them as a flag
to verify or drop.
```

## How to Trace

Start at the entry point (the route handler, webhook handler, cron function, or queue consumer). Follow every function call, every DB query, every external API call, every side effect. Read each function fully — do not skim. Continue until there is nothing left to trace.

At EVERY step in the trace, ask these 7 questions:

1. **What function is called?** Read it fully. Not just the signature — the body.
2. **What data is read? From where?** Could it be stale? Is it from cache, DB, or a replica?
3. **What data is written? To where?** DB table, cache, queue, external API, file system?
4. **What else in the system depends on the data that was just written?** Find those dependencies. Do they see the new data or old data? Is there a cache, replica lag, or missing invalidation?
5. **What side effects fire?** Events emitted, emails sent, webhooks called, background tasks launched. And crucially: what side effects fire for SIMILAR actions but NOT this one? (Inconsistencies between similar actions are high-signal bugs.)
6. **What happens if this step fails halfway?** Is there a transaction boundary? Does partial state get committed? Is the error propagated or swallowed? Can the user retry safely?
7. **What happens if this action runs concurrently with itself?** Two users clicking the same button at once. A retry hitting while the first request is still processing. Is there a lock, a unique constraint, or an optimistic concurrency check?

## Rules

1. **No feature requests** — "No rate limiting" is a feature gap, not a bug. Only report things that are BROKEN or produce WRONG STATE.

2. **Web search before flagging library issues** — If your finding involves a framework behavior, verify with WebSearch. Include what you searched and found.

3. **Apply the severity calibration** — Read it. Use it. A finding is only CRITICAL if it matches the CRITICAL definition.

4. **Check for mitigations** — Before flagging anything, trace the full execution path for mitigations. Check middleware, decorators, base classes, callers, and configuration that could handle the issue. Search for how similar patterns are handled elsewhere in the codebase — if other callsites handle the same edge case correctly, the one that doesn't is a real finding; if none do, it might be by design.

4a. **Safety-net check before claiming severity.** Before writing a finding at any severity, you must explicitly enumerate and check for the common safety nets that would neutralize the bug:

- **Stream/transform finalization** — is there a `flush` handler, terminal control chunk, or close event that the buggy path actually reaches?
- **Error propagation** — is the error caught upstream? Does it get converted into a typed result that the caller can observe?
- **Recovery paths** — is there a `stop()`, `cancel()`, abort signal, regenerate, or similar escape hatch that a real user would hit?
- **Schema/type validation** — would downstream validation (`safeValidateTypes`, `safeParseJSON`, provider-side schema check) catch the corrupted value before it does damage?
- **Default configuration** — does the bug require non-default settings, rare inputs, or a specific runtime? If so, the severity is bounded by how often real users hit those conditions.
- **Test coverage** — does the repo already have a test that asserts the buggy behavior as expected? If yes, your finding is redefining intent, not finding a bug — treat as OBSERVATION, not BUG.

If any safety net neutralizes the bug in the common path, either downgrade severity or drop the finding. Your finding must explicitly state which safety nets you checked for and why each is absent or insufficient in the specific path you're flagging. Saying "no safety net exists" without enumeration is inflation.

4b. **Severity requires a realistic trigger.** CRITICAL/HIGH require the bug to fire under defaults or common configuration. A bug that only triggers with non-default settings, or only when a developer passes a type not in the documented union, is MEDIUM at most. Describe the trigger condition explicitly; if it's niche, cap severity accordingly.

4c. **One finding is one specific broken behavior. Specificity beats dumps.** A finding names exactly one thing that goes wrong, for one identifiable user, at one identifiable moment, along one specific code path. Not a theme, not a pattern, not a class of problems.

- **No bundling.** If the finding contains "and also…", it is two findings. Write the higher-impact one and drop the rest. Bundled findings read as dumps and get discounted.
- **No area-dumps.** If one area has five weaknesses, surface the single one with the highest user impact and drop the rest. A long list of small issues in the same file is noise.
- **Architectural observations are not findings.** "Error handling is inconsistent across providers," "this pattern is not defensive," "callbacks swallow errors as a class" — these describe code shape, not broken moments. Drop them unless you can point at one specific execution path where one specific user experiences one specific wrong outcome.
- This is distinct from rule 5 below: rule 5 is about the same bug (same root cause, same symptom) appearing at multiple call sites — that is legitimately one finding. Rule 4c is about different bugs being smuggled into one entry — that is bundling, and it is not allowed.

5. **Consolidate** — If the same root cause produces failures at multiple steps in the trace (e.g., a missing model attribute breaks create, update, and query), report it ONCE listing all affected code paths. The developer will fix them in one PR — they should be one finding.

6. **Compare similar actions** — When you find something wrong, check if other actions that do similar things (e.g., other create endpoints, other cancel flows) have the same issue or handle it correctly. This comparison is the highest-signal evidence.

7. **No minimum, no maximum** — Report every bug that meets the quality bar. If you find zero bugs along this trace, say so. If you find twelve, report twelve. Do not pad. Do not cut.

## Output Format (sub-agent returns this)

### Action Trace

```
ACTION: [What the user/system is doing — verb phrase]

TRACE:
1. [Entry point] → [file:line]
   → [what happens: auth check, validation, data read]
   → [reads from: table/cache/API]
   → [writes to: table/cache/API]

2. [Next function call] → [file:line]
   → [what happens]
   → [what depends on this data being current?]
   → [side effects: what fires, what doesn't fire but should]

3. [Background work / async follow-up] → [file:line]
   → [what happens after the HTTP response returns]
   → [what can go wrong: no commit before task launch, stale ORM objects, etc.]

...continue until there is nothing left to trace...
```

### Findings (from this trace)

For EACH bug found along the trace, report it in consequence-chain format:

```
FINDING: [What happens to the user or the business — NOT what's wrong in the code]

WHAT HAPPENS:
  1. [User does X]
  2. [System does Y]
  3. [But Z goes wrong because...]
  4. [The consequence is...]
  5. [The user sees / the business loses...]

IMPACT_CATEGORY: [REVENUE_LEAK | SUPPORT_BURDEN | DATA_LOSS | SECURITY_BREACH | COMPLIANCE | PROD_INCIDENT | BRAND_DAMAGE]
  [If you can't pick exactly one, this isn't an issue — drop it.]

BLAST RADIUS: [Who is affected, how often — code-observable only. "100% of callers that hit X", "every deployment with default config Y", "every user of feature Z". Do NOT invent user counts, $ values, or %-of-users without evidence.]

VERIFICATION ARTIFACT:
  [Structured code-reference pair — file:line on both sides]
  [Use the appropriate format: auth, race condition, error handling, etc.]

HOW TO VERIFY: (if a simple repro exists — omit if it requires complex setup)
  1. [Step to reproduce]
  2. [What to observe]
  3. [Expected vs actual]

WHY:
  [Root cause — 1-2 sentences with file:line references]

FIX:
  [Specific code change — not "add error handling" but the actual fix]

SEVERITY: [CRITICAL | HIGH | MEDIUM | LOW]
  [One sentence justifying against the impact taxonomy + calibration from Phase 1]
```

**THREE RULES FOR FINDINGS:**

**Rule 1 — The title is what happens to the user or business, not what's wrong in the code.**

- BAD: "Missing auth_token in cancel_pod_agent call"
- GOOD: "Cancelled sessions keep running silently, burning cloud credits"
- BAD: "Race condition in send_message status check"
- GOOD: "Double-clicking Send fires two agents on the same pod, corrupting the conversation"

Test: if a CTO wouldn't prioritize this in their next sprint, rewrite it.

**Rule 2 — Lead with the consequence chain.**

The WHAT HAPPENS block is a numbered timeline of what actually happens, written from the user's perspective down to the system failure. A CTO reads the first 5 lines and immediately knows this matters. The root cause comes AFTER, not before.

**Rule 3 — The verification artifact must be checkable in under 60 seconds.**

A reviewer reads the two cited code locations and can confirm or deny the bug without reading anything else. Every artifact needs concrete file:line references on both sides. If you can't produce this, you don't have enough evidence.

If you found NO bugs along this trace, say so explicitly: "No bugs found along this trace." Do not invent findings to fill a quota.

### Cross-Action Clues

If you notice something suspicious that depends on another action's behavior, report it separately:

- **This action assumes**: [what assumption this action makes]
- **Would break if**: [what other action or condition would violate the assumption]
- **Check action**: [which other action to verify against]

## After Sub-agents Return

1. Read all action traces and findings.
2. **Cross-action consolidation**: If the same bug appears in 3+ action traces (e.g., "no SSE broadcast on status change"), pick the single highest-impact action as the primary finding and list the other affected actions under `All locations` in that finding. Do not create a separate "systemic" umbrella finding.
3. **Variant verification**: Check if a bug found in one action also exists in similar actions.
4. **Workflow verification**: Compare findings against the Critical Workflow Ledger. If the code contradicts a workflow invariant, elevate it.
5. **Consolidation**: If multiple findings share the same root cause, merge them into ONE finding listing all affected code paths.
6. **Quality bar**: Report every finding that meets ALL of these criteria. There is no minimum and no maximum.
   - Something IS broken or produces wrong state (not something that SHOULD exist)
   - You can cite specific file:line on both sides of the bug
   - You can describe the consequence in business terms
   - You read 50 lines of context and found no mitigation
   - A CTO would prioritize fixing this in their next sprint

## Output

Write to `$DEEP_REVIEW_DIR/action-traces.md`:

```markdown
# Action Trace Findings

## Phase Metadata
- Actions traced: [N]
- Actions with bugs found: [N]
- Cross-action synthesis passes: [N]
- Raw findings before consolidation: [N]
- Final findings: [N]

## Findings

### Finding 1: [What happens to the user/business — consequence, not code description]

**WHAT HAPPENS:**
1. [Timeline of what goes wrong, from user action to consequence]
2. ...

**IMPACT_CATEGORY:** [REVENUE_LEAK | SUPPORT_BURDEN | DATA_LOSS | SECURITY_BREACH | COMPLIANCE | PROD_INCIDENT | BRAND_DAMAGE]

**BLAST RADIUS:** [Who is affected, how often — code-observable only.]

**VERIFICATION ARTIFACT:**
[Structured code-reference pair with file:line on both sides]

**HOW TO VERIFY:** (if applicable)
1. [Quick repro steps]

**WHY:** [Root cause with file:line references]

**FIX:** [Specific code change]

**SEVERITY:** [level] — [one sentence justification against the impact taxonomy from Phase 1]

**DISCOVERED VIA:** Action trace: "[action name]"

**LABELS:** ["cancel", "split-brain", etc.]

### Finding 2: ...
```

Do not emit a "Systemic Issues" section. If you observe a pattern across many files, either (a) pick the single highest-impact instance and file it as one concrete finding with the others listed under `All locations`, or (b) drop it. Pure pattern-level observations without a specific broken moment are not findings.

## Done

When `$DEEP_REVIEW_DIR/action-traces.md` exists with a Findings section (even an empty one), this phase is complete. Output a one-line summary:

```
Found N candidate findings across M traced actions. Validation in Phase 5 will confirm or disprove each one.
```

The next phase (`product-scan`) runs in parallel against the same architecture map. The validation phase (5) reads both and produces the final `./findings.md`.
