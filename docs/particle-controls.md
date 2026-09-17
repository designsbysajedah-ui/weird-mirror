# Particle Controls — what to change and where

Every knob lives on a node in `/project1`. Click the node, hit **P** to open its parameters, change
the value. Changes are live — no restart. Current values as of `Projectixd415.11.toe`.

---

## The look of the dots

| Want to change | Node | Parameter | Now | Try |
|---|---|---|---|---|
| **How many dots** | `dot_thresh` | Threshold | 0.58 | Lower = more dots. 0.3 is dense, 0.8 is sparse |
| **Dot size / softness** | `glow_blur` | Size | 2 | Higher = fatter, blurrier. 1 = pinpricks, 8 = blobs |
| **Dot pattern** | `dot_noise` | Seed | 3 | Any number. Just reshuffles which pixels are dots |
| **Grain character** | `dot_noise` | Type | random | `random` = even scatter. Try `sparse` or `simplex2d` for clumping (note: `sparse` is slow) |
| **Color** | `glow_tint` | Color R/G/B | 0.45 / 0.85 / 1.0 | Currently cyan-white. Warm: 1.0 / 0.6 / 0.3 |
| **Overall brightness** | `glow_level` | Brightness | 2.4 | Raise for a projector, lower if it blows out |

## The movement

| Want to change | Node | Parameter | Now | Try |
|---|---|---|---|---|
| **How much they scatter** | `drift` | Displace Weight X/Y | expression on MOTION | Edit the `0.10` in the expression. Higher = wilder |
| **Scale of the flow field** | `flow_noise` | Period | 0.35 | Small = tight turbulence. Large = broad drifting currents |
| **Speed of the flow** | `flow_noise` | Translate Z | expression | Edit the `0.15`. Higher = faster churn |
| **Trail length** | `trail_decay` | Brightness | 0.91 | 0.80 = short trails. 0.97 = long smears. **Never 1.0** — it never fades and the screen fills up |

## What counts as motion

| Want to change | Node | Parameter | Now | Try |
|---|---|---|---|---|
| **Sensitivity** | `silhouette` | Threshold | 0.102 | Lower = faint movement registers. Higher = only big gestures |
| **Blobbiness of the mask** | `mo_blur` | Size | 6 | Higher = softer, more forgiving clumps |
| **Slow-motion pickup** | `mo_prev` | Index | -4 | More negative = compares further back = catches slower movement. -8 ≈ 130ms |
| **Brightness response** | `body_lvl` | Brightness | expression on MOTION | `0.20 + 1.10 * motion`. Raise 0.20 for a stronger resting glow |

---

## Notes

- **Threshold TOP comparators are inverted** from the plain reading. `less` outputs white where the
  source is *above* the threshold. Verified by measurement, twice. Don't trust the label.
- `silhouette` threshold is room-specific. Re-measure in the classroom.
- Parked, not deleted: `state_machine`, `rise`, `spirit_fade` (the release-and-depart behavior), and
  `bg_plate` / `bg_ref` / `body_diff` / `diff_blur` (background subtraction). All disconnected but
  intact if that design comes back.

## Performance

`node.cookTime` sorted descending finds the slow node in one step. Budget is **16.7 ms** for 60fps;
currently around 13.7. If it gets tight, drop `dot_noise` and `mo_diff` to 480x270 first.

32-bit float only where values are tiny (the motion chain). Everything visual is 8-bit — going 32-bit
across the visual chain once cost 1000 ms a frame for no benefit.
