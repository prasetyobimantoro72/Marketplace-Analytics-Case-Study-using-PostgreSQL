# Marketplace Analytics SQL Case Study

## Project Overview

This project is a PostgreSQL-based marketplace analytics case study using a fictional marketplace dataset. The objective is to demonstrate how SQL can be used to answer practical business questions commonly faced by marketplace, e-commerce, and digital product teams.

The analysis focuses on foundational marketplace metrics such as data scale, order status distribution, GMV, discounts, net paid amount, category performance, seller contribution, product demand, payment behavior, cancellation risk, monthly active buyers/sellers, and official-store performance.

## Dataset Structure

The dataset contains transaction, catalog, seller, customer, campaign, logistics, review, refund, and event data. The image below summarizes what is included in each table.

![Marketplace table overview](assets/marketplace_table_content.png)

The simplified relationship map below shows the main analytical joins used in this project.

![Marketplace table relationship map](assets/marketplace_table_relationship_diagram.png)

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

## Analysis Questions, SQL Queries, Results, and Insights

The result tables below are concise previews generated from the same fictional marketplace dataset. The column names in each result preview match the SQL aliases shown in the query.

### 1. Count Rows in Every Table

This query profiles the number of rows in every major marketplace table. It is important because the first analytical step is understanding which tables are transaction-heavy and which tables are smaller dimensions.

```sql
SELECT 'campaigns' AS table_name, COUNT(*) AS row_number FROM campaigns
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
UNION ALL SELECT 'vouchers', COUNT(*) FROM vouchers;
```

**Result preview**

| table_name     | row_number   |
|:---------------|:-------------|
| campaigns      | 14           |
| categories     | 32           |
| customers      | 60,000       |
| orders         | 250,000      |
| order_items    | 424,097      |
| payments       | 250,000      |
| product_events | 1,000,000    |
| products       | 40,000       |
| refunds        | 16,517       |
| reviews        | 135,892      |
| sellers        | 4,000        |
| shipments      | 225,600      |
| vouchers       | 70           |

**Brief insight**

The dataset contains more than 2.4 million rows across 13 tables. `product_events` is the largest table with 1,000,000 rows, while `orders` and `payments` each contain 250,000 rows. Event-level and transaction-level analyses will therefore be the heaviest parts of the database.

### 2. Show Order Volume by Order Status and Month

This query shows order volume by monthly period and `order_status`. It helps identify whether most orders are completed or whether operational issues appear through cancelled, pending, returned, or shipped statuses.

```sql
SELECT
    DATE_TRUNC('month', order_datetime) AS order_month,
    order_status,
    COUNT(order_id) AS order_count
FROM orders
GROUP BY 2, 1
ORDER BY 2, 1 ASC;
```

This query intentionally does not filter only completed orders because the purpose is to understand the full operational status distribution.


**Result Preview: Overall Status Distribution**

| order_status   | order_count   |   share_pct |
|:---------------|:--------------|------------:|
| completed      | 205,170       |       82.07 |
| cancelled      | 20,380        |        8.15 |
| shipped        | 12,877        |        5.15 |
| returned       | 7,553         |        3.02 |
| pending        | 4,020         |        1.61 |

**Result preview: latest monthly rows**

| order_month   | order_status   | order_count   |
|:--------------|:---------------|:--------------|
| 2026-04-01    | cancelled      | 573           |
| 2026-04-01    | completed      | 6,227         |
| 2026-04-01    | pending        | 116           |
| 2026-04-01    | returned       | 214           |
| 2026-04-01    | shipped        | 376           |
| 2026-05-01    | cancelled      | 786           |
| 2026-05-01    | completed      | 7,715         |
| 2026-05-01    | pending        | 158           |
| 2026-05-01    | returned       | 271           |
| 2026-05-01    | shipped        | 510           |

**Brief insight**

Completed orders dominate the marketplace at 82.07% of total order, but cancellation remains a meaningful share of total orders, account for 8.15%. This makes cancellation a valid follow-up area for payment, platform, seller, or fulfillment analysis.

### 3. Calculate Monthly GMV, Net Paid Amount, Product Discount, Voucher Discount, and Shipping Fee

This query summarizes completed-order commercial performance by month. It separates GMV, shipping fee, product discount, voucher discount, and net paid amount so that sales value and buyer-paid value are not confused.

```sql
SELECT 
    DATE_TRUNC('month', order_datetime) AS order_month,
    SUM(gross_amount) AS gmv,
    SUM(shipping_fee) AS shipping_fee_total,
    SUM(product_discount_amount) AS product_discount_total,
    SUM(voucher_discount_amount) AS voucher_discount_total,
    SUM(net_paid_amount) AS net_paid_total
FROM orders
WHERE order_status = 'completed'
GROUP BY 1
ORDER BY 1;
```

**Result Preview**

| order_month   | gmv            | shipping_fee_total   | product_discount_total   | voucher_discount_total   | net_paid_total   |
|:--------------|:---------------|:---------------------|:-------------------------|:-------------------------|:-----------------|
| 2026-01-01    | 12,599,214,000 | 76,897,000           | 308,709,880              | 0                        | 12,367,401,120   |
| 2026-02-01    | 19,126,648,000 | 115,250,000          | 1,171,532,640            | 114,738,809              | 17,955,626,551   |
| 2026-03-01    | 25,917,835,000 | 158,482,000          | 1,601,350,860            | 165,512,656              | 24,309,453,484   |
| 2026-04-01    | 13,382,138,000 | 79,256,000           | 319,876,810              | 0                        | 13,141,517,190   |
| 2026-05-01    | 16,588,962,000 | 99,040,000           | 831,688,310              | 94,612,828               | 15,761,700,862   |

**Brief insight**

Across completed orders, total GMV reaches 430,792,971,000, while total net paid amount reaches 411,901,598,002. Product discounts (20,099,242,320) are much larger than voucher discounts (1,406,947,678), which suggests that product-level discounting is the bigger discount lever in this dataset. The validation difference is 0, confirming that the net paid formula is consistent.

### 4. Find Top 20 Categories by Completed GMV and Units Sold

This query ranks categories by completed GMV and units sold. It supports category prioritization, campaign planning, seller acquisition, and merchandising decisions.

```sql
SELECT
    c.category_id,
    c.parent_category,
    c.category_name,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.item_gross_amount) AS gmv
FROM order_items oi
JOIN categories c
    ON oi.category_id = c.category_id
JOIN orders o
    ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
GROUP BY 1, 2, 3
ORDER BY 5 DESC, 4 DESC
LIMIT 20;
```

**Result preview**

| category_id   | parent_category   | category_name       | units_sold   | gmv             |
|:--------------|:------------------|:--------------------|:-------------|:----------------|
| CAT004        | Electronics       | Computers & Laptops | 13,823       | 172,408,752,000 |
| CAT001        | Electronics       | Smartphones         | 12,912       | 70,690,224,000  |
| CAT015        | Home & Living     | Furniture           | 13,619       | 30,257,522,000  |
| CAT003        | Electronics       | Home Appliances     | 13,500       | 19,973,004,000  |
| CAT012        | Beauty            | Fragrance           | 22,768       | 12,469,337,000  |

**Brief insight**

Computers & Laptops is the strongest category by GMV, followed by Smartphones. Some categories such as Fragrance, Bags, and Shoes generate high unit volume but lower GMV than electronics, showing why units sold and GMV should be interpreted separately.

### 5. Find Top 20 Sellers by Completed GMV and Marketplace Commission

This query ranks sellers by completed GMV and marketplace commission. It helps identify high-value sellers that may deserve retention support, account management, or operational monitoring.

```sql
SELECT
    s.seller_id,
    s.shop_name,
    s.seller_tier,
    SUM(oi.item_gross_amount) AS gmv,
    SUM(oi.platform_commission_amount) AS marketplace_commission
FROM order_items oi
JOIN sellers s
    ON oi.seller_id = s.seller_id
JOIN orders o
    ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
GROUP BY 1, 2, 3
ORDER BY 4 DESC, 5 DESC
LIMIT 20;
```

**Result preview**

| seller_id   | shop_name          | seller_tier   | gmv           | marketplace_commission   |
|:------------|:-------------------|:--------------|:--------------|:-------------------------|
| S00721      | Sinar House 721    | Silver        | 1,551,820,000 | 87,218,641               |
| S01581      | Murah Tech 1581    | Silver        | 1,074,377,000 | 47,249,443               |
| S00852      | Global Gallery 852 | Silver        | 1,023,008,000 | 57,155,455               |
| S00900      | Prima Gallery 900  | Silver        | 974,264,000   | 53,300,364               |
| S02936      | Global House 2936  | Bronze        | 966,312,000   | 46,579,689               |

**Brief insight**

The top sellers are concentrated mostly in Silver and Bronze tiers. The highest-GMV seller generated over 1.55 billion in completed item GMV, but the ranking also shows that strong seller value can come from relatively small order counts when the seller carries higher-ticket products.

### 6. Find Top 20 Products by Units Sold

This query ranks products by total units sold. It is useful for identifying high-demand products for inventory monitoring, campaign promotion, and recommendation support.

```sql
SELECT
    p.product_id,
    p.product_name,
    SUM(oi.quantity) AS unit_sold,
    SUM(oi.item_gross_amount) AS gmv
FROM order_items oi
JOIN products p
    ON oi.product_id = p.product_id
JOIN orders o
    ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
GROUP BY 1, 2
ORDER BY 3 DESC, 4 DESC
LIMIT 20;
```

**Result preview**

| product_id   | product_name                      |   unit_sold | gmv       |
|:-------------|:----------------------------------|------------:|:----------|
| P0020067     | Tupperware Kitchenware Item 20067 |         167 | 5,845,000 |
| P0007794     | Scarlett Skincare Item 7794       |          72 | 3,600,000 |
| P0012128     | Wings Fresh Food Item 12128       |          72 | 936,000   |
| P0036893     | Unilever Fresh Food Item 36893    |          70 | 5,460,000 |
| P0039271     | Berrybenka Bags Item 39271        |          69 | 6,624,000 |

**Brief insight**

The highest-unit product sold 167 units, but its GMV is much lower than some products with fewer units. This reinforces that top products by quantity are not always the same as top products by revenue.

### 7. Calculate AOV by Payment Method and Source Platform

This query calculates AOV by `payment_method` and `source_platform`. It helps identify payment-platform combinations associated with higher-value completed orders.

```sql
SELECT 
    payment_method,
    source_platform,
    AVG(gross_amount) AS aov
FROM orders
WHERE order_status = 'completed'
GROUP BY 1, 2
ORDER BY 2, 1;
```

**Result preview**

| payment_method   | source_platform   | aov          |
|:-----------------|:------------------|:-------------|
| Bank Transfer    | android           | 2,197,597.44 |
| COD              | android           | 2,125,389.45 |
| Credit Card      | android           | 2,107,849.65 |
| PayLater         | android           | 2,062,562.42 |
| QRIS             | android           | 2,127,539.17 |
| ShopeePay        | android           | 2,086,370.75 |
| Virtual Account  | android           | 2,050,887.57 |
| Bank Transfer    | ios               | 2,014,224.52 |

**Brief insight**

The highest AOV appears in iOS Virtual Account transactions, but the order count is relatively small. Android Bank Transfer combines high AOV with much larger volume, making it more commercially meaningful than a small-volume high-AOV segment.

### 8. Calculate Cancellation Rate by Payment Method and Source Platform

This query calculates cancellation rate by `payment_method` and `source_platform`. It helps identify where checkout friction, payment failure, or user behavior issues may be concentrated.

```sql
SELECT 
    payment_method,
    source_platform,
    COUNT(*) FILTER (WHERE order_status = 'cancelled') AS cancel_orders,
    COUNT(*) AS total_orders,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE order_status = 'cancelled')
        / COUNT(*),
        2
    ) AS cancel_rate
FROM orders
GROUP BY 1, 2
ORDER BY 5 DESC, 2, 1;
```

**Result preview**

| payment_method   | source_platform   | cancel_orders   | total_orders   |   cancel_rate |
|:-----------------|:------------------|:----------------|:---------------|--------------:|
| Virtual Account  | web               | 139             | 1,483          |          9.37 |
| COD              | android           | 2,993           | 33,171         |          9.02 |
| COD              | web               | 428             | 4,808          |          8.9  |
| QRIS             | ios               | 212             | 2,455          |          8.64 |
| COD              | ios               | 801             | 9,422          |          8.5  |
| QRIS             | android           | 722             | 8,749          |          8.25 |

**Brief insight**

The highest cancellation rate appears in web Virtual Account transactions at 9.37%, while COD also appears repeatedly among high-cancellation segments. This points to payment flow and checkout behavior as useful next areas for root-cause analysis.

### 9. Calculate Monthly Active Buyers and Monthly Active Sellers

This query calculates monthly active buyers and active sellers using completed orders. It gives a basic view of demand-side and supply-side marketplace activity over time.

```sql
SELECT
    DATE_TRUNC('month', order_datetime) AS order_month,
    COUNT(DISTINCT customer_id) AS active_buyers,
    COUNT(DISTINCT seller_id) AS active_sellers
FROM orders
WHERE order_status = 'completed'
GROUP BY 1
ORDER BY 1;
```

**Result preview**

| order_month   | active_buyers   | active_sellers   |
|:--------------|:----------------|:-----------------|
| 2025-12-01    | 7,613           | 3,494            |
| 2026-01-01    | 5,784           | 3,126            |
| 2026-02-01    | 8,415           | 3,622            |
| 2026-03-01    | 11,245          | 3,819            |
| 2026-04-01    | 5,923           | 3,183            |
| 2026-05-01    | 7,213           | 3,422            |

**Brief insight**

Monthly active buyers and sellers both peak in March 2026 in the preview period. The simultaneous increase in buyer and seller activity suggests stronger marketplace liquidity during that month, which may be linked to campaign or seasonal effects.

### 10. Compare Official Store vs Non-Official Store Performance

This query compares official and not-official stores by active stores, completed orders, goods sold, GMV, and marketplace commission. It helps evaluate whether official-store status is associated with stronger commercial performance.

```sql
SELECT
    DATE_TRUNC('month', o.order_datetime) AS order_month,
    CASE 
        WHEN s.is_official_store = TRUE THEN 'official'
        ELSE 'not official'
    END AS official_store,
    COUNT(DISTINCT oi.seller_id) AS store_active,
    COUNT(DISTINCT o.order_id) AS order_complete,
    SUM(oi.quantity) AS goods_sold,
    SUM(oi.item_gross_amount) AS gmv,
    SUM(oi.platform_commission_amount) AS marketplace_commission
FROM order_items oi
JOIN sellers s
    ON oi.seller_id = s.seller_id
JOIN orders o
    ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
GROUP BY 1, 2
ORDER BY 1, 2;
```

**Result preview: latest monthly rows**

| order_month   | official_store   | store_active   | order_complete   | goods_sold   | gmv            | marketplace_commission   |
|:--------------|:-----------------|:---------------|:-----------------|:-------------|:---------------|:-------------------------|
| 2026-03-01    | not official     | 3,319          | 10,812           | 26,712       | 22,848,588,000 | 1,055,498,611            |
| 2026-03-01    | official         | 500            | 1,624            | 3,953        | 3,069,247,000  | 192,557,135              |
| 2026-04-01    | not official     | 2,751          | 5,376            | 13,350       | 11,916,460,000 | 573,541,891              |
| 2026-04-01    | official         | 432            | 851              | 2,058        | 1,465,678,000  | 97,657,914               |
| 2026-05-01    | not official     | 2,963          | 6,685            | 16,150       | 14,251,455,000 | 658,660,890              |
| 2026-05-01    | official         | 459            | 1,030            | 2,432        | 2,337,507,000  | 145,452,165              |

**Result preview: overall store-type comparison**

| official_store   | store_active   | order_complete   | goods_sold   | gmv             | marketplace_commission   |
|:-----------------|:---------------|:-----------------|:-------------|:----------------|:-------------------------|
| not official     | 3,468          | 177,877          | 437,943      | 378,112,383,000 | 17,707,903,156           |
| official         | 532            | 27,293           | 66,705       | 52,680,588,000  | 3,444,363,216            |

**Brief insight**

Non-official stores contribute most of the GMV and order volume because they have far more active sellers. However, official stores still represent a meaningful GMV contribution despite a smaller seller base. The overall AOV is higher for non-official stores in this dataset, so the data does not support a blanket assumption that official stores always drive higher order value.
