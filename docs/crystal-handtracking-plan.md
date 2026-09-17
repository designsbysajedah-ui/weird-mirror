# Variant C — Enchanted crystal, hand-tracked

Started Sept 15 2026. File: `Projectixd415-crystal.1.toe` (Desktop).

A viewer moves a floating crystal with their hands and explodes it into particles with a gesture.
No projector — this one is a screen piece, at least for now.

Reference: @stormypyeatte's TouchDesigner hand-tracking reel.

---

## What this changes

This is a bigger pivot than a new look. Naming it plainly so the decision is on the record:

| | Variants A & B | Variant C |
|---|---|---|
| Input | whole body, motion from a webcam | hands, landmark tracking |
| Subject | the viewer's own silhouette | an object the viewer manipulates |
| Viewer's role | they *are* the piece | they're a controller |
| Output | projection onto a wall | a screen |

**The projector is the thing being dropped.** A projection-mapping piece was the stated reason for
picking this medium, so if that's still wanted for the portfolio, Variant C either needs to end up
back on a projector or live alongside A/B rather than replacing them. Worth deciding deliberately
rather than by drift.

Also worth noting: hand tracking is a *much* more legible affordance than stillness. People
understand "reach out and it responds" instantly. The 300ms discovery problem that dominates A and B
mostly disappears here.

---

## Can a 3D model be dropped in? Yes.

TouchDesigner reads, by format:

| Format | Operator | Notes |
|---|---|---|
| `.obj` | File In SOP | Simplest and most reliable. Geometry only. Best first try. |
| `.fbx` | FBX COMP | Brings materials and hierarchy. Heavier. |
| `.usd` / `.usdz` | USD COMP | Good for scanned assets. |
| `.gltf` / `.glb` | glTF COMP | Materials + PBR. |
| `.abc` | Alembic SOP | Animated caches. |
| `.stl` | File In SOP | Geometry only, no UVs. |

**What matters for this piece:** the model feeds a silhouette that gets speckled and exploded, so
polygon detail matters more than textures. A dense mesh gives finer particles. A low-poly model
gives chunky shards — which for a crystal might be the better look anyway.

Save the file anywhere reachable (Desktop is fine) and say the filename; it can be wired into
`form/shape_pick` as a fourth input so it swaps against the procedural shapes.

---

## Hand tracking on macOS — the real constraint

**TouchDesigner 2025.33230 on macOS has no native hand tracking.** The operator list has
`bodytrackCHOP`, which is Windows/NVIDIA only. There is no Hand Tracking CHOP in this build.

The working route is **torinmb's MediaPipe TouchDesigner plugin** — free, GPU-accelerated, and it
explicitly runs on Mac:

- Latest release v0.5.2 (Nov 2024), tested against TD 2023.11880 and TD 2025.31500
- Download `release.zip`, not the source zip
- **Do not embed `MediaPipe.tox` into the project** — it's 500MB+ and makes every save crawl.
  Reference it from its own folder.
- Data arrives over WebSocket as JSON and is unpacked by the included decoder `.tox` components
- Outputs hand landmarks *and* recognized gestures (open palm, closed fist, pinch, point)

Gesture recognition is included, so "explode on a closed fist" doesn't need to be hand-rolled from
landmark math.

---

## What's built and working now

The interaction is wired end to end against a **mouse stub**, so it's testable before MediaPipe is
installed. Swapping in real hand data is a two-wire change.

```
mouse_in → hand_lag → HAND (null CHOP)
              channels: hand_x, hand_y, grip
```

- `hand_x` / `hand_y` → the crystal's rotation
- `grip` (left mouse button, later a closed fist) → explode

`state_machine` now has a `MODE` at the top:

- `MODE = 'hand'` — grip charges the explosion, releasing reassembles (currently active)
- `MODE = 'stillness'` — the Variant B behavior, kept intact

Both write the same `STATE` channels, so everything downstream is shared. Tunables in hand mode:

```python
GRIP_THRESH   = 0.5
EXPLODE_FILL  = 1.9    # explode fast
EXPLODE_DRAIN = 1.3    # reassemble slightly slower
LIFT_RATE     = 0.22   # exploded cloud drifts upward
LIFT_MAX      = 0.45
```

### Geometry

`form` (Geometry COMP) holds a Switch SOP, `shape_pick`:

- **0** — organic warped ring
- **1** — noise-displaced sphere (the artichoke-ish blob)
- **2** — quartz crystal: hex shaft with pyramid caps, faceted (currently active)

The crystal is built from three Tube SOPs merged and run through a Facet SOP with `unique` and
`cusp` on, which is what gives it hard faceted shading instead of a smooth-shaded tube.

### The enchanted palette

Ramp keys rewritten from the fire/ice spectrum to deep violet → magenta → cyan → white, vertical.
Combined with the existing bloom and chromatic aberration, the intact crystal reads as glass with
colored fringing at its edges; exploded, it becomes violet and cyan shards.

---

## Gotchas found this session

- **`sin()` is not defined in TouchDesigner parameter expressions** — it's `math.sin()`. A bad
  expression on the ramp's Phase made the whole output go black with no node error visible
  downstream; the ramp itself held the error. When the output goes black, walk *upstream* and check
  each TOP, and check parameter expressions, not just node errors.
- Sphere SOP defaults to `type = prim` — a **single point**, so a Noise SOP downstream displaces
  nothing and the shape stays perfectly smooth. Set `type = mesh` and give it rows/cols.
- Noise SOP keeps the original normals by default (`keepnormals` on), so even correctly displaced
  geometry shades as though it were still smooth. Turn it off.
- Transform TOP scale is `sx`/`sy`, not `scalex`/`scaley`.
- Facet SOP has no `computenormals` — it's `postnml`.
- Switch SOP's parameter is `input`, not `index` (the Switch **TOP** uses `index`).

---

## Next

1. Install the MediaPipe plugin, confirm it runs on this Mac, and check the frame rate cost.
2. Replace `mouse_in` with the hand landmark stream; map pinch distance to explode rather than a
   binary grip, so the crystal can be held half-open.
3. Drop in the real crystal model once it exists.
4. Decide the projector question above.
