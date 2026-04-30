# Fusion 360 → Charmhigh CHM-T36VA Converter

A free, browser-based tool that converts Fusion 360 / EAGLE pick-and-place files into `.dpv` work files for the Charmhigh CHM-T36VA pick-and-place machine.

**Live:** [geocentricllc.github.io/chm-t36va](https://geocentricllc.github.io/chm-t36va/)

Runs entirely in your browser. No uploads, no install, no telemetry. Save the page to disk and use it offline.

---

## Table of contents

- [What this is for](#what-this-is-for)
- [Quick start](#quick-start)
- [Supported input files](#supported-input-files)
- [The 5-step workflow](#the-5-step-workflow)
  - [Step 1 — Load file](#step-1--load-file)
  - [Step 2 — Feeder mapping](#step-2--feeder-mapping)
  - [Step 3 — Assign feeders](#step-3--assign-feeders)
  - [Step 4 — Board / calibration](#step-4--board--calibration)
  - [Step 5 — Export DPV](#step-5--export-dpv)
- [CHM-T36VA reference](#chm-t36va-reference)
  - [Station ID ranges](#station-id-ranges)
  - [Nozzle selection](#nozzle-selection)
  - [Tape pitch reference](#tape-pitch-reference)
- [Saving and reusing feeder configs](#saving-and-reusing-feeder-configs)
- [Common questions](#common-questions)
- [Troubleshooting](#troubleshooting)
- [Privacy](#privacy)
- [License](#license)

---

## What this is for

The Charmhigh CHM-T36VA expects pick-and-place data in its proprietary `.dpv` format — a plain-text file with sections for stations (feeders), placements, fiducials, and calibration. PCB CAD tools like Fusion 360 and EAGLE export pick-and-place data in a different format (`.mnt` from `mountsmd.ulp`, or generic CSV from various CAM processors).

This converter bridges the gap: drop your CAD output in, pick which physical feeder reels hold which parts, and download a `.dpv` ready to load on the machine.

It's designed for hobbyists and small-shop assemblers running the CHM-T36VA — the same audience the machine itself targets. Everything happens locally in your browser, so it works offline and your design files never leave your computer.

---

## Quick start

1. Open the [live site](https://geocentricllc.github.io/chm-t36va/), or download `index.html` and open it in any modern browser.
2. **Step 1:** drop your `.mnt` file (or CSV) on the dropzone. Click **Load demo data** to try the workflow without your own file.
3. **Step 2 (optional):** for each unique part type, pick a Feeder ID and Nozzle. Save the mapping to JSON to reuse on future boards.
4. **Step 3:** review per-placement settings. Tweak feeder assignments, nozzle, tape pitch, etc. Skip individual placements you don't want.
5. **Step 4:** set the board X/Y offset if needed. Pick fiducial reference RefDes from the dropdowns (auto-detects `FID*`-named refs).
6. **Step 5:** review the DPV preview, click Download. Copy the `.dpv` to a USB stick and load on the CHM-T36VA. Calibrate fiducials at the machine before running.

---

## Supported input files

### Fusion 360 / EAGLE `.mnt` and `.mnb`

This is the primary input format. It comes from EAGLE's classic `mountsmd.ulp` script, which both EAGLE and Fusion 360's Electronics workspace ship with.

**Generate it via:**
- **Fusion 360 (Electronics):** *Utilities → Automate → Run ULP → mountsmd.ulp*
- **EAGLE:** *File → Run ULP → mountsmd.ulp* from the board editor

The script writes one file per side: `.mnt` = top, `.mnb` = bottom. Column layout is fixed:

```
RefDes  LocationX  LocationY  Rotation  Value  Package
```

Whitespace-delimited, no header row. The converter recognizes the extension and parses without column-mapping.

> ⚠ **Don't confuse with Manufacturing → Outputs → Pick and Place.** Fusion 360 has a separate Manufacturing-workspace pick-and-place generator, but it omits the Package column — the file is incomplete for this converter's nozzle/tape-pitch defaults. Always use the `mountsmd.ulp` path.

### Generic CSV / TXT pick-and-place

Any comma/tab/semicolon-delimited file with a header row works. Column detection scans for these names (case-insensitive):

| Field | Recognized header names | Required |
|---|---|---|
| Reference | Designator, RefDes, Reference, Name, Part, Component | Yes |
| Value | Value, Comment, Val, Description | No |
| Package | Package, Footprint, Pattern, Pkg | No |
| X coord | Mid X, X, Center X, Loc X, Pos X, Ref X | Yes |
| Y coord | Mid Y, Y, Center Y, Loc Y, Pos Y, Ref Y | Yes |
| Rotation | Rotation, Rot, Angle, Orientation, Theta | Yes |
| Side | Side, Layer, TB, Mirror | No |
| DNP | DNP, DNI, Do Not Place/Populate/Install, Populate | No |

Files from KiCad, Altium, EAGLE's CAM Processor, OrCAD, and others should auto-detect cleanly. Override dropdowns on Step 1 let you correct the auto-detection if the heuristic gets units, side, or DNP wrong.

### Coordinate units

Auto-detected from the magnitude of X/Y values:
- Max coordinate < 13 → **inches**
- Max coordinate ≤ 300 → **millimeters**
- Max coordinate > 300 → **mils** (1/1000 inch)

The override dropdown lets you correct this for unusual cases.

---

## The 5-step workflow

### Step 1 — Load file

Drop or click to browse. After parsing, you'll see file stats and a one-line summary of which columns mapped to what. For most files, just hit **Continue**.

Touch the override dropdowns when:
- The unit auto-detection picked wrong (a tiny mil-units board misread as mm)
- You want to include both top and bottom parts — change Side filter to "All rows"
- You want to include parts marked DNP — set DNP filter to "Include them"

### Step 2 — Feeder mapping

A table with one row per **unique Value+Package** pair. For each, you can optionally set:

- **Feeder ID** (1–99): which station on your machine holds that reel
- **Nozzle** (N1, N2, or `—` for auto): which nozzle picks this part type

Inputs are blank by default — leave them blank to skip a row. Unmapped rows get auto-assigned defaults on Step 3.

**Why this step exists:** most physical reels stay in the same station across multiple boards. Once you've loaded a reel of `10k 0402` at station 5, every future board with that part should use station 5. Save the mapping once, reuse it on every subsequent board.

**Save Config** writes a JSON file with only the rows you've filled in. **Load Config** applies a saved JSON to the current board's mapping table. Case-insensitive matching means a config saved from board A's `10K` will match board B's `10k`.

### Step 3 — Assign feeders

The main editing screen. **One row per placement** on your board. Identical part types share most settings — editing them on one row updates every row of that type, with sibling inputs visually synced.

Column behavior:

| Column | Per-placement | Per-part-type (shared) |
|---|---|---|
| RefDes, Value, Package | display only | — |
| X / Y / Rotation | ✓ editable | — |
| **Skip** | ✓ editable | — |
| Feeder | — | ✓ editable |
| Nozzle (N1/N2 toggle) | — | ✓ editable |
| Tape pitch | — | ✓ editable |
| Height | — | ✓ editable |
| Speed | — | ✓ editable |
| Vision | — | ✓ editable |

The **Feeder column has color-coded left edges** — same color = same feeder ID. Quick visual sanity check that you've got distinct reels mapped where you intended.

**Sort:** click RefDes, Value, or Package column headers (X/Y/Rotation aren't sortable). Click again to reverse, third time to return to insertion order. RefDes uses natural sort (R1 < R2 < R10).

**Skip behavior:** ticking Skip on a row immediately moves it to the end of the table. Skipped rows always partition to the tail regardless of sort. Skip is per-placement, so ticking it on R1 doesn't affect R2.

**Pagination:** the table shows 20 rows per page. Sorting always jumps to page 1.

**Defaults:** feeder/nozzle come from your Step 2 mapping if set; otherwise auto-assigned. The auto-assigner skips feeder IDs claimed by mappings, so you don't get accidental collisions. Tape pitch, height, and nozzle (when not mapped) are guessed from the package name.

### Step 4 — Board / calibration

Set the **X/Y offset** if your file's origin isn't the PCB's lower-left corner. Most boards from Fusion 360 work with offset `(0, 0)`; you'll need to adjust when:
- Your CAD design uses board-center origin
- Negative X or Y values appear in the placements
- You're panelizing into a larger panel coordinate system

**Quick test:** after Step 3 the table displays raw placement coords. If smallest X/Y values are near zero — good. If they're at, say, X = −18 to +18 — the file uses centered origin and you need ~+18mm offset.

**Fiducial dropdowns:** each of UL, LR, LL has a dropdown listing every RefDes plus `(none)` and `(custom)`. Selecting a RefDes auto-fills X/Y from that placement (with offset applied). `(custom)` enables manual entry. `(none)` zeroes the slot.

The converter auto-detects refdes matching `FID1`/`FID2`/`FID3` (also `MARK*` and `F1`/`F2`/`F3`) and pre-selects them as UL/LR/LL on first arrival, with an `auto-detected` badge until you change the selection.

> Calibrating on the machine is still standard practice. Even with fiducial coords set in the DPV, you'll typically jog the camera to each physical mark at the machine, which writes the actual position. The DPV's fiducial values are a starting hint, not the final word.

### Step 5 — Export DPV

The DPV preview shows exactly what will be written. Filename is editable. Download saves a plain-text `.dpv` file ready to copy to a USB stick.

---

## CHM-T36VA reference

### Station ID ranges

| Range | Type | Notes |
|---|---|---|
| 1–29 | Reel feeders | Standard SMD tape reels — most parts go here |
| 30–59 | — | Not used / reserved |
| 60–74 | Front IC trays | Tubed ICs and small trays at the front |
| 75–79 | — | Not used / reserved |
| 80–99 | IC trays | Matrix-tray ICs (waffle pack) |

The converter accepts any value 1–99. The duplicate-feeder warning at Step 3 → Step 4 catches collisions before you generate the DPV.

### Nozzle selection

The CHM-T36VA has two nozzles mounted side by side:
- **N1 (left)** — typically a smaller nozzle for 0201, 0402, 0603, SOT-23. Default for passives.
- **N2 (right)** — typically a larger nozzle for SOIC, QFP, QFN, BGA, electrolytics. Default for ICs.

The converter auto-picks based on package name. Override per part type on Step 2 or Step 3.

### Tape pitch reference

Tape pitch is the distance the reel advances between picks. It must match the physical spacing of parts on your tape — too short re-picks the same pocket; too long skips parts.

| Pitch (mm) | Typical packages | Reel widths |
|---|---|---|
| 2 | 0201 | 8 mm |
| 4 | 0402, 0603, 0805, 1206, SOT-23, SC-70/75 | 8 mm |
| 8 | 1210, SOT-223, SOIC-8 | 8 mm or 12 mm |
| 12 | SOIC, TSSOP, small QFN | 12 mm |
| 16 | QFP, larger QFN, BGA | 16 mm or 24 mm |
| 24 | Large ICs, connectors, modules | 24 mm or 32 mm |

The DPV format calls this field `FeedRates`, but it's a single value, not a rate.

---

## Saving and reusing feeder configs

The whole point of Step 2 is making your work reusable. The Save Config / Load Config buttons handle the round trip.

### What's stored

Only **feeder ID and nozzle**, keyed by Value+Package. Tape pitch, height, speed, and vision are not saved — they're auto-guessed from package names on every board, which is correct since they depend on the part footprint, not the project. Per-placement coordinates are also not stored — those are board-specific.

### Suggested workflow

1. On your first board, fill out Step 2 carefully. Pick deliberate feeder numbers based on which physical reels you've loaded at which stations.
2. Save the config.
3. On every subsequent board, load the config first. Any matching parts (Value+Package, case-insensitive) get pre-mapped automatically. New parts get blank rows you fill in.
4. Save the config again whenever you load a new reel onto the machine.

### Old config compatibility

Older versions of this tool saved richer configs (tape pitch, height, etc.). Those files still load fine — extra fields are silently ignored. Feeder ID and nozzle still apply correctly.

---

## Common questions

**Why is it telling me the package column is missing?**
You probably exported via Fusion 360's *Manufacturing → Outputs → Pick and Place*. That generator omits the Package column. Use *Utilities → Automate → Run ULP → mountsmd.ulp* instead — same machine, same project, but the right export path.

**Why are some part rotations off by 90° or 180°?**
Almost always a reel-vs-footprint orientation mismatch in your CAD library. Two ways to fix:
- **Best (permanent):** fix the footprint in your CAD library so future boards come through correctly.
- **Quick (per-board):** sort by Value or Package, edit the Rotation column on each affected row to add 90 / 180 / -90 as needed.

**The placements end up off the board on the machine.**
Usually a coordinate-origin issue. Check Step 4's X/Y offset. If your Fusion file uses board-center origin, the offset should be roughly *(half board width, half board height)*.

**Can I use this with KiCad or Altium?**
Yes — anything that exports a CSV with the standard column names listed above. Most CAD tools have a pick-and-place export option that produces this.

**Does this work offline?**
Yes. Save the HTML file to disk and open it in any browser. The only network call is for Google Fonts on first load — if your browser blocks it, the page falls back to system fonts and still works.

**What about the bottom side of my board?**
The converter currently outputs top-side only. Bottom-side parts in your file are filtered out at Step 1 (you'll see the count in the diagnostic). For a two-sided board, run the bottom side as a second job: re-export with `mountsmd.ulp` (the script automatically writes a `.mnb` for bottom), load that file separately, and generate a second `.dpv`.

---

## Troubleshooting

**"Couldn't find required column(s)"**
Auto-detection couldn't identify a required column from the header row. Most common causes:
- File has no header row (open it in a text editor to check)
- Headers use unusual names (see the recognized-names table above)
- File has a long preamble that's confusing the header detector — the detector scans the first 15 non-comment lines and picks the most header-like row, but unusual structures can trip it

**"After filtering, 0 placements remain"**
The diagnostic alert breaks down where every row went. Most common causes:
- **Side filter mismatch** — your file uses unusual Side values (`F.Cu`, weird numbers). Set Side filter to "All rows" or set the Side column to "none" on Step 1.
- **All rows have non-numeric coordinates** — the file might have units appended (`"12.5mm"`); the parser strips common unit suffixes but not all.

**"Duplicate feeder IDs assigned"**
Two part types are pointing at the same feeder. The DPV will still generate, but the machine will physically have only one reel at that station and pick the wrong one for half the parts. Re-assign one to an unused feeder ID on Step 2 or Step 3.

**The Skip flag is "moving" to other rows when I tick it.**
Hard-refresh your browser (Ctrl+Shift+R or Cmd+Shift+R). This was a real bug in an earlier version where the input event fired after a re-render and read stale row indices. Fixed in current versions.

---

## Privacy

This is a single static HTML file. Everything runs locally:

- No file you load is uploaded anywhere
- No analytics, no telemetry, no remote calls
- The page fetches Google Fonts on first load — that's the only network call. Replace those `<link>` tags with system fonts if you want zero external requests.
- Saved feeder config JSON files stay entirely on your machine

Open the HTML file in a browser with no internet connection and the tool works fine — the page just falls back to system fonts.

---

## License

MIT License — see the source file or click the *MIT License* link in the footer for the full text.

Copyright © 2026
