# 8 Week SQL Challenges by Danny | Case Study #5 - Data Mart
### ERD for Database
![image](https://github.com/user-attachments/assets/881845f8-fb0c-4744-a72e-b0cb4451c582)

## Result of Questions
### 1. Data Cleansing Steps

 ```sql
-- Convert the week_date to a DATE format
ALTER TABLE data_mart.weekly_sales
ADD COLUMN clean_week_date date;

UPDATE data_mart.weekly_sales
SET clean_week_date = to_date(week_date,'DD-MM-YY');

-- Add a week_number, month_number, calendar_year
ALTER TABLE data_mart.weekly_sales
ADD COLUMN week_number integer,
ADD COLUMN month_number integer,
ADD COLUMN calendar_year integer;

UPDATE data_mart.weekly_sales
SET week_number = extract(week from clean_week_date) 
,month_number = extract(month from clean_week_date)
,calendar_year = extract (year from clean_week_date);

-- Add a new column called age_band after the original segment column
ALTER TABLE data_mart.weekly_sales
ADD COLUMN age_band varchar;

UPDATE data_mart.weekly_sales
SET age_band = case when right(segment,1) = '1' then 'Young Adults'
					when right(segment,1) = '2' then 'Middle Aged'
					when right(segment,1) = '3' or right(segment,1) = '4' then 'Retirees'
					else 'unknown'
					end;

-- Add a new demographic
ALTER TABLE data_mart.weekly_sales
ADD COLUMN demographic varchar;

UPDATE data_mart.weekly_sales
SET demographic = case when left(segment,1) = 'C' then 'Couples'
						when left(segment,1) = 'F' then 'Family'
						else 'unknown'
						end;

-- Add new avg_transaction
ALTER TABLE data_mart.weekly_sales
ADD COLUMN avg_transaction float;

UPDATE data_mart.weekly_sales
SET avg_transaction = round(sales / transactions,2);
```
### 2. Data Exploration
#### 2.1 What day of the week is used for each week_date value?
```sql
SELECT distinct clean_week_date
,to_char(clean_week_date,'Day') as day_of_week
FROM data_mart.weekly_sales
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/edf0c724-3dcb-4680-afac-1e6bc759dcac)

#### 2.2 What range of week numbers are missing from the dataset?

```sql
WITH all_weeks as
(
	SELECT generate_series(1,52) as week_number
)
SELECT aw.week_number
FROM all_weeks aw
LEFT JOIN data_mart.weekly_sales ws on ws.week_number = aw.week_number
WHERE ws.week_number is null
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/597fa5a7-fd32-46f7-8aa1-45385b2594e3)


#### 2.3 How many total transactions were there for each year in the dataset?

```sql
SELECT calendar_year
,sum(transactions) as total_transactions
FROM data_mart.weekly_sales
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/fa1934c3-66cf-4569-9cd5-fefe6d575cdd)

#### 2.4 What is the total sales for each region for each month?

```sql
SELECT region
,to_char(clean_week_date,'MM-YYYY') as month
,sum(sales)
FROM data_mart.weekly_sales
GROUP BY 1,2
ORDER BY 1,2;
```

_Result:_

![image](https://github.com/user-attachments/assets/85bd23a0-a2ef-4fe9-a62e-414e3d80e956)

#### 2.5 What is the total count of transactions for each platform

```sql
SELECT platform
,sum(transactions) as transactions_sum
FROM data_mart.weekly_sales
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/8dc9a441-8af7-4a82-b0f8-5f935b18d6ef)


#### 2.6 What is the percentage of sales for Retail vs Shopify for each month?

```sql
WITH platform_metrics as
(
	SELECT to_char(clean_week_date,'MM-YYYY') as month
	,sum(case when lower(platform) = 'retail' then sales else 0 end) as retail_sales
	,sum(case when lower(platform) = 'shopify' then sales else 0 end) as shopify_sales
	FROM data_mart.weekly_sales
	GROUP BY 1
)
SELECT month
,round(retail_sales *100 / nullif(retail_sales + shopify_sales,0),2) || '%' as percent_retail_sales
,round(shopify_sales *100/ nullif(retail_sales + shopify_sales,0),2) || '%' as percent_shopify_sales -- nullif to avoid dividing 0
FROM platform_metrics
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/dcdcb07b-ba1d-438b-9d82-a86367623fdf)

#### 2.7 What is the percentage of sales by demographic for each year in the dataset?

```sql
WITH demographic_metric as
(
	SELECT calendar_year
	,sum(case when lower(demographic) = 'family' then sales else 0 end) as family_sales
	,sum(case when lower(demographic) = 'couples' then sales else 0 end) as couple_sales
	,sum(case when lower(demographic) = 'unknown' then sales else 0 end) as unknown_sales
	FROM data_mart.weekly_sales
	GROUP BY 1
)
SELECT calendar_year
,round(family_sales *100 / (family_sales + couple_sales + unknown_sales),2) || '%' as percent_family_sales
,round(couple_sales *100/ (family_sales + couple_sales + unknown_sales),2) || '%' as percent_couples_sales
,round(unknown_sales *100/ (family_sales + couple_sales + unknown_sales),2) || '%' as percent_unknown_sales
FROM demographic_metric
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/85d9c6fd-591f-4236-8311-b943927b582e)


#### 2.8 Which age_band and demographic values contribute the most to Retail sales?

```sql
SELECT age_band
,demographic
,sum(sales) as retail_sales
FROM data_mart.weekly_sales
WHERE lower(platform) = 'retail'
GROUP BY 1,2
ORDER BY 3 DESC
LIMIT 2;
```

_Result:_

![image](https://github.com/user-attachments/assets/d5e606b4-946b-4323-a169-26a87413ced3)

#### 2.9 Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?

```sql
SELECT calendar_year
,platform
,round(sum(sales)/sum(transactions),2) as avg_transaction
FROM data_mart.weekly_sales
WHERE lower(platform) in ('retail', 'shopify')
GROUP BY 1,2
ORDER BY 1,2;
```

_Result:_

![image](https://github.com/user-attachments/assets/3e3ed5c3-3a6e-4450-8583-fa71c0f80c1a)

### 3. Before & After Analysis
_Taking the week_date value of 2020-06-15 as the baseline week_
#### 3.1 What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?

```sql
WITH sales_period as
(
	SELECT
	sum(case when clean_week_date between '2020-06-15'::date and '2020-06-15'::date + interval '4 weeks' then sales else 0 end) as sales_after
	,sum(case when clean_week_date between '2020-06-15'::date - interval '4 weeks' and '2020-06-14'::date then sales else 0 end) as sales_before
	FROM data_mart.weekly_sales
)
SELECT sales_before
,sales_after
,(sales_after + sales_before) as "total sales for the 4 weeks before and after 2020-06-15"
,(sales_after - sales_before) as difference_before_after_sales
FROM sales_period;
```

_Result:_

![image](https://github.com/user-attachments/assets/1c147010-7185-449e-88a0-d7d998ca1109)

#### 3.2 What about the entire 12 weeks before and after?

```sql
WITH sales_period as
(
	SELECT sum(case when clean_week_date between '2020-06-15'::date and '2020-06-15'::date + interval '12 weeks' then sales else 0 end) as sales_after
	,sum(case when clean_week_date between '2020-06-15'::date - interval '12 weeks' and '2020-06-14'::date then sales else 0 end) as sales_before
	FROM data_mart.weekly_sales
)
SELECT sales_before
,sales_after
,(sales_after + sales_before) as "total sales for 12 weeks before and after 2020-06-15"
,(sales_after - sales_before) as difference_before_after_sales
FROM sales_period;
```

_Result:_

![image](https://github.com/user-attachments/assets/1a6815c5-5d7b-46dc-8c96-b5ad23f32161)


#### 3.3 How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?

```sql
WITH sales_comparison as
(
	SELECT calendar_year
	,sum(case when clean_week_date between '2020-06-15'::date and '2020=06-15'::date + interval '12 weeks'
				or clean_week_date between '2019-06-15'::date and '2019-06-15'::date + interval '12 weeks'
				or clean_week_date between '2018-06-15'::date and '2018-06-15'::date + interval '12 weeks'
				then sales else 0 end) as sales_after
	,sum(case when clean_week_date between '2020-06-15'::date - interval '12 weeks' and '2020-06-14'::date
				or clean_week_date between '2019-06-15'::date - interval '12 weeks' and '2019-06-14'::date
				or clean_week_date between '2018-06-15'::date - interval '12 weeks' and '2018-06-14'::date
				then sales else 0 end) as sales_before
	FROM data_mart.weekly_sales
	GROUP BY 1
)
SELECT calendar_year 
,sales_before
,sales_after
,(sales_before + sales_after) as total_sales
,(sales_after - sales_before) as difference_before_after_sales
FROM sales_comparison
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/43749214-b3ae-4e2e-a20c-33bce50c71c9)

#### Bonus question

```sql
WITH sales_by_category as
(
	SELECT region
	,platform
	,age_band
	,demographic
	,customer_type
	,sum(case when clean_week_date between '2020-06-15'::date and '2020-06-15'::date + interval '12 weeks' then sales else 0 end) as sales_after
	,sum(case when clean_week_date between '2020-06-15'::date - interval '12 weeks' and '2020-06-14'::date then sales else 0 end) as sales_before
	FROM data_mart.weekly_sales
	GROUP BY 1,2,3,4,5
)
SELECT region
,platform
,age_band
,demographic
,customer_type
,sales_before
,sales_after
,(sales_after - sales_before) as sales_change
,(sales_after/nullif(sales_before,0) - 1) *100 || '%' as percent_change
FROM sales_by_category
ORDER BY sales_change DESC;
```

_Result:_

![image](https://github.com/user-attachments/assets/837966e3-6d17-4488-b838-98ff820ed475)

Thank you for stopping by, and I'm pleased to connect with you, my new friend!

**Please do not forget to FOLLOW and star ⭐ the repository if you find it valuable.**

Wish you a day filled with happiness and energy!

Warm regards,

Hien Moon

