# Step 1 Result: Calls Grep Run Clean

Operator-only preserved result from a clean workshop run.

- Thread: `019e96b2-7923-7303-b460-38721aa6d89f`
- Mode: `grep-only`
- Feature: `voice/video calls`
- CWD: `/Users/defendend/workshop`
- Duration: about 4 minutes
- Status: completed

Do not use this file as input to a new clean run.

## Summary

The grep-only run produced a strong Android vs iOS architecture comparison for `voice/video calls`. It found the main literal anchors quickly because this feature has highly searchable names: `call`, `voip`, `phone.*`, `video`, `CallKit`, `PushKit`, and related file names.

## Android Findings

- Entry/gating: `VoIPHelper.startCall(...)`.
- Central owner: `VoIPService`.
- Incoming path: `MessagesController` phone-call updates into the service/pre-notification path.
- Private call UI: `VoIPFragment`.
- Group/video call UI: `GroupCallActivity`.
- Native handoff: Java service code into `Instance` / `NativeInstance`.
- OS integration: `AndroidManifest.xml` permissions, foreground service, notification actions, Telecom-related declarations.

## iOS Findings

- Entry/gating: `PresentationCallManagerImpl.requestCall(...)` and app/peer-info call actions.
- Central owner: `PresentationCallManagerImpl`.
- Protocol/session owner: `CallSessionManager`.
- Runtime owner: `OngoingCallContext` and group-call runtime contexts.
- UI: `CallController`, `CallControllerNodeV2`, `VoiceChatController`, `VideoChatScreen`.
- OS integration: PushKit and CallKit via `AppDelegate` / `CallKitIntegration`, plus `Info.plist` privacy/background declarations.

## Key Comparison

Android is service-centric: `VoIPService` owns protocol, native runtime, UI state callbacks, foreground notification, and OS audio behavior.

iOS is more layered: `PresentationCallManagerImpl` coordinates current calls, `CallSessionManager` owns MTProto session state, presentation objects bind UI, and `TelegramVoip` / `TgVoipWebrtc` own runtime.

## Demo Use

This artifact is useful for showing that `grep-only` can be fast and quite good when literal anchors are strong, but it still requires manual reconstruction of ownership and is more exposed to generated/vendor/runtime noise.
