# Variant B — "Dissolve" (stillness → particles)

Planned Sept 15 2026. A second .toe, kept separate from `Projectixd415.21.toe`, to test the
original reading of the interaction: **hold still and your body grains apart into particles.**
Movement pulls you back together.

Reference: @stormypyeatte's hand-tracking reel (TouchDesigner, 3D-scanned artichoke flower as a
soft glowing point cloud with chromatic aberration). We are borrowing the *look*, not the input.
Hand tracking is scoped as a separate experiment.

---

## What changes from Variant A

| | Variant A (`.21`) | Variant B (this file) |
|---|---|---|
| Charges on | movement | **stillness** |
| Recalls on | stillness | **movement** |
| What leaves | a copy peels off, body stays | **the body itself disperses** |
| Look | edge glow + speckle | soft point cloud, bloom, chromatic aberration |

The important difference is the second row from the bottom. In A there are two things on screen
(body + spirit). In B there is one thing that stops being solid. That should read more clearly at
a distance, and it removes the "which one am I?" confusion.

---

## State machine

Same seven STATE channels, so nothing downstream breaks. `glow` is reinterpreted as **dissolve
amount**, 0–1.

```python
# Dissolve state machine  (stillness disperses, movement reassembles)
THRESH      = 0.15   # motion above this counts as "moving"
HOLD        = 3.0    # seconds of stillness before full dispersal
FILL        = 1.0    # dissolve gained per second while still
DRAIN       = 1.6    # dissolve lost per second while moving — snapping back is fast
RECALL_HOLD = 0.4    # movement needed to yank a released cloud home
RISE_RATE   = 0.30
RISE_MAX    = 1.30

IDLE, CHARGING, RELEASED = 0, 1, 2

def onFrameStart(frame):
    st = op('/project1/STATE')
    m  = op('/project1/MOTION')['motion'].eval()
    dt = 1.0 / max(project.cookRate, 1)

    charge = st.par.const0value.eval()
    state  = int(st.par.const3value.eval())
    recall = st.par.const4value.eval()
    rise   = st.par.const5value.eval()
    moving = m > THRESH

    if state in (IDLE, CHARGING):
        rise = 0.0
        if not moving:
            charge = min(HOLD, charge + FILL * dt)
        else:
            charge = max(0.0, charge - DRAIN * dt)
        state = CHARGING if charge > 0 else IDLE
        if charge >= HOLD:
            recall, rise, state = 0.0, 0.0, RELEASED

    elif state == RELEASED:
        rise = min(RISE_MAX, rise + RISE_RATE * dt)
        if moving:
            recall = min(RECALL_HOLD, recall + dt)
        else:
            recall = max(0.0, recall - dt)
        if recall >= RECALL_HOLD or rise >= RISE_MAX:
            charge, recall, rise, state = 0.0, 0.0, 0.0, IDLE

    st.par.const0value = charge
    st.par.const1value = charge / HOLD      # dissolve 0–1
    st.par.const2value = 1.0 if state == RELEASED else 0.0
    st.par.const3value = state
    st.par.const4value = recall
    st.par.const5value = rise
    st.par.const6value = (1.0 - rise / RISE_MAX) if state == RELEASED else 0.0
```

Asymmetric FILL/DRAIN on purpose: dispersal is slow and earned, reassembly is immediate. Flinching
should visibly cost you.

### Timing note still applies

The 300ms rule from the Variant A spec is unchanged and is the whole ballgame here. If nothing
happens in the first third of a second of stillness, nobody discovers the piece. The first grains
should appear almost immediately — sparse, then accelerating.

---

## Visual chain

```
silhouette ─┬─────────────────────────────┐
            │                             ▼
dot_noise → dot_thresh ── × ──────────► dissolve_mix ─► drift ─► rise ─► trail_comp
            (threshold driven by glow)   (Cross, glow)   (Displace,   (ty=rise)   ↕ trail_fb
                                                          weight×glow)             trail_decay
                                                                    │
                                                                    ▼
                                                    bloom ─► ca_r/ca_b ─► reorder ─► glow_level ─► OUT
```

- **dot_thresh** — as `glow` rises, the surviving fraction of noise drops, so the solid body turns
  to scattered dots rather than simply fading.
- **dissolve_mix** — Cross TOP between solid silhouette and dotted silhouette, crossed by `glow`.
  Keeps a readable body at rest.
- **drift** — Displace weight scaled by `glow`; particles only wander once they're loose.
- **Chromatic aberration** — two Transform TOPs (scale ≈1.006 and ≈0.994) off the bloom, recombined
  channel-wise with two Reorder TOPs. R pushed out, B pulled in.

---

## Open

- Does a dissolved person read as *dispersed* or just as *gone*? If the latter, the particles need
  to hold a body-shaped cloud longer before rising.
- Two people in frame: one motion number can't tell them apart. Probably fine for the demo; note it
  as a known limit.
