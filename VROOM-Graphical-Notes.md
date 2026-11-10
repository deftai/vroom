# VROOM-Graphical: Spec Review Notes

**Not normative.** These are open questions about [VROOM-Graphical 0.2](./VROOM-Graphical.md), with test results, for discussion. Nothing here changes the protocol.

Date: 2026-11-10

## Evidence

All tests ran on a single Apple M4 (10 cores) on macOS 15.7. These are short runs on one machine, so treat the numbers as rough. M1, encoder power, and congestion on an internet path with a real bottleneck have not been tested.

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

**Update:** §3.3 now uses `ceil(lowest_rtt × target_fps) + 1`, kept between 2 and a cap (16 is RECOMMENDED), counting only frames really sent.

**Under congestion.** A simulation tested this rule against a fixed 2 and other rules. It used vroomd's sender and delay-based adapter, every frame through the M4's hardware H.264 encoder, and a modelled bottleneck with no transport congestion control, in virtual time. With the adapter running, at 100 ms RTT, on the 60 fps clip:

| Rule | Normal link | Link cut from 20 to 4 Mbit/s for 30 s: fps | Lag p50 / p95 / worst |
|---|---:|---:|---:|
| Fixed limit of 2 | 16.6 fps | 13.3 | 140 / 297 / 571 ms |
| §3.3, lowest RTT, cap 16 | 59.1 fps | 52.9 | 71 / 290 / 719 ms |
| Current RTT instead of lowest | 59.1 fps | 52.7 | 71 / 539 / 799 ms |
| No limit | 59.1 fps | 54.9 | 71 / 1081 / 1310 ms |

Lag is the time from a screen change being captured until a frame at least that new is decoded.

- **The scaled limit holds up, with the adapter.** It keeps about four times the frame rate of a fixed 2, with similar p95 lag. Using the current round trip nearly doubles p95 lag.
- **It depends on adapting to delay.** With the adapter off, p95 lag in the same cut was 2214 ms for the scaled limit and 731 ms for a fixed 2. The limit alone bounds the queue only to its frame count times the frame size.
- **The limit is still needed.** On a link held at 3 Mbit/s, p95 lag was 851 ms with it and 7536 ms without.
- **Refreshing the lowest round trip lets a queue raise the limit.** vroomd resets it after 10 s, and during a long cut with the adapter off the limit rose from 8 to 16, taking worst lag from 1975 to 2811 ms. Never refreshing it is worse: after a real 80 ms rise in path length, frame rate fell from 58 to 33 fps.
- **A keyframe after every encoder drop made congestion worse.** Each drop forced a keyframe, which was too big at the low bitrate, so the encoder dropped again. On a link held at 3 Mbit/s, median lag was 5037 ms with it and 73 ms without. vroomd has since stopped doing this. H.264 streams with 5-64 of 130 frames dropped by the encoder decoded with no errors and no keyframe, and reached 35-50 dB by their end.

The network is modelled, and the adapter is vroomd's from before it changed its start bitrate and delay handling, so these show how the rules interact rather than real-network numbers. Limiting bytes instead was tried with one formula, and cost frame rate on a normal link (51 fps against 59).

**On a real host.** The current vroomd host and a headless Chromium viewer ran over WebTransport, through vroomd's benchmark proxy, which adds 50 ms of delay each way and a bottleneck towards the viewer. The source was vroomd's `synthetic-noisy` at about 3.5 Mbit/s, and each case ran 3 times for 20 s. The scaled limit here is 7 frames.

| Link | Fixed limit of 2: lag p50 / p95 / p99, fps | §3.3 scaled limit: lag p50 / p95 / p99, fps |
|---|---:|---:|
| 20 Mbit/s | 108 / 125 / 130 ms, 14 fps | 75 / 81 / 88 ms, 56 fps |
| 4 Mbit/s | 252 / 272 / 285 ms, 7 fps | 89 / 138 / 270 ms, 52 fps |
| 4 Mbit/s with a competing download | 259 / 319 / 324 ms, 7 fps | 144 / 204 / 302 ms, 47 fps |
| 3 Mbit/s with a 1 s buffer | 350 / 375 / 386 ms, 5 fps | 102 / 199 / 422 ms, 38 fps |

Lag here runs from the page drawing to the viewer's canvas. The scaled limit had lower median lag and 4-7 times the frame rate on every link. The fixed limit lags too, since each frame waits a round trip to be acknowledged. The one worse result is the scaled limit's 99th percentile on the 1 s buffer. Each run had one keyframe, even with up to 323 frames dropped by the encoder, which matches the simulation's finding on forced keyframes. This ran on one M4 over localhost with synthetic content, alongside another host, and nothing here tests having no limit. The scripts and results are in [`deftai/vroom-lab`](https://github.com/deftai/vroom-lab) (`80_realnet.sh`).

**Over an internet path.** The same comparison ran with host and viewer on this machine and the traffic going out to a relay on a MacBook and back over Tailscale, about 40 ms round trip, over WebSocket. WebTransport couldn't use the relay, because Tailscale's MTU of 1280 is smaller than Chromium's QUIC packets. There was no shaping, so whatever the path itself did applied. Each run started on a mostly idle machine, 4 runs each:

| | Fixed limit of 2 | §3.3 scaled limit (4 frames here) |
|---|---:|---:|
| Lag p50 / p95 / p99 | 50 / 138 / 270.5 ms | 39 / 104 / 247.5 ms |
| Frame rate | 25.5 fps | 39.2 fps |
| Stalls over 100 ms per run | 27 | 22 |

The scaled limit was better on every measure on this path. On localhost it was too, except the 99th percentile with a 1 s buffer. This path had no bottleneck tighter than what the bitrate adapter settled on, so it doesn't test a congested internet path.


### 2. Making text crisp in the viewer (§1 goal 4, §3.3)

**Update:** measurement agreed, so §1 goal 4 and §3.3 no longer ask for effectively lossless video. Making text fully crisp in the viewer is still open.

Results that bear on what's left:

- **The 4:2:0 limit is large for colored text.** With no compression at all, a 4:2:0 round trip left the red channel of pink text on a dark background at 28.5 dB against the RGB original, compared with 57.7 dB for 4:4:4.
- **Refinement improves the 4:2:0 picture, but not past the 4:2:0 limit.** Scored against the 4:2:0 source, a bitrate ladder over 11 frames took hardware encoders from 37-51 dB to 61-70 dB luma. vroomd's test of real pages scored against the RGB original, where the 4:2:0 limit is 30.9-33.2 dB: H.264 already reached it at about 4-5 Mbit/s, and refinement added little, as §3.3 says. Below about 4 Mbit/s it lifted PSNR 1.4-2.5 dB.
- **How refinement is driven matters.** On VideoToolbox, raising `Quality` or forcing keyframes stalled at 30-47 dB, and capping QP stalled at 47-57 dB. Only raising the bitrate reached 60 dB and above.
- **Only AV1 reached an exact match** with the 4:2:0 source: 72-693 KB in total, against 61-70 dB at similar sizes for H.264 and HEVC.
- **4:4:4 has no hardware decode path found for AV1.** VideoToolbox rejects AV1 High profile (`-12910`). In WebCodecs, Chromium offers it in software only, and Firefox and Safari don't offer it.

**Suggestion:** say whether text in the viewer needs a separate 4:4:4 path, or whether the lossless screenshot in §5 is enough.

### 3. Pacing frames larger than one interval's budget (§3.3)

Senders SHOULD spread each frame's bytes over the frame interval, but some frames can't fit. The largest frame per encoder across the six clips at a 3 Mbit/s target was 76-229 KB, which takes about 0.2-0.6 s to send at that rate, against a 33 ms interval at 30 fps.

**Suggestion:** say how pacing and frame dropping behave when a frame exceeds one interval's budget. Intra refresh (already a SHOULD) reduces how often this happens, but the first frame still needs a rule.

**Update:** pacing is now optional (MAY), and a sender that paces SHOULD pace at twice its bitrate target or faster, so that `rate_kbps` can still measure the path. At twice a 3 Mbit/s target, the largest frames above still take about 0.1-0.3 s. §3.3 now says such a frame is sent whole, captures keep being dropped before encoding while it goes, and its delay isn't used to lower the bitrate.

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

**Update:** measured again at 12 Mbit/s, vroomd's default maximum, with hardware H.264 and HEVC: the largest frames were 190-273 KiB at 1080p (always the first keyframe) and 813-820 KiB on the synthetic 4K clip, where the largest was a delta frame about three times the first keyframe. [Session §1](./VROOM-Session.md) now suggests twice the largest frame at the host's highest bitrate, about 512 KiB at 1080p and 2 MiB at 4K, and sizes for the largest frame rather than only keyframes. Other encoders and chips aren't measured.

### 5. Recovery from FRAME_ACK (§3.3, §3.4)

"Avoid keyframe spikes" mentions reference invalidation. FRAME_ACK tells the sender which frames the receiver has finished with, so after a loss it could predict from an acknowledged frame rather than send a keyframe. But a receiver also acknowledges frames it discarded (§3.4), so an acknowledgement alone doesn't show that a frame can serve as a reference. This would need FRAME_ACK to say whether the frame was decoded.

**Suggestion:** add a `decoded` flag to FRAME_ACK if this is pursued, then an informative note. Encoder support for choosing references this way has not been checked.

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

### Encoder CPU

CPU time per source frame, on the capture clock: the encoder process plus VideoToolbox's encoder service, including the harness reading and converting frames. Medians, with each encode started on a mostly idle machine:

| Encoder | 1080p (6 clips, 12 runs) | 4K60 synthetic (2 runs) |
|---|---:|---:|
| Hardware HEVC | 2.0 ms | 4.3 ms |
| Hardware H.264 | 2.2 ms | 5.3 ms |
| libaom speed 11 | 18 ms | 58 ms |
| libaom speed 7 | 25 ms | 90 ms |
| SVT-AV1 preset 8 | 38 ms | 178 ms |

Software AV1 costs 8-40 times the CPU of hardware encoding. At 60 fps a whole core is 16.7 ms a frame, so at 4K60 software AV1 needs 3.5-11 cores' worth and couldn't keep up on this M4. Power wasn't measured.

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

- Encoder power. CPU time is under Codec results.
- M1, M2, M3 and M5. Software decoders and encoders will be slower on older chips.
- Scaled in-flight limits on an internet path with a real bottleneck. Issue 1 has a simulation, a shaped localhost path and an unshaped internet path.
- Visual comparison of text quality.
