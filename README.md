# Chicago Taxi Trips — SQL Business Intelligence Analysis

Menganalisis 50+ juta transaksi taksi Chicago menggunakan Google BigQuery untuk mengungkap tren bisnis, dampak COVID-19, dan peluang optimasi operasional.

![BigQuery](https://img.shields.io/badge/Google_BigQuery-4285F4?logo=googlebigquery&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-Advanced-orange)
![Status](https://img.shields.io/badge/Status-Complete-2ea44f)

---

## Hasil singkat

| Metrik | Nilai |
|---|---|
| Dataset | Chicago Taxi Trips (BigQuery Public Data) |
| Periode analisis | 2018–2022 |
| Total queries | 6 analytical queries |
| Teknik SQL | CTE, Window Functions, LAG, EXTRACT, NULLIF |
| Tools | Google BigQuery |

---

## Temuan utama

**1. COVID-19 menghancurkan industri taksi — turun 96% dalam sebulan**
April 2020: trips anjlok dari 1.457.117 (April 2019) menjadi hanya 55.529 — penurunan **96.19% YoY**. Ini adalah dampak lockdown yang terdokumentasi langsung dari data transaksional nyata.

**2. Recovery lambat — 2 tahun belum balik ke level pre-COVID**
Meski trips mulai naik di 2021 (YoY +314% di April 2021 vs April 2020), angka absolut masih jauh di bawah 2018–2019. Desember 2022 hanya 253.720 trips vs 1.441.594 di Desember 2018 — masih -82% dari level normal.

**3. Flash Cab mendominasi market share tapi bukan yang paling efisien**
Flash Cab memimpin revenue ($165M) dan total trips (8.3M), tapi average revenue per trip hanya $19.81. Chicago Independents dan Taxicab Insurance Agency menghasilkan $23+ per trip dengan trip distance lebih jauh — lebih efisien secara per-trip.

**4. Credit Card menghasilkan tips 200x lebih besar dari Cash**
Credit Card: avg tips $4.01 (tip rate 18.43%) vs Cash: avg tips $0.01 (tip rate 0.09%). Pengguna kartu kredit secara konsisten memberi tip jauh lebih besar.

**5. Peak hour adalah Senin–Jumat jam 16.00–19.00**
Top 20 peak hours didominasi weekday sore hari — konsisten dengan pola commuter. Hari 3–6 (Selasa–Jumat) jam 17.00 adalah slot paling sibuk dengan 200.000–223.000 trips.

---

## Queries & insight

### Query 1 — Eksplorasi struktur data
```sql
SELECT trip_id, trip_start_timestamp, fare, tips, 
       trip_total, payment_type, company
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_start_timestamp IS NOT NULL
  AND fare IS NOT NULL AND trip_total > 0
LIMIT 10
```
**Insight:** Kolom lokasi (latitude/longitude, community area) banyak null — analisis difokuskan pada kolom finansial dan temporal yang lebih lengkap.

---

### Query 2 — Revenue trend bulanan
```sql
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
ORDER BY year, month
```
**Insight:** Revenue 2018 mencapai $25–34M per bulan. Pola musiman jelas — Juni adalah peak ($34.5M), Februari terendah ($24.8M).

---

### Query 3 — Performa per perusahaan taksi
```sql
SELECT
  company,
  COUNT(*) AS total_trips,
  ROUND(SUM(trip_total), 2) AS total_revenue,
  ROUND(AVG(trip_total), 2) AS avg_revenue_per_trip,
  ROUND(AVG(tips / NULLIF(trip_total, 0)) * 100, 2) AS tip_rate_pct,
  ROUND(AVG(trip_miles), 2) AS avg_miles,
  ROUND(AVG(trip_seconds/60), 2) AS avg_duration_minutes
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_total > 0 AND trip_miles > 0
  AND company IS NOT NULL
  AND EXTRACT(YEAR FROM trip_start_timestamp) BETWEEN 2018 AND 2022
GROUP BY company
HAVING total_trips > 10000
ORDER BY total_revenue DESC
LIMIT 15
```
**Insight:** Flash Cab (#1 revenue) vs Chicago Independents (#11) — revenue per trip $19.81 vs $23.07. Volume tinggi tidak selalu berarti efisiensi tinggi.

---

### Query 4 — Analisis payment method
```sql
SELECT
  payment_type,
  COUNT(*) AS total_trips,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS pct_of_total,
  ROUND(AVG(trip_total), 2) AS avg_trip_total,
  ROUND(AVG(tips), 2) AS avg_tips,
  ROUND(AVG(tips / NULLIF(trip_total, 0)) * 100, 2) AS tip_rate_pct
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_total > 0 AND payment_type IS NOT NULL
  AND EXTRACT(YEAR FROM trip_start_timestamp) BETWEEN 2018 AND 2022
GROUP BY payment_type
ORDER BY total_trips DESC
```
**Insight:** Cash (47.45%) vs Credit Card (44.55%) hampir setara volume, tapi tip rate sangat berbeda: 0.09% vs 18.43%. Driver lebih untung dari penumpang kartu kredit.

---

### Query 5 — Peak hours analysis
```sql
SELECT
  EXTRACT(DAYOFWEEK FROM trip_start_timestamp) AS day_of_week,
  EXTRACT(HOUR FROM trip_start_timestamp) AS hour,
  COUNT(*) AS total_trips,
  ROUND(AVG(trip_total), 2) AS avg_revenue,
  ROUND(AVG(tips), 2) AS avg_tips
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_start_timestamp IS NOT NULL AND trip_total > 0
  AND EXTRACT(YEAR FROM trip_start_timestamp) = 2019
GROUP BY day_of_week, hour
ORDER BY total_trips DESC
LIMIT 20
```
**Insight:** Weekday jam 16–19 adalah peak — 200K+ trips per slot. Operasional dan harga surge sebaiknya difokuskan di window ini.

---

### Query 6 — COVID-19 YoY Impact (Window Function)
```sql
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
SELECT *,
  LAG(total_trips, 12) OVER (ORDER BY year, month) AS trips_same_month_prev_year,
  ROUND(
    (total_trips - LAG(total_trips, 12) OVER (ORDER BY year, month)) * 100.0 /
    NULLIF(LAG(total_trips, 12) OVER (ORDER BY year, month), 0)
  , 2) AS yoy_growth_pct
FROM monthly
ORDER BY year, month
```
**Insight:** April 2020 YoY: **-96.19%**. Recovery April 2021: +314% (base effect). Desember 2022 masih -40.92% vs Desember 2021 — industri belum pulih sepenuhnya.

---

## Rekomendasi bisnis

| Prioritas | Temuan | Rekomendasi |
|---|---|---|
| 1 | Peak hour Senin–Jumat 16–19 | Terapkan surge pricing di slot ini untuk maksimalkan revenue |
| 2 | Credit card tip rate 18x lebih tinggi | Dorong adopsi cashless payment — untungkan driver dan platform |
| 3 | Flash Cab volume tinggi tapi avg trip rendah | Evaluasi apakah strategi volume > efficiency optimal jangka panjang |
| 4 | Recovery pasca-COVID masih -40% di akhir 2022 | Perlu strategi akuisisi pelanggan baru — pengguna ride-hailing tidak kembali ke taksi konvensional |

---

## Teknik SQL yang digunakan

| Teknik | Query | Kegunaan |
|---|---|---|
| `EXTRACT` | 2, 5, 6 | Ekstrak tahun, bulan, jam dari timestamp |
| `GROUP BY` + agregasi | 2, 3, 4, 5 | Summarize data per dimensi |
| `HAVING` | 3 | Filter setelah agregasi |
| `NULLIF` | 3, 4, 6 | Hindari division by zero |
| Window function `LAG` | 6 | Bandingkan nilai dengan periode sebelumnya |
| CTE (`WITH`) | 6 | Modularisasi query kompleks |
| `OVER()` | 4, 6 | Kalkulasi proporsi dan window aggregation |

---

## Struktur project

```
chicago-taxi-bi/
├── README.md
├── queries/
│   ├── 01_data_exploration.sql
│   ├── 02_monthly_revenue_trend.sql
│   ├── 03_company_performance.sql
│   ├── 04_payment_method_analysis.sql
│   ├── 05_peak_hours_analysis.sql
│   └── 06_covid_yoy_impact.sql
└── results/
    ├── monthly_revenue.csv
    ├── company_performance.csv
    ├── payment_analysis.csv
    ├── peak_hours.csv
    └── covid_impact.csv
```

---

## Cara replikasi

1. Buka [Google BigQuery Console](https://console.cloud.google.com/bigquery)
2. Buat project baru (gratis tier cukup)
3. Dataset sudah tersedia sebagai public data:
   `bigquery-public-data.chicago_taxi_trips.taxi_trips`
4. Copy-paste query dari folder `queries/` dan jalankan langsung

Tidak perlu upload data apapun — semua query menggunakan BigQuery public dataset.

---

## Tech stack

- **Query engine:** Google BigQuery
- **Dataset:** Chicago Taxi Trips (BigQuery Public Data) — 200M+ rows
- **Teknik:** CTE, Window Functions, Time Series Analysis, YoY Comparison

---

*Project ini dibuat sebagai bagian dari portofolio BI analyst. Feedback dan pertanyaan sangat disambut — silakan buka issue atau hubungi saya di LinkedIn.*
