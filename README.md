# Case Study #8 Fresh Segments

*Note: All information and data related to the case study were obtained from [here](https://8weeksqlchallenge.com/case-study-8/).*
![Screenshot 2023-08-13 at 8 39 06 pm](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/5871743d-78e8-4147-b95c-dba999a62a78)

## Business Task
"Fresh Segments" is a digital marketing agency founded by Danny. The agency specializes in analyzing online ad click behavior trends tailored to each client's unique customer base. Clients provide their customer lists, and Fresh Segments aggregates interest metrics, producing a consolidated dataset for in-depth analysis. This dataset showcases the composition and rankings of various interests, indicating the percentage of a client's customer list that engaged with online content related to each interest monthly.
Danny seeks our expertise to scrutinize the aggregated metrics for a sample client. Our goal is to offer high-level insights about the client's customer list and their respective interests.

## General Insights
- *Interest Analysis and Customer Segmentation:* The queries centered around understanding the average composition of interests of Fresh Segments' clients. Insights covered:
  - *Top Interests:* Identification of the top 10 interests based on average composition for each month gives insights into prevailing trends and preferences over time.
  - *Frequency Analysis:* Observing which interest appears most frequently among the top 10 across different months gives a perspective on enduring interests or dominant customer segments.
  - *Monthly Analysis:* Computing the average of the average composition for the top 10 interests each month provides a granular view of how interests are spread.
  - *Rolling Analysis:* A rolling 3-month average of the max average composition, including previous top-ranking interests, offers insights into both immediate and slightly long-term trends.

## Key SQL Syntax and Functions
- Joins (`FULL OUTER JOIN`, `INNER JOIN`, LEFT SEMI JOIN with `WHERE EXISTS`)
- Aggregation Functions (`AVG`, `COUNT`, `MAX`, `MIN`, `SUM`)
- Window Functions (`RANK`, `LAG`)
- Common Table Expressions (CTE)
- Conditional Logic (`CASE WHEN`)
- Update Functions (`UPDATE`)
- Table Modification (`ALTER TABLE`)

## Questions and Solutions
### Part A: Data Exploration and Cleansing
> 1. Update the fresh_segments.interest_metrics table by modifying the month_year column to be a date data type with the start of the month
```sql
UPDATE fresh_segments.interest_metrics
SET month_year = TO_DATE(month_year, 'MM-YYYY');

ALTER TABLE fresh_segments.interest_metrics
ALTER month_year TYPE DATE USING month_year::DATE;
```

> 2. What is count of records in the fresh_segments.interest_metrics for each month_year value sorted in chronological order (earliest to latest) with the null values appearing first?
```sql
SELECT 
    month_year
  , COUNT(*) AS record_count
FROM fresh_segments.interest_metrics
GROUP BY month_year
ORDER BY month_year NULLS FIRST;

--another way
SELECT 
    month_year
  , COUNT(*) AS record_count
FROM fresh_segments.interest_metrics
GROUP BY month_year
ORDER BY 
    CASE WHEN month_year IS NULL THEN 0 ELSE 1 END
  , month_year;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/854f1046-3f96-4012-b4c1-a43209f4de79)

> 3. What do you think we should do with these null values in the fresh_segments.interest_metrics?
These are some common approaches that one might consider for the null values.
  - Investigate the cause: Determine why the null vlues are present. Are they due to system error, data entry error or somethuing else?
  - Remove the rows: If the null values represent a small portion of the data and don't contain critical information, you might choose to remove the rows.
  - Impute the values: It can be logically filled in based on other data.
  - Leave them as-is: If the null values are meaningful, you might choose to leave them as-is and handle them appropriately in the analysis.

> 4. How many interest_id values exist in the fresh_segments.interest_metrics table but not in the fresh_segments.interest_map table? What about the other way around?
```sql
SELECT
    COUNT(DISTINCT interest_metrics.interest_id) AS all_interest_metrics
  , COUNT(DISTINCT interest_map.id) AS all_interest_map
  , COUNT(
        CASE WHEN interest_map.id IS NULL THEN interest_metrics.interest_id ELSE NULL END
      ) AS not_in_map
  , COUNT(
        CASE WHEN interest_metrics.interest_id IS NULL THEN interest_map.id ELSE NULL end
      ) AS not_in_metrics
FROM fresh_segments.interest_metrics
FULL OUTER JOIN fresh_segments.interest_map
  ON interest_metrics.interest_id = interest_map.id;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/431231ca-ba30-42c0-9ced-78ea75d2a396)

> 5. Summarise the id values in the fresh_segments.interest_map by its total record count in this table
```sql
WITH id_records AS (
  SELECT
      id 
    , COUNT(*) AS record_count
  FROM fresh_segments.interest_map
  GROUP BY id
)
SELECT
    record_count
  , COUNT(id) AS id_count
FROM id_records
GROUP BY record_count;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/39b6fb2a-f3ec-4f2e-9537-7d2d09685a2f)

> 6. What sort of table join should we perform for our analysis and why? Check your logic by checking the rows where interest_id = 21246 in your joined output and include all columns from fresh_segments.interest_metrics and all columns from fresh_segments.interest_map except from the id column.
```sql
WITH joined_map_metrics AS (
  SELECT
      interest_metrics.*
    , interest_map.interest_name
    , interest_map.interest_summary
    , interest_map.created_at
    , interest_map.last_modified
  FROM fresh_segments.interest_metrics
  INNER JOIN fresh_segments.interest_map
    ON interest_metrics.interest_id = interest_map.id
  WHERE interest_metrics.month_year IS NOT NULL
)
SELECT *
FROM joined_map_metrics
WHERE interest_id = 21246;


--another way
SELECT
    interest_metrics.*
  , interest_map.interest_name
  , interest_map.interest_summary
  , interest_map.created_at
  , interest_map.last_modified
FROM fresh_segments.interest_metrics
INNER JOIN fresh_segments.interest_map
  ON interest_metrics.interest_id = interest_map.id 
WHERE interest_metrics.month_year IS NOT NULL
  AND interest_id = 21246
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/9f50b431-846b-46b1-abc3-adeb1226590f)

> 7. Are there any records in your joined table where the month_year value is before the created_at value from the fresh_segments.interest_map table? Do you think these values are valid and why?
```sql
SELECT
  id,
  COUNT(*)
FROM fresh_segments.interest_map
GROUP BY 1
ORDER BY 2 DESC
LIMIT 5;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/88152f6c-c4ec-47e7-b3f5-4bddfaa32f00)

```sql
WITH cte_join AS (
  SELECT
    interest_metrics.*,
    interest_map.interest_name,
    interest_map.interest_summary,
    interest_map.created_at,
    interest_map.last_modified
  FROM fresh_segments.interest_metrics
  INNER JOIN fresh_segments.interest_map
    ON interest_metrics.interest_id = interest_map.id
  WHERE interest_metrics.month_year IS NOT NULL
)
SELECT
  COUNT(*)
FROM cte_join
WHERE month_year < created_at;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/7a17caac-9b42-40f7-b5ba-3f823d1ed42c)

```sql
WITH cte_join AS (
  SELECT
    interest_metrics.*,
    interest_map.interest_name,
    interest_map.interest_summary,
    interest_map.created_at,
    interest_map.last_modified
  FROM fresh_segments.interest_metrics
  INNER JOIN fresh_segments.interest_map
    ON interest_metrics.interest_id = interest_map.id
  WHERE interest_metrics.month_year IS NOT NULL
)
SELECT
  COUNT(*)
FROM cte_join
WHERE month_year < DATE_TRUNC('month', created_at);
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/1b6c79cf-344e-46ef-8115-0f0e51062b78)

### Part B: Interest Analysis
> 1. Which interests have been present in all month_year dates in our dataset?
```sql
WITH cte_interest_months AS (
  SELECT
      interest_id
    , COUNT(DISTINCT month_year) AS total_months
  FROM fresh_segments.interest_metrics
  WHERE interest_id IS NOT NULL
  GROUP BY interest_id
)
SELECT
    total_months
  , COUNT(DISTINCT interest_id) AS interest_count
FROM cte_interest_months
GROUP BY total_months
ORDER BY total_months DESC;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/df8ba73b-e3f6-4697-9e51-22a9b50731be)

> 2. Using this same total_months measure - calculate the cumulative percentage of all records starting at 14 months - which total_months value passes the 90% cumulative percentage value?
```sql
WITH interest_months AS (
  SELECT
      interest_id
    , COUNT(DISTINCT month_year) AS total_months
  FROM fresh_segments.interest_metrics
  WHERE month_year IS NOT NULL
  GROUP BY interest_id
),
interest_counts AS (
SELECT
    total_months
  , COUNT(DISTINCT interest_id) AS interest_count
FROM interest_months
GROUP BY total_months
)
SELECT
    total_months
  , interest_count
  , ROUND(
          100 * SUM(interest_count) OVER(ORDER BY total_months DESC)::NUMERIC /
              (SUM(interest_count) OVER()), 
          2
         ) AS cumulative_percentage
FROM interest_counts
ORDER BY total_months DESC;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/7d0d8f8c-2588-49e5-92e3-8ca944639d60)

> 3. If we were to remove all interest_id values which are lower than the total_months value we found in the previous question - how many total data points would we be removing?
```sql
WITH removed_interest AS (
  SELECT
    interest_id
  FROM fresh_segments.interest_metrics
  WHERE interest_id IS NOT NULL
  GROUP BY interest_id
  HAVING COUNT(DISTINCT month_year) < 6
)
SELECT
  COUNT(*) AS removed_rows
FROM fresh_segments.interest_metrics
WHERE EXISTS (
  SELECT 1
  FROM removed_interest
  WHERE interest_metrics.interest_id = removed_interest.interest_id
);
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/11e1380e-b1f9-4ff6-b509-4fe6c21ca111)

### Pasrt C: Segment Analysis
> 1. Using our filtered dataset by removing the interests with less than 6 months worth of data, which are the top 10 and bottom 10 interests which have the largest composition values in any month_year? Only use the maximum composition value for each interest but you must keep the corresponding month_year
```sql
WITH ranked_interest AS (
  SELECT
      interest_metrics.month_year
    , interest_map.interest_name
    , interest_metrics.composition
    , RANK() OVER (
              PARTITION BY interest_map.interest_name
              ORDER BY composition DESC
            ) AS interest_rank
  FROM fresh_segments.interest_metrics
  INNER JOIN fresh_segments.interest_map
    ON interest_metrics.interest_id = interest_map.id
  WHERE interest_metrics.month_year IS NOT NULL
),
top_10 AS (
  SELECT
      month_year
    , interest_name
    , composition
  FROM ranked_interest
  WHERE interest_rank = 1
  ORDER BY composition DESC
  LIMIT 10
),
bottom_10 AS (
  SELECT
      month_year
    , interest_name
    , composition
  FROM ranked_interest
  WHERE interest_rank = 1
  ORDER BY composition
  LIMIT 10
),
final_output AS (
  SELECT *
  FROM top_10
  
  UNION
  
  SELECT *
  FROM bottom_10
)
SELECT *
FROM final_output
ORDER BY composition DESC;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/2c3c8c42-cb76-4d1c-b182-bf8e8a9ee934)

> 2. Which 5 interests had the lowest average ranking value?
```sql
SELECT
    interest_map.interest_name
  , ROUND(AVG(interest_metrics.ranking), 1) AS average_ranking
  , SUM(1) AS record_count
FROM fresh_segments.interest_metrics
INNER JOIN fresh_segments.interest_map
  ON interest_metrics.interest_id = interest_map.id
WHERE interest_metrics.month_year IS NOT NULL
GROUP BY interest_map.interest_name
ORDER BY average_ranking
LIMIT 5;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/f1ea39c8-6d14-412d-9c11-df5b63777af6)

> 3. Which 5 interests had the largest standard deviation in their percentile_ranking value?
```sql
SELECT
    interest_metrics.interest_id
  , interest_map.interest_name
  , ROUND(STDDEV(interest_metrics.percentile_ranking)::NUMERIC, 1) AS stddev_pc_ranking
  , MAX(interest_metrics.percentile_ranking) AS max_pc_ranking
  , MIN(interest_metrics.percentile_ranking) AS min_pc_ranking
  , SUM(1) AS record_count
FROM fresh_segments.interest_metrics
INNER JOIN fresh_segments.interest_map
  ON interest_metrics.interest_id = interest_map.id
WHERE interest_metrics.month_year IS NOT NULL
  AND interest_metrics.percentile_ranking IS NOT NULL
GROUP BY
    interest_metrics.interest_id
  , interest_map.interest_name
ORDER BY stddev_pc_ranking DESC NULLS LAST
LIMIT 5;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/ffb22da1-4bfc-454d-a920-a7c34ac6ae17)

> 4. For the 5 interests found in the previous question - what was minimum and maximum percentile_ranking values for each interest and its corresponding year_month value? Can you describe what is happening for these 5 interests?
```sql

```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/82b9444a-f5a2-4038-bebe-2070e118a23e)
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/1fe0674f-a800-4139-9ca8-8e15525ae160)

### Part D: Index Analysis
> 1. What is the top 10 interests by the average composition for each month?
```sql
WITH index_composition_cte AS (
  SELECT
      interest_metrics.month_year
    , interest_map.interest_name
    , ROUND((interest_metrics.composition / interest_metrics.index_value)::NUMERIC, 2) AS index_composition
    , RANK() OVER (
              PARTITION BY interest_metrics.month_year
              ORDER BY (interest_metrics.composition / interest_metrics.index_value) DESC
          ) AS index_rank
  FROM fresh_segments.interest_metrics
  INNER JOIN fresh_segments.interest_map
    ON interest_metrics.interest_id = interest_map.id
  WHERE interest_metrics.month_year IS NOT NULL
)
SELECT *
FROM index_composition_cte
LIMIT 10;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/84256802-d1ce-45ec-b1e2-f3a3fdca6d68)

> 2. For all of these top 10 interests - which interest appears the most often?
```sql
WITH index_composition_cte AS (
  SELECT
      interest_metrics.month_year
    , interest_map.interest_name
    , ROUND((interest_metrics.composition / interest_metrics.index_value)::NUMERIC, 2) AS index_composition
    , RANK() OVER (
              PARTITION BY interest_metrics.month_year
              ORDER BY (interest_metrics.composition / interest_metrics.index_value) DESC
          ) AS index_rank
  FROM fresh_segments.interest_metrics
  INNER JOIN fresh_segments.interest_map
    ON interest_metrics.interest_id = interest_map.id
  WHERE interest_metrics.month_year IS NOT NULL
)
SELECT 
    interest_name
  , COUNT(*) AS appearances
FROM index_composition_cte
WHERE index_rank <= 10
GROUP BY interest_name
ORDER BY appearances DESC
LIMIT 3;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/b29a6c13-3c05-436e-9a95-cd4b05e2dbe6)

> 3. What is the average of the average composition for the top 10 interests for each month?
```sql
WITH index_composition_cte AS (
  SELECT
      interest_metrics.month_year
    , interest_map.interest_name
    , ROUND((interest_metrics.composition / interest_metrics.index_value)::NUMERIC, 2) AS index_composition
    , RANK() OVER (
              PARTITION BY interest_metrics.month_year
              ORDER BY (interest_metrics.composition / interest_metrics.index_value) DESC
          ) AS index_rank
  FROM fresh_segments.interest_metrics
  INNER JOIN fresh_segments.interest_map
    ON interest_metrics.interest_id = interest_map.id
  WHERE interest_metrics.month_year IS NOT NULL
)
SELECT 
    month_year
  , ROUND(AVG(index_composition), 2) AS avg_index_composition
FROM index_composition_cte
WHERE index_rank <= 10
GROUP BY month_year
ORDER BY month_year;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/76fac1ea-978d-42ac-ba06-17793e5bd233)

> 4. What is the 3 month rolling average of the max average composition value from September 2018 to August 2019 and include the previous top ranking interests in the same output shown below.
```sql
WITH index_composition AS (
  SELECT
      interest_metrics.month_year
    , interest_map.interest_name
    , ROUND(
          (interest_metrics.composition / interest_metrics.index_value)::NUMERIC,
          2
        ) AS index_composition
    , RANK() OVER (
          PARTITION BY interest_metrics.month_year
          ORDER BY interest_metrics.composition / interest_metrics.index_value DESC
        ) AS index_rank
  FROM fresh_segments.interest_metrics
  INNER JOIN fresh_segments.interest_map
    ON interest_metrics.interest_id = interest_map.id
  WHERE interest_metrics.month_year IS NOT NULL
),
final_output AS (
SELECT
    month_year
  , interest_name
  , index_composition AS max_index_composition
  , ROUND(
        MAX(index_composition) OVER (
              ORDER BY month_year
              RANGE BETWEEN '2 months' PRECEDING AND CURRENT ROW
          ),
        2
      ) AS "3_month_moving_avg"
  , LAG(interest_name || ': ' || index_composition) OVER (ORDER BY month_year) AS "1_month_ago"
  , LAG(interest_name || ': ' || index_composition, 2) OVER (ORDER BY month_year) AS "2_months_ago"
FROM index_composition
WHERE index_rank = 1
)
SELECT * FROM final_output
WHERE "2_months_ago" is not null
ORDER BY month_year;
```
![image](https://github.com/jef-fortunahamid/CaseStudy8_FreshSegments/assets/125134025/670d49ee-24d6-4660-b5f0-3fed202c50e4)
