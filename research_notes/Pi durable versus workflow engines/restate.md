# Restate as a substrate for a long-lived, event-driven LLM agent system (state: 14 November 2026)

Conventions used in these notes:

- **[docs]** = official Restate documentation (docs.restate.dev), read 14 Nov 2026.
- **[release]** = GitHub release notes of `restatedev/restate` or `restatedev/sdk-typescript`.
- **[repo]** = files in a Restate-owned GitHub repository (README, docs folder, LICENSE). No Restate server or SDK source code was read; "repo" here means repository documentation, not code.
- **[vendor]** = restate.dev marketing pages and blog.
- **[3p]** = third party.
- **unconfirmed** = could not be confirmed from a fetched primary source.
- Nothing was installed or run. The GitHub release pages show day and month only; the year 2026 is taken from asset timestamps (server) and assumed for the SDK.

Versions found: Restate server **v1.8.1** (21 Oct 2026, marked "Latest"), with v1.8.1-hotfix.2 (2 Nov 2026) and v1.8.0 (14 Oct 2026); TypeScript SDK `@restatedev/restate-sdk` **v1.18.3** (4 Nov 2026).

## 1. Execution model: what is durable, determinism, where LLM and tool calls live, retries, idempotency

### Takeaway
Restate journals the result of every context operation of a handler invocation (`ctx.run` steps, calls, sends, timers, state operations, signals) and, after a failure, re-runs the handler code and replays journaled results instead of re-executing them. LLM calls and tool side effects go inside `ctx.run`; code between context operations must be deterministic, and a `ctx.run` step that fails or is cut off is retried, so external side effects still need their own idempotency.

### Cited Findings
- [docs] `ctx.run` wraps non-deterministic work (HTTP calls, database responses); its result is stored in the execution log and replayed instead of re-run. Results are serialized as JSON by default. — [Journaling results (TS)](https://docs.restate.dev/develop/ts/journaling-results.md)
- [docs] The Restate context cannot be used inside `ctx.run` (no `ctx.get`, `ctx.sleep`, or nested `ctx.run`). — [Journaling results (TS)](https://docs.restate.dev/develop/ts/journaling-results.md)
- [docs] Deterministic replacements: `ctx.rand.uuidv4()`, `ctx.rand.random()` ("behaves like `Math.random()` but is deterministically replayable"), `await ctx.date.now()`. — [Journaling results (TS)](https://docs.restate.dev/develop/ts/journaling-results.md)
- [docs] "Failures in `ctx.run` are treated the same as any other handler error": retried unless a `TerminalError` is thrown or a retry policy is exhausted. Per-step options: `initialRetryInterval`, `retryIntervalFactor`, `maxRetryInterval`, `maxRetryAttempts`, `maxRetryDuration`. "When the policy is exhausted, a `TerminalError` is thrown." — [Journaling results (TS)](https://docs.restate.dev/develop/ts/journaling-results.md)
- [docs] Default invocation retry policy: `initial-interval = "50ms"`, `exponentiation-factor = 2.0`, `max-attempts = 70`, `max-interval = "60s"`, `on-max-attempts = "pause"` (alternative `"kill"`). — [Service configuration](https://docs.restate.dev/services/configuration.md)
- [docs] Default inactivity timeout 1 minute, abort timeout 10 minutes (`worker.invoker.inactivity-timeout`, `worker.invoker.abort-timeout`); both can be set per service or handler in the TS SDK (`inactivityTimeout`, `abortTimeout`). If no new journal entries arrive for more than one minute (default), Restate suspends and retries the invocation; longer work needs these timeouts raised. — [Service configuration](https://docs.restate.dev/services/configuration.md); [Journaling results (TS)](https://docs.restate.dev/develop/ts/journaling-results.md)
- [docs] In the agent guides, LLM calls are journaled either with `ctx.run("LLM call", ...)` (plain SDK) or by wrapping the model with `durableCalls(ctx, { maxRetryAttempts: 3 })` from `@restatedev/vercel-ai-middleware`; each tool wraps its side effect in a durable step. The guide says an LLM call is "journaled so it won't re-execute on recovery" and "Side effects are executed exactly once." — [Durable Agents](https://docs.restate.dev/ai/patterns/durable-agents.md)
- [docs] With the plain SDK "you write the loop yourself: call the LLM, run any tool calls, and repeat". For the LangChain middleware: "The middleware does not auto-journal tool calls — you choose which steps are durable." — [Durable Agents](https://docs.restate.dev/ai/patterns/durable-agents.md)
- [docs] Idempotency: calls from one handler execution are deduplicated by the journal. `idempotencyKey` can be added with `restate.rpc.opts` / `restate.rpc.sendOpts`; at the HTTP ingress an `Idempotency-Key` header makes Restate persist the response for 24 hours by default (tunable). — [Service communication (TS)](https://docs.restate.dev/develop/ts/service-communication.md); [HTTP invocation](https://docs.restate.dev/services/invocation/http.md)
- [docs] Deployments are immutable: new requests route to the latest registered version while in-flight invocations finish on their original deployment. Resuming a paused invocation on code whose flow differs can cause non-determinism errors. — [Services](https://docs.restate.dev/foundations/services.md); [Managing invocations](https://docs.restate.dev/services/invocation/managing-invocations.md)
- [release] SDK v1.18.3 (4 Nov 2026): "Propagate RetryableError code and metadata to the TerminalError when ctx.run retries are exhausted". — [sdk-typescript releases](https://github.com/restatedev/sdk-typescript/releases)

### Inferences
- The "executed exactly once" wording applies to the journaled result. A `ctx.run` body that crashes or times out after the external effect but before the result is journaled is run again (the docs say failed steps are retried), so remote commands and PR/comment posting need an idempotency key or reconciliation, the same class of concern as pi-durable's `replay: safe`.
- A long LLM request inside one `ctx.run` writes no journal entry while in flight; with the 1-minute default inactivity timeout a slow model call would be suspended and retried, so the timeout must be raised for agent handlers.
- Because in-flight invocations are pinned to their deployment version, a months-long single invocation would pin an old code version; short invocations over stored state avoid that.

### Gaps
- The docs page does not use the term "at-least-once" for `ctx.run`; the mid-step failure semantics above are inferred from "failed runs are retried".
- No numeric journal-entry or journal-byte limit was found (see section 4).

## 2. Service types: which fits a months-long keyed agent session

### Takeaway
Of the three types, the Virtual Object is the one Restate documents for sessions: keyed, K/V state kept indefinitely, one exclusive (writing) handler at a time per key, plus read-only shared handlers that run concurrently. A Workflow's `run` handler executes once per ID and its state is retained only for a retention period (24 h default) after completion.

### Cited Findings
- [docs] Basic Service: "No shared state between requests"; handlers run in parallel. — [Services](https://docs.restate.dev/foundations/services.md)
- [docs] Virtual Object: stateful entity identified by a key; K/V state is "retained indefinitely and shared across requests." "At most one handler with write access can run at a time per object key" (a queue per key); different keys run concurrently; shared handlers are read-only and concurrent. — [Services](https://docs.restate.dev/foundations/services.md)
- [docs] Workflow: "The `run` handler executes exactly once per workflow ID." "Shared handlers run concurrently with the `run` handler" and can resolve durable promises or query state. After `run` completes, other handlers remain callable for 24 hours by default (workflow retention, configurable). — [Services](https://docs.restate.dev/foundations/services.md); [External events (TS)](https://docs.restate.dev/develop/ts/external-events.md)
- [docs] State API: `ctx.get`, `ctx.set`, `ctx.clear`, `ctx.clearAll`, `ctx.stateKeys`. "State is only available for Virtual Objects and Workflows." Virtual Object state is kept until explicitly cleared; workflow state only for the retention time. Shared handlers cannot mutate state. — [State (TS)](https://docs.restate.dev/develop/ts/state.md)
- [docs] Eager state (default): all state is sent with the invocation request. Lazy state: fetched per `get`, suggested "for large state that most handlers don't need". No explicit state size limit is given. — [State (TS)](https://docs.restate.dev/develop/ts/state.md)
- [release] SDK v1.18.0 (28 Oct 2026) adds "Selective state preloading with `state.preload`" (all keys, none, or a listed subset) and deprecates `enableLazyState`. — [sdk-typescript releases](https://github.com/restatedev/sdk-typescript/releases)
- [docs] The Durable Sessions guide implements a session as a Virtual Object keyed by user/conversation/session ID with history in K/V state (`"messages"` or `"memory"`), a writing `message` handler and a shared `getHistory` handler; state is said to "live forever". It states no limit on history size, queue length or session count. — [Durable Sessions](https://docs.restate.dev/ai/patterns/sessions.md)
- [docs] State can be inspected and edited in the UI and CLI; `restate state get <SERVICE> <KEY>` and SQL over the `state` table (`value_utf8`, `value`, `value_length`). — [State (TS)](https://docs.restate.dev/develop/ts/state.md); [Introspection](https://docs.restate.dev/services/introspection.md); [SQL introspection reference](https://docs.restate.dev/references/sql-introspection.md)
- [3p] Commentary notes that Virtual Object serialization covers only same-key writes inside Restate, not exclusive control of an external database. — [xenospectrum](https://xenospectrum.com/en/restate-ai-agent-durable-execution-funding/)

### Inferences
- A Router living for months maps to one Virtual Object key whose state holds the transcript; with eager state the whole transcript would be shipped on every invocation, so chunked keys with lazy/selective loading are needed (Restate's own reference agent uses lazy state and chunked history, see section 8).
- A Workflow is a poor fit for the session itself (run-once, 24 h retention), but fits a single bounded task.

### Gaps
- No documented maximum for a single state value, total state per key, or number of keys. The server has a `message-size-limit` setting that also bounds per-invocation memory, but its default value was not stated on the fetched page — [Server memory](https://docs.restate.dev/server/memory.md). The actual default is unconfirmed.

## 3. Input during a running invocation: queueing, steering, batching, signals

### Takeaway
An exclusive handler blocks its key: further exclusive calls wait in a per-key inbox and are dispatched one at a time, so Restate itself neither delivers them mid-handler nor hands them over as a batch. Mid-turn delivery is possible with signals (addressed to a running invocation ID, multi-shot, stored if early) sent by another handler, and Restate's own reference agent uses exactly that; when the signal is consumed (between LLM calls or during them) is application logic. Batching is also application logic.

### Cited Findings
- [docs] Requests to the same session key "are processed sequentially"; only one write handler runs per key; shared handlers can run alongside it. — [Durable Sessions](https://docs.restate.dev/ai/patterns/sessions.md)
- [docs] SQL table `sys_inbox` (`service_name`, `service_key`, `id`, `sequence_number`): "Only one invocation runs at a time per key; the others wait here, ordered by `sequence_number`." — [SQL introspection reference](https://docs.restate.dev/references/sql-introspection.md)
- [docs] Awakeable: `const { id, promise } = ctx.awakeable<string>()`; resolved once, from a handler (`ctx.resolveAwakeable`) or over HTTP (`/restate/awakeables/<id>/resolve` or `/reject`). Waiting on an awakeable inside an exclusive Virtual Object handler means "all other calls to this object will be queued." — [External events (TS)](https://docs.restate.dev/develop/ts/external-events.md)
- [docs] Signal: addressed by target invocation ID plus a name; can be resolved multiple times, each wait receives the next resolution; a signal that arrives before the invocation waits is stored. Wait with `await ctx.signal<T>("name")`; send from another handler with `ctx.invocation(id).signal<T>("name").resolve(value)` (or `.reject()`). — [External events (TS)](https://docs.restate.dev/develop/ts/external-events.md)
- [docs] Guidance: signal = another Restate invocation notifies a running invocation one or more times; awakeable = external one-shot callback token over HTTP; workflow promise = value owned by a workflow key, resolved once, readable by its handlers. — [External events (TS)](https://docs.restate.dev/develop/ts/external-events.md)
- [release] SDK v1.18.0 (28 Oct 2026): repeated signals are no longer overwritten; "Every delivery is now kept and handed out in order, one per `ctx.signal(name)` call." Server v1.8.1 (21 Oct 2026): signals to not-yet-started invocations are recorded (opt-in flag). — [sdk-typescript releases](https://github.com/restatedev/sdk-typescript/releases); [restate releases](https://github.com/restatedev/restate/releases)
- [docs] Restate's documented "Interrupt & Regenerate" pattern does not steer: the session object stores the invocation ID of the running task (a separate Service invocation), and on a new message calls `ctx.cancel(id)`, waits with `ctx.attach(id).orTimeout(30_000)`, then dispatches a new task. Cancellation "causes it to terminally fail and raise an error at the next durable step" and reaches nested sub-invocations. Shown for the plain TS and Python SDKs only. — [Interrupt & Regenerate](https://docs.restate.dev/ai/patterns/interrupt-regenerate.md)
- [repo] Restate's reference agent (`restatedev/agent`, TypeScript) splits each agent into two Virtual Objects with the same ID: `Agent` (controller: active invocation ID, queued input, approvals, schedules) and `AgentSession` (history, exclusive `doTurn`). "The controller's exclusive lock is short-lived. It never waits for the whole turn." `ask` queues FIFO while busy; `steer` "drains queued input into an ordered signal for the active turn"; steering, interruption and approval decisions are "durable signals addressed to that invocation's `turnId`." — [agent docs/architecture.md](https://raw.githubusercontent.com/restatedev/agent/main/docs/architecture.md)
- [repo] In that implementation "Steering is drained only at defined boundaries after the step settles"; it never cancels a step or pending operation; a text or model-error result is discarded as stale if steering arrived during its step. A steering signal carries `{queued, message}`, so queued messages are folded into the running turn as a batch. — [agent docs/turn-runtime.md](https://raw.githubusercontent.com/restatedev/agent/main/docs/turn-runtime.md)
- [repo] README summary: "Restate provides durable signal delivery. The reference implements the routing, turn state machine, and cleanup rules on top of it." — [restatedev/agent README](https://raw.githubusercontent.com/restatedev/agent/main/README.md)

### Inferences
- Batching "all events queued during a turn into the next turn" is not a Restate primitive: a handler cannot read its own inbox. The documented way is a short-lived controller handler that appends each event to a state list and a turn handler that drains the list (or receives it via signal), as the reference agent does.
- New input cannot be injected into an LLM HTTP request that is already in flight; in the reference agent it is applied at the next step boundary, comparable to pi-durable's "input is never placed mid-tool". An application could race `ctx.signal` against the LLM `ctx.run` promise to abandon the call early, but that is unconfirmed from docs.
- Signals need the sender to be a Restate handler ("from another handler"); an HTTP path for signals was not shown, unlike awakeables.

### Gaps
- Whether signals are available in every service type and from which server version they are generally available is not stated on the docs page. The reference agent's README requires `RESTATE_EXPERIMENTAL_ENABLE_PROTOCOL_V7=true`, while server v1.8.0 notes say "Service protocol v7 is negotiated by default"; whether signals depend on protocol v7 is unconfirmed.
- The docs do not show a TS SDK example of racing a signal against an in-flight `ctx.run`.

## 4. Journal and invocation length; one long invocation or many short ones

### Takeaway
No numeric journal limit was found in the docs. All Restate-published session designs use many short invocations over Virtual Object state (one invocation per message or per turn), not one invocation per session, and completed-invocation journals are kept only 24 hours by default.

### Cited Findings
- [docs] Journal retention default 24 hours (`default-journal-retention`); idempotency retention 24 hours; workflow retention 24 hours from completion of `run`. — [Service configuration](https://docs.restate.dev/services/configuration.md)
- [docs] `sys_invocation` exposes `journal_size` (number of durably logged entries) and `journal_commands_size`; per-entry rows are in `sys_journal`. — [SQL introspection reference](https://docs.restate.dev/references/sql-introspection.md)
- [docs] Server memory settings: invoker pool `worker.invoker.memory-limit` 1.5 GiB default; `per-invocation-initial-memory` 32 KiB; `per-invocation-memory-limit` defaults to and is clamped by `message-size-limit` (value not given on the page). An invocation at its memory ceiling waits or yields to the scheduler. — [Server memory](https://docs.restate.dev/server/memory.md)
- [repo] Reference agent: "A turn is one `AgentSession.doTurn` invocation. Its invocation ID is its `turnId`." A run performs at most 50 loop iterations; history is append-only in object state and compaction writes a summary checkpoint. — [agent docs/architecture.md](https://raw.githubusercontent.com/restatedev/agent/main/docs/architecture.md); [agent docs/turn-runtime.md](https://raw.githubusercontent.com/restatedev/agent/main/docs/turn-runtime.md)
- [docs] Sessions guide: one `message` invocation per user message over state in the object. — [Durable Sessions](https://docs.restate.dev/ai/patterns/sessions.md)
- [repo] A suspended invocation holds no process: when a guardrail needs a person "the turn suspends: no process, only stored state", and can wait for weeks across service versions. — [restatedev/agent](https://github.com/restatedev/agent)
- [vendor] Restate blog "Code that sleeps for a month" discusses long sleeps and state rebuilt from the journal on replay (older post, date not checked). — [restate.dev blog](https://restate.dev/blog/code-that-sleeps-for-a-month)

### Inferences
- A session as a single months-long invocation would replay an ever-growing journal on every resume and stay pinned to one deployment version; the documented and reference shape is session = object state, turn = invocation.
- The transcript therefore lives in application-managed K/V state, not in the journal; the journal of a finished turn disappears after the retention period, so anything needed later (tool outputs, audit) must be written to state by the application.

### Gaps
- Hard limits (max journal entries, max entry size, default `message-size-limit`) not found in fetched docs: unconfirmed.

## 5. Workers as separate objects: sends, delayed sends, attach, cancel, reporting back

### Takeaway
Restate has first-class one-way and delayed sends, invocation IDs from send handles, `ctx.attach` and `ctx.cancel`, so "start a Worker asynchronously, stop it later" is direct. Reporting back is not a built-in: the child sends a message to the parent object (which queues behind the parent's exclusive handler) or the parent awaits/attaches.

### Cited Findings
- [docs] One-way: `ctx.objectSendClient(myObject, "key").myHandler(arg)`; Restate handles delivery and retries. Delayed: `restate.rpc.sendOpts({ delay: { hours: 5 } })`. `genericSend`/`genericCall` exist for dynamic targets. — [Service communication (TS)](https://docs.restate.dev/develop/ts/service-communication.md)
- [docs] `const invocationId = await handle.invocationId;` from a send handle; `ctx.attach(invocationId)` (with an idempotency key attach returns the result, otherwise only waits); `ctx.cancel(invocationId)`. — [Service communication (TS)](https://docs.restate.dev/develop/ts/service-communication.md)
- [docs] Request-response calls between exclusive handlers of Virtual Objects can deadlock (A→B and B→A on the same keys, or cycles); resolution is cancelling via UI or CLI. — [Service communication (TS)](https://docs.restate.dev/develop/ts/service-communication.md)
- [docs] Cancel is non-blocking, needs the deployment reachable, throws at the next await point in the target, and propagates to nested sub-invocations; one-way and delayed sends made by the cancelled invocation "still run". Kill stops the invocation tree "without executing compensation logic". — [Managing invocations](https://docs.restate.dev/services/invocation/managing-invocations.md); [Interrupt & Regenerate](https://docs.restate.dev/ai/patterns/interrupt-regenerate.md)
- [docs] In the Interrupt & Regenerate pattern the cancelled task catches the `TerminalError`, sends a cleanup message to the agent object through `appendMessage`, and rethrows: an example of child-to-parent reporting by one-way send. — [Interrupt & Regenerate](https://docs.restate.dev/ai/patterns/interrupt-regenerate.md)
- [docs] The Multi-Agent guide shows only request-response: the parent waits for sub-agents ("Once all sub-agents return, the main agent continues"). The Vercel example sets `parallelToolCalls: false`. — [Multi-Agent Orchestration](https://docs.restate.dev/ai/patterns/multi-agent.md)
- [repo] Reference agent: sub-agents are persistent agents with their own history, memory and sandbox, run as separate durable invocations; `createSubAgent`/`messageSubAgent` are "Durable child-turn wait" and the answers return as tool results: "The parent session waits for that exact invocation; its controller stays responsive." Child IDs are deterministic (parent ID, turn ID, tool-call ID). "A child accepts only one task at a time." Children cannot create children or schedules. — [agent docs/tools.md](https://raw.githubusercontent.com/restatedev/agent/main/docs/tools.md)
- [repo] The same repo has "pending tools": the tool acknowledges immediately and "Pending completion becomes a runtime message in a later model round"; but "A turn cannot finish while pending work exists unless the model cancels it". — [agent docs/tools.md](https://raw.githubusercontent.com/restatedev/agent/main/docs/tools.md)

### Inferences
- The article's non-blocking `start()` maps to: Router handler does a one-way send to a `Worker` object keyed by task ID and stores the invocation ID; `stop()` is `ctx.cancel(id)`. The Worker's final step is a one-way send to the Router object, which is an ordinary queued exclusive invocation, i.e. it arrives as a new event. This is application code but each step is a documented primitive.
- Restate's own multi-agent material is blocking parent-waits-for-child; the fully detached Worker that outlives the Router's turn is not shown in the docs fetched.

### Gaps
- No documented example of a detached child reporting to a long-lived parent across turns was found.

## 6. Watchdog: scheduling, inspection, intervention, stuck detection

### Takeaway
There is no built-in cron; a schedule is a Virtual Object that sends itself a delayed message (documented guide). Inspection from outside the owning object is well supported: SQL introspection over invocations, journals, inboxes, locks and object state via CLI or HTTP on the admin port, and intervention via admin API/CLI (cancel, kill, pause, resume, restart-as-new, purge) or from handler code (`ctx.cancel`, signals, sends).

### Cited Findings
- [docs] "Restate doesn't have built-in cron functionality". The guide's pattern: a `CronJob` Virtual Object stores the job in state, computes the next run, sends a delayed `execute` call to itself and stores that invocation ID; `cancel` cancels the pending invocation. Caveats: time zones are yours to handle; if an execution is still retrying when the next fires, you "would have concurrent executions of the same cron job" unless you check and cancel. — [Cron Jobs guide](https://docs.restate.dev/guides/cron.md)
- [docs] Introspection access: `restate sql --json "<query>"`, or HTTP `POST localhost:9070/query` with a JSON body containing `query`; plus the UI. — [Introspection](https://docs.restate.dev/services/introspection.md)
- [docs] Tables: `sys_invocation` (`id`, `status`, `completion_result`, `retry_count`, `next_retry_at`, `last_failure`, `last_failure_error_code`, `modified_at`, `journal_size`, `target_service_key`), `sys_journal`, `sys_inbox`, `state`, `sys_locks`. Status values: `pending`, `scheduled`, `ready`, `running`, `paused`, `backing-off`, `suspended`, `completed`. `retry_count` counts attempts since the current leader started executing, not a global counter. `modified_at` is the last status transition. — [SQL introspection reference](https://docs.restate.dev/references/sql-introspection.md)
- [docs] Documented stuck-detection queries: `restate invocations list --status backing-off`; `select * from sys_invocation where modified_at <= now() - interval '1' hour;`; blocked key: `select acquired_by from sys_locks where lock_name = 'MyServiceName/myKey' and scope is null;`. Another object's state: `select * from state where service_name = '...' and service_key = '...'` or `restate state get`. — [Introspection](https://docs.restate.dev/services/introspection.md)
- [docs] Interventions (CLI and `PATCH http://localhost:9070/invocations/<id>/...`): `cancel`, `kill` (last resort, no compensation), `pause`, `resume` (optionally `--deployment latest` to move to new code), `restart-as-new` (completed invocations only; new ID, same input and headers, no partial progress; for workflows UI-only), "restart from prefix" (keeps part of the journal; UI only), `purge`. — [Managing invocations](https://docs.restate.dev/services/invocation/managing-invocations.md)
- [docs] Restate pauses an invocation automatically "if an invocation retries for too many times" (default 70 attempts, `on-max-attempts = "pause"`); it stays paused until resumed. — [Managing invocations](https://docs.restate.dev/services/invocation/managing-invocations.md); [Service configuration](https://docs.restate.dev/services/configuration.md)
- [docs] Shared handlers on the Worker object can be called concurrently with its running exclusive handler and can read its state. — [Services](https://docs.restate.dev/foundations/services.md)
- [repo] Reference agent schedules: "one-off or recurring messages, with a policy to queue, steer, or interrupt when the agent is busy." — [restatedev/agent README](https://raw.githubusercontent.com/restatedev/agent/main/README.md)
- [vendor] Restate 1.7 (7 Jul 2026) revamped tracing so spans are published as they are ready ("when work takes days, you want live visibility"), and added a prototype flow-control UI showing an invocation's blocking condition. — [Restate 1.7 announcement](https://restate.dev/blog/announcing-restate-1-7)

### Inferences
- A Watchdog can be a Virtual Object with a delayed self-send loop. It can read Worker state through shared handlers (typed, inside Restate) or through the admin SQL endpoint called inside `ctx.run`, and intervene with `ctx.cancel`, a signal to the Worker's turn invocation, or a send; "restart" of a running Worker is cancel plus a new send, because restart-as-new applies only to completed invocations.
- Unlike pi-durable on Node, no separate OS-level relay is needed for another session's state, since all objects live in the same Restate server; but a supervisor for the Restate server process and the service process is still needed when self-hosting.
- `modified_at` tracks status transitions, so "running but making no progress" (a live LLM loop that drifts) is not visible from status alone; progress judgement needs application state (e.g. last-step timestamp) or `journal_size` deltas.

### Gaps
- Whether a handler can query introspection other than by an HTTP call to the admin port is not stated. — [Introspection](https://docs.restate.dev/services/introspection.md)
- Admin API authentication for a self-hosted server was not covered in the fetched pages.

## 7. Event ingestion: HTTP ingress, Kafka, idempotency; can webhooks hit Restate directly

### Takeaway
Any handler is callable over the HTTP ingress (port 8080) including fire-and-forget `/send` with optional delay and an `Idempotency-Key` header (24 h dedup by default). A webhook provider can therefore POST to Restate directly only if it can target that URL shape and JSON body; signature verification and deriving a dedup key from the payload were not documented at the ingress, which points to a thin receiver in front or verification inside a handler.

### Cited Findings
- [docs] URL shapes: `localhost:8080/restate/call/MyService/myHandler`, `.../restate/call/MyVirtualObject/myObjectKey/myHandler`, `.../restate/call/MyWorkflow/myWorkflowId/run`; replace `call` with `send` for fire-and-forget; `?delay=10s` or ISO 8601 on `/send`. Restate 1.6 and earlier used paths without the `/restate` prefix. "Request/response bodies should be encoded as JSON." — [HTTP invocation](https://docs.restate.dev/services/invocation/http.md)
- [docs] `Idempotency-Key` header: "Restate persists the response for a retention period of one day (24 hours)"; tunable. From Restate 1.7 the ingress injects its own idempotency keys for internal retries, which "will not" deduplicate your ingress requests; you must supply your own key. — [HTTP invocation](https://docs.restate.dev/services/invocation/http.md)
- [docs] `GET /restate/attach/{invocationId}`, `GET /restate/output/{invocationId}`, `POST /restate/lookup` for idempotent targets. Ingress 8080; admin/UI 9070. — [HTTP invocation](https://docs.restate.dev/services/invocation/http.md)
- [docs] Services are public on the ingress by default; `ingressPrivate: true` hides a service. — [Service configuration](https://docs.restate.dev/services/configuration.md)
- [docs] Kafka subscriptions: each record becomes an invocation; for Virtual Object handlers the record key is the object key; exclusive handlers receive events in topic-partition order; headers include `kafka.offset`, `kafka.partition`. The page is marked as "the old Kafka support which we plan to remove in future releases"; a standalone Kafka ingress integration is recommended for new projects. — [Kafka invocation](https://docs.restate.dev/services/invocation/kafka.md)
- [release] Server v1.8.1 (21 Oct 2026) changes Kafka subscription deletion to ease migration to the Kafka ingress integration; v1.8.0 drops client-supplied `x-restate-*` headers. — [restate releases](https://github.com/restatedev/restate/releases)
- [release] SDK v1.18.0 adds an experimental integration client package; v1.18.1 adds `connect()` to serve over an outbound connection to Restate Cloud. — [sdk-typescript releases](https://github.com/restatedev/sdk-typescript/releases)

### Inferences
- GitHub/IM webhooks carry their dedup ID in a provider-specific header or body field and need HMAC verification on the raw body; neither is documented as an ingress feature, so the usual shape is a small HTTP receiver (or a Restate handler doing the check) that verifies, then calls `/send` with `Idempotency-Key` = delivery ID. Alternatively dedup by storing seen IDs in Virtual Object state, which has no 24 h window.
- Kafka is optional and adds a broker; not needed for a single-developer setup.

### Gaps
- Ingress authentication for self-hosted servers, raw (non-JSON) body handling at ingress, and header access inside handlers were not confirmed from the fetched pages: unconfirmed.

## 8. LLM agent support as of November 2026

### Takeaway
Restate documents integrations for Vercel AI SDK (TypeScript) and, in Python only, OpenAI Agents SDK, Google ADK, Pydantic AI and LangChain, plus a pattern catalogue and a TypeScript reference agent (`restatedev/agent`) that already implements queued input, steering, interrupts, sub-agents, schedules, memory and compaction. The integrations themselves make LLM/tool steps durable; transcript storage, steering, compaction and the session inbox are application code (or taken from the reference agent), and token streaming is not native.

### Cited Findings
- [docs] Integrations listed: any LLM SDK in a custom loop (OpenAI, Anthropic, Vercel AI SDK, Google Gen AI, LiteLLM), and frameworks Vercel AI SDK, OpenAI Agents SDK, Google ADK, Pydantic AI, LangChain. No maturity labels given. Mastra is not listed. — [AI / Agents with Restate](https://docs.restate.dev/ai/index.md)
- [docs] Language coverage: TypeScript = Vercel AI (`@restatedev/vercel-ai-middleware`, `durableCalls`) and plain `@restatedev/restate-sdk`; Python = OpenAI Agents (`restate.ext.openai`: `DurableRunner`, `durable_function_tool`), Google ADK (`restate.ext.adk`: `RestatePlugin`), Pydantic AI (`restate.ext.pydantic`: `RestateAgent`), LangChain (`restate.ext.langchain`: `RestateMiddleware`), plain `restate` with LiteLLM. Package versions not stated. — [Durable Agents](https://docs.restate.dev/ai/patterns/durable-agents.md)
- [docs] Pattern pages: Durable Agents, Durable Sessions, Approvals with Pause & Resume, Multi-Agent Orchestration, Observability & Control, Parallel Tool Calls, Orchestrator-Worker, Workflows as Tools, Remote Agents, Racing agents, Interrupt & Regenerate, Retries & Error Handling, Rollback, Streaming Responses. — [docs index](https://docs.restate.dev/llms.txt)
- [docs] Sessions: OpenAI Agents has `use_restate_session=True` to persist history automatically (Python); in TypeScript the history is read and written by application code with `ctx.get`/`ctx.set`. The page recommends external memory systems such as mem0 or graphiti for specialised memory. — [Durable Sessions](https://docs.restate.dev/ai/patterns/sessions.md)
- [docs] Streaming: a TypeScript pubsub library (a pubsub Virtual Object registered on the endpoint) pushes updates from services to frontends over SSE; the agent publishes with an idempotency key so replays do not duplicate events; example is a Next.js chat app. — [Streaming Responses](https://docs.restate.dev/ai/patterns/streaming-responses.md) (content seen via search summary only, page not fetched in full)
- [repo] `restatedev/agent`: "A durable AI agent on Restate: turns survive crashes and new versions, with human approvals, sandboxed code, sub-agents, schedules and memory." TypeScript, Node 22+, pnpm; 15 stars, 4 forks, 307 commits on `main` when read; described as a "reference architecture" / "starting point". "User accounts, OAuth flows, and evals are not included." The docs URL `docs.restate.dev/ai/reference-architecture` redirects to this repo. — [restatedev/agent](https://github.com/restatedev/agent)
- [repo] It provides `ask` (start or queue), `steer` ("Redirect the running turn without cancelling its tools"), `interrupt`; memory retrieved on demand; compaction (before each model call, past 60% of the context window the newest messages fitting 15% are kept verbatim and older ones summarised; a conversation summary runs after a turn once 32 or more messages are uncompacted); sandboxes (local workspace or Modal); tools from built-ins, MCP servers and Restate handlers; a UI on HTTP long-polling with revision tags (not token streaming). Models: OpenAI default, also Anthropic, Google, xAI, DeepSeek, OpenAI-compatible servers. — [restatedev/agent README](https://raw.githubusercontent.com/restatedev/agent/main/README.md); [agent docs/turn-runtime.md](https://raw.githubusercontent.com/restatedev/agent/main/docs/turn-runtime.md)
- [vendor] Blog posts: Google ADK integration (12 Jan 2026), Pydantic AI integration (2 Apr 2026). — [ADK post](https://restate.dev/blog/build-resilient-ai-agents-with-restate-and-google-adk); [Pydantic AI post](https://restate.dev/blog/durable-orchestration-for-ai-agents-with-restate-and-pydantic-ai)
- [3p] Pydantic published its own article on the Restate integration (12 May 2026 per search result). — [pydantic.dev](https://pydantic.dev/articles/restate-durable-execution-pydanticai.md)

### Inferences
- What the core SDK plus Vercel middleware do NOT provide in TypeScript: a transcript store (you write state keys), a session inbox with steer/follow-up modes (you build controller + signal routing), context compaction, token streaming to the model consumer (journaled steps return complete results; streaming is a side channel via pubsub), and tool-approval UI. The reference agent supplies most of these as copyable code, not as a versioned library; no npm package for it was seen (unconfirmed).
- Because `ctx.run` journals a completed result, an LLM call interrupted mid-stream is re-issued from scratch on retry; partial output is not resumed.
- For a TypeScript developer, the OpenAI Agents SDK integration is not usable (Python only as documented).

### Gaps
- Licence, release cadence and last commit date of `restatedev/agent` were not visible: unconfirmed.
- Version and stability label of `@restatedev/vercel-ai-middleware`: not found.
- Whether an OpenAI Agents SDK integration for TypeScript exists: not listed in the docs fetched; treat as absent/unconfirmed.

## 9. Hosting and cost for a single developer

### Takeaway
Self-hosting means running one Restate server binary (or Docker image) with a persistent volume, plus your own Node process serving the handlers; no external database is required for a single node. The server is licensed under BSL 1.1 (not an OSI open-source licence; hosting it as a public Restate platform service is excluded; converts to Apache 2.0 after 4 years). Restate Cloud has a free tier of 100k durable actions per month and paid plans "from $75/mo"; detailed per-action rates were not published on the fetched pages.

### Cited Findings
- [docs] "Restate is distributed as a single binary that implements all features required to run a single- or multi-node cluster". Requirements: compute, "a persistent volume attached to store Restate's state" (default directory `restate-data`), and for clusters an object store for snapshots (S3, MinIO). Metadata store is built in (Raft). A single node suits development, testing, or workloads that tolerate short downtime. — [Self-hosted overview](https://docs.restate.dev/server/overview.md)
- [docs] Memory defaults: RocksDB 2 GiB, query engine 1 GiB, record cache 250 MiB, invoker pool 1.5 GiB. A "minimal development setup" totals about 1.5 GiB (1 GiB RocksDB, 256 MiB query engine, 128 MiB record cache); example sizing for a 4 GiB container is given. — [Server memory](https://docs.restate.dev/server/memory.md)
- [docs] Ports: ingress 8080, admin API/UI 9070. — [HTTP invocation](https://docs.restate.dev/services/invocation/http.md)
- [repo] Licence: Business Source License 1.1; licensor Restate Software, Inc. and Restate GmbH; Additional Use Grant: "You may make use of the Licensed Work, provided that you may not use the Licensed Work for a Public Restate Platform Service."; Change Date 4 years after release; Change License Apache 2.0. — [restate LICENSE](https://github.com/restatedev/restate/blob/main/LICENSE)
- [vendor] The pricing page describes the server as "open source under a permissive license" allowing production use and free self-hosting; this wording conflicts with the LICENSE file, which states BSL "is not an Open Source license". — [Pricing](https://restate.dev/pricing/); contradicted in wording by [LICENSE](https://github.com/restatedev/restate/blob/main/LICENSE)
- [vendor] Restate Cloud: "The Free plan includes 100k durable actions per month with community support", no credit card. Paid: "From $75/mo", usage-based with volume discounts; usage beyond the included allotment billed per million actions (rate not shown in fetched text). Payload-carrying actions consume "one action per started 64KiB of data". BYOC (AWS or GCP) is capacity-priced, by request. — [Pricing](https://restate.dev/pricing/); [Restate Cloud](https://restate.dev/cloud)
- [3p] One comparison site states a free tier of 50,000 actions per month, which conflicts with the vendor's 100k; treat the third-party figure as outdated or wrong. — [aitoolsatlas](https://aitoolsatlas.ai/tools/restate/pricing)
- [release] Server: v1.8.0 (14 Oct 2026) "VQueues On by Default", service protocol v7 default, protocol v3 and older dropped, GCS for cluster metadata, CLI `--json`/`--dry-run`; v1.8.1 (21 Oct 2026) drop-in; v1.7.14 (22 Oct 2026) security release for the 1.7 line; v1.8.1-hotfix.2 (2 Nov 2026) without notes. — [restate releases](https://github.com/restatedev/restate/releases)
- [release] TypeScript SDK: v1.18.3 (4 Nov), v1.18.0 (28 Oct), v1.17.0 (31 Aug, `restate.iface` service contracts); releases page has nine pages of history. — [sdk-typescript releases](https://github.com/restatedev/sdk-typescript/releases)
- [3p] Restate announced a $20M Series A on 30 Sep 2026 (secondary coverage). — [remio.ai](https://www.remio.ai/post/restate-series-a-pits-a-20m-bet-against-database-backed-durable-execution)

### Inferences
- Minimal self-hosted topology for one developer: one Restate server process with a data directory, one Node service process (handlers), each under a process supervisor; the service process is stateless and can be restarted freely, the server's data directory is the single point to back up.
- A single-node server has no replication; durability equals the durability of that volume.
- On Cloud, a chatty agent's action count grows with every `ctx.run`, state read/write and message, and large LLM payloads count per started 64 KiB; without the per-million rate the monthly cost cannot be computed from these notes.

### Gaps
- Per-million action price, included allotments of paid plans, Cloud retention and throughput limits: not on fetched pages (unconfirmed).
- Licence of the TypeScript SDK (believed MIT, not fetched): unconfirmed.
- Minimum Node version for SDK 1.18 and whether the service must be served over HTTP/2: not fetched; unconfirmed.
- Backup/restore procedure for a single node was not read.

## 10. Published examples of long-lived, always-on, multi-session agents on Restate

### Takeaway
The closest published artefact is Restate's own `restatedev/agent` reference (per-agent controller + session objects, sub-agents, schedules). No write-up, first- or third-party, was found of a Router/Worker/Watchdog system with an always-on router, detached workers reporting back asynchronously and an external supervising agent.

### Cited Findings
- [repo] `restatedev/agent`: persistent agents with queued input, steering, interrupts, sub-agents ("persistent agents with their own history, memory, and sandbox"), schedules and memory; sub-agents are awaited by the parent's turn and cannot themselves create children. — [restatedev/agent](https://github.com/restatedev/agent); [agent docs/tools.md](https://raw.githubusercontent.com/restatedev/agent/main/docs/tools.md)
- [docs] Multi-Agent Orchestration: routing/handoff examples; a Virtual Object stores the last agent used; plain-SDK examples use a stateless `AgentRouter` service; all request-response. — [Multi-Agent Orchestration](https://docs.restate.dev/ai/patterns/multi-agent.md)
- [docs] An "Orchestrator-Worker" workflow pattern page and "Remote Agents" page exist (titles only; not fetched). — [docs index](https://docs.restate.dev/llms.txt)
- [docs] An AI examples page exists pointing to example code (not fetched). — [AI Examples](https://docs.restate.dev/ai/ai-examples.md)
- [vendor] "What is an agent runtime" article, updated 31 Jul 2026 (not fetched in full). — [restate.dev](https://restate.dev/what-is-an-agent-runtime)
- Search for an always-on personal agent with router/worker layout on Restate returned no matching example. — [Streaming Responses](https://docs.restate.dev/ai/patterns/streaming-responses.md) was the nearest hit.

### Inferences
- Relative to the Super-agent article, the reference agent covers Router input handling (queue, steer, interrupt), compaction, schedules and sub-agents, but its sub-agents are tool calls awaited inside the parent's turn; Thread subscriptions with pass-through, trigger filtering, an LLM-driven Watchdog and an endpoint daemon for remote machines are not present in what was read.
- Remote command execution on user machines is outside Restate's scope; the reference agent's sandboxes are a local workspace or Modal.

### Gaps
- No third-party engineering write-up of such a system on Restate was found within the search budget; absence of evidence only.
- The Orchestrator-Worker, Remote Agents, Observability & Control and AI Examples pages were not fetched; they may contain closer shapes.
