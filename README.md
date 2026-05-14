# [ FINAL PROJECT ] — RETAIL PERFORMANCE 2025

**Author:** Vo Phuong Anh
**Role:** Data Analyst
**Target Audience:** BOD & Retail Department Head
**Tech Stacks:** SQL Server · Power BI

---

## I — Project Overview

This report delivers a full-year 2025 performance review of the retail department across all 20 stores in 3 regions: North, Central, and South.

The analysis is built on **1,000 sales transactions**, **800 inventory records**, **100 active SKUs** across 5 product categories, and **20 store profiles** ranging from Mini format to Hypermarket. Source data was loaded from a single Excel file into SQL Server, then modeled in Power BI using a star schema (see Section IV).

The report answers 24 structured business questions grouped into 6 sections that progressively narrow from company-level down to individual store and manager level. Each section is written for a specific audience so that the Board, the Sales Head, and Store Managers only engage with what is relevant to them.

**What this report covers:**
- Full-year revenue, transaction volume, and margin performance
- Revenue breakdown by region, store type, store size, and product category
- Individual store ranking and performance gap vs regional averages
- Promotion effectiveness — measured by revenue and units uplift per category and per store
- Inventory health — stockout risk rate, system vs physical stock accuracy, and data integrity flags
- Manager accountability scorecard combining revenue, margin, and inventory discipline into a single composite rank

**What this report does not cover:**
- Customer-level or loyalty data — not captured in the dataset
- Cost of running promotions — discount depth is not recorded; only a binary promotion_flag is available. Uplift calculations in Section 4 measure revenue change, not net profit after discount cost
- Forward forecast for Jan 2026 — this requires a separate time-series model and is outside the current scope

---

## II — Business Objectives

1. **Assess overall 2025 sales performance.** Set the full-year revenue ($266K), transaction count (1,000), and average basket ($266) as the baseline against which all store and category performance is measured.

2. **Identify where revenue is coming from.** Break down performance by region, store type, store size, and product category to understand which parts of the business are generating value and which are underperforming relative to their size.

3. **Rank individual store performance.** Find the top 5 and bottom 5 stores by revenue. Measure each store against its own regional average — not just the network average — because a store in South should not be benchmarked the same way as a store in Central.

4. **Evaluate whether the promotion program is delivering a return.** Compare revenue and units in promoted vs non-promoted transactions both overall and per category. Identify which categories respond well enough to justify promotion spend and which do not.

5. **Audit inventory health across the network.** Quantify how many transactions occurred when physical stock was already below the reorder point, identify the most exposed categories and stores, and flag stores where system stock data cannot be trusted for replenishment decisions.

6. **Build a manager accountability layer.** Combine revenue rank, margin rank, and inventory discipline into a composite scorecard. Use it to recognize strong operators and give underperformers specific, data-backed targets for 2026.

---

## III — Target Audience

| Section | Content | Primary Audience | Notes |
|---|---|---|---|
| 1 — Business at a Glance | Total revenue, transactions, monthly trend, channel split | Everyone | No store names. Board-safe. |
| 2 — Where Is the Money Coming From? | Revenue by region, store type, size, category | Everyone | BOD can exit after this section. |
| 3 — Individual Store Performance | Top/bottom stores, regional ranking, dual underperformers | Sales Head + Store Managers | Store names and revenue gaps appear here. |
| 4 — Are Promotions Working? | Promo uplift by category and by store | Sales Head | Requires familiarity with the category portfolio. |
| 5 — Inventory Health | Stockout risk, stock accuracy, negative system stock | Operations + Store Managers | Most operationally specific section. |
| 6 — Manager Accountability | Per-manager scorecard, composite rank | Store Managers | Individual names and composite scores. |

---

## IV — Data & Model

### Data Source

Raw file: `retail_sales_inventory_raw_data.xlsx`

All transaction and inventory data for FY2025 is held in a single flat table in this file. The ETL script (`etl_retail_to_sqlserver.py`) splits it into 4 structured tables and loads them into SQL Server on server `MACLIFEB100`, database `retail`, using Windows Authentication.

### Data Model — Star Schema

```
DateTable
    │
    └──► fact_orders ◄─── dim_store
               │
               └──► dim_product

fact_inventory ◄─── dim_store
               │
               └──► dim_product
```

| Table | Type | Row Count | Key Columns |
|---|---|---|---|
| `fact_orders` | Fact | 1,000 | sale_id, date, store_id, product_id, units_sold, revenue, promotion_flag, channel, cost, price, total_cost, total_price |
| `fact_inventory` | Fact | 800 | inventory_id, store_id, product_id, stock_on_hand, stock_system, reorder_point |
| `dim_store` | Dimension | 20 | store_id, region, store_type, size, manager |
| `dim_product` | Dimension | 100 | product_id, category, brand, cost, price |
| `DateTable` | Date | 365 | Date, Year, Month, MonthNum, Quarter, MonthYear |

### Key Relationships

- `fact_orders[store_id]` → `dim_store[store_id]` (Many-to-One)
- `fact_orders[product_id]` → `dim_product[product_id]` (Many-to-One)
- `fact_orders[date]` → `DateTable[Date]` (Many-to-One)
- `fact_inventory[store_id]` → `dim_store[store_id]` (Many-to-One)
- `fact_inventory[product_id]` → `dim_product[product_id]` (Many-to-One)

### Dataset Coverage

| Dimension | Detail |
|---|---|
| Date range | 2 Jan 2025 – 31 Dec 2025 |
| Regions | 3 — North (9 stores), South (6 stores), Central (5 stores) |
| Store formats | Mini (10), Supermarket (5), Hypermarket (5) |
| Store sizes | Medium (8), Small (7), Large (5) |
| Products | 100 SKUs across 5 categories, 18 brands |
| Categories | Dairy, Personal Care, Beverage, Household, Snack |

### Data Quality Issues Found in EDA

**Issue 1 — Negative system stock (6 records):**
Six SKU-store combinations in `fact_inventory` have a negative `stock_system` value. An inventory system cannot record negative stock — this is a hard data integrity error. Any replenishment rule that reads `stock_system` for these records will produce incorrect reorder signals. These must be corrected before any automated reorder logic is built.

Affected stores: STR002, STR009, STR014, STR017 (twice), STR001.

**Issue 2 — Sales below reorder point (196 transactions):**
19.6% of transactions in `fact_orders` occurred when `stock_on_hand < reorder_point`. These are real completed sales — not a data error — but they indicate the business has been drawing down safety stock at scale throughout 2025.

---

## V — Initial EDA

### 5.1 Top-Line KPIs

| Metric | Value |
|---|---|
| Total Revenue | $266,117 |
| Total Transactions | 1,000 |
| Avg Revenue per Transaction | $266.12 |
| Total Units Sold | 10,402 |
| Avg Units per Transaction | 10.40 |
| Active Stores | 20 |
| Date Coverage | Jan 2 – Dec 31, 2025 |

The business ran an average of **83 transactions per month** at a **$266 basket**. Both numbers serve as the benchmark throughout the rest of this analysis.

---

### 5.2 Channel Split

| Channel | Revenue | Share |
|---|---|---|
| Online | $136,005 | 51.1% |
| Offline | $130,112 | 48.9% |

Online has crossed the 50% mark, but the lead is only 2.2 percentage points — roughly 11 transactions separate the two channels. Neither can be deprioritized. The split warrants monitoring: a 2–3% shift either way has implications for store investment vs digital spend in 2026.

---

### 5.3 Monthly Revenue Trend

| Month | Revenue | vs March (Peak) |
|---|---|---|
| January | $23,480 | −17% |
| February | $20,040 | −29% |
| **March** | **$28,132** | **Peak** |
| April | $25,791 | −8% |
| May | $23,455 | −17% |
| June | $22,042 | −22% |
| July | $19,651 | −30% |
| **August** | **$15,483** | **−45%** |
| September | $27,756 | −1% |
| October | $20,118 | −29% |
| November | $21,675 | −23% |
| December | $18,494 | −34% |

**August is 45% below March**, and September recovers almost fully to March's level. A drop this sharp followed by a quick recovery is more consistent with a supply disruption or operational issue than a demand decline. This needs root-cause investigation by region and category before the Board presentation.

**December is also unexpectedly weak.** Year-end is typically a volume peak for retail. At $18,494 — below January and 34% below the March peak — this suggests either poor Q4 promotional execution, or a product category mix that does not respond to seasonal demand patterns.

---

### 5.4 Revenue by Region

| Region | Stores | Revenue | Share | Transactions | Avg/Transaction | Avg Revenue/Store |
|---|---|---|---|---|---|---|
| North | 9 | $114,626 | 43.1% | 435 | $263.51 | $12,736 |
| Central | 5 | $78,644 | 29.6% | 274 | $287.02 | $15,729 |
| South | 6 | $72,847 | 27.4% | 291 | $250.33 | $12,141 |

North's headline lead is a volume story — 9 stores driving 435 transactions. On a per-store basis, **Central is the most efficient region at $15,729/store**, 23% higher than North and 30% higher than South. Central also has the highest average transaction value ($287), meaning customers spend more per visit regardless of visit frequency. South trails on every metric: lowest average basket, lowest revenue per store.

---

### 5.5 Revenue by Store Type

| Store Type | Stores | Revenue | Share | Transactions | Avg/Transaction |
|---|---|---|---|---|---|
| Mini | 10 | $120,294 | 45.2% | 456 | $263.80 |
| Supermarket | 5 | $75,086 | 28.2% | 287 | $261.62 |
| Hypermarket | 5 | $70,737 | 26.6% | 257 | $275.24 |

Mini stores generate 45% of total revenue with a per-transaction value ($264) that is competitive against much larger format stores. Hypermarkets have the highest basket ($275) but the fewest transactions (257) — the format has pricing power but not traffic. Supermarkets sit in the middle on every metric with no standout characteristic, which is a strategic risk for a format competing against both extremes.

---

### 5.6 Revenue by Store Size

| Size | Stores | Revenue | Avg Revenue/Store | Avg/Transaction |
|---|---|---|---|---|
| Medium (M) | 8 | $112,224 | $14,028 | $282.68 |
| Small (S) | 7 | $90,788 | $12,970 | $267.02 |
| Large (L) | 5 | $63,106 | $12,621 | $239.95 |

**Larger stores are underperforming smaller ones on both a per-store and per-transaction basis.** Large stores average $12,621 in revenue per store — the lowest size band — and $239.95 per transaction, $43 less than Medium. This is counter-intuitive and needs a location or ranging explanation before drawing conclusions (see Section 5.10).

---

### 5.7 Revenue and Margin by Category

| Category | Revenue | Share | Transactions | Avg/Transaction | Gross Margin % |
|---|---|---|---|---|---|
| Dairy | $74,610 | 28.0% | 281 | $265.52 | 36.9% |
| Personal Care | $60,377 | 22.7% | 225 | $268.34 | 40.5% |
| Beverage | $54,157 | 20.3% | 214 | $253.07 | 18.4% |
| Household | $50,580 | 19.0% | 191 | $264.82 | 19.4% |
| Snack | $26,394 | 9.9% | 89 | $296.56 | 14.6% |

Three observations with direct business implications:

**Dairy leads on revenue volume, not margin.** At 28% of total revenue and 281 transactions, it is the business's volume foundation. Its 36.9% margin is healthy but not the best. A 1% margin improvement on Dairy outweighs the same improvement on any other category due to its scale.

**Personal Care has the best margin (40.5%) but low transaction count (225).** If stockout events are limiting availability — and they are (34 events, see Section 5.9) — there is a profit-mix improvement available without needing to grow total revenue.

**Snack has the highest basket ($296.56) and the fewest transactions (89).** 21% of Snack transactions (19 out of 89) occurred when stock was below reorder point. The most likely constraint on Snack's low volume is availability, not demand. This is explored further in Insight 6.

---

### 5.8 Promotion Effectiveness

**Overall:**

| Group | Transactions | Share | Avg Revenue/Tx | Avg Units/Tx |
|---|---|---|---|---|
| No Promotion | 718 | 71.8% | $264.30 | 10.29 |
| With Promotion | 282 | 28.2% | $270.74 | 10.68 |
| **Uplift** | — | — | **+2.4%** | **+3.8%** |

The program is running well below the 10–20% uplift threshold that typically justifies promotion cost. At 2.4% revenue uplift, the program is likely generating a net loss when discount depth is factored in — though this cannot be confirmed without the actual discount amounts per transaction.

**By category:**

| Category | No-Promo Avg Rev/Tx | Promo Avg Rev/Tx | Uplift |
|---|---|---|---|
| Snack | $283.52 | $322.22 | **+13.7%** |
| Personal Care | $260.04 | $286.72 | **+10.3%** |
| Beverage | $249.80 | $260.57 | +4.3% |
| Household | $263.31 | $269.42 | +2.3% |
| Dairy | $272.98 | $243.02 | **−11.0%** |

Snack (+13.7%) and Personal Care (+10.3%) are the only categories where promotion uplift clears the 10% threshold. Dairy promotions are actively reducing average revenue by 11% per transaction — approximately $30 less per promoted sale compared to a non-promoted one.

**By store — no meaningful correlation between promo rate and revenue:**

| Store | Promo Rate | Revenue | Revenue Rank |
|---|---|---|---|
| STR004 | 38.6% | $14,215 | 8th |
| STR013 | 34.2% | $9,644 | 19th |
| STR019 | 31.7% | $9,906 | 17th |
| STR010 | 32.8% | $18,534 | 1st |

STR010 runs promos on 32.8% of transactions and ranks 1st by revenue. STR013 runs promos on 34.2% of transactions and ranks 19th. Promotion frequency alone does not predict revenue outcomes.

---

### 5.9 Inventory Health

**Stockout risk:**

| Metric | Value |
|---|---|
| Transactions below reorder point | 196 / 1,000 (19.6%) |
| Revenue in at-risk transactions | $53,340 |
| Stores with at least 1 stockout event | 20 of 20 |

Every store in the network recorded at least one stockout-risk sale in 2025. At 1 in 5 transactions, the business is consistently operating inside its safety stock buffer.

**By category:**

| Category | Events | Revenue at Risk |
|---|---|---|
| Beverage | 50 | $11,381 |
| Dairy | 49 | $14,013 |
| Household | 44 | $14,238 |
| Personal Care | 34 | $7,707 |
| Snack | 19 | $6,001 |

Beverage and Dairy together account for 99 of 196 events — half the total risk in two daily-use categories where a stockout means an immediate lost sale with no substitution.

**By store (top 5 most exposed):**

| Store | Events | Revenue at Risk |
|---|---|---|
| STR014 | 24 | $4,851 |
| STR003 | 15 | $4,377 |
| STR010 | 14 | $4,019 |
| STR002 | 13 | $3,183 |
| STR005 | 13 | $3,675 |

STR014 has 24 events across 58 total transactions — 41% of its own sales occurred when stock was below reorder point.

**Stock system accuracy (avg gap = system minus physical):**

| Store | Avg Gap | Negative System Events |
|---|---|---|
| STR002 | −18.26 | 1 ⚠ |
| STR011 | −14.95 | 0 |
| STR015 | −12.00 | 0 |
| STR001 | −9.79 | 1 ⚠ |
| STR017 | +4.73 | 2 ⚠ |
| STR010 | +11.44 | 0 |
| STR016 | +18.32 | 0 |

A negative gap means the system understates physical stock — reorder triggers fire too early. A positive gap means the system overstates physical stock — reorder triggers fire too late, creating actual stockout risk. STR016 (+18.32) and STR010 (+11.44) have the largest positive gaps, consistent with their elevated stockout event counts.

**Confirmed negative system stock records (data integrity failures):**

| Store | Product | Physical Stock | System Stock |
|---|---|---|---|
| STR002 | PROD0001 | 0 | −10 |
| STR009 | PROD0057 | 19 | −1 |
| STR017 | PROD0054 | 6 | −7 |
| STR014 | PROD0039 | 0 | −19 |
| STR001 | PROD0056 | 14 | −3 |
| STR017 | PROD0043 | 8 | −4 |

STR017 appears twice, indicating a possible systematic sync failure at that store specifically.

---

### 5.10 What to Investigate Before the Final Presentation

1. **What caused the August revenue drop?** Break monthly revenue by region and store type. If the decline is concentrated in one segment, it is operational. If it is uniform across the network, it is seasonal or demand-related. The recommendation changes depending on the answer.

2. **Why does Snack have only 89 transactions?** 19 of those 89 (21%) occurred below reorder point. Confirm that Snack is availability-constrained, not demand-constrained, before presenting it as a growth opportunity.

3. **Why are Large stores underperforming per store?** Pull the 5 Large stores (STR002, STR009, STR011, STR013, STR017) and check their regions and revenue ranks. Three of the five are in the bottom 8 by revenue. Determine whether this is a location problem, a format problem, or a management problem.

4. **Fix the 6 negative stock_system records before the inventory section is presented.** Presenting reorder recommendations while leaving known data integrity errors in place will undermine credibility with the Operations team.

---

## VI — Dashboard Structure

The dashboard covers 28 slides across 6 sections. Each section opens with a divider slide and is self-contained for its intended audience.

### Section 1 — Business at a Glance *(Slides 3–6 · Audience: Everyone)*

No store names. Board-safe. Answers 4 questions using KPI Cards, a monthly column chart, and a Donut chart.

| Q# | Question | Visual Type | Key Output |
|---|---|---|---|
| Q1 | What was total revenue for 2025? | KPI Card | $266,117 |
| Q2 | How many transactions, and what was the avg basket? | KPI Cards | 1,000 tx · $266.12 avg |
| Q3 | How did revenue trend month by month? | Column chart by month | Peak: March $28,132 · Low: August $15,483 |
| Q4 | What is the Online vs Offline split? | Donut chart | Online 51.1% · Offline 48.9% |

DAX measures: `[Total Revenue]`, `[Total Transactions]`, `[Avg Transaction Value]`, `[MoM Revenue Change %]`, `[Online Revenue %]`, `[Offline Revenue %]`

---

### Section 2 — Where Is the Money Coming From? *(Slides 7–10 · Audience: Everyone)*

Introduces regional, format, size, and category breakdowns. BOD can exit after this section.

| Q# | Question | Visual Type | Key Output |
|---|---|---|---|
| Q5 | Which region contributed the most revenue — and most efficiently? | Horizontal Bar + regional KPI cards | North leads volume · Central leads efficiency ($15,729/store) |
| Q6 | Which store type generated the most revenue? | Column chart | Mini 45.2% of total revenue |
| Q7 | Do larger stores outperform smaller ones? | Clustered Bar (revenue + avg tx by size) | Large stores underperform Medium by 10% per store |
| Q8 | Which category brought in the most revenue and margin? | Column chart + margin table | Dairy leads revenue · Personal Care leads margin at 40.5% |

DAX measures: `[Total Revenue]`, `[Revenue per Store by Region]`, `[Store Type Revenue Share %]`, `[Gross Profit]`, `[Gross Margin %]`

---

### Section 3 — Individual Store Performance *(Slides 11–14 · Audience: Sales Head + Store Managers)*

Store names and individual revenue gaps appear from this section onward.

| Q# | Question | Visual Type | Key Output |
|---|---|---|---|
| Q9 | Who are the top 5 and bottom 5 stores by revenue? | Two ranked Bar charts | STR010 $18,534 (#1) vs STR007 $9,407 (#20) — 97% gap |
| Q10 | Which stores are above and below their regional average? | Table with conditional formatting | STR001 −30% vs Central avg · STR011 −25% vs North avg |
| Q11 | Which store type has the highest avg revenue per transaction? | Bar chart | Hypermarket $275.24 · Mini $263.80 · Supermarket $261.62 |
| Q12 | Which stores underperform on both revenue and units? | Filtered table (Dual Underperformer Flag = 1) | STR007, STR011, STR019, STR013 in bottom half on both dimensions |

DAX measures: `[Store Revenue Rank]`, `[Regional Avg Revenue per Store]`, `[vs Regional Avg %]`, `[Dual Underperformer Flag]`, `[Revenue Rank]`, `[Units Rank]`

---

### Section 4 — Are Promotions Working? *(Slides 15–18 · Audience: Sales Head)*

The most analytically dense section — requires the audience to understand comparing promoted vs non-promoted transaction groups.

| Q# | Question | Visual Type | Key Output |
|---|---|---|---|
| Q13 | What share of transactions used a promotion? | Donut + KPI Card | 28.2% — 282 of 1,000 transactions |
| Q14 | Do promotions lift revenue and units? | Clustered Bar (promo vs no-promo) | +2.4% revenue · +3.8% units — below the 10% minimum threshold |
| Q15 | Which categories respond best to promotions? | Horizontal Bar by category | Snack +13.7% · Personal Care +10.3% · Dairy −11.0% |
| Q16 | Which stores run the most promotions, and does it correlate with revenue? | Scatter plot + Bar chart | No meaningful correlation. STR013 runs 34.2% promo rate, ranks 19th |

DAX measures: `[Promo Transaction Share %]`, `[Promo Revenue Uplift %]`, `[Promo Units Uplift %]`, `[Category Promo Revenue Uplift %]`, `[Store Promo Rate %]`, `[Promo Rate Rank]`

---

### Section 5 — Inventory Health *(Slides 19–22 · Audience: Operations + Store Managers)*

Most operationally specific section. Contains data integrity flags that need to be resolved before this section can be presented with full confidence.

| Q# | Question | Visual Type | Key Output |
|---|---|---|---|
| Q17 | How many stores sold below reorder point at time of sale? | KPI Cards + ranked store table | 196 transactions (19.6%) · All 20 stores affected · STR014 worst with 24 events |
| Q18 | Which categories are most exposed to stockout risk? | Bar chart by category | Beverage 50 events · Dairy 49 · Household 44 |
| Q19 | How accurate is the stock system vs physical count? | Table with gap column + conditional formatting | STR002 avg gap −18.26 · STR016 avg gap +18.32 |
| Q20 | Which stores have negative system stock (data integrity issue)? | Flagged table (Has Negative System Stock = 1) | 6 records across 5 stores — STR017 appears twice |

DAX measures: `[Stockout Risk Transactions]`, `[Stockout Risk %]`, `[Stores with Stockout Risk]`, `[Category Stockout Risk Rate %]`, `[Stock Gap (System minus Physical)]`, `[Stock Accuracy %]`, `[Negative System Stock Events]`, `[Has Negative System Stock]`

---

### Section 6 — Manager Accountability *(Slides 23–27 · Audience: Store Managers)*

Individual-level. Every row in every table has a manager name attached.

| Q# | Question | Visual Type | Key Output |
|---|---|---|---|
| Q21 | What were each manager's revenue, margin, and avg basket in 2025? | Full 20-row scorecard table | Manager_10 leads $18,534 · Manager_7 last $9,407 · Manager_7 lowest margin 6.4% |
| Q22 | How does each store rank within its own region? | 3-column regional ranking table | Central avg $15,729 is 23% above North, 30% above South |
| Q23 | Which managers have inventory issues that likely cost them sales? | Highlighted panel: worst 4 offenders | Manager_14: 24 stockout events = 41% of own transactions |
| Q24 | Who are the top performers across revenue, margin, and inventory discipline combined? | Composite score table, sorted ascending | STR008 rank #1 · STR012 rank #2 · STR020 rank #3 |

DAX measures: `[Manager Regional Rank]`, `[Manager Stockout Events]`, `[Estimated Revenue at Inventory Risk]`, `[Inventory Risk Score]`, `[Composite Performance Score]`, `[Composite Performance Rank]`

---

## VII — Insights

### Insight 1 — Central Is the Network's Most Efficient Region, but the Headline Number Buries It

North's $114,626 total revenue will dominate any Board-level conversation. But North has 9 stores. Control for store count and the picture changes: **Central generates $15,729 per store vs North's $12,736** — a 23% efficiency gap. Central also posts the highest average transaction value at $287, vs $263 in North and $250 in South.

The business should study what Central stores are doing differently — product mix, manager quality, pricing discipline, or customer demographics — before prioritizing further investment in North. Replicating Central's operational model in South (which trails on every metric) is a more direct path to network improvement than adding more North volume.

---

### Insight 2 — The Promotion Program Is Not Paying for Itself, and Dairy Promotions Are Actively Destructive

The network-wide promotion uplift is 2.4% in revenue and 3.8% in units. A program typically needs 10–20% uplift to cover the cost of discounts plus operational and marketing spend. At 2.4%, the program is running at a probable net loss in 2025.

The breakdown by category makes the decision straightforward:

- **Snack +13.7% and Personal Care +10.3%** — the only two categories that justify promotion spend. Both clear the 10% minimum threshold.
- **Dairy −11.0%** — promoted Dairy transactions generate $30 less per sale than non-promoted ones. The category accounts for 281 transactions (28% of all sales). Every Dairy promotion that fires is reducing revenue in the business's largest category.

Recommended action: Reallocate the promotion budget entirely to Snack and Personal Care. Pause all Dairy promotions immediately. Audit which specific Dairy SKUs are being promoted and at what discount depth before redesigning the program.

---

### Insight 3 — There Is a 97% Revenue Gap Between the Best and Worst Store, and Both Are Mini Format

STR010 (Central, Mini, Manager_10): **$18,534** — #1 in the network.
STR007 (South, Mini, Manager_7): **$9,407** — #20 in the network.

Same format. The gap is 97%. Store format is not the explanation. What the data does show:

STR010 runs a $319.55 average basket — highest in the network. Its weakness is inventory (14 stockout events), which is a ceiling risk on a store that is otherwise executing well.

STR007 runs a $191.97 average basket — second lowest in the network — and a **6.4% gross margin**, the worst of any store. The margin problem is not a promotion issue or an inventory issue. It is a SKU mix problem — STR007 is predominantly selling very low-margin products. A product ranging audit is the first action, not a promotion campaign.

---

### Insight 4 — 19.6% of Transactions Are Running on Safety Stock

196 of 1,000 2025 transactions occurred when physical stock was below the reorder point. All 20 stores contributed. The total revenue processed in those at-risk moments is $53,340.

These were completed sales — not losses. But the safety stock buffer is being consumed at roughly 16 events per month network-wide. Given that monthly revenue already swings by up to $12,649 within the year (August vs March), a demand spike coinciding with depleted safety stock will produce real stockouts and real lost revenue.

The two most exposed categories — Beverage (50 events) and Dairy (49 events) — are daily-use staples with no substitution tolerance. A customer who cannot find milk or a soft drink does not wait; they leave.

STR014's situation is the most urgent: 24 stockout events out of 58 total transactions means 41% of its sales in 2025 happened below reorder point. It also carries a positive system-physical gap (+3.21), meaning the system is overstating what is physically on the shelf, which delays the reorder trigger. The data and the process are both working against this store.

Recommended actions: Review and increase Beverage and Dairy reorder points network-wide. Run a physical stock audit at STR014 and STR002 before making any reorder point changes at those two stores — their system data cannot be trusted in its current state.

---

### Insight 5 — The Best All-Round Operator Is Not the Top Revenue Store

When stores are ranked on a composite of revenue rank + gross margin rank + inventory discipline rank (fewer stockout events = better rank), the top performers shift away from pure revenue leaders:

| Composite Rank | Store | Revenue | Revenue Rank | Gross Margin | Stockout Events |
|---|---|---|---|---|---|
| #1 | STR008 | $14,536 | 7th | 37.1% | 8 |
| #2 | STR012 | $16,014 | 3rd | 44.7% | 11 |
| #3 | STR020 | $13,747 | 10th | 31.6% | 5 |
| #4 | STR003 | $17,332 | 2nd | 40.9% | 15 |
| #5 | STR015 | $13,735 | 11th | 28.6% | 5 |

STR008 (Manager_8, North, Hypermarket, Small) is the most balanced operator in the network. Its revenue rank (7th) is solid, its margin (37.1%) is among the highest, and it keeps stockout events at 8 — well-managed relative to stores with similar revenue. It is not the flashiest number in the table, but it is the store that is doing the most right across the full operating picture.

STR010 — the #1 revenue store — does not appear in the top 5 composite performers. Its 14 stockout events and 19.9% gross margin (5th lowest in the network) keep it out. Revenue alone is not a sufficient measure of a well-run store.

Recommended action: Use the composite score as the primary accountability metric presented to store managers in the 2026 planning cycle. Display it alongside absolute revenue so managers understand both what they produce and how efficiently they produce it.

---

### Insight 6 — Snack Is an Underserved, High-Potential Category Being Held Back by Availability

Snack has the highest average basket in the entire portfolio at **$296.56** — $28 more than the next best category (Personal Care at $268). It also has the strongest promotion response at +13.7%. But it generates only 89 transactions — the fewest of any category, roughly one-third of Dairy's 281.

The critical data point: **19 of those 89 Snack transactions (21%) occurred below reorder point.** That is the highest stockout rate of any category when measured relative to its own transaction volume. Customers who want to buy Snack are finding near-empty or empty shelves more than 1 in 5 times.

This combination — highest basket, strongest promotion response, highest relative stockout rate, lowest transaction count — strongly suggests the constraint on Snack is **availability, not demand**. The low transaction count is not a signal of weak demand; it is a signal that the category is not being stocked adequately to capture the demand that already exists.

If Snack reorder points are increased and the category is consistently available across all 20 stores, even a 30% increase in Snack transactions (27 additional sales at $297 average) would add approximately $8,000 in revenue at 14.6% margin — roughly $1,170 in additional gross profit from a single inventory policy change.

Recommended action: Increase Snack reorder points immediately. Run a targeted Snack promotion in the first 60 days after the reorder point change. Use Snack transaction count (not just revenue) as the primary KPI to confirm whether availability was the binding constraint.
