# Does pi-durable make sense for this? Comparison with OpenAI and Anthropic

Assessment, 14 November 2026. Builds on
[feasibility-report.md](feasibility-report.md) and
[super-agent-article-summary.md](super-agent-article-summary.md).

The vendor comparison comes from one docs-checking research agent per vendor,
not the three-vote adversarial verification used for the feasibility report.
Items the agents could not confirm are marked "undocumented" or "not checked".

## Verdict

**Yes, pi-durable makes sense for this repo's goal of exploring the
architecture, but it is not the fastest route to a working Super-agent.** If
the goal shifts to running one in production soon, Anthropic's Managed Agents
is the closer fit.

None of the three vendors supplies the orchestration layer (Trigger ingestion,
Thread subscriptions, recall, Watchdog judgement). That glue cost is roughly
the same everywhere, so the choice is about the substrate underneath.

## Comparison

| Need | pi-durable | Anthropic Managed Agents | OpenAI Agents API |
|---|---|---|---|
| Who runs the loop | You (Node or Cloudflare) | Anthropic | OpenAI |
| Crash recovery mid-turn | Step checkpoints with documented replay rules | Server-side; tool replay behaviour undocumented | Server-side; undocumented |
| Queue input during a turn | Yes, with batch mode | Yes, queued in order | Steers the turn; queueing undocumented |
| Sessions addressable by ID | Yes | Yes | Yes |
| Router-to-Worker routing | Build it | Build it | Build it |
| Worker shell and filesystem | Build it | Hosted container, or your own machine via a polling worker | Hosted, or your own via `codex exec-server` |
| Scheduling for a Watchdog | Build it (alarms on Cloudflare) | Cron deployments | Mentioned, thinly documented |
| Credentials kept from the model | Build it | Vaults | Not checked |
| Models | Not checked | Claude only | OpenAI only |
| Maturity | Experimental | Beta | Beta, reportedly since Sept 2026 |

## Where pi-durable wins

- **Router fit:** its persisted inbox with batch placement is exactly the
  article's Trigger batching, and it is the only option where that behaviour
  is fully documented.
- **Known failure semantics:** you know which tools rerun after a crash
  (`replay: safe`). Both hosted runtimes say "recovery" without stating
  whether in-flight tools replay.
- **You own the state and the host:** no session-hour fee and no
  data-residency limits. The OpenAI Agents API is US-only, and neither
  vendor's hosted sandbox is eligible for zero data retention.

## Where it loses

- **Execution environment:** the article's "agent loop in the cloud, daemon on
  your machine" design is what Managed Agents' self-hosted sandboxes already
  are. On pi-durable this is the largest piece to build, and the research
  could not confirm what pi's remote executor provides.
- **Watchdog trigger, credential vaults, session viewer:** provided by Managed
  Agents, absent in pi-durable.
- **Churn:** pi-durable's API changes between releases. The hosted betas will
  change too, but there you are not maintaining the runtime.

## Notes per vendor

### Anthropic

- **Managed Agents (beta):** hosted loop, sessions by ID, server-side event
  queue, `user.interrupt`, custom tools executed client-side, webhooks,
  scheduled deployments, memory stores, vaults. Priced at $0.08 per
  session-hour while running, plus tokens.
- **Self-hosted sandboxes (beta):** tool execution moves to your own machine
  through an outbound-polling worker. File and GitHub resources are not
  supported there.
- **Its subagents are not the Worker layer.** Subagent threads share one
  container, allow one level of delegation and cap at 25 child threads per
  session. Router and Workers would be separate sessions with your own
  routing between them; there is no session-to-session messaging.
- **Limits:** sandbox state survives 30 days after creation; webhooks are
  dropped after three failed attempts and are not a durable log.
- **Claude Agent SDK:** a harness you host. Transcripts are JSONL files (a
  pluggable store mirrors them best-effort), input queues in streaming mode,
  and no mid-turn checkpointing is documented. It is weaker than pi-durable on
  durability but usable as a Worker on your own machine.

### OpenAI

- **Agents API (beta):** hosted Codex-harness runtime with durable sessions by
  ID, hosted or self-hosted sandbox, subagents inside one session, MCP and
  session webhooks. A message sent during a turn steers it. US-only data
  residency, no zero data retention. Retention and idle limits are
  undocumented.
- **Agents SDK (stable, Python v0.24.0):** the weakest fit. It takes no input
  during a run and checkpoints only when the app serialises state at an
  approval pause. Step durability comes only through Temporal, Restate, DBOS
  or Dapr. Queue-or-steer submission and portable session checkpoints exist
  only in a pre-release alpha.
- **Responses API:** background mode and server-side conversations, but a
  concurrent write to a conversation is rejected, so no input during a
  response.
- **Codex app-server and SDK:** threads resume by ID and a running turn can be
  steered, so it could serve as a Worker.
- **Platform churn:** the Assistants API shut down on 26 August 2026, and
  Agent Builder shuts down on 30 November 2026.

## Recommendation

Proceed with pi-durable on a Node host first, and build the Router and routing
layer there. Treat the Worker execution environment as the decision point: the
layers are separable, so if that glue becomes the main cost, Workers could run
as Managed Agents sessions behind the same routing code. That hybrid is an
inference, not something the research tested.

## Sources

**Anthropic**

- [Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview)
- [Events and streaming](https://platform.claude.com/docs/en/managed-agents/events-and-streaming)
- [Multiagent orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)
- [Self-hosted sandboxes](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes)
- [Scheduled deployments](https://platform.claude.com/docs/en/managed-agents/scheduled-deployments)
- [Webhooks](https://platform.claude.com/docs/en/managed-agents/webhooks)
- [Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Scaling Managed Agents: Decoupling the brain from the hands](https://www.anthropic.com/engineering/managed-agents)
- [Agent SDK sessions](https://code.claude.com/docs/en/agent-sdk/sessions.md)
- [Agent SDK session storage](https://code.claude.com/docs/en/agent-sdk/session-storage.md)
- [Agent SDK hosting](https://code.claude.com/docs/en/agent-sdk/hosting.md)

**OpenAI**

- [Agents API overview](https://developers.openai.com/api/docs/guides/agents-api/overview)
- [Agents API sessions](https://developers.openai.com/api/docs/guides/agents-api/sessions.md)
- [Agents API session webhooks](https://developers.openai.com/api/docs/guides/agents-api/sessions/webhooks.md)
- [Agents SDK sessions](https://openai.github.io/openai-agents-python/sessions/)
- [Agents SDK human-in-the-loop](https://openai.github.io/openai-agents-python/human_in_the_loop/)
- [Agents SDK running agents](https://openai.github.io/openai-agents-python/running_agents/)
- [Agents SDK sandbox agents](https://openai.github.io/openai-agents-python/sandbox_agents/)
- [Agents SDK releases](https://github.com/openai/openai-agents-python/releases)
- [Responses API background mode](https://developers.openai.com/api/docs/guides/background.md)
- [Conversation state](https://developers.openai.com/api/docs/guides/conversation-state)
- [Deprecations](https://developers.openai.com/api/docs/deprecations)
- [Codex app-server README](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md)
