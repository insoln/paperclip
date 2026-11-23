## Draft PR update / review reply

Thanks to @getmoneyjj for the original Stop settlement, ambiguous-create retry, and SSE resume implementation in #17963. This follow-up preserves the author's commit and closes the review findings.

### Design

- `GET /v1/capabilities` is required before the first create. Ambiguous creates are retried only when that response proves durable `runs_idempotency` with numeric retention of at least 60 seconds. The gateway is checked again before every retry; any missing, malformed, changed, or failed capability response fails closed without a retry.
- Create capability probes, POSTs, retry waits, and retry logging share a monotonic wall-clock budget: at most 45 seconds and at least 10 seconds shorter than advertised idempotency retention. Their abort signals fire at the budget deadline.
- Stop settlement continues to use one 60-second deadline for stop and status fetches. Logging in `stopRun`, final-status retry handling, and settlement is deadline-bounded/best-effort, so a hung callback cannot extend cancellation settlement.
- SSE has an explicit at-least-once log delivery contract. The cursor advances only after the log callback resolves. If a callback persisted an event and then rejects, the event is replayed; output aggregation deduplicates by SSE sequence id, but consumers of log delivery must be idempotent by run/event id. The adapter does not claim exactly-once external persistence.

### Tests

- The fake gateway accepts and executes the first create, drops that response as `ECONNRESET`, then returns the same run for the idempotent replay. It asserts one execution.
- Tests cover pre-create capability probing, retry revalidation, non-retry on unsupported/unavailable/malformed capability, stalled cancellation stop fetch, transient final-status retry, rejected event application, and partial-persist-then-reject SSE delivery.
- Red/green evidence: the added accepted-then-reset test fails against `857095c0` because its first request is `POST /v1/runs`, while the test requires `GET /v1/capabilities` first. The partial-persist SSE test fails against `857095c0` because the replay appends `Hello` twice. Deadline/callback behavior was covered by extending the existing cancellation-deadline test; the old implementation did not bound the awaited log callbacks.

### Verification

- `pnpm --filter @paperclipai/hermes-paperclip-adapter test -- execute.test.ts` — 115 passed.
- `pnpm --filter @paperclipai/hermes-paperclip-adapter typecheck` — passed.
- `pnpm --filter @paperclipai/hermes-paperclip-adapter build` — passed.
- Lint was not run: this checkout has no ESLint 10 flat config. No lint result is claimed.

Commands ran under Node 22.22.1 although the repository requests Node >=24.11.0.

No remote branch, PR, or comment was created or updated.
