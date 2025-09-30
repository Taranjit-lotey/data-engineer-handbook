# Week 2 Fact Data Modeling
The homework this week will be using the `devices` and `events` dataset

Construct the following eight queries:

- A query to deduplicate `game_details` from Day 1 so there's no duplicates

```sql
-- Remove duplicate rows from game_details table
SELECT DISTINCT
    game_id,
    team_id,
    team_abbreviation,
    team_city,
    player_id,
    player_name,
    nickname,
    start_position,
    comment,
    min,
    fgm,
    fga,
    fg_pct,
    fg3m,
    fg3a,
    fg3_pct,
    ftm,
    fta,
    ft_pct,
    oreb,
    dreb,
    reb,
    ast,
    stl,
    blk,
    "TO",
    pf,
    pts,
    plus_minus
FROM game_details;
```

- A DDL for an `user_devices_cumulated` table that has:
  - a `device_activity_datelist` which tracks a users active days by `browser_type`
  - data type here should look similar to `MAP<STRING, ARRAY[DATE]>`
    - or you could have `browser_type` as a column with multiple rows for each user (either way works, just be consistent!)

```sql
-- Create cumulated table tracking user activity by browser type
CREATE TABLE user_devices_cumulated (
    user_id BIGINT,
    browser_type TEXT,
    device_activity_datelist DATE[],
    date DATE,
    PRIMARY KEY (user_id, browser_type, date)
);
```

- A cumulative query to generate `device_activity_datelist` from `events`

```sql
-- Incrementally build device_activity_datelist by joining yesterday's data with today's events
WITH yesterday AS (
    SELECT *
    FROM user_devices_cumulated
    WHERE date = DATE('2023-01-30')
),
today AS (
    SELECT
        e.user_id,
        d.browser_type,
        DATE_TRUNC('day', e.event_time) AS today_date
    FROM events e
    JOIN devices d ON e.device_id = d.device_id
    WHERE DATE_TRUNC('day', e.event_time) = DATE('2023-01-31')
        AND e.user_id IS NOT NULL
    GROUP BY e.user_id, d.browser_type, DATE_TRUNC('day', e.event_time)
)
INSERT INTO user_devices_cumulated
SELECT
    COALESCE(t.user_id, y.user_id) AS user_id,
    COALESCE(t.browser_type, y.browser_type) AS browser_type,
    COALESCE(y.device_activity_datelist, ARRAY[]::DATE[])
        || CASE WHEN t.user_id IS NOT NULL
           THEN ARRAY[t.today_date]
           ELSE ARRAY[]::DATE[]
           END AS device_activity_datelist,
    COALESCE(t.today_date, y.date + INTERVAL '1 day') AS date
FROM yesterday y
FULL OUTER JOIN today t
    ON y.user_id = t.user_id
    AND y.browser_type = t.browser_type;
```

- A `datelist_int` generation query. Convert the `device_activity_datelist` column into a `datelist_int` column

```sql
-- Convert date array into 32-bit integer representation for efficient storage
WITH starter AS (
    SELECT
        udc.device_activity_datelist @> ARRAY[DATE(d.valid_date)] AS is_active,
        EXTRACT(DAY FROM DATE('2023-01-31') - d.valid_date) AS days_since,
        udc.user_id,
        udc.browser_type
    FROM user_devices_cumulated udc
    CROSS JOIN (
        SELECT generate_series('2023-01-01', '2023-01-31', INTERVAL '1 day') AS valid_date
    ) AS d
    WHERE udc.date = DATE('2023-01-31')
),
bits AS (
    SELECT
        user_id,
        browser_type,
        SUM(CASE WHEN is_active THEN POW(2, 32 - days_since) ELSE 0 END)::BIGINT::BIT(32) AS datelist_int,
        DATE('2023-01-31') AS date
    FROM starter
    GROUP BY user_id, browser_type
)
SELECT * FROM bits;
``` 

- A DDL for `hosts_cumulated` table
  - a `host_activity_datelist` which logs to see which dates each host is experiencing any activity

```sql
-- Create cumulated table tracking activity dates for each host
CREATE TABLE hosts_cumulated (
    host TEXT,
    host_activity_datelist DATE[],
    date DATE,
    PRIMARY KEY (host, date)
);
```
  
- The incremental query to generate `host_activity_datelist`

```sql
-- Incrementally build host_activity_datelist by joining yesterday's data with today's events
WITH yesterday AS (
    SELECT *
    FROM hosts_cumulated
    WHERE date = DATE('2023-01-30')
),
today AS (
    SELECT
        host,
        DATE_TRUNC('day', event_time) AS today_date
    FROM events
    WHERE DATE_TRUNC('day', event_time) = DATE('2023-01-31')
        AND host IS NOT NULL
    GROUP BY host, DATE_TRUNC('day', event_time)
)
INSERT INTO hosts_cumulated
SELECT
    COALESCE(t.host, y.host) AS host,
    COALESCE(y.host_activity_datelist, ARRAY[]::DATE[])
        || CASE WHEN t.host IS NOT NULL
           THEN ARRAY[t.today_date]
           ELSE ARRAY[]::DATE[]
           END AS host_activity_datelist,
    COALESCE(t.today_date, y.date + INTERVAL '1 day') AS date
FROM yesterday y
FULL OUTER JOIN today t
    ON y.host = t.host;
```

- A monthly, reduced fact table DDL `host_activity_reduced`
   - month
   - host
   - hit_array - think COUNT(1)
   - unique_visitors array -  think COUNT(DISTINCT user_id)

```sql
-- Create monthly reduced fact table for host activity metrics
CREATE TABLE host_activity_reduced (
    host TEXT,
    month_start DATE,
    hit_array BIGINT[],
    unique_visitors_array BIGINT[],
    date_partition DATE,
    PRIMARY KEY (host, date_partition)
);
```

- An incremental query that loads `host_activity_reduced`
  - day-by-day

```sql
-- Day-by-day incremental load of monthly host activity metrics
WITH yesterday AS (
    SELECT *
    FROM host_activity_reduced
    WHERE date_partition = DATE('2023-01-30')
),
today AS (
    SELECT
        host,
        DATE_TRUNC('day', event_time) AS today_date,
        COUNT(1) AS num_hits,
        COUNT(DISTINCT user_id) AS unique_visitors
    FROM events
    WHERE DATE_TRUNC('day', event_time) = DATE('2023-01-31')
        AND host IS NOT NULL
    GROUP BY host, DATE_TRUNC('day', event_time)
)
INSERT INTO host_activity_reduced
SELECT
    COALESCE(y.host, t.host) AS host,
    DATE('2023-01-01') AS month_start,
    COALESCE(y.hit_array, array_fill(NULL::BIGINT, ARRAY[DATE('2023-01-31') - DATE('2023-01-01')]))
        || ARRAY[t.num_hits] AS hit_array,
    COALESCE(y.unique_visitors_array, array_fill(NULL::BIGINT, ARRAY[DATE('2023-01-31') - DATE('2023-01-01')]))
        || ARRAY[t.unique_visitors] AS unique_visitors_array,
    DATE('2023-01-31') AS date_partition
FROM yesterday y
FULL OUTER JOIN today t
    ON y.host = t.host;
```

Please add these queries into a folder, zip them up and submit [here](https://bootcamp.techcreator.io)