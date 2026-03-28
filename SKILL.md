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

## Phase 1: Reconnaissance

Before you can find what's broken, you need to understand what this product is *supposed to do*.

### 1.1 Understand the Product

Read these files (whichever exist) to build a mental model of the product's intent:
- README, CLAUDE.md, docs/, wiki/
- package.json, Cargo.toml, pyproject.toml (name, description, scripts)
- App store descriptions, marketing copy, onboarding screens
- API documentation, OpenAPI specs
- Environment variable documentation

Write down (in your thinking) a clear statement: "This product is supposed to let users do X, Y, Z."

### 1.2 Map the Architecture

Identify:
- **Entry points**: Where does a user's journey begin? (routes, screens, CLI commands, API endpoints)
- **Core flows**: What are the 5-10 most important things a user does? (sign up, create X, view Y, configure Z)
- **Integrations**: What external services does this connect to? (databases, APIs, payment providers, auth providers)
- **Data flow**: How does data move from input → processing → storage → display?
- **Background work**: Crons, queues, webhooks, sync jobs — what runs without a user watching?

Use Glob and Grep extensively. Read key files. Map the terrain before you start digging.

### 1.3 Identify Claimed Capabilities

From docs, README, config, and code structure, compile a list of every feature or capability
this product claims to offer. This becomes your checklist for Phase 3.

---

## Phase 2: Broad Sweep

Scan the entire codebase for patterns that signal production gaps. Use parallel subagents where
possible to cover more ground. Each pattern below is something a unit test would likely miss.

### 2.1 Silent Failures

Things that fail without anyone knowing:

- **Swallowed errors**: `catch (e) {}`, `catch (e) { console.log(e) }`, `.catch(() => {})` —
  errors that are caught but never surfaced to the user or monitoring
- **Ignored return values**: Async functions called without await, promises that aren't handled,
  results that are computed but never used
- **Fallback defaults that hide problems**: Functions that return `[]`, `null`, `0`, or `""` on
  failure instead of propagating the error — the UI shows "no results" instead of "something broke"
- **Logging without acting**: Code that logs errors but continues as if nothing happened

Search patterns:
```
catch\s*\([^)]*\)\s*\{[\s]*\}
catch\s*\([^)]*\)\s*\{\s*console\.(log|warn|error)
\.catch\(\s*\(\)\s*=>
\.catch\(\s*console\.(log|warn|error)\s*\)
```

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

### 2.6 Security Theater

Security measures that look right but aren't:

- **Auth checks on some routes but not others**: Protected pages that fetch from unprotected APIs
- **Client-side-only validation**: Input validated in JS but not on the server
- **Tokens stored insecurely**: Auth tokens in localStorage, credentials in client-side code
- **CORS configured too permissively**: `Access-Control-Allow-Origin: *` on authenticated endpoints
- **Rate limiting gaps**: Rate limits on login but not on password reset or API endpoints
- **Permission checks that don't check**: Middleware that's defined but not applied, or that
  checks role but not resource ownership

### 2.7 Performance Traps That Affect Behavior

Performance issues that manifest as functional bugs to users:

- **Unbounded queries**: `SELECT * FROM table` without LIMIT, list endpoints that return everything
- **N+1 queries**: Loops that make a database/API call per item
- **Missing pagination**: Endpoints that work fine with 10 items but crash/timeout with 10,000
- **Memory leaks via subscriptions**: Event listeners, websocket connections, or intervals that
  are set up but never torn down
- **Blocking operations on main thread**: Heavy computation that freezes the UI

---

## Phase 3: Deep Investigation

This is where the real value is. For every suspicious pattern found in Phase 2, don't just
flag it — **trace the full execution path** and understand exactly what a real user would
experience.

### How to Deep Dive

For each finding:

1. **Start from the user's perspective**: What action triggers this code? A button tap?
   A page load? A background sync?
2. **Trace the full path**: Follow the code from the user action through every layer
   (UI → handler → service → API → database → response → UI update)
3. **Identify the exact failure mode**: What does the user see when this goes wrong?
   Not "an error occurs" but "the save button appears to work but the data isn't persisted,
   so when the user returns to the screen, their changes are gone"
4. **Assess real-world impact**: How often would a real user hit this? Is it an edge case or
   a mainline flow? Does it corrupt data or just look wrong?
5. **Check if there's monitoring**: Would anyone know this is happening? Is there alerting,
   logging, or error tracking that would catch it?

### Verify Claimed Capabilities

Go through the list from Phase 1.3 and verify each claimed capability:

- Does the feature actually work end-to-end?
- Is it complete or only partially implemented?
- Are there edge cases in the feature that silently fail?
- Is the feature tested? If so, do the tests actually exercise the real behavior?

### Simulate Critical User Journeys

For the 5-10 core flows identified in Phase 1.2, mentally walk through each one:

- What happens on first use? (empty state, no data, no permissions)
- What happens on the happy path?
- What happens when things go wrong? (network failure, invalid data, expired session)
- What happens with concurrent users?
- What happens at scale? (1000 items instead of 10)

---

## Phase 4: Report

Generate `production-gap-audit.md` in the project root. If a previous audit exists,
number the new one (e.g., `production-gap-audit-2.md`, `production-gap-audit-3.md`).

### Report Structure

```markdown
# Production Gap Audit
**Date**: [date]
**Codebase**: [project name and description]
**Scope**: [what was audited]

## Executive Summary
[2-3 sentences: overall production readiness assessment and the most critical findings]

## Critical Findings
[Issues that will cause data loss, security breaches, or complete feature failure for users]

### [Finding Title]
- **Location**: `file:line` (and related files)
- **What happens**: [describe from the user's perspective]
- **Why it matters**: [impact — data loss? security? broken flow?]
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
[Brief description of what was examined and how]
```

### Severity Classification

- **Critical**: Real users will lose data, face security exposure, or find core features
  completely non-functional. Needs fixing before any release.
- **High**: Common user flows are degraded. Features work sometimes but fail in realistic
  scenarios. Users will notice and be frustrated.
- **Medium**: Edge cases fail, non-critical features are incomplete, or the UX degrades under
  specific (but realistic) conditions.
- **Low**: Unlikely to affect most users today, but represents latent risk that could escalate.

### Writing Findings

Each finding must answer these questions:
1. What would the user be trying to do?
2. What would they experience instead?
3. Where exactly in the code does this happen?
4. How confident are you this is a real issue (vs. a pattern you might be misreading)?

Be specific. "Error handling could be improved" is useless. "When `syncService.sync()` fails
due to network timeout, the catch block on line 142 of `syncService.ts` logs the error but
doesn't update the UI, so the user sees 'Syncing...' forever with no way to retry" is useful.

---

## Execution Strategy

You will be investigating a lot of code. Use subagents aggressively to parallelize work:

- Spawn separate agents for each major area (frontend, backend, integrations, etc.)
- Spawn agents for each category in Phase 2 (silent failures, incomplete implementations, etc.)
- Use the Explore agent type for initial reconnaissance
- Use general-purpose agents for deep dives that require reading multiple files and tracing paths

After all agents report back, synthesize findings, deduplicate, verify the most critical ones
yourself by reading the actual code, and produce the final report.

When investigating deep, always read the actual code. Don't infer from file names or function
signatures alone. The whole point of this audit is to catch things that *look* right but aren't.
