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

Select the most recent 5 trades for Amazon in order
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
ORDER BY symbol;
```
Aggregate by an arbitrary length of time
```sql
SELECT
  time_bucket('1 day', time) AS bucket, symbol, avg(price)
FROM stocks_real_time srt
WHERE time > (make_timestamp(2022, 12, 3, 0, 0, 0) - INTERVAL '1 week')
GROUP BY bucket, symbol
ORDER BY bucket, symbol;
```

**Step 7:** Create a continuous aggregate  

Query high, low, open, close price (without continuous aggregate)
```sql
SELECT
  time_bucket('1 day', "time") AS day,
  symbol,
  max(price) AS high,
  first(price, time) AS open,
  last(price, time) AS close,
  min(price) AS low
FROM stocks_real_time srt
GROUP BY day, symbol
ORDER BY day DESC, symbol;
```

Query high, low, open, close price (with continuous aggregate)
```sql
CREATE MATERIALIZED VIEW stock_candlestick_daily
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 day', "time") AS day,
  symbol,
  max(price) AS high,
  first(price, time) AS open,
  last(price, time) AS close,
  min(price) AS low
FROM stocks_real_time srt
GROUP BY day, symbol;
```
```sql
SELECT * FROM stock_candlestick_daily
  ORDER BY day DESC, symbol;
```

Create a continuous aggregate refresh policy  

(Automatic)  

```sql
SELECT add_continuous_aggregate_policy('stock_candlestick_daily',
  start_offset => INTERVAL '3 days',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 days');
```
(Manual)  
```sql
CALL refresh_continuous_aggregate(
  'stock_candlestick_daily',
  now() - INTERVAL '1 week',
  now()
);
```
**Step 8:** Save space with compression  

Enable TimescaleDB compression on the hypertable
```sql
ALTER TABLE stocks_real_time SET (
  timescaledb.compress,
  timescaledb.compress_orderby = 'time DESC',
  timescaledb.compress_segmentby = 'symbol'
);
```

Verify the compression settings
```sql
SELECT * FROM timescaledb_information.compression_settings;
```

Automatic compression
```sql
SELECT add_compression_policy('stocks_real_time', INTERVAL '2 weeks');
```

Verify your compression
```sql
SELECT pg_size_pretty(before_compression_total_bytes) as "before compression",
  pg_size_pretty(after_compression_total_bytes) as "after compression"
  FROM hypertable_compression_stats('stocks_real_time');
```

**Step 9:** Learn about data retention  

Create an automated data retention policy 
```sql
SELECT add_retention_policy('stocks_real_time', INTERVAL '3 weeks');
```

See information about retention policy and others
```sql
SELECT * FROM timescaledb_information.jobs;
```

## Section 2: Tutorials (NYC taxi)
