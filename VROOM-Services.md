# VROOM Services

**Optional file transfer and TCP forwarding beside the four surface types**

Version: 0.1.0-draft | Status: Draft | Date: 2026-11-13

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

These services use an authenticated [VROOM-Session](./VROOM-Session.md). They are not surface types: a host may offer file transfer without a desktop, or terminals without these file-transfer or forwarding services. Surface leases and lease grants do not authorize either service.

This version defines SFTP v3, direct TCP connections and reverse TCP listeners. It does not define UDP, dynamic SOCKS, arbitrary process execution, Unix-socket forwarding, credential delegation or switching operating-system users.

## 1. Discovery and authorization

SESSION_WELCOME MAY contain `services`. An absent service is unavailable; clients MUST NOT infer it from an offered surface type. Older clients ignore this optional field.

```json
{
  "services": {
    "sftp": {
      "version": 3,
      "channel_profile": "pipe/1",
      "aliases": [{ "name": "files", "read_only": false }],
      "limits": {
        "max_streams": 4,
        "max_packet_bytes": 262144,
        "max_requests": 64,
        "max_handles": 256,
        "open_timeout_ms": 10000,
        "drain_timeout_ms": 10000
      }
    },
    "tcp": {
      "profile": "vroom-tcp/1",
      "direct": true,
      "reverse": true,
      "limits": {
        "max_ids": 65536,
        "max_pending_connects": 16,
        "max_connections": 64,
        "max_listeners": 4,
        "max_pending_accepts": 16,
        "connect_timeout_ms": 10000,
        "reservation_timeout_ms": 10000,
        "channel_timeout_ms": 10000,
        "drain_timeout_ms": 30000
      }
    }
  }
}
```

All advertised limits are required positive integers for the service offered. The example values are not defaults. `max_streams`, TCP reservations/connections/listeners and pending accepts are counted per authenticated VROOM session. SFTP `max_requests` and `max_handles` apply to each SFTP stream; `max_packet_bytes` applies to each complete packet. TCP `max_ids` bounds the total connection and listener IDs issued during the entire authenticated VROOM session, across control-channel instances, including expired and failed allocations. Hosts MUST also enforce finite global limits. Services MUST NOT advertise features that their backends cannot enforce. A host offering only direct or only reverse TCP sets the other boolean to `false` and may omit limits used exclusively by that disabled direction.

This separation governs protocol entry points; it does not sandbox commands already authorized through a terminal or applications controlled through a desktop. SFTP names the file protocol here, not an SSH transport: confidentiality and integrity come from the VROOM transport or its enclosing secure channel.

The host lists only services and aliases enabled for this principal. Advertisement describes availability, not irrevocable permission. Each channel, file operation, connection and bind remains subject to operator policy. Policy MUST independently control file-service aliases, direct destinations and reverse bind addresses and ports. `client.kind`, a terminal attachment, a surface lease, an origin address and an opaque service ID are not substitutes for this authorization.

Services MUST open only after SESSION_WELCOME. An unsupported profile, disabled service or unauthorized alias/channel is rejected before opening its endpoint. Changing policy may revoke active resources; revocation MUST close the affected resources and MUST NOT silently attach them to a different alias or destination.

### 1.1 Channels and codecs

| Channel | Opener | Effective delivery | `metadata.profile` | Payload |
|---|---|---|---|---|
| `vroom.sftp.<id>` | Client | Reliable, ordered | `pipe/1` | SFTP byte stream inside pipe DATA |
| `vroom.tcp.control` | Client | Reliable, ordered | `vroom-tcp/1` | Session-selected CBOR or JSON |
| `vroom.tcp.<connection_id>` | Client for direct; host for reverse | Reliable, ordered | `pipe/1` | TCP byte stream inside pipe DATA |

Host-generated IDs in channel names are nonempty ASCII strings of at most 64 characters, using letters, digits, `_` or `-`. The client-chosen SFTP suffix is a canonical decimal integer string from 1 through 18446744073709551615, strictly increasing in OPEN_CHANNEL receive order across SFTP opens in that VROOM session. Parallel client opens MUST serialize their channel-open emission in that order; the host records the highest received suffix even if setup fails. This prevents replay without retaining every old SFTP name. Data-channel names MUST NOT be reused within the VROOM session. The fixed control name `vroom.tcp.control` may be reopened only after its prior instance has closed and its resources have been retired; at most one instance is active. Host-generated connection and listener IDs are opaque and MUST NOT be reused in that session, even after cancellation or reconnection of a service control channel. ID exhaustion is a refusal, never wraparound or reuse.

Pipe channels use the [xumux `pipe/1` profile](https://github.com/deftai/xumux/blob/main/spec/profiles/pipe.md), not the session's structured encoding. Service metadata is carried in OPEN_CHANNEL. The generic pipe `target` field MUST be absent on these channels: an endpoint comes only from an authorized alias or a bound TCP reservation. A peer MUST reject conflicting metadata rather than prefer one endpoint field over another. Nothing defined here is an application message on xumux channel 0.

## 2. SFTP

The client chooses a new channel ID suffix and opens a pipe with an advertised alias:

```json
{
  "name": "vroom.sftp.1",
  "reliable": true,
  "ordered": true,
  "metadata": { "profile": "pipe/1", "service": "sftp", "alias": "files" }
}
```

`alias` selects an operator-configured file service. It is not a path, command, executable, operating-system account or credential. The host MUST authorize the alias and reserve capacity before starting its backend. CHANNEL_ACK means the backend is ready; opening, including backend setup, MUST finish within `open_timeout_ms`. Rejection, cancellation or session loss during setup MUST stop and release any partially created backend.

### 2.1 Byte protocol

The stream is [SFTP version 3](https://datatracker.ietf.org/doc/html/draft-ietf-secsh-filexfer-02), an archived Internet-Draft rather than an RFC. The client sends SSH_FXP_INIT with version 3; the server answers SSH_FXP_VERSION with version 3. A client offering a higher version may negotiate down to 3; a peer requiring another version MUST end this service explicitly, not reinterpret its packets as v3.

SFTP's packet lengths, binary values, request IDs, handle values and response ordering remain SFTP's. Pipe DATA boundaries have no relationship to SFTP packets: a length prefix or packet may span several DATA messages, and one DATA may contain several packets. There is no JSON/CBOR translation, terminal escape framing or additional VROOM file-operation wrapper.

`max_packet_bytes` includes the SFTP four-byte packet-length prefix and MUST be at least 34000. Both directions MUST respect it. A receiver MUST validate the length before allocating the packet and detect truncated packets. READ replies and directory batches MUST fit the limit; they may be shorter than the requested amount as SFTP permits. A byte-stream backend MUST still be subject to these limits.

`max_requests` bounds requests awaiting their SFTP responses per stream, and `max_handles` bounds open file and directory handles per stream. The client MUST obey these limits; the server MUST enforce them. An ordinary operation refusal uses SFTP status codes. A malformed stream that cannot safely receive an SFTP status is reset rather than buffered indefinitely. Handle and request-ID namespaces are private to that stream. A duplicate outstanding request ID resets the stream: a status reply carrying that ambiguous ID would not identify which request it refused.

Core v3 operations retain their v3 meaning. Extensions are advertised only through the SFTP version exchange and only when implemented and authorized. Unknown extension announcements in INIT/VERSION are ignored as v3 specifies. An unsupported SSH_FXP_EXTENDED request receives the SFTP unsupported-operation status. Availability in another SFTP implementation is not a promise of support here; [OpenSSH's specification index](https://www.openbsd.org/openssh/specs.html) is a reference, not an implicit extension list.

### 2.2 Filesystem policy and lifetime

The authenticated principal and server policy determine access. A read-only alias MUST refuse every modifying operation, including mutations exposed by extensions. The host MUST enforce filesystem permissions and its configured path policy. If it claims confinement to a root, that confinement MUST withstand symlinks, alternate path spellings and rename/race escapes; choosing a starting directory is not confinement. Backend stderr, local executable names and credentials MUST NOT be inserted into the SFTP data stream.

A clean client EOF closes backend input only after preceding DATA has been written. The server may finish accepted operations and send their responses, then sends EOF. This drain is bounded by `drain_timeout_ms`; timeout resets the service and closes remaining handles. Reset, aborted channel setup, VROOM session loss or policy revocation closes every handle and terminates that stream's backend, if it owns one. A backend MUST NOT survive merely because it has not read stdin or produced stdout. Cleanup MUST be bounded and must not wait indefinitely for the peer.

A transport failure is not a rollback guarantee: a write may have taken effect before its response was lost. Clients MUST NOT automatically replay mutations on a new stream as if they were known not to have executed.

## 3. TCP control

At most one `vroom.tcp.control` channel may be active in a VROOM session. All reservations, listeners and TCP pipes belong to that control-channel instance. Opening a new control channel does not recover resources or IDs from a previous instance.

Every issued connection or listener ID consumes one of `max_ids` for the VROOM session, including IDs whose resources later fail, expire or close. Once exhausted, new allocations are refused with `4603` and `detail.reason: "limit"`; existing resources remain usable and cancellable. The peer can therefore bound its seen-ID and cancellation tracking. Reopening the control channel does not reset this budget or its seen-ID history. A fresh budget requires a new authenticated VROOM session; existing resources are not automatically interrupted when allocation capacity is exhausted.

Every request has a `request_id`: the canonical decimal string of an integer from 1 through 18446744073709551615, with no leading zero. IDs MUST strictly increase in control-channel receive order; replies echo them and may complete out of order. The host records the largest received ID before dispatch, including requests later refused by policy. An ID at or below that value is refused with ERROR and MUST NOT allocate another resource. This requires one counter, not an unbounded request-history table. Exhaustion requires closing the control instance and retiring its resources before opening another; counters never wrap. Unknown fields that could change endpoint selection or authorization MUST be refused, not ignored. Unknown message types are refused with `4605`.

| Type | Name | Direction | Body |
|---|---|---|---|
| `0x01` | TCP_CONNECT | C→S | `request_id`, `host`, `port`, `origin?` |
| `0x02` | TCP_READY | S→C | `request_id`, `connection_id`, `channel`, `expires_ms` |
| `0x03` | TCP_LISTEN | C→S | `request_id`, `bind_host?`, `bind_port` |
| `0x04` | TCP_LISTENED | S→C | `request_id`, `listener_id`, `bound`, `expires_ms` |
| `0x05` | TCP_ACTIVATE | C→S | `request_id`, `listener_id` |
| `0x06` | TCP_ACTIVE | S→C | `request_id`, `listener_id` |
| `0x07` | TCP_CANCEL | C→S | `request_id`, exactly one of `listener_id` or `connection_id`, `close_connections?` |
| `0x08` | TCP_CANCELLED | S→C | `request_id`, the same resource ID |
| `0xFF` | ERROR | S→C | `request_id`, `code`, `message`, `detail?` |

`host` is a DNS name or IP literal, without a scheme or embedded port; an IPv6 literal has no brackets. `port` is an integer from 1 through 65535. An origin is `{ "host": "192.0.2.8", "port": 53000 }`, with an IP literal and port from 0 through 65535. Origin is informational; in a direct request it is supplied by the client, not authenticated identity.

### 3.1 Direct connections

```json
{ "request_id": "1", "host": "service.example", "port": 443 }
```

If direct forwarding and the requested destination are permitted, TCP_READY reserves a one-use connection ID and its exact channel name:

```json
{ "request_id": "1", "connection_id": "c1", "channel": "vroom.tcp.c1", "expires_ms": 10000 }
```

`expires_ms` is an upper bound on the remaining lifetime when the host creates the response, measured on the host's monotonic clock. The timer starts with allocation, never after response delivery; queueing and network delay consume it. Clients MUST NOT assume a fresh full interval on receipt. This rule also applies to TCP_LISTENED.

This is a reservation, not a claim that TCP is connected. Unconsumed reservations count toward both `max_pending_connects` and `max_connections`; channel setup retains those reservations' capacity until it succeeds or is cleaned up. It expires after the returned interval, which MUST NOT exceed `reservation_timeout_ms`; expiry releases its capacity. The client opens the named pipe before expiry:

```json
{
  "name": "vroom.tcp.c1",
  "reliable": true,
  "ordered": true,
  "metadata": { "profile": "pipe/1", "service": "tcp", "direction": "direct", "connection_id": "c1" }
}
```

The host MUST atomically consume the reservation once, on the same authenticated session and active control-channel instance, before beginning connection setup. It MUST refuse unknown, expired, reused or mismatched IDs. No endpoint metadata may override the reservation. A second concurrent opener cannot create a second socket.

The host rechecks policy, resolves the destination and connects before CHANNEL_ACK, as `pipe/1` requires. DNS lookup and all address attempts together are bounded by `connect_timeout_ms`. Configured address restrictions MUST be checked against the actual address used; the host MUST connect the checked address rather than resolve the name again after checking. A failed or cancelled open consumes the reservation and closes any partial socket. Retrying requires a fresh authorized request and ID.

### 3.2 Reverse listeners

```json
{ "request_id": "2", "bind_host": "localhost", "bind_port": 0 }
```

`bind_host` defaults to `localhost`. It is either that literal or an IP literal. `localhost` means loopback only, never a wildcard. `bind_port` is an integer from 0 through 65535; 0 requests an available unprivileged port. Non-loopback binds, including `0.0.0.0` and `::`, require explicit operator permission. A host MUST NOT silently substitute a broader bind. It may refuse an unsupported address family.

TCP_LISTENED reports the actual socket addresses; `localhost` may produce one or both loopback families. Every address in `bound` MUST use the same actual port, and a partial multi-address bind MUST be rolled back if the complete requested result cannot be provided.

```json
{
  "request_id": "2",
  "listener_id": "l1",
  "bound": [{ "host": "127.0.0.1", "port": 43000 }],
  "expires_ms": 10000
}
```

The listener is bound but not yet accepting connections for forwarding. The client first records the listener ID and its own authorized local destination, then sends TCP_ACTIVATE. The host activates it and answers TCP_ACTIVE. Incoming channels may arrive before TCP_ACTIVE, but never before TCP_ACTIVATE; the client already knows their listener ID. Failure to activate within `expires_ms` (at most `reservation_timeout_ms`) closes the pending listener. A repeated activation of that active listener is idempotent.

For each accepted TCP connection the host allocates a fresh connection ID and opens a pipe:

```json
{
  "name": "vroom.tcp.c2",
  "reliable": true,
  "ordered": true,
  "metadata": {
    "profile": "pipe/1",
    "service": "tcp",
    "direction": "reverse",
    "connection_id": "c2",
    "listener_id": "l1",
    "bound": { "host": "127.0.0.1", "port": 43000 },
    "origin": { "host": "127.0.0.1", "port": 53001 }
  }
}
```

The client MUST accept only an ID it has not seen and a listener it explicitly requested and activated in this control-channel instance. `bound` must match a reported address. It connects only to the local destination already associated with that listener, under client-side policy, before CHANNEL_ACK. The host cannot supply a new client-side target in metadata. Origin and listener IDs do not grant access to any other client resource.

The host MUST bound accepted sockets awaiting channel acknowledgment by `max_pending_accepts` and `channel_timeout_ms`. A rejection, timeout or cancelled open closes its accepted socket. Accepted sockets, active pipes and pending direct opens all count toward `max_connections`; an implementation cannot bypass the limit by remaining in a handshake.

### 3.3 Cancellation and cleanup

TCP_CANCEL with `connection_id` cancels a reservation or aborts that connection and its pipe. With `listener_id`, it stops accepting and cancels pending accepts/channel opens. `close_connections` is valid only for a listener and defaults to `false`: already established pipes may drain independently. If `true`, those pipes are also reset and closed. A draining pipe remains subject to `drain_timeout_ms` after its listener is cancelled; it cannot retain resources indefinitely.

The client MUST stop accepting new channels for a listener when it sends its cancellation. A late channel-open request can cross the cancellation on another transport stream; it MUST be rejected, not attached to a new listener. The host sends TCP_CANCELLED only after the listener is closed and pending opens have been cancelled. A concurrently completing accept MUST either have become an established pipe before that cancellation or be closed; it cannot escape cleanup.

Known IDs may be cancelled repeatedly with the same successful result, but MUST NOT be resurrected. An unknown ID or an ID from another session/control instance receives `4603`. Implementations MUST bound retained cancellation records; when a record is discarded, the ID becomes unknown, never reusable.

Closing the control channel cancels every reservation, closes every listener and resets every TCP pipe owned by it, even if the VROOM session remains alive. Session loss does the same. Setup tasks, DNS requests, accepted sockets and channel-open futures need the same ownership and cleanup as established resources. Cleanup MUST finish within a finite host policy deadline without waiting for peer acknowledgment. A backend operation that cannot be cancelled immediately MUST remain owned and counted against resource limits until it reaches its own finite execution deadline, then discard its result. It cannot publish a resource into a newer control instance. Sockets, listeners, file handles and owned processes MUST still be closed within the cleanup deadline. A backend that cannot satisfy these bounded execution and cleanup requirements MUST NOT be offered for this service.

## 4. Byte flow, errors and resource bounds

All data pipes follow `pipe/1` DATA (`0x01`), EOF (`0x02`) and RESET (`0x03`) semantics. DATA is never fragmented; a sender sizes each message to the smaller of available credit and `maxFramePayload`. Receivers return credit only after bytes have been written to the local socket or file-service input. Receiving or copying into an intermediate queue is not consumption.

An EOF takes effect after all earlier DATA has been written. For TCP it shuts down the local write half, while the read half remains open. A gateway MUST preserve independent half-close; EOF is not a request to discard the opposite direction's response. Normal close after both EOFs preserves queued writes through the close exchange. RESET or session loss aborts the pipe and closes its local endpoint. Socket resets and abnormal backend exits MUST NOT be reported as successful EOF.

Opening a forbidden or unsupported service channel uses CHANNEL_REJECT `4010`; shared resource exhaustion uses `4011`; an unreachable endpoint uses `4500`. These are xumux/pipe channel-opening codes. Once the TCP control channel is open, structured ERROR uses Session codes `4603` (policy or unknown resource), `4604` (timeout) and `4605` (unsupported or invalid operation). It MAY use `detail.reason: "limit"` with `4603` for a resource limit. File-operation errors remain SFTP status responses. A malformed pipe message uses the pipe profile's protocol-error behavior. Endpoint failure after acknowledgment uses pipe RESET; it is not a fabricated TCP control success.

Hosts MUST bound counts, packet lengths, partial-packet buffering, listener backlogs, pending handshakes, queued writes and cleanup times. Capacity remains charged from reservation through completed cleanup, including pending and closing endpoints; cancelling a request does not release capacity while its backend still owns resources. A single SFTP stream or listener MUST NOT block unrelated session channels while waiting for a socket, process or peer. Backpressure must propagate to the original endpoint; a gateway MUST NOT regain credit early by adding an unbounded queue. Session/input traffic retains priority over these bulk services.

## 5. SSH gateways (interoperability guidance)

This section maps these services to [SSH connection semantics (RFC 4254)](https://datatracker.ietf.org/doc/html/rfc4254); it does not define an SSH transport or authentication protocol.

- A gateway serving VROOM over an authenticated SSH connection can map an authorized SFTP alias to the SSH `sftp` subsystem, direct TCP to `direct-tcpip`, and reverse listeners to `tcpip-forward`/`forwarded-tcpip`. Cancelling a reverse listener maps to `cancel-tcpip-forward`.
- A gateway accepting SSH and using an administrator-configured VROOM target maps those requests in the other direction. It must authorize terminal access, file services and each forwarding direction separately. Terminal mutations still require the terminal lease kinds specified by VROOM-Terminal.
- The gateway must preserve SSH host-key verification, authentication boundaries, byte order, half-close and explicit request failures. It must not turn an SSH username, subsystem name or destination into an unchecked VROOM host alias, executable or local target.
- An SSH reverse listener may produce backend channel opens before VROOM activation. The gateway must withhold VROOM channel publication until TCP_ACTIVATE and either bound those pending backend opens or refuse them. Activation expiry or cancellation still closes the backend listener and its pending opens.
- If an SSH backend resolves or connects outside the gateway, required destination/address policy must be enforced there or the gateway must pass a checked address. A gateway that cannot enforce a configured restriction must refuse, not claim that a name-only check enforces an address restriction.
- Unsupported SSH requests must fail explicitly. Agent forwarding, X11, arbitrary subsystem execution, credential delegation and per-user operating-system process switching are outside this version. Advertised extensions and bind-address semantics must be preserved or refused, never silently weakened.

## 6. Conformance cases

An implementation claiming these services MUST test at least:

1. Absent capabilities, unauthorized aliases/destinations and forbidden non-loopback binds allocate no endpoint resources; a surface lease or grant does not change the result.
2. SFTP INIT/VERSION v3, split/coalesced packet framing, binary names/data, out-of-order response IDs, read-only refusal and advertised extensions interoperate with an independent v3 client/server.
3. Oversized/truncated SFTP packets, excess outstanding requests and handle exhaustion remain bounded and leave no leaked handles or processes after reset.
4. DATA followed by EOF under stalled writes delivers every earlier byte before half-close; the other direction can still send its response. RESET and abrupt session loss do not masquerade as clean EOF.
5. Concurrent use of a direct reservation opens at most one socket; expired/cancelled IDs and wrong-session opens are refused. DNS-policy checks apply to the connected address.
6. Reverse activation cannot race an unknown listener ID; port-zero results report the actual bound addresses. Rejection/timeout of an incoming pipe closes the accepted socket.
7. Cancellation races with bind, connect, accept, channel acknowledgment and response delivery leave either the intended live resource or complete cleanup, never an orphan. Reopening the control channel does not recover old authority.
8. Listener cancellation tests both draining and forced-close modes; control-channel closure and session loss clean every pending and active resource within the configured bounds.
9. Slow file/network endpoints apply backpressure without starving session/input traffic, and aggregate resource limits apply across concurrent principals and streams.

## License

MIT
