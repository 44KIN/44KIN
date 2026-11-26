---
name: motion-design
description: Code-only motion design pipeline (no After Effects): product demo videos and app walkthroughs, launch films, promos, showreels, explainers, logo stings, LinkedIn/X/TikTok clips. Use when asked to make a motion graphics or animated video, a demo/promo/launch video, a showreel, "remake this video", a video from a prompt, or to change a video's music/SFX. Covers brief → beat map → stills → seek(t) HTML engine → Playwright render → ffmpeg → music/SFX → QA, plus asset sourcing (Mixkit, Pexels, Unsplash, svgl, 21st.dev).
---

# Motion design, 100 % code

Adapted from howseen-ai/claude-motion-design (MIT, see LICENSE). Deterministic HTML animation rendered frame by frame with Playwright, encoded with ffmpeg, music/SFX from Mixkit or synthesized. Free and fully reproducible.

## 00. Workspace & setup
- **Workdir: `video-factory/` at the repo root.** One folder per film: `video-factory/<name>/` = `<name>.html` + `render.py` + `audio.py` + `assets/` + `out/`. Shared inputs go in `video-factory/assets/` (logos, fonts, music, SFX), `video-factory/references/` (reference videos/frames), `video-factory/sources/` (raw footage, screenshots). Copy the closest earlier film folder instead of starting from zero once there are some.
- **Never commit rendered video** (`out/`, `sub*/`, `probe/`, frame dumps): send finished files to the user directly. The repo `.gitignore` ignores everything inside `video-factory/*/`, so film sources stay local too; commit one (HTML, scripts, small assets) only when the user asks, with `git add -f`.
- **Tools**: ffmpeg on PATH (scripts fall back to `imageio_ffmpeg`). Python deps: `pip install playwright numpy` if missing; never run `playwright install`: the scripts launch the preinstalled Chromium at `/opt/pw-browsers/chromium` when it exists.
- Templates: copy `scripts/render_template.py` and `scripts/audio_template.py` (in this skill folder) into the film folder and edit the constants at the top.
- **Related skill**: `video-editing` handles cutting/trimming/subtitles of existing footage; this skill builds animation. Combine them (animate an intro here, edit it onto footage there).

## 1. Flow (always in this order)
1. **Inputs**: if the brief has an `<inputs>` block, ask for them (AskUserQuestion, recommended defaults first). Otherwise pick sensible defaults and say so.
2. **Beat map** (`BEATMAP.md`): BPM → beat length, every scene on a beat, the **music drop on the key visual moment** (flood, logo, big reveal). Nothing still for > 1 s.
3. **4 stills** (or a one-frame-per-beat sheet) → look at them (Read) → fix → only then the full render.
4. Full render → pops scan → audio → mux → send the file to the user with its path.

## 2. The engine (one HTML file)
- Everything computed from time inside `window.seek = async (t) => {…}`; **no CSS transitions, no timers, no state between frames**. Declare all constants before the first `seek()`. Set `window.ready = true` after fonts/images load.
- **Springs** = closed-form step response `step(tau, f, z)`; a value with many targets = sum of one spring per change. Easings: `io` (cubic in-out), `out`, `in`, `o5`, `expo`. Linear motion = cheap, never.
- **Camera** = one transform on a container, keys `[t, zoom, x, y]`, eased segments, **zoom interpolated in log space**, never zoom in/out back-to-back. Beat punches: `+0.012` per beat, `+0.03` per bar after the drop, exp decay.
- **Shared elements** for every handoff (the bubble carries its words into the flood, the button carries its label into the page). Text that swaps inside a morphing shape gets its own mask.
- **Masked text rise** (translateY 105% inside overflow:hidden), word-by-word stagger (55 ms) with a small rotation; accent words with a moving gradient (`background-clip:text`).
- **Floods**: circle from the source object, must **clear the farthest corner** (`hypot` to the 4 corners ×1.05) in ~0.3-0.35 s, then contract into the next object.
- Effects that port well to seek(t): liquid glass (canvas displacement), goo (SVG filter), 6-blade iris, variable-font width squeeze, morph shapes, 3D cube, equalizer, blob mask, animated beam / border beam, dotted grid, drifting blobs, sheen sweep.
- `z-index` on every layer. `visibility:inherit` (not `visible`) on children of hidden parents.
- **Look**: set the brand font and ONE accent colour per film; neutral bases like warm off-white `#f5f5f2` or ink `#0b0b0c` work well. Self-host fonts as woff2 in `assets/`.
- **Preview controls** on every film page: Space = play/pause, ←/→ = previous/next frame, R = replay, a time readout. Playback calls the same `seek(t)` from a rAF loop (the only place a clock is allowed); the render never touches it.
- Frame-number variant (brief given as frame tables f0–fN): register per-frame keyframe tables, a motion law per move, settle/exit/join helpers, and scan tables for one-frame outliers. Split the shot registry into files so several agents can build shot groups in parallel (one file each).

## 3. Render (scripts/render_template.py)
- Serve the folder over HTTP (`python -m http.server 876x --directory …`, background), Playwright Chromium, viewport = video size (1920×1080, 1080×1350 for 4:5, 1080×1920 for 9:16, 1440×1440 square).
- `probe t1 t2…` → `probe/sheet.png`; `beats` → one frame per beat; `draft` → 30 fps, 1 capture/frame, 960×540, crf 23 veryfast (**always before the master**: judge rhythm, not sharpness); `full` → **N subframes per frame blended with `tmix`** (6-8 for fast moves, 4 = ghosting) spread over `SHUTTER = 0.5` of the frame (180° film shutter; 1.0 = smeary), 60 fps; `pops` → frame-diff spikes > 3× neighbours (intentional beat cuts show up too: say so, don't hide); `loop` → seam check in **position (last vs frame 0) AND velocity (motion into vs out of frame 0)**.
- Capture hygiene (built into the template): wait `document.fonts.ready`, pause `document.getAnimations()` and kill CSS transitions after load, and **wait 2 `requestAnimationFrame` after every `seek(t)`** before the screenshot. `WEBGL=1` launches Chromium with SwiftShader/ANGLE flags (headless WebGL otherwise renders black).
- **Films > 25 s: render in 3 parallel chunks** (frame ranges in background processes, then concat). Re-render only the changed seconds when possible. Run long renders in the background.
- Use a separate `sub*/` folder per version so parallel renders don't clash. ~1-1.5 min of wall time per second of film at 8 subframes.
- Final encode: `scale=in_range=pc:out_range=tv:out_color_matrix=bt709,format=yuv420p`, `-color_range tv -colorspace bt709`, libx264 crf 16, AAC 256k, `+faststart`.

## 4. Music & SFX (scripts/audio_template.py, analyze_song.py)
- **Music = Mixkit** (free commercial). Direct file: `https://assets.mixkit.co/music/<id>/<id>.mp3`. IDs: grep `music/[0-9]+/[0-9]+\.mp3` in the listing page HTML. Mixkit search ignores `?q=`: crawl genre pages (`/free-stock-music/<genre>/`) and match `item-grid-card__title` to `music/<id>/<id>.mp3`.
- Measured tracks: `mixkit-207` 120 BPM (drop 31.97 s), `mixkit-190` 120 BPM (drop bar 8 = 16.01 s), `mixkit-129` 120 BPM (drop 16.09 s), Cat Walk (Arulo #371) 130 BPM drop 14.769 s, Waka Floka Type (#364, trap) drop 14.75 s, Driving Ambition (#32, uplifting piano ~99 BPM) hit 37.66 s, Classical vibes 4 (#684, ~94 BPM) lift ~7.95 s, Head Bang (#357, hip-hop half-time 74 BPM) drop 12.96 s. Tempo feel: 60-80 BPM regal, 90-110 smooth, 115-123 sophisticated, > 125 hype.
- **Find the drop by energy**, never trust an auto grid: per-bar low/full band energy, then 20-50 ms windows around the jump. Start the song at `drop_in_song - drop_in_film`.
- **SFX = Mixkit**, download into `video-factory/assets/sfx/` (`https://assets.mixkit.co/active_storage/sfx/<id>/<id>-preview.mp3`); search with `scripts/mixkit_sfx_search.py <tag>`. Known IDs: click 1125, key 2568, soft tick 1117, check 1113, toggle 1120, toast 2573, pop 2364 / bubble 2357 / soap 2925, whoosh 1490, rise 1489, flip 1485, impact 1143, shutter 1430 / lens 1433, sparkle 3083, success 2865.
- **Place every SFX by its measured peak** (argmax of |s|), gain 0.04-0.3, keystrokes follow the same per-character rhythm as the typing animation. Fade the tail, **two-pass loudnorm to −14 LUFS**. Voice-over: TTS with word timestamps → cues.json, music ducked ~9 dB under the voice.
- **Synthesized sound option**: SFX in code (click = short decaying sine, pop = rising sine, thump = falling sine, whoosh = windowed noise) on the same timeline; no downloads needed.
- Minimal sound design for "premium" films: a handful of soft hits, remove anything loud or out of place.

## 5. Assets
- **Photos**: Unsplash `https://unsplash.com/napi/search/photos?query=…&per_page=30` (curl ok) → `urls.raw + &w=2600&q=85&fm=jpg`; fallback `picsum.photos/id/<id>/<w>/<h>` when napi returns 401. Pexels CDN `https://images.pexels.com/photos/<ID>/pexels-photo-<ID>.jpeg?auto=compress&cs=tinysrgb&w=1600` (search pages block curl: use WebFetch/WebSearch for IDs). Always build a contact sheet and **look at it** before using.
- **Video**: Mixkit `assets.mixkit.co/videos/<ID>/<ID>-1080.mp4`, Pexels `pexels.com/download/video/<id>/`. Re-encode all-intra (`-g 1`), load as blob URL, await `seeked` (or pre-extract frames with ffmpeg if seeking flakes).
- **Logos**: `scripts/svgl_logos.py` (svgl.app API, colour SVGs of common brands); fallback simple-icons (`cdn.jsdelivr.net/npm/simple-icons@13/icons/<name>.svg`).
- **21st.dev** components (Animated Beam id 919, Border Beam 1268, Orbiting Circles 1411…): `scripts/mcp21_client.py tools | call search '{…}' | call get_component '{"id":…}'`, key in env `API_KEY_21ST` or `~/.config/21st.key` (free tier: 2 code retrievals/day). They're React/framer-motion: **port the idea to seek(t)**, never run them live.
- **3D icons**: 3dicons.co (CC0, 1,500+ renders, transparent PNGs).
- **Fictional people in UIs**: AI-generated faces (thispersondoesnotexist.com/random-person.jpeg, crop centre 76 % to drop the watermark), names chosen to match each face.
- **Seek-compatible animation libs**: Anime.js (`autoplay: false`, then `anim.seek(t*1000)`), Theatre.js (`sheet.sequence.position = t`, drop the Studio in render builds), GSAP (`tl.pause(); tl.totalTime(t + 0.001, true); tl.totalTime(t, true)`), WAAPI/CSS (`document.getAnimations().forEach(a => { a.pause(); a.currentTime = t * 1000 })`), Lottie `goToAndStop(t*1000)`. Never use real-time-only libs (Spline runtime, Unicorn Studio, live framer-motion): frames drift.
- **Ideas to port** (don't run live): Kinetics (kinetics.colorion.co), CSS Text Effects (text-effects.colorion.co), Liquid Glass (glass.samasante.com), Motion Primitives, Magic UI, Aceternity, 21st.dev.

## 6. Gotchas (all hit for real)
- An easing solved by bisection returns ~1e-9 at x=0: `if (e > 0)` guards fire early. Return exact 0/1 at the ends.
- Logo reveal from a line: squash the real mark vertically (scaleY 0.014 → 1, "eye opening") and overlay the solid line for the first 12 %; never crossfade a drawn shape into the PNG (grey ghost).
- Cloudflare blocks Python's default UA on some APIs: send a generic `User-Agent`.
- Hash-only `goto` doesn't reload: set state via `evaluate`. Measure text with canvas (`measureText`) not DOM rects when a camera scale is applied.
- Text that must stay sharp during a handoff: never scale a blurry copy, crossfade only the fill.
- Platform formats: LinkedIn 4:5 1080×1350; X 16:9 or 1:1, ≤ 2:20; TikTok/Reels/Shorts 9:16 with text inside the centre 1080×1080.

## 7. Delivery checklist
☐ stills approved ☐ 0 unexplained pops ☐ drop on the key moment ☐ −14 LUFS ☐ TV-range BT.709 ☐ file sent to the user + path given.

## 8. Critique loop (make the model watch its own frames)
Before any full render, and after it:
```
ffmpeg -i out/final.mp4 -vf "fps=2,scale=270:-1,tile=6x5" -frames:v 1 out/contact.png      # overview
ffmpeg -ss <t-0.1> -i out/final.mp4 -vf "scale=320:-1,tile=12x1" -frames:v 1 out/strip.png  # 12 frames around a fast move
ffmpeg -i out/final.mp4 -vf "fps=1,scale=360:-1,tile=5x3" -frames:v 1 out/phone.png         # readability at phone width
ffmpeg -stream_loop 1 -i out/final.mp4 -c copy out/loop_check.mp4                            # loop seam (loops only)
```
Open them and **score 1-10**: hook in the first 2 s · readability at 360 px · motion quality (springs, no dead frames) · variety (something new every 2-4 s) · composition · brand consistency · sound sync. Write the 3 worst problems with timestamps (hunt for: text overlapping during swaps, anything moving linearly, corner labels/frame borders, centred title on a gradient, blurry scaled text, a dead beat, a loop stutter). Fix, re-render only the affected seconds, re-score. **Repeat until every score is 8+.** Be a harsh motion director, not a proud author. Best run as a separate read-only sub-agent (judge ≠ builder); have a fresh agent restate the message from the frames alone; if it can't, the film fails.
- Lessons from past critiques (apply from the first pass): never key settles in 3-frame steps (per-frame law, fastest step first), exits geometric ×1.5, **no frozen frame anywhere except the final hold** (keep a micro settle/drift alive), the next shot enters already moving in the exit's direction, real icons (no empty placeholder squares), one face per name everywhere, grade photos with a continuous per-pixel formula (hue thresholds blotch JPEG blocks).

## 9. Extra rules
- **Determinism**: never `Math.random`; use a seeded PRNG (mulberry32). Rendering the same second twice must give identical frames. Chromium flags for pixel-identical reruns: `--deterministic-mode --run-all-compositor-stages-before-draw --disable-threaded-animation --disable-checker-imaging --font-render-hinting=none --force-color-profile=srgb`.
- **Reference first**: with a reference video/frame, extract a frame every 0.5 s with ffmpeg, write `style_guide.md` (palette hex, type, shot lengths, transitions, camera, texture, text in/out) and `shotlist.md` on the beat grid. Take the grammar, never the content or logos. Wait for OK before code.
- **Spring presets** (stiffness k, damping d): snappy UI 320/30, default containers/camera 170/26, heavy type/logos 120/24, playful mascots 180/12. Leading and trailing edges of a stretching indicator on different springs.
- **Formats**: write scenes against a layout function, then render 9:16, 1:1, 16:9 and 4:5 from the same timeline, reframing type and UI per format (never crop).
- **Effort**: medium for small fixes, high/xhigh for a new film.
- **Adaptive subframes**: measure the fastest element's px/frame; still → 1 sample, ≤ 20 px → 4, above → `ceil(px/5)` up to 64. **Never blur across a hard cut** (list cuts in `CUTS`; the template clamps subframes to the frame's side of the cut). Blur only the 1-3 slam moves; never blur text meant to be read. **Supersample 2×** (`device_scale_factor=2`, Lanczos down) for razor text.
- **Cut seams**: cut at peak velocity; direction AND speed match on both sides; zoom keeps the sign of d(scale)/dt across the cut.
- **Poster in frame 0** (X/Slack/Discord ignore the cover): `-filter_complex "[0:v][1:v]overlay=0:0:enable='eq(n,0)'[v]"` with the poster PNG as input 1.
- **Music cues**: onset strength per track, intensity = 0.45·onset + 0.25·contrast + 0.20·rms + 0.10·bass; reveals may shift ≤ 0.15 s to snap onto a cue.
- **Captions**: pop 1-3 words or karaoke 3-7, ≥ 0.7 s on screen, ≤ 17 chars/s, a word lights on its start frame never before; Whisper word timings, drop words hallucinated over trailing silence.

## 9b. Director's brief & style
"The prompt is 10 %, the harness is 90 %."
- **Film folder** = `brand/` (logo.svg, palette.txt, screenshots/, reference.mp4 optional but decisive) + `<name>.html` + `render.py` + `out/`.
- **Director's brief before any code**: duration, master format, deliverable formats from the SAME timeline, subject + one-line promise, audience, channel, **a named reference style** ("Linear launch", "Stripe docs", "Apple bumper", never "premium modern"), then **states** with timestamps (hook 0-2.5 s → shape becomes screen 1 → transforms to screen 2 with cursor click → proof → logo + URL, back to state 0 if loop). Output: 6 beats + **layer list** + **sound plan** (silence / coded SFX / Mixkit track), then STOP for OK.
- **Anti-"AI motion" rules**: ONE accent colour; **one thing moves at a time**; one shape/visual system from start to end, transformations not cuts; springs with damping ratio ≥ 0.72, tiny overshoot, never cartoon bounce; banned: rainbow gradients, particles, glowing chrome, emoji, lorem ipsum, gratuitous 3D flips; type big enough for a phone.
- **Fight the fingerprint**: always set the brand font (the default font is recognisable); on 9:16 talking heads keep speaker bottom / one graphic top / captions, nothing else.
- **Silent "one sentence" test**: show the loop muted to someone; if they summarise it in one sentence ("the button became a player"), it works; if they hesitate, too much moves.
- **Prompt starters** (adapt, don't paste raw):
  - *One-shot branded*: "Make a 15-second showreel at 1920x1080, 30fps. One self-contained index.html. HTML, CSS, JS, SVG or Canvas only. No CDN, no external URLs. Bold type and simple shapes. End on [BRAND]. Play and Replay. window.seek(t). Do not render the MP4 yet." + one accent hex + "no gradient template".
  - *UI morph*: one shape through 8-12 UI states (button → loader → player → slider → toggle → tabs → chart → command palette → toast), ~120 BPM, exact loop.
  - *Social loops*: one "A becomes B" transform, 5-9 s loop, one thing moves, easings `cubic-bezier(.6,0,.2,1)` and `cubic-bezier(.2,.8,.2,1)`; ideas: button → player, search → results, card → workspace, magnetic dock.
- **Use cases that work**: product demo/walkthrough, explainer from a doc ("make it fun, not too serious"), a mascot SVG narrator, animated slide loops (5-20 s), 3 s logo sting + jingle (batchable), restyle from a reference image (add a grain pass if it falls flat).
- **Other routes** (only if asked): HyperFrames (HTML + GSAP) and Remotion (React templates). Default route = bare HTML + seek(t).

**Diagnostic when it's ugly**
| Symptom | Cause → fix |
|---|---|
| Generic, looks like every AI reel | No brief/named style → director's brief + named reference |
| Blurry text / washed logo | Went through a pixel video model → back to code, real SVG |
| Stutters on export | Timers / CSS transitions → pure seek(t), 2 rAF wait |
| Loop jumps | Position matched, velocity not → add the spring tails of the previous 2 cycles; `render.py loop` |
| Cheap, bouncy | Bounce easing → springs, damping ≥ 0.72 |
| Black WebGL | `WEBGL=1` (SwiftShader/ANGLE flags) |

## 9c. B-roll, voice-over, shorts
- **B-roll over a talking head / VO**: ask the **density** first (≈ 33 s: medium = 3 clips, heavy = 4-5). Input = word-timestamped transcript (Whisper). Plan line by line, each graphic tagged *full cutaway* / *face stays* / *transparent panel*, approved before rendering. Graphics land on the spoken word. Full-frame cutaways 3-10 s with ≥ 2 s of face between. Alpha clips via `screenshot(omit_background=True)` → `format=gbrap` → ProRes 4444; never `will-change` on an element the camera scales.
- **VO pipeline**: TTS, speed up slightly + trim silences (adaptive `silencedetect` with the threshold read from `loudnorm print_format=json`), Whisper word timings → ONE timing file driving captions and animation cues.
- **Long → Short**: one topic per Short (split rather than compress), faster cuts.
- **Brand intake**: Playwright screenshots of the site + pixel sampling into `brand/` (colours, fonts, radii).

## 10. Remake mode: frame-locked 1:1 copy of an existing video (scripts/remake/)
Use when asked to "remake / recreate this video for my brand" (split-screen "original | remake" format).
- **Phase 0, analysis (no building):** download REF (yt-dlp) → `remake_analyze.py` extracts all frames 0-based to ref/full, audio to ref/audio.wav, detects hard cuts (mean-abs-diff spikes) and writes 6-frame contact sheets. Read the sheets, write SPEC.md: shot table (id, f0–f1, REF content, brand swap), swap rules. Most "cuts" in modern launch films are continuous camera/morph moves: expect ~10-15 hard cuts, and SPEC boundaries a few frames off.
- **Phase 1, engine:** copy `core.js` + `index.html` (seek(F) pure, SHOT registry, camera, cursor, words, pixelDissolve, palette filter that re-hues leftover reference-brand colours) and `remake_stub.py` (one placeholder file per group). **Replace the brand tokens and `assets/logo-mark.png` in core.js with yours.** Serve the folder, smoke-test with `remake_render.py compare out/test 10 600 1200`.
- **Phase 2, parallel build:** split shots into 4 contiguous groups, one agent each (fill `BRIEF_TEMPLATE.md`), each writes ONLY shots/Gx.js and verifies with side-by-side compare sheets. 5th agent = audio: analyse REF (BPM, drop, hard stop, SFX hits), royalty-free Mixkit track stretched ≤ 8 % and cut on bars so drops land on REF times, numpy SFX on REF hits, −14 LUFS. Never reuse REF music/voice.
- **Phase 3, integrate:** full render in 3 parallel chunks (`remake_render.py full out/full a b`), `remake_sync.py encode` (muxes out/mix.wav), `split` (two panels with a gap, setsar=1 or X distorts it), `stacked` (QA). `remake_qa.py`: ref|ours one frame per second + group seams + old-brand colour scan. Fix, re-render, deliver.

## 11. Product demo / SaaS film mode
Use for a 30-75 s film that shows a product in action (demo, walkthrough, homepage film). Lean pass first (one language, 16:9, no VO); full treatment only when asked.
- **Story = a chain**: the problem (3-5 s) → each step PRODUCES what the next one uses (the object leaving a step becomes the next scene, one continuous camera, no hard cuts) → the result on screen → final CTA. Each step 2-7 s. Every element finishes animating and stays readable ≥ 1.5 s.
- **Screens are rebuilt in code, not pasted**: get screenshots (Playwright captures of the site/app, or user-supplied into `video-factory/sources/`), scan each in zoomed tiles, extract ONE `ui-tokens` file (exact colours by pixel sampling, fonts, radii, shadows, borders, spacing), rebuild components that animate element by element (rows cascading, counters, gauges, typing, cursor, toggles). Check each rebuilt screen side by side with its screenshot until it's recognisable at first glance.
- **i18n from day one** if more than one language: no hard-coded text, one dictionary per language.
- **Muted-first**: autoplaying versions run without sound, so kinetic type carries the message. Safe margins 110 px (16:9) / 80 px (9:16); every title on ONE line.
- **Full treatment extras**: a "motion bible" (30-40 checkable rules) from the references; several competing concepts (one-take, beat montage, metaphor, director's cut) scored and merged; after v1, separate critics (motion, image, sound, story, muted readability, UI fidelity), ≥ 3 rounds, each defect with frame numbers + severity + measurable fix; a new version only replaces the previous one if side-by-side it's better (keep a version log). Deliver 16:9 + recomposed 9:16 (not a crop).
- **Machine hygiene**: one heavy render at a time, delete superseded renders (keep current + previous).
