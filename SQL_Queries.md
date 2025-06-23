## Files prepared for Union

Column names in the 2019 Q4 and 2020 Q1 datasets were standardized to ensure compatibility. Non-matching fields were removed to create a uniform schema, allowing for clean merging and consistent downstream analysis. Original files were preserved to maintain data traceability.

Full backups of the Q1 and Q4 datasets were created prior to any schema or data type modifications. This step preserved the original data for recovery or audit purposes, ensuring data integrity throughout the transformation process.

## Q1 File Modification

### Query 1: Standardized Q1 Column Names for Merging
```sql
ALTER TABLE bicycles.q1
RENAME COLUMN ride_id TO trip_id,
RENAME COLUMN started_at TO start_time,
RENAME COLUMN ended_at TO end_time,
RENAME COLUMN member_casual TO usertype;
```

### Query 2: Removed Non-Matching Columns from Q1
```sql
ALTER TABLE bicycles.q1 
DROP COLUMN start_lat,
DROP COLUMN start_lng,
DROP COLUMN end_lat,
DROP COLUMN end_lng,
DROP COLUMN rideable_type;
```

## Q4 File Modification

### Query 3: Standardized Q4 Column Names to Match Q1
```sql
ALTER TABLE bicycles.q4
RENAME COLUMN from_station_id TO start_station_id,
RENAME COLUMN from_station_name TO start_station_name,
RENAME COLUMN to_station_id TO end_station_id,
RENAME COLUMN to_station_name TO end_station_name;
```

### Query 4: Removed Non-Matching Columns from Q4
```sql
ALTER TABLE bicycles.q4
DROP COLUMN bikeid,
DROP COLUMN tripduration;
```

### Query 5: Backed Up Tables Before Data Type Changes
```sql
CREATE TABLE bicycles.q1_backup AS SELECT * FROM bicycles.q1;
CREATE TABLE bicycles.q4_backup AS SELECT * FROM bicycles.q4;
```

### Query 6: Confirmed No Duplicates for trip_id
```sql
SELECT trip_id, COUNT(*) AS count
FROM bicycles.q1
GROUP BY trip_id
HAVING count > 1;
```
```sql
SELECT trip_id, COUNT(*) AS count
FROM bicycles.q4
GROUP BY trip_id
HAVING count > 1;
```

### Query 7: Converted Q1 start_time and end_time to DATETIME
```sql
UPDATE bicycles.q1
SET start_time = STR_TO_DATE(start_time, '%m/%d/%Y %H:%i');
```
```sql
ALTER TABLE bicycles.q1
MODIFY COLUMN start_time DATETIME;
```
```sql
UPDATE bicycles.q1
SET end_time = STR_TO_DATE(end_time, '%m/%d/%Y %H:%i');
```
```sql
ALTER TABLE bicycles.q1
MODIFY COLUMN end_time DATETIME;
```

### Query 8: Standardized Q4 DateTime Columns
```sql
ALTER TABLE bicycles.q4
MODIFY COLUMN start_time DATETIME;
```
```sql
ALTER TABLE bicycles.q4
MODIFY COLUMN end_time DATETIME;
```

### Query 9: Schema Confirmation
```sql
DESCRIBE bicycles.q1;
DESCRIBE bicycles.q4;
```

### Query 10: Checked for Null Primary Keys
```sql
SELECT trip_id FROM bicycles.q4 WHERE trip_id IS NULL;
SELECT trip_id FROM bicycles.q1 WHERE trip_id IS NULL;
```

### Query 11: Created Merged Table Using FULL OUTER JOIN via UNION
```sql
CREATE TABLE bicycles.full_data AS
SELECT 
    COALESCE(q1.trip_id, q4.trip_id) AS trip_id,
    COALESCE(q1.start_time, q4.start_time) AS start_time,
    COALESCE(q1.end_time, q4.end_time) AS end_time,
    COALESCE(q1.start_station_id, q4.start_station_id) AS start_station_id,
    COALESCE(q1.start_station_name, q4.start_station_name) AS start_station_name,
    COALESCE(q1.end_station_id, q4.end_station_id) AS end_station_id,
    COALESCE(q1.end_station_name, q4.end_station_name) AS end_station_name,
    COALESCE(q1.usertype, q4.usertype) AS usertype
FROM bicycles.q1 q1
LEFT JOIN bicycles.q4 q4
ON q1.trip_id = q4.trip_id

UNION

SELECT 
    COALESCE(q1.trip_id, q4.trip_id) AS trip_id,
    COALESCE(q1.start_time, q4.start_time) AS start_time,
    COALESCE(q1.end_time, q4.end_time) AS end_time,
    COALESCE(q1.start_station_id, q4.start_station_id) AS start_station_id,
    COALESCE(q1.start_station_name, q4.start_station_name) AS start_station_name,
    COALESCE(q1.end_station_id, q4.end_station_id) AS end_station_id,
    COALESCE(q1.end_station_name, q4.end_station_name) AS end_station_name,
    COALESCE(q1.usertype, q4.usertype) AS usertype
FROM bicycles.q1 q1
RIGHT JOIN bicycles.q4 q4
ON q1.trip_id = q4.trip_id;
```

### Query 12: Added and Populated ride_length Column
```sql
ALTER TABLE bicycles.full_data
ADD COLUMN ride_length INT;
```
```sql
UPDATE bicycles.full_data
SET ride_length = TIMESTAMPDIFF(MINUTE, start_time, end_time);
```

### Query 13: Validated Date Columns Are Non-Null
```sql
SELECT *
FROM bicycles.full_data
WHERE start_time IS NULL OR end_time IS NULL;
```

### Query 14: Handled Invalid Ride Lengths (Negative Values)
```sql
UPDATE bicycles.full_data
SET ride_length = NULL
WHERE end_time < start_time;
```

### Query 15: Confirmed Start and End Station ID/Name Discrepancies
```sql
SELECT count(distinct(start_station_id)) FROM bicycles.full_data;
SELECT count(distinct(start_station_name)) FROM bicycles.full_data;
SELECT start_station_id, GROUP_CONCAT(DISTINCT start_station_name) AS station_names
FROM bicycles.full_data
GROUP BY start_station_id
HAVING COUNT(DISTINCT start_station_name) > 1;
```

### Query 16: Validated Reversed Station Mapping
```sql
SELECT start_station_name, GROUP_CONCAT(DISTINCT start_station_id) AS station_id
FROM bicycles.full_data
GROUP BY start_station_name
HAVING COUNT(DISTINCT start_station_id) > 1;
```

### Query 17: Reassigned Station ID for Duplicate Names
```sql
UPDATE bicycles.full_data
SET start_station_id = 700
WHERE start_station_id = 208 AND start_station_name = 'Laflin St & Cullerton St';
```

### Query 18: Confirmed End Station Discrepancies and Updated
```sql
SELECT count(distinct(end_station_id)) FROM bicycles.full_data;
SELECT count(distinct(end_station_name)) FROM bicycles.full_data;
SELECT end_station_name, GROUP_CONCAT(DISTINCT end_station_id) AS station_id
FROM bicycles.full_data
GROUP BY end_station_name
HAVING COUNT(DISTINCT end_station_id) > 1;
```

### Query 19: Identified Start Stations Not Found as End Stations
```sql
CREATE TEMPORARY TABLE bicycles.temp_start_stations AS
SELECT DISTINCT start_station_name
FROM bicycles.full_data;
```
```sql
CREATE TEMPORARY TABLE bicycles.temp_end_stations AS
SELECT DISTINCT end_station_name
FROM bicycles.full_data;
```
```sql
SELECT start_station_name
FROM bicycles.temp_start_stations
WHERE start_station_name NOT IN (
    SELECT end_station_name
    FROM bicycles.temp_end_stations
);
```

### Query 20: Confirmed No Null Values in Full Dataset
```sql
SELECT * 
FROM bicycles.full_data
WHERE trip_id IS NULL 
   OR start_time IS NULL 
   OR end_time IS NULL 
   OR start_station_name IS NULL 
   OR end_station_name IS NULL;
```

### Query 21: Confirmed No Duplicate Trips or Misassigned Stations
```sql
SELECT trip_id, COUNT(*)
FROM bicycles.full_data
GROUP BY trip_id
HAVING COUNT(*) > 1;
```
```sql
SELECT start_station_name, COUNT(*)
FROM bicycles.full_data
GROUP BY start_station_name
HAVING COUNT(DISTINCT start_station_id) > 1;
```
```sql
SELECT end_station_name, COUNT(*)
FROM bicycles.full_data
GROUP BY end_station_name
HAVING COUNT(DISTINCT end_station_id) > 1;
```

### Query 22: Removed HQ QR Station Entries
```sql
DELETE FROM bicycles.full_data
WHERE start_station_id = 675 OR end_station_id = 675;
```

### Query 23: Normalized Usertype Labels
```sql
UPDATE bicycles.full_data
SET usertype = 'Member'
WHERE usertype = 'Subscriber';
```
```sql
UPDATE bicycles.full_data
SET usertype = 'Casual'
WHERE usertype = 'Customer';
```

### Query 24: Removed Outliers Based on Ride Length
```sql
DELETE FROM bicycles.full_data
WHERE ride_length > 1744 OR ride_length < 0;
```

### Query 25: Aggregated Summary Metrics
```sql
SELECT usertype, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype;
```
```sql
SELECT usertype, AVG(ride_length) AS avg_ride_length
FROM bicycles.full_data
GROUP BY usertype;
```
```sql
SELECT usertype, hour(start_time) AS hour_of_day, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype, hour_of_day
ORDER BY trip_count DESC;
```

### Query 26: Popular Station Routes
```sql
SELECT usertype, start_station_name, end_station_name, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype, start_station_name, end_station_name
ORDER BY trip_count DESC
LIMIT 100;
```

### Query 27: Day of Week Analysis
```sql
SELECT 
    usertype, 
    DAYOFWEEK(start_time) AS day_of_week, 
    COUNT(*) AS trip_count,
    (COUNT(*) * 100.0 / (SELECT COUNT(*) FROM bicycles.full_data)) AS percentage_of_total
FROM bicycles.full_data
GROUP BY usertype, DAYOFWEEK(start_time)
ORDER BY usertype, day_of_week;
```

### Query 28: Seasonal and Duration Distribution
```sql
SELECT usertype, MONTH(start_time) AS month, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype, MONTH(start_time)
ORDER BY usertype, month;
```
```sql
SELECT usertype, 
       CASE 
           WHEN ride_length < 15 THEN '0-15 mins'
           WHEN ride_length BETWEEN 15 AND 30 THEN '15-30 mins'
           WHEN ride_length BETWEEN 30 AND 60 THEN '30-60 mins'
           ELSE '60+ mins'
       END AS ride_length_range,
       COUNT(*) AS trip_count,
       (COUNT(*) * 100.0 / (SELECT COUNT(*) FROM bicycles.full_data)) AS percentage_of_total
FROM bicycles.full_data
GROUP BY usertype, ride_length_range
ORDER BY usertype, ride_length_range;
```
