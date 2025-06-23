## Q1 File Modification:

### Query 1: Standardize Q1 Column Names for Merging

This query aligns column names in the `q1` table with those in `q4`, ensuring consistency for merging datasets. Columns are renamed to maintain structural integrity and to simplify downstream joins. The original tables remain unchanged.

```sql
ALTER TABLE bicycles.q1
RENAME COLUMN ride_id TO trip_id,
RENAME COLUMN started_at TO start_time,
RENAME COLUMN ended_at TO end_time,
RENAME COLUMN member_casual TO usertype;
```

### Query 2: Remove Non-Matching Columns from Q1

This query removes columns from the `q1` table that do not exist in the `q4` dataset. This step ensures schema compatibility for merging while preserving only the shared fields necessary for analysis.

```sql
ALTER TABLE bicycles.q1 
DROP COLUMN start_lat,
DROP COLUMN start_lng,
DROP COLUMN end_lat,
DROP COLUMN end_lng,
DROP COLUMN rideable_type;
```

## Q4 File Modification: 

### Query 3: Standardize Q4 Column Names to Match Q1

This query renames fields in the `q4` table to match the `q1` structure, ensuring consistent schema alignment for merging.

```sql
ALTER TABLE bicycles.q4
RENAME COLUMN from_station_id TO start_station_id,
RENAME COLUMN from_station_name TO start_station_name,
RENAME COLUMN to_station_id TO end_station_id,
RENAME COLUMN to_station_name TO end_station_name;
```

### Query 4: Remove Non-Matching Columns from Q4

This query removes columns from the `q4` table that are not present in `q1`, ensuring both tables share a common structure for consistent merging.

```sql
ALTER TABLE bicycles.q4
DROP COLUMN bikeid,
DROP COLUMN tripduration;
```

# Organized columns to merge. Renamed columns to match between the original two files (2019.Q4 and 2020.Q1) Created new Q1and new Q4 to align columns with designated matching names, while removing columns not held in common with other files. Also maintains original files untouched in the event I need to recover base data.



#Backup tablers before data type changes
```SQl
CREATE TABLE bicycles.q1_backup AS SELECT * FROM bicycles.q1;
CREATE TABLE bicycles.q4_backup AS SELECT * FROM bicycles.q4;
```
#confirmed no duplicates for trip_id column on either table
```sql
SELECT trip_id, COUNT(*) AS count
FROM bicycles.q1
GROUP BY column_name
HAVING count > 1;
```
```sql
SELECT trip_id, COUNT(*) AS count
FROM bicycles.q1
GROUP BY column_name
HAVING count > 1;
```
#changed dates/times to datetime datatype
#modified q1 start_time to meet datetime data type requiorements
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
```sql
ALTER TABLE bicycles.q4
MODIFY COLUMN start_time DATETIME;
```
```sql
ALTER TABLE bicycles.q4
MODIFY COLUMN end_time DATETIME;
```
#no duplicates detected upon changes. Schemas confirmed match.
```sql
describe bicycles.q1;
describe bicycles.q4;
```

#no null values found in primary key column for either tables
```sql
SELECT trip_id FROM bicycles.q4 WHERE trip_id IS NULL;
SELECT trip_id FROM bicycles.q1 WHERE trip_id IS NULL;
```
#full left and outer join  with union since MYSQL does not support full join function.
#COALESCE Function: Combines columns with the same name and prioritizes non-NULL values from either Q1 or Q4.

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
#created ride_length column for analysis
```sql
alter table bicycles.full_data
add column ride_length INT;
```
UPDATE bicycles.full_data
```sql
SET ride_length = TIMESTAMPDIFF(MINUTE, start_time, end_time);
```
#Data Validation
#Ensure that start_time and end_time have valid DATETIME values.
```sql
SELECT *
FROM bicycles.full_data
WHERE start_time IS NULL OR end_time IS NULL;
```
#Handle invalid or negative ride lengths (if end_time is before start_time):
```sql
UPDATE bicycles.full_data
SET ride_length = NULL
WHERE end_time < start_time;
```
#With tables combined. Confirming Data Validation.
```sql
select count(distinct(start_station_id))
from bicycles.full_data;
```
##Start_station_ID= 610

```sql
select count(distinct(start_station_name))
from bicycles.full_data;
``
##Start_station_name= 611

#Confirmed station 208 had multiple name values
```sql
SELECT start_station_id, GROUP_CONCAT(DISTINCT start_station_name) AS station_names
FROM bicycles.full_data
GROUP BY start_station_id
HAVING COUNT(DISTINCT start_station_name) > 1;
```
#Reversed query to confirm, no other named station had a different ID number
```sql
SELECT start_station_name, GROUP_CONCAT(DISTINCT start_station_id) AS station_id
FROM bicycles.full_data
GROUP BY start_station_name
HAVING COUNT(DISTINCT start_station_id) > 1;
```

#confirmed no station ID 700 in data. assigned to 2nd station assigned to 208
```sql
select count(start_station_id)
from bicycles.full_data
where start_station_id= 700;
```
```sql
UPDATE bicycles.full_data
SET start_station_id = 700
WHERE start_station_id = 208 AND start_station_name = 'Laflin St & Cullerton St';
```

#Confirmed similar issues with end_station_id and end_station_name
```sql
select count(distinct(end_station_id))
from bicycles.full_data;
```
#end_station_id = 609
```sql
select count(distinct(end_station_name))
from bicycles.full_data;
#end_station_name = 610
```
#ran queries to check for multiple assignments also returned station 208. Updated same way.
```sql
SELECT end_station_name, GROUP_CONCAT(DISTINCT end_station_id) AS station_id
FROM bicycles.full_data
GROUP BY end_station_name
HAVING COUNT(DISTINCT end_station_id) > 1;
```
```sql
SELECT start_station_id, GROUP_CONCAT(DISTINCT start_station_name) AS station_names
FROM bicycles.full_data
GROUP BY start_station_id
HAVING COUNT(DISTINCT start_station_name) > 1;
```
#NOTE: Bicycles don't seem to make it back to Calumet Ave & 71 st or Racine Ave & 61st St. Curious finding. Additionally, bikes from Vincernnes Ave & 75th don't leave this location. 
#I would ask about these locations if I had access to someone who knows the business. However this accounts for the different in count of stations IDs betweeen starting and ending locations.

# Create a Temporary Table for Start Stations
#Extract distinct start_station_name values into a temporary table:
```sql
CREATE TEMPORARY TABLE bicycles.temp_start_stations AS
SELECT DISTINCT start_station_name
FROM bicycles.full_data;
```
#Create a Temporary Table for End Stations
#Extract distinct end_station_name values into another temporary table:
```sql
CREATE TEMPORARY TABLE bicycles.temp_end_stations AS
SELECT DISTINCT end_station_name
FROM bicycles.full_data;
```
#Find Missing Stations
#Compare the two temporary tables to find start_station_name values that are not in end_station_name:
```sql
SELECT start_station_name
FROM bicycles.temp_start_stations
WHERE start_station_name NOT IN (
    SELECT end_station_name
    FROM bicycles.temp_end_stations
);
```
#Confirmed no null values 
```sql
SELECT * 
FROM bicycles.full_data
WHERE trip_id IS NULL 
   OR start_time IS NULL 
   OR end_time IS NULL 
   OR start_station_name IS NULL 
   OR end_station_name IS NULL;
```
#Confirmed no duplicate trips
```sql
SELECT trip_id, COUNT(*)
FROM bicycles.full_data
GROUP BY trip_id
HAVING COUNT(*) > 1;
```
#confirmed no duplicates for station names in start and end stations.
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
#Reviewed ride length negative values and discovered HQ QR station as maintenance tracking. Removed all HQ QR station entries.
```sql
SELECT *
FROM bicycles.full_data
WHERE start_time > end_time;
 ```
DELETE FROM bicycles.full_data
```sql
WHERE start_station_id = 675 OR end_station_id = 675;
```
#normalized usertype entries
```sql
select distinct(usertype)
from bicycles.full_data;
```
#Subscriber->Members
```sql
UPDATE bicycles.full_data
SET usertype = 'Member'
WHERE usertype IN ('Subscriber');
```

#Customer-> Casual
```sql
UPDATE bicycles.full_data
SET usertype = 'Member'
WHERE usertype IN ('Subscriber');
```
#checked for outliers 311 count for removal
```sql
SELECT *
FROM bicycles.full_data
WHERE ride_length > (SELECT AVG(ride_length) + 3 * STDDEV(ride_length) FROM bicycles.full_data)
   OR ride_length < (SELECT AVG(ride_length) - 3 * STDDEV(ride_length) FROM bicycles.full_data);

SELECT count(*)
FROM bicycles.full_data
WHERE ride_length > (SELECT AVG(ride_length) + 3 * STDDEV(ride_length) FROM bicycles.full_data)
   OR ride_length < (SELECT AVG(ride_length) - 3 * STDDEV(ride_length) FROM bicycles.full_data);

SELECT usertype, count(*)
FROM bicycles.full_data
WHERE ride_length > (SELECT AVG(ride_length) + 3 * STDDEV(ride_length) FROM bicycles.full_data)
   OR ride_length < (SELECT AVG(ride_length) - 3 * STDDEV(ride_length) FROM bicycles.full_data)
group by usertype;
```

#had to figure out the actual thresholds manually as MySQL did not allow deletion at the same time as referencing the same table.
```sql
DELETE FROM bicycles.full_data
WHERE ride_length > 1744.11 OR ride_length < 0;
```
##NOTE: 258 out of the 311 outlier entries were for higher than average rentals to "casual" customers. That 82% of outliers.

#READY FOR ANALAYSIS! :D

#Trip count distributions per usertype
```sql
SELECT usertype, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype;

#member	541809 86%
#casual	86289 14%
#total 628098

#Average ride length by usertype
select usertype, avg(ride_length) as AVG_Ride_Legnth
from bicycles.full_data
group by usertype;

##member	11.6484
##casual	39.0543
###Casual customers are likely to keep bikes about 4 times longer. 

#Peak Time Assessement

select usertype, hour(start_time) as Hour_of_Day, count(*) as Trip_Count
from bicycles.full_data
group by usertype, hour_of_day
order by trip_count desc;

## Members Drive overall Peak Time Ranging from 7-9AM and then again 3-5PM. 
##Casuals Strongest peak isfrom 1-5PM. Morning volume is weak.... what could be causing this? Could there be a reservation system that allows members priority to bikes for morning commutes, shortage of bikes once reservations are full? Unknown membership benefit? Maybe casuals prefer to walk in the mornings full of energy and spend the money at night when tired?
Usertype| Hour| Trip_Count
member	17	70236
member	8	58613
member	16	55765
member	7	45315
member	18	41580
member	15	32360
member	9	26554
casual	15	9137
casual	14	8940
casual	16	8905
casual	13	8504
casual	17	8408
casual	12	7672

##Most popular Stations by usertype
SELECT usertype, start_station_name, COUNT(*) AS trip_count
FROM bicycles.full_data
where usertype = 'member'
GROUP BY usertype, start_station_name
ORDER BY trip_count DESC
LIMIT 10;

SELECT usertype, start_station_name, COUNT(*) AS trip_count
FROM bicycles.full_data
where usertype = 'casual'
GROUP BY usertype, start_station_name
ORDER BY trip_count DESC
LIMIT 10;

SELECT usertype, end_station_name, COUNT(*) AS trip_count
FROM bicycles.full_data
where usertype = 'casual'
GROUP BY usertype, end_station_name
ORDER BY trip_count DESC
LIMIT 10;

SELECT usertype, end_station_name, COUNT(*) AS trip_count
FROM bicycles.full_data
where usertype = 'member'
GROUP BY usertype, end_station_name
ORDER BY trip_count DESC
LIMIT 10;

#Top Three for members are not even on the list of of top 10 for casuals. Could we be looking at a different market of customer?

#Day of the week analysis
SELECT 
    usertype, 
    DAYOFWEEK(start_time) AS day_of_week, 
    COUNT(*) AS trip_count,
    (COUNT(*) * 100.0 / (SELECT COUNT(*) FROM bicycles.full_data)) AS percentage_of_total
FROM bicycles.full_data
GROUP BY usertype, DAYOFWEEK(start_time)
ORDER BY usertype, day_of_week;

#Members have higher usage mon-fri, casuals sat/sun.

##Seasonal trends not possible with data sample provided. Data seems to include Oct, Jan, Feb, March. However, there does seem to be an increase in the warmer months based on this limited view of the year.

SELECT usertype, MONTH(start_time) AS month, COUNT(*) AS trip_count
FROM bicycles.full_data
GROUP BY usertype, MONTH(start_time)
ORDER BY usertype, month;

#Time distribution per member shows that both groups are most active in the 0-15 min and 15-30 range. Members in particular dominate 65% of all trips in this time range.
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

#top routes showed different locations for top activity. Still ideal for advertising and incentive, sign on bonus, etc

SELECT usertype, start_station_name, end_station_name, COUNT(*) AS trip_count,        (COUNT(*) * 100.0 / (SELECT COUNT(*) FROM bicycles.full_data)) AS percentage_of_total
FROM bicycles.full_data
GROUP BY usertype, start_station_name, end_station_name
ORDER BY trip_count DESC
LIMIT 100;
```