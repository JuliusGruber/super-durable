# What the pi-durable source settles

Source reading, 14 November 2026, of
[earendil-works/pi](https://github.com/earendil-works/pi) at commit `c1d4b0a`
(`main`, same day). It follows up the open questions in
[feasibility-report.md](feasibility-report.md), which was based on
documentation only.

Read: the pi-durable README, changelog and selected source files, the
background-subagent example, and the pi-env README and protocol docs. Nothing
was built or run. Paths below are relative to the pi repository.

## Summary

| Open question | Status | Answer |
|---|---|---|
| Can a Watchdog inspect another session? | Resolved | Yes, by opening the same storage with a read-only Harness from a second process. It cannot intervene that way. |
| What does pi's remote execution environment provide? | Resolved | `@earendil-works/pi-env`: a Rust daemon deployed and started over SSH. No offline queueing. |
| Does the background-subagent example exist? | Resolved | Yes, as `22-subagent-background.ts`, and a product adaptation exists in `apps/p2`. |
| What restarts a Worker that exhausted `restartLimit`? | Resolved | Nothing automatic. The run ends with a recorded failure, and a `continue` submission restarts it. |
| Does pi-durable compact context? | Resolved | Yes, automatically and by default, with a manual `compact()` as well. Raw history stays in storage. |
| Does `PiHarness` work inside an Agents SDK facet? | Still open | Not in this repository. |
| Comparable builds on Temporal, Restate, Inngest, DBOS | Still open | Not looked at. |

## Watchdog inspection

- **Read-only open exists.** `HarnessOptions.readOnly` opens a Harness "as a
  viewer of a session that another process drives". Reads and watches work;
  every commit with writes, `resume()` and every call that enables scheduling
  reject with `ReadOnlySession`
  (`packages/durable/src/harness/types.ts:846-853`).
- **SQLite needs nothing extra.** The README says SQLite storage "opens as
  usual: SQLite's file locks keep it safe beside the writer". JSONL storage
  must also be opened with `{ readOnly: true }`, because a writable JSONL open
  repairs files the writer may be appending to.
- **This settles the earlier conflict.** Storage is single-writer, and a second
  process can still read it. So a Watchdog process on a Node host can read
  every Worker's transcript, `pi.live` state and task graph directly. To
  steer, abort or resubmit, it must call the owning process.
- **Not checked:** whether a read-only Harness on SQLite sees commits made
  after it opened. The JSONL read-only open is documented as holding the
  state committed at open. A Watchdog that reopens on each scheduled check
  does not depend on this.
- **Cloudflare is unchanged.** A Durable Object's storage is private to the
  object, so the relay methods from the feasibility report are still needed
  there.

## Remote execution environment

`packages/env` (`@earendil-works/pi-env`, version 1.5.2) is the remote
executor.

- **Shape:** a small Rust daemon (`pi-env`) runs on the remote machine and
  performs file operations and commands. `RemoteExecutionEnv` is a pi-durable
  `ExecutionEnv` that talks to it. Storage and credentials stay on the Harness
  host.
- **Transport:** the daemon's stdin and stdout over `ssh`, with a framed
  binary protocol (`packages/env/docs/protocol.md`). `ssh` runs in batch mode
  with strict host-key checking against the application's own known-hosts
  file and no forwarding.
- **Deployment:** `connectSsh()` detects the remote system and deploys the
  daemon if missing. The binary is named by its SHA-256 and verified before
  every start.
- **Platforms:** Linux, macOS, Android (Termux) and Windows, on x86-64 and
  arm64.
- **Extras:** pseudo-terminals (not on Windows), piped processes (for
  example a remote MCP server), file watching and tree listing.
- **Connection loss:** the connection starts lazily and is started again
  after it is lost. Requests in flight fail with code `unknown`, and for
  mutations the outcome is then unknown
  (`packages/env/src/connection.ts:226-227`).

Differences from the article's endpoint daemon:

- **Direction is reversed.** The Harness host dials the endpoint over SSH. The
  article's daemon connects outward from the endpoint, which also works for
  machines that accept no inbound connections.
- **No offline queueing.** A command for an unreachable machine fails; nothing
  queues it for the next reconnect. That part stays custom, for example as a
  tool that parks and retries.
- **Node host only (inference).** The client starts `ssh` as a child process,
  so it cannot run inside a Cloudflare Worker.

## Background subagents

- **The example exists** at
  `packages/durable/test/examples/22-subagent-background.ts`. The 404 in the
  research was a wrong path.
- **Pattern confirmed as the report described it:**
  - An `Anchor` background task, terminal at once, owns the child
    conversation. The parent's abort and idle waits stop at it.
  - A `Reporter` background task per message has two phases. `deliver`
    submits to the child and waits for the answer. `report` submits the
    answer to the parent as a follow-up input.
  - Request IDs (`subagent:<task id>`, `subagent-report:<task id>`) keep a
    restart from delivering or reporting twice.
  - The child is configured without the subagent extension.
  - One tool with actions `spawn`, `send`, `stop` and `status`. `stop` is the
    child conversation's `abort()`. `status` reads the child's `pi.live` run.
- **A product version exists** at `apps/p2/plugins/subagents/tools.ts`. It
  adds a model, thinking level, tool selection and an own location
  (environment and working directory) per subagent. This is close to the
  article's Worker.

## After `restartLimit`

- A run cut off `settings.restartLimit` times (default 5) ends with
  `docs["pi.live"].runFailure` reason `restart_limit`, and the host's
  `onReport` gets a `task_interrupted` report. Nothing restarts it.
- Submitting `{ type: "continue" }` starts a new run on the context as it is,
  without a user message.
- So the Watchdog's job is concrete: read `runFailure` through the read-only
  Harness, judge, and ask the owning process to submit a `continue` or a
  corrective input.

## Context compaction

Checked later the same day at the same commit, after
[blind-spots.md](blind-spots.md) and
[workflow-engine-comparison.md](workflow-engine-comparison.md) listed it as
unchecked. Read: the README "Compaction" section, the changelog,
`src/harness/agent.ts`, `src/harness/types.ts` and
`test/examples/24-compaction.ts`. Nothing was run.

- **Compaction is built in and on by default.** `DEFAULT_COMPACTION_POLICY`
  has `enabled: true` (`packages/durable/src/harness/agent.ts:30-35`).
- **How it works:** older entries are summarised by a model call, and a
  `pi.compaction` entry holding the summary is appended. Older entries stay in
  storage, so the raw history is never changed.
- **Three triggers:**
  - *Threshold:* above `contextWindow - reserveTokens` (default 16,384) the
    next request waits for a compaction.
  - *Background:* a compaction starts `backgroundTokens` (default 32,768)
    below that threshold while the conversation keeps working.
  - *Overflow:* when a provider rejects a request as too long, generation
    compacts and retries once.
- **Recent context is kept verbatim,** roughly `keepRecentTokens` (default
  20,000).
- **Manual compaction:** `conversation.compact(instructions)` works even when
  automatic compaction is off.
- **Settings per model and per conversation.** `modelOverrides` is keyed by
  exact `"provider/modelId"`, and `configure({ compaction })` sets one
  conversation's values. So a Router on a light model and Workers on a strong
  one can differ, unlike the Harness-wide batching mode.
- **A busy conversation is not blocked.** The summary is placed at once when
  the conversation is idle, otherwise at the next turn boundary.
- **A custom summary is possible.** A `beforeCompact` hook can decline a
  compaction or supply its own summary. This is where the article's Inline &
  Spill references could be written into the Router's summary.
- **Failure is visible.** A failed blocking compaction fails the generation
  with `model_error`, and the newest failure stays in
  `docs["pi.live"].compactionFailure`. Summarisation spend counts in
  `pi.usage`.
- **Released, not only on `main`.** The changelog has compaction fixes under
  1.2.0, `modelOverrides` under 1.4.0 and per-conversation settings under
  1.5.1.
- **Not checked:**
  - the summarisation prompt, and so what a months-long Router loses in each
    summary;
  - how inbox items queued during a compaction are placed;
  - the effect on the provider's prompt cache.

Compaction bounds the model context, not the database. Old entries are kept
and conversations still cannot be deleted, so storage keeps growing.

## Corrections to the feasibility report

- **Tool-call approval exists on `main`.** A `beforeTool` hook can block a
  call, or park it until a signal or deadline
  (`test/examples/44-approval-gate.ts`). The report listed approval as
  missing.
- **Queued Triggers need no host nudge.** `settings.afterFailedRun:
  "continue"` starts the next run with the inputs queued behind a failed run.
  The default `keep` leaves them waiting.
- **Worker isolation can go beyond the transcript.** `HarnessOptions.env`
  receives the conversation ID and `cwd`, so each Worker can get its own
  directory, container or remote machine
  (`test/examples/28-sandbox-per-conversation.ts`). The sandbox itself is
  still yours to supply.
- **Batching is a Harness-wide setting.** `steeringMode` and `followUpMode`
  are in `HarnessSettings`, "run policy shared by every conversation"
  (`packages/durable/src/harness/types.ts:726-727`). A Router and its Workers
  in one Harness share the mode.
- **A parked tool is not the article's `start()`.** Confirmed: once woken, a
  parked tool's result is still that call's result, and the run waits for it.
  A non-blocking `start()` needs the background-task pattern.
- **Conversation deletion:** still none found in `packages/durable/src`.

## Released versus `main`

Signals, tools that park, `notify` on submissions, approval gates and examples
38 to 46 are under "Unreleased" in the changelog. The published version is
1.5.2 (10 November 2026). A prototype that needs them must use `main` or wait
for the next release. The read-only Harness, the inbox modes, `continue`,
`afterFailedRun` and the background-subagent pattern were not checked against
the 1.5.2 tag.

## Toolchain facts for a prototype

- TypeScript, ES modules, Node 22.19 or later.
- Packages: `@earendil-works/pi-durable`, `@earendil-works/pi-ai`,
  `@earendil-works/chord`, and `@earendil-works/pi-env` for remote tools.
- `pi-ai` has a scripted faux provider
  (`@earendil-works/pi-ai/providers/faux`), so Router and Worker logic can be
  tested without a model API key.
- Storage on a Node host: `openNodeSqliteStorage(file)`, one database file.
