# konica206uri — PAPPL primary queue for the Konica Minolta 206

How the `konica206uri` print queue (PAPPL / `legacy-printer-app`, port 8000) was
built and fixed on Ubuntu 26.04. This is the **system default** and primary queue;
the classic CUPS queue is documented separately in `README.md`.

## Architecture

```
GUI ──▶ cupsd (:631) ──▶ legacy-printer-app (:8000, PAPPL)
                              │ pdftopdf → ghostscript → 245igdirf → chunked usb backend
                              ▼
                    usb://KONICA%20MINOLTA/206?serial=A8A6041029423&interface=1
```

- cupsd exposes it as `ipp://localhost:8000/ipp/print/konica206uri`
  (MakeModel `KONICA Printer, driverless, 2.1.1`, `<DefaultPrinter>` in
  `/etc/cups/printers.conf`).
- Filter chain per job: `pdftopdf` → `ghostscript` (`-sDEVICE=cups`) →
  patched `/usr/local/lib/konica/KonicaMinolta/245igdi/Filters/245igdirf`
  (with `mtorf.ocm`) → custom chunked USB backend
  `/usr/local/libexec/konica-backend/usb`.
- Duplex rides on the IPP `sides` attribute, independent of the PPD.

## Driver — real margins, not full-bleed

Server-side driver ID (note: **no** `-user-added-en` suffix — that suffix only
appears in `legacy-printer-app drivers` standalone output; the server IDs come
from the `PPD Collections` log):

```
konica-minolta--206--real-margin-retrofit-en
```

PPD: `/var/lib/legacy-printer-app/ppd/KonicaMinolta-206-real-margins.ppd`
(verified byte-identical to its `.bak` pristine copy). Key lines:

```sh
39:*cupsFilter: "application/vnd.cups-raster 0 /usr/local/lib/konica/KonicaMinolta/245igdi/Filters/245igdirf"
43:*cupsVersion: "1.3"
50:*cupsBackSide: Rotated
143:*DefaultDuplex: None            # single-sided default; duplex selected per job via IPP sides
136:*DefaultDuplexer: false         # duplexer negotiated over IPP, not the PPD
158:*DefaultPageSize: A4
216:*ImageableArea A4/A4: "6 12 589 830"   # real (non-borderless) margins
```

## Queue creation (state-file editing, not `add`)

`legacy-printer-app add` is broken on this box for every driver
(`Driver 'X' cannot be used with this printer` — empty device ID), so the printer
is maintained by editing `/var/lib/legacy-printer-app/legacy-printer-app.state`
(stop service → edit → start), plus an auto-create script:

- `/usr/local/bin/ensure-konica206uri.sh`
  (`URI='cups:usb://…'`, `DRIVER='konica-minolta--206--real-margin-retrofit-en'`),
  run as `ExecStartPost` from the override below.
- `/etc/systemd/system/legacy-printer-app.service.d/override.conf`:

```ini
[Service]
Environment=PPD_PATHS=/var/lib/legacy-printer-app/ppd:/usr/share/cups/model:/usr/lib/cups/driver
Environment=LC_PAPER=en_IN.UTF-8
Environment=LC_ALL=en_IN.UTF-8
ExecStart=
ExecStart=legacy-printer-app server -o log-level=debug -o backend-directory=/usr/local/libexec/konica-backend
ExecStartPost=/usr/local/bin/ensure-konica206uri.sh
```

`LC_PAPER`/`LC_ALL` force A4 instead of Letter.

## State fixes

In `/var/lib/legacy-printer-app/legacy-printer-app.state`:

- `driver="konica-minolta--206--real-margin-retrofit-en"` (was full-bleed).
- `media-col-default` and all `media-col-ready*` = `iso_a4_210x297mm`
  (8 A4 entries, 0 `na_letter` — the ready list once leaked Letter).
- `DefaultPrinterID 6` matches `<Printer … id="6" name="konica206uri" …>`
  (a `1` vs `6` mismatch once caused `CUPS-Get-Default not-found` /
  `No default printer available`).

## `*cupsBackSide: Rotated` — do not remove

Line 50 was once deleted as a borderless experiment (a `#` comment breaks PPD
parsing — `Missing asterisk in column 1` — so it had been deleted outright).
Result: duplex printed at the **wrong edge** (short edge). Restoring the line
fixed long-edge duplex; the PPD is now byte-identical to the pristine `.bak`.

## Duplex verification

Job log proof (`journalctl -u legacy-printer-app`) for a 2-page
`sides=two-sided-long-edge` job on A4:

```
Duplex=DuplexNoTumble, Duplexer=true, MediaType=Plain
media-col.size-name='iso_a4_210x297mm'
gs … -dDuplex … -scupsPageSizeName=A4 -scupsBackSideOrientation=Rotated …
245igdirf argv: back-side-orientation=Rotated … Duplex=DuplexNoTumble …
Printing page 1, 1 copies / Printing page 2, 1 copies
job-completed-successfully
```

Notes:

- Ghostscript logs one extra trailing `Processing page N` line on a 2-page input
  (flush page) — normal, not an input mismatch.
- A 1-page duplex job prints only the front — correct behaviour, not a bug.
- `.Borderless` names in `lpoptions -l` are synthesized by `libppd`, not in the
  PPD — ignore them; the default is `*A4` and real jobs use normal margins.

```sh
lp -d konica206uri -o sides=two-sided-long-edge -o media=A4 file.pdf
lp -d konica206uri -o page-ranges=1-2 -o sides=two-sided-long-edge -o media=A4 file.pdf
```

## Document sizing gotchas (learned from a real 17-page DOCX)

- The DOCX declared `<w:pgSz w:w="11906" w:h="17338"/>` (305.82 mm) instead of
  A4's `w:h="16838"` (297 mm) — 8.82 mm too tall, width exactly A4. Only its last
  section was true A4.
- With `print-scaling=auto`, `pdftopdf` chose *center + crop*, cutting ~8.8 mm off
  the page bottom. To scale instead: `-o print-scaling=shrink`.
- The user found **OnlyOffice renders/prints this document correctly** while the
  LibreOffice export behaved oddly — prefer OnlyOffice (`onlyoffice-desktopeditors`,
  also installed here) for authoring/converting such files.

## Warnings

- Never send CUPS banner/test-page PDFs (`application/vnd.cups-pdf-banner`)
  through this queue — they wedge the printer. Use the classic queue for those.
- Only print on one queue at a time — both drive the same USB device.
- Backups: `KonicaMinolta-206-real-margins.ppd.bak` (pristine PPD),
  state-file `.bak-*` snapshots, `override.conf.bak`.

## Recreate from scratch (checklist)

1. Install `konica-minolta-245igdi-cups` 2.01 and place the retrofit PPDs in
   `/var/lib/legacy-printer-app/ppd/`.
2. Write the systemd override (PPD paths, `LC_*`, backend dir,
   `ExecStartPost` ensure script).
3. Stop the service, edit the state file (driver, A4 media everywhere,
   matching `DefaultPrinterID`), start the service.
4. `sudo legacy-printer-app printers` must list `konica206uri`; `lpstat -d`
   should show it as system default.
5. Print a 2-page `sides=two-sided-long-edge` job; confirm
   `back-side-orientation=Rotated`, `Duplex=DuplexNoTumble`, both pages, and
   `job-completed-successfully` in the journal.
