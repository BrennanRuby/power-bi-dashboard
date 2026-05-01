# Data Dictionary — Business Performance Dashboard

## Schema Overview

Star schema with 4 dimension tables and 4 fact tables. All tables join on `date_key`. Fact tables with additional dimensions join on their respective `_key` columns.

```
                    ┌──────────────────┐
                    │    dim_date       │
                    │  (date_key PK)   │
                    └────────┬─────────┘
                             │
          ┌──────────────────┼──────────────────┬──────────────────┐
          │                  │                  │                  │
┌─────────▼────────┐ ┌──────▼──────────┐ ┌─────▼──────────┐ ┌────▼───────────┐
│ fact_monthly     │ │ fact_product    │ │ fact_channel   │ │ fact_regional  │
│ _financials      │ │ _sales          │ │ _performance   │ │ _sales         │
└──────────────────┘ └──────┬──────────┘ └─────┬──────────┘ └────┬───────────┘
                            │                  │                  │
                   ┌────────▼─────────┐ ┌──────▼──────────┐ ┌────▼───────────┐
                   │ dim_product      │ │ dim_channel     │ │ dim_region     │
                   │ _category        │ │                 │ │                │
                   └──────────────────┘ └─────────────────┘ └────────────────┘
```

---

## Dimension Tables

### dim_date.csv
Date dimension covering Jan 2023 — Dec 2026 (48 rows). Monthly grain.

| Column | Type | Description |
|--------|------|-------------|
| date_key | string | Primary key. Format: YYYY-MM-DD (first of month) |
| date | date | Full date |
| year | int | Calendar year |
| quarter | int | Calendar quarter (1–4) |
| quarter_label | string | "Q1", "Q2", "Q3", "Q4" |
| month_num | int | Month number (1–12) |
| month_name | string | Full month name |
| month_short | string | 3-letter abbreviation |
| month_label | string | "Jan 2023" format |
| year_month | string | "2023-01" sortable format |
| fiscal_year | int | Oct–Sep fiscal year (Oct starts new FY) |
| fiscal_quarter | string | "FQ1"–"FQ4" (FQ1 = Oct–Dec) |
| is_holiday_season | int | 1 if Nov or Dec, 0 otherwise |

### dim_product_category.csv
Product category dimension (5 rows).

| Column | Type | Description |
|--------|------|-------------|
| category_key | string | Primary key (ELEC, HOME, APRL, SPRT, HLTH) |
| category_name | string | Display name |
| department | string | Department grouping |
| target_margin | decimal | Target gross margin (e.g., 0.25 = 25%) |

### dim_channel.csv
Marketing channel dimension (6 rows).

| Column | Type | Description |
|--------|------|-------------|
| channel_key | string | Primary key (ORGANIC, PAID_SRC, SOCIAL, EMAIL, DIRECT, REFERRAL) |
| channel_name | string | Display name |
| channel_type | string | Owned / Paid / Earned |
| is_paid | int | 1 if channel has ad spend, 0 otherwise |

### dim_region.csv
Geographic region dimension (5 rows).

| Column | Type | Description |
|--------|------|-------------|
| region_key | string | Primary key (NE, SE, MW, WE, SW) |
| region_name | string | Display name |
| timezone | string | Primary timezone |

---

## Fact Tables

### fact_monthly_financials.csv
Company-level P&L and customer metrics. Monthly grain, 36 rows (Jan 2023 — Dec 2025).

| Column | Type | Description |
|--------|------|-------------|
| date_key | string | FK → dim_date |
| revenue | decimal | Total revenue for the month |
| cogs | decimal | Cost of goods sold |
| operating_expenses | decimal | OpEx (includes marketing) |
| marketing_spend | decimal | Marketing/advertising spend (subset of OpEx) |
| orders | int | Total orders placed |
| new_customers | int | First-time buyers |
| returning_customers | int | Repeat buyers |
| website_sessions | int | Total site sessions |

**Suggested DAX measures:**
- Gross Profit = revenue - cogs
- Net Profit = revenue - cogs - operating_expenses
- Gross Margin % = Gross Profit / revenue
- Net Margin % = Net Profit / revenue
- AOV = revenue / orders
- CAC = marketing_spend / new_customers
- Conversion Rate = orders / website_sessions
- MoM Growth = (current - prior) / prior
- YoY Growth = compare to same month prior year
- Rolling averages (3, 6, 12 month)

### fact_product_sales.csv
Revenue and cost by product category. Monthly grain, 180 rows (36 months x 5 categories).

| Column | Type | Description |
|--------|------|-------------|
| date_key | string | FK → dim_date |
| category_key | string | FK → dim_product_category |
| revenue | decimal | Category revenue |
| orders | int | Category orders |
| cogs | decimal | Category cost of goods sold |

**Suggested DAX measures:**
- Gross Profit = revenue - cogs
- Gross Margin % = Gross Profit / revenue
- Revenue Share % = category revenue / total revenue
- Margin vs Target = Gross Margin % - dim_product_category[target_margin]

### fact_channel_performance.csv
Marketing channel traffic and attribution. Monthly grain, 216 rows (36 months x 6 channels).

| Column | Type | Description |
|--------|------|-------------|
| date_key | string | FK → dim_date |
| channel_key | string | FK → dim_channel |
| sessions | int | Site sessions from this channel |
| orders | int | Orders attributed to this channel |
| revenue | decimal | Revenue attributed to this channel |
| spend | decimal | Ad spend (0 for non-paid channels) |

**Suggested DAX measures:**
- Conversion Rate = orders / sessions
- ROAS = revenue / spend (paid channels only)
- CPA = spend / orders (paid channels only)
- Revenue Share % = channel revenue / total revenue

### fact_regional_sales.csv
Sales by geographic region. Monthly grain, 180 rows (36 months x 5 regions).

| Column | Type | Description |
|--------|------|-------------|
| date_key | string | FK → dim_date |
| region_key | string | FK → dim_region |
| revenue | decimal | Regional revenue |
| orders | int | Regional orders |

**Suggested DAX measures:**
- AOV = revenue / orders
- Revenue Share % = regional revenue / total revenue

---

## Relationships (Power BI Model)

```
dim_date[date_key]              1 ──── * fact_monthly_financials[date_key]
dim_date[date_key]              1 ──── * fact_product_sales[date_key]
dim_date[date_key]              1 ──── * fact_channel_performance[date_key]
dim_date[date_key]              1 ──── * fact_regional_sales[date_key]
dim_product_category[category_key] 1 ──── * fact_product_sales[category_key]
dim_channel[channel_key]        1 ──── * fact_channel_performance[channel_key]
dim_region[region_key]          1 ──── * fact_regional_sales[region_key]
```

All relationships: single direction, one-to-many, active.

---

## Data Notes

- **Grain:** Monthly (first-of-month dates)
- **Historical period:** Jan 2023 — Dec 2025 (36 months)
- **dim_date extends to:** Dec 2026 (for forecast visuals)
- **Currency:** USD
- **Data is synthetic** — modeled after a mid-size e-commerce/retail business with ~$180K–$330K monthly revenue, ~12% annual growth, and retail seasonality (Q4 spike)
