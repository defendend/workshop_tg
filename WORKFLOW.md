# Workshop Workflow

Текущий согласованный workflow для воркшопа.

## Общая идея

- Оба проекта лежат под одним общим корнем: `/Users/defendend/workshop`
- Android: `/Users/defendend/workshop/telegram-android`
- iOS: `/Users/defendend/workshop/telegram-ios`
- Цель воркшопа: сравнивать одну и ту же фичу между Android и iOS
- Режимы:
  - `grep-only`
  - `ast-only`

## Базовая рекомендация по анализу

Есть два разных сценария:

1. Чистый benchmark `grep-only` vs `ast-only`
2. Практический рабочий анализ фичи

Для benchmark:

- режимы нужно держать раздельно и не смешивать

Для практического рабочего анализа:

- preferred sequence:
  1. `ast-index` first
  2. `grep` second only for literal/details confirmation

Это означает:

- AST лучше как структурная разведка
- grep лучше как слой подтверждения строк, manifest/plist/resource details и literal-констант

## Главный принцип AST

Для воркшопа AST по умолчанию запускается из общего корня:

```bash
cd /Users/defendend/workshop
ast-index ...
```

Это preferred path.

Причина:

- не нужно делать предположения о том, какой индекс выберется в подпроекте
- не нужно делать обязательным `--walk-up`
- проще контролировать `pwd`, `db-path` и `stats`

`--walk-up`:

- разрешен как вспомогательный флаг
- не считается обязательным
- не считается гарантией использования общего root-индекса

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

### 3. Выполнять AST-команды из общего корня

Примеры:

```bash
cd /Users/defendend/workshop
ast-index search "premium"
ast-index class --pattern "*Premium*"
ast-index usages PremiumFeature
ast-index callers startCall
ast-index call-tree processPayment -d 3
ast-index outline telegram-ios/submodules/TelegramCallsUI/Sources/CallController.swift
```

### 4. Как разделять Android и iOS

Когда AST-прогон идет из общего корня:

- различай платформы по путям из результатов
- для чтения файлов используй root-relative или абсолютные пути
- если нужно, используй:
  - `file`
  - `search`
  - `symbol`
  - `class`
  - `module`
  - `imports`
  - `outline`

### 5. Если AST все же запускается из подпроекта

Это допустимо, но только после явной проверки:

```bash
pwd
ast-index db-path
ast-index stats
```

Нельзя заранее предполагать, что:

- подпроект возьмет общий root-индекс
- `--walk-up` исправит выбор индекса

Если запускаешь AST из подпроекта, фактически используемый `DB_PATH` должен быть подтверждаем через command trace или внутренний аудит; печатать его в пользовательском отчете не обязательно.

## Что нельзя делать в AST-режиме

- Нельзя использовать `rg`, `grep`, `findstr`, IDE search, MCP и любые другие текстовые поисковые механики
- Нельзя завершать прогон сообщением `Index not found`, пока не выполнена проверка `db-path`/`stats` и при необходимости `rebuild --sub-projects`
- Нельзя смешивать разные AST root в одном прогоне без возможности доказать, какой `ROOT` и `DB_PATH` реально использовались

## Grep Workflow

Для grep-only общий AST-индекс не нужен.

Android:

```bash
cd /Users/defendend/workshop/telegram-android
rg -n "premium" .
```

iOS:

```bash
cd /Users/defendend/workshop/telegram-ios
rg -n "premium" .
```

## Формат сравнения фичи

Нужен не просто поиск файлов, а сравнение:

1. entry points
2. user flow end-to-end
3. gating / premium checks / restrictions / flags
4. state and data flow
5. side effects: network, persistence, cache, updates
6. UI composition
7. ключевые различия Android vs iOS

## Ближайший план

Для следующего прогона:

1. AST: `rebuild --sub-projects` из `/Users/defendend/workshop`
2. AST: запускать команды из `/Users/defendend/workshop`
3. Grep Android: обычный поиск из `telegram-android`
4. Grep iOS: обычный поиск из `telegram-ios`
5. Потом сводное сравнение

## Зафиксированный вывод по первому кейсу

Для кейса `voice/video calls` текущий вывод воркшопа такой:

- как метод структурной разведки выигрывает `ast-only`
- как метод подтверждения literal/details нужен `grep-only`
- practical mode:
  - `AST first`
  - `grep confirm`
