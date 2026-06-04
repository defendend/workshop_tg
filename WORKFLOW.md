# Workshop Workflow

Текущий согласованный workflow для воркшопа.

## Общая идея

- Оба проекта лежат под одним общим корнем: `/Users/defendend/workshop`
- Android: `/Users/defendend/workshop/telegram-android`
- iOS: `/Users/defendend/workshop/telegram-ios`
- Цель воркшопа: сравнивать одну и ту же фичу между Android и iOS
- Режимы:
  - `grep-only`
  - `ast-first-confirm`

## Режимы анализа

Есть два разных сценария:

1. Benchmark `grep-only` vs `ast-first-confirm`
2. Практический рабочий анализ фичи

Для benchmark:

- режимы нужно держать раздельно и не смешивать
- в `ast-first-confirm` AST отвечает за discovery/structural analysis, а обычный файловый поиск разрешен только как поздний точечный confirmation слой

## Главный принцип AST

Для честного feature-analysis AST не должен делать discovery по общему workspace root.

Предпочтительный режим:

- Android анализировать из `/Users/defendend/workshop/telegram-android`
- iOS анализировать из `/Users/defendend/workshop/telegram-ios`

Если общий индекс построен из `/Users/defendend/workshop`, это допустимый tooling detail, но не повод искать по всему workspace сразу.

`--walk-up`:

- разрешен как вспомогательный флаг
- не считается обязательным
- не считается гарантией использования общего root-индекса
- не должен использоваться как оправдание для broad search по всему `/Users/defendend/workshop`

## AST Workflow

### 1. Построить или обновить индекс из общего корня

```bash
cd /Users/defendend/workshop
ast-index rebuild --sub-projects
```

### 2. Перед AST-анализом проверить среду

```bash
cd /Users/defendend/workshop
pwd
ast-index db-path
ast-index stats
```

Если индекс не найден:

```bash
cd /Users/defendend/workshop
ast-index rebuild --sub-projects
```

### 3. Выполнять AST-команды из repo roots

Примеры:

```bash
cd /Users/defendend/workshop/telegram-android
ast-index search "<feature-token>"
ast-index class --pattern "*<FeaturePattern>*"

cd /Users/defendend/workshop/telegram-ios
ast-index search "<feature-token>"
ast-index callers <entryPointSymbol>
ast-index outline <path/to/relevant/file>
```

### 4. Как разделять Android и iOS

Когда AST-прогон идет честно по repo roots:

- не ищи в sibling-проекте, пока не переключился в его repo root
- для чтения файлов используй пути внутри текущего repo root
- если результат поиска уводит за пределы текущего repo root, не используй его как evidence
- если нужно, используй:
  - `file`
  - `search`
  - `symbol`
  - `class`
  - `module`
  - `imports`
  - `outline`

### 5. Если AST все же запускается из общего корня

Это допустимо только как технический recovery/inspection режим, а не как основной discovery-путь.

Перед тем как доверять таким результатам, нужна явная проверка:

```bash
pwd
ast-index db-path
ast-index stats
```

Нельзя заранее предполагать, что:

- подпроект возьмет общий root-индекс
- `--walk-up` исправит выбор индекса
- общий workspace root даст честный per-project discovery без лишнего шума

Если запускаешь AST не из repo root, фактически используемый `DB_PATH` должен быть подтверждаем через command trace или внутренний аудит; печатать его в пользовательском отчете не обязательно.

## Что нельзя делать в AST first / grep confirm режиме

- Нельзя использовать `rg`, `grep`, `findstr`, IDE search, MCP и любые другие текстовые поисковые механики для discovery, ownership analysis или reconstruction of flow
- Нельзя использовать confirm-search до того, как AST уже локализовал core architecture
- Нельзя превращать confirm-search в широкий второй discovery-проход
- Нельзя делать broad discovery по общему `/Users/defendend/workshop`, если цель — честно сравнить два отдельных проекта
- Нельзя завершать прогон сообщением `Index not found`, пока не выполнена проверка `db-path`/`stats` и при необходимости `rebuild --sub-projects`
- Нельзя смешивать разные AST root в одном прогоне без возможности доказать, какой `ROOT` и `DB_PATH` реально использовались

## Grep Workflow

Для grep-only общий AST-индекс не нужен.

Android:

```bash
cd /Users/defendend/workshop/telegram-android
rg -n "<feature-token>" .
```

iOS:

```bash
cd /Users/defendend/workshop/telegram-ios
rg -n "<feature-token>" .
```

## Формат сравнения фичи

Нужен не просто поиск файлов, а сравнение:

Core architecture:

1. entry points
2. user flow end-to-end
3. architecture / module boundaries
4. central orchestrator / state owner
5. state and data flow
6. network / protocol / runtime side effects
7. UI composition
8. variants / subflows
9. ключевые различия Android vs iOS

Optional integration appendix:

1. permissions / manifest / plist / entitlements
2. deep links / push / system-event hooks
3. feature flags / alerts / fallback UI
4. bridge boundaries: JNI / Obj-C wrappers / generated bindings / native runtime handoff

## Нейтральность benchmark

В thread-facing инструкциях не должно быть:

- verdict по предыдущим прогонам
- preferred winner для конкретной фичи
- verdict или практической рекомендации о победителе метода

Такие выводы нужно хранить отдельно от инструкций, по которым запускаются новые прогоны.
