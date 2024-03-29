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
- Sales Performance Analysis: Enabled the identification of sales trends, informing revenue management strategies.

### Top Customers, Products, and Markets

**SQL Highlights:**

- Database Views: Created for simplified and efficient retrieval of aggregated sales data.

Create the view `sales_preinv_discount` and store all the data in like a virtual table.

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
    		pre.fiscal_year=s.fiscal_year;
```

Create a view for post invoice deductions: `sales_postinv_discount`.

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

Create the view `net_sales`.

```sql
	CREATE VIEW `net_sales` AS
	SELECT 
            *, 
    	    net_invoice_sales*(1-post_invoice_discount_pct) as net_sales
	FROM gdb041.sales_postinv_discount;
```

- Get top 5 market by net sales in fiscal year 2021.

```sql
	SELECT 
    	    market, 
            round(sum(net_sales)/1000000,2) as net_sales_mln
	FROM gdb041.net_sales
	where fiscal_year=2021
	group by market
	order by net_sales_mln desc
	limit 5;
```
|market|net_sales_mln|
|:----|----:|
|India|210.67|
|USA|132.05|
|South Korea|64.01|
|Canada|45.89|
|United Kingdom|44.73|

- Window Functions: Utilized for ranking and comparative analysis to identify top-performing products and customer segments.

Find out customer-wise net sales percentage contribution.

```sql
	with cte1 as (
		select 
                    customer, 
                    round(sum(net_sales)/1000000,2) as net_sales_mln
        	from net_sales s
        	join dim_customer c
                    on s.customer_code=c.customer_code
        	where s.fiscal_year=2021
        	group by customer)
	select 
            *,
            net_sales_mln*100/sum(net_sales_mln) over() as pct_net_sales
	from cte1
	order by net_sales_mln desc
	LIMIT 5;
```
|customer|net_sales_mln|pct_net_sales|
|:----|----:|----:|
|Amazon|109.03|13.233402|
|Atliq Exclusive|79.92|9.700206|
|Atliq e Store|70.31|8.533803|
|Sage|27.07|3.285593|
|Flipkart|25.25|3.064692|

Find customer-wise net sales distribution per region for FY 2021.

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
	LIMIT 5;
```

|customer|region|net_sales_mln|pct_share_region|
|:----|:----|----:|----:|
|Amazon|APAC|57.41|12.988688|
|Atliq Exclusive|APAC|51.58|11.669683|
|Atliq e Store|APAC|36.97|8.364253|
|Leader|APAC|24.52|5.547511|
|Sage|APAC|22.85|5.169683|

Find out top 3 products from each division by total quantity sold in a given year.

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
	select * from cte2 where drnk <= 3;
```

|division|product|total_qty|drnk|
|:----|:----|----:|:----:|
|N & S|AQ Pen Drive DRC|2034569|1|
|N & S|AQ Digit SSD|1240149|2|
|N & S|AQ Clx1|1238683|3|
|P & A|AQ Gamers Ms|2477098|1|
|P & A|AQ Maxima Ms|2461991|2|
|P & A|AQ Master wireless x1 Ms|2448784|3|
|PC|AQ Digit|135092|1|
|PC|AQ Gen Y|135031|2|
|PC|AQ Elite|134431|3|

**Business Implications:**

- Market Segmentation Analysis: Analyzed sales by customer demographics and product preferences, guiding targeted marketing campaigns.
- Product Popularity Insights: Informed inventory and product development strategies by identifying best-selling products and emerging trends.

### Supply Chain Analytics

**SQL Highlights:**

- Database Triggers: Automated the synchronization of actual sales and forecast data, ensuring data integrity and timeliness.

Create a Helper Table.

```sql
	drop table if exists fact_act_est;

	create table fact_act_est
	(
        	select 
                    s.date as date,
                    s.fiscal_year as fiscal_year,
                    s.product_code as product_code,
                    s.customer_code as customer_code,
                    s.sold_quantity as sold_quantity,
                    f.forecast_quantity as forecast_quantity
        	from 
                    fact_sales_monthly s
        	left join fact_forecast_monthly f 
        	using (date, customer_code, product_code)
	)
	union
	(
        	select 
                    f.date as date,
                    f.fiscal_year as fiscal_year,
                    f.product_code as product_code,
                    f.customer_code as customer_code,
                    s.sold_quantity as sold_quantity,
                    f.forecast_quantity as forecast_quantity
        	from 
		    fact_forecast_monthly  f
        	left join fact_sales_monthly s 
        	using (date, customer_code, product_code)
	);

	update fact_act_est
	set sold_quantity = 0
	where sold_quantity is null;

	update fact_act_est
	set forecast_quantity = 0
	where forecast_quantity is null;
```

Create the trigger to automatically insert record in `fact_act_est` table whenever insertion happens in `fact_sales_monthly`.

```sql
	CREATE DEFINER=CURRENT_USER TRIGGER `fact_sales_monthly_AFTER_INSERT` AFTER INSERT ON `fact_sales_monthly` FOR EACH ROW 
	BEGIN
        	insert into fact_act_est 
                        (date, product_code, customer_code, sold_quantity)
    		values (
                	NEW.date, 
        		NEW.product_code, 
        		NEW.customer_code, 
        		NEW.sold_quantity
    		 )
    		on duplicate key update
                         sold_quantity = values(sold_quantity);
	END
```

Create the trigger to automatically insert record in `fact_act_est` table whenever insertion happens in `fact_forecast_monthly`.

```sql
	CREATE DEFINER=CURRENT_USER TRIGGER `fact_forecast_monthly_AFTER_INSERT` AFTER INSERT ON `fact_forecast_monthly` FOR EACH ROW 
	BEGIN
        	insert into fact_act_est 
                        (date, product_code, customer_code, forecast_quantity)
    		values (
                	NEW.date, 
        		NEW.product_code, 
        		NEW.customer_code, 
        		NEW.forecast_quantity
    		 )
    		on duplicate key update
                         forecast_quantity = values(forecast_quantity);
	END
```

- Forecast Accuracy Assessment: Employed temporary tables and CTEs for evaluating the precision of sales forecasts against actual sales data.

```sql
	with forecast_err_table as (
             select
                  s.customer_code as customer_code,
                  c.customer as customer_name,
                  c.market as market,
                  sum(s.sold_quantity) as total_sold_qty,
                  sum(s.forecast_quantity) as total_forecast_qty,
                  sum(s.forecast_quantity-s.sold_quantity) as net_error,
                  round(sum(s.forecast_quantity-s.sold_quantity)*100/sum(s.forecast_quantity),1) as net_error_pct,
                  sum(abs(s.forecast_quantity-s.sold_quantity)) as abs_error,
                  round(sum(abs(s.forecast_quantity-sold_quantity))*100/sum(s.forecast_quantity),2) as abs_error_pct
             from fact_act_est s
             join dim_customer c
             on s.customer_code = c.customer_code
             where s.fiscal_year=2021
             group by customer_code
	)
	select 
            *,
            if (abs_error_pct > 100, 0, 100.0 - abs_error_pct) as forecast_accuracy
	from forecast_err_table
        order by forecast_accuracy desc
	LIMIT 10;
```

|customer_code|customer_name|market|total_sold_qty|total_forecast_qty|net_error|net_error_pct|abs_error|abs_error_pct|forecast_accuracy|
|:----|:----|:----|----:|----:|----:|----:|----:|----:|----:|
|90025209|Electricalsbea Stores|Columbia|13178|15428|2065|13.4|8051|52.18|47.82|
|90013120|Coolblue|Italy|109547|133532|23944|17.9|70378|52.70|47.30|
|70010048|Atliq e Store|Bangladesh|119439|142010|22547|15.9|75645|53.27|46.73|
|90023027|Costco|Canada|236189|279962|43760|15.6|149274|53.32|46.68|
|70027208|Atliq e Store|Brazil|33713|47321|13342|28.2|25398|53.67','46.33|
|90023026|Relief|Canada|228988|273492|44495|16.3|146921|53.72|46.28|
|90017051|Forward Stores|Portugal|86823|118067|31151|26.4|63449|53.74|46.26|
|90017058|Mbit|Portugal|86860|110195|23242|21.1|59348|53.86|46.14|
|90023028|walmart|Canada|239081|283323|44233|15.6|153039|54.02|45.98|
|90023024|Sage|Canada|246397|287233|40835|14.2|155585|54.17|45.83|

**Business Implications:**

- Supply Chain Optimization: By creating a unified view of forecasted and actual sales data, informed more accurate inventory planning, reducing stock-outs and overstock situations.
- Operational Efficiency: Automated data entry and updates streamlined operations, reducing manual workload and potential for error.

### Business Impact

- Finance Analytics: Facilitated strategic financial planning and performance monitoring.
- Top Customers, Products, and Markets: Supported targeted marketing and product development strategies based on customer behavior and product performance insights.
- Supply Chain Analytics: Improved supply chain resilience and responsiveness through enhanced forecast accuracy and operational efficiency.

## Technologies Used

- MySQL: For executing SQL queries and managing database objects.
- Advanced SQL Techniques: Including triggers, window functions, and user-defined functions, showcasing an adept ability to tackle complex analytical tasks efficiently.

## Conclusion

This project underscores the vital role of SQL in deriving actionable business insights. The detailed analytics and optimizations implemented not only demonstrate technical SQL proficiency but also strategic thinking in applying these insights towards business improvement.
