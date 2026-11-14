# Goal

Stated by the project owner on 14 November 2026. This answers point 1 of
[blind-spots.md](blind-spots.md).

**super-durable is to run as a personal office agent, for e-mail, to-dos,
calendar and similar office work, deployed as a project on the TU Wien
Datafactory.**

- **User:** one person.
- **Work:** office work, not coding.
- **Host:** a project on the TU Wien Datafactory
  (`dashboard.apps.datalab.unicloud.at`).

## What the host provides

Read from the dashboard on 14 November 2026. Only the project list and one
existing project page were looked at; nothing was created or changed.

- **A project is an OpenShift namespace** (`project-<name>`), in an
  environment called "Playground".
- **Kubernetes resources** are managed through the OpenShift console, ArgoCD
  or the command line tools.
- **Managed PostgreSQL** can be added per project (an existing project runs
  version 16 with 1 GB of storage).
- **Object storage** and **user management** are offered per project.
- **Not checked:** quotas, whether a project can expose a public route,
  outbound network rules, uptime or retention guarantees of the Playground
  environment, and the platform's rules on personal data.

## What the goal changes

The earlier research assumed the article's use case: a coding agent for a
team. An office agent for one person shifts the weight of almost every
finding. These are inferences from the existing documents, not new research.

| Topic | Earlier assumption | With this goal |
|---|---|---|
| Remote tool execution | The largest piece to build (shell on your machines through pi-env) | Mostly drops out. Workers call mail, calendar and to-do APIs, not a shell. |
| Hosting and restart | Unassigned; an external supervisor was needed | OpenShift runs and restarts the container. |
| Trigger ingestion | Webhooks need a public endpoint | An OpenShift route if allowed, otherwise polling the mail and calendar servers. |
| Router batching | pi-durable's cleanest fit | Matters little: one person does not produce 40 Triggers a minute. |
| Threads | PRs and chat topics | An e-mail chain is a Thread, so the routing design fits naturally. |
| Threat model | Important before real use | The first thing to settle. Anyone can send an e-mail, and the agent could send mail in the owner's name. |
| Approval before acting | A missing feature, later found on `main` | Central. Sending, accepting and deleting need approval or a draft-only mode. |
| Privacy | Not examined | Mail content goes to a model provider. Which provider is acceptable is undecided. |
| Storage | One SQLite file | SQLite on a persistent volume with a single replica, or the managed PostgreSQL. Whether pi-durable has PostgreSQL storage is unchecked. |
| Substrate comparison | pi-durable for the Router's behaviour | Unchanged in direction. Managed PostgreSQL does make DBOS (Node plus Postgres) cheaper to run here than the comparison assumed. |

## Still open

- **Stopping rule:** what result would count as "it worked", and what would
  end the experiment.
- **Systems:** which mail, calendar and to-do services the agent connects to.
  This decides the integrations and whether events are pushed or polled.
- **Authority:** whether the agent may act (send, accept, delete) or only
  prepare drafts and suggestions.
- **Model:** which model provider may see the mail.
- **Channel:** how the owner talks to the agent (e-mail, chat, a web page).
