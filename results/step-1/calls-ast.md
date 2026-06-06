# Step 1 Result: Calls AST Run Clean

Operator-only preserved result from a clean workshop run.

- Thread: `019e96b2-7df4-7973-8fc3-4cc23d3bdd31`
- Mode: `AST first, grep confirm`
- Feature: `voice/video calls`
- CWD: `/Users/defendend/workshop`
- Duration: about 7.5 minutes
- Status: completed

Do not use this file as input to a new clean run.

## Summary

The AST-first run produced a more methodical ownership map for `voice/video calls`. It was slower than grep-only, but better separated entry points, state owners, UI owners, protocol/session boundaries, native runtime boundaries, and OS integration.

## Android Findings

- Entry/gating: `VoIPHelper.startCall(...)`.
- Central owner: `VoIPService`, the high-value service/state owner.
- Private flow: `VoIPHelper` -> `VoIPService.startOutgoingCall` -> DH setup / `phone.requestCall` -> `phone.confirmCall` -> native media.
- Incoming flow: `MessagesController` -> `VoIPService.callIShouldHavePutIntoIntent` -> service -> `onCallUpdated` / `acceptIncomingCall`.
- Private UI: `VoIPFragment`.
- Group/video UI: `GroupCallActivity`.
- Native boundary: `Instance.makeInstance(...)` and `NativeInstance.makeGroup(...)`.

## iOS Findings

- Entry/gating: `PresentationCallManagerImpl.requestCall(...)`.
- App-level call owner: `PresentationCallManagerImpl`.
- Protocol/session owner: `CallSessionManager`.
- Private presentation state: `PresentationCallImpl`.
- Group/conference presentation state: `PresentationGroupCallImpl`.
- Runtime boundary: `OngoingCallContext` / `GroupCallContext` and `TgVoipWebrtc`.
- UI: `CallController`, `VoiceChatController`, and `VideoChatScreen`.

## Key Comparison

Android concentrates the feature in one Android service. That service owns protocol calls, media runtime, notifications, audio routing, sensors, and UI callbacks.

iOS splits responsibilities across modules: `TelegramCore` for protocol session state, `TelegramCallsUI` for presentation and call management, and `TelegramVoip` / native bindings for runtime.

## Demo Use

This artifact is useful for showing the workshop claim: AST-first wins on structural coverage and planning usefulness even when grep-only is faster.
