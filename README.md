# Marketplace-Analytics-Case-Study-using-PostgreSQL
A fictional marketplace dataset used to answer business questions around customer base, seller performance, GMV, order status, product demand, payment behavior, and platform health.

# Marketplace Analytics SQL Case Study

## Project Overview

This project is a PostgreSQL-based marketplace analytics case study using a fictional Shopee-like e-commerce dataset. The goal is to demonstrate how SQL can be used to answer practical business questions related to marketplace growth, customer behavior, seller performance, product demand, payment behavior, and operational efficiency.

Rather than treating SQL as a technical exercise only, this project frames each query around a business problem. Each analysis starts from a business question, defines the relevant metric, applies SQL logic, and produces an output that can support decision-making.

## Dataset Context

The dataset represents a fictional online marketplace with multiple related tables, including customers, sellers, products, orders, order items, payments, shipments, reviews, refunds, campaigns, vouchers, and product events.

The dataset is designed to mimic common marketplace analytics scenarios, but it does not contain real Shopee data or proprietary company information.

## Business Objective

As a marketplace analyst, the objective is to understand:

* How large the marketplace ecosystem is
* How order status is distributed across the platform
* How GMV, discounts, shipping fee, and net paid amount relate to each other
* Which customer locations and product categories dominate the marketplace
* Which products and sellers contribute most to marketplace performance
* How payment methods differ in average order value
* Which user platforms show higher cancellation risk

## Skills Demonstrated

This project demonstrates:

* SQL aggregation using `COUNT`, `SUM`, `AVG`, and `COUNT DISTINCT`
* Filtering business-valid transactions using order status
* Joining fact and dimension tables
* Understanding table grain, especially order-level vs item-level metrics
* Calculating marketplace KPIs such as GMV, AOV, discount amount, net paid amount, cancellation rate, and units sold
* Translating business questions into SQL logic
* Avoiding common analytics mistakes such as duplicated order values after joining order-level and item-level tables

## Tools Used

* PostgreSQL
* SQL
* Marketplace analytics concepts
* Relational data modeling
* GitHub for project documentation

## Key Metric Definitions

| Metric            | Definition                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------------- |
| GMV               | Total merchandise value before discounts, calculated from `gross_amount` or item-level gross amount |
| Net Paid Amount   | Final amount paid by the buyer after product discount, voucher discount, and shipping fee           |
| AOV               | Average order value, calculated as GMV divided by number of completed orders                        |
| Units Sold        | Total product quantity sold, calculated using `SUM(quantity)`                                       |
| Cancellation Rate | Cancelled orders divided by total orders                                                            |
| Active Products   | Products with active catalog status                                                                 |
| Completed Orders  | Orders where `order_status = 'completed'`                                                           |

## Important Analytical Assumptions

For revenue and sales analysis, only completed orders are used unless the business question explicitly requires all order statuses. This avoids mixing successful sales with cancelled, pending, returned, or incomplete transactions.

For product-level and seller-level analysis, item-level data is used carefully to avoid duplicating order-level values. For example, units sold should be calculated using `SUM(quantity)`, not `COUNT(order_id)` or `COUNT(order_item_id)`.

For order-level metrics such as GMV and AOV, calculations are performed at the order level unless item-level granularity is required.
