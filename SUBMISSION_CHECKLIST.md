# Submission Checklist and Test Report

Generated from real runs of `qa/test.py` (headless Chromium 157, Linux) and `qa/profile.py`. Nothing here is marked PASS unless it was executed.

## 1. Requirement mapping (EA assignment)

| Requirement | Where | Status |
|---|---|---|
| Interactive stage with control bar | `index.html`, `css/style.css` | Done, tested |
| Four reactions and Send Reaction | `index.html`, `app.js` | Done, tested (check 2) |
| 3 s upward flight, sway, scale up, fade out | both engines | Done, tested (checks 3 to 8) |
| Removed from DOM on completion, no leaks | `reaction-manager.js` | Done, tested (checks 13, 14) |
| 50+ concurrent reactions | Spawn 50 button | Done, tested with 100 (check 5) |
| Phase 1: CSS `@keyframes` on layout properties | `css/style.css`, `css-engine.js` | Done |
| Phase 2: GSAP on `x`, `y`, `scale`, `opacity` only | `gsap-engine.js` | Done |
| Resolution independence | 64px glyph scaled at most 1.0x | Done (emoji glyphs; SVG recommended for production) |
| Slider 0.1 to 2.0, step 0.1, default 1.0, live `timeScale` | `index.html`, `app.js`, `gsap-engine.js` | Done, tested (checks 9, 10) |
| Chrome profiling: Paint Flashing, 6x CPU, 50+ reactions, both phases | `qa/profile.py`, `screenshots/` | **Done in headless Chromium via CDP; DevTools UI screenshots NOT captured** |
| README with screenshots, observations, reflections | `README.md` | Done |

## 2. Automated test results

| # | Test | Result | Evidence |
|---|---|---|---|
| 1 | Application loads (all scripts, GSAP present) | **PASS** | - |
| 2 | All four reactions spawn (Send Reaction, both engines) | **PASS** | 8 combos |
| 3 | CSS (Phase 1) animation runs and completes | **PASS** | rise=442px |
| 4 | GSAP (Phase 2) animation runs and completes | **PASS** | rise=423px |
| 5 | 100 concurrent reactions (2x Spawn 50), both engines | **PASS** | css active=100, gsap active=100, gsap timelines=100 |
| 6 | Reaction lifespan about 3 s at 1x | **PASS** | css=3.03s gsap=3.04s (includes ~0.05s polling) |
| 7 | Horizontal sway (x range and direction reversals) | **PASS** | css range=68px (2 reversals), gsap range=54px (2 reversals) |
| 8 | Scaling up and fading out | **PASS** | width ratio css=2.29 gsap=2.22; last opacity css=0.00 gsap=0.01 |
| 9 | Speed slider: min 0.1, max 2.0, step 0.1, default 1.0; 0.1x/2.0x/1.0x reach all 50 timelines | **PASS** | attrs=['0.1', '2.0', '0.1', '1.0', 'range']; outputs=['0.1x', '2.0x', '1.0x'] |
| 10 | Speed change affects reactions already in flight (no restart) and new ones inherit it | **PASS** | timeline time before=1.00s after-change=1.02s 1.5s later=1.17s; new reaction timeScale=0.1 |
| 11 | Engine switching clears in-flight reactions, updates label, toggles slider visibility | **PASS** | labels='Phase 1 · CSS Keyframes' / 'Phase 2 · GSAP' |
| 12 | Reset clears reactions and zeroes all counters (incl. HUD) | **PASS** | hud=['0', '0', '0', '0'] |
| 13 | Completed reactions are removed from the DOM (50 burst, both engines) | **PASS** | css: 3.05s, dom=0, completed=50/50; gsap: 3.05s, dom=0, completed=50/50 |
| 14 | GSAP timelines are destroyed (after completion and after Reset) | **PASS** | after completion timelines=0, after reset=0 |
| 15 | No console errors / page errors / failed requests | **PASS** | - |
| 16 | Extra: works over file:// (double-click index.html) | **PASS** | - |
| 17 | Extra: GSAP blocked -> Phase 2 disabled with message, Phase 1 still works | **PASS** | GSAP failed to load, so Phase 2 is disabled. Phase 1 still works. |

Totals: 17 PASS, 0 FAIL.

Note on the test script: two tests failed on the first run because of mistakes in the test script itself (sway reversals were miscounted on frames where x did not move; the slider's default was read from the DOM property, which normalises `1.0` to `1`, instead of the HTML attribute). I fixed the script, not the app, and reran; the table above is from the final run.

## 3. Profiling status

| Item | Status |
|---|---|
| Paint Flashing enabled, Phase 1 and Phase 2 | COMPLETED (real Chromium output): `screenshots/phase1-paint.png`, `phase2-paint.png` |
| 6x CPU throttling, 50+ concurrent reactions, same workload both phases | COMPLETED (CDP `Emulation.setCPUThrottlingRate`, 3 x Spawn 50, 3 runs per phase) |
| Performance trace data | COMPLETED: `qa/profile.json`, `qa/medians.json` |
| `phase1-performance.png`, `phase2-performance.png` | **Charts generated from the real trace data, not DevTools flame-chart screenshots** |
| Frame Rendering Stats overlay | NOT TESTED (FPS measured with a `requestAnimationFrame` counter instead) |
| DevTools UI Performance panel recording | NOT COMPLETED: needs desktop Chrome on your machine |
| Layers panel check of compositor layer promotion | NOT TESTED |
| Real GPU / physical mobile device | NOT TESTED |

## 4. Not tested

* Physical mouse drag of the slider (value set programmatically with an `input` event).
* Browsers other than Chromium; macOS/Windows emoji fonts.
* Mobile touch interaction and very small viewports (layout is responsive but not exercised).

## 5. Remaining tasks for you

- [ ] Open the project in desktop Chrome and click through both phases once by hand.
- [ ] Repeat the DevTools profiling (README section 14) and replace or add real DevTools screenshots in `screenshots/`.
- [ ] Update the README numbers if your machine's results differ, and keep the honesty note accurate.
- [ ] Read every source file once and rehearse `INTERVIEW_GUIDE.md`.
