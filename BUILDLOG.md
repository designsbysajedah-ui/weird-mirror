# Build Log

Honest notes, newest at the bottom. Rough is fine.

---

## Sept 9, 2026

Started this project.

---

## Sept 15, 2026 — Getting Claude connected to TouchDesigner

**Goal:** connect an MCP server so Claude can build inside my TD project directly instead of just
describing what to do.

**What we picked:** `8beeeaaat/touchdesigner-mcp` v2.0.0. There are several forks on GitHub; this one
has the two things that matter — it can read node *errors*, and it can send back a TOP's output as an
image, so Claude can actually see what it built instead of guessing.

**What I did:**

1. Downloaded `touchdesigner-mcp-td.zip` and `touchdesigner-mcp.mcpb` from the releases page.
2. Unzipped the first one and kept it *outside* `ixd415-starter/` so git doesn't try to track it.
   The folder can't be rearranged — the component uses relative paths.
3. Double-clicked the `.mcpb` to install into Claude Desktop.

**What broke:** the `.mcpb` refused to install — "Node.js >= 20.0.0" required. Claude had told me this
route didn't need Node; that was wrong. The bundle ships the server code but needs a Node runtime on
the machine to run it.

**Fix:** `brew install node`.

Two things worth noting for anyone repeating this:
- Homebrew sat at a `[y/n]` prompt and looked frozen. It wasn't — my `y` was typed but I hadn't
  pressed Enter. Terminal won't act on a line until you do.
- The install ended with two scary-looking lines about "Single Executable Application is disabled"
  and "Temporal support is disabled." Both are Homebrew noting optional Node features are off in its
  build. Irrelevant here.

Then quit Claude Desktop with Cmd+Q (not just closing the window) and reopened, so it picked up the
new Node. The `.mcpb` installed on the second try.

**Then it still didn't work,** which turned out to be correct behavior. The tools appeared on Claude's
side but calling one gave `ECONNREFUSED 127.0.0.1:9981` — the Claude half was installed, the
TouchDesigner half wasn't running. Two separate pieces; installing one doesn't do the other.

Opened TD, dragged `mcp_webserver_base.tox` in at `/project1/mcp_webserver_base`, left TD running.

**What finally worked:**

```
TouchDesigner Version: 099.2025.33230
Operating System: macOS 26.1
API Server Version: 1.5.0
MCP Server Version: 2.0.0
```

Claude then created a Noise TOP in my project, set its parameters, and read the rendered image back.
Full round trip.

**Time:** about 30 minutes, most of it waiting on Homebrew.

---

## Sept 15, 2026 — The motion detector

**Goal:** reduce the whole camera feed to a single number — *how much did this frame change*.
Everything in the spirit piece runs off that one value, so it's the right first thing to build.

**Final chain:**

```
cam_in (Video Device In)         camera, 1280x720 @ 30fps
  → mono (Monochrome)            color is irrelevant, throw it away
  → downsample (Resolution)      160x90, mono32float
  → framecache (Cache, size 8)   rolling buffer of recent frames
  → prev_frame (Cache Select)    index -4 = ~66ms back
  → frame_diff (Composite)       operand = difference, rgba32float
  → motion_amt (Analyze)         op = average → collapses to a single pixel
  → motion_chop (TOP to CHOP)    pixel becomes a number
  → pick_r (Select)              keep one channel, rename to "motion"
  → motion_amp (Math)            remap measured range to 0–1
  → motion_clamp (Limit)         clamp 0–1
  → motion_lag (Lag)             rise 0.02s, fall 0.35s
  → MOTION (Null)                the named handle everything else reads
```

Downsampling to 160x90 before the difference matters more than it looks. Webcam sensor noise makes
every pixel jitter slightly even in a still room; averaging 8 pixels into 1 cancels most of it.

### Measured calibration

| State | Raw value |
|---|---|
| Sitting still (normal breathing) | 0.00045 – 0.00053 |
| Ordinary movement | 0.0102 – 0.0130 |

About a 20× gap, which is a comfortable margin. Math CHOP maps `0.0005 → 0` and `0.012 → 1`, so
`MOTION` now reads ~0 still and ~1 moving.

**Don't inherit these numbers.** They're specific to this room, this light, this camera. Re-measure
in the classroom before the demo — that's a build step, not a formality.

### What went wrong

Nine separate things. Only three announced themselves with an error.

**1. First camera capture came back pure black.** Not permissions — the camera needs a second to wake
up. Captured again, fine.

**2. `topToCHOP` doesn't exist.** It's `toptoCHOP`, lowercase 't' in "to". Found it with
`[n for n in dir(td) if n.endswith('CHOP')]` — the general trick for this whole class of problem.

**3. Some operators take their source as a *parameter*, not a wire.** TOP to CHOP and Cache Select
both look like they should accept an input connection. They don't —
`inputConnectors[0].connect()` throws `IndexError: list index out of range`. You set `.par.top` and
`.par.cachetop` instead.

**4. The Feedback TOP was the wrong tool entirely.** First attempt used one to hold the previous
frame. It output black at a default 128x128 and ignored every resolution parameter we set. The reason:
a Feedback TOP's target has to be **downstream** of it — it exists for loops, not for "give me the
previous frame of this upstream node." Pointing it upstream gives an invalid target, which fails
silently as black.

**5. Black differenced against an image returns the image.** So bug 4 produced a network that looked
correctly wired, threw no errors, and reported a confident 0.58 that had nothing to do with motion.

**6. Parameter names are guesses until verified.** `analyzeTOP` has no `type` parameter — it's `op`.
`cacheselectTOP` has no `top` parameter — it's `cachetop`. Both failed *silently* inside a
try/except, leaving the node on a default that looked fine. Fix: print
`[p.name for p in node.pars()]` and read the real list.

**7. Cache Select's index counts backwards.** We set it to `1` for "one frame back." It kept reading
back as `0`. The parameter is clamped to a **maximum** of 0 and has no minimum — `0` is the newest
frame, `-1` is one back. Setting `1` silently clamped to `0`, so the chain was differencing the
current frame against **itself**. Always exactly zero.

And `0.0` is precisely what a correctly-working detector reads in a still room. The broken version and
the working version were indistinguishable while sitting still. Only waving exposed it.

**8. 8-bit quantization was hiding everything.** Once fixed, "still" still read a flat `0.0` — because
the difference image was stored as 8-bit, and anything subtler than 1/255 rounded to nothing. Setting
the TOPs to 32-bit float turned a flat zero into a usable `0.00045`.

**9. Camera fps vs project fps.** Readings alternated between a real value and exactly `0.0`, every
other sample. The webcam runs at **30fps**, TouchDesigner cooks at **60fps** — so on half the project
frames the camera hands over a *duplicate* image, and the difference between a frame and itself is
zero.

Fixed by comparing against a frame **4 back** (~66ms, guaranteed to be a genuinely new camera frame)
and adding asymmetric lag — rise 0.02s, fall 0.35s — so a stray duplicate can't drag the value down.

This one generalizes: **any time you difference frames, check your source fps against your cook rate.**

### The tell, every time

All the silent bugs were caught the same way: **compare the picture against the number.**

- `frame_diff` rendered pure black while `MOTION` insisted 0.545. Can't both be true.
- `MOTION` read a plausible 0.0, but the image during movement showed a clear ghost-outline of a
  person with a raised hand. Picture said motion, number said none.

None of these produced an error. All produced confident, wrong, reasonable-looking answers.

### Small gotcha

`time.sleep()` inside a script blocks TouchDesigner's cook, so sampling a value in a loop returns the
same number five times. You cannot measure change over time from inside one script — take separate
readings instead.

### Unexpected, and worth keeping

The raw `frame_diff` output — a glowing edge-outline of a moving body against pure black — already
looks like the piece we're trying to make. The debug view is the aesthetic. Revisit this before
layering anything decorative on top.

---

## Sept 15, 2026 — The state machine (first working interaction)

**Goal:** turn the motion number into behavior. Stand still → charge builds → spirit releases. Move →
it comes back.

**Decision: wrote this in Python, not CHOPs.** A Speed CHOP integrating a rate would work, but it
accumulates without bound — stand still for 60 seconds and the internal value hits 60, so moving again
takes forever to unwind. Clamping *after* the integration doesn't fix the integration. Doing it in a
`state_machine` Execute DAT keeps the state bounded, readable, and tweakable by editing four numbers
at the top of the file.

**How it works.** An Execute DAT fires `onFrameStart` every frame. It reads `MOTION`, updates a charge
value, and writes four channels into a `STATE` Constant CHOP:

| Channel | Meaning |
|---|---|
| `charge_s` | seconds of stillness accumulated (0–3) |
| `glow` | same thing normalized 0–1 — this is what will drive the visual |
| `released` | 0 or 1 |
| `state` | 0 = idle, 1 = charging, 2 = released |

The tunables, all at the top of the DAT:

```python
THRESH = 0.15    # motion above this counts as "moving"
HOLD   = 3.0     # seconds of stillness required before release
FILL   = 1.0     # charge gained per second while still
DRAIN  = 0.7     # charge lost per second while moving
```

`DRAIN` is deliberately slower than `FILL`. Twitch during the build-up and you lose a little progress,
not all of it. Forgiving rather than punishing — this is a design decision, not a technical one, and
it's the number most worth playing with once the visuals exist.

**It worked on the first try**, which after nine bugs in the previous session was suspicious enough
that it got tested properly rather than trusted:

```
hold still  →  state 1 (charging), charge climbing
5 seconds   →  state 2 (RELEASED), charge 3.0
wave a hand →  motion 1.0, charge 0.0, state 0 (idle)
```

Full cycle. Standing still in front of a laptop now causes something to happen and then undo itself.

Saved as `Projectixd415.2.toe`.

---

## Sept 15, 2026 — The glow, and why the desk test can't validate it

**Goal:** make `glow` visible. Silhouette outlined in light, brightening as the charge builds.

### The design problem that came first

`frame_diff` only shows you while you're *moving*. Stand still and it goes black — which is exactly
when the glow is supposed to appear. So the motion chain can't also be the silhouette source. They
need to be two separate things.

Solution: **background subtraction.** Capture what the room looks like empty, freeze it, and anything
that differs from that plate is a person.

### The chain

```
mono → sil_res (640x360, mono32float)
     → bg_plate (Cache, size 1, active toggled off to FREEZE the empty-room frame)
     → bg_ref (Cache Select)
body_diff (Composite difference: sil_res vs bg_ref)
     → diff_blur (Blur, size 7)
     → silhouette (Threshold)
     → body_edge (Edge)
     → glow_blur (Blur, size 9)
     → glow_mix (Composite multiply, against glow_tint constant — cyan-white)
     → glow_level (Level, brightness1 driven by expression op('STATE')['glow'])
     → OUT (Null)
```

The brightness expression is the whole point: `op('/project1/STATE')['glow']`. The charge value drives
the light directly. Hold still, the outline fades up. Move, it drops.

### Bugs

**10. The Threshold TOP was thresholding the alpha channel.** Output was pure white no matter what
threshold we set. Sampling the actual pixels showed RGB at 0.017–0.11 — far below the threshold — but
**alpha at 1.0 everywhere**. The `rgb` and `alpha` parameters are menus, not booleans, and alpha was
following along. White frame, no error.

Caught by sampling pixels with `top.sample(x=, y=)` instead of trusting the preview. Note
`.sample()` returns a plain tuple, not an object with `.r` — `c.r` raises `AttributeError`.

**11. Webcam auto-exposure destroys background subtraction.** With `autoge` on, stepping into frame
made the camera re-expose the *entire* image, so every pixel differed from the plate — not just the
person. The empty-room difference was reading 0.02–0.05 of pure noise.

Turning off `autoge` and `autowb` on the Video Device In TOP dropped the empty-room difference to
**exactly 0.0** across the frame. Enormous improvement, one parameter.

**12. Sub-pixel camera shake destroys it too.** After re-capturing the plate, the output traced every
high-contrast edge in the room — door frame, picture frame, bed — in a thin glowing line. That
signature (edges only, not regions) means the camera moved by a fraction of a pixel. Laptop sitting on
a soft surface; standing up and sitting back down shifted it.

A blur before the threshold helps — thin edge artifacts smear out while large body regions survive —
but it doesn't fix the cause.

### Honest status

The pipeline is **complete and correct end to end**: motion → state → charge → brightness → rendered
outline. It renders. Cyan glow on black, driven by stillness.

The silhouette **quality** is not validated, and can't be from this setup. Background subtraction needs
a camera that does not move and lighting that does not change. A laptop on a bed in a dark room fails
both.

**This is a test-rig limitation, not a design flaw — and the real installation avoids it entirely.**
With a projector on a wall and the camera fixed beside it, a person standing in the beam is a *dark
shadow against bright light*. That silhouette is a plain luminance threshold. No background plate, no
freeze, no exposure lock, no drift. The hardest part of tonight's session is a problem the actual
piece doesn't have.

### What this means for the build

- Camera must be on a tripod or otherwise fixed. Non-negotiable. Anything that can be nudged will be.
- Lock `autoge` and `autowb` and leave them locked.
- Expect to swap `bg_plate` / `body_diff` for a single Threshold on `mono` once the projector is up —
  simpler network, better result.
- Re-measure every threshold in the real space.

Saved as `Projectixd415.3.toe`.

**Next:** get the projector pointed at a wall with the camera fixed beside it, and re-test with the
shadow-based silhouette. That's the first test that actually resembles the piece.

---

## Sept 15, 2026 — Variant B: stillness dissolves you into particles

**Goal:** a second .toe testing the *original* reading of the interaction. Variant A (movement
charges, a spirit copy peels off) stays in `Projectixd415.22.toe` and is untouched. This file asks
the opposite question: hold still and **your own body** grains apart into particles.

Reference for the look: @stormypyeatte's TouchDesigner hand-tracking reel — a 3D-scanned artichoke
flower as a soft glowing point cloud with chromatic aberration. Borrowing the aesthetic, not the
input. Hand tracking is scoped as a separate experiment.

Saved as `Projectixd415-dissolve.1.toe`.

### The design difference that matters

In Variant A there are two things on screen — your body, plus a spirit that leaves it. In B there's
**one** thing that stops being solid. Fewer objects, so it should read at a distance, and it removes
the "wait, which one is me?" question.

### The problem I hit in the first ten minutes

The silhouette in the current file isn't from background subtraction anymore — it's built from
accumulated *motion* (`mo_diff → mo_blur → mask_smooth` with a decaying feedback loop). That's a
smart fix for Variant A, where movement is the trigger.

It is exactly backwards for Variant B. **If the mask is made of motion, standing still erases the
body** — the thing that's supposed to dissolve deletes itself first, for the wrong reason.

Fix: a `body_hold` Cache TOP latching the last mask, with its Active parameter on an expression:

```
op('/project1/STATE')['glow'] < 0.02
```

So while you're moving or idle, the cache tracks you live. The instant dissolve starts, it freezes
the shape you were last seen in, and *that* is what grains apart. Accidentally a better idea than
the one I set out to build — what disperses is your afterimage, not you.

**Still unverified.** The latch can't be tested until there's a real silhouette to latch.

### The dissolve

```
dot_noise (120x68 random, mono)
  → dot_up (Resolution 960x540, filter = nearest)      chunky 8px dots, not 1px static
  → dot_thresh (Threshold, comparator "less")
       threshold = 0.02 + 0.93 * STATE['glow']
  → dots (Composite multiply against body_src)
  → drift (Displace, weight = 0.005 + 0.055 * glow)
  → rise (Transform, ty = STATE['rise'])
  → body_lvl (Level, brightness = 0.25 + 0.75 * glow)
  → … trails → bloom → chroma → glow_level → OUT
```

One idea does all the work: **the threshold rises with the dissolve amount.** At `glow` 0 the
threshold is 0.02, so nearly every noise pixel passes and the body is solid. At `glow` 1 it's 0.95
and only the top ~5% survive — a sparse scatter of points in the shape of a person. No separate
"solid" and "particle" layers to crossfade, no particle system. One parameter, one expression.

Two things that mattered more than expected:

- **Noise resolution is the dot size.** At 960x540 it's per-pixel TV static. Dropping the noise to
  120x68 and upscaling with a *nearest* filter gives actual dots you can see.
- The Threshold TOP comparator is still inverted from the naive reading — `less` outputs white where
  the source is **above** the threshold. Verified again rather than trusted.

### The look

- `bloom` (threshold 0.12, radius 0.004–0.06, intensity 1.6) turns each hard dot into a soft point
  of light. This is what makes it read as a point cloud rather than as noise.
- Chromatic aberration in three nodes: `ca_r` (Transform, scale 1.008) and `ca_b` (scale 0.992) off
  the bloom, recombined by a single `chroma` Reorder TOP — red from input1, green from input2, blue
  from input3. A Reorder TOP takes **four** inputs, which I didn't know; the whole effect is one node
  instead of the stack of tinted composites I'd planned.
- **Turning the color ramp from horizontal to vertical was the single biggest visual improvement of
  the session.** Horizontally, a standing body only samples the narrow middle of the ramp, so
  everything came out one muddy olive. Vertically it samples the entire spectrum head to foot —
  cool blue at the top, embers at the bottom. Same nodes, same everything, one menu.

### Bugs

**13. Transform TOP scale parameters are `sx`/`sy`, not `scalex`/`scaley`.** Threw
`'td.ParCollection' object has no attribute 'scalex'`. Same lesson as bug 6, learned again: print
`[p.name for p in node.pars()]` first.

**14. Sampling in a loop with `time.sleep()` — again.** Got twelve identical readings of `0.858` and
briefly believed the motion chain had frozen. It hadn't; the sleep blocks TD's cook. This is already
written down in this log from two sessions ago, and I still did it.

### Bench mode

Since the current desk rig can't produce a usable silhouette, there's now a `test_body` Circle TOP
and a `body_src` Switch TOP:

- `body_src` index **1** = bench shape, develop the look with no camera
- `body_src` index **0** = live silhouette

Worth keeping permanently. The look got tuned in about ten minutes once it stopped depending on
whether the camera could see a person.

### Verified

Sweeping `glow` by hand with the state machine frozen:

| glow | Output |
|---|---|
| 0.0 | Solid body |
| 0.5 | Body full of holes, edges breaking up |
| 0.85 | Body-shaped cloud of separate points |
| 1.0 + rise 0.55 | Cloud lifted, leaving the top of the frame |

Then re-enabled the state machine live: stillness charged to `state 1, glow 0.33`, movement drained
it back to `state 0, glow 0.0`. Loop confirmed. No node errors.

### Not done / known bad

- **Calibration is wrong and will be wrong again.** The old range (0.0005–0.012) pegged `MOTION` at
  1.0 permanently in a bright room with a handheld laptop; measured raw was 0.002 still / 0.09
  moving, so it's temporarily set to 0.003–0.05. These numbers are disposable. Re-measure in the room.
- The `body_hold` latch is untested, per above.
- The solid (`glow` 0) state looks muddy — a flat colored slab. In the real install that's a person's
  shadow filled with color, so it may not matter, but it needs eyes on it in the room.
- Two people in frame still share one motion number. Known limit; probably fine for the demo.

---

## Sept 15, 2026 — Variant C: pivoting to a hand-tracked crystal

**The pivot, stated plainly so it's on the record:** Variants A and B are body pieces on a
projector. Variant C is an object you manipulate with your hands, on a screen. The projector — the
original reason for choosing this medium — is what gets dropped. Either C ends up back on a
projector later, or it lives alongside A/B rather than replacing them. Full comparison table in
`docs/crystal-handtracking-plan.md`.

File: `Projectixd415-crystal.1.toe`.

**What got built:** a quartz crystal — three Tube SOPs merged, run through a Facet SOP with `unique`
and `cusp` on for hard faceted shading — as input 2 of `shape_pick` inside the `form` Geometry COMP.
Enchanted palette: deep violet → magenta → cyan → white, vertical.

**The move that saved the session:** the interaction was wired against a **mouse stub** first. A
`mouse_in` CHOP feeding `HAND` (`hand_x`, `hand_y`, `grip`), so the whole rotate-and-explode loop was
testable before MediaPipe existed on this machine. Swapping in real hand data later was a two-wire
change.

`state_machine` gained a `MODE` at the top — `'hand'` or `'stillness'` — with both writing the same
`STATE` channels, so nothing downstream had to know which one was driving.

**The constraint that shaped everything:** TouchDesigner 2025.33230 on macOS has **no native hand
tracking.** The operator list shows `bodytrackCHOP`, which is Windows/NVIDIA only. The working route
is torinmb/mediapipe-touchdesigner — free, runs on Mac.

### Bugs

**15. `sin()` is not defined in parameter expressions** — it's `math.sin()`. A bad expression on the
ramp's Phase blanked the entire output, and the error lived on the ramp with nothing visible
downstream. When the output goes black, walk *upstream* and check parameter expressions, not just
node errors.

**16. Sphere SOP defaults to `type = prim`** — a single point. A Noise SOP downstream then displaces
nothing and the sphere stays perfectly smooth, with no error anywhere. Set `type = mesh`.

**17. Noise SOP keeps the original normals** (`keepnormals` on by default), so even correctly
displaced geometry still shades as if smooth.

**18. Facet SOP's normals parameter is `postnml`**, not `computenormals`. And Switch **SOP** uses
`input` while Switch **TOP** uses `index` — same idea, different word, one network apart.

---

## Sept 16, 2026 — Real hands

**Goal:** replace the mouse stub with MediaPipe and make the crystal respond to an actual hand.

Downloaded torinmb/mediapipe-touchdesigner **v0.5.3** (Aug 2026), `release.zip`, 173MB. Unzipped to
`~/Desktop/mediapipe-touchdesigner/`. Both `MediaPipe.tox` and `hand_tracking.tox` are loaded as
**external tox references, not embedded** — the engine tox alone is 172MB and embedding it makes
every save crawl.

**Wiring:**

```
MediaPipe 'hands' out DAT → mp_hands (Select DAT) → hand_tracking COMP
  hand_xy   (Select from hand_tracking/helpers: h1:pinch_midpoint x, y)
  hand_fit  (Math, 0..1 → -1..1)
  hand_on   (h1:hand_active)
  hand_grip (Select from hand_tracking/gestures: h1:Closed_Fist)
    → hand_merge → hand_lag → HAND
```

`mouse_in` is left in the network, disconnected, as a fallback.

Confirmed working the same evening: `hand_present` 1.0, spin accumulating, tilt responding. Camera
permission was already granted, which I'd expected to be the hard part and wasn't.

### Gotchas

**19. TouchDesigner's bundled Python has no CA certificates.** Any `urllib` HTTPS request dies with
`CERTIFICATE_VERIFY_FAILED`. Workaround: shell out to `curl` via `subprocess` — and read the output
as bytes then `.decode('utf-8')`, because subprocess text mode assumes ascii and chokes.

**20. A parameter's expression MODE can silently revert to CONSTANT while the expression text stays
stored.** The parameter reads 0, the effect dies, and there is **no node error** — `par.expr` still
shows the right expression, so it looks fine. Check `par.mode.name`, not `par.expr`. Fix with
`par.mode = ParMode.EXPRESSION`.

This is the third variation on the same theme in this log: TouchDesigner's failure mode is almost
never an error message. It's a plausible-looking wrong value.

---

## Sept 16, 2026 — Making it feel like an object

**Spin became momentum-based.** Hand travel adds angular velocity; friction bleeds it off. So a flick
keeps the crystal turning instead of it snapping to wherever your hand is. `STATE` grew to 9 channels
to carry `spin` and `tilt`.

**Then the pinch scheme.** Chose pinch-to-grab and turning in place on three axes, rather than
dragging the crystal around the frame. Pinch grabs and drags it on spin/tilt/twist with momentum on
release; rolling the pinched wrist rolls it; an open-hand sweep shoves the spin; a closed fist
shatters it and suspends rotation. `STATE` to 10 channels for `twist`.

### Calibrating the pinch — the method is the useful part

`PINCH_ON 0.06 / PINCH_OFF 0.10` were guesses and felt wrong. Measuring them properly is awkward
because `time.sleep()` blocks TD's cook (bug 9 and bug 14, third appearance) and a single sample can't
capture a range.

**What worked:** a temporary Execute DAT storing a running min/max of pinch distance every frame while
`hand_active` was 1. Then just use your hand normally for a while and read the numbers off.

From 929 recorded frames of my own hand:

| Gesture | Pinch distance |
|---|---|
| Firm pinch | 0.012 |
| Loose pinch | ~0.067 |
| Open hand | 0.234 |

Set `PINCH_ON 0.07` / `PINCH_OFF 0.13`, with hysteresis so it doesn't chatter at the boundary. Worth
reusing for any continuous gesture value.

### Too sensitive. Twice.

First tuning pass felt like the crystal was swatting away from me. Damped it, tried again, still too
twitchy. Second pass:

| | First guess | Now |
|---|---|---|
| `DRAG_GAIN` | 260 | **70** |
| `SWEEP_GAIN` | 420 | **100** |
| `TWIST_GAIN` | 1.10 | **0.32** |
| `VEL_MAX` | 22 | **6** |
| `FRICTION` | 0.94 | **0.86** |

Plus `DEADZONE 0.005` and `TWIST_DEAD 1.2` so tracker jitter can't creep the crystal while my hand
sits still.

**Lesson for next time: start gesture gains low.** Heavy and slow reads as a solid object you're
moving. Fast and responsive reads as a glitch. My first guesses were roughly 4× too high across the
board.

### Up and down should move it, not tilt it

Changed my mind on the mapping: vertical hand position now drives **position**, not rotation.
`STATE` gained an 11th channel `posy` driving `crystal_model.ty` / `form.ty`, eased toward hand height
while a hand is present and drifting back to centre when it leaves. `LIFT_SPAN 1.10`,
`LIFT_SIGN -1.0` (flip if inverted), `LIFT_EASE 0.08`, `LIFT_RETURN 0.02`. The `tilt` channel is now
unused and parked at 0. Horizontal still spins, wrist roll still twists.

### The performance bug that was hiding everything

Hand detection was landing on **10% of frames** and `Closed_Fist` had never once fired. I'd assumed
the gesture recognizer was just unreliable.

It wasn't. After a Reset, the MediaPipe component had come back with **face landmarks, face detection
and pose detection all enabled** alongside gestures, at 1280x720:

```
detectTime      193 ms
realTimeRatio   5.85
isRealtime      0
```

It was doing four kinds of detection to answer one question. Turning off every detector except
gestures and dropping to 640x360:

```
detectTime      ~0 ms
isRealtime      1
```

Detection went to **45% of TD frames**, which is effectively the ceiling — the camera runs 30fps while
TD cooks at 60, so ~50% is the maximum possible. `Closed_Fist` then registered at 0.95 confidence,
pinch reached 0.0112, `posy` swept -0.58 to 1.04, spin covered the full 0–358. Whole interaction
verified working.

**Note:** parameter changes on the MediaPipe COMP need a **Reset pulse** to reach the browser engine.
Changing them without a reset does nothing and gives no indication of that.

### The white background

The output had a milky white field behind the particles. Cause: `glow_mix` (Composite, multiply) let
the colour ramp show through wherever the particle layer had alpha 0, and `bloom` then blew that
mid-tone to pure white. Fix: a `glow_opaque` Reorder TOP forcing alpha = 1, between `glow_blur` and
`glow_mix`.

### Failure mode to check before a demo

The MediaPipe engine can restart on its own — its port changes and it comes back with a **black camera
feed and no detections at all.** Nothing errors. The fix is a **Reset pulse** on the MediaPipe COMP.
If nothing responds, check `MediaPipe/video` and the `realTime` CHOP first, before touching any of
the interaction code.
