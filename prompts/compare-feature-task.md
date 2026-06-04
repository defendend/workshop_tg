Нужно сравнить реализацию одной и той же фичи в двух репозиториях:

- Android: `/Users/defendend/workshop/telegram-android`
- iOS: `/Users/defendend/workshop/telegram-ios`

Исследуемая фича: `<FEATURE_NAME>`

Важное правило:

- Тебе известны только корни двух репозиториев и имя фичи.
- Тебе НЕ даны никакие path hints, имена файлов, имена классов, модули или заранее известные точки входа этой фичи.
- Локализовать фичу ты должен полностью самостоятельно, исходя только из разрешенного метода поиска.

Что нужно сделать:

1. Найти entry points этой фичи в Android и iOS.
2. Показать, как фича активируется из продукта:
   - экран
   - действие пользователя
   - deep link / push / системное событие / настройка, если есть
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
   - background work / notifications
   - audio/video/media runtime
   - OS integration
8. Показать gating / restrictions / error / fallback paths:
   - permissions
   - privacy / feature flags / availability checks
   - active-session conflicts
   - unsupported-version / offline / disabled-integration cases
   - fallback UI / alerts / recovery behavior
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

Формат ответа:

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
- список использованных поисковых команд в точном виде

Если доказательств недостаточно, так и напиши. Не выдумывай связи, которые не подтверждены найденными файлами или символами.
