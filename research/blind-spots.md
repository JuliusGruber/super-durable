# Blind spots in the plan to build the Super-agent on pi-durable

Blind spot analysis, 14 November 2026. Reviews
[feasibility-report.md](feasibility-report.md),
[source-reading.md](source-reading.md),
[platform-assessment.md](platform-assessment.md) and
[user-value.md](user-value.md).

This is reasoning over those documents. No new sources were read and nothing
was built or run. Where a point depends on a fact nobody has checked, it says
so.

## Summary

**The research answers "can it be built?" thoroughly and "should it, and what
would we learn?" hardly at all.** The largest blind spots are not in
pi-durable. They are the missing goal, the missing behavioural evaluation and
the missing threat model.

| # | Blind spot | Why it matters | Cheapest way to close it |
|---|---|---|---|
| 1 | No goal, user or workload is defined | Every later choice depends on it | Write three sentences: who uses it, for what, and what "worked" means |
| 2 | Durability is treated as the hard part | The article's own failures were behavioural | Define an evaluation before any plumbing |
| 3 | pi-durable's clean fit covers the smallest part | 2 of 12 mechanisms map cleanly | Estimate effort per row |
| 4 | No "no framework" baseline | The prototype itself uses SQLite and a polling loop | Compare against plain Node plus SQLite |
| 5 | A durable transcript is not a durable environment | A resumed Worker meets a half-changed machine | Decide the Worker recovery rule |
| 6 | The Router's context grows without limit | Compaction on pi-durable was never checked | Read the source for compaction |
| 7 | Months-long persistence on an experimental API | Upgrades can leave conversations unreadable | Decide whether state must survive upgrades |
| 8 | No threat model | Untrusted input plus SSH to your machines | Write one before connecting a real channel |
| 9 | No cost model or spending limit | Always-on Router, Watchdog and restart loops | Estimate tokens per day; set a cap |
| 10 | Hosting and reachability are unassigned | "Always on" needs a server that can reach endpoints | Name the host and the endpoint |
| 11 | Findings were not folded back into the topologies | Two corrections break the recommended designs | Redraw the topologies |
| 12 | Nothing has been run | Three desk studies in one day | A two-hour spike |

## 1. No goal, user or workload

The README says the repo exists "to explore the idea". None of the documents
says who would use the result, on what work, or what outcome would count as
success or failure.

- **The article's case is a team.** Its premise is a person limited to 3 to 5
  sessions, and its batching experiment runs from 1 to 40 Triggers per minute.
  One person does not produce that load.
- **Three different goals are possible,** and each points somewhere else:
  - *Learn the architecture:* use the most boring substrate available.
  - *Learn pi-durable:* the Super-agent is a weak vehicle (see point 3).
  - *Have a working assistant:* the platform assessment already says Managed
    Agents is closer.
- **Without a goal there is no stopping rule.** The exploration cannot fail,
  so it cannot conclude either.

## 2. Durability is treated as the hard part

All three studies examine the substrate: checkpoints, replay, inboxes,
alarms. The article's own evidence points elsewhere.

- **Delegation failed by default.** With a neutral prompt the single-session
  design never delegated in 5 runs. That is a prompt and model result, not a
  storage result.
- **The valuable behaviours are judgements:** filter policy, "stay quiet
  otherwise", when to speak up after speculative work, whether a Worker is
  making real progress. None depends on pi-durable.
- **The faux provider tests plumbing only.** It cannot show whether a Router
  routes well or a Watchdog judges well.
- **The article is itself unvalidated.** Its experiments are synthetic and
  simulate coding with a slow tool. Building an unproven design on an
  experimental substrate stacks two uncertainties; a failure cannot be
  attributed to either.

## 3. The clean fit covers the smallest part

- **By the report's own table:** 2 rows map cleanly, 6 need glue and 4 are
  pure host code.
- **By user value:** of the 10 items in user-value.md, pi-durable directly
  supplies parts of two (stays responsive, always on). The other eight are
  application code.
- **The verdict wording hides this.** "Yes, nothing verified is blocked" is
  true of any general-purpose runtime. The useful question is how much
  pi-durable saves, and no document estimates effort for any row.
- **"Glue cost is roughly the same everywhere" is asserted, not estimated.**
  Channel integrations (a chat platform, a GitHub App, email) are each a
  project and appear nowhere as a cost.

## 4. No "no framework" baseline

- **The comparison set was pi-durable, Anthropic and OpenAI.** Missing is the
  simplest option: a Node process, a SQLite file and a loop. The prototype is
  exactly that (single process, one SQLite file, 2-second polling).
- **An agent loop is already resumable from its transcript.** Step
  checkpointing adds most when a turn is long and its tools are replay-safe.
  What it adds for this design was never weighed.
- **Running the prototype itself was not considered.** It is distributed as a
  bundle for DeepSeek Harness. Using it for a week would show which of the 10
  user values matter before any is rebuilt.
- **Durable-execution engines** (Temporal, Restate, DBOS, Inngest) are listed
  as an open question in both passes and were never examined.
- **`apps/p2` was noticed and not followed up.** The source reading found a
  product in the pi repository whose subagents are "close to the article's
  Worker". What p2 is, and whether Earendil is building toward the same
  design, is unchecked.

## 5. A durable transcript is not a durable environment

pi-durable makes the conversation survive a crash. The Worker's real state is
a working directory, running processes and half-finished side effects on
another machine.

- **Remote mutations can end with an unknown outcome.** The source reading
  records this for a lost pi-env connection.
- **A resumed Worker is told "interrupted"** for tools that are not
  `replay: safe`. Whether a model reliably re-inspects the environment before
  continuing is a behavioural question nobody has tested.
- **No recovery rule exists** for a Worker's directory: resume in place, reset
  to the last commit, or discard and restart the Ticket.

## 6. The Router's context grows without limit

- **The Router is one LLM session meant to run for months.** The article
  lists long-term memory as an open problem.
- **Context compaction on pi-durable was never checked.** Neither was what
  happens to queued inbox items or the prompt cache across a compaction.
- **Inline & Spill and Chain of Recall are filed as "host code".** They are in
  fact the Router's memory design, and probably the hardest application part.
- **Idle periods defeat prompt caching.** A Router woken after a quiet hour
  re-reads its whole context at the uncached price. The article reports cache
  reads of 23 to 32 million tokens per run under continuous load, which is the
  favourable case.

## 7. Months-long persistence on an experimental API

- **The two facts are recorded but not connected.** The design's main promise
  is persistence; the report notes that the API changes between releases and
  that rolling back Pi can leave conversations unreadable.
- **Needed features are unreleased.** Signals, parked tools, `notify` and
  approval gates exist on `main` only, so a prototype pins a commit of a
  moving branch.
- **Several findings were not checked against the released 1.5.2.**
- **One vendor, no deletion.** Conversations cannot be deleted, so Worker
  sessions accumulate in one database file with no backup plan.
- **Models were not checked.** The comparison table says "Not checked" for
  which models pi-durable can use, although the design needs a light Router
  model and a strong Worker model.

## 8. No threat model

The article lists security as an open problem. The research repeats that as a
"cost" and goes no further.

- **The combination is the risk:** an always-on agent that reads text from
  strangers (emails, review comments), acts without being asked, and holds SSH
  access to your machines.
- **Credentials sit on the Harness host.** Keeping them out of model context
  is marked "build it".
- **Signals have no sender authentication.**
- **The Watchdog can rewrite prompts and trigger rules at runtime.** A
  successful injection can therefore make itself permanent.
- **This limits evaluation too.** Without a threat model the agent cannot
  safely be given real work, and without real work point 2 cannot be answered.

## 9. No cost model or spending limit

- **Three standing consumers:** the Router on every surviving Trigger, the
  Watchdog on every scheduled check for every Worker, and each Worker on the
  most capable model.
- **Restart loops spend money.** A Watchdog that submits `continue` to a
  Worker that keeps failing has no stated limit.
- **No estimate exists** for tokens per day at any workload, and no budget
  cap appears in either topology.

## 10. Hosting and reachability are unassigned

- **"Always on" needs a machine that is always on.** The Node topology does
  not say where the process runs, who restarts it, or how the database is
  backed up.
- **Webhooks need a public endpoint,** or every source must be polled.
- **pi-env dials in over SSH.** A laptop behind a home router is not reachable
  from a cloud host without a tunnel. User value 9 ("works on your own
  machines, queued work runs on reconnect") is therefore the item least
  supported, not a side detail.
- **Windows endpoints get no pseudo-terminals,** and need an SSH server.
- **The Cloudflare topology has no Worker environment at all.** pi-env cannot
  run inside a Cloudflare Worker (an inference in the source reading), and
  Sandbox SDK and Containers were never examined.

## 11. Findings not folded back into the topologies

The source reading corrected the feasibility report, but the recommended
topologies were not redrawn.

- **Batching is Harness-wide.** A Router and its Workers in one Harness share
  `steeringMode`. The Node topology puts both in one Harness as parent and
  child, so either Workers inherit batch placement or they need a second
  Harness, which the anchor-and-reporter pattern does not cover.
- **The Watchdog can read but not act.** It still needs an API on the owning
  process. That API is undesigned and is itself an unauthenticated control
  surface.
- **Workers cannot spawn subagents** in the documented pattern, while the
  article allows a Worker to be an Agent Team.
- **Worker quality is unexamined.** Workers produce all the actual work. The
  research treats a Worker as a session slot and never asks whether a
  pi-durable conversation with tools matches a mature coding agent. The
  platform assessment notes that Codex app-server or the Claude Agent SDK
  could serve as a Worker, and stops there.

## 12. Nothing has been run

- **All three studies are desk research from one day.** The three-vote
  verification checks claims against documentation, not behaviour.
- **Requirements come from one reading** of a machine-translated article that
  crawlers cannot fetch.
- **A small spike would outweigh another research pass:** a Router
  conversation, one anchor-owned Worker, a killed process and a resume, with
  the faux provider.

## What to do first

1. **State the goal and a stopping rule** (point 1). Everything else depends
   on it.
2. **Run the smallest spike** (point 12) on a pinned commit, and record what
   differs from the documents.
3. **Build the same spike without pi-durable** (point 4), on plain Node and
   SQLite, and compare the line counts and the crash behaviour. This is the
   only direct measure of what pi-durable saves.
4. **Write the evaluation and the threat model** (points 2 and 8) before
   connecting any real channel.
5. **Check the three unread facts:** compaction, supported models, and what
   `apps/p2` is (points 4, 6 and 7).
