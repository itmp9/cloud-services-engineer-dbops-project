# dbops-project

Проектная работа дисциплины "DBOps".

## Подготовка базы данных

Создана отдельная база данных `store`:

```sql
CREATE DATABASE store;
```

Создан пользователь для запуска Flyway-миграций и автотестов:

```sql
CREATE USER dbops_user WITH PASSWORD 'dbops_password';
```

Пользователю выданы права на базу данных `store`:

```sql
GRANT ALL PRIVILEGES ON DATABASE store TO dbops_user;
```

Права на схему `public` в базе `store`:

```sql
\c store

GRANT ALL ON SCHEMA public TO dbops_user;
ALTER SCHEMA public OWNER TO dbops_user;
```

## GitHub Secrets

Для workflow нужны секреты:

```text
DB_HOST=89.169.139.72
DB_PORT=5432
DB_NAME=store
DB_USER=dbops_user
DB_PASSWORD=dbops_password
```

## Миграции

В директории `migrations` создана миграция `V001__create_tables.sql`. Она описывает исходную структуру таблиц:

- `product`;
- `product_info`;
- `orders`;
- `orders_date`;
- `order_product`.

Миграция запускается в GitHub Actions через Flyway перед автотестами.

## Нормализация

Таблицы `product` и `product_info` частично дублировали друг друга: обе хранили идентификатор и название товара. Для нормализации в таблицу `product` добавлено поле `price`, после чего таблица `product_info` удаляется.

Таблицы `orders` и `orders_date` также частично дублировали друг друга: обе хранили идентификатор и статус заказа. Для нормализации в таблицу `orders` добавлено поле `date_created`, после чего таблица `orders_date` удаляется.

Миграция `V002__change_schema.sql` изменяет структуру базы:

- добавляет `product.price`;
- добавляет `orders.date_created`;
- добавляет первичные ключи для `product` и `orders`;
- добавляет связи `order_product.order_id -> orders.id` и `order_product.product_id -> product.id`;
- удаляет неиспользуемые таблицы `product_info` и `orders_date`.

Миграция `V003__insert_data.sql` заполняет нормализованные таблицы тестовыми данными.

## Индексы

Для ускорения запроса по проданным сосискам за предыдущую неделю добавлена миграция `V004__create_index.sql`.

Она создает индексы:

- `idx_orders_status_date_created` для фильтрации заказов по статусу и дате;
- `idx_order_product_order_id` для соединения заказов с товарами в заказах.

## Запрос по продажам за предыдущую неделю

Количество проданных сосисок по дням за предыдущую неделю:

```sql
SELECT o.date_created, SUM(op.quantity) AS total_quantity
FROM orders AS o
JOIN order_product AS op ON o.id = op.order_id
WHERE o.status = 'shipped'
  AND o.date_created > now() - interval '7 days'
GROUP BY o.date_created
ORDER BY o.date_created;
```

## Сравнение производительности

До создания индексов запрос выполнялся примерно 6 секунд:

```text
Time: 5826.138 ms
```

После добавления индексов:

```sql
\timing

SELECT o.date_created, SUM(op.quantity) AS total_quantity
FROM orders AS o
JOIN order_product AS op ON o.id = op.order_id
WHERE o.status = 'shipped'
  AND o.date_created > now() - interval '7 days'
GROUP BY o.date_created
ORDER BY o.date_created;
```

Результат выполнения:

```text
 date_created | total_quantity
--------------+----------------
 2026-05-26   |         950650
 2026-05-27   |         944520
 2026-05-28   |         938622
 2026-05-29   |         958025
 2026-05-30   |         939832
 2026-05-31   |         943742
 2026-06-01   |         467641
(7 rows)

Time: 5602.802 ms
```

План запроса после создания индексов показывает использование индекса `idx_orders_status_date_created`:

```sql
EXPLAIN ANALYZE
SELECT o.date_created, SUM(op.quantity) AS total_quantity
FROM orders AS o
JOIN order_product AS op ON o.id = op.order_id
WHERE o.status = 'shipped'
  AND o.date_created > now() - interval '7 days'
GROUP BY o.date_created
ORDER BY o.date_created;
```
