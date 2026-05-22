# WindowFunctions


Function	Purpose
LAG()	Previous row
LEAD()	Next row
ROW_NUMBER()	Unique sequence
RANK()	Ranking with gaps
DENSE_RANK()	Ranking without gaps
NTILE()	Buckets/percentiles
FIRST_VALUE()	First value in window
LAST_VALUE()	Last value in window
Frame clauses	Control window size


ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING

ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW



COMPLEX SQL QUESTIONS — OIL & GAS INDUSTRY

O&G Question 1 — Daily Production + 7‑Day Rolling Average

table:
well_production
----------------------------------------------------
well_id | prod_date   | oil_bbl | gas_mcf | water_bbl
--------+-------------+----------+---------+-----------
101     | 2024-01-01  | 120      | 300     | 40
101     | 2024-01-02  | 130      | 310     | 42

**Task**
For each well:
Show daily oil production
Compute 7‑day moving average of oil
Compute % change vs previous day
Compute cumulative oil production since the start of the year


**SOL**
select well_id, prod_date, oil_bbl, SUM(oil_bbl) OVER (ORDER BY prod_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as 7‑day_moving_average_oil, ABS(LAG(oil_bbl) OVER (ORDER BY  prod_date) - oill_bbl) /100 as change_percentage, SUM(oil_bbl) OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as cumulative_oil_production from well_production;

**Correct**

SELECT
    well_id,
    prod_date,
    oil_bbl,

    -- 7‑day moving average
    AVG(oil_bbl) OVER (
        PARTITION BY well_id
        ORDER BY prod_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7_days,

    -- % change vs previous day
    ROUND(
        (oil_bbl - LAG(oil_bbl) OVER (
            PARTITION BY well_id
            ORDER BY prod_date
        )) 
        / NULLIF(LAG(oil_bbl) OVER (
            PARTITION BY well_id
            ORDER BY prod_date
        ), 0) * 100,
    2) AS pct_change_vs_prev_day,

    -- cumulative oil production since start of year
    SUM(oil_bbl) OVER (
        PARTITION BY well_id
        ORDER BY prod_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_oil_production

FROM well_production
ORDER BY well_id, prod_date;


O&G Question 2 — Detect Production Drop Events

A “production drop event” is when: oil_bbl_today < 0.8 * oil_bbl_yesterday

Task
Write a query that:
Identifies all drop events
Shows the previous day’s production
Shows the % drop
Flags wells with 3 consecutive drop days (use windowed COUNT)

**SOL**

select 
    well_id,
    prod_date,
    oil_bbl,
    IF (oil_bbl < (0.8* (LAG(oil_bbl) OVER (PARTATION BY well_id ORDER BY prod_date)))) THEN DROP ELSE NULL ,
    LAG(oil_bbl) OVER (PARTATION BY well_id ORDER BY prod_date) as previous_day_production,
    (oil_bbl - (LAG(oil_bbl) OVER (PARTATION BY well_id ORDER BY prod_date)))/(LAG(oil_bbl) OVER (PARTATION BY well_id ORDER BY prod_date))*100 as persont_droop
    COUNT(
    oil_bbl < (0.8* (LAG(oil_bbl) OVER (PARTATION BY well_id ORDER BY prod_date)))
    ) OVER (PARTITION BY well_id ORDER BY prod_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as flag_droop from well_production
ORDER BY well_id, prod_date;
    
**Correct** 

    SELECT
    well_id,
    prod_date,
    oil_bbl,

    -- Previous day's production
    LAG(oil_bbl) OVER (
        PARTITION BY well_id
        ORDER BY prod_date
    ) AS previous_day_production,

    -- Drop flag (1 = drop event)
    CASE
        WHEN oil_bbl < 0.8 * LAG(oil_bbl) OVER (
            PARTITION BY well_id
            ORDER BY prod_date
        ) THEN 1
        ELSE 0
    END AS drop_event,

    -- Percent change vs previous day
    ROUND(
        (oil_bbl - LAG(oil_bbl) OVER (
            PARTITION BY well_id
            ORDER BY prod_date
        )) /
        NULLIF(LAG(oil_bbl) OVER (
            PARTITION BY well_id
            ORDER BY prod_date
        ), 0) * 100,
        2
    ) AS percent_drop,

    -- Count of drop events in last 3 days (including today)
    SUM(
        CASE
            WHEN oil_bbl < 0.8 * LAG(oil_bbl) OVER (
                PARTITION BY well_id
                ORDER BY prod_date
            ) THEN 1
            ELSE 0
        END
    ) OVER (
        PARTITION BY well_id
        ORDER BY prod_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS drop_streak_3_days

FROM well_production
ORDER BY well_id, prod_date;

O&G Question 3 — Monthly Allocation of Production to Tanks
Tables: 
production(well_id, prod_date, oil_bbl)
tank_capacity(tank_id, well_id, capacity_bbl)


Task
Allocate each day’s oil production to tanks in order of tank_id, filling each tank until capacity is reached.

This requires:
Running totals
Window frames
Gaps and islands logic
Possibly recursive CTE (Oracle)

SELECT
    well_id,
    tank_id,
    oil_bbl,
    ROUND((capacity_bbl - SUM(oil_bbl) OVER (PARTITION BY well_id ORDER BY prod_date))/NULLIF((SUM(oil_bbl) OVER (PARTITION BY well_id ORDER BY prod_date),0)*100,2) as tank_percentage_filled_with_oil
FROM well_production 
ORDER BY well_id, tank_id;

**Correct** 

WITH production_with_running_totals AS (
    SELECT
        well_id,
        prod_date,
        oil_bbl,
        SUM(oil_bbl) OVER (
            PARTITION BY well_id
            ORDER BY prod_date
        ) AS running_total_oil_bbl
    FROM production
),

tanks_with_running_capacity AS (
    SELECT
        well_id,
        tank_id,
        capacity_bbl,
        SUM(capacity_bbl) OVER (
            PARTITION BY well_id
            ORDER BY tank_id
        ) AS running_total_capacity_bbl
    FROM tanks
),

oil_allocated_to_each_tank AS (
    SELECT
        p.well_id,
        p.prod_date,
        p.oil_bbl,
        t.tank_id,
        t.capacity_bbl,

        -- Allocation logic:
        -- 1. Determine how much oil has reached this tank
        -- 2. Cannot exceed tank capacity
        -- 3. Cannot be negative
        LEAST(
            GREATEST(
                p.running_total_oil_bbl
                - (t.running_total_capacity_bbl - t.capacity_bbl),
                0
            ),
            t.capacity_bbl
        ) AS oil_allocated_to_tank
    FROM production_with_running_totals p
    CROSS JOIN tanks_with_running_capacity t
    WHERE p.running_total_oil_bbl >
          (t.running_total_capacity_bbl - t.capacity_bbl)
)

SELECT
    well_id,
    prod_date,
    tank_id,
    oil_allocated_to_tank
FROM oil_allocated_to_each_tank
WHERE oil_allocated_to_tank > 0
ORDER BY well_id, prod_date, tank_id;


O&G Question 4 — Find Top 3 Wells by Monthly Production

For each month:
Rank wells by total oil
Return only the top 3 wells per month
Use RANK() or DENSE_RANK()

WITH production_with_running_totals AS (
    SELECT
        well_id,
        prod_date,
        oil_bbl,
        SUM(oil_bbl) OVER (
            PARTITION BY well_id
            ORDER BY prod_date
        ) AS running_total_oil_bbl
    FROM production),
    rank_wells_total_oil AS (
        select  
            well_id,
            prod_date,
            oil_bbl,
            RANK(running_total_oil_bbl) OVER (PARTITION BY well_id ORDER BY prod_date) as rank_wells_totaloil
            from production_with_running_totals
    )
    
select 
    well_id,
    prod_date,
    rank_wells_totaloil,
    from rank_wells_total_oil
    ORDER by rank_wells_totaloil DESC 
    LIMIT(3);



    **Correct** 

    WITH monthly_production AS (
    SELECT
        well_id,
        TRUNC(prod_date, 'MM') AS month_start,
        SUM(oil_bbl) AS total_monthly_oil
    FROM production
    GROUP BY
        well_id,
        TRUNC(prod_date, 'MM')
),

ranked_wells AS (
    SELECT
        well_id,
        month_start,
        total_monthly_oil,
        RANK() OVER (
            PARTITION BY month_start
            ORDER BY total_monthly_oil DESC
        ) AS monthly_rank
    FROM monthly_production
)

SELECT
    well_id,
    month_start,
    total_monthly_oil,
    monthly_rank
FROM ranked_wells
WHERE monthly_rank <= 3
ORDER BY month_start, monthly_rank;

    

    
