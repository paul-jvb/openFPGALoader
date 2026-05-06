# Style — openFPGALoader fork

Defer to upstream conventions where they apply. Specifics for this
fork:

## Commit messages

Match upstream style: `<area>: <short summary>`. Examples seen in
upstream history:

- `microchip: add PolarFire SoC IDCODE detection`
- `microchip/directc: import MIT sources from microchip-fpga/directc`
- `CMakeLists: wire microchip backend`
- `doc: PolarFire SoC support`

Body: optional, but for non-trivial changes include the reasoning
(not just "what" — the diff says what; the body says **why**).

## License headers

Every new C/C++ file gets an SPDX header. Two cases:

**1. New code we wrote (clean-room or original).** Apache-2.0,
matching the rest of openFPGALoader:

```cpp
// SPDX-License-Identifier: Apache-2.0
/*
 * Copyright (C) 2026 Paul Vilas-Boas <paul.vilas-boas@esseuro.com>
 */
```

**2. Code imported from `microchip-fpga/directc`.** Preserve the
original MIT header verbatim:

```c
// SPDX-License-Identifier: MIT
/*
 * Copyright (c) 2023 Microchip Technology Inc. All rights reserved.
 */
```

If we modify imported files non-trivially, append a co-copyright line
keeping both attributions:

```c
// SPDX-License-Identifier: MIT
/*
 * Copyright (c) 2023 Microchip Technology Inc. All rights reserved.
 * Copyright (C) 2026 Paul Vilas-Boas — adapted for openFPGALoader
 */
```

A top-level `LICENSE-directc.txt` documents the MIT origin of the
imported sources for upstream PR reviewers.

## File layout

Microchip-specific code under `src/microchip/` to keep it
co-locatable + greppable. Don't scatter directc files across `src/`
root.

## Code conventions

- C++ style for the openFPGALoader-side wrapper (mirrors
  `lattice.cpp`, `altera.cpp`)
- C style preserved for imported directc sources (don't rewrite —
  adapter only)
- Build via `cmake` per upstream README — don't introduce a separate
  build system
- Test against the Icicle Kit (we have one attached). Document
  results in `.ai/NOTES.md`

## What goes in `.ai/` vs README

| Information | Where |
|---|---|
| ARC-project context | `.ai/` only |
| Internal decision logs | `.ai/NOTES.md` |
| Handoffs between sessions | `.ai/HANDOFF_*.md` |
| User-facing PolarFire SoC docs | upstream `doc/` folder + README |
| PR description text | upstream-suitable, no ARC references |

When the integration is mature enough for an upstream PR, the PR
description gets written fresh — not copy-pasted from `.ai/`. ARC
context stays internal.
