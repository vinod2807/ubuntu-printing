# Ubuntu printing — KONICA_MINOLTA_206 (Konica Minolta 206 GDI over USB)

How the `KONICA_MINOLTA_206` classic CUPS queue was created on Ubuntu 26.04
(`resolute`), and every fix needed to make it print — including duplex.

## Environment

| Item | Value |
|---|---|
| OS | Ubuntu 26.04 `resolute`, Ghostscript **10.06.0**, CUPS **2.4.16**, cups-filters **2.0.1** |
| Printer | Konica Minolta 206 (GDI, USB): `usb://KONICA%20MINOLTA/206?serial=A8A6041029423&interface=1` |
| Driver | `konica-minolta-245igdi-cups` **2.01**, installed from `BH225iGDILinux_201MU.zip` (`BH225iGDILinux_201MU/For_x86_64/konica-minolta-245igdi-cups_2.01_amd64.deb`) via `dpkg -i` |
| Vendor filter (patched, must be used) | `/usr/local/lib/konica/KonicaMinolta/245igdi/Filters/245igdirf` (+ `mtorf.ocm` via the `245igdirf.ocm` symlink) — **not** the stock `/usr/lib/cups/filter/…` copy |
| Queue PPD | `/etc/cups/ppd/KONICA_MINOLTA_206.ppd` — MakeModel `KONICA MINOLTA 206 (ICC-free, full-bleed retrofit)` |

## Architecture (why two queues exist)

```
GUI ──▶ cupsd (:631) ──▶ legacy-printer-app / konica206uri (PAPPL, :8000, primary queue)
                        classic CUPS queue KONICA_MINOLTA_206 (this doc, CUPS-2.x fallback)
                              │ pdftopdf → pdftoraster (poppler) → 245igdirf → usb backend
```

`konica206uri` (PAPPL) is the system default and primary queue; duplex there rides on
the IPP `sides` attribute. `KONICA_MINOLTA_206` is the classic native CUPS queue
documented here. Both contend for the same USB device — only print on one at a time.

## Queue creation

```sh
lpadmin -p KONICA_MINOLTA_206 \
  -v "usb://KONICA%20MINOLTA/206?serial=A8A6041029423&interface=1" \
  -P /var/lib/legacy-printer-app/ppd/KonicaMinolta-206-fullbleed.ppd \
  -D "KONICA MINOLTA 206" -L "USB" -E
```

Base PPD is the full-bleed retrofit PPD, then edited as described below
(`*DefaultPageSize: A4`, `*DefaultDuplexer: true`, ICC lines removed,
`*cupsFilter` pointed at the patched filter).

## The 4 stacked bugs and their fixes

All four had to be fixed — each one alone still produced no output.

### 1. Ghostscript 10.06 has no ICC support

`gs -sOutputICCProfile=…` fails with `Unrecoverable error: undefined in
.putdeviceprops`. The stock PPD references ICC profiles, so every job died in
`gstoraster`.

**Fix:** delete all `*cupsICCProfile` lines from the queue PPD
(verify with `grep -c cupsICCProfile …` → `0`).

### 2. `gstoraster` emits 0 bytes with this PPD on gs 10.06

`gs -sDEVICE=cups -c '<</.HWMargins…>>setpagedevice'` produced a 0-byte raster,
while poppler's `pdftoraster` renders fine. So the `gstoraster` MIME mappings were
disabled and poppler (cost 100) takes over:

`/usr/share/cups/mime/cupsfilters-ghostscript.convs` (backup: same path + `.orig`):

```
# DISABLED-gstoraster-broken-on-gs-10.06 application/vnd.cups-pdf	application/vnd.cups-raster	99	gstoraster
# DISABLED-gstoraster-broken-on-gs-10.06 application/vnd.cups-postscript	application/vnd.cups-raster	175	gstoraster
```

> **Upgrade protection (applied):** this file is registered with
> `sudo dpkg-divert --add --rename
> /usr/share/cups/mime/cupsfilters-ghostscript.convs`, so `cups-filters`
> upgrades (including release upgrades) install upstream to
> `….convs.distrib` and leave this edited copy in place.
> Verify with `dpkg-divert --list | grep ghostscript`.
> Revert with `sudo dpkg-divert --remove --rename <path>`.

### 3. Stale colord profile broke color management

A stale `KONICA_MINOLTA_206___` profile made the ColorManager resolve an empty ICC
profile. Removed via D-Bus `org.freedesktop.ColorManager.Device.RemoveProfile`
(system bus). The device now reports no bogus profile.

### 4. AppArmor denied execution of the vendor filter (exit 113)

`audit: apparmor="DENIED" operation="exec" … 245igdirf`. Added to
`/etc/apparmor.d/local/usr.sbin.cupsd`:

```
  /usr/local/lib/konica/** Cxr -> third_party,
```

then `sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.cupsd`.

## Final PPD state (`/etc/cups/ppd/KONICA_MINOLTA_206.ppd`)

```sh
sudo grep -n -E "^\*cupsFilter|^\*DefaultPageSize|^\*DefaultDuplex|^\*DefaultDuplexer|^\*cupsBackSide|^\*DefaultResolution" \
  /etc/cups/ppd/KONICA_MINOLTA_206.ppd
# 39:*cupsFilter: "application/vnd.cups-raster 0 /usr/local/lib/konica/KonicaMinolta/245igdi/Filters/245igdirf"
# 50:*cupsBackSide: Rotated
# 136:*DefaultDuplexer: true
# 143:*DefaultDuplex: None            ← single-sided default; duplex is selected per job
# 158:*DefaultPageSize: A4
# 459:*DefaultResolution: 600x600dpi
```

Other relevant facts:

- `*ImageableArea A4/A4: "0 0 595 842"` — this PPD is **full-bleed** (normal
  margins live on the `konica206uri` real-margin driver, not here).
- `*Duplex DuplexNoTumble/Long Edge: "<</Duplex true/Tumble false>>setpagedevice"`,
  `*Duplex DuplexTumble/Short Edge: "<</Duplex true/Tumble true>>setpagedevice"`.
- `*cupsBackSide: Rotated` (line 50) is **required for correct duplex orientation**
  — deleting it makes the back side print at the wrong edge. PPD comment lines must
  start with `*` (use `*%`); a `#` comment makes CUPS reject the PPD with
  `Missing asterisk in column 1`.
- `.Borderless` entries in `lpoptions -l` (e.g. `A4.Borderless`) are synthesized by
  `libppd`, not present in the PPD — ignore them.

## Duplex

Duplex is driven by the IPP `sides` option, mapped to `Duplex=DuplexNoTumble`
(`two-sided-long-edge`) in the filter environment. Verified by capturing the vendor
filter's output with a pass-through `tee` wrapper (removed afterwards):

| Test | Output | PJL |
|---|---|---|
| 1 page, `sides=one-sided` | 58,301 B, `DUPLEX=OFF` | baseline |
| 1 page, `sides=two-sided-long-edge` | 58,328 B | `DUPLEX=ON` + `BINDING=SHORTEDGE` |
| **2 pages, `sides=two-sided-long-edge`** | **116,600 B (≈2×)** | **`PAGESTATUS=START` ×2, `DUPLEX=ON` ×2, `@PJL EOJ`** |

CUPS log (`/var/log/cups/error_log`): `Sent 116600 bytes…`, `245igdirf exited with
no errors`, `Job completed` (jobs 269, 271 — physically confirmed duplex,
long-edge, correct).

Note: a **1-page** document printed with `sides=two-sided-long-edge` yields only the
front side — that is correct behaviour, not a bug (this exact misunderstanding once
looked like a duplex failure: `default-testpage.pdf` is 1 page).

```sh
# Simplex / duplex usage (default is single-sided):
lp -d KONICA_MINOLTA_206 -o media=A4 -o sides=one-sided file.pdf
lp -d KONICA_MINOLTA_206 -o media=A4 -o sides=two-sided-long-edge file.pdf
lp -d KONICA_MINOLTA_206 -o media=A4 -o sides=two-sided-short-edge file.pdf
```

## Backups (all kept on the machine)

| Path | Purpose |
|---|---|
| `/etc/cups/ppd/KONICA_MINOLTA_206.ppd.orig` | pre-edit copy |
| `/etc/cups/ppd/KONICA_MINOLTA_206.ppd.noicc` | ICC lines removed |
| `/etc/cups/ppd/KONICA_MINOLTA_206.ppd.O`, `.bak-20260805-084935` | older copies |
| `/usr/share/cups/mime/cupsfilters-ghostscript.convs.orig` | stock convs file |
| `/usr/local/lib/konica/KonicaMinolta/245igdi/Filters/245igdirf.orig{,2}` | pre-patch filter |

## Recreate from scratch (checklist)

1. `sudo dpkg -i konica-minolta-245igdi-cups_2.01_amd64.deb`
2. `lpadmin -p KONICA_MINOLTA_206 -v "<DeviceURI above>" -P <full-bleed PPD> -E`
3. Edit PPD: `DefaultPageSize A4`, `DefaultDuplexer true`, `DefaultDuplex None`,
   keep `*cupsBackSide: Rotated`, delete `*cupsICCProfile` lines, set `*cupsFilter`
   to the patched `245igdirf`.
4. Comment out the two `gstoraster` lines in `cupsfilters-ghostscript.convs`
   (keep the `.orig` backup).
5. Remove any stale `KONICA_MINOLTA_206___` colord profile.
6. Add the AppArmor `Cxr -> third_party` rule, reload the profile.
7. `sudo systemctl restart cups`, then print the simplex test, then the 2-page
   duplex test, and confirm `Sent … bytes` + `Job completed` in
   `/var/log/cups/error_log`.

## Appendix — `konica206uri` (PAPPL) notes that also apply here

- Server-side PAPPL driver IDs have **no** `-user-added-en` suffix; the real one is
  `konica-minolta--206--real-margin-retrofit-en` (`legacy-printer-app add` is broken
  on this box — empty device ID — so the queue is maintained via the state file).
- `LC_PAPER`/`LC_ALL=en_IN.UTF-8` in the `legacy-printer-app` override keeps A4;
  `media-col-default`/`media-col-ready*` in
  `/var/lib/legacy-printer-app/legacy-printer-app.state` must all be
  `iso_a4_210x297mm` (they once leaked `na_letter_8.5x11in`).
- Never send CUPS banner/test-page PDFs through the PAPPL queue (they wedge it);
  use the classic queue for those.
