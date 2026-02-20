---
name: acquisition-funnel
assetId: 72f07af7-c73a-4fff-8b30-f7c9330d846c
type: page
---

# Acquisition Funnel

This page analyzes the end-to-end user acquisition journey — from app download through registration, verification, and first investment.

_Events data from 2026 onward. Time-based metrics exclude users registered before May 2025 (data migration cutoff)._

---

## 1. End-to-End Acquisition Funnel

```sql funnel_data
SELECT
  CASE event_name
    WHEN 'first_open' THEN '1. App Download'
    WHEN 'sign_up' THEN '2. Registration'
    WHEN 'verify_identity' THEN '3. Account Verification'
    WHEN 'purchase' THEN '4. First Investment'
  END as step,
  CASE event_name
    WHEN 'first_open' THEN 1
    WHEN 'sign_up' THEN 2
    WHEN 'verify_identity' THEN 3
    WHEN 'purchase' THEN 4
  END as step_order,
  count(distinct user_pseudo_id) as users
FROM events_clean
WHERE event_datetime >= '2026-01-01'
  AND event_name IN ('first_open', 'sign_up', 'verify_identity', 'purchase')
GROUP BY event_name
ORDER BY step_order
```

{% funnel_chart
    data="funnel_data"
    category="step"
    value="users"
    value_fmt="num0"
    align="left"
    label_position="outside"
    show_percent=true
    title="Acquisition Funnel"
    subtitle="Unique users at each stage"
    info="Distinct users (user_pseudo_id) per event from events_clean (2026+). Steps: first_open (app download proxy), sign_up (registration), verify_identity (KYC completion), purchase (first investment)."
/%}

---

## 2. Guest Mode vs Registered Users

```sql guest_vs_registered
WITH first_openers AS (
  SELECT DISTINCT user_pseudo_id
  FROM events_clean
  WHERE event_datetime >= '2026-01-01'
    AND event_name = 'first_open'
),
signers AS (
  SELECT DISTINCT user_pseudo_id
  FROM events_clean
  WHERE event_datetime >= '2026-01-01'
    AND event_name = 'sign_up'
)
SELECT
  count(f.user_pseudo_id) as total_app_opens,
  countIf(s.user_pseudo_id IS NULL) as guest_only_users,
  countIf(s.user_pseudo_id IS NOT NULL) as registered_users,
  countIf(s.user_pseudo_id IS NULL) / count(f.user_pseudo_id) as guest_rate,
  countIf(s.user_pseudo_id IS NOT NULL) / count(f.user_pseudo_id) as registration_rate
FROM first_openers f
LEFT JOIN signers s ON f.user_pseudo_id = s.user_pseudo_id
```

{% big_value
    data="guest_vs_registered"
    value="total_app_opens"
    title="Total App Opens"
    fmt="num0"
    info="Distinct users who triggered the first_open event (2026+)."
/%}

{% big_value
    data="guest_vs_registered"
    value="guest_rate"
    title="Guest Mode Rate"
    fmt="pct1"
    info="% of first_open users who never triggered sign_up. Formula: guest_only_users / total_first_open_users."
/%}

{% big_value
    data="guest_vs_registered"
    value="registration_rate"
    title="Registration Rate"
    fmt="pct1"
    info="% of first_open users who also triggered sign_up. Formula: registered_users / total_first_open_users."
/%}

---

## 3. Verification Method: Passport vs National ID

```sql verification_method
SELECT
  CASE
    WHEN is_national_id_verified = true THEN 'National ID'
    WHEN is_passport_verified = true THEN 'Passport'
  END as verification_method,
  count() as users
FROM users_enriched
WHERE is_kyc_verified = true
  AND (is_national_id_verified = true OR is_passport_verified = true)
GROUP BY verification_method
ORDER BY users DESC
```

{% bar_chart
    data="verification_method"
    x="verification_method"
    y="users"
    y_fmt="num0"
    title="Verified Users by Document Type"
    info="KYC-verified users by document type from users_enriched. 33 legacy users with is_kyc_verified=true but no document type recorded are excluded."
    order="users desc"
    data_labels={
        position="above"
        fmt="num0"
    }
/%}

---

## 4. Average Time from Registration to Verification

_Excludes users registered before May 2025 due to data migration._

```sql reg_to_verify
SELECT
  round(avg(dateDiff('day', registered_at, kyc_verified_at)), 1) as avg_days,
  round(median(dateDiff('day', registered_at, kyc_verified_at)), 1) as median_days,
  round(min(dateDiff('day', registered_at, kyc_verified_at)), 1) as min_days,
  round(max(dateDiff('day', registered_at, kyc_verified_at)), 1) as max_days,
  count() as users_in_calc
FROM users_enriched
WHERE is_kyc_verified = true
  AND registered_at >= '2025-05-01'
  AND kyc_verified_at >= registered_at
```

{% big_value
    data="reg_to_verify"
    value="users_in_calc"
    title="Users in Calculation"
    fmt="num0"
    info="Number of verified users included (registered after May 2025 with kyc_verified_at >= registered_at)."
/%}

### In Days

{% row %}
{% big_value
    data="reg_to_verify"
    value="avg_days"
    title="Avg Days"
    fmt="num1"
    info="Average of dateDiff(day, registered_at, kyc_verified_at) for verified users registered after May 2025."
/%}

{% big_value
    data="reg_to_verify"
    value="median_days"
    title="Median Days"
    fmt="num1"
    info="Median of dateDiff(day, registered_at, kyc_verified_at). Less sensitive to outliers than the average."
/%}

{% big_value
    data="reg_to_verify"
    value="min_days"
    title="Min Days"
    fmt="num1"
    info="Fastest time from registration to KYC verification."
/%}

{% big_value
    data="reg_to_verify"
    value="max_days"
    title="Max Days"
    fmt="num1"
    info="Longest time from registration to KYC verification."
/%}
{% /row %}

### In Hours

```sql reg_to_verify_hours
SELECT
  round(avg(dateDiff('hour', registered_at, kyc_verified_at)), 1) as avg_hours,
  round(median(dateDiff('hour', registered_at, kyc_verified_at)), 1) as median_hours,
  round(min(dateDiff('hour', registered_at, kyc_verified_at)), 1) as min_hours,
  round(max(dateDiff('hour', registered_at, kyc_verified_at)), 1) as max_hours
FROM users_enriched
WHERE is_kyc_verified = true
  AND registered_at >= '2025-05-01'
  AND kyc_verified_at >= registered_at
```

{% row %}
{% big_value
    data="reg_to_verify_hours"
    value="avg_hours"
    title="Avg Hours"
    fmt="num1"
    info="Average of dateDiff(hour, registered_at, kyc_verified_at) for verified users registered after May 2025."
/%}

{% big_value
    data="reg_to_verify_hours"
    value="median_hours"
    title="Median Hours"
    fmt="num1"
    info="Median of dateDiff(hour, registered_at, kyc_verified_at). Less sensitive to outliers than the average."
/%}

{% big_value
    data="reg_to_verify_hours"
    value="min_hours"
    title="Min Hours"
    fmt="num1"
    info="Fastest time from registration to KYC verification in hours."
/%}

{% big_value
    data="reg_to_verify_hours"
    value="max_hours"
    title="Max Hours"
    fmt="num1"
    info="Longest time from registration to KYC verification in hours."
/%}
{% /row %}

---

## 5. Average Time from Registration to First Investment

_Excludes users registered before May 2025._

```sql reg_to_invest
WITH first_purchase AS (
  SELECT
    user_id,
    min(`o.created_at`) as first_purchase_date
  FROM orders_enriched
  WHERE `o.status` = 'SUCCESS'
  GROUP BY user_id
)
SELECT
  round(avg(dateDiff('day', u.registered_at, fp.first_purchase_date)), 1) as avg_days,
  round(median(dateDiff('day', u.registered_at, fp.first_purchase_date)), 1) as median_days,
  round(min(dateDiff('day', u.registered_at, fp.first_purchase_date)), 1) as min_days,
  round(max(dateDiff('day', u.registered_at, fp.first_purchase_date)), 1) as max_days,
  count() as investors_in_calc
FROM users_enriched u
JOIN first_purchase fp ON toString(u.id) = toString(fp.user_id)
WHERE u.registered_at >= '2025-05-01'
  AND fp.first_purchase_date >= u.registered_at
```

{% big_value
    data="reg_to_invest"
    value="investors_in_calc"
    title="Investors in Calculation"
    fmt="num0"
    info="Number of investors included (registered after May 2025 with first_purchase_date >= registered_at)."
/%}

### In Days

{% row %}
{% big_value
    data="reg_to_invest"
    value="avg_days"
    title="Avg Days"
    fmt="num1"
    info="Average of dateDiff(day, registered_at, first_purchase_date). First purchase = earliest successful order per user."
/%}

{% big_value
    data="reg_to_invest"
    value="median_days"
    title="Median Days"
    fmt="num1"
    info="Median of dateDiff(day, registered_at, first_purchase_date). Less sensitive to outliers than the average."
/%}

{% big_value
    data="reg_to_invest"
    value="min_days"
    title="Min Days"
    fmt="num1"
    info="Fastest time from registration to first successful purchase."
/%}

{% big_value
    data="reg_to_invest"
    value="max_days"
    title="Max Days"
    fmt="num1"
    info="Longest time from registration to first successful purchase."
/%}
{% /row %}

### In Hours

```sql reg_to_invest_hours
WITH first_purchase AS (
  SELECT
    user_id,
    min(`o.created_at`) as first_purchase_date
  FROM orders_enriched
  WHERE `o.status` = 'SUCCESS'
  GROUP BY user_id
)
SELECT
  round(avg(dateDiff('hour', u.registered_at, fp.first_purchase_date)), 1) as avg_hours,
  round(median(dateDiff('hour', u.registered_at, fp.first_purchase_date)), 1) as median_hours,
  round(min(dateDiff('hour', u.registered_at, fp.first_purchase_date)), 1) as min_hours,
  round(max(dateDiff('hour', u.registered_at, fp.first_purchase_date)), 1) as max_hours
FROM users_enriched u
JOIN first_purchase fp ON toString(u.id) = toString(fp.user_id)
WHERE u.registered_at >= '2025-05-01'
  AND fp.first_purchase_date >= u.registered_at
```

{% row %}
{% big_value
    data="reg_to_invest_hours"
    value="avg_hours"
    title="Avg Hours"
    fmt="num1"
    info="Average of dateDiff(hour, registered_at, first_purchase_date). First purchase = earliest successful order per user."
/%}

{% big_value
    data="reg_to_invest_hours"
    value="median_hours"
    title="Median Hours"
    fmt="num1"
    info="Median of dateDiff(hour, registered_at, first_purchase_date). Less sensitive to outliers than the average."
/%}

{% big_value
    data="reg_to_invest_hours"
    value="min_hours"
    title="Min Hours"
    fmt="num1"
    info="Fastest time from registration to first successful purchase in hours."
/%}

{% big_value
    data="reg_to_invest_hours"
    value="max_hours"
    title="Max Hours"
    fmt="num1"
    info="Longest time from registration to first successful purchase in hours."
/%}
{% /row %}