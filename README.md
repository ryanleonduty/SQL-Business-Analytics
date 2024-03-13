# Business Analytics & Strategy

## Overview
This portfolio emphasizes the strategic application of SQL in business analytics across Finance Analytics, Customer and Product Analysis, and Supply Chain Optimization. It details the SQL queries used, the steps taken for data analysis, and the business implications of these actions.

## Project Sections

### Finance Analytics

**SQL Highlights:**

- User-Defined Functions: Created to dynamically calculate fiscal years, enhancing the flexibility and accuracy of financial reporting.

```sql
	CREATE FUNCTION `get_fiscal_year`(calendar_date DATE) 
	RETURNS int
    	DETERMINISTIC
	BEGIN
        	DECLARE fiscal_year INT;
        	SET fiscal_year = YEAR(DATE_ADD(calendar_date, INTERVAL 4 MONTH));
        	RETURN fiscal_year;
	END
```

- Complex Reporting: Generated detailed sales transaction reports using advanced JOIN operations and conditional aggregation to understand sales trends.

```sql
	SELECT 
    	    s.date, 
            s.product_code, 
            p.product, 
            p.variant, 
            s.sold_quantity, 
            g.gross_price,
            ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total
	FROM fact_sales_monthly s
	JOIN dim_product p
            ON s.product_code=p.product_code
	JOIN fact_gross_price g
            ON g.fiscal_year=get_fiscal_year(s.date)
    	AND g.product_code=s.product_code
	WHERE 
    	    customer_code=90002002 AND 
            get_fiscal_year(s.date)=2021     
	LIMIT 1000000;
```

- Stored Procedure: This stored procedure facilitates generating detailed monthly gross sales reports for specified customers.

```sql
	CREATE PROCEDURE `get_monthly_gross_sales_for_customer`(
        	in_customer_codes TEXT
	)
	BEGIN
        	SELECT 
                    s.date, 
                    SUM(ROUND(s.sold_quantity*g.gross_price,2)) as monthly_sales
        	FROM fact_sales_monthly s
        	JOIN fact_gross_price g
               	    ON g.fiscal_year=get_fiscal_year(s.date)
                    AND g.product_code=s.product_code
        	WHERE 
                    FIND_IN_SET(s.customer_code, in_customer_codes) > 0
        	GROUP BY s.date
        	ORDER BY s.date DESC;
	END
```

**Business Implications:**

- Customer Code Retrieval: Identified specific customer segments for targeted financial analysis, supporting focused strategy development.
- Sales Performance Analysis: Enabled the identification of sales trends and fiscal health, informing revenue management strategies.

### Top Customers, Products, and Markets

**SQL Highlights:**

- Database Views: Created for simplified and efficient retrieval of aggregated sales data.

Create the view `sales_preinv_discount` and store all the data in like a virtual table:

```sql
	CREATE  VIEW `sales_preinv_discount` AS
	SELECT 
    	    s.date, 
            s.fiscal_year,
            s.customer_code,
            c.market,
            s.product_code, 
            p.product, 
            p.variant, 
            s.sold_quantity, 
            g.gross_price as gross_price_per_item,
            ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total,
            pre.pre_invoice_discount_pct
	FROM fact_sales_monthly s
	JOIN dim_customer c 
		ON s.customer_code = c.customer_code
	JOIN dim_product p
        	ON s.product_code=p.product_code
	JOIN fact_gross_price g
    		ON g.fiscal_year=s.fiscal_year
    		AND g.product_code=s.product_code
	JOIN fact_pre_invoice_deductions as pre
        	ON pre.customer_code = s.customer_code AND
    		pre.fiscal_year=s.fiscal_year
```

Create a view for post invoice deductions: `sales_postinv_discount`:

```sql
	CREATE VIEW `sales_postinv_discount` AS
	SELECT 
    	    s.date, s.fiscal_year,
            s.customer_code, s.market,
            s.product_code, s.product, s.variant,
            s.sold_quantity, s.gross_price_total,
            s.pre_invoice_discount_pct,
            (s.gross_price_total-s.pre_invoice_discount_pct*s.gross_price_total) as net_invoice_sales,
            (po.discounts_pct+po.other_deductions_pct) as post_invoice_discount_pct
	FROM sales_preinv_discount s
	JOIN fact_post_invoice_deductions po
		ON po.customer_code = s.customer_code AND
   		po.product_code = s.product_code AND
   		po.date = s.date;
```

Create the view `net_sales`:

```sql
	CREATE VIEW `net_sales` AS
	SELECT 
            *, 
    	    net_invoice_sales*(1-post_invoice_discount_pct) as net_sales
	FROM gdb041.sales_postinv_discount;
```

- Get top 5 market by net sales in fiscal year 2021:

```sql
	SELECT 
    	    market, 
            round(sum(net_sales)/1000000,2) as net_sales_mln
	FROM gdb041.net_sales
	where fiscal_year=2021
	group by market
	order by net_sales_mln desc
	limit 5
```

- Window Functions: Utilized for ranking and comparative analysis to identify top-performing products and customer segments.

Find out customer-wise net sales percentage contribution:

```sql
	with cte1 as (
		select 
        	    c.customer,
                    c.region,
                    round(sum(net_sales)/1000000,2) as net_sales_mln
                from gdb041.net_sales n
                join dim_customer c
                    on n.customer_code=c.customer_code
		where fiscal_year=2021
		group by c.customer, c.region)
	select
             *,
             net_sales_mln*100/sum(net_sales_mln) over (partition by region) as pct_share_region
	from cte1
	order by region, pct_share_region desc
```

Find out top 3 products from each division by total quantity sold in a given year:

```sql
	with cte1 as 
		(select
                     p.division,
                     p.product,
                     sum(sold_quantity) as total_qty
                from fact_sales_monthly s
                join dim_product p
                      on p.product_code=s.product_code
                where fiscal_year=2021
                group by p.product),
           cte2 as 
	        (select 
                     *,
                     dense_rank() over (partition by division order by total_qty desc) as drnk
                from cte1)
	select * from cte2 where drnk<=3
```

**Business Implications:**

- Market Segmentation Analysis: Analyzed sales by customer demographics and product preferences, guiding targeted marketing campaigns.
- Product Popularity Insights: Informed inventory and product development strategies by identifying best-selling products and emerging trends.

### Supply Chain Analytics

**SQL Highlights:**

- Database Triggers: Automated the synchronization of actual sales and forecast data, ensuring data integrity and timeliness.

```sql

```

- Forecast Accuracy Assessment: Employed temporary tables and CTEs for evaluating the precision of sales forecasts against actual sales data.

```sql

```

**Business Implications:**

- Supply Chain Optimization: By creating a unified view of forecasted and actual sales data, informed more accurate inventory planning, reducing stock-outs and overstock situations.
- Operational Efficiency: Automated data entry and updates streamlined operations, reducing manual workload and potential for error.

### Business Impact

- Finance Analytics: Facilitated strategic financial planning and performance monitoring.
- Top Customers, Products, and Markets: Supported targeted marketing and product development strategies based on customer behavior and product performance insights.
- Supply Chain Analytics: Improved supply chain resilience and responsiveness through enhanced forecast accuracy and operational efficiency.

## Technologies and Techniques Used

- MySQL: For executing SQL queries and managing database objects.
- Advanced SQL Techniques: Including triggers, window functions, and user-defined functions, showcasing an adept ability to tackle complex analytical tasks efficiently.

## Conclusion

This project underscores the vital role of SQL in deriving actionable business insights. The detailed analytics and optimizations implemented not only demonstrate technical SQL proficiency but also strategic thinking in applying these insights towards business improvement.
