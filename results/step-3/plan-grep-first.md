# Step 3 Result: Video Warning Plan - Grep First

Operator-only preserved result from a clean workshop run.

- Thread: `019e96c8-25b1-7143-9545-efaa18253cfc`
- Mode: `grep-first planning`
- CWD: `/Users/defendend/workshop`
- Duration: about 221 seconds
- Status: completed

Do not use this file as input to a new clean run.

## Task Interpretation

Add a product-level privacy/safety warning before actions that start local video or screen publishing when `video_call_privacy_warning_enabled` is enabled.

The warning must run before OS permissions and before native media runtime, not inside capturer/runtime code.

## Grep Discovery Strategy

The run started with broad text queries such as `video call`, `videoCall`, `isVideo`, `camera`, `requestVideo`, `screen share`, `screencast`, `CallKit`, and `authorizeAccess(.camera)`.

Those queries were noisy because they hit media, stories, resources, generated code, and third-party/runtime code. The run then narrowed to likely owners:

- Android: `VoIPHelper`, `VoIPService`, `VoIPFragment`, `GroupCallActivity`, `PrivateVideoPreview`.
- iOS: `PresentationCallManager`, `CallControllerNodeV2`, `VideoChatScreen`, `VoiceChatCameraPreviewController`, `PresentationCall`, `PresentationGroupCall`, `RPSystemBroadcastPickerView`.

The exact flag `video_call_privacy_warning_enabled` was not found, so the plan treats it as a new config integration.

## Android Plan

- Read the new flag through the existing app/config path.
- Add a reusable warning helper near VoIP UI/action code.
- Wrap `VoIPHelper.startCall(... videoCall=true ...)`.
- Wrap private in-call video enable in `VoIPFragment`.
- Wrap group camera enable in `GroupCallActivity`.
- Wrap group screen sharing before `startScreenCapture()` / MediaProjection.
- Keep `VoIPService`, `NativeInstance`, WebRTC/tgcalls, renderers, and `setVideoState` as downstream runtime, not policy source.

## iOS Plan

- Add a reusable warning presenter/helper in `TelegramCallsUI`.
- Gate private outgoing video around `PresentationCallManager.requestCall(... isVideo: true ...)`.
- Gate private in-call video enable around `CallControllerNodeV2.toggleVideo`.
- Gate group camera around `VideoChatScreen.onCameraPressed`.
- Intercept screen sharing before `RPSystemBroadcastPickerView` or related IPC handoff.
- Keep `OngoingCallContext`, `GroupCallContext`, native capturers, CallKit incoming answer, and broadcast upload extension downstream and unchanged.

## Remaining Uncertainty

Grep-first could find concrete call-site candidates, but it could not prove every hidden caller, every native handoff ordering, or the cleanest screen sharing interception point without more semantic tracing or UI testing.

## Demo Use

The grep-first plan is richer in literal call-site details, but it takes longer and visibly spends effort separating real owners from `video` / `camera` / `call` noise.
