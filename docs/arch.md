# Cпецификация БД clife (release 1)
 
Статус: готово к реализации. Дата: 2026-10-17.

## 0. Для исполнителя
 
- Этот документ — **единственный источник правды** по БД и доменной логике. Не добавляй поля, таблицы, значения enum и не меняй имена.
- Неоднозначность или противоречие → **задай вопрос**, не догадывайся.
- Русские названия — термины предметной области. В коде и БД используются имена из колонки «Колонка» (латиница, snake_case).
- Соглашения нумеруются `С-xx`, инварианты `И-xx`, тесты `Т-xx`. Каждый инвариант покрывается минимум одним тестом.
- Логика из §5 — **чистые функции в `domain/`**, без зависимостей от Android и Room, с инжектируемым `Clock`/`ZoneId`.

---
 
## 1. Соглашения
 
| № | Правило |
|---|---|
| С-1 | ID — UUID v4 строкой. Генерирует приложение (репозиторий), не БД. Автоинкремента нет. |
| С-2 | Момент времени — `INTEGER`, миллисекунды UTC (`Instant.toEpochMilli()`). |
| С-3 | Дата без времени («локальная дата») — `TEXT` в ISO `YYYY-MM-DD` (`LocalDate.toString()`). Часовой пояс не хранится: дата — календарное понятие, не момент. |
| С-4 | Часовой пояс — `TEXT`, IANA (`Europe/Moscow`). «Сегодня» и границы периодов считаются в поясе устройства на момент расчёта. |
| С-5 | Мягкое удаление: `deleted_at`. Все выборки для UI фильтруют `deleted_at IS NULL`. Физически строки не удаляются. |
| С-6 | `updated_at` обновляется при любом изменении сущности (для событий не нужен: они неизменяемы, кроме `deleted_at`). |
| С-7 | Время получать только через инжектируемый `Clock`. Прямые `System.currentTimeMillis()` и `LocalDate.now()` запрещены. |
| С-8 | Неделя начинается с понедельника. Дни недели в числах — ISO: 1=пн … 7=вс. |
| С-9 | Enum хранятся как `TEXT` по имени константы. |
| С-10 | Room: `version = 1`, `exportSchema = true`, JSON-схема коммитится. Room не выражает `CHECK` в аннотациях, поэтому инварианты проверяет репозиторий + unit-тесты. |
| С-11 | Внешние ключи включены. Индексы на все FK-колонки. |
| С-12 | Новая сущность получает `sort_order = max(sort_order в своей таблице среди неудалённых) + 1`. |
| С-13 | Любое изменение схемы после v1: повышение версии + `Migration` + новый JSON схемы. Без `fallbackToDestructiveMigration`. |
 
---

## 2. Enum
 
| Kotlin enum | Значения (в БД) | Русские термины |
|---|---|---|
| `ColorKey` | `RED, ORANGE, YELLOW, GREEN, TEAL, BLUE, PURPLE, PINK, BROWN, GRAY` | палитра (константа в коде, таблицы нет) |
| `GoalPeriod` | `DAY, WEEK, MONTH` | день, неделя, месяц |
| `CounterEventType` | `CHANGE, RESET` | изменение, сброс |
| `RepeatType` | `BY_CALENDAR, FROM_DONE, NO_DEADLINE` | по_календарю, от_выполнения, без_срока (`NULL` = разовая) |
| `RepeatUnit` | `DAY, WEEK, MONTH` | день, неделя, месяц |
| `TaskEventType` | `DONE, SERIES_END` | выполнение, завершение_серии |
| `OutboxTarget` | `GCAL, NOTION` | |
| `OutboxEntity` | `TASK, COUNTER, DAY_NOTE` | |
| `OutboxOp` | `UPSERT, DELETE` | |
| `OutboxStatus` | `PENDING, DONE, FAILED` | |
 
Дни недели в `repeat_weekdays` — коды `MO, TU, WE, TH, FR, SA, SU`.
 
---
 
## 3. Таблицы
 
### 3.1 `category` — категория
 
| Поле | Колонка | Тип | NULL | Описание |
|---|---|---|---|---|
| айди | `id` | TEXT PK | нет | UUID |
| название | `name` | TEXT | нет | |
| цвет | `color_key` | TEXT | да | `ColorKey` |
| порядок | `sort_order` | INTEGER | нет | по умолчанию 0 |
| создана_в | `created_at` | INTEGER | нет | UTC мс |
| изменена_в | `updated_at` | INTEGER | нет | UTC мс |
| удалена_в | `deleted_at` | INTEGER | да | |

### 3.2 `counter` — счётчик
 
| Поле | Колонка | Тип | NULL | Описание |
|---|---|---|---|---|
| айди | `id` | TEXT PK | нет | UUID |
| категория_айди | `category_id` | TEXT FK → category | да | |
| цвет | `color_key` | TEXT | да | `ColorKey`; `NULL` = цвет категории |
| название | `title` | TEXT | нет | |
| описание | `description` | TEXT | да | |
| цель_значение | `goal_target` | INTEGER | да | > 0 |
| цель_период | `goal_period` | TEXT | да | `GoalPeriod`; `NULL` при заданной цели = «за всё время» |
| период_старт | `period_start` | INTEGER | да | WEEK: 1..7 (1=пн); MONTH: 1..28 (число месяца) |
| порядок | `sort_order` | INTEGER | нет | по умолчанию 0 |
| создан_в | `created_at` | INTEGER | нет | |
| изменён_в | `updated_at` | INTEGER | нет | |
| архивирован_в | `archived_at` | INTEGER | да | остановлен, история сохранена |
| удалён_в | `deleted_at` | INTEGER | да | |

### 3.3 `counter_event` — событие счётчика
 
| Поле | Колонка | Тип | NULL | Описание |
|---|---|---|---|---|
| айди | `id` | TEXT PK | нет | UUID |
| счётчик_айди | `counter_id` | TEXT FK → counter | нет | |
| тип | `type` | TEXT | нет | `CounterEventType` |
| дельта | `delta` | INTEGER | нет | сброс: 0; изменение: не 0 (сейчас в UI ±1) |
| произошло_в | `occurred_at` | INTEGER | нет | когда случилось; по умолчанию = момент нажатия |
| tz_id | `tz_id` | TEXT | нет | пояс устройства при записи |
| создано_в | `created_at` | INTEGER | нет | когда записано |
| удалено_в | `deleted_at` | INTEGER | да | отмена события |
 
### 3.4 `task` — задача
 
| Поле | Колонка | Тип | NULL | Описание |
|---|---|---|---|---|
| айди | `id` | TEXT PK | нет | UUID |
| категория_айди | `category_id` | TEXT FK → category | да | |
| цвет | `color_key` | TEXT | да | `NULL` = цвет категории |
| название | `title` | TEXT | нет | |
| описание | `description` | TEXT | да | |
| тип_повтора | `repeat_type` | TEXT | да | `RepeatType`; `NULL` = разовая |
| повтор_интервал | `repeat_interval` | INTEGER | да | N > 0 («каждые N …») |
| повтор_единица | `repeat_unit` | TEXT | да | `RepeatUnit` |
| повтор_начало | `repeat_start` | TEXT | да | локальная дата, якорь серии (только BY_CALENDAR) |
| повтор_дни_недели | `repeat_weekdays` | TEXT | да | `'MO,WE,FR'` (только BY_CALENDAR + WEEK) |
| повтор_до | `repeat_until` | TEXT | да | локальная дата, включительно |
| следующая_дата | `next_date` | TEXT | да | локальная дата ближайшей копии; для разовой — срок |
| последнее_выполнение | `last_done_at` | INTEGER | да | кэш: UTC мс последнего неудалённого события DONE |
| выполнена_в | `done_at` | INTEGER | да | разовая выполнена или серия закрыта |
| порядок | `sort_order` | INTEGER | нет | по умолчанию 0 |
| создана_в | `created_at` | INTEGER | нет | |
| изменена_в | `updated_at` | INTEGER | нет | |
| архивирована_в | `archived_at` | INTEGER | да | прекратили вести (актуально для NO_DEADLINE) |
| удалена_в | `deleted_at` | INTEGER | да | |

### 3.5 `task_event` — событие задачи
 
| Поле | Колонка | Тип | NULL | Описание |
|---|---|---|---|---|
| айди | `id` | TEXT PK | нет | UUID |
| задача_айди | `task_id` | TEXT FK → task | нет | |
| тип | `type` | TEXT | нет | `TaskEventType` |
| для_даты | `for_date` | TEXT | да | локальная дата закрытой копии (только BY_CALENDAR + DONE) |
| произошло_в | `occurred_at` | INTEGER | нет | UTC мс; сейчас = момент нажатия |
| tz_id | `tz_id` | TEXT | нет | |
| создано_в | `created_at` | INTEGER | нет | |
| удалено_в | `deleted_at` | INTEGER | да | отмена (в MVP только для NO_DEADLINE, см. И-30) |
 
### 3.6 Создать пустыми (UI и логики в MVP нет)
 
`day_note` (заметка дня): `id TEXT PK`, `local_date TEXT NOT NULL UNIQUE`, `body TEXT NOT NULL`, `created_at INTEGER NOT NULL`, `updated_at INTEGER NOT NULL`, `deleted_at INTEGER`.
 
`outbox` (очередь экспорта): `id TEXT PK`, `target TEXT NOT NULL`, `entity_type TEXT NOT NULL`, `entity_id TEXT NOT NULL`, `op TEXT NOT NULL`, `status TEXT NOT NULL`, `attempts INTEGER NOT NULL DEFAULT 0`, `last_error TEXT`, `created_at INTEGER NOT NULL`.

### 3.7 Индексы
 
| Таблица | Индекс |
|---|---|
| `counter_event` | `(counter_id, occurred_at)` |
| `task_event` | `(task_id, occurred_at)` |
| `counter` | `(category_id)` |
| `task` | `(category_id)` |
| `outbox` | `(status, created_at)` |

## 4. Инварианты
 
Проверяются в репозитории при каждой записи. Нарушение → исключение домена, в БД ничего не пишется.

### Общие
 
| № | Правило |
|---|---|
| И-01 | `title`/`name` после `trim()` не пусто. |
| И-02 | `category.name` уникально среди неудалённых, без учёта регистра. |
| И-03 | `color_key` — значение `ColorKey` или `NULL`. |
| И-04 | Эффективный цвет = `color_key` элемента → цвет его категории → нейтральный. Считается в `domain`, не в SQL. |
| И-05 | Удаление категории — одна транзакция: `category_id = NULL` у всех её `counter` и `task`, затем `category.deleted_at = now`. |

### Счётчик
 
| № | Правило |
|---|---|
| И-10 | `goal_target IS NULL` ⇒ `goal_period IS NULL` и `period_start IS NULL`. |
| И-11 | `goal_target IS NOT NULL` ⇒ `goal_target > 0`. |
| И-12 | `goal_period IS NULL` ⇒ `period_start IS NULL`. Цель при этом допустима: значение считается «за всё время» (от последнего сброса). |
| И-13 | `DAY` ⇒ `period_start IS NULL`. `WEEK` ⇒ `period_start` в 1..7, обязателен (по умолчанию 1). `MONTH` ⇒ `period_start` в 1..28, обязателен (по умолчанию 1). |
| И-14 | Событие: `RESET` ⇒ `delta = 0`; `CHANGE` ⇒ `delta ≠ 0`. |
| И-15 | `occurred_at` не в будущем относительно `now` на момент записи. |
| И-16 | Значение счётчика не хранится, только вычисляется (§5.2). |

### Задача
 
| № | Правило |
|---|---|
| И-20 | Разовая (`repeat_type IS NULL`): `repeat_interval`, `repeat_unit`, `repeat_start`, `repeat_weekdays`, `repeat_until` = `NULL`. `next_date` — необязательный срок. |
| И-21 | `NO_DEADLINE`: все `repeat_*` = `NULL`, `next_date = NULL`, `done_at` всегда `NULL`. Задача не исчезает после выполнения. |
| И-22 | `FROM_DONE`: `repeat_interval > 0`, `repeat_unit` не `NULL`, `repeat_start` и `repeat_weekdays` = `NULL`. `next_date` не `NULL`, пока серия открыта (при создании по умолчанию = сегодня). |
| И-23 | `BY_CALENDAR`: `repeat_interval > 0`, `repeat_unit` не `NULL`, `repeat_start` не `NULL`. `next_date` не `NULL`, пока серия открыта. `repeat_weekdays` допустим только при `repeat_unit = WEEK`. |
| И-24 | `repeat_weekdays`: непустое подмножество `MO..SU`, без повторов, порядок пн→вс, разделитель запятая без пробелов. `NULL` = день недели `repeat_start`. |
| И-25 | `repeat_until` ≥ `repeat_start` (BY_CALENDAR) или ≥ `next_date` на момент создания (FROM_DONE). |
| И-26 | Серия закрыта ⇔ `done_at IS NOT NULL`. Закрытую задачу нельзя выполнить, в активном списке её нет. |
| И-27 | `last_done_at` = `MAX(occurred_at)` по неудалённым событиям `DONE`. Обновляется в **той же транзакции**, что вставка/откат события. |
| И-28 | `for_date IS NOT NULL` только у `DONE` задачи `BY_CALENDAR`. У остальных событий `NULL`. |
| И-29 | Разовая закрывается событием `DONE` (+ `done_at`), без `SERIES_END`. Повторяющаяся серия закрывается `DONE` + `SERIES_END` в одной транзакции. |
| И-30 | Откат события (`deleted_at`) в MVP разрешён только для `NO_DEADLINE`. После отката `last_done_at` пересчитывается. |
| И-31 | Активная задача: `deleted_at IS NULL AND archived_at IS NULL AND done_at IS NULL`. Активный счётчик: `deleted_at IS NULL AND archived_at IS NULL`. |

---
 
## 5. Алгоритмы
 
Все — чистые функции `domain/`. `today = LocalDate` из `now` в поясе устройства.
 
### 5.1 Период счётчика
 
Возвращает `[start, end)` в UTC мс.
 
```
DAY:    start = today;  end = today + 1 день
WEEK(W):  start = today.with(previousOrSame(W));  end = start + 7 дней
MONTH(D): start = today.dayOfMonth >= D ? today.withDayOfMonth(D)
                                        : today.minusMonths(1).withDayOfMonth(D)
          end   = start.plusMonths(1)
перевод в мс: start.atStartOfDay(zone).toInstant()   // не «+24 часа»: в день перевода часов длина 23/25 ч
```
 
`D ≤ 28`, поэтому `withDayOfMonth` не бросает исключение и клампинг не нужен (И-13).
 

 ### 5.2 Значение счётчика
 
```
lastResetAt = MAX(occurred_at) по событиям RESET, deleted_at IS NULL    (или NULL)
если goal_period IS NULL:  окно = [lastResetAt или -∞,  +∞)
иначе:                     окно = [max(lastResetAt, period.start),  period.end)
значение = SUM(delta) по событиям CHANGE, deleted_at IS NULL, occurred_at внутри окна
прогресс цели = значение / goal_target   (может быть > 100%, может быть < 0, не ограничивать)
```
 
Смена `period_start` пересчитывает историю сама, миграций не нужно. События, у которых `occurred_at` раньше последнего сброса, в значение не входят.
 
Отмена `+1` в UI: мягко удалить неудалённое событие `CHANGE` с **максимальным `created_at`** (не `occurred_at`: события можно записывать задним числом).
 
### 5.3 Копии серии BY_CALENDAR
 
«Копия» — дата, на которую серия порождает экземпляр задачи. Копии вычисляются **от якоря `repeat_start`**, а не от предыдущей копии (иначе дрейф: 31 янв → 28 фев → 28 мар).
 
Копия допустима, только если `≤ repeat_until` (при заданном). Иначе функции возвращают `null`.
 
**Без дней недели** (`repeat_weekdays IS NULL`), `k ≥ 0`:
 
```
copy(k) = DAY:   repeat_start.plusDays(k·N)
          WEEK:  repeat_start.plusWeeks(k·N)
          MONTH: repeat_start.plusMonths(k·N)     // Java клампит до последнего дня месяца
firstCopyOnOrAfter(d):  наименьшее k ≥ 0 с copy(k) ≥ d   (если d ≤ repeat_start, то k = 0)
lastCopyOnOrBefore(d):  наибольшее k с copy(k) ≤ d        (null, если d < repeat_start)
// реализация: оценить k = floor(единиц_между(repeat_start, d) / N), затем сдвинуть на ±1, пока условие не выполнится
```
 
**С днями недели** (`repeat_unit = WEEK`, `repeat_weekdays` задан). День `x` — копия, если:
 
```
x ≥ repeat_start
и день_недели(x) ∈ repeat_weekdays
и weeksBetween(monday(repeat_start), monday(x)) % N == 0
```
 
```
firstCopyOnOrAfter(d): перебор x от max(d, repeat_start) вверх, не более 7·N + 7 дней
lastCopyOnOrBefore(d): перебор x от d вниз до repeat_start, не более 7·N + 7 дней
```
 
Общие функции:
 
```
firstCopyAfter(d) = firstCopyOnOrAfter(d + 1 день)
```
 
### 5.4 Создание задачи
 
```
BY_CALENDAR:  next_date = firstCopyOnOrAfter(repeat_start)
FROM_DONE:    next_date = дата из формы (по умолчанию today)
NO_DEADLINE:  next_date = NULL
разовая:      next_date = срок из формы или NULL
```
 
Если `next_date` оказалась `null` для BY_CALENDAR (копий нет из-за `repeat_until`) — ошибка валидации формы.
 
### 5.5 Выполнение задачи
 
Одна `@Transaction`. `now` и `tz` из `Clock`.
 
```
complete(task):
  today = дата(now, tz)
  вставить TaskEvent(DONE, occurred_at = now, tz_id = tz, created_at = now, for_date = ?)
  task.last_done_at = now;  task.updated_at = now
 
  repeat_type == NULL:
      task.done_at = now                                  // next_date остаётся как был (срок)
 
  NO_DEADLINE:
      // больше ничего
 
  FROM_DONE:
      n = today + N единиц (plusDays/plusWeeks/plusMonths)
      если repeat_until != NULL и n > repeat_until → closeSeries
      иначе next_date = n
 
  BY_CALENDAR:
      closed = next_date > today ? next_date                       // досрочно
                                 : lastCopyOnOrBefore(today)       // самая свежая наступившая копия
      for_date события = closed
      n = firstCopyAfter(closed)
      если n == NULL → closeSeries
      иначе next_date = n
 
closeSeries:
  task.done_at = now;  task.next_date = NULL
  вставить TaskEvent(SERIES_END, occurred_at = now, tz_id = tz, created_at = now, for_date = NULL)
```
 
Пропущенные копии (между `repeat_start` и `closed` без события `DONE` с этой `for_date`) не хранятся, а вычисляются при необходимости. Для будущего экрана «выполнить прошлые копии».
 
### 5.6 Статусы для отображения
 
```
просрочена: next_date != NULL и next_date < today
сегодня:    next_date == today
позже:      next_date > today
без срока:  next_date IS NULL (разовая без срока и NO_DEADLINE)
давность:   now - last_done_at   (показывать у всех типов, где last_done_at не NULL)
```
 
---
 
## 6. Контракт доступа к данным (MVP)
 
| Операция | Реализация |
|---|---|
| Список активных счётчиков | `Flow`, фильтр И-31, сортировка `sort_order, created_at` |
| `lastResetAt(counterId)` | `MAX(occurred_at)` по `RESET`, `deleted_at IS NULL` |
| `sumDelta(counterId, fromMs, toMs)` | `SUM(delta)` по `CHANGE`, `deleted_at IS NULL`, `occurred_at ≥ from AND < to` |
| Добавить событие счётчика | вставка по И-14, И-15 |
| Отменить последнее `CHANGE` | §5.2 |
| Список активных задач | `Flow`, фильтр И-31 |
| Выполнить задачу | §5.5, одна транзакция |
| Откатить выполнение | только NO_DEADLINE (И-30), пересчёт `last_done_at` |
 
Значение счётчика на главном экране: один запрос списка + по одному индексному запросу `lastResetAt` и `sumDelta` на счётчик. Для десятков счётчиков это нормально, оптимизировать до измеренной проблемы не нужно.
 
---
 
## 7. Тестовые примеры
 
Опорная дата для календаря: **2026-10-17 — суббота**. Понедельники: 09-21, 09-28, 10-05, 10-12, 10-19, 10-26, 11-02, 11-09. Четверги: 10-15, 10-22, 10-29.
 
### Копии и выполнение
 
| № | Вход | Ожидание |
|---|---|---|
| Т-01 | MONTH, N=1, начало 2026-01-31 | copy(0..3) = 01-31, 02-28, 03-31, 04-30 |
| Т-02 | MONTH, N=2, начало 2026-01-31 | copy(0..3) = 01-31, 03-31, 05-31, 07-31 |
| Т-03 | WEEK, N=1, MO+TH, начало ср 2026-10-14 | `firstCopyOnOrAfter(10-14)` = 10-15; `firstCopyAfter(10-15)` = 10-19; `firstCopyAfter(10-19)` = 10-22 |
| Т-04 | WEEK, N=2, MO+TH, начало ср 2026-10-14 | `firstCopyOnOrAfter(10-14)` = 10-15; затем 10-26, 10-29, 11-09 |
| Т-05 | Т-03 с `repeat_until = 2026-10-19` | `firstCopyAfter(10-19)` = `null` |
| Т-06 | Т-03 и Т-04 | `lastCopyOnOrBefore(10-17)` = 10-15; `lastCopyOnOrBefore(10-14)` = `null` |
| Т-07 | WEEK, N=1, без дней, начало пн 09-21, `next_date` = 09-21, today = 10-17 | закрыта копия 10-12 (`for_date`), новая `next_date` = 10-19 |
| Т-08 | та же серия, `next_date` = 10-19, today = 10-17 | досрочно: `for_date` = 10-19, новая `next_date` = 10-26 |
| Т-09 | Т-07 с `repeat_until = 2026-10-18` | серия закрыта: `done_at` задан, `next_date` = `null`, есть событие `SERIES_END` |
| Т-10 | FROM_DONE, MONTH, N=1 | today 2026-01-31 → `next_date` 2026-02-28; today 2026-02-28 → 2026-03-28 (дрейф допустим: режим не якорный) |
| Т-11 | разовая | `DONE` + `done_at`, события `SERIES_END` нет |
| Т-12 | NO_DEADLINE | два выполнения подряд → два события, `done_at = NULL`, `last_done_at` = момент второго; откат последнего → `last_done_at` = момент первого |
 
### Счётчики
 
| № | Вход | Ожидание |
|---|---|---|
| Т-20 | MONTH, `period_start = 15` | today 10-17 → `[10-15, 11-15)`; today 10-14 → `[09-15, 10-15)`; today 10-15 → `[10-15, 11-15)` |
| Т-21 | MONTH, `period_start = 1`, today 2026-02-10 | `[02-01, 03-01)` |
| Т-22 | WEEK, today сб 10-17 | `period_start = 1` → `[10-12, 10-19)`; `period_start = 7` → `[10-11, 10-18)`; today пн 10-12, `period_start = 1` → `[10-12, 10-19)` |
| Т-23 | DAY, зона Europe/Warsaw, today 2026-10-25 (конец летнего времени) | длина окна 25 часов |
| Т-24 | MONTH, `period_start = 1`, today 10-17; события: `+1` 10-03, `+1` 10-05, `RESET` 10-08, `+1` 10-10, `+1` 10-12 | значение = 2 |
| Т-25 | `goal_target = 100`, `goal_period = NULL`, есть `RESET` | окно от последнего сброса без конца |
| Т-26 | `CHANGE` с `occurred_at` раньше последнего сброса | в значение не входит |
| Т-27 | смена `period_start` с 1 на 15 | окно пересчитано, миграция не нужна |
| Т-28 | отмена `+1` | удаляется событие с максимальным `created_at`, а не `occurred_at` |
 
---
 
## 8. Порядок работ
 
| Этап | Что | Результат |
|---|---|---|
| Э0 | Проект: Kotlin, Compose, Room + KSP, пакеты `data/`, `domain/`, `ui/` | собирается |
| Э1 | Room: entities, DAO, DB v1 для **всех** таблиц §3, JSON-схема | схема сгенерирована |
| Э2 | `domain/`: §5.1, §5.2 + тесты Т-20…Т-28 | зелёные тесты |
| Э3 | Репозиторий счётчиков, ViewModel, главный экран (карточки, +1/−1, сброс, цель), экран создания | **можно пользоваться** |
| Э4 | Задачи: разовые, `FROM_DONE`, `NO_DEADLINE` (§5.4, §5.5), тесты Т-10…Т-12 | «когда последний раз поливал цветы» работает |
| Э5 | `BY_CALENDAR` и дни недели (§5.3) + тесты Т-01…Т-09 | повторы |
 
Правило: одна задача исполнителю = один этап, одна ветка. Следующий этап только после проверки.
 
---
 
## 9. Вне MVP
 
Категории в UI (таблица уже есть), заметки дня, outbox и экспорт, отмена выполнения для разовых и повторяющихся задач, экран прошлых копий, скользящие окна («последние 30 дней»), «день начинается в 4 утра», задачи со временем суток и напоминания.
 
## 10. Известные риски
 
- **Двойной тап по BY_CALENDAR.** Второе нажатие закрывает следующую копию досрочно (§5.5). Защита в UI: если `next_date > today`, требовать подтверждение. Отката в MVP нет.
- **`last_done_at` — кэш.** Источник правды — события. При подозрении на расхождение пересчитать из событий.
- **N запросов на счётчики.** Осознанный компромисс (§6).

## 11. Решения для ADR
 
1. Локальная БД — источник правды. 2. UUID, генерируемые клиентом. 3. Момент = UTC мс, дата = локальная строка. 4. События, а не хранимые счётчики. 5. Палитра — константа в коде, в БД ключ. 6. Копии серии считаются от якоря, не итеративно. 7. Период счётчика: календарный, число сброса 1..28.