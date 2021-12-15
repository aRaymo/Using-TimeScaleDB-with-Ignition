# Using-TimeScaleDB-with-Ignition

I wrote this document over a year ago, so some things may have changed with timescale and you will have to adjust accordingly.

## Install Postgresql

1. [Download Postgres](https://www.postgresql.org/download/)
2. Run the install file.

## Install TimescaleDB

TimescaleDB is an extension for PostgreSQL database.  Follow the installation guide bellow for getting the extension installed on your computer.

[Installation Guide and Download](https://docs.timescale.com/latest/getting-started/)

1. Download TimescaleDB installer zip file and extract the `timescaledb` folder to the desktop.

2. Add Postgres file path to system environment variables.

   1. Search Environment Variables and click `Edit the system environment variables`

   2. When the system properties windows comes up, make sure it's on the `Advanced` tab  and click the `Environment Variables` button.

   3. Click on the `Path` variable in the System variables table and click the `Edit...` button.

   4. Double-click the next empty row in the table and paste in the path to the bin folder in the PostgreSQL installation folder.

      Example path:

      > C:\Program Files\PostgreSQL\11\bin

   5. Click `OK`.

3. Stop the PostgreSQL service.

4. Right click on `setup.exe` within the extracted timescaledb folder and click **Run as administrator**. A command prompt will appear.

5. Press `y` to tune the PostgresDB installation

6. When prompted, paste in the path to the data folder in the PostgreSQL installation folder and hit `enter`

   Example path:

   > C:\Program Files\PostgreSQL\11\data

7. Continue to type `y` and then `enter` until no longer prompted to do so. 

   * If the installation is not successful due to an "access denied" error, make sure you ran the setup.exe as administrator.

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
	create_hypertable('sqlth_1_data','t_stamp', if_not_exists => True, chunk_time_interval => 86400000, migrate_data => True);
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

