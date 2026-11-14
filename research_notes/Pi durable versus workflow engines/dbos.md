# DBOS (DBOS Transact, TypeScript) as a substrate for a long-lived, event-driven LLM agent system

Research notes, 14 November 2026. Sources were read on that day. Each finding
is tagged **[docs]** (official DBOS documentation or dbos.dev), **[source]**
(dbos-inc GitHub repository, GitHub API or npm registry), or **[third party]**.
Anything not confirmed is marked **unconfirmed**. Nothing was installed or run.

Version context: `@dbos-inc/dbos-sdk` 5.5.0 (release v5.5, 10 November 2026).

## 1. Execution model: what is durable, determinism, steps, recovery and side effects

### Takeaway
DBOS is a library linked into the application process. It records the output of
every workflow step in Postgres and, after a crash, re-runs the workflow
function and replays recorded step outputs up to the last completed step. LLM
and tool calls must be steps; steps are at-least-once, so external side effects
still need their own idempotency.

### Cited Findings
- **[docs]** Workflow functions must be deterministic. Database access, API calls, random numbers and local time belong in steps. — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial)
- **[docs]** After an interruption "the workflow automatically resumes execution from the last completed step". — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial)
- **[docs]** Steps are "tried at least once but are never re-executed after they complete". Transactions (datasource steps against the application's Postgres) "commit exactly once". — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial)
- **[docs]** An uncaught exception in the workflow function sets status `ERROR`; DBOS does not recover such a workflow. — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial)
- **[docs]** If an execution records different results for the same step position, DBOS throws `DBOSStepNondeterminismError`. Parallel steps are allowed only if started in a fixed order; `DBOS.reserveStep` reserves step positions for concurrent step sequences (added in v5.4). — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial), [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[docs]** A workflow ID must be unique per application and "acts as an idempotency key": a second start with the same ID does not run again. Default reuse policy returns a handle to the existing workflow; `workflowIDReusePolicy: 'reject'` throws `DBOSWorkflowIDInUseError`. — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial), [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** `DBOS.sleep()` is durable: the wake-up time is stored and survives restarts. — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial)
- **[docs]** What is stored: `workflow_status` (one row per execution), `workflow_input`, `workflow_output`, `operation_outputs` (output of each step, including `DBOS.sleep`, `DBOS.send` and child workflow starts), `notifications` (messages, with a consumed flag), `workflow_events` and `workflow_events_history`, `streams`, `workflow_schedules`, `queues`, `application_versions`. — [System tables](https://docs.dbos.dev/explanations/system-tables)
- **[docs]** Default serialization in TypeScript is SuperJSON; a custom serializer can be configured. Step outputs, inputs and messages must therefore be serializable. — [Configuration reference](https://docs.dbos.dev/typescript/reference/configuration)
- **[docs]** Single-process recovery: on each restart DBOS recovers all `PENDING` workflows. With several processes and no Conductor, each needs an executor ID and recovers only its own workflows. — [Workflow recovery](https://docs.dbos.dev/production/workflow-recovery)
- **[source]** `DEFAULT_MAX_RECOVERY_ATTEMPTS = 100`; a per-workflow `maxRecoveryAttempts` option exists ("before sending to dead-letter queue"); the resulting status is `MAX_RECOVERY_ATTEMPTS_EXCEEDED`. — [src/workflow.ts on main](https://github.com/dbos-inc/dbos-transact-ts/blob/main/src/workflow.ts)
- **[docs]** `DBOS.resumeWorkflow` restarts a cancelled or `MAX_RECOVERY_ATTEMPTS_EXCEEDED` workflow from its last completed step. — [Methods reference](https://docs.dbos.dev/typescript/reference/methods)

### Inferences
- An LLM call interrupted mid-flight is re-issued from scratch on recovery (step at-least-once); nothing records a partial model response. The Vercel AI integration states this explicitly for streams (section 8).
- A tool that runs a remote command, opens a pull request or posts a comment is an ordinary step: it can run twice if the process dies between the side effect and the checkpoint. Exactly-once applies only to database transactions in the application's own Postgres. This is the same class of problem as pi-durable's `replay: safe` distinction, but DBOS has no built-in "do not rerun, report interrupted" mode; unconfirmed whether step retry configuration can express that.
- The determinism rule means the agent loop itself (deciding what to do next) must depend only on step outputs and received messages.

### Gaps
- Exact per-step size limit for serialized outputs: none stated in the pages read. Postgres limits apply (inference).
- Whether `maxRecoveryAttempts` default of 100 is documented anywhere in the TypeScript docs: not found in the methods or configuration pages.

## 2. Storage and what runs besides the application process

### Takeaway
The TypeScript library requires Postgres as its system database; SQLite is not supported in TypeScript (it is in Python and Go). Nothing else is required: no orchestration server. Conductor is optional and paid.

### Cited Findings
- **[docs]** TypeScript `systemDatabaseUrl` is "a connection string to a Postgres database in which DBOS can store internal state"; default `postgresql://postgres:dbos@localhost:5432/[application name]_dbos_sys`; DBOS tries to create the database if missing. SQLite is not mentioned. — [Configuration reference](https://docs.dbos.dev/typescript/reference/configuration)
- **[source]** Root `package.json` of dbos-transact-ts depends on `pg` and no SQLite driver; the `src` tree contains `system_database.ts` and no SQLite-named file; engines `node >= 20`. — [dbos-transact-ts](https://github.com/dbos-inc/dbos-transact-ts), [npm registry](https://registry.npmjs.org/@dbos-inc/dbos-sdk/latest)
- **[docs]** The system tables page describes PL/pgSQL functions in "a PostgreSQL system database". — [System tables](https://docs.dbos.dev/explanations/system-tables)
- **[docs]** The AI quickstart says "SQLite is the default with no setup" and Postgres is recommended for production. This sentence applies to Python: Python defaults to SQLite; Go accepts a `sqlite:` URL. — [AI quickstart](https://docs.dbos.dev/ai/ai-quickstart), [Python database connection](https://docs.dbos.dev/python/tutorials/database-connection), [Go configuration](https://docs.dbos.dev/golang/reference/configuration)
- **[docs]** The Vercel AI integration "requires ... a Postgres database for DBOS". — [Vercel AI integration](https://docs.dbos.dev/integrations/vercel-ai)
- **[docs]** v5.0 (16 September 2026) removed the in-memory queues and scheduler and the admin server; the internal admin server was deprecated and off by default from v4.25. — [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[docs]** Queues, schedules, messages and versions are all rows in the system database; no separate broker or scheduler process is described. — [System tables](https://docs.dbos.dev/explanations/system-tables), [Scheduled workflows](https://docs.dbos.dev/typescript/tutorials/scheduled-workflows)

### Inferences
- Minimum self-hosted footprint: one Node process plus one Postgres instance. Compared with pi-durable on Node (one process plus one SQLite file), the extra component is the Postgres server.
- Because state is in Postgres, several processes (for example a separate Watchdog process or an ingest process using `DBOSClient`) can read and write the same system database concurrently. This differs from pi-durable's single-writer storage.

### Gaps
- Whether SQLite support for TypeScript is planned: not found. Treat "TypeScript is Postgres-only" as confirmed for v5.5 by docs and package dependencies, not by an explicit statement.
- Minimum Postgres version: not checked.

## 3. Input during a running workflow: send/recv, events, streams

### Takeaway
`DBOS.send` writes a message to a per-workflow, per-topic durable queue in Postgres regardless of what the receiver is doing; `DBOS.recv` consumes one message at a time, FIFO. A session inbox that accumulates during an LLM step and is drained as a batch can be built by looping `recv` with a short timeout, but there is no "drain all" call and no built-in steering of a running step.

### Cited Findings
- **[docs]** `DBOS.send(destinationID, message, topic?, idempotencyKey?, options?)`: messages are "queued on the receiver per topic"; "all messages are persisted to the database". Sending from inside a workflow is exactly-once; from ordinary code an idempotency key gives send-once. — [Workflow communication](https://docs.dbos.dev/typescript/tutorials/workflow-communication), [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** `DBOS.recv<T>(topic?, options?)` is workflow-only, dequeues FIFO, returns `null` on timeout (default 60 s); options `timeoutSeconds`, `deadlineEpochMS`, `pollingIntervalMs`, `suspend`. "If a message is already waiting, `recv` returns it immediately." — [Methods reference](https://docs.dbos.dev/typescript/reference/methods), [Workflow communication](https://docs.dbos.dev/typescript/tutorials/workflow-communication)
- **[docs]** `suspend: true` on `recv`/`sleep` releases memory and queue capacity while waiting (status `SUSPENDED`); this is an Enterprise feature requiring `suspendMode`. Without it the waiting workflow stays in memory in the process. — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial), [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** Messages can be sent from outside the application with `DBOSClient` (another process, given the database URL). — [Workflow communication](https://docs.dbos.dev/typescript/tutorials/workflow-communication)
- **[docs]** `DBOS.setEvent(key, value)` publishes or updates a keyed value on the workflow; `DBOS.getEvent(workflowID, key, timeout)` reads it from anywhere (default timeout 60 s). Past values are kept in `workflow_events_history`. — [Workflow communication](https://docs.dbos.dev/typescript/tutorials/workflow-communication), [System tables](https://docs.dbos.dev/explanations/system-tables)
- **[docs]** Streams: `DBOS.writeStream(key, value)`, `closeStream`, `readStream(workflowID, key)` (async generator, readable from other processes via `DBOSClient`). Append-only; exactly-once when written from a workflow, at-least-once from a step. — [Workflow communication](https://docs.dbos.dev/typescript/tutorials/workflow-communication)
- **[docs]** v4.24 added "send idempotency keyed by recipient"; v5.5 fixed a recorded-receive check to verify workflow ownership. — [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[docs]** Cancellation does not interrupt a running step; a step can observe `DBOS.stepStatus.cancelSignal` (an `AbortSignal`) and stop early. — [Workflow management](https://docs.dbos.dev/typescript/tutorials/workflow-management)
- **[third party]** Helix Agents' DBOS runtime implements exactly this shape: "persistent mode" is one long-lived workflow per session blocking on `DBOS.recv()` between turns; later messages arrive by `DBOS.send(workflowId, msg, 'inbox')`; wake-up uses Postgres LISTEN/NOTIFY. — [Helix Agents DBOS runtime](https://agents.open-source.onhelix.ai/runtimes/dbos.html)

### Inferences
- Queue-while-busy: confirmed by composition, not by an explicit sentence. Messages are persisted on `send` independently of the receiver's state, and `recv` returns immediately when one is waiting, so messages sent during an LLM step sit in `notifications` until the next `recv`.
- Batch drain: no documented "receive all" call. The application loops `recv` with a zero or very short timeout until `null`. Each `recv` is itself a recorded step, so a batch of N messages costs N+1 checkpoints. Unconfirmed: whether a timeout of 0 is accepted.
- Mid-turn steering (pi-durable's `whenBusy: steer`): not a DBOS concept. The agent loop is application code, so it can call `recv` between model/tool steps and inject messages there. Injection during a single in-flight LLM step is not possible; aborting the step would need application logic around `cancelSignal` or an `AbortController`.
- `setEvent`/`getEvent` gives the Watchdog or UI a cheap "live state" read (comparable to pi-durable's `pi.live` document), but the content must be published by the application.

### Gaps
- Ordering guarantee wording: the reference says FIFO per topic; the tutorial only says "next message to arrive". No statement on ordering across concurrent senders.
- Any size or count limit on unconsumed messages: none stated.

## 4. Months-long workflows: limits, never-ending sessions, versioning

### Takeaway
No documented limit on step count or duration was found, and DBOS has no documented "continue-as-new". The real constraints for a months-long workflow are code upgrades (recovery is tied to the application version unless changes are patched) and unbounded growth of the step history, since each loop iteration adds rows that are kept while the workflow is alive.

### Cited Findings
- **[docs]** The workflow tutorial states no limit on steps or duration beyond optional timeouts. Timeouts are start-to-completion, durable, and on expiry "the workflow and all its children are cancelled". — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial)
- **[docs]** DBOS marketing/AI docs describe agents that run "for hours, days, or weeks"; the HITL page says a `recv` wait "can safely wait for a long time". No maximum is given. — [AI quickstart](https://docs.dbos.dev/ai/ai-quickstart), [Human-in-the-loop](https://docs.dbos.dev/ai/hitl)
- **[docs]** "A breaking change to a workflow is one that changes which steps are run, or the order in which the steps are run." Two upgrade strategies: patching and versioning. — [Upgrading workflows](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows)
- **[docs]** Versioning: the application version defaults to a hash of workflow source code (can be set manually). "When DBOS tries to recover workflows, it only recovers workflows whose version matches the current application version." For long-running workflows the docs recommend blue-green deployment: old processes drain old workflows. — [Upgrading workflows](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows)
- **[docs]** Patching: `DBOS.patch()` returns `true` for new runs and `false` for runs that started before the patch; it inserts a patch marker into history. `DBOS.deprecatePatch` is required for a transition period before removal; mistakes throw `DBOSUnexpectedStepError`. Recovery of old workflows by new code needs a pinned application version or auto-upgrade. — [Upgrading workflows](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows)
- **[docs]** Auto-upgrade (moving `PENDING`/`ENQUEUED`/`SUSPENDED` workflows to the latest version) is an Enterprise feature. — [Upgrading workflows](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows)
- **[docs]** `DBOS.forkWorkflow(workflowID, startStep, { applicationVersion })` creates a new workflow that reuses recorded steps before `startStep` and can target a newer version; `DBOS.rewindWorkflow` keeps the ID and discards steps from a point (workflow must be terminal). — [Workflow management](https://docs.dbos.dev/typescript/tutorials/workflow-management), [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** `workflow_input`, `workflow_output` and `operation_outputs` carry a `retention_timestamp` "used when applying retention policies"; retention policies are listed as a Conductor (paid) feature. `DBOS.deleteWorkflow` permanently deletes a workflow. v5.0 "optimized retention and garbage collection". — [System tables](https://docs.dbos.dev/explanations/system-tables), [Pricing](https://www.dbos.dev/dbos-pricing), [Methods reference](https://docs.dbos.dev/typescript/reference/methods), [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[third party]** Helix Agents reports for its DBOS runtime: the loaded conversation history is stored as a step output on every step, so storage grows roughly O(N²) for very long sessions (tracked as a follow-up); a release that changed step order without `DBOS.patch` markers made pending workflows unreplayable; "persistent-mode workflows never finish on their own and must be cancelled before rollout". — [Helix Agents DBOS runtime](https://agents.open-source.onhelix.ai/runtimes/dbos.html)
- **[third party]** Helix's default ("standard") mode avoids the long-lived workflow: one short workflow per turn that loads accumulated session state from a store. — [Helix Agents DBOS runtime](https://agents.open-source.onhelix.ai/runtimes/dbos.html)

### Inferences
- A Router modelled as one workflow that loops for months accumulates one `operation_outputs` row per step forever, and recovery after a restart replays the function from the top through every recorded step. Unconfirmed: recovery time for a workflow with, say, 10^5 recorded steps; no figure was found.
- With the default source-hash version, any code change gives a new version, and the old Router workflow is then not recovered by the new process. A months-long session therefore needs a manually pinned version plus `DBOS.patch` discipline, or a per-turn design.
- The two workable patterns are (a) per-turn workflows with the transcript in an application table (Helix standard mode; equivalent to "continue-as-new" by hand), or (b) a generation scheme: the loop ends after N turns and starts a successor workflow with a new ID, moving unconsumed messages. Pattern (b) needs care because messages are addressed to a workflow ID. Neither is prescribed by DBOS docs.
- In pattern (a) the "inbox while busy" must be an application table or a DBOS queue with a per-session partition/concurrency of 1, since there is no long-lived workflow to `send` to. The `Debouncer` (per-key delay, runs with the last arguments) batches by replacement, not accumulation, so it does not by itself deliver "all queued events in one turn".

### Gaps
- No official statement on maximum step count, history size or recovery cost for very long workflows was found. A search for a DBOS "continue as new" equivalent returned nothing official.
- The workflow retention docs page (`/production/workflow-retention`) returned 404, so retention options and whether a garbage-collection API exists in open-source TypeScript are unconfirmed.

## 5. Workers as child workflows

### Takeaway
Child workflows, handles, durable queues with concurrency limits, cancellation, resume and fork are all first-class. Reporting back to a parent is done either by awaiting the handle or by `DBOS.send` to the parent's workflow ID; the second matches the article's asynchronous `start()`.

### Cited Findings
- **[docs]** `DBOS.startWorkflow(target, params)(...args)` starts a workflow in the background and returns a `WorkflowHandle` once it is durably started. Called from a workflow without an explicit ID, the child's ID is derived deterministically from the parent's. — [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** Handle: `workflowID`, `getResult()`, `getStatus()`. `DBOS.waitFirst` / `DBOS.waitAll` wait on lists of handles. — [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** `WorkflowStatus.parentWorkflowID`; `listWorkflows` filters `parentWorkflowID` and `hasParent`; `StepInfo.childWorkflowID` links a parent step to its child. `workflowAttributes` are not inherited by children. — [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** Queues: per-process worker concurrency, global concurrency across processes, rate limits, priority, delay (`DELAYED` status), deduplication ID (one enqueued workflow per ID; `duplicationPolicy: 'return-existing'` for singletons), partitioned queues with per-partition limits. FIFO by default. — [Queue tutorial](https://docs.dbos.dev/typescript/tutorials/queue-tutorial)
- **[docs]** `DBOS.cancelWorkflow(workflowID, { cancelChildren? })` sets `CANCELLED`, removes from queue, and stops the workflow at the start of its next step; a running step is not interrupted but can watch `cancelSignal`. A workflow timeout cancels the workflow and all its children. — [Methods reference](https://docs.dbos.dev/typescript/reference/methods), [Workflow management](https://docs.dbos.dev/typescript/tutorials/workflow-management)
- **[docs]** `DBOS.resumeWorkflow` restarts from the last completed step; `DBOS.forkWorkflow` starts a new workflow from a chosen step, with `replacementChildren` to remap child IDs. — [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** In the Vercel AI integration, `agentTool` implements subagents "as child workflows of the parent agent workflow"; it must be called before `DBOS.launch()`. — [Vercel AI integration](https://docs.dbos.dev/integrations/vercel-ai)
- **[third party]** Helix runs persistent sub-agents as their own workflows, messages them with `DBOS.send`, and waits for results by polling with durable `DBOS.sleep`. — [Helix Agents DBOS runtime](https://agents.open-source.onhelix.ai/runtimes/dbos.html)

### Inferences
- Worker = workflow with ID such as `worker:<ticket>`, started on a queue whose global concurrency is the worker-slot limit. Non-blocking `start()` = a tool step that calls `startWorkflow` and returns the ID; the Worker's last step does `DBOS.send(routerId, report, 'worker-report')`, which is exactly-once from inside a workflow. `stop()` = `cancelWorkflow`. This is less glue than pi-durable's anchor/reporter pattern, but it presupposes the Router is a `recv`-able workflow (see section 4).
- Whether a child started with `startWorkflow` outlives a cancelled parent depends on `cancelChildren` and timeouts; default of `cancelChildren` is unconfirmed.
- Worker isolation is at workflow level only. Per-Worker directories, sandboxes and model choice are application code, as with pi-durable.

### Gaps
- Default value of `cancelChildren`, and whether a parent's completion affects running children: not stated in the pages read.

## 6. Watchdog: schedules, inspection, intervention

### Takeaway
All Watchdog primitives exist in the open-source library: cron schedules, cross-workflow listing and step inspection, and cancel/resume/fork/send from any process connected to the same Postgres. "Is it making progress" detection and automatic failover of dead executors are not in the free library (the latter is Conductor).

### Cited Findings
- **[docs]** Cron schedules via `DBOS.createSchedule({ scheduleName, workflowFn, schedule, context })`, parsed by node-cron (seconds field supported, `cronTimezone` option). Also `applySchedules`, `updateSchedule`, `pauseSchedule`, `resumeSchedule`, `deleteSchedule`, `backfillSchedule`; manageable from `DBOSClient`. — [Scheduled workflows](https://docs.dbos.dev/typescript/tutorials/scheduled-workflows)
- **[docs]** Each firing is run "by exactly one worker process", with an idempotency key built from schedule name and scheduled time, "ensuring each scheduled invocation occurs exactly once while your application is active". Missed runs are backfilled at startup when `automaticBackfill: true`. — [Scheduled workflows](https://docs.dbos.dev/typescript/tutorials/scheduled-workflows)
- **[docs]** `DBOS.listWorkflows` filters: status (`ENQUEUED`, `DELAYED`, `SUSPENDED`, `PENDING`, `SUCCESS`, `ERROR`, `CANCELLED`, `MAX_RECOVERY_ATTEMPTS_EXCEEDED`), name, ID prefix, time ranges, queue, parent, attributes (JSONB containment), application version, with `loadInput`/`loadOutput`. Non-ID queries have a 30-second `observabilityQueryTimeoutMs`. — [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** `DBOS.listWorkflowSteps(workflowID)` returns per step: `functionID`, `name`, `output`, `error`, `childWorkflowID`, `startedAtEpochMs`, `completedAtEpochMs`. CLI equivalents: `npx dbos workflow list|steps|cancel|resume`. — [Methods reference](https://docs.dbos.dev/typescript/reference/methods), [Workflow management](https://docs.dbos.dev/typescript/tutorials/workflow-management)
- **[docs]** Interventions: `cancelWorkflow`, `resumeWorkflow`, `forkWorkflow` (from a step, optionally on a new application version), `rewindWorkflow`, `deleteWorkflow`; also available from the Conductor web console. — [Workflow management](https://docs.dbos.dev/typescript/tutorials/workflow-management)
- **[docs]** With Conductor, "when Conductor detects that an executor is unhealthy, it signals another executor to recover its workflows". Without it, recovery happens only when the owning process restarts. From v5.3 executor self-recovery is skipped when Conductor is enabled. — [Workflow recovery](https://docs.dbos.dev/production/workflow-recovery), [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[docs]** Custom alerting is listed from the Conductor Teams tier ($499/month). — [Pricing](https://www.dbos.dev/dbos-pricing)
- **[docs]** A DBOS MCP server exists for coding agents. — [AI quickstart](https://docs.dbos.dev/ai/ai-quickstart)

### Inferences
- The Watchdog can be a scheduled workflow in the same process or a separate process with `DBOSClient`. Unlike pi-durable (read-only second opener, interventions must go through the owning process), a second DBOS process can both read and intervene directly, because coordination is in Postgres.
- Steering a Worker = `DBOS.send(workerId, instruction, 'steer')`, provided the Worker loop calls `recv` between steps. Abort = `cancelWorkflow`. Restart = `resumeWorkflow` (same history) or `forkWorkflow` from an earlier step (new ID; the Router's reference to the Worker ID must be updated by the application).
- Stuck detection is application logic: compare `completedAtEpochMs` of the latest step with the current time, or read a heartbeat published with `setEvent`. The transcript is visible only as step outputs (model responses, tool results) unless the application also stores it.
- On a single Node host nothing restarts a crashed process; an external supervisor (systemd, container restart policy) is still needed, same as pi-durable on Node.
- A Watchdog that lives inside the same process dies with it; the article's "independent" Watchdog implies a second process or host.

### Gaps
- Whether the open-source library alone exposes any "workflow has not progressed for X" signal: none found.
- Conductor documentation pages returned empty content or 404 on fetch, so Conductor's exact feature list is taken from the pricing page and the recovery page only.

## 7. Event ingestion and dedup

### Takeaway
DBOS has Kafka consumer packages but no built-in HTTP/webhook server in the TypeScript library; a webhook endpoint is ordinary application code (Express, Fastify, Hono) that starts or messages a workflow. Dedup is native by using the event's delivery ID as workflow ID, queue deduplication ID or `send` idempotency key.

### Cited Findings
- **[source]** The repository's `packages` directory contains `kafkajs-receive` and `confluent-kafka-receive` (Kafka consumers), plus datasource packages (`drizzle`, `knex`, `kysely`, `nodepg`, `postgres`, `prisma`, `typeorm`), `otel`, `create` and `dbos-cloud`. No SQS or HTTP package is present on `main`. — [dbos-transact-ts/packages](https://github.com/dbos-inc/dbos-transact-ts/tree/main/packages)
- **[source]** README feature list includes "Exactly-Once Event Processing", "Durable Queues", "Durable Scheduling", "Durable Notifications". — [dbos-transact-ts](https://github.com/dbos-inc/dbos-transact-ts)
- **[docs]** v5.0 removed "deprecated packages" and the admin server; v5.5 "reintroduced initialize/deactivate event receivers". — [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[docs]** Workflow ID as idempotency key; queue `deduplicationID` ("only one workflow with a specific deduplication ID can be enqueued in the specified queue" at a time; duplicate raises `DBOSQueueDuplicatedError`); `DBOS.send` idempotency key. — [Workflow tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial), [Queue tutorial](https://docs.dbos.dev/typescript/tutorials/queue-tutorial), [Methods reference](https://docs.dbos.dev/typescript/reference/methods)
- **[docs]** `DBOSClient` lets a separate process (for example a stateless webhook receiver) enqueue workflows, send messages and read events with only the database URL. — [Workflow communication](https://docs.dbos.dev/typescript/tutorials/workflow-communication), [Scheduled workflows](https://docs.dbos.dev/typescript/tutorials/scheduled-workflows)
- **[docs]** The HITL example delivers a human reply by an HTTP endpoint that calls `DBOS.send(agent_id, response)` (Python example). — [Human-in-the-loop](https://docs.dbos.dev/ai/hitl)

### Inferences
- Webhook ingestion pattern: HTTP handler verifies signature, applies the code-level allowlist, then `startWorkflow(ingest, { workflowID: 'gh:' + deliveryId })` or `DBOS.send(routerId, event, 'trigger', deliveryId)`. A workflow-ID dedup lasts as long as the workflow row is retained; a queue `deduplicationID` only dedups while the earlier item is still enqueued or running.
- Thread subscriptions and pass-through, two-stage filtering, Inline & Spill and Chain of Recall are application tables and code, as with pi-durable.

### Gaps
- The Kafka tutorial URL tried returned 404; the Kafka packages' exactly-once semantics were not read. "Exactly-Once Event Processing" is from the README headline only.
- "No built-in HTTP server" is inferred from the package list, the removal of the admin server and the absence of any HTTP section in the pages read; no page states it outright. Unconfirmed.

## 8. LLM agent support

### Takeaway
DBOS has no agent framework of its own. Official integrations are mostly Python (Pydantic AI, LlamaIndex, OpenAI Agents SDK, Google ADK); TypeScript has one: `@dbos-inc/vercel-ai` for the Vercel AI SDK, first published July 2026 and at 0.5.4. It makes model and tool calls durable steps and adds durable streams and subagents, but supplies no transcript store, session inbox, steering or compaction.

### Cited Findings
- **[docs]** Official integrations: Pydantic AI (`DBOSAgent`, Python), LlamaIndex (`llama-agents-dbos`, `DBOSRuntime`, Python), OpenAI Agents SDK (`DBOSRunner`, Python), Google ADK (`DBOSPlugin`, Python), Vercel AI SDK (`@dbos-inc/vercel-ai`, TypeScript). The quickstart otherwise says to annotate your own agent loop as a workflow and each LLM/tool call as a step, in Python or TypeScript. No DBOS-native agent framework is described. — [AI quickstart](https://docs.dbos.dev/ai/ai-quickstart)
- **[docs]** `@dbos-inc/vercel-ai` "requires DBOS v4.27+ or v5, AI SDK v7+, and a Postgres database for DBOS". — [Vercel AI integration](https://docs.dbos.dev/integrations/vercel-ai)
- **[source]** `@dbos-inc/vercel-ai` latest is 0.5.4; package created 7 July 2026, last modified 30 September 2026. — [npm registry](https://registry.npmjs.org/@dbos-inc/vercel-ai)
- **[docs]** API: `durableCalls` (model middleware via `wrapLanguageModel`; retries on by default, `maxAttempts` 3, backoff 2), `durableTools` (retries off by default, `timeoutMS`), `durableMCPTools` (MCP listing and calls as steps), `agentTool` (subagents as child workflows), `durableEmbeddingCalls`, `durableImageCalls`, `readDurableStream`, `writeDurableStream`, `toolWriter`, `closeDurableStream`. — [Vercel AI integration](https://docs.dbos.dev/integrations/vercel-ai)
- **[docs]** "This integration automatically checkpoints every action your agents take in Postgres." "If your process is interrupted, DBOS replays your agent from its checkpoints." — [Vercel AI integration](https://docs.dbos.dev/integrations/vercel-ai)
- **[docs]** Streaming: `durableStream` makes output readable from another process as AI SDK `UIMessageChunk`s. On recovery an interrupted model call "restarts the model call and streams its output again"; readers get a `data-dbos-superseded` chunk (and `data-dbos-tool-superseded` for re-executed tools). — [Vercel AI integration](https://docs.dbos.dev/integrations/vercel-ai)
- **[docs]** Caveat: for consistent parallel tool ordering on recovery, "do not await I/O in callbacks that run before a tool executes". — [Vercel AI integration](https://docs.dbos.dev/integrations/vercel-ai)
- **[docs]** The integration page's chat example persists messages with a user-defined `saveMessage` in `onFinish`; no storage facility is provided. The page does not cover steering or message injection during a run. — [Vercel AI integration](https://docs.dbos.dev/integrations/vercel-ai)
- **[docs]** The HITL page (send/recv approval pattern, status via `set_event`) contains Python examples only. — [Human-in-the-loop](https://docs.dbos.dev/ai/hitl)
- **[docs]** TypeScript example: a Hacker News deep-research agent. — [TypeScript Hacker News agent](https://docs.dbos.dev/typescript/examples/hacker-news-agent)
- **[third party]** Helix Agents ships `@helix-agents/runtime-dbos` (MIT). It sends per-token chunks through Redis rather than DBOS streams "because Postgres write latency is too high for that fan-out", and lists gaps of its own (memory auto-injection unsupported, options dropped across step boundaries). — [Helix Agents DBOS runtime](https://agents.open-source.onhelix.ai/runtimes/dbos.html)
- **[third party]** A DEV Community post gives 20 July 2026 as the `@dbos-inc/vercel-ai` launch date; npm shows the package was created 7 July 2026. The launch date is unconfirmed. — [DEV post](https://dev.to/mr_manushukla/durable-ai-agents-without-temporal-exactly-once-workflows-on-postgres-with-dbos-2026-5a6n), [npm registry](https://registry.npmjs.org/@dbos-inc/vercel-ai)

### Inferences
- What DBOS plus the Vercel AI integration does NOT provide, and pi-durable does: a persisted conversation transcript as a first-class object, a session inbox with busy modes (`steer`/`followUp`), context compaction, model/provider abstraction of its own, a remote execution environment (pi-env), and tool replay classes. With DBOS these come from the AI SDK (provider abstraction, tool loop) or are application code (transcript tables, inbox loop, compaction, remote execution over SSH).
- What DBOS provides that pi-durable does not: multi-process access to one store, durable queues with concurrency and rate limits, cron schedules, workflow listing/cancel/resume/fork as public APIs, a CLI and (paid) web console.
- TypeScript coverage is real but young: one integration at 0.x, four months old, versus four Python integrations.

### Gaps
- Token streaming durability granularity (per chunk versus per call) is not described.
- Maturity signals for `@dbos-inc/vercel-ai` (download counts, open issues) were not checked.
- Whether the Vercel AI integration works with a `recv` loop inside the same workflow for multi-turn sessions is not documented; unconfirmed.

## 9. Hosting, cost, licence, SDK maturity for a single developer

### Takeaway
The library is MIT and free; a single developer self-hosting needs only Node 20+ and Postgres and pays nothing to DBOS. Conductor (web console, automatic failover between executors, retention policies) starts at $99/month with no free tier listed; a few features are gated behind an Enterprise key.

### Cited Findings
- **[source]** Licence MIT; about 1.4k GitHub stars, 106 forks; `engines: node >= 20`. — [dbos-transact-ts](https://github.com/dbos-inc/dbos-transact-ts), [npm registry](https://registry.npmjs.org/@dbos-inc/dbos-sdk/latest)
- **[source]** Latest release v5.5, published 10 November 2026 (npm 5.5.0); v5.4 on 28 October 2026; v5.3 on 15 October 2026. — [GitHub releases API](https://api.github.com/repos/dbos-inc/dbos-transact-ts/releases?per_page=3)
- **[docs]** Release cadence: v4.24 (21 July) to v5.5 (10 November), roughly every two weeks; v5.0 (16 September) was a breaking major release with a migration guide (workflow inputs/outputs moved to separate tables; deprecated packages, in-memory queues/scheduler and admin server removed). Year 2026 is confirmed by the API for v5.3 to v5.5 and inferred for the earlier ones. — [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[docs]** From v5.3 the Conductor client is in a separate package `@dbos-inc/dbos-enterprise`; Conductor or Cloud users must install it. — [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[docs]** Pricing page: the open-source DBOS Transact library is "free forever". Conductor Pro $99/month: 2 seats, up to 3 apps, 1 million checkpoints (workflows, steps, transactions) per month, $50 per extra million, retention policies, unlimited connected workers, community support. Teams $499/month: 10 seats, 10 apps, 10 million checkpoints, $40 per extra million, custom alerting, RBAC, SOC 2/HIPAA. Enterprise: custom price, sharded Postgres, SSO, option to self-host Conductor. Pro and Teams have a free trial. DBOS Cloud: "Contact sales", no published prices. — [DBOS pricing](https://www.dbos.dev/dbos-pricing)
- **[docs]** Enterprise-gated library features: `suspend` on `recv`/`sleep`, external object stores (`externalStores`), workflow auto-upgrade. `transactEnterpriseKey` is "for development only"; production Enterprise apps connect to Conductor with a `conductorKey`. — [Configuration reference](https://docs.dbos.dev/typescript/reference/configuration), [Methods reference](https://docs.dbos.dev/typescript/reference/methods), [Upgrading workflows](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows)
- **[docs]** v5.1 added a Conductor "metadata-only mode for local use". — [Releases](https://github.com/dbos-inc/dbos-transact-ts/releases)
- **[third party]** Search-result summary of the DBOS self-hosting docs: self-hosted Conductor is under a proprietary licence, needs a licence key, with a trial via docker-compose; each executor keeps a long-lived WebSocket to Conductor. The docs page itself could not be read (empty response). Unconfirmed. — [Self-hosting Conductor](https://docs.dbos.dev/production/hosting-conductor)
- **[third party]** Aggregator sites disagree on a former free hosted tier (one says a DBOS Cloud free tier no longer appears on the pricing page). Unconfirmed; the official page read today lists none. — [agentdeals.dev](https://agentdeals.dev/vendor/dbos), [DBOS pricing](https://www.dbos.dev/dbos-pricing)

### Inferences
- Cost for the target setup without Conductor: Postgres hosting only. An always-on agent's checkpoint volume matters only if Conductor is used: every step, `recv`, `send` and `sleep` is a checkpoint, so a Router plus Workers doing a few thousand steps a day stays within 1 million per month; heavy polling loops would not.
- Without Conductor the developer gives up: web console, cross-executor failover, managed retention, alerting. The CLI and programmatic management API remain.
- "Workflow stays in memory while waiting in `recv`" (no Enterprise suspend) is acceptable for one Router and tens of Workers in one process; it is a scaling concern, not a correctness one.
- API stability: a breaking v5.0 two months before this note and fortnightly releases indicate a fast-moving SDK, though with a longer production history than pi-durable (which is labelled experimental).

### Gaps
- Whether Conductor has any free tier for a single developer: the pricing page shows only trials; trial length is not stated.
- DBOS Cloud prices are not published.
- Open-source retention/garbage-collection capability without Conductor: unconfirmed (docs page 404).

## 10. Published examples of long-lived, always-on, multi-session agents on DBOS

### Takeaway
No published router/worker/watchdog (Super-agent-shaped) system on DBOS was found. The closest evidence is one third-party agent runtime with a "persistent session" mode and its candid caveats, plus DBOS's own single-agent examples.

### Cited Findings
- **[third party]** Helix Agents `@helix-agents/runtime-dbos`: persistent mode = one long-lived `recv`-loop workflow per session; sub-agents as separate workflows messaged with `DBOS.send`; interrupt = control message plus `DBOS.cancelWorkflow`, settling after 500 ms; HITL = `DBOS.recv(toolCallId)`. Documents O(N²) history growth and the need to cancel persistent workflows before rollouts. No version or date on the page (copyright 2024-2026). — [Helix Agents DBOS runtime](https://agents.open-source.onhelix.ai/runtimes/dbos.html)
- **[docs]** DBOS examples: Hacker News deep-research agent in TypeScript and Python; HITL approval agent (Python); parallel/distributed agents with queues. These are single-task agents, not always-on multi-session systems. — [AI quickstart](https://docs.dbos.dev/ai/ai-quickstart), [TypeScript Hacker News agent](https://docs.dbos.dev/typescript/examples/hacker-news-agent), [Human-in-the-loop](https://docs.dbos.dev/ai/hitl)
- **[third party]** Framework-side documentation of DBOS durability exists for Pydantic AI and LlamaIndex (Python). — [Pydantic AI DBOS](https://pydantic.dev/docs/ai/integrations/durable_execution/dbos/), [LlamaIndex DBOS](https://developers.llamaindex.ai/python/llamaagents/workflows/dbos/)
- **[third party]** Comparison and tutorial posts exist but were not read in full and are not used for facts here. — [Capital & Compute](https://capitalandcompute.net/blog/dbos-durable-execution-agent-harness/), [Developers Digest comparison](https://www.developersdigest.tech/blog/durable-ai-agent-frameworks-compared-2026)

### Inferences
- As with pi-durable, a Super-agent on DBOS would be inference from primitives, not a reproduction of a published build.
- Helix's choice to make per-turn workflows the default and long-lived workflows opt-in is the only field evidence found on which shape is more practical.

### Gaps
- No search was run specifically for "supervisor/watchdog agent on DBOS" blog posts beyond the queries above; absence is not proven.
- The DBOS blog (for example monthly "DBOS Enhancements" posts) was not read; it may contain agent case studies.
