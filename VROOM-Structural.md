# VROOM-Structural

**Virtual Remoting Over Open Methods — Structural Protocol**

*One UI tree for agents. Hosts map it to AX, UIA or AT-SPI.*

Official Name: VROOM-Structural | Base: [VROOM-Session](./VROOM-Session.md) | Companions: [Graphical](./VROOM-Graphical.md), [Terminal](./VROOM-Terminal.md), [Browsable](./VROOM-Browsable.md)
Version: 0.2.0-draft | Status: Draft | Date: 2026-11-08

---

## What is VROOM-Structural?

VROOM-Structural is the **structured UI access** protocol in the VROOM family. Agents see and act on a **normalized component tree** — roles, names, values, bounds, states, actions — not OS-specific Accessibility APIs and not raw pixels (unless the tree cannot serve the action).

| Protocol | Surface | Primary signal |
|----------|---------|----------------|
| **VROOM-Terminal** | Character grid / PTY | Bytes + control |
| **VROOM-Graphical** | Framebuffer + A/V | Video, pointer, voice |
| **VROOM-Structural** | Native app component tree | Snapshot / find / act |
| **VROOM-Browsable** | Browser tabs and pages | Page tree, navigation, network, video |

VROOM-Structural is an application protocol on [xumux](https://xumux.org). It MAY share one xumux connection with Graphical and/or Terminal.

**Implementation note:** the Structural messages, in the encoding selected in SESSION_WELCOME (CBOR by default), are a **channel codec** on `vroom.structural.control`, decoded after demux. See [I/O stack](./docs/io-stack.md).

```mermaid
flowchart TD
  agent[Agent]
  xumux[xumux]
  structural[VROOM-Structural]
  ir[Normalized UI IR]
  mac[macOS AX adapter]
  win[Windows UIA adapter]
  lin[Linux AT-SPI adapter]

  agent --> xumux --> structural --> ir
  ir --> mac
  ir --> win
  ir --> lin
```

**Design rule:** agents speak **only** the IR and Structural messages. Per-OS translation happens **on the host**, on the fly. No `axRole` / `ControlType` / ATK role in the agent-facing schema.

---

## Goals

- One agent vocabulary across macOS, Windows and Linux, shared with [Browsable](./VROOM-Browsable.md)
- Prefer tree fidelity over vision when the platform exposes a usable node
- Clear signals when the tree is empty or opaque, so the agent can move to [Graphical](./VROOM-Graphical.md#5-agent-access)
- Reliable, ordered delivery for snapshots and acts (never ride the unreliable pointer channel)
- Coexist with VROOM-Graphical (human View/Interact) without requiring video for agent-only sessions

## Non-Goals

- Replacing Splashtop/Parsec for humans
- Defining LLM tool-calling JSON (that sits above Structural)
- Guaranteeing a rich tree for games, raw OpenGL/Metal canvases, or remote-desktop mirrors
- Shipping OS-specific message dialects to agents

---

## Intermediate Representation (IR)

The IR is an **ARIA-inspired** tree. Host adapters MUST map native nodes into this shape and MUST NOT leak native type enums into agent-visible fields (optional `native` debug bag is allowed only when `debug: true` in the session).

### Node

```json
{
  "id": "n_8f3a",
  "role": "button",
  "name": "Save",
  "value": null,
  "description": null,
  "states": ["enabled", "focusable"],
  "actions": ["click", "focus"],
  "bounds": { "x": 120, "y": 80, "w": 96, "h": 32, "space": "screen" },
  "children": ["n_8f3b"],
  "parent": "n_8f39",
  "attrs": {}
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | string | MUST | Session-stable while the native backend id is stable; opaque to agents |
| `role` | Role | MUST | From the role enum below |
| `name` | string \| null | MUST | Accessible name; empty string → prefer `null` |
| `value` | string \| number \| boolean \| null | MUST | Current value when meaningful |
| `description` | string \| null | MAY | Help/ax description |
| `states` | string[] | MUST | From the state enum (may be empty) |
| `actions` | string[] | MUST | Invokable verbs the host can perform on this node |
| `bounds` | Bounds \| null | SHOULD | Screen or window space; null if unknown |
| `children` | string[] | MUST | Child ids in paint/tab order when known |
| `parent` | string \| null | MUST | Null for roots |
| `attrs` | object | MAY | IR-level only (e.g. `level` for headings, `selected` index). No OS enums |

### Bounds

```json
{ "x": 0, "y": 0, "w": 100, "h": 20, "space": "screen" }
```

`space`: `"screen"` \| `"window"` \| `"frame"`. Agents treat coordinates as host-local; Graphical RESIZE does not rewrite Structural ids.

### Roles (v1 closed set)

Agents MUST only emit/expect these roles. Hosts MUST map unknown native roles to the closest fit or `unknown`.

`window`, `dialog`, `sheet`, `alert`, `menu`, `menubar`, `menuitem`, `toolbar`, `tab`, `tablist`, `tabpanel`, `tree`, `treeitem`, `list`, `listitem`, `table`, `row`, `cell`, `grid`, `gridcell`, `group`, `region`, `scrollarea`, `scrollbar`, `splitter`, `slider`, `progressbar`, `statusbar`, `text`, `heading`, `link`, `button`, `checkbox`, `radio`, `switch`, `textbox`, `searchbox`, `combobox`, `spinbutton`, `image`, `icon`, `canvas`, `web`, `application`, `document`, `unknown`

### States (v1)

`enabled`, `disabled`, `focused`, `focusable`, `selected`, `checked`, `unchecked`, `mixed`, `expanded`, `collapsed`, `busy`, `hidden`, `offscreen`, `readonly`, `required`, `invalid`, `modal`, `pressed`

### Actions (v1)

`focus`, `click`, `double_click`, `right_click`, `set_value`, `clear`, `toggle`, `expand`, `collapse`, `scroll_into_view`, `press` (keyboard default activation)

Hosts SHOULD omit actions they cannot perform. Agents MUST NOT invent action names.

### Query language (FIND)

FIND uses a small declarative predicate (not CSS, not XPath) so hosts can compile to AX/UIA/AT-SPI efficiently.

```json
{
  "role": "button",
  "name": { "eq": "Save" },
  "states_all": ["enabled"],
  "ancestor": { "role": "dialog", "name": { "contains": "Settings" } },
  "limit": 10
}
```

Name match ops: `eq`, `contains`, `prefix`, `regex` (host MAY refuse `regex`).  
Boolean: `states_all`, `states_any`, `actions_any`.  
Scope: `root` (id), `ancestor` (nested query), `within` (id).

---

## Resolution ladder

For every ACT, the host MUST resolve in order and report which rung succeeded:

1. **Tree** — native action/pattern (AX press, UIA Invoke, ATK doAction)
2. **Hybrid** — tree bounds exist → synthesize click/type at bounds center (or focused node)
3. **Pixel** — the client supplies coordinates, usually from a [Graphical screenshot](./VROOM-Graphical.md#5-agent-access). The host never guesses from pixels itself; running vision is the agent's job

When rungs 1 and 2 fail, the host returns `ok: false`, `rung: "none"`, and `error.detail.needs_pixels: true`. A rung-3 ACT carries a `bounds` target and reports `rung: "pixel"`.

Agents SHOULD prefer FIND → ACT with `id` over raw coordinates. Coordinates are always a degradation.

---

## xumux Channels

| Channel | Reliability | Purpose |
|---------|-------------|---------|
| `vroom.structural.control` | **Reliable, ordered** | All Structural messages (STRUCT_HANDSHAKE, SNAPSHOT, FIND, ACT, …) |

Profile: `vroom-structural/1`. Surfaces are `app:<id>` and `window:<n>`. Leases, guards and settling are [Session's](./VROOM-Session.md).

```mermaid
sequenceDiagram
  participant A as Agent
  participant H as Host vroomd

  A->>H: xumux HELLO/WELCOME
  A->>H: structural STRUCT_HANDSHAKE
  H->>A: STRUCT_HANDSHAKE (caps)
  A->>H: SNAPSHOT_REQ
  H->>A: SNAPSHOT (IR fragment or full)
  A->>H: FIND
  H->>A: FIND_RESULT
  A->>H: ACT click id=n_8f3a
  H->>A: ACT_RESULT rung=tree ok
```

---

## Message types

Encoding is selected in [SESSION_WELCOME](./VROOM-Session.md#21-session_hello-and-session_welcome). `0xA1`–`0xA6` are also used, with these bodies, by [Browsable](./VROOM-Browsable.md).

| Code | Name | Dir | Body |
|------|------|-----|------|
| `0xA0` | STRUCT_HANDSHAKE | Both | versions, caps |
| `0xA1` | SNAPSHOT_REQ | C→S | depth, root, include |
| `0xA2` | SNAPSHOT | S→C | nodes[], root_ids[], rev, truncated? |
| `0xA3` | FIND | C→S | query |
| `0xA4` | FIND_RESULT | S→C | ids[], nodes? |
| `0xA5` | ACT | C→S | id \| bounds, action, args, guard?, wait? |
| `0xA6` | ACT_RESULT | S→C | [Act result](./VROOM-Session.md#7-the-act-result) |
| `0xA7` | EVENT | S→C | focus/value/structure deltas |
| `0xAF` | ERROR | Both | code, message, detail? |

### STRUCT_HANDSHAKE (`0xA0`)

```json
{
  "protocol": "vroom-structural",
  "version": "0.1.0",
  "platforms": ["macos"],
  "adapters": ["ax"],
  "max_snapshot_bytes": 1048576,
  "roles": ["button", "textbox", "..."],
  "actions": ["click", "set_value", "..."]
}
```

Server response MUST echo negotiated caps. `platforms` / `adapters` are informational (one host, one primary adapter).

### SNAPSHOT_REQ (`0xA1`) / SNAPSHOT (`0xA2`)

```json
{
  "root": null,
  "depth": 8,
  "include": ["bounds", "actions", "states"],
  "omit_hidden": true
}
```

```json
{
  "rev": 42,
  "root_ids": ["n_root"],
  "nodes": [ { "...": "Node" } ],
  "truncated": false,
  "partial": false
}
```

Large trees: host MAY send multiple SNAPSHOT messages with `partial: true` then a final `partial: false`, same `rev`. Clients MUST assemble by `id`.

### FIND (`0xA3`) / FIND_RESULT (`0xA4`)

See query language. FIND_RESULT MAY inline matching nodes or only ids (client then uses cached SNAPSHOT).

### ACT (`0xA5`) / ACT_RESULT (`0xA6`)

```json
{
  "request_id": "r1",
  "target": { "id": "n_8f3a" },
  "action": "click",
  "args": {},
  "guard": { "if_rev": 42, "scope": "node" },
  "wait": { "until": "settled", "quiet_ms": 150, "timeout_ms": 5000 }
}
```

Or degraded:

```json
{
  "request_id": "r2",
  "target": { "bounds": { "x": 10, "y": 10, "w": 1, "h": 1, "space": "screen" } },
  "action": "click"
}
```

ACT_RESULT is [Session's act result](./VROOM-Session.md#7-the-act-result).

### EVENT (`0xA7`)

Optional push updates: focus changes, value changes, subtree invalidate. Clients MAY ignore and re-SNAPSHOT.

---

## Host adapter contract

| Platform | Native API | Notes |
|----------|------------|-------|
| macOS | Accessibility (`AXUIElement`) | First reference adapter |
| Windows | UI Automation (UIA) | Prefer UIA over MSAA |
| Linux | AT-SPI2 | Best-effort; declare weak coverage in handshake |

Adapters MUST:

1. Map native → IR roles/states/actions
2. Maintain `id` stability for the session when the backend allows
3. Implement the resolution ladder
4. Never require the agent to send native identifiers

Adapters SHOULD expose `canvas` / `web` roles when the subtree is opaque so agents know to use pixels, or Browsable for a browser's content.

---

## Relationship to VROOM-Graphical

| Concern | Graphical | Structural |
|---------|-----------|------------|
| Human watch / voice | Yes | No (unless co-session) |
| Agent click-by-name | No | First-class FIND/ACT |
| Coordinates | Primary | Last resort |
| Video required | For humans only | No |

A session MAY open both. Structural acts SHOULD update the same desktop Graphical is streaming.

---

## Comparison: Structural vs raw AX/UIA vs vision-only

| | Per-OS a11y APIs | Vision-only | **VROOM-Structural** |
|--|------------------|-------------|----------------------|
| Agent portability | Poor | Good | **Good** |
| Click reliability on native widgets | High | Medium | **High (tree) → medium (pixel)** |
| Canvas/games | Poor | Required | Explicit fallback |
| Bandwidth | Low | High | Low for tree; burst on pixel |

---

## Implementation notes

- macOS: AX IPC is slow, one call per attribute unless batched, so hosts SHOULD batch attribute reads and cache subtrees, invalidating them from AX notifications.
- Windows: UI Automation through a cache request.
- Linux: AT-SPI2. Declare weak coverage in the handshake.

---

## Status

Draft 0.2.0. Subject to change. Part of the VROOM protocol family on xumux.

## License

MIT
