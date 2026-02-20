---
name: Home
assetId: 231ca4a4-94d9-4e4c-8356-e4765f2d0a3a
type: page
---

# Analytics Hub

Welcome to the analytics dashboard. Select a dashboard below to explore.

```sql data_freshness
SELECT max(`o.created_at`) as latest_order_date
FROM orders_enriched
WHERE `o.status` = 'SUCCESS'
```

_Last order: {% value data="data_freshness" value="max(latest_order_date)" fmt="longdate" /%} at {% value data="data_freshness" value="max(latest_order_date)" fmt="H:mm" /%} — used as a reference for data freshness._

```sql headline_kpis
SELECT
  (SELECT count(distinct user_pseudo_id) FROM events_clean WHERE event_datetime >= '2026-01-01' AND event_name = 'first_open') as total_app_opens,
  (SELECT count(distinct id) FROM users_enriched WHERE user_segment = 'Investor') as total_investors,
  (SELECT sum(shares) FROM orders_enriched WHERE `o.status` = 'SUCCESS') as total_shares_sold,
  (SELECT count(distinct id) FROM properties_enriched WHERE shares_sold > 0) as properties_with_sales
```

{% big_value
    data="headline_kpis"
    value="total_app_opens"
    title="App Opens"
    fmt="num0"
/%}

{% big_value
    data="headline_kpis"
    value="total_investors"
    title="Investors"
    fmt="num0"
/%}

{% big_value
    data="headline_kpis"
    value="total_shares_sold"
    title="Shares Sold"
    fmt="num0"
/%}

{% big_value
    data="headline_kpis"
    value="properties_with_sales"
    title="Properties"
    fmt="num0"
/%}

---

## Dashboards

### 📱 Acquisition Funnel
_App downloads, registration, verification, and first investment. Guest mode analysis, verification methods, and time-to-action metrics._

{% link_button url="acquisition-funnel" title="Open Dashboard →" /%}

---

### 👥 Cohort & Channel Analysis
_Registration cohort performance and acquisition channel effectiveness. Conversion rates and total shares by cohort month._

{% link_button url="cohort-channel-analysis" title="Open Dashboard →" /%}

---

### 🛒 Investment Action Funnel
_Property browsing through checkout to purchase. Checkout conversion, property views before checkout, shares distribution._

{% link_button url="investment-action-funnel" title="Open Dashboard →" /%}

---

### 🔄 Retention, Resale & Property
_Repeat purchase behavior, property sell-out rankings, resale market analysis, shares by type, and demographics._

{% link_button url="retention-resale-property" title="Open Dashboard →" /%}