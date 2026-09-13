# Upstream GNOME submission

This document contains a ready-to-use bug report and merge-request description for GNOME Control Center.

## Current upstream status

The bug is still present in the current `main` branch of `GNOME/gnome-control-center`: `panels/printers/pp-printer-entry.c` accepts `ink`, `toner`, `inkCartridge` and `tonerCartridge`, but not the hyphenated `ink-cartridge` / `toner-cartridge` values.

The repository `main` branch is currently in the GNOME 51 development cycle, so this is not only a GNOME 48 issue.

Use this patch for current upstream `main`:

- `patches/gnome-control-center-main-hyphenated-marker-types.patch`

The Debian 13 / GNOME 48.4 patch remains available separately for users of that release.

---

## Suggested GitLab issue title

**Printers: toner levels hidden when CUPS reports `toner-cartridge` marker type**

## Suggested GitLab issue body

GNOME Settings can hide otherwise valid toner levels for printers whose CUPS/IPP marker type is reported as `toner-cartridge`.

### Environment

- Debian 13 (Trixie)
- GNOME Control Center 48.4 (`1:48.4-1~deb13u1`)
- HP Color LaserJet Pro MFP M479fdn
- Driverless IPP Everywhere queue

I also checked the current upstream `main` branch and the same marker-type filter is still present there.

### Actual behavior

The printer works normally and CUPS reports valid marker data, for example:

```text
marker-levels=60,10,30,60
marker-types=toner-cartridge,toner-cartridge,toner-cartridge,toner-cartridge
```

`system-config-printer` shows all four toner levels correctly, but GNOME Settings shows no usable toner level bar for the same queue.

### Root cause

`panels/printers/pp-printer-entry.c` filters marker entries by type. It currently accepts:

```text
ink
toner
inkCartridge
tonerCartridge
```

but not:

```text
ink-cartridge
toner-cartridge
```

Because the HP M479fdn reports `toner-cartridge`, GNOME discards valid marker data.

### Proposed fix

Extend the existing condition to also accept the hyphenated forms:

```c
if (g_strcmp0 (marker_typesv[i], "ink") == 0 || g_strcmp0 (marker_typesv[i], "toner") == 0
    || g_strcmp0 (marker_typesv[i], "inkCartridge") == 0
    || g_strcmp0 (marker_typesv[i], "tonerCartridge") == 0
    || g_strcmp0 (marker_typesv[i], "ink-cartridge") == 0
    || g_strcmp0 (marker_typesv[i], "toner-cartridge") == 0) {
```

### Verification

I rebuilt GNOME Control Center 48.4 with only this change and installed the resulting Debian packages.

Before the patch, GNOME Settings did not render the M479 toner levels.

After the patch, the existing IPP Everywhere queue immediately displayed the toner bar, matching the values already reported by CUPS:

```text
Black    60%
Cyan     10%
Magenta  30%
Yellow   60%
```

No HPLIP queue or printer-side change was required.

Full reproduction notes, before/after visuals and tested Debian build instructions:

https://github.com/josefseger/gnome-printer-ink-level-fix

---

## Suggested commit message

```text
printers: Accept hyphenated ink and toner cartridge marker types

Some IPP/CUPS printers report their marker type as `ink-cartridge` or
`toner-cartridge`, while the printer panel currently only recognizes the
legacy `inkCartridge` and `tonerCartridge` spellings in addition to `ink`
and `toner`.

Accept the hyphenated forms too so valid supply levels are not discarded.

Tested with an HP Color LaserJet Pro MFP M479fdn using IPP Everywhere,
where this makes the toner levels visible and matches the values already
reported by CUPS.
```

## Suggested merge request title

**printers: Accept hyphenated ink and toner cartridge marker types**

## Suggested merge request description

This fixes missing toner levels in the Printers panel for IPP/CUPS devices which report `marker-types=toner-cartridge`.

The change is intentionally small: it extends the existing marker-type filter with the hyphenated `ink-cartridge` and `toner-cartridge` spellings.

Tested on an HP Color LaserJet Pro MFP M479fdn with IPP Everywhere. Before the change, CUPS contained valid levels but GNOME Settings did not render them. After the change, the toner bar is shown and matches the CUPS values.

Reproduction details and before/after evidence:

https://github.com/josefseger/gnome-printer-ink-level-fix

Please allow maintainer edits on the merge request, per GNOME Settings contribution guidelines.
