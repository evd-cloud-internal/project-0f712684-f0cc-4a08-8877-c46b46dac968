---
name: investment-action-funnel
assetId: 7570c337-f439-415a-89c7-72895685f0e2
type: page
---

# Investment Action Funnel

Investment journey analysis — from property browsing through checkout to purchase. Includes shares distribution and investor segmentation.

_Events data from 2026 onward. Orders data is all-time._

---

## 1. Investment Action Funnel

```sql investment_funnel
SELECT
  CASE event_name
    WHEN 'property_view' THEN '1. Property View'
    WHEN 'click_invest_now' THEN '2. Click Invest Now'
    WHEN 'begin_checkout' THEN '3. Begin Checkout'
    WHEN 'purchase' THEN '4. Purchase'
  END as step,
  CASE event_name
    WHEN 'property_view' THEN 1
    WHEN 'click_invest_now' THEN 2
    WHEN 'begin_checkout' THEN 3
    WHEN 'purchase' THEN 4
  END as step_order,
  count(distinct user_pseudo_id) as users
FROM events_clean
WHERE event_datetime >= '2026-01-01'
  AND event_name IN ('property_view', 'click_invest_now', 'begin_checkout', 'purchase')
GROUP BY event_name
ORDER BY step_order
```

{% funnel_chart
    data="investment_funnel"
    category="step"
    value="users"
    value_fmt="num0"
    align="left"
    label_position="outside"
    show_percent=true
    title="Investment Action Funnel"
    info="Distinct users (user_pseudo_id) per event from events_clean (2026+). Note: click_invest_now count exceeds property_view — some users click invest without a tracked property view event."
/%}

---

## 2. Checkout Funnel Conversion

```sql checkout_conversion
SELECT
  count(distinct CASE WHEN event_name = 'begin_checkout' THEN user_pseudo_id END) as begin_checkout_users,
  count(distinct CASE WHEN event_name = 'purchase' THEN user_pseudo_id END) as purchase_users,
  count(distinct CASE WHEN event_name = 'purchase' THEN user_pseudo_id END)
    / count(distinct CASE WHEN event_name = 'begin_checkout' THEN user_pseudo_id END) as checkout_conversion_rate
FROM events_clean
WHERE event_datetime >= '2026-01-01'
  AND event_name IN ('begin_checkout', 'purchase')
```

{% big_value
    data="checkout_conversion"
    value="begin_checkout_users"
    title="Begin Checkout"
    fmt="num0"
    info="Distinct users who triggered begin_checkout event (2026+)."
/%}

{% big_value
    data="checkout_conversion"
    value="purchase_users"
    title="Completed Purchase"
    fmt="num0"
    info="Distinct users who triggered purchase event (2026+)."
/%}

{% big_value
    data="checkout_conversion"
    value="checkout_conversion_rate"
    title="Checkout Conversion Rate"
    fmt="pct1"
    info="purchase_users / begin_checkout_users. Measures what % of users who start checkout complete a purchase."
/%}

---

## 3. Property Views Before First Checkout

```sql views_before_checkout
WITH user_first_checkout AS (
  SELECT user_pseudo_id, min(event_datetime) as first_checkout
  FROM events_clean
  WHERE event_datetime >= '2026-01-01' AND event_name = 'begin_checkout'
  GROUP BY user_pseudo_id
),
user_views_before AS (
  SELECT
    ufc.user_pseudo_id,
    count(e.event_datetime) as views_before_checkout
  FROM user_first_checkout ufc
  LEFT JOIN events_clean e
    ON ufc.user_pseudo_id = e.user_pseudo_id
    AND e.event_name = 'property_view'
    AND e.event_datetime < ufc.first_checkout
    AND e.event_datetime >= '2026-01-01'
  GROUP BY ufc.user_pseudo_id
)
SELECT
  CASE
    WHEN views_before_checkout = 0 THEN '0 views'
    WHEN views_before_checkout = 1 THEN '1 view'
    WHEN views_before_checkout = 2 THEN '2 views'
    WHEN views_before_checkout BETWEEN 3 AND 4 THEN '3-4 views'
    ELSE '5+ views'
  END as view_bucket,
  CASE
    WHEN views_before_checkout = 0 THEN 0
    WHEN views_before_checkout = 1 THEN 1
    WHEN views_before_checkout = 2 THEN 2
    WHEN views_before_checkout BETWEEN 3 AND 4 THEN 3
    ELSE 5
  END as bucket_order,
  count() as users
FROM user_views_before
GROUP BY view_bucket, bucket_order
ORDER BY bucket_order
```

{% bar_chart
    data="views_before_checkout"
    x="view_bucket"
    y="users"
    y_fmt="num0"
    title="Property Views Before First Checkout"
    info="Number of property_view events per user before their first begin_checkout event. Source: events_clean (2026+). 30% of users who begin checkout had zero tracked property views beforehand."
    data_labels={
        position="above"
        fmt="num0"
    }
/%}

---

## 4. Distribution of Shares Per Order

```sql shares_per_order
SELECT
  CASE
    WHEN shares = 1 THEN '1 share'
    WHEN shares = 2 THEN '2 shares'
    WHEN shares BETWEEN 3 AND 4 THEN '3-4 shares'
    WHEN shares BETWEEN 5 AND 9 THEN '5-9 shares'
    WHEN shares BETWEEN 10 AND 19 THEN '10-19 shares'
    ELSE '20+ shares'
  END as shares_bucket,
  CASE
    WHEN shares = 1 THEN 1
    WHEN shares = 2 THEN 2
    WHEN shares BETWEEN 3 AND 4 THEN 3
    WHEN shares BETWEEN 5 AND 9 THEN 5
    WHEN shares BETWEEN 10 AND 19 THEN 10
    ELSE 20
  END as bucket_order,
  count() as orders
FROM orders_enriched
WHERE `o.status` = 'SUCCESS'
GROUP BY shares_bucket, bucket_order
ORDER BY bucket_order
```

{% bar_chart
    data="shares_per_order"
    x="shares_bucket"
    y="orders"
    y_fmt="num0"
    title="Distribution of Shares Per Order"
    info="Number of successful orders by shares purchased per order. Source: orders_enriched (status = SUCCESS, all time). ~74% of orders are for just 1 share."
    data_labels={
        position="above"
        fmt="num0"
    }
/%}

---

## 5. Average Shares Per Investor by Age Group & Gender

```sql avg_shares_demographics
SELECT
  u.age_group,
  u.gender,
  count(distinct u.id) as investors,
  sum(o.shares) as total_shares,
  round(sum(o.shares) / count(distinct u.id), 1) as avg_shares_per_investor
FROM users_enriched u
JOIN orders_enriched o ON toString(u.id) = toString(o.user_id)
WHERE o.`o.status` = 'SUCCESS'
  AND u.age_group IS NOT NULL AND u.age_group != ''
  AND u.gender IS NOT NULL AND u.gender != ''
GROUP BY u.age_group, u.gender
ORDER BY u.age_group, u.gender
```

{% bar_chart
    data="avg_shares_demographics"
    x="age_group"
    y="avg_shares_per_investor"
    series="gender"
    y_fmt="#,##0.0"
    title="Average Shares Per Investor by Age Group & Gender"
    info="Total shares from successful orders divided by distinct investors, grouped by age_group and gender. Source: users_enriched + orders_enriched (status = SUCCESS). Only users with non-empty age_group and gender."
    data_labels={
        position="above"
        fmt="#,##0.0"
    }
/%}