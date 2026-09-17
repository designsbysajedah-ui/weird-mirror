# Classroom Test — Checklist

First test in the actual space. Goal: replace background subtraction with the shadow threshold, and
re-measure everything in the real room.

---

## Bring

- [ ] Laptop + charger
- [ ] Projector + its power cable
- [ ] **Video adapter.** M4 Air has Thunderbolt/USB-C. If the projector is HDMI, you need USB-C → HDMI.
      This is the single most likely thing to end the session before it starts.
- [ ] Something to fix the camera in place — small tripod, clamp, or tape. **Not** a laptop on a soft
      surface. Camera movement is what broke last night's test.
- [ ] Tape measure (throw distance, image size) — phone is fine

## Set up

- [ ] Lights off
- [ ] Projector pointed at a wall. Note the throw distance and how big the image is.
- [ ] Camera fixed **beside the projector, facing the same wall** — it watches the projection surface,
      not the person. The shadow is already the right shape in the right place.
- [ ] Laptop: set `preview_window`'s `display` parameter to the projector, turn on fullscreen

## Measure (don't skip — last session's numbers are void here)

- [ ] Stand in the beam. Check `cam_in` — can you see a clear dark shadow against bright projection?
- [ ] Sample `mono` to find the luminance split between shadow and lit wall
- [ ] Re-run the motion calibration: still vs moving, new `fromrange` values for `motion_amp`
- [ ] Re-set `THRESH` in `state_machine` if the new range demands it

## Switch the silhouette source

The background-plate chain (`bg_plate` / `bg_ref` / `body_diff` / `diff_blur`) gets replaced by a
single Threshold on `mono` — dark = body. Simpler network, no drift, no exposure lock needed.

Keep the old nodes around until the new path is proven.

## Watch for

- **Camera seeing the projection.** As spirits accumulate, the wall gets brighter and the threshold
  may drift. Bodies are dark, spirits are bright — that separation should hold, but verify it.
- **Projector fan noise** — irrelevant for this piece (no mic), but note it for the demo.
- **Where a person naturally stands.** If they block the beam at the wrong distance the shadow is
  huge or tiny. Find the sweet spot and note it.

## Questions the room will answer

- Can the room actually go dark enough for this projector?
- How far back can people stand before the shadow leaves the frame?
- Is there a wall corner or surface with real geometry worth mapping onto, or only flat wall?
- Where would a second projector go if one becomes available?

---

Take photos of the setup. They're the start of your documentation video, and you'll want to remember
the geometry that worked.
