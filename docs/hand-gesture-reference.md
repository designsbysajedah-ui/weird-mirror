# Hand tracking — what's available, what's mapped

MediaPipe TouchDesigner plugin v0.5.3, running on the MacBook Air camera. Confirmed working
Sept 16 2026.

---

## Currently mapped

| Input | Effect |
|---|---|
| Hand moves **left/right** | Spins the crystal. Momentum-based — a flick keeps it turning, friction settles it. |
| Hand moves **up/down** | Tilts the crystal. Eased, returns to level when the hand leaves frame. |
| **Closed fist** | Shatters the crystal into particles. Spin is suspended while clenched. |
| Open the hand | Particles pull back together. |

Tunables live at the top of the `state_machine` DAT:

```python
SPIN_GAIN     = 420.0   # degrees of spin per unit of hand travel
SPIN_FRICTION = 0.94    # 1.0 = spins forever, lower = settles sooner
SPIN_MAX      = 22.0    # degrees per frame ceiling
TILT_RANGE    = 55.0    # max degrees of X tilt
TILT_EASE     = 0.10
EXPLODE_FILL  = 1.9     # how fast a fist shatters it
EXPLODE_DRAIN = 1.3     # how fast it reassembles
```

---

## Every gesture the plugin recognizes

Per hand, as a 0–1 confidence value, in `hand_tracking/gestures`:

| Gesture | Channel |
|---|---|
| Closed fist | `h1:Closed_Fist` ← *in use: shatter* |
| Open palm | `h1:Open_Palm` |
| Pointing up | `h1:Pointing_Up` |
| Thumb up | `h1:Thumb_Up` |
| Thumb down | `h1:Thumb_Down` |
| Victory / peace | `h1:Victory` |
| "I love you" sign | `h1:ILoveYou` |
| Nothing recognized | `h1:None` |

`h2:` prefixes give the same set for a second hand.

### Continuous values — often more useful than discrete gestures

In `hand_tracking/helpers`:

| Channel | What it gives you |
|---|---|
| `h1:hand_active` | 0 or 1 — is a hand in frame ← *in use* |
| `h1:hand_velocity` | How fast the hand is moving |
| `h1:pinch_midpoint:x` / `:y` / `:z` | Position between thumb and index ← *in use: spin + tilt* |
| `h1:pinch_midpoint:distance` | How far apart thumb and index are — a **continuous** pinch |
| `h1:pinch_midpoint:rotation` | Angle of the pinch |
| `hand_distance` | Distance between two hands |
| `h1:Leftness` / `h1:Rightness` | Which hand this is |

There's also `normalized_data` with all 21 landmarks per hand (wrist, and four joints per finger)
in x/y/z, if something needs finer control than the presets.

---

## Design notes

**Discrete gestures are on/off; pinch distance is a dial.** A fist either registers or doesn't, so
the shatter has no in-between — it's a switch, and the `EXPLODE_FILL` rate is what creates the
sense of a gradual break-up. Mapping `pinch_midpoint:distance` to the dissolve instead would let
someone hold the crystal *half* shattered and feel the material give as they squeeze. That is
probably the more interesting interaction and is worth trying.

**Two hands are available and unused.** `hand_distance` between two hands is an obvious candidate
for scale — pull your hands apart and the crystal grows, or the particles spread wider.

**Gesture recognition needs a reasonably clear view.** Detection drops if the hand is edge-on to
the camera, partly out of frame, or badly backlit. Worth knowing before a live demo: the lighting
that makes the *projection* look good is not necessarily lighting that a webcam can track hands in.

**One conflict to watch.** Spin and shatter both use the same hand. Clenching mid-spin currently
freezes the spin, which reads fine, but if it feels abrupt in testing, splitting them across two
hands — left spins, right shatters — is a clean fix.
