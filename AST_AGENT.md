# AST-Only Agent

Ты работаешь в режиме `ast-only` для воркшопа.

## Цель

Сравнить одну и ту же фичу в Android и iOS Telegram, используя именно `ast-index` как единственный механизм поиска по коду.

## Что тебе известно

- общий корень для AST: `/Users/defendend/workshop`
- Android: `/Users/defendend/workshop/telegram-android`
- iOS: `/Users/defendend/workshop/telegram-ios`
- имя исследуемой фичи задается отдельно

Кроме root и имени фичи, тебе ничего заранее не известно.

## Базовое правило

Для этого режима:

- `ast-index` — единственный инструмент поиска по коду
- `rg`, `grep`, `findstr`, IDE search, MCP и любые другие текстовые поисковые механики запрещены
- если AST-маршрутом не удалось доказать часть ответа, это надо явно писать в ограничениях, а не делать fallback на текстовый поиск

## Жесткое правило вызова AST

Все AST-команды в этом воркшопе должны вызываться только напрямую как actual shell command:

```bash
ast-index <subcommand> ...
```

`HOME=/Users/defendend` задается окружением Codex Desktop (`shell_environment_policy`). В отчете, `RAW_*` блоках и списке команд показывай logical command как:

```bash
HOME=/Users/defendend ast-index <subcommand> ...
```

Запрещено:

- `bash -c 'ast-index ...'`
- `zsh -lc 'ast-index ...'`
- `sh -c 'ast-index ...'`
- любые wrapper-скрипты вокруг `ast-index`
- любые alias/function/launcher-обертки, меняющие способ запуска
- inline `HOME=/Users/defendend ast-index ...` как actual shell command в Codex Desktop

Если в Codex Desktop actual shell command не начинается с `ast-index <subcommand> ...`, прогон считается невалидным. Logical command в отчете при этом должен оставаться `HOME=/Users/defendend ast-index ...`.

## Preferred AST Workflow

По умолчанию весь AST-анализ делается из общего корня:

```bash
cd /Users/defendend/workshop
ast-index ...
```

Это preferred mode.

`--walk-up`:

- разрешен
- не обязателен
- не считается гарантией использования общего root-индекса

## Silent Startup

Если в prompt заданы обязательные instruction files:

- прочитай их молча
- не пиши отдельные сообщения про чтение инструкций, `source of truth`, дочитывание хвостов или `Explored N files`
- не объявляй подключение локальных skills, MCP, plugin-инструкций или других вспомогательных правил, если режим уже жестко задан воркшопом

До начала реального AST-discovery:

- не пиши промежуточные апдейты вообще
- не пересказывай состояние среды
- не объясняй, что сейчас проверяешь `HOME`, индекс или среду

В обычном валидном прогоне:

- не печатай отдельные meta-блоки `INDEX_ROOT`, `PROJECT_ROOT_ANDROID`, `PROJECT_ROOT_IOS`, `DB_PATH`, `STATS_RESULT`, `REBUILD_PERFORMED`
- не печатай отдельные блоки `RAW_BOOTSTRAP` и `RAW_SEARCH_SMOKE`
- не давай пошаговые AST status-updates после bootstrap
- либо молча работай до финального ответа, либо показывай только доказательство реальной anomaly

Отдельные env-команды до начала discovery запрещены:

- не запускай `printf '%s\n' "$HOME"`
- не запускай `env`, `printenv`, `which ast-index` и похожие подготовительные проверки
- не делай обязательный ритуал `pwd` / `ast-index db-path` / `ast-index stats`
- не делай обязательный ранний `search`-smoke-check по фразе фичи

## Discovery Start

Начинай сразу с реального AST-discovery по фиче.

Для кейса `voice/video calls`:

- не делай обязательный отдельный `search "video calls"` как smoke-check
- начинай сразу с коротких AST-friendly токенов вроде `call`, `calls`, `voice`, `video`, `VoIP`
- переходи к `class`, `file`, `symbol`, `usages`, `callers`, `outline` по мере сигнала

`pwd`, `ast-index db-path`, `ast-index stats`, `ast-index rebuild --sub-projects` допустимы только как recovery/diagnostics, если:

- AST-команда реально вернула `Index not found`
- есть подозрение, что выбран не тот root/index
- нужно доказать anomaly

Если индекс реально не найден, тогда:

```bash
cd /Users/defendend/workshop
ast-index rebuild --sub-projects
```

В отчете фиксируй это как logical command `HOME=/Users/defendend ast-index rebuild --sub-projects` только если rebuild действительно понадобился.

## Команды

Можно использовать любые команды `ast-index`, если они помогают локализовать и объяснить фичу.

Например:

- `search`
- `agrep`
- `file`
- `symbol`
- `class`
- `usages`
- `refs`
- `callers`
- `call-tree`
- `implementations`
- `hierarchy`
- `extensions`
- `module`
- `outline`
- `imports`
- `changed`
- `map`
- `conventions`
- `xml-usages`
- `resource-usages`
- `swiftui`
- `async-funcs`
- `todo`
- `deprecated`
- `stats`

Правило простое:

- любые прямые команды `ast-index` разрешены
- `search` обязателен как первый маршрут discovery
- запрещены только не-`ast-index` поисковые механики

Android XML/resource rule:

- `xml-usages <name>` ищет usages классов внутри XML
- layout/resource references проверяй через `resource-usages @layout/<name>`
- пример: для layout `call_notification` используй `resource-usages @layout/call_notification`, а не `xml-usages call_notification`

## Правило чтения больших файлов

Перед точечным чтением большого файла сначала делай:

```bash
ast-index outline <file>
```

Потом читай только нужные диапазоны, к которым привел outline или найденный символ.

## Рабочий порядок

1. Убедись, что индекс существует в `/Users/defendend/workshop`
2. Если индекса нет, выполни `ast-index rebuild --sub-projects` из `/Users/defendend/workshop`
3. Все AST-команды по умолчанию выполняй из `/Users/defendend/workshop`
4. Если решишь использовать AST из подпроекта и видишь anomaly, отдельно проверь `pwd`, `db-path`, `stats` как диагностику
5. Сначала сделай discovery по имени фичи и его словоформам
6. Начинай discovery именно с `search`
7. Но не начинай с многословного `search` по человеческой фразе целиком
8. Сначала нормализуй фичу в короткие токены и паттерны вроде `call`, `calls`, `voice`, `video`, `VoIP`
9. Первые AST-запросы должны сразу работать на discovery по таким коротким нормализованным токенам, без обязательного smoke-check
10. После первых `search` переходи к `class`, `file`, `symbol`, а затем к `usages`, `callers`, `outline`
11. Если `search` на одном токене ведет себя странно, не зависай на его диагностике в середине benchmark-прогона: попробуй следующий нормализованный `search`, затем продолжай discovery через `class`, `file`, `symbol`, `usages`, `callers`, `outline`
12. Не запускай `ast-index rebuild --sub-projects` только из-за странного `search`/`file`/`symbol`; сначала трактуй это как возможную anomaly и продолжай discovery другими AST-маршрутами
13. Затем углубляйся через:
   - `usages`
   - `refs`
   - `callers`
   - `call-tree`
   - `implementations`
   - `hierarchy`
   - `module`
14. Для Android/iOS артефактов используй также:
   - `xml-usages`
   - `resource-usages`
   - `swiftui`
   - `async-funcs`
15. Для Android layout resources не подменяй `resource-usages` командой `xml-usages`: `xml-usages` ищет usages классов внутри XML, а layout references проверяются как `resource-usages @layout/<name>`. Например: `resource-usages @layout/call_notification`.
16. После первого осмысленного попадания в каждом репозитории найди не больше двух независимых якорей:
   - один UI/entry point
   - один data/model/service anchor
17. Читай исходники только точечно, когда AST уже привел тебя к конкретному файлу или символу
18. Как только у тебя есть entry point, 2-4 ключевых файла и понятный trigger, прекращай расширять поиск

## Что нужно сделать

Нужен не просто поиск файлов, а сравнение фичи между Android и iOS:

1. entry points
2. user flow end-to-end
3. gating / restrictions / permissions / flags
4. state and data flow
5. side effects: network, persistence, cache, updates
6. UI composition
7. ключевые различия Android vs iOS

## Формат ответа

В ответе обязательно выведи:

- краткое summary
- Android
- iOS
- сравнение
- `Key Differences`
- таблица артефактов
- список использованных AST-команд в точном виде
- если есть реально существенная anomaly, кратко покажи ее как доказанную пару `команда -> сырой вывод`
