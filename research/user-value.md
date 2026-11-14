# What the Super-agent offers a user

The 10 most valuable things compared with today's coding agents (Codex,
Claude Code), taken from the
[article summary](super-agent-article-summary.md). Today's agents are
session-centric: you open a session, give it a task and watch it. The
Super-agent is one always-on agent you talk to like a colleague.

It is a design and a research prototype, not a shipped product. Today's agents
already have partial versions of some items (background tasks, scheduled runs,
cloud sessions); the difference is that here they are the default and work
together.

| # | What you get | Today's coding agents |
|---|---|---|
| 1 | **One agent instead of many sessions.** You talk to a single agent that remembers everything across tasks and days. | You manage separate sessions, each starting without the others' context. |
| 2 | **Many tasks in parallel without babysitting.** Each task gets its own Worker; you are no longer the bottleneck. | A person can follow only about 3 to 5 active sessions. |
| 3 | **It stays responsive while working.** You can add instructions, ask for status or change direction mid-task and get an answer straight away. | Input usually waits until the current tool call or turn finishes. |
| 4 | **It reacts to events on its own.** A review comment, a failed CI run, an email or a calendar reminder starts work without a prompt from you. | Work starts when you type a prompt, or from triggers you set up one by one. |
| 5 | **It acts ahead and stays quiet otherwise.** For example it reproduces a reported bug in the background and only speaks up once a fix is confirmed. | The agent does what was asked, when asked. |
| 6 | **Reach it where you already work.** Chat, GitHub and email all lead to the same agent, with short replies in chat and detailed ones in PRs and documents. | Mostly a terminal, IDE or web app, with integrations that act as separate entry points. |
| 7 | **Follow-ups land in the right place.** Later events on the same PR, chat topic or email chain go directly to the Worker already handling it. | You find the right session yourself and paste the new information in. |
| 8 | **Always on.** The agent runs in the cloud and its state is in a database, so closing your laptop or a crash does not lose the work. | A local session ends with the terminal; cloud sessions persist but are separate tasks. |
| 9 | **It works on your own machines.** A small daemon runs commands in your existing environments, and queued work runs when an offline device reconnects. | Either local on one machine or in a vendor sandbox you have to set up. |
| 10 | **It checks itself.** A Watchdog notices Workers that are stuck or drifting from the goal, retries transient failures, and can explain why something failed. | You notice a stalled or off-track session yourself and restart it. |

## What it costs

- **More tokens.** The experiments show higher cache-read token use than a
  single-session design.
- **More trust required.** The agent reads more untrusted input and acts
  without being asked; the article lists security, privacy and credential
  handling as open problems.
- **Less visibility by default.** How you inspect a task or step in mid-task
  when there are no sessions to open is still unsolved.
