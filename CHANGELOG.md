# Changelog

All notable changes to VROOM will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).

## [Unreleased]

### Added
- VROOM-Terminal spec (v0.2.0-draft) — terminal access protocol with PTY, Kitty keyboard/graphics, CBOR control plane, and capability negotiation
- `control_encoding` negotiation field in both VROOM-Graphical and VROOM-Terminal handshakes — CBOR preferred, JSON accepted
- HELLO bootstrap protocol for encoding negotiation (always JSON, selects encoding for subsequent messages)
- README landing page linking both companion protocol specs

### Changed
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
