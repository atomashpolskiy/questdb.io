---
id: gettestdata
title: Obtaining Test Data
sidebar_label: Obtaining Test Data
---

## Sample data files
We encourage you to test QuestDB with your own data. If, however, you do not have data available
and would like to test QuestDB with test data, you can use some of the following test files:

- New York Taxi Data **[download](https://data.cityofnewyork.us/Transportation/2018-Yellow-Taxi-Trip-Data/t29m-gskq)**
- Movielens project **[download](https://grouplens.org/datasets/movielens/)**

Simply download & extract the file. We recommend you do this in the `/tmp/` folder.

## Generating data

### Overview
You can generate sequences of data for testing. You can either use it to `SELECT` or to `CREATE TABLE`.

### Example

The following will generate a series of timestamps "ts" and random doubles as "asks"
```sql
SELECT timestamp_sequence(to_timestamp('2019-10-17T00:00:00', 'yyyy-MM-ddTHH:mm:ss'), 100000L) ts,
 rnd_double() ask 
FROM long_sequence(100);
```

In addition, you can use the above to `CREATE TABLE` on the fly:
```sql
CREATE TABLE asks AS
(
SELECT timestamp_sequence(to_timestamp('2019-10-17T00:00:00', 'yyyy-MM-ddTHH:mm:ss'), 100000L) ts,
rnd_double() ask 
FROM long_sequence(100)
) 
TIMESTAMP(ts);
```

> Note that the query of the first example can be used within a `CREATE TABLE` statement.
>In addition, we use `TIMESTAMP()` to designate `ts` as the timestamp column in this new table which
>will be used for partitioning and time-series related queries (ASOF joins, SPLICE joins etc)
>

Other example:
```sql
CREATE TABLE test AS (SELECT rnd_str(2,2,0) name, rnd_double() reading FROM long_sequence(100));
```
