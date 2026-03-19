-- Перевіряємо перші 10 рядків користувачів
SELECT user_id, signup_datetime, promo_signup_flag 
FROM cohort_users_raw 
LIMIT 10;


-- 1. Очищуємо дані користувачів
WITH users_cleaned AS (
    SELECT 
        user_id,
        promo_signup_flag,
        CASE 
            WHEN TO_DATE(REPLACE(REPLACE(SPLIT_PART(TRIM(signup_datetime), ' ', 1), '.', '-'), '/', '-'), 'DD-MM-YY') < '2000-01-01'
            THEN TO_DATE(REPLACE(REPLACE(SPLIT_PART(TRIM(signup_datetime), ' ', 1), '.', '-'), '/', '-'), 'DD-MM-YY') + INTERVAL '2000 years'
            ELSE TO_DATE(REPLACE(REPLACE(SPLIT_PART(TRIM(signup_datetime), ' ', 1), '.', '-'), '/', '-'), 'DD-MM-YY')
        END AS signup_date
    FROM cohort_users_raw
    WHERE signup_datetime IS NOT NULL
),
-- 2. Очищуємо та фільтруємо події (додаємо revenue)
events_cleaned AS (
    SELECT 
        user_id,
        event_type,
        revenue, -- ДОДАНО ПОЛЕ
        CASE 
            WHEN TO_DATE(REPLACE(REPLACE(SPLIT_PART(TRIM(event_datetime), ' ', 1), '.', '-'), '/', '-'), 'DD-MM-YY') < '2000-01-01'
            THEN TO_DATE(REPLACE(REPLACE(SPLIT_PART(TRIM(event_datetime), ' ', 1), '.', '-'), '/', '-'), 'DD-MM-YY') + INTERVAL '2000 years'
            ELSE TO_DATE(REPLACE(REPLACE(SPLIT_PART(TRIM(event_datetime), ' ', 1), '.', '-'), '/', '-'), 'DD-MM-YY')
        END AS event_date
    FROM cohort_events_raw
    WHERE event_type IS NOT NULL 
      AND event_type != 'test_event'
      AND event_datetime IS NOT NULL
),
-- 3. Об'єднуємо (додаємо revenue)
joined_data AS (
    SELECT 
        u.user_id,
        u.promo_signup_flag,
        e.revenue, -- ДОДАНО ПОЛЕ
        DATE_TRUNC('month', u.signup_date)::date AS cohort_month,
        (EXTRACT(YEAR FROM e.event_date) - EXTRACT(YEAR FROM u.signup_date)) * 12 +
        (EXTRACT(MONTH FROM e.event_date) - EXTRACT(MONTH FROM u.signup_date)) AS month_offset
    FROM users_cleaned u
    JOIN events_cleaned e ON u.user_id = e.user_id
    WHERE e.event_date >= u.signup_date
)
-- 4. Фінальна агрегація (рахуємо і людей, і гроші)
SELECT 
    promo_signup_flag,
    cohort_month,
    month_offset,
    COUNT(DISTINCT user_id) AS users_total,
    SUM(revenue) AS total_revenue -- ТЕПЕР У НАС БУДУТЬ ГРОШІ!
FROM joined_data
WHERE cohort_month BETWEEN '2025-01-01' AND '2025-06-01'
GROUP BY 1, 2, 3
ORDER BY 1, 2, 3;

