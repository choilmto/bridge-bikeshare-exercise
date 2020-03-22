# Part 1

```
WITH station_to_from_count AS (
  SELECT
    from_station_id AS station_id,
    from_station_name AS station_name,
    COUNT(from_station_name) AS count
  FROM trips
  GROUP BY from_station_id, from_station_name
  UNION
  SELECT
    to_station_id AS station_id,
    to_station_name AS station_name,
    COUNT(to_station_name) AS count
  FROM trips
  GROUP BY to_station_id, to_station_name
), station_count AS (
  SELECT
    station_id,
    station_name,
    SUM(count)
  FROM station_to_from_count
  GROUP BY station_id, station_name
), popular_station_names_and_ids AS (
  SELECT
    station_count.station_id,
    station_count.station_name,
    station_count.sum
  FROM station_count
  JOIN
  (
    SELECT
      MAX(sum),
      station_count.station_id
    FROM station_count
    GROUP BY station_count.station_id
  ) AS max_count
  ON max_count.station_id=station_count.station_id
  WHERE max_count.max=station_count.sum
), null_count AS (
  SELECT *
  FROM station_count
  WHERE (station_count.station_id IS NULL) AND (sum > 3)
), levenshtein_names AS (
  SELECT DISTINCT
    null_count.station_name,
    stations.name,
    levenshtein(stations.name, null_count.station_name)
  FROM null_count
  LEFT JOIN stations
  ON levenshtein(stations.name, null_count.station_name) <= 4
)
```
This `WITH` creates 5 tables.

Tables | Description
--- | ---
`station_to_from_count` | groups station names by column and counts their frequencies; ids are `null` when name doesn't match station names with ids
`station_count` | sums start and points to stations with same name
`popular_station_names_and_ids` | picks station names with largest sum
`null_count` | list station names with `null` for id
`levenshtein_names` | matches station names with `null` for id to popular station names as long the levenshtein is 4 or under

## Question 1
```
INSERT INTO stations
SELECT station_id, station_name
FROM popular_station_names_and_ids;
```
This will populate the `stations` table with ids and names.

## Question 3
```
UPDATE trips
SET to_station_name=stations.name
FROM stations
WHERE trips.to_station_id=stations.id
```
```
UPDATE trips
SET from_station_name=stations.name
FROM stations
WHERE trips.from_station_id=stations.id
```
Now make the stations with the same id have the same name in trips table.

```
UPDATE trips
SET to_station_id = stations.id
FROM stations
WHERE (name = trips.to_station_name) AND (trips.to_station_id IS NULL);
```
```
UPDATE trips
SET from_station_id = stations.id
FROM stations
WHERE name = (trips.from_station_name) AND (trips.from_station_id IS NULL);
```
Where station id is null and station name is consistent, populate id.

```
UPDATE trips
SET to_station_name = stations.name
FROM stations
WHERE (levenshtein(stations.name, trips.to_station_name) <= 4) AND (trips.to_station_id IS NULL);
```
```
UPDATE trips
SET from_station_name = stations.name
FROM stations
WHERE (levenshtein(stations.name, trips.from_station_name) <= 4) AND (trips.from_station_id IS NULL);
```
Replace names with suitable matches -- levenshtein of 4 is good, whereas 6 is not good.
There are a handful of stations that don't have a good match...what to do with them?

#Part 2
## Question 1

```
SELECT
 mode() WITHIN GROUP (ORDER BY start_time_str),
 original_filename
FROM trips
GROUP BY original_filename;
```
This will sample one `start_time_str` per quarter.

Quarter | String formats
--- | ---
2017 Q1 | dd/mm/yyyy HH:MM
2017 Q2 | dd/mm/yyyy HH:MM
2017 Q3 | mm/dd/yyyy HH:MM
2017 Q4 | mm/dd/yyyy HH:MM:SS
2018 Q1 | mm/dd/yyyy HH:MM
2018 Q2 | mm/dd/yyyy HH:MM
2018 Q3 | mm/dd/yyyy HH:MM
2018 Q4 | mm/dd/yyyy HH:MM

## Question 2

```
UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'DD/MM/YYYY HH24:MI')
WHERE original_filename='Bikeshare Ridership (2017 Q1).csv' OR original_filename='Bikeshare Ridership (2017 Q2).csv';
```
```
UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'DD/MM/YYYY HH24:MI')
WHERE original_filename='Bikeshare Ridership (2017 Q1).csv' OR original_filename='Bikeshare Ridership (2017 Q2).csv';
```
```
UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'MM/DD/YYYY HH24:MI:SS')
WHERE original_filename='Bikeshare Ridership (2017 Q4).csv';
```
```
UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'MM/DD/YYYY HH24:MI:SS')
WHERE (original_filename='Bikeshare Ridership (2017 Q4).csv') AND (id != 2302635);
```
```
UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'MM/DD/YYYY HH24:MI')
WHERE start_time IS NULL;
```
```
UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'MM/DD/YYYY HH24:MI')
WHERE end_time IS NULL AND (id != 2302635);
```
