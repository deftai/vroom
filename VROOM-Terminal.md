# VROOM-Terminal

**Virtual Remoting Over Open Methods: Terminal Protocol**

*Terminals for humans and agents: a PTY's bytes for people, its screen for agents.*

Version: 0.3.0-draft | Status: Draft | Date: 2026-11-11 | Base: [VROOM-Session](./VROOM-Session.md)

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

VROOM-Terminal runs programs on a host's pseudo-terminals. A person gets the PTY's bytes, untouched, for their own terminal to draw, with Kitty keyboard, Kitty graphics and Sixel passing through. An agent gets the screen the host keeps by emulating the terminal: its text, cursor and changes, and actions on it with guards and act results, so it never has to emulate a terminal itself.

All four types can run over the same xumux connection, with one session and one lease table.

**Profile:** `vroom-terminal/1`. Each terminal is a surface with the ID `pty:<n>`. Its logical unit is the **cell**: `x` is the column and `y` the row, from `0` at the top left of the screen. Cells have a size in pixels, and what's drawn isn't only one character a cell: images, larger text and pixel-precise pointers sit in and across cells ([§5.1](#51-geometry)). A host is also a multiplexer ([§7](#7-multiplexing)): terminals are panes in windows in sessions, and a session is a surface with the ID `tsession:<n>`.

**Implementation note:** each channel's payload format is a **channel codec** on xumux, decoded after demux. See [I/O stack](./docs/io-stack.md).

**Changes from 0.2.0:** this revision aligns Terminal with VROOM-Session. Its own HELLO, authentication and 8-byte TLV header are gone: the session's handshake, xumux `auth` and xumux message types replace them. Terminals are surfaces under the session's leases. Agent access (§5) and multiplexing (§7) are new.

---

## 1. Goals and non-goals

- People get a real terminal: every byte of the program's output, unchanged, so their own terminal's features work.
- Agents get the screen, not a byte stream: they read text, wait for output to settle or for something to appear, and act by keys and text, as they do in the other types.
- One vocabulary with the other types: surfaces, leases, guards, settling and the act result come from VROOM-Session.
- Several terminals on one connection, and a terminal that outlives the connection that opened it.
- Multiplexing as tmux and its successors do it: sessions of windows of panes, one arrangement shared by every viewer, detached and attached at will. A host may also serve the panes of a multiplexer already running ([§7.5](#75-bridged-multiplexers)).

Not goals: file transfer and port forwarding ([§10](#10-later)), and replacing SSH's user and host authentication.

---

## 2. Channels

| Channel | Requested delivery | Purpose |
|---|---|---|
| `vroom.terminal.control` | Reliable, ordered | Everything in [§4](#4-messages) |
| `vroom.terminal.pty.<n>` | Reliable, ordered | The terminal's bytes: output to the client, input to the program ([§6](#6-the-byte-stream)) |

The client opens `vroom.terminal.control`. The host opens `vroom.terminal.pty.<n>` when a client asks for a terminal's bytes (PTY_ATTACH), and a client may attach to several terminals at once.

---

## 3. Handshake

TERM_HELLO answers the client's TERM_HELLO with what the host can do. The client describes the terminal it draws in, if it has one, so that the host can tell programs what they may use.

Client:

```json
{
  "terminal": { "name": "xterm-kitty", "truecolor": true, "unicode_version": "15.1" },
  "features": ["kitty_keyboard", "kitty_graphics", "kitty_text_sizing", "sixel", "synchronized_output", "bracketed_paste", "osc52_clipboard", "hyperlinks", "sgr_pixel_mouse"]
}
```

Host:

```json
{
  "shells": true,
  "exec": true,
  "agent": ["screen", "changes", "act", "images", "render"],
  "scrollback": 10000,
  "features": ["kitty_keyboard", "kitty_graphics", "bracketed_paste", "osc52_clipboard"]
}
```

| Field | Description |
|---|---|
| `terminal` | The client's terminal. The host uses `name` for `TERM` when the client opens a terminal, unless the request names `term` |
| `features` (client) | What the client's terminal can draw or send |
| `shells`, `exec` | Whether the host starts a login shell, or a command, on request |
| `agent` | The agent access offered ([§5](#5-agent-access)): `screen`, `changes`, `act`, `images` (images on the screen and their data), `render` (screenshots of the terminal as drawn) |
| `scrollback` | Lines of scrollback the host keeps per terminal, `0` for none |
| `features` (host) | The features the host's emulator understands, so that agents see them on the screen too |

---

## 4. Messages

On `vroom.terminal.control`. Requests carry `request_id`, and errors are Session ERRORs ([§9](./VROOM-Session.md#9-error-codes)). The host MAY answer requests in any order, so every reply and every error about a request echoes its `request_id`, and a reply about a terminal also names its `pty`. Bytes, such as IMAGE's and SCREENSHOT's `data`, are a byte string in CBOR and a base64 string (RFC 4648, with padding) in JSON.

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x01` | TERM_HELLO | Both | [§3](#3-handshake) |
| `0x02` | PTY_LIST | Both | C→S: nothing. S→C: `ptys`, each `{id, title, rows, cols, command, running, exit?, session, window, source?, stream?}` |
| `0x03` | PTY_OPEN | C→S | `shell?: true` or `command` and `args`, `cwd?`, `env?`, `term?`, `rows`, `cols`, `xpix?`, `ypix?`. Makes a new session with one window holding the terminal. Answered with PTY_STATE (with its `session` and `window`) |
| `0x04` | PTY_STATE | S→C | `pty`, `state` (`opened`, `exited`, `closed`), `exit_code?`, `signal?`. Sent in answer and when a terminal's program exits |
| `0x05` | PTY_CLOSE | C→S | `pty`: hangs the terminal up (SIGHUP) and removes it |
| `0x06` | PTY_ATTACH | C→S | `pty`: the host opens `vroom.terminal.pty.<n>` ([§6](#6-the-byte-stream)) |
| `0x07` | RESIZE | C→S | `pty`, `rows`, `cols`, `xpix?`, `ypix?`: the size of a terminal that is its window's only pane. A pane in a split window is refused with `4605`: its window is resized instead (WINDOW_RESIZE) |
| `0x08` | SIGNAL | C→S | `pty`, `signal` by name: `INT`, `TERM`, `HUP`, `QUIT`, `TSTP`, `CONT`, `WINCH`, `KILL` |
| `0x10` | SCREEN_REQ | C→S | `pty`, `scrollback?` (how many lines of scrollback, counted up from the screen), `scrollback_from?` (lines above the screen to start from, `0` by default, to page further back), `styles?`, `if_newer_than?` |
| `0x11` | SCREEN | S→C | [§5.2](#52-the-screen) |
| `0x12` | CHANGES_REQ | C→S | `pty`, `since` (a `rev`) |
| `0x13` | CHANGES | S→C | `rev`, `rows` (changed row indices), or `too_old: true` |
| `0x14` | ACT | C→S | `pty`, `action`, `args`, `guard?`, `wait?` ([§5.4](#54-actions)) |
| `0x15` | ACT_RESULT | S→C | The [act result](./VROOM-Session.md#7-the-act-result) |
| `0x16` | IMAGE_REQ | C→S | `pty`, `image` (an ID from the screen), `format?` (`png` by default, or `webp`), `max_edge?` |
| `0x17` | IMAGE | S→C | `image`, `width`, `height` (pixels), `format`, `data` |
| `0x18` | SCREENSHOT_REQ | C→S | `pty`, and [Graphical's](./VROOM-Graphical.md#5-agent-access) `format`, `max_edge`, `region` (in cells), `if_newer_than`. Only when the host renders (TERM_HELLO `agent` has `render`) |
| `0x19` | SCREENSHOT | S→C | [Graphical's result](./VROOM-Graphical.md#5-agent-access), with `origin` and `scale` in the terminal's pixels, not cells, and `cell_px` ([§5.1](#51-geometry)) |
| `0x20` | MUX_STATE | S→C | Sessions, windows and layouts ([§7.1](#71-the-model)): in answer to MUX_SUBSCRIBE, then on every change |
| `0x21` | MUX_SUBSCRIBE | C→S | `on`: whether to receive MUX_STATE |
| `0x22` | SESSION_NEW | C→S | `name?`, and a first pane as PTY_OPEN's fields. Answered with MUX_STATE |
| `0x23` | WINDOW_NEW | C→S | `session`, `name?`, and a first pane as PTY_OPEN's fields |
| `0x24` | SPLIT | C→S | `pty`, `direction` (`right` or `down`), `size?` (the new pane's share, `0`–`1`), and the new pane as PTY_OPEN's fields |
| `0x25` | ARRANGE | C→S | `window`, `layout` ([§7.1](#71-the-model)): resizes, moves and swaps panes in one step |
| `0x26` | FOCUS | C→S | `pty`: makes its window the active one and the pane the focused one |
| `0x27` | ZOOM | C→S | `pty`, `on`: the pane fills its window, or not |
| `0x28` | RENAME | C→S | `session`, `window` or `pty`, and `name` |
| `0x29` | KILL | C→S | `session`, `window` or `pty`: closes it and everything in it |
| `0x2A` | WINDOW_RESIZE | C→S | `window`, `rows`, `cols`, `xpix?`, `ypix?`: the whole window, its panes taking their shares of it |

Opening, closing, resizing, signalling, input, actions and arranging ([§7.3](#73-leases)) need the terminal's lease ([Session §4](./VROOM-Session.md#4-the-control-lease)). PTY_OPEN and SESSION_NEW need none, as they make a session of their own and change no one else's, and give the opener no lease. Watching (PTY_LIST, PTY_ATTACH, SCREEN_REQ, CHANGES_REQ) needs none. A terminal whose program has exited stays listed, with its last screen, until PTY_CLOSE.

---

## 5. Agent access

The host feeds each terminal's output to a terminal emulator as it arrives and keeps the result: the screen, its images, the cursor, the modes the program has set, and up to `scrollback` lines above the screen. Agents read and act on that.

### 5.1 Geometry

A terminal is a grid of cells, and also a picture in pixels. Its cell size in pixels comes from the client that sized it (`xpix` and `ypix` of PTY_OPEN, RESIZE or WINDOW_RESIZE, divided by `cols` and `rows`). A host that renders uses the same size, and one that hasn't been told a size MAY assume its own.

- Positions are in cells, and MAY add a pixel offset within the cell, `{x, y, px?, py?}`, as Kitty image placements and pixel-precise pointers have.
- A pixel position is `x * cell_w + px` across and `y * cell_h + py` down, from the top left of the screen.
- Pixels are square and cells usually aren't, so pictures of a terminal (SCREENSHOT) are measured in its pixels: a point in the image is at terminal pixel `origin + image_pixel / scale`, on each axis, however the image was scaled, and in cell `floor(px / cell_w)`, `floor(py / cell_h)`. [Session §3](./VROOM-Session.md#3-surfaces-and-coordinates)'s rule holds with the pixel as the unit.
- Programs learn the cell size as they would from a local terminal: the PTY's window size in pixels, and the terminal's answers to their queries.

### 5.2 The screen

```json
{
  "rev": 52,
  "rows": 24, "cols": 80,
  "cursor": { "x": 12, "y": 3, "visible": true },
  "alternate": false,
  "title": "vim notes.md",
  "lines": ["$ ls", "notes.md  todo.txt", "$ vim notes.md", ""],
  "scrollback": ["…"],
  "styles": [[{ "x": 0, "w": 8, "fg": "#5fd7ff", "bold": true }]],
  "cell_px": { "w": 9, "h": 18 },
  "images": [
    { "image": "i7", "source": "kitty", "x": 2, "y": 5, "px": 0, "py": 4, "cols": 30, "rows": 10, "z": 0, "width": 640, "height": 360 }
  ],
  "sized": [{ "x": 0, "y": 0, "scale": 2, "text": "Title" }]
}
```

| Field | Description |
|---|---|
| `rev` | The terminal's revision ([Session §5](./VROOM-Session.md#5-revisions-and-guards)): it increases with any change to the screen, scrollback, cursor or title |
| `lines` | The screen's rows as text, one string each, without trailing spaces. A wide character takes two cells and appears once |
| `alternate` | The program is using the alternate screen, as full-screen programs do. Scrollback then belongs to the main screen |
| `scrollback` | Present when asked for: the lines above the screen, oldest first |
| `styles` | Present when asked for: per row, runs of cells with non-default styling |
| `cell_px` | The cell size in pixels ([§5.1](#51-geometry)) |
| `images` | Images on the screen: Kitty graphics placements (Unicode placeholders included) and Sixel images, each with an `image` ID for IMAGE_REQ, where it's placed (a cell, a pixel offset in it, and the cells it spans), `z` (negative is under the text), and its size in pixels. A Sixel image is listed as the cells it covers |
| `sized` | Text drawn larger than a cell, as with Kitty's text sizing (OSC 66): its first cell, `scale` (cells a character spans), and the text. `lines` hold the text too, at its first cell |

`if_newer_than` answers `{rev, unchanged: true}` when nothing changed since.

A screen's text says what it says, and its images say where they are and how big. What can't be said in text or in placements, a chart drawn in braille or an image's content, needs the picture: IMAGE_REQ for one image, or SCREENSHOT_REQ for the terminal as drawn, where the host renders.

### 5.3 Settling and waiting

For Terminal, **changed** in [Session §6](./VROOM-Session.md#6-settling) means the program wrote output that changed the screen, its images or the cursor. An `ignore` region is in cells.

WAIT and `wait` on ACT also accept `until: "text"`, which is met when the screen (or, with `scrollback: true`, its scrollback) contains `text`, a plain string, or matches `regex`:

```json
{ "until": "text", "text": "$ ", "timeout_ms": 10000 }
```

A program that exits also ends a wait, with `settled.state: "exited"`.

### 5.4 Actions

| `action` | `args` | Effect |
|---|---|---|
| `text` | `text` | Types the text as keystrokes would. `\n` is Return |
| `key` | `key`: a chord such as `ctrl+c`, `up`, `shift+tab`, `f5`, `alt+b` | One key, encoded as the program asked ([§5.5](#55-keys-and-the-pointer)) |
| `paste` | `text` | Pastes, bracketed if the program turned bracketed paste on |
| `click` | `at` (a position, [§5.1](#51-geometry)), `button?` (`left`, `middle`, `right`), `count?` | A mouse click, if the program asked for mouse input |
| `move` | `at` | The pointer moves there, if the program asked for motion |
| `drag` | `from`, `to`, `button?` | Press, move and release |
| `wheel` | `at`, `dy`, `dx?` | Wheel steps; positive `dy` scrolls down |

Each returns the act result with `rung: "none"` and, when it waited, `changes.regions` as the changed cells: whole rows `{x: 0, y: <row>, w: <cols>, h: 1}`, and the cells of images that changed.

A guard's `scope` is `surface` (any change) or `region`, which names its cells: `{"if_rev": 41, "scope": "region", "region": {"x": 0, "y": 20, "w": 80, "h": 4}}`, refused if any of them changed. `node` is refused with `4605`.

### 5.5 Keys and the pointer

The host encodes an agent's keys by the modes the program has set: application cursor and keypad modes, bracketed paste, and the Kitty keyboard protocol at the level the program pushed. An agent names keys; it never writes escape sequences. A key the encoding can't express is refused with `4605`.

Pointer actions are encoded by the program's mouse modes: which events it asked for (presses, drags, all motion) and how (SGR, or SGR with pixel positions, 1016). With pixel positions the host sends the pixel the agent named, and otherwise the cell. A pointer action on a program that asked for no mouse input is refused with `4605`: there is nothing to send it to.

A person's input on the byte stream is not encoded: their terminal has already encoded it.

---

## 6. The byte stream

On `vroom.terminal.pty.<n>`:

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x01` | OUTPUT | S→C | Bytes the program wrote, as written |
| `0x02` | INPUT | C→S | Bytes for the program, as the client's terminal produced them |
| `0x03` | REDRAW | S→C | Bytes that draw the current screen from blank, images included, sent first on attaching |

- The host MUST pass output through unchanged, in order, including sequences its emulator doesn't understand.
- On attaching, the host sends REDRAW, then OUTPUT from that point. A client that reconnects to a running terminal sees its screen at once: its text, styles and sized text, and its images, transmitted and placed again.
- REDRAW also restores the modes the program set, so that the client's terminal sends what the program expects: application cursor and keypad modes, bracketed paste, the Kitty keyboard flags in force, mouse reporting modes, focus reporting, the alternate screen, the scroll region, the cursor's shape and visibility, and the title. A host keeps the image data placements need for as long as they're on the screen or in scrollback.
- Graphics a client's terminal can't show (a Kitty image to a terminal without Kitty graphics, say, from its TERM_HELLO) are still passed through: the program asked for them, and a host doesn't rewrite output. A program should check what the terminal supports, as it would locally.
- INPUT needs the terminal's lease. Without it the host drops the input and reports `4600` on the session, at most once a second.
- OUTPUT that a slow client can't take is not dropped: xumux flow control holds the program back. A host MAY instead close a channel that stays full, so that one slow viewer cannot stall the program for everyone.

---

## 7. Multiplexing

### 7.1 The model

A host keeps **sessions**. A session has **windows**, in order, one of them active. A window has a **layout**: a tree whose leaves are panes, each a terminal (`pty:<n>`), and whose other nodes are splits. One pane of each window has the focus. All of it lasts until it's killed, whether or not any client is connected.

```json
{
  "rev": 18,
  "sessions": [{
    "id": "tsession:1", "name": "work", "active_window": 2,
    "windows": [{
      "id": 2, "name": "editor", "focus": "pty:4", "zoom": null,
      "rows": 50, "cols": 200,
      "layout": { "split": "right", "children": [
        { "pane": "pty:4", "size": 0.6 },
        { "split": "down", "size": 0.4, "children": [ { "pane": "pty:5", "size": 0.5 }, { "pane": "pty:6", "size": 0.5 } ] }
      ] }
    }]
  }]
}
```

`split: "right"` lays children side by side, and `split: "down"` one above the next. `size` is each child's share of its parent, and the shares of a node's children add up to `1`. The host turns shares into whole cells, with a one-cell border between panes, and each pane's terminal has the resulting size.

### 7.2 Viewers

Every client sees the same sessions, windows and layouts. Attaching to a session is subscribing to MUX_STATE and attaching (PTY_ATTACH) to the panes it shows. Detaching is closing those channels: the session runs on.

A window has one size, set by WINDOW_RESIZE (or RESIZE, for a window of one pane), which needs the session's lease. Taking a lease doesn't resize anything: a client that wants the window to fit its own terminal asks for it. Viewers whose terminals differ draw the window in theirs as best they can, by cropping or padding.

### 7.3 Leases

A lease on a session (`tsession:<n>`) covers its windows and panes, as a lease on a display covers its windows. Arranging (SPLIT, ARRANGE, FOCUS, ZOOM, RENAME, KILL, WINDOW_NEW, WINDOW_RESIZE) needs the session's lease. Input and actions on one pane need that pane's lease, or its session's.

A pane has one holder at a time, by its own lease or its session's:

- A request for a session's lease conflicts with every lease on its panes held by others, and a request for a pane's lease conflicts with its session's lease held by another. Each conflict is settled as [Session §4](./VROOM-Session.md#4-the-control-lease) settles a held surface: a person's request takes the leases from agents, and an agent's waits (`wait_ms`) or fails with `4604`.
- Leases taken from an agent this way end at once, and LEASE_STATE says so. Its actions and input after that fail with `4600`.
- A client holding a session's lease may also hold its panes' leases. It needs nothing more to act on them.

So a person who takes a session stops every agent in it, and an agent can work one pane while a person holds a different session.

### 7.4 Drawing (informative)

A client may draw a session's layout itself, as a tmux client does: one terminal, its panes laid out inside it, each fed by its byte stream. It may instead give each pane to a multiplexer of its own, such as one block, pane or window per VROOM pane, and leave the layout to it. MUX_STATE serves both.

### 7.5 Bridged multiplexers

A host MAY serve the sessions of a multiplexer already running on it, such as tmux, as its own: their panes are `pty:<n>` surfaces and their arrangement is MUX_STATE. Each session and pane says where it comes from, with `source` (`"tmux"`, say), and what the bridge can do:

| Field | Meaning |
|---|---|
| `stream` | The pane's bytes are available, so PTY_ATTACH works. Without it, PTY_ATTACH is refused with `4605` and only the screen is offered |
| `arrange` | The multiplexer's layout can be changed through VROOM |

A bridged pane's screen comes from the host's emulator when the bytes are available, and otherwise from the multiplexer's own capture of its screen. Changes made in the multiplexer itself, by its own clients, appear in MUX_STATE as they happen.

---

## 8. Leases and people

As in every type, a person outranks an agent ([Session §4](./VROOM-Session.md#4-the-control-lease)). A person typing into a terminal holds its lease, and an agent's actions there fail with `4600` until they release it. Watching is open to both: an agent may read the screen while a person types, and a person may watch an agent work.

---

## 9. Security

- A terminal runs programs as the host's user. A host MUST NOT offer terminals unless configured to, and SHOULD limit `exec` and `shells` to what it was configured for.
- ACTIVITY ([Session §8](./VROOM-Session.md#8-audit)) names the terminal and the action, never the text typed or pasted.
- The screen shows what the program printed. Input the program doesn't echo, such as a password prompt's, never appears on the screen; anything printed does, secrets included.
- OSC 52 clipboard writes from a program reach a person's clipboard only through their own terminal's policy. A host MUST NOT apply them to its own clipboard.

---

## 10. Later

- File transfer and port forwarding, as further channels.
- Gateways: `vroom-to-ssh` and `ssh-to-vroom`.
- Hybrid sessions: a terminal and a Graphical display in one view.

---

## 11. Comparison (informative)

| | VROOM-Terminal | OpenSSH | Mosh |
|---|---|---|---|
| Transport | Any xumux binding | TCP | UDP |
| Agents read the screen | Yes (§5) | No | No |
| Kitty keyboard and graphics | Passed through | Passed through | No |
| Reconnect to a running terminal | Yes (REDRAW) | No | Yes |
| Several terminals on one connection | Yes | Yes (channels) | No |
| Sessions, windows and panes | Yes (§7), and bridges to tmux and others | No (tmux alongside) | No |
| With Graphical and Browsable | One connection and session | No | No |

---

## Status

Draft. The agent access in §5 and the multiplexing in §7 are new in 0.3.0 and expected to change as hosts implement them.

## License

MIT
