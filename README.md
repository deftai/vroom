# VROOM

**Virtual Remoting Over Open Methods**

Version: 0.3.0-draft | Status: Draft | Date: 2026-11-08

---

## What is VROOM?

VROOM is a protocol family for remote interactive sessions, for humans and AI agents alike, built on [xumux](https://xumux.org), a transport-agnostic channel multiplexing standard.

VROOM has four types, matched to what is being controlled, on one shared base. They can run over the same xumux connection.

| Protocol | Spec | Description |
|----------|------|-------------|
| **VROOM-Session** | [VROOM-Session.md](./VROOM-Session.md) | The shared base: control lease, revisions and guards, settling, act results |
| **VROOM-Graphical** | [VROOM-Graphical.md](./VROOM-Graphical.md) | Displays and windows as pixels: video, audio, OS-level input, agent screenshots |
| **VROOM-Terminal** | [VROOM-Terminal.md](./VROOM-Terminal.md) | Terminal access — PTY, capability negotiation, Kitty keyboard/graphics, CBOR control plane |
| **VROOM-Structural** | [VROOM-Structural.md](./VROOM-Structural.md) | Native app UI as a tree: one IR; hosts map AX / UIA / AT-SPI |
| **VROOM-Browsable** | [VROOM-Browsable.md](./VROOM-Browsable.md) | Browsers: tabs, page tree, navigation, network, tab video |

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'lineColor': '#404040'}}}%%
graph TB
    T["Terminal"] --> S
    G["Graphical"] --> S
    U["Structural"] --> S
    B["Browsable"] --> S
    B -. video and input formats .-> G
    B -. node schema .-> U
    S["VROOM-Session<br/>lease · revisions · settling · act result"] --> X["xumux<br/>WebSocket · WebRTC · QUIC · TCP · stdio"]
```

Optional [VROOM Services](./VROOM-Services.md) add SFTP v3 and direct/reverse TCP forwarding beside these four types. They are advertised separately and require separate operator authorization; terminal or desktop access does not imply file or network access.

## Which type do I use?

| To control | Use | Because |
|---|---|---|
| A shell | Terminal | Bytes are the real interface |
| A web page | Browsable | It understands pages, and needs no OS permissions when it launches the browser |
| A native app | Structural | Names and roles are more reliable than coordinates |
| Anything else, or to watch and take over | Graphical | Pixels always work |

## Implementation

Implementation is under way in a separate repository, starting with Browsable. The specs here remain the source of truth. [docs/io-stack.md](./docs/io-stack.md) is a non-normative guide to how VROOM channels and their codecs sit on an xumux session.

## Related projects

| Project | Role |
|---------|------|
| [xumux](https://github.com/deftai/xumux) | Mux protocol: 8-byte framing, channels, transport bindings ([xumux.org](https://xumux.org)) |
| [libxumux](https://github.com/deftai/libxumux) | TypeScript implementation of xumux |
| [I/O stack](./docs/io-stack.md) | How VROOM channels and their **channel codecs** sit on an xumux session |

## Status

VROOM is in active development. All documents are drafts and subject to change.

## License

MIT
