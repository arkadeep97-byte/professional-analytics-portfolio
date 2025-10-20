# Sales Performance Dashboard (Retail + eCommerce)

![Power BI](https://img.shields.io/badge/Power%20BI-Expert-F2C811?style=flat-square&logo=powerbi)
![SQL](https://img.shields.io/badge/SQL-Advanced-blue?style=flat-square&logo=postgresql)
![SAP](https://img.shields.io/badge/SAP_Analytics_Cloud-Advanced-0FAAFF?style=flat-square&logo=sap)

---

## 📋 Executive Summary

**Company:** Multi-brand beauty and cosmetics portfolio (200+ SKUs)  
**Role:** Business Analyst & BI Developer  
**Duration:** Oct 2023 - Aug 2025 (ongoing project with iterations)  
**Team:** Collaborated with Brand Managers, Category Leads, Key Account Managers, Finance

---

## 🎯 Business Problem

### Context
The company managed a diverse portfolio of 200+ beauty and cosmetics SKUs across multiple channels:
- Retail partnerships (drugstores, department stores)
- eCommerce platforms (own website + marketplaces)
- International markets (Germany, Netherlands, other EU)

### Challenges

**1. Data Fragmentation**
- Sales data scattered across 5+ systems (SAP, Shopify, marketplace APIs, retail EDI feeds)
- No single source of truth for performance metrics
- Different definitions of "sales" (sell-in vs sell-out) causing confusion

**2. Manual Reporting Overhead**
- Brand managers spending 6+ hours weekly compiling Excel reports
- Data freshness: 3-5 days lag from actuals
- Human error in manual consolidation

**3. Poor Forecast Accuracy**
- Forecast accuracy below 70%
- Reactive decision-making due to delayed insights
- Missed opportunities in high-performing SKUs

**4. Limited Self-Service**
- Ad-hoc requests overwhelming analytics team
- No ability for stakeholders to explore data independently
- C-suite visibility limited to monthly static reports

### Success Criteria
- Reporting time reduction by 30%+
- Forecast accuracy improvement to 85%+
- Self-service adoption by 80% of target users
- Daily data refresh with automated quality checks

---

## 🔧 My Solution

### Phase 1: Requirements Gathering (2 weeks)

**Stakeholder Interviews:**
- Conducted 12 interviews across Brand, Category, Finance, and Commercial teams
- Identified 15+ core KPIs required
- Mapped existing reporting workflows and pain points

**Key Requirements Identified:**
- Sell-in vs sell-out reconciliation
- Forecast vs actuals variance tracking
- Category and brand performance comparisons
- Channel-level profitability insights
- YoY and MoM trend analysis
- Drill-down capability to SKU level

---

### Phase 2: Data Architecture & Modeling (3 weeks)

**Data Sources Integrated:**

1. **SAP S/4HANA** (sell-in data, inventory)
   - Purchase orders
   - Invoicing data
   - Stock levels

2. **SAP Analytics Cloud** (forecasts, planning)
   - Monthly forecast submissions
   - Category planning data

3. **BigQuery** (eCommerce transactions)
   - Shopify sales
   - Google Analytics eCommerce events

4. **PostgreSQL** (retail sell-out)
   - EDI feeds from retail partners
   - POS data

5. **Excel Files** (promotional calendars, product master)
   - Promotional periods
   - SKU metadata and hierarchy

**Data Model Design:**

Created **star schema** with:
- **Fact Tables:** Sales transactions, Inventory snapshots, Forecast submissions
- **Dimension Tables:** Date, Product (with category/brand hierarchy), Channel, Geography, Customer

**Key Calculated Measures (DAX):**
```dax
// Sell-In (shipped to retailers)
Sell_In = 
    SUM(Sales[Quantity]) * 
    RELATED(Products[Unit_Price])

// Sell-Out (sold to end consumers)
Sell_Out = 
    CALCULATE(
        SUM(POS_Data[Quantity_Sold]),
        USERELATIONSHIP(Date[Date], POS_Data[Sale_Date])
    )

// Forecast Accuracy
Forecast_Accuracy = 
    1 - ABS(
        DIVIDE(
            [Actual_Sales] - [Forecasted_Sales],
            [Forecasted_Sales],
            0
        )
    )

// YoY Growth
YoY_Growth_Pct = 
    VAR CurrentYear = [Total_Sales]
    VAR PreviousYear = 
        CALCULATE(
            [Total_Sales],
            SAMEPERIODLASTYEAR(Date[Date])
        )
    RETURN
        DIVIDE(
            CurrentYear - PreviousYear,
            PreviousYear,
            BLANK()
        )

// Contribution Margin %
Contribution_Margin_Pct = 
    DIVIDE(
        [Gross_Margin],
        [Net_Revenue],
        0
    )
```

**ETL Pipeline:**

Built automated pipeline using **Power Query** and **SQL scripts**:
```sql
-- Example: Daily incremental load from SAP
WITH new_transactions AS (
    SELECT 
        transaction_id,
        product_id,
        order_date,
        quantity,
        net_amount,
        customer_id,
        channel
    FROM sap_sales
    WHERE order_date >= CURRENT_DATE - INTERVAL '7 days'
      AND processed_flag = 0
)
INSERT INTO dwh.fact_sales
SELECT 
    t.*,
    p.category_id,
    p.brand_id,
    CURRENT_TIMESTAMP as loaded_at
FROM new_transactions t
LEFT JOIN dwh.dim_products p 
    ON t.product_id = p.product_id;

-- Mark as processed
UPDATE sap_sales
SET processed_flag = 1
WHERE transaction_id IN (SELECT transaction_id FROM new_transactions);
```

---

### Phase 3: Dashboard Development (4 weeks)

**Dashboard Pages Created:**

**1. Executive Overview**
- High-level KPIs: Total sales, YoY growth, forecast accuracy
- Sales trend line (12-month rolling)
- Top/bottom performers (products, categories, channels)
- Alerts for significant variances

**2. Sell-In vs Sell-Out Analysis**
- Waterfall chart showing pipeline
- Inventory coverage days by SKU
- Potential stock-out alerts
- Slow-moving inventory identification

**3. Category Performance**
- Category comparison matrix
- Share of sales by category
- Margin analysis by category
- Promotional lift tracking

**4. Forecast Accuracy**
- Accuracy trending over time
- Decomposition by product, category, channel
- Error distribution (over-forecast vs under-forecast)
- Recommended adjustment factors

**5. Channel & Geography**
- Channel mix and profitability
- Geographic heatmap
- Retail partner performance scorecards
- eCommerce funnel analysis

**Design Principles:**
- **Color coding:** Green (on target), Yellow (warning), Red (alert)
- **Drill-through:** Click any chart to see SKU-level detail
- **Bookmarks:** Save filtered views for quick access
- **Mobile layout:** Optimized for tablet viewing

---

### Phase 4: Automation & Data Quality (2 weeks)

**Automated Refresh Schedule:**
- **Daily refresh** at 6:00 AM CET (before business hours)
- **Incremental load** for transactions (past 7 days)
- **Full refresh** on Sundays for dimension tables

**Data Quality Checks Implemented:**
```sql
-- Check 1: Revenue reconciliation
SELECT 
    'Revenue Mismatch' AS alert_type,
    SUM(dwh_revenue) AS dwh_total,
    SUM(sap_revenue) AS sap_total,
    ABS(SUM(dwh_revenue) - SUM(sap_revenue)) AS variance
FROM reconciliation_view
WHERE load_date = CURRENT_DATE
HAVING ABS(SUM(dwh_revenue) - SUM(sap_revenue)) > 1000;

-- Check 2: Missing product mappings
SELECT 
    'Unmapped Products' AS alert_type,
    COUNT(DISTINCT product_id) AS unmapped_count
FROM fact_sales
WHERE product_id NOT IN (SELECT product_id FROM dim_products)
  AND load_date = CURRENT_DATE;

-- Check 3: Duplicate transactions
SELECT 
    'Duplicate Check' AS alert_type,
    transaction_id,
    COUNT(*) AS dup_count
FROM fact_sales
WHERE load_date = CURRENT_DATE
GROUP BY transaction_id
HAVING COUNT(*) > 1;
```

**Alerting System:**
- Email alerts sent to analytics team if data quality checks fail
- Dashboard banner if data is stale (>24 hours old)
- Automatic rollback if critical errors detected

---

### Phase 5: Training & Rollout (2 weeks)

**User Enablement:**
- Conducted 4 training sessions (30 people total)
- Created user guide with screenshots and FAQs
- Established "office hours" for Q&A during first month
- Built video tutorials for common tasks

**Adoption Strategy:**
- Phased rollout: Executive team → Brand managers → Category leads
- Power users identified in each team as champions
- Weekly feedback sessions for first 6 weeks
- Iterative improvements based on user feedback

---

## 📈 Business Impact

### Quantitative Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Reporting Time** | 6 hrs/week | 3.9 hrs/week | **-35%** |
| **Data Freshness** | 3-5 days lag | <24 hours | **Real-time** |
| **Forecast Accuracy** | ~70% | ~85% | **+15 pts** |
| **Self-Service Adoption** | 0% | 85% | **85%** |
| **Ad-hoc Requests** | 15/week | 4/week | **-73%** |

### Qualitative Outcomes

**Brand Manager Feedback:**
> "I can now answer executive questions in real-time during meetings instead of saying 'I'll get back to you.' The drill-through features are incredibly powerful."

**Category Lead Feedback:**
> "Forecast accuracy improvements have allowed us to reduce safety stock and free up €200K in working capital."

**CFO Feedback:**
> "This dashboard has become the single source of truth for our monthly business reviews. The visibility into sell-through rates helps us make faster decisions on promotions and inventory."

### Strategic Impact
- ✅ Enabled **double-digit YoY growth** through faster insights
- ✅ Supported **€500K+ promotional spend optimization**
- ✅ Improved **collaboration between Brand and Finance teams**
- ✅ Freed up **10+ hours weekly** for analytics team to focus on strategic projects

---

## 🛠️ Technical Details

### Tech Stack

**BI & Visualization:**
- Power BI Desktop (report development)
- Power BI Service (publishing, sharing, scheduled refresh)
- DAX for calculations and measures
- Power Query M for ETL transformations

**Data Storage & Processing:**
- SAP S/4HANA (source system)
- SAP Analytics Cloud (planning data)
- BigQuery (eCommerce warehouse)
- PostgreSQL (retail POS data)

**Orchestration & Automation:**
- Power Automate (refresh triggers, alert emails)
- SQL Server Agent (database maintenance)
- Python scripts (data validation, backup)

**Development & Version Control:**
- Git (for SQL scripts and documentation)
- Confluence (technical documentation)
- Jira (requirement tracking, bug fixes)

---

### Data Model Specifications

**Fact Tables:**
- `fact_sales` - 2M+ rows, daily grain
- `fact_inventory` - 50K+ rows, daily snapshots
- `fact_forecast` - 100K+ rows, monthly submissions

**Dimension Tables:**
- `dim_date` - 3 years of dates with fiscal calendar
- `dim_products` - 200+ SKUs with 3-level hierarchy
- `dim_channels` - Retail, eCommerce, Wholesale
- `dim_geography` - Countries, regions, cities
- `dim_customers` - Retail partners, marketplaces

**Refresh Logic:**
- Incremental refresh on fact tables (past 7 days)
- Full refresh on dimensions (weekly)
- Partitioning by month for performance

---

### Power BI Performance Optimization

**Measures Used:**
- DAX variables to avoid repeated calculations
- `CALCULATE` with explicit filters
- `TREATAS` for virtual relationships
- Aggregation tables for large datasets

**Model Optimization:**
- Removed unnecessary columns
- Changed data types to minimize memory
- Disabled auto-date/time tables
- Used DirectQuery for real-time inventory views

**Query Performance:**
- Average dashboard load time: 3-4 seconds
- Largest visual render time: <2 seconds
- Data model size: ~500 MB compressed

---

## 📁 Assets

Due to confidentiality, actual files cannot be shared. Below are descriptions of available anonymized materials:

### Available in `assets/` folder:
- `dashboard_overview.png` - Homepage with KPIs (anonymized labels)
- `data_model_diagram.png` - Star schema ERD
- `forecast_accuracy_page.png` - Accuracy tracking visuals
- `before_after_comparison.png` - Old vs new reporting process

### Available in `methodology/` folder:
- `technical_approach.md` - Detailed ETL logic and DAX formulas
- `data_dictionary.md` - Field definitions and business rules
- `user_guide.pdf` - Training materials (sanitized)

---

## 🔍 Challenges & Learnings

### Challenges Faced

**1. Data Quality Issues**
- Initial data had 15%+ error rate (mismatched product IDs, duplicate transactions)
- **Solution:** Built reconciliation layer and automated quality checks

**2. Stakeholder Alignment**
- Different teams had conflicting definitions of metrics
- **Solution:** Facilitated workshops to agree on standard KPI definitions

**3. Performance with Large Datasets**
- Initial model took 30+ seconds to load
- **Solution:** Implemented aggregation tables and optimized DAX

**4. Change Management**
- Some users resistant to moving from Excel
- **Solution:** Showed side-by-side comparison of time saved, ran parallel systems for 1 month

### Key Learnings

✅ **Requirements must be crystal clear** - Spent extra time upfront defining KPIs, saved weeks of rework  
✅ **Data governance is critical** - Without clear ownership and definitions, dashboards become "garbage in, garbage out"  
✅ **Training is ongoing** - One training session isn't enough; needed continuous support  
✅ **Iterate based on feedback** - Launched v1 quickly, improved iteratively based on user input  
✅ **Performance matters** - Slow dashboards = low adoption; invest in optimization early

---

## 🚀 Future Enhancements

**Short-term (Next 3 months):**
- [ ] Add predictive analytics for demand forecasting
- [ ] Integrate social media sentiment data
- [ ] Build mobile app for field sales team
- [ ] Add customer segmentation analysis

**Long-term (6-12 months):**
- [ ] Implement machine learning for price optimization
- [ ] Build real-time alert system for anomalies
- [ ] Integrate competitive intelligence data
- [ ] Create scenario planning module for promotions

---

## 📚 Related Work

- [A/B Testing Framework](../project_05_ab_testing/) - Campaign performance tracking
- [Pricing & P&L Analysis](../project_04_pricing_pnl/) - Financial modeling

---

**Project Duration:** 11 weeks (requirements → rollout)  
**Maintenance:** Ongoing support and enhancements  
**Team Size:** 1 analyst (me) + stakeholders across 4 teams

---

⭐️ **Interested in similar dashboards?** Check out my other [professional projects](../)

---

*Last Updated: Oct 2025*
