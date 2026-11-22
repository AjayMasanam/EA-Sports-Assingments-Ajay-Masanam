# Interview Guide (matches the code in this submission)

Know this project cold: the brief says you may use AI but must explain the code and reasoning. Read each file once before the interview and be able to point at the lines mentioned here.

## 30-second pitch

"I built the same floating-reactions effect twice. Phase 1 uses CSS keyframes that animate `bottom`, `margin-left` and `font-size`, which are layout properties, so the browser re-lays-out and repaints every frame. Phase 2 uses one GSAP timeline per reaction animating only `x`, `y`, `scale` and `opacity`, which skips layout and paint. A manager owns the lifecycle and cleanup; the two engines only know how to animate. I profiled both with 6x CPU throttling: Phase 2 had far less Layout and Paint and better frame times, but it also showed a scripting spike when 50 timelines are created at once."

## Architecture

```
app.js  ──► ReactionManager ──► active engine (CssEngine | GsapEngine)
 UI/events    lifecycle, Map,        start(el, ctx, done) -> { cancel }
              counters, HUD          setSpeed(v), timelineCount
```

* **Engine contract:** `start(el, ctx, done)` begins the animation and returns a handle with `cancel()`. `ctx` = `{ leftPct, sway, stageHeight }`. `done` is called once when the animation ends. This is what lets the manager stay engine-agnostic.
* **`reaction-manager.js`:** `active` is a `Map` (element to handle) and is the single source of truth. `spawn()` builds all nodes into a `DocumentFragment`, inserts them in one write, then starts each. `finish(el, completed)` is idempotent: it does nothing if the element is no longer in the Map, otherwise it deletes it, calls `handle.cancel()`, and calls `el.remove()`. `clearAll()` runs on engine switch, Reset and `pagehide`. `notify()` coalesces HUD updates into one `requestAnimationFrame`. A `ResizeObserver` caches the stage height so spawning never reads layout after writing.
* **`css-engine.js`:** sets `left` and `--sway`, adds the class `reaction--css`, listens for `animationend` (ignoring events from other targets) and also sets a 3.5 s fallback timer. `cancel()` removes the listener and clears the timer. `setSpeed` is a no-op, by design.
* **`gsap-engine.js`:** per reaction creates one timeline: `y` (3 s, linear), `scale` 0.45 to 1 (3 s), `opacity` in 0.3 s and out over the last 0.8 s, and four 0.75 s `sine.inOut` sway segments on `x`. A `Set` holds all live timelines so `setSpeed(v)` can call `timeScale(v)` on each. `onComplete` removes the timeline from the Set and calls `done`; `cancel()` calls `kill()`. New timelines call `tl.timeScale(speed)` on creation so they inherit the slider.
* **`app.js`:** wires radios, buttons and slider; hides the slider outside Phase 2; disables Phase 2 with a message if `window.gsap` is missing; exposes `__reactions` and `__gsapEngine` for testing.

## Key concepts and how they appear in this code

* **Animation lifecycle:** create node, insert, start, animate, end event, `finish()`, remove. Both engines end in the same `finish()`.
* **Memory management and zombie nodes:** one Map; idempotent `finish()`; `kill()` plus Set deletion for timelines; listener and timer cleanup for CSS; `clearAll()` on switch, Reset and page hide. Verified: after bursts, DOM nodes, active count and timeline count all return to 0.
* **Data structures:** `Map` for element-to-handle with O(1) add/remove; `Set` for timelines so `forEach` can retime them and deletion is O(1).
* **DOM batching:** `DocumentFragment` means one insertion for 50 nodes. Caching stage height via `ResizeObserver` avoids forced synchronous layout.
* **Layout, Paint, Composite:** changing geometry (`bottom`, `margin-left`, `font-size`) invalidates layout and then paint; changing `transform` and `opacity` can skip both and be handled at the compositor.
* **GPU compositing limits:** not guaranteed. Chrome decides which elements get their own layer; GSAP alone is not GPU acceleration, because it still runs JavaScript every frame via `requestAnimationFrame`. My profiling ran with software compositing, and I did not inspect the Layers panel, so I cannot claim each icon got a layer.
* **Resolution independence:** the glyph is 64px and only scaled from 0.45x to 1.0x, never scaled up past native size, so it stays crisp. A production version would use SVG.
* **`will-change`:** deliberately not used. Forcing 50+ layers costs memory, and Chrome already promotes actively animating transform/opacity elements.
* **`timeScale()`:** scales how fast a timeline's playhead advances each tick, so position is preserved and nothing restarts.

## What I measured (3 runs per phase, medians, headless Chromium 157, 6x throttle)

| | Phase 1 CSS | Phase 2 GSAP |
|---|---|---|
| Avg FPS | 25.6 | 39.1 |
| Layout | 1,465 ms | 149 ms |
| Paint | 296 ms (about 9.6k events) | 14.5 ms (165 events) |
| Scripting | 217 ms | 1,756 ms |
| Tasks over 50 ms | 18 | 5 |
| Total main-thread busy | 5.4 s | 5.1 s |

Say plainly: this was headless Chromium with software compositing, not DevTools on a real GPU. The "performance" images are charts built from real trace data, not DevTools screenshots. Results varied between runs; the direction did not. If time allows before the interview, repeat the run in desktop Chrome and replace the screenshots.

## Likely questions and sample answers

1. **Why is Phase 1 slower?** "It changes geometry every frame, so the browser runs Style, Layout, Paint, Composite on the main thread for every element. In my trace Layout was about 1.5 s with 18 long tasks, versus about 0.15 s in Phase 2."
2. **Why are transform and opacity cheaper?** "They do not change geometry, so layout is skipped, and Chrome can often composite them without repainting. 'Often': Chrome decides layer promotion, and I would check the Layers panel to confirm."
3. **Is GSAP faster than CSS?** "Not inherently. GSAP is JavaScript ticking each frame, which showed up as about 1.8 s of Scripting. The win here is that it animates cheap properties. A CSS animation of `transform` would also be cheap, but I could not change its speed mid-flight as easily."
4. **Why did Phase 2 spike on each burst?** "Creating 50 timelines in one frame under 6x throttle is a lot of synchronous JavaScript. It is a deliberate worst case; staggering creation or pooling timelines would smooth it. I observed the spike in the trace but did not profile it further."
5. **How does `timeScale` avoid restarting animations?** "It changes the rate at which the playhead advances. Position is kept. I tested it: at 0.1x with reactions about 1 s in, the timeline time advanced about 0.15 s over 1.5 s and nothing reset."
6. **How do you prevent leaks?** "One Map as source of truth, idempotent `finish()`, `kill()` plus Set deletion, listener and timer cleanup, and `clearAll()` on switch, Reset and `pagehide`. The HUD and `__reactions.stats()` show counts returning to 0."
7. **Why is `finish()` idempotent?** "For the CSS engine both `animationend` and the fallback timer can fire. Whichever comes second finds nothing in the Map and returns."
8. **Why a fallback timer?** "`animationend` does not fire if the element is hidden or the animation is cancelled by style changes. 3.5 s is 0.5 s after the 3 s animation."
9. **Why a DocumentFragment and a cached height?** "One DOM write for the batch, and no layout read after writing, which would force synchronous layout."
10. **Why clear on engine switch?** "To avoid mixed engines and orphaned timelines or listeners."
11. **Why no `will-change`?** "It forces a layer per element up front; 50+ layers cost memory. Chrome promotes animating transform/opacity elements anyway."
12. **How would you improve it?** "SVG assets, object pooling to reduce DOM churn and GC, stagger timeline creation, compare against the Web Animations API, add a frame-time overlay, and automated browser tests in CI."
13. **How did you test it?** "A Playwright suite in `qa/test.py` drives real Chromium: 17 checks including 100 concurrent reactions, lifespan about 3 s, sway, scale and fade, slider extremes, in-flight retiming, engine switching, Reset, cleanup and console errors. The slider was set programmatically, not dragged by hand."
14. **What would you do with real device data?** "Repeat on a mid-range phone with remote debugging, because software-compositing numbers on a desktop CI machine do not predict mobile GPU behaviour."

## Be ready to say honestly

* Which statements you measured and which are expectations.
* That the performance charts are trace-derived and that Layers-panel promotion was not verified.
* The code was produced with AI assistance; you reviewed and understood it (read each file before the interview).
