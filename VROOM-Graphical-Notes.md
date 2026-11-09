# VROOM-Graphical: Spec Review Notes

**Not normative.** These are open questions about [VROOM-Graphical 0.2](./VROOM-Graphical.md), for discussion. Nothing here changes the protocol.

Date: 2026-11-08

## Evidence

The frame sizes below come from software AV1 encoders (libaom and SVT-AV1) on a single Apple M4, using a synthetic screen clip at a 3 or 8 Mbit/s target. Treat them as rough sizes, not limits. The frame-rate figures are calculated from the spec's own numbers, not measured.

## Open issues

### 1. A fixed in-flight limit caps frame rate on long links (§3.3, §3.4)

The RECOMMENDED limit is 2 unacknowledged frames, and FRAME_ACK is sent when a frame is decoded. So throughput can't exceed about 2 frames per (RTT + decode time), before counting transmission time.

| RTT | Decode | Frame-rate ceiling |
|---|---:|---:|
| 50 ms | 7 ms | about 35 fps |
| 100 ms | 7 ms | about 19 fps |

**Suggestion:** scale the limit with measured RTT, or limit bytes in flight instead of frames, while keeping the goal of not queueing stale video.

### 2. "Effectively lossless" may be unreachable in 4:2:0 (§1 goal 4, §3.3)

"Refine at rest" asks for refinement until the image is effectively lossless. With 4:2:0 video, chroma detail is subsampled, so refinement can't fully restore colored text. H.264 is the mandatory baseline and is typically 4:2:0 in practice, though the spec doesn't state a profile.

**Suggestion:** either define the target as luma quality for 4:2:0 streams, or add a 4:4:4 path. A 4:4:4 path needs decoder support checks, which have not been done.

### 3. Pacing frames larger than one interval's budget (§3.3)

Senders SHOULD spread each frame's bytes over the frame interval, but some frames can't fit. The largest 1080p frames observed were 97-205 KB, which take about 0.26-0.56 s to send at 3 Mbit/s, against a 33 ms interval at 30 fps.

**Suggestion:** say how pacing and frame dropping behave when a frame exceeds one interval's budget. Intra refresh (already a SHOULD) reduces how often this happens, but the first frame still needs a rule.

### 4. Suggested values for keyframe-sized limits (§3.3, [Session §1](./VROOM-Session.md))

`maxMessageSize` must hold the largest keyframe, and `initialWindow` must be at least that value. The largest frame observed was 593 KB, at 4K30 with an 8 Mbit/s target and libaom at speed 11.

**Suggestion:** publish suggested values per resolution, with margin, once they've been measured on more encoders, including hardware H.264.

### 5. Recovery from FRAME_ACK (§3.3, §3.4)

"Avoid keyframe spikes" mentions reference invalidation. FRAME_ACK already tells the sender which frames were decoded, so after a loss it could predict from an acknowledged frame rather than send a keyframe.

**Suggestion:** add an informative note, which needs no wire change. Encoder support for choosing references this way has not been checked.

## Not yet tested

- Hardware H.264 against AV1. H.264 is the mandatory baseline, so optional AV1 has to justify itself against it.
- The §3.3 sender rules, which the test harness doesn't implement yet: it queues frames instead of dropping before encoding, and has no in-flight limit.
- How quickly and how sharply refinement converges on a still screen.
- Decode latency through WebCodecs in browsers. The tests called VideoToolbox and dav1d directly.
