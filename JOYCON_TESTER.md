# Joy-Con & Phone Tester — AI Context Guide

> Read this first before changing `joycon-tester.html`. It explains what the tool does, how the code is laid out, the Nintendo Joy-Con HID protocol it uses, and how to extend and test it safely.
> **Status:** every feature below has been confirmed working on real hardware (Joy-Con L + R over Bluetooth, Chrome desktop).

---

## 1. What it is

A single self-contained web page (`joycon-tester.html`, no build step, no external libraries) that connects to Nintendo Switch Joy-Cons over **Bluetooth via WebHID** and tests every hardware feature:

| Feature | Joy-Con L | Joy-Con R | Pro Controller |
|---|---|---|---|
| Buttons (incl. SL/SR, stick click) | ✓ | ✓ | ✓ (no SL/SR) |
| Analog stick (factory/user calibration, range outline, dead zone) | ✓ | ✓ | ✓ (both sticks) |
| IMU: gyro + accelerometer, 3D orientation, live graph | ✓ | ✓ | ✓ |
| HD Rumble (manual, presets, 16 haptic scenes) | ✓ | ✓ | ✓ |
| Player lights (4 LEDs, flash, chase) | ✓ | ✓ | ✓ |
| Home button light (subcmd 0x38) | – | ✓ | ✓ |
| IR motion camera (MCU, image transfer) | – | ✓ | – |
| Battery, firmware, MAC, body colour, raw reports | ✓ | ✓ | ✓ |
| Test checklist (pass/fail) | ✓ | ✓ | ✓ |

Fallback: if `navigator.hid` is missing (Safari, Firefox, mobile), a **Gamepad API** "basic mode" shows buttons/axes only.

### Runtime requirements
- Chrome or Edge on desktop (Windows / macOS / Linux / ChromeOS).
- Open as a local file (`file://`) or via HTTPS. WebHID needs a secure context; `file://` counts.
- **Will not work inside a sandboxed iframe** (e.g. hosted artifact previews) because WebHID is blocked by permissions policy.
- Joy-Con must be paired in the OS Bluetooth settings first (hold rail sync button until lights chase).
- Linux: the `hid-nintendo` kernel driver may need udev rules for browser hidraw access.

---

## 2. Code map (all inside one `<script>`)

| Section / symbol | Purpose |
|---|---|
| `NAMES`, `ACCENT`, `NEUTRAL`, `BTN` | Product IDs, colours, neutral rumble bytes, button table `[id, label, byteIndex, mask]` |
| `encodeRumble(freq, amp)` | Single-frequency HD rumble encoder (used by manual controls/presets) |
| `encAmp`, `encodeDual(hf, ha, lf, la)` | Two-band HD rumble encoder (used by haptic scenes) |
| `RND`, `nEvents`, `anyBtn`, `jerkOf` | Helpers for haptic scenes |
| `SCENES[]` | Haptic scene definitions (see §7) |
| `CRC8T`, `crc8` | MCU CRC-8 (poly 0x07) for IR/MCU packets |
| `IR_RES`, `IR_PAL` | IR resolution table and colour look-up tables |
| `HOME_LED` | Home-light pattern byte builders |
| `dec9`, `defaultStick`, `stickCal` | Stick calibration decoding |
| `class JoyCon` | One instance per connected device: I/O, init, parsing, UI, drawing |
| `svgL()`, `svgR()` | Controller drawings; elements carry `data-btn`, `data-stick`, `.homeglow` |
| `addDevice`, `removeDevice`, `loop` | Device lifecycle + single `requestAnimationFrame` render loop |
| `drawGamepads()` | Gamepad API fallback |
| boot block | `requestDevice` button, `navigator.hid` connect/disconnect, auto-reconnect via `getDevices()` |

### `JoyCon` class — key members

**I/O**
- `send01(id, args)` — output report `0x01` (rumble + subcommand), variable length.
- `subcmd(id, args, timeout, retries)` — sends subcmd, resolves with reply payload (bytes from data[14] of the `0x21` reply). **Serialized** through `this.queue`.
- `pkt(reportId, bytes, crc)` — full 48-byte output report; `crc: "21"` or `"11"` adds MCU CRC (see §8).
- `request(sendFn, pred, timeout, retries)` — send, then wait for any input report matching `pred(id, dataView)`. Also serialized through `this.queue`. Uses `this.waiters`.
- `spi(addr, len)` — SPI flash read via subcmd `0x10`.
- `sendRumble(bytes8)` — output report `0x10` (rumble only).

**Init sequence** — `init()`: open → device info (0x02) → enable IMU (0x40 01) → enable vibration (0x48 01) → report mode 0x30 → colours (SPI 0x6050) → stick calibration (SPI 0x603D, user 0x8010, params 0x6086/0x6098) → IMU calibration (SPI 0x6020, user 0x8026) → player LEDs → auto gyro calibration.

**Input** — `onReport(e)` → waiters → `parseStd` (buttons, sticks, battery) → `parseIMU` (3 samples per report) → `irOnReport` (report 0x31 only) → subcmd reply matching.

**State** — `btnState`, `btnCount`, `btnDone`, `sticks[L|R]`, `imu` (calibration, bias, quaternion `q`, peaks, axis hits), `G` (graph ring buffer), `scene`, `hOut`, `hHist`, `ir`, `irCfg`, `homeMode`, `homeLevel`, `leds`, `ledFlash`, `info`.

**UI** — `buildUI()` builds the card from a template string; `wireUI()` uses one delegated click listener plus `input`/`change` listeners. Elements are looked up by attributes:
- `data-k="…"` text/meter outputs (cached in `this.ui.k`)
- `data-act="…"` action buttons, `data-rp` rumble presets, `data-scene` haptic scenes, `data-home` home-light modes, `data-led` player LEDs
- `data-in="…"` rumble/gain/home inputs, `data-irin="…"` IR camera inputs

**Drawing** — `draw(now)` runs every frame; heavy text updates are throttled to 10 Hz. Sub-drawers: `drawStick`, `drawModel`, `drawGraph`, `drawHaptics`, `drawIR`. `checks()` builds the test checklist.

---

## 3. HID basics

- Vendor `0x057E`. Product IDs: `0x2006` Joy-Con L, `0x2007` Joy-Con R, `0x2009` Pro Controller.
- WebHID `sendReport(reportId, data)` — `data` **excludes** the report ID. All byte offsets in this doc are into `data` (i.e. C-style `buf[n]` from reference tools = `data[n-1]`).
- Input event: `e.reportId`, `e.data` (DataView, no report ID).

### Output reports
| ID | Layout (data indices) |
|---|---|
| `0x01` | `[0]` packet counter (0–F), `[1..8]` rumble, `[9]` subcmd, `[10..]` args |
| `0x10` | `[0]` counter, `[1..8]` rumble only |
| `0x11` | `[0]` counter, `[1..8]` rumble, `[9]` MCU subcmd, `[10..]` args (IR/NFC MCU requests) |

### Input reports
| ID | Meaning |
|---|---|
| `0x3F` | Default simple HID mode (ignored except for rate) |
| `0x21` | Subcommand reply: standard input + `[12]` ACK, `[13]` subcmd id, `[14..]` reply data |
| `0x30` | Standard full mode, 60 Hz-ish, includes IMU |
| `0x31` | Same as 0x30 + MCU data from `[48]` (used for IR camera). 361 bytes |

### Standard input layout (0x21 / 0x30 / 0x31)
| Index | Content |
|---|---|
| `[0]` | timer |
| `[1]` | high nibble = battery (`>>1` level 0–4, bit0 charging) |
| `[2]` | right buttons: Y 01, X 02, B 04, A 08, SR 10, SL 20, R 40, ZR 80 |
| `[3]` | shared: Minus 01, Plus 02, RStick 04, LStick 08, Home 10, Capture 20 |
| `[4]` | left buttons: Down 01, Up 02, Right 04, Left 08, SR 10, SL 20, L 40, ZL 80 |
| `[5..7]` | left stick (12-bit X, 12-bit Y) |
| `[8..10]` | right stick |
| `[11]` | vibrator report |
| `[12..47]` | IMU: 3 frames × 12 bytes = accel X,Y,Z + gyro X,Y,Z (int16 LE) |

Stick decode: `x = b0 | ((b1 & 0x0F) << 8)`, `y = (b1 >> 4) | (b2 << 4)`.

---

## 4. Subcommands used

| Subcmd | Args | Purpose |
|---|---|---|
| `0x02` | – | Device info → `[0..1]` FW, `[2]` type (1 L, 2 R, 3 Pro), `[4..9]` MAC |
| `0x03` | `0x30` / `0x31` / `0x3F` | Input report mode |
| `0x10` | addr (u32 LE) + len | SPI read; reply `[0..3]` addr, `[4]` len, `[5..]` data |
| `0x21` | MCU config (see §8) | Set MCU mode / IR config / write IR registers |
| `0x22` | `0x01` / `0x00` | MCU resume / suspend (IR camera power) |
| `0x30` | `(flash<<4) \| on` | Player LEDs |
| `0x38` | pattern bytes | Home button light |
| `0x40` | `0x01` | Enable IMU |
| `0x48` | `0x01` | Enable vibration |

### SPI addresses
| Addr | Len | Content |
|---|---|---|
| `0x6020` | 24 | Factory IMU calibration (acc origin, acc coeff, gyro origin, gyro coeff; int16 ×3 each) |
| `0x603D` | 18 | Factory stick calibration (L 9 bytes, R 9 bytes) |
| `0x6050` | 6 | Body RGB, button RGB |
| `0x6086` / `0x6098` | 18 | Stick parameters (dead zone = `((p[4]<<8)&0xF00)\|p[3]`) |
| `0x8010` | 22 | User stick calibration (magic `B2 A1` before each 9-byte block) |
| `0x8026` | 26 | User IMU calibration (magic `B2 A1` + 24 bytes) |

Stick calibration 9 bytes → `dec9` gives 6 values. Left: `xMax, yMax, xCenter, yCenter, xMin, yMin`. Right: `xCenter, yCenter, xMin, yMin, xMax, yMax` (max/min are ranges above/below centre).

---

## 5. IMU

- Scale: accel `G = (raw − accOrigin) × 4 / (accCoeff − accOrigin)`; gyro `°/s = (raw − gyroOrigin) × 936 / (gyroCoeff − gyroOrigin)`. Fallbacks: `4/16384`, `936/13371`.
- Right Joy-Con: Y and Z axes are negated so both sides share one frame.
- Gyro bias: auto-calibrated over the first 160 samples after connect (rejected if moved, range > 10 °/s), plus a manual "Calibrate gyro" button.
- Orientation: Mahony-style complementary filter in `fuse()` (gyro integration + gravity correction, gain 1.5, dt 5 ms). Yaw drifts (no magnetometer) → "Reset view".
- Frame mapping for the CSS 3D model (`drawModel`): device x = towards top, y = left, z = out of the face, then mapped onto CSS axes via a fixed permutation.

---

## 6. HD Rumble encoding

8 bytes per output: bytes 0–3 left motor, 4–7 right motor (same data sent to both). Neutral = `00 01 40 40`.

Two bands per motor: **high band 82–1252 Hz**, **low band 41–626 Hz**, each with own amplitude.

```
ef  = round(log2(freq/10) * 32)
hf  = (ef_high - 0x60) * 4         // 9-bit
lf  = ef_low - 0x40                // 7-bit
ea  = amp>0.23 ? round(log2(amp*8.7)*32) : amp>0.12 ? round(log2(amp*17)*16) : linear   // 0..100
hfa = ea_high * 2
lfa = floor(ea_low / 2) + 64
bytes = [hf & 0xFF, hfa + ((hf>>8)&1), lf & 0x7F, lfa]
```
Rumble must be refreshed continuously (code resends every ~30 ms; scenes every ~12 ms + send time). Ownership is controlled by `rumbleToken`: any new play/scene increments it, and older loops exit.

---

## 7. Haptic scenes (`SCENES[]`)

Each scene: `{id, name, tag, desc, init?(s), tick(s, c), draw?(g, W, H, s)}`.

`tick` runs each engine frame. Context `c`:
- `c.t` seconds since start, `c.dt` frame delta
- `c.acc` [G], `c.gyro` [°/s] (latest calibrated IMU sample), `c.btn` button state map, `c.stick` [x,y] normalized, `c.stickY`
- `c.hit(band 'h'|'l', freqHz, amp0to1, ms, flat=false)` — one-shot impact (exponential decay unless `flat`; always rendered at least one frame)
- `c.at(seconds, fn)` — schedule a callback

Return value (optional) = continuous base `{hf, ha, lf, la}`. Engine (`startScene`) mixes base + impacts per band (amps add, frequency of strongest contributor wins), applies `hapticGain`, encodes with `encodeDual`, records `hOut`/`hHist` for the meter/graph.

Current scenes: `ice` (shake/swirl clinks + slosh), `ball` (tilt marble physics, draws box), `soda` (shake to fizz), `dice`, `sword` (swing whoosh + clang on sudden stop), `engine` (ZR/ZL/R/L or stick throttle, rev limiter), `gun` (any button, full-auto), `heart` (motion raises BPM), `rain`, `steps`, `purr`, `pong`, `boom`, `clock`, `phone`, `ratchet` (stick rotation detents every 15°).

**To add a scene:** append an object to `SCENES`. The UI button is generated automatically.

---

## 8. MCU packets and CRC

- CRC-8, polynomial `0x07`, init 0 (`crc8()`).
- **Config via `0x01` + subcmd `0x21`:** `data[9]=0x21`, `data[10]` MCU cmd, `data[11]` MCU subcmd, …, `data[47] = crc8(data[11..46])`. Built with `pkt(0x01, bytes, "21")`.
- **Requests/ACKs via `0x11`:** `data[9]` subcmd, `data[46] = crc8(data[10..45])`, `data[47] = 0xFF`. Built with `pkt(0x11, bytes, "11")`. Status request (`0x11` subcmd `0x01`) needs no CRC.

---

## 9. IR camera (Joy-Con R only) — `irStart(resKey)`

Protocol follows CTCaer's **jc_toolkit** (`ir_sensor()` / `get_raw_ir_image()`), verified on hardware.

| Step | Send | Wait for |
|---|---|---|
| 1 | subcmd `0x03 [0x31]` | normal reply |
| 2 | subcmd `0x22 [0x01]` (MCU on) | normal reply |
| 3 | `0x11` `[0x01]` (status) | `0x31` with `data[48]==0x01 && data[55]==0x01` (standby) |
| 4 | `0x21 [0x21, 0x00, 0x05]` (+CRC) set MCU to IR mode | `0x21` with `data[13]==0x21 && data[14]==0x01` |
| 5 | `0x11` `[0x01]` | `data[55]==0x05` (IR mode) |
| 6 | `0x21 [0x23, 0x01, 0x07, lastFrag, 0x00,0x05, 0x00,0x18]` image-transfer mode, MCU FW 5.18 | `0x21` with `data[14]==0x0B` |
| 7 | `0x21 [0x23, 0x04, 9, regs…]` then `0x11 [0x03, 0x02]` | `data[14]==0x13` |
| 8 | `0x21 [0x23, 0x04, 8, regs…]` (ends with finalize `00 07 01`) | `data[14]==0x13` or `0x23` |
| 9 | ACK fragment 0 → stream | — |

Registers are written as `[page, offset, value]` triples (max 9 per packet, finalize `[0x00,0x07,0x01]` required for changes to apply):

| Reg | Meaning |
|---|---|
| `00 2E` | Resolution: `0x00` 320×240, `0x50` 160×120, `0x64` 80×60, `0x69` 40×30 |
| `01 30` / `01 31` | Exposure LSB/MSB, value = µs × 31200 / 1000 (max 600 µs) |
| `01 32` | Max exposure enable (0 = manual) |
| `00 10` | IR LED groups: `0x00` both, `0x20` far only, `0x10` near only, `0x30` off |
| `00 11` / `00 12` | Far (0–15) / near (0–16) LED intensity |
| `01 2E` / `01 2F` | Digital gain low/high nibble |
| `00 0E` | External light filter (`0x03` on) |
| `01 43` | ExLF / white-pixel threshold (200) |
| `00 2D` | Flip (`0` normal, `2` horizontal) |
| `01 67` / `01 68` / `01 69` | De-noise enable / edge smoothing (0x23) / colour interpolation (0x44) |
| `00 04` | Buffer update time (`0x2D` for 40×30, else `0x32`) |
| `00 07` | Finalize = `0x01` |

**Streaming** (`irOnReport`, report `0x31`):
- `data[48]==0x03` → image fragment; `data[51]` fragment number; 300 bytes of 8-bit greyscale at `data[58..357]`; stats `data[52]` avg intensity, `data[54..55]` white pixels, `data[56..57]` ambient noise.
- ACK every fragment: `pkt(0x11, [0x03, 0x00, 0x00, 0x00, frag], "11")`.
- `data[48]==0xFF` → re-ACK last fragment; `0x00` → request missing: `[0x03, 0x00, 0x01, next, 0x00]`.
- Frame complete when fragment == `lastFrag` (`0xFF`, `0x3F`, `0x0F`, `0x03` for the four resolutions; frames = (lastFrag+1) × 300 bytes).
- Auto exposure (every 3rd frame): jc_toolkit rule, `+10 µs` if no white pixels, `−(pct/4)×20 µs` if > 5 %.
- Live settings changes call `irWriteRegs()` (fire-and-forget).
- `irStop()`: subcmd `0x22 [0x00]`, then back to report mode `0x30` (IMU keeps working).

`drawIR()` computes: palette render, brightest-spot centroid (≥ 85 % of max), mean intensity → "proximity", mean abs frame diff → "motion" (lamp on > 1.5 %). Rotation is applied with a canvas transform, so overlays rotate with the image.

---

## 10. Home button light — `setHome(mode)` (subcmd 0x38)

Byte layout: `[0]` = mini-cycle count (high nibble) | global cycle duration (low), `[1]` = start intensity (high) | repeat count, 0 = forever (low), then per pair of mini cycles `[int1|int2], [fade1|hold1], [fade2|hold2]`.

| Mode | Bytes | Source |
|---|---|---|
| off | `00 00` | – |
| on (L = 1–15) | `1F, L<<4, L<<4, 0F, 00` | built |
| breathe | `28 20 F2 F0 F0` | jc_toolkit |
| pulse | `2F 00 F0 81 81` | built |
| blink | `28 F0 F0 02 02` | built |
| heartbeat | `F1 00 F0×6 (00 FF FF)×5` | jc_toolkit |

The on-screen `.homeglow` ring mirrors the mode with CSS animations (approximation only).

---

## 11. Test checklist (`checks()`)

All buttons pressed · stick ≥ 90 % of 36 angle bins reach r > 0.85 (per stick) · gyro > 150 °/s on X, Y and Z · shake > 2 G · resting |a| within 1 ± 0.12 G · rumble sent · (R/Pro) Home light set · (R) ≥ 5 IR frames received.

---

## 12. Testing without hardware (mock harness)

Development was verified with Playwright + a fake `navigator.hid` injected via `page.addInitScript`. The mock:
- Extends `EventTarget`, has `vendorId/productId/opened/open/close/sendReport`.
- Streams `0x30` reports every 15 ms (or 361-byte `0x31` when mode is 0x31).
- Replies to `0x01` subcommands with a `0x21` report (`data[12]=0x80`, `data[13]=subcmd`, payload from `data[14]`).
- For IR: answers status requests, the `0x21` MCU commands (verifying CRC), switches to streaming once the resolution register is written, and sends the next fragment after each ACK.

Use `executablePath: '/opt/pw-browsers/chromium'` if Playwright's bundled browser isn't installed. Check for `pageerror` events, the status text, and the bytes captured in `sendReport`.

---

## 13. Rules / gotchas for future edits

1. **Never send two command/response exchanges concurrently** — always go through `subcmd()` or `request()` (they share `this.queue`). Fire-and-forget is only OK for rumble (`0x10`), IR ACKs, and live IR register writes.
2. Keep offsets as **WebHID data indices** (report ID excluded). Reference C code using `buf[n]` with the ID included is off by one.
3. Rumble ownership: increment `rumbleToken` before starting any new rumble loop; check it inside loops.
4. Report 0x31 also carries standard input + IMU — keep parsing those when adding MCU features (NFC uses the same MCU path, MCU mode `0x04`).
5. The page must stay a single self-contained HTML file with no external dependencies.
6. Per-frame work belongs in `draw()`; throttle DOM text writes (the 10 Hz block).
7. UI hooks rely on `data-*` attributes; when adding a control, add the attribute in `buildUI()` and handle it in `wireUI()`.
8. IR resolution trade-off: 40×30 / 80×60 are fast; 320×240 = 256 fragments per frame, only a few FPS.

## 14. Phone sensors tab

The second tab (phone sensors, live phone → laptop link, phone haptic scenes, iPhone rules) is documented in **[`PHONE_TESTER.md`](PHONE_TESTER.md)**.

## 15. References
- dekuNukem, *Nintendo_Switch_Reverse_Engineering* (HID reports, subcommands, SPI map, rumble, IMU).
- CTCaer, *jc_toolkit* (`jctool.cpp`: IR camera, MCU CRC, Home LED patterns).
