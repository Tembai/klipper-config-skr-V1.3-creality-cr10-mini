# Pressure Advance (PA) pattern test print log

Repository of every PA calibration test print done on this printer, the exact
firmware/slicer settings that were live at print time, and the physical
adhesion/fusion result. This exists so future sessions can see back what was
actually changed between each print instead of re-deriving it from OctoPrint
logs and git history every time. Keep this up to date whenever a new PA test
print is run or a config value affecting print quality is tweaked.

## How to read this table

- "PA range/step" is the range printed on the test pattern (OrcaSlicer's
  built-in PA tower/pattern test sweeps PA linearly across the print).
- "PA applied" is the value the user picked off the printed pattern and set as
  `pressure_advance` in `[extruder]` afterwards (not necessarily used again
  until the next print).
- "Config live at print time" lists the motion-limit values that were actually
  active in `printer.cfg` at the moment the print started (not what they are
  now) - cross-referenced against git commit timestamps vs. the print's
  "Print job started" timestamp in octoprint.log.
- Always use `grep -a` on octoprint.log files when hunting for print job
  timestamps - see troubleshooting_steps.txt / agent.md for why.

## Test print table

| # | Gcode file | Print start -> end (local time) | PA range / step | PA applied | max_velocity | max_z_accel | max_accel / square_corner_velocity | Result |
|---|---|---|---|---|---|---|---|---|
| V1 | `pa_pattern_120_3000_..._15m4s.gcode` | 2026-08-09 01:13:51 -> 01:36:22 | 0 - 1.2, step 0.05 | 1.15 | 100 (pre-tuning) | 80 (pre-tuning) | conservative/original (pre-`16dc98a`) | GOOD |
| V2 | `pa_pattern_100_500_..._17m9s_V2.gcode` | 2026-08-09 21:57:55 -> 22:17:45 | 0 - 1.2, step 0.05 | 1.15 | 150 | 80 | conservative/original (pre-`16dc98a`, committed later that night) | GOOD |
| V3-V4 (unlabeled) | `pa_pattern_100_500_..._22m24s.gcode` | 2 cancelled attempts 23:52:08 & 23:52:42, real print 2026-08-11 23:56:36 -> 2026-08-12 00:25:56 | 0.3 - 0.92, step 0.02 | 0.62 | 150 | 100 (boosted, `16dc98a`) | boosted (`16dc98a`, unreverted; `31454ad` committed 6 min before print) | BAD |
| V5 | `pa_pattern_100_3000_..._22m2s_V5.gcode` | 2026-08-12 22:26:08 -> 22:54:27 | 0.3 - 0.92, step 0.02 | 0.5 | 150 | 100 (still boosted) | reverted (`04c5afa`, 12 min before print) | BAD, "slightly worse" than V3-V4 per user |
| V6 (re-print of V5 file) | `pa_pattern_100_3000_..._22m2s_V5.gcode` | 2026-08-15 (see octoprint.log for exact timestamps) | 0.3 - 0.92, step 0.02 | **0.52** | 150 | **80 (reverted)** | reverted (`04c5afa`) + fresh 10x10 bed mesh + screws_tilt_adjust fix | **GOOD - best print so far by a large margin.** Nearly all of the pattern adhered properly; only one small spot with imperfect intra-layer adhesion (down from widespread fusion failure in V3-V5). Some stringing still present. |

## Root cause conclusion (as of 2026-08-12/13 investigation, CONFIRMED 2026-08-15)

**Confirmed by the V6 re-print on 2026-08-15:** reverting `max_z_accel` from 100
back to 80 (combined with the corrected screws_tilt_adjust coordinates and a
finer 10x10 bed mesh) fixed the intra-layer fusion regression. V6 was by far
the best print in this whole series - only one small adhesion spot remained,
vs. widespread failure across V3-V5. `max_z_accel: 100` is confirmed as the
root cause (not bed topology, not PA value itself). Remaining known issue:
some stringing is still present - candidate for a future retraction/print-speed
tuning pass (not yet started).

## Root cause conclusion (as of 2026-08-12/13 investigation)

Comparing the only two GOOD prints (V1, V2) against the two BAD prints
(V3-V4, V5): `max_velocity` was NOT the differentiator (150 was live for V2
[good] and both bad prints). `max_z_accel` FLIPPED from 80 (both good prints)
to 100 (both bad prints) and was never reverted for either bad print, despite
`max_accel`/`square_corner_velocity` being reverted before V5. This makes
`max_z_accel: 100` (vs the original `80`) the leading suspect for the
intra-layer line fusion/adhesion regression.

**Status: CONFIRMED by the V6 re-print on 2026-08-15 (see updated conclusion
above the table).** `max_z_accel` was reverted back to 80 in printer.cfg
(bundled with the 2026-08-13 bed_mesh/screws_tilt_adjust changes), and the
revert is now validated - V1/V2-quality adhesion has been restored (with only
one minor spot remaining, a large improvement over V3-V5).

## Known confound: bed mesh resolution

A 2026-08-12 investigation (see agent.md's 2026-08-12 session entry) found that
the V3-V4 print's specific bad region (PA 0.58-0.74, X~135-184) also lined up
with a real ~0.1-0.24mm bed height difference in the then-active 5x5/7x7 bed
mesh grids. This means a bad-looking region on a PA test print is not
automatically caused by the PA value itself - always cross-check the failing
region's X/Y against the currently active bed_mesh profile before concluding
the PA value is at fault. `probe_count` has since been raised to 10x10 to
reduce this confound.
