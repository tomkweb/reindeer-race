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
- **Current version:** v8.18.4 (version string appears in exactly 4 places —
  bump all together, verify with `grep -c`)

## Deployment
Edit `index.html` on GitHub (web UI works from Android), paste full file,
commit. Pages redeploys in 2–5 min. Other repo files: `scene.gltf`
+ `scene.bin` (deer model, CC-BY-4.0 by DarryleH) and `audio/crowd_*.mp3`
(crowd samples, CC-BY-4.0 by Gregor Quendel) — keep both credit lines.

## Race drama system (the heart of it)
- Base speeds tightly banded (13.6–14.4) + sinusoidal surges → lead
  changes. Measured (v8.18.2, Node mirror of updateRace, 600 races): ~11
  lead changes/race that last ≥1s (Tom asked for "a little" fewer than the
  previous ~13). The lever is surge FREQUENCY (`surgeF` 0.08–0.19); surge
  amplitude and the per-frame ±1.5% noise barely change the count. Closer
  still wins ~80%. Headless Chrome is too slow for multi-race stats — use
  the Node mirror (scratchpad `leadsim.js` pattern)
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

## Announcer (Web Speech API) — modelled on real broadcasts
Tuned (v8.8.0) from an analysis of WHR's "12 Best Races of 2025" (11 race calls
transcribed; audio + frames measured locally, nothing from it committed):
- **Voice:** British male (the Ascot/York/Leopardstown sound). `GB_MALE`
  preference list: Google UK English Male → Edge "Ryan/Thomas… (United
  Kingdom)" natural voices → Apple Daniel/Arthur → Android `en-gb-x-rjs|gbd`.
  Android Chrome exposes only one "English United Kingdom" voice; its gender
  is whatever the phone's Google TTS en-GB voice is set to (system setting).
- **Pacing:** real callers hold ~3 words/s the whole race and do *not* speed up
  at the finish. Excitement is pitch (calm F0 ~120-135 Hz → ~230-265 Hz at the
  wire, nearly an octave) and volume. `say()` keeps rate brisk and near-flat
  (1.15-1.25) and maps call-site pitch onto 1.15-1.9 — Tom wants a high-energy,
  higher-pitched caller (v8.9.0), not a low baritone.
- **Sync (v8.12.0):** `say()` is priority-aware — pri 1 colour (idle only),
  pri 2 lead change, pri 3 landmark, pri 4/force (break, Grinch, winner) cuts
  in. Pri 2-3 wait in a one-line queue instead of chopping the current line;
  queued lines expire (1.8/2.5s), re-check `opts.valid()`, and `txt` may be a
  function so the text describes the race when it's actually spoken.
  Landmarks fire at the REAL track boundaries (`SEG` from S1..S3: first turn
  ~20%, backstretch ~42%, far turn ~60%, homestretch ~82%) — the old round
  numbers called "backstretch" mid first turn. Lead changes (v8.13.0, Tom
  found every-change calls too many): before the turn for home only a change
  held 2.5s gets its own call, ≤1 per 12s (~1.5/race; roll calls name the
  leader anyway); in the stretch every change held 0.6s, spaced 1.2s
  (~4/race). Calls name who was passed. Short lines only (≤3 names). A pre-race line on the Race tap
  warms up the TTS engine so "And they're off!" lands on the break.
  Verify with the Node harness pattern: extract `const ann={` … `// ── CROWD
  NOISE` into a vm with mocked speechSynthesis (duration ≈ words/3s + 0.35s)
  and simplified race physics; v8.11 called 26% of lead changes, v8.12 ~73%
  (the rest are re-passed before they can be called), median delay ~1.1s.
- The race is 1.25 "miles" = 10 furlongs.

## Voice fallback (v8.18.1) — IMPORTANT
Cloud TTS voices (Edge "Microsoft … Online (Natural)", Chrome "Google …") are
the nicest but intermittently fail with `synthesis-failed`; on 2026-10-25
Edge 155 on Tom's PC had ALL of them failing and the game went silent (only
the crowd played). `ttsVoice()` + `ttsGuard()` now wrap every utterance (race
calls and interviews): on a fatal error or no `start` (6s for cloud voices,
3s offline) the line is re-spoken at once on an OFFLINE (`localService`)
voice — male for Adam/Santa/caller, non-male for jockeys. v8.18.3: one blip
only affects that line (`avoid`); a voice is dropped for the session only
after 2 failures in a row (`ttsFails`, reset whenever a line starts) — the
first version dropped it on one blip and Adam switched voices mid-interview.
ROOT CAUSE on Tom's PC (2026-10-26): **Proton VPN** — with it on, every Edge
"Online (Natural)" voice fails; off, they work. Keep the VPN on by excluding
msedge.exe via Proton split tunnelling. The offline fallback (Microsoft
David) is slow and monotone — it's a safety net, not the intended sound.
Diagnose speech with Claude in Chrome on the live page: wrap
`speechSynthesis.speak` to log start/end/error events, then REALLY click a
button (speech without a user gesture fails `not-allowed`).

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
- Name tags (v8.10.0): `makeTag()` pills trimmed to the name's width;
  `sizeTag()` holds them at ~3.5% of frame height (0.35-0.9m world clamp) so
  they don't balloon in close shots. Blimp keeps big 3.75m-tall tags
- Framing rule (Tom): the tightest leader-tracking shot must still show the
  leader AND the next two. `fitTop3Fov()` widens (never narrows) the FOV of
  every mode in `FIT_TOP3` until the top 3 (+ landed Grinch) fit in the
  current aspect
- Long-lens shots (v8.14.0): `fitFov()` is the shared real-lens fit (aspect-
  aware, 2°-80°). Wire cam + telephoto now SET their FOV from it on
  `packPositions()` — the old wire formula ignored aspect and had a 14° floor,
  so a 300m+ shot on a landscape PC/TV showed a speck. Landscape frames
  everyone within 30m of the leader; portrait the leading group (12m, ≥ top 3)
- Steadiness (v8.15.0, Tom saw jitter in the tight shots): `steadyGroup()`
  feeds wire + telephoto — soft 12m membership fade, centre = weighted mean
  progress/lane mapped onto the track (lead changes don't move it), lag lead
  from 1s-smoothed `packSpeed` × 1/sm. Fit is on the ground plane (leaps don't
  pump the zoom); zoom out fast/in slow; camera.fov updates on a RELATIVE
  threshold (the old fixed 0.05° made a 1-2% zoom staircase at 3°). Measured
  with a headless harness driving animate() by hand at fixed 1/60 (headless
  rAF barely ticks): telephoto zoom jitter 46→2.8, aim 0.17→0.04; wire zoom
  70→1.7. Telephoto camera now rides OUTSIDE the track (150m off the rail) —
  from its old infield spot the 7-14m infield trees blocked the tight lens
- Run to the line (grandstand, raceFrac>0.88, v8.13.0): side-on from the
  outside rail framing lead room ahead of the leader back to 3rd place, so
  the leader is on the RIGHT (travel is left→right) with chasers trailing;
  camera distance set from that span at a fixed 40° FOV
- Photo finish: camera ON the finish-line axis at the outside rail, deer
  snapped so nose touches the line, forced portrait 480×854 render,
  fenceGroup + finishPoles hidden during the one-frame capture

## Track & geometry
STR=320, TR=120, CIRC≈2394m, TW=14. Oval centre (0, 160). Track runs CCW
(`u = 1-p` in trackPos). Gate at p=0.75, finish at cumulative 2.0
(1.25 laps). Ground plane 2400×2400 centred on the oval — it was 800×800
at the origin once and the far turn hung off the edge (the "blue band").
Changing STR/TR requires auditing every camera position and camera.far.

## Pre-race interview — Intro button (v8.17.0)
- Title screen: 🎤 Intro · Start Race · 🔴 Calibrate Nose. Calibration was
  removed in v8.17.0 and RESTORED in v8.17.1 — seeing Rudolph up close in the
  interview showed the nose needs re-tuning. Tom reads x/y/z off the panel
  ("Tell Claude: …") and the values get baked into `noseOff`. Current:
  x=0.02 y=-0.72 z=0.02 (v8.17.2). Keep the slider `value=`s in the
  calibration panel in sync with `noseOff` so the panel opens at the real spot.
- NPN (North Pole News) reporter **Adam** interviews **Santa** and Rudolph's
  jockey **Hermey** (`ELF_NAME` — the 1964 TV special's would-be dentist) on
  the track in front of the grandstand (`INTRO_SPOT`). Santa/Adam are built
  from primitives (`buildSanta`/`buildReporter`) at race scale (~1.7×) with
  sRGB→linear colours (raw hex reads pastel under the sRGB output).
- `INTRO_SCRIPT`: 7 segments × 2-7 variants (~1,000 combos), ~85-95s spoken.
  Grinch is only a RUMOUR — never hint which race (Tom: keep it a surprise).
- Voices: Adam en-US, Santa = race caller's en-GB male at pitch .55, Hermey
  en-AU at pitch 1.9 — separate accents because Android has one voice per
  locale. Captions + name supers in an HTML lower-third; a timer fallback
  advances lines if speech is muted/unavailable.
- Camera: TV cuts per line (wide on open/close, two-shot on questions,
  close-up on answers) framed from the characters' real head positions;
  camera stays ≤11m out so the inner rail isn't in the lens. Lights are
  softened during the interview and restored after.
- Rudolph is moved to the spot and restored on exit; tags hidden meanwhile.
- v8.18.0 (Tom's review): ⏸ Pause beside Skip (cancels the line + freezes;
  Resume replays the line — Android ignores speechSynthesis.pause()). Order is
  Santa · Adam · Rudolph; Adam's mic arm is one-bone IK aimed at the
  speaker's mouth (`setFromUnitVectors` in Adam's local space). Jockey rides
  facing forward; only his HEAD turns to camera (`elf.userData.head` — head
  parts live in their own group in `makeElf`). Elves got a mouth and real
  riding legs (thighs over the back, shins down the sides) — the old straight
  legs ended inside the deer and the boot box poked out of the flank. Nose is
  Santa red (sRGB→linear, emissive 1.1). Adam's mic arm has an ELBOW
  (v8.18.2): talking = two-bone IK (hand under his chin, elbow down/out,
  forearm + mic pointing up at his mouth); guest talking = straight reach. "Louder" = crowd target 0.04 during
  shows + softer show cheers (speech volume can't exceed 1).

## Winner's interview (v8.18.0) — results screen "🎤 Winner Interview"
- Same engine (`startShow(kind)`, kind 'pre' | 'win'; `intro.star` is the
  featured runner). Winner's circle at `WIN_SPOT` (z=24, just past the line);
  Santa presents the Golden Carrot (gold prop, raised on his presenting line).
- `JOCKEYS` (NAMES order): Pepper Minstix, Sugarplum Mary, Shinny Upatree,
  Alabaster Snowball, Wunorse Openslae, Bushy Evergreen (popular elf lore),
  Jingle Jangleton, Tinsel Twinkletoes (ours), Hermey. Each jockey gets a
  slightly different voice pitch.
- `buildWinnerScript` uses `raceStats` (reset in configureRace): `margin` in
  lengths when the winner crossed (`recordWinner`), `far` = running order at
  the far turn. Story branch: photo finish (<0.6 L: nose/head/neck) → came
  from ≥4th at the far turn → runaway (≥4 L) → else the runner-up "pushed you".
  Never declares a result the race didn't produce.
- Grinch wins → Adam interviews the Grinch (US voice pitched to a sneer,
  laugh pose on "heh/mwah"), Santa: "Somebody check his pockets."
- Test harness pattern: stage results (set `place`, `raceStats`), call
  `startShow('win')`, log `intro.lines` — no need to run a race.

## Grandstand + crowd (v8.16.0)
- `STAND` (x0=TR+TW/2+8, 12 rows × 1.7m deep × 0.85m rise, z 6→240) along the
  OUTSIDE of the homestretch; finish line (z=0) at its near end. Banner wall
  "★ THE REINDEER DASH ★" above the top row.
- ~1,700 spectators = 4 InstancedMeshes (body/head/arms/Santa hats on ~35%),
  per-instance colour, `frustumCulled=false` (instances span 230m; base
  bounds would cull them). ALL animation is in the vertex shader
  (`crowdMaterial` onBeforeCompile, per-instance `aPhase`) driven by
  `crowdU.uT/uEx`; `uEx` follows `crowd.level`, so the stand erupts with the
  audio. ~100k tris, 4 draw calls, no per-frame CPU. No shadows on the crowd.
- Cameras: `liftOverStand(tp,ta)` raises any camera whose line of sight
  crosses the stand (`standTopAt` incl. the taller back/end walls) — applied
  to every directed shot and the winner orbit. Telephoto on the homestretch
  switches to the INFIELD side at 25m (over the 7-14m trees) so the crowd is
  backdrop, not a wall in front.
- Headless note: the blimp view crashes headless SwiftShader ("Abnormal
  renderer termination") — pre-existing, not the crowd. Use a fixed-vantage
  `__VIEW` override in the test page to inspect the stand.

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
Playwright headless (SwiftShader) for rendering/UI. On this Windows PC,
plain headless Chrome works too: serve the folder (`python -m http.server`),
inject a click on #startBtn + `JUMP`, and for still frames patch the camera
lerp to snap (`a=1`, `currentFov=targetFov`) and drop leadersFrame's 0.55s
lead. Headless windows are never narrower than 500px — a 412px screenshot
silently crops the right edge, so shoot at 500x1040; **race dynamics are
verified in Node simulations** mirroring the exact speed equations —
headless rAF throttling makes live-race timing useless. Syntax-check the
main script block with node vm before every delivery.

## Open items
- Real-phone verification of v7.9 drama pacing and camera framing
- Camera ideas from the broadcast analysis (not yet done): real coverage
  uses far fewer cuts — one long telephoto side-on pan carries most of the
  backstretch (Belmont: ~60s unbroken), head-on telephoto into turns, 7-14s
  typical shots; our 4-6s backstretch cuts are busier than TV. Also: a
  running-order strip of silk colours + mini track map with distance-to-go
  (Japan Cup graphics), and a name/number lower-third at the gate
- Closer win-rate knob: `dramaCloserWins = Math.random() < 0.80`
- Better deer model (Rifat3D, Sketchfab, needs PC purchase)
- Real-phone listen to the interview voices (Android accents/pitches)
- Real-phone check of the grandstand's frame rate (~100k extra tris)
- Sound: hoofbeats, Grinch sting still to do. Crowd noise (v8.7.0) uses
  real recordings in `audio/` — a synthesized noise crowd (v8.6.0) sounded
  like static and was dropped. Two loops (`bed` ambience, `excited` chatter)
  crossfade with race phase in `crowdUpdate`; one-shots via
  `crowdRoar(name,vol)`: cheer at gates, surge for the closer, roar at the
  wire. Clips were cut from Gregor Quendel's "Free Crowd Cheering Sounds"
  (opengameart.org) with 3s equal-power loop crossfades; loops skip 50ms at
  each end for MP3 padding. 🔊 mutes both. Crowd must stay clearly UNDER the
  announcer (Tom's call): master `CROWD_VOL` 0.35, ducked ×`CROWD_DUCK` 0.4
  while speaking. Silent until the first Race tap (`crowd.started`; context
  suspended at load because Chrome auto-runs AudioContexts on familiar sites).
  Pre-race it starts soft and swells through the lineup + 3-2-1 countdown,
  peaking as the gates open (`crowd.preT`).
  One-shot clips are cut to start AT their peak (the source recordings take
  5-6s to swell — the first cut made the gates-open cheer land ~9s late). They
  play on their own `shotBus` that barely ducks (`SHOT_DUCK`), so the roar
  isn't buried under "And they're off!". `crowdUpdate` steps on the audio
  clock, not frame dt, so swells don't lag on slow phones. Gates open: cheer
  + instant level jump, no announcer duck for 3.5s. Audio URLs carry
  `?v=VERSION` — Pages caches files ~10 min, and a stale clip once made the
  start surge seem ~9s late on Tom's phone.
  v8.11.0: Tom said it sounded like a rugby match and wants men/women 50/50.
  `excited` loop is now CC0 Freesound #486215 (men & women egging on);
  cheer/surge/roar one-shots blend CC0 women's/mixed cheering (#480755,
  #380398, #380400) with the Quendel male roars at equal RMS. Rejected:
  #204950 trackside race-meet recording (constant hum + PA caller under it).
  Pitch-based gender estimates are useless on shouting crowds (male shouts
  land in the female F0 range) — judge by source description/spectrogram.
  v8.14.0: dropped #480755 entirely — one woman's rising held "wooo"
  (~740→1240 Hz, >1s) read as a siren. Screen new clips for sustained tones
  (spectral-peak runs ≥0.3s in 350-1600 Hz) before using them.
  Never rip audio from broadcasts (e.g. NBC Derby on YouTube) — the repo is public

## Related project
**Derby Day betting app** — `tomkweb/derbyDay2026`, live at
tomkderby.pythonanywhere.com, branch `derby-2027-dev`. Flask/SQLite
pari-mutuel betting: guests bet on these reindeer races at Christmas
(name + PIN at /bet, teller collects cash, TV shows pools). Needs a
9-reindeer seed and a between-races reset flow. Its CLAUDE.md lives in
the DerbyDaySMS project folder.
