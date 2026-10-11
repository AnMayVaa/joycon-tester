# Phone Sensor Tester — AI Context Guide

> Read this before changing the **Phone sensors** tab of `joycon-tester.html` (published as `index.html` at https://anmayvaa.github.io/joycon-tester/). The Joy-Con tab is documented separately in `JOYCON_TESTER.md`.
> **Primary use case (from the user):** the page is shown on a **desktop/laptop** for monitoring and control; an **iPhone** connects by scanning a QR code and streams its sensors to the laptop in real time.

---

## 1. Status

| Feature | Status | Notes |
|---|---|---|
| Live link phone → laptop (QR + WebRTC) | ✅ confirmed by user | |
| Motion: accel, gyro, 3D phone, graph, shake | ✅ confirmed (3D direction fixed in v`4fbc6fb`) | |
| Bubble level, compass, multi-touch, GPS, camera stream | ✅ verified in emulation | |
| Haptic scenes (all 16) on iPhone | ✅ confirmed by user, **as sound** | iPhone can't vibrate from a website |
| Vibration presets on iPhone | ⚠ played as buzz **sound** (`phBuzzSound`) | Real motor only via the phone's "Tap haptic test" button |
| Speaker tone from laptop | ⚠ fix shipped in `503ab8d`, waiting for user confirmation | Was silent on iPhone |
| Microphone | ⚠ fix shipped in `503ab8d`, waiting for user confirmation | Broke when audio session was set to "playback" |
| Rotation | ✅ physical turn detection (works with iPhone rotation lock) | |
| Android vibration motor | ✅ verified in emulation (`navigator.vibrate`) | Not tested by the user (no Android phone) |

---

## 2. Requirements

- **Phone:** must load the page over **HTTPS** (secure context) or motion, camera, mic and GPS are blocked. GitHub Pages provides this.
- **Laptop:** any modern desktop browser. `file://` works for the laptop view, but the QR then needs the hosted https URL typed into "Phone page URL".
- **Network:** both devices need internet for the WebRTC handshake (PeerJS public server `0.peerjs.com` + Google STUN). No TURN server is configured, so very strict networks (symmetric NAT, some school/office Wi-Fi) may fail.
- **External scripts (loaded lazily):** `https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js`, `https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js`.
- Bluetooth is **not** possible: a phone browser can't act as a Bluetooth peripheral, so the link is WebRTC.

---

## 3. How it is used

1. Laptop opens the page → **Phone sensors** tab → a QR code appears (host mode).
2. iPhone scans the QR → opens `…/#pair=<id>` → taps **Connect & start sensors** (grants Motion & Orientation; unlocks audio; requests a screen wake lock).
3. Every panel on the laptop updates live. Laptop buttons for vibration, speaker and haptic scenes act on the phone.
4. Phone screen (sender view) has: status, GPS / Microphone / Share camera / Switch camera / **Tap haptic test** buttons, a touch pad (shown as multi-touch on the laptop and used as trigger/stick for scenes), readouts and the active scene name.

A phone that opens the page directly (no `#pair`) gets **local mode**: all panels run on the phone itself.

---

## 4. Code layout

The phone code is the **second `<script>`** in the page and reuses globals from the Joy-Con script (`$`, `clamp`, `fmt`, `SCENES`, `RND`, `nEvents`, `anyBtn`, `jerkOf`).

| Symbol | Purpose |
|---|---|
| `PH` | All sensor state (acc, lin, rot, ori, heading, pointers, gps, mic, battery, tests, rotation counters…) |
| `P(k)` | Cached lookup of `[data-p="k"]` elements (outputs) |
| `phMotion` / `phApplyMotion` | devicemotion handler / shared apply (local or remote) — peak, shakes, gyro test, graph ring buffer |
| `phOri` / `phOriAbs` / `phApplyOri` | deviceorientation handlers / shared apply — tilt, heading |
| `phRotDetect(β, γ)` | Physical portrait ↔ landscape detection |
| `phStart` | Motion permission (iOS `requestPermission`) + listeners + Generic Sensor probe |
| `phProbeSensors` | Generic Sensor API table (Chrome Android) |
| `phTouchSetup(cv, onChange)` | Pointer tracking on a canvas (multi-touch panel and phone touch pad) |
| `phVibrate(kind, ms)` | Vibration presets (Android API / iPhone sound fallback / remote send) |
| `phAC`, `phAudioSession`, `phUnlockAudio` | Shared AudioContext and iOS audio session handling |
| `phTone(mode, opts)` | Speaker tone (both / left / right / sweep) |
| `phMic` | Microphone → analyser (level, peak, pitch, 96-bar spectrum) |
| `phCam` | Local camera + torch |
| `phGps` | `watchPosition` high-accuracy |
| `phInfo` / `phInfoRows` | Screen & device table (remote rows replace local when linked) |
| `phChecks` | Test checklist |
| `phDraw(now)` | Render loop (only while the tab is visible); text throttled to 10 Hz |
| `LINK`, `linkHost`, `linkRx`, `linkJoin`, `linkSenderStart`, `linkSenderTick`, `linkShareCam` | Live link (see §6) |
| `PHH`, `phSceneStart/Stop/Click`, `phSceneCtx`, `phPadState`, `phSynth*`, `phBuzzSound`, `iosTick`, `phDrawHaptics` | Haptic scenes (see §7) |
| `setView`, `linkAutoStart` | Tab switching and mode selection |

UI hooks: `data-p` (outputs), `data-pact` (actions), `data-pin` (inputs), `data-vib`, `data-tone`, `data-cam`, `data-pscene`, `data-snd` (phone sender buttons). Containers: `#view-phone`, `#ph-host` (laptop QR card), `#ph-sender` (phone link screen), `#ph-local` (start notice), `#ph-card` (all panels).

### Mode selection (`linkAutoStart`)
- URL `#pair=<id>` → **sender** (phone)
- phone-like UA + touch → **local**
- otherwise → **host** (laptop)

`setView` never overwrites a `#pair=` hash.

---

## 5. Panels and Web APIs

| Panel | API | Details |
|---|---|---|
| Motion (3D) + values | `devicemotion` | `accelerationIncludingGravity`, `acceleration`, `rotationRate` (mapped x=β, y=γ, z=α). Shake = \|a\| > 2 G (re-armed below 1.3 G) |
| 3D model | `deviceorientation` | Device frame (x right, y top, z out) → CSS (y down): `rotateX(35deg) rotateZ(-α) rotateX(-β) rotateY(γ)`; the first rotateX is only the camera tilt |
| Bubble level | β/γ angles | Angles avoid iOS/Android accel sign differences; "Level ✓" when both < 1° |
| Compass | `webkitCompassHeading` (iOS) or `deviceorientationabsolute` → 360 − α | |
| Motion graph | ring buffer `PH.G` (400 samples) | gyro ±360 °/s, accel ±3 G |
| Multi-touch | Pointer events, `touch-action:none` | All pointers, trails, pressure, contact size, max simultaneous |
| Vibration motor | `navigator.vibrate` | iPhone → sound (see §8) |
| Speaker | Web Audio | osc → gain → destination; L/R via `ChannelMerger` (ch0/ch1); sweep 40 Hz → 16 kHz over 8 s; live freq/volume/wave |
| Microphone | `getUserMedia` (echo/noise/AGC off) → `AnalyserNode` (fft 2048) | RMS dBFS, peak, dominant pitch, log-spaced 96 bars |
| Camera | `getUserMedia` video, `facingMode` | Torch via `applyConstraints({advanced:[{torch}]})` when supported; when linked, the phone's camera is streamed to the laptop over a PeerJS media call |
| GPS | `geolocation.watchPosition` | lat/lon, accuracy, altitude, speed, heading |
| Sensor API | Generic Sensor classes | Chrome Android only; Magnetometer/AmbientLight need a chrome://flags flag |
| Screen & device | `screen`, `matchMedia`, `navigator.connection/deviceMemory/getBattery`, `wakeLock` | Refresh rate from rAF; rotation counts |

### Checklist (`phChecks`)
Motion data · gyro > 90 °/s · 3 shakes · orientation · compass · multi-touch (2+ fingers) · vibration · speaker · mic > −40 dBFS · camera · GPS fix · turn phone portrait ↔ landscape · haptic scene played.

---

## 6. Live link protocol (`LINK`)

Host (`linkHost`): creates PeerJS peer `jct-<random>`, renders QR of `<Phone page URL>#pair=<id>` (URL defaults to the current https URL, else typed in and saved to `localStorage["jct-phone-url"]`), accepts one data connection (JSON), answers the phone's media call (camera), sends `ping` every 2 s.

Sender (`linkSenderStart`, on tap): `phUnlockAudio()` → `phStart()` → wake lock → `peer.connect(hostId, {serialization:"json", reliable:true})` → `linkSenderTick` every **20 ms**.

All incoming phone data is applied through the same functions/state the local sensors use (`phApplyMotion`, `phApplyOri`, `PH.*`), so panels and the checklist need no remote-specific code.

### Phone → laptop
| `t` | Payload | Rate |
|---|---|---|
| `m` | `acc`, `lin`, `rot`, `o:[α,β,γ]`, `hd` (heading), `hs` (source) | when new motion, ≤ 50 Hz |
| `touch` | `p:[[id, x, y, pressure, w, h]]` normalized 0–1 | on pointer change |
| `gps` | `ts`, `c` (coords) | on fix |
| `mic` / `micoff` | `db`, `peak`, `pitch`, `sr`, `bars[96]` | 25 Hz while on |
| `info` | `name`, `rows`, `bat`, `ios` | every 4 s |
| `sensors` | Generic Sensor statuses | every 1.5 s |
| `hout` | `id`, `L` (0–1), `ball:[x,y]` | every scene tick (40 ms) |
| `ev` | `k: vibrate {ok, why} · speaker {state} · mic {ok, why} · rotate {n}` | on event |
| `pong` | `ts` | reply to ping |

### Laptop → phone
| `t` | Payload | Effect on phone |
|---|---|---|
| `vib` | `k`, `ms` | `phVibrate` |
| `tone` | `mode`, `freq`, `vol`, `wave` | `phTone` |
| `scene` / `scenestop` / `hgain` | `id`, `gain` | `phSceneStart` / `phSceneStop` / gain |
| `ping` | `ts` | replies `pong` (latency = RTT/2) |

---

## 7. Haptic scenes on the phone (`PHH`)

- Reuses the Joy-Con `SCENES[]` definitions unchanged; the **scene runs on the phone** (lowest latency) and reports `hout` to the laptop for its meter, graph and marble view.
- **Context (`phSceneCtx`)**: gravity "up" from β/γ = `(-cosβ·sinγ, sinβ, cosβ·cosγ)` plus linear acceleration, mapped phone → Joy-Con scene frame: `joy.x = phone.y`, `joy.y = −phone.x`, `joy.z = phone.z` (gyro the same). Units: G and °/s.
- **Input (`phPadState`)**: any touch on the pad = `zr`/`a` pressed; first finger position = stick. Tags shown for phone: engine "touch pad = throttle", gun "tap the pad", boom "auto · tap the pad", ratchet "circle a finger on the pad".
- **Tick:** every 40 ms. Continuous level `L = la + 0.6·ha` (+ long decays) × gain; impact onsets give a kick.
- **Android output:** PWM with `navigator.vibrate(ms)` — `L × 40 ms` (dithered below 8 ms; ≥ 0.9 overlaps for a solid buzz) plus 12–57 ms kick pulses.
- **iPhone output (sound):** `phSynthStart` creates two oscillators — low band (triangle, frequency × 2 because phone speakers can't play 40–160 Hz) and high band (sine) — whose gains follow the scene; `phSynthHit` plays each impact as a short enveloped burst.

---

## 8. iPhone rules (important)

1. **No remote vibration.** iPhone Safari has no Vibration API. The only web haptic is the iOS 18 `<input type="checkbox" switch>` tap (`iosTick`), and Safari honours it only inside a user gesture on the phone. Anything triggered from the laptop can't vibrate the iPhone — it is rendered as sound instead (`phBuzzSound` for presets, synth for scenes). The **Tap haptic test** button on the phone verifies the real motor (needs iOS 18+ and System Haptics on).
2. **Audio unlock.** Web Audio stays locked until a sound starts inside a tap: `phUnlockAudio()` runs on the Connect tap and again on any `touchend` if the context isn't running.
3. **Silent switch vs microphone.** `phAudioSession()` sets `navigator.audioSession.type` (Safari 16.4+) to `"playback"` (plays even in silent mode) normally, and `"play-and-record"` while the mic is on or starting — `"playback"` blocks `getUserMedia` audio. Always call `phAudioSession()` when mic state changes.
4. **Motion permission** must be requested from a tap (`DeviceMotionEvent.requestPermission()`); if denied: Settings › Apps › Safari › Motion & Orientation Access.
5. **Rotation lock** is usually on, so screen-orientation events never fire. `phRotDetect` counts physical turns: portrait when |β| > 55° and |γ| < 30°, landscape when |γ| > 60° and |β| < 30°.
6. **No Generic Sensor API, no battery API, no torch** on iPhone Safari — the panels show "not in this browser" / "not exposed".

---

## 9. Testing without a phone

Verified with Playwright (Chromium) in this setup:
- Serve the HTML from `http://localhost` (secure context) and launch Chromium with `--use-fake-ui-for-media-stream --use-fake-device-for-media-stream --autoplay-policy=no-user-gesture-required`.
- Two contexts: a desktop page (host) and a phone page (`devices['Pixel 7']` or `devices['iPhone 14']`) opened at `#pair=<id>` read from `[data-p="lurltext"]`. The real PeerJS server is used.
- Synthetic sensors: dispatch `new DeviceMotionEvent('devicemotion', {...})` and `new DeviceOrientationEvent('deviceorientation', {...})` on an interval.
- Multi-touch: CDP `Input.dispatchTouchEvent` (scroll the canvas into view first; coordinates are viewport-relative).
- iPhone emulation: `delete Navigator.prototype.vibrate` and count `HTMLLabelElement` clicks to observe `iosTick`. Chromium has no `navigator.audioSession`, so iOS audio-session behaviour can only be checked on a real iPhone.
- Spy on Android vibration by replacing `navigator.vibrate` in an init script.

---

## 10. Rules for future edits

1. Keep local and remote paths unified: new sensor data goes through an apply function that updates `PH`, so the laptop and the phone render the same way.
2. Anything that must run on the phone when linked (vibration, sound, scenes) checks `LINK.role === "host" && LINK.open` on the laptop and sends a message instead.
3. On iPhone, never rely on vibration; provide a sound or visual fallback and tell the laptop via an `ev` message.
4. Any change to microphone or audio output state must call `phAudioSession()`.
5. Keep messages small JSON; motion is rounded to 2 decimals.
6. Don't overwrite a `#pair=` URL hash.
7. Commit/push only after the user approves; commits are authored by AnMayVaa with no AI attribution.
