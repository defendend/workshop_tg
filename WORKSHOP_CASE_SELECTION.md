# Workshop Case Selection

Итог по трем чистым воркшоп-прогонам:

- `voice/video calls`
- `stories`
- `media picker / attachment flow`

Сравнение делалось по двум режимам на каждую фичу:

- `grep-only`
- `ast-only`

Все выводы ниже основаны только на чистых прогонах без follow-up вмешательств после стартового prompt.

## Чистые прогоны

### Voice/Video Calls

- `Calls Grep Run Clean`: `019e8043-ffe0-7a22-aeea-e3cf9c8ba528`
- `Calls AST Run Clean`: `019e8044-03d8-76b0-af53-f09c1024fabd`

### Stories

- `Stories Grep Run Clean`: `019e8044-083c-7f32-ab11-8b056be549db`
- `Stories AST Run Clean`: `019e8044-0bc5-71c2-a7fe-db0edd666aba`

### Media Picker / Attachment Flow

- `Media Picker Grep Run Clean`: `019e8044-0ff0-74e1-9e6a-374cbc08e7e5`
- `Media Picker AST Run Clean`: `019e8044-145f-7901-9488-393c94f44a19`

## Итоговый рейтинг кейсов

1. `voice/video calls`
2. `media picker / attachment flow`
3. `stories`

## Почему первым кейсом берем `voice/video calls`

- Фича достаточно сложная, чтобы `ast-only` реально показал преимущество над `grep-only`.
- При этом она еще не настолько расползается по объему, как `stories`.
- Хорошо демонстрирует различие архитектур:
  - Android: service-centric / helper-centric flow
  - iOS: manager/session/CallKit-centric flow
- Внутри одной фичи есть:
  - entry points
  - permissions
  - gating / restrictions
  - network/session lifecycle
  - state/data flow
  - OS integration
  - private call и group/video call ветки

## Почему `media picker / attachment flow` идет вторым

- Это самый чистый и управляемый кейс.
- Шума меньше, чем у calls и stories.
- Очень хорошо показывает UI/container architecture:
  - Android: `ChatAttachAlert`
  - iOS: `AttachmentController` / `MediaPickerScreen`
- Хорош как контрольный кейс, но уступает звонкам по системной глубине и эффектности для AST-воркшопа.

## Почему `stories` не берем первым

- Это самая широкая и шумная фича из трех.
- Она отлично подходит как stress-test.
- Но для первого воркшоп-кейса слишком велика:
  - viewer
  - recorder
  - upload pipeline
  - privacy
  - limits
  - premium/boost gates
  - cache/read state
  - reactions/archive/secondary flows
- Есть высокий риск, что первый прогон уйдет в объем вместо ясного сравнения методов.

## Рекомендация

Если нужен один главный первый кейс для воркшопа:

- брать `voice/video calls`

Если нужен более мягкий разогрев перед главным кейсом:

1. `media picker / attachment flow`
2. `voice/video calls`
3. `stories`

## Короткий вывод

- Лучший первый кейс: `voice/video calls`
- Самый управляемый кейс: `media picker / attachment flow`
- Самый тяжелый stress-test: `stories`

## Вывод по методу на первом кейсе

На кейсе `voice/video calls` зафиксирован такой результат:

- `ast-only` лучше как основной режим structural discovery
- `grep-only` лучше как слой подтверждения literal/details

При сравнении методов важнее:

1. structural coverage
2. completeness of Android vs iOS comparison

а не:

- меньшее число команд
- более быстрое завершение

Рабочая практическая схема после воркшопа:

1. `AST first`
2. `grep confirm`
