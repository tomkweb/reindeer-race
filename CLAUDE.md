# Reindeer Race — Project Context (CLAUDE.md)

## What this project is
A 3D browser reindeer racing game — nine named reindeer (Dasher through
Rudolph) with elf jockeys race a dirt oval with broadcast-style cameras,
photo finishes, and engineered drama. Single `index.html`, Three.js r128,
no build step, deployed to GitHub Pages. Designed for portrait mobile and
TV casting.

**Purpose:** entertainment for the family Christmas gathering. Guests place
bets through the Derby Day betting app (see Related Project), then the race
is cast to the TV and everyone cheers their runner. Races run on demand,
several per evening.

- **Live:** https://tomkweb.github.io/reindeer-race
- **Repo:** github.com/tomkweb/reindeer-race
- **Current version:** v7.9 (version string appears in exactly 4 places —
  bump all together, verify with `grep -c`)

## Deployment
Edit `index.html` on GitHub (web UI works from Android), paste full file,
commit. Pages redeploys in 2–5 min. Only other repo files are `scene.gltf`
+ `scene.bin` (deer model, CC-BY-4.0 by DarryleH — keep the credit line).

## Race drama system (the heart of it)
- Base speeds tightly banded (13.6–14.4) + big sinusoidal surges →
  constant lead changes (~69/race in simulation)
- Rubber-band compresses the pack (front eased, back helped, ±15% cap)
- **The closer:** one random deer per race sits 7th–9th through 70%, then
  runs a gap-aware kick (speed computed per frame from what's needed to
  catch the front-runner; 1.65× cap). Wins ~80% of races; the other ~20%
  it draws alongside and a nose-cap (99% of rival speed inside 2.5m) keeps
  it a whisker short
- Closer is band-exempt during its kick — WITHOUT this, the band drags it
  back after it takes the lead and it gets re-passed (bug fixed in v7.8)
- **The Grinch:** every 3rd race (raceCount % 3), drops from 42m at the
  stretch halfway point, lands ahead of the leader, steals the race
  (2.2× cap, verified 20/20 vs a winning-mode closer). Takes place 1 in
  results, winner cam, and photo finish. Built from primitives in
  buildGrinch(). NOT bettable in the Derby Day app — his win triggers the
  betting app's unclaimed-pool refund
- No-fly zone: no new leaps after 72% race distance — hooves down for
  the drive

## Camera system (broadcast structure)
gate head-on (2s hold) → wire cam first pass (frames whole pack, real lens
optics: FOV = 2·atan(halfWidth/dist)) → grandstand through turn 1 →
backstretch crane/telephoto/blimp cuts every 4–6s → head-on far turn →
wire cam with surge-adaptive zoom (blends aim toward the charging closer)
→ grandstand side view for the final half to the line.
- Blimp tracks the pack from 175m, up-vector rolled (1,0,0) for portrait;
  blimp tags enlarge via `activeCam` (NOT camMode — activeCam resolves
  auto-director picks)
- Stretch drive: tags fan out by running order (rank-based heights) so
  nose-to-nose runners stay readable
- Photo finish: camera ON the finish-line axis at the outside rail, deer
  snapped so nose touches the line, forced portrait 480×854 render,
  fenceGroup + finishPoles hidden during the one-frame capture

## Track & geometry
STR=320, TR=120, CIRC≈2394m, TW=14. Oval centre (0, 160). Track runs CCW
(`u = 1-p` in trackPos). Gate at p=0.75, finish at cumulative 2.0
(1.25 laps). Ground plane 2400×2400 centred on the oval — it was 800×800
at the origin once and the far turn hung off the edge (the "blue band").
Changing STR/TR requires auditing every camera position and camera.far.

## Three.js r128 gotchas
- 9 cache-busted model loads (`scene.gltf?n=0..8`) or all deer share one
  skeleton; SkeletonUtils.clone per deer
- normalizeClip() shifts Rundeer keyframes to t=0 or animation never starts
- No CapsuleGeometry (r142+); no OrbitControls without import maps
- camera.lookAt gimbal-locks pointing straight down — blimp uses rolled
  up-vector + offset instead
- Blimp rolls camera.up and photo capture must reset it or captures rotate

## Testing protocol (container)
Local libs at `race_v2/libs/` (npm pack three@0.128.0), model fetched from
raw.githubusercontent (allowed domain). Build test_local.html by swapping
CDN URLs for local paths; inject window.JUMP(p) / window.CAM(m) helpers.
Playwright headless (SwiftShader) for rendering/UI; **race dynamics are
verified in Node simulations** mirroring the exact speed equations —
headless rAF throttling makes live-race timing useless. Syntax-check the
main script block with node vm before every delivery.

## Open items
- Real-phone verification of v7.9 drama pacing and camera framing
- Closer win-rate knob: `dramaCloserWins = Math.random() < 0.80`
- Better deer model (Rifat3D, Sketchfab, needs PC purchase)
- Sound: crowd, hoofbeats, Grinch sting

## Related project
**Derby Day betting app** — `tomkweb/derbyDay2026`, live at
tomkderby.pythonanywhere.com, branch `derby-2027-dev`. Flask/SQLite
pari-mutuel betting: guests bet on these reindeer races at Christmas
(name + PIN at /bet, teller collects cash, TV shows pools). Needs a
9-reindeer seed and a between-races reset flow. Its CLAUDE.md lives in
the DerbyDaySMS project folder.
