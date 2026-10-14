# Reimplementing the Konica 206 driver in C for musl — feasibility assessment

**Verdict: technically feasible, but it is a real reverse-engineering project, not
a port.** This document scopes it against the binaries as measured on 2026-10-14.

Companion to [alpine-musl-test.md](alpine-musl-test.md), which established that
`245igdirf` cannot run on musl. That answer is "the vendor binary will never work
here" — this document is the only remaining question: **can we replace it?**

---

## 1. What is already musl-clean

`konica206-native/src/konica_usb.c` — 276 lines, pure `libusb`, **done**:

| Capability | Status |
|---|---|
| Serial match, endpoint discovery on interface 1 | working |
| 8192-byte chunked writes | working |
| Partial-write resume, stall → `ETIMEDOUT` after bounded retries | working |
| Unplug → `ENODEV` | working |

This is the layer that made the business USB-safe, and it builds against musl
unchanged. **The hard part is not transport.**

---

## 2. What would have to be written

| Layer | Assessment |
|---|---|
| Raster ingest (CUPS raster v2/v3) | Easy — documented format |
| PPD parsing | Easy — plain-text format, ~300 lines |
| Halftone / error-diffusion dithering | Textbook. The binary's own symbols name it: `ImxErrDiffAMTCreate/Configure/Image/Destroy` |
| Colour transform | Use `lcms2` — it builds on musl (`apk add lcms2-dev`) and maps directly onto the binary's `ImxCmmProfile*` / `PCM_f*` layer |
| **Device command stream (PJL/TBCP)** | **The hard part — proprietary, undocumented, inferable only from captures** |

The vendor's own export list is effectively a table of contents for what must be
reproduced:

```
ImxErrDiffAMT*        error-diffusion halftoning
ImxCmmProfile*        ICC colour management
PCM_fHLS2sRGB         colour space conversions
PCM_fJab2XYZ
PCM_fLab2Lch
PalFileRead/Seek/Write    indexed/palette raster I/O
cupsRasterOpen/ReadHeader/ReadPixels
```

---

## 3. The scale correction — it is ~1.3 MB, not 530 KB

An earlier estimate of "weeks, not hours" was made before `mtorf.ocm` was measured.
It is **not a data file**:

```
$ file mtorf.ocm; stat -c%s mtorf.ocm
ELF 64-bit LSB shared object, x86-64   798904 bytes

not stripped · 696 exported symbols · C++
zlib statically linked in (deflate/adler32/crc32/crc32_combine family)
own ConfigIntGet / ConfigStringGet configuration layer
own dlopen / dlsym / dlclose  (it loads further plugins itself)
same ppd* + cupsParseOptions/cupsMarkOptions deps as the main filter
heavy pthread use; C++ runtime (__cxa_throw, __gxx_personality_v0, _ZTVSt9bad_alloc)
```

So the vendor IP is:

| Component | Size |
|---|---|
| `245igdirf` | 530,848 B (patched; 522,656 B stock) |
| `mtorf.ocm` | 798,904 B — C++, zlib embedded, 696 symbols |
| Halftones | 2 × `.hft` screens |
| Colorworlds | 6 × `.cw` |
| Profiles | 7 × `.icm` |

**≈1.3 MB of closed code plus data.** Treat "weeks" as optimistic until L1 lands.

---

## 4. Why this is nonetheless tractable: you have an oracle

The decisive asset is that **the glibc setup works**. That converts blind RE into
differential testing:

```
   same raster input
          │
    ┌─────┴─────┐
    ▼           ▼
 vendor filter   musl implementation
 (glibc host)    (the new work)
    │           │
    ▼           ▼
 capture.bin    output.bin
          └─────┬─────┘
                ▼
          byte-for-byte diff
```

The loop is tight and objective. Reverse-engineering a print driver *blind* is
brutal, because failures are plausible rather than loud — slightly wrong dithering
still produces a page that looks mostly right. Diffing against known-good output
turns "looks wrong" into "differs at byte 41,203".

`ubuntu-printing`'s README already documents the capture technique (a pass-through
`tee` wrapper in the `*cupsFilter` line, since removed).

### Step 0 is a prerequisite, and it is missing

There are **no captured printer streams on disk** — checked `/var/spool/cups`,
`/var/tmp`, and the `konica-retrofit` repo. Nothing to diff against yet.

Building a capture corpus is cheap, runs entirely on the working Ubuntu setup, and
is useful **regardless** of which path is chosen afterwards. It is the first thing
to build.

A useful corpus captures at least: monochrome simplex A4; monochrome duplex A4
(2 pages, long-edge and short-edge); colour A4; each paper size the PPD offers;
and the resolution settings (300 / 600 dpi).

---

## 5. Realistic ladder

| | Target | Notes |
|---|---|---|
| **L0** | Capture corpus | Prerequisite. Cheap. Unblocks everything. |
| **L1** | Monochrome A4 simplex | Smallest thing that proves the protocol |
| **L2** | Duplex | Needs correct back-side orientation — the `*cupsBackSide: Rotated` behaviour already fought once |
| **L3** | Colour CMYK | 4 halftone screens + ICC CMM via `lcms2` |
| **L4** | Paper path, error recovery, toner save | Open-ended; likely the real tail |

**L1–L2 look realistic.** L3+ is where the schedule goes to infinity, and L4 may
never terminate usefully.

A practical sequencing note: L1 does not need a perfect colour pipeline — start
1-bit monochrome, where halftoning is a single screen and ICC is a pass-through.

---

## 6. The alternative worth pricing first

If the goal is a **small container**, a **glibc-based image** delivers the working
driver today:

```dockerfile
FROM debian:bookworm-slim
# vendor filter + patched binary + private libcups, all as on the Ubuntu host
```

That is hours of work, not weeks, and it actually prints. Reimplementation is only
the right trade if **musl is a hard requirement** — smaller images, no glibc CVE
surface, static linking — rather than a preference.

Worth deciding before writing L0, because the capture corpus is useful either way
but the rest is not.

---

## 7. Licensing note on the data files

Reimplementing the protocol for hardware you own is ordinary interoperability
work. **Redistributing Konica's data files is not** — `Halftones/*.hft`,
`Colorworlds/*.cw` and `Profiles/*.icm` carry their own terms (see `COPYING` in the
deb). A replacement driver would need its own halftone screens and profiles, or
clean-room equivalents.

The `COPYING` file shipped in the deb is the authoritative statement; read it
before publishing anything derived from the driver.

---

## 8. Recommendation

1. **Decide first** whether musl is a hard requirement or a preference (§6). If
   a preference, stop here — use a glibc image and keep printing.
2. If musl is hard: build **L0** (capture corpus) before anything else. It is cheap,
   independent of the decision, and it is what makes the rest measurable.
3. Attempt **L1** (monochrome A4 simplex). It is the honest go/no-go: if the device
   stream can be reproduced and matched byte-for-byte for the simple case, the
   approach is validated. If L1 stalls, L3 is not going to happen.
4. Only then consider L2+.

Useful instrument for step 3: the binary is `not stripped` with a readable symbol
table, so a toolchain that drives Ghidra (e.g. `morluto/rea`) helps *read* it. It
will not port anything — it is a comprehension aid, not a converter.

---

## 9. Cross-references

- [alpine-musl-test.md](alpine-musl-test.md) — why the vendor binary cannot run on
  musl (loader loop, `LD_BIND_NOW=1` evidence)
- [README.md](README.md) — the working glibc reference setup, including the
  capture-wrapper technique
- [konica206uri.md](konica206uri.md) — the PAPPL primary queue
- `vinod2807/konica-retrofit` → `install-konica-245igdi-alpine.sh` — musl install
  with the negative result documented in its header
- `vinod2807/konica206-native` — the PAPPL app whose `konica_usb.c` this reuses