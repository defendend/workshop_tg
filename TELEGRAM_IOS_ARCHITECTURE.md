# Telegram iOS Architecture

Reusable project architecture memory for `/Users/defendend/workshop/telegram-ios`. This is not a feature map.

## Scope

- Project root: `/Users/defendend/workshop/telegram-ios`
- App shell: `/Users/defendend/workshop/telegram-ios/Telegram`
- Main application logic: `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI`
- Core state/network/storage modules: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore`, `/Users/defendend/workshop/telegram-ios/submodules/Postbox`, `/Users/defendend/workshop/telegram-ios/submodules/MtProtoKit`

## High-Level Shape

iOS is a Bazel/XcodeGen-adjacent monorepo-style app with many Swift submodules and native/third-party islands. The main app shell under `Telegram/Telegram-iOS` is thin; the actual app delegate, shared context, root UI, navigation, feature factories, account state, storage, and networking live mostly in `submodules`.

The central runtime shape is:

`AppDelegate` -> `SharedAccountContextImpl` -> active `AccountContext`/`Account` -> `TelegramEngine` + `Postbox` + `Network` -> UI controllers built through factory closures and protocols.

Future agents should start from the relevant module owner rather than scanning `Telegram/Telegram-iOS` as if it contained most application logic.

## Key Artifacts

| Area | Artifact | Why it matters |
|---|---|---|
| Bazel project entry | `/Users/defendend/workshop/telegram-ios/BUILD.bazel:1` | SourceKit/Bazel BSP setup watches `Telegram/**` and `submodules/**`, target `//Telegram:Telegram`. |
| Bazel module deps | `/Users/defendend/workshop/telegram-ios/MODULE.bazel:1` | Local overrides for rules_apple/rules_swift/rules_xcodeproj/apple_support and build configuration. |
| Main app marker | `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Application.swift:3` | Thin `UIApplication` subclass overriding `sendEvent`. |
| Main app plist | `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:183` | Bundle ID, URL schemes, privacy keys, background modes, user activity intents. |
| App delegate | `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:216` | Main lifecycle/push/URL/session/account bootstrap owner. |
| Shared app context | `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/SharedAccountContext.swift:139` | Implementation of shared account context, account switching, active contexts, UI factories. |
| Shared context protocol | `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1321` | Protocol surface for app-wide services and factories. |
| Account context protocol | `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1641` | Per-account UI/domain bridge exposing account, engine, managers, configs, media/cache. |
| Root controller | `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/TelegramRootController.swift:74` | Main tabs/root controllers and top-level UI navigation entry. |
| Navigation controller | `/Users/defendend/workshop/telegram-ios/submodules/Display/Source/Navigation/NavigationController.swift:148` | Custom navigation stack, overlays, modals, master/detail, layout. |
| Base view controller | `/Users/defendend/workshop/telegram-ios/submodules/Display/Source/ViewController.swift:85` | App-specific ViewController abstraction with display node, navigation, presentation hooks. |
| Chat protocol | `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/ChatController.swift:1075` | Chat UI contract; actual implementations live in TelegramUI chat files. |
| Chat list protocol | `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/ChatListController.swift:12` | Chat list UI contract and actions. |
| Account manager API | `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/AccountManager.swift:84` | Account manager types and app account management helpers. |
| Account manager impl | `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/AccountManager/AccountManagerImpl.swift:33` | ValueBox-backed account metadata/shared data/current account state. |
| Account runtime | `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:1150` | Authorized account runtime: postbox, network, managers, pending operations, state. |
| Unauthorized runtime | `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:68` | Unauthorized account state/network path for login/auth. |
| Engine facade | `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/TelegramEngine/TelegramEngine.swift:4` | High-level feature API groups over an account. |
| Postbox storage | `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:4609` | Public storage/view/transaction facade. |
| Postbox internals | `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:1590` | ValueBox-backed internal storage tables and transaction state. |
| Network layer | `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:804` | MTProto network wrapper, request service, datacenter/proxy/status. |
| Network init | `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:429` | API ID/hash, app version, encryption provider, recaptcha/request verification arguments. |
| TelegramCore package | `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Package.swift:1` | SwiftPM boundary for Core depending on Postbox, MtProtoKit, TelegramApi, Crypto, Reachability, etc. |
| Postbox package | `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Package.swift:1` | SwiftPM storage boundary depending on sqlcipher, ManagedFile, RangeSet, Crypto, SSignalKit. |
| Share extension | `/Users/defendend/workshop/telegram-ios/Telegram/Share/Info.plist:23` | Share extension with principal class `ShareRootController`. |
| Notification service extension | `/Users/defendend/workshop/telegram-ios/Telegram/NotificationService/Info.plist:23` | Notification service extension with principal class `NotificationService`. |

## Layer Cards

### Build And Module Boundaries

The project uses Bazel module configuration and many Swift packages/submodules. `BUILD.bazel` configures SourceKit Bazel BSP around target `//Telegram:Telegram` and watches both app and submodule source trees. `MODULE.bazel` pins local build rule overrides and build input configuration. Individual submodules still expose `Package.swift` files useful for understanding logical ownership and dependencies.

Owner/source-of-truth notes:

- Build graph entry: `/Users/defendend/workshop/telegram-ios/BUILD.bazel:1`.
- Bazel dependency/rules setup: `/Users/defendend/workshop/telegram-ios/MODULE.bazel:1`.
- Logical Core module deps: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Package.swift:1`.
- Logical storage deps: `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Package.swift:1`.

### App Entry And Lifecycle

`Application.swift` is a thin `UIApplication` subclass. Real lifecycle ownership is in `AppDelegate` under `submodules/TelegramUI`. Its outline shows `SharedApplicationContext`, `AccountManagerState`, main window fields, account manager/context/auth context, push registry, notification token promises, background URL sessions, app group/bundle configuration, lifecycle callbacks, URL opening, PushKit, notification center handling, Siri intents, notification registration, update checks, notification payload parsing, and download helpers.

Owner/source-of-truth notes:

- App delegate/lifecycle/push/URL: `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:216`.
- Main app plist declarations: `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:183`.
- Background uploads/downloads: app delegate methods around `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:1697`.
- PushKit/notification token handling: app delegate properties around `:254`, methods around `:2112` and `:2938`.

### Shared Context And Account Context

`SharedAccountContext` is the cross-account/app-wide protocol. The implementation, `SharedAccountContextImpl`, owns main window, application bindings, base paths, account manager, app lock, notification controller, active authorized/unauthorized accounts, media/contact/location/call managers, current presentation data/settings, notification token registration, account switching, auth start, and a very large set of feature controller factories.

`AccountContext` is the per-account protocol exposing `sharedContext`, `account`, `engine`, live location/fetch/prefetch/download managers, upload and purchase managers, cached group calls, animation/cache resources, configuration streams, and methods for chat/call actions.

Owner/source-of-truth notes:

- Cross-account state and factory implementation: `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/SharedAccountContext.swift:139`.
- Protocol contract for shared app services: `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1321`.
- Protocol contract for per-account services: `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1641`.
- New feature controller factory methods often belong in `SharedAccountContextImpl`, but feature state should usually remain in Core/engine/Postbox or a feature module.

### Root UI And Navigation

`TelegramRootController` owns the main root tabs/controllers: contacts, calls, chat list, settings, root compose/camera/story flows, and tab/root controller updates. `NavigationController` in `Display` is a custom container, not vanilla UIKit navigation. It manages view controller stacks, overlays, modal containers, master/detail layouts, global overlay containers, minimized controllers, status bar/layout, and transitions. `ViewController` in `Display` is the base app UI abstraction over display nodes and navigation/presentation hooks.

Owner/source-of-truth notes:

- Root tab/controller composition: `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/TelegramRootController.swift:202`.
- Push/pop/replace stack operations: `/Users/defendend/workshop/telegram-ios/submodules/Display/Source/Navigation/NavigationController.swift:1485`.
- Base controller lifecycle/presentation: `/Users/defendend/workshop/telegram-ios/submodules/Display/Source/ViewController.swift:85`.

### UI Protocols And Feature Factories

`AccountContext` contains many UI protocol and factory definitions rather than concrete UI implementations: `TelegramRootControllerInterface`, `ChatController`, `ChatListController`, `ComposeController`, `AttachmentFileController`, camera/media/story screens, premium/stars/gifts/business/settings factories, and navigation params. Concrete implementations are spread across `submodules/TelegramUI`, `ChatListUI`, `SettingsUI`, `TelegramCallsUI`, and many feature submodules.

Owner/source-of-truth notes:

- Chat contract: `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/ChatController.swift:1075`.
- Chat list contract: `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/ChatListController.swift:12`.
- Feature factory implementation catalog: `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/SharedAccountContext.swift:1933` and below.
- If a factory exists in `SharedAccountContextImpl`, still inspect the target feature module for state and UI implementation.

### Account, Auth, And State

`AccountManager`/`AccountManagerImpl` manage account records, shared data, access challenge data, notices, stored login tokens, current account selection, and ValueBox-backed tables. `Account.swift` defines both `UnauthorizedAccount` and `Account`. Authorized `Account` owns Postbox, Network, account manager, state manager, contact sync, call session manager, pending message/story/update managers, peer input/presence, notification autolock, storage settings, cache eviction, network state/type, important tasks, and account setup/reset routines.

Owner/source-of-truth notes:

- Account metadata and current-record state: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/AccountManager/AccountManagerImpl.swift:33`.
- Authorized runtime: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:1150`.
- Unauthorized/login runtime: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:68`.
- Authorization UI lives under `/Users/defendend/workshop/telegram-ios/submodules/AuthorizationUI/Sources`; Core auth functions surfaced in AST search under `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Authorization.swift`.

### Engine API Layer

`TelegramEngine` is the high-level Core API facade over an `Account`. It groups feature APIs: secure ID, nearby peers, payments, peers, auth, account data, stickers, localization, themes, messages, privacy, calls, history import, contacts, resources, resolve, data, ordered lists, item cache, notices, and preferences. `TelegramEngineUnauthorized` provides a smaller unauthenticated surface.

Owner/source-of-truth notes:

- For app feature work, prefer `context.engine.<domain>` over direct network/Postbox edits when such API exists.
- `TelegramEngine` itself is a facade; domain implementations live in subdirectories under `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/TelegramEngine/`.

### Storage And Views

`Postbox` is the storage transaction and view layer. The public facade exposes transactions, message/history/chat list views, peer views, unread counts, holes, operation log, preferences, ordered lists, item cache, search, story state, and more. Internally, `PostboxImpl` is ValueBox-backed and owns many tables: peer, message history, chat list, media, read state, preferences, ordered lists, notice, story, pending message actions, operation logs, etc.

Owner/source-of-truth notes:

- Public storage facade: `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:4609`.
- Internal storage tables and transaction machinery: `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:1590`.
- Feature persistence often flows through engine/domain methods into Postbox transactions; avoid editing UI-only caches as source of truth.

### Network And Protocol Boundary

`Network.swift` wraps MTProto/MtProtoKit integration. It defines `NetworkInitializationArguments` with API ID/hash/app version/languages/VOIP layer/app data/verification/encryption parameters. `Network` owns datacenter, context, MTProto instance, request service, proxy state, connection status, worker/download/upload helpers, request APIs, retry logic, and network speed limited events.

Owner/source-of-truth notes:

- Initialization arguments: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:429`.
- Network facade: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:804`.
- Protocol schema/generated API boundary is the `TelegramApi` submodule and `MtProtoKit`; confirm generation/build flow before hand-editing protocol objects.

### Platform Integration

The main plist confirms:

- URL schemes: `telegram`, `tg`, app-specific URL scheme, `ton`.
- Privacy strings for camera, contacts, Face ID, location, microphone, motion, photo library, Siri.
- User activity intents: `INSendMessageIntent`, `RemindAboutChatIntent`.
- Background modes: `audio`, `fetch`, `location`, `remote-notification`, `voip`.

Extensions:

- Share extension plist uses `NSExtensionPointIdentifier` `com.apple.share-services` and principal class `ShareRootController`.
- Notification service extension plist uses `com.apple.usernotifications.service` and principal class `NotificationService`.

Owner/source-of-truth notes:

- App runtime handling for these declarations is mostly in `AppDelegate`, `SharedAccountContextImpl`, `Telegram/Share`, `Telegram/NotificationService`, and Siri/Watch folders.
- Entitlements/provisioning are build-configuration driven; do not infer final capabilities only from checked-in plist.

### Native, Generated, Runtime Boundaries

Important non-Swift or generated/runtime zones:

- `/Users/defendend/workshop/telegram-ios/submodules/MtProtoKit` for MTProto Objective-C/C++ runtime.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramApi` for generated Telegram API objects.
- `/Users/defendend/workshop/telegram-ios/submodules/Postbox` plus `sqlcipher` dependency for storage.
- `/Users/defendend/workshop/telegram-ios/submodules/TgVoipWebrtc`, `TelegramVoip`, `third-party/webrtc`, `submodules/ffmpeg`, `rlottie`, `lottie-ios`, `boringssl`, `libvpx`, `libjxl`, etc. for media/calls/native runtime.
- `LegacyComponents`, `LegacyUI`, and Obj-C headers are real runtime bridges, not dead code by default.

## Start Here By Task Type

- App launch/lifecycle/push/URL/background session: start at `AppDelegate`, then confirm plist declarations.
- Cross-account settings/account switching/global factories: start at `SharedAccountContextImpl`.
- Per-account feature work: start at `AccountContext` protocol to identify service surfaces, then `TelegramEngine` domain or feature module.
- New screen or UI behavior: start at feature module/controller factory, then `ViewController`/`NavigationController` only for navigation/layout mechanics.
- Root tabs/chat list/settings composition: start at `TelegramRootController`.
- Chat behavior: start at `AccountContext/Sources/ChatController.swift` for contract, then concrete `submodules/TelegramUI/Sources/Chat/*` implementation localized by AST.
- Chat list behavior: start at `AccountContext/Sources/ChatListController.swift`, then `ChatListUI`.
- Server-backed state: start at `TelegramEngine.<domain>` and related files under `TelegramCore/Sources/TelegramEngine`, then `Network` only if request transport behavior matters.
- Local persistence/cache/view updates: start at `Postbox` public transaction/view APIs, then relevant table/view files.
- Auth/login: start at `UnauthorizedAccount`, `TelegramCore/Sources/Authorization.swift`, and `AuthorizationUI`.
- Notifications: start at `AppDelegate` token/notification handling and `Telegram/NotificationService`, then account/shared context notification token registration.
- Calls/VoIP: start at `TelegramCallsUI`, `TelegramVoip`, `TgVoipWebrtc`, and app delegate PushKit/CallKit hooks.
- Build/index/project graph: start at `BUILD.bazel`, `MODULE.bazel`, and the specific submodule `Package.swift`.

## Do Not Start Here / Common Traps

- Do not treat `Telegram/Telegram-iOS/Application.swift` as the app architecture owner; it is nearly empty.
- Do not infer ownership from `Controller` suffixes alone; many controllers are protocol contracts, factories, or UI shells around state owned elsewhere.
- Do not assume concrete feature ownership from `AccountContext` protocol alone. It often declares contracts/factories; implementation lives elsewhere.
- Do not bypass `TelegramEngine` and mutate `Network` or `Postbox` directly unless the task is truly transport/storage-level.
- Do not treat checked-in plist as complete entitlements truth. Provisioning/profile and build configuration matter.
- Do not confuse `third-party/td` with the main iOS app architecture unless a task explicitly crosses into TDLib.
- Do not ignore `LegacyComponents`/Obj-C modules; many bridges are still active.
- Do not reconstruct ownership from plist/build declarations alone; they show packaging and integration, not the runtime owner.

## Open Questions / Uncertainties

- Concrete implementations for every `AccountContext` factory were not exhaustively mapped; future agents should re-confirm per feature.
- The generated API/schema workflow for `TelegramApi` was identified as a boundary but not reconstructed.
- Entitlements/provisioning capabilities require build-input configuration or profiles beyond the plist snippets.
- `Telegram/Telegram-iOS` target composition under Bazel/Xcode generation was only confirmed from top-level `BUILD.bazel` and module files, not the full target graph.
- Watch, Siri, Widget, Share, and Notification extensions were identified as integration zones but not fully decomposed.
