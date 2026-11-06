Я считаю GMV по успешно доставленным заказам (`success_order_flg = TRUE`), т.к. в описании отдельно введено понятие выполненного заказа. Если преподаватель ожидает GMV по всем заказам, нужно убрать фильтр `o.success_order_flg = TRUE`.

## 1. Создать таблицу `orders`

```sql
DROP TABLE IF EXISTS orders;

CREATE TABLE orders (
    user_id          text        NOT NULL,
    order_id         text        PRIMARY KEY,
    order_time       bigint      NOT NULL, -- unixtime, секунды UTC
    order_cost       double precision NOT NULL,
    success_order_flg boolean    NOT NULL
);
```

## 2. Заполнить таблицу 10 «ручными» событиями

Сценарии подобраны так, чтобы были:
- первые заказы;
- повторные заказы в пределах 90 дней;
- реактивации после паузы больше 90 дней;
- неуспешный заказ;
- дата, где одновременно есть новый клиент и реактивированный.

```sql
INSERT INTO orders (user_id, order_id, order_time, order_cost, success_order_flg)
VALUES
('U1', 'O001', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-01-01 10:00:00+00')::bigint, 100.0, TRUE),
('U1', 'O002', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-01-10 12:00:00+00')::bigint,  50.0, TRUE),
('U1', 'O003', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-04-15 09:00:00+00')::bigint, 200.0, TRUE),

('U2', 'O004', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-01-01 15:00:00+00')::bigint, 300.0, TRUE),
('U2', 'O005', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-02-01 11:00:00+00')::bigint,  70.0, TRUE),

('U3', 'O006', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-01-15 08:00:00+00')::bigint, 400.0, TRUE),
('U3', 'O007', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-06-01 10:00:00+00')::bigint,  90.0, TRUE),

('U4', 'O008', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-01-01 18:00:00+00')::bigint, 500.0, FALSE),
('U4', 'O009', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-01-05 14:00:00+00')::bigint,  80.0, TRUE),

('U5', 'O010', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-04-15 16:00:00+00')::bigint,  60.0, TRUE);
```

## 3. Проверить конвертацию unixtime в нормальную дату

```sql
SELECT
    order_id,
    user_id,
    order_time,
    to_timestamp(order_time) AT TIME ZONE 'UTC' AS order_ts_utc,
    (to_timestamp(order_time) AT TIME ZONE 'UTC')::date AS order_date,
    order_cost,
    success_order_flg
FROM orders
ORDER BY order_time, order_id;
```

`AT TIME ZONE 'UTC'` нужен, чтобы дата не зависела от часового пояса сессии.

## 4. Создать результирующую таблицу

Логика запроса:
1. Переводим `order_time` в дату UTC.
2. Берём по каждому пользователю уникальные дни заказов, чтобы несколько заказов в один день не ломали определение реактивации.
3. Через `LAG` находим предыдущий день заказа.
4. Первый день пользователя — `new`.
5. День, где разница с предыдущим днём заказа больше 90 дней — `reactivated`. Именно `> 90`, а не `>= 90`.
6. Для каждого события `new`/`reactivated` определяем следующий эпизод активности пользователя. Это нужно, чтобы заказы после реактивации не попадали в GMV предыдущего периода.
7. Считаем GMV за 360 дней как `[episode_start, episode_start + 360 дней)` и только до следующего эпизода активности.
8. Агрегируем по дате события.

```sql
DROP TABLE IF EXISTS cohort_gmv;

CREATE TABLE cohort_gmv AS
WITH order_dates AS (
    SELECT
        user_id,
        order_id,
        order_cost,
        success_order_flg,
        (to_timestamp(order_time) AT TIME ZONE 'UTC')::date AS order_date
    FROM orders
),
user_days AS (
    SELECT DISTINCT
        user_id,
        order_date
    FROM order_dates
),
with_prev AS (
    SELECT
        user_id,
        order_date,
        LAG(order_date) OVER (
            PARTITION BY user_id
            ORDER BY order_date
        ) AS prev_order_date,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY order_date
        ) AS rn
    FROM user_days
),
events AS (
    SELECT
        user_id,
        order_date AS event_date,
        CASE
            WHEN rn = 1 THEN 'new'
            WHEN order_date - prev_order_date > 90 THEN 'reactivated'
            ELSE NULL
        END AS event_type
    FROM with_prev
),
episodes AS (
    SELECT
        user_id,
        event_date AS episode_start,
        event_type,
        LEAD(event_date) OVER (
            PARTITION BY user_id
            ORDER BY event_date
        ) AS next_episode_start
    FROM events
    WHERE event_type IS NOT NULL
),
episode_orders AS (
    SELECT
        e.user_id,
        e.episode_start,
        e.event_type,
        o.order_cost
    FROM episodes e
    JOIN order_dates o
      ON o.user_id = e.user_id
     AND o.order_date >= e.episode_start
     AND (e.next_episode_start IS NULL OR o.order_date < e.next_episode_start)
     AND o.order_date < e.episode_start + 360
    WHERE o.success_order_flg = TRUE
),
episode_gmv AS (
    SELECT
        user_id,
        episode_start,
        event_type,
        SUM(order_cost) AS gmv360d
    FROM episode_orders
    GROUP BY user_id, episode_start, event_type
)
SELECT
    e.episode_start AS "date",
    COALESCE(SUM(CASE WHEN e.event_type = 'new' THEN eg.gmv360d ELSE 0 END), 0) AS gmv360d_new,
    COALESCE(SUM(CASE WHEN e.event_type = 'reactivated' THEN eg.gmv360d ELSE 0 END), 0) AS gmv360d_reactivated,
    COUNT(DISTINCT CASE WHEN e.event_type = 'new' THEN e.user_id END) AS users_count_new,
    COUNT(DISTINCT CASE WHEN e.event_type = 'reactivated' THEN e.user_id END) AS users_count_reactivated
FROM episodes e
LEFT JOIN episode_gmv eg
  ON eg.user_id = e.user_id
 AND eg.episode_start = e.episode_start
 AND eg.event_type = e.event_type
GROUP BY e.episode_start;
```

## 5. Посмотреть результат

```sql
SELECT *
FROM cohort_gmv
ORDER BY "date";
```

Ожидаемый результат на этих 10 событиях:

| date       | gmv360d_new | gmv360d_reactivated | users_count_new | users_count_reactivated |
|------------|------------:|--------------------:|----------------:|------------------------:|
| 2024-01-01 | 600         | 0                   | 3               | 0                       |
| 2024-01-15 | 400         | 0                   | 1               | 0                       |
| 2024-04-15 | 60          | 200                 | 1               | 1                       |
| 2024-06-01 | 0           | 90                  | 0               | 1                       |

Проверка ключевых моментов:
- `U1` привлечён 2024-01-01, заказы 01.01 и 10.01 дают `100 + 50 = 150`; заказ 15.04 уже относится к реактивации и не попадает в `gmv360d_new` за 01.01.
- `U3` привлечён 15.01, заказ 01.06 — реактивация, поэтому в `gmv360d_new` за 15.01 попадает только `400`.
- `U4` привлечён 01.01, но первый заказ неуспешный; в GMV попадает только успешный заказ 05.01 на `80`.
- 15.04 одновременно есть новый `U5` и реактивированный `U1`, поэтому в одной строке заполнены обе группы метрик.

## 6. Что важно проговорить на защите

- `user_days` нужен, чтобы несколько заказов в один день не считались разными событиями активности.
- Реактивация: `order_date - prev_order_date > 90`.
- `next_episode_start` реализует требование «заказы после реактивации не учитываются в GMV предыдущих периодов».
- GMV считается по успешным заказам, если следовать определению выполненного заказа. Для GMV по всем заказам нужно убрать `WHERE o.success_order_flg = TRUE`.
- Окно 360 дней взято как `[date, date + 360 дней)`. Если преподаватель требует включать 360-й день, замените условие `o.order_date < e.episode_start + 360` на `o.order_date <= e.episode_start + 360`.

## 7. Возможные вопросы

### Теория

1. Какие поля содержит таблица `orders` и зачем нужен `success_order_flg`?
**Ответ:**
`user_id` — идентификатор пользователя, `order_id` — идентификатор заказа, `order_time` — время заказа в unixtime UTC, `order_cost` — стоимость заказа, `success_order_flg` — признак успешной доставки.
`success_order_flg` нужен, чтобы отличать выполненные заказы от неуспешных. В лабораторной GMV считается по `success_order_flg = TRUE`.

2. Почему `order_time` хранится как `bigint`, и как перевести его в календарную дату?
**Ответ:**
`bigint` хранит unixtime — количество секунд с 1970-01-01 UTC. Для перевода используется:
```sql
(to_timestamp(order_time) AT TIME ZONE 'UTC')::date
```
`to_timestamp` даёт `timestamptz`, а `AT TIME ZONE 'UTC'` фиксирует UTC, чтобы дата не зависела от часового пояса сессии.

3. Зачем в CTE `user_days` используется `SELECT DISTINCT user_id, order_date`?
**Ответ:**
Чтобы несколько заказов одного пользователя в один день не считались разными событиями активности. Для определения первого заказа и реактивации важны уникальные дни заказов, а не количество заказов внутри дня.

4. Что делает `LAG(order_date)` в CTE `with_prev`?
**Ответ:**
`LAG` возвращает предыдущую дату заказа для этого же пользователя при сортировке по `order_date`. Это нужно, чтобы найти паузу между текущим и предыдущим днём заказа и определить реактивацию.

5. Зачем нужен `ROW_NUMBER()` в `with_prev`?
**Ответ:**
`ROW_NUMBER()` нумерует уникальные дни заказов пользователя по возрастанию даты. Строка с `rn = 1` — первый заказ пользователя, то есть день привлечения `new`.

6. Почему реактивация проверяется условием `order_date - prev_order_date > 90`, а не `>= 90`?
**Ответ:**
По условию клиент реактивирован, если не совершал заказов в течение предшествующих 90 дней. Если предыдущий заказ был ровно 90 дней назад, он ещё попадает в 90-дневное окно, поэтому реактивации нет. Реактивация — когда пауза строго больше 90 дней. Если бизнес-требование изменится, условие можно заменить на `>= 90`.

7. Что формирует CTE `events`?
**Ответ:**
`events` оставляет только значимые события пользователя: первый день заказа помечается как `new`, а день после паузы больше 90 дней — как `reactivated`. Остальные дни получают `NULL` и отфильтровываются.

8. Зачем нужен CTE `episodes` и поле `next_episode_start`?
**Ответ:**
`episodes` задаёт эпизоды активности пользователя: начало эпизода — это `new` или `reactivated`, а `next_episode_start` — начало следующего такого события. Это позволяет не учитывать заказы после реактивации в GMV предыдущего периода активности.

9. Почему в `episode_orders` стоит фильтр `o.success_order_flg = TRUE`?
**Ответ:**
Потому что GMV в лабораторной считается по выполненным заказам, а выполненный заказ — это заказ с `success_order_flg = TRUE`. Если преподаватель просит GMV по всем заказам, этот фильтр нужно убрать.

10. Зачем в итоговом запросе используется `LEFT JOIN episode_gmv` и `COUNT(DISTINCT ...)`?
**Ответ:**
`LEFT JOIN` нужен, чтобы не потерять дату события, даже если у пользователя нет успешных заказов и GMV равна 0 или `NULL`. `COUNT(DISTINCT user_id)` считает уникальных клиентов отдельно для `new` и `reactivated`, чтобы один пользователь не посчитался дважды.

---

### Практика

11. Создайте таблицу `orders` по описанию.
**Решение:**
```sql
DROP TABLE IF EXISTS orders;

CREATE TABLE orders (
    user_id          text        NOT NULL,
    order_id         text        PRIMARY KEY,
    order_time       bigint      NOT NULL,
    order_cost       double precision NOT NULL,
    success_order_flg boolean    NOT NULL
);
```

12. Добавьте пользователя `U6`: первый заказ 2024-07-01 на 120 успешно, реактивация 2024-10-15 на 80 успешно.
**Решение:**
```sql
INSERT INTO orders (user_id, order_id, order_time, order_cost, success_order_flg)
VALUES
('U6', 'O011', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-07-01 10:00:00+00')::bigint, 120.0, TRUE),
('U6', 'O012', EXTRACT(EPOCH FROM TIMESTAMPTZ '2024-10-15 12:00:00+00')::bigint,  80.0, TRUE);
```
Пауза между 01.07 и 15.10 больше 90 дней, поэтому второй заказ будет реактивацией.

13. Измените расчёт GMV так, чтобы учитывались все заказы, а не только успешные.
**Решение:**
В CTE `episode_orders` удалить условие:
```sql
WHERE o.success_order_flg = TRUE
```
То есть join остаётся таким:
```sql
JOIN order_dates o
  ON o.user_id = e.user_id
 AND o.order_date >= e.episode_start
 AND (e.next_episode_start IS NULL OR o.order_date < e.next_episode_start)
 AND o.order_date < e.episode_start + 360
```

14. Измените порог реактивации с 90 на 60 дней.
**Решение:**
В CTE `events` заменить:
```sql
WHEN order_date - prev_order_date > 90 THEN 'reactivated'
```
на:
```sql
WHEN order_date - prev_order_date > 60 THEN 'reactivated'
```

15. Сделайте окно 360 дней включающим 360-й день.
**Решение:**
В CTE `episode_orders` заменить:
```sql
AND o.order_date < e.episode_start + 360
```
на:
```sql
AND o.order_date <= e.episode_start + 360
```

16. Добавьте в результирующую таблицу `cohort_gmv` колонку `gmv360d_total`.
**Решение:**
```sql
ALTER TABLE cohort_gmv
ADD COLUMN gmv360d_total double precision;

UPDATE cohort_gmv
SET gmv360d_total = gmv360d_new + gmv360d_reactivated;
```

17. Создайте результирующую таблицу `cohort_gmv` с явными типами и первичным ключом по дате.
**Решение:**
```sql
DROP TABLE IF EXISTS cohort_gmv;

CREATE TABLE cohort_gmv (
    "date" date PRIMARY KEY,
    gmv360d_new double precision NOT NULL DEFAULT 0,
    gmv360d_reactivated double precision NOT NULL DEFAULT 0,
    users_count_new integer NOT NULL DEFAULT 0,
    users_count_reactivated integer NOT NULL DEFAULT 0
);
```

18. Выведите даты заказов в часовом поясе `Europe/Moscow`.
**Решение:**
```sql
SELECT
    order_id,
    user_id,
    order_time,
    (to_timestamp(order_time) AT TIME ZONE 'Europe/Moscow')::date AS order_date_msk
FROM orders
ORDER BY order_time;
```

19. Создайте индекс, который поможет запросам с оконными функциями и join по пользователю и времени.
**Решение:**
```sql
CREATE INDEX idx_orders_user_time
ON orders (user_id, order_time);
```
Он полезен для сортировки/поиска заказов по пользователю и времени.

20. Измените итоговый запрос так, чтобы выводились только даты, где были реактивации.
**Решение:**
Если считать напрямую из `episodes`, нужно добавить `HAVING` после `GROUP BY`:
```sql
HAVING COUNT(DISTINCT CASE WHEN e.event_type = 'reactivated' THEN e.user_id END) > 0
```
Если уже есть готовая таблица `cohort_gmv`, можно так:
```sql
SELECT *
FROM cohort_gmv
WHERE users_count_reactivated > 0
ORDER BY "date";
```
