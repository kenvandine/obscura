# OBSBOT Tiny SE protocol notes

Device: `3564:feff` "OBSBOT Tiny SE", Remo Tech Co., Ltd., bcdDevice 5.10,
firmware v6.4.4.1. Decoded from the USB captures in `captures/` (decoder:
`tools/obsbot_pcap.py`) and confirmed on the camera from Linux unless
marked *unverified*. Entries from the 2026-09-27 batch (sleep, HDR, AF/AE
modes, audio, device sleep, status light, AI lock, queries) were also set and
read back on the camera from Linux.

## Channels

| Path | What |
|---|---|
| Standard UVC (V4L2) | brightness, contrast, saturation, sharpness, hue (PU unit 3); pan/tilt (±130°/±90°), zoom (0..12), focus, exposure, gain, white balance, power-line frequency |
| XU unit 2, GUID `9a1e7291-6843-4683-6d92-39bc7906ee49` | 22 selectors, all 60 bytes, GET+SET |
| XU selector 2 | framed commands and queries (below) |
| XU selector 6 | SET: short settings `[id, len, value LE]`; GET: status block |

## Framed commands (selector 2)

```
0      aa           magic
1      flags        0x25 command, 0x01 query (0x21/0x05 also seen); responses 0x29, errors 0x89
2..4   seq u16le    the camera accepts any value, including repeats
4..6   0x000c
6..8   token u16le  CRC-16/USB over bytes 0..12 with the token zeroed
8      sender       0x0a host (responses: 0x04/0x02 -> 0x0a)
9      receiver     0x04 AI/gimbal module, 0x02 ISP, 0x0d system, 0x12 ?
10..12 cmd          u16le = (id << 7) | op ; op 0x44 write / 0x04 read (dst 04), 0x42 (dst 02)
12..14 len u16le
14..16 crc u16le    CRC-16/USB over bytes 12..16+len with this field zeroed
16..   payload      bytes after the payload are ignored (captures contain stale data there)
```

CRC-16/USB = poly 0x8005 reflected, init 0xFFFF, xorout 0xFFFF. **The camera
silently drops frames with a wrong token or payload CRC.** The status block
reflects a command about 1.1 s later.

Queries: send the frame with flags `0x01`, then GET_CUR selector 2 about 50 ms
later to read an `aa 29` response carrying the same cmd and a payload.

> ⚠️ **Never send a write command ID (op 0x44) as an empty query.** The camera
> answers with `aa 89` errors, and during probing it then stopped responding
> and dropped off USB. Only send queries that were seen in captures.

## Commands

| Feature | Write | Read (query) | Value |
|---|---|---|---|
| Gesture: Locked Target | dst 04 `c430` | `8430` → u8 | u8 0/1; also status[1] |
| Gesture: Zoom | dst 04 `4431` | `0431` → u8 | u8 0/1 |
| Gesture: Zoom Factor | dst 02 `4219` payload `02000000` + u32 (factor×100), then dst 04 `4432` f32 factor | `0468` → f32? (*unverified*) | 1.00–4.00 |
| Gesture: Dynamic Zoom | dst 04 `4433` | ? (`0433` query was sent just before the camera hung) | u8 0/1 |
| Gesture: Direction Flip | dst 04 `c433` | `8433` → u8 | u8 0/1 |
| Gesture Control (master) | the four switch frames above, same value | status[1] | |
| Gimbal velocity (joystick) | dst 04 `8464` 3×f32 `[0, pitch, yaw]`, ~10 Hz while held; zeros to stop | – | ✅ measured from Linux: positive pitch tilts the camera down (the picture moves up); positive yaw turns the view right (the picture moves left), with Mirror Image on or off. OBSBOT Center sends negative pitch for joystick up, and wraps each move in dst 04 `4402` u8 0 … 1 |
| Re-center (View & Gimbal reset) | query dst 04 `8438` payload `01`, then `0439` with flags `0x05` and no payload | – | ✅ UVC pan/tilt 0 does *not* re-center after velocity moves |
| Manual zoom | dst 02 `4219` payload `02000000` + u32 (factor×100, 100–400) | `0468` → f32 | ✅ ignored while the camera sleeps |
| View and Gimbal Reverse | dst 04 `843b` u8 | none seen | |
| Gimbal position | – | dst 04 `0466` payload `01` → 24 bytes | i16 angles ×10 at [8] and [10] |
| Preset: save | dst 04 `4439`: slot u32, f32 angle[10]×0.1, f32 angle[8]×0.1, 0.0, zoom f32, −1000.0 | | ✅ 3 slots (0–2) |
| Preset: name | dst 04 `843a`: slot u32 + name bytes | `043b` flags `0x21`, payload slot u32 → name, or flags `0x09` if empty | ✅ |
| Preset: recall | dst 04 `c439`: slot u32, 1.0 ×4 | | ✅ |
| Hand-tracking option (dropdown) | dst 04 `4422` u8 | ? | *meaning pending NOTES* |
| Hand-tracking limits? | `4420`/`c420` f32 −45/45, `4421`/`c421` f32 −30/30, `c424` 24 bytes, `c426` u8 | | *unverified* |
| Gimbal Speed | none (host-side scale for the joystick) | | ✅ `02-known-state-readback` shows no traffic |
| Sleep / Resume | dst 02 `c2a0` u32 1 / 0 | status[2] | ✅ status[9] also goes 01 → 03 while asleep |
| Auto Focus Mode | dst 02 `0236` u32 | status[13] | 0 Global, 1 Face |
| Firmware version | – | dst 0d `0804` → 4 bytes, last component first | `01 04 04 06` = 6.4.4.1 |
| Serial number | – | dst 0d `c818` → ASCII | 14 characters |
| Gesture state | – | dst 04 `0401` → 12 bytes | ✅ [3] locked target, [4] zoom, [5] dynamic zoom, [6] direction flip, [7] zoom factor ×10 |
| Current zoom? | – | dst 04 `0468` → f32 | 1.01–2.57 seen; follows tracking zoom, not the gesture zoom factor |

Queries without a payload leave the payload CRC (bytes 14..16) zeroed;
queries with one (`0466`, `043b`, `8438`) carry a normal CRC. Responses echo
the query's seq, which matters when several queries are in flight.

OBSBOT Center saves a preset by querying the position (`0466`) and zoom
(`0468`), writing `4439` and then naming it with `843a` ("Preset1".."Preset3"
by default). At startup it lists names with `043b` for each slot. It also
sends `443a` (read a stored preset) and `c43d` around each recall, which
aren't needed to move there. No delete command was seen.

**Rotate, flip and Portrait** send nothing to the camera
(`33-rotate-flip`, `35-resolution-aspect`): OBSBOT Center applies them to
its virtual camera on the PC. Resolution and frame rate are standard UVC
stream negotiation. OBSBOT Center's startup
sequence (`02-known-state-readback`) is dst 0d `0818`, `0804`, `4819`,
`c818`, then dst 02 `42c4`, dst 04 `0401` and `0468`, dst 02 `c229` and
`4229`. It never queries the individual gesture switches.

`0401` layout, confirmed from Linux by toggling each switch and re-querying:
`00 00 00 LT ZM DZ DF ZF 00 02 00 00`, where ZF is the zoom factor ×10
(`0f` = 1.5×, `19` = 2.5×, `28` = 4.0×). Byte [0] was 1 in the capture
(AI mode Group) and 0 on Linux (AI mode off); [9] was always 2.

**AI lock** has no command of its own. Locking sends AI mode 0 and turns off
the gesture switches that were on; unlocking restores the AI mode only.

## Short settings (selector 6)

| Feature | Write | Status byte | Values |
|---|---|---|---|
| Mirror Image | `14 01 v` | [19] | 0/1 ✅ |
| AI mode | `16 02 v 00` | [24] | 0 off, 1 group, 2 human, 3 hand ✅ |
| ? (hand dropdown) | `18 01 v` | | *pending* |
| HDR | `01 01 v` | [6] | 0/1 |
| Auto Exposure Mode | `03 01 v` | [7] | 0 Global, 1 Face |
| Noise Reduction | `0a 01 v` | [8] | 0 Off, 1–3 (labels pending) |
| Auto Sleep + Sleep Time | `0b 02 t` (i16 seconds) | [10..12] i16 | 30 / 120 / 600; negated (−30) when Auto Sleep is off |
| Sleep Background Mirror | `0e 02 02 v` | [29] | 0/1 |
| Mic Status During Sleep | `13 01 v` | [16] | 0/1 |
| Auto Gain | `17 01 v` | [25] | 0/1 |
| Status Light | `1a 01 v` | [33] | 0 off, 1–3 brightness |
| Radio Distance | `1b 01 v` | [34] low bits | 0 Close, 1–2 (labels pending) |
| Microphone enabled (OBSBOT Center: "Disable Microphone", inverted) | `1c 01 v` | [34] bit 4 (0x10) | 1 enabled, 0 disabled ✅; the camera re-enumerates on USB, without its audio interface when disabled |

## OBSBOT Meet Flip (3564:ff06, firmware 4.5.3.3)

Same envelope as above (XU 2, selector 2 frames and selector 6 short
settings), decoded from OBSBOT Center captures and then written from Linux and
checked against the selector 6 status block. Commands that match the Tiny SE
are not repeated here.

| Feature | Write | Status byte | Values |
|---|---|---|---|
| HDR | `01 01 v` | [6] | 0/1 ✅ |
| Mic during sleep | `13 01 v` | [16] | 0/1 ✅ |
| Brightness of the status light | `1a 01 v` | [33] | 1..3 ✅ |
| Status light on/off | `1a 01 v` | [45] | 0 off, 1 on ✅ (separate from brightness, unlike the Tiny) |
| Microphone enabled | `1c 01 v` | [34] bit 0x10 | 1 enabled, 0 disabled ✅; the camera re-enumerates on USB, without its audio interfaces when disabled |
| Pickup mode | `22 02 00 v` | [40] bits 0x38 (`v << 3`) | 0 Smart Omni, 2 Directional, 4 Dual-Directional, 5 Pure Audio ✅ |
| Field of view | `04 01 v` | none | 0 Wide, 1 Medium, 2 Narrow ✅ wide and narrow checked on the picture |
| Auto focus mode | dst 02 `0236` u32 | [13] | 0 Global, 1 Face ✅ |
| Sleep / Resume | dst 02 `c2a0` u32 1 / 0 | [9] bit 0x02 | [9] is 01 awake, 03 asleep ✅ |
| Mirror image | `14 01 v` | [19] | 0/1 ✅ (same as the Tiny) |
| Auto framing, composition, gesture zoom and its factor | `20 05 A B G fLo fHi`, one setting rewritten whole on every change | u16 at [38..40]: `factor << 4 \| G << 3 \| B << 1 \| A` | A auto framing 0/1; B composition 0 Group, 1 Single close-up, 2 Upper body; G gesture zoom 0/1; f gesture zoom factor x100 (u16 LE; 100..400 seen, default 200); decoded from captures, *not bound* (needs a read-modify-write binding) |
| Digital zoom / crop window | `21 10` + 4 x i32 | ? | bursts while zooming or panning in OBSBOT Center, *not bound* |
| Presets | dst 02 `42d0` read, `82d0` save/delete, `c2d0` recall | | 3 slots; save carries the slot and a name; *not bound* |

Byte 43 of the status block changes on its own and is ignored. Beauty, blur
and rotation in OBSBOT Center produced no camera command (rotation only
restarts the video stream), so they are applied on the host; `12 01 01` is
sent repeatedly at unrelated times. The Flip has no noise reduction, auto
gain or radio distance.

## Standard UVC controls used by OBSBOT Center

Confirmed in the captures: Auto Focus and Focus are Camera Terminal
selectors 8 and 6; Exposure Compensation is Processing Unit selector 1
(backlight compensation, 0..18 = −3..+3 EV in 1/3 steps); Anti-Flicker is
PU selector 5; Contrast and Saturation are PU selectors 3 and 7.

Status block bytes seen changing: [1] locked target, [2] asleep, [4] and
[17] tracking state, [6] HDR, [7] AE mode, [8] noise reduction, [9] device
state (1 awake, 3 asleep), [10..12] sleep time, [13] AF mode, [14] AF on,
[15] focus, [16] mic during sleep, [19] mirror, [24] AI mode, [25] auto gain,
[29] sleep background mirror, [31] 0x3c/0x1e (changes with [9]), [33] status
light, [34] microphone flags.
