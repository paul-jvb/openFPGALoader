# `.ai/` — session-shared context for the openFPGALoader fork

This directory mirrors the pattern used in the ARC project
(`/mnt/c/Programming/AI/ARC/.ai/`) so future sessions in this repo
can pick up where the previous one left off.

## What lives here

- **`HANDOFF_*.md`** — primary input for the next session. Read this
  first. Each handoff is self-contained: goal, prior state, concrete
  next steps, references.
- **`NOTES.md`** — decision log, dated entries newest-first. Capture
  decisions with non-obvious context that future sessions need but
  can't derive from the code or commit history.
- **`STYLE.md`** — short repo-specific conventions (commit message
  shape, file headers, license tags). Defers to upstream
  `trabucayre/openFPGALoader` conventions where they apply.

## Why this fork exists

This is a fork of [trabucayre/openFPGALoader](https://github.com/trabucayre/openFPGALoader)
created to **add Microchip PolarFire / PolarFire SoC support** to
openFPGALoader's already-broad multi-vendor coverage (Xilinx, Intel,
Lattice, Gowin, Efinix, Anlogic, Cologne Chip).

The work integrates [microchip-fpga/directc](https://github.com/microchip-fpga/directc)
(Microchip's MIT-licensed JTAG programming library for PolarFire) into
openFPGALoader's Apache-2.0 codebase. License compatibility is clean:
MIT and Apache-2.0 are both permissive and combine without restriction
when each file's headers are preserved.

## Why this matters to the ARC project

This fork exists in service of ARC's **Universal Programming Bridge**
proposal:

- See `/mnt/c/Programming/AI/ARC/docs/architecture/UNIVERSAL_PROGRAMMING_BRIDGE.md`
- ARC's bootloader work hit an architectural ceiling on the
  PolarFire SoC Icicle Kit (BANK3 SC-SPI silicon-locked, see ARC's
  `.ai/NOTES.md` 2026-05-06 entries)
- The pivot is to an external bridge SBC running open-source JTAG
  tooling — which requires open-source PolarFire programming, which
  doesn't exist today
- This fork closes that gap

## How to use this folder

- New session, no context: start with the most recent `HANDOFF_*.md`.
- Mid-task: update the active handoff with progress + next concrete
  steps before ending the session.
- Decision worth preserving: add a dated entry to `NOTES.md` (newest
  first).
- No need to mirror everything from upstream README — that's
  authoritative. `.ai/` is for ARC-specific context that wouldn't
  belong in a public PR.

## Branch layout

- `main` — tracks `upstream/main` from `trabucayre/openFPGALoader`.
  Don't commit local work here. Pull from upstream periodically to
  stay current.
- `feat/microchip-polarfire-soc` — active integration branch. All
  Microchip-related work happens here.

## Remotes

- `origin` → `paul-jvb/openFPGALoader` (your fork on GitHub)
- `upstream` → `trabucayre/openFPGALoader` (the canonical project)
