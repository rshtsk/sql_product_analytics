# Экономика продукта — Задача 2

## Анализ выручки на пользователя и платящего клиента, средний чек

**Цель:**  
Рассчитать ключевые метрики выручки для оценки платёжной активности и поведения клиентов.

---

## Задача

Для каждого дня необходимо рассчитать:

- **ARPU** — средняя выручка на одного пользователя;
- **ARPPU** — средняя выручка на одного платящего пользователя;
- **AOV** — средний чек (Average Order Value).

### Требуемые поля:

- `date` — дата;
- `arpu` — выручка на пользователя;
- `arppu` — выручка на платящего пользователя;
- `aov` — средний чек.

---

## Подход

- **Выручка** считается на основе данных из заказов и таблицы продуктов.
- **Платящими пользователями** считаются те, кто создал хотя бы один неотменённый заказ.
- Для AOV используется средняя сумма заказа за день.

---

## SQL-запрос

```sql
WITH orders_product_id AS (
    SELECT creation_time,
           order_id,
           unnest(product_ids) AS product_id
    FROM orders
    WHERE order_id NOT IN (
        SELECT order_id FROM user_actions WHERE action = 'cancel_order'
    )
),
avg_revenue AS (
    SELECT date,
           AVG(order_price) AS aov,
           SUM(order_price) AS revenue
    FROM (
        SELECT creation_time::date AS date,
               order_id,
               SUM(price) AS order_price
        FROM (
            SELECT creation_time,
                   order_id,
                   l.product_id,
                   price
            FROM orders_product_id AS l
            LEFT JOIN products AS r ON l.product_id = r.product_id
        ) t1
        GROUP BY date, order_id
    ) t2
    GROUP BY date
),
total_users_t AS (
    SELECT time::date AS date,
           COUNT(DISTINCT user_id) AS total_users
    FROM user_actions
    GROUP BY date
),
paying_users_t AS (
    SELECT time::date AS date,
           COUNT(DISTINCT user_id) AS paying_users
    FROM user_actions
    WHERE order_id NOT IN (
        SELECT order_id FROM user_actions WHERE action = 'cancel_order'
    )
    GROUP BY date
)
SELECT l.date,
       ROUND(revenue::numeric / total_users, 2) AS arpu,
       ROUND(revenue::numeric / paying_users, 2) AS arppu,
       ROUND(aov, 2) AS aov
FROM avg_revenue AS l
JOIN total_users_t AS r ON l.date = r.date
JOIN paying_users_t AS rr ON l.date = rr.date
ORDER BY date
```

## Визуализация

![ARPU, ARPPU, AOV](../img/economika_task_2_viz_1.png


## Выводы

- ARPPU стабильно выше ARPU, что говорит о наличии большого числа пользователей, не совершающих оплат.
- AOV остаётся достаточно стабильным, с незначительными колебаниями.
- В пиковые дни наблюдается резкое увеличение ARPPU и ARPU, вероятно из-за более высокой активности или крупной покупки.


