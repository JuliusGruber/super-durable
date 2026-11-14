# Spike: Router and one Worker on pi-durable

Throwaway code. It checks whether the core of the Super-agent design runs on
pi-durable 1.5.2 on a Node host: a Router conversation with batched Trigger
intake, one background Worker, read-only inspection, and a restart.

The model is pi-ai's scripted faux provider, so no API key is needed. All of it
is in [main.ts](main.ts).

## Run

Needs Node 22.19 or later (pi-durable's requirement).

```bash
npm install
npm start
```

With fnm and an older system Node: `fnm exec --using=22.22.3 npm start`.

## What it showed (run of 14 November 2026, Node 22.22.3, Windows)

| Question | Result |
|---|---|
| Does the Router stay responsive while a Worker runs? | Yes. It answered 1.3 s after the work request; the Worker's tool takes 6 s. |
| Are Triggers that arrive during a turn batched? | Yes. Triggers per Router run were `[1, 1, 3, 1]`: the three that queued behind a running turn were placed together with `followUpMode: "all"`. |
| Does request-ID dedup work for Trigger retries? | Yes. Six Triggers in, one dropped by the allowlist, one retry deduplicated, four submissions admitted. |
| Does the Worker's answer reach the Router as a new input? | Yes, through the reporter task, once. |
| Can a second Harness inspect a Worker while the writer is open? | Yes. A `readOnly` Harness on the same SQLite file saw the Worker as working with one live tool slot, and its write was refused with `ReadOnlySession`. |
| Does it survive a restart mid-work? | Yes for a clean close: queued Triggers, the Worker's run and the report all continued after reopening. |

## What it did not show

- **A real crash.** The restart is `harness.close()`, not a killed process.
  Close waited about 5.6 s for the Worker's tool, because the tool ignores its
  abort signal. The tool then ran again from the start after reopening (it is
  declared `replay: "safe"`), so the work took twice as long.
- **A real model.** The faux model is a script. Whether a real Router model
  delegates reliably and stays quiet on noise is untested.
- **Real execution.** The Worker's tool is a timer. No shell, files, remote
  machine or per-Worker directory.
- **The rest of the design.** No Thread subscriptions or pass-through, no
  model-side Trigger filtering, no `stop`, no Watchdog judgement, no recall.

## Notes for a real build

- The Worker's child conversation starts as a copy of the Router's agent, so
  it must be reconfigured in the creating commit (here: `extensions:
  [WorkerTools]`).
- The Router's interrupted turn was sent again after reopening, as the README
  says for a close.
- The types were not checked: Node strips them without checking, and the code
  uses a few `as never` casts for ID types.
