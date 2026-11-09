# VROOM-Browsable

**Virtual Remoting Over Open Methods: Browsable Protocol**

*Browsers as browsers: tabs, pages, network and video, for humans and agents.*

Version: 0.1.0-draft | Status: Draft | Date: 2026-11-08 | Base: [VROOM-Session](./VROOM-Session.md)

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

VROOM-Browsable controls web browsers. It knows what a desktop protocol cannot: URLs, frames, requests, console output, downloads, dialogs and storage. It needs no OS screen or accessibility permissions when the host launches the browser, so it also works headless, in containers and in CI.

A browser on a real desktop can also be reached through [Graphical](./VROOM-Graphical.md) and [Structural](./VROOM-Structural.md). Browsable is richer, and those are what remains when it cannot attach.

**Profile:** `vroom-browsable/1`. Each tab is a surface with the ID `tab:<n>`, and its logical unit is the CSS pixel.

---

## 1. Goals and non-goals

- One vocabulary with Structural, so an agent that knows one knows most of the other.
- No CSS, XPath or CDP required of agents. Hosts MAY accept selectors as an explicit capability.
- Live video and takeover for humans, on any xumux binding.
- Works with every engine the host offers, with capabilities stating the differences.

Not goals: defining LLM tool schemas, or being a test runner.

---

## 2. Channels

| Channel | Requested delivery | Purpose |
|---|---|---|
| `vroom.browsable.control` | Reliable, ordered | Everything in [§4](#4-messages) |
| `vroom.browsable.events` | Reliable, ordered | Network and console events, when subscribed |
| `vroom.browsable.video.<tab>` | See [Graphical §3](./VROOM-Graphical.md#3-video) | The tab's video, in Graphical's video format |
| `vroom.browsable.feedback` | Reliable, ordered | [Graphical's feedback messages](./VROOM-Graphical.md#34-feedback), naming the numeric tab ID |
| `vroom.browsable.audio.<tab>` | See [Graphical §6](./VROOM-Graphical.md#6-audio) | The tab's audio |
| `vroom.browsable.pointer` | Unreliable, unordered | [Graphical's pointer format](./VROOM-Graphical.md#41-pointer-channel) |
| `vroom.browsable.input` | Reliable, ordered | [Graphical's input format](./VROOM-Graphical.md#42-input-channel) |

Video and input are byte-for-byte Graphical's formats, so a viewer needs one decoder and one input path. The surface in those messages is the numeric tab ID.

---

## 3. Handshake

BROWSE_HELLO answers the client's BROWSE_HELLO with what the host can do.

```json
{
  "engines": ["chromium", "firefox", "webkit"],
  "mode": "launched",
  "capabilities": ["video", "audio", "input", "network", "console", "storage", "downloads"],
  "video_sources": ["screencast"],
  "selectors": false,
  "evaluate": false
}
```

| Field | Description |
|---|---|
| `mode` | `launched`: the host owns the browser and its profile. `attached`: the host joined a browser it did not start, possibly a person's real profile. |
| `video_sources` | How tab video is produced, best first: `tab_capture` (browser-native frames) or `screencast` (compressed frames re-encoded by the host). Availability varies by engine and mode. |
| `selectors`, `evaluate` | Whether those capabilities are enabled. Hosts MUST default both to false in `attached` mode. |

---

## 4. Messages

Structured, on `vroom.browsable.control`. Every action takes a `request_id`, and `guard` and `wait` as in [Session](./VROOM-Session.md#5-revisions-and-guards), and returns the [act result](./VROOM-Session.md#7-the-act-result). Type `0xA3`–`0xA6` bodies follow [Structural](./VROOM-Structural.md#message-types), using the same codes.

| Type | Name | Dir | Body |
|---|---|---|---|
| `0xB0` | BROWSE_HELLO | Both | [§3](#3-handshake) |
| `0xB1` | TAB_LIST | C→S, S→C | Tabs: `{id, url, title, loading, audible, context}` |
| `0xB2` | TAB_OPEN | C→S | `url?`, `context?` |
| `0xB3` | TAB_CLOSE | C→S | `tab` |
| `0xB4` | TAB_EVENT | S→C | Opened, closed, navigated, crashed; a new tab reports its `opener` |
| `0xB5` | NAVIGATE | C→S | `tab`, `url`, `wait` |
| `0xB6` | HISTORY | C→S | `tab`, `action`: `back`, `forward` or `reload` |
| `0xA1` | SNAPSHOT_REQ | C→S | As Structural, plus `tab` and `frame?` |
| `0xA2` | SNAPSHOT | S→C | As Structural |
| `0xA3` | FIND | C→S | As Structural, plus `tab` |
| `0xA4` | FIND_RESULT | S→C | As Structural |
| `0xA5` | ACT | C→S | As Structural |
| `0xA6` | ACT_RESULT | S→C | [Act result](./VROOM-Session.md#7-the-act-result) |
| `0xB7` | SCREENSHOT_REQ | C→S | [Graphical's request](./VROOM-Graphical.md#5-agent-access), with a tab surface, plus `full_page?` |
| `0xB8` | SCREENSHOT | S→C | [Graphical's result](./VROOM-Graphical.md#5-agent-access) |
| `0xB9` | SUBSCRIBE | C→S | `network`, `console`: booleans, `tab?` |
| `0xBA` | DIALOG | S→C | `tab`, `kind`, `message`. The page is blocked until it is answered |
| `0xBB` | DIALOG_ANSWER | C→S | `tab`, `accept`, `text?` |
| `0xBC` | FILE_CHOOSER | S→C | `tab`, `multiple`, `accept` |
| `0xBD` | FILE_SET | C→S | `tab`, `paths` or `transfer` |
| `0xBE` | DOWNLOAD | S→C | `tab`, `name`, `url`, `state`, `path?` |
| `0xC0` | STORAGE_GET | C→S | `context`, `include`: `cookies`, `local_storage` |
| `0xC1` | STORAGE_STATE | S→C | The state |
| `0xC2` | STORAGE_SET | C→S | `context`, the state |
| `0xC3` | EVALUATE | C→S | `tab`, `frame?`, `expression`. Refused unless `evaluate` is true |
| `0xC4` | EVALUATE_RESULT | S→C | `request_id`, `value` or `error` |
| `0xC5` | STREAM_START | C→S | As [Graphical](./VROOM-Graphical.md#31-starting-a-stream), with `tab` for `surface` |
| `0xC6` | STREAM_STOP | C→S | `tab` |
| `0xC7` | STREAM_STATE | S→C | As Graphical, with `tab` |

On `vroom.browsable.events`: `0xD0` NETWORK_EVENT (`tab`, `id`, `phase`, `method`, `url`, `status?`, `bytes?`) and `0xD1` CONSOLE_EVENT (`tab`, `level`, `text`, `url?`, `line?`). Bodies are metadata only. Hosts MUST NOT include request or response bodies or credential headers.

---

## 5. The page tree

Nodes are [Structural's Node](./VROOM-Structural.md#node), and the same closed lists of roles, states and actions apply. `document`, `link`, `heading` and `textbox` map from ARIA without translation. Web-only facts go in `attrs`:

| `attrs` key | Meaning |
|---|---|
| `url` | A link's target |
| `frame` | The frame a node belongs to |
| `shadow` | The node is inside a shadow root |
| `input_type` | The HTML input type of a `textbox` |
| `selector` | A usable selector, only when `selectors` is true |

Rules for hosts:

- IDs MUST stay the same while an element keeps its role and name, and MUST NOT expose engine IDs. A host MAY issue a new ID when either changes, and the old ID then returns `4602` or a guard refusal. It MUST NOT silently point at a different element.
- Values of `password` fields MUST NOT appear in any node, whatever role the page gives them. Engines' accessibility trees commonly include them, so a host must remove them. If a host cannot determine whether a field is a password field, it MUST treat it as one. Such fields report `value: null` and `attrs.input_type: "password"`.
- Frames and shadow roots are flattened into one tree, with `attrs.frame` and `attrs.shadow` marking where a node came from.
- Cross-origin frames are included where the engine allows. A frame that cannot be read is a `web` node, so that the agent knows to use pixels.
- A `canvas` or empty tree is reported as a `canvas` node, not as an empty result.
- Virtualized lists bump `rev` as they re-render. A guard with scope `node` catches an element that has gone.

### Resolving actions

The ladder is Structural's, in web terms:

1. **Tree:** the engine's own element action, with its actionability checks (visible, stable, enabled, not obscured).
2. **Hybrid:** the node's bounds, using trusted pointer and keyboard input at that point.
3. **Pixel:** coordinates supplied by the client, usually from a screenshot.

A host MUST report which rung it used. It MUST NOT fall down a rung because an actionability check failed unless the action said `force: true`: a covered button usually means the page is not ready, or that a dialog is in the way.

---

## 6. Settling

For Browsable, "changed" means that the DOM changed, or that a document, script or stylesheet request is in flight. CSS animations do not count. A host that cannot locate changes MUST refuse `ignore` with `4605`. Long-lived connections (WebSocket, streaming fetch, event streams) MUST NOT prevent settling. `wait.until` also accepts `"load"` and `"domcontentloaded"`.

---

## 7. Video

Tab video follows [Graphical's rules](./VROOM-Graphical.md#3-video), including dropping frames before they are queued.

- `screencast` delivers compressed frames that the host re-encodes, so text passes through two lossy steps. It works on every engine.
- `tab_capture` takes frames from the browser's own capture path and SHOULD be preferred where available.
- A host follows the selected tab unless the client pinned one. When the tab changes, it sends VIDEO_CONFIG and a keyframe.

Hosts SHOULD stop capturing when no viewer is connected, and report `video_sources` truthfully: agents without viewers use SCREENSHOT.

---

## 8. Security

Page content is untrusted, and a page can address an agent directly through its text. This protocol cannot prevent that, but it limits what follows from it.

- **Origin policy.** A host SHOULD support an allow-list and a block-list of origins, refusing NAVIGATE and blocking top-level navigation elsewhere with `4603`. Agents get this at the host, where a page cannot talk them out of it.
- **Attached mode is the sensitive case.** It reaches a person's real sessions and cookies. Hosts MUST show a visible indicator, MUST require explicit consent for each attach, and MUST default `evaluate`, `selectors` and STORAGE_GET to refused.
- **Secrets.** A host SHOULD let a client refer to a stored secret by name, typing it without returning it. Values typed into `password` fields MUST be redacted from ACTIVITY, and SNAPSHOT MUST NOT include their values.
- **Files.** FILE_SET and DOWNLOAD paths MUST be confined to directories the host configured.
- **Local listeners.** A viewer endpoint MUST require a token, and MUST check the `Origin` header so that a web page cannot drive it from a person's own browser.

---

## 9. Host guidance (informative)

| Concern | Suggested approach |
|---|---|
| Engine control | [Playwright](https://playwright.dev) as a library, with CDP sessions where it needs Chromium specifics |
| Tree | An ARIA snapshot with stable references, mapped into Node |
| Attached mode | A browser extension, since current Chrome restricts remote debugging on the default profile |
| H.264 encoding | WebCodecs in a hidden page of the same browser, which needs no native code |
| Tab capture | An extension capture stream fed to a hardware encoder through WebCodecs |

---

## Status

Draft 0.1.0. Subject to change.

## License

MIT
