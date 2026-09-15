# WebRTC Calling Service

## Table of Contents

1. [Overview](#1-overview)
2. [Call State Machine](#2-call-state-machine)
3. [Android Implementation](#3-android-implementation)
4. [Desktop Implementation](#4-desktop-implementation)
5. [Common Call API](#5-common-call-api)
6. [Reconnection](#reconnection)
7. [IncomingCallAlertView](#7-incomingcallalertview)
8. [Source Files](#8-source-files)

## Executive Summary

WebRTC calling in SimpleX Chat operates over SMP (SimpleX Messaging Protocol) for signaling, with platform-specific WebRTC media implementations. Android uses a WebView-based approach with a dedicated `CallActivity` and foreground `CallService`, while Desktop opens the system browser and communicates via a NanoWSD WebSocket server on localhost. Both platforms share a common `CallManager` for call lifecycle and a `CallState` enum for state tracking. Call commands and responses are serialized as JSON and exchanged between the native layer and the WebRTC JavaScript layer.

---

## 1. Overview

Call signaling uses the same SMP protocol on all platforms -- call invitations, offers, answers, ICE candidates, and status updates flow through the chat backend via API commands. The WebRTC media plane, however, is implemented differently per platform:

- **Android**: WebView loads `call.html` from bundled assets; a `@JavascriptInterface` bridge (`WebRTCInterface`) forwards JSON messages between Kotlin and JavaScript.
- **Desktop**: The system browser opens `http://localhost:50395/simplex/call/`; a NanoWSD HTTP+WebSocket server serves `call.html` from classpath resources and relays JSON commands/responses over WebSocket.

Both platforms share the [`CallManager`](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/CallManager.kt) class (119 lines), which orchestrates incoming call acceptance, call ending, and notification management.

---

<a id="CallState"></a>

## 2. Call State Machine

Defined in [`WebRTC.kt`](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/WebRTC.kt#L50):

```
enum class CallState {
  WaitCapabilities,    // Call initiated, waiting for local WebRTC capabilities
  InvitationSent,      // Invitation sent to peer via SMP
  InvitationAccepted,  // Peer's invitation accepted locally
  OfferSent,           // SDP offer sent to peer
  OfferReceived,       // SDP offer received from peer
  AnswerReceived,      // SDP answer received from peer
  Negotiated,          // ICE negotiation in progress
  Connected,           // Media flowing
  Reconnecting,        // Transport lost, ICE is being restarted (section 6)
  Ended;               // Call terminated
}
```

**Outgoing call flow**: `WaitCapabilities` -> `InvitationSent` -> `OfferSent` -> `AnswerReceived` -> `Negotiated` -> `Connected` -> `Ended`

**Incoming call flow**: `InvitationAccepted` -> `OfferReceived` -> `Negotiated` -> `Connected` -> `Ended`

**On a lost transport**: `Connected` -> `Reconnecting` -> `Connected`, or -> `Ended` when recovery fails.

State transitions are driven by `WCallResponse` messages from the WebRTC layer. Each transition typically triggers a corresponding API command (e.g., `apiSendCallInvitation`, `apiSendCallOffer`).

---

<a id="ActiveCallView"></a>

## 3. Android Implementation

### 3.1 CallActivity.kt (464 lines)

[`CallActivity.kt`](../../android/src/main/java/chat/simplex/app/views/call/CallActivity.kt)

A dedicated `ComponentActivity` that hosts the call UI. Key responsibilities:

- **Intent handling** ([line 64](../../android/src/main/java/chat/simplex/app/views/call/CallActivity.kt#L64)): On `AcceptCallAction` intent, looks up the matching `RcvCallInvitation` and calls `callManager.acceptIncomingCall()`.
- **Lock screen support** ([line 160](../../android/src/main/java/chat/simplex/app/views/call/CallActivity.kt#L160)): `unlockForIncomingCall()` uses `setShowWhenLocked(true)` / `setTurnScreenOn(true)` on API 27+, falls back to window flags on older versions. `lockAfterIncomingCall()` reverses these settings.
- **Picture-in-Picture** ([line 99](../../android/src/main/java/chat/simplex/app/views/call/CallActivity.kt#L99)): `setPipParams()` configures PiP aspect ratio and source rect hint. On Android 12+ (`Build.VERSION_CODES.S`), auto-enter PiP is enabled for video calls. `onPictureInPictureModeChanged` toggles `activeCallViewIsCollapsed` and sends a `WCallCommand.Layout` command.
- **Permission checks** ([line 122](../../android/src/main/java/chat/simplex/app/views/call/CallActivity.kt#L122)): Checks `RECORD_AUDIO` and conditionally `CAMERA` permissions.
- **Service binding** ([line 181](../../android/src/main/java/chat/simplex/app/views/call/CallActivity.kt#L181)): Binds to `CallService` as a workaround for Android 12 background activity launch restrictions.
- **CallActivityView composable** ([line 208](../../android/src/main/java/chat/simplex/app/views/call/CallActivity.kt#L208)): Renders `ActiveCallView()` when permissions are granted and a call is active. Shows `CallPermissionsView` when permissions are needed. Shows `IncomingCallLockScreenAlert` for incoming calls on the lock screen.

### 3.2 CallService.kt (207 lines)

[`CallService.kt`](../../android/src/main/java/chat/simplex/app/CallService.kt)

An Android foreground `Service` that keeps the call alive when the app is backgrounded:

- **Foreground notification** ([line 131](../../android/src/main/java/chat/simplex/app/CallService.kt#L131)): Shows contact name (respecting `NotificationPreviewMode`), call type (audio/video), a chronometer when connected, and an "End call" action button.
- **WakeLock** ([line 66](../../android/src/main/java/chat/simplex/app/CallService.kt#L66)): Acquires `PARTIAL_WAKE_LOCK` to prevent CPU sleep during calls.
- **Notification channel** ([line 121](../../android/src/main/java/chat/simplex/app/CallService.kt#L121)): Creates `CALL_NOTIFICATION_CHANNEL_ID` with `IMPORTANCE_DEFAULT`.
- **Foreground service type** ([line 100](../../android/src/main/java/chat/simplex/app/CallService.kt#L100)): Uses `MEDIA_PLAYBACK | MICROPHONE` (+ `CAMERA` for video) on API 30+, `REMOTE_MESSAGING` on API 34+ when no active call.
- **Binder** ([line 158](../../android/src/main/java/chat/simplex/app/CallService.kt#L158)): `CallServiceBinder` allows `CallActivity` to call `updateNotification()` when call state changes.
- **CallActionReceiver** ([line 170](../../android/src/main/java/chat/simplex/app/CallService.kt#L170)): `BroadcastReceiver` that handles the `EndCallAction` from the notification.

### 3.3 CallView.android.kt (891 lines)

[`CallView.android.kt`](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt)

The `actual` platform implementation of `ActiveCallView()` and supporting composables:

- **ActiveCallState** ([line 74](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt#L74)): Manages proximity lock (screen-off wake lock), `CallAudioDeviceManager` for audio routing (earpiece/speaker/bluetooth), `CallSoundsPlayer` for ringtones and vibration. Implements `Closeable` to clean up resources on call end.
- **ActiveCallView** ([line 114](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt#L114)): Renders `WebRTCView` plus `ActiveCallOverlay`. Handles `WCallResponse` messages and dispatches corresponding API calls. Manages volume control stream (`STREAM_VOICE_CALL`), screen keep-on, and call command lifecycle.
- **WebRTCView** ([line 691](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt#L691)): Creates/reuses a static `WebView` via `AndroidView`. Configures `WebViewAssetLoader` for local asset loading. Sets up `WebRTCInterface` JavaScript bridge. Loads `file:android_asset/www/android/call.html`. Processes `WCallCommand` queue by evaluating `processCommand()` JavaScript.
- **ActiveCallOverlayLayout** ([line 329](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt#L329)): Full overlay with mic toggle, speaker/device selector, end call, video toggle, and camera flip buttons. Adapts layout for video vs audio calls.
- **CallPermissionsView** ([line 569](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt#L569)): Handles runtime permission requests for microphone and camera with a fallback to settings if the system dialog is not shown.

### 3.4 ActiveCallState

[`ActiveCallState`](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt#L74) (line 74 of `CallView.android.kt`):

| Component | Purpose |
|---|---|
| `proximityLock` | `PROXIMITY_SCREEN_OFF_WAKE_LOCK` -- turns screen off when phone is held to ear |
| `callAudioDeviceManager` | Manages audio routing between earpiece, speaker, Bluetooth, wired headset |
| `CallSoundsPlayer` | Plays connecting/ringing sounds and vibration patterns |
| `wasConnected` | Tracks if call ever connected (for end-of-call vibration) |
| `close()` | Stops sounds, vibrates on disconnect, releases proximity lock, clears audio manager overrides |

---

## 4. Desktop Implementation

### 4.1 CallView.desktop.kt (263 lines)

[`CallView.desktop.kt`](../../common/src/desktopMain/kotlin/chat/simplex/common/views/call/CallView.desktop.kt)

Desktop calls run WebRTC in the system browser, not an embedded WebView:

- **NanoWSD server** ([line 209](../../common/src/desktopMain/kotlin/chat/simplex/common/views/call/CallView.desktop.kt#L209)): `startServer()` creates a `NanoWSD` instance bound to `localhost:50395`. If that port is already in use it falls back to an OS-assigned free port (`port 0`); `WebRTCController` reads `server.listeningPort` for the browser URL. The server serves `call.html` from JAR resources at `/assets/www/desktop/call.html` for the path `/simplex/call/`. All other paths serve resources from `/assets/www/`.
- **WebSocket communication** ([line 238](../../common/src/desktopMain/kotlin/chat/simplex/common/views/call/CallView.desktop.kt#L238)): `MyWebSocket` handles WebSocket frames from the browser. `onMessage` deserializes JSON into `WVAPIMessage` and forwards to the response handler. `onClose` triggers `WCallResponse.End`.
- **WebRTCController** ([line 153](../../common/src/desktopMain/kotlin/chat/simplex/common/views/call/CallView.desktop.kt#L153)): Starts the server, then opens `http://localhost:<listeningPort>/simplex/call/` (normally `50395`) via `LocalUriHandler`. Processes `WCallCommand` queue by sending JSON over WebSocket to all active connections. On dispose, sends `WCallCommand.End` and stops the server.
- **SendStateUpdates** ([line 137](../../common/src/desktopMain/kotlin/chat/simplex/common/views/call/CallView.desktop.kt#L137)): Sends `WCallCommand.Description` with call state and encryption info text to the browser for display.
- **ActiveCallView** ([line 28](../../common/src/desktopMain/kotlin/chat/simplex/common/views/call/CallView.desktop.kt#L28)): Handles `WCallResponse` messages identically to Android (same state machine), plus a `WCallCommand.Permission` message on `Capabilities` error for browser permission denial guidance.

---

## 5. Common Call API

Defined in [`SimpleXAPI.kt`](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt):

| Function | Line | Description |
|---|---|---|
| `apiGetCallInvitations` | [L1842](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt#L1842) | Retrieve pending call invitations from the backend |
| `apiSendCallInvitation` | [L1849](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt#L1849) | Send call invitation to a contact with `CallType` |
| `apiRejectCall` | [L1854](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt#L1854) | Reject an incoming call |
| `apiSendCallOffer` | [L1859](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt#L1859) | Send SDP offer with ICE candidates and capabilities |
| `apiSendCallAnswer` | [L1866](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt#L1866) | Send SDP answer with ICE candidates |
| `apiSendCallExtraInfo` | [L1872](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt#L1872) | Send additional ICE candidates discovered after initial exchange |
| `apiEndCall` | [L1878](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt#L1878) | Terminate a call |
| `apiCallStatus` | [L1883](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt#L1883) | Report WebRTC connection status to the backend |

All functions send commands via `sendCmd()` to the chat core and return `Boolean` success status (except `apiGetCallInvitations` which returns `List<RcvCallInvitation>`).

---

<a id="reconnection"></a>

## 6. Reconnection

A connected call whose WebRTC connection reports `disconnected` or `failed` is not ended. `disconnected` is
reported on the first unanswered consent check, and consent checks run every 4-6 seconds while consent only
expires after 30 seconds ([RFC 7675](https://www.rfc-editor.org/rfc/rfc7675), section 5.1) - so the state
usually means a blip or a network handover that invalidated the candidate pair, not a dead session. The call
enters `CallState.Reconnecting` and ICE is restarted; the peer connection, its DTLS association, the
insertable-streams key and the transceivers are all kept, so media resumes after a freeze.

All of it is implemented in [`call.ts`](../../../../packages/simplex-chat-webrtc/src/call.ts); the native
layer only renders the state.

### 6.1 Timings

| Constant | Value | Meaning |
|---|---|---|
| `reconnectGrace` | 2 s | wait before the first restart - most interruptions heal on their own |
| `reconnectAfterOnline` | 10 s | a loss reported within this of the network coming back skips the grace: it is that network change, and it will not heal |
| `reconnectAttemptTimeout` | 10 s | no recovery within this - send another restart offer |
| `reconnectBudget` | 60 s | total, from losing the connection; then the call is ended |
| `reconnectCheckInterval` | 2 s | poll for recovery, an ICE restart does not always change the connection state on the answering side |

`window.ononline` starts a restart immediately instead of waiting out the grace period. The event can arrive
either side of the loss being reported, and on a handover it usually arrives **first** - the new network is
validated within a few seconds while WebRTC only notices at its next consent check. So the event both starts
a restart when a reconnection is already running, and is remembered (`lastOnlineAt`) so that a loss reported
shortly after it skips the grace period.

On Android that event does not fire on its own: in a WebView `navigator.onLine` is whatever the app last
passed to [`WebView.setNetworkAvailable()`](https://developer.android.com/reference/android/webkit/WebView#setNetworkAvailable(boolean))
and defaults to `true`. [`WebRTCView`](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt)
forwards `chatModel.networkInfo`, which [`NetworkObserver`](../../common/src/androidMain/kotlin/chat/simplex/common/helpers/NetworkObserver.kt)
already maintains from `registerDefaultNetworkCallback` with `NET_CAPABILITY_VALIDATED`. A handover usually
produces no offline state there at all -- `NetworkObserver` debounces it by 3 seconds, and the new network is
up before that -- so the property is toggled to `false` and back on every network change to force the event.

### 6.2 Roles

The roles are those of the offer/answer model (RFC 3264): the offerer is the party that generated the
session description, the answerer the party that replied. An ICE restart is an offer, so the offerer - the
callee, which took the `start` command path - is the only one that sends restart offers (`Call.isOfferer`);
the caller only answers. Offers therefore cannot collide, and no SDP rollback is needed - rollback cannot be
relied on in the WebView versions that `webView69Or70()` exists for. If the offerer is the side that lost the
network, its offer is queued by the agent and delivered when the network returns.

### 6.3 Signaling

Restart offers and answers are sent as `WCallResponse.Ice`, that is through `apiSendCallExtraInfo` and
`x.call.extra`, which the core accepts in `CallNegotiated` on both sides and whose payload it never parses.
Nothing in the chat core changes.

The decompressed payload of `ice` is a union: an array is candidates, as before, and an object is a
reconnection message (`ReconnectMessage` in `call.ts`):

```jsonc
{"t": "restartOffer" | "restartAnswer", "gen": 3, "sdp": "v=0\r\n…", "candidates": [...]}
{"t": "candidates", "gen": 3, "candidates": [...]}
```

`gen` is monotonic per call. Answers and candidates of a superseded generation are dropped, and candidates
that arrive before the description of their generation is applied are buffered in
`ReconnectState.pendingCandidates` - the same problem `afterCallInitializedCandidates` solves at call setup.
`getIceCandidates` is re-armed per generation, with the timings of the initial exchange.

A restart may not change the media of the call: `sameMediaSections` compares the `m=` sections of the new
description with the previous one, and the call is ended if they differ - otherwise a peer could add a track
mid-call through a restart.

### 6.4 Capability

`rtcSession` in `x.call.offer` / `x.call.answer` is also an opaque payload, and unknown keys are ignored both
by `RTCSessionDescription` and by the iOS client, so support is announced in it:

```jsonc
{"type": "offer", "sdp": "…", "smpReconnect": 1}
```

Without the flag from the peer, `canReconnect` is false and the call ends on `disconnected` as before, so
nobody waits out the budget for a client that will never answer. If a restart message does reach a client
without support, `addIceCandidates` throws on the non-iterable object and the `try`/`catch` in
`processCommand` turns it into an `error` response - the call is unaffected.

### 6.5 Call status

While reconnecting, `connectionState` is reported to the native layer as `"reconnecting"` and **no**
`apiCallStatus` is sent, until the call is connected again. The report is suppressed for every state, not only
for the lost ones: an ICE restart takes the connection through `"connecting"`, and both states damage the call
item in `callStatusItemContent`:

| reported | effect on an in-progress call item |
|---|---|
| `WCSDisconnected` | `CISCallProgress` -> `CISCallEnded`, and `(CISCallEnded, _)` -> nothing: the item can never leave "ended" |
| `WCSConnecting` | -> `CISCallNegotiated`. A successful reconnection returns it to `CISCallProgress`, but a call ended while in `CISCallNegotiated` matches `(Just _, WCSDisconnected) -> (CISCallEnded, 0)` and is recorded with a zero duration |

`WCSDisconnected` is sent once, when the call really ends. `connectedAt` is kept on recovery, so the duration
shown in the UI does not restart. `updated_at`, from which the stored duration is computed, is not affected by
these updates at all: `updatedChatItem` does not touch it and `updateDirectChatItem_` writes back the value it
read.

`"reconnecting"` is not a `WebRTCCallStatus` value: the existing `json.decodeFromString` of an unknown status
throws and is logged as not used, so a native layer without this change cannot mis-report the call.

The native layer also plays `CallSoundsPlayer.startConnectingCallSound` while reconnecting - the same sound as
when a call is being established - and stops it when the connection is back. The state is reported on every
connection state change while reconnecting, so the sound is started only on the transition into
`CallState.Reconnecting`.

---

<a id="IncomingCallAlertView"></a>

## 7. IncomingCallAlertView

[`IncomingCallAlertView.kt`](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/IncomingCallAlertView.kt) (128 lines)

An in-app notification banner shown when a call invitation arrives while the app is in the foreground:

- **IncomingCallAlertView** ([line 27](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/IncomingCallAlertView.kt#L27)): Starts `SoundPlayer` for the ringtone (suppressed if already in a call view). Shows `IncomingCallAlertLayout`.
- **IncomingCallAlertLayout** ([line 49](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/IncomingCallAlertView.kt#L49)): Colored banner with `ProfilePreview` of the caller, call type icon (audio/video), and three action buttons: Reject (red), Ignore (primary), Accept (green).
- **IncomingCallInfo** ([line 74](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/IncomingCallAlertView.kt#L74)): Shows the user profile image (for multi-user), call media type icon, and call type text (encrypted/unencrypted audio/video).

---

## 8. Source Files

| File | Path | Lines | Description |
|---|---|---|---|
| `CallView.kt` | [`common/src/commonMain/.../views/call/CallView.kt`](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/CallView.kt) | 28 | `expect fun ActiveCallView()`, delivery receipt waiting |
| `CallView.android.kt` | [`common/src/androidMain/.../views/call/CallView.android.kt`](../../common/src/androidMain/kotlin/chat/simplex/common/views/call/CallView.android.kt) | 916 | Android WebView WebRTC, overlay, permissions |
| `CallView.desktop.kt` | [`common/src/desktopMain/.../views/call/CallView.desktop.kt`](../../common/src/desktopMain/kotlin/chat/simplex/common/views/call/CallView.desktop.kt) | 309 | Desktop browser WebRTC via NanoWSD |
| `CallActivity.kt` | [`android/src/main/java/.../views/call/CallActivity.kt`](../../android/src/main/java/chat/simplex/app/views/call/CallActivity.kt) | 464 | Android call Activity, PiP, lock screen |
| `CallService.kt` | [`android/src/main/java/.../CallService.kt`](../../android/src/main/java/chat/simplex/app/CallService.kt) | 207 | Android foreground service for calls |
| `CallManager.kt` | [`common/src/commonMain/.../views/call/CallManager.kt`](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/CallManager.kt) | 119 | Call lifecycle management |
| `WebRTC.kt` | [`common/src/commonMain/.../views/call/WebRTC.kt`](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/WebRTC.kt) | -- | `CallState` enum, `WCallCommand`, `WCallResponse` types |
| `IncomingCallAlertView.kt` | [`common/src/commonMain/.../views/call/IncomingCallAlertView.kt`](../../common/src/commonMain/kotlin/chat/simplex/common/views/call/IncomingCallAlertView.kt) | 128 | In-app incoming call notification banner |
| `SimpleXAPI.kt` | [`common/src/commonMain/.../model/SimpleXAPI.kt`](../../common/src/commonMain/kotlin/chat/simplex/common/model/SimpleXAPI.kt) | -- | Call API commands (L1837--L1881) |
