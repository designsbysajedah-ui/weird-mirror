# weird-mirror

An interactive installation built in TouchDesigner for **IXD 415 — Emerging Tech**.
Sajedah Andalsi, Fall 2026.

A camera watches. The piece responds to a body — or, in the current variant, to a hand — and the
viewer figures out the rule without being told. No instructions, no labels, no buttons.

---

## Where it stands

Three variants have been built, each one a separate `.toe`. They share a state machine and most of
the visual chain, which is why they can diverge cheaply.

| Variant | Input | What happens | File |
|---|---|---|---|
| **A — Spirit** | whole body, webcam motion | Movement charges a build-up; a glowing copy of you peels off and rises | `Projectixd415.22.toe` |
| **B — Dissolve** | whole body, webcam motion | Hold still and your own silhouette grains apart into a drifting point cloud | `Projectixd415-dissolve.1.toe` |
| **C — Crystal** | hands, MediaPipe tracking | Pinch to grab and spin a crystal; open-hand sweep shoves it; closed fist shatters it | `Projectixd415-crystal.*.toe` |

**Currently active: Variant C.** Hand tracking is working end to end — pinch calibrated against 929
recorded frames, rotation damped to feel like a heavy object rather than a twitchy one, and the
MediaPipe engine tuned from 10% detection to its practical ceiling of ~45%.

**Open decision:** A and B are projector pieces; C is a screen piece. A projection-mapped work was
the original reason for choosing this medium, so C either goes back onto a projector or lives
alongside A/B rather than replacing them.

---

## How it works, in one paragraph

Everything runs off a small number of values written each frame into a `STATE` Constant CHOP by a
single Python Execute DAT. In the body variants that's a motion number reduced from the whole camera
frame; in the crystal variant it's hand position, pinch state and gesture confidence. Every visual
node downstream reads `STATE` through parameter expressions, so the interaction design lives in one
readable file of tunable constants rather than being scattered across the network.

---

## Repo contents

```
BUILDLOG.md    the build log — what broke, what I asked Claude, what finally worked
docs/          specs, plans, and reference research
```

| Doc | What's in it |
|---|---|
| [`docs/interaction-spec-spirit.md`](docs/interaction-spec-spirit.md) | The original interaction spec — states, timing, physical setup |
| [`docs/reference-interactive-projection-works.md`](docs/reference-interactive-projection-works.md) | Reference works, sorted by what one projector and one laptop can actually reach |
| [`docs/classroom-test-checklist.md`](docs/classroom-test-checklist.md) | What to bring and what to measure when testing in the real room |
| [`docs/dissolve-variant-plan.md`](docs/dissolve-variant-plan.md) | Variant B plan — state machine and visual chain |
| [`docs/crystal-handtracking-plan.md`](docs/crystal-handtracking-plan.md) | Variant C plan — the pivot, and hand tracking on macOS |
| [`docs/particle-controls.md`](docs/particle-controls.md) | Every knob worth turning, which node it's on, and what to try |
| [`docs/hand-gesture-reference.md`](docs/hand-gesture-reference.md) | Every channel the MediaPipe plugin exposes, and which are mapped |

**`.toe` files are not committed here** — they're large and binary, and per the assignment they stay
off the repo. They live on the build machine.

---

## Running it

- TouchDesigner **099.2025.33230**, macOS 26.1, Apple Silicon (M4 MacBook Air)
- Hand tracking: [torinmb/mediapipe-touchdesigner](https://github.com/torinmb/mediapipe-touchdesigner)
  **v0.5.3** — TouchDesigner has no native hand tracking on macOS. `MediaPipe.tox` and
  `hand_tracking.tox` are referenced as **external** tox files, never embedded (the engine tox alone
  is 172MB).
- Claude is connected to TouchDesigner through
  [8beeeaaat/touchdesigner-mcp](https://github.com/8beeeaaat/touchdesigner-mcp) v2.0.0, which is how
  most of the network gets built and inspected. Setup notes are the first entry in the build log.

Calibration numbers in these docs are **specific to the room they were measured in** and must be
re-measured on site. That's a build step, not a formality.
