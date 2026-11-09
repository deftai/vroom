# VROOM-Session

**Virtual Remoting Over Open Methods: the shared base**

Version: 0.1.0-draft | Status: Draft | Date: 2026-11-08

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

VROOM-Session is what the four VROOM types have in common: [Terminal](./VROOM-Terminal.md), [Graphical](./VROOM-Graphical.md), [Structural](./VROOM-Structural.md) and [Browsable](./VROOM-Browsable.md). It defines how a connection starts, who may act, how an actor knows what it is acting on, and what an action reports back.

---

## 1. On xumux

VROOM runs on [xumux](https://xumux.org) 0.3 and sits at its application layer, with each VROOM channel declaring a profile.

- HELLO `application` is `"vroom/1"`. HELLO `auth` carries the credentials; the host validates them.
- Each channel is named `vroom.<type>.<purpose>` and declares `metadata.profile`, for example `"vroom-browsable/1"`. A host that does not offer a profile rejects the channel with `4010`.
- The xumux frame **Type** is the VROOM message type, and each channel has its own type space.
- Implementations MUST read each channel's **effective** properties from CHANNEL_ACK or WELCOME, and MUST NOT assume a channel is unreliable because they asked for that. On single-stream transports every channel is reliable and ordered.
- Hosts declare xumux limits large enough for their largest message. A video host needs `maxMessageSize` larger than its largest keyframe, and `initialWindow` must be at least that value.
- Input and session channels MUST take priority over video and bulk data on a shared path. This is local scheduling and needs no wire change.

### 1.1 Channels and encoding

The client opens `vroom.session` in HELLO: reliable, ordered, profile `vroom-session/1`. Everything else opens after SESSION_WELCOME.

Structured messages use CBOR by default and JSON where negotiated. SESSION_HELLO and SESSION_WELCOME are always JSON. Hot-path messages are binary structs, defined by the type that uses them. All binary integers are big-endian.

---

## 2. Messages on `vroom.session`

| Type | Name | Direction | Purpose |
|---|---|---|---|
| `0x01` | SESSION_HELLO | C→S | Version, encodings, client identity |
| `0x02` | SESSION_WELCOME | S→C | Selected encoding, offered types, surfaces |
| `0x03` | SURFACES | S→C | The surfaces changed |
| `0x10` | LEASE_REQUEST | C→S | Ask to act on a surface |
| `0x11` | LEASE_RELEASE | C→S | Give a lease up |
| `0x12` | LEASE_STATE | S→C | Who holds what, sent on every change |
| `0x13` | LEASE_REVOKE_ALL | C→S | Kill switch |
| `0x20` | WAIT | C→S | Wait until a surface settles |
| `0x21` | WAIT_RESULT | S→C | Result of WAIT |
| `0x30` | ACTIVITY | S→C | Audit record of an action taken |
| `0x31` | AGENT_STATE | Both | What an agent is doing, for humans watching |
| `0x32` | CHAT | Both | Text chat beside the session |
| `0xFF` | ERROR | Both | Code, message, optional `request_id` |

### 2.1 SESSION_HELLO and SESSION_WELCOME

```json
{
  "version": "0.1.0",
  "encodings": ["cbor", "json"],
  "client": { "name": "vroom-cli", "version": "0.1.0", "kind": "agent" }
}
```

`client.kind` is `"human"` or `"agent"`. The host uses it for lease policy and audit, not for trust.

```json
{
  "version": "0.1.0",
  "encoding": "cbor",
  "session": "s_19f2",
  "types": {
    "graphical": { "profile": "vroom-graphical/1", "capabilities": ["video", "cursor", "input", "clipboard"] },
    "browsable": { "profile": "vroom-browsable/1", "capabilities": ["tree", "act", "video", "input"] }
  },
  "surfaces": []
}
```

The host lists only the types it offers. A daemon MAY offer one type. A deployment MAY run one process per type, and clients MUST NOT assume that the types share a process.

---

## 3. Surfaces and coordinates

A **surface** is something that can be observed and acted on. Its ID is a string with a type prefix: `display:1`, `window:88213`, `app:com.example.editor`, `tab:3`, `pty:1`.

```json
{
  "id": "display:1",
  "title": "Built-in Display",
  "bounds": { "x": 0, "y": 0, "w": 1728, "h": 1117 },
  "scale": 2.0
}
```

**Coordinates are logical and scoped to a surface**, with the origin at its top-left. A logical unit is a macOS point, a Windows DIP, a Wayland logical pixel or a CSS pixel. Every coordinate names its surface, so multiple monitors and HiDPI are never ambiguous.

Every image a host returns reports how it maps to its surface: `origin` in logical units, and `scale` in image pixels per logical unit. A client computes `logical = origin + image_pixel / scale`. Clients MUST NOT assume an image is 1:1 with its surface.

---

## 4. The control lease

A client needs a lease to act on a surface. Watching needs none. This replaces the View and Interact modes of earlier drafts: viewing is having no lease, and interacting is holding one.

```json
{ "request_id": "r1", "surface": "display:1", "ttl_ms": 30000, "wait_ms": 30000 }
```

`wait_ms` is how long to wait for a held lease before failing with `4604`. `0` fails at once with `4600`.

- A host MUST refuse an action without a matching lease with `4600` (LEASE_REQUIRED).
- A lease on a display covers the windows shown on it. A lease on a tab does not cover its display.
- One client holds a surface's lease at a time. By default a human's request takes a lease from an agent, and an agent's request waits for a human to release it.
- Physical input at the host SHOULD suspend an agent's lease on the affected display until the input has been idle for a configurable time. A suspended lease refuses actions with `4600` and `detail.reason: "local_input"`.
- `ttl_ms` is renewed by each action or by another LEASE_REQUEST, so a stalled agent loses control by itself.
- LEASE_REVOKE_ALL ends every lease immediately. Hosts SHOULD also provide a local kill switch that needs no connection.
- Hosts acting on a real desktop SHOULD show a visible indicator while an agent holds a lease.

---

## 5. Revisions and guards

Each surface has a `rev`: a `uint64` that increases whenever the host observes a change. Every observation (frame, screenshot, snapshot) reports the `rev` it reflects. Two observations that differ MUST NOT report the same `rev`, even if the host learned of the change from the observation itself.

An action MAY carry a **guard**, so that it is refused if what the actor saw has since changed:

```json
{ "if_rev": 41, "scope": "node" }
```

| `scope` | Refused when, since `if_rev` | Use |
|---|---|---|
| `node` | The target node is gone, or its role, name or bounds changed | Default for tree actions |
| `region` | Pixels changed in the target's region | Default for coordinate actions |
| `surface` | Anything changed | Rarely useful: a blinking cursor or clock trips it |

A refused action returns `4601` (STALE) and has no effect. A host that cannot evaluate a scope MUST refuse with `4605` (UNSUPPORTED) and MUST NOT ignore the guard.

---

## 6. Settling

An action's effects often arrive after it returns. Settling is a single definition of "the surface has stopped changing", so that agents do not need sleeps.

```json
{ "until": "settled", "quiet_ms": 150, "timeout_ms": 5000, "ignore": [{ "x": 0, "y": 0, "w": 200, "h": 24 }] }
```

A surface is settled when nothing has changed for `quiet_ms`, outside the logical regions in `ignore`. A host that cannot honour `ignore` MUST refuse with `4605`. Each type defines "changed":

| Type | Changed means |
|---|---|
| Graphical | The capture reported damage |
| Structural | A tree event arrived |
| Browsable | The DOM changed, or a document, script or stylesheet request is in flight |

`wait` is accepted on any action and in WAIT. On timeout the action itself still counts as performed, and `settled.state` is `"timeout"`.

---

## 7. The act result

Every action in every type except Terminal returns this shape, so that an agent always learns what happened without taking another look.

```json
{
  "request_id": "r7",
  "ok": true,
  "rung": "tree",
  "rev_before": 41,
  "rev_after": 44,
  "settled": { "state": "settled", "ms": 212 },
  "changes": {
    "regions": [{ "x": 120, "y": 80, "w": 640, "h": 300 }],
    "nodes": { "added": ["n_90"], "removed": ["n_8f3a"], "changed": ["n_12"] }
  },
  "error": null
}
```

| Field | Req. | Description |
|---|---|---|
| `request_id` | MUST | From the action |
| `ok` | MUST | The action was performed |
| `rung` | MUST | `tree`, `hybrid`, `pixel` or `none` ([Structural](./VROOM-Structural.md#resolution-ladder)) |
| `rev_before`, `rev_after` | SHOULD | Surface revisions around the action |
| `settled` | MAY | Present when the action asked to `wait` |
| `changes` | MAY | What the host observed. Absent means unknown, not unchanged |
| `error` | MUST when `ok` is false | `{code, message, detail?}` |

---

## 8. Audit

A host MUST emit ACTIVITY for every action it performs: time, client, surface, action, target, and result. It SHOULD also keep a local log. Text typed into a field the host knows is secret MUST be redacted.

---

## 9. Error codes

From the xumux application range. They are used in ERROR, in `error.code`, and in CLOSE_CHANNEL.

| Code | Name | Meaning |
|---|---|---|
| 4600 | LEASE_REQUIRED | No current lease on this surface |
| 4601 | STALE | A guard refused the action |
| 4602 | TARGET_NOT_FOUND | No such surface, node or tab |
| 4603 | NOT_PERMITTED | Host policy refuses this |
| 4604 | TIMEOUT | Did not complete in time |
| 4605 | UNSUPPORTED | This host cannot do this |
| 4606 | OS_PERMISSION_DENIED | The OS has not granted a permission. `detail.permission` names it |

---

## License

MIT
