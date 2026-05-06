# HANDOFF — Phase A: Microchip / PolarFire SoC integration into openFPGALoader

**Status:** workspace initialised, no code changes yet. This handoff
is the primary input for the next session that picks up the work.

**Branch:** `feat/microchip-polarfire-soc` (already created, clean)

**Session originator:** 2026-05-06

---

## Read this first if you've never seen this fork

Two-minute orientation — the rest of the handoff assumes this:

1. **What:** Add Microchip PolarFire / PolarFire SoC support to
   [openFPGALoader](https://github.com/trabucayre/openFPGALoader),
   an Apache-2.0 multi-vendor FPGA programmer that today covers
   Xilinx, Intel, Lattice, Gowin, Efinix, Anlogic, and Cologne Chip
   — but conspicuously not Microchip.

2. **How:** Integrate
   [microchip-fpga/directc](https://github.com/microchip-fpga/directc)
   (Microchip's MIT-licensed JTAG programming library — designed
   explicitly for "Linux SBC + JTAG cable + PolarFire" use cases)
   as a new vendor backend in openFPGALoader.

3. **Why:** ARC's Universal Programming Bridge proposal needs a
   single license-clean, license-server-free open-source binary that
   programs every FPGA family ARC targets. openFPGALoader handles all
   non-Microchip vendors; this fork closes the Microchip gap.

4. **License:** MIT + Apache-2.0 are fully compatible. directc files
   keep their MIT headers when imported; new wrapper files we write
   are Apache-2.0. See `STYLE.md` in this folder.

5. **Scope of "Phase A":** skeleton + IDCODE detection. The kit gets
   identified when probed; programming itself is **Phase B**. Don't
   get pulled into programming until Phase A builds clean and probes
   the kit successfully.

---

## Repo layout you're working in

```
/mnt/c/Programming/AI/
├── ARC/                    The parent project — all session
│                           context, BOOTLOADER_DEPLOYMENT.md,
│                           UNIVERSAL_PROGRAMMING_BRIDGE.md, etc.
├── openFPGALoader/         You are here. Active branch:
│   ├── .git/               feat/microchip-polarfire-soc
│   ├── .ai/                BOOTSTRAP, STYLE, NOTES, this handoff
│   ├── src/                Existing openFPGALoader code — read
│   │                       lattice.cpp + altera.cpp first
│   ├── CMakeLists.txt      Top-level build — needs new entries
│   ├── doc/                User-facing docs — add a microchip page
│   └── README.md           Add Microchip to supported vendors list
└── directc/                Reference clone, MIT, READ-ONLY for
                            our purposes — copy files OUT of here,
                            never push back.
```

`origin` = `paul-jvb/openFPGALoader.git`.
`upstream` = `https://github.com/trabucayre/openFPGALoader.git` (read-only).

Don't push to `main`. Push to `feat/microchip-polarfire-soc` only.

---

## What "done for Phase A" looks like

A successful Phase A end-state:

```bash
cd /mnt/c/Programming/AI/openFPGALoader
mkdir -p build && cd build
cmake .. && make -j$(nproc)
# build succeeds with no errors / no -Werror failures

# kit is attached via FlashPro JTAG (J5 on Icicle Kit)
sudo ./openFPGALoader --detect
# expected output (or similar): identifies the chip as PolarFire SoC
# IDCODE 0xf8121cf1 (or whatever MPFS250T's actual IDCODE is)
```

No programming yet. No bitstream support yet. Just: `--detect` works
and prints a sensible "this is a Microchip PolarFire SoC". That's
the Phase A bar.

---

## File layout to create

```
src/microchip/                     <- new directory, holds all
│                                     directc-derived sources
├── LICENSE-directc.txt            <- copy of directc's MIT LICENSE
├── dpalg.c        (MIT, copied)
├── dpalg.h        (MIT, copied)
├── dpcom.c        (MIT, copied)
├── dpcom.h        (MIT, copied)
├── dputil.c       (MIT, copied)
├── dputil.h       (MIT, copied)
├── dpuser.c       (MIT origin, ADAPTED for openFPGALoader Jtag*)
├── dpuser.h       (MIT origin, ADAPTED)
├── G5Algo/
│   ├── dpG5alg.c  (MIT, copied)
│   └── dpG5alg.h  (MIT, copied)
├── JTAG/
│   ├── dpchain.c  (MIT, copied)
│   ├── dpchain.h  (MIT, copied)
│   ├── dpjtag.c   (MIT, copied; may need light adaptation)
│   └── dpjtag.h   (MIT, copied)
└── SPIFlash/
    ├── dpS25F.c   (MIT, copied — only needed if SPI-direct path used)
    ├── dpS25F.h
    ├── dpSPIalg.c
    ├── dpSPIalg.h
    ├── dpSPIprog.c
    └── dpSPIprog.h

src/microchip.cpp                  <- NEW Apache-2.0 wrapper class
src/microchip.hpp                  <- NEW Apache-2.0 header
```

The wrapper class `Microchip` extends `Device` (and probably also
`FlashInterface` once Phase B starts the programming path). Mirrors
the shape of `Lattice : public Device, FlashInterface`.

---

## Concrete Phase A starting tasks

In order:

### A.1 — Copy directc sources into `src/microchip/`

```bash
cd /mnt/c/Programming/AI/openFPGALoader
mkdir -p src/microchip/G5Algo src/microchip/JTAG src/microchip/SPIFlash
cp /mnt/c/Programming/AI/directc/dp{alg,com,util,user}.{c,h} src/microchip/
cp /mnt/c/Programming/AI/directc/G5Algo/* src/microchip/G5Algo/
cp /mnt/c/Programming/AI/directc/JTAG/* src/microchip/JTAG/
cp /mnt/c/Programming/AI/directc/SPIFlash/* src/microchip/SPIFlash/
cp /mnt/c/Programming/AI/directc/LICENSE src/microchip/LICENSE-directc.txt
```

Don't modify these files yet (except dpuser.c — see A.2). Keep their
SPDX MIT headers verbatim.

### A.2 — Rewrite `src/microchip/dpuser.c` to bridge to openFPGALoader Jtag*

`dpuser.c` is directc's "platform abstraction" — currently uses
`libgpiod` to bit-bang JTAG signals on Raspberry Pi GPIO pins. We
need to replace this with calls into openFPGALoader's `Jtag *jtag`
class.

The Jtag API surface (from `src/jtag.hpp`):

```cpp
int  shiftIR(unsigned char *tdi, unsigned char *tdo, int irlen, ...);
int  shiftIR(unsigned char tdi, int irlen, ...);
int  shiftDR(const uint8_t *tdi, unsigned char *tdo, int drlen, ...);
int  read_write(const uint8_t *tdi, unsigned char *tdo, int len, char last);
void toggleClk(int nb, uint8_t tdi = 0);
void set_state(tapState_t newState, const uint8_t tdi = 1);
```

directc functions to bridge (search `dpuser.c` and `dpjtag.c`):

| directc function | openFPGALoader equivalent |
|---|---|
| `dp_jtag_init(...)` | no-op (openFPGALoader's Jtag is already initialised by main) |
| `dp_jtag_tms()` set | `jtag->set_state(...)` to navigate TAP |
| `dp_delay_ms(n)` | `usleep(n * 1000)` or use openFPGALoader's existing helper |
| `IRSCAN_in(IR_value, IR_size)` | `jtag->shiftIR(IR_value, IR_size, ...)` |
| `DRSCAN_in(...)` / `DRSCAN_out(...)` | `jtag->shiftDR(...)` |
| `goto_jtag_state(tap_state)` | `jtag->set_state(tapState_t)` |

The cleanest approach: **rewrite `dpuser.c` as a thin C shim that
hands JTAG operations to a global `Jtag *` set by the Microchip
class on construction**. directc keeps its C interface; the C++
class owns the bridge.

Pattern (sketch):

```c
/* src/microchip/dpuser.c — adapted */
#include "dpuser.h"
extern void *g_microchip_jtag;  /* opaque ptr to openFPGALoader Jtag* */

void dp_jtag_init(void) {
    /* no-op; Jtag is initialised by openFPGALoader main */
}

void dp_delay_ms(unsigned int ms) {
    usleep(ms * 1000u);
}

/* IR/DR shifts dispatch via a C-callable shim into the C++ class.
 * The shim is in microchip.cpp under extern "C" linkage. */
extern unsigned int microchip_shift_ir(unsigned int ir, unsigned int len);
extern void microchip_shift_dr(const unsigned char *tdi,
                               unsigned char *tdo, unsigned int len);
extern void microchip_set_tap_state(int state);

void IRSCAN_in(unsigned int ir, unsigned int len) {
    microchip_shift_ir(ir, len);
}
/* ... etc ... */
```

`microchip.cpp` defines the shims under `extern "C"` and routes them
to the held `Jtag*` member.

### A.3 — Write `src/microchip.hpp` + `src/microchip.cpp` skeleton

Mirror `lattice.hpp` / `lattice.cpp`'s shape. Initial scope:

```cpp
// src/microchip.hpp — Apache-2.0
class Microchip: public Device {
public:
    Microchip(Jtag *jtag, std::string filename, const std::string &file_type,
              Device::prog_type_t prg_type, bool verify, int8_t verbose);
    uint32_t idCode() override;
    void reset() override;
    void program(unsigned int offset, bool unprotect_flash) override;
    /* Phase A: program() can be a stub that prints
     * "Programming not yet implemented" and returns. */
private:
    /* family identifier set during construction based on idCode */
    enum mchp_family { UNKNOWN, POLARFIRE, POLARFIRE_SOC } _family;
};
```

`Microchip::idCode()` reads the standard JTAG IDCODE register (0x01
or per-family). For PolarFire SoC, IDCODE returns vendor + part info.

### A.4 — Find the actual PolarFire SoC IDCODE

The Icicle Kit has MPFS250T-FCVG484. Need its JTAG IDCODE for the
`fpga_list` table in `src/part.hpp`.

Two ways to find it:
- **From directc source:** grep for `IDCODE` / `0x` in directc — the
  G5 algo will reference the expected ID
- **From the kit empirically:** before adding to `part.hpp`, run
  `openFPGALoader --detect --cable <some-ftdi-cable>`. With a Microchip
  vendor not yet registered, openFPGALoader will print the unknown
  IDCODE but fail to dispatch. Capture that hex value, then add it
  to `part.hpp`.

Add to `src/part.hpp`'s `fpga_list` map:

```cpp
/**************************************************************************/
/*                            Microchip / Microsemi                       */
/**************************************************************************/

/* PolarFire SoC */
{0xfXXXXXXX, {"microchip", "polarfire-soc", "MPFS250T-FCVG484", 8}},
/* PolarFire (non-SoC) */
{0xfYYYYYYY, {"microchip", "polarfire", "MPF300T", 8}},
```

(Replace XX with actual IDCODEs.)

### A.5 — Wire CMakeLists.txt

Two parts:

**Top-level `CMakeLists.txt`:** add the C source files to the build:

```cmake
# add to OPENFPGALOADER_SRCS or equivalent list:
src/microchip.cpp
src/microchip/dpalg.c
src/microchip/dpcom.c
src/microchip/dputil.c
src/microchip/dpuser.c
src/microchip/G5Algo/dpG5alg.c
src/microchip/JTAG/dpchain.c
src/microchip/JTAG/dpjtag.c
src/microchip/SPIFlash/dpS25F.c
src/microchip/SPIFlash/dpSPIalg.c
src/microchip/SPIFlash/dpSPIprog.c
```

Plus include paths:

```cmake
target_include_directories(openFPGALoader PRIVATE
    src/microchip
    src/microchip/G5Algo
    src/microchip/JTAG
    src/microchip/SPIFlash
)
```

**Build flags:** directc uses `-Werror -Wunused-function`. If
openFPGALoader uses different warning flags, the C files might fail
under `-Wunused`. Two options: (a) add `-Wno-unused-function` for
the microchip subdir, (b) edit directc files to mark functions
`static` where appropriate. (a) is faster and cleaner for upstream.

### A.6 — Register Microchip in the dispatch table

openFPGALoader maps IDCODE → vendor class somewhere. Find it:

```bash
cd /mnt/c/Programming/AI/openFPGALoader
grep -nE 'new Lattice|new Altera|new Anlogic' src/*.cpp
```

Whichever file does the dispatch — typically `src/openFPGALoader.cpp`
or similar — add a Microchip case:

```cpp
} else if (vendor_name == "microchip") {
    target = new Microchip(jtag, ..., ...);
}
```

### A.7 — Build and probe

```bash
cd /mnt/c/Programming/AI/openFPGALoader
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Debug
make -j$(nproc) 2>&1 | tee build.log
```

Iterate until clean compile. Then run `--detect` against the kit
(needs an FTDI-class JTAG cable — FlashPro5 should work via libusb,
or use a generic FT2232H breakout).

---

## Reference: directc structure

```
dpalg.{c,h}          Init + data integrity checks (top-level entry
                     point: dp_top())
dpcom.{c,h}          Image streaming / file I/O abstraction
dputil.{c,h}         Utility functions
dpuser.{c,h}         Platform abstraction — THIS IS THE FILE TO REWRITE
                     (currently libgpiod-based; we replace with Jtag*)
G5Algo/dpG5alg.{c,h} PolarFire-specific programming algorithm
                     (G5 = "Generation 5", Microsemi naming)
JTAG/dpchain.{c,h}   JTAG chain handling (multi-device chains)
JTAG/dpjtag.{c,h}    JTAG state-machine + IR/DR shifts
SPIFlash/...         SPI-direct programming variant (less relevant —
                     openFPGALoader already has SPI flash support;
                     evaluate per-need)
```

Top-level entry from directc's main: `dp_top()` in `dpalg.c`. This
takes a code (program / verify / erase / read DSN / etc.) and
dispatches to the right algorithm.

For Phase A's IDCODE-only goal, you don't even need to call
`dp_top()` — `Microchip::idCode()` can do a direct
`jtag->shiftIR(IDCODE_OPCODE, irlen=8); jtag->shiftDR(...)` and skip
directc entirely. directc only becomes relevant in Phase B.

---

## How to test against the Icicle Kit

The kit is currently programmed with the **ARC bootloader** (commit
`4d36573` from the parent ARC repo). Don't reflash the kit during
Phase A — just probe via JTAG.

JTAG access:

- The kit's JTAG header is J5 (a 10-pin Microsemi-style header).
- Standard FlashPro5 USB cable plugs into J5. openFPGALoader supports
  FlashPro5 via libusb — pass `--cable flashpro5` or whatever the
  exact name is in openFPGALoader's cable table.
- Alternative: any FT2232H breakout wired to TCK/TMS/TDI/TDO on J5
  per the directc README's pin diagram.

Probe command:

```bash
sudo ./openFPGALoader --detect --cable flashpro5
```

Or list cables to find the right name:

```bash
./openFPGALoader --list-cables
./openFPGALoader --list-fpgas
```

If the kit doesn't enumerate, JTAG cable is the most likely problem
— openFPGALoader has well-known support for FT2232H and J-Link;
FlashPro5 may need libusb permissions setup
(`/etc/udev/rules.d/70-openfpgaloader.rules` is in the repo top
level — install it).

---

## Outstanding research questions for Phase A

1. **PolarFire SoC IDCODE format.** The high nibble convention
   openFPGALoader uses is "version" (set to 0). Need to confirm
   PolarFire's IDCODE encoding — Microsemi/Microchip IDCODEs have a
   unique JEDEC manufacturer ID (`0x0F1` per JEP106). Look at
   Microchip's *PolarFire FPGA Programming User Guide* (DS5000003)
   §JTAG IDCODE for the exact bits.

2. **IR length.** Lattice uses 8 bits, Xilinx 6 bits, etc. PolarFire
   SoC needs to be confirmed — `irlength` field in `fpga_list`.
   Likely 8.

3. **Cable support — FlashPro5/6 in openFPGALoader.** Verify whether
   openFPGALoader has direct FlashPro5/6 support or whether we need
   a generic FT2232H breakout. If FlashPro5 isn't supported, that's
   either a separate sub-project or we use a generic cable for the
   bring-up.

4. **directc adaptation effort: how invasive is dpuser.c rewrite?**
   Estimate after reading the file once. If the directc code paths
   are tightly coupled to libgpiod assumptions (timing dependencies,
   GPIO direction control mid-shift), the rewrite is bigger than the
   handoff suggests.

5. **Microchip vendor's `UNIQUE_ID` / silicon signature read.**
   directc supports `READ_DSN` (Device Serial Number) — useful for
   ARC's bundle target_id field. Worth exposing from the Microchip
   class, but only after Phase A baseline.

---

## Cross-references back into ARC

- `/mnt/c/Programming/AI/ARC/docs/architecture/UNIVERSAL_PROGRAMMING_BRIDGE.md`
  — full proposal context. Phase 2b in that doc IS this fork's
  work.
- `/mnt/c/Programming/AI/ARC/docs/architecture/BOOTLOADER_DEPLOYMENT.md`
  — why this work exists (BANK3 silicon-lockdown blocking
  bootloader-internal OTA).
- `/mnt/c/Programming/AI/ARC/.ai/NOTES.md` 2026-05-06 entries —
  empirical silicon evidence + DS60001727L citation that closed the
  in-MSS path question.

When this fork lands a working PolarFire backend upstream, ARC's UPB
proposal can move from "two binaries on the bridge" to "one
binary" — that's the long-term simplification this work delivers.

---

## Building openFPGALoader from source — quick reference

Per upstream README:

```bash
cd /mnt/c/Programming/AI/openFPGALoader
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Debug ..
make -j$(nproc)
sudo make install        # optional; installs to /usr/local
```

Dependencies (Ubuntu/Debian): `libusb-1.0-0-dev libftdi1-dev libudev-dev pkg-config zlib1g-dev cmake`.

For development without `sudo make install`, run the binary in-tree:

```bash
sudo ./openFPGALoader --detect
```

(Needs sudo unless udev rules are installed for the cable's USB
vendor:product.)

---

## What NOT to do in Phase A

- Don't try to program the kit. That's Phase B.
- Don't push to `origin/main`. Use `feat/microchip-polarfire-soc`.
- Don't open an upstream PR yet. PR happens after Phase B works.
- Don't modify upstream files outside `src/microchip/`,
  `src/microchip.{cpp,hpp}`, `src/part.hpp`, `CMakeLists.txt`, the
  dispatch file, `README.md`, and `doc/`. Keep the diff surgical.
- Don't bring in `libgpiod` as a dep — directc uses it; we replace
  it with openFPGALoader's `Jtag*`.

---

## When you finish Phase A

1. Update `.ai/NOTES.md` with a dated entry summarizing what landed
   and what the IDCODE actually returns.
2. Either update this handoff in place with a "Phase A: COMPLETE"
   marker + the diff size, or write a fresh
   `HANDOFF_phase_b_microchip_programming.md` describing the next
   session's scope.
3. Commit + push to `feat/microchip-polarfire-soc`. Don't merge to
   `main` — keep the upstream-tracking branch clean.
4. If the IDCODE detection works on silicon, log the result back in
   the parent ARC project's `.ai/NOTES.md` 2026-05-06 thread for
   posterity.

---

## Estimated scope

- Phase A: ~3-4 hours of focused work for someone familiar with
  CMake + openFPGALoader's structure. Most time goes into
  `dpuser.c` rewrite + first clean build.
- Phase B (programming): ~1 day.
- Phase C (silicon validation): ~half day.
- Phase D (polish + upstream PR): ~half day.

Total: about 1 week of focused work spread across ~3-4 sessions.

If something blocks beyond a session boundary (toolchain issues,
unexpected directc coupling, IDCODE mystery), capture the blocker in
`.ai/NOTES.md` and write a new handoff with the smallest
unblocking experiment as its scope.
