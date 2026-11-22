# Livestream Reaction Engine: Submission

EA SPORTS FC Mobile, Software Engineering Intern technical assignment.

A floating-reactions engine (❤️ 👍 🔥 ⭐) that renders the same effect two ways so the browser rendering cost of each can be compared:

| | Phase 1 | Phase 2 |

|---|---|---|

| Engine | Pure CSS `@keyframes` | GSAP timelines (one per reaction) |

| Animated properties | `bottom`, `margin-left`, `font-size` (layout) + `opacity` | `x`, `y`, `scale`, `opacity` (transform/opacity) |

| Live speed control | No (static keyframes) | Yes: `timeScale()` on every in-flight timeline |

> **Honesty note on the evidence.** All numbers and images below come from runs I executed in **headless Chromium 157 on Linux with software compositing** (no physical GPU), using Chrome DevTools Protocol tracing with 6x CPU throttling. The two `*-performance.png` files are **charts I generated from the real trace data, not DevTools UI screenshots**. The two paint-flashing screenshots are real Chromium output with the Paint Flashing overlay on. Numbers will differ on other machines. See [section 14](#14-performance-testing-methodology) and [section 15](#15-known-limitations).

## 1. Project overview

Requirements covered: interactive stage, four reactions, Send Reaction, 3 s upward flight with sway, scale-up and fade, DOM removal on completion, 50+ concurrent reactions, CSS-keyframe phase, GSAP phase using only `x/y/scale/opacity`, a 0.1x to 2.0x speed slider that retimes in-flight animations, profiling, and written reflections. Automated test results are included in `qa/results.json`.

## 2. Setup and running

No build step. GSAP 3.15.1 is vendored in `js/vendor/gsap.min.js` and all scripts are classic scripts.

```bash

# Option A: double-click index.html (works over file://) and open it in Google Chrome

# Option B: local server

npm install        # optional: only refreshes the vendored GSAP copy

npm start          # http://localhost:8080

```

Use the **Animation engine** radio buttons to switch phases. The speed slider appears in Phase 2. The HUD shows Active / GSAP timelines / Spawned / Completed. From the DevTools console, `__reactions.stats()` returns the same numbers plus the real DOM node count.

## 3. Project structure

```

index.html                 UI markup, HUD, controls

css/style.css              theme, reaction base style, Phase 1 @keyframes

js/css-engine.js           Phase 1 engine (adds class, listens for animationend)

js/gsap-engine.js          Phase 2 engine (one timeline per reaction, timeScale control)

js/reaction-manager.js     engine-agnostic lifecycle: create, track (Map), clean up, HUD

js/app.js                  UI wiring, engine switching, slider, GSAP-missing fallback

js/vendor/gsap.min.js      GSAP 3.15.1

screenshots/               phase1-paint, phase2-paint (real) + phase1/phase2-performance (trace charts)

qa/                        test.py (functional suite), profile.py (CDP profiling), raw results (JSON)

INTERVIEW_GUIDE.md         interview preparation

SUBMISSION_CHECKLIST.md    requirement mapping and PASS/FAIL/NOT TESTED results

```

## 4. Phase 1: CSS keyframes analysis

Each reaction is a new `<span>`. The engine sets `left` and a `--sway` custom property, adds the class `reaction--css`, and the stylesheet's `float-css` keyframes (3 s, linear) animate `bottom` 0% to 85%, `margin-left` through a sine-like sway (nine keyframes), `font-size` 28px to 64px, and `opacity` (in at 8%, out by 100%). On `animationend` the manager removes the node; a 3.5 s fallback timer covers a missed event. These are deliberately layout-affecting properties, as the assignment asks.

## 5. Phase 1 paint-flashing screenshot

![Phase 1 paint flashing](screenshots/phase1-paint.png)

*Real Chromium output with Paint Flashing on, captured about 0.5 s after the second "Spawn 50" click (100 reactions active).*

## 6. Phase 1 performance screenshot

![Phase 1 trace](screenshots/phase1-performance.png)

*Generated from a real Chrome trace (see the honesty note above). Rows are main-thread events; red shading marks tasks longer than 50 ms; the lower panel is the `requestAnimationFrame` interval.*

## 7. Phase 1 observations

* **Green regions:** wide, overlapping green boxes cover the whole column the hearts travel through, not just the glyph. Because `bottom`, `margin-left` and `font-size` change every frame, each element's box changes and the area it moves through is repainted.

* **Main thread:** Layout runs continuously for the whole flight (about 135 layouts, about 1.46 s of Layout time in the median run), alongside Recalculate Style (about 0.56 s) and a very large number of Paint events (about 9,600). Layout activity shows up continuously across the trace.

* **Frames:** median frame interval 33.4 ms (about 30 fps), 18 main-thread tasks over 50 ms (median), average 25.6 fps over the run.

## 8. Phase 2: GSAP analysis

Each reaction gets one timeline: `y` rises 85% of the cached stage height over 3 s, `scale` goes 0.45 to 1.0, `opacity` fades in over 0.3 s and out over the last 0.8 s, and `x` sways through four 0.75 s `sine.inOut` segments. The glyph is rendered at 64px and only scaled between 0.45x and 1.0x, so it is never scaled up past its native size (no raster blur). `setSpeed(v)` calls `timeScale(v)` on every timeline in a `Set`, and new timelines read the current speed when created. `will-change` is intentionally not forced.

## 9. Phase 2 performance screenshot

![Phase 2 trace](screenshots/phase2-performance.png)

![Phase 2 paint flashing](screenshots/phase2-paint.png)

*Same load and throttle as Phase 1. Trace chart generated from real trace data; the paint-flashing image is real Chromium output.*

## 10. Phase 2 observations

* **Green regions:** only the HUD counters and the spawn row at the bottom flash. The hearts already in flight are not repainted, because `x/y/scale/opacity` changes do not affect layout and only need compositing.

* **Layout and paint:** Layout drops to about 0.15 s (from about 1.46 s) and Paint to about 15 ms over 165 events (from about 296 ms over about 9,600).

* **Frames:** median 16.7 ms (about 60 fps) between bursts, average 39.1 fps over the whole run, and 5 tasks over 50 ms (median) instead of 18.

* **A cost I did not expect:** each "Spawn 50" click causes a visible spike (about 200 to 250 ms frame, a long Scripting block) while 50 timelines are created at once under 6x throttle. Total main-thread busy time was similar in both phases (about 5.4 s vs about 5.1 s): GSAP moved work out of Layout/Paint into Scripting (about 1.76 s) and Commit (about 1.60 s). I did not open the Layers panel, so I have not confirmed that each icon got its own compositor layer. The higher Layerize, Commit, DrawFrame and raster-task counts are consistent with that but do not prove it.

* **Slow motion:** with the slider at 0.1x while 50 reactions were mid-flight, timeline time advanced about 0.15 s over 1.5 s of wall time and nothing restarted (automated check 10).

## 11. CSS versus GSAP comparison

Medians of 3 runs per phase, same workload (3 x Spawn 50, 600 ms apart, 6x CPU throttle). Raw data: `qa/profile.json`.

| Metric | Phase 1: CSS keyframes | Phase 2: GSAP |

|---|---|---|

| Average FPS over the run (rAF) | 25.6 | 39.1 |

| Median / p95 / max frame interval (ms) | 33.4 / 83.2 / 100.0 | 16.7 / 33.4 / 233.4 |

| Frames over 50 ms | 26 | 5 |

| Layout (total ms / count) | 1,465 / 135 | 149 / 162 |

| Recalculate Style (total ms) | 556 | 367 |

| Paint (total ms / events) | 296 / 9,588 | 14.5 / 165 |

| Scripting (total ms) | 217 | 1,756 |

| Commit (total ms) | 1,066 | 1,602 |

| Compositor DrawFrame count / raster tasks | 132 / 9,404 | 197 / 15,171 |

| Main-thread tasks: total ms / over 50 ms / longest ms | 5,403 / 18 / 98.5 | 5,096 / 5 / 241.1 |

Run-to-run variation was real: across the three runs of the final batch, Phase 1 averaged 24.5 to 29.6 fps and Phase 2 38.5 to 39.7 fps; an earlier batch gave about 27 and about 47 fps. The direction (Phase 2 smoother, far less Layout and Paint, but a spike per burst) was the same every time.

## 12. Runtime control and developer experience

A CSS keyframe animation is a fixed declarative recipe: duration and path are baked into the stylesheet. Changing the speed of an animation that is already running means editing `animation-duration` (which restarts or jumps the animation) or using the Web Animations API `playbackRate`. Randomised paths need per-element CSS variables set before the animation starts and cannot be changed afterwards. With GSAP each reaction is an object (a timeline): `timeScale()` changes speed instantly without restarting, `pause`, `reverse`, `seek` and `kill` are available, and paths are computed in JavaScript (random sway, start position, rise height from the current stage size), so any value can respond to application state at runtime. The trade-off is JavaScript work every frame (visible as the Scripting row in my Phase 2 trace) and a library dependency.

## 13. Pipeline considerations

Programmatic vector manipulation (SVG or canvas paths, or SVG/font glyphs driven by code) beats pre-rendered raster sequences when assets must change at runtime:

* **Tint and re-theme:** change a fill or a CSS variable instead of re-exporting every frame of every variant for each user preference or team colour.

* **Resolution independence:** vectors re-rasterise sharply at any device pixel ratio and during scale animations; bitmaps blur or need @2x/@3x sets.

* **Download size:** one SVG versus many frames multiplied by variants.

* **Memory:** no large decoded sprite sheets held in memory.

Raster sequences remain better for very complex painterly or particle-heavy art that would be expensive to draw procedurally. Here the reactions are emoji text glyphs; a production version should use SVG assets because colour-emoji fonts are bitmap-based on some platforms.

## 14. Performance testing methodology

Reproduce with `python3 qa/profile.py` (needs Python Playwright with Chromium). For each phase: fresh browser context at 1280x800; `Emulation.setCPUThrottlingRate` = 6; start a CDP trace (`devtools.timeline` and related categories); click **Spawn 50 Reactions** three times 600 ms apart; wait until all reactions complete; stop the trace. Main-thread event totals are computed from the renderer main thread (nested scripting events are merged so they are not double counted). FPS is the `requestAnimationFrame` rate across the run. Three runs per phase, medians reported. Paint flashing uses `Overlay.setShowPaintRects` in a separate run, because the overlay adds its own cost.

**Manual steps for DevTools screenshots on your own machine** (recommended, to replace or supplement the generated charts):

1. Open the app in Chrome, press `F12`; `Cmd/Ctrl+Shift+P`, "Show Rendering"; tick **Paint flashing** and **Frame Rendering Stats**.

2. Performance panel, gear icon, **CPU: 6x slowdown**.

3. Phase 1: Record, click **Spawn 50 Reactions** two or three times, wait about 4 s, Stop. Save the paint-flashing view as `screenshots/phase1-paint.png` and the flame chart as `screenshots/phase1-performance.png`.

4. Phase 2: repeat identically, save `screenshots/phase2-performance.png`, then drag the slider to 0.1x mid-flight.

## 15. Known limitations

* Profiling was done in headless Chromium with software compositing, not on a physical GPU, a real phone, or desktop Chrome with DevTools open. Absolute numbers are not transferable.

* The benchmark performance charts are trace-derived, not screenshots of the DevTools interface. The desktop Chrome captures are included separately. Layer promotion was not inspected in the Layers panel.

* The slider was exercised programmatically (value set plus `input` event); a physical mouse drag was not tested.

* Phase 1 speed cannot change mid-flight, by design.

* Switching engines clears in-flight reactions and resets counters instead of migrating them.

* "Spawn 50" inserts all nodes in one frame (a deliberate worst case), so the 50 reactions of one click rise as a flat row.

* GSAP creates 50 timelines in one frame, causing the per-burst spike seen in the Phase 2 trace. Staggering creation or pooling timelines would reduce it.

* Emoji rendering depends on the OS emoji font. The Phase 1 `bottom: 85%` is relative to stage height, while GSAP uses the stage height cached at spawn, so resizing mid-flight only affects new reactions. Target browser: current Chrome.

## 16. Final submission checklist

- [x] Phase 1 and Phase 2 functional and easily testable (engine radio buttons)

- [x] Floating elements removed and destroyed on completion (verified: DOM nodes and timelines return to 0)

- [x] Written reflection questions answered (sections 12 and 13)

- [x] Paint-flashing screenshots for both phases (real Chromium output)

- [x] Phase 1 and Phase 2 performance evidence (trace-derived charts, real data)

- [ ] **DevTools UI screenshots on your own machine (recommended; see section 14)**