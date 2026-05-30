-- ============================================
-- 01_data_exploration.sql
-- Tujuan: Kenali struktur dan kualitas data
-- ============================================
SELECT
  trip_id,
  taxi_id,
  trip_start_timestamp,
  trip_end_timestamp,
  trip_seconds,
  trip_miles,
  fare,
  tips,
  tolls,
  extras,
  trip_total,
  payment_type,
  company
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_start_timestamp IS NOT NULL
  AND fare IS NOT NULL
  AND trip_total > 0
LIMIT 10;


-- ============================================
-- 02_monthly_revenue_trend.sql
-- Tujuan: Analisis tren revenue bulanan 2018-2022
-- ============================================
SELECT
  EXTRACT(YEAR FROM trip_start_timestamp) AS year,
  EXTRACT(MONTH FROM trip_start_timestamp) AS month,
  COUNT(*) AS total_trips,
  ROUND(SUM(trip_total), 2) AS total_revenue,
  ROUND(AVG(trip_total), 2) AS avg_revenue_per_trip,
  ROUND(AVG(tips), 2) AS avg_tips
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_start_timestamp IS NOT NULL
  AND trip_total > 0
  AND EXTRACT(YEAR FROM trip_start_timestamp) BETWEEN 2018 AND 2022
GROUP BY year, month
ORDER BY year, month;


-- ============================================
-- 03_company_performance.sql
-- Tujuan: Bandingkan performa antar perusahaan taksi
-- ============================================
SELECT
  company,
  COUNT(*) AS total_trips,
  ROUND(SUM(trip_total), 2) AS total_revenue,
  ROUND(AVG(trip_total), 2) AS avg_revenue_per_trip,
  ROUND(AVG(tips / NULLIF(trip_total, 0)) * 100, 2) AS tip_rate_pct,
  ROUND(AVG(trip_miles), 2) AS avg_miles,
  ROUND(AVG(trip_seconds / 60), 2) AS avg_duration_minutes
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_total > 0
  AND trip_miles > 0
  AND company IS NOT NULL
  AND EXTRACT(YEAR FROM trip_start_timestamp) BETWEEN 2018 AND 2022
GROUP BY company
HAVING total_trips > 10000
ORDER BY total_revenue DESC
LIMIT 15;


-- ============================================
-- 04_payment_method_analysis.sql
-- Tujuan: Analisis perilaku tipping per metode pembayaran
-- ============================================
SELECT
  payment_type,
  COUNT(*) AS total_trips,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS pct_of_total,
  ROUND(AVG(trip_total), 2) AS avg_trip_total,
  ROUND(AVG(tips), 2) AS avg_tips,
  ROUND(AVG(tips / NULLIF(trip_total, 0)) * 100, 2) AS tip_rate_pct
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_total > 0
  AND payment_type IS NOT NULL
  AND EXTRACT(YEAR FROM trip_start_timestamp) BETWEEN 2018 AND 2022
GROUP BY payment_type
ORDER BY total_trips DESC;


-- ============================================
-- 05_peak_hours_analysis.sql
-- Tujuan: Identifikasi jam dan hari paling sibuk
-- Catatan: day_of_week 1=Minggu, 7=Sabtu
-- ============================================
SELECT
  EXTRACT(DAYOFWEEK FROM trip_start_timestamp) AS day_of_week,
  EXTRACT(HOUR FROM trip_start_timestamp) AS hour,
  COUNT(*) AS total_trips,
  ROUND(AVG(trip_total), 2) AS avg_revenue,
  ROUND(AVG(tips), 2) AS avg_tips
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_start_timestamp IS NOT NULL
  AND trip_total > 0
  AND EXTRACT(YEAR FROM trip_start_timestamp) = 2019
GROUP BY day_of_week, hour
ORDER BY total_trips DESC
LIMIT 20;


-- ============================================
-- 06_covid_yoy_impact.sql
-- Tujuan: Ukur dampak COVID-19 menggunakan YoY comparison
-- Teknik: CTE + Window Function LAG
-- ============================================
WITH monthly AS (
  SELECT
    EXTRACT(YEAR FROM trip_start_timestamp) AS year,
    EXTRACT(MONTH FROM trip_start_timestamp) AS month,
    COUNT(*) AS total_trips,
    ROUND(SUM(trip_total), 2) AS total_revenue,
    ROUND(AVG(trip_total), 2) AS avg_revenue_per_trip,
    ROUND(AVG(tips), 2) AS avg_tips
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_total > 0
    AND trip_start_timestamp IS NOT NULL
    AND EXTRACT(YEAR FROM trip_start_timestamp) BETWEEN 2018 AND 2022
  GROUP BY year, month
)

SELECT
  year,
  month,
  total_trips,
  total_revenue,
  avg_revenue_per_trip,
  avg_tips,
  LAG(total_trips, 12) OVER (ORDER BY year, month) AS trips_same_month_prev_year,
  ROUND(
    (total_trips - LAG(total_trips, 12) OVER (ORDER BY year, month)) * 100.0 /
    NULLIF(LAG(total_trips, 12) OVER (ORDER BY year, month), 0)
  , 2) AS yoy_growth_pct
FROM monthly
ORDER BY year, month;
