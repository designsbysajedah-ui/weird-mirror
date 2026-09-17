# Interaction Spec — "Spirit" (working title)

Draft, Sept 2026. Concept by Sajedah. One projector, one webcam, one laptop.

---

## The loop

A person walks into a projected field of light and casts a shadow. If they hold still, they begin
to glow. Hold still long enough and a luminous copy of them separates and drifts away. Move, and it
comes back.

No instructions. The glow is the instruction.

---

## State machine

Everything runs off **one number**: how much the camera image changed since the last frame.

| State | Entered when | What the viewer sees |
|---|---|---|
| `IDLE` | motion above threshold | Ordinary shadow on the wall |
| `CHARGING` | motion drops below threshold | Glow builds along the silhouette edge, rising with the timer |
| `RELEASED` | timer reaches ~3s | Silhouette snapshot detaches, drifts, tinted and glowing |
| `RECALL` | motion spikes while spirit is out | Spirit drifts back toward the body and reabsorbs |

Moving during `CHARGING` **drains** the glow rather than snapping it off — drain slower than fill,
so it feels forgiving rather than punishing.

---

## Critical timing note

Three seconds is a long time for a stranger to stand still with no feedback. The glow must begin
within roughly **300ms** of stillness or nobody will ever discover the piece — they'll pause briefly,
see nothing, and walk away. Early glow is faint; the last second is the payoff.

---

## Physical setup

Laptop and projector sit side by side, both facing the wall. Person stands in the beam.
**The webcam points at the wall, not at the person.**

Why: the camera then sees the shadow directly — a dark human-shaped region against bright
projection. That silhouette is already the right shape in the right place, so there is no mapping
between "where the camera saw them" and "where to project." The alignment problem solves itself.

Threshold logic stays valid as spirits accumulate, because spirits are *bright* and bodies are *dark*.

---

## TouchDesigner sketch

- Video Device In TOP → Monochrome → **Threshold TOP** = body mask
- Cache TOP (1 frame) + Difference + **Analyze TOP (sum)** = the single motion number
- Motion number → Logic/Timer CHOP = the state machine
- Body mask edge + timer value = glow ramp
- On release: freeze the mask to a texture, advect with **Feedback TOP + Noise TOP** displacement
- Tint per spirit; composite over the base field

Nothing here needs blob tracking or a depth sensor.

---

## Open questions

- Where does the spirit go? Ceiling is out with one projector (keystone is brutal) — likely drifts up
  the wall and out of frame.
- Do multiple people each get a spirit, or does the piece only accept one at a time?
- Do spirits persist and accumulate over a session, or fade?
