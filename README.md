# Enterprise Data Lakehouse - Microsoft Fabric

## Project Overview

This project implements an end-to-end Data Lakehouse on Microsoft Fabric. Raw e-commerce data is ingested from Kaggle API, cleaned through a Medallion Architecture (Bronze → Silver → Gold), and visualized in a Power BI dashboard.

---

## Data Source

**Dataset:** Brazilian E-Commerce Public Dataset by Olist  
**Source:** Kaggle REST API  
**Size:** ~120 MB | 9 CSV files | 100,000+ orders  
**Period:** September 2016 – October 2018  

Files ingested:
- Orders, Order Items, Customers, Products, Sellers
- Payments, Reviews, Geolocation, Category Translations

---

## Process Followed

### Bronze Layer — Raw Ingestion

- Called Kaggle REST API using Python `requests` library
- Downloaded dataset as ZIP file into memory
- Extracted 9 CSV files to OneLake path: `Files/bronze/raw/`
- No transformations applied — raw data preserved as-is

---

### Silver Layer — Cleaning & Transformation

Read each CSV from Bronze, applied data quality rules, saved as Delta tables.

| Table | Transformations Applied |
|-------|------------------------|
| silver_orders | Added delivery_status (DELIVERED/NOT_DELIVERED) and approval_status (APPROVED/NOT_APPROVED) based on null timestamps |
| silver_customers | Standardized city names to proper case |
| silver_products | Fixed column typos (lenght→length), null categories set to "uncategorized", null dimensions set to 0, joined English category names from translation file |
| silver_sellers | Standardized city names to proper case |
| silver_payments | Standardized payment type names to proper case |
| silver_reviews | Dropped null ID rows, cast review_score to integer, filled missing comments with defaults, cast date strings to timestamps |
| silver_geolocation | Deduplicated from 1,000,163 rows to 27,912 unique zip codes by averaging lat/lng coordinates |
| silver_order_items | Added calculated column: total_item_value = price + freight_value |

---

### Gold Layer — Business Aggregation

Joined Silver tables and created pre-aggregated tables for reporting. 

**Key Rule:** Only orders with `order_status = "delivered"` are included in all Gold calculations.

#### gold_executive_kpis
Single-row summary with overall business metrics:
- Total Revenue (~$15.42M)
- Total Orders (~96,478)
- Total Customers (~93,471)
- Avg Order Value (~$159.83)
- Avg Review Score (~4.07)
- Total Sellers (~3,095)

#### gold_monthly_revenue
Revenue and order trends grouped by year and month:
- total_revenue, total_orders, total_items_sold, avg_order_value

#### gold_customer_analysis
Customer segmentation by lifetime spending:
- Grouped by customer_unique_id (true customer identity)
- Tier classification:
  - High Value: spent ≥ $1,000 (1,145 customers)
  - Mid Value: spent ≥ $500 (3,115 customers)
  - Low Value: spent < $500 (89,211 customers)

#### gold_product_performance
Product category metrics:
- units_sold, total_revenue, avg_price, avg_review_score per product

#### gold_seller_performance
Seller evaluation with delivery efficiency:
- total_revenue, avg_review_score, avg_delivery_days per seller
- Delivery days = (delivered_timestamp - purchase_timestamp) / 86400
- Tier classification:
  - Top Seller: revenue ≥ $50,000
  - Mid Seller: revenue ≥ $10,000
  - Regular Seller: revenue < $10,000

---

## Power BI Dashboard

Single-page executive dashboard built from Gold tables:

| Visual | Source | What It Shows |
|--------|--------|---------------|
| 6 KPI Cards | gold_executive_kpis | Revenue, Orders, Customers, AOV, Rating, Sellers |
| Line Chart | gold_monthly_revenue | Monthly revenue trend with separate lines per year |
| Column Chart | gold_monthly_revenue | Monthly order volume grouped by year |

Connected via Semantic Model created from the 5 Gold Delta tables in Fabric.