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

## Decisions

Given by the project owner on 14 November 2026.

| Question | Decision |
|---|---|
| Mail, calendar and to-do system | The owner's TU Wien Outlook mailbox, reached in the browser at `mail.intern.tuwien.ac.at/owa`. |
| Authority | The agent may act (send, accept, delete), not only draft. |
| Model provider | TU Wien Aqueduct. |
| Channel to the owner | Chat and a web page. |
| Stopping rule | Deferred; too hard to answer now. |

## What the decisions imply

Inferences only. The mailbox was not opened and none of these services was
checked.

- **The mailbox looks like an on-premises Exchange server.** An `/owa` address
  on the university's own domain suggests that. If so, Microsoft's cloud API
  (Graph) does not apply, and access goes through Exchange Web Services, IMAP
  and SMTP, or ActiveSync. Which of these TU Wien enables is unchecked.
- **Credentials are the first hard problem.** The agent would hold a login to
  the owner's university account, in a container on a shared platform. Whether
  an app password or token exists, or only the main password, is unchecked,
  and so is whether TU Wien permits automated access.
- **Events will probably be polled.** Exchange push notifications need an
  endpoint the mail server can reach, which is unconfirmed. Polling needs
  nothing inbound.
- **The model stays inside TU Wien,** which answers the privacy question for
  mail content. Which models Aqueduct offers, whether they call tools
  reliably, and whether pi-durable's model layer can talk to it are unchecked.
  Model strength matters: the article's weaker setups failed to delegate at
  all.
- **"May act" makes the threat model the next document to write.** Anyone can
  e-mail the owner, and the agent can now answer in the owner's name. Acting
  is decided; what remains is which actions run freely and which wait for a
  confirmation in the chat.
- **The web page needs a login.** It would be served from the OpenShift
  project and would control the mailbox.
- **"Chat and a web page" is read here as a chat interface on a web page.** A
  separate chat service (for example a messenger) would be an extra
  integration.

## Still open

- **Stopping rule:** what result would count as "it worked", and what would
  end the experiment. Deferred by the owner.
- **Which actions need confirmation** before the agent carries them out.
- **Whether TU.it permits automated access** to a staff mailbox, and whether
  a login other than the main password is available.

The questions on mail protocols, Aqueduct's models and pi-durable's model
layer were answered later the same day; see
[tu-wien-services.md](tu-wien-services.md).
