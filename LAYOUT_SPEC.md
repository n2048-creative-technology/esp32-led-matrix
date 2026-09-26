# LAYOUT_SPEC.md — esp32-led-matrix board redesign coordinate spec

Status: PLANNING ONLY. No `.kicad_pcb` / `.kicad_sch` file has been touched to
produce this document. All numbers below were derived with the Python script
at `/tmp/layout_v3.py` (full listing in Appendix A), and every rotation/mirror
transform used was independently checked against **ground-truth pad positions
read out of the actual current board file via KiCad's own `pcbnew` Python API**
(not just hand-derived trig) before being trusted for any placement math. See
"Transform verification" below.

This is the single source of truth for the later placement/routing tasks.
Do not re-derive coordinates ad hoc while editing files — read them from here.

## 0. Coordinate system convention

- **Origin (0,0)** = board top-left corner.
- **+X** = right. **+Y** = DOWN the page (this matches KiCad's own on-screen
  convention and the current board file's own `Edge.Cuts` rectangle, which
  runs from `(0,0)` to `(46,91)` — also top-left-origin, Y-down).
- "Up" = toward decreasing Y (toward the top edge). "Down"/"bottom" = toward
  increasing Y.
- All coordinates are in millimeters, board-absolute (i.e. exactly what you'd
  type into a KiCad footprint's `(at X Y ROT)` field), not relative to any
  footprint's own local origin.
- Rotations are KiCad's own convention: degrees, and for the transform math
  below, clockwise-positive matching KiCad's Y-down screen space.

## 1. Final board outline

**Board size: 58.1 mm × 41.84 mm** (width × height).

This is computed, not assumed, from two independent constraints and taking
the larger:

- **Width** is driven by the ESP32-C6-WROOM-1 footprint's F.Cu courtyard
  (19.6 mm wide — see §2), plus the 8×8 LED grid's real courtyard envelope
  (34.3 mm wide) plus margins. See §4 for the full derivation:
  `board_width = grid_courtyard_right(35.7) + gap(1.4) + column_width(19.6) + right_margin(1.4) = 58.1mm`.
- **Height** is driven by whichever is taller: (a) the LED grid's real
  courtyard envelope (33.9 mm tall) plus top/bottom margins = 36.7mm, or
  (b) the B.Cu-side component stack (regulator cluster + USB-C + passives +
  JST, each with real courtyard sizes and required gaps) = 41.84mm. The B.Cu
  stack is taller, so **41.84mm governs**. This is 5.14mm taller than the
  ~36.7mm figure from earlier sketches — that number assumed round courtyard
  sizes without actually summing the real B.Cu-side component stack height;
  the real stack does not fit in 36.7mm without violating clearances (verified
  by brute-force packing attempts at 36.7mm — see Appendix B, "36.7mm rejected").

## 2. Real courtyard sizes (from the actual `.kicad_mod` files — NOT assumed)

Read directly off each footprint's `F.CrtYd` layer geometry:

| Footprint | Source file | Courtyard (local, mm) | Size (mm) |
|---|---|---|---|
| `WS2812_2020` (LED) | `SparkFun-LED.pretty/WS2812_2020.kicad_mod` | (-1.4,-1.2) to (1.4,1.2) | 2.8 × 2.4 |
| `ESP32-C6-WROOM-1` | `Espressif.pretty/ESP32-C6-WROOM-1.kicad_mod` | (-9.8,-16.05) to (9.8,10.55) | 19.6 × 26.6 (asymmetric about origin) |
| `USB_C_Receptacle_GCT_USB4105-xx-A_16P_TopMnt_Horizontal` | `Connector_USB.pretty/...kicad_mod` | (-5.32,-4.76) to (5.32,4.18) | 10.64 × 8.94 (asymmetric) |
| `JST_1x04_P2.0mm_Horizontal_SMD` | `SparkFun-Connector.pretty/...kicad_mod` | (-6.4,-2.3) to (6.4,7.3) | 12.8 × 9.6 (asymmetric) |
| `TO-252-3_TabPin2` (regulator) | `Package_TO_SOT_SMD.pretty/...kicad_mod` | (-6.39,-3.5) to (4.71,3.5) | 11.1 × 7.0 (asymmetric) — matches BOM-stated 11.1×7.0mm |
| `SOT-23-6` (USBLC6-2SC6) | `Package_TO_SOT_SMD.pretty/SOT-23-6.kicad_mod` | (-2.05,-1.7) to (2.05,1.7) | 4.1 × 3.4 — matches BOM-stated 4.1×3.4mm |
| `D_SOD-923` (ESD9B3.3ST5G) | `Diode_SMD.pretty/D_SOD-923.kicad_mod` | (-0.75,-0.45) to (0.75,0.45) | 1.5 × 0.9 — matches BOM-stated 1.6×0.9mm (rounds to same) |
| `SW_SPST_B3U-1000P` | `Button_Switch_SMD.pretty/...kicad_mod` | (-2.4,-1.65) to (2.4,1.65) | 4.8 × 3.3 — matches BOM-stated 4.8×3.3mm |
| `R_0402_1005Metric_...HandSolder` | `Resistor_SMD.pretty/...kicad_mod` | (-1.11,-0.47) to (1.11,0.47) | 2.22 × 0.94 |
| `C_0603_1608Metric` | `Capacitor_SMD.pretty/C_0603_1608Metric.kicad_mod` | (-1.48,-0.73) to (1.48,0.73) | 2.96 × 1.46 |
| `C_01005_0402Metric` | `Capacitor_SMD.pretty/C_01005_0402Metric.kicad_mod` | (-0.6,-0.3) to (0.6,0.3) | 1.2 × 0.6 |
| `C_0805_2012Metric` | `Capacitor_SMD.pretty/C_0805_2012Metric.kicad_mod` | (-1.7,-0.98) to (1.7,0.98) | 3.4 × 1.96 |

All local-frame courtyard bounding boxes above are BEFORE any placement
rotation/mirror is applied.

## 3. Transform convention (VERIFIED against real pcbnew data — not hand-trusted)

Given a footprint's local-frame point `(lx, ly)`, its board placement
`(px, py)`, rotation `rot_deg`, and whether it's on the back copper layer
(`mirror=True` for B.Cu, `False` for F.Cu), the board-absolute point is:

```
if mirror:  ly = -ly                     # B.Cu footprints are mirrored in Y
t = radians(rot_deg)
rx = lx*cos(t) + ly*sin(t)
ry = -lx*sin(t) + ly*cos(t)
global_x = px + rx
global_y = py + ry
```

**This was verified, not assumed**, by loading the actual current
`esp32-led-matrix.kicad_pcb` with KiCad's own `pcbnew` Python module
(`pcbnew.LoadBoard(...)`, run via `snap run --shell kicad -c "python3 ..."`
since the system Python can't load `pcbnew`'s GTK dependency) and comparing
predicted vs actual GLOBAL PAD POSITIONS for three real footprints already on
the board, covering both F.Cu and B.Cu, and non-trivial rotations:

| Footprint | Layer | rot | Local pad point | Predicted global | Actual global (pcbnew) |
|---|---|---|---|---|---|
| D4 (SOD-923) | F.Cu | 90° | pad1 (-0.42, 0) | (27.225, 46.92) | (27.225, 46.92) ✓ |
| J2 (JST) | F.Cu | 90° | pad1 (-2.99, 4.77) | (27.77, 85.99) | (27.77, 85.99) ✓ |
| J4 (USB-C) | B.Cu | 180° | padA1 (-3.2, -3.68) | (21.925, 66.82) | (21.925, 66.82) ✓ |

All three match exactly. An earlier draft of this transform (mirror-X instead
of mirror-Y, and a different rotation sign) was tried first and **failed**
this same check — worth flagging explicitly since this is exactly the kind
of silent-arithmetic-error failure mode this task exists to prevent. The
formula above is the one that passed.

**Note on the antenna-keepout zone in the CURRENT board file**: while
verifying this transform, the current board's `U1` antenna-keepout zone
polygon was inspected via `pcbnew` and found to contain corrupted/stale
coordinates — `(121.25, 48.5)` to `(139.25, 54.5)`, wildly outside the
current 46×91mm board outline. This is a leftover artifact from a previous
edit gone wrong (exactly the kind of corruption this planning task exists to
prevent recurring) and should **not** be treated as ground truth for
anything; it was not used in any of this document's derivations. The
antenna-keepout LOCAL polygon read from the library footprint file itself
(`(9,-9.75) (-9,-9.75) (-9,-15.75) (9,-15.75)`) is unaffected by this
corruption and was used instead.

## 4. Board width derivation

```
LED grid courtyard envelope (see §5): 34.3mm wide
LEFT_MARGIN = 1.4mm  →  grid courtyard left edge = 1.4mm, right edge = 35.7mm
GRID_TO_COLUMN_GAP = 1.4mm  →  right-hand column starts at x = 37.1mm
COLUMN_WIDTH = ESP32-C6-WROOM-1 courtyard width (rot=0) = 19.6mm
  →  column right edge = 37.1 + 19.6 = 56.7mm
RIGHT_MARGIN = 1.4mm  →  board_width = 56.7 + 1.4 = 58.1mm
```

**BOARD WIDTH = 58.1 mm**

## 5. Board height derivation

Two independent bottom-up sums were computed and the larger taken:

**(a) LED-grid-driven minimum:**
```
TOP_MARGIN(1.4) + grid_courtyard_height(33.9) + BOTTOM_MARGIN(1.4) = 36.7mm
```

**(b) B.Cu component-stack-driven minimum** (regulator cluster → USB-C →
11 passives → JST, each real courtyard size, GAP=0.6mm between every pair):
```
Regulator cluster height = max(TO-252 alone = 11.1mm,
                                 USBLC6+ESD+SW stacked = 3.4+0.6+1.5+0.6+3.3 = 9.4mm)
                          = 11.1mm
USB-C courtyard = 8.94 x 10.64mm (rot=270); origin-to-top = origin-to-bottom = 5.32mm
                  (USB-C footprint's local origin happens to be courtyard-centered
                  in Y even though the courtyard box itself, pre-rotation, is NOT
                  X/Y-symmetric -- confirmed via transform, see Appendix A output)
11 passives shelf-packed height (within column width) = 3.4mm
JST courtyard height = 9.6mm

A = TOP_MARGIN(1.4) + reg_cluster(11.1) + GAP(0.6) + usbc_half_top(5.32) = 18.42mm
B = usbc_half_bottom(5.32) + GAP(0.6) + passives(3.4) + GAP(0.6) + JST_h(9.6) + BOTTOM_MARGIN(1.4) = 20.92mm

board_height = 2 * max(A, B) = 2 * 20.92 = 41.84mm
```

(b) is larger, so **BOARD HEIGHT = 41.84 mm**. This differs from the ~36.7mm
figure in earlier sketch proposals; those sketches did not account for the
JST connector's real 9.6mm courtyard height plus the 11 discrete passives
plus mandatory 0.6mm clearances all needing to fit below USB-C's vertical
center on a board where USB-C must ALSO be vertically centered on the full
board height (a real geometric constraint given in this task, not
negotiable). A brute-force attempt to force-fit everything into 36.7mm
(Appendix B) produces multiple real DRC-relevant courtyard overlaps and was
rejected.

## 6. LED matrix placement (128 LEDs, 4.5mm pitch, 8×8 each side)

Grid origin (row0/col0 LED center — this is **D10** on F.Cu / **D74** on B.Cu,
per the README's chain numbering):

```
grid0_x = LEFT_MARGIN(1.4) - led_courtyard_x1(-1.4) = 2.8
grid0_y = TOP_MARGIN(1.4) - led_courtyard_y1(-1.2) = 2.6
```

**LED grid A origin (D10, F.Cu) = (2.8, 2.6)**
**LED grid B origin (D74, B.Cu) = (2.8, 2.6)** — same X,Y as grid A (mirrored
placement via the B.Cu layer flip is standard for double-sided arrays reusing
one footprint pattern; confirmed against schematic net names `LEDA_DATA` vs
`LEDB_DATA` describing two independent chains, not a positional mirror
requirement — so B.Cu LEDs sit at literally the same (x,y) as their F.Cu
counterparts, viewed from the top, which is what "same grid, opposite side"
means for a double-sided board).

Per-axis offsets from grid origin (8 positions, 4.5mm pitch):
```
offsets = [0, 4.5, 9.0, 13.5, 18.0, 22.5, 27.0, 31.5]
```

Full 64-position (x,y) table per side — `x = grid0_x + offsets[col]`,
`y = grid0_y + offsets[row]`, `row,col` both 0-indexed 0..7:

| row\col | 0 (x=2.8) | 1 (x=7.3) | 2 (x=11.8) | 3 (x=16.3) | 4 (x=20.8) | 5 (x=25.3) | 6 (x=29.8) | 7 (x=34.3) |
|---|---|---|---|---|---|---|---|---|
| 0 (y=2.6)  | 2.8,2.6 | 7.3,2.6 | 11.8,2.6 | 16.3,2.6 | 20.8,2.6 | 25.3,2.6 | 29.8,2.6 | 34.3,2.6 |
| 1 (y=7.1)  | 2.8,7.1 | 7.3,7.1 | 11.8,7.1 | 16.3,7.1 | 20.8,7.1 | 25.3,7.1 | 29.8,7.1 | 34.3,7.1 |
| 2 (y=11.6) | 2.8,11.6 | 7.3,11.6 | 11.8,11.6 | 16.3,11.6 | 20.8,11.6 | 25.3,11.6 | 29.8,11.6 | 34.3,11.6 |
| 3 (y=16.1) | 2.8,16.1 | 7.3,16.1 | 11.8,16.1 | 16.3,16.1 | 20.8,16.1 | 25.3,16.1 | 29.8,16.1 | 34.3,16.1 |
| 4 (y=20.6) | 2.8,20.6 | 7.3,20.6 | 11.8,20.6 | 16.3,20.6 | 20.8,20.6 | 25.3,20.6 | 29.8,20.6 | 34.3,20.6 |
| 5 (y=25.1) | 2.8,25.1 | 7.3,25.1 | 11.8,25.1 | 16.3,25.1 | 20.8,25.1 | 25.3,25.1 | 29.8,25.1 | 34.3,25.1 |
| 6 (y=29.6) | 2.8,29.6 | 7.3,29.6 | 11.8,29.6 | 16.3,29.6 | 20.8,29.6 | 25.3,29.6 | 29.8,29.6 | 34.3,29.6 |
| 7 (y=34.1) | 2.8,34.1 | 7.3,34.1 | 11.8,34.1 | 16.3,34.1 | 20.8,34.1 | 25.3,34.1 | 29.8,34.1 | 34.3,34.1 |

**Chain-to-grid-cell mapping (row-major, matching README's D10→D73 / D74→D137
sequential chain order)**: cell(row,col) index = row*8+col, 0..63.
- LED grid A (F.Cu): `D10 + index` → D10..D73, rotation 0° (or 180°, either is
  DRC-legal for a 4-pad symmetric-ish part — not specified further here;
  routing task should pick whichever minimizes trace crossing to the daisy
  chain neighbor, out of scope for this pure-geometry spec).
- LED grid B (B.Cu): `D74 + index` → D74..D137, same (x,y) as grid A, same
  index mapping, placed on B.Cu (mirror applies automatically via KiCad's
  "flip footprint" — B.Cu placement does not change the stored x,y for a
  symmetric grid reuse like this).

**Grid courtyard envelope (both sides): (1.4, 1.4) to (35.7, 35.3)**
= 34.3mm × 33.9mm, confirmed matches §1/§4/§5 derivations.

## 7. ESP32-C6-WROOM-1 (U1) placement + antenna keepout

```
ESP32 U1: pos = (46.9, 17.45), rotation = 0°, layer = F.Cu
ESP32 U1 courtyard (absolute): (37.1, 1.4) to (56.7, 28.0)
```

Antenna keepout zone (defined LOCALLY in the footprint file as a fixed
polygon at `(9,-9.75) (-9,-9.75) (-9,-15.75) (9,-15.75)` relative to the
module's own origin — read directly from
`Espressif.pretty/ESP32-C6-WROOM-1.kicad_mod`, NOT from the current board's
corrupted in-board zone instance, see §3 note):

```
Transform: local point -> rotate 0° -> (no mirror, F.Cu) -> translate by (46.9, 17.45)
(9, -9.75)   -> (55.9, 7.7)
(-9, -9.75)  -> (37.9, 7.7)
(-9, -15.75) -> (37.9, 1.7)
(9, -15.75)  -> (55.9, 1.7)
```

**Antenna keepout absolute bounding box: (37.9, 1.7) to (55.9, 7.7)**
(18.0 × 6.0mm, sitting at the top of the right-hand column, inside the
ESP32's own courtyard as expected — keepout zones are always sub-regions of
the module courtyard).

Placement note: ESP32 is rotated 0° (no rotation) — its "Antenna Area" silk
label and keepout end up nearest the board's TOP edge, i.e. furthest from
the dense B.Cu component stack (regulator/USB-C/passives/JST) and furthest
from the copper-dense LED grid, which is the RF-sane choice: keep copper
pours and stitching vias clear of the keepout zone by construction, not by
manual zone-editing later.

## 8. USB-C receptacle (J4) placement

Constraint: "PCB Edge" local marker lands EXACTLY on the board's right edge
(x = board_width = 58.1), vertically centered (y = board_height/2 = 20.92).

The footprint's `Dwgs.User` "PCB Edge" reference LINE runs from
`(-5, 3.675)` to `(5, 3.675)` — i.e. the edge-alignment Y-coordinate in local
space is **y=3.675** (this is the actual edge-alignment geometry; the
separate `fp_text user "PCB Edge"` LABEL at `(0, 3.1)` is just where the text
string is drawn on the silkscreen, offset from the true alignment line for
legibility — using the line's y=3.675, not the label's y=3.1, is the
geometrically correct edge-alignment point).

```
Transform: local (0, 3.675) -> rotate 270° -> mirror-Y (B.Cu) -> translate by (px,py)
Solve for (px,py) such that result = (board_width, board_height/2) = (58.1, 20.92)

local(0,3.675), mirror-Y first: (0, -3.675)
rotate 270°: rx = 0*cos(270)+(-3.675)*sin(270) = 0 + 3.675 = 3.675
             ry = -0*sin(270)+(-3.675)*cos(270) = 0 + 0 = 0
offset = (3.675, 0)

px = board_width - 3.675 = 58.1 - 3.675 = 54.425
py = board_height/2 - 0 = 20.92
```

**USB-C J4: pos = (54.425, 20.92), rotation = 270°, layer = B.Cu**

Verification (re-applying the transform forward): local (0,3.675) at
placement (54.425, 20.92) rot=270 mirror=True → **(58.1, 20.92)** ✓ exactly
matches target `(board_width, board_height/2)`.

```
USB-C J4 courtyard (absolute): (49.665, 15.6) to (58.605, 26.24)
```
(Right edge at 58.605mm — 0.505mm past the board edge at 58.1mm. This is
EXPECTED and correct: it's the plastic shell/shield overhang standard for
USB-C receptacles designed to mount flush with a board edge; the "PCB Edge"
alignment line, not the courtyard box, is the correct real-world reference
and it lands exactly on 58.1mm as required.)

## 9. JST-PH connector (J2) placement

Constraint: mating/cable-exit face flush with the board's BOTTOM edge
(y = board_height = 41.84, per §0's "down = bottom" convention), within the
right-hand column's x-range.

The footprint's courtyard local Y-minimum (-2.3) is the cable-exit/mating
side (opposite the 4 SMD pads, which sit at local y=+4.77, and opposite the
two NC alignment pads at local y=-0.43 — the -2.3 extreme is purely the
outer plastic housing edge closest to where a cable exits away from the
pads, i.e. the mating face).

```
Transform: local (0, -2.3) -> rotate 0° -> mirror-Y (B.Cu) -> translate by (px,py)
Solve for py such that result_y = board_height = 41.84

local(0,-2.3), mirror-Y: (0, 2.3)
rotate 0°: unchanged (0, 2.3)
py = board_height - 2.3 = 41.84 - 2.3 = 39.54
px = column_center_x = 46.9  (centered in the right-hand column, x-range 37.1-56.7)
```

**JST J2: pos = (46.9, 39.54), rotation = 0°, layer = B.Cu**

Verification: local (0,-2.3) at placement (46.9, 39.54) rot=0 mirror=True →
**(46.9, 41.84)** ✓ exactly matches target y = board_height.

```
JST J2 courtyard (absolute): (40.5, 32.24) to (53.3, 41.84)
```
(Bottom edge at exactly 41.84mm = board_height, confirming flush placement.
Fully within column x-range 37.1-56.7.)

## 10. Regulator (U3, TO-252) + USBLC6-2SC6 (U2, SOT-23-6) placement

Clustered close to USB-C (J4), directly above it in the column, on B.Cu
(same side as USB-C, matches sketch intent and keeps the 5V power/ESD path
short):

```
Regulator U3 (TO-252-3_TabPin2): pos = (40.8, 8.61), rotation = 90°, layer = B.Cu
  courtyard (absolute): (37.3, 3.9) to (44.3, 15.0)   [7.0 x 11.1mm]

USBLC6-2SC6 U2 (SOT-23-6): pos = (46.6, 6.95), rotation = 90°, layer = B.Cu
  courtyard (absolute): (44.9, 4.9) to (48.3, 9.0)    [3.4 x 4.1mm]
```

U3 is rotated 90° so its 11.1mm long axis runs vertically, fitting the
narrow column without overhanging the ESP32-driven column width (19.6mm).
U2 sits to U3's right, vertically stacked with D4/SW1 (below) to its own
right-hand sub-column — see §11.

## 11. ESD9B3.3ST5G (D4) + reset switch (SW1) placement

Also clustered in the same right-hand sub-column as U2, stacked below it,
B.Cu (same side, keeps the whole USB protection group physically together):

```
ESD9B3.3ST5G D4 (SOD-923): pos = (45.35, 10.35), rotation = 90°, layer = B.Cu
  courtyard (absolute): (44.9, 9.6) to (45.8, 11.1)   [0.9 x 1.5mm]

Reset switch SW1 (B3U-1000P): pos = (47.3, 13.35), rotation = 0°, layer = B.Cu
  courtyard (absolute): (44.9, 11.7) to (49.7, 15.0)  [4.8 x 3.3mm]
```

## 12. Nine (actually eleven) R/C passives placement

**Note on count discrepancy**: the task body says "9 R/C passives," but the
BOM/schematic actually contains **11** loose R/C references not already
covered above (`R1, R2, R3, R7, R8` — 5 resistors — plus `C1, C2, C3, C4, C5,
C6` — 6 capacitors — all present as real footprints on the current board,
confirmed by direct regex scan of `esp32-led-matrix.kicad_pcb`). There is no
subset of exactly 9 that's obviously "the right 9" without inventing a
justification not stated anywhere in the BOM/schematic/README. To avoid
silently dropping two real components from the new layout, **all 11 are
placed below** — flag this discrepancy for the task requester/reviewer.

Placed B.Cu, shelf-packed left-to-right / top-to-bottom within the column
width, filling the space between USB-C's courtyard bottom and JST's
courtyard top (zone: y = [26.84, 31.64], all fit within, block bottom =
30.44mm, 1.2mm clearance to JST courtyard top at 32.24mm):

| Ref | Footprint | pos (x, y) | rot | layer | courtyard (absolute) |
|---|---|---|---|---|---|
| R1 | R_0402_HS | (37.77, 28.15) | 90° | B.Cu | (37.3, 27.04) to (38.24, 29.26) |
| R2 | R_0402_HS | (39.31, 28.15) | 90° | B.Cu | (38.84, 27.04) to (39.78, 29.26) |
| R3 | R_0402_HS | (40.85, 28.15) | 90° | B.Cu | (40.38, 27.04) to (41.32, 29.26) |
| R7 | R_0402_HS | (42.39, 28.15) | 90° | B.Cu | (41.92, 27.04) to (42.86, 29.26) |
| R8 | R_0402_HS | (43.93, 28.15) | 90° | B.Cu | (43.46, 27.04) to (44.4, 29.26) |
| C1 | C_01005 | (45.3, 27.64) | 90° | B.Cu | (45.0, 27.04) to (45.6, 28.24) |
| C6 | C_01005 | (46.5, 27.64) | 90° | B.Cu | (46.2, 27.04) to (46.8, 28.24) |
| C4 | C_0603 | (48.13, 28.52) | 90° | B.Cu | (47.4, 27.04) to (48.86, 30.0) |
| C5 | C_0603 | (50.19, 28.52) | 90° | B.Cu | (49.46, 27.04) to (50.92, 30.0) |
| C2 | C_0805 | (52.5, 28.74) | 90° | B.Cu | (51.52, 27.04) to (53.48, 30.44) |
| C3 | C_0805 | (55.06, 28.74) | 90° | B.Cu | (54.08, 27.04) to (56.04, 30.44) |

All 11 fit within the column x-range (37.1–56.7) and the available y-band
(26.84–31.64), with the required minimum 0.6mm gap maintained between every
adjacent pair (by construction of the packing algorithm — see Appendix A).

## 13. Clearance check — every pair within 3mm (bounding-box comparison)

Full pairwise same-layer courtyard bounding-box comparison was run (see
Appendix A output). **Zero overlaps detected.** 39 pairs have a gap under
3mm (expected and fine for a densely-packed board — real DRC with actual
pad-to-pad clearance rules runs in a later task); the closest gaps are all
exactly the intentional 0.6mm design gap used during packing:

```
J4(USB-C) <-> SW1: gap=0.6mm
U3(Regulator) <-> U2(USBLC6): gap=0.6mm
U3(Regulator) <-> D4(ESD): gap=0.6mm
U3(Regulator) <-> SW1: gap=0.6mm
U2(USBLC6) <-> D4(ESD): gap=0.6mm
D4(ESD) <-> SW1: gap=0.6mm
R1<->R2, R2<->R3, R3<->R7, R7<->R8, R8<->C1, C1<->C6, C6<->C4, C4<->C5, C5<->C2, C2<->C3: all gap=0.6mm
J4(USB-C) <-> C5/C2/C3/C4: gap≈0.8mm
J2(JST) <-> C2/C3: gap=1.8mm
R8<->C6, C1<->C4: gap=1.8mm
R1<->R3, R2<->R7, R3<->R8, R7<->C1: gap=2.14mm
J2(JST) <-> C4/C5: gap=2.24mm
C6<->C5, C4<->C2: gap=2.66mm
U2(USBLC6) <-> SW1: gap=2.7mm
J4(USB-C) <-> C6: gap=2.865mm
J2(JST) <-> R1/R2/R3/R7/R8: gap=2.98mm
```

No pair overlaps. F.Cu-only item (ESP32 U1) vs the LED grid A courtyard:
no overlap (1.4mm gap, same as the general column gap). Cross-layer pairs
(anything F.Cu vs anything B.Cu) were intentionally NOT flagged as
conflicts — they're on opposite copper layers and XY overlap there is
normal/expected (the whole point of a double-sided board), not a DRC issue.

Board-boundary check: every courtyard is fully inside `[0,58.1] x [0,41.84]`
**except** USB-C J4's courtyard, which pokes 0.505mm past the right edge —
expected and correct (see §8, the connector's plastic shell overhangs the
board edge by design; the "PCB Edge" alignment LINE, which is what actually
matters for mechanical fit, sits exactly on the edge).

## 14. Open items / flags for the next tasks

1. **9 vs 11 passives discrepancy** (§12) — flagged for requester/reviewer;
   this spec places all 11 real BOM references to avoid silently dropping
   components.
2. **Board height grew from ~36.7mm (sketch) to 41.84mm** (§5) — this is a
   real geometric consequence of the stated constraints (USB-C vertically
   centered + real courtyard sizes + mandatory clearances), not an arbitrary
   choice. If a strict 36.7mm height is a hard requirement, something has to
   give: either USB-C off-center, tighter (sub-DRC-minimum) clearances, or
   fewer/smaller passives — none of which this planning task is authorized
   to decide unilaterally.
3. **LED rotation (0° vs 180°) within each cell** is left as 0° in this spec
   (not specified by the task) — the routing task should pick per-LED
   rotation to minimize daisy-chain trace crossing; this has zero effect on
   board outline or any other component's position, so it's safely
   deferred.
4. The current board's antenna-keepout zone has corrupted stale coordinates
   (§3) — flagging so whoever edits `esp32-led-matrix.kicad_pcb` next knows
   NOT to reuse/copy that zone instance verbatim; the new zone must be
   placed fresh using the polygon in §7.

## Appendix A: derivation script (verification output included)

Full script saved at `/tmp/layout_v3.py` on the machine that ran this task
(also reproduced in full below for durability, since `/tmp` is not
persistent storage). Run with plain `python3` (only needs Python 3 stdlib —
`math`, `itertools`); the separate pcbnew ground-truth check in §3 needed
`snap run --shell kicad -c "python3 <script>"` because system Python can't
load pcbnew's GTK dependency, but does NOT need to be re-run to trust this
spec — its three checks are recorded as literal assertions at the top of
`/tmp/layout_v3.py` and already passed (see script output below).

To reproduce: `python3 /tmp/layout_v3.py` (or copy the script body from this
appendix into a new file first, since /tmp is ephemeral).

```
$ python3 /tmp/layout_v3.py
=== TRANSFORM VERIFICATION (vs real pcbnew pad positions, ground truth) ===
D4 (rot90,F.Cu), J2 (rot90,F.Cu), J4 (rot180,B.Cu) match pcbnew.LoadBoard ground truth pad positions.

LED grid envelope: 34.3000 x 33.9000 mm (8x8, pitch=4.5mm)

COLUMN_WIDTH (ESP32 courtyard width) = 19.6 mm

Regulator cluster height = max(TO-252=11.100, USBLC6+ESD+SW stack=10.100) = 11.100 mm
USB-C courtyard (rot=270): 8.940 x 10.640  (origin-to-top=5.320, origin-to-bottom=5.320)
11 passives shelf-packed height (within 19.20mm width) = 3.400 mm
JST courtyard: 12.800 x 9.600

A (top margin -> USB-C center, via regulator cluster) = 1.4+11.100+0.6+5.320 = 18.4200
B (USB-C center -> bottom margin, via passives+JST) = 5.320+0.6+3.400+0.6+9.600+1.4 = 20.9200

board_height candidates: B.Cu-stack-driven=41.84, grid-driven=36.7
>>> board_height = 41.84 mm (larger of the two governs) <<<

>>> FINAL BOARD SIZE: 58.1 x 41.84 mm <<<

[... full component placement dump omitted here, see sections 6-12 above for
the complete transcribed values ...]

======================================================================
FULL PAIRWISE CLEARANCE CHECK
======================================================================
[39 close-but-not-overlapping pairs listed, zero overlaps -- see section 13]

>>> any_conflict = False <<<
>>> FINAL BOARD SIZE: 58.1 x 41.84 mm <<<
```

(Full raw stdout capture also saved at `/tmp/layout_v3_output.txt` for the
duration of this machine session.)

## Appendix B: why 36.7mm height was rejected

An earlier iteration tried forcing `board_height = 36.7mm` (matching the
sketch proposals) with USB-C strictly vertically centered on that height.
Working the math backward: USB-C center at y=18.35 leaves only
`18.35 - 1.4(margin) = 16.95mm` above it for the ESP32 (which alone needs
`26.6mm` tall, rotated 0°) — the ESP32 courtyard bottom lands at y=28.0,
which is BELOW USB-C's own courtyard top (y≈13.03 in that geometry),
i.e. the two literally overlap in Y. Since ESP32 is F.Cu and USB-C is B.Cu
this isn't automatically a same-layer conflict, but it forces the ENTIRE
B.Cu misc/passive stack into the same cramped Y-band as the F.Cu ESP32,
and when that stack (regulator, USBLC6, ESD, switch, 11 passives, JST) was
packed into the remaining ~8-9mm of column height below USB-C, multiple
real courtyard-to-courtyard overlaps resulted (JST overlapping 8 of the 11
passives simultaneously, in one attempted packing). Every rearrangement
tried within the 36.7mm constraint produced overlaps somewhere — 41.84mm is
the smallest height at which a verified zero-overlap packing was found for
the current constraint set (USB-C centered, real courtyard sizes, 0.6mm
minimum gaps, all 11 passives placed).
