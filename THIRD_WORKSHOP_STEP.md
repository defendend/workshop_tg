# Map-Guided Change Planning Runbook

Этот файл — операционная инструкция для агента.

Если пользователь просит:

- запустить третий этап воркшопа
- запустить третий workflow
- проверить, как агент использует architecture maps
- сделать map-guided planning
- "а теперь запусти третий этап"

то default action:

- запускать один map-guided AST-confirm thread
- цель thread: подготовить implementation plan для сложной cross-platform задачи
- агент должен сначала прочитать project architecture maps, затем подтвердить owners через AST, затем использовать grep только для literal confirmation

## Root

- workspace root: `/Users/defendend/workshop`
- Android root: `/Users/defendend/workshop/telegram-android`
- iOS root: `/Users/defendend/workshop/telegram-ios`

## Prerequisites

Перед запуском третьего этапа в workspace должны существовать или быть доступны агенту как контекст два документа:

- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`

Если этих файлов нет:

- не запускай третий этап как валидный
- сообщи пользователю, что сначала нужно завершить второй этап и сохранить/предоставить две project-level architecture maps

## Source Of Truth

Перед запуском thread агент обязан опираться на:

- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/AST_AGENT.md`
- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`

Architecture maps использовать как стартовую архитектурную память, но не как абсолютную истину. Все актуальные owners нужно подтвердить через AST.

## Зачем Нужен Третий Этап

Первые два этапа показывают цепочку:

1. `grep-only` vs `AST first, grep confirm`: почему grep-first агенту трудно восстановить ownership в большой кодовой базе.
2. Project architecture maps: как превратить AST-first discovery в reusable memory.
3. Map-guided change planning: как будущий агент использует эту memory для сложной задачи и не начинает с пустого grep-first блуждания.

Третий этап не пишет код. Он должен выдать planning artifact, который показывает:

- какие слои затронуты
- где настоящие owners
- какие entry points надо подтвердить
- где возможны ложные owners
- какие AST confirmation шаги нужны
- какие grep confirmation шаги разрешены
- какие риски и QA matrix

## Что Считается Валидным Третьим Этапом

1. Запускается ровно один локальный `project thread`.
2. Thread создан как:
   - `target.type = project`
   - `target.projectId = /Users/defendend/workshop`
   - `target.environment.type = local`
3. Thread получает title `Map Guided Video Warning Plan`.
4. Thread получает только один стартовый prompt.
5. После старта нельзя отправлять follow-up сообщения.
6. Стартовый prompt должен включать саму product/change задачу, но не должен подсказывать конкретные implementation files или owners.
7. Thread должен:
   - сначала читать две project architecture maps
   - определить затронутые слои
   - подтвердить актуальных owners через AST
   - использовать grep только после AST confirmation и только для literal details
   - не писать код
   - выдать implementation plan

Если после старта был хотя бы один follow-up message:

- прогон считать невалидным
- старый thread архивировать
- запускать третий этап заново с нуля

## Что Запускать

Поднять один thread:

1. `Map Guided Video Warning Plan`

Параметры:

- `target.type = project`
- `target.projectId = /Users/defendend/workshop`
- `target.environment.type = local`
- `thinking = high`

## Правило Невмешательства

После создания thread:

- не отправлять follow-up prompt
- не уточнять scope
- не подсказывать маршруты
- не подсказывать файлы или классы
- не просить "добери еще"
- не ускорять принудительным "заканчивай сейчас"
- только ждать финальный ответ

Разрешено:

- читать статус thread
- читать финальный ответ
- архивировать невалидный thread

## Product Task Для Третьего Этапа

Сложная задача специально выбрана так, чтобы grep-first агенту было трудно:

```text
Нужно спроектировать изменение: при попытке начать видеозвонок или включить камеру в уже активном звонке показывать единый privacy/safety warning, если у пользователя включен новый флаг `video_call_privacy_warning_enabled`.

Warning должен срабатывать:
1. для исходящего private video call;
2. для включения видео внутри уже начатого private voice call;
3. для включения камеры в group/video chat;
4. для screen sharing / screencast, если он идет через тот же video/media path;
5. на Android и iOS.

Warning не должен ломать:
- incoming audio call accept;
- обычный audio-only outgoing call;
- group voice chat без камеры;
- уже существующие permission prompts;
- CallKit/Telecom incoming flow;
- native WebRTC/tgcalls runtime initialization.

Не пиши код. Подготовь implementation plan.
```

Почему задача хороша для воркшопа:

- затрагивает private outgoing, in-call video enable, group video, screencast/screen sharing
- требует отличать UI entry, preflight/gating, state owner, runtime/native owner и permissions
- на Android и iOS ownership model различается
- grep-first легко утонет в `video`, `camera`, `call`, `permission`
- architecture maps должны помочь агенту быстрее выбрать слой и owners

## Стартовый Prompt: Map Guided Video Warning Plan

```text
Работаем в `/Users/defendend/workshop`.

Перед любой работой сначала прочитай:
- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/AST_AGENT.md`
- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`

Используй `TELEGRAM_ANDROID_ARCHITECTURE.md` и `TELEGRAM_IOS_ARCHITECTURE.md` как стартовую архитектурную память, но не как абсолютную истину.
После чтения карт подтверди актуальных owners через AST.

Это третий workflow AI workshop: map-guided change planning.

Режим: `map-guided AST confirm, grep literal confirm`.

Критично:
- стартуй из root `/Users/defendend/workshop`
- не пиши код
- не редактируй файлы
- не используй grep/text search для architecture discovery
- AST нужен для подтверждения owners/callers/usages/current boundaries
- grep разрешен только после AST confirmation и только для literal details: manifest/plist/permissions/strings/entitlements/exact keys
- не используй `cd ... && ...`
- перед чтением большого файла сначала делай `ast-index outline <file>`

Задача:
Нужно спроектировать изменение: при попытке начать видеозвонок или включить камеру в уже активном звонке показывать единый privacy/safety warning, если у пользователя включен новый флаг `video_call_privacy_warning_enabled`.

Warning должен срабатывать:
1. для исходящего private video call;
2. для включения видео внутри уже начатого private voice call;
3. для включения камеры в group/video chat;
4. для screen sharing / screencast, если он идет через тот же video/media path;
5. на Android и iOS.

Warning не должен ломать:
- incoming audio call accept;
- обычный audio-only outgoing call;
- group voice chat без камеры;
- уже существующие permission prompts;
- CallKit/Telecom incoming flow;
- native WebRTC/tgcalls runtime initialization.

Нужно подготовить implementation plan, а не код.

Формат ответа:

# Map-Guided Implementation Plan

## Task Interpretation
Кратко переформулируй задачу и явно отдели policy/warning decision от permission prompt и native runtime.

## Architecture Map Signals Used
Что из Android/iOS project maps помогло выбрать слои и owners. Не пересказывай карты целиком.

## Affected Layers
Таблица: layer, Android impact, iOS impact, why it matters.

## Android Plan
### Owners
Какие owners нужно подтвердить через AST и за что они отвечают.
### Entry Points To Confirm
Private outgoing video, in-call enable video, group camera, screencast/screen sharing.
### Likely Change Points
Не больше 3-7 зон, с объяснением почему.
### Do Not Change
Какие runtime/native/UI-only места не должны быть source of truth для warning policy.
### AST Confirmation Needed
Конкретные AST routes: callers/usages/outline/symbol/module.
### Grep Confirmation Needed
Только literal confirmation: permissions, strings, manifest, exact keys.
### Risks
Что можно сломать, если выбрать неправильный слой.

## iOS Plan
### Owners
Какие owners нужно подтвердить через AST и за что они отвечают.
### Entry Points To Confirm
Private outgoing video, in-call enable video, group camera, screencast/screen sharing, CallKit/incoming exclusion.
### Likely Change Points
Не больше 3-7 зон, с объяснением почему.
### Do Not Change
Какие runtime/native/UI-only места не должны быть source of truth для warning policy.
### AST Confirmation Needed
Конкретные AST routes: callers/usages/outline/symbol/module.
### Grep Confirmation Needed
Только literal confirmation: plist, entitlements, strings, exact keys.
### Risks
Что можно сломать, если выбрать неправильный слой.

## Cross-Platform Consistency
Как сохранить одинаковую product semantics при разных ownership models Android/iOS.

## Test / Manual QA Matrix
Матрица: scenario, Android check, iOS check, expected result.

## Open Questions
Что нужно уточнить у product/engineering перед implementation.

## Final Recommendation
Короткий recommended implementation strategy.
```

## Что Делать После Завершения

После того как thread вернул финальный ответ:

1. Считать прогон валидным только если:
   - не было follow-up вмешательств
   - thread стартовал из `/Users/defendend/workshop`
   - thread создан как `project + local`
   - финальный ответ является implementation plan, а не кодом
   - агент явно использовал architecture maps как input
   - агент подтвердил owners через AST
   - grep использован только для literal confirmation или не использован вовсе
2. Проверить, что ответ содержит:
   - task interpretation
   - architecture map signals used
   - affected layers
   - Android plan
   - iOS plan
   - cross-platform consistency
   - QA matrix
   - open questions
   - final recommendation
3. Не превращать результат в benchmark.
4. Итоговый ответ пользователю должен коротко сообщить:
   - thread id
   - статус валидности
   - что implementation plan получен

## Как Объяснять Третий Этап На Воркшопе

Нарратив:

1. Первый этап показал, почему grep-first агент теряет architecture ownership.
2. Второй этап построил reusable project maps.
3. Третий этап показывает, как агент использует maps как рабочую память для сложной задачи.
4. Хороший агент не начинает с `video/camera/call` grep-а по всему repo.
5. Он читает карту, выбирает затронутые слои, подтверждает owners через AST и только потом добирает literals.

## Что Не Делать

- не запускать этот этап без project architecture maps
- не запускать grep-only benchmark
- не использовать `JUDGE_BENCHMARK.md` как source для thread prompt
- не писать код
- не подсказывать конкретные implementation files
- не просить агента менять файлы
- не считать карты абсолютной истиной без AST confirmation
