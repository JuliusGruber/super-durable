# Temporal as a substrate for a long-lived, event-driven LLM agent system (state: 14 November 2026)

Source-type labels used below: **[docs]** = official Temporal documentation
(docs.temporal.io), **[release notes]** = GitHub release notes of a Temporal
repository, **[vendor]** = Temporal's own blog, changelog or pricing page,
**[third party]** = anyone else. Nothing was built or run. No Temporal source
code was read; where a fact would need source code to confirm, it is listed
under Gaps as unconfirmed.

## 1. Execution model: what is durable, determinism, where LLM and tool calls live, side-effect semantics

### Takeaway
Temporal persists an append-only Event History per Workflow Execution and recovers state by re-running the workflow function against it, so workflow code must be deterministic and every LLM call, tool call or other I/O must run in an Activity. Activities can run more than once; exactly-once effects need idempotency keys enforced by the called service.

### Cited Findings
- [docs] Event History is "an append-only log of Events for your application", durably persisted by the Temporal Service, and is what lets an execution recover after a crash. — [Events and Event History](https://docs.temporal.io/workflow-execution/event)
- [docs] Replay: Temporal re-executes workflow code to recover state, and "the Commands that are emitted are compared with the existing Event History"; a mismatch fails with a non-determinism error. The code must "make the same Workflow API calls in the same sequence" for the same input. — [Workflow Definition](https://docs.temporal.io/workflow-definition)
- [docs] Workflow code must not branch on local clock time or random numbers, and must not do API calls, LLM calls or database queries directly; that work goes in Activities, which run outside the replay path and retry automatically. — [Workflow Definition](https://docs.temporal.io/workflow-definition)
- [docs] Unsafe code changes for running executions: reordering, adding or removing command-producing calls (Timers, Activities, Child Workflows, external Signals, Nexus operations, completion, Search Attribute upserts), and changing the types or IDs of Child Workflows or Activities. "The Workflow Definition can change in very limited ways once there is a Workflow Execution depending on it." Worker Versioning (recommended) or patching is required. Versioning matters more when executions "live long enough to run on multiple versions of your Worker." — [Workflow Definition](https://docs.temporal.io/workflow-definition)
- [docs] TypeScript: "Workflows run in a deterministic sandboxed environment"; workflow code "can import any package as long as it does not reference Node.js or DOM APIs" and is bundled with Webpack when the Worker is created. `Math.random()`, `Date` and `setTimeout()` "are replaced by deterministic versions"; `FinalizationRegistry` and `WeakRef` are removed; `crypto.randomUUID()` is unavailable (use `uuid4()`). Workflow code cannot import the Activity implementation, only its types. — [TypeScript workflow basics](https://docs.temporal.io/develop/typescript/workflows/basics)
- [docs] "Temporal recommends that Activities be idempotent." "Because Activities may be retried, these functions may be executed more than once." A Worker can finish an Activity and crash before reporting success, so it runs again. The Activity "may be executed multiple times and may even partially complete more than once". Suggested idempotency key: Workflow Run ID plus Activity ID, enforced "by the service you are calling from your Activity, not by the Activity itself." — [Activity Definition](https://docs.temporal.io/activity-definition)
- [docs] Temporal does not detect Activity task loss directly; it relies on the Start-To-Close timeout, after which the retry policy applies. "Activities must heartbeat to receive cancellations from a Temporal Service." Asynchronous completion lets an Activity function return without completing the Activity; an external system completes it later by task token or by Activity ID plus Workflow ID. — [Activity Execution](https://docs.temporal.io/activity-execution)
- [docs] Local Activities run in the Workflow's worker process, "lack full durability" and "are not a replacement for regular Activities." Standalone Activities are top-level executions started by a client, with their own lifetime and no workflow Event History. — [Activity Execution](https://docs.temporal.io/activity-execution)
- [vendor] Standalone Activities became GA "across all six SDKs" on 15 September 2026. — [TypeScript SDK changelog](https://temporal.io/changelog/product-area/typescript-sdk)
- [docs] In the OpenAI Agents integration, Event History stores model requests and responses, tool results and sandbox inputs and outputs; a Payload Codec is recommended for sensitive data. — [OpenAI Agents SDK integration, TypeScript](https://docs.temporal.io/develop/typescript/integrations/openai-agents.md)

### Inferences
- The durable unit is the sequence of recorded results, not a process snapshot: after a crash, an LLM call whose result was recorded is not repeated, while one that was in flight is re-run from the start by the retry policy (a new, possibly different, model response and a second charge).
- A remote command run by a tool Activity has the same "outcome unknown" problem as pi-env when the worker dies mid-call; Temporal offers heartbeating, timeouts and a stable (Run ID, Activity ID) key, but the remote side must enforce the key.
- A Router or Worker session that lives for months will span many deployments, so workflow-code versioning (Worker Versioning or patching) is a standing obligation that an agent loop written as plain application code does not have.

### Gaps
- The docs pages read do not use the phrase "at-least-once"; the semantics above are quoted instead.
- The default Activity retry policy values were not on the pages read (unconfirmed here).
- The TypeScript sandbox mechanism (V8 isolate) is not named on the page read; unconfirmed from source.

## 2. Input during a running workflow: Signals, Updates, Queries; queueing and batching

### Takeaway
A workflow can accept Signals and Updates while its main function is awaiting an Activity (for example an LLM call); handlers run interleaved with the main function at `await` points, so a handler can push messages into a workflow-local queue that the main loop drains as a batch after the current call returns. The queue and the batching policy are application code; Temporal provides the durable delivery.

### Cited Findings
- [docs] Queries are read-only, "cannot block", add nothing to Event History and "can operate on completed Workflows". Signals are asynchronous writes; the sender cannot await a result, does not depend on a Worker being available and is not limited by per-workflow Update concurrency. Updates are "synchronous, tracked write requests"; the sender can wait for a result or error, the Worker must be available to acknowledge, and a validator can reject an Update before it is written to history. — [Workflow message passing](https://docs.temporal.io/encyclopedia/workflow-message-passing)
- [docs] TypeScript API: `defineSignal`, `defineUpdate`, `defineQuery`, `setHandler`, `condition`; client side `handle.signal`, `handle.query`, `handle.executeUpdate`, `handle.startUpdate` (with `waitForStage: ACCEPTED`). `handle.signal` "returns when the server accepts the Signal; it does not wait for the Signal to be delivered." — [TypeScript message passing](https://docs.temporal.io/develop/typescript/message-passing)
- [docs] Signal and Update handlers can be `async` and can await Activities, Child Workflows, `sleep` and `condition`. Handlers and the main method run concurrently, with control switching at `await` points, so shared state can change mid-operation; a mutex is recommended where handler instances must not interleave. A Query handler cannot be `async`. — [TypeScript message passing](https://docs.temporal.io/develop/typescript/message-passing)
- [docs] `workflow.condition(wf.allHandlersFinished)` keeps the main method from returning while handlers are running. Continue-As-New is not supported inside Update handlers. — [TypeScript message passing](https://docs.temporal.io/develop/typescript/message-passing)
- [docs] Workflow-to-workflow: `getExternalWorkflowHandle(workflowId).signal(...)` works from workflow code. Updates cannot be sent directly between workflows; use an Activity. Signal-With-Start "is limited to Client use". — [TypeScript message passing](https://docs.temporal.io/develop/typescript/message-passing)
- [docs] Limits per Workflow Execution (Temporal Cloud): 10,000 Signals received ("for sustained traffic, keep to 5 or fewer per second"), 10 Updates in flight, 2,000 Updates total in history. — [Temporal Cloud limits](https://docs.temporal.io/cloud/limits)
- [docs] The execution is terminated if history contains more than 2,000 Updates or more than 10,000 Signals. — [Events and Event History](https://docs.temporal.io/workflow-execution/event)
- [third party] Scale AI's procurement-agent tutorial (19 November 2025, Python) uses a workflow Signal that validates and queues external events for asynchronous processing, and keeps accepting events while a `wait_for_human` tool waits. — [Scale AI Agentex and Temporal tutorial](https://scale.com/blog/agentex-temporal-tutorial)

### Inferences
- The Router's "queue Triggers during a turn, handle them together next turn" maps to: Signal handler appends to an array; main loop awaits the LLM Activity, then takes the whole array. This is the same capability pi-durable gives through `pi.inbox` with `steeringMode: all`, but written by hand.
- Nothing delivers input into an in-flight LLM Activity. "Steering mid-turn" means either waiting for the Activity to return, or cancelling it (cancellation reaches an Activity only through heartbeats) and re-issuing the call with the new input.
- Every Signal costs one history event and one billable Action, and counts toward the 10,000-per-run limit, so a busy Router must continue-as-new on Signal count as well as on history size.

### Gaps
- The docs pages read do not show a queue-and-drain batching example in TypeScript, and do not state Signal ordering guarantees; both unconfirmed here.
- The self-hosted defaults page lists no separate total-signals default beyond the 2,000 pending-signal limit, while the Event History page states termination above 10,000 Signals; whether the 10,000 figure applies to self-hosted by default is unconfirmed. — [Self-hosted defaults](https://docs.temporal.io/self-hosted-guide/defaults)

## 3. Months-long entities: history limits, Continue-As-New, payload limits

### Takeaway
A single run cannot live for months if it does much: it is terminated at 51,200 events or 50 MB of history, and any one payload is capped at 2 MB. A never-ending Router must be an "entity workflow" that periodically Continue-As-News, passing its state (under 2 MB, or by reference) to a fresh run with the same Workflow ID.

### Cited Findings
- [docs] Event History: warning at 10,240 events or 10 MB; hard limit 51,200 events or 50 MB, at which the execution is terminated. Same values on Temporal Cloud and as self-hosted defaults (configurable self-hosted via `HistoryCountLimitError`, `HistorySizeLimitError`). — [Temporal Cloud limits](https://docs.temporal.io/cloud/limits); [Self-hosted defaults](https://docs.temporal.io/self-hosted-guide/defaults)
- [docs] Payload (blob) size: self-hosted warns at 512 KB and errors at 2 MB ("Blob data size exceeds limit"); Cloud single request payload 2 MB; gRPC message 4 MB; event batch (transaction) 4 MB, not configurable on Cloud. — [Self-hosted defaults](https://docs.temporal.io/self-hosted-guide/defaults); [Temporal Cloud limits](https://docs.temporal.io/cloud/limits)
- [docs] Per execution, incomplete Activities, Signals, Child Workflows and external cancel requests: 2,000 each by default (self-hosted, since v1.21; 2,000 combined on Cloud); "for best performance, keep these at 500 or fewer". — [Self-hosted defaults](https://docs.temporal.io/self-hosted-guide/defaults); [Temporal Cloud limits](https://docs.temporal.io/cloud/limits)
- [docs] "Continue-As-New allows you to checkpoint your Workflow's state and start a fresh Workflow." "The new Workflow Execution has the same Workflow Id, but a different Run Id, and starts its own Event History." State is passed as arguments to the new run. Temporal sets a "Continue-As-New Suggested" indicator; workflows using this pattern "can run indefinitely" and are called Entity Workflows. — [Continue-As-New](https://docs.temporal.io/workflow-execution/continue-as-new)
- [docs] Entity Workflow pattern (examples in Python, Go, Java, TypeScript): one workflow per long-lived entity with the entity ID as Workflow ID. Use the SDK's continue-as-new-suggested check, not a hardcoded counter. "Always call Continue-As-New from the main Workflow method, never from handlers." "Wait for all handlers to finish before continuing." Continue-As-New input is subject to the 2 MB blob limit. Update-ID dedup is scoped to one run: "Carry processed IDs in the Continue-As-New input if deduplication is needed." Not a good fit above about 100 updates per second per entity. — [Entity Workflow pattern](https://docs.temporal.io/design-patterns/entity-workflow.md)
- [docs, via search snippet] The Entity Workflow page also warns that Continue-As-New starts a new run, "so you will lose any pending Signals if you don't handle them first". — [Entity Workflow pattern](https://docs.temporal.io/design-patterns/entity-workflow)
- [docs] "Child Workflows do not carry over when the Parent uses Continue-As-New." — [Child Workflows](https://docs.temporal.io/child-workflows)
- [docs] Retention of closed executions on Cloud: default 30 days, configurable 1 to 90 days. — [Temporal Cloud limits](https://docs.temporal.io/cloud/limits)
- [release notes] TypeScript SDK v1.21.0 (23 July) introduced "external storage" on the `DataConverter` with AWS S3 and GCS drivers, marked prerelease and experimental; v1.25.0 (27 October) added error types, concurrency limits and opt-in payload caching, still marked experimental. v1.21.0 also made Workers validate outbound payload sizes by default. — [sdk-typescript releases](https://github.com/temporalio/sdk-typescript/releases)
- [docs] The AI Cookbook has a "Claim check pattern" recipe (Python) for offloading large payloads to S3 to keep them out of Event History. — [AI Cookbook](https://docs.temporal.io/ai-cookbook)
- [docs] In the OpenAI Agents integration, "A Continue-As-New starts with an empty session"; history must be carried explicitly (`session.getItems()` into `continueAsNew()`, `initialItems` in the new run). — [OpenAI Agents SDK integration, TypeScript](https://docs.temporal.io/develop/typescript/integrations/openai-agents.md)

### Inferences
- An LLM transcript cannot be the workflow's carried state for long: a context of a few hundred thousand tokens exceeds 2 MB as one payload, and each LLM Activity that takes the whole transcript as input and returns a response writes both into history, so 50 MB is reached after a few dozen large calls. The practical design keeps the transcript in an external store (database or object store) and passes references through the workflow, or uses the experimental external-storage data converter. The transcript store is then application-built.
- Each LLM turn with one tool call costs roughly 6 to 10 history events (activity scheduled, started, completed, plus workflow-task events), so 51,200 events is on the order of a few thousand turns per run; a Router handling hundreds of events per day needs a Continue-As-New every few days to weeks. This arithmetic is an estimate, not a sourced figure.
- Because children do not carry over a parent's Continue-As-New and the default Parent Close Policy is Terminate, a Router that owns Workers as children would have to drain or abandon them at each rollover; this favours Workers as independent top-level workflows or abandoned children.
- Retention matters to the Watchdog and to audit: closed Worker runs (and earlier Router runs in the chain) disappear after the retention period unless exported.

### Gaps
- The exact server threshold at which "Continue-As-New Suggested" is set was not found (unconfirmed).
- Whether Signals arriving during the Continue-As-New transition are rejected, buffered or carried over was not confirmed beyond the search-snippet warning above.

## 4. Workers as children: child workflows versus separate workflows, async start, cancellation, reporting back

### Takeaway
A Worker can be a Child Workflow (started asynchronously, policy Abandon) or an independent workflow started from an Activity; either way it is a separate execution with its own history, and it reports to the Router by sending Signals to the Router's Workflow ID. Parent and child share no state and talk only through Signals.

### Cited Findings
- [docs] A Child Workflow Execution is started by another workflow in the same Namespace. The parent must await the child's spawn and may optionally wait for its result. Parent and child share no local state and communicate only asynchronously through Signals. — [Child Workflows](https://docs.temporal.io/child-workflows)
- [docs] Parent Close Policy options: Abandon ("the Child Workflow Execution is not affected"), Request Cancel, Terminate. Default: Terminate. Abandon is "useful for starting Child Workflows asynchronously". Policies are set per child. — [Parent Close Policy](https://docs.temporal.io/parent-close-policy)
- [docs] A single parent "should not spawn more than 1,000 Child Workflow Executions" (guidance). Reasons to use a child: separate service on its own Workers, partition work across histories, represent a single resource by Workflow ID. "When in doubt, use an Activity." Children record all state changes in history; Activities record only input, output and retries. — [Child Workflows](https://docs.temporal.io/child-workflows)
- [docs] When a child Continue-As-News, the parent treats the whole chain of runs as one execution. — [Child Workflows](https://docs.temporal.io/child-workflows)
- [docs] Cancel records a request in history and lets workflow code clean up; Terminate ends the execution immediately and workflow code cannot react. — [temporal workflow CLI](https://docs.temporal.io/cli/workflow)
- [docs] A workflow can signal any other workflow by ID with `getExternalWorkflowHandle`. — [TypeScript message passing](https://docs.temporal.io/develop/typescript/message-passing)
- [docs] "Temporal guarantees that there can be at most one Workflow Execution with a given ID running at any point in time." — [Workflow ID and Run ID](https://docs.temporal.io/workflow-execution/workflowid-runid)
- [vendor] Temporal's "ambient agents" post (18 September 2025, Python) implements each agent (broker, execution, judge) as "a long-running Workflow that maintains state and waits for Signals", "similar to an actor model", with agents signalling and querying each other. — [Orchestrating ambient agents with Temporal](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal)

### Inferences
- The article's `start()`/`stop()` map directly: `start()` = start child with Abandon (or start a workflow with ID `worker-<ticket>`) and return once started; the result arrives later as a Signal into the Router's queue; `stop()` = request cancel on that Workflow ID. This is less glue than pi-durable's Anchor/Reporter tasks, because cross-session delivery and by-ID addressing are built in.
- Worker isolation is at the level of a separate history and workflow state; process, sandbox and working-directory isolation still come from where the tool Activities run (task queues per machine are the Temporal way to pin Activities to a host).
- Thread-subscription pass-through (code delivering an event straight to a Worker) is a Signal to the Worker's Workflow ID from the ingestion server; the subscription table is application state.

### Gaps
- TypeScript child-workflow API details (`startChild`, `parentClosePolicy` option names) were not read from the SDK reference; names are unconfirmed here.
- No source read states how a Signal to a workflow that has already closed fails from inside another workflow.

## 5. Watchdog: schedules, reading another workflow, intervening, detecting stuck workflows

### Takeaway
Temporal supplies most of the Watchdog's mechanics as platform features: Schedules start a workflow on an interval, and any client or Activity can describe, query, read the history of, signal, update, cancel, terminate or reset any other workflow by ID. Judging whether a Worker is making real progress remains an LLM call the application writes.

### Cited Findings
- [docs] A Schedule starts Workflow Executions at specified times; specs are interval (for example `45m`) and calendar (cron-like), with jitter, time zones and exclusions. Overlap policies: Skip (default), BufferOne, BufferAll, CancelOther, TerminateOther, AllowAll. Catchup window default one year, minimum ten seconds. Schedules can be paused, triggered manually and backfilled. "Actions should generally start within 1 second of the specified time". Each Schedule is backed by a Workflow Execution that stays open. — [Schedule](https://docs.temporal.io/schedule)
- [docs] Temporal Cloud: schedule requests limited to 10 per second per Namespace. — [Temporal Cloud limits](https://docs.temporal.io/cloud/limits)
- [docs] CLI and API operations on any execution: `describe`, `show` (Event History, optionally following live), `query`, `stack` (a `__stack_trace` Query), `signal`, `update`, `cancel`, `terminate`, `reset`, `trace` (progress of an execution and its children), `list` and `count` with visibility queries; `pause`/`unpause` are experimental. — [temporal workflow CLI](https://docs.temporal.io/cli/workflow)
- [docs] Reset ends an execution and starts a new one with the same Workflow Type and ID, copying history up to the reset point; valid reset points are WorkflowTaskStarted, Completed, TimedOut and Failed; Signals after the reset point can optionally be copied. — [Events and Event History](https://docs.temporal.io/workflow-execution/event)
- [docs] Queries need no history write and work on completed workflows; a Query handler reads current workflow state. — [Workflow message passing](https://docs.temporal.io/encyclopedia/workflow-message-passing)
- [docs] Activity heartbeat timeout: the server can treat an Activity as failed if no heartbeat arrives within it; Start-To-Close timeout bounds a lost task. — [Activity Execution](https://docs.temporal.io/activity-execution)
- [vendor] The ambient-agents post uses a Temporal Schedule (`ensemble-nudge`, every 25 seconds in the code sample) to start a nudge workflow, and a "judge" agent that queries another agent's state and signals it a new system prompt. — [Orchestrating ambient agents with Temporal](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal)

### Inferences
- The Watchdog's read path does not depend on the Worker's owner exposing a relay: `describe`, history fetch and Queries go through the Temporal Service, and history and `describe` work even if no worker process for that Worker is alive. A Query, in contrast, needs a worker able to replay the workflow (standard Temporal behaviour; not stated on the pages read, so unconfirmed here).
- Infrastructure-level stuckness (hung LLM call, dead tool) is covered by Activity timeouts, heartbeats and retries without a Watchdog; the Watchdog is needed only for semantic drift and for decisions such as restart.
- "Restart a Worker" has three distinct tools: Signal a corrective message, cancel and start a new Worker with carried context, or reset to an earlier history point (which re-runs the agent from that turn with recorded results before it).
- Since the transcript will usually live outside workflow state (section 3), the Watchdog reads the transcript from the application's store and uses Temporal for status and intervention.
- Nothing restarts the application's own worker processes; a process supervisor (systemd, container restart policy) is still needed, as with pi-durable on Node. The Temporal Service itself must also be kept running when self-hosted.

### Gaps
- What `describe` returns (pending activities, attempt counts, last failure) was not stated on the CLI page read; unconfirmed here.
- Whether `stack`/Queries require a running worker is not stated on the page read; unconfirmed here.
- No built-in "stuck workflow" detector was found in the pages read; none is claimed.

## 6. Event ingestion: webhooks, Signal-With-Start, dedup

### Takeaway
No built-in webhook or event-ingestion endpoint was found in the documentation read; an application HTTP server must verify the webhook and call the Temporal client (Signal, Signal-With-Start or Update-With-Start). Temporal contributes dedup primitives: one open execution per Workflow ID, ID reuse and conflict policies, and Update IDs.

### Cited Findings
- [docs] `client.workflow.signalWithStart` sends a Signal and starts the workflow first if it is not running; Update-With-Start (`executeUpdateWithStart`, `startUpdateWithStart`) needs a `WithStartWorkflowOperation` with a `workflowIdConflictPolicy`. — [TypeScript message passing](https://docs.temporal.io/develop/typescript/message-passing)
- [docs] CLI `signal-with-start` exists; `execute-update-with-start` and `start-update-with-start` are marked experimental in the CLI. — [temporal workflow CLI](https://docs.temporal.io/cli/workflow)
- [docs] Workflow ID Reuse Policy (closed executions): Allow Duplicate (default), Allow Duplicate Failed Only, Reject Duplicate, Terminate if Running. Workflow ID Conflict Policy (open executions): Fail (default), Use Existing, Terminate Existing. Reuse checks only see closed executions still within the Retention Period. — [Workflow ID and Run ID](https://docs.temporal.io/workflow-execution/workflowid-runid)
- [docs] Update IDs support dedup (`workflow.currentUpdateInfo`, `getUpdateHandle`), but are scoped to one run and are lost at Continue-As-New unless carried in the input. — [TypeScript message passing](https://docs.temporal.io/develop/typescript/message-passing); [Entity Workflow pattern](https://docs.temporal.io/design-patterns/entity-workflow.md)
- [docs] Update validators can reject a message before it is written to history. — [Workflow message passing](https://docs.temporal.io/encyclopedia/workflow-message-passing)
- [docs] "Temporal Hosts Should Not Be Exposed to the Open Internet." — [Self-hosted deployment](https://docs.temporal.io/self-hosted-guide/deployment)
- [docs] The dev server exposes an HTTP API port (`--http-port`) in addition to gRPC. — [temporal server CLI](https://docs.temporal.io/cli/server)

### Inferences
- Ingestion shape: a small app server (signature check, allowlist fast path) calls `signalWithStart` on the Router's fixed Workflow ID, or signals a subscribed Worker's ID directly. This is the same host-code row as in the pi-durable mapping, with delivery and "start if absent" provided.
- Signals carry no built-in dedup key visible on the pages read, so webhook redelivery dedup is either (a) an Update with the delivery ID as Update ID plus a carried set of processed IDs, or (b) a processed-ID set kept in workflow state or in the app server's database. A one-workflow-per-event design gets dedup from Workflow ID with Reject Duplicate, limited to the retention window.
- The Router workflow has no sender authentication of its own; access control is at the namespace/API level, so the ingestion server is the trust boundary.

### Gaps
- Absence of a first-party webhook receiver is inferred from the docs pages read (message passing, schedules, AI pages, integrations list); it was not confirmed by an explicit statement. Mark as unconfirmed.
- Whether Signal requests are deduplicated by a client-supplied request ID is unconfirmed.

## 7. LLM agent support as of November 2026

### Takeaway
Temporal now ships first-party agent-framework integrations, and TypeScript is covered (AI SDK by Vercel, OpenAI Agents SDK, Google ADK, Strands, LangSmith; Mastra via its own integration). They make a framework's model and tool calls durable Activities inside a workflow; they do not supply a months-long session with an inbox, a transcript store, compaction or mid-turn steering. Status labels conflict between the 30 October 2026 GA announcement and some docs pages.

### Cited Findings
- [vendor] 30 October 2026: "Today, we are making integrations for several popular AI libraries and agent frameworks generally available": Google ADK (Python, Go, TypeScript), OpenAI Agents SDK (Python already GA, plus TypeScript), AWS Strands (Python, TypeScript), LangGraph (Python), LangSmith (Python, TypeScript), AI SDK by Vercel (TypeScript), Gemini SDK (Python). Packages live in the `temporalio/ai-integrations` repository. "TemporalModel runs each model call as an Activity." Framework-level retries are turned off in favour of Temporal's. — [AI framework integrations post](https://temporal.io/blog/ai-framework-integrations)
- [vendor] The OpenAI Agents SDK integration for the Python SDK became GA on 23 March 2026 (per search snippet of the partnership post; page not fetched). — [OpenAI Agents SDK partnership post](https://temporal.io/blog/announcing-openai-agents-sdk-partnership)
- [vendor] The AI SDK by Vercel integration entered Public Preview on 20 January 2026. — [TypeScript SDK changelog](https://temporal.io/changelog/product-area/typescript-sdk)
- [docs] **Conflict:** the TypeScript OpenAI Agents SDK docs page was still marked "Pre-release" when fetched on 14 November 2026, while the 30 October post lists it as GA. — [OpenAI Agents SDK integration, TypeScript](https://docs.temporal.io/develop/typescript/integrations/openai-agents.md); contradicted by [AI framework integrations post](https://temporal.io/blog/ai-framework-integrations)
- [docs] **Conflict:** the AI SDK integration docs page showed no status label when fetched; a search-index copy of the same page read "Public Preview". — [AI SDK by Vercel integration](https://docs.temporal.io/develop/typescript/integrations/ai-sdk.md)
- [docs] Integrations listed on the Durable AI page: AI SDK by Vercel (TS), Claude Agent SDK (Python), Deep Agents (Python), Google ADK (Python, Go, TS), Google GenAI (Python), LangGraph (Python), Mastra (TS, documented by Mastra), OpenAI Agents SDK (Python, TS), Pydantic AI (Python, documented by Pydantic), Spring AI (Java), Strands Agents (Python, TS). No status labels on that page. — [Durable AI](https://docs.temporal.io/ai.md)
- [docs] AI SDK by Vercel integration: package `@temporalio/vercel-ai-sdk`, "versioned separately from the Temporal TypeScript SDK". It wraps `generateText()`/`streamText()` model calls in Activities ("you cannot make LLM calls directly from a Workflow; they must run as Activities"). Tool functions run in workflow context and must call Activities or Child Workflows for non-deterministic work. MCP tool listing and calls run as Activities; the MCP client is stateless. Streaming: "Inside the Workflow, the streamed result is reassembled after the Activity completes"; live deltas go to external consumers through Workflow Streams. The page does not mention conversation history, chat Signals/Updates or Continue-As-New. — [AI SDK by Vercel integration](https://docs.temporal.io/develop/typescript/integrations/ai-sdk.md)
- [docs] OpenAI Agents SDK integration (TypeScript): package `@temporalio/openai-agents`. The workflow holds "the agent loop, tool selection, and handoffs"; each model call, MCP operation and sandbox operation is an Activity. "Agent state lives in the Workflow". Sessions use `WorkflowSafeMemorySession`; the SDK's `MemorySession` is rejected. Tool approval (`needsApproval`) pauses the run until a Signal or Update. Not supported: voice and real-time agents, model objects on agents, MCP prompts through MCP handles, cancellation through `signal`. Experimental: sandbox agents, streaming (needs a `streamingTopic`, not with Local Activities). Continue-As-New recommended before history gets too large. — [OpenAI Agents SDK integration, TypeScript](https://docs.temporal.io/develop/typescript/integrations/openai-agents.md)
- [docs] AI Cookbook: 16 recipes, 15 in Python and 1 in TypeScript (durable agent with tools using the AI SDK by Vercel). Python recipes include agentic loops with Claude and OpenAI, human-in-the-loop via Signals, claim check to S3, Google ADK multi-agent, Strands, a durable MCP server and deep research. — [AI Cookbook](https://docs.temporal.io/ai-cookbook)
- [vendor] Workflow Streams (announced 17 June 2026): "in Public Preview for Python and TypeScript"; Go and Java "coming next". A library built on Signals, Updates and Queries giving per-workflow topics of ordered events, for streaming LLM output to clients. Default batch interval 2 seconds; AI plugins use 100 ms. Publisher dedup window 15 minutes by default. Each Signal is recorded in history. If an Activity retry yields different model output, the consumer must retract partial output itself. The post says a person can "interrupt or steer" an agent but shows no mechanism beyond Signals and Updates. — [Workflow Streams post](https://temporal.io/blog/workflow-streams-live-interactivity-agents-other-applications)
- [docs] Workflow Streams, TypeScript: Public Preview, package `@temporalio/workflow-streams`. Events are kept in an in-memory log in the workflow; the carried state at Continue-As-New "includes the entire in-memory log of the previous run", so large logs hit the payload limit (use `truncate()` or External Storage). Per-token `forceFlush` "would generate one Signal per token". — [Workflow Streams, TypeScript](https://docs.temporal.io/develop/typescript/workflows/workflow-streams.md)
- [third party] Ably (a competing vendor) describes a Workflow Streams stream as one-way, poll-based and delivered to a single client, without multi-device fan-out or offline delivery (per search snippet; page not fetched, and the Temporal post says streams are readable by multiple subscribers). — [Ably on Workflow Streams](https://ably.com/temporal/temporal-workflow-streams-vs-ably); partly contradicted by [Workflow Streams post](https://temporal.io/blog/workflow-streams-live-interactivity-agents-other-applications)
- [third party] Scale AI's tutorial implements context summarisation itself ("once the context exceeds about 40k tokens") and keeps conversation history in workflow state. — [Scale AI Agentex and Temporal tutorial](https://scale.com/blog/agentex-temporal-tutorial)

### Inferences
- What the integrations do NOT provide, judged from the docs pages read (absence, not explicit denial): a persisted session inbox with busy/steer/follow-up modes; a transcript store outside Event History; context compaction; delivery of new input into a turn in flight; a subagent/worker lifecycle; trigger filtering. Each is application code on top of Signals, Activities and Child Workflows. pi-durable supplies the inbox, transcript and run model natively; Temporal supplies cross-session addressing, schedules, timeouts/retries and operations tooling natively.
- Token streaming exists but is preview/experimental and costs history events and Actions per batch; it is optional for a Router whose outputs are chat messages and PR comments.
- For TypeScript, the agent loop can be either a framework run inside the workflow (AI SDK, OpenAI Agents) or a hand-written loop calling an LLM Activity; the hand-written loop is the documented cookbook shape (Python recipes) and gives direct control over the queue-drain batching in section 2.
- Using pi's own agent loop (`pi-ai`) inside a Temporal workflow has no integration; its calls would have to be wrapped as Activities by hand (inference, no source).

### Gaps
- GA status of the TypeScript integrations is not settled: blog says GA (30 October 2026), docs page for OpenAI Agents says Pre-release. Current npm versions of `@temporalio/vercel-ai-sdk` and `@temporalio/openai-agents` were not confirmed.
- No first-party "Temporal agent framework" (a Temporal-owned agent loop with sessions and memory) was found; unconfirmed whether one exists.
- Pydantic AI, Mastra, Claude Agent SDK and Deep Agents integration pages were not read; their maturity is unconfirmed.

## 8. Hosting and cost for a single developer; licence; TypeScript SDK maturity

### Takeaway
Self-hosting means running the Temporal Server (one Go binary containing the services) plus a UI server and a database (PostgreSQL, MySQL or Cassandra), in addition to the application's own worker and ingestion processes; the single-binary dev server with SQLite is documented as not for production. Temporal Cloud has a pay-as-you-go plan with no base fee at 50 USD per million Actions. The server is MIT-licensed and the TypeScript SDK is at v1.25.0.

### Cited Findings
- [docs] Self-hosting is "deploying two Go binaries -- the core Temporal Server, and the Temporal UI Server"; "Each service can be deployed separately." Options: Docker Compose (default stack includes PostgreSQL and Elasticsearch, UI on 8080, gRPC on 7233), Helm (chart 0.73.1 or later for Server 1.30+), or binaries under systemd. Schema upgrades are the operator's job (`admin-tools`). — [Self-hosted deployment](https://docs.temporal.io/self-hosted-guide/deployment)
- [docs] Tested persistence stores: Cassandra 3.11, 4.0, 5.0.4+; PostgreSQL 13.18, 14.15, 15.10, 16.6; MySQL 5.7 and 8.0.19+. SQLite 3.x is supported but "meant only for development and testing, not production usage." Elasticsearch is recommended for production Visibility; since Server 1.20 advanced Visibility also works on MySQL 8.0.17+, PostgreSQL 12+ and SQLite 3.31.0+. — [Persistence](https://docs.temporal.io/temporal-service/persistence.md)
- [docs] `temporal server start-dev`: "By default, Workflow Executions are lost when the server process dies"; `--db-filename` gives a persistent SQLite file; Web UI on port 8233. "The development server is not intended for production use" and it "skips certain HTTP security checks". The deployment guide is for "sustained workloads that exceed what the development server is designed to handle." — [temporal server CLI](https://docs.temporal.io/cli/server); [Self-hosted deployment](https://docs.temporal.io/self-hosted-guide/deployment)
- [source repo] The `temporalio/temporal` server repository is MIT-licensed; the README's local quick start is `brew install temporal` then `temporal server start-dev`. — [temporalio/temporal](https://github.com/temporalio/temporal)
- [third party, via search snippet] Server release v1.31.0 was listed by a release tracker fetched 29 April 2026 (PostgreSQL schema v1.19, SQLite schema v1.11). The current server version in November 2026 was not confirmed. — [newreleases.io: temporal v1.31.0](https://newreleases.io/project/github/temporalio/temporal/release/v1.31.0)
- [vendor] Temporal Cloud pricing: pay-as-you-go from 0 USD/month with no minimum, support at 10% of usage; Business from 500 USD/month (greater of 500 USD or 10% of usage; includes 2.5M Actions, 2.5 GB active and 100 GB retained storage); Enterprise and Mission Critical by annual contract. Actions: 50 USD per million for the first 5M per month, falling to 25 USD at 100M to 200M. Storage: active 0.042 USD per GBh, retained 0.00105 USD per GBh. New accounts get 150 USD of credits for 90 days; startups under 30M USD funding can apply for 6,000 USD of credits. — [Temporal pricing](https://temporal.io/pricing)
- [vendor] Billable Actions include: workflow start, reset, Continue-As-New, child start; each Activity start or retry (all Local Activities in one Workflow Task count as one); each timer; each Signal; each accepted or rejected Update; each Query; Schedule actions. Replay is not billed. — [Temporal pricing](https://temporal.io/pricing)
- [release notes] TypeScript SDK latest release v1.25.0, dated 27 October (year not shown on the page; 2026 inferred from the sequence v1.21.0 23 Jul, v1.22.0 5 Aug, v1.23.0 26 Aug, v1.24.0 15 Sep). v1.25.0 moved AI integrations to a separate repository, leaving `@temporalio/ai-sdk` and `@temporalio/google-adk-agents` as compatibility wrappers; Workers now refuse workflow bundles built with a different SDK version. v1.21.0: `@temporalio/ai-sdk` targets Vercel AI SDK v7 and needs Node.js 22.12.0 or later. v1.23.0: major `protobufjs` upgrade (breaking). — [sdk-typescript releases](https://github.com/temporalio/sdk-typescript/releases)
- [vendor, via search snippets] The TypeScript SDK reached 1.0.0 in 2022 and v1.10.0 (2024) dropped Node 14 and 16, recommending Node 18+ (older information). — [TypeScript SDK changelog](https://temporal.io/changelog/product-area/typescript-sdk); [TypeScript 1.0.0 post](https://temporal.io/blog/typescript-1-0-0)

### Inferences
- Minimum durable self-host for one developer: Temporal Server + UI + PostgreSQL (Elasticsearch optional since 1.20) via Docker Compose on one machine, plus the Node worker process and the webhook server. That is three to four more long-running services than pi-durable's single process with one SQLite file. Running the dev server with `--db-filename` long-term works mechanically but is explicitly outside what the docs support.
- Cloud cost estimate (own arithmetic, not a sourced figure): if one agent turn is about 5 Actions (one Signal in, one LLM Activity, one or two tool Activities, one Signal out) and the Watchdog adds a few Actions per check per Worker, 1,000 turns a day is roughly 150,000 Actions a month, about 7.50 USD at 50 USD per million, plus support 10% and small storage charges. A Watchdog or Schedule firing every 30 seconds adds about 86,000 schedule actions a month before its own Activities. LLM token cost would dominate.
- The TypeScript SDK is mature (1.x since 2022, monthly minor releases in 2026) but the agent-related parts (AI integrations, Workflow Streams, external storage) are new in 2026 and still moving, with breaking changes in minor releases.

### Gaps
- Hardware requirements for a single-node self-host are not stated in the docs read.
- Current Temporal Server version and date (November 2026) unconfirmed.
- Exact Node.js version support matrix for TypeScript SDK v1.25.0 unconfirmed.
- One third-party site claims Apache 2.0 for the server; the repository itself shows MIT. — [temporalio/temporal](https://github.com/temporalio/temporal)

## 9. Published examples of long-lived, always-on, multi-session agents on Temporal

### Takeaway
There are published examples of agents as long-running, Signal-driven workflows and of multi-agent systems with a schedule-driven nudge and a judging agent, but all found are Python and demo-scale; no write-up was found of a router/worker/watchdog system matching the Super-agent article, and none in TypeScript.

### Cited Findings
- [vendor] "Orchestrating ambient agents with Temporal" (Kevin Martin, 18 September 2025, Python): broker, execution and judge agents each as a long-running workflow waiting for Signals; the judge queries the execution agent's state and signals a new system prompt; a Schedule nudges the ensemble; MCP tools backed by workflows. Continue-As-New is not mentioned; the order tool is a mock. Repo: [Aslan11/crypto-trading-agents](https://github.com/Aslan11/crypto-trading-agents). — [Orchestrating ambient agents with Temporal](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal)
- [third party] Scale AI, "Agentex and Temporal" tutorial (19 November 2025, Python): a procurement agent in a workflow that "can persist for months", external events by Signal into a queue, human-in-the-loop wait with 24-hour timeout, self-built summarisation above about 40k tokens. Demo scope. Repo: [scale-agentex-python procurement_agent](https://github.com/scaleapi/scale-agentex-python/tree/main/examples/demos/procurement_agent). — [Scale AI Agentex and Temporal tutorial](https://scale.com/blog/agentex-temporal-tutorial)
- [docs] Temporal's Durable AI page points agents to the Approval and Entity Workflow design patterns and features two videos: "memory-state-stripe" and "AI agent gateway with Temporal Nexus" (routing agent tool calls through one gateway with policy checks and approval). — [Durable AI](https://docs.temporal.io/ai.md)
- [docs] AI Cookbook recipes relevant to the shape: human-in-the-loop agent (Signals), multi-agent orchestration with Google ADK, deep research multi-agent (all Python). — [AI Cookbook](https://docs.temporal.io/ai-cookbook)
- [vendor, via search snippet, not fetched] Temporal posts "From agent zoo to agent orchestra" (agentic control plane; Workflow IDs to ensure one agent acts on an entity at a time) and "What are multi-agent workflows". — [Agentic control plane post](https://temporal.io/blog/from-agent-zoo-to-agent-orchestra-temporal-agentic-control-plane); [Multi-agent workflows post](https://temporal.io/blog/what-are-multi-agent-workflows)
- [third party, unverified] "Cruvero" publishes a roadmap for a Temporal-based agent runtime with Continue-As-New and supervisor workflows; it is a design document from an unofficial project. — [Cruvero roadmap](https://cruvero.pages.dev/architecture/roadmap)

### Inferences
- The closest published shape to Router + Watchdog is the ambient-agents demo (long-lived agent workflows + Schedule + judge agent that steers by Signal), which shows the primitives fit but does not address months of operation (no Continue-As-New, no transcript storage design).
- The Super-agent on Temporal would be a new design assembled from documented patterns, as it is on pi-durable; neither side has a reference implementation.

### Gaps
- No engineering write-up was found from a company running an always-on router/worker/supervisor agent on Temporal in production with details on history rollover and transcript storage.
- No TypeScript example of an entity-workflow chat agent with a Signal inbox was found in the pages read; the `temporalio/samples-typescript` and `temporalio/ai-integrations` repositories were not inspected.
- The Stripe "memory state" video and the Nexus gateway video were not viewed.
