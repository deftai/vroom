# VROOM-Terminal

**Virtual Remoting Over Open Methods: Terminal Protocol**

*Terminals for humans and agents: a PTY's bytes for people, its screen for agents.*

Version: 0.3.0-draft | Status: Draft | Date: 2026-11-11 | Base: [VROOM-Session](./VROOM-Session.md)

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

VROOM-Terminal runs programs on a host's pseudo-terminals. A person gets the PTY's bytes, untouched, for their own terminal to draw, with Kitty keyboard, Kitty graphics and Sixel passing through. An agent gets the screen the host keeps by emulating the terminal: its text, cursor and changes, and actions on it with guards and act results, so it never has to emulate a terminal itself.

All four types can run over the same xumux connection, with one session and one lease table.

**Profile:** `vroom-terminal/1`. Each terminal is a surface with the ID `pty:<n>`. Its logical unit is the **cell**: `x` is the column and `y` the row, from `0` at the top left of the screen. Cells have a size in pixels, and what's drawn isn't only one character a cell: images, larger text and pixel-precise pointers sit in and across cells ([§5.1](#51-geometry)). A host is also a multiplexer ([§7](#7-multiplexing)): terminals are panes in windows in sessions, and a session is a surface with the ID `tsession:<n>`.

**Implementation note:** each channel's payload format is a **channel codec** on xumux, decoded after demux. See [I/O stack](./docs/io-stack.md).

**Changes from 0.2.0:** this revision aligns Terminal with VROOM-Session. Its own HELLO, authentication and 8-byte TLV header are gone: the session's handshake, xumux `auth` and xumux message types replace them. Terminals are surfaces under the session's leases. Agent access (§5), multiplexing (§7), and programs that own panes of any type (§7.6, §7.7) are new.

---

## 1. Goals and non-goals

- People get a real terminal: every byte of the program's output, unchanged, so their own terminal's features work.
- Agents get the screen, not a byte stream: they read text, wait for output to settle or for something to appear, and act by keys and text, as they do in the other types.
- One vocabulary with the other types: surfaces, leases, guards, settling and the act result come from VROOM-Session.
- Several terminals on one connection, and a terminal that outlives the connection that opened it.
- Multiplexing as tmux and its successors do it: sessions of windows of panes, one arrangement shared by every viewer, detached and attached at will. A host may also serve the panes of a multiplexer already running ([§7.5](#75-bridged-multiplexers)).
- Programs that lay out panes of their own: a program can find out whether its host offers panes, open and arrange them, hear which is focused, and read and write each one, locally or over SSH ([§7.6](#76-programs-that-own-panes)). Panes may be browser tabs, displays and other hosts' surfaces as well as terminals ([§7.7](#77-panes-of-other-types)).

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

- The host MUST pass output through unchanged, in order, including sequences its emulator doesn't understand. The one exception is the pane interface's own query and start request, which are addressed to the host ([§7.6.1](#761-finding-the-host)).
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

`split: "right"` lays children side by side, and `split: "down"` one above the next. `size` is each child's share of its parent, and the shares of a node's children add up to `1`. The host turns shares into whole cells, with a one-cell border between panes, and each pane's terminal has the resulting size. A leaf may be a surface of another type, such as a browser tab ([§7.7](#77-panes-of-other-types)).

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

### 7.6 Programs that own panes

A program running in a pane can open panes of its own in its window, lay them out, hear which pane is focused and when panes change size or close, and read and write each one. This lets an editor, a debugger or an agent's interface put its views side by side without drawing a layout itself. The program uses the **pane interface**, not a VROOM connection. It needs no credentials, because the host started it.

#### 7.6.1 Finding the host

A host offers the pane interface to the programs it runs in one of two ways. A program checks for them in this order:

1. **Local.** The host sets `VROOM_PANES` and `VROOM_MUX_FD` in the environment of the program it starts.
   - `VROOM_PANES` lists the program's panes and the file descriptors holding them, as `name:fd` pairs separated by commas, such as `main:0,log:3`. `main` is the pane the program runs in. A terminal pane is the slave end of its own PTY.
   - `VROOM_MUX_FD` is a Unix datagram socket. It carries one message ([§7.6.2](#762-messages)) per datagram, and file descriptors as `SCM_RIGHTS`.
   - The program SHOULD set close-on-exec (`FD_CLOEXEC`) on `VROOM_MUX_FD` and on every pane descriptor, including each one it receives later. Then the only descriptor a child gets is one handed to it on purpose ([§7.6.4](#764-children)).
2. **In band.** The program's terminal is a host's pane further away, for example over SSH, so neither the environment nor descriptors reach it. The program:
   1. puts its terminal in raw mode;
   2. writes XTGETTCAP for the name `vroom-mux` (`DCS + q 76726f6f6d2d6d7578 ST`), followed by DA1 (`CSI c`).

   A host that offers the interface answers `DCS 1 + r 76726f6f6d2d6d7578=<token> ST` before its DA1 reply. `<token>` is at least 16 random bytes in hex, new for each query. A terminal that doesn't offer it answers only DA1, so the program knows after one round trip and needs no timeout.

   To start, the program writes `DCS vroom-mux;start;<token> ST`, with the token from the reply. From the next byte on, both directions carry frames ([§7.6.5](#765-frames)) until framing ends.

Without either, the program has its one terminal, as anywhere else.

**The token keeps output from starting the interface.** Anything printed to a pane can contain the start request: a file, a log line, a remote login message. Only the program reading the pane's input has seen the token. So a host:
- ignores a start request with no token, a wrong token, or a used one;
- ignores a token that is more than a minute old.

**Where the query is answered.** The host answers the query and the start request only on the pane they arrive on. It answers the query before anything else on that pane answers anything, so its reply comes before any DA1 reply, from its own emulator or from a viewer's terminal.

**Only these two sequences are removed.** The host doesn't pass the query or the start request to viewers, nor draw them. Every other DCS string, such as a Sixel image or tmux passthrough, is output like any other ([§6](#6-the-byte-stream)).

Raw mode matters because, in cooked mode, the reply would wait for a newline. Setting raw mode can discard input typed but not yet read.

The reply lands in the input of whatever reads the pane, as any terminal query's reply does. A program that never asked for it, such as a shell while `cat` prints the query, simply reads it as input. That's harmless: the token is useless without a start request from the same reader, which is why the token is needed at all.

#### 7.6.2 Messages

Every message is a JSON object whose `type` names it. The Session's [error codes](./VROOM-Session.md#9-error-codes) apply.
- A request carries `request_id`, and its answer echoes it.
- A request with no answer of its own is answered with OK.
- A refusal is ERROR, with `error: {code, message}`.
- The program names its panes. Names are unique among its panes, and the host also gives each pane a surface ID.

```json
{"type": "OPEN", "request_id": 7, "name": "log", "kind": "terminal", "next_to": "main", "direction": "right", "size": 0.4}
{"type": "OPENED", "request_id": 7, "name": "log", "kind": "terminal", "surface": "pty:9", "rows": 39, "cols": 47, "cell_px": [9, 18], "channel": 2}
{"type": "FOCUS", "request_id": 8, "name": "nope"}
{"type": "ERROR", "request_id": 8, "error": {"code": 4602, "message": "no pane nope"}}
{"type": "RESIZE", "name": "main", "rows": 39, "cols": 72, "cell_px": [9, 18]}
```

Program to host:

| Type | Body |
|---|---|
| HELLO | Nothing. Answered with HELLO: `version`, `kinds` (the pane kinds offered, [§7.7](#77-panes-of-other-types)), `panes`, `layout`, `focus`. `panes` is an object keyed by pane name; each value has `surface`, `kind`, `rows`, `cols`, `cell_px` and, in band, `channel`. In band, the host sends HELLO unasked, as the first frame, with no `request_id` |
| OPEN | `name`, `kind` (`terminal` unless given), `next_to?` (one of the program's panes; the focused one unless given), `direction` (`right` or `down`), `size?` (the new pane's share, `0`–`1`), and the kind's own fields ([§7.7](#77-panes-of-other-types)). Answered with OPENED: `name`, and the fields of one of HELLO's `panes`. Locally, a terminal pane's descriptor comes with OPENED |
| ARRANGE | `layout`: [§7.1](#71-the-model)'s tree, with pane names as leaves, naming each of the program's panes once ([§7.6.3](#763-where-the-panes-go)). Answered with LAYOUT |
| FOCUS | `name` |
| CLOSE | `name`. `main` can't be closed: it's the program's own terminal (`4603`) |
| RENAME | `name`, `title` |
| ACT | `name`, and an action of the pane's kind ([§7.7](#77-panes-of-other-types)). Answered as that kind answers it |
| END | In band only: the program stops framing. The host closes the program's other panes, answers with OK, and from then on the stream is a plain terminal |

Host to program:

| Type | Body |
|---|---|
| LAYOUT | `layout`, with each leaf's `rows` and `cols`. Sent after every change to the arrangement, whoever made it |
| RESIZE | `name`, `rows`, `cols`, `cell_px` |
| FOCUSED | `name`, `focused` |
| CLOSED | `name`, `by`: `user` (a person or a client closed it), `process` (its PTY's last process went away while the program kept running), or `end` (framing ended) |
| EVENT | `name`, and an event of the pane's kind |
| OK, ERROR | Answers, as above |

#### 7.6.3 Where the panes go

- **The program's panes fill the space `main` had.** In its window, they form one subtree of the layout. OPEN and ARRANGE change only that subtree, so they never move panes the program doesn't own. When people or clients resize that space, the subtree is resized as a whole.
- **MUX_STATE shows the subtree as an ordinary split.** Each of the program's panes has `owner`: the surface ID of `main`. A client can tell from it that closing the pane changes a program's layout, and a person doesn't take a program's panes for strays.
- **Geometry is the host's.** The host places ARRANGE's layout in whole cells, and LAYOUT and RESIZE report what it gave.
- **RESIZE is the only resize notice for every pane except `main`.** A process has one controlling terminal, so SIGWINCH comes only for `main`. Locally, the new size is also on each pane's PTY (`TIOCGWINSZ`).
- **Arranging needs no lease.** The program needs none to open, arrange, focus, rename or close its own panes, or to write to its terminal panes, which is output. It's the program running there, not a client. People and clients with the session's lease may still rearrange, resize, focus and close those panes ([§7.3](#73-leases)). The program hears of it through LAYOUT, FOCUSED and CLOSED, and SHOULD NOT undo it.
- **The program's actions give way to people.** A program driving a pane that isn't a terminal acts as an agent does ([§8](#8-leases-and-people)).
  - **Its own panes** (a `browser` tab it opened). Its ACT on one fails with `4600` while a person holds that pane's lease or its session's. So it works the tab freely until a person takes over.
  - **Surfaces it didn't make** (a `display` or a `remote` pane, [§7.7](#77-panes-of-other-types)). Showing one needs no lease. For the program's ACT on one, the host takes the surface's lease for the program, as it does for an agent:
    - the action goes ahead if the lease is free or already the program's;
    - it fails with `4600` if another holds the lease;
    - a person's request takes the lease from the program, as from any agent;
    - the lease ends when the pane closes.
- **What people and clients type into a pane is theirs.** Typing into a pane that isn't a terminal goes to the pane, not to the program ([§7.7](#77-panes-of-other-types)), under the typist's own lease, as on the surface itself. A lease taken for the program never refuses a person's typing.
- **Input to a program's pane is input like any other.** What a person or an agent types into one of the program's terminal panes reaches the program on that pane, under that pane's lease ([§6](#6-the-byte-stream)).

#### 7.6.4 Children

- **A program MAY hand a terminal pane to a child, as the child's terminal.**
  - Locally, the child takes the pane's PTY as its controlling terminal.
  - In band, the program gives the child a PTY of its own, and copies bytes and sizes between that PTY and the pane's channel.

  Either way, the child gets SIGWINCH and ^C as in any terminal.
- **Children don't inherit the pane interface.** The program removes `VROOM_PANES` and `VROOM_MUX_FD` from their environment. With close-on-exec set ([§7.6.1](#761-finding-the-host)), no descriptor reaches them except the one handed over.
- **The program's panes close when it exits.** A terminal pane stays open while any process still holds its PTY.
- **Extra panes need ISIG off.** Locally, a terminal pane that isn't `main` is no process's controlling terminal. In cooked mode, ^C typed there is swallowed, the line typed so far is dropped, and no process gets a signal. A program SHOULD turn off `ISIG` on such panes, or use raw mode. Then ^C arrives as a byte.

#### 7.6.5 Frames

```
[length: u32, big-endian][channel: u16, big-endian][payload: length bytes]
```

- A payload is at most 1 MiB (1,048,576 bytes). A sender splits larger output into several frames.
- Channel `0` carries [§7.6.2](#762-messages)'s messages, one UTF-8 JSON object per frame.
- Channel `1` is `main`. Every other terminal pane has the channel given in HELLO or OPENED.

**The host never shows a frame.** Once framing starts, the pane's raw stream is frames, never output.
- The host's emulator and viewers get channel 1's payload as `main`'s output ([§6](#6-the-byte-stream)'s OUTPUT), and each other channel's payload as its own pane's.
- Input from people and agents goes to the program as frames on its pane's channel.
- Neither the emulator nor a viewer ever sees a frame header.

**Framing ends** when the program sends END, or as soon as the stream stops making sense. That is any of:
- a frame longer than the limit;
- a frame on a channel the host didn't give;
- a channel-0 payload that isn't a JSON object.

When the stream stops making sense, the host closes the program's other panes and treats the bytes from the bad header on as `main`'s plain output again. That's what happens when a program dies without END and the shell that started it prints a prompt: printable text read as a length is far over the limit. The terminal is then still in raw mode, as after any program that dies without restoring it. A host SHOULD also let a person end framing on a pane from their client, in the same way.

**Frames have no line discipline.** Input arrives exactly as typed, with no echo and no signals. A program in band echoes and edits lines itself, or gives the pane to a child ([§7.6.4](#764-children)). It keeps its terminal in raw mode while framing, and restores it when framing ends.

### 7.7 Panes of other types

A window's layout may hold surfaces of any VROOM type, not only terminals. A leaf's `pane` is a surface ID, and its prefix says its type. The prefix is always that of a surface type this host serves itself, any of [Session §3](./VROOM-Session.md#3-surfaces-and-coordinates)'s (`pty`, `tab`, `display`, `window`, `app`, ...). A `remote` pane is served under a local ID of the other surface's own type, so there is no `remote:` prefix.

| OPEN `kind` | Surface | OPEN fields | ACT and EVENT |
|---|---|---|---|
| `terminal` | `pty:<n>` | | Bytes ([§6](#6-the-byte-stream)); ACT is refused with `4605` |
| `browser` | `tab:<n>`, a new tab | `url?` | [VROOM-Browsable](./VROOM-Browsable.md)'s actions and navigation, by their names there; its page events as EVENT |
| `display` | `display:<id>` or `window:<id>`, one of this host's | `surface` | [VROOM-Graphical](./VROOM-Graphical.md)'s actions, under the surface's lease ([§7.6.3](#763-where-the-panes-go)) |
| `remote` | A local ID of the other surface's type, such as `tab:<n>` | `host`, `surface` | The other surface's type's actions and events, under its lease |

- **A `remote` pane is served as one of the host's own surfaces.** The host reaches the other host as a client and serves the surface with the same type and a local ID that doesn't collide with its other surfaces: a remote tab becomes a local `tab:<n>`.
  - Video, input and requests use the local ID on that type's own channels, exactly as for a local surface. The host translates them to the other host's.
  - MUX_STATE gives the pane's `source` as `{"host": ..., "surface": ...}`.
  - The host connects with credentials it was configured with, never ones from the program.
- **Viewers draw panes by type.** A client draws a terminal pane from its byte stream, and other panes from their type's video, which Graphical and Browsable both have. A client drawing in a terminal can show video panes with Kitty graphics. A client that can't draw a type shows the pane's title in its place.
- **Agents work each pane by its type.** For example, a tab through Browsable's page tree, and a display through Graphical's screenshots. Leases work as in [§7.3](#73-leases), with a pane's lease covering whatever surface it is.
- **Keys typed into a pane that isn't a terminal go to that pane**, not to the program.
- **A kind the host doesn't offer is refused.** It is missing from HELLO's `kinds`, and OPEN with it fails with `4605`.

---

## 8. Leases and people

As in every type, a person outranks an agent ([Session §4](./VROOM-Session.md#4-the-control-lease)). A person typing into a terminal holds its lease, and an agent's actions there fail with `4600` until they release it. Watching is open to both: an agent may read the screen while a person types, and a person may watch an agent work.

---

## 9. Security

- A terminal runs programs as the host's user. A host MUST NOT offer terminals unless configured to, and SHOULD limit `exec` and `shells` to what it was configured for.
- ACTIVITY ([Session §8](./VROOM-Session.md#8-audit)) names the terminal and the action, never the text typed or pasted.
- The screen shows what the program printed. Input the program doesn't echo, such as a password prompt's, never appears on the screen; anything printed does, secrets included.
- OSC 52 clipboard writes from a program reach a person's clipboard only through their own terminal's policy. A host MUST NOT apply them to its own clipboard.
- The pane interface ([§7.6](#76-programs-that-own-panes)) gives a program only its own panes, and surfaces it shows under their leases. A host MUST offer only the pane kinds its configuration allows (browser tabs only where it serves Browsable, for example), refusing others with `4605`, and SHOULD refuse with `4603` a `display` or `remote` pane its policy doesn't permit for that program.
- In band, the start request needs the host's one-time token ([§7.6.1](#761-finding-the-host)), so output alone, such as a printed file, can't start the pane interface.
- A `remote` pane reaches only hosts the host is configured to reach, with its own credentials. A program can't supply credentials, nor learn them.

---

## 10. Later

- File transfer and port forwarding, as further channels.
- Gateways: `vroom-to-ssh` and `ssh-to-vroom`.
- Nesting: a child opening panes of its own, through a pane interface of its own.

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

Draft. The agent access in §5, the multiplexing in §7 and the pane interface in §7.6 and §7.7 are new in 0.3.0 and expected to change as hosts implement them. The pane interface's local and in-band forms have been tried in a prototype on macOS and Linux.

## License

MIT
