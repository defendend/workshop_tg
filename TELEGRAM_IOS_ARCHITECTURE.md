# Telegram iOS Architecture

Operational architecture map for `/Users/defendend/workshop/telegram-ios`.
Use it as a starting map, then confirm current owners with AST.

## Scope

- Project root: `/Users/defendend/workshop/telegram-ios`
- Thin app shell: `/Users/defendend/workshop/telegram-ios/Telegram`
- Main UI/app logic: `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI`
- Core state/network/storage: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore`, `/Users/defendend/workshop/telegram-ios/submodules/Postbox`, `/Users/defendend/workshop/telegram-ios/submodules/MtProtoKit`

## Project Shape

iOS is a Bazel/XcodeGen-adjacent app with many Swift submodules plus native/third-party islands. The `Telegram/Telegram-iOS` shell is thin. Most application lifecycle, root UI, feature factories, account state, storage, networking, and calls logic lives under `submodules`.

Core runtime shape:

`AppDelegate` -> `SharedAccountContextImpl` -> active `AccountContext` / `Account` -> `TelegramEngine` + `Postbox` + `Network` -> feature UI controllers and native/runtime modules.

## Core Owners

| Layer | Primary owner | Use when |
|---|---|---|
| Build/project graph | `BUILD.bazel`, `MODULE.bazel`, submodule `Package.swift` | Build/index/project boundary questions. |
| Main app marker | `Telegram/Telegram-iOS/Application.swift` | Thin `UIApplication` subclass only; rarely the real owner. |
| App plist | `Telegram/Telegram-iOS/Info.plist` | URL schemes, privacy strings, background modes, intents. |
| Lifecycle / push / URL / session | `TelegramUI/Sources/AppDelegate.swift` | App bootstrap, PushKit, notifications, URL handling, background sessions. |
| Shared app context | `TelegramUI/Sources/SharedAccountContext.swift` / `SharedAccountContextImpl` | Cross-account state, app services, feature factories, call managers. |
| Shared/account protocols | `AccountContext/Sources/AccountContext.swift` | Protocol surface for app-wide and per-account services/factories. |
| Root UI | `TelegramUI/Sources/TelegramRootController.swift` | Main tabs/root controllers and top-level navigation composition. |
| Navigation | `Display/Source/Navigation/NavigationController.swift` | Custom stack, overlays, modals, master/detail. |
| Base UI controller | `Display/Source/ViewController.swift` | App-specific view controller lifecycle and presentation hooks. |
| Account metadata | `TelegramCore/Sources/AccountManager/*` | Account records, shared data, current account selection. |
| Authorized account runtime | `TelegramCore/Sources/Account/Account.swift` | Postbox, Network, managers, pending operations, account state. |
| Engine facade | `TelegramCore/Sources/TelegramEngine/TelegramEngine.swift` and domain files | High-level feature APIs over an account. Prefer this before direct Network/Postbox work. |
| Storage/views | `Postbox/Sources/Postbox.swift` and related files | Transactions, history/chat list/peer views, preferences, operation logs. |
| Network/protocol | `TelegramCore/Sources/Network/Network.swift`, `MtProtoKit`, `TelegramApi` | MTProto requests, datacenters, generated protocol objects. |
| Calls / VoIP | `TelegramCallsUI`, `TelegramVoip`, `TgVoipWebrtc`, AppDelegate PushKit/CallKit hooks | Private calls, group calls, call UI, CallKit/PushKit, native call runtime. |
| Extensions | `Telegram/Share`, `Telegram/NotificationService`, Watch/Siri/Widget modules | Share/notification/service/companion integration paths. |

## High-Signal Entry Points

- App lifecycle, push, URL, background session: `AppDelegate`, then plist declarations.
- Cross-account state, app-level services, controller factories: `SharedAccountContextImpl`.
- Per-account feature work: `AccountContext` protocol -> `TelegramEngine.<domain>` or feature module.
- New screen/UI behavior: feature controller/factory first; use `ViewController`/`NavigationController` only for lifecycle/navigation mechanics.
- Root tabs/chat list/settings composition: `TelegramRootController`.
- Chat behavior: `AccountContext/Sources/ChatController.swift`, then concrete `TelegramUI` chat implementation.
- Chat list behavior: `AccountContext/Sources/ChatListController.swift`, then `ChatListUI`.
- Server-backed state: `TelegramEngine.<domain>` and related `TelegramCore/Sources/TelegramEngine/*` domain files.
- Local persistence/cache/view updates: `Postbox` public transaction/view APIs, then relevant table/view files.
- Auth/login: `UnauthorizedAccount`, `TelegramCore/Sources/Authorization.swift`, and `AuthorizationUI`.
- Notifications: `AppDelegate`, `Telegram/NotificationService`, shared/account notification registration.
- Calls/VoIP: `TelegramCallsUI`, `TelegramVoip`, `TgVoipWebrtc`, plus PushKit/CallKit paths in `AppDelegate`.

## Calls / Video / Screen Sharing Notes

For call-related planning, separate these layers:

- UI entry points: private call UI, in-call controls, group call/video chat UI, screen sharing UI.
- Product policy / gating: should happen before permission prompts and before native/WebRTC capture starts.
- Permission prompts: plist privacy keys and camera/microphone prompts are integration details, not the source of truth for product warning policy.
- CallKit/PushKit: incoming/system call integration must be excluded unless the task explicitly changes incoming handling.
- Runtime/native: `TelegramVoip`, `TgVoipWebrtc`, WebRTC/third-party modules own media session mechanics; avoid user-facing policy there unless every relevant entry point safely converges there.
- Screen sharing / screencast: confirm whether it uses the same video/media path before sharing the warning gate.

## Boundaries And Traps

- `Telegram/Telegram-iOS/Application.swift` is nearly empty and usually not the app owner.
- `AccountContext` often declares protocols/factories; concrete state/UI lives elsewhere.
- Controller suffixes do not prove ownership; many controllers are UI shells around Core/Postbox/engine state.
- Prefer `TelegramEngine` domain APIs before direct `Network` or `Postbox` edits.
- Checked-in plist is not full entitlements truth; provisioning/build inputs also matter.
- `TelegramApi` is generated protocol surface; confirm generation flow before editing.
- `LegacyComponents`, Obj-C bridges, and native modules may still be active.
- Do not infer runtime ownership from plist/build declarations alone.

## Confirmation Workflow

- Use this map to pick likely owners and layers.
- Confirm owners/callers/usages/current boundaries with AST.
- Use grep only for literal details: plist keys, entitlements, strings, exact keys, resource names.
