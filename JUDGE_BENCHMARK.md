# Benchmark Judge Prompt

Этот файл предназначен только для оператора/судьи после завершения прогонов.

Его нельзя:

- добавлять в стартовый prompt `grep-only` треда
- добавлять в стартовый prompt `ast-only` треда
- пересказывать агентам до старта
- использовать как source of truth для discovery

## Что оценивается

Сравниваются только валидные прогоны.

Сначала проверь валидность:

1. thread создан как `project + local`
2. `cwd = /Users/defendend/workshop`
3. после старта не было follow-up steering
4. `grep-only` не выходил за разрешенные grep/file-search команды
5. `ast-only` не выходил за `ast-index` и соблюдал direct invocation policy настолько, насколько это можно подтвердить имеющимся аудитом
6. если raw tool trace недоступен, не делай чрезмерно сильных выводов о shell-level деталях; помечай это как audit limit

Если один из прогонов невалиден:

- не объявляй его победителем
- либо исключи его из сравнения, либо явно пометь результат как invalid / not comparable

## Как сравнивать

Primary criteria:

1. structural coverage
2. completeness of core Android vs iOS comparison
3. качество объяснения:
   - entry points
   - user flow
   - module boundaries
   - central orchestrator / state owner
   - state/data flow
   - runtime / protocol side effects
   - variants / subflows

Secondary criteria:

1. literal/integration appendix coverage
2. speed
3. command count
4. noise

## Как трактовать structural gaps у grep

Если `ast` нашел архитектурные слои, которые `grep` не собрал в цельную картину, это нужно считать не косметическим плюсом, а практическим преимуществом для дальнейшей реализации.

Особенно важно явно отметить, если у `grep` не хватает:

1. ownership map:
   - app-level orchestrator
   - protocol / session owner
   - runtime / presentation owner
2. разделения private call vs group / conference call
3. runtime / native boundary:
   - JNI
   - Obj-C / Swift wrapper
   - WebRTC / tgvoip context
4. incoming / system-mediated flow:
   - push / CallKit / Telecom / notification / pre-notification path
5. module boundaries и source of truth

Если такие пробелы есть, судья должен прямо написать, почему это важно practically:

- можно выбрать не того owner-а для изменения
- можно недооценить число модулей или слоев
- можно не заметить отдельную variant / system path
- можно ошибочно считать UI-слой местом реализации, хотя логика сидит в session/runtime owner
- это повышает риск позднего расширения scope и переделывания задачи

Иными словами:

- если `grep` нашел literal evidence, но не собрал ownership / boundaries / runtime split, это реальный architectural gap
- если `ast` собрал эти слои, это нужно засчитывать как полезное преимущество для implementation planning, а не только как “более красивое объяснение”

## Что считать core architecture

Core architecture:

- entry points
- user flow end-to-end
- module / layer boundaries
- central orchestrator / state owner
- protocol / session owner
- runtime boundary
- state propagation
- major variants / subflows

Optional integration appendix:

- permissions / manifest / plist / entitlements
- deep links / push / system-event hooks
- alerts / strings / fallback UI
- analytics / keys / intent-filters
- bridge boundaries like JNI / Obj-C wrappers / generated bindings

Важно:

- appendix не должен автоматически перевешивать core architecture
- literal-evidence сам по себе не равен architectural superiority
- если `grep` богаче по manifest/plist/details, а `ast` лучше по ownership/boundaries/flow, это не автоматическая победа `grep`

## Как трактовать bridge boundaries

Для JNI / Obj-C wrappers / `.proto` / generated bindings / native runtime:

- достаточно оценить, нашел ли агент boundary artifact
- понял ли направление перехода
- связал ли boundary с orchestration/state owner

Не штрафуй `ast-only` автоматически за отсутствие полного traversal по ту сторону native boundary, если core ownership уже доказан.

## Как трактовать speed

- speed важен, но вторичен после core architecture
- если разница в качестве core architecture небольшая, speed может решить verdict
- если один метод явно сильнее в structural coverage, он может выиграть даже при большем времени

## Рекомендуемый формат judge-ответа

- `Валидность`
- `Качество`
- `Вердикт`

В `Качество` разделяй:

- `Core architecture`
- `Integration appendix`

Внутри `Core architecture`, если разница между методами заметна, желательно отдельно указать:

- `Что AST собрал, чего не хватает grep`
- `Почему эти пробелы grep опасны для дальнейшей реализации`

В `Вердикт` явно указывай:

- winner
- насколько уверенная победа
- почему

## Anti-bias reminders

- не опирайся на зафиксированные выводы прошлых прогонов
- не путай плотность literal-цитат с качеством architecture comparison
- не награждай `ast-only` просто за то, что это AST
- не награждай `grep-only` просто за то, что он нашел больше manifest/plist/details
- оцени только текущую пару валидных прогонов
