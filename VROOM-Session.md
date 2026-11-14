# VROOM-Session

**Virtual Remoting Over Open Methods: the shared base**

Version: 0.2.0-draft | Status: Draft | Date: 2026-11-13

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

VROOM-Session is what the four VROOM types have in common: [Terminal](./VROOM-Terminal.md), [Graphical](./VROOM-Graphical.md), [Structural](./VROOM-Structural.md) and [Browsable](./VROOM-Browsable.md). It defines how a connection starts, who may act, how an actor knows what it is acting on, and what an action reports back.

---

## 1. On xumux

VROOM runs on [xumux](https://xumux.org) 0.3 and sits at its application layer, with each VROOM channel declaring a profile.

- HELLO `application` is `"vroom/1"`. HELLO `auth` carries the credentials; the host validates them.
- Each channel is named `vroom.<type>.<purpose>` and declares `metadata.profile`, for example `"vroom-browsable/1"`. A host that does not offer a profile rejects the channel with `4010`.
- The xumux frame **Type** is the VROOM message type, and each channel has its own type space.
- Implementations MUST read each channel's **effective** properties from CHANNEL_ACK or WELCOME, and MUST NOT assume a channel is unreliable because they asked for that. On single-stream transports every channel is reliable and ordered.
- Hosts declare xumux limits large enough for their largest message. A video host needs `maxMessageSize` larger than its largest frame at its highest bitrate, which isn't always a keyframe, and `initialWindow` must be at least that value. Twice the largest frame is a reasonable margin. In one test at 12 Mbit/s with hardware H.264 and HEVC on an Apple M4, the largest frames were 190-273 KiB at 1080p and 813-820 KiB at 4K, where the largest was a delta frame about three times the first keyframe, so 512 KiB and 2 MiB. These are measurements from one encoder family, not limits for all.
- Input and session channels MUST take priority over video and bulk data on a shared path. This is local scheduling and needs no wire change.

### 1.1 Channels and encoding

The client opens `vroom.session` in HELLO: reliable, ordered, profile `vroom-session/1`. Everything else opens after SESSION_WELCOME.

Structured messages use CBOR by default and JSON where negotiated. SESSION_HELLO and SESSION_WELCOME are always JSON. Hot-path messages are binary structs, defined by the type that uses them. All binary integers are big-endian.

**Implementation note:** these encodings are **channel codecs**, decoded after xumux demux, never in the mux framer. See [I/O stack](./docs/io-stack.md).

---

## 2. Messages on `vroom.session`

| Type | Name | Direction | Purpose |
|---|---|---|---|
| `0x01` | SESSION_HELLO | C→S | Version, encodings, client identity |
| `0x02` | SESSION_WELCOME | S→C | Selected encoding, offered types, surfaces |
| `0x03` | SURFACES | S→C | The surfaces changed |
| `0x10` | LEASE_REQUEST | C→S | Ask to act on surfaces, or renew a lease |
| `0x11` | LEASE_RELEASE | C→S | Give a lease up |
| `0x12` | LEASE_STATE | S→C | Who holds what, sent on every change |
| `0x13` | LEASE_REVOKE_ALL | C→S | Kill switch |
| `0x14` | LEASE | S→C | A lease granted, joined or renewed, in answer |
| `0x15` | LEASE_GRANT | C→S | Let other clients join a lease ([§4.4](#44-grants)) |
| `0x16` | GRANT | S→C | The grant's ID and secret, in answer |
| `0x17` | LEASE_JOIN | C→S | Join a lease with a grant's secret |
| `0x18` | GRANT_REVOKE | C→S | Stop a grant, and take its joiners out |
| `0x20` | WAIT | C→S | Wait until a surface settles |
| `0x21` | WAIT_RESULT | S→C | Result of WAIT |
| `0x30` | ACTIVITY | S→C | Audit record of an action taken |
| `0x31` | AGENT_STATE | Both | What an agent is doing, for humans watching |
| `0x32` | CHAT | Both | Text chat beside the session |
| `0xFF` | ERROR | Both | Code, message, optional `request_id` |

### 2.1 SESSION_HELLO and SESSION_WELCOME

```json
{
  "version": "0.2.0",
  "encodings": ["cbor", "json"],
  "client": { "name": "vroom-cli", "version": "0.1.0", "kind": "agent" }
}
```

`client.kind` is `"human"` or `"agent"`. The host uses it for lease policy and audit, not for trust.

```json
{
  "version": "0.2.0",
  "encoding": "cbor",
  "session": "s_19f2",
  "client_id": "c3",
  "leases": ["surfaces", "kinds", "shared", "grants"],
  "types": {
    "graphical": { "profile": "vroom-graphical/1", "capabilities": ["video", "cursor", "input", "clipboard"] },
    "browsable": { "profile": "vroom-browsable/1", "capabilities": ["tree", "act", "video", "input"] }
  },
  "surfaces": []
}
```

`client_id` is the host's name for this connection, as LEASE_STATE and ACTIVITY use it. `leases` is what the host supports of [§4](#4-the-control-lease)'s lease features.

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

Graphical surfaces may separately report optional [desktop placement](./VROOM-Graphical.md#21-surface-placement) as `position`. That metadata does not change their surface-local coordinates.

Every image a host returns reports how it maps to its surface: `origin` in logical units, and `scale` in image pixels per logical unit. A client computes `logical = origin + image_pixel / scale`. Clients MUST NOT assume an image is 1:1 with its surface.

---

## 4. The control lease

A client needs a lease to act on a surface. Watching needs none. This replaces the View and Interact modes of earlier drafts: viewing is having no lease, and interacting is holding one.

A lease is held by one client, its **owner**, and other clients may join it ([§4.4](#44-grants)). It covers one or more surfaces, for one or more kinds of action ([§4.2](#42-kinds)), alone or shared with others ([§4.3](#43-conflicts-rank-and-sharing)).

### 4.1 Asking, renewing and releasing

```json
{ "request_id": "r1", "surfaces": ["display:1", "tab:2"], "kinds": ["input", "act"], "mode": "exclusive", "ttl_ms": 30000, "wait_ms": 30000 }
```

| Field | Description |
|---|---|
| `surfaces` | The surfaces the lease covers. `surface`, one ID, is shorthand for a list of one |
| `kinds` | What the lease is for ([§4.2](#42-kinds)). Absent means every kind |
| `mode` | `exclusive`, the default, or `shared` ([§4.3](#43-conflicts-rank-and-sharing)) |
| `ttl_ms` | How long the lease lasts unused |
| `wait_ms` | How long to wait for conflicting leases to end before failing with `4604`. `0` fails at once with `4600` |
| `lease_id` | Renews that lease, with a new `ttl_ms` if given, instead of asking for another. No other field is then given |

The host answers with LEASE (`0x14`): `request_id`, `lease_id`, `surfaces`, `kinds`, `mode` and `ttl_ms`, or with an ERROR. A client of an earlier draft is answered as before ([§4.5](#45-state-and-compatibility)).

- **All or nothing.** A lease's surfaces are granted together, renewed together and released together. While a request waits, a host MUST NOT hold some of its surfaces for it: so two clients asking for the same surfaces in different orders can't each hold half and wait for the other.
- A request for exactly the surfaces, kinds and mode of a lease the client owns renews that lease.
- A joiner ([§4.4](#44-grants)) may renew the lease it joined by `lease_id`, but not change its `ttl_ms`, which is the owner's: a joiner's renewal with `ttl_ms` is refused with `4603`.
- `ttl_ms` is renewed by each action under the lease, by any of its holders, and by renewing it, so a stalled agent loses control by itself.
- LEASE_RELEASE names `lease_id`, or `surface`. Naming a surface ends every lease the client owns that covers it, and takes the client out of every lease it joined that covers it, so it means the same on every host. The owner's release ends a lease. A joiner's release takes only that joiner out of it.
- A host MUST refuse an action without a lease of the kind it needs on its surface with `4600` (LEASE_REQUIRED), with `detail.surface` and `detail.kind` naming what was missing.
- A lease on a display covers the windows shown on it. A lease on a tab does not cover its display. Each type says what its surfaces cover.
- Physical input at the host SHOULD suspend leases of agent rank on the affected display until the input has been idle for a configurable time. A suspended lease refuses actions with `4600` and `detail.reason: "local_input"`.
- LEASE_REVOKE_ALL ends every lease and every grant immediately. Hosts SHOULD also provide a local kill switch that needs no connection.
- Hosts acting on a real desktop SHOULD show a visible indicator while an agent holds or has joined a lease.

### 4.2 Kinds

| Kind | For |
|---|---|
| `input` | Input as a person gives it, and actions that stand in for it: keys, text, the pointer, touch, a terminal's byte stream |
| `act` | Actions through the surface's own model rather than as input: tree actions, navigation, dialogs, files, the clipboard |
| `arrange` | What a surface holds and how: opening, closing, moving and focusing what's in it, its layout, and names |
| `size` | The surface's size |

Each type says which kind each of its operations needs. A kind a type has no operations for covers nothing there. Leases of different kinds never conflict, so one client can type into a terminal while another rearranges the session it is in.

### 4.3 Conflicts, rank and sharing

Two leases conflict when all of these hold:

- they cover a common surface, directly or through one that covers it (a display and one of its windows);
- they have a kind in common;
- they have different owners;
- at least one of them is exclusive.

A request that would conflict with held leases is settled by rank. A lease's rank is its owner's `client.kind`, `human` or `agent` ([§2.1](#21-session_hello-and-session_welcome)), and joiners act with it ([§4.4](#44-grants)).

- A person's request takes from agents. The conflicting leases of agent rank end at once, whole, and LEASE_STATE says so. Their holders' actions after that fail with `4600`.
- Every other request waits until the conflicting leases end, up to `wait_ms`.
- So a person's request takes a lease from an agent, and an agent's request waits for a person to release it, as in earlier drafts, but only where both want the same kind.

**Sharing.** Shared leases with a kind in common coexist on the same surfaces, whatever their ranks: two people pairing at one terminal, or a person and an agent working one page. An exclusive request conflicts with them, and a shared request with an exclusive lease, settled by rank as above.

- Shared input reaches the surface in the order the host receives it.
- A host MUST keep each holder's held keys and buttons apart, and release only that holder's when it leaves or loses the lease.
- A host MAY refuse `shared` for a kind with `4605`.

### 4.4 Grants

The owner of a lease can let other clients join it, as when a program starts others that act on its behalf. Joiners act under the owner's lease rather than holding leases of their own, so they never conflict with each other or with it. A lease a joiner asks for itself is its own, owned by it and ranked by its own `client.kind`, and conflicts with the lease it joined like any other.

| Type | Name | Dir | Body |
|---|---|---|---|
| `0x15` | LEASE_GRANT | C→S | `request_id`, `lease_id`, `surfaces?`, `kinds?`, `joins?`, `expires_ms?` |
| `0x16` | GRANT | S→C | `request_id`, `grant_id`, `secret` |
| `0x17` | LEASE_JOIN | C→S | `request_id`, `secret`. Answered with LEASE, for what the grant allows |
| `0x18` | GRANT_REVOKE | C→S | `request_id`, `grant_id`. Answered with LEASE_STATE ([§4.5](#45-state-and-compatibility)), with `request_id` |

- **Narrowing.** `surfaces` and `kinds` limit joiners to some of what the lease covers: its surfaces, or surfaces they cover ([§4.1](#41-asking-renewing-and-releasing)), such as one pane of a session, and some of its kinds. A grant never widens a lease: one that names a surface the lease doesn't cover, or a kind it lacks, is refused with `4603`.
- **Limits.** `joins` is how many clients may join by the grant, `1` by default, and `expires_ms` how long it can be used to join. Hosts SHOULD bound both.
- **Rank.** A joiner acts with the lease's rank, whatever its own `client.kind`, within the grant's surfaces and kinds. Its actions renew the lease.
- **Only the owner grants and revokes.** LEASE_GRANT or GRANT_REVOKE from anyone else, or naming a `lease_id` or `grant_id` the host doesn't know, is refused with `4603`.
- **Refusals.** LEASE_JOIN is refused with `4603`, the same for each, so a secret can't be probed, when:
  - its secret is unknown, used up, expired or revoked;
  - the client is the lease's owner;
  - the client has joined the lease already.
- **Revoking.** GRANT_REVOKE stops new joins, and every client that joined by the grant leaves the lease.
- **Ending.** The lease ends when its owner releases it, when it lapses, when it is taken from its owner, or when the owner's connection closes. Its joiners lose it then too. A joiner's connection closing takes only that joiner out of the lease.

A grant's secret is a bearer capability: whoever presents it joins, with the owner's rank.

- A host MUST make it unguessable (at least 128 random bits from a cryptographic source), and MUST NOT show it in LEASE_STATE or ACTIVITY.
- A client MUST pass it only over channels as private as its own credentials: never on a command line, in an environment other users can read, or in logs.
- A person's grant lets the joiner act as that person, so a client SHOULD grant only to programs it started or otherwise acts for.

### 4.5 State and compatibility

LEASE_STATE lists every lease, on every change:

```json
{
  "leases": [{
    "lease_id": "l7", "surfaces": ["tsession:1"], "kinds": ["arrange", "size"], "mode": "exclusive",
    "rank": "human", "owner": "c3", "joiners": ["c4", "c5"], "suspended": false,
    "surface": "tsession:1", "client": "c3", "kind": "human"
  }]
}
```

Clients are named by the `client_id` each one gets in SESSION_WELCOME. LEASE_STATE never carries a grant's secret.

- **Every change** includes a lease becoming suspended ([§4.1](#41-asking-renewing-and-releasing)) and ceasing to be, so people and viewers see when local input has stopped an agent.
- **Kept for compatibility.** Each entry also carries the fields earlier drafts' clients read: `surface`, the first of `surfaces`; `client`, the owner; and `kind`, the rank. So a client of an earlier draft reads a one-surface lease as before. `kind` is the rank, not one of `kinds`, and newer clients read `surfaces`, `owner` and `rank` instead.

SESSION_WELCOME's `leases` lists what the host supports of `surfaces` (more than one per lease), `kinds`, `shared` and `grants`. A request using something the host doesn't list is refused with `4605`. A client that uses none of them gets exactly the rules of earlier drafts: one surface per lease, every kind, and one holder at a time.

- **Telling them apart.** A host answers LEASE_REQUEST with LEASE_STATE, with `request_id`, as before, when the client's SESSION_HELLO `version` is earlier than 0.2, or absent, and the request uses none of `surfaces`, `kinds`, `mode` or `lease_id`. Every other request is answered with LEASE.
- **Not which operations need a lease.** Each type says which kind each operation needs, and on which surfaces, and that is the same for every client, whichever draft it speaks. A client of an earlier draft meets it with leases of every kind.

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
| Terminal | The program's output changed the screen or cursor |

`wait` is accepted on any action and in WAIT. On timeout the action itself still counts as performed, and `settled.state` is `"timeout"`.

---

## 7. The act result

Every action in every type returns this shape, so that an agent always learns what happened without taking another look.

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

A host MUST emit ACTIVITY for every action it performs: time, client, surface, action, target, result, and the lease it was taken under. For a client that joined the lease, ACTIVITY names both that client and the lease's owner. It SHOULD also keep a local log. Text typed into a field the host knows is secret MUST be redacted.

---

## 9. Error codes

From the xumux application range. They are used in ERROR, in `error.code`, and in CLOSE_CHANNEL.

| Code | Name | Meaning |
|---|---|---|
| 4600 | LEASE_REQUIRED | No current lease of the needed kind on this surface. `detail.surface` and `detail.kind` name what was missing |
| 4601 | STALE | A guard refused the action |
| 4602 | TARGET_NOT_FOUND | No such surface, node or tab |
| 4603 | NOT_PERMITTED | Host policy refuses this |
| 4604 | TIMEOUT | Did not complete in time |
| 4605 | UNSUPPORTED | This host cannot do this |
| 4606 | OS_PERMISSION_DENIED | The OS has not granted a permission. `detail.permission` names it |

---

## License

MIT
