# Interaction Spec: Plasma

The current direction, as of Oct 1. Replaces the spirit, dissolve and crystal specs.

A visitor steps up to a projected orb of glowing pastel plasma, uses their hands to enter it and
move through it, and leaves a trace behind when they go.

Working file: `Projectixd415.44.toe`. Model: *Evanescent Plasma* by Tycho Magnetic Anomaly
(Sketchfab, CC-BY 4.0).

---

## Requirement checklist

| Requirement | How the piece covers it | Status |
|---|---|---|
| A mirror that reflects the person | Fingertips draw flowing ribbons in the plasma, exactly where the visitor's hands are | Built |
| Affordances | Idle movement shows it's alive; the orb "notices" a raised hand; plucked strings make sound | Planned |
| Actions | Rotate, zoom, colour, warp, and later pluck | Built (pluck planned) |
| Feedback | Ribbons change colour and brightness with each gesture; the scene responds | Built |
| Journey | Idle → Enter → Explore → Exit, with a trace left behind | Planned |

---

## The journey

### 1. Idle (nobody there)

- The orb is seen from **outside**, slowly turning, breathing in scale and drifting through its
  colours.
- Purpose: show that it moves, so people come closer.
- *Change from earlier:* this reverses the "freeze completely with no hands" decision. A still
  screen doesn't invite anyone.

### 2. Enter (start)

- **Trigger:** an open palm held up for about 1 second.
- The orb brightens as the hand is noticed, then the camera glides from outside to **inside** the
  orb.
- Idle motion eases out and control passes to the visitor.

### 3. Explore (middle)

| Gesture | Action | Feedback |
|---|---|---|
| Open palm, side to side | Rotate (a flick keeps spinning) | Ribbons turn cyan |
| Open palm, toward / away from camera | Zoom in / out | Ribbons turn cyan |
| Pinch + wrist twist | Shift the colours | Ribbons turn pink and brighten |
| Fist | Warp burst | Ribbons flash white |
| Fingertip crosses a string *(planned)* | Pluck a note | The string flashes; a chime plays |

### 4. Exit (end)

- **Trigger:** clap both hands together, or lower your hands for about 3 seconds.
- A clap is the deliberate ending: a soft burst, the rings fold shut, and the camera drifts back
  outside.
- Lowering your hands gives a quieter version of the same exit.
- **Trace (optional):** the visitor's ribbons are woven into the orb and fade over the next few
  minutes, so the next person sees that someone was here.

---

## Two hands

- **Either hand can do any gesture.** Right now gestures only read the first hand the tracker sees
  (ribbons already draw from both).
- If both hands make the same gesture, their movements combine. Opposite movements cancel out
  instead of fighting.
- **The clap is the only two-hand gesture.** Needing both hands makes it feel deliberate, which
  suits an ending.

---

## Plucking the strings (planned)

- **Detection:** sample the plasma's brightness under each fingertip on screen. When a fingertip
  crosses a bright line, the brightness spikes and drops, and that counts as a pluck. Check the
  whole path between frames so fast swipes don't skip over lines.
- **Sound:** short harp or chime notes on a pentatonic scale, so any combination sounds good.
  Pitch follows fingertip height; left/right stereo position follows hand position.
- **Feedback:** the plucked line flashes or ripples.
- **Needs:** speakers in the classroom.

---

## Build order

1. Idle movement + Enter / Exit flow
2. Either-hand gestures + clap
3. Plucking + sound
4. Trace (if time)

## Open questions

- How long should a visit last before it eases toward the exit on its own, if at all?
- How long should a trace stay: one visitor, several, or the whole session?
- Should each visitor get their own colour in the trace?
- Speakers: what does the classroom have?
