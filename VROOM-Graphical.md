# VROOM-Graphical

**Virtual Remoting Over Open Methods: Graphical Protocol**

*The pixels of a whole desktop, fast enough to work in, for humans and agents.*

Version: 0.2.0-draft | Status: Draft | Date: 2026-11-08 | Base: [VROOM-Session](./VROOM-Session.md)

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

VROOM-Graphical carries displays and windows as pixels, with OS-level input. One capture engine serves two consumers: humans get low-latency video, and agents get the latest frame on demand, with changed regions and settling.

**Profile:** `vroom-graphical/1`. Surfaces are `display:<n>` and `window:<n>`. Binary messages name a surface by its number, a `uint16` assigned by the host and listed in SURFACES as `num`.

**Changes from 0.1:** the protocol now targets OS desktops, not a browser driven by Playwright. Browser commands moved to [Browsable](./VROOM-Browsable.md). See [Appendix A](#appendix-a-changes-from-01).

---

## 1. Goals

Latency is the product, so these rules are normative:

1. **Never queue stale video.** A sender drops frames rather than building a backlog.
2. **The cursor never waits for video.** It has its own channel and is drawn by the viewer.
3. **Input outranks video** on any shared path.
4. **A still screen gets no worse.** Motion may be soft, but a still screen converges to the best its pixel format can carry. A client that needs exact pixels asks for a lossless screenshot (§5).

---

## 2. Channels

| Channel | Requested delivery | Dir | Purpose |
|---|---|---|---|
| `vroom.graphical.control` | Reliable, ordered | Both | Stream setup, clipboard, cursor shapes |
| `vroom.graphical.video.<surface>` | Unreliable, unordered, where available | S→C | Video |
| `vroom.graphical.feedback` | Reliable, ordered | C→S | Receiver reports, keyframe requests |
| `vroom.graphical.cursor` | Unreliable, unordered | S→C | Cursor position |
| `vroom.graphical.pointer` | Unreliable, unordered | C→S | Pointer movement |
| `vroom.graphical.input` | Reliable, ordered | C→S | Buttons, scroll, keys, text, touch |
| `vroom.graphical.agent` | Reliable, ordered | Both | [Agent access](#5-agent-access) |
| `vroom.graphical.audio.<id>` | Unreliable, unordered, where available | Both | Audio |

Channels open when needed: a watching client opens no input channels, and an agent without a viewer opens no video.

On a reliable path, `cursor` and `pointer` still work: senders coalesce to the latest value while the channel lacks credit.

---

## 3. Video

### 3.1 Starting a stream

On `vroom.graphical.control`:

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x01` | STREAM_START | C→S | `surface`, `codecs` (preference order, as WebCodecs codec strings), `max_fps?`, `max_bitrate?`, `max_edge?`, `include_cursor?` |
| `0x02` | STREAM_STOP | C→S | `surface` |
| `0x03` | STREAM_STATE | S→C | `surface`, `codec`, `fps`, `bitrate`, `source`, `encoder` |

The host opens `vroom.graphical.video.<surface>` after STREAM_START. H.264 is the baseline, and every host and client MUST support it. Clients discover decoders at run time, with `VideoDecoder.isConfigSupported()` in browsers.

### 3.2 Messages on the video channel

| Type | Name | Body |
|---|---|---|
| `0x01` | VIDEO_CONFIG | CBOR: `codec`, `coded_w`, `coded_h`, `description?`, `colorspace`, `scale`, `origin`. Sent before the first frame and whenever these change |
| `0x02` | VIDEO_FRAME | One encoded frame, on a reliable path |
| `0x03` | VIDEO_SLICE | Part of one encoded frame, on an unreliable path |

**VIDEO_FRAME** is a 32-byte binary header, then the frame:

| Field | Size | Description |
|---|---|---|
| `seq` | 4 | Frame number. Gaps are normal and mean dropped frames |
| `rev` | 8 | Surface revision this frame shows |
| `capture_us` | 8 | Host capture time, microseconds, in the host's monotonic clock |
| `send_us` | 8 | Host time at the end of encoding |
| `flags` | 1 | `0x01` keyframe, `0x02` refinement (sharpening a still image) |
| `reserved` | 3 | MUST be 0 |

**VIDEO_SLICE** carries the same header on every slice with `index` and `count` (`uint16` each) added, then the slice bytes. A slice MUST fit the path's largest message. Recovery is by NACK and optional FEC, to be specified with the QUIC datagram binding. A frame not completed in time is abandoned, and the receiver reports it.

### 3.3 Sender rules

- **Drop before encoding.** An encoded delta cannot be dropped without breaking the frames after it, so a sender drops frames before the encoder, keeping only the newest captured frame while the encoder or the viewer is behind.
- **Limit frames in flight.** A sender MUST limit how many frames it has sent that the receiver has not yet acknowledged with FRAME_ACK. A fixed limit caps the frame rate on a long path, so a sender SHOULD allow at least 2, plus what the path's lowest measured round trip holds at the current frame rate, up to a cap. It SHOULD use the lowest round trip and not the current one, which includes any queue, so that a queue cannot raise its own limit. In one test with acknowledgements delayed by 100 ms, a fixed limit of 2 reached 19 fps and this reached 60 fps. xumux credit cannot do this: `initialWindow` must be at least `maxMessageSize`, which must hold the largest keyframe, so a full window can be seconds of video.
- **Stay well under the link's capacity.** A sender SHOULD keep video well below its estimated capacity, and SHOULD spread each frame's bytes over the frame interval. Bytes already in flight queue at the bottleneck, where no priority can overtake them: in one test on a 50 Mbit/s link, input round trip was about 0.3 ms beside 20 Mbit/s of video, 10 ms beside 40 Mbit/s, and 80-340 ms beside a sender running flat out.
- **Adapt to delay, not only loss.** A sender SHOULD use channel probes ([xumux §8.1](https://github.com/deftai/xumux/blob/main/spec/core.md#81-ping-0x10-and-pong-0x11)), which include send queues, along with receiver reports, to lower bitrate as delay grows. It MUST NOT wait for loss, because on TCP loss arrives long after the queue has formed.
- **Avoid keyframe spikes.** Hosts SHOULD use intra refresh or reference invalidation where the encoder allows, and send a full keyframe only when needed.
- **Send nothing when nothing changes.** A still screen uses no bandwidth, except for refinement.
- **Refine at rest only if it helps.** After motion stops, a host MAY send refinement frames at higher quality. They can't exceed what the pixel format carries: in one test of H.264 in 4:2:0, text-heavy pages already reached the limit set by chroma subsampling, and refinement frames added little.

### 3.4 Feedback

On `vroom.graphical.feedback`:

| Type | Name | Body |
|---|---|---|
| `0x01` | RECEIVER_REPORT | CBOR, about once a second: `received`, `decoded`, `dropped`, `decode_ms`, `queue_ms`, `display_ms` |
| `0x02` | KEYFRAME_REQUEST | CBOR: `surface`, `reason` |
| `0x03` | NACK | CBOR: `seq`, `missing`, on unreliable paths only |
| `0x04` | FRAME_ACK | CBOR: `surface`, `seq`, sent when a frame is decoded |

### 3.5 WebRTC media tracks

xumux's WebRTC binding allows media tracks next to its DataChannels. Over that binding a host MAY send video and audio as tracks, for browser sessions that need NAT traversal and congestion control from the browser's media stack. STREAM_STATE then reports `source: "track"`, and no video channel opens.

---

## 4. Input

Requires a [lease](./VROOM-Session.md#4-the-control-lease). A host MUST ignore input from a client without one and SHOULD send ERROR `4600` at most once a second.

### 4.1 Pointer channel

| Type | Name | Fields |
|---|---|---|
| `0x01` | POINTER_MOVE | `seq:u16`, `surface:u16`, `x:i32`, `y:i32` |
| `0x02` | POINTER_MOVE_REL | `seq:u16`, `dx:i16`, `dy:i16` |

`x` and `y` are 24.8 fixed-point logical units ([Session §3](./VROOM-Session.md#3-surfaces-and-coordinates)). The channel may be unordered, so a receiver MUST discard a message whose `seq` is older than the newest seen, using serial-number comparison modulo 2¹⁶.

### 4.2 Input channel

| Type | Name | Fields |
|---|---|---|
| `0x10` | BUTTON_DOWN | `surface:u16`, `x:i32`, `y:i32`, `button:u8` |
| `0x11` | BUTTON_UP | Same |
| `0x12` | SCROLL | `surface:u16`, `x:i32`, `y:i32`, `dx:i32`, `dy:i32`, `unit:u8` (0 pixels, 1 lines) |
| `0x13` | KEY_DOWN | `usage:u32`, `mods:u8` |
| `0x14` | KEY_UP | Same |
| `0x15` | TEXT | UTF-8 |
| `0x16` | TOUCH | `surface:u16`, `id:u8`, `phase:u8` (0 start, 1 move, 2 end, 3 cancel), `x:i32`, `y:i32`, `pressure:u8` |
| `0x17` | RELEASE_ALL | Empty |

- Buttons: 0 left, 1 middle, 2 right, 3 back, 4 forward. Button events carry a position because pointer movement may have been lost.
- **Scroll is reliable**, because losing a scroll event loses the user's movement.
- **Keys are physical.** `usage` is a USB HID usage: `(page << 16) | usage`. KEY_DOWN means a key at a position, independent of layout. The host injects it through the OS and applies the host's layout.
- **Text is not keys.** TEXT inserts characters regardless of layout and carries IME output and pasted text. Agents typing words SHOULD use TEXT.
- `mods`: bit 0 Shift, 1 Control, 2 Alt/Option, 3 Meta/Command/Windows, 4 CapsLock.
- A host MUST release held keys and buttons when a lease ends, when the channel closes, and on RELEASE_ALL, so that a lost connection cannot leave a key held down.

### 4.3 Cursor channel

| Type | Name | Fields |
|---|---|---|
| `0x01` | CURSOR_POS | `seq:u16`, `surface:u16`, `x:i32`, `y:i32`, `shape:u16`, `visible:u8` |

Shapes travel reliably, on `vroom.graphical.control`: `0x10` CURSOR_SHAPE, CBOR `{shape, png, hot_x, hot_y, scale}`. Hosts send each shape once, and clients cache it by ID. Streams SHOULD exclude the cursor from captured frames, and viewers draw it.

### 4.4 Clipboard

On `vroom.graphical.control`: `0x20` CLIPBOARD_OFFER, `{id, types}`, and `0x21` CLIPBOARD_GET, `{id, type}`, answered by `0x22` CLIPBOARD_DATA, `{id, type, data}`, sent in either direction. Only offers travel eagerly, so large content moves only on demand, and a client can decide whether it wants the content at all. A client MUST opt in, and a host SHOULD let its owner disable clipboard sync.

---

## 5. Agent access

On `vroom.graphical.agent`. The agent needs no video or decoder, and each answer is available at once from the latest captured frame. All actions take a `request_id`, and `guard` and `wait` as in [Session](./VROOM-Session.md#5-revisions-and-guards).

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x01` | SCREENSHOT_REQ | C→S | `surface`, `region?`, `max_edge?`, `format` (`png`, `jpeg`, `webp`), `quality?`, `if_newer_than?` |
| `0x02` | SCREENSHOT | S→C | `rev`, `origin`, `scale`, `format`, `data`, `changed_since?`, or `unchanged: true` |
| `0x03` | CHANGES_REQ | C→S | `surface`, `since_rev` |
| `0x04` | CHANGES | S→C | `rev`, `regions` |
| `0x05` | ACT | C→S | `action`, `at`, `surface`, `args`, `guard?`, `wait?` |
| `0x06` | ACT_RESULT | S→C | [Act result](./VROOM-Session.md#7-the-act-result), `rung: "pixel"` |

- **`if_newer_than`** avoids sending an image the agent already has: the host answers `unchanged: true` if the surface has not changed since that revision.
- **`changed_since`** lists the regions changed since the given `rev`, so an agent can request crops and not spend tokens on whole screens.
- **Actions:** `click`, `double_click`, `right_click`, `move`, `drag` (`to`), `scroll` (`dx`, `dy`), `key` (a chord such as `"Meta+S"`), and `text`.
- **`at`** is `{x, y}` in logical units. Agents working from an image convert it using `origin` and `scale`.
- Hosts SHOULD keep the latest frame for each surface in use, without copying it until asked.

---

## 6. Audio

Opus. On a channel: `0x01` AUDIO_CONFIG (CBOR) and `0x02` AUDIO_PACKET, `seq:u32`, `capture_us:u64`, then one Opus packet. Audio is only for sessions that ask for it. Over WebRTC, a host MAY use audio tracks instead ([§3.5](#35-webrtc-media-tracks)).

Voice interaction is a matter for the agent above this protocol: speech arrives as audio or text, and the agent acts through whichever type fits.

---

## 7. Host requirements

| Platform | Capture | Input | Notes |
|---|---|---|---|
| macOS | ScreenCaptureKit | CGEvent | Screen Recording and Accessibility permissions. Report absences with `4606` |
| Windows | Windows.Graphics.Capture or Desktop Duplication | SendInput | Elevated windows and the secure desktop need a service helper |
| Linux X11 | XShm, XDamage | XTest | |
| Linux Wayland | PipeWire via the ScreenCast portal | RemoteDesktop portal, or compositor protocols | Declare limits on unattended access |

Hosts MUST report what they cannot do, in SESSION_WELCOME capabilities or with `4606`, and MUST NOT fail silently. A capture that returns black because a permission is missing is a defect in the host.

---

## 8. Security

- A host MUST require authentication on any non-local binding, and on local listeners a browser could reach.
- Sharing a real desktop exposes everything on it. Hosts SHOULD show a visible indicator while anyone is watching, not only while acting, and SHOULD let the owner scope a session to specific windows.
- Agents on a real desktop are subject to [lease](./VROOM-Session.md#4-the-control-lease) suspension when the owner uses the machine.

---

## Appendix A: Changes from 0.1

| 0.1 | 0.2 | Why |
|---|---|---|
| DOM `key` and `code` | Physical HID usage, plus TEXT | OS-level injection, and IME and layout independence |
| `uint16` viewport pixels | Surface plus 24.8 fixed-point logical units | Multiple monitors, HiDPI |
| Pointer without sequence numbers | `seq` on pointer and cursor | The channel is unordered, so a late packet moved the pointer backwards |
| SCROLL on the unreliable channel | SCROLL on the reliable channel | Lost scroll events lost the user's movement |
| JPEG video fallback | H.264 on every transport | Bandwidth and quality |
| Video only as a WebRTC track | A video channel, with tracks as an option | Works on every xumux binding |
| MODE | [Lease](./VROOM-Session.md#4-the-control-lease) | One model of who may act |
| COMMAND, NAVIGATE, PASTE | Removed: see [Browsable](./VROOM-Browsable.md), [Structural](./VROOM-Structural.md), TEXT | Two ways to do the same thing |
| Voice mode | Removed. Audio channels remain | A concern of the agent above the protocol |
| STATE, NOTIFICATION, CHAT | AGENT_STATE and CHAT in Session; act results | Shared by every type |
| CURSOR on the control channel | Cursor channel, shapes cached by ID | The cursor must not wait |
| Single-field clipboard | Offer and fetch | Large content on demand, with client opt-in |
| `omux/*` channels | `vroom.graphical.*`, xumux 0.3 | Current xumux |

---

## Design lineage

| Project | Insight |
|---|---|
| **n.eko** | Named events, debuggable control channel |
| **Selkies-GStreamer** | Compact binary for pointer events |
| **JetKVM** | Separate channels per input type, physical key input |
| **SocketPipe** | Transport-agnostic framing, which became xumux |

## Status

Draft 0.2.0. Subject to change.

## License

MIT
