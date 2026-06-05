# Telegram Android Architecture

Reusable project architecture memory for `/Users/defendend/workshop/telegram-android`. Do not treat this as a feature map.

## Scope

- Project root: `/Users/defendend/workshop/telegram-android`
- Primary app source: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main`
- Product wrappers: `/Users/defendend/workshop/telegram-android/TMessagesProj_App*`
- Native/runtime boundary: `/Users/defendend/workshop/telegram-android/TMessagesProj/jni`

## High-Level Shape

Android is a large Java-first app module with native runtime islands. `TMessagesProj/src/main` contains the primary application, UI, state, networking facade, persistence, media, notifications, and platform integration. Product wrapper modules (`TMessagesProj_App`, `TMessagesProj_AppHuawei`, `TMessagesProj_AppHockeyApp`, `TMessagesProj_AppStandalone`) customize build/store behavior around that core. JNI trees provide MTProto networking, VoIP/WebRTC, media codecs, sqlite, rlottie, boringssl, ffmpeg, opus, and other native dependencies.

The app is not architected as many Gradle feature modules. Future agents should think in terms of package/layer ownership inside the main source tree and per-account singleton owners rather than Gradle module boundaries.

## Key Artifacts

| Area | Artifact | Why it matters |
|---|---|---|
| Build modules | `/Users/defendend/workshop/telegram-android/settings.gradle:1` | Declares the main library module, product app wrappers, Huawei/Hockey/Standalone variants, and tests. |
| Core Gradle config | `/Users/defendend/workshop/telegram-android/TMessagesProj/build.gradle:1` | Main Android library, dependencies, compile/target SDK, NDK/CMake bridge, build types. |
| Android manifest | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:1` | Permissions, `ApplicationLoader`, `LaunchActivity`, aliases, services, receivers, widgets. |
| App process owner | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/ApplicationLoader.java:51` | `Application` subclass; process lifecycle, providers, push bootstrap, network availability. |
| Store/product override | `/Users/defendend/workshop/telegram-android/TMessagesProj_AppStandalone/src/main/java/org/telegram/messenger/ApplicationLoaderImpl.java:36` | Overrides update, package install, SMS jobs, and product-specific hooks. |
| Main Android shell | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:249` | Primary Activity; intent/deep-link/share dispatcher and root navigation host. |
| Account aggregate | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/AccountInstance.java:8` | Per-account accessor for controllers, storage, network, notifications, media/file helpers. |
| UI base | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/BaseFragment.java:74` | Base screen abstraction; lifecycle, dialogs, navigation helpers, per-account service accessors. |
| Navigation stack | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/ActionBarLayout.java:96` | Fragment stack container, transitions, predictive back, overlays, sheet navigation. |
| Main state owner | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java:134` | Huge per-account domain controller for peers, dialogs, updates, remote config, server actions. |
| Local persistence | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java:73` | SQLite-backed message/dialog/cache/state store and storage queue. |
| MTProto Java facade | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:79` | Java facade over native MTProto; request dispatch, callbacks, connection state, proxy/DNS. |
| MTProto native runtime | `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/tgnet/ConnectionsManager.cpp:47` | Native connection manager implementation discovered via AST search. |
| Account config | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/UserConfig.java:24` | Per-account identity, selected account, auth flags, config load/save, local user state. |
| Event bus | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/NotificationCenter.java:25` | Global and per-account event IDs, observers, delayed/postponed notifications. |
| Send pipeline | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java:123` | Message send/edit/retry/media upload orchestration and delayed send queues. |
| File/media fetch | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/FileLoader.java:33` | File upload/download queues, media directories, path resolution, progress callbacks. |
| System notifications | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/NotificationsController.java:100` | Push/message notification state, channels, unread counts, badge, reply actions. |
| Media runtime | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/MediaController.java:136` | Playback, recording, gallery scans, video conversion, media save/export. |
| VoIP native/UI boundary | `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/voip/VoIPController.java:28` and `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/voip/libtgvoip/VoIPController.h:182` | Java/native call control boundary. |

## Layer Cards

### Build And Product Wrappers

`settings.gradle` declares `TMessagesProj` as the shared core and separate wrapper app modules. `TMessagesProj/build.gradle` applies `com.android.library`, sets SDK 35 / min SDK 21, enables multidex, uses CMake under `TMessagesProj/jni/CMakeLists.txt`, and carries dependencies for Firebase messaging/config, Play services, billing, maps/location/wear, MLKit, Recaptcha, Markwon, and native build support.

Owner/source-of-truth notes:

- Build flags and dependency versions start in `/Users/defendend/workshop/telegram-android/TMessagesProj/build.gradle:1`.
- Store-specific behavior may live in wrapper modules, especially `ApplicationLoaderImpl` classes. Do not assume `TMessagesProj/src/main` is the only runtime code.
- Native C/C++ boundaries are under `/Users/defendend/workshop/telegram-android/TMessagesProj/jni`.

### App Entry And Lifecycle

`ApplicationLoader` is the `android:name` application class from the manifest. Its outline shows `attachBaseContext`, provider factories, `postInitApplication`, `onCreate`, foreground/background activity callbacks, push setup, Play Services checks, network state helpers, app update hooks, and custom update methods.

`LaunchActivity` is the main `singleTask` shell. Its outline shows `onCreate`, `handleIntent`, `onNewIntent`, account switching, passcode/TOS/update gates, fragment presentation, permission results, lifecycle hooks, and a very broad import surface.

Owner/source-of-truth notes:

- Process init: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/messenger/ApplicationLoader.java:190`.
- Main Activity lifecycle/navigation/integration: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:384`.
- Intent/deep-link/share routing starts around `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java:1501`, with many specialized helpers below.

### Navigation And UI Architecture

UI screens generally extend `BaseFragment`, not AndroidX Fragment. `BaseFragment` owns fragment view lifecycle, ActionBar creation, dialogs, sheet stack, preview/menu/bubble modes, and convenience accessors into account controllers (`getMessagesController`, `getConnectionsManager`, `getMessagesStorage`, etc.).

`ActionBarLayout` is the custom navigation container. It manages fragment stacks, push/pop/replace, transitions, predictive back, modal/sheet-ish behavior, theme animations, overlays, and keyboard/window insets. `LaunchActivity` has `mainFragmentsStack`, `layerFragmentsStack`, `rightFragmentsStack`, plus `actionBarLayout`, `layersActionBarLayout`, and `rightActionBarLayout`, so tablet/multi-pane behavior lives in the Activity/navigation layer.

Owner/source-of-truth notes:

- New screen lifecycle: start with `BaseFragment.createView`, `onFragmentCreate`, `onResume`, `onPause`, and accessor methods.
- Stack behavior: start with `ActionBarLayout.presentFragment`, `addFragmentToStack`, `closeLastFragment`.
- App-level routing or external intents: start with `LaunchActivity`, but move feature logic into the relevant fragment/controller once localized.

### Per-Account State And Data Flow

`AccountInstance` is the clearest indexable aggregator for per-account singletons. It exposes `MessagesController`, `MessagesStorage`, `ContactsController`, `MediaDataController`, `ConnectionsManager`, `NotificationsController`, `NotificationCenter`, `LocationController`, `UserConfig`, `DownloadController`, `SendMessagesHelper`, `SecretChatHelper`, `StatsController`, `FileLoader`, and `FileRefController`.

`UserConfig` stores selected account/current user/auth and local account flags. `MessagesController` is the broad server/domain state owner: dialogs, peers, updates queues, remote app config, unread/read state, feature limits, folders, stories, calls, admin actions, and many server requests. `MessagesStorage` owns local database tables, migrations, queues, message/dialog caches, pending tasks, and read state persistence.

Owner/source-of-truth notes:

- If a feature is per-account and not purely UI, inspect `AccountInstance` to identify the correct controller/storage/helper.
- If changing protocol-backed state, search AST from `MessagesController` or feature-specific controller classes, then confirm exact `ConnectionsManager.sendRequest` call sites.
- If changing local persistence, start with `MessagesStorage` and any helper-specific storage class before touching UI caches.

### Networking And Protocol Boundary

`ConnectionsManager.java` wraps native MTProto and exposes `sendRequest`, request callbacks, cancellation, request binding, connection state, proxy settings, DNS resolution, push connection flags, and native methods. Its native implementation lives under `TMessagesProj/jni/tgnet`. Protocol objects are in `org.telegram.tgnet.TLRPC` and `org.telegram.tgnet.tl.*` imports discovered from `LaunchActivity` and controller files.

Owner/source-of-truth notes:

- Java request API: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:312`.
- Native bridge methods: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java:958`.
- Native implementation: `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/tgnet/ConnectionsManager.cpp:47`.

### Eventing And UI Updates

`NotificationCenter` is a custom event bus, with many integer event IDs. It has per-account and global instances, observer groups, delayed posts, debouncing, animation/heavy-operation gating, and lifecycle helpers to listen from views.

Owner/source-of-truth notes:

- Do not add ad hoc callbacks before checking whether a `NotificationCenter` event already exists.
- Event IDs near the top of `NotificationCenter` are the discoverable vocabulary for UI update propagation.
- Use AST `usages <eventName>` or `refs <eventName>` after identifying an event constant.

### Send, File, Media, Notifications

`SendMessagesHelper` owns sending/editing/retry queues, delayed messages, upload preparation, media preparation, and send request execution. `FileLoader` owns upload/download queues, media directories, streaming loads, file path resolution, progress callbacks, and file database access. `MediaController` owns local media scanning, audio/video playback, recording, waveform/video conversion, and gallery export/save. `NotificationsController` owns push message display state, notification channels, unread counts, badge, popup/reply handling, stories notifications, and notification settings sync.

Owner/source-of-truth notes:

- Message sending starts in `SendMessagesHelper`, not in chat UI.
- File existence/progress/path questions start in `FileLoader`, then branch into media-specific UI.
- Playback/recording starts in `MediaController`; system notification behavior starts in `NotificationsController`.

### Platform Integration

The manifest confirms a large Android integration surface:

- Permissions include internet/network, foreground service types, audio, media, contacts/accounts/sync, overlay, phone, boot, biometrics, camera, billing, install packages, notifications, background location.
- `ApplicationLoader` is the app class.
- `LaunchActivity` handles main launch, send/share intents, Telegram URLs (`telegram.me`, `telegram.dog`, `t.me`), `tg`, `tonsite`, and profile MIME routes.
- Services include account authenticator, contacts sync adapter, keep-alive, notifications, video encoding, story uploading, importing, location sharing, VoIP, music player, and Telecom connection service.
- Receivers include boot/app start, install referrer, wear reply, live location stop, popup reply, notification callbacks/dismiss, media buttons, widgets, and share/custom tabs.

Owner/source-of-truth notes:

- Integration declarations: `/Users/defendend/workshop/telegram-android/TMessagesProj/src/main/AndroidManifest.xml:22`, `:98`, `:211`, `:444`.
- Runtime behavior for a manifest component often lives in `org.telegram.messenger` or `org.telegram.ui`, not beside the manifest declaration.

### Generated, Native, Runtime Boundaries

Important non-Java boundaries:

- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/tgnet` for native MTProto.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/voip` and `org.telegram.messenger.voip` for calls/WebRTC/libtgvoip.
- `/Users/defendend/workshop/telegram-android/TMessagesProj/jni/ffmpeg`, `opus`, `rlottie`, `boringssl`, `sqlite`, `mozjpeg`, `exoplayer`, `third_party` for native/media/runtime dependencies.
- `org.telegram.tgnet.TLRPC` and `org.telegram.tgnet.tl.*` are protocol object boundaries; do not hand-edit generated protocol surfaces without confirming generation rules.

## Start Here By Task Type

- App launch, process init, push bootstrap, global providers: start at `ApplicationLoader`, then product wrapper `ApplicationLoaderImpl` if behavior differs by build.
- Deep link, share sheet, external intent, app alias: start at `AndroidManifest.xml`, then `LaunchActivity.handleIntent`.
- New screen or screen behavior: start with the screen class extending `BaseFragment`; inspect `BaseFragment`/`ActionBarLayout` only for lifecycle/navigation questions.
- Navigation stack, tablet split, predictive back, overlay/sheet behavior: start at `LaunchActivity` fields and `ActionBarLayout`.
- Server-backed feature state: start at `MessagesController` or a narrower controller discovered via `AccountInstance`; confirm `ConnectionsManager.sendRequest` and TL objects.
- Local DB/cache/read state: start at `MessagesStorage`; use AST outline before reading large ranges.
- Sending messages/media: start at `SendMessagesHelper`; for files branch to `FileLoader`; for local media capture/edit/playback branch to `MediaController`.
- Notifications/badges/replies: start at `NotificationsController`, then manifest receivers/services for platform entry.
- Per-account config/auth flags: start at `UserConfig`, then `AccountInstance`.
- Calls/VoIP: start at `org.telegram.messenger.voip` Java classes, `VoIPService` manifest entry, and `TMessagesProj/jni/voip`.
- Build/product variant issue: start at `settings.gradle`, wrapper module `build.gradle`, and any wrapper `ApplicationLoaderImpl`.
- Native crash/runtime issue: start with Java facade imports/usages, then matching `TMessagesProj/jni/*` owner.

## Do Not Start Here / Common Traps

- Do not treat `LaunchActivity` as the source of truth for feature state. It is a huge routing/navigation shell and integration hub.
- Do not treat Gradle modules as feature modules; most feature ownership is inside the main source tree.
- Do not edit `NotificationCenter` constants blindly; first inspect usages and expected arguments.
- Do not put network logic into fragments if a controller/helper already owns the request/state flow.
- Do not assume `MessagesController` is always the narrowest owner. It is central but may delegate to feature controllers such as stories, boosts, media data, topics, stars, etc.
- Do not confuse manifest declarations with runtime ownership. Services/receivers usually delegate to controller/helper layers.
- Do not treat JNI/third-party directories as ordinary app source unless the issue crosses the Java/native boundary.
- Do not reconstruct ownership from manifest/build declarations alone; they show integration points, not the runtime owner.

## Open Questions / Uncertainties

- `MessagesController` is extremely broad; future agents should re-confirm narrower feature controllers before assigning ownership.
- Protocol/generation flow for `TLRPC` and `org.telegram.tgnet.tl.*` was not reconstructed in this pass.
- Resource XML ownership (`res/xml`, layouts, drawables) was only touched through manifest references and should be confirmed per task.
- Product wrapper differences beyond `ApplicationLoaderImpl` were not exhaustively mapped.
- Native CMake target structure was identified as a boundary via Gradle/JNI map but not fully decomposed.
