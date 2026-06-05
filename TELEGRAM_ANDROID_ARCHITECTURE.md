# Telegram Android Architecture

## Purpose

This map is reusable project architecture memory for future AI agents working in `/Users/defendend/workshop/telegram-android`. Use it to choose the right owner before editing code, then re-confirm the current implementation with AST. It is not a feature inventory and not a directory listing; it is a guide to the main layers, boundaries, and common wrong starting points.

## Scope

Covered in depth: application entry/lifecycle, navigation shell, UI ownership, account-scoped state, request/update flow, persistence, platform declarations, and native/generated boundaries.

Covered at map level only: individual feature screens, all app-flavor differences, every third-party/native codec subtree, and every generated TL type. Future agents should drill into those areas with AST only after identifying the affected layer.

## Project Mental Model

Telegram Android is a large Java/Kotlin Android app with a Java app shell and heavy native/runtime dependencies. The main flow is:

`ApplicationLoader` initializes process-wide app state, network availability, push/provider hooks, and account bootstrap. `LaunchActivity` owns the main activity lifecycle, intent/deeplink/share handling, tablet/multi-stack navigation, and permission dispatch. Feature screens are mostly `BaseFragment` subclasses presented through `ActionBarLayout` / `INavigationLayout` stacks. Account-scoped domain state is accessed through `AccountInstance`, whose getters route to singletons such as `MessagesController`, `MessagesStorage`, `ConnectionsManager`, `NotificationsController`, `FileLoader`, `SendMessagesHelper`, and feature managers. MTProto/API traffic goes through Java `org.telegram.tgnet` wrappers backed by JNI/native `TMessagesProj/jni/tgnet`. Message/dialog state is split between in-memory controller state, SQLite-backed `MessagesStorage`, and event propagation through `NotificationCenter`.

## Repository / Build Layout

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main` is the main application source set. It contains app Java, resources, and the primary `AndroidManifest.xml`.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger` owns app/domain/runtime services, account state, storage, notifications, media loading, controllers, helpers, and Android integration receivers/services.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui` owns the activity shell, fragments/screens, UI components, action bar/navigation primitives, stories, stars, bots, and feature presentation.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet` is the Java API/protocol bridge. It exposes `ConnectionsManager`, `TLRPC`, `TLObject`, request delegates, and native hooks.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni` owns native code and vendored native dependencies. Important subtrees include `tgnet`, `voip`, `ffmpeg`, `rlottie`, `boringssl`, `sqlite`, `opus`, `mozjpeg`, and third-party libraries.
- `/Users/defendend/workshop/telegram-android/TMessagesProj_App`, `TMessagesProj_AppStandalone`, `TMessagesProj_AppHuawei`, and `TMessagesProj_AppHockeyApp` are flavor/app wrapper modules with manifests/build files and some provider/updater/store-specific overrides. Do not treat them as the core architecture unless the task is flavor-specific.
- `/Users/defendend/workshop/telegram-android/TMessagesProj_AppTests` is test/instrumentation support, not production source of truth.
- `/Users/defendend/workshop/telegram-android/build.gradle`, `/Users/defendend/workshop/telegram-android/settings.gradle`, and per-module `build.gradle` files own Gradle/module wiring. Native build starts from `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/CMakeLists.txt`.

## Application Entry And Lifecycle

Primary app process owner:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/ApplicationLoader.java:51`
  - AST outline shows `ApplicationLoader` as the app class with `attachBaseContext`, `onCreate`, `postInitApplication`, provider factories, network state helpers, push initialization, update hooks, and pause/resume hooks.
  - Manifest confirm: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:98` declares `android:name="org.telegram.messenger.ApplicationLoader"`.

Primary UI/lifecycle entry:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:249`
  - AST outline shows `onCreate`, `handleIntent`, `onNewIntent`, `onActivityResult`, `onRequestPermissionsResult`, `onPause`, `onResume`, `onStart`, `onStop`, `onDestroy`, `presentFragment`, and navigation callbacks.
  - Manifest confirm: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:211` declares `org.telegram.ui.LaunchActivity`.

External/system entry points:

- Deeplinks and share intents flow into `LaunchActivity` via manifest filters around `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:219`, `:222`, `:257`, `:268`, and `:274`.
- Call UI entry is split between `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:292` (`CallsActivity`), `:387` (`VoIPPermissionActivity`), `:392` (`VoIPFeedbackActivity`), and services around `:518` and `:538`.
- Notification actions/receivers are declared around `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:398`, `:406`, `:414`, `:422`, `:438`, `:579`, `:583`, `:585`, `:591`, and `:593`.
- Background/system services are declared around `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:444`, `:456`, `:470`, `:476`, `:482`, `:488`, `:493`, `:499`, `:505`, `:512`, `:523`, `:626`, `:641`, and `:659`.

## Navigation And Presentation

Navigation is not Android Jetpack Navigation. AST evidence points to custom Telegram navigation:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:275` holds `mainFragmentsStack`, `layerFragmentsStack`, and `rightFragmentsStack`.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:292` holds `actionBarLayout`, `layersActionBarLayout`, and `rightActionBarLayout`.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:893` configures action bar layouts.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:6592` and nearby overloads present fragments.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/ActionBarLayout.java:96` is the stack/navigation container.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/BaseFragment.java` is the common screen abstraction. AST search showed more than a thousand usages.

Typical flow:

`LaunchActivity.handleIntent` or a UI callback resolves the destination, creates a `BaseFragment` subclass (for example dialogs/chat/settings/media/stories), then pushes/presents it through `INavigationLayout` / `ActionBarLayout`. Tablet/multi-pane behavior uses main/layer/right stacks in `LaunchActivity`.

## UI Architecture

UI is custom Android view code. Feature screens commonly extend `BaseFragment`; reusable widgets live under `org.telegram.ui.Components`; list rows live under `org.telegram.ui.Cells`; navigation chrome lives under `org.telegram.ui.ActionBar`.

Important UI zones:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/BaseFragment.java`
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/ActionBarLayout.java:96`
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:249`
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/DialogsActivity.java`
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java`
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/Components`
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/Cells`

Rendering/state update style is event-driven and imperative: controllers update in-memory state and post `NotificationCenter` events; fragments/components observe notifications, invalidate adapters/views, and call into account-scoped controllers/helpers.

## State, Data, And Domain Layer

Main owner pattern:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/AccountInstance.java:8` is a per-account facade.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/AccountInstance.java:29` through `:101` expose the major account-owned services/controllers.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/BaseController.java:6` is the base for many account-scoped controllers.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/UserConfig.java` owns per-account configuration and selected account state.

Core domain owners:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java:134` is the major in-memory domain coordinator for dialogs, users, chats, updates, server config, typing, read state, peer settings, folders, stories integration, and many feature-domain operations.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java:17228` and nearby update functions process MTProto update objects into local state.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/NotificationCenter.java` is the app-wide/account-wide event bus. AST search showed thousands of usages.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MediaDataController.java`, `ContactsController.java`, `LocationController.java`, `DownloadController.java`, `StatsController.java`, and feature controllers are accessed through `AccountInstance` or `BaseController`.

## Network / API / Protocol Layer

Request dispatch and update delivery:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:79` is the Java-side network/protocol owner.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:333` and nearby overloads expose `sendRequest`.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:793` handles native-delivered updates.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:958` through `:994` declare native method boundaries.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/tgnet/ConnectionsManager.cpp:47` is the native connection manager constructor and C++ runtime owner.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java` and `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/tl` are generated/protocol schema objects and namespaces.

Typical request path:

Feature/controller creates a `TLRPC` or `org.telegram.tgnet.tl.*` request, calls `ConnectionsManager.getInstance(account).sendRequest(...)`, receives callback on request completion, then updates `MessagesController`, `MessagesStorage`, `NotificationCenter`, or UI depending on the operation.

## Persistence / Cache / Storage

Primary persistent message/dialog store:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java:73`
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java:75` owns `storageQueue`.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java:76` owns the SQLite database.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java:115` declares DB version.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java:531` creates tables.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/DatabaseMigrationHelper.java` owns DB migrations.

Media/cache:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/FileLoader.java:33` owns file loading and media paths.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/ImageLoader.java` owns image/cache loading and posts file load notifications.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/FilePathDatabase.java` is a file path cache/database helper.

Preferences/config:

- `MessagesController` imports and exposes global/main/notification settings via `SharedPreferences`.
- `UserConfig`, `SharedConfig`, and `SharedPrefsHelper` are important for account and app preferences. Confirm ownership via AST before editing any setting key.

## Platform Integration

Manifest-declared integration:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:22` through `:85` declare broad permissions: network, foreground service, media, contacts, location, camera, audio, notifications, install packages, phone state, badges, billing, and background location.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:257` through `:279` confirm public web/deeplink schemes and custom schemes (`telegram.me`, `telegram.dog`, `t.me`, `tonsite`, `tg`).
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:597` and `:607` declare providers for file sharing and notification images.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:681` confirms Google Cast options provider metadata.

Code owners:

- `LaunchActivity` handles runtime permission results at `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:6691`.
- `ApplicationLoader` owns network connectivity and push provider bootstrap.
- `NotificationsController` owns Android notification channels/settings and notification behavior.
- `PushListenerController` is the likely push token/update owner; confirm with AST for push-specific work.
- `LocationController`, `MediaController`, `VoIPService`, `TelegramConnectionService`, widget providers/services, and receivers own OS-specific subflows.

## Native / Generated / Runtime Boundaries

- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:958` through `:994` is the Java native-method boundary for MTProto/network runtime.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/tgnet/ConnectionsManager.cpp` and `.h` own native connection scheduling, config, DC state, and request runtime.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/TgNetWrapper.cpp` exposes JNI wrappers.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java` and `org.telegram.tgnet.tl.*` are generated/protocol schema surfaces; do not hand-edit generated schema code unless the task is explicitly schema generation.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/voip` and `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/voip` bridge calls/VoIP.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/ffmpeg`, `rlottie`, `opus`, `mozjpeg`, `boringssl`, `sqlite`, and vendored WebRTC are runtime dependencies, not app feature owners.

## Major Feature Areas

- Chat/dialogs: start with `DialogsActivity`, `ChatActivity`, `MessagesController`, `MessagesStorage`, and `AccountInstance`. Confirm exact feature classes by AST from symbols/usages.
- Sending/composer: start with `ChatActivity`, `ChatActivityEnterView`, `SendMessagesHelper`, `MediaController`, `FileLoader`, and `MessagesStorage`.
- Media/cache: start with `FileLoader`, `ImageLoader`, `MediaController`, `FileLoadOperation`, `VideoEncodingService`, and native media subtrees only when runtime behavior is affected.
- Notifications/push: start with `NotificationsController`, `PushListenerController`, `ApplicationLoader`, manifest receivers/services, and `MessagesStorage` push-message persistence.
- Calls/VoIP: start with `VoIPService`, `VoIPController`, `VoIPHelper`, `GroupCallActivity`, `GroupCallMessagesController`, and `jni/voip`.
- Stories: start with `org.telegram.ui.Stories`, `StoriesController`, story upload service, and `MessagesStorage` story cache methods.
- Bots/web apps: start with `org.telegram.ui.bots`, `BotWebViewSheet`, `BotWebViewAttachedSheet`, `WebViewRequestProps`, and `MessagesController` app/deeplink handlers.
- Settings/account: start with `UserConfig`, `SharedConfig`, `MessagesController` settings accessors, and settings UI fragments.
- Payments/stars/premium/gifts: start with `org.telegram.ui.Stars`, `org.telegram.ui.Gifts`, `StarsController`, and `TL_stars` usages.
- Contacts/location: start with `ContactsController`, `LocationController`, manifest permissions, sync adapter services, and relevant UI.

## Module / Layer Cards

### App Shell / Process Lifecycle

- Responsibility: process-wide initialization, app context, provider selection, network availability, push bootstrap, global pause/resume state.
- Not responsible for: feature presentation details or durable message DB schema.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/ApplicationLoader.java:51`, manifest `:98`.
- Typical entry points: `onCreate`, `postInitApplication`, `initPushServices`, connectivity helpers.
- Important downstream dependencies: `ConnectionsManager`, `TLRPC`, push provider, `ForegroundDetector`, `LaunchActivity`.
- Change risk: high; mistakes affect startup, accounts, network, push, and every feature.
- How to confirm with AST: `ast-index outline TMessagesProj/src/main/java/org/telegram/messenger/ApplicationLoader.java`; `ast-index imports ...`; `ast-index usages ApplicationLoader`.

### Main Activity / Navigation Shell

- Responsibility: activity lifecycle, intents, deeplinks, shares, permissions, multi-stack navigation, tablet layout, passcode/update/TOS overlays.
- Not responsible for: low-level MTProto transport or DB storage implementation.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:249`, manifest `:211`.
- Typical entry points: `onCreate`, `handleIntent`, `onNewIntent`, `presentFragment`, `onRequestPermissionsResult`.
- Important downstream dependencies: `ActionBarLayout`, `BaseFragment`, `MessagesController`, `MessagesStorage`, `ConnectionsManager`, `NotificationsController`.
- Change risk: very high; wrong edits can break startup, routing, sharing, permissions, and tablet behavior.
- How to confirm with AST: outline `LaunchActivity`; trace callers/usages of target `handleIntent` or `presentFragment`.

### Navigation Framework

- Responsibility: fragment stacks, action bar navigation, bottom sheets, modal/layer/right stack behavior.
- Not responsible for: domain data loading or protocol calls.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/ActionBarLayout.java:96`, `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/BaseFragment.java`.
- Typical entry points: `LaunchActivity.setupActionBarLayout`, `BaseFragment.presentFragment`, `ActionBarLayout` stack operations.
- Important downstream dependencies: UI fragments and `INavigationLayout`.
- Change risk: high; breaks most screens.
- How to confirm with AST: `ast-index search "ActionBarLayout"`; `ast-index outline` target file; trace usages from a specific stack method.

### UI Screens And Components

- Responsibility: feature views, custom widgets, cells, adapters, bottom sheets, stories/stars/bots presentation.
- Not responsible for: generated protocol schema or native network runtime.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui`, `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/Components`, `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/Cells`.
- Typical entry points: `BaseFragment` subclasses, adapter callbacks, component constructors, `NotificationCenterDelegate`.
- Important downstream dependencies: `AccountInstance`, `MessagesController`, `FileLoader`, `NotificationCenter`.
- Change risk: medium to high depending on shared component usage.
- How to confirm with AST: locate class by AST `class --pattern`; inspect imports; trace `usages` before changing shared components.

### Account Context / Domain Facade

- Responsibility: per-account access to controllers, storage, network, notifications, config, media, contacts, downloads, sending.
- Not responsible for: UI stack ownership.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/AccountInstance.java:8`, `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/BaseController.java:6`.
- Typical entry points: `AccountInstance.getInstance(account)`, `BaseFragment.getMessagesController`, controller getters.
- Important downstream dependencies: all account-scoped controllers.
- Change risk: high; central wiring point.
- How to confirm with AST: outline `AccountInstance`; trace usages of the getter relevant to the task.

### Messages / Dialog Domain

- Responsibility: in-memory dialog/user/chat state, server config, updates, folders, read states, peer settings, many domain operations.
- Not responsible for: low-level socket scheduling or SQLite table implementation.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java:134`.
- Typical entry points: `processUpdates`, `loadDialogs`, `loadMessages`, `openByUserName`, server request callbacks.
- Important downstream dependencies: `ConnectionsManager`, `MessagesStorage`, `NotificationCenter`, `NotificationsController`.
- Change risk: very high; extremely broad surface.
- How to confirm with AST: outline first; use `ast-index symbol`/`usages` for specific method names; read narrow ranges only.

### Event Bus / UI Invalidation

- Responsibility: account/global notification broadcasting between controllers and UI.
- Not responsible for: durable persistence.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/NotificationCenter.java`.
- Typical entry points: `postNotificationName`, `addObserver`, `didReceivedNotification`.
- Important downstream dependencies: most fragments, `MessagesController`, `ImageLoader`, `NotificationsController`.
- Change risk: high; event name changes can silently break UI updates.
- How to confirm with AST: search exact event constant via AST, then trace usages; use grep only for literal event key confirmation if AST has already located the owner.

### Network / MTProto Java Bridge

- Responsibility: request dispatch, request lifecycle, app network state, DC/proxy/language/push registration, native callbacks.
- Not responsible for: UI decisions or feature-specific business state.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:79`, `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/tgnet/ConnectionsManager.cpp:47`.
- Typical entry points: `sendRequest`, `cancelRequest`, `onUpdate`, `onConnectionStateChanged`, native methods.
- Important downstream dependencies: `TLRPC`, native tgnet, `MessagesController`.
- Change risk: very high; breaks all API/network behavior.
- How to confirm with AST: outline Java and C++ connection manager; trace exact request method usages.

### Generated Protocol / TL Schema

- Responsibility: generated API object types, TL serialization/deserialization, namespace-specific request/response models.
- Not responsible for: feature ownership or UI state.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java`, `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/tl`.
- Typical entry points: constructing `TLRPC.*` or `TL_*` requests passed to `ConnectionsManager`.
- Important downstream dependencies: almost all controllers.
- Change risk: high if generated; prefer schema generation route.
- How to confirm with AST: search exact TL class; trace constructors/usages from feature controller.

### Persistence / SQLite Store

- Responsibility: local message/dialog/user/chat storage, DB schema, migrations, cache tables, pending tasks, search and widgets.
- Not responsible for: UI rendering or socket transport.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java:73`, `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/DatabaseMigrationHelper.java`.
- Typical entry points: `openDatabase`, `createTables`, `putMessages`, `loadMessages`, `updateDialogData`, queue methods.
- Important downstream dependencies: SQLite wrappers, `TLRPC`, `MessagesController`.
- Change risk: very high; DB migrations and storage queue mistakes corrupt state.
- How to confirm with AST: outline `MessagesStorage`; trace exact table/method; confirm schema literals only after AST.

### Media / Files / Cache

- Responsibility: media downloads/uploads, file paths, image cache, thumbnails, playback helpers, media encoding.
- Not responsible for: message domain semantics except as downstream of sending/loading.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/FileLoader.java:33`, `ImageLoader.java`, `MediaController.java`, `VideoEncodingService`.
- Typical entry points: `FileLoader.getInstance`, `loadFile`, `ImageLoader` callbacks, `SendMessagesHelper` media send paths.
- Important downstream dependencies: `MessagesStorage`, `NotificationCenter`, native ffmpeg/codec libraries.
- Change risk: high; performance and cache behavior are sensitive.
- How to confirm with AST: trace file/media class usages; only enter native codec tree if AST reaches it from Java boundary.

### Notifications / Push / Badges

- Responsibility: channels, notification rendering, push registration handling, reply/heard/dismiss actions, badge integration.
- Not responsible for: generic message storage except notification-related cached push messages.
- Important artifacts: `NotificationsController.java:100`, `PushListenerController.java`, manifest receivers/services around `:398` through `:593`.
- Typical entry points: push receive path, notification action receivers, `NotificationsController.getInstance`.
- Important downstream dependencies: `MessagesController`, `MessagesStorage`, `ApplicationLoader`, Android notification APIs.
- Change risk: high; background/platform behavior can regress silently.
- How to confirm with AST: search exact receiver/service class; outline before reading; confirm manifest declarations with grep.

### Platform Services / Receivers / Providers

- Responsibility: contacts sync, account authenticator, widgets, file provider, live location, foreground services, install referrer, boot receiver.
- Not responsible for: primary feature domain logic unless a specific service owns that background task.
- Important artifacts: manifest declarations at `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:444` through `:659`.
- Typical entry points: Android service/receiver callbacks.
- Important downstream dependencies: account/domain controllers.
- Change risk: medium to high; exported flags and permissions are security-sensitive.
- How to confirm with AST: locate class via manifest literal, then AST `class`/`outline`/`imports`.

### Native Runtime / VoIP / Codecs

- Responsibility: native MTProto, VoIP, media codecs, animation/rendering, crypto, SQLite, third-party runtime.
- Not responsible for: app feature ownership by default.
- Important artifacts: `/Users/defendend/workshop/telegram-android/TMessagesProj/jni`, `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/tgnet`, `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/voip`, `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/voip`.
- Typical entry points: JNI native methods, `VoIPService`, `VoIPController`, `TgNetWrapper.cpp`.
- Important downstream dependencies: Java wrappers and Android services.
- Change risk: very high; platform/runtime issues require focused native tracing.
- How to confirm with AST: start from Java native declaration or VoIP service; trace to C++ symbols via AST.

## Key Artifacts

| absolute path | role | layer | why future agents should care | confirm/read strategy |
|---|---|---|---|---|
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/ApplicationLoader.java:51` | app class/process bootstrap | lifecycle | process-wide initialization and network/push hooks | AST outline/imports, then narrow `sed` ranges |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:249` | main activity/navigation/intents | app shell/UI | central owner for lifecycle, deeplinks, permissions, navigation stacks | AST outline, trace target methods |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:98` | app declaration | platform | declares `ApplicationLoader` and platform capabilities | grep-confirm only after AST owner identified |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:211` | main activity declaration | platform | connects `LaunchActivity` to launcher/share/deeplink filters | grep-confirm literal declarations |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/BaseFragment.java` | screen base class | UI/navigation | most feature screens inherit or interact with it | AST class/usages, outline before reading |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/ActionBarLayout.java:96` | navigation stack container | UI/navigation | push/pop/present stack behavior | AST outline, trace stack methods |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/AccountInstance.java:8` | account facade | state/domain | maps account id to controllers/storage/network | AST outline and usages of target getter |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/BaseController.java:6` | controller base | state/domain | common access to account-scoped managers | AST outline/imports |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java:134` | main domain coordinator | state/domain | dialogs/users/chats/updates/config/read state | AST outline, symbol-specific tracing |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java:73` | SQLite-backed store | persistence | durable messages/dialogs/users/cache/pending tasks | AST outline, then narrow schema/method reads |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:79` | Java MTProto bridge | network/protocol | all request dispatch and native callbacks | AST outline/usages of send/update methods |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/tgnet/ConnectionsManager.cpp:47` | native MTProto runtime | native/network | C++ runtime behind Java bridge | AST outline/callers from JNI wrappers |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java` | generated protocol types | generated/protocol | schema objects for requests/responses/updates | AST search exact TL class; avoid broad edits |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/NotificationCenter.java` | event bus | state/UI | many UI invalidations and controller events | AST search constants/usages |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/NotificationsController.java:100` | notifications owner | platform/state | notification channels/settings/action behavior | AST outline/usages; manifest grep confirm |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/FileLoader.java:33` | media file loader | media/cache | file transfer/cache paths and media load events | AST outline/usages |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java` | outgoing message helper | state/network/media | sending, retry, unsent messages, media send | AST search/outline target methods |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/CMakeLists.txt` | native build entry | build/native | native runtime wiring | grep-confirm build literals after AST reaches native layer |
| `/Users/defendend/workshop/telegram-android/TMessagesProj/build.gradle` | main Android module build | build | app source/resources/native config | grep-confirm only for build/config tasks |
| `/Users/defendend/workshop/telegram-android/settings.gradle` | Gradle module inclusion | build | project module ownership/flavor wiring | grep-confirm only for build tasks |

## Start Here By Task Type

### UI change

- Start here: specific `BaseFragment`/component class found by AST, then `LaunchActivity` only if navigation/shell behavior changes.
- Confirm with AST: `class --pattern`, `search` exact class, `imports`, `usages`, `outline`.
- Confirm with grep only after AST: exact XML resource names, string keys, manifest activity declarations.
- Do not confuse with: `TMessagesProj_App*` flavor wrappers or native UI-adjacent libraries.
- Risk if you edit the wrong layer: shared component or navigation regressions across many screens.

### navigation/routing change

- Start here: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:1501`, `:6100`, `:6592`, `ActionBarLayout`, `BaseFragment`.
- Confirm with AST: trace caller/usages of `handleIntent`, `presentFragment`, destination fragment constructors.
- Confirm with grep only after AST: deeplink schemes/hosts and exact manifest intent filters.
- Do not confuse with: feature controllers that only open a screen but do not own routing.
- Risk if you edit the wrong layer: broken share/deeplink launch, tablet stacks, or back behavior.

### network/protocol change

- Start here: controller/helper that creates the request, then `ConnectionsManager.sendRequest` and exact `TLRPC`/`TL_*` types.
- Confirm with AST: search exact TL request class; trace usages to controllers; outline `ConnectionsManager` only for transport-level changes.
- Confirm with grep only after AST: generated schema/build file names if generation path is needed.
- Do not confuse with: UI callback classes that only consume the result.
- Risk if you edit the wrong layer: server request mismatch, stale local state, or native transport regression.

### state/data change

- Start here: `AccountInstance`, `MessagesController`, feature controller, `NotificationCenter` event, then storage if persistence is needed.
- Confirm with AST: trace exact state field/method usages and notification constants.
- Confirm with grep only after AST: literal preference keys or DB table names.
- Do not confuse with: UI cached display state in fragments.
- Risk if you edit the wrong layer: UI appears fixed but data/update consistency breaks.

### persistence/cache change

- Start here: `MessagesStorage`, `DatabaseMigrationHelper`, `FileLoader`, `ImageLoader`, or `FilePathDatabase` depending on data type.
- Confirm with AST: outline storage/cache class; trace exact methods to controller callers.
- Confirm with grep only after AST: table names, migration version literals, file path constants.
- Do not confuse with: `MessagesController` in-memory maps.
- Risk if you edit the wrong layer: DB corruption, missing migrations, cache leaks, broken offline behavior.

### notification/push/deeplink change

- Start here: `NotificationsController`, `PushListenerController`, `ApplicationLoader`, `LaunchActivity.handleIntent`, manifest declarations.
- Confirm with AST: trace receiver/service classes and controller callbacks.
- Confirm with grep only after AST: manifest actions, categories, schemes, notification channel literals.
- Do not confuse with: UI alert/bulletin components.
- Risk if you edit the wrong layer: platform integration breaks only in background/locked states.

### permission/platform integration change

- Start here: `LaunchActivity.onRequestPermissionsResult`, manifest permissions, specific controller needing permission.
- Confirm with AST: trace permission request helper/caller; outline affected service/receiver.
- Confirm with grep only after AST: `AndroidManifest.xml` permission/activity/service/provider literals.
- Do not confuse with: UI prompt text or settings screen labels.
- Risk if you edit the wrong layer: runtime permission loop, Play policy issue, or exported component bug.

### media/native/runtime change

- Start here: Java owner (`FileLoader`, `MediaController`, `VoIPService`, `VoIPController`, `ConnectionsManager` native declarations), then native subtree reached from AST.
- Confirm with AST: trace Java native methods/callers before entering C++.
- Confirm with grep only after AST: CMake/build flags, library names, codec literals.
- Do not confuse with: vendored third-party examples/tests.
- Risk if you edit the wrong layer: ABI/runtime crash or performance regression.

### build/generated/schema change

- Start here: Gradle files, `TMessagesProj/jni/CMakeLists.txt`, `org.telegram.tgnet` generated output, generation scripts if AST/confirm identifies them.
- Confirm with AST: identify Java/C++ symbols affected by generated output.
- Confirm with grep only after AST: exact build target names, schema file names, Gradle config.
- Do not confuse with: generated classes as source-of-truth.
- Risk if you edit the wrong layer: hand-edited generated code is overwritten or build variants diverge.

## Do Not Start Here / Common Traps

- Do not start from `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/voip/webrtc` for ordinary app features. It is huge vendored/runtime code.
- Do not start from `TMessagesProj_AppStandalone`, `TMessagesProj_AppHuawei`, or `TMessagesProj_AppHockeyApp` unless the task is flavor/store-specific.
- Do not treat `TLRPC.java` or `org.telegram.tgnet.tl` as business owners; they are protocol/generated surfaces.
- Do not start from manifest permissions for feature behavior. Manifest confirms platform declarations after AST finds the code owner.
- Do not assume `LaunchActivity` owns every UI change just because it imports many features; for screen internals, start at the target fragment/component.
- Do not edit `MessagesStorage` for display-only state unless AST proves persistence is required.
- Do not edit `NotificationCenter` constants or event names without tracing every usage.
- Do not use broad text search to reconstruct owners. Use AST discovery, then literal grep-confirm for manifests/build/config only.

## Recommended Agent Workflow

1. Read this map.
2. Identify the affected layer: app shell, navigation, UI, account state, network, persistence, platform, native, or build/generated.
3. Confirm current owners via AST from `/Users/defendend/workshop/telegram-android`.
4. Trace callers/usages of the exact symbol or class.
5. Before reading a large file, run `ast-index outline <file>` and read only narrow ranges.
6. Use grep only for literal confirmation: manifest, build files, exact resource/key/table names, service declarations, schemes/hosts.
7. Produce an implementation plan with uncertainties, especially if the task crosses state/network/storage/platform boundaries.

## Open Questions / Uncertainties

- The map identifies core owners, but individual feature ownership must be re-confirmed with AST because many features are embedded in very large files such as `MessagesController` and `LaunchActivity`.
- TL schema generation source was not traced to its generator; future schema tasks should locate the generation pipeline before editing generated protocol files.
- App flavor differences were only mapped at module/declaration level. Store-specific behavior should be re-confirmed in the relevant `TMessagesProj_App*` module.
- Native third-party subtrees are intentionally summarized. Runtime/codec/VoIP tasks must trace from Java/JNI boundary to exact C++ symbols.
- Resource XML ownership was not broadly enumerated; use `resource-usages` / `xml-usages` and only then grep-confirm exact resource files.

## Appendix: Evidence Checklist

- Confirm `ApplicationLoader` app declaration in manifest before startup/lifecycle edits.
- Confirm `LaunchActivity` route by AST outline and exact intent/deeplink manifest filters for routing tasks.
- Confirm destination screens are `BaseFragment` or component owners before UI edits.
- Confirm `AccountInstance` getter path for account-scoped controller changes.
- Confirm `MessagesController` method/field ownership with AST before touching state/update logic.
- Confirm `MessagesStorage` table/migration/storage queue path before persistence edits.
- Confirm `ConnectionsManager` request/update/native boundary before network edits.
- Confirm `TLRPC` / `TL_*` generated status before protocol/schema edits.
- Confirm `NotificationCenter` event constants and observer usages before changing events.
- Confirm `NotificationsController` plus manifest receiver/service declarations before notification changes.
- Confirm native C++ owner from Java native declaration before editing `jni`.
- Confirm build/module ownership with exact build files only after AST points to build/generated/native needs.
