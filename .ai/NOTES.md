# Notes — openFPGALoader fork

Decision log, dated entries newest first. One entry per decision.

---

## 2026-05-06 — Workspace initialised

Cloned `paul-jvb/openFPGALoader` (origin) and `paul-jvb/directc`
(reference). Added `upstream` remote pointing at
`trabucayre/openFPGALoader`. Created branch
`feat/microchip-polarfire-soc` for the integration work — `main`
stays sync-able with upstream.

`.ai/` scaffolding created:
- `BOOTSTRAP.md` — folder purpose
- `STYLE.md` — commit + license-header conventions
- `NOTES.md` — this file
- `HANDOFF_phase_a_microchip_integration.md` — primary input for the
  next session

No code changed yet. The only diff from upstream so far is `.ai/`.
First code commit for the next session is the file-layout + skeleton
microchip backend (see handoff).

---

## 2026-05-06 — License compatibility confirmed

`microchip-fpga/directc` ships under MIT (per its top-level `LICENSE`
file and per-file SPDX headers — `dpalg.c`, `dpcom.c`, `dputil.c`,
`dpuser.c` and headers all carry `SPDX-License-Identifier: MIT`).

openFPGALoader is Apache-2.0.

MIT and Apache-2.0 are fully compatible — MIT can be incorporated
into Apache-2.0 projects with attribution preservation. Per-file
SPDX headers stay as written. Imported directc files keep MIT;
new wrapper code is Apache-2.0. A top-level `LICENSE-directc.txt`
documents the import origin for upstream PR reviewers.

This is significantly cleaner than the original assumption that
directc shipped under Microchip's standard FPGA-tools EULA (which
would have required clean-room reimplementation). Direct code import
is legally clean.

---

## 2026-05-06 — Why this fork exists (cross-ref)

In service of ARC's Universal Programming Bridge proposal:
`/mnt/c/Programming/AI/ARC/docs/architecture/UNIVERSAL_PROGRAMMING_BRIDGE.md`.
ARC's bootloader work was blocked by Icicle Kit's BANK3 SC-SPI silicon
lockdown (see ARC's `.ai/NOTES.md` 2026-05-06 entries). The pivot is
external bridge SBC + open-source JTAG tooling. PolarFire SoC isn't
covered by openFPGALoader today — this fork closes that gap.
