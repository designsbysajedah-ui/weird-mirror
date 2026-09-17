# Reference: Interactive Projection Works

Research pass, Sept 2026. Sorted by what one projector + one webcam + a MacBook Air can actually reach.
Companion artifact: "One Projector, One Camera".

---

## Within reach — current kit

### Text Rain — Camille Utterback & Romy Achituv, 1999
Letters fall and land on anything darker than a brightness threshold. Lift your arms, catch them.
The letters spell a poem about bodies and language.

- **Technique:** luminance threshold on live camera feed
- **In TD:** Video Device In TOP → Threshold TOP → particles colliding with dark pixels
- **Read:** literally the occlusion trick. Most imitated interaction in the medium. Build as a study.
- https://camilleutterback.com/projects/text-rain/

### Shadow (Screen Series) — Scott Snibbe, 2002
Cast a shadow on a white rectangle. Step out and the shadow replays without you, fading each pass.

- **Technique:** frame buffer + decay. No tracking at all.
- **In TD:** Cache TOP or Feedback TOP with falloff
- **Read:** cheapest technically, strongest conceptually. Fallback option if demo day goes wrong.
- https://www.snibbe.com/art/shadow

### Shadow Monsters — Philip Worthington, 2004 (MoMA collection)
Your shadow sprouts teeth, tendrils, fur. Hand shapes become creatures.

- **Technique:** blob contour detection on the silhouette outline
- **In TD:** Blob Track TOP → contour points drive geometry grown along the edge
- **Read:** tracking is a weekend; creature design is the semester. Original built in Processing + BlobDetection.
- https://www.moma.org/collection/works/110196

---

## Stretch — needs built surface, mount, or better tracking

### Boundary Functions — Scott Snibbe, 1998
Floor projection. Lines appear between people, partitioning the floor into personal territories.
Nothing happens with one person.

- **Technique:** overhead blob tracking → Voronoi partition
- **In TD:** ceiling camera → Blob Track TOP → Voronoi in GLSL TOP
- **Read:** math is a known shader; the hard part is physical (overhead mount in a room we don't control).
  Solves the "requires two people" problem.

### Hakanaï — Adrien M & Claire B, 2013
Dancer inside a scrim cube, projection responding to her movement.

- **Technique:** mapped volume + performer-driven simulation
- **In TD:** kantanMapper corner-pin onto real geometry, driven by tracking
- **Read:** full cube needs multiple projectors; ONE FACE does not. Hang a scrim or use a wall corner.
  Closest match to the projection-*mapping* portfolio goal.
- https://www.arch2o.com/adrien-m-claire-b-hakanai/

---

## Steal the idea — out of budget

### Under Scan — Rafael Lozano-Hemmer, 2005
Pedestrian shadows reveal video portraits of strangers who wake and look at you.

- **Transferable mechanic:** the shadow is a *lens*, not a hole — a second layer visible only where
  light is blocked. Runs on current kit: Composite TOP + mask.
- **Out of range:** square-scale rigging, hundreds of commissioned portraits.
- https://www.lozano-hemmer.com/projects.php?keyword=shadows

### Architectural mapping (Luftwerk / Fallingwater type)
Building geometry traced in light. Eight projectors, surveyed to the centimetre.

- **Read:** included as a warning. This is what people picture when they hear "projection mapping."
  Chasing it with one dim projector produces a weak imitation. A small object mapped precisely beats
  a building mapped badly.

---

## Takeaway

None of the top three explain themselves. No instruction card, no "wave here." A person walks into a
beam, sees their shadow do something it shouldn't, and learns the rule in four seconds using a body
part they've had their whole life.

That's the bar — and it's the argument against projected buttons. A button has to be labelled.
A shadow doesn't.
