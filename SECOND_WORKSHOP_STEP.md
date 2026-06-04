# Project Architecture Maps Runbook

Этот файл — операционная инструкция для агента.

Если пользователь просит:

- запустить второй этап воркшопа
- запустить второй workflow
- собрать architecture map
- подготовить reusable context для будущих AI-агентов
- собрать карты проекта
- "а теперь запусти второй этап"

то default action:

- запускать один AST-first thread
- цель thread: собрать две полноценные project-level Markdown-карты:
  - `TELEGRAM_ANDROID_ARCHITECTURE.md`
  - `TELEGRAM_IOS_ARCHITECTURE.md`
- агент обязан создать эти два файла именно в корне workspace: `/Users/defendend/workshop`
- карты должны описывать архитектуру проектов в целом, а не архитектуру одной фичи
- результат должен быть reusable memory для будущих AI-агентов

Важно:

- это не shell-сценарий
- не пытайся запускать `./scripts/2-analyze.sh`
- второй этап не является benchmark-прогоном
- не запускай `grep-only` thread
- не сравнивай методы
- не собирай карту только по `voice/video calls`
- не используй выводы первого этапа как source of truth для discovery
- новый прогон второго этапа создается только через отдельный Codex `project + local` thread, описанный ниже

## Root

- workspace root: `/Users/defendend/workshop`
- Android root: `/Users/defendend/workshop/telegram-android`
- iOS root: `/Users/defendend/workshop/telegram-ios`

## Source Of Truth

Перед запуском thread агент обязан опираться на:

- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/AST_AGENT.md`

Если что-то в этом файле конфликтует с ними, сначала проверить конфликт, но для второго этапа этот файл задает конкретный сценарий запуска.

## Зачем Нужен Второй Этап

Первый этап воркшопа показывает проблему на конкретной фиче: grep-first агент может найти файлы, но не гарантирует architecture ownership.

Второй этап делает практический следующий шаг:

1. Запустить AST-first discovery на уровне проектов.
2. Составить отдельную reusable architecture map для Android.
3. Составить отдельную reusable architecture map для iOS.
4. Дать будущим AI-агентам project memory: где входы, слои, owners, boundaries, generated/native/runtime зоны, platform integration и типичные ловушки.

Главная идея второго этапа:

```text
AST-first project discovery + grep confirmation + project architecture memory
```

## Что Считается Валидным Вторым Этапом

1. Запускается ровно один локальный `project thread`.
2. Thread создан как:
   - `target.type = project`
   - `target.projectId = /Users/defendend/workshop`
   - `target.environment.type = local`
3. Thread получает title `Project Architecture Maps`.
4. Thread получает только один стартовый prompt.
5. После старта нельзя отправлять follow-up сообщения.
6. В стартовом prompt нельзя давать:
   - feature-specific hints
   - имена файлов реализации конкретной фичи
   - имена классов реализации конкретной фичи
   - entry points конкретной фичи
   - результаты предыдущих прогонов
7. Thread знает только:
   - workspace root
   - Android root
   - iOS root
   - обязательные root-instruction files
   - требуемую структуру итоговых Markdown-документов
8. Thread должен создать или обновить два Markdown-файла именно в корне workspace `/Users/defendend/workshop`:
   - `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
   - `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`
9. Финальный результат считается недостаточным, если он:
   - описывает только одну фичу
   - смешивает Android и iOS в одну карту без отдельных документов
   - просто перечисляет каталоги без ownership/boundaries
   - не содержит module/layer cards
   - не содержит agent usage guide
   - не объясняет, где будущему агенту начинать при разных типах задач
   - не содержит проверяемых evidence refs с абсолютными путями и, где возможно, line anchors

Если после старта был хотя бы один follow-up message:

- прогон считать невалидным
- старый thread архивировать
- запускать второй этап заново с нуля

## Что Запускать

Поднять один thread:

1. `Project Architecture Maps`

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
- не подсказывать файлы или классы из первого этапа
- не просить "добери еще"
- не ускорять принудительным "заканчивай сейчас"
- только ждать финальный ответ

Разрешено:

- читать статус thread
- читать финальный ответ
- архивировать невалидный thread

## Стартовый Prompt: Project Architecture Maps

```text
Работаем в `/Users/defendend/workshop`.

Перед любой работой сначала прочитай:
- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/AST_AGENT.md`

Используй их как source of truth.
Прочитай их молча: не пиши отдельные подготовительные апдейты про чтение инструкций, локальные skills, проверку HOME, индекс или bootstrap.

Это не benchmark-прогон и не grep-only сравнение. Это второй workflow для AI workshop: AST-first project discovery с целью собрать reusable project architecture memory для будущих AI-агентов.

Режим: `AST first, grep confirm`.

Цель: подготовить два полноценных Markdown-документа в финальном ответе:
- `TELEGRAM_ANDROID_ARCHITECTURE.md`
- `TELEGRAM_IOS_ARCHITECTURE.md`

Создай или обнови два Markdown-файла именно в корне workspace `/Users/defendend/workshop`:
- `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
- `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`

Не ограничивайся выводом markdown в финальном ответе: файлы должны реально появиться на диске именно по этим двум абсолютным путям.
Не клади карты внутрь `telegram-android`, `telegram-ios`, `prompts`, `scripts` или любой другой подпапки.

Критично:
- стартуй из root `/Users/defendend/workshop`
- анализируй Android и iOS отдельно
- Android project root: `/Users/defendend/workshop/telegram-android`
- iOS project root: `/Users/defendend/workshop/telegram-ios`
- не используй заранее известные feature-specific path hints, классы, модули или entry points из внешнего контекста
- не используй результаты прошлых прогонов как source of truth
- сначала сам найди архитектуру через AST discovery
- вызывай AST-команды actual shell command только напрямую как `ast-index <subcommand> ...`
- в Codex Desktop `HOME=/Users/defendend` уже задан через `shell_environment_policy`; actual shell command должна начинаться с `ast-index <subcommand> ...`, без inline `HOME=...`
- в отчете можно фиксировать logical command как `HOME=/Users/defendend ast-index <subcommand> ...`, но actual shell command не должна использовать inline env assignment

Жесткие правила discovery:
- не используй `rg`, `grep`, `findstr`, MCP, IDE search и любые текстовые поисковые механики для discovery, ownership analysis или reconstruction of architecture
- после того как AST уже локализовал core architecture, можно использовать только точечный confirm-search: `rg --files`, `rg -n`, `find`, `ls`, `sed -n`
- confirm-search разрешен только для literal/integration хвостов: manifests, plist, entitlements, build files, permissions, generated config, service declarations, URL schemes, intent-filters, exact strings/keys
- confirm-search должен быть ограничен repo roots `/Users/defendend/workshop/telegram-android` и `/Users/defendend/workshop/telegram-ios`
- не превращай confirm-search во второй discovery-проход
- не используй `cd ... && ...`
- если AST-команда шумит или падает, переходи на другой AST route, а не на текстовый fallback
- перед чтением большого файла сначала делай `ast-index outline <file>`
- используй любые полезные AST routes: `search`, `class`, `file`, `symbol`, `module`, `imports`, `callers`, `usages`, `outline`, а также при необходимости `xml-usages`, `resource-usages`, `swiftui`, `async-funcs`

Нужно собрать project architecture maps, а не фичевый обзор и не список директорий.

Качество важнее краткости. Это reference artifact для будущих AI-агентов, поэтому каждая карта должна помогать следующему агенту:
- понять основные слои проекта
- понять app entry / lifecycle / navigation
- понять UI architecture
- понять data/state/network/persistence layers
- понять platform integration
- понять generated/native/runtime boundaries
- понять, где искать owners для типовых задач
- понять, какие зоны не стоит путать с source of truth
- знать, что подтвердить через AST и что добрать через grep-confirm

Минимальные требования к каждой карте:
- минимум 12-20 key artifacts, если AST discovery нашел достаточно evidence
- module/layer cards для ключевых подсистем
- owner/source-of-truth notes для major areas
- отдельный `Start Here By Task Type`
- отдельный `Do Not Start Here / Common Traps`
- absolute paths для key artifacts; если точный range был прочитан, добавляй line anchor
- честные uncertainties: какие зоны future agent должен re-confirm

Если не получается собрать полноценную карту из-за tooling limits, так и напиши в `Open Questions / Uncertainties`, но не заменяй карту компактным обзором.

Финальный ответ должен иметь:

1. Короткое summary о том, что две карты собраны и записаны на диск.
2. Абсолютные пути созданных/обновленных файлов.
3. Краткую проверку, что оба файла содержат required sections.
4. `Tooling Notes` только если была существенная AST/tooling anomaly.

Не вставляй полный текст двух карт в финальный ответ, если файлы уже записаны. Финальный ответ должен быть кратким operator report.

Не добавляй benchmark verdict.
Не сравнивай с grep-only прогоном.
Не используй результаты прошлых прогонов как источник истины.

Структура каждого Markdown-документа:

# <Project Name> Architecture

## Purpose
Зачем эта карта будущим AI-агентам и как ее использовать.

## Scope
Что покрывает карта и что покрыто поверхностно.

## Project Mental Model
Короткая high-level модель проекта: app shell, lifecycle, navigation, UI, data/state, network, persistence, platform integration, generated/native boundaries.

## Repository / Build Layout
Главные части repo/build system. Не просто список директорий: объясни role and ownership.

## Application Entry And Lifecycle
App entry points, startup/lifecycle owners, account/session bootstrap, notification/deeplink/system-event entry points.

## Navigation And Presentation
Основные navigation/presentation owners и как UI flow обычно попадает в feature screens/controllers/components.

## UI Architecture
UI framework/patterns, major UI modules, component boundaries, rendering/state update style.

## State, Data, And Domain Layer
Где живут state owners, account/context/session abstractions, domain managers, feature state patterns.

## Network / API / Protocol Layer
Где живут API/protocol calls, request dispatch, update handling, generated schemas/bindings.

## Persistence / Cache / Storage
Основные storage/cache/database/media-cache layers и ownership.

## Platform Integration
Permissions, manifest/plist/entitlements, push/notifications, deep links, background services/modes, OS-specific bridges.

## Native / Generated / Runtime Boundaries
Generated code, JNI/Obj-C/C++/Swift wrappers, media/runtime engines, protobuf/TL/schema boundaries.

## Major Feature Areas
Карта крупных feature zones. Не углубляйся до полной фичевой архитектуры, но укажи owners/start points для будущих агентов.

## Module / Layer Cards
Для каждого крупного слоя или модуля:
- Responsibility
- Not responsible for
- Important artifacts
- Typical entry points
- Important downstream dependencies
- Change risk
- How to confirm with AST

## Key Artifacts
Таблица:
- absolute path
- role
- layer
- why future agents should care
- confirm/read strategy

## Start Here By Task Type
Практические инструкции:
- UI change
- navigation/routing change
- network/protocol change
- state/data change
- persistence/cache change
- notification/push/deeplink change
- permission/platform integration change
- media/native/runtime change
- build/generated/schema change
Для каждого пункта:
- Start here
- Confirm with AST
- Confirm with grep only after AST
- Do not confuse with
- Risk if you edit the wrong layer

## Do Not Start Here / Common Traps
Типичные ложные owners и зоны шума.

## Recommended Agent Workflow
Пошаговый workflow для будущих AI-агентов:
1. Read this map
2. Identify affected layer
3. Confirm current owners via AST
4. Trace callers/usages
5. Read narrow ranges only after outline
6. Use grep only for literal confirmation
7. Produce implementation plan with uncertainties

## Open Questions / Uncertainties
Честные границы расследования и зоны, которые future agent должен re-confirm.

## Appendix: Evidence Checklist
Checklist evidence, который future agents должны подтвердить перед изменениями.
```

## Что Делать После Завершения

После того как thread вернул финальный ответ:

1. Считать прогон валидным только если:
   - не было follow-up вмешательств
   - thread стартовал из `/Users/defendend/workshop`
   - thread создан как `project + local`
   - на диске существуют два полноценных Markdown-документа:
     - `/Users/defendend/workshop/TELEGRAM_ANDROID_ARCHITECTURE.md`
     - `/Users/defendend/workshop/TELEGRAM_IOS_ARCHITECTURE.md`
   - оба файла лежат в корне `/Users/defendend/workshop`, а не в подпапках проектов
2. Проверить, что оба документа содержат:
   - purpose/scope
   - project mental model
   - repository/build layout
   - application entry/lifecycle
   - navigation/presentation
   - UI architecture
   - state/data/domain layer
   - network/API/protocol layer
   - persistence/cache/storage
   - platform integration
   - native/generated/runtime boundaries
   - major feature areas
   - module/layer cards
   - key artifacts
   - start-here by task type
   - common traps
   - recommended agent workflow
   - uncertainties
   - evidence checklist
3. Не превращать результат во второй benchmark.
4. Итоговый ответ пользователю должен коротко сообщить:
   - thread id
   - статус валидности
   - что две project-level Markdown-карты собраны и записаны на диск
   - абсолютные пути файлов

## Как Объяснять Второй Этап На Воркшопе

Нарратив:

1. Первый этап показал проблему на фиче: grep-first агент может найти файлы, но не гарантирует architecture ownership.
2. Второй этап поднимает workflow на уровень проекта.
3. AST-first агент строит reusable project maps отдельно для Android и iOS.
4. Future agents больше не начинают с пустого grep-first блуждания по огромному repo.
5. Они читают карту проекта, определяют затронутый слой, подтверждают актуальность через AST и добирают literal details через grep.

## Как Использовать Карты В Будущих Agent Prompts

Пример follow-up prompt для будущих задач:

```text
Перед работой прочитай relevant project map:
- `TELEGRAM_ANDROID_ARCHITECTURE.md` для Android
- `TELEGRAM_IOS_ARCHITECTURE.md` для iOS

Используй карту как стартовую архитектурную память, не как абсолютную истину.
Определи затронутый слой: UI, navigation, state/data, network/protocol, persistence/cache, platform integration, native/runtime, build/generated.
Подтверди актуальных owners через AST callers/usages/outline.
Используй grep только для literal confirmation: manifest, plist, permissions, strings, entitlements, exact keys.
В финале дай implementation plan и uncertainties.
```

## Что Не Делать

- не запускать grep-only thread
- не запускать benchmark judge
- не использовать `JUDGE_BENCHMARK.md` как source для thread prompt
- не собирать только карту `voice/video calls`
- не смешивать Android и iOS в один документ
- не подсказывать агенту feature-specific owners из первого этапа
- не просить агента писать файлы вне `/Users/defendend/workshop`
- не принимать результат второго этапа, если карты выданы только в финальном ответе и не созданы как `.md` файлы в корне workspace
- не считать карты абсолютной истиной для будущих задач без AST confirmation
