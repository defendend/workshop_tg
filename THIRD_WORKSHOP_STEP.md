# Third Workshop Step: Map-Guided vs Grep-First Planning

Этот файл - операционная инструкция для агента.

## Demo Outcome

К концу третьего этапа должны быть видны два implementation plan'а для одной задачи и operator/judge verdict: map-guided как strategy, grep-first как noisy baseline/literal appendix.

## Reference Artifacts

Operator-only replay artifacts from a completed clean run:

- `/Users/defendend/workshop/results/step-3/plan-map-guided.md`
- `/Users/defendend/workshop/results/step-3/plan-grep-first.md`
- `/Users/defendend/workshop/results/step-3/judge.md`

Не добавляй эти файлы в стартовые prompts и не используй их как source of truth для нового валидного прогона.

Если пользователь просит:

- запустить третий этап воркшопа
- запустить третий workflow
- запустить третий пример
- проверить, как architecture maps помогают агенту
- сравнить map-guided агента с grep-first агентом на сложной задаче
- "а теперь запусти третий этап"

то default action:

- запускать два независимых local project thread
- оба thread получают одну и ту же сложную cross-platform product task
- вариант A использует выбранные project architecture maps как основную архитектурную память
- вариант B не читает maps и начинает с grep-first discovery
- после завершения сравнить результаты как operator/judge, не отправляя follow-up сообщения агентам

## Root

- workspace root: `/Users/defendend/workshop`
- Android root: `/Users/defendend/workshop/telegram-android`
- iOS root: `/Users/defendend/workshop/telegram-ios`

## Prerequisites

По умолчанию третий этап использует seed/reference maps, закоммиченные в repo:

- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`

Если этих файлов нет:

- не запускай третий этап как валидный
- сообщи пользователю, что отсутствуют seed/reference maps

Если пользователь явно просит запустить третий этап на candidate maps, используй вместо seed maps:

- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE_CANDIDATE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE_CANDIDATE.md`

Candidate-вариант валиден только если оба candidate-файла уже существуют.

## Зачем Нужен Третий Этап

Первые два этапа показывают цепочку:

1. `grep-only` vs `AST first, grep confirm`: почему grep-first агенту трудно восстановить ownership в большой кодовой базе.
2. Project architecture maps: как превратить AST-first discovery в reusable memory.
3. Map-guided vs grep-first planning: как architecture memory меняет поведение агента на сложной задаче.

Третий этап не пишет код. Он должен показать разницу между двумя подходами к planning:

- map-guided агент начинает с architecture memory и выбирает затронутые слои по карте
- grep-first агент начинает с текстовых запросов вроде `video`, `camera`, `call`, `screencast` и должен сам выбрать owners из шума

Ожидаемая демонстрационная разница:

- map-guided агент быстрее отделяет product warning policy от permissions/native runtime
- grep-first агент получает больше шумных совпадений и чаще рискует принять runtime/UI/details за source of truth

## Что Считается Валидным Третьим Этапом

1. Запускаются ровно два локальных `project thread`.
2. Оба thread созданы как:
   - `target.type = project`
   - `target.projectId = /Users/defendend/workshop`
   - `target.environment.type = local`
3. Thread titles:
   - `Video Warning Plan - Map Guided`
   - `Video Warning Plan - Grep First`
4. Каждый thread получает только один стартовый prompt.
5. После старта нельзя отправлять follow-up сообщения.
6. Оба thread получают одинаковую product task.
7. Нельзя добавлять в prompts `JUDGE_BENCHMARK.md` или результаты прошлых прогонов.
8. Нельзя подсказывать конкретные implementation files/classes/owners в product task.

Если после старта был хотя бы один follow-up message:

- соответствующий thread считать невалидным
- старый thread архивировать
- запускать этот вариант заново с нуля

## Что Запускать

Поднять два thread:

1. `Video Warning Plan - Map Guided`
2. `Video Warning Plan - Grep First`

Параметры для обоих:

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
- только ждать финальные ответы

Разрешено:

- читать статус thread
- читать финальный ответ
- архивировать невалидный thread

## Product Task Для Обоих Агентов

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
- grep-first легко утонет в `video`, `camera`, `call`, `permission`, `screencast`, `webrtc`
- architecture maps должны помочь агенту быстрее выбрать слой и owners

## Стартовый Prompt A: Map Guided

Ниже prompt для default seed-map run. Если пользователь явно попросил candidate-map run, замени в prompt только два path:

- `TELEGRAM_ANDROID_ARCHITECTURE.md` -> `TELEGRAM_ANDROID_ARCHITECTURE_CANDIDATE.md`
- `TELEGRAM_IOS_ARCHITECTURE.md` -> `TELEGRAM_IOS_ARCHITECTURE_CANDIDATE.md`

```text
Работаем в `/Users/defendend/workshop`.

Перед любой работой сначала прочитай как обычные markdown-документы:
- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`

Важно про чтение файлов:
- Markdown-инструкции и architecture maps (`*.md`) читай напрямую как документы.
- Не вызывай `ast-index outline` для markdown-файлов, instruction files или architecture maps.
- Для валидного прогона агент должен реально прочитать эти markdown-файлы через tool/file reads, а не отвечать без вызовов инструментов.

Используй `TELEGRAM_ANDROID_ARCHITECTURE.md` и `TELEGRAM_IOS_ARCHITECTURE.md` как основную архитектурную память для planning.
Не запускай обязательный AST-confirm карты: цель варианта A - проверить, помогает ли готовая architecture map планировать без повторного discovery.

Это третий workflow AI workshop, вариант A: map-guided change planning.

Режим: `map-guided planning from architecture maps`.

Критично:
- стартуй из root `/Users/defendend/workshop`
- не пиши код
- не редактируй файлы
- не используй grep/text search для architecture discovery
- не используй `ast-index` для обязательного подтверждения карты или reconstruction
- если карта явно недостаточна для одного конкретного пункта плана, можно указать это как uncertainty вместо запуска discovery
- grep разрешен только для literal details: manifest/plist/permissions/strings/entitlements/exact keys
- не используй `cd ... && ...`
- не отправляй запросы обратно в этот thread; работай автономно
- в ответе обязательно упомяни 3-6 конкретных signals, реально взятых из карт, а не общие догадки

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
Какие owners карта считает релевантными и за что они отвечают.
### Entry Points To Confirm
Private outgoing video, in-call enable video, group camera, screencast/screen sharing.
### Likely Change Points
Не больше 3-7 зон, с объяснением почему.
### Do Not Change
Какие runtime/native/UI-only места не должны быть source of truth для warning policy.
### Map-Based Assumptions / Uncertainties
Какие выводы взяты из карты и что нужно будет подтвердить уже перед implementation, но не в этом planning-прогоне.
### Grep Confirmation Needed
Только literal confirmation: permissions, strings, manifest, exact keys.
### Risks
Что можно сломать, если выбрать неправильный слой.

## iOS Plan
### Owners
Какие owners карта считает релевантными и за что они отвечают.
### Entry Points To Confirm
Private outgoing video, in-call enable video, group camera, screencast/screen sharing, CallKit/incoming exclusion.
### Likely Change Points
Не больше 3-7 зон, с объяснением почему.
### Do Not Change
Какие runtime/native/UI-only места не должны быть source of truth для warning policy.
### Map-Based Assumptions / Uncertainties
Какие выводы взяты из карты и что нужно будет подтвердить уже перед implementation, но не в этом planning-прогоне.
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

## Стартовый Prompt B: Grep First

```text
Работаем в `/Users/defendend/workshop`.

Перед любой работой сначала прочитай:
- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/GREP_AGENT.md`

Важно: это третий workflow AI workshop, вариант B: grep-first/no-map baseline для сравнения с map-guided агентом.

Запрещено читать или использовать эти файлы:
- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`
- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE_CANDIDATE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE_CANDIDATE.md`
- `/Users/defendend/workshop/JUDGE_BENCHMARK.md`

Не используй AST-команды, MCP, IDE search или результаты прошлых прогонов.

Режим: `grep-first planning`.

Критично:
- стартуй из root `/Users/defendend/workshop`
- Android root: `/Users/defendend/workshop/telegram-android`
- iOS root: `/Users/defendend/workshop/telegram-ios`
- не пиши код
- не редактируй файлы
- используй только текстовый/file discovery: `rg --files`, `rg -n`, `find`, `ls`, `sed -n`
- не используй `ast-index`
- не используй `cd ... && ...`
- не подсказывай себе concrete implementation files из внешнего контекста; найди candidates через grep/text discovery
- не отправляй запросы обратно в этот thread; работай автономно

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

# Grep-First Implementation Plan

## Task Interpretation
Кратко переформулируй задачу и явно отдели policy/warning decision от permission prompt и native runtime.

## Grep Discovery Strategy
Какие текстовые запросы ты использовал, какие оказались шумными, как ты сузил candidates.

## Affected Layers
Таблица: layer, Android impact, iOS impact, why it matters.

## Android Plan
### Candidate Owners
Какие owners ты нашел через grep и почему считаешь их candidates.
### Entry Points To Confirm
Private outgoing video, in-call enable video, group camera, screencast/screen sharing.
### Likely Change Points
Не больше 3-7 зон, с объяснением почему.
### Do Not Change
Какие runtime/native/UI-only места не должны быть source of truth для warning policy.
### Remaining Uncertainty
Что невозможно уверенно доказать grep-only подходом.
### Risks
Что можно сломать, если выбрать неправильный слой.

## iOS Plan
### Candidate Owners
Какие owners ты нашел через grep и почему считаешь их candidates.
### Entry Points To Confirm
Private outgoing video, in-call enable video, group camera, screencast/screen sharing, CallKit/incoming exclusion.
### Likely Change Points
Не больше 3-7 зон, с объяснением почему.
### Do Not Change
Какие runtime/native/UI-only места не должны быть source of truth для warning policy.
### Remaining Uncertainty
Что невозможно уверенно доказать grep-only подходом.
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

После того как оба thread вернули финальные ответы:

1. Считать map-guided прогон валидным только если:
   - не было follow-up вмешательств
   - thread стартовал из `/Users/defendend/workshop`
   - thread создан как `project + local`
   - финальный ответ является implementation plan, а не кодом
   - агент явно использовал architecture maps как input
   - в thread есть реальные tool/file reads markdown-карт
   - агент полагался на maps для выбора слоев/owners, а не реконструировал ownership заново
   - агент не запускал обязательный AST-confirm карты
   - grep использован только для literal confirmation или не использован вовсе
2. Считать grep-first прогон валидным только если:
   - не было follow-up вмешательств
   - thread стартовал из `/Users/defendend/workshop`
   - thread создан как `project + local`
   - финальный ответ является implementation plan, а не кодом
   - агент не читал architecture maps
   - агент не использовал `ast-index`
   - агент явно показал grep discovery strategy и uncertainty
3. Сравнить:
   - кто точнее отделил warning policy от permissions/runtime
   - кто лучше нашел Android owners
   - кто лучше нашел iOS owners
   - кто меньше ушел в native/generated/third-party шум
   - где больше конкретики по entry points
   - где честнее обозначены uncertainties
   - какой результат лучше использовать как planning artifact
4. Итоговый ответ пользователю должен коротко сообщить:
   - оба thread id
   - статус валидности
   - кто сильнее и почему
   - 2-4 наблюдения для воркшопа

## Как Объяснять Третий Этап На Воркшопе

Нарратив:

1. Первый этап показал, что grep-first плохо восстанавливает architecture ownership.
2. Второй этап построил reusable project maps.
3. Третий этап проверяет, превращают ли maps сложную planning-задачу из поиска в навигацию.
4. Map-guided агент читает карту, выбирает затронутые слои и сразу планирует изменение, добирая только literals при необходимости.
5. Grep-first агент начинает с слов `video`, `camera`, `call`, `screencast` и вынужден вручную выбираться из шума.

## Что Не Делать

- не запускать этот этап без project architecture maps
- не запускать no-map AST baseline как основной контраст: он слишком сильный и размывает демонстрацию
- не использовать `JUDGE_BENCHMARK.md` как source для thread prompts
- не писать код
- не подсказывать конкретные implementation files
- не просить агентов менять файлы
- не превращать map-guided вариант обратно в AST-discovery/AST-confirm прогон
