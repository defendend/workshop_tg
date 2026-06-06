# Step 3 Result: Video Warning Plan - Map Guided

Operator-only preserved result from a clean workshop run.

- Thread: `019e96c8-20e4-7521-ae5d-ff7f9d917c58`
- Mode: `map-guided planning from architecture maps`
- CWD: `/Users/defendend/workshop`
- Duration: about 70 seconds
- Status: completed

Do not use this file as input to a new clean run.

## Task Interpretation

Add a unified privacy/safety warning before the user starts publishing local video or screen content when `video_call_privacy_warning_enabled` is enabled.

The warning policy must be separate from OS permission prompts and from WebRTC/tgcalls/native runtime initialization. It should run before camera, MediaProjection, broadcast picker, or native media startup.

## Architecture Map Signals Used

- Android calls are centered around `org.telegram.messenger.voip.*`, with `VoIPService` / `VoIPController` as call and media owners.
- Android product policy should sit before permission prompts and native capture/runtime initialization.
- Android `LaunchActivity` is routing/navigation shell, not feature state source of truth.
- iOS call ownership is split across `TelegramCallsUI`, `TelegramVoip`, `TgVoipWebrtc`, `SharedAccountContextImpl`, and `AccountContext`.
- iOS CallKit/PushKit incoming/system integration should be excluded unless incoming behavior is intentionally changed.

## Android Plan

- Put the reusable warning gate near the call/video action layer, not in native runtime.
- Gate private outgoing video before existing permission/runtime flow.
- Gate private in-call video enable before camera activation.
- Gate group/video chat camera enable before permission/preview/native capture.
- Gate screen sharing before MediaProjection if it is part of the same visual-media semantics.
- Read the new flag from an account/config owner rather than UI-only state.

Do not make JNI, WebRTC, `VoIPTextureView`, renderers, or generic routing code the source of warning policy.

## iOS Plan

- Put the reusable warning gate in `TelegramCallsUI` / account-context call action flow.
- Gate private outgoing video in the call request path.
- Gate private in-call camera toggle before permission/runtime.
- Gate group video chat camera enable before preview/native capture.
- Gate screen sharing before broadcast picker / IPC handoff.
- Keep `TelegramVoip`, `TgVoipWebrtc`, CallKit incoming answer, and PushKit handling downstream and unchanged.

## Cross-Platform Semantics

Trigger when the user is about to publish local visual media:

- outgoing private video call
- enabling camera in an active private voice call
- enabling camera in group/video chat
- screen sharing, if product scope includes that media path

Do not trigger for audio-only outgoing calls, incoming audio accept, group voice without camera, existing permission prompts alone, or native runtime internals.

## Demo Use

The map-guided plan is compact and strategic. It quickly identifies the correct layer and explicitly avoids permissions/runtime as the policy owner.
