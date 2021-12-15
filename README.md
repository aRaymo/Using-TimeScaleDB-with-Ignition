# Using-TimeScaleDB-with-Ignition

I wrote this document over a year ago, so some things may have changed with timescale and you will have to adjust accordingly.

## Install Postgresql

1. [Download Postgres](https://www.postgresql.org/download/)
2. Run the install file.

## Install TimescaleDB

TimescaleDB is an extension for PostgreSQL database.  Follow the installation guide bellow for getting the extension installed on your computer.

1. [Installation Guide and Download](https://docs.timescale.com/latest/getting-started/)

## Setup TimescaleDB for Ignition

Now we need to go into postgres and setup timescaledb for use with the ignition tables.  Use a database editor to run the following SQL commands.  pgAdmin comes with postgres and can be used.

Create a database for ignition to use or use your existing ignition postgres database.  Use this database when performing the following commands.

### Load the extension into the database.

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;
```

### Create the hyper table.

```plsql
SELECT
	* 
FROM 
	create_hypertable('sqlth_1_data','t_stamp', if_not_exists => True, 			chunk_time_interval => 86400000, migrate_data => True);
```

### Setup the hypertable 

Add compression to the table.

```plsql
ALTER TABLE sqlth_1_data SET (timescaledb.compress, timescaledb.compress_orderby = 't_stamp DESC', timescaledb.compress_segmentby = 'tagid'); 
```

Create a function that will help translate your time stamp columns format for TimescaleDB.

```plsql
CREATE OR REPLACE FUNCTION unix_now() returns BIGINT LANGUAGE SQL STABLE as $$ SELECT (extract(epoch from now())*1000)::bigint $$;
```

Set integer now function

```plsql
SELECT set_integer_now_func('sqlth_1_data', 'unix_now');
```

Add compress chunks policy

```plsql
SELECT add_compress_chunks_policy('sqlth_1_data', CAST ('604800000' AS INTEGER));
```

~~**IMPORTANT** If you are using the enterprise version of timescale, you can run the following command to automatically drop chunks older than the specified cutoff.~~
Enterprise version is no longer required.

```plsql
SELECT add_drop_chunks_policy('sqlth_1_data', CAST ('2592000000' AS BIGINT)); # 30 days
```

Otherwise, you must manually run the following query on a schedule to re-create the same functionality.

```plsql
SELECT drop_chunks(CAST ('2592000000' AS BIGINT),'sqlth_1_data'); #30 days
```

