# Frict: native Swift port, scope and build prompt

This document is a prompt for an agent that will port Frict from the web build in this repository (`index.html`, `manifest.json`, two icons) to a native iOS app written in Swift. The finished app has to look, sound and play the same as the web build. Read all of `index.html` before writing any Swift. Its comments explain why most numbers are what they are, and several of them describe bugs that were already fixed once and must not come back.

## 1. What you are porting

The whole game lives in one 1,778-line HTML file. There is no DOM UI. Everything is drawn into a single `<canvas>` with the Canvas 2D API, at a fixed logical size of 320 by 693 field units, scaled uniformly to fit the screen.

| Part of `index.html` | Approx. lines | What it holds |
|---|---|---|
| CSS and orientation script | 1–75 | Full-bleed black page, a CSS rotation that fakes a portrait lock on iOS |
| Embedded assets (`const A`) | one long line | Six audio clips and one PNG as base64 data URIs |
| Config and constants | ~80–200 | Field geometry, physics constants (`P`), fonts, sound schemes |
| Audio | ~140–241 | Web Audio setup, decoding, `play(name, vol, delay)` |
| Canvas sizing | ~243–366 | Device-pixel fitting, rotation handling, resize auditing |
| Scenes and menus | ~369–570 | Scene enum, circle slots, every menu's buttons, tutorial text |
| Game state, deal, save/load | ~572–730 | Ball dealing with the razor pity counter, save and replay of an in-flight shot |
| Input | ~732–811 | Pointer and keyboard handling, hit testing |
| Rules and physics | ~813–1009 | Scoring, collision, friction, growth, game over |
| Drawing | ~1011–1694 | Balls, buttons, text, every screen |
| Main loop and boot | ~1696–1774 | Fixed 1/120 s timestep with interpolation, debug overlay, boot |

A large share of the file exists to work around the browser: device-pixel-ratio fitting, CSS rotation for a portrait lock that iOS Safari does not offer, `offsetX` versus bounding-box pointer math, font loading, a suspended `AudioContext`, and `localStorage` stalls. Most of that code disappears in a native app. The game logic, the drawing and the timing constants carry over almost line for line.

Expect roughly 2,000 to 3,000 lines of Swift.

## 2. Target and technology choices

Use these unless the human says otherwise.

1. **Platform.** iPhone only, iOS 17 or later, portrait only. Set `UISupportedInterfaceOrientations` to portrait. Delete all of the rotation logic (`PORTRAIT_LOCK`, `data-rot`, `rotDir`, `rotated`, the landscape media query, and the rotated branch of `canvasPoint`).
2. **UI framework.** One full-screen `UIView` subclass that draws with Core Graphics in `draw(_:)`, hosted in a plain `UIViewController`. Do not use SpriteKit, SwiftUI shapes, or `UIButton`s. Canvas 2D maps almost one to one onto `CGContext` (`arc`, `quadCurve`, `fillPath(using: .evenOdd)`, `setLineDash`, `clip`, `translateBy`, `rotate`, `scaleBy`, `setAlpha`), and matching the original is far easier when the drawing code reads the same as the JS. A SwiftUI `Canvas` inside a `TimelineView` is an acceptable alternative if you prefer it, but keep the drawing code imperative and structured the same way as the JS.
3. **Frame loop.** `CADisplayLink` with `preferredFrameRateRange` of 60 to 120. Add `CADisableMinimumFrameDurationOnPhone = YES` to Info.plist so ProMotion phones run at 120 Hz. Port `frame()` exactly: accumulate real time, cap a single frame's `dt` at 0.05 s, step the world in fixed `STEP = 1/120` increments (guard of 60 steps), and draw with `alphaT = acc / STEP` interpolation. Call `setNeedsDisplay()` each tick.
4. **Coordinate system.** Keep every constant in field units (320 wide, 693 tall). In `draw(_:)`, compute `scale = min(bounds.width / 320, bounds.height / 693)`, centre the field, fill the whole view black, and draw through that one uniform transform. This replaces `measure()`, `remeasure()`, `auditSize()` and `watchDpr()`. Keep `q` meaning "device pixels per field unit" (`scale * contentScaleFactor`), because `hair()` uses it so that 1-unit lines never fall below one device pixel.
5. **Language and structure.** Swift 5.9 or later, no third-party dependencies. Suggested files:
   - `Constants.swift`: `W`, `H`, `V()`, `P`, `SLOT`, `LAY`, colours, every named constant.
   - `GameState.swift`: statics, ball, score, turn, pity, the deal, save and load, replay lock. No UIKit imports, so it can be unit tested.
   - `Physics.swift`: `stepBall`, `stepWorld`, `land`, `hitStatic`, `removeStatic`, `gameOver`, `endRound`, `spinKick`.
   - `Scenes.swift`: the `SC` enum, `MENUS`, `ask()` confirm flow, tutorial text.
   - `Renderer.swift`: every `draw*` function, `drawBall`, `drawButton`, `inkLabel`, `inkText`, `body`, `edgeBox`, `baseline`, `chevron`, icons.
   - `TextMetrics.swift`: Canvas `measureText` equivalents (see section 5).
   - `SVGPath.swift`: a small parser that turns the `Path2D` strings into `CGPath`.
   - `Audio.swift`: the sound engine.
   - `Store.swift`: `UserDefaults` persistence.
   - `GameView.swift` and `GameViewController.swift`: display link, touches, lifecycle.

## 3. Behaviour that must match exactly

Port these as literal translations. Keep the JS names where you can, and keep the comments that explain a number, so a later diff against `index.html` is easy.

### Physics and rules

- All of `P`: speed 420, friction 0.6, bounce 0.92, stopAt 5, gripAt 70, grip 0.30, growTime 0.50, sweep 100°/s, aimDelay 0.03, startBalls 0.
- Geometry: `FOOT 45`, `ARROW_DROP 70`, `TOP 48`, `TOP_LINE 1`, `SIDE 1`, `SHOT_R 14`, `RAZOR_SCALE 1.1`, `POP_PULL 0.65`, `FIELD_TOP`, `FIELD_L`, `FIELD_R`, and `P.launch` / `P.baseline` from `layout()`.
- `stepBall`: substep count `max(1, min(8, ceil(sp*dt/3)))`; per-substep friction with the grip ramp; static collisions iterated from the end of the array backwards; razor passes through and removes; reflection with bounce; walls checked after statics; the `armed` and baseline game-over rules; off-screen handling at `H + 40`; rest when speed falls below `stopAt`.
- Ball states `aim`, `fly`, `rest`, `spend`, `grow`, with `REST_HOLD 1/6`, the 0.32 s razor spend animation, and the grow-to-clearance logic in `land()` including its clamping order.
- Aim sweep: ±80°, starts rightward on every press, `aimDelay` dead time.
- `hitStatic` cooldown of 0.18 s, two hits to remove, `mult()` for double and triple, razor scores 1 per ball.
- `dealType` with the razor pity counter: first ball always normal, then `(2.5 + pity/2)%` razor chance, otherwise 7% triple, 11% double. Use `Double.random(in: 0..<1)` in place of `Math.random()`, behind a protocol so tests can inject a seeded generator.
- Death sequence: `DEATH_HOLD 0.35`, `DEATH_SLOWMO 0.30`, `OVER_DELAY 0.15` on the game-over sound, `overFlash` decay.
- Score popups: rise 26 units/s, fade over 0.9 s, clamp x to 16…W−16.
- Spin: razor spins from speed, striped balls get a cosmetic kick from `spinKick` that decays by `0.22^dt` and is clamped to ±21.
- `newGame()` random placement loop is currently a no-op because `startBalls` is 0. Port it anyway.

### Scenes and menus

Port all ten scenes: `MENU, PLAYMENU, TUT, ABOUT, SETTINGS, HISCORE, GAME, OVER, CONFIRM, PAUSE`, with every button in `MENUS[...]`, every label string, font size, rotation and line height, and the `LAY` nudge table (`confirm.reset`, `confirm.abandon`, `confirm.restart`, `confirm.yes`, `over.heading`, `pause.main`). `LAY_ITEMS` and `LAY_DEF` only feed a calibration panel that is not in this build, so leave them out.

Things that are easy to miss:

- Play skips the play menu and starts a game when nothing is resumable.
- Resume on the play menu draws at 0.28 alpha when not resumable, and the tutorial's back chevron does the same on step 0.
- The `ask()` confirm screen stores its own text, font size, rotation, yes-action, back scene and whether it draws over the field.
- The pause hit area during play is the rectangle `x <= 54, y >= SCORE_NUM_Y - 30`, not a circle, and it is checked before a press becomes an aim.
- The sound toggle and the sound-scheme toggle do not play the usual menu tap (`sound: true`, `quiet: true`). The scheme toggle plays the new scheme's tap instead. Turning sound on plays a tap.
- Reset high score writes 0 and returns to Settings.
- The tutorial has five steps with drawn illustrations whose coordinates are computed from `REST1`, `GROWN1`, `TUT_RULE` and the real field constants. Port the illustration code, not a screenshot of it.
- The wordmark block (`frict` / `forever`) sizes the subtitle so its ink width equals the mark's ink width, with one correction pass. This depends on text ink metrics (section 5).
- The version stamp reads `v 0.2a` and appears only on the title screen.
- The "best" number and caption turn bright green when `score > 0 && score >= high`.

### Save, resume and replay

- Keys and shapes stay the same as `localStorage` (`frict.high`, `frict.game`, `frict.sound`, `frict.scheme`, `frict.walls`), stored as JSON in `UserDefaults` or as `Codable` structs. Nothing migrates from the web app's storage. The human has confirmed that scores do not need to carry over.
- The save written at launch of a shot (`saveLaunch`) is synchronous and includes the ball's position and velocity. On load, that shot is replayed: the ball sits at the launch point with the arrow at the fired angle for `REPLAY_HOLD 0.5` s, then fires itself. Input is locked during replay except for the pause area.
- Periodic save every 1.5 s (`flushSave`) and on backgrounding. Native `UserDefaults` writes are cheap, but keep the deferred-save structure so behaviour stays identical.
- Handle `sceneDidEnterBackground` / `willResignActive` where the JS handles `pagehide` and `visibilitychange`. On return to the foreground, restart the audio engine if it stopped.
- The edges setting defaults to off on a touch device (`wallsDefault()` returns false on iOS), and stays whatever the player set after that.

## 4. Drawing

- Colours: `#00ff00` live green, `#007f00` quiet green (`QUIET`), `#004600` (`HUSH`), `#00b500` (`TUT_MID`), `#d8f5d8` body text, black background. Use sRGB colour space explicitly so the greens match the browser.
- Ball proportions from `drawBall`: ring stroke at radius `0.9375r` with width `0.125r`, inner disc `0.75r`, stripe bars `0.17r` wide rotated −45° plus spin, razor as a 12-point star with valleys at `0.72r` and a hole at `0.55r` filled even-odd.
- Rings everywhere use `RING_K = 12/91` of the radius. Toggles fill an inner disc at `TOGGLE_DR 0.78`.
- `edgeBox` uses quadratic curves with radius 34 at the bottom corners and `SHOT_R/2` at the top. The baseline is 11 dashes across 21 equal parts.
- The arrow, speaker, waves, mute and restart icons are SVG path strings passed to `Path2D`. Write a small SVG path parser that supports `M m L l H h V v C c Z z` and the relative arc command `a`, since the speaker icons use arcs. Converting an SVG arc to Bézier curves is a known algorithm (the W3C endpoint-to-centre conversion). Test it against the original by rendering both.
- The `A.arrow` PNG in the asset blob is not used anywhere. The arrow is drawn from `ARROW_PATH`. Do not ship the PNG.
- Global alpha, save/restore, and the semi-transparent scrims (`0.88`, `0.91 + overFlash*0.06`) map to `CGContext.setAlpha` and filled rects.
- Antialiasing on, and no image interpolation is involved since everything is vector.

## 5. Text, which is the hardest part to match

The game relies on Canvas text behaviour in ways that do not map directly onto Core Text. Plan for this to take the most iteration.

1. **Fonts.** Bundle the Google Fonts TTFs for Fredoka (weights 400, 500, 600, 700) and Barlow (300, 400, 700). Both are under the SIL Open Font License, so include the OFL text in the app. Fredoka on Google Fonts is a variable font, so download the static instances or create named instances for 500 and 600. Register them with `UIAppFonts`. The web build falls back to Trebuchet MS or Helvetica only if loading fails, so ignore the fallbacks.
2. **`textBaseline = "middle"`.** Canvas places the middle of the em box at y. In Core Text you draw at a baseline, so compute `baselineY = y + (ascent - descent) / 2` using the font's ascent and descent. Browsers take these from the font's `hhea` or `OS/2` tables, and Core Text may report different values. Check against a browser screenshot and adjust the formula, not the per-button numbers.
3. **`textBaseline = "top"`** (used by `body()`) and **`"alphabetic"`** (scores, version stamp) need the same treatment.
4. **`measureText` fields.** The code uses `width`, `actualBoundingBoxAscent`, `actualBoundingBoxDescent`, `actualBoundingBoxLeft` and `actualBoundingBoxRight`. Build a `measure(text, font)` that returns all five from `CTLineGetTypographicBounds` (advance width) and `CTLineGetImageBounds` or `CTLineGetBoundsWithOptions(.useGlyphPathBounds)` (ink box), converted into Canvas's sign conventions relative to the alignment point and baseline mode.
5. **`inkLabel`** centres a block on its ink, with the "tall word versus x-height word with a tail" rule from `faceM`. Port it exactly once the metrics in step 4 are right.
6. **`inkText`** offsets by the ink's left or right bearing so digits sit flush to the margin.
7. **`body()`** does greedy word wrap at `maxw` with a line height of `size * 1.42`. Port the loop rather than using `NSAttributedString` wrapping, which breaks lines differently.
8. **Rotated labels.** Most button labels are drawn after `translate` and `rotate(rot°)`. Keep that order.
9. **Multi-line labels** split on `\n` and space lines at `size * lh`, centred around y.

## 6. Sound

- Extract the six base64 clips from `const A` in `index.html` into bundle files, keeping their formats: `pop_original.mp3`, `tap_original.mp3`, `over_original.mp3`, `pop_tight.wav`, `tap_tight.wav`, `over_bubble.mp3`. A short script that pulls the data URIs out of the HTML and decodes them is enough. Do not re-encode them.
- Two schemes: `original` (pop_original, tap_original, over_original) and `bubble` (pop_tight, tap_tight, over_bubble). Scheme index persists as `frict.scheme`.
- Volumes: tap 0.5, pop 0.55, game over 0.7 delayed by 0.15 s.
- Use `AVAudioEngine` with all six files decoded into `AVAudioPCMBuffer`s at startup, and a small pool of `AVAudioPlayerNode`s (eight is plenty) so overlapping pops do not cut each other off. Schedule the delayed game-over sound against the audio clock (`AVAudioTime` with a host-time offset), not with `DispatchQueue.asyncAfter`, matching the JS comment about drift.
- Latency matters here. The original's comments describe trimming the clips so the attack lands within a quarter millisecond. Set `AVAudioSession.setPreferredIOBufferDuration(0.005)` and start the engine at launch, not on first tap.
- Audio session category: `.ambient`. That mixes with the player's music and respects the silent switch, which is how the web app behaves in Safari. The human has confirmed this choice.
- Respect the sound on/off setting (`frict.sound`, default on).
- Restart the engine after interruptions and route changes (`AVAudioSession.interruptionNotification`, `AVAudioEngineConfigurationChange`).

## 7. Input

- Map `pointerdown` on the canvas to `touchesBegan`, and `pointerup` / `pointercancel` to `touchesEnded` / `touchesCancelled`. Convert the touch location into field units by inverting the same transform used for drawing.
- Set `isMultipleTouchEnabled = false`. In the web build a second finger calls `press()` again and restarts the aim; one-touch handling avoids that and nobody will notice the difference. Mention it to the human.
- Hit testing is `hypot(p - centre) <= r` against each button in the current scene's `MENUS` list, using `btnY()` for the y.
- Defer system edge gestures at the bottom (`preferredScreenEdgesDeferringSystemGestures = .bottom`) and auto-hide the home indicator, because the pause target sits in the bottom-left corner. The JS comment about missing presses there is the reason.
- Hardware keyboard support (space or return to aim and fire, escape to menu) via `pressesBegan` / `pressesEnded` is optional. Add it if it is cheap.

## 8. Screen, launch and packaging

- Full-bleed black. The field's aspect ratio (0.4618) was chosen to match current iPhone screens. Draw edge to edge, ignoring safe-area insets, as the home-screen web app does with `viewport-fit=cover`.
- Status bar: hidden (`prefersStatusBarHidden = true`). The human has confirmed this.
- Launch screen: plain black, no logo, so the app opens to the menu with no flash.
- App icon: the human will supply a 1024×1024 icon later. Until then, upscale `icon-512.png` into the asset catalog as a placeholder.
- Display name `Frict`, which the human has confirmed. Use `com.example.frict` as a placeholder bundle identifier and leave the development team unset; the human will supply both later.
- Drop: the Google Fonts link, the manifest, `window.claude.hot` snapshot and boot hooks, `document.fonts.load`, the font-readiness check in `titleM` (fonts are available synchronously in a native app, so compute once), and the `?debug` query flag. If you keep the debug overlay, enable it with a launch argument instead.

## 9. Order of work

1. Create the Xcode project, Info.plist settings, fonts, extracted audio files and a black launch screen. Confirm it builds and runs on a simulator. Build the reference half of the screenshot harness (section 10a) now, so the web build's screenshots exist before you draw anything.
2. Port constants and the pure game model (`GameState`, `Physics`) with no rendering. Write unit tests (section 10) before going further.
3. Build the renderer for the game scene only: field box, baseline, balls, arrow, scores, popups. Wire up the display link and touches so a round is playable.
4. Add text metrics and verify `inkLabel`, `inkText` and `body` with the screenshot harness (section 10a) before porting menus.
5. Port every remaining scene and the confirm flow.
6. Add audio.
7. Add persistence, resume and the shot replay.
8. Lifecycle handling, edge-gesture deferral, status bar, icon.
9. Run the full comparison pass in section 10 and fix differences.

## 10. How to verify it matches

1. **Physics golden traces.** Copy the rule and physics functions from `index.html` into a small Node script (they have no DOM dependencies apart from `play()`, which you can stub). For a set of fixed fields and launch angles, record the ball's position every step and the final score. Run the same cases through the Swift `Physics` in XCTest and require agreement to within 1e-6 units. Seed or stub randomness on both sides.
2. **Screenshots.** This loop runs with no input from the human. See section 10a.
3. **Sound.** Check by ear on a device: each scheme, each event, overlapping pops, the delayed game-over sound, the silent switch, and music playing in the background.
4. **Resume.** Fire a shot and force-quit the app while the ball is moving. On relaunch, the play menu should offer resume, and resuming should replay that shot.
5. **Frame rate.** On a ProMotion device, confirm 120 Hz and that the game runs at the same speed as on a 60 Hz device.

## 10a. Automated screenshot comparison

Build this harness in step 1 of section 9, before any drawing code, and run it after every change to the renderer. It needs a Mac with Xcode and runs entirely through `xcodebuild test`, so nobody has to tap anything or look at a screen.

### Why the reference comes from WKWebView on the same simulator

Render the original game in a `WKWebView` on the same iOS simulator the native app is tested on. A home-screen web app on an iPhone runs in WebKit, and WebKit draws canvas text with Core Text, the same text engine the native app uses. Same device, same pixel density, same font rasterizer, and same fonts means a correct port should differ only by antialiasing noise. A Chromium render on Linux or macOS would differ in text rendering on every label and would hide real layout errors in that noise.

### Pieces to build

1. **A scene hook in the web build.** Do not edit `index.html`. At test time, load its contents as a string, insert a block just before the final `})();` of the main IIFE that sets `window.__frict` to an object with functions to set the scene, the tutorial step, the toggles (`soundOn`, `scheme`, `wallsOn`), the score and high score, the confirm prompt (call `ask(...)` with the same arguments the buttons use), and to load a fixed field of statics and a ball in a given state. Also replace `Math.random` with a seeded generator and stop the animation loop so a test can call `stepWorld` and `render` by hand. Inline the fonts as base64 `@font-face` rules, since the simulator may have no network.
2. **A reference host.** A test-only helper that creates a `WKWebView` sized 375×812 points (iPhone 13 mini, 3x), loads the patched HTML, waits for fonts with `document.fonts.ready`, calls `window.__frict` to set up a case, renders one frame, and captures the view with `WKWebView.takeSnapshot(with:)`.
3. **A matching hook in the native app.** A debug-only `GameState` initializer, or launch arguments such as `-scene settings -tutStep 3 -walls 1`, that sets up the same cases. Snapshot the native `GameView` at the same size with `UIGraphicsImageRenderer` and `drawHierarchy(in:afterScreenUpdates:)`.
4. **A case list.** One definition shared by both sides, in JSON. Cover the main menu, play menu with and without a resumable game, each of the three confirm prompts (reset, abandon, restart), settings in every combination of sound on/off, original/bubble and edges on/off, high score at 0 and at a three-digit number, about, all five tutorial steps with edges on and off, pause, game over, and mid-game fields showing every ball type, both hit states, the aim arrow at a few angles, the replay-locked arrow, score popups, and "best" lit and unlit.
5. **The comparison.** For each case, compare the two images in XCTest. Report two numbers: the percentage of pixels that differ by more than a small per-channel threshold (start at 24 out of 255), and, for each labelled button region, the offset between the ink bounding boxes of green pixels in the two images. The second number catches a label that sits 1 unit low even when the overall pixel difference looks small. Fail a case when more than 0.5% of pixels differ or any ink box is more than 1 device pixel out. Tune those thresholds once against a deliberately broken label, then leave them.
6. **Output.** Write each pair, a red-on-black diff image, and a summary table (case, pixel %, worst ink offset, pass or fail) to a results folder inside the repo, ignored by git, for example `ios/SnapshotResults/`. Read the summary and the diff images after each run, fix the worst case first, and repeat until everything passes.

Text metrics (section 5) are where this loop earns its keep. Run it on the title screen and one confirm prompt as soon as `inkLabel` exists, and settle the baseline formula there before porting the other screens.

The human will check the finished app on a real phone once. Everything before that point should come out of this loop.

## 11. Constraints for the agent

- Run on a Mac with Xcode and at least one iOS simulator installed, for example in Claude Code on the human's Mac. An iOS app cannot be built, tested or screenshotted on Linux. If you find yourself in a Linux container, stop and say so before writing code.
- Do not change any tuned number, label, angle, colour, timing or probability. If something cannot be matched natively, stop and describe the difference rather than picking a substitute.
- Draw every ball, button, icon, line and label as a vector path or font text at runtime, at the screen's native resolution. Do not ship image assets apart from the app icon, and do not pre-render anything into bitmaps or textures.
- Do not add features, analytics, ads, Game Center or haptics unless the human asks.
- Keep `index.html` untouched. Put the Xcode project in a new top-level folder such as `ios/`.

## 12. Decisions already made

1. The status bar is hidden.
2. Audio uses `.ambient`, so it follows the silent switch and mixes with music.
3. Scores from the web app do not carry over.
4. The app is named Frict.
5. The human will supply the bundle identifier, team ID and a 1024×1024 icon later. Use placeholders until then.
