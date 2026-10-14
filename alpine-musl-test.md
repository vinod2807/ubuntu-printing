# Alpine / musl test — does `245igdirf` work on Alpine Linux?

**Verdict: no.** It loads, links, and is correctly invoked by CUPS on musl — but it
never emits a single byte of device data. No page prints.

Tested 2026-10-14 against the real printer over USB. Companion docs:
[README.md](README.md) (glibc host, the working reference) and
[konica206uri.md](konica206uri.md).

This document is the negative result, including the evidence and the dead ends, so
the question does not have to be answered again.

---

## 1. Environment

| Item | Value |
|---|---|
| Host | Ubuntu 26.04 `resolute` (glibc), CUPS 2.4.16, cups-filters 2.0.1 |
| Test target | **Alpine 3.21.3 x86_64**, musl 1.2.5 |
| Isolation | plain `chroot` from `alpine-minirootfs` — no docker/podman/debootstrap on the host |
| CUPS in chroot | 2.4.x, listening on `localhost:6631` (host already owns 631) |
| Printer | the same bizhub 206, `usb://KONICA%20MINOLTA/206?serial=A8A6041029423&interface=1` |
| Driver | patched `245igdirf` + `mtorf.ocm`, copied from `/usr/local/lib/konica/` |

No container runtime was available, so the Alpine side was bootstrapped by
downloading the minirootfs and `chroot`ing into it. `/dev` was deliberately
**unmounted** for every static test so the filter could not touch the printer; it
was only bound for the two real print attempts.

---

## 2. What the binary actually needs

The question "does it need `libcups.so.2`?" drives everything else, so it was
measured rather than guessed.

### 2.1 The executable itself is old and modest

```
INTERP:  /lib64/ld-linux-x86-64.so.2
NEEDED:  libc.so.6 libpthread.so.0 libdl.so.2 libm.so.6 libcups.so.2 libcupsimage.so.2
max GLIBC version required: GLIBC_2.3.2
```

A 2013-era binary. Nothing exotic.

### 2.2 The real dependency surface is 13 unversioned symbols

135 undefined symbols total. **122 are plain libc/libm/libpthread.** Only 13 come
from CUPS, and every one is **unversioned** (`DF *UND*`, no `@GLIBC_x`/`@CUPS_x`):

```
libcups.so.2      cupsGetOption  cupsParseOptions  cupsMarkOptions
                   ppdOpenFile  ppdClose  ppdFindAttr  ppdFindMarkedChoice
                   ppdIsMarked  ppdMarkDefaults  ppdPageSize
libcupsimage.so.2 cupsRasterOpen  cupsRasterReadHeader  cupsRasterReadPixels
```

It never calls `cupsEnumDests`, `cupsConnectDest` or `cupsFreeDests` — so it has
**zero** use for the DNS-SD and GSSAPI machinery that makes `libcups.so.2` heavy.

### 2.3 The GLIBC_2.38 scare was self-inflicted

`/usr/local/lib/konica/lib/libcups.so.2` on the host is a **byte-identical copy of
the host's own CUPS 2.4.16**, planted by `install-konica-anylinux.sh` step 5b so
`245igdirf` keeps working across CUPS 3.0:

```
$ md5sum /usr/local/lib/konica/lib/libcups.so.2 /usr/lib/x86_64-linux-gnu/libcups.so.2
801d26fff3b358bee6b6c2fdad9792be  /usr/local/lib/konica/lib/libcups.so.2
801d26fff3b358bee6b6c2fdad9792be  /usr/lib/x86_64-linux-gnu/libcups.so.2
```

That copy requires symbols a musl system cannot have:

```
__isoc23_sscanf  __isoc23_strtol  __isoc23_strtoll  __strlcpy_chk   @GLIBC_2.38
arc4random                                                              @GLIBC_2.36
pthread_create pthread_join pthread_once pthread_rwlock_*  (13)           @GLIBC_2.34
```

`__isoc23_*` are C23 names added in glibc 2.38 (2022). Alpine's `gcompat` is a
hand-written shim — **34 KB, 83 KB installed** — and has no C23 redirect symbols.
Copying a glibc libcups to Alpine is therefore a dead end, and it was never needed.

### 2.4 The Konica deb ships no libraries at all

Full contents of `konica-minolta-245igdi-cups_2.01_amd64.deb`:

```
245igdirf  mtorf.ocm  COPYING
205igdi.ppd  225igdi.ppd  245igdi.ppd
7 x *.icm    6 x *.cw    2 x *.hft
```

No `.so` whatsoever.

---

## 3. What musl actually needs — exactly two shims

### 3.1 `gcompat`

Supplies `ld-linux-x86-64.so.2`, `libc.so.6`, `libm.so.6`, `libpthread.so.0`.
Because `245igdirf` only needs `GLIBC_2.3.2`, gcompat's floor is comfortably
enough for the loader.

### 3.2 A `libdl.so.2` stub — gcompat does not provide one

```
$ apk info -L gcompat
lib/ld-linux-x86-64.so.2  lib/libc.so.6  lib/libcrypt.so.1  lib/libgcompat.so.0
lib/libm.so.6  lib/libpthread.so.0  lib/libresolv.so.2  lib/librt.so.1
lib/libutil.so.1  lib64/ld-linux-x86-64.so.2
```

`libdl.so.2` is absent, but `245igdirf` `DT_NEEDED`s it while importing only
`dlopen`, `dlsym`, `dlclose`, `dlerror` — all merged into libc in glibc 2.34. An
empty stub satisfies it:

```sh
printf 'int _konica_dl_stub;\n' > dl.c
gcc -shared -fPIC -o /lib/libdl.so.2 dl.c
```

### 3.3 Alpine's own musl libcups resolves all 13 symbols

An earlier assumption that a musl-linked libcups could not be loaded by a
gcompat process was **wrong**. It loads and binds every symbol:

```
libcups.so.2      => /usr/lib/libcups.so.2      (Alpine musl)
libcupsimage.so.2 => /usr/lib/libcupsimage.so.2 (Alpine musl)
libc.so.6 / libm.so.6 / libpthread.so.0 / libdl.so.2 => gcompat loader
```

So there is no missing-library problem. The failure is elsewhere.

---

## 4. The musl-only Ghostscript bug (real, and worth knowing)

Independently of the driver, Alpine's Ghostscript **10.05.1** fails on every page
with a `cups`-device PPD:

```
**** Error: Page drawing error occurred.
Could not draw this page at all, page will be missing in the output.
Sent 0 bytes...
```

`cups-filters` injects `<</.HWMargins[…] /Margins[0 0]>>setpagedevice` regardless of
the PPD, and gs 10.05.1's `cups` device produces a 0-byte raster for it.

This is the **same class of failure as §2 of [README.md](README.md)** and is fixed
the same way — and on Alpine the fix is **mandatory**, not optional:

```sh
apk add poppler-utils          # provides /usr/lib/cups/filter/pdftoraster
sed -i '/gstoraster/ { /application\/vnd\.cups-raster/ { s|^|#DISABLED | } }' \
    /usr/share/cups/mime/cupsfilters-ghostscript.convs
```

After that CUPS builds the intended chain:

```
pdftopdf → pdftoraster (poppler) → 245igdirf → usb backend
```

**Also use the full-bleed PPD, not the real-margins one.** The real-margins PPD
feeds `HWMargins` into `setpagedevice` and makes this look like a driver bug rather
than a Ghostscript bug — which is exactly the trap this cost time here.

---

## 5. Real hardware test

A queue was created in the chroot by writing `printers.conf` directly (`lpadmin`
returns `Unauthorized` because CUPS 2.4 local auth needs certs the chroot lacks):

```
device for ALPINE206: usb://KONICA%20MINOLTA/206?serial=A8A6041029423&interface=1
```

The **musl stack found and opened the real printer**:

```
libusb_get_device_list=12
STATE: +connecting-to-device
Printer found with device ID: MFG:KONICA MINOLTA;CMD:GDI,XPS;MDL:206;CLS:PRINTER;CID:KONICA MINOLTA206
Device protocol: 2
Sending data to printer.
Sent 0 bytes...
```

Everything up to and including the USB backend works on musl. Then:

```
PID 39084 (/usr/local/lib/konica/.../245igdirf) stopped with status 2.
PID 39639 (/usr/lib/cups/filter/pdftoraster) did not catch or ignore signal 13.
```

`245igdirf` died; `pdftoraster` then took SIGPIPE.

### Simplex and duplex — neither could be tested

`-o media=A4 -o sides=one-sided` and `-o sides=two-sided-long-edge` jobs were both
submitted. **Both failed identically with `Sent 0 bytes` and produced no paper**,
because the filter fails before the duplex/simplex setting is ever applied. There
is no duplex result to report, because there was no output to duplex.

---

## 6. Root cause

`245igdirf` is a **glibc binary** and `gcompat` is an ~83 KB shim, not a libc. The
13 symbols bind, but the raster, PPD and colour-management work behind them runs
against a libc the process is not really using.

**Loader loop.** Under `strace` the loader re-reads one ELF header from an
inherited fd, 20 times, then exits:

```
read(3, "\177ELF\2\1\1\0..."..., 960) = 960      (×20)
+++ exited with 4 +++
```

**Substituting a glibc libcups changes the behaviour but not the outcome.** A
glibc-built `libcups.so.2` (the host's) was dropped in ahead of musl's:

| libcups | exit | ELF re-reads | bytes out |
|---|---|---|---|
| Alpine musl | 2 | 20 | 0 |
| glibc-built stub | 113 | 6 | 0 |
| host glibc CUPS 2.4.16 | 9 | — | 0 |

**Last escape hatch closed.** Forcing eager binding surfaces every symbol the shim
must satisfy:

```
$ LD_LIBRARY_PATH=/opt/glibccups LD_BIND_NOW=1 .../245igdirf
exit=1
readlink: No such file or directory
```

gcompat cannot satisfy a real glibc libcups. There is no musl configuration in
which this binary prints.

---

## 7. Correction: `konica206-native` does **not** bypass the vendor binary

This is the most important finding for anyone planning a musl port, because it
is the obvious candidate and it does not help.

`src/konica_filter.c` executes the vendor filter as a subprocess:

```c
 29:  env_or(c->vendor_filter, ..., "KONICA_VENDOR_FILTER",
 30:         "/usr/local/lib/konica/KonicaMinolta/245igdi/Filters/245igdirf");
 32:  env_or(c->libdir, ..., "KONICA_LIBDIR", "/usr/local/lib/konica/lib");
 99:  snprintf(envbuf[5], ..., "LD_LIBRARY_PATH=%s", cfg->libdir);
126:  execve(prog, argv, envp);
```

and `src/konica_usb.c` is pure transport — `libusb_bulk_transfer` in 8192-byte
chunks, with no protocol code at all:

```
173: rc = libusb_bulk_transfer(u->h, u->ep_out, ... );
221: rc = libusb_bulk_transfer(u->h, u->ep_in,  ... );
```

So the raster→TBCP conversion is **entirely vendor**, on both platforms. A musl
build of `konica206-native` would link cleanly, serve IPP and map
`sides`/`media` correctly, and then exec the exact glibc-binary + glibc-libcups
pair proven above to fail. Strictly worse than the working Ubuntu setup.

---

## 8. Conclusions

1. **Only two musl shims are needed** — `gcompat`, plus a `libdl.so.2` stub. No
   libcups shim, no vendored glibc.
2. **Loading is not the problem.** Every dependency resolves; CUPS invokes the
   filter correctly; the USB backend opens the printer.
3. **`245igdirf` cannot run on musl.** gcompat cannot provide a real libc for the
   raster/PPD/colour work the filter depends on. Proven with `LD_BIND_NOW=1`.
4. **`konica206-native` is not an escape route** — it execs the same binary.
5. **Alpine's gs 10.05.1 needs the `pdftoraster` workaround**, and the
   full-bleed PPD should be used. These are real Alpine findings regardless.

### Options

| | Approach | Assessment |
|---|---|---|
| **A** | Stay on Ubuntu/glibc | Costs nothing, works today. **Recommended.** |
| **B** | Reimplement raster→TBCP in C | The only route to musl. Genuine RE on a closed ~1.3 MB codebase (not just the 530 KB binary) — halftoning, `ImxCmmProfile*` colour management, PJL/TBCP framing, duplex `PAGESTATUS` sequences. Scope, ladder and prerequisites: **[alpine-musl-reimplementation.md](alpine-musl-reimplementation.md)** |
| **C** | Port `konica206-native` | Ruled out by §7. |

For **B**, the binary is `not stripped` with a friendly symbol table
(`ImxErrDiffAMT*`, `PCM_fHLS2sRGB`, `PCM_fJab2XYZ`, `PalFileRead/Write`), which
makes it tractable. A tool like `morluto/rea` driving Ghidra is useful for
*understanding* it — it will not port anything, but it is the right instrument for
reading it. See **[alpine-musl-reimplementation.md](alpine-musl-reimplementation.md)**
for the full scope, the staged ladder, and what to build first.

---

## 9. Housekeeping

- Chroot, `/dev` and `/proc` mounts, port 6631 — all removed after testing.
- Host verified untouched: cupsd alive, **6 queues all accepting**, default
  `konica206-native`, USB interface 1 free, vendor driver files intact.
- No paper was consumed by any successful job — both print attempts produced
  `Sent 0 bytes`.
- Reproducible installer kept at `vinod2807/konica-retrofit`, as
  `install-konica-245igdi-alpine.sh`, carrying the negative result in its header.

## Appendix — reusable Alpine install recipe

```sh
apk add gcompat cups cups-client cups-filters cups-dev ghostscript poppler-utils build-base

# driver from the deb (no dpkg on Alpine; .deb is an ar archive)
unzip -q BH225iGDILinux_201MU.zip 'BH225iGDILinux_201MU/For_x86_64/*' -d /tmp/k
cd /tmp/k && ar x *.deb && tar -xf data.tar.* -C /

# working layout: everything under /usr/local/lib/konica so the 206 PPD works verbatim
mkdir -p /usr/local/lib/konica/KonicaMinolta
cp -a /usr/lib/cups/filter/KonicaMinolta/245igdi/. /usr/local/lib/konica/KonicaMinolta/
ln -sf /usr/local/lib/konica/KonicaMinolta/245igdi/Filters/mtorf.ocm \
      /usr/local/lib/konica/KonicaMinolta/245igdi/Filters/245igdirf.ocm

# gcompat has no libdl.so.2; 245igdirf needs it but only imports dl* symbols
printf 'int _stub;\n' > /tmp/dl.c && gcc -shared -fPIC -o /lib/libdl.so.2 /tmp/dl.c

# gs 10.05.1 cannot rasterise these PPDs - use poppler instead
sed -i '/gstoraster/ { /application\/vnd\.cups-raster/ { s|^|#DISABLED | } }' \
    /usr/share/cups/mime/cupsfilters-ghostscript.convs
```

PPD requirements that also apply on the host:

- use a **206 retrofit** PPD, never the shipped 205/225/245i ones;
- **zero `*cupsICCProfile` lines** (Ghostscript is built without ICC support);
- keep `*cupsBackSide: Rotated` or duplex prints at the wrong edge;
- PPD comments must start with `*` (`*%`) — a leading `#` makes CUPS reject the
  file with `Missing asterisk in column 1`.