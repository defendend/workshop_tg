# Telegram iOS Architecture

## Purpose

This map is reusable project architecture memory for future AI agents working in `/Users/defendend/workshop/telegram-ios`. Use it to choose the right owner and avoid noisy third-party/build-system subtrees, then re-confirm the current code path with AST. The iOS project is strongly modular: the app shell wires contexts, while feature UI, account state, protocol, persistence, and runtime engines live in separate submodules.

## Scope

Covered in depth: app delegate/lifecycle, shared/account contexts, navigation and root presentation, UI module pattern, TelegramCore/Postbox/network ownership, platform declarations, and native/generated boundaries.

Covered at map level only: every individual feature module, all Bazel rule internals, every generated API file, every third-party C/C++/WebRTC/FFmpeg subtree, watch/share/widget extension internals. Future agents should use AST from the iOS repo root to drill into the affected layer.

## Project Mental Model

Telegram iOS is a Bazel-oriented Swift/Objective-C/C++ monorepo-style app. The main flow is:

`AppDelegate` in `submodules/TelegramUI` owns process lifecycle, windows, push, URL/open handling, notification extensions integration, account manager bootstrap, and construction of shared/account contexts. `SharedAccountContext` is the app-wide facade for presentation factories and app services. `AccountContext` is the active per-account facade that exposes `Account`, `TelegramEngine`, managers, caches, current configuration, and UI factories. `TelegramRootController` is the root navigation controller that builds tabs/controllers. `Display.NavigationController` is the custom navigation stack, not UIKit navigation as the primary abstraction. `TelegramCore` owns account lifecycle, MTProto network wrapping, engine APIs, pending operations, state managers, generated API model usage, and sync logic. `Postbox` is the durable local store and reactive view/transaction system. `TelegramApi` is generated TL/API schema output. Native/runtime boundaries include MtProtoKit, FFmpeg, WebRTC/TgVoip, Lottie/Metal/graphics, and Objective-C legacy components.

## Repository / Build Layout

- `/Users/defendend/workshop/telegram-ios/Telegram` owns app targets, app/extension plist templates, notification service, share extension, watch, widget, Siri intents, and Bazel target declarations.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources` owns the app delegate, root controller, application context implementation, and top-level UI wiring.
- `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources` defines shared/account context interfaces, navigation factories, chat contracts, and app bindings.
- `/Users/defendend/workshop/telegram-ios/submodules/Display/Source` provides the custom `ViewController`, `NavigationController`, layout/status bar/overlay infrastructure, and AsyncDisplayKit integration.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources` owns core account, engine, network, sync, pending messages, state managers, and business/domain APIs.
- `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources` owns local persistence, transactions, views, indexes, message history, preferences, and media box integration.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramApi/Sources` is generated API/TL schema Swift output.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Components` and many sibling UI submodules own feature UI and reusable UI components.
- `/Users/defendend/workshop/telegram-ios/submodules/LegacyComponents`, `LegacyUI`, `MtProtoKit`, `TgVoipWebrtc`, `ffmpeg`, and `third-party` are important boundaries, but often not the source of truth for feature behavior.
- `/Users/defendend/workshop/telegram-ios/BUILD.bazel`, `/Users/defendend/workshop/telegram-ios/MODULE.bazel`, and `/Users/defendend/workshop/telegram-ios/Telegram/BUILD` own Bazel/module/app target wiring.

## Application Entry And Lifecycle

Primary app entry/lifecycle owner:

- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:216`
  - AST outline shows `AppDelegate` with window/mainWindow/nativeWindow, build config, account manager, shared context, auth context, push registry, background sessions, notification token promises, lifecycle methods, URL handling, notification handling, CallKit/PushKit paths, background upload/download, and update checks.

Important lifecycle functions from AST:

- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:323` application launch/setup entry.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:1898` `applicationWillResignActive`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:1925` `applicationDidEnterBackground`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:1972` `applicationWillEnterForeground`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2004` `applicationDidBecomeActive`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2033` `applicationWillTerminate`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2112` through `:2433` PushKit/VoIP notification handling.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2501` URL opening.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2839` and `:3027` user notification center callbacks.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2938` notification registration.

Platform declaration confirm:

- `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:195` declares URL schemes.
- `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:296` declares `NSUserActivityTypes`.
- `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:305` declares background modes (`audio`, `fetch`, `location`, `remote-notification`, `voip`).
- `/Users/defendend/workshop/telegram-ios/Telegram/BUILD:517` and `:522` generate APS/app-group entitlements.

## Navigation And Presentation

Navigation is custom and context-driven:

- `/Users/defendend/workshop/telegram-ios/submodules/Display/Source/Navigation/NavigationController.swift:148` defines `NavigationController`.
- `/Users/defendend/workshop/telegram-ios/submodules/Display/Source/Navigation/NavigationController.swift:1485` and nearby methods push/replace/pop/present controllers.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/TelegramRootController.swift:74` defines `TelegramRootController`, a root controller conforming to app root expectations.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/TelegramRootController.swift:202` builds root controllers/tabs.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/TelegramRootController.swift:786`, `:798`, and `:810` open chats/contacts/settings.
- `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1321` defines `SharedAccountContext`.
- `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1366` through `:1589` define navigation and controller factories.

Typical flow:

App lifecycle builds `SharedAccountContext` and active `AccountContext`. Root UI is a `TelegramRootController` inside the custom `NavigationController` framework. Feature code usually asks `context.sharedContext` for a factory (`makeChatController`, `makeSettingsController`, `navigateToChat`, `openResolvedUrl`, etc.) and pushes/presents the resulting `ViewController`. Do not assume a single Android-style activity/router owns feature routing.

## UI Architecture

UI is mostly Swift with custom `Display`/AsyncDisplayKit abstractions, component modules, feature UI submodules, and some legacy Objective-C components.

Important UI layers:

- `/Users/defendend/workshop/telegram-ios/submodules/Display/Source`
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources`
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Components`
- `/Users/defendend/workshop/telegram-ios/submodules/ChatListUI/Sources`
- `/Users/defendend/workshop/telegram-ios/submodules/ContactListUI/Sources`
- `/Users/defendend/workshop/telegram-ios/submodules/SettingsUI/Sources`
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCallsUI/Sources`
- `/Users/defendend/workshop/telegram-ios/submodules/LegacyComponents/Sources`

Rendering/state style is reactive and componentized. SwiftSignalKit signals, Postbox views, engine data items, `ViewController` subclasses, AsyncDisplayKit nodes, and component containers are common. Feature UI should usually consume `AccountContext` / `TelegramEngine` / Postbox views rather than directly managing global singletons.

## State, Data, And Domain Layer

App/account context:

- `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1321` defines `SharedAccountContext`.
- `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1641` defines `AccountContext`.
- `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1642` through `:1689` expose account, engine, managers, caches, app configuration, reactions/effects, premium/frozen state, and call methods.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:172` defines `SharedApplicationContext`.

Core account/domain:

- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:68` defines `UnauthorizedAccount`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:1150` defines `Account`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:1158` and `:1159` show `postbox` and `network` owned by the active account.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:1167` through `:1184` list major account managers: account manager, state manager, contacts, calls, view tracker, pending messages/stories/updates, media preupload, input activity, presence, notification autolock.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/AccountManager.swift:249` initializes account management.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/State/AccountStateManager.swift` and `UnauthorizedAccountStateManager.swift` own state/update management; confirm exact methods with AST for state tasks.

Engine/API facade:

- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/TelegramEngine` contains grouped engine methods by domain (`AccountData`, `Auth`, `Calls`, `Contacts`, etc.).
- AST search showed `TelegramEngine` widely used from UI modules through `context.engine` and `engine.data`.

## Network / API / Protocol Layer

Primary network owners:

- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:429` defines `NetworkInitializationArguments`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:804` defines `Network`.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:867` initializes the network wrapper.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:1086` and `:1158` expose request methods.
- `/Users/defendend/workshop/telegram-ios/submodules/MtProtoKit/PublicHeaders/MtProtoKit/MTContext.h:72` defines `MTContext`, the Objective-C MTProto context.
- `/Users/defendend/workshop/telegram-ios/submodules/MtProtoKit/PublicHeaders/MtProtoKit/MTContext.h:128` through `:159` expose datacenter/address/auth/environment methods.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramApi/Sources/Api0.swift:2` defines the generated `Api` namespace.

Typical request path:

Feature UI or domain code calls `context.engine.<domain>` or `account.network.request(...)`. `TelegramCore` translates to generated `Api.functions.*` requests, dispatches through `Network`, backed by `MtProtoKit`, and writes/reads state through `Postbox` transactions and views. Updates are processed by account state managers and synchronization code, not by UI modules directly.

## Persistence / Cache / Storage

Primary persistence:

- `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:5` defines `PostboxTypes`.
- `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:22` defines `Transaction`.
- `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:1449` opens a postbox.
- `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:1590` defines `PostboxImpl`.
- `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:1781` through `:1862` list many internal tables: peers, notifications, messages, media, chat list, thread indexes, preferences, ordered lists, notices, pending actions, story tables, ratings.
- `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:4609` defines public `Postbox`.
- `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:4684` and `:4696` expose transaction APIs.

Media/cache:

- `Account.postbox.mediaBox` appears in account setup and Postbox integration.
- `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1647` through `:1651` expose fetch/prefetch/downloaded media/wallpaper managers.
- Media/resource fetch implementations live under `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network` and multiple media UI/resource submodules. Confirm exact owner by AST from the resource type or manager.

## Platform Integration

Main app platform declarations:

- `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:195` URL schemes: `telegram`, `tg`, app-specific scheme, `ton`.
- `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:233` query schemes for external app integrations.
- `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:296` Siri/user activity types.
- `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:305` background modes.
- `/Users/defendend/workshop/telegram-ios/Telegram/BUILD:517` APS environment entitlement template.
- `/Users/defendend/workshop/telegram-ios/Telegram/BUILD:522` app group entitlement template.

Code owners:

- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2501` URL open path.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2938` and `:2949` notification registration.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:2112` through `:2433` PushKit/VoIP notification handling.
- `/Users/defendend/workshop/telegram-ios/Telegram/NotificationService/Sources/NotificationService.swift` owns notification service extension behavior.
- `/Users/defendend/workshop/telegram-ios/Telegram/Share` and `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Components/ShareExtensionContext` own share extension behavior.
- `/Users/defendend/workshop/telegram-ios/Telegram/SiriIntents` owns Siri intent integration.
- `/Users/defendend/workshop/telegram-ios/Telegram/Watch` and `/Users/defendend/workshop/telegram-ios/Telegram/WidgetKitWidget` own watch/widget targets.

Extension declarations:

- `/Users/defendend/workshop/telegram-ios/Telegram/NotificationService/Info.plist:23` declares a notification service extension with `NotificationService` principal class.
- `/Users/defendend/workshop/telegram-ios/Telegram/BUILD:951` through `:980` confirm share extension plist template and `ShareRootController`.

## Native / Generated / Runtime Boundaries

- `/Users/defendend/workshop/telegram-ios/submodules/TelegramApi/Sources/Api0.swift:2` and sibling `Api*.swift` files are generated Telegram API/TL schema output.
- `/Users/defendend/workshop/telegram-ios/build-system/SwiftTL/Sources` is a likely schema/tooling area; re-confirm before schema-generation tasks.
- `/Users/defendend/workshop/telegram-ios/submodules/MtProtoKit` is Objective-C MTProto runtime/persistence/transport support.
- `/Users/defendend/workshop/telegram-ios/submodules/TgVoipWebrtc` and `/Users/defendend/workshop/telegram-ios/third-party/webrtc` are call/WebRTC runtime.
- `/Users/defendend/workshop/telegram-ios/submodules/ffmpeg` and `/Users/defendend/workshop/telegram-ios/third-party` media/native libraries are runtime dependencies.
- `/Users/defendend/workshop/telegram-ios/submodules/LegacyComponents` and `LegacyUI` are Objective-C/legacy UI bridges; do not assume they own new Swift feature logic without AST evidence.
- `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/FlatBuffers` is a generated/native-ish storage/serialization boundary.

## Major Feature Areas

- Chat/dialogs: start with `SharedAccountContext.makeChatController`, `ChatController` contracts, `TelegramUI/Sources/Chat`, `TelegramUI/Components/Chat`, `ChatListUI`, `TelegramCore`, and `Postbox`.
- Root tabs/settings/contacts: start with `TelegramRootController`, `ChatListUI`, `ContactListUI`, `SettingsUI`, and related factories in `SharedAccountContext`.
- Sending/composer: start with `ChatController` contracts, chat components, `TelegramCore/Sources/PendingMessages`, `TelegramEngine` message APIs, and Postbox pending actions.
- Network/API: start with `TelegramEngine` domain method, then `Network.request` and generated `Api.functions.*`.
- Persistence/local state: start with `Postbox` transaction/view APIs and the specific table/view class reached by AST.
- Notifications/push: start with `AppDelegate`, `NotificationService`, account manager access from extensions, notification token registration, and platform plists/build entitlements.
- Calls/VoIP: start with `TelegramCallsUI`, `PresentationCallManager`, `TelegramVoip`, `AppDelegate` PushKit/CallKit paths, and `TgVoipWebrtc`.
- Media/resources: start with `AccountContext` media/fetch managers, TelegramCore network fetch resources, media UI components, then native FFmpeg/WebRTC only when AST reaches runtime code.
- Stories/camera/editor: start with factories in `AccountContext`, `TelegramRootController.openStoryCamera`, `CameraScreen`, `MediaEditor`, `TelegramUI/Components/Stories`.
- Extensions: start in `/Telegram/NotificationService`, `/Telegram/Share`, `/Telegram/SiriIntents`, `/Telegram/Watch`, or `/Telegram/WidgetKitWidget` only when the task is extension-specific.

## Module / Layer Cards

### App Delegate / Process Shell

- Responsibility: launch, windows, shared/auth/account context bootstrap, notifications, push, URL handling, background sessions, lifecycle transitions.
- Not responsible for: feature screen internals or Postbox table implementation.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:216`.
- Typical entry points: launch application function, lifecycle methods, PushKit callbacks, notification center callbacks, `openUrl`.
- Important downstream dependencies: `TelegramCore`, `Postbox`, `AccountContext`, `TelegramCallsUI`, `BuildConfig`, `UserNotifications`, `PushKit`.
- Change risk: very high; mistakes affect startup, push, calls, background behavior.
- How to confirm with AST: outline `AppDelegate.swift`; trace exact lifecycle/callback method usages.

### Shared Application / Account Context

- Responsibility: app-wide services, presentation factories, account contexts, media/location/call managers, app bindings, navigation factory methods.
- Not responsible for: low-level DB table definitions or MTProto socket internals.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1321`, `:1641`.
- Typical entry points: `context.sharedContext.*`, `context.engine`, `context.account`, factory methods such as `makeChatController`.
- Important downstream dependencies: `TelegramCore`, `Postbox`, UI modules, SwiftSignalKit.
- Change risk: high; central interface used by many modules.
- How to confirm with AST: outline `AccountContext.swift`; trace target factory/property usages.

### Root Controller / Tabs

- Responsibility: root tabs/controllers, root navigation actions, contacts/chats/settings root flows, camera/story opening from root.
- Not responsible for: account/network persistence implementation.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/TelegramRootController.swift:74`.
- Typical entry points: `addRootControllers`, `updateRootControllers`, `openChats`, `openContacts`, `openSettings`, `openStoryCamera`.
- Important downstream dependencies: `Display.NavigationController`, `ChatListUI`, `ContactListUI`, `SettingsUI`, `AccountContext`.
- Change risk: high for tab/root flows.
- How to confirm with AST: outline root controller; trace usages of `TelegramRootControllerInterface`.

### Display Navigation

- Responsibility: navigation stack, overlays, modals, minimized controllers, status bar/layout transitions, push/pop/present mechanics.
- Not responsible for: feature business logic.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/Display/Source/Navigation/NavigationController.swift:148`.
- Typical entry points: `pushViewController`, `replaceTopController`, `replaceControllers`, `popViewController`, `present`, `dismiss`.
- Important downstream dependencies: `ViewController` hierarchy and UI modules.
- Change risk: very high; affects all presentation.
- How to confirm with AST: outline `NavigationController.swift`; trace exact navigation method usages before edits.

### Feature UI Modules

- Responsibility: feature screens, list nodes, components, settings screens, chat UI, call UI, premium/stars/stories/etc.
- Not responsible for: generated API types or base storage engine.
- Important artifacts: `submodules/TelegramUI/Components`, `submodules/ChatListUI`, `submodules/SettingsUI`, `submodules/ContactListUI`, `submodules/TelegramCallsUI`.
- Typical entry points: `ViewController` subclasses, component containers, node classes, factory functions from `SharedAccountContext`.
- Important downstream dependencies: `AccountContext`, `TelegramEngine`, `Postbox`, `Display`.
- Change risk: medium to high depending on component sharing.
- How to confirm with AST: search/class exact screen/component; inspect imports; trace factory/usages.

### TelegramCore Account Layer

- Responsibility: account/unauthorized account setup, managers, state management, pending operations, network and postbox wiring.
- Not responsible for: root UI presentation mechanics.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:1150`, `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/AccountManager.swift:249`.
- Typical entry points: account setup/current account, pending message managers, state managers, contacts/call managers.
- Important downstream dependencies: `Postbox`, `Network`, `TelegramApi`, SwiftSignalKit.
- Change risk: very high; account/session/data consistency.
- How to confirm with AST: outline `Account.swift`; search exact manager/state symbol; trace usages.

### TelegramEngine / Domain API Facade

- Responsibility: domain-level operations grouped by feature area, engine data reads, API abstractions consumed by UI.
- Not responsible for: raw socket transport or UI layout.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/TelegramEngine`.
- Typical entry points: `context.engine.<domain>` and `TelegramEngine.EngineData.Item.*`.
- Important downstream dependencies: `Account`, `Network`, `Postbox`, generated `Api`.
- Change risk: high; API semantics and state writes are shared.
- How to confirm with AST: search exact engine method/type; trace callers from UI and downstream network/postbox usage.

### Network / MtProtoKit

- Responsibility: initialized network, MTProto request dispatch, datacenter/auth/environment, proxy/connection status, network usage.
- Not responsible for: feature-specific state decisions.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:804`, `/Users/defendend/workshop/telegram-ios/submodules/MtProtoKit/PublicHeaders/MtProtoKit/MTContext.h:72`.
- Typical entry points: `Network.request`, `requestWithAdditionalInfo`, `MTContext` datacenter/auth methods.
- Important downstream dependencies: `TelegramApi`, `Postbox`, account state managers.
- Change risk: very high; all API traffic.
- How to confirm with AST: outline Network; trace exact request path; enter MtProtoKit only from Network/Account evidence.

### Postbox Persistence

- Responsibility: durable local store, transactions, views, message history, peers, preferences, operation logs, media box, indexes.
- Not responsible for: UIKit/Display presentation.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:1590`, `:4609`.
- Typical entry points: `postbox.transaction`, `transactionSignal`, `peerView`, `messageHistoryView`, `preferencesView`.
- Important downstream dependencies: TelegramCore account/state managers and UI view subscribers.
- Change risk: very high; storage/view invariants are delicate.
- How to confirm with AST: outline target Postbox file/table/view; trace transactions and view subscribers.

### Generated API / Schema

- Responsibility: generated `Api` namespace, API object parsing/serialization, function namespaces.
- Not responsible for: business logic or UI ownership.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/submodules/TelegramApi/Sources/Api0.swift:2`, sibling `Api*.swift`.
- Typical entry points: `Api.functions.*`, `Api.<namespace>.*` types.
- Important downstream dependencies: `TelegramCore` network/domain methods.
- Change risk: high if hand-edited; generation route should be found first.
- How to confirm with AST: search exact generated type; trace usage to TelegramCore owner; locate generator before editing schema output.

### Platform / Extensions

- Responsibility: Info.plist, entitlements, app groups, notification service, share extension, Siri, Watch, WidgetKit, background modes.
- Not responsible for: main app feature UI unless extension-specific.
- Important artifacts: `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:195`, `/Users/defendend/workshop/telegram-ios/Telegram/BUILD:517`, `/Users/defendend/workshop/telegram-ios/Telegram/NotificationService/Info.plist:23`, `/Users/defendend/workshop/telegram-ios/Telegram/Share/Info.plist:23`.
- Typical entry points: extension principal classes, AppDelegate callbacks, platform API delegates.
- Important downstream dependencies: account manager, Postbox, shared app group paths.
- Change risk: high; provisioning/background/extension behavior can break late.
- How to confirm with AST: AST locate extension class; grep-confirm plist/build literal only after owner is known.

### Native / Legacy Runtime

- Responsibility: MTProto Objective-C, WebRTC/VoIP, FFmpeg/media, legacy Objective-C UI, graphics/animation engines.
- Not responsible for: most Swift feature ownership.
- Important artifacts: `submodules/MtProtoKit`, `submodules/TgVoipWebrtc`, `submodules/ffmpeg`, `submodules/LegacyComponents`, `third-party/webrtc`.
- Typical entry points: Swift/Obj-C bridge symbols reached from TelegramCore or UI call/media owners.
- Important downstream dependencies: platform frameworks and C/C++ libraries.
- Change risk: very high; bridge/runtime changes can be ABI/performance sensitive.
- How to confirm with AST: start from Swift/Obj-C owner, then trace into native boundary; avoid third-party examples/tests.

## Key Artifacts

| absolute path | role | layer | why future agents should care | confirm/read strategy |
|---|---|---|---|---|
| `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/AppDelegate.swift:216` | app delegate/lifecycle | app shell | launch, account bootstrap, push, URLs, background, calls | AST outline/imports, trace exact callback |
| `/Users/defendend/workshop/telegram-ios/submodules/TelegramUI/Sources/TelegramRootController.swift:74` | root navigation/tabs | UI/navigation | root tabs and root flow openings | AST outline/usages |
| `/Users/defendend/workshop/telegram-ios/submodules/Display/Source/Navigation/NavigationController.swift:148` | custom navigation | UI/navigation | push/pop/present/modal/overlay mechanics | AST outline target method |
| `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1321` | shared context interface | app/UI/domain boundary | main factory/service facade | AST outline, trace factory usages |
| `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/AccountContext.swift:1641` | account context interface | state/UI boundary | exposes account, engine, managers, caches | AST outline/usages |
| `/Users/defendend/workshop/telegram-ios/submodules/AccountContext/Sources/ChatController.swift:1075` | chat controller contract | UI/chat | common chat screen interface and state contracts | AST outline, trace implementation |
| `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:68` | unauthorized account | auth/state | login/unauthorized account runtime | AST outline/symbol tracing |
| `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/Account.swift:1150` | active account | account/domain | owns postbox/network/managers | AST outline target manager |
| `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Account/AccountManager.swift:249` | account management setup | account/session | records, account switching/logout/cleanup | AST outline/usages |
| `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/Network/Network.swift:804` | network wrapper | network/protocol | request dispatch and MTProto integration | AST outline exact request method |
| `/Users/defendend/workshop/telegram-ios/submodules/MtProtoKit/PublicHeaders/MtProtoKit/MTContext.h:72` | MTProto context | native/network | DC/auth/environment native boundary | AST outline/callers |
| `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:1590` | Postbox implementation | persistence | tables, indexes, transaction internals | AST outline target table/view |
| `/Users/defendend/workshop/telegram-ios/submodules/Postbox/Sources/Postbox.swift:4609` | public Postbox facade | persistence | public transaction/view APIs | AST outline/usages |
| `/Users/defendend/workshop/telegram-ios/submodules/TelegramApi/Sources/Api0.swift:2` | generated API namespace | generated/protocol | TL/API object and function namespaces | AST search exact generated type |
| `/Users/defendend/workshop/telegram-ios/submodules/TelegramCore/Sources/TelegramEngine` | engine facade | domain/API | feature-domain methods consumed by UI | AST search exact engine method |
| `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:195` | URL schemes | platform | deeplink scheme declarations | grep-confirm after AST owner |
| `/Users/defendend/workshop/telegram-ios/Telegram/Telegram-iOS/Info.plist:305` | background modes | platform | audio/fetch/location/remote-notification/voip | grep-confirm after AST owner |
| `/Users/defendend/workshop/telegram-ios/Telegram/BUILD:517` | APS entitlement template | platform/build | push provisioning output | grep-confirm build literal |
| `/Users/defendend/workshop/telegram-ios/Telegram/NotificationService/Info.plist:23` | notification service extension | extension/platform | push notification extension entry | grep-confirm plist, AST extension class |
| `/Users/defendend/workshop/telegram-ios/Telegram/Share/Info.plist:23` | share extension declaration | extension/platform | share extension entry/activation | grep-confirm plist, AST ShareRootController |

## Start Here By Task Type

### UI change

- Start here: exact feature module/controller/component found by AST; use `AccountContext` factory only if the screen construction changes.
- Confirm with AST: `search`/`class` exact component, `imports`, `usages`, `outline`.
- Confirm with grep only after AST: plist/resource/build literals only if UI target membership or extension declaration matters.
- Do not confuse with: `TelegramRootController` unless changing root tabs/navigation.
- Risk if you edit the wrong layer: shared component regressions or context factory mismatch.

### navigation/routing change

- Start here: `SharedAccountContext` factory/navigate method, `TelegramRootController`, and `Display.NavigationController`.
- Confirm with AST: trace `navigateToChat`, `openResolvedUrl`, `make*Controller`, push/present callers.
- Confirm with grep only after AST: URL schemes, universal/user activity plist entries.
- Do not confuse with: feature screen internal callbacks.
- Risk if you edit the wrong layer: broken root stack, modal/overlay behavior, or deep link handling.

### network/protocol change

- Start here: `TelegramEngine` domain method, generated `Api.functions.*` type, `Network.request`.
- Confirm with AST: search exact API type/method; trace from UI to engine to network.
- Confirm with grep only after AST: generated file names or build target literals if schema/build pipeline is involved.
- Do not confuse with: UI screen that only displays request result.
- Risk if you edit the wrong layer: request serialization/state update mismatch.

### state/data change

- Start here: `AccountContext`, `Account`, `TelegramEngine`, account state manager, or Postbox view/transaction owner.
- Confirm with AST: trace engine data item, state manager, Postbox transaction/view subscribers.
- Confirm with grep only after AST: literal preference keys or plist/app group paths.
- Do not confuse with: component-local presentation state.
- Risk if you edit the wrong layer: UI updates without durable/account consistency.

### persistence/cache change

- Start here: `Postbox` public transaction/view API, exact table/view class, media box/fetch manager for resource cache.
- Confirm with AST: outline Postbox/table/view file; trace transaction callers and subscribers.
- Confirm with grep only after AST: exact table/key names or generated storage config.
- Do not confuse with: `TelegramEngine` facade methods that delegate to Postbox.
- Risk if you edit the wrong layer: corrupted local store, broken reactive views, migration issues.

### notification/push/deeplink change

- Start here: `AppDelegate` notification/URL methods, `NotificationService`, `AccountManager`, platform plist/build declarations.
- Confirm with AST: trace callback method and extension class.
- Confirm with grep only after AST: `Info.plist`, `Telegram/BUILD` entitlement and extension keys.
- Do not confuse with: in-app notification UI components.
- Risk if you edit the wrong layer: only background/extension/provisioning behavior breaks.

### permission/platform integration change

- Start here: `AppDelegate`, `TelegramApplicationBindings`, Info.plist usage descriptions/background modes, extension owner if applicable.
- Confirm with AST: trace binding closure or platform delegate.
- Confirm with grep only after AST: exact plist keys, entitlements, app groups, extension identifiers.
- Do not confuse with: settings UI descriptions.
- Risk if you edit the wrong layer: App Store/provisioning/runtime permission failure.

### media/native/runtime change

- Start here: account media/fetch manager, TelegramCore network resource fetch, `TelegramCallsUI`/VoIP owner, then native submodule reached from AST.
- Confirm with AST: trace Swift/Obj-C bridge before entering FFmpeg/WebRTC/MtProtoKit.
- Confirm with grep only after AST: Bazel target/linker flags or resource names.
- Do not confuse with: third-party examples/tests under build-system or third-party.
- Risk if you edit the wrong layer: ABI/runtime/performance crash.

### build/generated/schema change

- Start here: `Telegram/BUILD`, root `BUILD.bazel`/`MODULE.bazel`, `TelegramApi` generated usage, SwiftTL/build-system only after AST/confirm points there.
- Confirm with AST: exact generated type and downstream engine usage.
- Confirm with grep only after AST: Bazel target names, plist templates, entitlement fragments.
- Do not confuse with: generated `Api*.swift` as human-authored source of truth.
- Risk if you edit the wrong layer: generated output overwritten or target/provisioning divergence.

## Do Not Start Here / Common Traps

- Do not start in `/Users/defendend/workshop/telegram-ios/third-party` for app features.
- Do not treat `/Users/defendend/workshop/telegram-ios/build-system/bazel-rules` examples/tests as Telegram app owners.
- Do not edit generated `TelegramApi/Sources/Api*.swift` before finding the generator/schema route.
- Do not assume `AppDelegate` owns feature UI because it imports many modules; most UI is behind `SharedAccountContext` factories and feature submodules.
- Do not assume UIKit `UINavigationController` patterns; `Display.NavigationController` is the custom owner.
- Do not bypass `TelegramEngine`/Postbox by adding UI-local network or persistence logic.
- Do not treat `LegacyComponents` as the owner of new Swift features unless AST traces the feature there.
- Do not use broad text search to reconstruct architecture. Use AST discovery, then grep-confirm only for plist/build/entitlement/literal declarations.

## Recommended Agent Workflow

1. Read this map.
2. Identify the affected layer: app delegate, shared/account context, root navigation, feature UI, TelegramCore, Network, Postbox, platform/extension, native, or build/generated.
3. Confirm current owners via AST from `/Users/defendend/workshop/telegram-ios`.
4. Trace callers/usages of the exact symbol, factory, generated API type, or Postbox view.
5. Before reading a large file, run `ast-index outline <file>` and read only narrow ranges.
6. Use grep only for literal confirmation: plist, entitlements, Bazel target names, URL schemes, extension identifiers, exact keys.
7. Produce an implementation plan with uncertainties, especially for changes crossing UI/context/engine/postbox/network/platform boundaries.

## Open Questions / Uncertainties

- The exact concrete implementations behind many `AccountContext` factories should be re-confirmed per feature because the interface file is intentionally broad.
- `TelegramEngine` domain ownership is mapped by folder/module, not exhaustively by every engine method. Future network/state tasks should trace exact method usages.
- Generated schema pipeline was not fully traced. Schema changes should locate SwiftTL/API generation before editing generated `Api*.swift`.
- Extension behavior was confirmed at declaration level; extension internals should be re-confirmed with AST from the specific extension class.
- Native/runtime subtrees are summarized. VoIP/media/performance tasks must trace from Swift/Obj-C boundary into exact native symbols.
- Bazel target ownership is broad. Build tasks should confirm exact target and macro paths with narrow grep after AST identifies the affected module.

## Appendix: Evidence Checklist

- Confirm `AppDelegate` callback owner before lifecycle, URL, push, background, or CallKit edits.
- Confirm `SharedAccountContext` / `AccountContext` factory or property before feature UI/domain edits.
- Confirm `TelegramRootController` only for root tabs/root actions.
- Confirm `Display.NavigationController` for stack/modal/overlay behavior.
- Confirm feature UI module by AST from exact screen/component symbol.
- Confirm `Account` / `AccountManager` / state manager path for account/session changes.
- Confirm `TelegramEngine` method and generated `Api` type for API changes.
- Confirm `Network.request` path before transport-level changes.
- Confirm `Postbox` transaction/view/table owner before persistence edits.
- Confirm plist/entitlement/background mode/extension declaration with grep only after AST locates the code owner.
- Confirm native/legacy boundary from Swift/Obj-C call site before editing native or legacy code.
- Confirm Bazel target/build ownership before generated/build/platform changes.
