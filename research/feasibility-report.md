# Can the Super-agent architecture be implemented with pi-durable?

Research report, 14 November 2026. Produced by two deep-research passes
(web search, source fetch, three-vote adversarial verification per claim).

- **Pass 1:** requirements taken from the prototype repo; 25 of 25 verified
  claims confirmed.
- **Pass 2:** requirements taken from the X article (see
  [super-agent-article-summary.md](super-agent-article-summary.md)); 24 of 25
  verified claims confirmed, 1 refuted.

## Verdict

**Yes: the full article design can be built on pi-durable, and nothing
verified is blocked.** Only the Router's input handling maps cleanly. Every
other mechanism needs application code, and pi-durable supplies the
durable-session substrate, not the orchestration layer.

This verdict is inference from documentation. No source discusses routers,
workers, triggers or watchdogs on pi-durable, and nothing was built or run.

## Mechanism mapping

| Article mechanism | Fit | How |
|---|---|---|
| Router persistence and crash recovery | Maps cleanly | Tasks checkpoint at every step and resume from the last one. The host must reopen storage and call `harness.resume()`. A task is ended after `settings.restartLimit` crashes (default 5). |
| Router batching, input mid-turn | Maps cleanly | A conversation accepts input while a run is active and queues it in the persisted inbox (`pi.inbox`). Batching needs `whenBusy: steer` with `steeringMode: all`, or `followUpMode: all` for a successor run. The defaults are `whenBusy: followUp` and one item at a time. |
| Workers | Glue | No built-in subagent API. The documented background-subagent pattern fits: an anchor task owns a child conversation that outlives the parent's turns, and a reporter task posts the answer back to the parent as a follow-up input, deduplicated by request ID. |
| Async `start()`/`stop()` tools | Glue | A parked tool returns its late result as the same tool call's result, not as a new message. `start()` needs a background task that submits a follow-up input; `stop()` aborts that task. |
| Speculative background work | Glue | Same background-task pattern, plus an application filter for "deliver only confirmed results". |
| Compute/state/environment separation | Glue | State in a database and a stateless loop are native. Tools run through a pluggable execution environment built per tool call, so they can run on another machine. The article's endpoint daemon (offline queueing, run on reconnect) is custom. |
| Wake-ups and scheduling | Glue, less on Cloudflare | On a raw Durable Object the host arms the single alarm from `onWake` and calls `resume()`; a missing alarm strands work. The Agents SDK `PiHarness` does this with a 30-second heartbeat while work is pending. |
| Watchdog | Glue, least supported | It can run as its own session on a schedule, but cannot read another session directly (`session.watch()` is in-memory inside the owning object). Each Worker's owner must expose a relay for transcript and live state, plus methods to steer, abort or resubmit. On plain Node nothing restarts the process, so an external supervisor is also needed. |
| Trigger ingestion and two-stage filtering | Host code | No pi-durable primitive. Signals are single-key deliveries to one live task with no sender authentication, not an event bus. Filter in code, then submit into the Router's inbox; the semantic policy is Router prompt text. |
| Thread subscriptions and pass-through | Host code | An application table; subscribed events are submitted by code straight to the Worker's session. |
| Inline & Spill, Chain of Recall | Host code | Read tools over IDs stored in application tables or documents. |
| Ticket model, worker slots, per-Ticket directories, reply outbox | Host code | Present in the prototype, absent from pi-durable. |

The "host code" rows are reasoned from absence: no verified claim addresses
them directly.

## Recommended topologies

Both are synthesis of the verified primitives, not something any source
prescribes.

### Node host

- One process owns one storage; the Router is the root conversation.
- Workers are anchor-owned child conversations with reporter tasks.
- Host code ingests Triggers, filters them and submits to the Router.
- Thread subscriptions live in an application table.
- Tools are routed through a remote execution environment per Worker.
- The Watchdog is a separate conversation or process that goes through the
  owning process's API.
- An external supervisor restarts the process.

This is closest to the prototype's own single-process, single-writer design.

### Cloudflare

- A stateless ingress Worker (webhooks, queue, dedup, allowlist) sits in front
  of a Router Durable Object.
- Each Worker/Ticket gets its own top-level `PiHarness` Durable Object, not a
  child inside the Router's object.
- Router and Workers talk by RPC that ends in a submit or steer call. These
  cross-object calls need their own idempotency keys.
- The subscription table lives at the ingress or Router layer, so pass-through
  events skip the Router model.
- A Watchdog object on a schedule calls relay methods on each Worker object.

The separate-object choice comes from the Agents SDK sub-agent ("facet")
limits: facets share the root's single alarm, placement and broken state, are
reachable only through the parent, and the tree is capped at four levels.

## Things to design around

- **Input is never placed mid-tool.** A long blocking tool delays new input,
  so long work must go to background tasks.
- **Queued Triggers can stall after a failed run.** They stay in the inbox
  until a new input, queued write or continue places them; the host needs a
  nudge.
- **Side effects are not exactly-once.** Only tools marked `replay: safe` are
  rerun after an interruption; others return an interrupted result to the
  model. Tools that open pull requests or post comments need idempotency keys
  or reconciliation, and so do remote commands.
- **Workers cannot spawn subagents by default.** The documented pattern
  creates the child with the subagent tools removed.
- **Worker isolation is transcript-level only.** No process, sandbox or
  per-Ticket directory isolation is provided.
- **Cloudflare timing limits.** A 30-second heartbeat, so background
  completion can be noticed up to 30 seconds late. Alarm invocations are
  capped at 15 minutes, so a single model request streaming longer can be cut
  off. Storage is 10 GB of SQLite per object (Workers Paid).
- **No tool-call approval and no conversation deletion yet.** Both matter for
  a long-lived agent that acts on external events and accumulates Worker
  sessions.
- **Storage is single-writer by contract.** One process writes a storage at a
  time, with no cross-process locking.

## Maturity

- **pi-durable** is labelled experimental; its API changes between releases.
- **`PiHarness`** is in beta and expected to change. Each Agents SDK release
  targets one Pi minor version, and rolling back to an older Pi can leave
  conversations unreadable.
- **dsh-superagent-prototype** is a one-commit research prototype, not an
  official product. It omits some of the article's designs: it has no
  Watchdog in its architecture doc, uses a 2-second polling loop and a single
  local SQLite file.

## Open questions

- **Worker shell and filesystem on Cloudflare:** nothing verified on Sandbox
  SDK, Containers or git worktrees, nor on what pi's remote execution
  environment provides (transport, authentication, reconnect, queued
  commands).
- **Watchdog inspection:** whether another session's storage can be opened
  read-only is unsettled. A pass-2 claim on this was refuted 1-2, which
  conflicts with the pass-1 single-writer finding. What restarts a Worker that
  exhausted `restartLimit` is also uncovered.
- **`PiHarness` inside a facet:** not verified whether it works, or whether
  facet storage counts against the root's 10 GB.
- **Comparable builds:** neither pass produced verified evidence on
  router/worker/watchdog agents on Temporal, Restate, Inngest, DBOS or the
  plain Agents SDK.
- **Background-subagent example:** the example file returned 404, so the
  pattern is known from the spec text only.

## Caveats

- All sources are first-party documentation (READMEs, spec, engineering posts,
  Cloudflare docs). No behaviour was checked against source code or by
  testing.
- Sources date from October and November 2026 and the projects move quickly.
  Version-specific figures (30-second heartbeat, 10-minute handoff, four-level
  facet depth, Pi 1.5, Agents SDK 0.33.0) may change.
- The X article is not readable by crawlers. Pass 2 took its requirements from
  our own reading of it.

## Sources

**Super-agent**

- [X article](https://x.com/DeepSeekHarness/status/2116857257245987236)
- [dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype)
- [Prototype architecture doc](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/007-architecture.md)

**pi-durable**

- [Package source and README](https://github.com/earendil-works/pi/tree/main/packages/durable)
- [Spec](https://github.com/earendil-works/pi/blob/main/packages/durable/docs/spec.md)
- [README at 1.1.0](https://unpkg.com/@earendil-works/pi-durable@1.1.0/README.md)
- [Pi Durable post](https://earendil.com/posts/pi-durable/)
- [Pi Durable on Cloudflare post](https://earendil.com/posts/pi-durable-on-cloudflare/)

**Cloudflare**

- [Pi harness docs](https://developers.cloudflare.com/agents/harnesses/pi/)
- [PiHarness changelog, 2 Oct 2026](https://developers.cloudflare.com/changelog/post/2026-10-02-pi-harness/)
- [Durable Objects limits](https://developers.cloudflare.com/durable-objects/platform/limits/)
- [Agents SDK sub-agents](https://developers.cloudflare.com/agents/api-reference/sub-agents/)
- [Sub-agents doc, agents@0.24.0](https://unpkg.com/agents@0.24.0/docs/sub-agents.md)
