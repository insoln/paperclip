## Draft PR update / review reply

Thanks to @getmoneyjj for the original Stop settlement, ambiguous-create retry, and SSE resume implementation in #17963. This follow-up preserves the author's commit and closes the review findings.

### Design

- `GET /v1/capabilities` is required before the first create. Ambiguous creates are retried only when that response proves durable `runs_idempotency` with numeric retention of at least 60 seconds. The gateway is checked again before every retry; any missing, malformed, changed, or failed capability response fails closed without a retry.
- Create capability probes, POSTs, retry revalidations, retry logging, and retry waits share one monotonic wall-clock deadline set before the initial probe: `min(45 seconds, retention − 10 seconds)`. The advertised-retention cap is calculated from the same start instant, so probe time is charged to the operation rather than starting a new POST budget. All request abort signals fire at that deadline; when it is exhausted, retries fail closed.
- Stop settlement continues to use one 60-second deadline for stop and status fetches. Logging in `stopRun`, final-status retry handling, and settlement is deadline-bounded/best-effort, so a hung callback cannot extend cancellation settlement.
- SSE has an explicit at-least-once log delivery contract. The cursor advances only after the log callback resolves. If a callback persisted an event and then rejects, the event is replayed; output aggregation deduplicates by SSE sequence id, but consumers of log delivery must be idempotent by run/event id. The adapter does not claim exactly-once external persistence.

### Tests

- The fake gateway accepts and executes the first create, drops that response as `ECONNRESET`, then returns the same run for the idempotent replay. It asserts one execution.
- Tests cover pre-create capability probing, retry revalidation, non-retry on unsupported/unavailable/malformed capability, and a 44-second stalled probe followed by a stalled POST that is aborted at the single 45-second operation deadline; they also cover stalled cancellation stop fetch, transient final-status retry, rejected event application, and partial-persist-then-reject SSE delivery.
- Red/green evidence: the single-budget test fails against `260b09ec8` because its probe consumes 44 seconds and it then grants the POST a new 45-second deadline; it passes with the one-start-instant deadline. The added accepted-then-reset test fails against `857095c0` because its first request is `POST /v1/runs`, while the test requires `GET /v1/capabilities` first. The partial-persist SSE test fails against `857095c0` because the replay appends `Hello` twice. Deadline/callback behavior was covered by extending the existing cancellation-deadline test; the old implementation did not bound the awaited log callbacks.

### Verification

- `pnpm --filter @paperclipai/hermes-paperclip-adapter test -- execute.test.ts` — 116 passed.
- `pnpm --filter @paperclipai/hermes-paperclip-adapter test` — 189 passed across 10 files.
- `pnpm --filter @paperclipai/hermes-paperclip-adapter typecheck` — passed.
- `pnpm --filter @paperclipai/hermes-paperclip-adapter build` — passed.
- Repository-rule determination: CONTRIBUTING.md says, “Do not submit a PR that moves a code file from less than 1,000 lines of code to more than 1,000 lines of code.” It also says, “Do not submit a PR that includes code comments.” `execute.ts` was already 1,204 lines at the author's parent (`13c132b9e`) and was 1,319 before this follow-up, so this work did not cross the stated 1,000-line threshold. The prior two follow-up commits changed the existing file and added no code comments; this fix also adds no code comments. No extraction was made because the quoted LOC rule is threshold-based, not a blanket ban on modifying an existing >1,000-line file.

Commands ran under Node 22.22.1 although the repository requests Node >=24.11.0.

No remote branch, PR, or comment was created or updated.
