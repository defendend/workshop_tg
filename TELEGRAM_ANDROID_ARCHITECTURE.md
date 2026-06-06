# Telegram Android Architecture

Operational architecture map for `/Users/defendend/workshop/telegram-android`.
Use it as a starting map, then confirm current owners with AST.

## Scope

- Project root: `/Users/defendend/workshop/telegram-android`
- Primary app source: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main`
- Product wrappers: `/Users/defendend/workshop/telegram-android/TMessagesProj_App*`
- Native/runtime boundary: `/Users/defendend/workshop/telegram-android/TMessagesProj/jni`

## Project Shape

Android is mostly one large Java app module plus wrapper app modules and native runtime islands. Do not infer feature ownership from Gradle modules. Most ownership is inside `TMessagesProj/src/main/java` as package/layer ownership, often through per-account singleton controllers and helpers.

Core runtime shape:

`ApplicationLoader` -> `LaunchActivity` / `BaseFragment` UI -> per-account owners via `AccountInstance` -> controllers/helpers/storage/network -> JNI/native runtime only when the task crosses that boundary.

## Core Owners

| Layer | Primary owner | Use when |
|---|---|---|
| Process lifecycle | `org.telegram.messenger.ApplicationLoader` | App init, providers, push bootstrap, process-level state. |
| Main shell / external routing | `org.telegram.ui.LaunchActivity` | Deep links, share intents, app-level permission/integration routing, root navigation. |
| Screen lifecycle | `org.telegram.ui.ActionBar.BaseFragment` | Feature screens, dialogs, navigation helpers, per-account accessors. |
| Navigation stack | `org.telegram.ui.ActionBar.ActionBarLayout` | Push/pop/replace, overlays, tablet/multi-pane, predictive back. |
| Per-account service locator | `org.telegram.messenger.AccountInstance` | Find the right controller/helper/storage/network owner for account-scoped work. |
| Server/domain state | `org.telegram.messenger.MessagesController` or narrower controller | Protocol-backed feature state and server requests. Confirm narrower owner before using `MessagesController`. |
| Local persistence | `org.telegram.messenger.MessagesStorage` | SQLite-backed dialogs/messages/cache/state and storage queues. |
| Network facade | `org.telegram.tgnet.ConnectionsManager` | Java MTProto request dispatch and native bridge. |
| User/account config | `org.telegram.messenger.UserConfig` | Selected account, current user, auth/local account flags. |
| Event bus | `org.telegram.messenger.NotificationCenter` | Existing UI/domain update events and observers. |
| Sending pipeline | `org.telegram.messenger.SendMessagesHelper` | Message/media send/edit/retry/upload orchestration. |
| File/media loading | `org.telegram.messenger.FileLoader` | Upload/download queues, file paths, progress. |
| Media runtime | `org.telegram.messenger.MediaController` | Playback, recording, gallery scans, video conversion, media export. |
| Notifications | `org.telegram.messenger.NotificationsController` | Push display, notification channels, unread counts, badges, reply actions. |
| Calls / VoIP | `org.telegram.messenger.voip.*`, especially `VoIPService` / `VoIPController` | Private calls, in-call media state, group call media path, Java/native VoIP bridge. |
| Native runtime | `TMessagesProj/jni/*` | MTProto, VoIP/WebRTC, codecs, sqlite, media/native crash/runtime issues. |

## High-Signal Entry Points

- Build/product variants: `settings.gradle`, `TMessagesProj/build.gradle`, wrapper `ApplicationLoaderImpl`.
- Manifest/platform declarations: `TMessagesProj/src/main/AndroidManifest.xml`.
- Process init: `ApplicationLoader`.
- External intents/deep links/share routes: `LaunchActivity.handleIntent` and related helpers.
- Feature screen behavior: concrete `BaseFragment` subclass first; use `LaunchActivity` only for routing.
- Per-account feature state: start from `AccountInstance`, then the narrow controller/helper.
- Server-backed state: controller/helper -> `ConnectionsManager.sendRequest` -> TL object.
- Local persistence: `MessagesStorage` or feature-specific storage owner.
- Message/media sending: `SendMessagesHelper`, then `FileLoader`/`MediaController` as needed.
- Notifications/badges/replies: `NotificationsController`, then manifest receivers/services.
- Calls/VoIP: `org.telegram.messenger.voip`, `VoIPService` manifest service, and JNI only if runtime behavior is the subject.

## Calls / Video / Screen Sharing Notes

For call-related planning, separate these layers:

- UI entry points: private call screens, group call screens, and `LaunchActivity` integration paths.
- Product policy / gating: should happen before permission prompts and before native capture/runtime initialization.
- Permission prompts: Android camera/microphone/media permission checks are integration details, not the source of truth for product warning policy.
- Runtime/native: `VoIPService`, `VoIPController`, WebRTC/libtgvoip/JNI own capture/session mechanics; avoid putting user-facing policy there unless all earlier entry points truly converge there and side effects are safe.
- Screen sharing / screencast: confirm whether it uses the same video/capture path before sharing the warning gate.

## Boundaries And Traps

- `LaunchActivity` is a routing/navigation shell, not the source of truth for feature state.
- Gradle modules are not feature modules.
- Manifest declarations show platform integration, not runtime ownership.
- `MessagesController` is central but often not the narrowest owner.
- Do not add `NotificationCenter` events before checking existing constants/usages.
- Do not put network logic in fragments if a controller/helper owns request/state flow.
- Do not treat JNI, generated TL, WebRTC, ffmpeg, sqlite, or third-party trees as ordinary app source.
- Confirm wrappers under `TMessagesProj_App*` when behavior can differ by product/store build.

## Confirmation Workflow

- Use this map to pick likely owners and layers.
- Confirm owners/callers/usages/current boundaries with AST.
- Use grep only for literal details: manifest entries, permissions, strings, exact keys, resource names.
