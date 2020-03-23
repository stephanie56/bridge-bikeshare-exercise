# Homework

## Instructions

Clone this repo and make a PR with your answers to the questions below, and tag your reviewer in the PR.

Keep good notes of the queries you run.

## Part 1: Missing Station Data

As we've discussed in class, some of the station_id columns are NULL because of missing data. We also have inconsistent names for some of the station IDs.

Your database contains a `stations` schema:

```sql
CREATE TABLE stations
(
  id INT NOT NULL PRIMARY KEY,
  name VARCHAR(100) NOT NULL
);
```

1. Write a query that fills out this table. Try your best to pick the correct station name for each ID. You may have to make some manual choices or editing based on the inconsistencies we've found. Do try to pick the correct name for each station ID based on how popular it is in the trip data.

### Question 1 Answer:

```sql
CREATE TABLE station_popularity
(
    id INT,
    name VARCHAR(100),
    popularity integer
);

INSERT INTO station_popularity
SELECT Max(from_station_id) as station_id, from_station_name as station_name, count(from_station_name) as popularity
FROM trips
GROUP BY station_name;

INSERT INTO station_popularity
SELECT Max(to_station_id) as station_id, to_station_name as station_name, count(to_station_name) as popularity
FROM trips
GROUP BY station_name;

CREATE TABLE partition_stations
(
    id INT,
    name VARCHAR(100),
    popularity integer,
    row_number integer
);

INSERT INTO partition_stations
SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY popularity DESC)
FROM station_popularity;

DELETE FROM partition_stations WHERE row_number > 1;

INSERT INTO stations
SELECT id, name FROM partition_stations WHERE id IS NOT NULL;

DROP TABLE station_popularity, partition_stations;
```

2. Should we add any indexes to the stations table, why or why not?

### Question 2 Answer:

No - the station table is a relatively small data set (with 359 rows and 2 columns for each row), which is unlikely to benefit from an index.

3. Fill in the missing data in the `trips` table based on the work you did above

### Question 3 Answer:

```sql
UPDATE trips
SET from_station_id = stations.id
FROM stations WHERE trips.from_station_name = stations.name;

UPDATE trips
SET to_station_id = stations.id
FROM stations WHERE trips.to_station_name = stations.name;
```

## Part 2: Missing Date Data

You may have noticed that we have the dates as strings. This is because the dataset we have uses an inconsistent format for dates 😔😔😔

Note that the `original_filename` column is broken down into quarters, knowing this is helpful here!

There's an inconsistency in the dates in the string columns. For example trip `4461217` has a `start_time_str` of `11/25/2018 15:47` which implies that it's Month / Day / Year, since there's no 25th month, while trip `928659` has a `start_time_str` of `22/04/2017 0:04` which implies Day / Month / Year, since there's not 22nd month!

We can assume that each `original_filename` has dates in the same format.

1. What's the inconsistency in date formats? Which files are which?

### Question 1 Answer:

I run the sql below to find out the date format inconsistency

```sql
SELECT
array_agg(distinct substring(start_time_str, 1,2)),
array_agg(distinct substring(end_time_str, 1,2)),
original_filename
from trips
GROUP BY original_filename
```

From this result we can find that:

1. naming inconsistency in original file names
2. date format inconsistency for both start_time_str and end_time_str. Some of the time string has the format `MM/DD/YYYY`, and some of them have the date format `DD/MM/YYYY`.
3. based on the original_filename, `start_time_str` and `end_time_str` associated with Bikeshare Ridership(2017 Q1) and Bikeshare Ridership(2017 Q2).csv have inconsistent date format (most of the other date strings follow the date format `MM/DD/YYYY`)

| start_time  |            original_filename             |
| ----------- | :--------------------------------------: |
| {1/,10,11 } |     Bikeshare Ridership(2017 Q1).csv     |
| {1/,10,11 } |     Bikeshare Ridership(2017 Q2).csv     |
| {7/,8/,9/}  |     Bikeshare Ridership(2017 Q3).csv     |
| {10,11,12}  |     Bikeshare Ridership(2017 Q4).csv     |
| {1/,2/,3/}  |   Bike Share Toronto Ridership_Q1 2018   |
| {4/,5/,6/}  | Bike Share Toronto Ridership_Q2 2018.csv |
| {7/,8/,9/}  | Bike Share Toronto Ridership_Q3 2018.csv |
| {10,11,12}  | Bike Share Toronto Ridership_Q4 2018.csv |

As the result of the finding, I fixed the inconsistent date format of time strings

```sql
UPDATE trips
SET start_time_str = to_char(
date_trunc('second',
to_timestamp(start_time_str, 'DD/MM/YYYY HH24:MI:SS')),
'MM/DD/YYYY HH24:MI')
WHERE original_filename like '%2017 Q1%' or original_filename like '%2017 Q2%'
```

2. Take a look at Postgres's [date functions](https://www.postgresql.org/docs/12/functions-datetime.html), and fill in the missing date data using proper timestamps. You may have to write several queries to do this.

### Question 2 Answer:

With the fix above, I can fill in the missing date data by executing the following sql

```sql
UPDATE trips
SET start_time = start_time_str::timestamp
WHERE start_time_str NOT LIKE '%NULL%'

UPDATE trips
SET end_time = end_time_str::timestamp
WHERE end_time_str NOT LIKE '%NULL%'
```

3. Other than the index in class, would we benefit from any other indexes on this table? Why or why not?

### Question 3 Answer:

Yes, the trips table is a large database, each search sql takes a long time to execute. Adding index for this table makes searching in table faster.

## Part 3: Data-driven insights

Using the table you made in part 1 and the dates you added in part 2, let's answer some questions about the bike share data

1. Build a mini-report that does a breakdown of number of trips by month

### Question 1 Answer:

Run the following sql

```sql
SELECT EXTRACT(MONTH FROM start_time) as month, count(*) as number_of_trips
FROM trips
GROUP BY month
```

We get this report

| month | number_of_trips |
| ----- | :-------------: |
| 1     |      85155      |
| 2     |      91350      |
| 3     |     134172      |
| 4     |     173791      |
| 5     |     317373      |
| 6     |     400394      |
| 7     |     500019      |
| 8     |     504696      |
| 9     |     481362      |
| 10    |     361550      |
| 11    |     222996      |
| 12    |     142465      |

2. Build a mini-report that does a breakdown of number trips by time of day of their start and end times
3. What are the most popular stations to bike to in the summer?

### Question 2 Answer:

York St / Queens Quay W

```sql
SELECT from_station_name, count(from_station_name)
FROM trips
WHERE EXTRACT(MONTH FROM end_time) BETWEEN 6 AND 8
GROUP BY from_station_name
ORDER BY count DESC
LIMIT 1
```

Result

| station_name            | count |
| ----------------------- | :---: |
| York St / Queens Quay W | 21965 |

4. What are the most popular stations to bike from in the winter?

### Question 3 Answer:

Union Station

```sql
SELECT from_station_name, count(from_station_name)
FROM trips
WHERE EXTRACT(MONTH FROM end_time) = 12 OR EXTRACT(MONTH FROM end_time) = 1 OR EXTRACT(MONTH FROM end_time) = 2
GROUP BY from_station_name
ORDER BY count DESC
LIMIT 1
```

Result

| station_name  | count |
| ------------- | :---: |
| Union Station | 4515  |

5. Come up with a question that's interesting to you about this data that hasn't been asked and answer it.
