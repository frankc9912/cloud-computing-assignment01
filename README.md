# Assignment 01

**Complete by February 12, 2025**

## Datasets

* Indego Bikeshare station status data
* Indego Trip data
  - Q3 2021
  - Q3 2022

All data is available from [Indego's Data site](https://www.rideindego.com/about/data/).

For any questions that refer to Meyerson Hall, use latitude `39.952415` and longitude `-75.192584` as the coordinates for the building.

Load all three datasets into a PostgreSQL database schema named `indego` (the name of your database is not important). Your schema should have the following structure:

> This structure is important -- particularly the **table names** and the **lowercase field names**; if your queries are not built to work with this structure then _your assignment will fail the tests_.

* **Table**: `indego.trips_2021_q3`  
  **Fields**:
    * `trip_id TEXT`
    * `duration INTEGER`
    * `start_time TIMESTAMP`
    * `end_time TIMESTAMP`
    * `start_station TEXT`
    * `start_lat FLOAT`
    * `start_lon FLOAT`
    * `end_station TEXT`
    * `end_lat FLOAT`
    * `end_lon FLOAT`
    * `bike_id TEXT`
    * `plan_duration INTEGER`
    * `trip_route_category TEXT`
    * `passholder_type TEXT`
    * `bike_type TEXT`

* **Table**: `indego.trips_2022_q3`  
  **Fields**: (same as above)

* **Table**: `indego.station_statuses`  
  **Fields** (at a minimum -- there may be many more):
    * `id INTEGER`
    * `name TEXT` (or `CHARACTER VARYING`)
    * `geog GEOGRAPHY`
    * ...

## Questions

Write a query to answer each of the questions below.
* Your queries should produce results in the format specified.
* Write your query in a SQL file corresponding to the question number (e.g. a file named _query06.sql_ for the answer to question #6).
* Each SQL file should contain a single `SELECT` query.
* Any SQL that does things other than retrieve data (e.g. SQL that creates indexes or update columns) should be in the _db_structure.sql_ file.
* Some questions include a request for you to discuss your methods. Update this README file with your answers in the appropriate place.


1. [How many bike trips in Q3 2021?](query01.sql)

    This file is filled out for you, as an example.

    ```SQL
    select count(*)
    from indego.trips_2021_q3
    ```

    **Result:** 300,432

2. [What is the percent change in trips in Q3 2022 as compared to Q3 2021?](query02.sql)
    ```SQL
    WITH
      q3_2021 AS (
        SELECT COUNT(*) AS trips_2021
        FROM indego.trips_2021_q3
      ),
      q3_2022 AS (
        SELECT COUNT(*) AS trips_2022
        FROM indego.trips_2022_q3
      )
    SELECT
      ROUND(
        (q3_2022.trips_2022 - q3_2021.trips_2021) * 100.0
        / q3_2021.trips_2021
      , 2
      ) AS perc_change
    FROM q3_2021
    CROSS JOIN q3_2022;
    ```

    **Result:** 3.98%

3. [What is the average duration of a trip for 2021?](query03.sql)
    ```SQL
    SELECT
      ROUND(AVG(duration), 2) AS avg_duration
    FROM indego.trips_2021_q3;
    ```

    **Result:** 18.86 min

4. [What is the average duration of a trip for 2022?](query04.sql)
    ```SQL
    SELECT
      ROUND(AVG(duration), 2) AS avg_duration
    FROM indego.trips_2022_q3;
    ```
    
    **Result:** 17.88 min
5. [What is the longest duration trip across the two quarters?](query05.sql)
    ```SQL
    SELECT
      MAX(duration) AS max_duration
    FROM (
      SELECT duration FROM indego.trips_2021_q3
      UNION ALL
      SELECT duration FROM indego.trips_2022_q3
    ) AS combined;
    ```
    **Result:** 1,440 min
    _Why are there so many trips of this duration?_

    **Answer: likely due to the system limit, 1,440 min = 24 hours, after that the system automatically ends trip if the bike hasn't been docked.**

6. [How many trips in each quarter were shorter than 10 minutes?](query06.sql)
    ```SQL
    WITH all_trips AS (
      SELECT duration, start_time
      FROM indego.trips_2021_q3
      UNION ALL
      SELECT duration, start_time
      FROM indego.trips_2022_q3
    )
    SELECT
      EXTRACT(YEAR FROM start_time)::INT   AS trip_year,
      EXTRACT(QUARTER FROM start_time)::INT AS trip_quarter,
      COUNT(*)                             AS num_trips
    FROM all_trips
    WHERE duration < 10
    GROUP BY trip_year, trip_quarter
    ORDER BY trip_year;
    ```

    **Result:** 2021: 124,528; 2022: 137,372

7. [How many trips started on one day and ended on a different day?](query07.sql)
    ```SQL
    SELECT
        EXTRACT(YEAR FROM start_time)::INT AS trip_year,
        EXTRACT(QUARTER FROM start_time)::INT AS trip_quarter,
        COUNT(*) AS num_trips
    FROM indego.trips_2021_q3
    WHERE start_time::DATE <> end_time::DATE
    GROUP BY trip_year, trip_quarter

    UNION ALL

    SELECT
        EXTRACT(YEAR FROM start_time)::INT AS trip_year,
        EXTRACT(QUARTER FROM start_time)::INT AS trip_quarter,
        COUNT(*) AS num_trips
    FROM indego.trips_2022_q3
    WHERE start_time::DATE <> end_time::DATE
    GROUP BY trip_year, trip_quarter

    ORDER BY trip_year;
    ```

    **Result:** 2021: 2,301; 2022: 2,060

8. [Give the five most popular starting stations across all years between 7am and 9:59am.](query08.sql)

    _Hint: Use the `EXTRACT` function to get the hour of the day from the timestamp._
    ```SQL
    WITH morning_counts AS (
      -- Q3 2021 morning counts per station
      SELECT
        start_station::INT AS station_id,
        COUNT(*)          AS trips
      FROM indego.trips_2021_q3
      WHERE EXTRACT(HOUR FROM start_time) BETWEEN 7 AND 9
      GROUP BY station_id

      UNION ALL

      -- Q3 2022 morning counts per station
      SELECT
        start_station::INT AS station_id,
        COUNT(*)          AS trips
      FROM indego.trips_2022_q3
      WHERE EXTRACT(HOUR FROM start_time) BETWEEN 7 AND 9
      GROUP BY station_id
    )

    SELECT
      s.id         AS station_id,
      s.geog       AS station_geog,
      SUM(mc.trips) AS num_trips
    FROM morning_counts mc
    JOIN indego.station_statuses s
      ON mc.station_id = s.id
    GROUP BY s.id, s.geog
    ORDER BY num_trips DESC
    LIMIT 5;
    ```
    **Result:** Station ID 3032, 3102, 3012, 3066, 3007

9.  [List all the passholder types and number of trips for each across all years.](query09.sql)
    ```SQL
    WITH all_trips AS (
        SELECT passholder_type
        FROM indego.trips_2021_q3

        UNION ALL

        SELECT passholder_type
        FROM indego.trips_2022_q3
    )
    SELECT
        passholder_type,
        COUNT(*) AS num_trips
    FROM all_trips
    GROUP BY passholder_type
    ORDER BY num_trips DESC;
    ```

    **Result:** 
    | passholder\_type | num\_trips |
    | ---------------- | ---------- |
    | Indego30         | 441,856    |
    | Indego365        | 109,251    |
    | Day Pass         | 61,659     |
    | NULL             | 43         |
    | Walk-up          | 2          |


10. [Using the station status dataset, find the distance in meters of each station from Meyerson Hall.](query10.sql)
    ```SQL
    SELECT
      id AS station_id,
      geog AS station_geog,
      (ROUND(
        ST_Distance(
          geog,
          ST_SetSRID(
            ST_MakePoint(-75.192584, 39.952415),
            4326
          )::geography
        ) / 50.0
      ) * 50
      )::INT AS distance
    FROM indego.station_statuses;
    ```

11. [What is the average distance (in meters) of all stations from Meyerson Hall?](query11.sql)
    ```SQL
    SELECT
      ROUND(
        AVG(
          ST_Distance(
            geog,
            ST_SetSRID(
              ST_MakePoint(-75.192584, 39.952415),
              4326
            )::geography
          ) / 1000.0
        )
      ) AS avg_distance_km
    FROM indego.station_statuses;
    ```
    **Result:** 3 km

12. [How many stations are within 1km of Meyerson Hall?](query12.sql)
    ```SQL
    SELECT
      COUNT(*) AS num_stations
    FROM indego.station_statuses
    WHERE ST_DWithin(
      geog,
      ST_SetSRID(
        ST_MakePoint(-75.192584, 39.952415),
        4326
      )::geography,
      1000
    );
    ```
    **Result:** 17 stations

13. [Which station is furthest from Meyerson Hall?](query13.sql)
    ```SQL
    SELECT
      id           AS station_id,
      name         AS station_name,
      (
        ROUND(
          ST_Distance(
            geog,
            ST_SetSRID(ST_MakePoint(-75.192584, 39.952415), 4326)::geography
          ) / 50.0
        ) * 50
      )::INT      AS distance
    FROM indego.station_statuses
    ORDER BY distance DESC
    LIMIT 1;
    ```
    **Result:** Station ID 3323, 8,900 m

14. [Which station is closest to Meyerson Hall?](query14.sql)
    ```SQL
    SELECT
      id           AS station_id,
      name         AS station_name,
      (
        ROUND(
          ST_Distance(
            geog,
            ST_SetSRID(ST_MakePoint(-75.192584, 39.952415), 4326)::geography
          ) / 50.0
        ) * 50
      )::INT      AS distance
    FROM indego.station_statuses
    ORDER BY distance ASC
    LIMIT 1;
    ```
    **Result:** Station ID 3208, 200 m
