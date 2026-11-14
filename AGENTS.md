# Agent Instructions

Guidance for AI coding agents working in this repository.

## Project

`super-durable` explores whether the Super-agent architecture can be built on
pi-durable. It holds research notes in `research/` and throwaway spikes in
`spikes/`. There is no product code, build or test setup yet.

Spikes are TypeScript on Node 22.19 or later, run directly with `node` (no
build step). Each spike folder has its own `package.json` and README with how
to run it.

## Spikes

All spikes go into the `spikes/` folder, one subfolder per spike. A spike is
throwaway code written to learn something: it gets minimal or no tests, and
keeping what it produced is a separate decision.

## Git Workflow

This repository follows a continuous commit-and-push workflow:

- **Commit every change as soon as it is complete.** Don't batch unrelated
  changes together or let finished work sit uncommitted — each logical change
  gets its own commit with a clear, descriptive message.
- **Push immediately after every commit.** The remote is the source of truth;
  local work should never be more than one commit ahead of it.

The intent: the remote always reflects the current state of the work, every
change is individually traceable, and nothing is ever lost to an uncommitted
working tree.
