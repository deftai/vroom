# VROOM-Terminal

**Virtual Remoting Over Open Methods: Terminal Protocol**

*Terminals for humans and agents: a PTY's bytes for people, its screen for agents.*

Version: 0.3.0-draft | Status: Draft | Date: 2026-11-11 | Base: [VROOM-Session](./VROOM-Session.md)

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

VROOM-Terminal runs programs on a host's pseudo-terminals. A person gets the PTY's bytes, untouched, for their own terminal to draw, with Kitty keyboard, Kitty graphics and Sixel passing through. An agent gets the screen the host keeps by emulating the terminal: its text, cursor and changes, and actions on it with guards and act results, so it never has to emulate a terminal itself.

All four types can run over the same xumux connection, with one session and one lease table.

**Profile:** `vroom-terminal/1`. Each terminal is a surface with the ID `pty:<n>`. Its logical unit is the **cell**: `x` is the column and `y` the row, from `0` at the top left of the screen. A host is also a multiplexer ([§7](#7-multiplexing)): terminals are panes in windows in sessions, and a session is a surface with the ID `tsession:<n>`.

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
  "features": ["kitty_keyboard", "kitty_graphics", "sixel", "synchronized_output", "bracketed_paste", "osc52_clipboard", "hyperlinks"]
}
```

Host:

```json
{
  "shells": true,
  "exec": true,
  "agent": ["screen", "changes", "act"],
  "scrollback": 10000,
  "features": ["kitty_keyboard", "kitty_graphics", "bracketed_paste", "osc52_clipboard"]
}
```

| Field | Description |
|---|---|
| `terminal` | The client's terminal. The host uses `name` for `TERM` when the client opens a terminal, unless the request names `term` |
| `features` (client) | What the client's terminal can draw or send |
| `shells`, `exec` | Whether the host starts a login shell, or a command, on request |
| `agent` | The agent access offered ([§5](#5-agent-access)) |
| `scrollback` | Lines of scrollback the host keeps per terminal, `0` for none |
| `features` (host) | The features the host's emulator understands, so that agents see them on the screen too |

---

## 4. Messages

On `vroom.terminal.control`. Requests carry `request_id`, and errors are Session ERRORs ([§9](./VROOM-Session.md#9-error-codes)).

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x01` | TERM_HELLO | Both | [§3](#3-handshake) |
| `0x02` | PTY_LIST | Both | C→S: nothing. S→C: `ptys`, each `{id, title, rows, cols, command, running, exit?, session, window, source?, stream?}` |
| `0x03` | PTY_OPEN | C→S | `shell?: true` or `command` and `args`, `cwd?`, `env?`, `term?`, `rows`, `cols`, `xpix?`, `ypix?`. Answered with PTY_STATE |
| `0x04` | PTY_STATE | S→C | `pty`, `state` (`opened`, `exited`, `closed`), `exit_code?`, `signal?`. Sent in answer and when a terminal's program exits |
| `0x05` | PTY_CLOSE | C→S | `pty`: hangs the terminal up (SIGHUP) and removes it |
| `0x06` | PTY_ATTACH | C→S | `pty`: the host opens `vroom.terminal.pty.<n>` ([§6](#6-the-byte-stream)) |
| `0x07` | RESIZE | C→S | `pty`, `rows`, `cols`, `xpix?`, `ypix?` |
| `0x08` | SIGNAL | C→S | `pty`, `signal` by name: `INT`, `TERM`, `HUP`, `QUIT`, `TSTP`, `CONT`, `WINCH`, `KILL` |
| `0x10` | SCREEN_REQ | C→S | `pty`, `scrollback?` (lines before the screen), `styles?`, `if_newer_than?` |
| `0x11` | SCREEN | S→C | [§5.1](#51-the-screen) |
| `0x12` | CHANGES_REQ | C→S | `pty`, `since` (a `rev`) |
| `0x13` | CHANGES | S→C | `rev`, `rows` (changed row indices), or `too_old: true` |
| `0x14` | ACT | C→S | `pty`, `action`, `args`, `guard?`, `wait?` ([§5.3](#53-actions)) |
| `0x15` | ACT_RESULT | S→C | The [act result](./VROOM-Session.md#7-the-act-result) |
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

Opening, closing, resizing, signalling, input, actions and arranging ([§7.3](#73-leases)) need the terminal's lease ([Session §4](./VROOM-Session.md#4-the-control-lease)). PTY_OPEN needs none and gives the opener no lease. Watching (PTY_LIST, PTY_ATTACH, SCREEN_REQ, CHANGES_REQ) needs none. A terminal whose program has exited stays listed, with its last screen, until PTY_CLOSE.

---

## 5. Agent access

The host feeds each terminal's output to a terminal emulator as it arrives and keeps the result: the screen, the cursor, the modes the program has set, and up to `scrollback` lines above the screen. Agents read and act on that.

### 5.1 The screen

```json
{
  "rev": 52,
  "rows": 24, "cols": 80,
  "cursor": { "x": 12, "y": 3, "visible": true },
  "alternate": false,
  "title": "vim notes.md",
  "lines": ["$ ls", "notes.md  todo.txt", "$ vim notes.md", ""],
  "scrollback": ["…"],
  "styles": [[{ "x": 0, "w": 8, "fg": "#5fd7ff", "bold": true }]]
}
```

| Field | Description |
|---|---|
| `rev` | The terminal's revision ([Session §5](./VROOM-Session.md#5-revisions-and-guards)): it increases with any change to the screen, scrollback, cursor or title |
| `lines` | The screen's rows as text, one string each, without trailing spaces. A wide character takes two cells and appears once |
| `alternate` | The program is using the alternate screen, as full-screen programs do. Scrollback then belongs to the main screen |
| `scrollback` | Present when asked for: the lines above the screen, oldest first |
| `styles` | Present when asked for: per row, runs of cells with non-default styling |

`if_newer_than` answers `{rev, unchanged: true}` when nothing changed since.

### 5.2 Settling and waiting

For Terminal, **changed** in [Session §6](./VROOM-Session.md#6-settling) means the program wrote output that changed the screen or cursor. An `ignore` region is in cells.

WAIT and `wait` on ACT also accept `until: "text"`, which is met when the screen (or, with `scrollback: true`, its scrollback) contains `text`, a plain string, or matches `regex`:

```json
{ "until": "text", "text": "$ ", "timeout_ms": 10000 }
```

A program that exits also ends a wait, with `settled.state: "exited"`.

### 5.3 Actions

| `action` | `args` | Effect |
|---|---|---|
| `text` | `text` | Types the text as keystrokes would. `\n` is Return |
| `key` | `key`: a chord such as `ctrl+c`, `up`, `shift+tab`, `f5`, `alt+b` | One key, encoded as the program asked ([§5.4](#54-keys)) |
| `paste` | `text` | Pastes, bracketed if the program turned bracketed paste on |
| `scroll` | `lines` | Scrolls the agent's view of scrollback; the program sees nothing |

Each returns the act result with `rung: "none"` and, when it waited, `changes.regions` as the changed rows: `{x: 0, y: <row>, w: <cols>, h: 1}`.

A guard's `scope` is `surface` (any change) or `region` (a change in the given cells). `node` is refused with `4605`.

### 5.4 Keys

The host encodes an agent's keys by the modes the program has set: application cursor and keypad modes, bracketed paste, and the Kitty keyboard protocol at the level the program pushed. An agent names keys; it never writes escape sequences. A key the encoding can't express is refused with `4605`.

A person's input on the byte stream is not encoded: their terminal has already encoded it.

---

## 6. The byte stream

On `vroom.terminal.pty.<n>`:

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x01` | OUTPUT | S→C | Bytes the program wrote, as written |
| `0x02` | INPUT | C→S | Bytes for the program, as the client's terminal produced them |
| `0x03` | REDRAW | S→C | Bytes that draw the current screen from blank, sent first on attaching |

- The host MUST pass output through unchanged, in order, including sequences its emulator doesn't understand.
- On attaching, the host sends REDRAW, then OUTPUT from that point. A client that reconnects to a running terminal sees its screen at once.
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

A window has one size. When viewers' terminals differ, it takes the size of the last client to take the session's lease or to send RESIZE for one of its panes. Other viewers draw it in theirs as best they can, by cropping or padding.

### 7.3 Leases

A lease on a session (`tsession:<n>`) covers its windows and panes, as a lease on a display covers its windows. Arranging (SPLIT, ARRANGE, FOCUS, ZOOM, RENAME, KILL, WINDOW_NEW) needs the session's lease. Input and actions on one pane need that pane's lease, or its session's. A person may hold a session while an agent works one pane of another, and a person's request takes either from an agent as usual.

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
