---
name: production-gap-auditor
description: >
  Performs deep production-readiness audits of codebases to find deficiencies that unit tests
  cannot catch — the kind of bugs, gaps, and silent failures that only real users discover.
  Use this skill whenever the user asks to audit a codebase, find production gaps, check if
  something is "actually working", investigate why users are having issues that tests don't
  catch, review production readiness, or assess whether a product delivers on its promises.
  Also trigger when users say things like "something feels off", "tests pass but users complain",
  "is this app really working?", "what's broken that we're not seeing?", or "audit this".
  Supports a quick mode — trigger with "quick audit", "quick sweep", or "fast audit" to run
  only the broad sweep phase and surface the top 5 findings without deep investigation.
---

# Production Gap Auditor

You are performing a production gap audit. Your job is to find deficiencies that no unit test
would ever catch — the gaps between what the code *does* and what a real user *experiences*.

Think like a user, not a developer. A developer sees functions and return values. A user sees
a button that does nothing, a loading spinner that never stops, a setting that doesn't save,
a notification that never arrives. Your job is to find every place where the code technically
runs but the user's intent is silently betrayed.

This is not a code review. You are not looking for style issues, naming conventions, or
missing types. You are looking for **broken promises** — places where the product fails to
deliver what it claims to, what users expect, or what the architecture implies it should.

---

## Prerequisites

This skill requires access to:
- **Glob** — file pattern matching
- **Grep** — content search across the codebase
- **Read** — reading file contents
- **Bash** — running git commands for history analysis
- **Agent** — spawning subagents for parallel investigation

**Context budget**: This audit is context-intensive. It works best with large-context models
(100k+ tokens). On smaller context windows, use Quick Mode or manually scope the audit to
a specific subsystem.

**Quick Mode**: If the user requests a "quick audit", "quick sweep", or "fast audit", skip
Phase 3 (Deep Investigation) entirely. Run Phase 1 reconnaissance, Phase 2 broad sweep, then
produce a report with only the top 5 highest-severity findings. This should complete in a
fraction of the time of a full audit.

---

## Phase 1: Reconnaissance

Before you can find what's broken, you need to understand what this product is *supposed to do*.

### 1.1 Understand the Product

Read these files (whichever exist) to build a mental model of the product's intent:
- README, CLAUDE.md, docs/, wiki/
- package.json, Cargo.toml, pyproject.toml, go.mod, Gemfile, pom.xml (name, description, scripts)
- App store descriptions, marketing copy, onboarding screens
- API documentation, OpenAPI specs
- Environment variable documentation

Write down (in your thinking) a clear statement: "This product is supposed to let users do X, Y, Z."

### 1.2 Detect Language & Framework

Identify the primary language(s) and framework(s) in use. This determines which search patterns
to apply in Phase 2.

- Check file extensions: `.ts`/`.js` (Node/JS), `.py` (Python), `.go` (Go), `.rs` (Rust),
  `.java`/`.kt` (JVM), `.rb` (Ruby), `.swift` (Swift)
- Check package manifests: `package.json` (Node), `pyproject.toml`/`requirements.txt` (Python),
  `go.mod` (Go), `Cargo.toml` (Rust), `pom.xml`/`build.gradle` (JVM), `Gemfile` (Ruby)
- Identify frameworks: React/Next/Vue/Svelte (frontend), Express/FastAPI/Gin/Actix/Rails (backend),
  React Native/Flutter/SwiftUI (mobile)

Record the detected stack — you'll use it to select language-specific patterns in Phase 2.

### 1.3 Map the Architecture

Identify:
- **Entry points**: Where does a user's journey begin? (routes, screens, CLI commands, API endpoints)
- **Core flows**: What are the 5-10 most important things a user does? (sign up, create X, view Y, configure Z)
- **Integrations**: What external services does this connect to? (databases, APIs, payment providers, auth providers)
- **Data flow**: How does data move from input -> processing -> storage -> display?
- **Background work**: Crons, queues, webhooks, sync jobs — what runs without a user watching?

Use Glob and Grep extensively. Read key files. Map the terrain before you start digging.

#### Git History Signals

Run these commands to identify hotspots and areas of instability:

```bash
# Files with the most churn in the last 3 months — likely bug-prone areas
git log --since="3 months ago" --format=format: --name-only | sort | uniq -c | sort -rn | head -20

# Recent reverts, rollbacks, and hotfixes — features that failed in production
git log --since="6 months ago" --grep="revert\|rollback\|hotfix" --oneline

# Files changed in the last 2 weeks — actively evolving and potentially unstable
git log --since="2 weeks ago" --format=format: --name-only | sort -u
```

Prioritize high-churn files and recently reverted areas during Phase 2 investigation.

### 1.4 Identify Claimed Capabilities

From docs, README, config, and code structure, compile a list of every feature or capability
this product claims to offer. This becomes your checklist for Phase 3.

---

## Phase 1.5: Scope the Audit

Before diving into the broad sweep, determine how much of the codebase you can realistically
audit within context and time constraints.

**For codebases under ~500 files**: Audit everything.

**For larger codebases**: Prioritize and state your scope explicitly.
- Focus on files identified as hotspots from git history (Phase 1.3)
- Focus on files in critical user paths (Phase 1.3 core flows)
- Focus on recently changed code (last 2-4 weeks)
- Explicitly state in the report what you audited and what you excluded

Write a brief scope statement:
> "Auditing [X subsystem / the full codebase]. Prioritizing [critical user flows / recent changes / high-churn areas]. Excluding [vendored code / generated files / test fixtures]."

---

## Phase 2: Broad Sweep

Scan the codebase for patterns that signal production gaps. Use parallel subagents where
possible to cover more ground. Each pattern below is something a unit test would likely miss.

> **False Positive Filter**: Before including any finding in the report, verify it's not a
> false positive. A caught error that IS surfaced to the user via toast/alert/redirect is fine.
> A TODO in a test file is different from a TODO in production code. An empty catch that
> intentionally swallows a benign error (e.g., optional analytics) may be acceptable. Note
> deliberate trade-offs as "Accepted Risk" rather than flagging them as bugs. Pattern matches
> alone are not findings — you must trace the context before including them.

### 2.1 Silent Failures

Things that fail without anyone knowing:

- **Swallowed errors**: `catch (e) {}`, `catch (e) { console.log(e) }`, `.catch(() => {})` —
  errors that are caught but never surfaced to the user or monitoring
- **Ignored return values**: Async functions called without await, promises that aren't handled,
  results that are computed but never used
- **Fallback defaults that hide problems**: Functions that return `[]`, `null`, `0`, or `""` on
  failure instead of propagating the error — the UI shows "no results" instead of "something broke"
- **Logging without acting**: Code that logs errors but continues as if nothing happened

Search patterns (JavaScript/TypeScript):
```
catch\s*\([^)]*\)\s*\{[\s]*\}
catch\s*\([^)]*\)\s*\{\s*console\.(log|warn|error)
\.catch\(\s*\(\)\s*=>
\.catch\(\s*console\.(log|warn|error)\s*\)
```

Language-specific patterns:
- **Python**: `except:\s*pass`, `except Exception.*:\s*(pass|continue)`, `except.*:\s*logging\.(warn|error|info)`
- **Go**: `_ = `, `err\s*=.*\n[^i]` (error assigned but next line isn't `if err`), `//\s*nolint`
- **Rust**: `\.unwrap\(\)` in non-test files, `let _ =`, `\.ok\(\)` discarding errors
- **Java/Kotlin**: `catch\s*\([^)]*\)\s*\{\s*\}`, `catch\s*\([^)]*\)\s*\{\s*//`, `@SuppressWarnings`
- **Ruby**: `rescue\s*=>?\s*nil`, `rescue\s*StandardError`, `rescue\s*Exception`

### 2.2 Incomplete Implementations

Features that exist in skeleton but not in substance:

- **TODO/FIXME/HACK markers**: Not just their existence, but what they guard — is a TODO blocking
  a critical user flow?
- **Stub functions**: Functions that return hardcoded values, throw "not implemented", or just `pass`
- **Commented-out code**: Large blocks of commented code often indicate abandoned features that
  the UI still references
- **Placeholder data**: Mock data, hardcoded IDs, "lorem ipsum", `test@example.com` in production code
- **Partial CRUD**: Create exists but not Delete. Update exists but not Read. List exists but
  has no pagination
- **Feature flags stuck off**: Features gated behind flags that are permanently disabled

Search patterns:
```
TODO|FIXME|HACK|XXX|NOCOMMIT|STOPSHIP
lorem ipsum|placeholder|test@example\.com|changeme|CHANGEME|your-api-key|sk-xxx|pk_test
```

Language-specific patterns:
- **JavaScript/TypeScript**: `throw new Error\(.*not.implemented`, `return.*\/\/\s*TODO`
- **Python**: `raise NotImplementedError`, `pass\s*#\s*TODO`, `def.*:\s*\.\.\.\s*$`
- **Go**: `panic\("not implemented"\)`, `// TODO`, `func.*\{\s*return nil\s*\}`
- **Rust**: `todo!\(\)`, `unimplemented!\(\)`, `panic!\("not implemented"\)`
- **Java/Kotlin**: `throw.*UnsupportedOperationException`, `throw.*NotImplementedException`

### 2.3 Integration Gaps

Where different parts of the system don't properly talk to each other:

- **Frontend calls to nonexistent backends**: API calls in UI code that hit endpoints which
  don't exist or return different shapes than expected
- **Schema mismatches**: Frontend expects `{ user_name }` but backend sends `{ userName }`
- **Event system disconnects**: Events emitted but never listened to, or listeners for events
  never emitted
- **Webhook handlers that don't validate**: Endpoints that accept webhook payloads without
  verifying signatures or source
- **OAuth flows that dead-end**: Redirect URIs that point nowhere, callback handlers that don't
  complete the flow
- **Missing environment variables**: Code references env vars that aren't in .env.example or docs

Search patterns:
```
process\.env\.\w+|os\.environ|os\.getenv|env::var|System\.getenv
fetch\(|axios\.|httpClient\.|\.get\(|\.post\(|\.put\(|\.delete\(|\.patch\(
\.emit\(|\.on\(|addEventListener|removeEventListener|\.dispatch\(
webhook|callback.*url|redirect.*uri|oauth.*callback
```

**Verification approach**: Collect all API endpoint URLs called from the frontend. Collect all
route definitions from the backend. Cross-reference — every frontend call should have a
matching backend route. Collect all `process.env.*` / `os.environ` references and cross-reference
against `.env.example` or environment documentation.

### 2.4 State & Data Integrity

Where data gets lost, corrupted, or out of sync:

- **Optimistic updates without rollback**: UI updates immediately but never reverts if the
  server call fails
- **Cache without invalidation**: Data is cached but never refreshed, showing stale state
- **Race conditions**: Two async operations that can interleave and corrupt state
  (especially in stores/state management)
- **Partial writes**: Database operations that update multiple tables/records without transactions
- **Orphaned data**: Records that lose their parent reference but aren't cleaned up
- **Missing cascading deletes**: Delete a user but their data/sessions/tokens persist

Search patterns:
```
optimistic|\.update\(|\.set\(.*\).*(?!rollback|revert|catch)
setInterval|setTimeout
\.subscribe\(|\.on\(|addEventListener
cache|Cache|memo|Memo|staleTime|cacheTime|ttl|TTL
transaction|BEGIN|COMMIT|ROLLBACK
ON DELETE (SET NULL|NO ACTION|RESTRICT)
```

Language-specific patterns:
- **React**: `useState.*set.*fetch` (state set before async completes), `useEffect` without cleanup return
- **Python**: `with db.session` without `commit()`/`rollback()`, missing `finally` blocks
- **Go**: goroutines writing to shared state without mutex/channels

### 2.5 User Experience Black Holes

Places where the user gets stuck with no feedback:

- **Loading states without timeouts**: Spinners that spin forever if the request fails
- **Empty states not handled**: Lists that show blank instead of "No items yet"
- **Error messages that say nothing**: Generic "Something went wrong" without context or recovery
- **Unreachable UI states**: Buttons that are always disabled, screens that can't be navigated to
- **Forms that silently discard input**: Validation that prevents submission without telling
  the user what's wrong
- **Navigation dead ends**: Screens with no back button, flows that can't be cancelled
- **Onboarding that can be skipped but is required**: Setup flows that the UI allows skipping
  but the app requires completing

Search patterns:
```
loading|isLoading|spinner|pending|isFetching
empty.*state|no.*results|no.*data|no.*items
Something went wrong|An error occurred|Unknown error|Unexpected error
disabled={true}|\.disabled\s*=\s*true|aria-disabled
```

**Verification approach**: For each loading state found, check whether there is a corresponding
error state and timeout handler nearby. Loading without error/timeout handling = finding.
For disabled elements, check if there's ever a condition that re-enables them.

### 2.6 Security Theater

Security measures that look right but aren't:

- **Auth checks on some routes but not others**: Protected pages that fetch from unprotected APIs
- **Client-side-only validation**: Input validated in JS but not on the server
- **Tokens stored insecurely**: Auth tokens in localStorage, credentials in client-side code
- **CORS configured too permissively**: `Access-Control-Allow-Origin: *` on authenticated endpoints
- **Rate limiting gaps**: Rate limits on login but not on password reset or API endpoints
- **Permission checks that don't check**: Middleware that's defined but not applied, or that
  checks role but not resource ownership

Search patterns:
```
Access-Control-Allow-Origin.*\*
localStorage\.(set|get)Item.*(token|auth|session|key|secret|jwt|credential)
cors\(\s*\)|cors\(\{|allowOrigin|allow_origin
@Public|isPublic|skipAuth|noAuth|AllowAnonymous|permit_all
rate.?limit|throttle|RateLimit
innerHTML\s*=|v-html|safe\|
```

**Verification approach**: Map all routes/endpoints. For each, check whether auth middleware
is applied. Compare the set of protected frontend pages against the set of protected API
endpoints — mismatches are findings.

### 2.7 Performance Traps That Affect Behavior

Performance issues that manifest as functional bugs to users:

- **Unbounded queries**: `SELECT * FROM table` without LIMIT, list endpoints that return everything
- **N+1 queries**: Loops that make a database/API call per item
- **Missing pagination**: Endpoints that work fine with 10 items but crash/timeout with 10,000
- **Memory leaks via subscriptions**: Event listeners, websocket connections, or intervals that
  are set up but never torn down
- **Blocking operations on main thread**: Heavy computation that freezes the UI

Search patterns:
```
SELECT \*|SELECT.*FROM(?!.*LIMIT)(?!.*WHERE)
\.find\(\s*\)|\.findAll\(\s*\)|\.all\(\s*\)|\.list\(\s*\)
for.*await|\.forEach\(.*await|\.map\(.*await
setInterval|setTimeout|new WebSocket|\.connect\(
```

Language-specific patterns:
- **Python/Django/SQLAlchemy**: `.all()` without `.limit()`, `for obj in queryset` (lazy eval in loop)
- **Go**: unbuffered channels in loops, `defer` inside loops
- **Ruby/Rails**: `.each` on ActiveRecord relation without `.find_each` or `.in_batches`

### 2.8 Observability & Monitoring Gaps

Gaps that make production issues invisible — problems happen but nobody knows:

- **No error tracking integration**: No Sentry, Bugsnag, Datadog, New Relic, or equivalent
- **Missing structured logging on critical paths**: Auth flows, payment processing, data
  mutations, and background jobs should all have structured logs
- **No health check endpoint**: Infrastructure probes (load balancers, k8s) need a `/health`
  or `/healthz` endpoint
- **No alerting on background job failures**: Cron jobs and queue workers that fail silently
- **Missing request tracing**: No correlation IDs or distributed tracing for debugging
  request flows across services
- **No uptime monitoring**: No external checks that the service is responsive

Search patterns:
```
sentry|bugsnag|datadog|newrelic|errorTracking|captureException|captureMessage
structlog|winston|pino|bunyan|log4j|slog|tracing::
health|healthz|readiness|liveness|ping
correlation.?id|request.?id|trace.?id|x-request-id
```

**Verification approach**: Search for error tracking initialization. If absent, that's a
Critical finding for any production service. Check if critical paths (auth, payments, data
writes) have structured logging — not just `console.log` but logs with context (user ID,
action, result). Check for a health endpoint in the route definitions.

### 2.9 API Contract Validation

Where the API promises one thing but delivers another:

- **Frontend types don't match backend responses**: TypeScript interfaces that diverge from
  actual API response shapes
- **OpenAPI/GraphQL schema drift**: Spec says one thing, implementation does another
- **Missing error response handling**: Client handles `200` but not `422`, `429`, or `503`
- **Undocumented breaking changes**: API returns fields that aren't in any type definition,
  or omits fields that types mark as required

Search patterns:
```
interface.*Response|type.*Response|schema|Schema
openapi|swagger|graphql|\.graphql
\.json\(\)|\.data\.|response\.data|res\.json
status.*===.*200|response\.ok|res\.status
```

**Verification approach**: Find API call sites in frontend code. For each, identify the
expected response type/interface. Find the corresponding backend handler. Compare the actual
response shape against the expected type. Look for fields marked required in the frontend type
that the backend might not always return. Check if the frontend handles non-200 responses
(especially 401, 403, 404, 422, 429, 500, 503).

---

## Phase 3: Deep Investigation

> **Skip this phase in Quick Mode.** If the user requested a quick audit, proceed directly
> to Phase 4 with the top 5 findings from Phase 2.

This is where the real value is. For every suspicious pattern found in Phase 2, don't just
flag it — **trace the full execution path** and understand exactly what a real user would
experience.

### How to Deep Dive

For each finding:

1. **Start from the user's perspective**: What action triggers this code? A button tap?
   A page load? A background sync?
2. **Trace the full path**: Follow the code from the user action through every layer
   (UI -> handler -> service -> API -> database -> response -> UI update)
3. **Identify the exact failure mode**: What does the user see when this goes wrong?
   Not "an error occurs" but "the save button appears to work but the data isn't persisted,
   so when the user returns to the screen, their changes are gone"
4. **Assess real-world impact**: How often would a real user hit this? Is it an edge case or
   a mainline flow? Does it corrupt data or just look wrong?
5. **Check if there's monitoring**: Would anyone know this is happening? Is there alerting,
   logging, or error tracking that would catch it?
6. **Assign confidence**: How sure are you this is a real issue?
   - **High**: You traced the full execution path and confirmed the failure mode
   - **Medium**: You read the relevant code and the pattern is clearly problematic, but didn't
     trace every caller
   - **Low**: Pattern match only — the code looks suspicious but you haven't confirmed the
     actual behavior

### Verify Claimed Capabilities

Go through the list from Phase 1.4 and verify each claimed capability:

- Does the feature actually work end-to-end?
- Is it complete or only partially implemented?
- Are there edge cases in the feature that silently fail?
- Is the feature tested? If so, do the tests actually exercise the real behavior?

### Simulate Critical User Journeys

For the 5-10 core flows identified in Phase 1.3, mentally walk through each one:

- What happens on first use? (empty state, no data, no permissions)
- What happens on the happy path?
- What happens when things go wrong? (network failure, invalid data, expired session)
- What happens with concurrent users?
- What happens at scale? (1000 items instead of 10)

---

## Phase 4: Report

Generate the report at `.claude/audits/production-gap-audit-YYYY-MM-DD.md` if the `.claude/`
directory exists, otherwise fall back to `production-gap-audit.md` in the project root.
If a previous audit exists for the same date, number it (e.g., `production-gap-audit-2025-03-31-2.md`).

### Report Structure

```markdown
# Production Gap Audit
**Date**: [date]
**Codebase**: [project name and description]
**Scope**: [what was audited — and what was explicitly excluded]
**Stack**: [detected languages and frameworks]
**Mode**: [Full / Quick]

## Executive Summary
[2-3 sentences: overall production readiness assessment and the most critical findings]

## Critical Findings
[Issues that will cause data loss, security breaches, or complete feature failure for users]

### [Finding Title]
- **Location**: `file:line` (and related files)
- **What happens**: [describe from the user's perspective]
- **Why it matters**: [impact — data loss? security? broken flow?]
- **Confidence**: High / Medium / Low — [reason: "traced full path" / "pattern match, verified code" / "pattern match only"]
- **How to verify**: [steps a human tester could follow to reproduce]
- **Recommended fix**: [concrete suggestion]
- **Evidence**: [code snippets, grep results, trace of the execution path]

## High Severity
[Issues that cause significant degradation of user experience in common flows]

## Medium Severity
[Issues that affect edge cases, non-critical flows, or cause minor UX degradation]

## Low Severity
[Issues that are unlikely to affect users but represent technical debt that could escalate]

## Claimed vs. Actual Capability Matrix
| Capability (from docs/README) | Status | Notes |
|-------------------------------|--------|-------|
| [feature X]                   | Working / Partial / Broken / Missing | [details] |

## Positive Observations
[Things that are done well — not just to be nice, but so the team knows what patterns to keep]

## Methodology
[Brief description of what was examined, what tools/patterns were used, and what was excluded]
```

### Severity Classification

- **Critical**: Real users will lose data, face security exposure, or find core features
  completely non-functional. **Action: block release, escalate immediately.**
- **High**: Common user flows are degraded. Features work sometimes but fail in realistic
  scenarios. Users will notice and be frustrated. **Action: fix this sprint.**
- **Medium**: Edge cases fail, non-critical features are incomplete, or the UX degrades under
  specific (but realistic) conditions. **Action: backlog with priority.**
- **Low**: Unlikely to affect most users today, but represents latent risk that could escalate.
  **Action: tech debt tracker.**

### Writing Findings

Each finding must answer these questions:
1. What would the user be trying to do?
2. What would they experience instead?
3. Where exactly in the code does this happen?
4. How confident are you this is a real issue (vs. a pattern you might be misreading)?

Be specific. "Error handling could be improved" is useless. "When `syncService.sync()` fails
due to network timeout, the catch block on line 142 of `syncService.ts` logs the error but
doesn't update the UI, so the user sees 'Syncing...' forever with no way to retry" is useful.

**Confidence matters**: Only include findings at the confidence level warranted by your
investigation depth. A Critical-severity finding with Low confidence should either be
investigated further or downgraded. Never present a pattern-match-only grep hit as a
confirmed finding.

---

## Execution Strategy

You will be investigating a lot of code. Structure the work as follows:

### Parallel Agent Dispatch

Spawn these agents in parallel for maximum coverage:

1. **Explore agent** -> Phase 1 reconnaissance (architecture map, language detection, git history)
2. **General-purpose agent** -> Phase 2.1 + 2.2 (silent failures + incomplete implementations)
3. **General-purpose agent** -> Phase 2.3 + 2.4 + 2.9 (integration gaps + state integrity + API contracts)
4. **General-purpose agent** -> Phase 2.5 + 2.6 + 2.7 + 2.8 (UX + security + performance + observability)

Give each agent the detected language/framework from Phase 1.2 so it uses the right patterns.

### Synthesis

After all agents report back:

1. **Deduplicate**: Multiple agents may flag the same underlying issue from different angles.
   Merge them into a single finding with richer evidence.
2. **Verify Critical/High findings**: For every finding classified as Critical or High severity,
   read the actual code yourself. Do not rely on agent grep output alone. Confirm the execution
   path and failure mode.
3. **Assign confidence scores**: Based on how deeply each finding was investigated.
4. **Discard unverifiable findings**: If a finding has Low confidence and you can't quickly
   verify it, either drop it or include it in a separate "Needs Further Investigation" section
   rather than presenting it as a confirmed issue.
5. **Produce the final report**: Compile all verified findings into the report structure above.

When investigating deep, always read the actual code. Don't infer from file names or function
signatures alone. The whole point of this audit is to catch things that *look* right but aren't.
