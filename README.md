# super-durable

A repo to explore the idea of implementing the Super-agent architecture with
pi-durable.

The Super-agent architecture is the design described by the DeepSeek Harness
team: a persistent, event-driven agent built from a long-lived Router layer,
isolated Worker sessions, and an external Watchdog. Background reading on it is
collected in [research/links.md](research/links.md).

## Goal

A personal office agent for e-mail, to-dos and calendar, running as a project
on the TU Wien Datafactory. See [research/goal.md](research/goal.md).

## pi-durable

- [earendil-works/pi — packages/durable](https://github.com/earendil-works/pi/tree/main/packages/durable) — the pi-durable source in the pi repository.
- [Pi Durable](https://earendil.com/posts/pi-durable/) — the Earendil post introducing pi-durable.
- [Pi Durable on Cloudflare](https://earendil.com/posts/pi-durable-on-cloudflare/) — the Earendil post on running pi-durable on Cloudflare.
