# Start Chat Prompt

Работаем в `/Users/defendend/workshop`.

Контекст:

- здесь лежат два полных проекта:
  - `telegram-android`
  - `telegram-ios`
- цель воркшопа: сравнивать одну и ту же фичу между Android и iOS
- нужны два режима:
  - `grep-only`
  - `ast-first-confirm`

Перед началом работы сначала прочитай релевантные файлы в корне `~/workshop`:

- для `grep-only`: `WORKFLOW.md` и `GREP_AGENT.md`
- для `ast-first-confirm`: `WORKFLOW.md` и `AST_AGENT.md`

Если работаешь в `ast-first-confirm` режиме, эти файлы нужно читать молча. Не пиши отдельные апдейты про чтение инструкций, локальные skills, проверку `HOME`, bootstrap, `DB_PATH`, `STATS_RESULT` или другие meta-блоки; в обычном прогоне нужен только финальный содержательный ответ, если нет anomaly.

Ключевые правила:

1. Отвечай по-русски.
2. Не ограничивайся простой локализацией файлов. Нужен именно сравнительный разбор фичи между Android и iOS.
3. Сравнение должно включать:
   - entry points
   - user flow end-to-end
   - gating / premium checks / restrictions / feature flags
   - state and data flow
   - side effects: network / persistence / cache / updates
   - UI composition
   - key differences между Android и iOS
4. Не давай агентам path hints по фиче, если это отдельно не запрошено.
5. Если работаешь в `ast-first-confirm` режиме:
   - индекс строится из `/Users/defendend/workshop`
   - команда:

```bash
cd /Users/defendend/workshop
ast-index rebuild --sub-projects
```

6. Если нужен именно общий индекс из `/Users/defendend/workshop`, запускай AST-команды из общего корня.

7. Если работаешь из подпроекта, можно использовать AST как из него, так и с `--walk-up`, но это не считается гарантией общего root-индекса.

Android пример:

```bash
cd /Users/defendend/workshop/telegram-android
pwd
ast-index db-path
ast-index stats
ast-index class --pattern "*Premium*"
```

iOS пример:

```bash
cd /Users/defendend/workshop/telegram-ios
pwd
ast-index db-path
ast-index stats
ast-index class --pattern "*Premium*"
```

8. Если запускаешь AST-агента, он не должен начинать с обязательного bootstrap-ритуала `pwd` / `ast-index db-path` / `ast-index stats`. Эти команды допустимы только как диагностика, если в ходе AST-поиска реально возникла anomaly или `Index not found`.

9. Для воркшопа не считай AST-thread автоматически валидным запуском.

Правильный режим:

- `grep-only` можно запускать отдельным local project-thread
- `ast-first-confirm` тоже можно запускать отдельным local project-thread
- AST-thread валиден только если прошел приемку:
  - `cwd = /Users/defendend/workshop`
  - в первых шагах нет обязательного bootstrap-ритуала
  - нет follow-up steering

12. В `ast-first-confirm` режиме можно использовать любые команды `ast-index`.
13. В `ast-first-confirm` режиме `rg --files`, `rg -n`, `find`, `ls`, `sed -n` разрешены только после AST-discovery и только для точечного confirmation слоя.

Текущая задача по умолчанию:

- подготовить и прогонять сравнение сложных фич между `telegram-android` и `telegram-ios`
- сравнение должно быть полноценным, а не “просто найти где лежит”

Если я попрошу “запусти агентов”, используй для каждого режима его собственный agent-файл как source of truth.

После завершения пары тредов для оценки используй отдельный операторский файл:

- `/Users/defendend/workshop/JUDGE_BENCHMARK.md`

Важно:

- этот файл предназначен только для judge/evaluation stage
- не добавляй его в стартовые промпты агентов
- не пересказывай его агентам до старта
