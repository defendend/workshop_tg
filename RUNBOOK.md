# Workshop Runbook

Цель: провести воркшоп-сравнение двух методов на одной и той же задаче сравнения Android и iOS Telegram:

- `grep-only` агент: только текстовый поиск и точечное чтение файлов
- `ast-first-confirm`: `ast-index` для discovery/structural analysis, `rg`/`find`/`ls`/`sed` только для позднего точечного confirmation

## Папки

- Android: `/Users/defendend/workshop/telegram-android`
- iOS: `/Users/defendend/workshop/telegram-ios`

## Что такое нормальный тред

Для этого воркшопа "нормальный тред" — это thread, который:

1. создан через `codex_app.create_thread`
2. создан как:
   - `target.type = project`
   - `target.projectId = /Users/defendend/workshop`
   - `target.environment.type = local`
3. сразу получает понятный title
4. получает только один стартовый prompt
5. после старта не получает follow-up сообщений
6. стартовый prompt ссылается на релевантные instruction files для своего режима, а сам thread реально работает из `/Users/defendend/workshop`

Если хотя бы один из этих пунктов не выполнен, тред считать ненормальным и не использовать его как валидный воркшоп-прогон.

## Приемка треда

После старта тред нужно проверить через `read_thread`.

Минимальная приемка:

- `cwd = /Users/defendend/workshop`
- стартовый prompt ссылается на root-instruction files своего режима
- нет follow-up steering

Для AST-thread дополнительно:

- до substantive analysis нет никакого preparatory prose: ни про чтение инструкций, ни про локальные skills, ни про отдельную проверку `HOME`, ни пересказа состояния среды вроде “индекс есть” или “bootstrap успешный”

## Важное ограничение

Жестко отключить инструмент у модели нельзя. Поэтому для воркшопа нужен не только промпт, но и аудит:

1. Для AST-thread явно запретить fallback в `rg`, `grep`, IDE search и любые не-`ast-index` поисковые механики для discovery/structural reconstruction.
2. Разрешить `rg --files`, `rg -n`, `find`, `ls`, `sed -n` только как поздний точечный confirm-search после того, как AST уже локализовал core architecture.
3. Если AST не нашел structural ответ AST-командами, он должен так и написать, а не восстанавливать архитектуру fallback-поиском.
4. Не давать агентам path hints по самой фиче: никаких стартовых директорий, файлов, классов, модулей или заранее известных entry points. Агенту можно дать только:
   - корень Android-репозитория
   - корень iOS-репозитория
   - имя исследуемой фичи
5. Требовать раннюю остановку: как только агент нашел entry point и 2-4 ключевых артефакта на платформу, он должен перестать расширять AST-поиск и переходить к сравнению/confirmation.
6. Для tool-based запуска не использовать `cd ... && ...` в shell-командах. Нужный корень репозитория должен передаваться через `workdir`, иначе на таких прогонах легко получить ложный `Index not found`.
7. Если AST-команда реально показывает `Index not found` или есть явный сигнал неправильного root/index, тогда диагностика через `pwd`, `ast-index db-path`, `ast-index stats` и при необходимости `ast-index rebuild --sub-projects` допустима как recovery, а не как стартовый ritual.
8. Нельзя принимать ответ `Index not found` как итог AST-прогона, пока не была предпринята recovery-попытка через диагностику root/index и при необходимости `ast-index rebuild --sub-projects`.
9. Любая AST-anomaly (`Index not found`, пустой результат, syntax error, “команда баговая”) должна сопровождаться точной парой `команда -> сырой вывод`.
10. Если anomaly заявлена без сырого вывода, AST-прогон считать невалидным и не сравнивать его с grep-only.
11. Если AST-thread объявил `search` сломанным по одному единственному составному запросу и не попробовал хотя бы две естественные нормализованные формы запроса, такой прогон считать невалидным.
12. Если AST-thread заявил anomaly без показа текущих `pwd` и `ast-index db-path` рядом с проблемной командой, такой прогон считать невалидным: нельзя делать вывод о `search`, если не доказано, что вызов шел из правильного `ROOT`.
13. AST-команды валидны только если они вызваны напрямую как `HOME=/Users/defendend ast-index <subcommand> ...`.
14. Любой прогон, где AST вызывался через `bash -c`, `zsh -lc`, `sh -c`, wrapper-скрипт, alias или другую обертку, считать невалидным.
15. Любой прогон, где AST вызывался без явного `HOME=/Users/defendend`, считать невалидным.
16. Любой grep-прогон или AST confirm-search, где разрешенные команды были склеены через `|`, `&&`, `||`, `;`, process substitution или subshell, считать невалидным.

## Правило валидного AST-thread

Для этого воркшопа AST-thread можно запускать отдельным thread.

Но AST-thread считается валидным только если:

- создан как `project + local`
- стартовал из `/Users/defendend/workshop`
- не печатал preparatory prose или пересказ состояния среды до substantive analysis
- после старта не получил follow-up steering
- не использовал `rg`/`grep` как discovery/fallback до того, как AST локализовал core architecture
- не использовал IDE/MCP

Следствие:

- `grep-only` можно запускать thread
- AST-thread тоже можно запускать thread
- если AST-thread не проходит эту приемку, его надо архивировать и перезапускать как новый thread
- не подменять кривой AST-thread локальным ручным AST, если цель — именно валидный thread-прогон

## Подготовка AST-индекса

Нужно выполнить один раз перед AST-демо:

```bash
cd /Users/defendend/workshop
ast-index rebuild --sub-projects
```

## Как запускать

Если пользователь просит конкретный воркшоп-этап/шаг/сценарий, это deterministic dispatch, а не repo exploration:

- не ищи shell-скрипты
- не смотри `scripts/` как возможный entry point
- не запускай `./scripts/1-run.sh`, `./scripts/2-analyze.sh` или `./scripts/3-cleanup.sh`
- сразу открывай соответствующий root-level operator file ниже
- дальше создавай Codex `project + local` thread/thread-ы по инструкции из operator file

Для конкретных этапов воркшопа используй отдельные операторские файлы:

- первый этап / benchmark `grep-only` vs `AST first, grep confirm`: `/Users/defendend/workshop/FIRST_WORKSHOP_STEP.md`
- второй этап / reusable project architecture maps для будущих AI-агентов: `/Users/defendend/workshop/SECOND_WORKSHOP_STEP.md`
- третий этап / map-guided vs grep-first implementation planning: `/Users/defendend/workshop/THIRD_WORKSHOP_STEP.md`

Для валидного воркшоп-прогона:

- `grep-only` запускать отдельным локальным thread
- `ast-first-confirm` запускать отдельным локальным thread

Это thread-based сценарий, а не shell script:

- не запускай `./scripts/1-run.sh`
- не ищи shell-скрипт как primary entry point
- если пользователь просит "первый сценарий" или "новый прогон первого сценария", сразу создавай два Codex thread по правилам ниже

Оба thread:

- `target.type = project`
- `target.projectId = /Users/defendend/workshop`
- `target.environment.type = local`

В задаче не должно быть path hints по фиче:

- общий task: `prompts/compare-feature-task.md`
- ограничения grep-only: `prompts/grep-only-agent.md`
- ограничения AST first / grep confirm: `prompts/ast-only-agent.md`

Формула запуска:

1. Берешь текст из `compare-feature-task.md`
2. Добавляешь сверху текст из `grep-only-agent.md`
3. Отправляешь его в локальный `project thread`
4. Берешь тот же `compare-feature-task.md`
5. Добавляешь сверху текст из `ast-only-agent.md`
6. Отправляешь его во второй локальный `project thread`
7. Проверяешь AST-thread через `read_thread`
8. Если AST-thread не прошел приемку, архивируешь и запускаешь новый AST-thread
9. Никаких follow-up сообщений после старта

Нельзя:

- считать любой AST-thread автоматически валидным без приемки
- досылать follow-up steering после старта

## Что сравнивать

Лучше всего заходят фичи, у которых есть и UI, и routing/navigation, и data flow:

- Deep links
- Chat list / folder filters
- Message reactions
- Voice/video calls
- Notifications and badge counters
- Themes / appearance
- Media picker / attachment flow
- Premium / subscriptions

## Что оценивать в сравнении

- Время до первого полезного результата
- Сколько команд понадобилось
- Сколько шумных попаданий было
- Насколько уверенно агент нашел entry point
- Смог ли он дотянуться до usage chain
- Насколько хорошо агент объяснил различия Android vs iOS
- Насколько рано агент отсекает test/generated/third-party шум

## Как выносить verdict

Для judge/evaluation stage используй отдельный операторский файл:

- `/Users/defendend/workshop/JUDGE_BENCHMARK.md`

Его нельзя добавлять в стартовые промпты `grep-only` и AST-thread тредов.

Primary:

1. structural coverage
2. completeness of core Android vs iOS comparison
3. качество объяснения entry points / flow / module boundaries / orchestrator / state / runtime side effects / UI

Secondary:

1. literal/integration appendix coverage
2. скорость
3. число команд
4. шум

Если оба метода валидны и оба дошли до полноценного сравнения, AST может считаться победителем даже при большем количестве команд и худшем времени, если он дал более сильную structural картину.

Tooling issues у AST:

- фиксировать отдельно
- не смешивать автоматически с verdict по качеству метода
- считать подтвержденными только если в ответе есть точная пара `команда -> сырой вывод`

## Зафиксированный вывод по кейсу `voice/video calls`

Для первого кейса:

- AST-thread сильнее как structural discovery
- `grep-only` сильнее как literal/details confirmation

Практический режим после воркшопа:

1. `ast-index` first
2. `grep` second only for confirmation

## Для честного AST first / grep confirm режима

Не ограничивай агента whitelist-ом index-команд. Пусть использует любые команды `ast-index`, которые помогают локализовать фичу.

Confirm-search через `rg --files`, `rg -n`, `find`, `ls`, `sed -n` разрешен только после AST-discovery и только для точечного добора literal/integration evidence. Он не должен подменять поиск entry points, ownership, flow или module boundaries.

Если какая-то конкретная index-backed команда на данном репозитории падает или дает ложный `Index not found`, это нужно фиксировать в отчете как баг/ограничение CLI, но не превращать в глобальный запрет на весь AST first режим.

## Для честного grep-only режима

Разрешай только обычный файловый и текстовый поиск:

- `rg --files`
- `rg -n`
- `find`
- `ls`
- `sed -n`

Запрещай:

- `ast-index`
- любые MCP/IDE code search
- bulk-чтение больших файлов без причины
