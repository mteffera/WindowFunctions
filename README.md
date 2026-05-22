# WindowFunctions

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



O&G Question 4 — Find Top 3 Wells by Monthly Production

For each month:
Rank wells by total oil
Return only the top 3 wells per month
Use RANK() or DENSE_RANK()
