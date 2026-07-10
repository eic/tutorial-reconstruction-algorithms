---
title: "Instructor Notes"
---

This lesson assumes learners already have a working `eic-shell` environment and a checkout of
[EICrecon](https://github.com/eic/EICrecon); make sure they have completed the
[Setup](../learners/setup.md) page before the session, since building EICrecon from scratch can take
a while.

## Emphasis

- The single most important conceptual point is the split between framework-independent
  **algorithms** (`src/algorithms`) and framework-facing **factories** (`src/factories`). Keep
  returning to it — most of the later episodes are variations on this theme.
- `JOmniFactory` is the only factory base class learners need to write new code. Mention the older
  base classes only so learners recognize them in existing code.

## Pacing

- Episodes 02–07 build up a single running example (the electron finder). Encourage learners to keep
  their own EICrecon checkout open and edit along, rather than only reading.
- The exercises are cumulative: each one extends the factory/algorithm from the previous episode.
  If a learner falls behind, the final code in episode 07 ("Putting everything together") can be
  used as a reference to catch up.
