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

## Data Preparation Queries

### Query 3: Standardize Q4 Column Names to Match Q1

This query renames fields in the `q4` table to match the `q1` structure, ensuring consistent schema alignment for merging.

```sql
ALTER TABLE bicycles.q4
RENAME COLUMN from_station_id TO start_station_id,
RENAME COLUMN from_station_name TO start_station_name,
RENAME COLUMN to_station_id TO end_station_id,
RENAME COLUMN to_station_name TO end_station_name;

### Query 4: Remove Non-Matching Columns from Q4

This query removes columns from the `q4` table that are not present in `q1`, ensuring both tables share a common structure for consistent merging.

```sql
ALTER TABLE bicycles.q4
DROP COLUMN bikeid,
DROP COLUMN tripduration;
