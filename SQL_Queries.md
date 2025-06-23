## 📁 Project Overview: Quarterly Bike Share Data Integration & Analysis

This project integrates, cleans, and analyzes two bike-share datasets: Q4 2019 and Q1 2020. The focus was on preparing both files for accurate merging, validating the data, handling anomalies, and drawing business-relevant insights through SQL-based exploration.

---

## 🧼 Data Preparation and Cleaning

### Step 1: Backup of Original Tables
Original tables (`q1` and `q4`) were backed up before any transformations, preserving source data for auditing or rollback.

```sql
CREATE TABLE bicycles.q1_backup AS SELECT * FROM bicycles.q1;
CREATE TABLE bicycles.q4_backup AS SELECT * FROM bicycles.q4;
```

### Step 2: Schema Alignment Across Q1 and Q4
Column names were standardized, and non-matching fields were removed to unify the schema. This ensured compatibility for merging and downstream analytics.

```sql
-- Q1 Standardization
ALTER TABLE bicycles.q1
RENAME COLUMN ride_id TO trip_id,
RENAME COLUMN started_at TO start_time,
RENAME COLUMN ended_at TO end_time,
RENAME COLUMN member_casual TO usertype;

ALTER TABLE bicycles.q1 
DROP COLUMN start_lat,
DROP COLUMN start_lng,
DROP COLUMN end_lat,
DROP COLUMN end_lng,
DROP COLUMN rideable_type;

-- Q4 Standardization
ALTER TABLE bicycles.q4
RENAME COLUMN from_station_id TO start_station_id,
RENAME COLUMN from_station_name TO start_station_name,
RENAME COLUMN to_station_id TO end_station_id,
RENAME COLUMN to_station_name TO end_station_name;

ALTER TABLE bicycles.q4
DROP COLUMN bikeid,
DROP COLUMN tripduration;
```

### Step 3: Data Type Standardization
String-based timestamps were converted to the DATETIME format. This transformation allowed for accurate time-based calculations such as trip duration.

```sql
-- Q1
UPDATE bicycles.q1
SET start_time = STR_TO_DATE(start_time, '%m/%d/%Y %H:%i');
ALTER TABLE bicycles.q1
MODIFY COLUMN start_time DATETIME;

UPDATE bicycles.q1
SET end_time = STR_TO_DATE(end_time, '%m/%d/%Y %H:%i');
ALTER TABLE bicycles.q1
MODIFY COLUMN end_time DATETIME;

-- Q4
ALTER TABLE bicycles.q4
MODIFY COLUMN start_time DATETIME;
ALTER TABLE bicycles.q4
MODIFY COLUMN end_time DATETIME;
```

### Step 4: Uniqueness and Null Validation
Verified that `trip_id` values were unique and that primary columns (trip IDs, timestamps, station names) contained no null values.

```sql
SELECT trip_id, COUNT(*) AS count
FROM bicycles.q1
GROUP BY trip_id
HAVING count > 1;

SELECT trip_id, COUNT(*) AS count
FROM bicycles.q4
GROUP BY trip_id
HAVING count > 1;

SELECT trip_id FROM bicycles.q1 WHERE trip_id IS NULL;
SELECT trip_id FROM bicycles.q4 WHERE trip_id IS NULL;
```

---

## 🔗 Data Integration

### Step 5: Full Outer Join via Left + Right Union
Since MySQL does not support native full joins, a combination of LEFT and RIGHT joins was used with the `COALESCE()` function to unify both tables into a single merged dataset.

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
LEFT JOIN bicycles.q4 q4 ON q1.trip_id = q4.trip_id
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
RIGHT JOIN bicycles.q4 q4 ON q1.trip_id = q4.trip_id;
```

### Step 6: Ride Duration Calculation
A new `ride_length` column was created using the time difference between `start_time` and `end_time`. Negative durations were handled appropriately to remove invalid entries.

```sql
ALTER TABLE bicycles.full_data ADD COLUMN ride_length INT;
UPDATE bicycles.full_data
SET ride_length = TIMESTAMPDIFF(MINUTE, start_time, end_time);

-- Remove invalid durations
UPDATE bicycles.full_data
SET ride_length = NULL
WHERE end_time < start_time;
```

---

## 🥪 Data Validation

### Step 7: Station ID and Name Conflicts
Identified and resolved mismatches between station names and IDs (e.g., station 208 had multiple associated names). Custom logic was applied to resolve duplicates and assign consistent identifiers.

```sql
-- Conflict checks
SELECT start_station_id, GROUP_CONCAT(DISTINCT start_station_name) AS station_names
FROM bicycles.full_data
GROUP BY start_station_id
HAVING COUNT(DISTINCT start_station_name) > 1;

-- Reassignment
UPDATE bicycles.full_data
SET start_station_id = 700
WHERE start_station_id = 208 AND start_station_name = 'Laflin St & Cullerton St';
```

### Step 8: Maintenance Station Filtering
Trips that started or ended at internal maintenance locations (e.g., "HQ QR") were removed from the dataset to avoid skewing duration or traffic data.

```sql
DELETE FROM bicycles.full_data
WHERE start_station_id = 675 OR end_station_id = 675;
```

### Step 9: Usertype Normalization
Standardized user types by mapping `Subscriber` to `Member` and `Customer` to `Casual` for cleaner analysis.

```sql
UPDATE bicycles.full_data
SET usertype = 'Member'
WHERE usertype = 'Subscriber';

UPDATE bicycles.full_data
SET usertype = 'Casual'
WHERE usertype = 'Customer';
```

---

## 📊 Outlier Detection and Removal

### Step 10: Outlier Filtering on Ride Duration
Outliers were removed using a ±3 standard deviation rule from the average ride length. Manual thresholds were applied due to MySQL constraints in deleting while referencing the same table. Approximately 82% of outliers were tied to `Casual` riders.

```sql
DELETE FROM bicycles.full_data
WHERE ride_length > 1744 OR ride_length < 0;
```

---

## 📈 Exploratory Data Analysis

### Step 11: User Type Trip Distribution
```sql
SELECT usertype, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype;
```

### Step 12: Average Ride Duration by Usertype
```sql
SELECT usertype, AVG(ride_length) AS avg_ride_length
FROM bicycles.full_data
GROUP BY usertype;
```

### Step 13: Hourly Peak Analysis
```sql
SELECT usertype, HOUR(start_time) AS hour_of_day, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype, hour_of_day
ORDER BY trip_count DESC;
```

### Step 14: Station Popularity
```sql
SELECT usertype, start_station_name, end_station_name, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype, start_station_name, end_station_name
ORDER BY trip_count DESC
LIMIT 100;
```

### Step 15: Day-of-Week Usage
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

### Step 16: Time Distribution Segments
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

### Step 17: Top Routes Analysis
(See Station Popularity section above for route-based counts.)

---

## 🔍 Business Insight

- **Membership Behavior**: Members follow predictable commuting schedules and short ride patterns.
- **Casual Behavior**: Casual riders spend significantly more time per trip and tend to ride during non-working hours.
- **Operational Consideration**: Stations with ID-name mismatches may represent outdated labels, technical errors, or internal tracking issues that require operational review.
- **Growth Opportunities**: High-usage leisure locations may benefit from targeted marketing or seasonal pricing strategies.

---

## ✅ Outcome

The dataset is clean, validated, and merged, ready for dashboarding or further modeling. The project showcases data wrangling, validation, and insights generation from raw operational data, demonstrating readiness for business-facing analytics roles.
