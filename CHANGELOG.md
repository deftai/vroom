# Changelog

All notable changes to VROOM will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).

## [Unreleased]

### Added
- VROOM-Session spec: the shared base (control lease, revisions and guards, settling, act result, surfaces and coordinates, error codes)
- VROOM-Browsable spec: browsers as a fourth type (tabs, page tree, navigation, network and console events, storage, dialogs, tab video)
- VROOM-Terminal spec (v0.2.0-draft) — terminal access protocol with PTY, Kitty keyboard/graphics, CBOR control plane, and capability negotiation
- `control_encoding` negotiation field in both VROOM-Graphical and VROOM-Terminal handshakes — CBOR preferred, JSON accepted
- HELLO bootstrap protocol for encoding negotiation (always JSON, selects encoding for subsequent messages)
- README landing page linking both companion protocol specs
- VROOM-Graphical review notes: non-normative open questions

### Changed
- VROOM-Graphical 0.2: targets OS desktops; physical key input plus TEXT; surface-scoped logical coordinates; video channel on every xumux binding with drop-before-queue rules; separate cursor channel; agent screenshots and changed regions. Pointer gains sequence numbers, and scroll moves to the reliable channel. MODE replaced by the lease; COMMAND, NAVIGATE and PASTE removed
- VROOM-Structural 0.2: the browser CDP adapter moves to VROOM-Browsable; PIXEL_NEED and PIXEL_RESULT removed in favour of Graphical screenshots; ACT gains guards and settling
- VROOM-Graphical: the limit on unacknowledged frames scales with the path's round trip, because a fixed limit capped the frame rate on long paths
- VROOM-Graphical: the limit's inputs are defined, with a suggested cap of 16, and only frames really sent count towards it
- VROOM-Graphical: video defaults to a reliable channel and VIDEO_SLICE becomes experimental, because slices did worse than QUIC's own recovery under packet loss
- VROOM-Graphical: FRAME_ACK also covers discarded frames, and senders ignore delay caused by a keyframe's size
- VROOM-Graphical: FRAME_ACK can carry the frame's delivery rate, and senders measure the path and hold a share against other traffic
- VROOM-Graphical: receivers omit `rate_kbps` when a frame can't be timed, and senders that pace video pace at twice the target or faster, so the delivery rate still measures the path
- VROOM-Graphical: STREAM_STATS has a defined baseline body, STREAM_STATE's `path` is absent for media tracks, and the frame limit's units are stated
- VROOM-Graphical: RECEIVER_REPORT's fields are defined, and STREAM_START, STREAM_STATE and the control channel gain `path`, `stats`, VIDEO_CONFIG and STREAM_STATS
- VROOM-Graphical: still video no longer has to reach effectively lossless, because 4:2:0 cannot carry it; refinement frames become optional
- All channels move from `omux/*` to `vroom.<type>.*`, on xumux 0.3
- Renamed existing VROOM spec to VROOM-Graphical (graphical/desktop access protocol)
- VROOM-Graphical control channel updated from JSON-only to CBOR/JSON with encoding negotiation
- All VROOM-Graphical message types now listed as CBOR/JSON payload format
- Design lineage diagram updated to reflect CBOR/JSON

## [0.1.0-draft] - 2026-02-14

### Added
- Initial VROOM protocol spec with full message definitions
- Three interaction modes: View, Interact, Voice
- xumux channel definitions: `omux/control`, `omux/pointer`, `omux/button`
- Binary-packed pointer/button channels for low-latency input
- VROOM_HANDSHAKE, MODE, RESIZE, CLIPBOARD, NAVIGATE, COMMAND, PASTE, CURSOR, STATE, NOTIFICATION, CHAT message types
- Connection flow with WebRTC signaling and xumux handshake
- WebSocket fallback mode
- Coordinate system specification
- Mermaid diagrams with grayscale theme throughout
- Design lineage documenting influences from n.eko, Selkies-GStreamer, JetKVM, and SocketPipe
