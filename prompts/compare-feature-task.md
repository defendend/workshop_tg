Нужно сравнить реализацию одной и той же фичи в двух репозиториях:

- Android: `/Users/defendend/workshop/telegram-android`
- iOS: `/Users/defendend/workshop/telegram-ios`

Исследуемая фича: `<FEATURE_NAME>`

Подстановка `<FEATURE_NAME>`:

- Оператор воркшопа заменяет плейсхолдер `<FEATURE_NAME>` на имя конкретной фичи (например, `voice/video calls`, `chat folders`) непосредственно перед отправкой промта агенту.
- Подстановка делается только здесь, в `compare-feature-task.md`. Режимные промты `prompts/ast-only-agent.md` и `prompts/grep-only-agent.md` ничего не знают про конкретную фичу и не должны быть отредактированы под нее.
- Если ты видишь литеральную строку `<FEATURE_NAME>` без подстановки, считай это ошибкой запуска: не приступай к discovery, сообщи об ошибке и остановись.

Важное правило:

- Тебе известны только корни двух репозиториев и имя фичи.
- Тебе НЕ даны никакие path hints, имена файлов, имена классов, модули или заранее известные точки входа этой фичи.
- Локализовать фичу ты должен полностью самостоятельно, исходя только из разрешенного метода поиска.

Что нужно сделать:

1. Найти entry points этой фичи в Android и iOS.
2. Показать, как фича активируется из продукта:
   - экран
   - действие пользователя
   - публичный app-level trigger, если он есть
3. Найти основные артефакты по слоям:
   - UI / presentation
   - navigation / routing
   - orchestration / coordinator / manager / service layer
   - state owner / session owner
   - network / protocol / storage / cache
   - platform / OS integration
4. Для каждой платформы восстановить end-to-end execution path:
   - первый пользовательский trigger
   - первый кодовый hop
   - центральный orchestrator
   - как доходит до network/runtime side effects
   - как результат возвращается обратно в UI
5. Показать архитектурные boundaries:
   - какие модули / директории / типы образуют feature boundary
   - какой слой является source of truth
   - какие API или типы являются “узким горлом” фичи
6. Показать state/data model:
   - где хранится состояние
   - кто имеет право его менять
   - как state propagates между слоями
   - есть ли state machine / enum states / lifecycle phases
7. Показать side effects:
   - network / protocol calls
   - persistence / cache / preferences
   - background/runtime work
   - audio/video/media runtime
   - OS integration boundary
8. Показать gating / restrictions / error / fallback paths:
   - privacy / availability / feature gating checks
   - active-session conflicts
   - unsupported-version / offline / disabled-integration cases
   - recovery behavior, если он явно доказан
9. Если у фичи есть явные subflows или variants, выделить их отдельно.
   Примеры:
   - 1:1 vs group
   - voice vs video
   - outgoing vs incoming
   - in-app vs system-mediated flow
10. Дать короткую таблицу:
   - Android artifact
   - iOS artifact
   - слой
   - роль в системе
11. Отдельно перечислить 3-5 самых полезных находок, которые реально помогли локализовать архитектуру фичи.
12. Если есть места, где доказательств не хватает, явно пометить их как uncertainty, а не заполнять догадкой.
13. Отдельно, как optional appendix, можно добавить literal/integration evidence, если твой метод реально умеет это доказать:
   - permissions / manifest / plist / entitlements
   - deep links / push / system-event hooks
   - feature flags / strings / alerts / fallback UI
   - analytics / keys / intent-filters / notification declarations
   Этот слой не должен подменять core architecture comparison.
14. Для bridge-границ (JNI, Obj-C wrappers, libtgvoip/WebRTC wrappers, `.proto`, generated bindings) достаточно:
   - назвать boundary artifact
   - показать направление вызова или ownership
   - объяснить, какой слой уходит в native/runtime boundary
   Полный traversal по ту сторону boundary не обязателен.

Формат ответа (единый источник правды; режимные промты `ast-only-agent.md` и `grep-only-agent.md` не переопределяют этот формат, а лишь могут добавить mode-specific блок anomaly в самом конце):

- краткое summary
- Android
- iOS
- сравнение
- `Architecture Map`
- `State And Flow`
- `Side Effects And OS Integration`
- `Variants / Subflows`
- таблица артефактов
- `High-Value Findings`
- `Uncertainties`
- `Integration Appendix (Optional)`
- (опционально, mode-specific) `RAW_AST_ANOMALIES` или `RAW_SEARCH_ANOMALIES` — только если anomaly доказана по правилам соответствующего режимного промта

Если доказательств недостаточно, так и напиши. Не выдумывай связи, которые не подтверждены найденными файлами или символами.
