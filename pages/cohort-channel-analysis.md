---
name: cohort-channel-analysis
assetId: 6b2d779b-ed64-4fc8-8ba7-b5232417d6eb
type: page
---

# Cohort & Channel Analysis

Registration cohort performance and acquisition channel effectiveness.

_Cohort data uses users registered after May 2025 (data migration cutoff). Channel data uses events from 2026 onward._

---

## 1. Investment Conversion by Registration Cohort

```sql cohort_conversion
SELECT
  toStartOfMonth(u.registered_at) as cohort_month,
  count(distinct u.id) as total_registered,
  countIf(u.user_segment = 'Investor') as investors,
  countIf(u.user_segment = 'Investor') / count(distinct u.id) as conversion_rate
FROM users_enriched u
WHERE u.registered_at >= '2025-05-01'
GROUP BY cohort_month
ORDER BY cohort_month
```

{% bar_chart
    data="cohort_conversion"
    x="cohort_month"
    y="conversion_rate"
    y_fmt="pct1"
    title="Investment Conversion Rate by Registration Cohort"
    info="% of registered users who became investors (user_segment = Investor) per registration month. Source: users_enriched. Excludes users registered before May 2025 (data migration). Recent cohorts may show lower rates as users haven't had time to convert."
    data_labels={
        position="above"
        fmt="pct1"
    }
/%}

---

## 2. Total Shares Purchased by Registration Cohort

```sql cohort_shares
WITH user_shares AS (
  SELECT
    user_id,
    sum(shares) as total_shares
  FROM orders_enriched
  WHERE `o.status` = 'SUCCESS'
  GROUP BY user_id
)
SELECT
  toStartOfMonth(u.registered_at) as cohort_month,
  count(distinct u.id) as total_registered,
  countIf(us.user_id IS NOT NULL) as investors,
  coalesce(sum(us.total_shares), 0) as total_shares
FROM users_enriched u
LEFT JOIN user_shares us ON toString(u.id) = toString(us.user_id)
WHERE u.registered_at >= '2025-05-01'
GROUP BY cohort_month
ORDER BY cohort_month
```

{% bar_chart
    data="cohort_shares"
    x="cohort_month"
    y="total_shares"
    y_fmt="num0"
    title="Total Shares Purchased by Registration Cohort"
    info="Total shares from successful orders, grouped by the user's registration month. Source: users_enriched + orders_enriched (status = SUCCESS). Excludes users registered before May 2025 (data migration)."
    data_labels={
        position="above"
        fmt="num0"
    }
/%}

---

## 3. Acquisition Channel Performance

```sql channel_performance
WITH user_channels AS (
  SELECT
    user_pseudo_id,
    argMin(
      CASE traffic_source_name
        WHEN '(direct)' THEN 'Direct'
        WHEN 'fb4a' THEN 'Facebook'
        WHEN 'ig4a' THEN 'Instagram'
        WHEN 'App promotion-App- SAFE' THEN 'App promotion-App- SAFE'
        WHEN 'test campaign' THEN 'Test Campaign'
        ELSE 'Firebase Push Notifications'
      END,
      event_datetime
    ) as channel
  FROM events_clean
  WHERE event_datetime >= '2026-01-01'
    AND traffic_source_name NOT IN ('', '(not set)')
  GROUP BY user_pseudo_id
),
user_signups AS (
  SELECT DISTINCT user_pseudo_id
  FROM events_clean
  WHERE event_datetime >= '2026-01-01'
    AND event_name = 'sign_up'
),
user_purchases AS (
  SELECT DISTINCT user_pseudo_id
  FROM events_clean
  WHERE event_datetime >= '2026-01-01'
    AND event_name = 'purchase'
)
SELECT
  uc.channel,
  count(distinct uc.user_pseudo_id) as total_users,
  countIf(us.user_pseudo_id IS NOT NULL) as registered,
  countIf(up.user_pseudo_id IS NOT NULL) as investors,
  countIf(up.user_pseudo_id IS NOT NULL) / count(distinct uc.user_pseudo_id) as investment_conversion_rate
FROM user_channels uc
LEFT JOIN user_signups us ON uc.user_pseudo_id = us.user_pseudo_id
LEFT JOIN user_purchases up ON uc.user_pseudo_id = up.user_pseudo_id
GROUP BY uc.channel
ORDER BY total_users DESC
```

{% bar_chart
    data="channel_performance"
    x="channel"
    y="investment_conversion_rate"
    y_fmt="pct1"
    title="Investment Conversion Rate by Acquisition Channel"
    info="First traffic_source_name per user from events_clean (2026+). Channels: Direct, Facebook (fb4a), Instagram (ig4a), Firebase Push Notifications, App promotion-App-SAFE. Conversion = user triggered a purchase event."
    order="investment_conversion_rate desc"
    data_labels={
        position="above"
        fmt="pct1"
    }
/%}

{% table
    data="channel_performance"
    title="Channel Breakdown"
    info="Full breakdown of acquisition channels. First traffic_source_name per user from events_clean (2026+). Registered = triggered sign_up event. Investors = triggered purchase event."
%}
    {% dimension
        value="channel"
    /%}
    {% measure
        value="sum(total_users)"
        title="Total Users"
        fmt="num0"
    /%}
    {% measure
        value="sum(registered)"
        title="Registered"
        fmt="num0"
    /%}
    {% measure
        value="sum(investors)"
        title="Investors"
        fmt="num0"
    /%}
    {% measure
        value="sum(investment_conversion_rate)"
        title="Conversion Rate"
        fmt="pct1"
    /%}
{% /table %}