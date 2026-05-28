# Phase 4 — Product Scan

> Loaded from `SKILL.md` as the fourth phase of Deep Review. Reads `$DEEP_REVIEW_DIR/architecture-map.md` and `$DEEP_REVIEW_DIR/action-traces.md`. Writes `$DEEP_REVIEW_DIR/product-scan.md`. Finds user-facing bugs the infrastructure-focused action-trace phase missed.

You are finding bugs that hurt users. Not security vulnerabilities, not infrastructure concerns — bugs where a real person using this product gets confused, stuck, misled, or loses work.

Phase 3 (action-trace) asked "what can go wrong?" You are asking a different question: "what does the user experience when it does?"

These bugs live everywhere in the stack. A backend state machine that forgets to revert a status is a user-facing bug. A query that undercounts is a user-facing bug. An error handler that returns a useful message nobody reads is a user-facing bug. You trace from user pain backwards to root cause, wherever it lives.

## Mental model: 100% confidence, zero assumptions

A finding is something you would defend at 100% confidence — a real user will hit a real consequence on real code paths in real configurations. If you are not certain of that, you have a hypothesis, not a finding.

When a finding feels "probably right" but not certain, name the assumption you are leaning on. Common ones:

- "I'm assuming this user action is reachable in the UI" — find the path
- "I'm assuming this state can occur" — trace the transitions that reach it
- "I'm assuming this query returns this shape" — read the actual query and the data
- "I'm assuming this error is shown to the user" — check the rendering layer

Then attempt to remove the assumption by reading the code, the spec, or the actual data. If after that work the assumption still stands, your confidence is not 100%. **Drop the finding.** Three findings the team will fix beats twelve they have to re-verify.

The phrases "assume," "suppose," "imagine," "could," "may," "probably," "potentially" are inflation markers. If they appear in your finding's reasoning, treat them as a flag to verify or drop.

## Setup

```bash
DEEP_REVIEW_DIR="${DEEP_REVIEW_DIR:-./.deep-review}"
```

Read the feature map, context profile, action inventory, and severity calibration from `$DEEP_REVIEW_DIR/architecture-map.md`.

Read the action-trace findings from `$DEEP_REVIEW_DIR/action-traces.md`. Do NOT duplicate findings already reported there. Your job is to find what Phase 3's infrastructure lens missed.

Check for developer-authored context files (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`) in the repo root and key subdirectories. These describe the project's architecture, conventions, and known patterns from the developer's perspective — use them to understand user flows, expected behavior, and areas the developer considers fragile or important.

## How to Think

Phase 3 asks 7 questions about data flow, side effects, and concurrency. You ask 5 different questions:

1. **What does the user see when this succeeds?** Is the displayed result correct? Does the count match reality? Does the status reflect the actual state? Is the success feedback adequate — or does the action succeed silently?

2. **What does the user see when this fails?** Trace every error the backend can throw for this action forward to the UI. Is the error message useful or generic? Does the UI recover to a usable state or get stuck? Can the user retry?

3. **What does the user see when the data is partial, empty, or transitional?** First use with zero data. A run that hasn't completed yet. An entity that was deleted by another user. A field that's null because an optional step was skipped.

4. **Is the state machine complete?** Map every status an entity can have. Check every transition. When an operation fails or is interrupted, does the status revert or does it get stuck in an intermediate state forever?

5. **Does the code keep its promises?** If the UI says "you can restore this later" — can you? If a count says "5 issues" — are there really 5? If a filter says "critical" — does the query actually filter by critical?

## How to Work

### Wave 1: Five Parallel Lenses

Launch 5 sub-agents IN PARALLEL via the `Task` tool, one for each lens. Each sub-agent gets the full action inventory and works across the entire codebase (backend AND frontend).

**Every lens sub-agent prompt MUST begin with the Mental Model section verbatim** (the "100% confidence, zero assumptions" block above). The lens body that follows describes *what to look for*; the mental model describes *what bar a finding has to clear before being written*. Without the mental model, the lens sub-agents inflate findings with unverified assumptions — this has been measured across prior runs.

The lenses are:

### Lens A: Error Paths

```
For every feature area in the action inventory, find every error the backend can produce
(HTTPException, ValueError, database errors, external service failures). For each error:

1. What HTTP status and detail message does the backend return?
2. Trace forward to the frontend: what does the user actually see?
3. Is the backend's useful error message preserved or discarded?
4. After the error, is the UI in a usable state? Can the user retry?
5. Are there errors that should exist but don't? (e.g., validation on create but not update)

Look for:
- Generic catch-all error handlers that discard specific backend messages
- Errors that leave the UI in a broken state (stuck spinner, dialog won't close, button permanently disabled)
- Backend operations that silently succeed with wrong data instead of failing (no validation, no constraint)
- Inconsistency: one endpoint validates input, a similar endpoint doesn't
- Query-level errors (fetch/GET) that show infinite loading instead of an error state

Cross-reference error handling patterns. If you find that ALL mutation hooks discard error detail, pick the single highest-impact hook (most-used flow or most-user-exposed) as the concrete finding and list the other affected hooks under `All locations` in that finding. Do not create a separate "systemic" umbrella finding.
```

### Lens B: State Machine Completeness

```
For every entity with a status field (issues, runs, steps, sessions, subscriptions, jobs):

1. Map every status value from the backend enum/model
2. Map every transition: what action moves status from A to B?
3. For every transition TO an intermediate state (in_progress, running, pending):
   check — what moves it OUT? Is the exit guaranteed?
4. What happens on failure? Does the status revert, or get stuck?
5. Check the frontend: does it handle every status value?
   Is there a switch/case or if/else that covers all variants?

Look for:
- States with no exit path (entity gets stuck forever after a failure)
- Transitions that aren't enforced (API accepts any-to-any status change)
- Status values the backend can set but the frontend doesn't render
- Intermediate states that persist after the operation that set them has failed
- Counts or queries that filter on one status but the user expects a broader filter
  (e.g., "open issues" excluding acknowledged/in_progress)
```

### Lens C: Data Integrity and Display Truth

```
For every API response that includes counts, aggregations, lists, or computed values:

1. Read the backend query. Is the WHERE clause correct?
   Could it include/exclude wrong records?
2. Is workspace/tenant isolation enforced in every query?
3. If the UI shows a count and a list — do they always agree?
   (e.g., "5 issues" badge but only 3 in the list)
4. If the UI has filters — does the backend query actually support them?
   What happens when the frontend sends a filter the backend ignores?
5. After a mutation, does every related display update?
   (count in sidebar, badge on card, total in header, items in list)

Look for:
- Queries that only search one source but should search multiple
- Counts that use a different filter than the list they describe
- Client-side filtering on paginated data (produces wrong totals and broken pagination)
- Deduplication logic that depends on exact string matching
- Mutations that invalidate one cache but miss a related one
```

### Lens D: Boundary Conditions and QA

```
For each user-facing feature (not internal/system actions), write the test cases
a QA tester would check. Then verify against the code.

For each feature, check AT MINIMUM:
- Create with missing/empty required fields → does validation exist?
- Create with extremely long input → is there a max length?
- Create with duplicate/conflicting data → is there a uniqueness check?
- Update that removes required data → does the update schema validate?
- Delete while the entity is actively used → what happens to dependents?
- The same action via different entry points → do they behave identically?
  (e.g., keyboard shortcut vs button, API vs UI)
- Actions with implicit preconditions → are they checked?
  (e.g., "needs GitHub connected" — is this validated, or does it fail at runtime?)

Look for:
- Validation present on create but missing on update (common pattern)
- Button handlers that bypass guards the button's disabled state provides
  (keyboard shortcuts, programmatic triggers)
- Destructive actions with no confirmation dialog or undo path
- Operations that promise reversibility but don't deliver
  ("you can restore this later" but no restore mechanism exists)
```

### Lens E: Frontend State and User Experience

```
For each major page/flow, read the frontend component AND the hooks/API calls it uses.

1. Loading states: Is there a loading indicator? Can it get stuck?
   (e.g., query error makes loading spin forever because error state isn't handled)
2. Empty states: What renders with zero data? Blank page? Helpful guidance?
3. Mutation feedback: After create/update/delete, does the user get confirmation?
   Does the UI update immediately or require a manual refresh?
4. Error recovery: After an error, is the UI in a usable state?
   Can the user retry without refreshing the page?
5. Stale data after navigation: User triggers action on page A,
   navigates to page B, comes back to page A — is data fresh?

Look for:
- Pages that check isLoading but not isError (infinite spinner on query failure)
- Mutation handlers that set local state (e.g., setTriggered(true)) but never
  clear it on failure (button stuck in triggered state forever)
- Forms/dialogs that close before the mutation completes, hiding errors
- Polling that conflicts with mutations (poll overwrites optimistic update)
- Post-mutation navigation (router.push) that fires after the user already navigated away
```

## Sub-agent Rules

1. **No security findings** — Phase 3 covered those. If you find something that's BOTH a security issue and a user-facing bug, report the user-facing aspect only.
2. **No feature requests** — "should have search" is a feature. "search returns wrong results" is a bug.
3. **No performance issues** — "this query is slow" is not your scope. "this query returns wrong data" is.
4. **Full-stack tracing required** — every finding must trace from root cause (wherever it lives) to user-visible consequence. A backend bug without a user consequence is not a finding. A frontend bug without checking the backend is incomplete.
5. **Check for mitigations** — Before reporting, trace the full execution path for mitigations. Check middleware, decorators, base classes, callers, and configuration that could handle the issue.

5a. **Safety-net check before claiming severity.** Before writing a finding at any severity, explicitly enumerate and check for safety nets that would neutralize the bug: stream/transform finalization, error propagation (caught upstream, converted to typed result), recovery paths (stop/cancel/abort/regenerate), schema/type validation catching bad data downstream, default configuration (bug only fires under non-default settings), and existing tests that assert the behavior as expected. If any net neutralizes the bug in the common path, downgrade or drop the finding. Your finding must state which nets you checked and why each is absent or insufficient.

5b. **Severity requires a realistic trigger.** CRITICAL/HIGH require the bug to fire under defaults or common user configuration. A bug that only triggers with non-default settings, rare input types, or unusual runtime conditions is MEDIUM at most. Describe the trigger condition explicitly; if it's niche, cap severity.

5c. **One finding is one specific broken behavior. Specificity beats dumps.** A finding names one thing that goes wrong, for one identifiable user, at one identifiable moment, along one specific code path.

- **No bundling.** If the finding contains "and also…", it is two findings.
- **No area-dumps.** If one feature has five weaknesses, surface the single one with the highest user impact and drop the rest.
- **Architectural observations are not findings.** "Error messages are inconsistent," "this flow is not defensive," "state transitions are loose as a class" — these describe code shape, not broken moments. Drop them unless you can point at one specific execution path where one specific user experiences one specific wrong outcome.

6. **Compare similar features** — if you find a bug in one place, check if similar features handle it correctly. The comparison IS the evidence.

7. **No minimum, no maximum** — report every finding that meets the quality bar.

## Finding Format

For each bug, report in this format:

```
FINDING: [What the user experiences — their words, not yours]

WHAT THE USER SEES:
  1. [User does X — the action they're taking]
  2. [They expect Y — what should happen]
  3. [Instead they see Z — what actually happens]
  4. [They are left in state W — confused/stuck/misled/lost data]

ROOT CAUSE: [where in the stack, file:line, what's wrong]

EVIDENCE:
  [file:line — the code that's wrong]
  [file:line — comparison code that handles it correctly, if applicable]

IMPACT_CATEGORY: [SUPPORT_BURDEN | DATA_LOSS | BRAND_DAMAGE | REVENUE_LEAK | SECURITY_BREACH | COMPLIANCE | PROD_INCIDENT]
  [Most product/UX findings are SUPPORT_BURDEN, DATA_LOSS, or BRAND_DAMAGE. If you can't pick one, drop it.]

IMPACT_FLAVOR: blocking | confusing | misleading | data-loss
  [The user-facing texture of the impact, in addition to the category above.]

FIX: [specific code change]

SEVERITY: [CRITICAL | HIGH | MEDIUM | LOW]
  Use the impact taxonomy + calibration from Phase 1. As an additional product/UX guideline:
  - CRITICAL: user loses data or gets permanently stuck with no workaround
  - HIGH: user is blocked or significantly confused on a primary flow, workaround exists
  - MEDIUM: user sees wrong information or has degraded experience
  - LOW: cosmetic inconsistency or edge case most users won't hit
```

**Title rule: write what the user would type in a support ticket.**

- BAD: "Missing cache invalidation in useTriggerAdHocFix onSuccess"
- GOOD: "Dashboard still shows old issue count after triggering a fix"
- BAD: "Issue status not reverted on AutomationRun failure"
- GOOD: "Issue shows 'fixing...' forever after the fix actually failed"

## Output

Write to `$DEEP_REVIEW_DIR/product-scan.md`:

```markdown
# Product Scan Findings

## Phase Metadata
- Lenses run: [N]
- Features/pages covered: [N]
- Raw findings before synthesis: [N]
- Final findings: [N]

## Findings

### Finding 1: [What the user would type in a support ticket]

**WHAT THE USER SEES:**
1. [Action → Expectation → Reality → Consequence]

**ROOT CAUSE:** [file:line — what's wrong in the code]

**EVIDENCE:**
- [file:line — broken code]
- [file:line — comparison code that works correctly]

**IMPACT_CATEGORY:** [SUPPORT_BURDEN | DATA_LOSS | BRAND_DAMAGE | REVENUE_LEAK | SECURITY_BREACH | COMPLIANCE | PROD_INCIDENT]

**IMPACT_FLAVOR:** [blocking|confusing|misleading|data-loss] — [who, how often]

**FIX:** [specific code change]

**SEVERITY:** [level] — [one sentence justification against the impact taxonomy from Phase 1]

**DISCOVERED VIA:** [Lens A/B/C/D/E]

**LABELS:** ["state-machine", "error-handling", "stale-ui", etc.]

### Finding 2: ...
```

Do not emit a "Systemic Issues" section. If you observe a pattern across many features, either (a) pick the single highest-impact instance and file it as one concrete finding with the others listed under `All locations`, or (b) drop it. Pure pattern-level observations without a specific broken moment are not findings.

## Done

When `$DEEP_REVIEW_DIR/product-scan.md` exists with a Findings section, this phase is complete. Output a one-line summary:

```
Found N user-facing candidate findings across M features. Validation in Phase 5 will confirm or disprove each one.
```

The next phase (`adversarial-validate`) merges these findings with the action-trace findings and produces the final `./findings.md`.
