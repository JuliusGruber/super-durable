# Inngest as a substrate for a long-lived, event-driven LLM agent system (state: 14 November 2026)

Research notes for the comparison with pi-durable. Source labels used below:
**[docs]** official Inngest documentation or pricing page, **[api]** official
REST API OpenAPI spec, **[source]** GitHub repository contents or release
metadata read directly, **[blog]** Inngest's own blog, **[3p]** third party.
"Unconfirmed" marks anything not verified in a primary source. Nothing was
built or run. Docs pages were read through a summarising fetcher, so wording is
paraphrased unless quoted.

Versions observed on 14 November 2026 **[source]**: Inngest server/CLI
`v1.53.0` (12 Nov 2026), TypeScript SDK `inngest@4.26.2` (29 Oct 2026),
`@inngest/agent-kit@0.13.2` (13 Nov 2025), `@inngest/use-agent@0.4.0`
(1 Oct 2025).

## 1. Execution model: what is durable, constraints, where LLM and tool calls live, retries and side effects

### Takeaway
An Inngest function is an event-triggered handler whose `step.*` results are memoised in Inngest's state store; the handler is replayed from the top and completed steps return saved results. Durability is per step result (at-least-once execution of each step), not of in-memory state, and all LLM and tool calls must sit inside steps (`step.run`, `step.ai.infer`, `step.ai.wrap`).

### Cited Findings
- [docs] Each step result is saved once as a checkpoint; on continuation the handler replays from the top and any step with a saved result returns it without re-running its callback. Stored state: each completed step's result keyed by a hash of the step ID (plus index), serialised errors and attempt history, and the run trace. — [How functions are executed](https://www.inngest.com/docs/learn/how-functions-are-executed)
- [docs] Two execution modes. "Standard orchestration": each step runs in a separate request to the app; after capturing a step result the SDK stops the handler and Inngest calls it again with the event and saved state. "Checkpointing" (default in TypeScript SDK v4): steps run back to back in the same handler execution and results are saved as checkpoints without restarting the handler; after a wait Inngest injects saved state into a new execution. TypeScript v4 falls back to standard orchestration when a step retries or steps run in parallel. — [How functions are executed](https://www.inngest.com/docs/learn/how-functions-are-executed)
- [docs] Checkpointing options: `maxRuntime` (default 0 = unlimited), `bufferedSteps` (default 1 = no buffering; buffering risks losing un-checkpointed steps on ungraceful termination), `maxInterval`. Requires TypeScript SDK 3.51.0+, default on in v4. Supported on the Dev Server, self-hosted Inngest and Inngest Cloud. — [Checkpointing](https://www.inngest.com/docs-markdown/durable-execution/guides-and-advanced/checkpointing)
- [docs] Code outside steps runs again on every replay, so non-deterministic work (DB or API calls) belongs inside `step.run()`. — [How functions are executed](https://www.inngest.com/docs/learn/how-functions-are-executed)
- [docs] Retries: a thrown error stops the handler; Inngest records the attempt and retries if attempts remain; earlier steps return saved results and only the failed step re-runs; when attempts are exhausted the error is thrown into the handler, which can catch it. — [How functions are executed](https://www.inngest.com/docs/learn/how-functions-are-executed)
- [docs] Side effects are not exactly-once: "A step can succeed externally before its result is saved", so side effects should be idempotent; event and function dedup do not prevent a repeated external call, and the docs advise sending a stable operation key to the external service. — [How functions are executed](https://www.inngest.com/docs/learn/how-functions-are-executed); [Idempotency](https://www.inngest.com/docs/guides/handling-idempotency)
- [docs] Step results must be JSON-serialisable (stated for Python; `step.invoke` return values must be JSON-serialisable in all SDKs). — [Agent tool loops](https://www.inngest.com/docs-markdown/durable-execution/durable-agents/agent-tool-loops); [step.invoke](https://www.inngest.com/docs-markdown/durable-execution/primitives/step-invoke)
- [docs] Two transports. `serve` exposes functions over HTTP. `connect` opens an outbound persistent WebSocket from the app to Inngest (TypeScript SDK 3.34.1+, Node 22.4.0+); needs a long-running server (not Lambda/Vercel); steps are not bound by platform HTTP timeouts; functions sync automatically on connect; on SIGTERM/SIGINT in-flight steps complete; `maxWorkerConcurrency` caps steps per worker. Supported against self-hosted Inngest via the gateway on port 8289. — [Connect](https://www.inngest.com/docs-markdown/durable-execution/deploying-functions/connect)
- [docs] `step.ai.infer()` offloads the inference HTTP request to Inngest's infrastructure; the function is not executing while the request is pending and restarts with the result. Providers: OpenAI (and OpenAI-compatible), Gemini, Anthropic, Grok, Azure OpenAI. Streaming is "not currently supported" ("coming soon"). Inngest states it never logs or stores API keys or auth headers. — [step.ai](https://www.inngest.com/docs/features/inngest-functions/steps-workflows/step-ai-orchestration)
- [docs] `step.ai.wrap()` (TypeScript only) wraps an existing OpenAI, Anthropic or Vercel AI SDK call as a step for observability; the call runs in the app's own process; arguments must be JSON-serialisable to allow editing and re-running prompts from the dashboard. Tool calling is not covered by `step.ai`; the page points to AgentKit and the agent tool loop pattern. — [step.ai](https://www.inngest.com/docs/features/inngest-functions/steps-workflows/step-ai-orchestration)
- [docs] The documented agent pattern: a ReAct loop in application code; the LLM call is a `step.run` named "think"; each tool call is its own step `tool-<name>`; parallel tools via `Promise.all` / `group.parallel` with indexed step IDs. "The agent loop runs in your app, not inside a VM." — [Agent tool loops](https://www.inngest.com/docs-markdown/durable-execution/durable-agents/agent-tool-loops); [Durable agents](https://www.inngest.com/docs-markdown/durable-execution/durable-agents)
- [docs] Step timeout: up to 2 hours, subject to the function host's shorter timeout. — [Usage limits](https://www.inngest.com/docs/usage-limits/inngest)

### Inferences
- An LLM call inside `step.run` is atomic from Inngest's point of view: no result or partial stream is persisted until the step returns, and a crash mid-call re-runs the whole call (and re-bills the tokens).
- Because every tool is a step with at-least-once semantics, remote-command tools need idempotency keys or reconciliation, the same design burden as pi-durable's non-`replay: safe` tools. Inngest has no equivalent of "return an interrupted result to the model instead of re-running": the default is re-run until retries are exhausted.
- With `connect` on a long-lived Node process, the HTTP-timeout constraint disappears, which suits a single-developer self-hosted worker.

### Gaps
- What happens when two steps share an ID, and the exact determinism rules for changing function code while runs are active (the Versioning guide was not read).
- Default retry count: an AgentKit page says "4 retries with exponential backoff"; the core docs page read did not state the default. Treat as likely but unconfirmed for SDK v4.
- Whether `step.ai.infer` is available on self-hosted Inngest: unconfirmed.

## 2. Limits and what they mean for a months-long session; where the transcript lives

### Takeaway
A single run is capped at 1,000 steps, 32 MiB of total state, 4 MiB per step output, and 30/90/366 days of wall-clock life depending on Cloud plan, so a months-long Router cannot be one run; the documented shape is one short run per message or turn over a transcript kept in the application's own store. Inngest provides no application state or transcript store; "Sessions" are only an observability grouping tag.

### Cited Findings
- [docs] Maximum function run length including sleeps: Free 30 days, Pro 90 days, Business 366 days, Enterprise custom. Time in `step.waitForEvent()` counts toward the run-length limit. — [Usage limits](https://www.inngest.com/docs/usage-limits/inngest); [Pricing](https://www.inngest.com/pricing)
- [docs] Steps per function run: 1,000. Step output: 4 MiB per step. Function run state: 32 MiB across event data, step data, return data and metadata. Sleep: up to 1 year, within the plan's run-length maximum. — [Usage limits](https://www.inngest.com/docs/usage-limits/inngest)
- [docs] Event payload: 256 KiB (Free), 3 MiB (Pro, Business). Event name: 256 characters. Up to 5,000 events per `inngest.send()` / `step.sendEvent()` request. Batch hard cap 10 MiB. — [Usage limits](https://www.inngest.com/docs/usage-limits/inngest)
- [docs] `waitForEvent` maximum timeout is not stated separately; it is bounded by the run-length limit. — [Usage limits](https://www.inngest.com/docs/usage-limits/inngest); [waitForEvent](https://www.inngest.com/docs/features/inngest-functions/steps-workflows/wait-for-event)
- [docs] `step.invoke` is suggested for "splitting work that would exceed the 1,000-step limit". — [step.invoke](https://www.inngest.com/docs-markdown/durable-execution/primitives/step-invoke)
- [docs] On transcript storage: a resumed run rebuilds history from saved step results ("the history lives in your code; the steps are what make it durable"); for chat the page recommends "a separate memory store keyed by conversation or user ID, loaded and saved in steps", storing summaries or facts and not only raw transcripts. Each run is one agent function execution; related runs are tied together by a session tag. — [Durable agents](https://www.inngest.com/docs-markdown/durable-execution/durable-agents)
- [docs] Sessions are a grouping tag set in event `meta.sessions` (`{ key: id }`, up to five per event); the dashboard (AI → Sessions) shows the runs together. They do not change triggering and the page describes no stored state or transcript. Propagation to emitted events needs TypeScript SDK 4.18.0+. — [Sessions](https://www.inngest.com/docs-markdown/agent-evals/sessions)
- [docs] Context management is described as application techniques (pruning, compaction by a summarising step, references, isolation in sub-agents), not as a built-in feature. — [Durable agents](https://www.inngest.com/docs-markdown/durable-execution/durable-agents)
- [docs] Trace and log history retention on Cloud: 24 hours (Free), 7 days (Pro), 14 days (Business). — [Usage limits](https://www.inngest.com/docs/usage-limits/inngest)
- [docs] Self-hosted: Postgres tables are not cleaned up automatically; old events, runs and traces must be pruned manually. — [Self-hosting](https://www.inngest.com/docs/self-hosting)
- [source] Inngest's own example agent (Utah) stores sessions as JSONL files (`src/lib/session.ts`), has a file-based memory (`memory.ts`) and LLM-powered compaction (`compaction.ts`), all application code. — [inngest/utah README](https://github.com/inngest/utah)

### Inferences
- A think/act loop spends at least two steps per tool round (one LLM step, one or more tool steps), so 1,000 steps is roughly a few hundred tool rounds; a Worker doing a long coding task can hit this and must chain runs (`step.invoke` or a continuation event), reloading the transcript each time.
- Each LLM response and tool result is stored as step output, so a run's 32 MiB state also bounds a long loop; the transcript effectively exists twice (step outputs in Inngest, and the application's store).
- The Router "session" on Inngest is therefore a key (`sessionKey`) plus an application database, not an Inngest object. Compared with pi-durable, where the conversation, inbox and checkpoints are one persisted object, Inngest supplies the run engine and the application supplies the session.
- On Cloud, short trace retention (24 hours on Free) means run history cannot serve as the transcript record.

### Gaps
- Whether the 1,000-step, 32 MiB and run-length limits are enforced, configurable or absent on a self-hosted server: unconfirmed (the limits page describes Cloud plans plus "platform limits").
- Session retention period and whether a Sessions query API exists: not documented on the page read.

## 3. Input during a running function

### Takeaway
A run can receive new input only at a step boundary where it is explicitly parked in `step.waitForEvent` or `step.waitForSignal`; nothing reaches a run while an LLM step is in flight, and events sent before the wait starts are not matched. Native per-key batching (`batchEvents`) exists but is time/size-window based, cannot be combined with singleton, debounce, `cancelOn`, idempotency, rate limiting or priority, and is capped at 5 events / 30 s on the Free plan; there is no "queue everything that arrived during the turn and deliver it as the next turn" primitive.

### Cited Findings
- [docs] `step.waitForEvent(id, { event, timeout, match | if })` suspends the run with no compute held; returns the event or `null` on timeout. "The wait only sees events sent after it begins listening"; a matching event sent earlier does not resume it. The docs' advice: register the wait first or check external state before waiting. — [waitForEvent](https://www.inngest.com/docs/features/inngest-functions/steps-workflows/wait-for-event)
- [docs] `step.waitForSignal(id, { signal, timeout, onConflict })` resumes exactly one run addressed by a unique signal name; sent with `inngest.sendSignal()` / `step.sendSignal()`; `onConflict: "fail" | "replace"`. Delivery is described as transactional (versus eventually consistent events). "Signals are currently experimental." — [step.waitForSignal](https://www.inngest.com/docs-markdown/durable-execution/primitives/step-waitforsignal)
- [api] `POST /v1/signals` resumes a function waiting on a signal and returns the resumed `run_id`. — [REST API v1 spec](https://api-docs.inngest.com/api-specs/v1.json)
- [docs] `batchEvents: { maxSize, timeout, key, if }`: the first matching event opens a batch; a run starts when the batch reaches `maxSize`, the timeout expires or the batch reaches 10 MiB; `key` gives one batch per distinct value; the handler reads `events`. Plan limits: Free 5 events / 30 s, Pro 100 / 5 min, Business 500 / 5 min. — [Batching](https://www.inngest.com/docs/guides/batching)
- [docs] Batching cannot be used on the same function with idempotency, rate limiting, cancellation events, priority or debounce. A concurrency key on a batched function is evaluated on the batch's first event. — [Batching](https://www.inngest.com/docs/guides/batching)
- [docs] Singleton: `singleton: { key, mode }`, run-level, one active run per key. `skip` discards the new event's run; `cancel` cancels the active run and starts a new one (an executing `step.run` finishes first; in rapid bursts cancel mode may skip some new runs). "Singleton cannot be combined with batching. Function registration fails if both are set." — [Singleton](https://www.inngest.com/docs/guides/singleton)
- [docs] Concurrency: `concurrency: { limit, key, scope }` caps executing steps, not runs; waiting steps (`sleep`, `waitForEvent`, `invoke`) hold no slot; with `limit: 1` per key "this does not guarantee that entire [runs] run one after another". Queued work within one function and key is best-effort FIFO. Up to two concurrency constraints per function. — [Concurrency](https://www.inngest.com/docs-markdown/durable-execution/flow-control/concurrency)
- [docs] Debounce: `debounce: { period (1s to 7d), key, timeout }`; only the last event of a burst is delivered, not an array; cannot be combined with batching. — [Debounce](https://www.inngest.com/docs-markdown/durable-execution/flow-control/debounce)
- [docs] Cancellation "stops active runs between steps"; a step that is already executing finishes, then the run is marked cancelled. — [Cancellation](https://www.inngest.com/docs/features/inngest-functions/cancellation)
- [source] Inngest's Utah agent handles a new message during a turn with `singleton: { key: "event.data.sessionKey", mode: "cancel" }`: the in-flight run for that chat is cancelled and a new run starts, after the user message was appended to the session file in a first step. — [utah/src/functions/message.ts](https://github.com/inngest/utah/blob/main/src/functions/message.ts)
- [docs] The durable-agents page does not cover steering an agent mid-run; the closest feature it names is cancellation. — [Durable agents](https://www.inngest.com/docs-markdown/durable-execution/durable-agents)

### Inferences
- The article's Router behaviour ("Triggers arriving during a turn queue up and are handled together in the next turn") has no single native mapping. Options an application could assemble: (a) `batchEvents` keyed by session with a short timeout, accepting that batches are formed by time window, not by "previous turn finished", and that two batch runs for one key can overlap between steps because singleton is unavailable and concurrency limits steps, not runs; (b) an application inbox table: every event is appended by an ingest function, and one drain function per session (singleton `skip` or a concurrency key of 1) reads all pending rows at the start of each turn and re-triggers itself if more arrived; (c) a long run that loops on `waitForEvent`, which loses events sent while an LLM step is executing unless they are also written to an inbox. Option (b) is the only one that gives the article's semantics, and it is application code over an external store.
- pi-durable's persisted inbox with `steeringMode: all` / `followUpMode: all` is a built-in equivalent of option (b); on Inngest it must be built.
- Mid-turn steering (inserting input between tool calls of a running turn) is possible only if the application loop checks an inbox between steps; Inngest itself offers cancel-and-restart.

### Gaps
- Docs do not say whether events arriving while a batch run executes start the next batch immediately (inferred yes, since batching is independent of run state): unconfirmed.
- Whether a signal sent before its wait is registered is retained: not stated.
- Behaviour of debounce when a run for the key is already active: not stated.

## 4. Workers

### Takeaway
Workers map to separate functions started either synchronously with `step.invoke` (parent parks until the child returns) or asynchronously with `step.sendEvent` (fire and continue); asynchronous report-back is an event the application must route into the parent's session, and Inngest's own example delivers async results to the user, not into the parent conversation.

### Cited Findings
- [docs] `step.invoke` starts another function and waits for its return value; the child has its own steps and retries; `timeout` option; if the child exhausts retries, is not found or times out, `step.invoke` throws a `NonRetriableError` in the parent, which can catch it. "A timed-out child keeps running." Cross-app invocation via `referenceFunction({ appId, functionId })`. — [step.invoke](https://www.inngest.com/docs-markdown/durable-execution/primitives/step-invoke)
- [docs] `step.sendEvent()` "starts one and continues immediately, for long tasks"; "a failed sub-agent doesn't lose the parent's progress"; "Leave delegation tools out of the sub-agent's tool set so agents can't spawn each other without limit." — [Durable agents](https://www.inngest.com/docs-markdown/durable-execution/durable-agents)
- [blog] Dan Farrelly, "Three sub-agent patterns you need for your agentic system" (11 March 2026): sync (`step.invoke`), async (`agent.subagent.spawn` event with `async: true`; the sub-agent function emits `agent.reply.ready` when done and a send-reply function posts to the originating channel), scheduled (same event with a future `ts`). Async results go to the user, not back into the parent; a "handle" for the parent to query output is described as a planned next iteration. Delegation is depth 1. Scheduled tasks are "hard to cancel or modify once queued". Async failures are caught by a global `inngest/function.failed` handler. — [Three patterns](https://www.inngest.com/blog/three-patterns-you-need-for-agentic-systems)
- [docs] Stopping a Worker: `cancelOn` (event-matched cancellation defined on the function), timeouts, REST `DELETE /v1/runs/{runID}`, and bulk cancellation by function and time range; an executing step finishes before the cancel takes effect. Every cancelled run emits `inngest/function.cancelled` with `function_id` and `run_id`. — [Cancellation](https://www.inngest.com/docs/features/inngest-functions/cancellation); [REST API v1 spec](https://api-docs.inngest.com/api-specs/v1.json)
- [docs] Per-key concurrency and throttling can bound how many Worker steps execute at once (plan limit: 5 concurrent steps on Free, 100 on Pro). — [Concurrency](https://www.inngest.com/docs-markdown/durable-execution/flow-control/concurrency); [Pricing](https://www.inngest.com/pricing)
- [docs] Sandboxes: isolated microVMs driven from steps (`step.sandbox.python`, `step.sandbox.typescript`, `sb.$` to run a command as one step), pausable, snapshot/clone; "Sandboxes are in beta and you own cleanup". Pricing page: maximum sandbox duration 24 hours, 100 concurrent sandboxes on Free and Pro. — [Sandboxes overview](https://www.inngest.com/docs-markdown/sandboxes/overview); [Durable agents](https://www.inngest.com/docs-markdown/durable-execution/durable-agents); [Pricing](https://www.inngest.com/pricing)

### Inferences
- The article's `start()` / `stop()` tools map naturally: `start()` = `step.sendEvent` of a `worker.start` event carrying a task ID; `stop()` = an event matched by the Worker function's `cancelOn`, or a REST cancel by run ID. This is less glue than pi-durable's anchor-plus-reporter pattern, because fire-and-forget functions and event-matched cancellation are native.
- Report-back is the same amount of glue on both: the Worker emits a `worker.report` event; code must turn it into Router input (on Inngest, an event to the Router's function, subject to the inbox problem in section 3).
- Worker isolation is at the level of a separate run plus whatever environment the tools use; steering a Worker (as opposed to cancelling it) again needs an application inbox checked between steps.
- Tools that run commands on remote machines are ordinary `step.run` code (for example an SSH call) in the worker process; Inngest Sandboxes are hosted microVMs, not bring-your-own machines.

### Gaps
- Whether cancelling a parent cancels a child started with `step.invoke`: not stated.
- Whether Sandboxes are available with self-hosted Inngest and their price: not stated on the pages read (unconfirmed).
- No native concept of the article's endpoint daemon or offline queueing for intermittently connected machines was found (a `connect` worker running on the endpoint would receive queued steps when it reconnects; this is an inference, not a documented pattern).

## 5. Watchdog

### Takeaway
A cron-triggered function plus the REST API covers most of the Watchdog's mechanics natively: list a function's runs by status, read a run, cancel a run, bulk-cancel, replay, pause functions, and react to `inngest/function.failed` / `inngest/function.cancelled` system events or per-function `onFailure`. Judging "is this Worker making progress" needs the Worker's transcript, which lives in the application store, and "steer" has no native mechanism.

### Cited Findings
- [docs] Cron triggers use five-field cron with optional `TZ=` prefix (UTC default; no DST correction); each firing delivers an `inngest/scheduled.timer` event; optional `jitter` of 1 s to 5 min. On the Free plan a function is paused automatically after 20 consecutive failures. — [Schedules and delayed starts](https://www.inngest.com/docs-markdown/durable-execution/guides-and-advanced/events-and-triggers/schedules-and-delayed-starts)
- [api] REST v1 (base `https://api.inngest.com/v1`, Dev Server `http://localhost:8288`; auth `Authorization: Bearer <signing key>`): `GET /v1/runs/{runID}` (status `Running | Completed | Failed | Cancelled`, `output`, `event_id`, `original_run_id`), `DELETE /v1/runs/{runID}` (cancel), `GET /v1/runs/{runID}/jobs` (queue position and attempt; cached 5 s), `GET /v1/events/{id}/runs`, `POST /v1/cancellations` (bulk: `app_id`, `function_id`, `started_before`, optional `started_after`, `if`), `POST /v1/signals`, webhook CRUD. — [REST API v1 spec](https://api-docs.inngest.com/api-specs/v1.json)
- [api] REST v2 (described as stable but still in development): `GET /apps/{appId}/functions/{functionId}/runs` with filters `status`, time range, CEL `query`, and `include=output`; `POST .../cancellations`; `POST .../replays` (asynchronous replay of matching runs); `POST .../invoke` (sync or async, with idempotency key); pause/resume of functions; `GET /events`; `POST /insights/query` (SQL over events). Only the first ~40% of the spec was read. — [REST API v2 spec](https://api-docs.inngest.com/api-specs/v2.json); [API docs home](https://api-docs.inngest.com)
- [docs] `inngest/function.failed` is sent whenever a function fails (after retries); payload has `error` (message, name, stack), the original `event`, `function_id`, `run_id`. `onFailure` is the per-function handler for this event. — [inngest/function.failed](https://www.inngest.com/docs-markdown/reference/system-events/inngest-function-failed)
- [docs] Bulk cancellation does not stop new runs from being enqueued; pausing the function does. A cancelled run can be replayed from the dashboard. — [Cancellation](https://www.inngest.com/docs/features/inngest-functions/cancellation)
- [docs] Timeouts can cancel a run that waits too long for its first step or exceeds an execution window (function configuration). — [Cancellation](https://www.inngest.com/docs/features/inngest-functions/cancellation)
- [source] Utah's sidecar injects a heartbeat function (every 30 minutes) and its cron functions "talk back" to the main agent by sending `agent.message.received` events to a `system-alerts` session; a `failure-handler.ts` function exists. — [inngest/utah README](https://github.com/inngest/utah)

### Inferences
- Compared with pi-durable (read-only open of another session's storage; intervene only through the owning process), Inngest gives the Watchdog a uniform external control plane: status, cancel and replay over HTTP from any process, with no relay methods to write.
- Infrastructure-level recovery (retries, resuming after a worker crash, re-delivery when a worker reconnects) is done by the engine, so the Watchdog's remaining job is semantic: read the Worker's transcript from the application store, judge progress with a model, and then cancel/restart (native) or steer (application inbox).
- "Stuck" detection is indirect: a run stays `Running` while parked or looping; there is no stalled-run system event in the pages read, so the Watchdog would compare run age and last transcript activity itself, or rely on function timeouts.
- Restart = send a new start/continue event for the same session key; because the transcript is external, a fresh run can continue from it. `replays` re-run the original event from the beginning, which is not the same as "continue".

### Gaps
- Whether per-step outputs and traces of a run are retrievable over REST (only run-level `output` was seen; the rest of the v2 spec was not read): unconfirmed.
- Minimum cron interval and missed-schedule behaviour: not stated.
- Which REST v2 endpoints work on a self-hosted server (the spec notes some are Cloud only, e.g. app syncs): partly unconfirmed.

## 6. Event ingestion (the article's Trigger ingestion)

### Takeaway
Ingestion is the best-covered requirement: an authenticated event API, hosted webhook endpoints with JavaScript transforms, allow/deny lists by event name and IP, and 24-hour deduplication by event ID are native. Signature verification, the code-level fast-path filter beyond name/IP lists, and Thread subscription routing remain application code.

### Cited Findings
- [docs] Webhooks: created per environment (Manage → Webhooks), each with its own generated URL and a JavaScript transform that runs on Inngest's servers. The transform receives `evt`, `headers`, `queryParams`, `raw` and returns `name` and `data`, optionally `id` (used for deduplication) and `ts`. A throwing transform returns 400 to the provider. Allow and deny lists for event names and IP addresses can be set. Webhooks can be created, updated and deleted through the REST API. — [Webhooks](https://www.inngest.com/docs/platform/webhooks); [REST API v1 spec](https://api-docs.inngest.com/api-specs/v1.json)
- [docs] "Inngest does not verify provider webhook signatures." The transform passes the raw body and signature header through and the function must verify and throw `NonRetriableError` on failure. — [Webhooks](https://www.inngest.com/docs/platform/webhooks)
- [docs] Event ID dedup: a stable event `id` is an idempotency key for 24 hours; a duplicate is still stored and visible in history but triggers no functions. Function-level `idempotency` (CEL key) allows one run per key per 24 hours. "Event-ID deduplication does not apply to debouncing or event batching." — [Idempotency](https://www.inngest.com/docs/guides/handling-idempotency)
- [docs] Events fan out: one event can trigger any number of functions; events can be scheduled with a future `ts`. — [Idempotency](https://www.inngest.com/docs/guides/handling-idempotency); [Schedules and delayed starts](https://www.inngest.com/docs-markdown/durable-execution/guides-and-advanced/events-and-triggers/schedules-and-delayed-starts)
- [docs] Cloud event quotas: 500k events/month on Free, 5M on Pro; queue depth 100k (Free). — [Pricing](https://www.inngest.com/pricing)
- [source] Utah's worker sets up channel webhooks and transforms automatically at start-up (Telegram fully automated; Slack needs a Slack app with Event Subscriptions), and the worker receives the resulting events over `connect` with no public endpoint. — [inngest/utah README](https://github.com/inngest/utah)

### Inferences
- With Inngest Cloud, webhook reception needs no public endpoint on the developer's side. With self-hosting, the developer's own Inngest server must be publicly reachable to receive webhooks.
- The article's two-stage filter maps as: stage 1 = transform plus trigger `if` expressions (CEL) and allow/deny lists, in code; stage 2 = Router prompt. Thread subscriptions (pass-through to a subscribed Worker) would be an application table consulted by a small routing function, or a Worker parked in `waitForEvent` with `match` on a thread ID (subject to the missed-event race in section 3).
- Dedup caveat for the Router: if the Router function uses `batchEvents`, event-ID dedup does not apply to it, so dedup would have to happen in an upstream ingest function.

### Gaps
- Whether webhook endpoints and transforms are available in the self-hosted server: not stated on the webhook page (unconfirmed; the v1 REST spec lists the Dev Server as a server, which suggests yes).
- Webhook payload size limit (presumably the plan's event size limit): not stated.

## 7. LLM agent support: AgentKit, step.ai, and what is not provided

### Takeaway
AgentKit (`@inngest/agent-kit`) is a 0.x TypeScript library whose last npm release is 0.13.2 from 13 November 2025, with the repository last pushed on 29 April 2026; Inngest's 2026 agent documentation and blog posts do not mention it and instead teach a hand-written loop on `step.run` / `step.invoke` / `step.sendEvent` ("a harness, not a framework"). Neither AgentKit nor the core platform provides a durable transcript store, a session inbox, mid-turn steering or built-in context compaction.

### Cited Findings
- [source] Releases: `@inngest/agent-kit@0.13.2` 13 Nov 2025, `0.13.1` 7 Oct 2025, `0.13.0` 27 Sep 2025; `@inngest/use-agent@0.4.0` 1 Oct 2025. npm `latest` is 0.13.2 (package metadata last modified 6 Feb 2026). Last commits on the repo: 29 Apr 2026 ("Improve support for reasoning models"), 8 Apr 2026 (docs). Apache-2.0, 963 stars. — [inngest/agent-kit](https://github.com/inngest/agent-kit); [releases](https://github.com/inngest/agent-kit/releases)
- [source] README: since v0.9.0 `inngest` must be installed as a separate dependency. Core concepts: Agents, Networks (agents collaborating over shared State), State (conversation history plus typed data), Routers (code-based to LLM-based), Tracing. — [inngest/agent-kit README](https://github.com/inngest/agent-kit)
- [docs] AgentKit details: `createNetwork({ agents, defaultModel, router, history })`; routers receive `input, network, stack, callCount, lastResult` and return the next agent or nothing to stop; code-based, routing-agent and hybrid routers; agents are stateless; tools via `createTool` (zod parameters; handler receives `network`, `agent`, `step`); MCP servers over `sse`, `ws` and `streamable-http`; human-in-the-loop via `step.waitForEvent` inside a tool; models OpenAI, Anthropic, Gemini, Grok and OpenAI-compatible. Inngest is an optional dependency, but retries, offloaded inference and tracing need it. — [AgentKit docs (llms-full)](https://agentkit.inngest.com/llms-full.txt)
- [docs] History adapter (`HistoryConfig`): four optional methods `createThread`, `get`, `appendUserMessage`, `appendResults`, implemented by the application against its own database; pre-built adapters are described as "coming soon". Progressive summarisation is listed only as a future enhancement. — [AgentKit docs (llms-full)](https://agentkit.inngest.com/llms-full.txt)
- [docs] `useAgent` React hook (`@inngest/use-agent`) streams agent events over WebSocket via `@inngest/realtime`; statuses `idle | thinking | calling-tool | responding | error`; enabled with `streaming.publish` in `network.run`. Realtime quotas on Cloud: 50 connections and 250k messages/day on Free. — [AgentKit docs (llms-full)](https://agentkit.inngest.com/llms-full.txt); [Pricing](https://www.inngest.com/pricing)
- [docs] The AgentKit docs read do not document interrupting or steering a running network, cancellation, or long-lived sessions beyond persistent threads via history adapters. Only the first quarter of the 396k-character docs dump was read. — [AgentKit docs (llms-full)](https://agentkit.inngest.com/llms-full.txt)
- [docs] The current "Durable agents" docs section (agent tool loops, sub-agent delegation, sessions, Sandboxes, Realtime) uses direct model SDK calls in `step.run` and does not mention AgentKit. — [Durable agents](https://www.inngest.com/docs-markdown/durable-execution/durable-agents); [Agent tool loops](https://www.inngest.com/docs-markdown/durable-execution/durable-agents/agent-tool-loops)
- [docs] Loop safeguards are application code in the documented pattern: maximum iterations (example 10), token budget (example 500,000 input tokens), stuck-loop detection (same tool three iterations running), `pruneMessages` helper keeping the first and most recent messages. — [Agent tool loops](https://www.inngest.com/docs-markdown/durable-execution/durable-agents/agent-tool-loops)
- [docs] Platform AI features around the loop: Realtime ("streams tokens and progress to the browser while the agent runs"), Sessions and Scores (Agent Evals; 10,000 scores/month on Free), Sandboxes (beta), an `extendedTracesMiddleware` for OpenTelemetry spans. — [Durable agents](https://www.inngest.com/docs-markdown/durable-execution/durable-agents); [Pricing](https://www.inngest.com/pricing); [blog](https://www.inngest.com/blog/ai-agents-inngest-durable-steps)
- [3p] An AgentKit docs snippet surfaced in search describes AgentKit as "in early access"; the page date is unknown. — [AgentKit overview](https://inngest.com/docs/agent-kit/overview)

### Inferences
- AgentKit appears to be in maintenance rather than active development as of November 2026 (no release in 12 months, no mention in current first-party agent guidance). No deprecation notice was found, so "superseded" is an inference.
- AgentKit's "router" is an in-run agent-selection function inside one network run, unrelated to the article's long-lived Router session; its "state" lives for one `network.run` unless a history adapter persists it.
- What is absent on Inngest relative to pi-durable: a persisted conversation object, a persisted per-session inbox with steer/follow-up modes, a checkpointed model stream, replay-safety classes for tools, and a session-level `continue`. What is present relative to pi-durable: an event bus with dedup, cron, hosted webhooks, flow control, an external run control API and a dashboard with traces.

### Gaps
- AgentKit's roadmap or any official statement about its status in 2026 was not found.
- Whether `step.ai.infer` supports tool-call responses fully and whether streaming has shipped since the docs sentence "coming soon" was written: unconfirmed.

## 8. Hosting and cost for a single developer

### Takeaway
Self-hosting is one binary (`inngest start`, labelled "[Beta] Run Inngest as a single-node service") with SQLite plus an in-process Redis by default and optional external Postgres and Redis; it is free of charge under SSPL with a delayed Apache 2.0 licence, with no guaranteed support and manual data pruning. Inngest Cloud's free tier (50k executions/month, 5 concurrent steps, 30-day maximum run, 24-hour trace history) is usable for a personal agent, and the first paid tier starts at $99/month.

### Cited Findings
- [docs] Self-hosting has been supported since the 1.0 release. "Inngest's support team does not guarantee direct support for self-hosted instances." — [Self-hosting](https://www.inngest.com/docs/self-hosting)
- [docs] The CLI is a single binary containing all services (`docker pull inngest/inngest`, `npm install -g inngest-cli`, or the install script). `inngest start` help text: "[Beta] Run Inngest as a single-node service." It serves the Event API, API and dashboard on port 8288 and the Connect gateway on 8289. — [Self-hosting](https://www.inngest.com/docs/self-hosting)
- [docs] Default persistence is SQLite at `./.inngest/main.db`; the default queue and state store is an in-memory Redis inside the Inngest process with periodic snapshots (also taken before shutdown). SQLite "does not support scaling beyond a single node". Optional `--postgres-uri` (configuration and history) and `--redis-uri` (queue and run state); the page says an external Redis can improve performance and reliability. — [Self-hosting](https://www.inngest.com/docs/self-hosting)
- [docs] Requires a signing key (hex) and event key(s); SDK side: `INNGEST_DEV=0`, `INNGEST_BASE_URL`, `INNGEST_EVENT_KEY`, `INNGEST_SIGNING_KEY`. Docker Compose example with Postgres 17 and Redis 7; a Helm chart ("production-ready Kubernetes deployments") at `inngest/inngest-helm`. Prometheus `/metrics` (only `inngest_queue_depth` by default); no authentication on it by default. Postgres tables are not pruned automatically. `inngest alpha doctor` diagnostics since v1.19.3. — [Self-hosting](https://www.inngest.com/docs/self-hosting)
- [source] Server releases are frequent: v1.53.0 (12 Nov 2026), v1.52.0 (10 Nov), v1.51.0 and v1.50.0 (6 Nov), v1.49.0 (23 Oct). 6,035 stars. — [inngest/inngest releases](https://github.com/inngest/inngest/releases)
- [source] Licence of the server: `LICENSE.md` is headed "Server Side Public License, Version 1.0" and "Apache 2.0 Future License", copyright Inngest, Inc. 2022. The delay period before the Apache 2.0 conversion was not read (unconfirmed; commonly cited as a delayed open-source publication model). — [inngest/inngest LICENSE.md](https://github.com/inngest/inngest/blob/main/LICENSE.md)
- [source] TypeScript SDK: `inngest@4.26.2` (29 Oct 2026). GitHub's licence detection reports GPL-3.0 for `inngest/inngest-js` (unconfirmed against the package's own `license` field; earlier major versions were published as Apache-2.0). — [inngest/inngest-js](https://github.com/inngest/inngest-js)
- [docs] Cloud plans: Free $0 (50k executions/month, execution pauses at quota; 5 concurrent steps; 3 connect workers; 500k events/month; 256 KiB events; batch 5 events / 30 s; 24-hour trace history; 30-day max run); Pro from $99/month (1M executions, then $50 per 1M; 100 concurrent steps; 20 workers; 5M events; 3 MiB events; 7-day history; 90-day max run); Business from $499/month (10M executions; 500 concurrent steps; 14-day history; 366-day max run). — [Pricing](https://www.inngest.com/pricing)
- [docs] "An execution is one function run or one step execution. A run with five steps uses six executions." — [Usage limits](https://www.inngest.com/docs/usage-limits/inngest)
- [docs] Connect worker limits on Cloud: 3 concurrent worker connections on Free, 20 on paid; 10 apps per connection. — [Connect](https://www.inngest.com/docs-markdown/durable-execution/deploying-functions/connect)
- [3p] A 2026 comparison article states that Connect "remains in public beta"; the official Connect page read does not state a maturity level. — [ZenML, Inngest alternatives](https://www.zenml.io/blog/inngest-alternatives)

### Inferences
- A personal agent doing, say, 200 tool rounds a day uses roughly 400 to 600 executions a day plus one per run, i.e. on the order of 15k to 20k per month, inside the Free tier's 50k; the binding Free-tier limits are more likely 5 concurrent steps (Router plus a few Workers each executing a step), batch size 5, and 24-hour trace history.
- Self-hosting adds a second always-on process (the Inngest server) next to the worker, against pi-durable's single process with one SQLite file. With the default in-memory Redis, run state durability between snapshots depends on graceful shutdown; a single developer wanting crash-safe run state would add an external Redis with persistence and, optionally, Postgres.
- The SSPL matters only if the system is offered to others as a service; private self-hosting is unaffected (general reading of SSPL, not legal advice and not from an Inngest statement).

### Gaps
- Snapshot interval of the in-process Redis and what is lost on a hard crash: not stated.
- Whether Cloud-plan limits (steps, state size, run length) apply to self-hosted servers: unconfirmed.
- Length of the SSPL-to-Apache delay and the exact SDK licence: unconfirmed.

## 9. Published examples of long-lived, always-on, multi-session agents on Inngest

### Takeaway
The closest published example is Inngest's own Utah ("Universally Triggered Agent Harness", Apache-2.0, created February 2026, last pushed July 2026, about 139 stars): a personal Slack/Telegram agent with one function run per message, singleton cancel-on-new-message per session, file-based transcript, sync/async/scheduled sub-agents, and a sidecar of cron functions. No write-up of a router/worker/watchdog system with a months-long batching Router on Inngest was found.

### Cited Findings
- [source] Utah: "A durable AI agent built with Inngest and pi-ai. No framework." Features: every LLM call and tool execution is a step; "Singleton concurrency — one conversation at a time per chat"; "Cancel on new message — user sends again? Current run cancels, new one starts"; Slack and Telegram channels; runs locally through `connect()`. Flow: channel → Inngest Cloud (webhook + transform) → WebSocket → local worker → LLM → reply event → channel API. Requires Node.js 23+. Functions: `message.ts`, `sub-agent.ts`, `send-reply.ts`, `acknowledge-message.ts`, `failure-handler.ts`, `heartbeat.ts`. Session persistence is JSONL (`src/lib/session.ts`), with `memory.ts` and `compaction.ts`. — [inngest/utah](https://github.com/inngest/utah); [message.ts](https://github.com/inngest/utah/blob/main/src/functions/message.ts)
- [source] Utah's sidecar: a second process hot-loads Inngest functions the agent writes to `workspace/functions/*.ts` (cron jobs, event handlers), with a 30-minute heartbeat; these functions message the main agent by sending `agent.message.received` events. "The agent is ephemeral. Its output is durable." — [inngest/utah README](https://github.com/inngest/utah)
- [blog] Dan Farrelly, "How to Build AI Agents That Don't Fail Silently" (page dated 10 Aug 2026; a search result dated an earlier version 17 Mar 2026): `createAgentLoop(userMessage, sessionKey)`; `handleMessage` with `retries: 2` and singleton cancel per `sessionKey`; history reloaded at the start of each run via `buildConversationHistory(sessionKey)`. — [blog](https://www.inngest.com/blog/ai-agents-inngest-durable-steps)
- [blog] Companion posts: [Three sub-agent patterns](https://www.inngest.com/blog/three-patterns-you-need-for-agentic-systems) (11 Mar 2026), [Your Agent Needs a Harness, Not a Framework](https://www.inngest.com/blog/your-agent-needs-a-harness-not-a-framework), [Building durable agents](https://www.inngest.com/blog/building-durable-agents), [Building a durable goal agent loop that survives crashes](https://www.inngest.com/blog/building-a-durable-goal-agent-loop-that-survives-crashes), [Don't pay for the same step twice](https://www.inngest.com/blog/dont-pay-for-the-same-step-twice). Only the first two listed here and the main post were read; the others are listed by title only.
- [blog] A customer story about an internal ops agent built on Inngest by a user exists (title only seen in search; not read). — [blog](https://www.inngest.com/blog/user-built-windmill-internal-ops-agent)
- [3p] A third-party comparison says Inngest fits best when a function has a small number of durable steps and is "less tidy" when an agent creates a long, changing loop at runtime. — [ZenML, Inngest alternatives](https://www.zenml.io/blog/inngest-alternatives)
- [3p] Search surfaced a community project ("joelclaw system-bus") that appears to use Inngest as an event bus for a personal agent; it was not read (unconfirmed). — [openskillindex listing](https://openskillindex.com/skills/joelhooks-joelclaw-system-bus)

### Inferences
- Utah differs from the Super-agent Router in the key behaviour: a new message cancels the running turn and restarts it with the new message appended, instead of queueing input and batching it into the next turn. Its async sub-agents reply to the user directly and are not fed back into the parent conversation, so the article's "Workers report to the Router, which sends all replies" is not implemented there.
- Utah has no LLM-judging Watchdog; it has a failure handler and heartbeat. Its sidecar cron-functions-that-message-the-agent pattern is, however, the same mechanism a Watchdog would use.
- Utah uses pi-ai (the same LLM layer as pi-durable) for model access, so the two stacks are not mutually exclusive at the model-client level.

### Gaps
- No independent engineering write-up of a production, always-on multi-session agent on Inngest with router/worker/supervisor roles was found in the searches run; absence in these searches is not proof of absence.
- The "harness not framework", "building durable agents" and "goal agent loop" posts were not read; they may contain further detail on long-running loops and context management.
