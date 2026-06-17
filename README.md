# Marketplace Analytics SQL Case Study

## Project Overview

This project is a PostgreSQL-based marketplace analytics case study using a fictional marketplace dataset. The objective is to demonstrate how SQL can be used to answer practical business questions commonly faced by marketplace, e-commerce, and digital product teams.

The analysis focuses on foundational marketplace metrics such as data scale, order status distribution, GMV, discounts, net paid amount, category performance, seller contribution, product demand, payment behavior, cancellation risk, monthly active buyers/sellers, and official-store performance.

## Dataset Structure

The dataset contains transaction, catalog, seller, customer, campaign, logistics, review, refund, and event data. The image below summarizes what is included in each table.

![Marketplace table overview](assets/marketplace_table_overview.png)

The simplified relationship map below shows the main analytical joins used in this project.

![Marketplace table relationship map](assets/marketplace_table_relationship_map.png)

## Key Metric Definitions

| Metric | Definition |
|---|---|
| GMV | Gross merchandise value before discounts. In the `orders` table, this is represented by `gross_amount`. At item level, it can be calculated using `SUM(item_gross_amount)` or `SUM(price_at_purchase * quantity)`. |
| Product Discount | Discount applied directly to product/item price. |
| Voucher Discount | Discount funded through voucher or campaign mechanism. |
| Shipping Fee | Delivery fee charged to the buyer. This is added to the buyer's payment amount. |
| Net Paid Amount | Final buyer payment amount after discounts and shipping fee. Formula: `gross_amount - product_discount_amount - voucher_discount_amount + shipping_fee`. |
| AOV | Average order value. Formula: `GMV / order count`. |
| Units Sold | Total quantity of products sold. This should be calculated using `SUM(quantity)`, not `COUNT(order_id)` or `COUNT(order_item_id)`. |
| Cancellation Rate | Cancelled orders divided by total orders in the same segment. |
| Active Buyers | Unique customers with at least one completed order in a given period. |
| Active Sellers | Unique sellers with at least one completed order in a given period. |
| Completed Orders | Orders where `order_status = 'completed'`. These are used as the baseline for most revenue and sales analyses. |

## Important Analytical Assumptions

Revenue and sales analysis mainly uses completed orders because cancelled, pending, returned, or incomplete orders can distort GMV, AOV, and seller/product performance. GMV and net paid amount are treated as different concepts: GMV reflects merchandise value before discounts, while net paid amount reflects what the buyer actually pays after discounts and shipping fee. Product-level and seller-level analysis uses item-level data carefully because joining order-level and item-level tables can duplicate metrics when the grain is not handled properly.

## Analysis Questions and SQL Queries

### 1. Count Rows in Every Table

This analysis profiles the scale of the marketplace dataset across each core entity, including transactions, customers, sellers, catalog, payments, logistics, reviews, refunds, campaigns, and behavior events. It matters because a strong analyst should first understand dataset coverage before interpreting performance metrics.

```sql
SELECT 'campaigns' AS table_name, COUNT(*) AS row_count FROM campaigns
UNION ALL SELECT 'categories', COUNT(*) FROM categories
UNION ALL SELECT 'customers', COUNT(*) FROM customers
UNION ALL SELECT 'orders', COUNT(*) FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL SELECT 'payments', COUNT(*) FROM payments
UNION ALL SELECT 'product_events', COUNT(*) FROM product_events
UNION ALL SELECT 'products', COUNT(*) FROM products
UNION ALL SELECT 'refunds', COUNT(*) FROM refunds
UNION ALL SELECT 'reviews', COUNT(*) FROM reviews
UNION ALL SELECT 'sellers', COUNT(*) FROM sellers
UNION ALL SELECT 'shipments', COUNT(*) FROM shipments
UNION ALL SELECT 'vouchers', COUNT(*) FROM vouchers
ORDER BY table_name;
```

The `orders` table is included because it is the main transaction table. Removing it from profiling would make the dataset overview incomplete.

### 2. Show Order Volume by Order Status and Month

This analysis shows how orders are distributed across statuses over time. It helps detect whether the marketplace is mostly generating completed transactions or whether operational issues appear through cancelled, pending, returned, or shipped-but-not-completed orders.

```sql
SELECT
    DATE_TRUNC('month', order_datetime)::date AS order_month,
    order_status,
    COUNT(*) AS order_count
FROM orders
GROUP BY 1, 2
ORDER BY 1, 2;
```

This query intentionally does not filter only completed orders because the purpose is to understand the full operational status distribution.

### 3. Calculate Monthly GMV, Net Paid Amount, Product Discount, Voucher Discount, and Shipping Fee

This analysis summarizes monthly commercial performance by separating GMV, product discount, voucher discount, shipping fee, and net paid amount. It matters because GMV and buyer payment are different business concepts, and mixing them can lead to wrong conclusions about revenue, discount burn, and marketplace value.

```sql
SELECT
    DATE_TRUNC('month', order_datetime)::date AS order_month,
    SUM(gross_amount) AS gmv,
    SUM(product_discount_amount) AS product_discount_total,
    SUM(voucher_discount_amount) AS voucher_discount_total,
    SUM(shipping_fee) AS shipping_fee_total,
    SUM(net_paid_amount) AS net_paid_total,
    SUM(
        gross_amount
        - product_discount_amount
        - voucher_discount_amount
        + shipping_fee
    ) AS recalculated_net_paid,
    SUM(net_paid_amount)
    - SUM(
        gross_amount
        - product_discount_amount
        - voucher_discount_amount
        + shipping_fee
    ) AS net_paid_difference
FROM orders
WHERE order_status = 'completed'
GROUP BY 1
ORDER BY 1;
```

Net paid amount is validated using `gross_amount - product_discount_amount - voucher_discount_amount + shipping_fee`. Shipping fee is added because it is part of the buyer's final payment.

### 4. Find Top 20 Categories by Completed GMV and Units Sold

This analysis identifies the categories generating the highest completed GMV and product units sold. It supports category prioritization, campaign planning, seller acquisition focus, and merchandising decisions.

```sql
SELECT
    c.category_id,
    c.parent_category,
    c.category_name,
    SUM(oi.quantity) AS units_sold,
    COUNT(DISTINCT o.order_id) AS completed_orders,
    SUM(oi.item_gross_amount) AS gmv
FROM order_items oi
JOIN categories c
    ON oi.category_id = c.category_id
JOIN orders o
    ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
GROUP BY 1, 2, 3
ORDER BY gmv DESC, units_sold DESC
LIMIT 20;
```

Units sold must use `SUM(oi.quantity)`, not `COUNT(oi.quantity)`. Counting quantity only counts item rows, while summing quantity counts actual units sold.

### 5. Find Top 20 Sellers by Completed GMV and Marketplace Commission

This analysis identifies sellers that contribute the most completed GMV and marketplace commission. It is useful for seller management because high-value sellers may require retention support, account management, operational monitoring, or partnership treatment.

```sql
SELECT
    s.seller_id,
    s.shop_name,
    s.seller_tier,
    COUNT(DISTINCT o.order_id) AS completed_orders,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.item_gross_amount) AS gmv,
    SUM(oi.platform_commission_amount) AS marketplace_commission
FROM order_items oi
JOIN sellers s
    ON oi.seller_id = s.seller_id
JOIN orders o
    ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
GROUP BY 1, 2, 3
ORDER BY gmv DESC, marketplace_commission DESC
LIMIT 20;
```

GMV is calculated from item-level data using `SUM(oi.item_gross_amount)` to avoid duplicating order-level values when working at seller level.

### 6. Find Top 20 Products by Units Sold

This analysis finds the strongest products by actual units sold. It helps identify high-demand products for inventory planning, campaign promotion, seller support, and product recommendation strategy.

```sql
SELECT
    p.product_id,
    p.product_name,
    SUM(oi.quantity) AS units_sold,
    COUNT(DISTINCT o.order_id) AS completed_orders,
    SUM(oi.item_gross_amount) AS gmv
FROM order_items oi
JOIN products p
    ON oi.product_id = p.product_id
JOIN orders o
    ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
GROUP BY 1, 2
ORDER BY units_sold DESC, gmv DESC
LIMIT 20;
```

The important correction is to use `SUM(oi.quantity)` for product sold quantity. A single order item row can represent more than one unit.

### 7. Calculate AOV by Payment Method and Source Platform

This analysis compares average order value across payment methods and source platforms. It helps reveal differences in customer behavior, transaction size, payment preference, and platform-level monetization quality.

```sql
SELECT
    payment_method,
    source_platform,
    COUNT(*) AS completed_orders,
    SUM(gross_amount) AS gmv,
    ROUND(
        SUM(gross_amount) / NULLIF(COUNT(*), 0),
        2
    ) AS aov
FROM orders
WHERE order_status = 'completed'
GROUP BY 1, 2
ORDER BY source_platform, payment_method;
```

`AVG(gross_amount)` would work at pure order grain, but `SUM(gross_amount) / COUNT(*)` makes the business definition clearer: AOV equals GMV divided by completed order count.

### 8. Calculate Cancellation Rate by Payment Method and Source Platform

This analysis identifies payment methods and source platforms with higher cancellation risk. It helps narrow down where the business should investigate checkout friction, payment failure, UX issues, buyer behavior, or operational problems.

```sql
SELECT
    payment_method,
    source_platform,
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (
        WHERE order_status = 'cancelled'
    ) AS cancelled_orders,
    ROUND(
        100.0
        * COUNT(*) FILTER (WHERE order_status = 'cancelled')
        / NULLIF(COUNT(*), 0),
        2
    ) AS cancellation_rate_pct
FROM orders
GROUP BY 1, 2
ORDER BY cancellation_rate_pct DESC, total_orders DESC;
```

Cancellation should be analyzed as a rate, not only as raw cancelled order count. A segment with more orders may naturally have more cancellations, so percentage-based comparison is fairer.

### 9. Calculate Monthly Active Buyers and Monthly Active Sellers

This analysis tracks active demand and active supply in the marketplace by month. Active buyers represent customer purchasing activity, while active sellers represent supply-side participation in successful transactions.

```sql
SELECT
    DATE_TRUNC('month', order_datetime)::date AS order_month,
    COUNT(DISTINCT customer_id) AS active_buyers,
    COUNT(DISTINCT seller_id) AS active_sellers
FROM orders
WHERE order_status = 'completed'
GROUP BY 1
ORDER BY 1;
```

This query defines activity based on completed orders. A buyer or seller is only counted as active if they participated in at least one successful transaction in the month.

### 10. Compare Official Store vs Non-Official Store Performance

This analysis compares official and non-official stores across active sellers, completed orders, units sold, GMV, marketplace commission, and AOV. It helps evaluate whether official-store status is associated with stronger commercial performance or seller productivity.

```sql
SELECT
    DATE_TRUNC('month', o.order_datetime)::date AS order_month,
    CASE
        WHEN s.is_official_store = TRUE THEN 'official'
        ELSE 'non_official'
    END AS store_type,
    COUNT(DISTINCT oi.seller_id) AS active_sellers,
    COUNT(DISTINCT o.order_id) AS completed_orders,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.item_gross_amount) AS gmv,
    SUM(oi.platform_commission_amount) AS marketplace_commission,
    ROUND(
        SUM(oi.item_gross_amount)
        / NULLIF(COUNT(DISTINCT o.order_id), 0),
        2
    ) AS aov
FROM order_items oi
JOIN sellers s
    ON oi.seller_id = s.seller_id
JOIN orders o
    ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
GROUP BY 1, 2
ORDER BY 1, 2;
```

Because this query joins `orders` and `order_items`, order count must use `COUNT(DISTINCT o.order_id)` to avoid duplicate orders. Units sold should use `SUM(oi.quantity)` because item rows do not always equal product units.

## Review Notes from Query Revision

The submitted SQL logic was generally on the right track, but several corrections were needed before publishing. Debug statements were removed, the `orders` table was added to row-count profiling, `COUNT(quantity)` was replaced with `SUM(quantity)`, AOV was written explicitly as `GMV / order count`, and order counts after item-level joins were changed to `COUNT(DISTINCT order_id)` where needed.

## Analytical Learnings

This project highlights that SQL analysis is not only about writing syntactically correct queries. Good marketplace analytics requires choosing the right metric grain, separating GMV from net paid amount, avoiding duplicated order-level values after joins, using completed orders for clean commercial baselines, calculating units sold from quantity, and comparing operational issues with rates rather than raw counts.

## Repository Structure

```text
marketplace-sql-analytics-case-study/
│
├── README.md
├── assets/
│   ├── marketplace_table_overview.png
│   └── marketplace_table_relationship_map.png
│
├── sql/
│   ├── 01_10_foundational_marketplace_analysis.sql
│   └── individual_query_files.sql
│
└── outputs/
    └── screenshots/
```

## Portfolio Positioning

This project demonstrates my ability to use SQL not only for data extraction, but also for business analysis. The queries are designed to answer practical marketplace questions, define relevant KPIs, handle relational data correctly, and avoid common analytical mistakes such as double-counting, incorrect denominator selection, and confusion between order-level and item-level metrics.
