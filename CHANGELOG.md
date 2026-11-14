# Changelog

All notable changes to VROOM will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).

## [Unreleased]

### Added
- VROOM-Session spec: the shared base (control lease, revisions and guards, settling, act result, surfaces and coordinates, error codes)
- VROOM-Browsable spec: browsers as a fourth type (tabs, page tree, navigation, network and console events, storage, dialogs, tab video)
- [docs/io-stack.md](./docs/io-stack.md): non-normative guide to running VROOM 0.2 channels on xumux 0.3, with per-channel codecs and presets; README and spec pointers
- VROOM-Terminal spec (v0.2.0-draft) — terminal access protocol with PTY, Kitty keyboard/graphics, CBOR control plane, and capability negotiation
- `control_encoding` negotiation field in both VROOM-Graphical and VROOM-Terminal handshakes — CBOR preferred, JSON accepted
- HELLO bootstrap protocol for encoding negotiation (always JSON, selects encoding for subsequent messages)
- README landing page linking both companion protocol specs
- VROOM-Graphical review notes: non-normative open questions
- VROOM-Graphical: READ_TEXT_REQ/READ_TEXT (`0x07`/`0x08`), reading a surface's text with line boxes; the `open` and `clipboard` actions; CLIPBOARD_SYNC (`0x23`), how a client opts in to the clipboard, answered with the state in force; capability names `read_text`, `open` and `clipboard`

### Changed
- VROOM-Session 0.2.0-draft: a broader control lease (§4). One lease can cover several surfaces, granted, renewed and released together, so clients asking for the same surfaces in different orders can't deadlock. Leases are per kind (`input`, `act`, `arrange`, `size`), and leases of different kinds don't conflict. A `shared` mode lets several holders of one kind coexist, for pairing. Grants let other clients join a lease with its rank, narrowed to some surfaces and kinds, through a secret. New messages LEASE (`0x14`), LEASE_GRANT (`0x15`), GRANT (`0x16`), LEASE_JOIN (`0x17`) and GRANT_REVOKE (`0x18`); `lease_id`, `kinds` and `mode` on LEASE_REQUEST; leases by ID in LEASE_STATE; `client_id` and `leases` in SESSION_WELCOME; `detail.surface` and `detail.kind` on `4600`; the lease and any joiner in ACTIVITY. A client using none of it gets the earlier rules, and a client of an earlier draft using none of it is answered with LEASE_STATE, as before. LEASE_STATE entries keep `surface`, `client` and `kind` for earlier clients, and a lease becoming suspended or ceasing to be is a change. A grant may narrow to a surface the lease covers, such as one pane of its session. GRANT_REVOKE takes `request_id` and is answered with LEASE_STATE. A joiner may renew by `lease_id` but not change `ttl_ms`. An unknown `lease_id` or `grant_id`, and the owner joining its own lease, get `4603` like any other refusal
- VROOM-Terminal: §7.3 maps each message to a lease kind (arranging to `arrange`, RESIZE and WINDOW_RESIZE to `size`, INPUT, SIGNAL and ACT to `input`; RESIZE on the terminal or its session), so a person rearranging a session no longer stops agents typing in it, and gives the example of a session handed to another multiplexer. Graphical, Browsable and Structural say which kind their input and actions need
- VROOM-Terminal: programs that own panes (§7.6): a program finds out whether its host offers panes (environment locally, XTGETTCAP `vroom-mux` with DA1 in band), opens and arranges panes in its window, hears focus, resizes, layout changes and closes, and reads and writes each pane; locally as PTY descriptors and a Unix socket, in band as frames on one stream, started only with a one-time token from the host. The program's panes fill the space its own pane had and carry an `owner` in MUX_STATE; acting on a display or remote surface needs that surface's lease. Panes of other types (§7.7): browser tabs, displays and other hosts' surfaces in a window's layout
- VROOM-Terminal 0.3.0-draft: aligned with VROOM-Session (its own HELLO, authentication and TLV header replaced by the session's handshake, xumux `auth` and one-byte message types; terminals are `pty:<n>` surfaces under the session's leases), and agent access added: the screen as text, changes, waiting for text, and `text`, `key` and `paste` actions with guards and act results; and multiplexing: sessions of windows of panes in a layout tree shared by every viewer, detach and attach, session leases covering their panes, and bridged multiplexers (tmux and others) whose panes are served as the host's own; terminals as pictures as well as cells: cell size in pixels and pixel offsets, images (Kitty graphics, Sixel) and sized text (OSC 66) on the agent's screen, IMAGE_REQ and a rendered SCREENSHOT_REQ, mouse actions by the program's mouse modes (pixel-precise with 1016), and images redrawn on attaching
- VROOM-Session: Terminal returns the act result and has a settling definition, like the other types
- VROOM-Graphical review notes: the in-flight limit over an internet path with a bottleneck, and OCR text legibility per encoder
- VROOM-Graphical: in the `text` action, newlines press Return and tabs press Tab; agent-channel answers and errors echo `request_id`
- VROOM-Graphical review notes: encoder CPU per frame, and the in-flight limit over an internet path
- VROOM-Graphical: frames bigger than one interval are sent whole, with captures dropped meanwhile and their delay not used to lower the bitrate
- VROOM-Session: xumux limits are sized for the largest frame, not only keyframes, with suggested values from measurement
- VROOM-Graphical 0.2: targets OS desktops; physical key input plus TEXT; surface-scoped logical coordinates; video channel on every xumux binding with drop-before-queue rules; separate cursor channel; agent screenshots and changed regions. Pointer gains sequence numbers, and scroll moves to the reliable channel. MODE replaced by the lease; COMMAND, NAVIGATE and PASTE removed
- VROOM-Structural 0.2: the browser CDP adapter moves to VROOM-Browsable; PIXEL_NEED and PIXEL_RESULT removed in favour of Graphical screenshots; ACT gains guards and settling
- VROOM-Graphical: the limit on unacknowledged frames scales with the path's round trip, because a fixed limit capped the frame rate on long paths
- VROOM-Graphical: the limit's inputs are defined, with a suggested cap of 16, and only frames really sent count towards it
- VROOM-Graphical: video defaults to a reliable channel and VIDEO_SLICE becomes experimental, because slices did worse than QUIC's own recovery under packet loss
- VROOM-Graphical: FRAME_ACK also covers discarded frames, and senders ignore delay caused by a keyframe's size
- VROOM-Graphical: draft profiles, `interactive` and `fidelity`, for people and agents, which set input, video and audio policy together; audio states whether it conceals losses
- VROOM-Graphical: lossless screenshot formats are defined, with guidance on lossless WebP
- VROOM-Graphical: PROFILE chooses a profile without starting video, the audio channel's delivery follows the profile, and screenshot patches are exact only when lossless
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
