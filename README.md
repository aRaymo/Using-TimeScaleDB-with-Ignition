# Using-TimeScaleDB-with-Ignition

How to store Ignition tag history in PostgreSQL with the TimescaleDB extension, so that TimescaleDB handles partitioning, compression, and retention instead of Ignition.

_Last updated October 2026 for:_

| Software | Versions covered | Notes |
| --- | --- | --- |
| Ignition | 8.1 and 8.3 | Uses the built-in SQL historian (called a **SQL Historian** in 8.3 and a **Datasource History Provider** in 8.1). |
| PostgreSQL | 16, 17, or 18 | TimescaleDB 2.29 and later no longer support PostgreSQL 15 or older. |
| TimescaleDB | 2.30.x | Use the free **Community** edition. Columnstore (compression) and retention policies are not in the Apache-only build. |

The SQL in this guide was checked against TimescaleDB 2.30.2 on PostgreSQL 16.

> The company behind TimescaleDB, formerly Timescale, is now **Tiger Data**. Its docs moved to [tigerdata.com/docs](https://www.tigerdata.com/docs/), and its hosted service is now called Tiger Cloud. The extension is still called TimescaleDB.

If you followed the 2021 version of this guide, several of its commands no longer work. See [Upgrading from the 2021 version of this guide](#upgrading-from-the-2021-version-of-this-guide).

## Install Postgresql

1. [Download Postgres](https://www.postgresql.org/download/windows/). Pick **16, 17, or 18**, because TimescaleDB only ships Windows builds for those versions. Don't pick a newer major version than TimescaleDB supports. For example, PostgreSQL 19 isn't supported as of TimescaleDB 2.30.
2. Run the install file. In the **Select Components** step, keep **Command Line Tools** checked.

## Install TimescaleDB

TimescaleDB is an extension for PostgreSQL. The steps below are for Windows. For other platforms, see [Other ways to run TimescaleDB](#other-ways-to-run-timescaledb).

[Official Windows installation guide](https://www.tigerdata.com/docs/self-hosted/latest/install/installation-windows)

1. Install the prerequisites:
   * [Visual C++ Redistributable](https://aka.ms/vs/17/release/vc_redist.x64.exe)
   * OpenSSL 3.x (required by TimescaleDB 2.11.2 and later)

2. Go to the [TimescaleDB releases page](https://github.com/timescale/timescaledb/releases) and download the zip that matches your PostgreSQL major version, for example `timescaledb-postgresql-18-windows-amd64.zip`. Extract the `timescaledb` folder to the desktop.

3. Add the Postgres bin folder to the system `Path` environment variable. The installer uses `pg_config` to find your PostgreSQL install.

   1. Search Environment Variables and click `Edit the system environment variables`.

   2. When the System Properties window comes up, make sure it's on the `Advanced` tab and click the `Environment Variables` button.

   3. Click on the `Path` variable in the System variables table and click the `Edit...` button.

   4. Click `New` and paste in the path to the bin folder in the PostgreSQL installation folder.

      Example path:

      > C:\Program Files\PostgreSQL\18\bin

   5. Click `OK`.

4. Stop the PostgreSQL service, either from the Services app or with `Stop-Service postgresql-x64-18` in an administrator PowerShell window. This step is required when you upgrade an existing TimescaleDB install, because Windows locks the DLLs that are loaded.

5. Right-click `setup.exe` in the extracted timescaledb folder and click **Run as administrator**. A command prompt will appear.

6. Press `y` to tune the PostgreSQL installation with `timescaledb-tune`. This step also adds `timescaledb` to `shared_preload_libraries` in `postgresql.conf`.

7. When prompted, paste in the path to the data folder in the PostgreSQL installation folder (the folder that contains `postgresql.conf`) and hit `enter`.

   Example path:

   > C:\Program Files\PostgreSQL\18\data

8. Continue to type `y` and then `enter` until you are no longer prompted.

   * If the installation fails with an "access denied" error, make sure you ran `setup.exe` as administrator.

9. Start the PostgreSQL service again. From an administrator PowerShell window, run the following command, changing `18` to your version:

   ```powershell
   Start-Service postgresql-x64-18
   ```

### Other ways to run TimescaleDB

* **Linux:** Install the `timescaledb-2-postgresql-<version>` package from Tiger Data's package repository, then run `sudo timescaledb-tune`. See the [Linux install guide](https://www.tigerdata.com/docs/self-hosted/latest/install/installation-linux).
* **Docker:** For example:

  ```bash
  docker run -d --name timescaledb -p 5432:5432 \
    -v /path/on/host:/pgdata -e PGDATA=/pgdata \
    -e POSTGRES_PASSWORD=change_me timescale/timescaledb:latest-pg18
  ```

  Mount a volume so that your history survives the container being re-created. See the [Docker install guide](https://www.tigerdata.com/docs/self-hosted/latest/install/installation-docker), which also covers the larger `timescale/timescaledb-ha` image.
* **Managed/cloud:** Tiger Cloud runs TimescaleDB for you. Azure Database for PostgreSQL only offers the Apache-licensed TimescaleDB build, so columnstore and retention policies aren't available there, but `drop_chunks` still works. Amazon RDS doesn't offer TimescaleDB.

## Setup TimescaleDB for Ignition

Now we need to go into Postgres and set up TimescaleDB for use with the Ignition tables. Use a database editor to run the following SQL commands. pgAdmin and `psql` both come with Postgres and can be used.

### Create the database and Ignition user

Connect as the `postgres` superuser and create a login for Ignition and a database owned by that login:

```sql
CREATE ROLE ignition LOGIN PASSWORD 'change_me';
CREATE DATABASE ignition OWNER ignition;
```

Make the Ignition user the **owner** of the database. Since PostgreSQL 15, ordinary users can no longer create tables in the `public` schema of a database they don't own. If the Ignition user isn't the owner, Ignition can't create its historian tables.

You can also use an existing Ignition Postgres database. Run the rest of the commands while connected to that database.

### Load the extension into the database

Still connected as `postgres`, but now in the `ignition` database, run:

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

Check that the extension loaded:

```sql
SELECT extversion FROM pg_extension WHERE extname = 'timescaledb';
```

Optional: TimescaleDB sends anonymous telemetry by default. To turn it off, which is common on plant networks, run the following command:

```sql
ALTER SYSTEM SET timescaledb.telemetry_level = 'off';
SELECT pg_reload_conf();
```

## Setup Ignition

### Connect Ignition to the database

Ignition connects to TimescaleDB with its normal PostgreSQL JDBC driver.

* **Ignition 8.3:** In the Gateway webpage, go to **Connections > Databases > Connections** and click **Create Database Connection**. Then choose the PostgreSQL JDBC driver. In 8.3, JDBC drivers are installed as modules, so make sure the PostgreSQL driver module is enabled.
* **Ignition 8.1:** In the Gateway webpage, go to **Config > Databases > Connections** and click **Create new Database Connection...**. Then choose PostgreSQL.

Use a connect URL like `jdbc:postgresql://localhost:5432/ignition`, along with the `ignition` user and password created above.

See the Ignition docs for more detail: [8.3](https://docs.inductiveautomation.com/docs/8.3/platform/database-connections/connecting-to-databases/connecting-to-postgresql) / [8.1](https://docs.inductiveautomation.com/docs/8.1/platform/database-connections/connecting-to-databases/connecting-to-postgresql).

### Configure the historian: turn off Ignition partitioning and pruning

**This is the most important Ignition setting.** By default, Ignition splits history into monthly tables named `sqlt_data_1_2026_10` and so on, and it can prune old partitions. TimescaleDB does both jobs itself with chunks and retention policies. Running both partitioning schemes on the same data causes problems.

When partitioning is disabled, Ignition writes all history to a single table named `sqlth_1_data`, and that table is what we turn into a hypertable.

* **Ignition 8.3:** Go to **Services > Historians > Historians**. Click **Create Historian** and choose **SQL Historian**, or edit an existing SQL Historian. Set **Data Source** to the connection above.
* **Ignition 8.1:** Go to **Config > Tags > History** and edit the Datasource History Provider that Ignition created for the connection.

Then set:

| Setting | Value |
| --- | --- |
| Enable Partitioning | **Off** |
| Enable Pre-processed Partitions | **Off** |
| Enable Data Pruning | **Off**, because TimescaleDB's retention policy handles it |

Turning off partitioning on a historian that already has data does **not** move the existing `sqlt_data_*` tables into `sqlth_1_data`. It's easiest to start with a new historian and database. If you need the old data, plan a migration and test it on a non-production gateway first.

See [History Providers (8.3)](https://docs.inductiveautomation.com/docs/8.3/ignition-modules/tag-historian/tag-history-providers) / [Tag History Providers (8.1)](https://docs.inductiveautomation.com/docs/8.1/ignition-modules/tag-historian/tag-history-providers).

### Let Ignition create its tables

Ignition creates the historian tables itself. Point at least one tag's history at the new historian and let it store some values. Then check that the table exists:

```sql
SELECT to_regclass('sqlth_1_data');
```

If this command returns nothing, and you see tables like `sqlt_data_1_2026_10` instead, partitioning is still turned on.

Do the conversion below as early as you can, ideally while the table is still close to empty.

## Convert sqlth_1_data to a hypertable

### Create the hypertable

`t_stamp` is a `BIGINT` that holds milliseconds since the Unix epoch. So all of the time intervals below are in **milliseconds**. `86400000` is one day.

```sql
SELECT create_hypertable(
    'sqlth_1_data',
    by_range('t_stamp', 86400000),  -- chunk interval in ms: 1 day (see "Choosing the chunk interval")
    if_not_exists => TRUE,
    migrate_data => TRUE
);
```

* TimescaleDB prints warnings that `character varying` and `timestamp without time zone` columns "do not follow best practices". These warnings are expected and harmless. Don't change Ignition's column types.
* If the table already holds a lot of data, `migrate_data` copies it into chunks and locks the table until the copy finishes. Ignition's store-and-forward buffers history while the table is locked, but run the conversion during a quiet period.
* **The chunk interval.** `86400000` (1 day) suits most Ignition systems: up to about 20 million stored values a day, kept for up to about a year. If you store more than that, or keep raw history for years, read [Choosing the chunk interval](#choosing-the-chunk-interval) before you run this. You can change the interval later, but only new chunks use the new value.
* **Always pass an interval.** If you leave it out, TimescaleDB doesn't raise an error on Ignition's `BIGINT` column. It silently uses 1,000,000 ms (about 17-minute chunks), which is about 2,600 chunks for 30 days of history. `INTERVAL '1 day'` is rejected, so give the value in milliseconds.

### Set the integer "now" function

Because `t_stamp` is an integer, TimescaleDB needs a function that returns the current time in the same units. The columnstore and retention policies use this function to work out how old each chunk is.

```sql
CREATE OR REPLACE FUNCTION unix_now() RETURNS BIGINT LANGUAGE SQL STABLE AS $$ SELECT (extract(epoch FROM now()) * 1000)::BIGINT $$;

SELECT set_integer_now_func('sqlth_1_data', 'unix_now');
```

### Enable the columnstore (compression)

TimescaleDB 2.18 renamed compression to the **columnstore** (part of what Tiger Data calls "hypercore"). Segmenting by `tagid` and ordering by `t_stamp` matches the way Ignition queries history.

```sql
ALTER TABLE sqlth_1_data SET (
    timescaledb.enable_columnstore = true,
    timescaledb.segmentby = 'tagid',
    timescaledb.orderby = 't_stamp DESC'
);
```

Add a policy that converts chunks to the columnstore once they're older than 7 days:

```sql
CALL add_columnstore_policy('sqlth_1_data', after => BIGINT '604800000');  -- 7 days; keep this >= the chunk interval
```

* `add_columnstore_policy` is a procedure, so it's run with `CALL`, not `SELECT`.
* Pass the age as an integer number of milliseconds. `INTERVAL '7 days'` fails on Ignition's `BIGINT` `t_stamp`, with the error `invalid value for parameter compress_after`.
* Late data still works. When store-and-forward flushes a backlog or you backfill history, the inserts go into compressed chunks without a problem. Updates and deletes on compressed chunks also work.
* Choose an `after` value that's longer than the time window your trends and dashboards query most often, so the most-queried data stays in the faster row format.
* If the table already had data when you converted it, the first run of this policy may log warnings that some chunks are "already compressed" (seen on 2.30.2). As long as `timescaledb_information.job_errors` is empty and the older chunks show as compressed (see [Verify it's working](#verify-its-working)), nothing is wrong.

### Add a retention policy

Retention policies are free in the Community edition. The Enterprise edition is no longer required.

```sql
SELECT add_retention_policy('sqlth_1_data', drop_after => BIGINT '2592000000');  -- 30 days; whole chunks are dropped, so expect 30-31 days with 1-day chunks
```

Make `drop_after` longer than the columnstore `after` value. Some handy values in milliseconds:

| Period | Milliseconds |
| --- | --- |
| 1 hour | `3600000` |
| 1 day | `86400000` |
| 7 days | `604800000` |
| 30 days | `2592000000` |
| 90 days | `7776000000` |
| 365 days | `31536000000` |

If you can't use policies, for example on an Apache-only build, schedule this query yourself to get the same result:

```sql
SELECT drop_chunks('sqlth_1_data', older_than => unix_now() - BIGINT '2592000000');  -- 30 days
```

With an integer time column, `older_than` is an absolute `t_stamp` cutoff, which is why the query subtracts from `unix_now()`.

## Verify it's working

Check the hypertable and its policies:

```sql
SELECT hypertable_name, num_chunks, compression_enabled
FROM timescaledb_information.hypertables;

SELECT job_id, proc_name, schedule_interval, config, next_start
FROM timescaledb_information.jobs
WHERE hypertable_name = 'sqlth_1_data';
```

To run a policy now instead of waiting for its schedule, run `CALL run_job(<job_id>);`. To check for failed jobs, query `timescaledb_information.job_errors`.

Check how many chunks have been converted to the columnstore:

```sql
SELECT is_compressed, count(*)
FROM timescaledb_information.chunks
WHERE hypertable_name = 'sqlth_1_data'
GROUP BY is_compressed;
```

See how much space the columnstore is saving:

```sql
SELECT pg_size_pretty(before_compression_total_bytes) AS before,
       pg_size_pretty(after_compression_total_bytes)  AS after
FROM hypertable_columnstore_stats('sqlth_1_data');
```

## Choosing the chunk interval

The chunk interval is how much time each chunk covers. It's the `86400000` (1 day) in `by_range('t_stamp', 86400000)`. One day is a good default for most Ignition systems. The right value depends on two things: how many values Ignition stores per day, and how long you keep raw history.

### Quick guide

| Your system | Stored values per day | Raw history kept | Chunk interval | Value in ms |
| --- | --- | --- | --- | --- |
| Most systems | Up to ~20 million (~230/s) | Up to ~1 year | 1 day | `86400000` |
| Low rate, long history | Up to ~3 million (~35/s) | Years (up to ~10) | 7 days | `604800000` |
| Busy | ~20–100 million (~230–1,150/s) | Up to a few months | 6–12 hours | `21600000` to `43200000` |
| Very busy | Over ~100 million (1,150/s and up) | Weeks | 1–3 hours | `3600000` to `10800000` |

At the top of each row's rate range, PostgreSQL needs about 5–6 GB of RAM to follow the memory rule below. If you need years of raw history at a high rate, the rows above conflict. In that case, keep raw data for less time and use [continuous aggregates](#optional-continuous-aggregates-for-long-term-trends) for long-term trends, or give PostgreSQL more RAM and use a longer interval.

Examples, using Ignition's default 10-second storage rate:

* 1,000 tags with 10% changing per check is about 10 values/s, or 0.9 million a day. Use 1 day, or 7 days if you keep years of history.
* 10,000 tags with 10% changing is about 100 values/s, or 8.6 million a day. Use 1 day.
* 50,000 tags at a 1-second rate with 20% changing is about 10,000 values/s, or 860 million a day. Use 1 hour. That's about 36 million rows per chunk, so PostgreSQL needs about 9 GB of RAM.

### What the interval trades off

**Not too big: the newest chunk has to fit in memory.** Every insert updates the newest chunk's indexes. Tiger Data's [sizing rule](https://www.tigerdata.com/docs/learn/hypertables/sizing-hypertable-chunks) is that the indexes of the chunks being written should fit in 25% of main memory, which is what `timescaledb-tune` sets `shared_buffers` to. Uncompressed, one Ignition history row takes about 120 bytes, and roughly half of that is index. (Inductive Automation's own estimate is about 100 bytes.) So:

* newest chunk ≈ values per day × chunk length in days × 120 bytes
* its indexes ≈ half of that
* longest interval in days ≈ 25% of PostgreSQL's RAM ÷ (values per day × 60 bytes)

For example, 8.6 million values a day is about 1 GB per 1-day chunk, with about 0.5 GB of indexes. That fits easily on a server that gives PostgreSQL 4 GB or more. If Ignition and PostgreSQL share a machine, count only the memory PostgreSQL gets.

**Not too small, part 1: keep the total number of chunks low.** The number of chunks is your retention divided by the interval. Ignition's seed-value query has no lower time bound (see [Known issues and tips](#known-issues-and-tips)). So PostgreSQL plans **every** chunk in the table each time a chart loads, even when the tag has fresh data. The new last-value optimization in TimescaleDB 2.30 doesn't apply to Ignition's query, because the query filters on `t_stamp`. Here are measured planning times for that query:

| Chunks in the table | Example | Planning time per seed query (uncompressed / compressed) |
| --- | --- | --- |
| 31 | 1-day chunks, 30 days | ~1 ms / ~2 ms |
| 366 | 1-day chunks, 1 year | ~14 ms / ~46 ms |
| ~2,200 | 4-hour chunks, 1 year | ~0.5 s / ~2.6 s |
| 8,760 | 1-hour chunks, 1 year | Fails with `out of shared memory` on PostgreSQL's default lock settings |

These numbers come from a small test server running TimescaleDB 2.30.2 and PostgreSQL 16, with synthetic Ignition-style data. Your times will differ, but the trend holds.

Each chunk also takes about 3 locks per query, or about 5 once it's compressed. Every pooled Ignition connection also caches metadata for every chunk: about 88 MB per connection at ~2,200 compressed chunks. Tiger Data calls more than 1,000 chunks in one hypertable an anti-pattern. Aim to stay under about 500: interval ≥ retention ÷ 500. For example, keep 1 day for up to a year of history, and use 7 days for multi-year history.

`timescaledb-tune` raises `max_locks_per_transaction` from PostgreSQL's default of 64 to 256 on an 8 GB machine, 512 on 16 GB, and 1024 on 32 GB. That helps, but it doesn't fix the planning time. Check your value with `SHOW max_locks_per_transaction;`.

**Not too small, part 2: slow-changing tags need enough rows per chunk to compress well.** The columnstore compresses each tag's rows within a chunk, in batches of up to 1,000 rows. A tag that stores only a few values per chunk ends up in tiny batches, which compress badly. In testing, compressed storage per value was:

* about 40 bytes at 10 values per tag per chunk
* about 9 bytes at 100
* about 6 bytes at 500 or more

For comparison, a value takes about 120 bytes uncompressed. A tag that logs once a minute took 2.7 times more compressed space with 1-hour chunks than with 1-day chunks. For the tags that make up most of your data, aim for at least 100 stored values per chunk, ideally 500 or more. In other words, the interval should be at least 100 times the typical time between stored values.

**Policies act on whole chunks.** A chunk is compressed only when its newest data is older than the columnstore `after` value. It's dropped only when its newest data is older than `drop_after`. So:

* With 1-day chunks, data is compressed when it's 7–8 days old and deleted when it's 30–31 days old.
* With 7-day chunks, the same settings mean 7–14 days and 30–37 days.

On Ignition's integer `t_stamp`, both jobs run once a day by default, which can add up to another day. Keep the chunk interval no longer than `after`, and small compared with your retention. Chunks are aligned to the Unix epoch, so 1-day chunks start at 00:00 UTC, not local midnight. 7-day chunks start on Thursdays at 00:00 UTC.

### Measure your system

Inductive Automation's estimate is **values per second ≈ tags × fraction changing per check ÷ storage rate in seconds**. Ignition stores a value only when it changes by more than the deadband, so the real rate is usually far below tags × scan rate. Max Time Between Samples sets a minimum rate.

To see the actual rate, check the Gateway's Store & Forward page and multiply by 86,400 to get values per day. That rate can include other writes on the same database connection, such as transaction groups.

* **8.3:** Platform > System > Store & Forward, under Store Rate and Forward Rate.
* **8.1:** Status > Connections > Store & Forward, under Aggregate Throughput.

If Ignition is still partitioning (before you switch), count the rows in one full monthly table and divide by the number of days in that month:

```sql
SELECT count(*) / 30 AS avg_rows_per_day FROM sqlt_data_1_2026_09;  -- September has 30 days
```

After the conversion, use these queries.

Average values per day over the last week:

```sql
SELECT count(*) / 7 AS avg_rows_per_day
FROM sqlth_1_data
WHERE t_stamp >= unix_now() - BIGINT '604800000';
```

Current chunk interval in milliseconds:

```sql
SELECT integer_interval FROM timescaledb_information.dimensions WHERE hypertable_name = 'sqlth_1_data';
```

Size of the most recent chunks. Compare the newest full chunk's `index_size` with 25% of PostgreSQL's RAM:

```sql
SELECT c.chunk_name,
       to_timestamp(c.range_start_integer / 1000.0) AS range_start,
       to_timestamp(c.range_end_integer / 1000.0)   AS range_end,
       c.is_compressed,
       pg_size_pretty(s.total_bytes) AS total_size,
       pg_size_pretty(s.index_bytes) AS index_size
FROM timescaledb_information.chunks c
JOIN chunks_detailed_size('sqlth_1_data') s
  ON s.chunk_schema = c.chunk_schema AND s.chunk_name = c.chunk_name
WHERE c.hypertable_name = 'sqlth_1_data'
ORDER BY c.range_start_integer DESC
LIMIT 10;
```

### Changing the interval later

```sql
SELECT set_chunk_time_interval('sqlth_1_data', BIGINT '604800000');  -- 7 days, for chunks created from now on
```

* Only new chunks use the new interval. Existing chunks keep their size, and the policies keep working across the mix.
* The first new chunk may be shorter than the new interval, so that it lines up with the existing chunks.
* If you already have far too many small chunks, either lengthen the interval and let retention age out the small ones, or combine adjacent chunks with [`merge_chunks`](https://www.tigerdata.com/docs/reference/timescaledb/hypertables/merge_chunks).
* Don't use the `timescaledb.compress_chunk_time_interval` option, which merges chunks during compression, on this table. On Ignition's millisecond column, its value is read as a PostgreSQL interval in microseconds. So `'7 days'` merges everything into one ever-growing chunk, and retention silently stops deleting data. The option is also marked experimental, and its merges can't be undone.

## Optional: continuous aggregates for long-term trends

A continuous aggregate keeps a rolled-up copy of the data, such as hourly averages, up to date automatically. You can keep the rollups much longer than the raw data.

```sql
CREATE MATERIALIZED VIEW sqlth_1_data_hourly
WITH (timescaledb.continuous) AS
SELECT tagid,
       time_bucket(BIGINT '3600000', t_stamp) AS bucket,  -- 1 hour
       avg(floatvalue) AS avg_value,
       min(floatvalue) AS min_value,
       max(floatvalue) AS max_value
FROM sqlth_1_data
GROUP BY tagid, bucket
WITH NO DATA;

SELECT add_continuous_aggregate_policy('sqlth_1_data_hourly',
    start_offset      => BIGINT '604800000',  -- 7 days
    end_offset        => BIGINT '3600000',    -- 1 hour
    schedule_interval => INTERVAL '1 hour');
```

Ignition's tag history bindings and `system.tag.queryTagHistory` don't read continuous aggregates. Query them with Named Queries or `system.db.runPrepQuery` instead, and join to `sqlth_te` to get tag paths.

## Ignition 8.3 notes

* **Historian options.** 8.3 replaced the Tag Historian module with the **Historian Core** module, which includes a built-in **Core Historian** powered by QuestDB that needs no external database, and the **SQL Historian** module, which this guide uses. If you don't specifically need your history in PostgreSQL, the Core Historian is worth a look.
* **Historian API.** 8.3 also exposes a Historian API for third-party historian modules. Some of these modules store data in TimescaleDB with their own schema and push aggregation down into the database. One example is the commercial [TimescaleDB historian module from Mustry Solutions](https://forum.inductiveautomation.com/t/new-module-timescaledb-historian-for-ignition-8-3/116295). This guide doesn't cover those modules. It uses Ignition's own SQL Historian tables.
* **Table schema.** The SQL Historian in 8.3 uses the same `sqlth_*` tables as 8.1, so the steps above are the same for both versions. See the [8.3 database table reference](https://docs.inductiveautomation.com/docs/8.3/appendix/reference-pages/ignition-database-table-reference).

## Known issues and tips

* **Seed-value queries have no lower time bound.** Charts and some history queries look up the last value before the start of the requested range with a query like `... WHERE tagid IN (?) AND t_stamp <= ? ORDER BY t_stamp DESC LIMIT 1`. This query is fast when the tag has recent data. For tags that rarely change, it can scan back through many chunks. A retention policy keeps the number of chunks bounded. See [this forum thread](https://forum.inductiveautomation.com/t/inefficient-queries-no-lower-time-bound-bug-feature-request/95260) for details and workarounds.
* **Don't turn Ignition partitioning or pruning back on** after you convert the table.
* **Back up before upgrades.** `pg_dump` works on TimescaleDB databases, but restoring one needs a few extra steps. See [Logical backup](https://www.tigerdata.com/docs/self-hosted/latest/backup-and-restore/logical-backup).

## Upgrading TimescaleDB

1. Download the new Windows zip for your PostgreSQL version, stop the PostgreSQL service, run `setup.exe` as administrator, then start the service again.
2. In **each** database that uses TimescaleDB, run the following update as the first command in a fresh session. The `-X` flag stops `psql` from loading your startup file.

   ```bash
   psql -X -U postgres -d ignition -c "ALTER EXTENSION timescaledb UPDATE;"
   ```

3. TimescaleDB 2.29 and later require PostgreSQL 16 or newer. If you're on PostgreSQL 15 or older, upgrade PostgreSQL first. The same TimescaleDB version must be installed for both the old and the new PostgreSQL version during `pg_upgrade`. See [Upgrade PostgreSQL](https://www.tigerdata.com/docs/self-hosted/latest/upgrades/upgrade-pg) and [Minor TimescaleDB upgrades](https://www.tigerdata.com/docs/self-hosted/latest/upgrades/minor-upgrade).

## Upgrading from the 2021 version of this guide

The 2021 guide was written for TimescaleDB 1.x. TimescaleDB 2.0 renamed several functions, and 2.13 and 2.18 introduced newer APIs. Here's what happens if you run the old commands on TimescaleDB 2.30:

| 2021 command | Result on TimescaleDB 2.30 | Use instead |
| --- | --- | --- |
| `create_hypertable('sqlth_1_data', 't_stamp', chunk_time_interval => ...)` | Still works (older API) | `create_hypertable('sqlth_1_data', by_range('t_stamp', 86400000))` |
| `ALTER TABLE ... SET (timescaledb.compress, timescaledb.compress_orderby, timescaledb.compress_segmentby)` | Still works (deprecated naming since 2.18) | `timescaledb.enable_columnstore`, `timescaledb.orderby`, `timescaledb.segmentby` |
| `add_compress_chunks_policy(...)` | **Error:** function does not exist (removed in 2.0) | `CALL add_columnstore_policy('sqlth_1_data', after => BIGINT '604800000')` |
| `add_drop_chunks_policy(...)` | **Error:** function does not exist (removed in 2.0) | `add_retention_policy('sqlth_1_data', drop_after => BIGINT '2592000000')` |
| `drop_chunks(CAST('2592000000' AS BIGINT), 'sqlth_1_data')` | **Error:** invalid hypertable (the argument order changed in 2.0) | `drop_chunks('sqlth_1_data', older_than => unix_now() - BIGINT '2592000000')` |
| `CREATE EXTENSION ... CASCADE` | Still works | `CASCADE` isn't needed |

If your database was set up with the old guide, your hypertable and compression settings carry over when you upgrade the extension. Check `timescaledb_information.jobs`. If a compression or retention job is missing, add it with the commands above. The `unix_now()` function and `set_integer_now_func` call from the old guide are still correct.
