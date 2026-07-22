# Better E2E/unit testing for the relay/catcher watcher scripts

## Problem

On 2026-07-22, `relay/idle_shutdown.ts` powered the relay VM off while a real player
(stan/"Kinsfather") was actively connected and chatting through it, killing his session
twice in one evening. Root cause: its established-connection check could return `0`
despite a genuinely active TCP session (an untested `dport` filter clause, plus a
`catch { return 0 }` that silently conflated "check failed" with "confirmed idle").
Confirmed only through live GCE forensics (`gcloud compute instances list`, journal
correlation on mc-home), not by any existing test.

The existing E2E harness (`mliem-landing/src/lib/mc-status/transfer-test/transfer-test.js`)
already proves the transfer flow works and holds the post-transfer connection open for
`MC_HOLD_MS` (default 60s), but 60s never crosses the relay's 300s idle-shutdown
threshold, so this exact bug class had no way to surface under test. Separately, the two
watcher scripts most responsible for connection-activity decisions
(`relay/idle_shutdown.ts`, `catcher/catcher_wake_watcher.ts`) have zero test coverage of
their own: their `ss`-output parsing and threshold logic can only be exercised by running
them against a live VM for minutes at a time.

## Goals

1. Close the specific gap that let tonight's bug through: prove a connection survives
   past the 300s idle threshold, end to end, against the real relay.
2. Add fast (sub-second), no-live-infra-required unit tests for the two watcher
   scripts' decision logic, so a future regression in the same class (a bad filter, a
   parsing gotcha, a fail-open bug) is caught in CI-speed feedback instead of requiring
   live forensics again.

## Non-goals

- No continuous production canary/monitoring (considered, deferred; the existing
  `mc-status-cron.yml` 5-minute ping is unrelated and untouched).
- No CI wiring for either the long-hold E2E test or the new unit tests. Both stay
  manually invoked for now.
- No behavior change to the watcher scripts beyond what was already fixed and deployed
  earlier this session (this spec is about *testability* of that logic, not further
  logic changes).

## Design

### 1. `relay/idle_shutdown.ts`: extract 4 pure functions

Refactor the file so its top-level side-effecting script body (the `ss`/`poweroff`/DNS
calls) only runs `if (import.meta.main)`, unchanged in behavior, while the decision
logic becomes independently callable and exported:

```ts
export function parseEstablishedCount(ssOutput: string): number
export function isBedrockActive(now: number, lastActivity: number, maxAgeSeconds: number): boolean
export function shouldStayAwake(tcpCount: number | null, bedrockActive: boolean): boolean
export function shouldPowerOff(elapsedSeconds: number, idleLimitSeconds: number): boolean
```

`shouldStayAwake` is tonight's exact fix expressed as a pure truth table:
`tcpCount === null || tcpCount > 0 || bedrockActive`. The `import.meta.main` guard calls
these with real `ss` output/timestamps exactly as today; nothing about the deployed
behavior changes.

**Test file**: `relay/idle_shutdown.test.ts`, run via `bun test`.

Test cases:
- `parseEstablishedCount`: empty output → 0; one `ESTAB` line → 1; multiple → N;
  header-only output (no `ESTAB` line) → 0; garbage input → 0, no throw.
- `shouldStayAwake`: `(0, false)` → false; `(1, false)` → true; `(0, true)` → true;
  **`(null, false)` → true** (the direct regression test for tonight's bug, must never
  flip back to false).
- `shouldPowerOff`: elapsed just under limit → false; elapsed exactly at limit → true;
  elapsed well over → true.

### 2. `catcher/catcher_wake_watcher.ts`: extract 2 pure functions

```ts
export function parseConnections(ssOutput: string): Connection[]
export function shouldTriggerWake(streak: number, grownBytes: number, confirmCount: number, minActivityBytes: number): boolean
```

`parseConnections` is the existing two-line (summary + indented byte-counter info line)
parser, made independently testable. Same `import.meta.main` guard pattern as above.

**Test file**: `catcher/catcher_wake_watcher.test.ts`, run via `bun test`.

Test cases:
- `parseConnections`: a real captured `ss -tin` sample (summary line + `bytes_acked`/
  `bytes_received` info line) parses to the correct `{conn, totalBytes}`; multiple
  connections in one output parse independently; **`ss`'s own header line
  ("State Recv-Q Send-Q ...") must not be miscounted as a connection** (direct
  regression test for the header-line parsing gotcha already documented as a past bug
  in this exact file); a connection with only one of `bytes_acked`/`bytes_received`
  present sums correctly with the missing one defaulted to 0; empty output → `[]`.
- `shouldTriggerWake`: streak below `confirmCount` → false regardless of bytes; streak
  at/above `confirmCount` but bytes under `minActivityBytes` → false (the "port
  scanner" case); streak at/above `confirmCount` and bytes at/above threshold → true.

Both test files are plain `bun test` files, no new `package.json`/dependencies, no
mocking library (the functions under test take plain strings/numbers, no need to fake
`execFileSync`). Run manually via `cd infra && bun test` for now; no CI wiring.

### 3. Long-hold E2E: packaging only, no new test logic

`transfer-test.js` already reads `MC_HOLD_MS` from the environment and already fails the
run on any `disconnect`/`kick_disconnect`/`error` event during the hold window, which is
exactly what a relay self-poweroff produces (confirmed live tonight: Kinsfather's client
got a real `Disconnected` event mid-session). No code change needed to the hold
mechanism itself.

Add to `transfer-test/package.json`:
```json
"scripts": {
  "test:long-hold": "MC_HOLD_MS=330000 node transfer-test.js"
}
```
330000ms (5:30) gives 30s of margin past the 300s idle limit, past the ~20s poll
granularity of `mc-relay-idle-check.timer`. Add a short README note explaining this
mode exists specifically to catch relay-idle-shutdown-under-active-load regressions,
and should be run by hand when investigating a DC report or before relay/catcher
changes, using a real registered account (`MC_USERNAME`/`MC_AUTHME_PASSWORD`), not a
throwaway one.

## Testing

- `bun test` in `infra/` covers the two watcher scripts' decision logic (this spec's
  main deliverable).
- `npm run test:long-hold` in `transfer-test/` is the manual regression check against
  the real relay for the idle-shutdown-under-active-load bug class specifically.
- Existing default `npm test`-equivalent invocation (`node transfer-test.js` with the
  60s default hold) is unchanged and still the fast smoke check for the transfer flow
  itself.
