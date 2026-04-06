Ниже приведено **пошаговое решение** для выполнения всех 5 заданий в **pgAdmin** (PostgreSQL).  
Код можно запускать последовательно, каждый пункт — отдельным запросом или командой.

---

## 1. День недели, когда ВСЕ сотрудники города работают в офисе

```sql
WITH city_days AS (
    SELECT
        city,
        unnest(office_days) AS day_num
    FROM employees
),
city_all_days AS (
    SELECT
        city,
        day_num,
        COUNT(DISTINCT id) AS cnt_in_day,
        (SELECT COUNT(*) FROM employees e2 WHERE e2.city = e.city) AS total_in_city
    FROM city_days cd
    JOIN employees e USING(city)
    GROUP BY city, day_num
)
SELECT
    city,
    COALESCE(
        (SELECT
            CASE day_num
                WHEN 1 THEN 'Понедельник'
                WHEN 2 THEN 'Вторник'
                WHEN 3 THEN 'Среда'
                WHEN 4 THEN 'Четверг'
                WHEN 5 THEN 'Пятница'
            END
         FROM city_all_days cad2
         WHERE cad2.city = cad.city
           AND cad2.cnt_in_day = cad2.total_in_city
         LIMIT 1
        ),
        'Такого дня нет'
    ) AS day_name
FROM (SELECT DISTINCT city FROM employees) cities
LEFT JOIN city_all_days cad USING(city)
GROUP BY city;
```

> **Пояснение**:  
> Считаем для каждого города и дня недели, сколько сотрудников работают в офисе.  
> Если это число равно общему числу сотрудников города — день подходит.  
> Если ни одного такого дня — выводим «Такого дня нет».

---

## 2. Материализованное представление с зарплатным фондом

```sql
CREATE MATERIALIZED VIEW salary_fund AS
SELECT
    COALESCE(city, 'Все города') AS city,
    COALESCE(department, 'Все отделы') AS department,
    COALESCE(grade, 'Все грейды') AS grade,
    SUM(salary) AS total_salary
FROM employees
GROUP BY ROLLUP(city, department, grade)
ORDER BY 
    CASE WHEN city = 'Все города' THEN 2 ELSE 1 END,
    city NULLS LAST,
    CASE WHEN department = 'Все отделы' THEN 2 ELSE 1 END,
    department NULLS LAST,
    CASE WHEN grade = 'Все грейды' THEN 2 ELSE 1 END,
    grade NULLS LAST;
```

Обновить представление (если данные изменятся):
```sql
REFRESH MATERIALIZED VIEW salary_fund;
```

Посмотреть результат:
```sql
SELECT * FROM salary_fund ORDER BY city, department, grade;
```

---

## 3. Иерархия начальников для каждого сотрудника (рекурсивный CTE)

```sql
WITH RECURSIVE hierarchy AS (
    SELECT
        id,
        name,
        manager_id,
        name::text AS path,
        1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.id,
        e.name,
        e.manager_id,
        h.path || ' -> ' || e.name,
        h.level + 1
    FROM employees e
    JOIN hierarchy h ON e.manager_id = h.id
)
SELECT
    name AS employee_name,
    path AS hierarchy_chain
FROM hierarchy
WHERE level > 1  -- чтобы не показывать начальников без подчинённых как самих себя
ORDER BY level, name;
```

> **Формат**: Имя_начальника -> ... -> Имя_сотрудника

---

## 4. Добавить телефон и сменить tg у Бориса, затем вывести номера, оканчивающиеся на «34»

```sql
-- Обновление контактов Бориса
UPDATE employees
SET contacts = jsonb_set(
    jsonb_set(contacts, '{phone}', (contacts->'phone') || '["89997778834"]'::jsonb),
    '{tg}',
    '["@already_senior"]'::jsonb
)
WHERE name = 'Борис';

-- Вывод сотрудников, у которых номер телефона заканчивается на «34»
SELECT
    name,
    jsonb_array_elements_text(contacts->'phone') AS phone_numbers
FROM employees
WHERE EXISTS (
    SELECT 1
    FROM jsonb_array_elements_text(contacts->'phone') AS phone
    WHERE phone LIKE '%34'
);
```

> Если нужно **все номера в одной строке** для сотрудника:
```sql
SELECT
    name,
    string_agg(jsonb_array_elements_text(contacts->'phone'), ', ') AS all_phones
FROM employees
WHERE EXISTS (
    SELECT 1
    FROM jsonb_array_elements_text(contacts->'phone') AS phone
    WHERE phone LIKE '%34'
)
GROUP BY name;
```

---

## 5. Функция department_leaderboard(f_city TEXT)

```sql
CREATE OR REPLACE FUNCTION department_leaderboard(f_city TEXT)
RETURNS TABLE(
    place BIGINT,
    department_name TEXT,
    total_salary BIGINT,
    employee_count BIGINT
) AS $$
BEGIN
    RETURN QUERY
    WITH dept_stats AS (
        SELECT
            department,
            SUM(salary) AS sum_salary,
            COUNT(*) AS cnt
        FROM employees
        WHERE city = f_city
        GROUP BY department
    ),
    ranked AS (
        SELECT
            department,
            sum_salary,
            cnt,
            DENSE_RANK() OVER (ORDER BY sum_salary DESC) AS rn
        FROM dept_stats
    )
    SELECT
        rn::BIGINT,
        department,
        sum_salary,
        cnt
    FROM ranked
    ORDER BY rn, department;
END;
$$ LANGUAGE plpgsql;
```

Пример вызова:
```sql
SELECT * FROM department_leaderboard('Москва');
```

---

## Как выполнять в pgAdmin

1. Откройте **Query Tool** (инструмент запросов).
2. Вставьте код из **пункта 1**, нажмите **Execute (F5)**.
3. Для **пункта 2** — выполните `CREATE MATERIALIZED VIEW`, затем `SELECT`.
4. Для **пункта 3** — выполните `WITH RECURSIVE` запрос.
5. Для **пункта 4** — сначала `UPDATE`, потом `SELECT`.
6. Для **пункта 5** — выполните `CREATE FUNCTION`, затем вызовите её.

Если какой-то шаг вызывает ошибку — проверьте, что таблица `employees` создана и заполнена (ваш файл `sr1.sql.txt` это делает).

Если нужны пояснения по любому пункту — спрашивайте, я подробно разберу.