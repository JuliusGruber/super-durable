# Exploring Super-agent Design: Towards a More Autonomous Agent System (summary)

- **Source:** <https://x.com/DeepSeekHarness/status/2116857257245987236>
- **Author:** DeepSeek Harness team (system and article by Yujia Qiao)
- **Published:** 1 November 2026 (machine translation of a Chinese original)
- **Read:** 14 November 2026

This is a summary in our own words, not a copy of the article. Read the
original for the diagrams, screenshots and exact wording. The article describes
architectural ideas, not shipped DeepSeek Harness features.

## 1. What a Super-agent is

Single-session agents make the human the bottleneck: a person can follow only
about 3 to 5 active sessions. A Super-agent is one continuous agent that keeps
context across sessions and runs many tasks at once. It has four properties:

| Property | Meaning |
|---|---|
| Continuous | Feels like one agent across IM, GitHub and other channels. |
| Productive | Runs several complex tasks in parallel and accepts new instructions mid-task. |
| Proactive | Watches environment events and acts without being asked; stays quiet otherwise. |
| Persistent | Always online and recovers from anomalies on its own. |

Five modules deliver these: a two-level architecture, event-driven Triggers,
full-duplex multi-channel interaction, separation of compute, state and
environments, and a Watchdog.

## 2. Two-level architecture

One session for everything gives continuity but low throughput; one session per
event gives throughput but no shared context. The design splits the work.

**Router layer**

- Long-lived; receives all human interaction and holds global memory.
- Only talks to people and dispatches tasks. It does not execute work and can
  run without a Bash tool.
- Does not decompose tasks (unlike an Agent Team lead), so a lighter model is
  usually enough.
- Implemented as a single long-lived LLM session. Triggers that arrive during a
  turn queue up and are handled together in the next turn (batching).
- Can later be sharded or given staged filtering, at the cost of shared context.

**Worker layer**

- One Worker per task, with isolated context and full execution tools; uses the
  most capable model.
- Never talks to humans. Reports status to the Router, which sends all replies.
- Peer Workers do not message each other. Conflicts surface through shared
  Thread subscriptions.
- A Worker may itself use subagents or be an Agent Team; this is invisible to
  the Router.

## 3. Event-driven Triggers

Triggers are semi-structured environment events: chat messages, emails,
calendar reminders, review comments, CI results.

- **Filtering (3.1):** two stages. Code checks source, group and allowlists
  first (fast path). Only surviving candidates go to the LLM, which applies
  policies written as prompts (slow path).
- **Routing (3.2):** consecutive events on the same PR, IM topic or email chain
  form a Thread. The Router handles the first event and creates a Worker that
  subscribes to the Thread. Later events are passed by code straight to the
  subscribed Worker. Subscriptions are many-to-many. A Worker can unsubscribe
  or hand an event back to the Router.
- **Context (3.3):** treat the context window as a cache. Inline what is likely
  needed soon (IDs, short excerpts, neighbouring messages); spill the rest
  behind a reference the model can fetch. Every object (Trigger, Thread, Worker
  Session) has an ID, and each recalled object inlines the IDs of related
  objects, so the model can recall context hop by hop ("Chain of Recall").
- **Acting ahead (3.4):** speculative background work on environment signals,
  for example silently reproducing a reported bug and only speaking up once a
  simple fix is confirmed.

## 4. Full-duplex multi-channel interaction

- **Full-duplex (4.1):** replace blocking tool calls with asynchronous
  `start()` and `stop()`. `start()` returns immediately and the result arrives
  later as a new event, so the session stays responsive.
- **Multi-channel (4.2):** per-channel norms set by prompt. IM replies are short
  (a 100-word cap worked for them); GitHub, email and documents get structured,
  dense output.

## 5. Separating compute, state and environments

- **Compute from state (5.1):** the agent loop runs in stateless cloud
  containers; sessions and other persistent state live in a database.
- **Compute from execution environment (5.2):** environments stay where they
  are. Each endpoint runs a lightweight, stateless daemon that only executes
  commands and returns results. This supports bring-your-own-machine, including
  devices that are only intermittently online: queued work runs when the device
  reconnects.

## 6. Watchdog

- An independent Watchdog Session outside the main system, driven by scheduled
  messages.
- Handles infrastructure failures (API errors, failed state recovery).
- Also uses the model to judge whether each Worker is making real progress or
  drifting from its goal; a live process is not enough.
- Keeps rechecking transient external failures until resolved.
- Has full tools, so maintainers can ask it why something failed and have it
  adjust prompts or trigger rules at runtime.

## 7. Preliminary experiments

Synthetic workloads, indicative only.

**Ablation (7.1):** about 300 messages per run, about 20 of them work requests,
5 runs per design, coding simulated by a time-consuming tool.

| Design | Result |
|---|---|
| Super-agent | Task completion 12.4 min p50, 15.7 min p95. |
| Super-agent without pass-through | Similar completion times; Router makes about 30% more decisions; cache-read tokens rise from 23.2M to 31.8M. |
| Single session + subagent tool, neutral prompt | Never delegated in 5 runs; p95 29.5 min. |
| Single session + subagent tool, delegation-first prompt | Delegated in 4 of 5 runs; still slower than Super-agent. |

Super-agent uses more cache-read tokens than the single-session designs.

**Router batching (7.2):** raising the Trigger rate from about 1 to about 40
per minute grew the average batch from 1.0 to 2.8 and time per request from
2.5 s to 3.9 s, while amortised inference time per decision fell from 6.4 s to
1.6 s. One Router session was enough at their team size.

## 8. Open problems

- **Scaling (8.1):** when to shard the Router, when Workers should talk
  directly, and long-term memory once the system runs for months.
- **Sessionless interaction (8.2):** how to present tasks and let users step in
  mid-task when UI conversations no longer map to sessions.
- **Context boundaries and agent-to-agent (8.3):** which context is shared per
  instance, and how isolated instances collaborate.
- **Security and privacy (8.4):** more untrusted input; a compromised cloud
  loop could take over every connected endpoint; credentials need to be usable
  without entering model context.

## Implementation status

The article says parts of the design are implemented in
[dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype),
a validation prototype and not an official product. The asynchronous task
mechanism exists in DeepSeek Harness but is not enabled by default.
