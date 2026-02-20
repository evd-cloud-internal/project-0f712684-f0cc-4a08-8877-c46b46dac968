---
name: retention-resale-property
assetId: fec406de-9455-43ad-933a-2791fbded1b3
type: page
---

# Retention, Resale & Property Performance

Investor retention, repeat purchase behavior, property sell-out performance, resale market analysis, and user demographics.

---

## 1. Repeat Purchase Rate

```sql repeat_purchase
WITH user_orders AS (
  SELECT
    user_id,
    `o.created_at` as order_date,
    row_number() OVER (PARTITION BY user_id ORDER BY `o.created_at`) as order_num
  FROM orders_enriched
  WHERE `o.status` = 'SUCCESS'
),
first_and_second AS (
  SELECT
    f.user_id,
    f.order_date as first_purchase,
    min(s.order_date) as second_purchase
  FROM user_orders f
  LEFT JOIN user_orders s ON f.user_id = s.user_id AND s.order_num > 1 AND f.order_num = 1
  WHERE f.order_num = 1
  GROUP BY f.user_id, f.order_date
)
SELECT
  count() as total_investors,
  countIf(second_purchase IS NOT NULL) as repeat_investors,
  countIf(dateDiff('day', first_purchase, second_purchase) <= 30) / count() as pct_30d,
  countIf(dateDiff('day', first_purchase, second_purchase) <= 60) / count() as pct_60d,
  countIf(dateDiff('day', first_purchase, second_purchase) <= 90) / count() as pct_90d
FROM first_and_second
```

{% big_value
    data="repeat_purchase"
    value="pct_30d"
    title="Repurchase Within 30 Days"
    fmt="pct1"
    info="% of all investors who made a second successful purchase within 30 days of their first. Source: orders_enriched (status = SUCCESS)."
/%}

{% big_value
    data="repeat_purchase"
    value="pct_60d"
    title="Repurchase Within 60 Days"
    fmt="pct1"
    info="% of all investors who made a second successful purchase within 60 days of their first. Source: orders_enriched (status = SUCCESS)."
/%}

{% big_value
    data="repeat_purchase"
    value="pct_90d"
    title="Repurchase Within 90 Days"
    fmt="pct1"
    info="% of all investors who made a second successful purchase within 90 days of their first. Source: orders_enriched (status = SUCCESS)."
/%}

---

## 2. Time to Second Purchase by First Order Size

```sql time_to_second
WITH user_orders AS (
  SELECT
    user_id,
    `o.created_at` as order_date,
    shares,
    row_number() OVER (PARTITION BY user_id ORDER BY `o.created_at`) as order_num
  FROM orders_enriched
  WHERE `o.status` = 'SUCCESS'
),
first_second AS (
  SELECT
    f.user_id,
    f.order_date as first_purchase,
    f.shares as first_order_shares,
    min(s.order_date) as second_purchase
  FROM user_orders f
  JOIN user_orders s ON f.user_id = s.user_id AND s.order_num = 2
  WHERE f.order_num = 1
  GROUP BY f.user_id, f.order_date, f.shares
)
SELECT
  CASE
    WHEN first_order_shares = 1 THEN '1 share'
    WHEN first_order_shares BETWEEN 2 AND 3 THEN '2-3 shares'
    WHEN first_order_shares BETWEEN 4 AND 9 THEN '4-9 shares'
    ELSE '10+ shares'
  END as first_order_bucket,
  CASE
    WHEN first_order_shares = 1 THEN 1
    WHEN first_order_shares BETWEEN 2 AND 3 THEN 2
    WHEN first_order_shares BETWEEN 4 AND 9 THEN 4
    ELSE 10
  END as bucket_order,
  count() as investors,
  round(avg(dateDiff('day', first_purchase, second_purchase)), 1) as avg_days,
  round(median(dateDiff('day', first_purchase, second_purchase)), 1) as median_days
FROM first_second
GROUP BY first_order_bucket, bucket_order
ORDER BY bucket_order
```

{% bar_chart
    data="time_to_second"
    x="first_order_bucket"
    y="median_days"
    y_fmt="#,##0.0"
    title="Median Days to Second Purchase by First Order Size"
    info="Median days between first and second successful purchase, grouped by number of shares in the first order. Only includes investors who made at least 2 purchases. Source: orders_enriched (status = SUCCESS)."
    data_labels={
        position="above"
        fmt="#,##0.0"
    }
/%}

---

## 3. Rented vs Non-Rented Property — Repeat Purchase Rate

```sql rented_repeat
WITH first_orders AS (
  SELECT
    user_id,
    argMin(property_id, `o.created_at`) as first_property_id,
    min(`o.created_at`) as first_purchase_date
  FROM orders_enriched
  WHERE `o.status` = 'SUCCESS'
  GROUP BY user_id
),
user_order_counts AS (
  SELECT user_id, count() as order_count
  FROM orders_enriched
  WHERE `o.status` = 'SUCCESS'
  GROUP BY user_id
)
SELECT
  CASE
    WHEN p.is_rented = true THEN 'Rented Property'
    ELSE 'Non-Rented Property'
  END as first_investment_type,
  count(distinct fo.user_id) as total_investors,
  countIf(uc.order_count > 1) as repeat_investors,
  countIf(uc.order_count > 1) / count(distinct fo.user_id) as repeat_rate
FROM first_orders fo
JOIN properties_enriched p ON fo.first_property_id = p.id
JOIN user_order_counts uc ON fo.user_id = uc.user_id
GROUP BY first_investment_type
ORDER BY total_investors DESC
```

{% bar_chart
    data="rented_repeat"
    x="first_investment_type"
    y="repeat_rate"
    y_fmt="pct1"
    title="Repeat Purchase Rate: Rented vs Non-Rented First Investment"
    info="% of investors who made more than one successful purchase, grouped by whether their first purchased property was rented or not. Source: orders_enriched + properties_enriched."
    data_labels={
        position="above"
        fmt="pct1"
    }
/%}

---

## 4. Properties Ranked by Fastest Sell-Out

```sql property_sellout
SELECT
  property_name_en as property,
  property_code,
  total_shares,
  shares_sold,
  round(shares_sold / total_shares, 4) as pct_sold,
  selling_duration_in_hours,
  round(selling_duration_in_hours / 24, 1) as selling_duration_days
FROM properties_enriched
WHERE shares_sold > 0
ORDER BY selling_duration_in_hours ASC
```

{% table
    data="property_sellout"
    title="Properties by Sell-Out Speed"
    info="Properties with shares sold, ranked by selling_duration_in_hours (ascending). Source: properties_enriched."
%}
    {% dimension
        value="property"
        title="Property"
    /%}
    {% dimension
        value="property_code"
        title="Code"
    /%}
    {% measure
        value="sum(total_shares)"
        title="Total Shares"
        fmt="num0"
    /%}
    {% measure
        value="sum(shares_sold)"
        title="Shares Sold"
        fmt="num0"
    /%}
    {% measure
        value="sum(pct_sold)"
        title="% Sold"
        fmt="pct1"
    /%}
    {% measure
        value="sum(selling_duration_days)"
        title="Days to Sell"
        fmt="#,##0.0"
    /%}
{% /table %}

---

## 5. Total Shares Sold: Cash vs Installments vs Resale

```sql shares_by_type
SELECT
  CASE
    WHEN is_resale_order = 1 THEN 'Resale'
    WHEN is_installment = 1 THEN 'Installments'
    ELSE 'Cash'
  END as order_type,
  count() as orders,
  sum(shares) as total_shares
FROM orders_enriched
WHERE `o.status` = 'SUCCESS'
GROUP BY order_type
ORDER BY total_shares DESC
```

{% bar_chart
    data="shares_by_type"
    x="order_type"
    y="total_shares"
    y_fmt="num0"
    title="Total Shares Sold by Order Type"
    info="Total shares from successful orders, segmented by type: Cash (not installment, not resale), Installments (is_installment = 1), Resale (is_resale_order = 1). Source: orders_enriched (status = SUCCESS)."
    order="total_shares desc"
    data_labels={
        position="above"
        fmt="num0"
    }
/%}

---

## 6. Monthly Installment & Resale Volume

```sql monthly_by_type
SELECT
  toStartOfMonth(`o.created_at`) as month,
  CASE
    WHEN is_resale_order = 1 THEN 'Resale'
    WHEN is_installment = 1 THEN 'Installments'
    ELSE 'Cash'
  END as order_type,
  sum(shares) as total_shares
FROM orders_enriched
WHERE `o.status` = 'SUCCESS'
GROUP BY month, order_type
ORDER BY month, order_type
```

```sql monthly_installments
SELECT month, total_shares
FROM {{monthly_by_type}}
WHERE order_type = 'Installments'
ORDER BY month
```

```sql monthly_resale
SELECT month, total_shares
FROM {{monthly_by_type}}
WHERE order_type = 'Resale'
ORDER BY month
```

{% bar_chart
    data="monthly_installments"
    x="month"
    y="total_shares"
    y_fmt="num0"
    title="Monthly Shares Sold via Installments"
    info="Monthly sum of shares from successful installment orders (is_installment = 1). Source: orders_enriched (status = SUCCESS)."
    data_labels={
        position="above"
        fmt="num0"
    }
/%}

{% bar_chart
    data="monthly_resale"
    x="month"
    y="total_shares"
    y_fmt="num0"
    title="Monthly Shares Sold via Resale"
    info="Monthly sum of shares from successful resale orders (is_resale_order = 1). Source: orders_enriched (status = SUCCESS)."
    data_labels={
        position="above"
        fmt="num0"
    }
/%}

---

## 7. Monthly Shares Distribution by Type

{% bar_chart
    data="monthly_by_type"
    x="month"
    y="total_shares"
    series="order_type"
    y_fmt="num0"
    stacked=true
    title="Monthly Shares Sold: Cash vs Installments vs Resale"
    info="Monthly sum of shares from successful orders, stacked by order type. Source: orders_enriched (status = SUCCESS)."
/%}