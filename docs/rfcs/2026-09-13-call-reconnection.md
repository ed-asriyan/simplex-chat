# Call reconnection on network loss
## Problem
A call ends as soon as WebRTC reports `disconnected`, and it ends for both parties.

`connectionHandler` in [`packages/simplex-chat-webrtc/src/call.ts`](../../packages/simplex-chat-webrtc/src/call.ts) removes the state listener, reports `ended` and closes the connection:

```ts
if (pc.connectionState == "disconnected" || pc.connectionState == "failed" || ...) {
  clearConnectionTimeout()
  pc.removeEventListener("connectionstatechange", connectionStateChange)
  if (activeCall) setTimeout(() => sendMessageToNative({resp: {type: "ended"}}), 0)
  endCall()
}
```

`ended` makes the native layer call `apiEndCall`, which sends `x.call.end`, so the peer's call is torn down as well - whichever side blinked first.

`disconnected` is not a failure. Consent checks run every 4-6 seconds and consent expires only after 30 seconds without a response ([RFC 7675, section 5.1](https://www.rfc-editor.org/rfc/rfc7675.html#section-5.1)); browsers report `disconnected` on the first missed check and `failed` when consent expires. A one-second gap, or a Wi-Fi ⇄ mobile handover that only invalidates the candidate pair, currently ends the call.

Two further defects in the same handler need fixing for any of this to work, since reconnection depends on both: a mid-call transition straight to `failed` is ignored (`connectionStateChange` skips `connectionHandler` for `failed`, and `connectionTimeout` is cleared on connect), so such a call hangs with frozen media; and `getIceCandidates` is one-shot, so the candidates of a second ICE generation would only be flushed when gathering completes.

## Roles
The roles are those of the offer/answer model ([RFC 3264, section 3](https://www.rfc-editor.org/rfc/rfc3264.html#section-3)): the **offerer** is the party that generated the session description, the **answerer** the party that replied to it. An ICE restart is an offer, so it is the offerer that makes it.

They are decided at call setup and never change afterwards. In SimpleX the offerer is the **callee** - the side that accepted the call and took the `start` command path; the caller took the `offer` path and is the answerer.

Keeping the roles fixed is what removes glare: only one side ever sends a restart offer, so two offers cannot cross and no SDP rollback is needed - rollback is not dependable in the WebView versions `call.ts` still supports (`webView69Or70`). If the offerer is the side that lost the network, its offer is composed straight away and the agent delivers it once the network is back.

## State
Per call, in addition to what is already kept in the call session:

| Property | Type | Initial value | Description |
|---|---|---|---|
| `isOfferer` | `boolean` | set at call setup `true`/`false` | represents the [role](#roles) |
| `wasConnected` | `boolean` | `false` | if the call was ever connected, `true`; otherwise `false` |
| `peerSupportsReconnect` | `boolean` | `false` | indicates whether the peer supports call reconnection. Is set from the [capability flag](#capability-flag) in the peer's session description |
| `restartGen` | `number` | `0` | the restart counter: [sending a restart offer](#sending-a-restart-offer) increments it, [applying one](#applying-a-restart-offer) copies it from the offer |
| `appliedGen` | `number` | `0` | the `restartGen` of the last description that was applied |
| `reconnect` | `ReconnectState \| undefined` | `undefined` | present exactly while the call is reconnecting: holds `since` (the time of the loss), the timers of R4, R6 and R14, and `pendingCandidates` |

"Generation" is the usual name for this counter - it is the `generation` attribute Chrome writes on every `a=candidate` line, and is defined in [XEP-0176](https://xmpp.org/extensions/xep-0176.html) as the value that "increments by 1 for each ICE restart". Candidates are only valid for the generation whose description is applied, since ICE credentials change with every restart; hence `appliedGen` and `pendingCandidates`.

## Rules
Every action links to the section that defines it. The rules hold on both sides of a call; those that name a role apply only to that side. What no rule matches has no effect - a message addressed to the other role, or one whose generation no rule admits, is ignored.

| # | IF | THEN |
|---|---|---|
| R1 | the call [loses the connection](#losing-the-connection) and `peerSupportsReconnect` | [enter reconnecting](#entering-reconnecting) |
| R2 | the call [loses the connection](#losing-the-connection) and not `peerSupportsReconnect` | [end the call](#ending-the-call) |
| R3 | [reconnecting](#entering-reconnecting), this side is the [offerer](#roles), and [the network changed](#the-online-event) less than `reconnectAfterOnline` ago | [send a restart offer](#sending-a-restart-offer) at once - the loss **is** that network change, so there is nothing to wait for |
| R4 | [reconnecting](#entering-reconnecting), this side is the [offerer](#roles), and the network did not change recently | wait `reconnectGrace`, then [send a restart offer](#sending-a-restart-offer) ([why wait?](#why-wait-before-the-first-restart)) |
| R5 | [reconnecting](#entering-reconnecting), this side is the [offerer](#roles), and [the `online` event fires](#the-online-event) | [send a restart offer](#sending-a-restart-offer) at once, without waiting for R6 |
| R6 | [reconnecting](#entering-reconnecting), this side is the [offerer](#roles), and `reconnectAttemptTimeout` passed since the last restart offer | [send a restart offer](#sending-a-restart-offer) again, which increments `restartGen` |
| R7 | this side is the [answerer](#roles), and a [`restartOffer`](#restartoffer) arrives with `gen > appliedGen` and its media matches | [apply it and answer](#applying-a-restart-offer) |
| R8 | this side is the [answerer](#roles), and a [`restartOffer`](#restartoffer) arrives with `gen <= appliedGen` | ignore it - it was superseded while in flight, and acting on it would renegotiate a connection that is already healthy |
| R9 | this side is the [offerer](#roles), and a [`restartAnswer`](#restartanswer) arrives with `gen == restartGen` and its media matches | [apply it](#applying-a-restart-answer) |
| R10 | a [`restartOffer`](#restartoffer) or [`restartAnswer`](#restartanswer) describes different media than the description it replaces | [end the call](#ending-the-call) - otherwise a peer could add a track mid-call through a restart |
| R11 | ICE gathers a candidate for a generation whose description this side has already sent | [send it](#sending-candidates) |
| R12 | [`candidates`](#candidates) arrive with `gen > appliedGen` | keep them in `pendingCandidates` and add them when that description is applied; `gen < appliedGen` is dropped |
| R13 | the connection comes back | [leave reconnecting](#leaving-reconnecting) |
| R14 | `reconnectBudget` passed since `reconnect.since` | [end the call](#ending-the-call) |

---

## Definitions

### Losing the connection

`pc.connectionState` becomes `disconnected` or `failed` - or, on a WebView without `connectionState`, `pc.iceConnectionState` does - on a call with `wasConnected`.

### Entering reconnecting

- `reconnect` is created, with `since` at the current time, which starts the budget of R14;
- the call moves to `CallState.Reconnecting`: the call status shows "Reconnecting…", the connecting sound plays, the remote video keeps its last frame, and the call timer keeps running;
- the state is reported to the native layer as `reconnecting`, and **nothing** is sent to the core - see [call status](#call-status);
- the peer connection is **not** closed. The DTLS association, the insertable-streams key and the transceivers survive, so media resumes after a freeze rather than being renegotiated from scratch.

Both roles enter this state. The answerer enters it either when its own connection reports the loss, or when a [`restartOffer`](#restartoffer) arrives before that.

### The `online` event

`window.ononline`, whose time is kept in `lastOnlineAt`.

The event can arrive on either side of the loss being reported, and on a handover it usually arrives **first**: the new network is validated within a few seconds while WebRTC only notices at its next consent check. So it is both acted on when a reconnection is already running (R5) and remembered, so that a loss reported shortly after it skips the wait (R3).

In an Android WebView that event never fires on its own: `navigator.onLine` is whatever the app last passed to `WebView.setNetworkAvailable()`, and defaults to `true`. The validated network state is already tracked by `NetworkObserver` for the core, so the call view forwards it into the WebView. A handover often produces no offline state there at all (`NetworkObserver` debounces it by 3 s), so the property is toggled to force the event.

### Sending a restart offer

Done by the [offerer](#roles) only.

1. `restartGen` is incremented;
2. ICE gathering is re-armed for the new generation, with the same timings as at call setup;
3. `createOffer({iceRestart: true})`, then `setLocalDescription` - this re-gathers candidates with new ICE credentials while the media pipeline is untouched;
4. the candidates gathered within `iceCandidates.delay` are collected;
5. a [`restartOffer`](#restartoffer) message is built and sent through `x.call.extra` - see [messages](#messages);
6. later candidates of this generation are trickled by R11.

If a newer generation started while step 4 was waiting, this offer is dropped instead of being sent.

### Applying a restart offer

Done by the [answerer](#roles) only, on a [`restartOffer`](#restartoffer) admitted by R7.

1. `restartGen` is set to the `gen` of the offer, and ICE gathering is re-armed for it - **this is the only point at which the answerer starts gathering during a reconnection**;
2. `setRemoteDescription` with the offer, then `createAnswer` and `setLocalDescription`; `appliedGen` becomes that `gen`;
3. the candidates carried by the offer are added, and any in `pendingCandidates` for this generation;
4. a [`restartAnswer`](#restartanswer) with the same `gen` is sent through `x.call.extra`;
5. later candidates of this generation are trickled by R11.

A failure in any step drops this generation only - the call is kept, and the next offer of R6 is applied instead.

### Applying a restart answer

Done by the [offerer](#roles). `setRemoteDescription` with the answer, which sets `appliedGen`, then the candidates carried by it and any in `pendingCandidates`. A failure drops this generation only.

### Sending candidates

ICE keeps producing candidates after a description was sent. They are trickled as [`candidates`](#candidates) messages, tagged with the generation they belong to.

**Who sends them, and when:** each side trickles candidates for the generation whose description *it* has sent. The offerer starts at step 2 of [sending a restart offer](#sending-a-restart-offer); the answerer starts at step 1 of [applying a restart offer](#applying-a-restart-offer), that is only after an offer has arrived. A side that is reconnecting but has nothing to answer yet sends nothing at all.

Before the first restart of a call the candidates are sent as a bare array, exactly as they are today, so a peer without reconnection support keeps receiving them in the shape it understands.

### Leaving reconnecting

`reconnect` is dropped with its timers and its buffered candidates, and `connected` is reported to the core. The call returns to `CallState.Connected` with its `connectedAt` unchanged, so the call duration continues rather than restarting.

The recovery is detected either by a connection state change or, every `reconnectCheckInterval`, by polling - an ICE restart does not always produce a state change on the answering side.

### Ending the call

`ended` is reported to the native layer, which calls `apiEndCall` → `x.call.end`. The call item is recorded with its full duration, because of [call status](#call-status).

### Call status

While reconnecting the connection state is reported as `reconnecting` and nothing reaches `apiCallStatus` until the call is connected again. Two states would otherwise be forwarded and both damage the call item, which `callStatusItemContent` moves through a small state machine:

- `WCSDisconnected` maps `CISCallProgress` to `CISCallEnded`, and `(CISCallEnded, _)` maps to nothing - the item can never leave "ended";
- `WCSConnecting` - which an ICE restart produces, since the connection state goes through `"connecting"` - maps the item to `CISCallNegotiated`. A successful reconnection puts it back into `CISCallProgress`, but a call that ends while it is in `CISCallNegotiated` matches `(Just _, WCSDisconnected) -> (CISCallEnded, 0)` and is recorded with a **zero duration**.

`WCSDisconnected` is sent once, when the call really ends. (`updated_at`, from which the duration is computed, is not affected by any of this: `updatedChatItem` does not touch it and `updateDirectChatItem_` writes back the value it read.)

### Why wait before the first restart

`disconnected` is documented as transient - the pair may resume on its own when packets flow again. A restart is not free: it is a full round trip over SMP plus a new ICE gathering, and it freezes media while the new pair is established. Spending that on a blip that would have healed itself is worse than waiting. The wait is skipped by R3 exactly where it cannot help - after a network change, where the old candidate pair is dead for certain.

---

## Messages

All of them travel as `x.call.extra`. `WebRTCExtraInfo.rtcIceCandidates` is LZW-compressed JSON that the core does not parse, and `x.call.extra` is already accepted in `CallNegotiated` on both sides (`APISendCallExtraInfo`, `xCallExtra`), which is the state of a connected call. So no core change is needed.

The decompressed payload becomes a union: an array is ICE candidates, as it is today, and an object is one of the messages below.

### restartOffer

Sent by the [offerer](#roles), see [sending a restart offer](#sending-a-restart-offer). `sdp` is the local description of `createOffer({iceRestart: true})`, `gen` is `restartGen`.

```jsonc
{"t": "restartOffer", "gen": 3, "sdp": "v=0\r\n…", "candidates": [...]}
```

### restartAnswer

Sent by the [answerer](#roles) in reply, with the `gen` of the offer, see [applying a restart offer](#applying-a-restart-offer).

```jsonc
{"t": "restartAnswer", "gen": 3, "sdp": "v=0\r\n…", "candidates": [...]}
```

### candidates

Candidates gathered after the description of that generation was sent, see [sending candidates](#sending-candidates).

```jsonc
{"t": "candidates", "gen": 3, "candidates": [...]}
```

### Capability flag

`rtcSession` in `x.call.offer` / `x.call.answer` is also an opaque payload, and both `RTCSessionDescription` and iOS `CustomRTCSessionDescription` ignore unknown keys, so support is announced next to `type` and `sdp`:

```jsonc
{"type": "offer", "sdp": "…", "smpReconnect": 1}
```

It is what sets `peerSupportsReconnect` on the other side; without it R2 applies. A client on WebView 69 or 70 does not announce it, so its calls keep ending on loss as they do now. If a restart message does reach a client without support, `addIceCandidates` throws on the non-iterable object, the `try`/`catch` in `processCommand` turns it into an `error` response, and the call is unaffected.

---

## Timings

| constant | value | used by |
|---|---|---|
| `reconnectGrace` | 2 s | R4 |
| `reconnectAfterOnline` | 10 s | R3 |
| `reconnectAttemptTimeout` | 10 s | R6 |
| `reconnectBudget` | 60 s | R14 |
| `reconnectCheckInterval` | 2 s | [leaving reconnecting](#leaving-reconnecting) |

None of these come from a standard - nothing standardises how long an application should keep trying. The bounds they sit between: the budget has to exceed the 30 s of consent expiry, or the call would be given up before WebRTC itself declares the connection dead; and it cannot buy much beyond that, because the agent's own reconnect backoff (`defaultReconnectInterval`: 2 s, growing after 10 s) means a restart offer cannot even be delivered promptly during a long outage.

## Scope
`call.ts` and the Kotlin call layer. The chat core does not change, so the apps can be built and tested against the core libraries of the matching release.

`WebRTCClient.swift` ends calls the same way (`case .disconnected, .failed: endCall()`) and needs the same rules against the native SDK; it follows separately, with the same wire format. An iOS client without it stays compatible: the message fails `decodeJSON` and is ignored.

Not covered: a transport that is gone rather than a candidate pair that is dead (a DTLS failure), which no ICE restart fixes and which needs a new `RTCPeerConnection` with the same `aesKey` and the same local streams; recovery after the app is killed; moving a call between devices. Until the first of these, R6 simply repeats until the budget runs out.

Desktop calls run in whichever browser the user has set as default, and [Chromium and Gecko differ](https://github.com/w3c/webrtc-pc/issues/3139) on whether a peer connection is recoverable at all once `connectionState` is `"failed"`. Restarts begin before consent expires, so `failed` is normally reached only when the restarts have already failed - but on a browser that cannot recover from it, the call waits out the budget instead of ending at once.
