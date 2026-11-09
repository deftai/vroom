# VROOM-Graphical: Spec Review Notes

**Not normative.** These are open questions about [VROOM-Graphical 0.2](./VROOM-Graphical.md), with test results, for discussion. Nothing here changes the protocol.

Date: 2026-11-08

## Evidence

All tests ran on a single Apple M4 (10 cores) on macOS 15.7. These are short runs on one machine, so treat the numbers as rough. M1, encoder CPU use and power, and real network congestion have not been tested.

**Clips.** Six lossless 1080p clips of 130 frames from the [AOM screen content test set](https://media.xiph.org/video/aomctc/test_set/b2_scc/), plus a synthetic clip for 4K results:

| Clip | Frame rate | Owner | License |
|---|---:|---|---|
| Debugging, Spreadsheet | 30 fps | Netflix Inc. | CC BY 4.0 |
| Wikipedia | 30 fps | Thomas Daede | CC0 |
| SceneComposition_1 | 15 fps | Yu-Chen Sun | CC BY-SA 4.0 |
| Slides1 | 30 fps | Tencent | Standards development and testing only |
| MissionControlClip3 | 60 fps | Apple Inc. (contains a Vidyo sequence) | Academic research and standards development only |

**Encoders.** Frames are fed on the capture clock, and latency runs from scheduled capture to encoded output.

| Encoder | Settings |
|---|---|
| libaom 3.15.3 | `AOM_USAGE_REALTIME`, CBR, `lag_in_frames=0`, `kf_max_dist=9999`, 8 threads, 2 tile columns at 1080p, `tune-content=screen`, speed 7 or 11 |
| SVT-AV1 4.3.0 | `rtc=1`, `rc=2`, `keyint=-1`, `scm=1`, preset 8. Screen tools are forced off at preset 9 and above |
| VideoToolbox H.264, HEVC | Hardware required, low-latency rate control, `RealTime`, no reordering, High or Main auto level, `AverageBitRate`, `MaxKeyFrameInterval=100000` |

**Scoring.** PSNR is against the lossless 4:2:0 source unless stated. In the codec comparison and sender tests, each source frame is scored against the latest decoded frame, because encoders can drop frames.

**Harness.** The code, per-clip results and per-frame stats are in the private repo [`deftai/vroom-lab`](https://github.com/deftai/vroom-lab), so not everyone reading this will have access.

## Open issues

### 1. A fixed in-flight limit caps frame rate on long links (§3.3, §3.4)

The RECOMMENDED limit is 2 unacknowledged frames, and FRAME_ACK is sent when a frame is decoded, so throughput can't exceed about 2 frames per (RTT + decode time). Measured with real encoders on a simulated link, with a 7 ms decode time and no congestion:

| Source | RTT | Limit of 2 | Limit scaled to RTT |
|---|---:|---:|---:|
| 30 fps | 100 ms | 14-16 fps | 30 fps (limit 5) |
| 60 fps | 100 ms | 14-15 fps | 54-58 fps (limit 8) |
| 60 fps | 50 ms | 21-24 fps | not tested |

Viewers hold frames for longer, so quality falls too: 42-43 dB against 48.5-49 dB on one clip at 100 ms RTT.

**Suggestion:** scale the limit with measured RTT, or limit bytes in flight instead of frames. The scaled limit used here was `ceil(fps * (RTT + decode)) + 1`. It has not been tested under congestion, where a larger limit could build a queue, so test that before changing the spec.

### 2. Making text crisp in the viewer (§1 goal 4, §3.3)

**Update:** measurement agreed, so §1 goal 4 and §3.3 no longer ask for effectively lossless video. Making text fully crisp in the viewer is still open.

Results that bear on what's left:

- **The 4:2:0 limit is large for colored text.** With no compression at all, a 4:2:0 round trip left the red channel of pink text on a dark background at 28.5 dB against the RGB original, compared with 57.7 dB for 4:4:4.
- **Refinement can still improve the 4:2:0 picture.** Scored against the 4:2:0 source, not the RGB original, a bitrate ladder over 11 frames took hardware encoders from 37-51 dB to 61-70 dB luma. So §3.3's "refinement frames add little" may depend on the starting quality and on what it's measured against.
- **How refinement is driven matters.** On VideoToolbox, raising `Quality` or forcing keyframes stalled at 30-47 dB, and capping QP stalled at 47-57 dB. Only raising the bitrate reached 60 dB and above.
- **Only AV1 reached an exact match** with the 4:2:0 source: 72-693 KB in total, against 61-70 dB at similar sizes for H.264 and HEVC.
- **4:4:4 has no hardware decode path found for AV1.** VideoToolbox rejects AV1 High profile (`-12910`). In WebCodecs, Chromium offers it in software only, and Firefox and Safari don't offer it.

**Suggestion:** say whether text in the viewer needs a separate 4:4:4 path, or whether the lossless screenshot in §5 is enough.

### 3. Pacing frames larger than one interval's budget (§3.3)

Senders SHOULD spread each frame's bytes over the frame interval, but some frames can't fit. The largest frame per encoder across the six clips at a 3 Mbit/s target was 76-229 KB, which takes about 0.2-0.6 s to send at that rate, against a 33 ms interval at 30 fps.

**Suggestion:** say how pacing and frame dropping behave when a frame exceeds one interval's budget. Intra refresh (already a SHOULD) reduces how often this happens, but the first frame still needs a rule.

### 4. Suggested values for keyframe-sized limits (§3.3, [Session §1](./VROOM-Session.md))

`maxMessageSize` must hold the largest keyframe, and `initialWindow` must be at least that value. Largest frames observed:

| Encoder | 1080p, 3 Mbit/s | 1080p, 6 Mbit/s |
|---|---:|---:|
| Hardware H.264 | 126 KB | 191 KB |
| Hardware HEVC | 122 KB | 176 KB |
| libaom | 218-229 KB | 315-323 KB |
| SVT-AV1 | 76 KB | 112 KB |

The largest frame on the synthetic 4K clip was 593 KB (libaom, 4K30, 8 Mbit/s).

**Suggestion:** publish suggested values per resolution, with margin.

### 5. Recovery from FRAME_ACK (§3.3, §3.4)

"Avoid keyframe spikes" mentions reference invalidation. FRAME_ACK already tells the sender which frames were decoded, so after a loss it could predict from an acknowledged frame rather than send a keyframe.

**Suggestion:** add an informative note, which needs no wire change. Encoder support for choosing references this way has not been checked.

## Codec results

### Hardware H.264 against the alternatives

Means across the six clips. BD-rate is the bitrate change against H.264 at equal luma PSNR, from 1.5, 3 and 6 Mbit/s targets, so negative is better. Latency is capture to encoded output at a 3 Mbit/s target, and "over budget" counts frames over one frame interval, out of 780.

| Encoder | BD-rate | Median | p99 | Worst | Over budget |
|---|---:|---:|---:|---:|---:|
| VideoToolbox H.264 | baseline | 11.9 ms | 18.2 ms | 63 ms | 8 |
| VideoToolbox HEVC | -18% | 12.6 ms | 19.0 ms | 58 ms | 7 |
| libaom speed 11 | +4% | 14.0 ms | 28.8 ms | 105 ms | 22 |
| libaom speed 7 | -24% | 15.1 ms | 40.3 ms | 232 ms | 49 |
| SVT-AV1 preset 8 | -10% | 17.7 ms | 38.7 ms | 153 ms | 79 |

- libaom speed 11's +4% is pulled up by one clip (+90%). Without it, the mean is -13%.
- Hardware encoders in low-latency mode drop frames at low bitrates: H.264 dropped frames on 5 of 6 clips at 1.5 Mbit/s, up to 64 of 130. At 3 Mbit/s, only one clip dropped any (14 frames). PSNR above holds the latest decoded frame, as a viewer would see it.
- Hardware's worst frame reached about 15 dB at 3 Mbit/s, against 24-35 dB for AV1. These frames haven't been traced.
- PSNR doesn't capture how text looks, so these need a visual comparison too.

**Reading:** the H.264 baseline holds up well, and hardware HEVC is the cheapest gain. Software AV1 saves bitrate at some settings, but with larger latency spikes. It's most useful for low-bitrate links and exact-match refinement.

### Per-clip results

BD-rate against H.264, so negative is better:

| Clip | HEVC | libaom 11 | libaom 7 | SVT-AV1 8 |
|---|---:|---:|---:|---:|
| Debugging | -13.0% | -11.3% | -39.8% | -21.3% |
| Spreadsheet | -21.5% | -9.1% | -16.9% | +30.3% |
| Wikipedia | -18.4% | -15.3% | -23.1% | -17.6% |
| Slides1 | -28.4% | +3.5% | -10.4% | -14.2% |
| SceneComposition | -6.0% | +89.9% | -10.4% | -9.5% |
| MissionControlClip3 | -21.3% | -33.4% | -40.8% | -28.0% |

Actual Mbit/s and mean luma PSNR in dB at each target. Encoders often undershoot their target, so compare the two together:

| Clip | Encoder | 1.5 Mbit/s | 3 Mbit/s | 6 Mbit/s |
|---|---|---:|---:|---:|
| Debugging | H.264 | 1.52 / 40.7 | 2.35 / 49.1 | 2.56 / 51.5 |
| Debugging | HEVC | 1.47 / 43.0 | 2.13 / 49.8 | 2.35 / 51.9 |
| Debugging | libaom 11 | 1.41 / 46.0 | 2.33 / 49.0 | 2.42 / 50.9 |
| Debugging | libaom 7 | 1.47 / 50.9 | 2.46 / 55.3 | 3.81 / 60.3 |
| Debugging | SVT-AV1 8 | 1.27 / 43.6 | 2.20 / 51.0 | 2.58 / 55.0 |
| Spreadsheet | H.264 | 1.50 / 35.0 | 3.04 / 43.8 | 5.02 / 49.1 |
| Spreadsheet | HEVC | 1.51 / 38.1 | 3.02 / 46.4 | 4.31 / 50.3 |
| Spreadsheet | libaom 11 | 1.22 / 34.9 | 1.82 / 38.4 | 2.31 / 39.8 |
| Spreadsheet | libaom 7 | 1.02 / 34.4 | 1.39 / 36.3 | 1.64 / 37.1 |
| Spreadsheet | SVT-AV1 8 | 1.10 / 30.4 | 2.16 / 36.8 | 3.96 / 43.0 |
| Wikipedia | H.264 | 1.39 / 41.3 | 2.27 / 47.4 | 3.08 / 49.5 |
| Wikipedia | HEVC | 1.14 / 42.8 | 2.04 / 47.7 | 2.67 / 49.2 |
| Wikipedia | libaom 11 | 1.13 / 45.1 | 2.31 / 47.9 | 4.03 / 50.1 |
| Wikipedia | libaom 7 | 1.47 / 47.0 | 2.42 / 49.1 | 3.59 / 50.3 |
| Wikipedia | SVT-AV1 8 | 1.13 / 41.4 | 1.85 / 47.1 | 2.95 / 50.4 |
| Slides1 | H.264 | 1.40 / 40.9 | 2.82 / 47.0 | 4.97 / 52.0 |
| Slides1 | HEVC | 1.38 / 43.6 | 2.73 / 49.7 | 4.39 / 54.3 |
| Slides1 | libaom 11 | 1.59 / 41.9 | 2.90 / 47.2 | 5.08 / 51.0 |
| Slides1 | libaom 7 | 1.54 / 43.2 | 2.77 / 47.8 | 4.29 / 51.1 |
| Slides1 | SVT-AV1 8 | 1.31 / 42.0 | 2.65 / 47.6 | 5.16 / 53.8 |
| SceneComposition | H.264 | 1.47 / 49.0 | 1.73 / 51.1 | 1.77 / 51.8 |
| SceneComposition | HEVC | 1.39 / 49.0 | 1.58 / 50.6 | 1.58 / 51.2 |
| SceneComposition | libaom 11 | 1.36 / 47.0 | 2.28 / 48.6 | 3.64 / 50.7 |
| SceneComposition | libaom 7 | 1.30 / 50.1 | 2.17 / 52.9 | 3.79 / 56.8 |
| SceneComposition | SVT-AV1 8 | 1.33 / 50.4 | 2.42 / 53.2 | 2.95 / 54.4 |
| MissionControlClip3 | H.264 | 1.48 / 28.6 | 3.07 / 34.3 | 6.01 / 42.9 |
| MissionControlClip3 | HEVC | 1.50 / 29.9 | 3.05 / 37.5 | 5.79 / 44.9 |
| MissionControlClip3 | libaom 11 | 2.20 / 36.8 | 3.62 / 41.1 | 6.04 / 44.7 |
| MissionControlClip3 | libaom 7 | 2.17 / 37.9 | 3.33 / 41.6 | 5.36 / 44.5 |
| MissionControlClip3 | SVT-AV1 8 | 1.32 / 30.7 | 2.37 / 35.5 | 4.81 / 43.1 |

### Dropping before encoding (§3.3)

The rule works as intended. On a synthetic 4K60 clip, encoders that queued frames fell seconds behind. With dropping and a 10 ms RTT, median latency was 26-47 ms, at 42 fps for libaom speed 11 and 35.5 fps for hardware H.264. The H.264 figure may be limited by the test harness, which allows one frame in the encoder at a time.

### Decoders (WebCodecs and direct)

Time from a frame's scheduled arrival to decoded output, on streams the browser supports. All frames decoded in every run.

| Decoder | Median | p99 | Worst |
|---|---:|---:|---:|
| VideoToolbox, direct | 4-10 ms | 8-14 ms | 17 ms |
| dav1d, direct | 5-13 ms | 11-30 ms | 34 ms |
| Chromium 158, headless | 3-4 ms | 6-11 ms | 44 ms |
| Firefox 159, headless | 3-6 ms | 6-61 ms | 93 ms |
| Safari 18.6, in a background window | 4-8 ms | 7-26 ms | 59 ms |

Browser figures are time to the output callback, so they aren't directly comparable with the direct figures. Browsers report that hardware decoding is available, not that it was used.

Codec support reported by each browser:

| Browser | AV1 Main | AV1 High 4:4:4 | H.264 | HEVC |
|---|---|---|---|---|
| Chromium 158 | Yes | Software only | Yes | Hardware only |
| Firefox 159 | Yes | No | Yes | No |
| Safari 18.6 | Yes | No | Yes | Yes |

VideoToolbox's output matched dav1d frame for frame on all tested AV1 streams, including streams using palette and intra block copy.

## Not yet tested

- Encoder CPU use and power.
- M1, M2, M3 and M5. Software decoders and encoders will be slower on older chips.
- Scaled in-flight limits under congestion.
- Visual comparison of text quality.
