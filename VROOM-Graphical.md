# VROOM-Graphical

**Virtual Remoting Over Open Methods: Graphical Protocol**

*The pixels of a whole desktop, fast enough to work in, for humans and agents.*

Version: 0.2.0-draft | Status: Draft | Date: 2026-11-13 | Base: [VROOM-Session](./VROOM-Session.md)

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

VROOM-Graphical carries displays and windows as pixels, with OS-level input. One capture engine serves two consumers: humans get low-latency video, and agents get the latest frame on demand, with changed regions and settling.

**Profile:** `vroom-graphical/1`. Surfaces are `display:<n>` and `window:<n>`. Binary messages name a surface by its number, a `uint16` assigned by the host and listed in SURFACES as `num`.

**Changes from 0.1:** the protocol now targets OS desktops, not a browser driven by Playwright. Browser commands moved to [Browsable](./VROOM-Browsable.md). See [Appendix A](#appendix-a-changes-from-01).

**Implementation note:** each channel's payload format (structured messages, binary structs, encoded video and audio) is a **channel codec** on xumux, decoded after demux. WebRTC media tracks (§3.5) sit beside the mux. See [I/O stack](./docs/io-stack.md).

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
| `vroom.graphical.control` | Reliable, ordered | Both | Stream setup, performance records, clipboard, cursor shapes |
| `vroom.graphical.video.<surface>` | Reliable, ordered, unless the client asks otherwise (§3.2) | S→C | Video |
| `vroom.graphical.feedback` | Reliable, ordered | C→S | Receiver reports, keyframe requests |
| `vroom.graphical.cursor` | Unreliable, unordered | S→C | Cursor position |
| `vroom.graphical.pointer` | Unreliable, unordered | C→S | Pointer movement |
| `vroom.graphical.input` | Reliable, ordered | C→S | Buttons, scroll, keys, text, touch |
| `vroom.graphical.agent` | Reliable, ordered | Both | [Agent access](#5-agent-access) |
| `vroom.graphical.audio.<id>` | Unreliable, unordered, where available, for `"timely"` audio; reliable, ordered for `"complete"` ([§9](#9-profiles)) | Both | Audio |

Channels open when needed: a watching client opens no input channels, and an agent without a viewer opens no video.

On a reliable path, `cursor` and `pointer` still work: senders coalesce to the latest value while the channel lacks credit.

---

## 3. Video

### 3.1 Starting a stream

On `vroom.graphical.control`:

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x01` | STREAM_START | C→S | `surface`, `codecs` (preference order, as WebCodecs codec strings), `max_fps?`, `max_bitrate?`, `max_edge?`, `include_cursor?`, `path?` (`"datagram"` asks for §3.2's experimental path), `stats?` (`true` asks for STREAM_STATS), `profile?` ([§9](#9-profiles)) |
| `0x02` | STREAM_STOP | C→S | `surface` |
| `0x03` | STREAM_STATE | S→C | `surface`, `codec`, `fps`, `bitrate`, `source`, `encoder`, `path?` (`"reliable"` or `"datagram"`; absent when `source` is `"track"`, which has no video channel, §3.5), `profile` |
| `0x04` | VIDEO_CONFIG | S→C | As in §3.2, with `surface`, on the datagram path, where the video channel could lose it |
| `0x05` | STREAM_STATS | S→C | A performance record (§3.4), about once a second, only if STREAM_START asked |
| `0x06` | PROFILE | Both | C→S: `profile`, `audio_delivery?` ([§9](#9-profiles)), without starting video. S→C: the `profile` and `audio_delivery` in use, in answer |

The host opens `vroom.graphical.video.<surface>` after STREAM_START. H.264 is the baseline, and every host and client MUST support it. Clients discover decoders at run time, with `VideoDecoder.isConfigSupported()` in browsers.

### 3.2 Messages on the video channel

| Type | Name | Body |
|---|---|---|
| `0x01` | VIDEO_CONFIG | CBOR: `codec`, `coded_w`, `coded_h`, `description?`, `colorspace`, `scale`, `origin`. Sent before the first frame and whenever these change |
| `0x02` | VIDEO_FRAME | One encoded frame, on a reliable path |
| `0x03` | VIDEO_SLICE | Part of one encoded frame, on the datagram path |

**VIDEO_FRAME** is a 32-byte binary header, then the frame:

| Field | Size | Description |
|---|---|---|
| `seq` | 4 | Frame number. Gaps are normal and mean dropped frames |
| `rev` | 8 | Surface revision this frame shows |
| `capture_us` | 8 | Host capture time, microseconds, in the host's monotonic clock |
| `send_us` | 8 | Host time at the end of encoding |
| `flags` | 1 | `0x01` keyframe, `0x02` refinement (sharpening a still image) |
| `reserved` | 3 | MUST be 0 |

**VIDEO_SLICE** carries the same header on every slice with `index` and `count` (`uint16` each) added, then the slice bytes. A slice MUST fit the path's largest message. Recovery is by NACK. A frame not completed in time is abandoned, and the receiver reports it.

**Slices are experimental, and senders SHOULD use VIDEO_FRAME on a reliable channel unless the client asks.** A delta frame needs the frame before it, so later frames wait for a lost packet on either path, and QUIC's own recovery did better than NACKs. In one test of a synthetic stream of about 1 Mbit/s with 20 ms of added round trip, at 3% packet loss the 99th percentile from page to canvas was 59-71 ms for QUIC's session stream and 102-106 ms for slices. At 10% loss the stream held 33-38 fps, while slices failed to start in 2 of 3 runs and reached 0.3 fps in the third. FEC and higher bitrates weren't tested, and may change this.

### 3.3 Sender rules

- **Drop before encoding.** An encoded delta cannot be dropped without breaking the frames after it, so a sender drops frames before the encoder, keeping only the newest captured frame while the encoder or the viewer is behind.
- **Limit frames in flight.** A sender MUST limit how many frames it has sent that the receiver has not yet acknowledged with FRAME_ACK. A fixed limit caps the frame rate on a long path, so a sender SHOULD use `ceil(lowest_rtt × target_fps) + 1`, with `lowest_rtt` in seconds and `target_fps` in frames a second, kept between 2 and a cap (16 is RECOMMENDED). For example, 0.021 s at 60 fps gives 3. It SHOULD use the lowest round trip and not the current one, which includes any queue, so that a queue cannot raise its own limit. It SHOULD use the target frame rate and not the achieved one, which falls when the limit is too tight and would keep it there. It MUST count only frames it really sent: the encoder may drop a frame, nothing will ever acknowledge it, and counting it can stop the stream for good. In one test with acknowledgements delayed by 100 ms, a fixed limit of 2 reached 19 fps and this reached 60 fps. xumux credit cannot do this: `initialWindow` must be at least `maxMessageSize`, which must hold the largest frame, so a full window can be seconds of video.
- **Stay well under the link's capacity.** A sender SHOULD keep video well below its estimated capacity. A sender MAY pace a frame's bytes so that it doesn't reach the bottleneck all at once, but SHOULD then pace at twice its bitrate target or faster: pacing at the target rate itself would hide spare capacity from `rate_kbps` (§3.4). Bytes already in flight queue at the bottleneck, where no priority can overtake them: in one test on a 50 Mbit/s link, input round trip was about 0.3 ms beside 20 Mbit/s of video, 10 ms beside 40 Mbit/s, and 80-340 ms beside a sender running flat out.
- **Send frames bigger than an interval whole.** A stream's first keyframe, and sometimes later frames, take longer to send than one frame interval, even paced at twice the target: in one test at 3 Mbit/s, the largest frames took about 0.1-0.3 s. A sender MUST send such a frame whole, at its pacing rate if it paces, and MUST NOT hold back part of it for a later interval's budget. While it is being sent, the sender keeps dropping captures before encoding as above, so the viewer next gets the newest picture rather than a backlog. Much of its delay comes from its size and not from a queue, so a sender SHOULD take off its transfer time at the delivery rate before judging its delay, and treat frames sent while it is unacknowledged as it does frames behind a keyframe (below). Once it is acknowledged, frames are judged as usual, and delay beyond the transfer time still counts as queueing. Intra refresh avoids most such frames after the first.
- **Adapt to delay, not only loss.** A sender SHOULD use channel probes ([xumux §8.1](https://github.com/deftai/xumux/blob/main/spec/core.md#81-ping-0x10-and-pong-0x11)), which include send queues, along with receiver reports, to lower bitrate as delay grows. It MUST NOT wait for loss, because on TCP loss arrives long after the queue has formed. It SHOULD ignore the delay of a keyframe and of frames sent while one is unacknowledged, which comes from the keyframe's size and not from a queue, and SHOULD forget old delay by time and not by sample count, since on a nearly still screen samples are far apart. In one session on a 21 ms path without either rule, one keyframe took the target from 12 to about 1 Mbit/s with nothing queued. For the same reason, a sender SHOULD take off each frame's transfer time at the delivery rate before judging its delay: on a 5G link delivering 10 Mbit/s, typical frames took 15-20 ms longer than small ones with no queue, and counting that held the target near 1 Mbit/s where it rose to about 6 without it.
- **Measure the path, and hold a share.** A sender SHOULD start below its maximum until it has measured the path, and SHOULD keep its bitrate below the delivery rate that `rate_kbps` reports. If cutting the bitrate doesn't shorten the delay, the queue is someone else's, as when a download fills it, and a sender MAY hold its rate rather than give up its share, but MUST still cut while the delay keeps growing. In one test on a simulated 5 Mbit/s path with a competing download, these rules took the 99th percentile from about 1 s to about 0.25 s.
- **Avoid keyframe spikes.** Hosts SHOULD use intra refresh or reference invalidation where the encoder allows, and send a full keyframe only when needed.
- **Send nothing when nothing changes.** A still screen uses no bandwidth, except for refinement.
- **Refine at rest only if it helps.** After motion stops, a host MAY send refinement frames at higher quality. They can't exceed what the pixel format carries: in one test of H.264 in 4:2:0, text-heavy pages already reached the limit set by chroma subsampling, and refinement frames added little.

### 3.4 Feedback

On `vroom.graphical.feedback`:

| Type | Name | Body |
|---|---|---|
| `0x01` | RECEIVER_REPORT | CBOR, about once a second: `surface`, `received`, `decoded`, `dropped`, `decode_ms`, `queue_ms`, `display_ms`, and optionally the fields below |
| `0x02` | KEYFRAME_REQUEST | CBOR: `surface`, `reason` |
| `0x03` | NACK | CBOR: `seq`, `missing`, on unreliable paths only |
| `0x04` | FRAME_ACK | CBOR: `surface`, `seq`, `rate_kbps?`, sent when a frame is decoded or discarded. Frames a receiver discards MUST be acknowledged too, or the sender's limit can stop the stream while it waits for a keyframe it can't send |

`rate_kbps` is how fast the frame arrived: its size divided by the time from when it could have started arriving to when it had. It could start once sent and across the path, but not before the frame ahead of it had arrived. The time across the path is the lowest recent `send_us`-to-arrival difference over every frame, small ones included, since a small frame's own transfer takes almost no time.

A receiver MUST omit `rate_kbps` when the frame is too small to time (4000 bytes or more is a reasonable floor), or when the time it took is under 1 ms. That includes a frame that set the lowest difference itself, whose time would be zero. If only large frames arrived recently, the lowest difference includes a transfer, and samples read high, so a sender SHOULD use the median of recent samples rather than any one.

This samples what the path delivers to this stream only if the frame left as one burst, or paced faster than the path could carry it, as in [SQP](https://arxiv.org/abs/2207.11857). A frame paced at the sender's own bitrate arrives at that rate whatever the path could carry, which is why §3.3 asks a sender that paces to pace at twice its target or faster. Otherwise, keeping below `rate_kbps` would hold the bitrate near where it started.

A RECEIVER_REPORT covers the time since the last one, and times are medians over it:

- `dropped` counts frames that arrived and were never shown.
- `decode_ms` runs from submitting a frame to the decoder until it comes out.
- `queue_ms` is how far a frame's arrival rose above its lowest recent delay, comparing `send_us` with the receiver's clock. This needs no shared clock, and unlike gaps between frames it isn't fooled by a still screen, which sends nothing.
- `display_ms` runs from decoded to drawn.
- Optional fields are `drawn`, `stalls` (runs of frames with `queue_ms` of 100 ms or more), `decode_p95_ms`, `queue_p95_ms`, `queue_max_ms`, `display_p95_ms`, `canvas_w`, `canvas_h` and `hidden`.

A host MAY combine reports with its own numbers into a performance record, which is experimental and whose fields may change. Records carry no screen content. STREAM_STATS carries one record as a CBOR map, covering the time since the last one:

- `surface`, and `interval_ms`, the time the record covers.
- `path`, as in STREAM_STATE.
- `host`, a map: `frames`, `keyframes` and `encoder_dropped` (counts), `bytes`, `bitrate_target` (bits a second), and `in_flight` and `in_flight_limit` (frames, §3.3).
- `viewer?`, the latest RECEIVER_REPORT's fields, once there is one.
- `estimated_latency_ms?`, labelled as below.

A host MAY add other keys, and a client MUST ignore keys it doesn't know. Two machines share no clock, so a host that reports latency MUST label it an estimate: for example capture to send, plus half the acknowledgement round trip without decoding, plus decoding and drawing. In testing on one machine that sum came within about 4 ms of measured median latency, and it missed the worst frames: 36 ms against a 99th percentile of 160 ms at 10% packet loss. It also leaves out the wait before capture and the display's own lag.

### 3.5 WebRTC media tracks

xumux's WebRTC binding allows media tracks next to its DataChannels. Over that binding a host MAY send video and audio as tracks, for browser sessions that need NAT traversal and congestion control from the browser's media stack. STREAM_STATE then reports `source: "track"`, and no video channel opens.

---

## 4. Input

Requires an `input` [lease](./VROOM-Session.md#4-the-control-lease) on the surface. A host MUST ignore input from a client without one and SHOULD send ERROR `4600` at most once a second.

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
- A host MUST release held keys and buttons when a lease ends or its holder leaves it, when the channel closes, and on RELEASE_ALL, so that a lost connection cannot leave a key held down. With a shared lease, it releases only that holder's ([Session §4.3](./VROOM-Session.md#43-conflicts-rank-and-sharing)).

### 4.3 Cursor channel

| Type | Name | Fields |
|---|---|---|
| `0x01` | CURSOR_POS | `seq:u16`, `surface:u16`, `x:i32`, `y:i32`, `shape:u16`, `visible:u8` |

Shapes travel reliably, on `vroom.graphical.control`: `0x10` CURSOR_SHAPE, CBOR `{shape, png, hot_x, hot_y, scale}`. Hosts send each shape once, and clients cache it by ID. Streams SHOULD exclude the cursor from captured frames, and viewers draw it.

### 4.4 Clipboard

On `vroom.graphical.control`: `0x20` CLIPBOARD_OFFER, `{id, types}`, and `0x21` CLIPBOARD_GET, `{id, type}`, answered by `0x22` CLIPBOARD_DATA, `{id, type, data}`, sent in either direction. Only offers travel eagerly, so large content moves only on demand, and a client can decide whether it wants the content at all. A client MUST opt in, and a host SHOULD let its owner disable clipboard sync.

Agents that need to set the clipboard in step with their actions use the `clipboard` action (§5) instead. A client opts in with `0x23` CLIPBOARD_SYNC, `{on}`, and opts out by sending it with `on: false`. Until a client opts in, the host sends it no offers and answers its CLIPBOARD_GET and CLIPBOARD_OFFER with `4603`. On opting in, the host SHOULD offer what its clipboard holds at that moment, so a client can read it without waiting for a change. The host answers CLIPBOARD_SYNC with CLIPBOARD_SYNC, `{on}`, giving the state now in force, and the opt-in lasts for the connection. A host that has disabled clipboard sync answers with `4603`, and one without clipboard support with `4605`.

---

## 5. Agent access

On `vroom.graphical.agent`. The agent needs no video or decoder, and each answer is available at once from the latest captured frame. Every request on this channel carries a `request_id`, which the host echoes in its answer and in any ERROR, so answers can be matched to requests even when they finish out of order. Actions also take `guard` and `wait` as in [Session](./VROOM-Session.md#5-revisions-and-guards).

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x01` | SCREENSHOT_REQ | C→S | `surface`, `region?`, `max_edge?`, `format` (`png`, `jpeg`, `webp`), `quality?`, `budget_ms?`, `if_newer_than?`, `patches_since?`, `tile?` |
| `0x02` | SCREENSHOT | S→C | `rev`, `origin`, `scale`, `format`, `lossless`, then `data` and `changed_since?`, or `since`, `tile` and `patches`, or `unchanged: true` |
| `0x03` | CHANGES_REQ | C→S | `surface`, `since_rev` |
| `0x04` | CHANGES | S→C | `rev`, `regions` |
| `0x05` | ACT | C→S | `action`, `at?`, `surface`, `args`, `guard?`, `wait?` |
| `0x06` | ACT_RESULT | S→C | [Act result](./VROOM-Session.md#7-the-act-result), `rung: "pixel"` |
| `0x07` | READ_TEXT_REQ | C→S | `surface`, `region?`, `level?` (`"accurate"`, the default, or `"fast"`) |
| `0x08` | READ_TEXT | S→C | `rev`, `lines`: `[{text, confidence?, x, y, w, h}]` |

- **`if_newer_than`** avoids sending an image the agent already has: the host answers `unchanged: true` if the surface has not changed since that revision.
- **`patches_since`** asks for only what changed since a revision the agent has a picture of. The host answers with `patches`, each `{x, y, w, h, width, height, data}` in logical units, which don't overlap and cover every change. Laying them over that picture gives the new one exactly only if both are lossless (`lossless: true`): a lossy base or lossy patches each keep their own compression errors, so a client that needs exact pixels SHOULD ask for a lossless format. A host MAY send the whole picture instead, as when much of the surface changed or it no longer knows what changed, and MUST then send `data` and no `patches`. Patches can't be combined with `region` or `max_edge`, and the picture at the given revision MUST be one the client holds whole and unscaled: of the whole surface, without `region` or `max_edge`. Patches can't restore area outside a crop or detail lost to scaling. If the surface's size or `scale` changed since that revision, the host MUST send the whole picture.
- **`tile`** is the patch grid in frame pixels, while patch positions are in logical units, related by `scale`. Smaller tiles send less unchanged area, and larger ones fewer, better-compressed images. A host spots changes on some grid of its own, and SHOULD round a requested size up to a multiple of it, since coarser tiles can be made exactly from finer ones and finer ones can't be known. Hosts SHOULD allow tiles up to 512 pixels. It says the size it used.
- **`changed_since`** lists the regions changed since the given `rev`, so an agent can request crops and not spend tokens on whole screens.
- **Actions:** `click`, `double_click`, `right_click`, `move`, `drag` (`to`), `scroll` (`dx`, `dy`), `key` (a chord such as `"Meta+S"`), `text`, `clipboard`, and `open`. In the `text` action, `\n`, `\r\n` and `\r` each press Return once and `\t` presses Tab, so multi-line text arrives as it would be typed; a literal tab character can't be typed this way. The input channel's TEXT (`0x15`) is unchanged and sends characters as they are.
- **`clipboard`** puts `args.text` on the host's clipboard. Unlike §4.4's offers, which travel on the control channel, it is ordered with the actions around it and has a result, so an agent that sets the clipboard and then pastes gets what it set. It needs an `act` lease.
- **`open`** opens an application (`args.app`, a name or platform identifier), a URL (`args.url`) or a file (`args.path`) on the host, as its user would, which usually brings it to the front. It takes no `at`. It needs an `act` lease, and hosts SHOULD let their owner restrict or disable it ([Security](#8-security)): unlike input, it can start software the screen doesn't show.
- **READ_TEXT** reads the text on a surface, or on `region` of it, from the latest frame, and answers the revision it read. Lines come in reading order, with boxes in logical units, and `confidence` from 0 to 1 where the recognizer gives one. Boxes are per line, so an agent can find a line and click within it without an image or a model call; a word's position inside a line is an estimate. `level` is a hint, and a host MAY use one level for both. It is OCR, not the accessibility tree: it sees what is drawn, including in canvases and remote sessions, and knows nothing of roles. For roles and names, use [VROOM-Structural](./VROOM-Structural.md). A `region` outside the surface is answered with `4602`, and a host that can't read text answers `4605`.
- **Leases.** The pointer, `key` and `text` actions need an `input` lease on the surface, as input does. `clipboard` and `open` need an `act` lease: they aren't input a person could give at that spot.
- **`at`** is `{x, y}` in logical units, required by the coordinate actions. Agents working from an image convert it using `origin` and `scale`. `clipboard` and `open` take no `at`, so their guards default to `surface` scope, and a host refuses a `region` guard on them with `4605`.
- **Formats:** `png` and `webp` without `quality` are lossless, and hosts MUST keep them lossless. Lossless describes the encoding: with `max_edge`, the scaled image is kept exactly, not the screen's own pixels. Lossy formats cost text: in one test JPEG at quality 80 lost up to 18% of words to OCR, and lossy WebP up to 11%. Lossless WebP is much smaller on text: in one test on rendered 1440x900 screens it was 4-14 times smaller than PNG at fast settings, for 20-45 ms of encoding, but it was slow on photographic content (0.2-0.6 s) for little gain. A host SHOULD cap the effort it spends, and MAY take `budget_ms` as the most a client will wait for a smaller file. 0 asks for the fastest encoding. Model APIs charge for images by their dimensions, not their size in bytes, so `max_edge`, `region` and `if_newer_than` usually save more than the format.
- Hosts SHOULD keep the latest frame for each surface in use, without copying it until asked.

---

## 6. Audio

Opus. On a channel: `0x01` AUDIO_CONFIG (CBOR) and `0x02` AUDIO_PACKET, `seq:u32`, `capture_us:u64`, then one Opus packet. Audio is only for sessions that ask for it. AUDIO_CONFIG states its `delivery`, which [§9](#9-profiles) sets. Over WebRTC, a host MAY use audio tracks instead ([§3.5](#35-webrtc-media-tracks)).

Voice interaction is a matter for the agent above this protocol: speech arrives as audio or text, and the agent acts through whichever type fits.

---

## 7. Host requirements

| Platform | Capture | Input | Notes |
|---|---|---|---|
| macOS | ScreenCaptureKit | CGEvent | Screen Recording and Accessibility permissions. Report absences with `4606` |
| Windows | Windows.Graphics.Capture or Desktop Duplication | SendInput | Elevated windows and the secure desktop need a service helper |
| Linux X11 | XShm, XDamage | XTest | |
| Linux Wayland | PipeWire via the ScreenCast portal | RemoteDesktop portal, or compositor protocols | Declare limits on unattended access |

Hosts MUST report what they cannot do, in SESSION_WELCOME capabilities or with `4606`, and MUST NOT fail silently. A Graphical host lists `"read_text"` and `"open"` when it supports them, and `"clipboard"` when it offers clipboard sync. A capture that returns black because a permission is missing is a defect in the host.

---

## 8. Security

- A host MUST require authentication on any non-local binding, and on local listeners a browser could reach.
- Sharing a real desktop exposes everything on it. Hosts SHOULD show a visible indicator while anyone is watching, not only while acting, and SHOULD let the owner scope a session to specific windows.
- Agents on a real desktop are subject to [lease](./VROOM-Session.md#4-the-control-lease) suspension when the owner uses the machine.

---

## 9. Profiles

*Draft. The defaults below are a starting point to test, not measured results.*

People and agents want different things from the same stream. A person controlling a desktop wants it on time, and can do without some of it: a late frame or a late scrap of audio is worth nothing. An agent wants it complete, and can wait: a missed word or a blurred character is wrong, while 200 ms is usually fine.

| Priority | `interactive` | `fidelity` |
|---|---|---|
| 1 | Smooth control | Reliable control |
| 2 | Audio without gaps | Exact images |
| 3 | Smooth video | Smooth video |
| 4 | Image quality | Audio for speech recognition |

A client chooses with `profile` in STREAM_START, or with PROFILE on the control channel, which starts no video, for a client that only takes screenshots, acts or listens. It MAY override the settings that have their own fields: `max_fps`, `max_bitrate` and `path` in STREAM_START, `format`, `quality` and `budget_ms` in SCREENSHOT_REQ, and `audio_delivery` in PROFILE. The default is `interactive` for a client of kind `"human"` and `fidelity` for `"agent"` ([Session §2](./VROOM-Session.md#2-messages-on-vroomsession)), but the choice is about the task, not the client: a person reading code may want `fidelity`, and an agent playing a game `interactive`. The host reports what it chose in STREAM_STATE, or in its PROFILE answer, and MUST NOT fail a request for a profile it doesn't support, only report the one it used.

| Setting | `interactive` | `fidelity` |
|---|---|---|
| Input | As soon as possible: pointer moves coalesced, the cursor drawn by the viewer | Through ACT with guards and settling ([§5](#5-agent-access)), each with a result |
| Under congestion | Keep the frame rate, and give up resolution and quality first | Keep quality, and give up frame rate first |
| Frames in flight | §3.3's limit | §3.3's limit. More would only queue: in one simulation of a congested link, removing the limit raised frame rate from 53 to 55 fps and the 95th percentile of lag from 290 to 1081 ms |
| Encoder | Low-latency rate control | A minimum bitrate, giving up frame rate first. Exact pixels come from screenshots, since video refinement can't pass the 4:2:0 limit (§3.3) |
| Exact pixels | Rarely | Lossless screenshots ([§5](#5-agent-access)), with video for knowing when to take them |
| Audio `delivery` | `"timely"`: unreliable channel, Opus in-band FEC, the receiver conceals losses | `"complete"`: reliable channel, the receiver MUST NOT conceal |
| Audio format | 48 kHz, as the source has it | 16 kHz mono is enough for most speech recognition |

**Audio is where the two differ most.** For a person, a gap is worse than a guess. A receiver SHOULD conceal lost packets (Opus decoders do this when told a packet is missing, and in-band FEC recovers single losses from the next packet), and SHOULD use a jitter buffer that adapts to the path. Unlike a video frame, an audio packet can be decoded without the ones before it, so an unreliable channel suits audio even where it didn't suit video ([§3.2](#32-messages-on-the-video-channel)). For speech recognition, a guess is worse than a gap: concealment invents sound, which can become words that weren't said. A `"complete"` receiver MUST NOT conceal, and MUST mark any gap it can't fill.

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
| COMMAND, NAVIGATE, PASTE | Removed: see [Browsable](./VROOM-Browsable.md), [Structural](./VROOM-Structural.md), input TEXT (`0x15`) | Two ways to do the same thing. The later `clipboard` action (§5) only sets the clipboard, ordered with other actions; pasting is a `key` chord |
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
