# Voice/Video Calls Workshop Runbook

Этот файл — операционная инструкция для агента.

Если пользователь просит:

- запустить первый воркшоп-кейс
- прогнать основной кейс
- сравнить `grep-only` vs `ast-first-confirm` на первом кейсе

то default action:

- запускать кейс `voice/video calls`

## Root

- workspace root: `/Users/defendend/workshop`
- Android root: `/Users/defendend/workshop/telegram-android`
- iOS root: `/Users/defendend/workshop/telegram-ios`

## Source Of Truth

Перед запуском тредов агент обязан опираться на:

- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/GREP_AGENT.md`
- `/Users/defendend/workshop/AST_AGENT.md`

Если что-то в этом файле конфликтует с ними, сначала проверить конфликт, но для кейса `voice/video calls` этот файл задаёт конкретный сценарий запуска.

## Что считается валидным прогоном

1. `grep-only` запускается отдельным локальным `project thread`.
2. `ast-first-confirm` тоже запускается отдельным локальным `project thread`.
3. AST-thread валиден только если:
   - создан как `project + local`
   - стартовал из `/Users/defendend/workshop`
   - после старта не получил follow-up steering
4. Каждый запуск получает только один стартовый prompt.
5. После старта нельзя отправлять follow-up сообщения.
6. В стартовом prompt нельзя давать:
   - path hints
   - имена файлов
   - имена классов
   - entry points
   - scope hints
7. Прогон знает только:
   - workspace root
   - имя фичи
   - обязательные root-instruction files

Если после старта был хотя бы один follow-up message:

- прогон считать невалидным
- старые треды архивировать
- запускать заново с нуля

## Что запускать

Поднимать два thread:

1. `Calls Grep Run Clean`
2. `Calls AST Run Clean`

Параметры:

- `target.type = project`
- `target.projectId = /Users/defendend/workshop`
- `target.environment.type = local`
- `thinking = high`

## Правило невмешательства

После создания thread:

- не отправлять в них follow-up prompt
- не уточнять scope
- не подсказывать им маршруты
- не ускорять их принудительным “заканчивай сейчас”
- только ждать финальные ответы

Разрешено:

- читать статус thread
- читать финальные ответы
- архивировать невалидные треды

## Стартовый Prompt: Calls Grep Run Clean

```text
Работаем в `/Users/defendend/workshop`.

Перед любой работой сначала прочитай:
- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/GREP_AGENT.md`

Используй их как source of truth.
Прочитай их молча: не пиши отдельные апдейты про чтение инструкций, дочитывание диапазонов, `source of truth`, локальные skills/навыки/MCP или подготовительные проверки среды.

Это чистый воркшоп-прогон. После этого стартового prompt follow-up сообщений не будет. Никакие дополнительные подсказки по scope, путям, классам, entry points не даны и не будут даны.

Режим: `grep-only`.
Фича: `voice/video calls`.

Критично:
- стартуй из root `/Users/defendend/workshop`
- тебе известны только root workspace и имя фичи
- path hints, классы, модули, entry points не даны

Жесткие правила:
- разрешены только `rg --files`, `rg -n`, `find`, `ls`, `sed -n`
- запрещены `ast-index`, MCP, IDE search, bulk-чтение больших файлов
- если файл большой, сначала локализуй через `rg -n`, потом читай узкие диапазоны через `sed -n`
- не используй `cd ... && ...`
- не используй `|`, `&&`, `||`, `;`, process substitution или subshell вокруг разрешенных команд

Нужен сравнительный разбор Android vs iOS:
- entry points
- user flow end-to-end
- architecture / module boundaries
- central orchestrator / state owner
- state and data flow
- network / protocol / runtime side effects
- UI composition
- variants / subflows
- key differences между Android и iOS

Optional integration appendix:
- permissions / manifest / plist / entitlements
- deep links / push / system-event hooks
- feature flags / alerts / fallback UI
- bridge boundaries / native runtime handoff

Формат ответа:
- краткое summary
- Android
- iOS
- сравнение
- `Key Differences`
- `Architecture Map`
- `State And Flow`
- `Side Effects And OS Integration`
- `Variants / Subflows`
- таблица артефактов
- `High-Value Findings`
- `Uncertainties`
- `Integration Appendix (Optional)`
```

## Стартовый Prompt: Calls AST Run Clean

```text
Работаем в `/Users/defendend/workshop`.

Перед любой работой сначала прочитай:
- `/Users/defendend/workshop/WORKFLOW.md`
- `/Users/defendend/workshop/AST_AGENT.md`

Используй их как source of truth.

Это чистый воркшоп-прогон. После этого стартового prompt follow-up сообщений не будет. Никакие дополнительные подсказки по scope, путям, классам, entry points не даны и не будут даны.

Режим: `AST first, grep confirm`.
Фича: `voice/video calls`.

Критично:
- стартуй из root `/Users/defendend/workshop`
- тебе известны только root workspace и имя фичи
- path hints, классы, модули, entry points не даны
- вызывай AST-команды actual shell command только напрямую как `ast-index <subcommand> ...`
- в Codex Desktop `HOME=/Users/defendend` уже задан через `shell_environment_policy`; чтобы prefix allow сработал, исполняемая команда должна начинаться с `ast-index <subcommand> ...`, без inline `HOME=...`
- в отчете и `RAW_*` блоках фиксируй logical command как `HOME=/Users/defendend ast-index <subcommand> ...`, но не запускай shell-команду с inline env assignment

Жесткие правила:
- не используй `rg`, `grep`, `findstr`, MCP, IDE search и любые текстовые поисковые механики для discovery, ownership analysis или reconstruction of flow
- после того как AST уже локализовал core architecture, можно использовать только точечный confirm-search: `rg --files`, `rg -n`, `find`, `ls`, `sed -n`
- confirm-search разрешен только для literal/integration хвостов: manifest / plist / entitlements / permissions / CallKit / PushKit / notifications / service declarations / URL schemes / intent-filters / точечные strings, keys, alerts, fallback UI
- confirm-search должен быть ограничен repo roots `/Users/defendend/workshop/telegram-android` и `/Users/defendend/workshop/telegram-ios`
- не превращай confirm-search во второй discovery-проход по фиче
- не используй `cd ... && ...`
- если команда шумит или падает, переходи на другой AST route, а не на текстовый fallback
- если `ast-index` ведет себя странно, фиксируй это как факт прогонa, но не получай никаких внешних подсказок
- не используй `bash -c`, `zsh -lc`, `sh -c`, wrapper-скрипты, alias или любые другие обертки вокруг `ast-index`
- не запускай `ast-index`, если `HOME` не равен `/Users/defendend`
- не используй inline `HOME=/Users/defendend ast-index ...` в actual shell command внутри Codex Desktop: такой формат не матчится prefix allow и может уйти в sandbox
- любые команды `HOME=/Users/defendend ast-index ...` выполняй только вне Codex sandbox / на host macOS
- не делай sandbox-first попытку для `ast-index`: если инструмент исполнения поддерживает escalation или unrestricted execution, используй его сразу для каждой AST-команды
- `Index not found` от sandboxed `ast-index` нельзя считать состоянием индекса или AST-anomaly; сначала перепроверь ту же команду вне sandbox
- не делай отдельные env-проверки до начала discovery вроде `printf '%s\n' "$HOME"`, `env`, `printenv`, `which ast-index`
- не делай обязательный bootstrap `pwd` / `ast-index db-path` / `ast-index stats` как ритуал старта
- не делай обязательный ранний smoke-check по полной фразе фичи
- сначала сам нормализуй имя фичи в более короткие AST-friendly токены, паттерны или structural формы
- конкретный AST-маршрут выбирай сам: `search`, `class`, `file`, `symbol`, `module` или другой прямой `ast-index` subcommand
- для Android/iOS артефактов используй также `xml-usages`, `resource-usages`, `swiftui`, `async-funcs`
- для Android layout resources не подменяй `resource-usages` командой `xml-usages`: `xml-usages` ищет usages классов внутри XML, а layout references проверяются как `resource-usages @layout/<name>`. Например: `resource-usages @layout/call_notification`
- перед чтением большого файла сначала делай `ast-index outline <file>`
- не пиши шумные подготовительные AST status-updates про чтение инструкций, локальные skills, проверку `HOME`, индекс или bootstrap
- после начала реального discovery можно писать короткие substantive progress-апдейты по находкам и направлению поиска
- любые `pwd`, `ast-index db-path`, `ast-index stats`, `ast-index rebuild --sub-projects` используй только как recovery/diagnostics, если реально возникла anomaly или `Index not found`
- любую AST-anomaly фиксируй только как точную пару `команда -> сырой вывод`
- если команда дала неожиданный результат, один раз перепроверь ее из того же `ROOT` и с тем же `DB_PATH`, и только потом фиксируй anomaly
- если речь о `search` по составной фразе, сначала попробуй хотя бы две естественные нормализованные формы запроса
- для любой AST-anomaly рядом с проблемной командой обязательно покажи текущие `pwd` и `ast-index db-path`, иначе anomaly не считается доказанной
- не запускай `ast-index rebuild --sub-projects` только из-за странного результата одной AST-команды; сначала трактуй это как anomaly и продолжай discovery другими AST-маршрутами

Нужен сравнительный разбор Android vs iOS:
- entry points
- user flow end-to-end
- architecture / module boundaries
- central orchestrator / state owner
- state and data flow
- network / protocol / runtime side effects
- UI composition
- variants / subflows
- key differences между Android и iOS

Optional integration appendix:
- permissions / manifest / plist / entitlements
- deep links / push / system-event hooks
- feature flags / alerts / fallback UI
- bridge boundaries / native runtime handoff

Формат ответа:
- краткое summary
- Android
- iOS
- сравнение
- `Key Differences`
- `Architecture Map`
- `State And Flow`
- `Side Effects And OS Integration`
- `Variants / Subflows`
- таблица артефактов
- `High-Value Findings`
- `Uncertainties`
- `Integration Appendix (Optional)`
- только если есть реально существенная anomaly: короткий блок с точной парой `команда -> сырой вывод`
```

## Что делать после завершения

После того как:

- grep-thread вернул финальный ответ
- ast-thread вернул финальный ответ

1. Считать их валидными только если:
   - не было follow-up вмешательств
   - grep-thread стартовал из `/Users/defendend/workshop`
   - ast-thread стартовал из `/Users/defendend/workshop`
2. Сводить результат по двум осям:
   - качество сравнения Android vs iOS по самой фиче
   - качество метода `grep-only` vs `ast-first-confirm`
3. Итоговый ответ пользователю должен опираться только на результаты текущих валидных прогонов, без ссылок на заранее зафиксированный “правильный” вывод
4. Для judge/evaluation stage используй отдельный операторский файл:
   - `/Users/defendend/workshop/JUDGE_BENCHMARK.md`
5. Этот judge-файл нельзя включать в стартовые промпты тредов и нельзя пересказывать агентам до завершения прогонов

## Как сравнивать методы

Primary criteria:

1. Насколько глубоко метод раскрыл архитектурную структуру фичи
2. Насколько полно метод прошил core Android vs iOS comparison
3. Насколько хорошо видны:
   - entry points
   - user flow
   - module boundaries
   - orchestrator / state owner
   - state/data flow
   - runtime / protocol side effects
   - variants / subflows

Secondary criteria:

1. Literal/integration appendix coverage
2. Время до первого сигнала
3. Общее число команд
4. Объем шума

Если оба прогона валидны и оба дошли до полноценного сравнения, structural coverage важнее, чем скорость или количество команд.

То есть:

- AST может проигрывать по количеству команд и всё равно выигрывать как основной метод
- grep может быть быстрее и стабильнее, но это не должно автоматически перевешивать structural win AST

CLI/tooling anomalies у AST:

- фиксировать отдельно
- не превращать автоматически в поражение метода
- считать их отдельной осью `tooling reliability`

Итоговый verdict по методу должен определяться так:

1. Сначала structural coverage
2. Потом completeness of Android vs iOS comparison
3. Потом уже speed / command count / noise

## Что не делать

- не переписывать prompt по ходу
- не отправлять “добери ещё”
- не отправлять “заканчивай сейчас”
- не подсказывать найденные классы/файлы одному треду из результатов другого
- не смешивать выводы невалидных прогонов с валидными
- не считать AST-thread валидным без приемки его старта
