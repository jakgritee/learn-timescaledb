# Learn timescaledb-pgadmin4-grafana
The purpose of this repository is to learn how to work with time-series data in timescaledb and explore their features using CLI, pgadmin4, and grafana

## Section 1: Getting started (stock)
```
docker-compose --env-file config/.env up -d
```
**Step 1:** Connect to postgres and create database  

```
psql -U jakgrits -h localhost -d postgres
```
```
CREATE DATABASE stock;
```
```
\c stock
```
Note: How to SET timezone
```
SHOW timezone;
```
```
SET timezone = 'Asia/Bangkok';
```
```
SET timezone = 'Etc/UTC';
```
```sql
SELECT now();
```
**Step 2:** Create a regular PostgreSQL table  

```sql
CREATE TABLE stocks_real_time(
time TIMESTAMPTZ NOT NULL,
symbol TEXT NOT NULL,
price DOUBLE PRECISION NULL,
day_volume INT NULL
);
```
```sql
CREATE TABLE company(
symbol TEXT NOT NULL,
name TEXT NOT NULL
);
```

**Step 3:** Convert the regular table into a hypertable partitioned on the `time` column  

```sql
SELECT create_hypertable('stocks_real_time', 'time');
```

**Step 4:** Create an index to support efficient queries on the `symbol` and `time` columns  

```sql
CREATE INDEX ix_symbol_time ON stocks_real_time (symbol, time DESC);
```

**Step 5:** Add time-series data  

```sql
COPY stocks_real_time FROM '/var/lib/postgresql/data/stock/tutorial_sample_tick.csv' DELIMITER ',' CSV HEADER;
```
```sql
COPY company FROM '/var/lib/postgresql/data/stock/tutorial_sample_company.csv' DELIMITER ',' CSV HEADER;
```

**Step 6:** Query your data  

Select the most recent 10 trades for Amazon in order
```sql
SELECT * FROM stocks_real_time srt
WHERE symbol='MSFT'
ORDER BY time DESC, price DESC
LIMIT 5;
```
Calculate the average trade price for Apple from the last four days
```sql
SELECT
    avg(price)
FROM stocks_real_time srt
JOIN company c ON c.symbol = srt.symbol
WHERE c.name = 'Apple' AND time > (make_timestamp(2022, 12, 3, 0, 0, 0) - INTERVAL '4 days');
```
Get the first and last value
```sql
SELECT symbol, first(price, time), last(price, time)
FROM stocks_real_time srt
WHERE time > (make_timestamp(2022, 12, 3, 0, 0, 0) - INTERVAL '4 days')
GROUP BY symbol
ORDER BY symbol
LIMIT 5;
```
Aggregate by an arbitrary length of time
```sql
SELECT
  time_bucket('1 day', time) AS bucket, symbol, avg(price)
FROM stocks_real_time srt
WHERE time > (make_timestamp(2022, 12, 3, 0, 0, 0) - INTERVAL '1 week')
GROUP BY bucket, symbol
ORDER BY bucket, symbol
LIMIT 5;
```

## Section 2: Tutorials (NYC taxi)
```sql
```



