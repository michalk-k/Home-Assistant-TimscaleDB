# Introduction

This article is the result of a deep dive into how to effectively store and visualize long-term data from Home Assistant. In the next sections, I'll explain why moving your data outside of Home Assistant can make a big difference and how you can do it.
For me, it was also a first real hands-on experience with TimescaleDB (even though I had known about it for years).

The article is structured as a step-by-step tutorial, starting with TimescaleDB basics and ending with powerful Grafana visualizations. You'll find ready-to-use examples you can copy and adapt to your own setup, depending on the sensors you have.

What's covered:

* Why it's worth adding TimescaleDB to your Home Assistant system
* What you need to set it up and how to configure it
* Basic concepts of TimescaleDB
* How to aggregate data efficiently
  * How to use Continuous Aggregates (CAGGs)
  * How to set up Real-time Aggregates
  * How to build Hierarchical CAGGs
* How to use data compression
* How to manage data retention
* How to create Grafana visualizations based on your data

## Why Not Rely Solely on Home Assistant for Long-Term Energy Data Storage?

While Home Assistant (HA) is a powerful automation platform, it's not ideally suited for long-term storage and analysis of electricity energy consumption data on its own. Here are a few disadvantages:

* Database limitations: By default, HA uses SQLite, which struggles with large datasets and concurrent access.
* Low reliability: SQLite introduces a high risk of data loss due to file corruption, crashes, or storage issues. It plays against storing very long-term data. Attempting recovery can be time-consuming and may result in extended Home Assistant outages.
* Performance issues: Complex queries against SQLite are virtually impossible.
* Limited analytics: HA’s built-in visualization and analytics capabilities are basic compared to specialized tools like Grafana.
* Inflexible data access: Data stored in HA's default database is difficult to manipulate or import from external sources. In contrast, PostgreSQL/TimescaleDB allows flexible modification and integration with third-party data, such as electricity provider exports, or recovery from backups if HA's internal storage fails.
* Refractory on entities replacement: In HA, it's difficult to maintain entity naming when making changes to the household systems, for example, installing solar panels (FVE). Tracking energy before and after such events results in fragmented or incomplete views without an option to visualise them as continuous data.

Note: Home Assistant's Long-Term Statistics (LTS) system is designed for infinite retention, but this applies only to specific statistics, not to the high-resolution state history used by the main recorder.

## Benefits of Using TimescaleDB

TimescaleDB, a time-series extension of PostgreSQL, offers a more robust solution for energy consumption data logging and analysis:

* Efficient time-series storage: Built specifically to handle large volumes of time-stamped data.
* Scalability: Handles months or years of high-resolution data with ease.
* Advanced queries: Supports SQL with time-series specific functions.
* Continuous Aggregates: Enables efficient pre-aggregation of data for fast queries.
* Flexible retention and compression: Easily drop or compress old data while preserving aggregate summaries (which can also be compressed)
* Changes to the data are up to your will. You can rename entities, import data from other systems. For example, you can import consumed energy data offered by the provider from time periods HA system did not measure house consumption, etc.
* On-the-fly data transformation like filtering, renaming, unit conversion, and more.

Bonus: Grafana can bring all this data to life with interactive dashboards right from the start.

All in one, with TimescaleDB, it’s possible to achieve energy analytics unavailable in HA, like the return value of FV installation collected over the years.

![|602x227](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdsORdqpQ79w_ul4NxFlKigsO4ZBhQlY0YYdkG_6epbxpn5dWOh3DA-kCZ8jGDQEyNhrUIIR5cG-gF6zBPbfAMXQ-34Lb2Z91lc70aqkEOuQC98tlqbclP9Ao6ry2mtA1XGMP5xbg?key=lxjGOMoRy8aXFNM_LTsRQxoa)

## Prerequisites

To get started, ensure the following components are installed and configured:

* Home Assistant sensors. Read below for the requirements about HA sensors
* Long-Term Statistics Store (LTSS) integration. Link: https://github.com/freol35241/ltss
* TimescaleDB Add-on running in your HA Supervisor environment or as an external service. Link: https://github.com/expaso/hassos-addon-timescaledb
* TimescaleDB Toolkit extension enabled.
* Periodic Energy sensors are provided by Home Assistant. At least hourly ones. Power sensors alone can be useful too, including the possibility of calculating energy out of them by TimescaleDB. But having energy sensors is more comfortable, and energy measurements provided directly by external devices like SmartMeters or inverters are considered more precise.

### Home Assistant Sensors

Before we move on, this topic deserves a bit of extra explanation.
LTSS doesn't publish long-term values. Only real-time values from original sensors, and only in the moment the sensor gets new value.
Home assistant records energy as continuously increasing values. It's allowed that these values are reset to zero. Then it increases again.
The energy might be reported by external devices or calculated from reported power.

The frequency of updates is not pre-determined, but depends mostly on source devices providing those values. It might be one sample per hour or 10 samples per second.

Our goal is to achieve precalculated energy values for specific periods, such as hourly, daily, or weekly totals.
In an ideal world, where we had infinitely dense data (measurements at every possible moment), we could calculate energy usage for any period at any time.
But with sparse data (only occasional measurements), it's technically impossible to do this accurately without interpolation.

Let's imagine this simple timeline of energy consumption measurements:

```
time ────o─────o─o─────o─────|───────o─────o─────o────o────>
                       x  midnight   y
```
Here:

* "o" represents a measurement point (sensor reading)
* x is the last reading before midnight
* y is the first reading after midnight

Suppose we want to calculate total energy usage for the day (midnight to midnight).
With data distributed as in the picture, it's impossible to precisely determine:

* the energy used between x and midnight
* the energy used between midnight and y

You could try to interpolate (estimate) these missing parts, but for our case (explained later), interpolation isn't the way to go.
Sometimes your energy readings will be frequent enough that the error is small, but you can’t always count on that.

How to solve this?

We can use Home Assistant's Utility Meters.
These sensors reset automatically at the start of each period (quarter-hourly, hourly, daily, monthly, etc.).
Based on my observations:

* The first recorded value for each period is always 0.
* The first value following the reset represents the energy collected since the last known value before zero-point.

In other words, the value at y would represent the delta (the amount of energy used) between x and y.
This method isn't 100% precise, but it helps avoid major issues like "energy leakage" between periods.

Worth mentioning that
* Values in datapoints are continuously increasing (until the next reset).
* Updates of utility sensors reflect the original sensor. So all publish the same values. 

It renders into conclusion that there is no need to publish daily sensors together with daily ones. The shortest time utility sensor is enough to cover our needs.

**Note:**
In this article, we assume hourly data as the baseline.
This is usually enough for most use cases, unless you're dealing with high-frequency needs like quarter-hourly data (for example, spot market energy trading).

### Consistent Units for Energy Sensors

Home Assistant can create energy sensors with different units — for example, Wh, kWh, or even MWh.
When you configure your utility meters, make sure all related sensors use the same unit.

If your sensors use mixed units, you have two options (but both are a bit messy):

* Convert the units when recording data into the database
* Convert the units later during data aggregation or calculations

Later in this article, you'll see that units are stored together with the data. However, unit conversion is not covered in the example queries — they assume everything is already consistent.

**My recommendation:**
Stick to `kWh` as your standard unit.
It’s the most common, easy to read, and fits well for hourly or daily energy tracking.

### Installing LTSS and TimescaleDB Add-on

1. Navigate to Home Assistant Add-on Store.
2. Search and install TimescaleDB.
3. Note the database name, username, password, and port — you’ll need them for LTSS.
4. Install LTSS integration via the HACS or YAML configuration.
5. Connect to your TimescaleDB instance and run:
```sql
CREATE EXTENSION IF NOT EXISTS timescaledb_toolkit;
CREATE EXTENSION IF NOT EXISTS btree_gist;
```
The timescaledb_toolkit enables additional time-series features, including advanced analytics. It's needed for work with continuously growing values, like energy.

The `btree_gist` will be needed later on for cost reports creation.

# TimescaleDB basics

This part of the article provides basic information about TimescaleDB objects, including information on how to perform some useful operations with them.

TimescaleDB is an extension (plug-in) to PostgreSQL database. It introduces a powerful row-columnar engine (hypercore), and the TimescaleDB features are built upon.

A Hypertable - it’s a new type of table, provided with TimescaleDB. It boasts of many, many features, which scope is beyond this article. For understanding what is coming next, there are the main features:

* automatic partitioning - in our case, we will partition by time
* optional compression
* optional data retention

Continuous Aggregates (CAGGs) are materialized views that maintain aggregated data over time. These views are automatically updated as new data arrives. They are useful for:

* Improving query performance by precomputing data (ie, daily totals)
* Enabling efficient storage via summarized data
* Allowing old data deletion from the source table
* CAGGs are based on hypertables, being able to make use of their features. Since data are stored in the table, data in CAGG might be modified manually using regular SELECT, UDPATE, DELETE statements

## Configuring LTSS to Export Energy Sensors

Once LTSS is installed, configure it to export all sensors you need to TimescaleDB. It's important to know that LTSS writes all data into a single table. So, at the end, you likely will publish not only energy data, but also temperatures and humidity, HA system metrics like CPU or Storage usage.

Configuration of LTSS typically involves selecting the appropriate sensors in the integration’s configuration in configuration.yaml, for example:

```yaml
ltss:
  db_url: postgresql://ltss_user:password@localhost/ltss_db
  include:
  domains:
    - sensor
  entities:
    - sensor.total_energy_consumption
    - sensor.solar_production
```
The configuration is similar to what the recorder offers, including globs. Unfortunately, it inherits the limitation that entities included by globs cannot be excluded anymore. In such a case, there are two options: replace globs with an explicit list of sensors, or let LTSS publish a broader set of data, to reject some entities with the use of a before trigger. While the second option seems tempting, it's easy to forget about such a trigger later on. Also it still costs additional communication and processing.

Anyway, here is an example of such a trigger:

```sql
CREATE OR REPLACE FUNCTION ltss_filter_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $code$
BEGIN
    -- Filter out unwanted entities before insertion
    IF NEW.entity_id NOT IN 
       (
           'sensor.unwanted_energy_consumption',
           'sensor.solar_production'
       )
    THEN
       RETURN NULL;  -- Skip insert
    END IF;
   
    RETURN NEW;
END;
$code$;


CREATE TRIGGER ltss_filter_trigger
BEFORE INSERT ON ltss
FOR EACH ROW
EXECUTE FUNCTION ltss_filter_trigger();
```

It's important to note that HA sensors often include extensive and unnecessary metadata in their attributes field. These attributes are stored as JSONB in the database, which can significantly increase disk usage. If the additional metadata is not essential for your use case, it's wise to strip it away to optimize storage. Below is a trigger that retains only the unit_of_measurement attribute, which is typically the most relevant, thereby minimizing storage overhead caused by extraneous data:

```sql
CREATE OR REPLACE FUNCTION ltss_strip_attributes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $code$
BEGIN
    -- Keep only 'unit_of_measurement' in the attributes JSONB
    IF NEW.attributes ? 'unit_of_measurement'
    THEN


        NEW.attributes = jsonb_build_object(
                         'unit_of_measurement',
                          NEW.attributes->'unit_of_measurement');
    ELSE
        NEW.attributes = '{}'::jsonb;
    END IF;


    RETURN NEW;
END;
$code$;


CREATE TRIGGER ltss_strip_attributes
BEFORE INSERT ON ltss
FOR EACH ROW
EXECUTE FUNCTION ltss_strip_attributes();
This text will be hidden
```

This trigger is especially useful when storing large amounts of sensor data where the attributes field may otherwise bloat the database.

# Understanding the TimescaleDB Architecture

## Hypertables
The TimescaleDB introduces a so-called hypertable. For a lot of use cases, it behaves like a traditional table, including DML operations. The underlying architecture makes it a special type of table designed for efficient time-series storage and querying.

**Hypertable Features:**

* Automatic partitioning: Data is split across time and optional space (e.g., entity_id) dimensions.
* Compression: Built-in support to compress historical chunks.
* Retention policies: You can automatically drop data older than a defined threshold.

The table, the LTSS inserts data into is the hypertable. Also, Continuous Aggregates (described below) store data into hyper tables, inheriting all their features.

## Using Continuous Aggregates

Continuous Aggregates (CAGGs) are materialized views that maintain aggregated data over time. These views are automatically updated as new data arrives. They are useful for:

* Improving query performance by precomputing aggregated data
* Enabling efficient storage via summarized data
* Allowing old data deletion from the source table, without affecting reports
* Being based on hypertables, CAGGs data can be compressed (or removed after some time, which is not what we need though)

### Defining Continuous Aggregates

Before we jump into CAGGs, let’s start with a helper function. Believe me or not, changing entity names, renaming them, is something that just happens, especially at the beginning of the setup. For example, I found that it’s better to have a generic name for injected/purchased energy rather than use names dependent on a measuring device. This way I can cover pre-FV a FV eras together.

Anyway, I found out that it’s easier to add a new sensor name or manipulate its name when the list of sensors is provided by the utility function, rather than hardcoded within a CAGG. There is no option to update the statement of the materialized view. At the same time, the function can be replaced at any time. Note the IMMUTABLE keyword in this function definition.

```sql
CREATE OR REPLACE FUNCTION public.get_entities_for_cagg_energy_hourly()
RETURNS text[]
LANGUAGE sql
IMMUTABLE
AS $function$
   SELECT ARRAY
       [
           'sensor.pg_mainhouse_total_energy_energy_hourly',
           'sensor.pg_cube_total_energy_energy_hourly',
           'sensor.energy_injected_hourly',
           'sensor.energy_purchased_hourly',
           'sensor.wattsonic_pv1_input_energy_2_hourly',
           'sensor.wattsonic_pv2_input_energy_2_hourly',
           'sensor.energy_discharged_from_battery_hourly',
           'sensor.energy_charged_to_battery_hourly'
       ];
$function$;
```

Now CAGG definition, making use of the function above. Mind excluding textural states

```sql
CREATE MATERIALIZED VIEW cagg_energy_hourly
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1h'::INTERVAL, "time", 'Europe/Prague'::text) AS bucket,
  entity_id,
  delta(counter_agg("time", ((state)::NUMERIC)::DOUBLE PRECISSION)) AS value
FROM ltss
WHERE entity_id = ANY (get_entities_for_cagg_energy_hourly())
AND state NOT IN ('unavailable', 'unknown')
FROM ltss
GROUP BY bucket, entity_id
WITH NO DATA;
```

To check the definition of already created CAGG, execute the query:

```sql
SELECT * FROM timescaledb_information.continuous_aggregates;
```
The `WITH NO DATA` clause makes the view be created immediately, but without computing its content. Without that clause, the view creation might take some time depending on the amount of data in the source table. If the CAGG has been created with no initial data population, generating it can be requested with a query:

```sql
CALL refresh_continuous_aggregate('cagg_energy_hourly', NULL, NOW()-'2h'::INTERVAL);
```

Note: This query will run a background process. The NULL passed to the second parameter means ‘take data from the beginning`. The 3rd argument adds a 2-hour buffer to not interfere with data being added in real-time.

There are several limitations when writing CAGGs. Most important to remember are:

* inability to use non-immutable functions in the view query projection (after select), in predicates (WHERE clause), as well as in GROUP BY
* Inability to use window functions
* The two above lead to Inability to reference data from beyond of CAGG processing window (ie, direct preceding record)
* need of using time_bucket()
* Inability to use time_bucket_fillgapp() instead of time_bucket().It makes it impossible to interpolate or extrapolate missing data during CAGG calculations.

These limitations can be worked around, most effectively by handling such calculations during data retrieval for front-end visualization or reporting. This allows for flexibility while keeping the continuous aggregate definitions simple and efficient.

Hopefully, this article provides ready-to-use solutions so the reader doesn't need to worry about that.

Ahh.. if there is a need to drop CAGG, dropping CAGG doesn’t drop the underlying hypertable. It has to be dropped manually.
With the use of the continuous_aggregates view, find the hypertable:

```sql
SELECT format('%I.%I', materialization_hypertable_schema, materialization_hypertable_name)
FROM timescaledb_information.continuous_aggregates
WHERE hypertable_name = '<cagg_name>'
```

Then drop the materialized view and the hypertable using general PostgreSQL syntax:

```sql
DROP MATERIALIZED VIEW <cagg_name> CASCADE;
DROP TABLE <hypertable_name>;
```
### Refreshing Aggregates

CAGGs might be refreshed manually and/or in a scheduled way.

Manual refresh was already mentioned in the previous paragraph to populate CAGG from already existing data. Worth mentioning that TimescaleDB internally stores information about already updated data range, refusing to refresh it twice. In some cases, ie, after manual data manipulation within CAGG’s hypertable, forcing a refresh might be required (see the TRUE value of the 4th parameter):

```sql
CALL refresh_continuous_aggregate('<cagg_name>', <window_start>, <window_end>, TRUE);
```
But for us, the most useful is to set CAGGs update policy, which ensures automatic updates:

```sql
SELECT add_continuous_aggregate_policy(
   'cagg_energy_hourly',
   start_offset => '1 hour'::INTERVAL,
   end_offset => '15 minutes'::INTERVAL,
   schedule_interval => '15 minutes'::INTERVAL
);
```

To improve performance, CAGG takes data for calculation only from the specified time range declared with two time offsets: start_offset and end_offset. Both are in the past, and relative to the moment of scheduled processing.

```
time —>———————|——————————————|———————————————|———————————|
         start offset   end offset     scheduled now    now
```

* start_offset: defines the lower boundary of how far back in time data should be refreshed. It also enables safe data deletion of older chunks from the origin hypertable. Otherwise, removing old data from the source table would be reflected in CAGG results.
* end_offset: defines the upper boundary of how far back in time data should be refreshed. It also prevents race conditions with incoming data. Aggressive end_offset values might miss performance expectations due to frequent refreshes.
* schedule_interval: How often should CAGG be refreshed

Because data coming from HA is appending only (no updates nor deletes), setting the policy is not so crucial, thus easier.

* For hourly aggregates: `start_offset = 1 hour, end_offset = 15 minutes, schedule_interval = 15` minutes
* For daily aggregates: `start_offset = 1 day, end_offset = 4 hours, schedule_interval = 2 hours`

Real-Time Aggregates

One valid concern with the configuration we discussed earlier is that queries against Continuous Aggregates (CAGGs) might show accurate historical data, but may not reflect the most recent changes in real-time.

And let's be honest — everyone wants real-time data in Grafana, right?

Luckily, TimescaleDB provides a solution: real-time aggregates.
When real-time aggregation is enabled, the database automatically blends:

* Precomputed (materialized) data
* The most recent raw records

This ensures that your queries always return up-to-date results, even if the continuous aggregate hasn't been refreshed yet.

By default, real-time aggregation is disabled, but you can easily turn it on (or off) at any time:

```sql
ALTER MATERIALIZED VIEW cagg_energy_hourly
SET (timescaledb.materialized_only = false);
```

Think of it like this:

It’s as if TimescaleDB performs an automatic union between your CAGG and the source table for the missing interval.

This way, you get the best of both worlds — high performance for historical data and freshness for new data.

### Hierarchical CAGGs

When working with large time ranges (like daily, monthly, or yearly views), it’s important to improve performance by aggregating at different levels.
One of the main reasons for using CAGGs is to eventually delete the original raw data — once it's safely summarized.

However, if you calculate monthly, quarterly, or annual aggregates directly from the original raw data, it wastes a lot of computing resources.
And if you’ve already deleted raw data (as planned), you wouldn't even be able to do it!

TimescaleDB offers a solution: Hierarchical Continuous Aggregates.
This means you can create a CAGG based on another, more detailed CAGG — without touching the raw data.

Here's a conceptual diagram (borrowed from the TimescaleDB blog):

![|191x250](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfqi-06JEio27didAnjeV7rrWtVvFhZuAAhGeNrJWTwTFg6laufqaxuyV6wWKz7H_uvEwMN25ovEvpA_SHNWFtAimqVQZyAoxOH8CWp7JHCt3tkUBb8iztw0BqJKIcVGuRihQoP?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Our Case: A Simple Daily CAGG

In our use case, we don’t need complicated dependency chains.
Instead, we can define a daily CAGG based on the existing hourly CAGG with just a few lines of SQL:

```sql
CREATE MATERIALIZED VIEW cagg_energy_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day'::interval, bucket) AS bucket,
    entity_id,
    SUM(value) AS value
FROM cagg_energy_hourly
GROUP BY time_bucket('1 day'::interval, bucket), entity_id;
```

To keep it automatically updated, add a continuous aggregation policy:

```sql
SELECT add_continuous_aggregate_policy
(
   'cagg_energy_daily',
   start_offset => '1 day::INTERVAL',
   end_offset => '4 hours'::INTERVAL,
   schedule_interval => '2 hours'::INTERVAL
);
```

And don’t forget to enable real-time aggregation if needed:

```sql
ALTER MATERIALIZED VIEW cagg_energy_daily
SET (timescaledb.materialized_only = false);
```

## Data Compression

As mentioned earlier, TimescaleDB hypertables offer built-in support for data compression — and I can say from experience:

It’s a game changer.

You can expect huge storage savings.
For example, in my setup:

* One month of raw energy data in the ltss table took around 7 GB.
* After compression, the same data took just 126 MB!

That's a compression ratio of more than 50× — seriously impressive.

### How Compression Works

To maintain good query performance on compressed data, the compressed format must match your query patterns. Specifically, you need to carefully choose:

* Partitioning (how the data is grouped)
* Ordering (how the data is sorted inside compressed blocks)

In our case:

* Partition by: entity_id
* Order by: time

You can configure and automate compression for older chunks like this:

```sql
ALTER TABLE cagg_energy_hourly
SET
(
   timescaledb.compress,
   timescaledb.compress_orderby='time',
   timescaledb.compress_segmentby='entity_id'
);
SELECT add_compression_policy('cagg_energy_hourly', compress_after => '30d'::INTERVAL);
```

This setup will automatically compress data older than 30 days.

### Checking Compression Stats

You can monitor compression results using the chunk_compression_stats() function. To get total compression for a table:
```sql
SELECT
    pg_size_pretty(SUM(before_compression_total_bytes)),
    pg_size_pretty(SUM(after_compression_total_bytes))
FROM chunk_compression_stats('ltss');
```
Or, to see compression details per chunk:
```sql
SELECT
    pg_size_pretty(before_compression_total_bytes),
    pg_size_pretty(after_compression_total_bytes),
    *
FROM chunk_compression_stats('ltss');
```
You may notice that some chunks have missing compression stats — these are still uncompressed.

### Finding the Size of Uncompressed Chunks

Since uncompressed chunks are regular PostgreSQL tables, you can check their size like this:

```sql
WITH
chunks AS
(
    SELECT format('%I.%I', chunk_schema, chunk_name)::regclass AS chunk_regclass
    FROM chunk_compression_stats('ltss')
    WHERE before_compression_table_bytes IS NULL
)
SELECT
   chunk_regclass,
   pg_size_pretty(pg_total_relation_size(chunk_regclass))
FROM chunks
```

The combined size of compressed and uncompressed chunks should match:

```sql
SELECT pg_size_pretty(hypertable_size('ltss'))
```
### Compression for Continuous Aggregates (CAGGs)

As mentioned earlier, it’s possible to compress data inside Continuous Aggregates (CAGGs).
Personally, I didn’t apply compression for CAGGs, because they usually consume very little disk space.
For example:

* 3 years of hourly aggregated energy data takes only 18 MB!

However, if you expect much larger datasets (or just want maximum storage optimization), you can enable compression for CAGGs as well.

Important note:
While it’s technically possible to compress the underlying hypertable directly, this is not recommended.
The reason is that continuous aggregate policies and compression policies must cooperate — otherwise you risk breaking updates or refreshes.

Instead, TimescaleDB provides a safe, dedicated method for CAGG compression:
```sql
ALTER MATERIALIZED VIEW <cagg_name> SET (timescaledb.compress = true);
SELECT add_compression_policy('<cagg_name>', compress_after=>'45 days'::INTERVAL);
```
This approach cleanly integrates compression into the automatic maintenance of your CAGGs.

Reminder:
Make sure the compress_after setting points further back in time than the start_offset you configured for add_continuous_aggregate_policy().
Otherwise, compression could interfere with the automatic refresh of newer data.

## Data retention

TBD

# Visualizing with Grafana

Once your data is successfully aggregated and stored in TimescaleDB, the final step is to present it effectively using Grafana. Grafana provides a powerful and flexible platform for creating visually appealing dashboards, offering various visualization options, from simple time series graphs to more complex multi-panel views.

The key question at this stage is: What should be visualized, and how?
The `what` depends on the data you’ve collected and your specific use cases (e.g., tracking energy consumption, performance metrics, etc.). The how depends on how you want the data to be displayed and interacted with, considering the audience and their needs.

In this section, I’ll walk you through the visualization setup that I’ve implemented, showing you the types of graphs used, how I approached the layout for an intuitive user experience, and how Grafana’s capabilities can enhance your monitoring setup

## House Consumption

Let’s start with the simplest case: monitoring the total energy consumption of the house. My house consists of two parts, and the total consumption is simply the sum of the consumption from the two sensors.

![|602x229](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfNzQB5BQUyBbZgpjM1Kkr4tJT8jHdd5X4jRzQ3vXop0yaSdcWXF5unqkwDaab5OGxfQOEo2m-YSoncQEogOeJZkcw4DpY5cvT3X_3YdmrhzFev-oNJoBqKo7H2omeOZroil2AZKQ?key=lxjGOMoRy8aXFNM_LTsRQxoa)

The Grafana query used for this visualization looks like this:

```sql
SELECT
    time_bucket_gapfill
    (
        '1day'::INTERVAL,      
        "bucket", 'Europe/Prague'
    ) AS timeb,
    entity_id,
    sum(value) AS value
FROM cagg_energy_hourly
WHERE entity_id  IN ('sensor.pg_mainhouse_total_energy_energy_hourly',
'sensor.pg_cube_total_energy_energy_hourly')
AND $__timeFilter("bucket")
GROUP BY timeb, entity_id
```

$__timeFilter("bucket") is a Grafana macro that automatically adjusts the time range based on the dashboard’s time selection. Grafana internally converts it to an expression like:
```sql
"bucket" BETWEEN '2022-12-31T23:00:00Z' AND '2025-04-27T08:37:12.797Z'
```
This ensures that your query only pulls data for the selected time range.

Transformations for Data Formatting

Once the query is set, it's time to adjust the data in Grafana. The next step involves using the Transformations tab in Grafana to manipulate the data into the right format for visualization.

1. Partition by Transformation:

  * Add a Partition by transformation.
  * Select entity_id as the field by which Grafana will partition the data into separate series.
  * Set As label for naming, and keep fields to “No” (this affects axis naming and color mapping).

2. This is a common setup for all graphs you'll create, ensuring consistent handling of axes.
3. Rename Fields by Regex:

  * Use the Rename by regex transformation to format field names appropriately.
  * This helps in cleaning up or adjusting axis labels and field names to make them more readable and useful in the context of the graph.

![|602x377](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcS3sA08CpPWgS543vTa7a-FXnyZAB8mLcnjCynA8-789apelH-_aqhS5vbGADkp6EPKSaDUD6K6LshTl5Vd-QrTI2EC4vIb-2WM0Q6fUrwC3dMi7RzFneyv5scmhQKTUQt6h1jyg?key=lxjGOMoRy8aXFNM_LTsRQxoa)

**Configuring the Visualization**

Now it’s time to configure the visualization itself. In this case, a Bar Chart is the most suitable type of graph for displaying the consumption data.

* If you’re visualizing data from multiple sensors, you might want to use the Stacking mode for the bars.

* Important Note: Stacking mode requires that all axes share the same time intervals. If the time intervals are different, the bars won’t stack correctly. This is why we use the time_bucket_gapfill() function in the query, which ensures consistent time intervals, even if data is missing for certain periods.

By following these steps, you can easily visualize the total energy consumption in your house and make the data more insightful in Grafana!

## Dynamic Data Source for Flexible Granularity

Previously, we visualized daily results based on hourly CAGGs. Now, let’s enhance the visualization with a dynamic data source selection and the ability to choose the aggregation level (hourly, daily, monthly, annually) based on user input or time range.

Understanding Grafana’s dynamic setup can take some time, but I’ll provide a ready-to-use solution to achieve this functionality.

#### Step 1: Define Variables in Grafana

1. Go to Dashboard Settings → Variables
2. Add the following variables to control granularity and data source selection:

![|602x140](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfwQ_cGKG20v12kaybBqw3wCj5kmbCtmlRgceFIQtNKNeefUoOVtbS3rKjE8TCvIh7AdnPupphGFIGV9dBAVEMGu4__kVb6iYIMxBK3A5QOlKKEACh93KThlDlwHqfBChqsBdib?key=lxjGOMoRy8aXFNM_LTsRQxoa)

**Variable 1: Granularity**

* Type: Custom
* Name: granularity
* Description: List of available granularity types (auto, hourly, daily, etc.)
* Show on Dashboard: Label and Value
* Custom Options:
  * Auto
  * hour: 1 hour
  * day: 1 day
  * month: 1 month
  * year: 1 year

This variable allows the user to manually select the granularity they want to use for the data aggregation.

**Variable 2: Query Granularity**

This variable will automatically determine the granularity based on the time range selected in Grafana. If the granularity is set to ‘Auto’, the code will adjust based on the time range.

* Type: Query
* Name: query_granularity
* Description: Determines how to group the data based on the selected time range.
* Query: 
```sql
SELECT 
    CASE WHEN '$granularity'  = 'Auto' THEN
        CASE 
           WHEN (to_timestamp($__to/1000) - to_timestamp($__from/1000)) > '2 months'::INTERVAL 
           THEN '1 month'::INTERVAL
           WHEN (to_timestamp($__to/1000) - to_timestamp($__from/1000)) > '3 days'::INTERVAL
           THEN '1 day'::INTERVAL
           ELSE '1 hour'::INTERVAL
        END
    ELSE REPLACE('$granularity', 'Auto', '0')::INTERVAL 
    END;
```

This query dynamically adjusts the granularity depending on the time range:

* Month if the range is greater than 2 months
* Day if the range is greater than 3 days
* Hour if the range is narrower than 3 days

The preview should show exactly this:

![|303x64](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfEL0YnakEO7KseeBOe4Bi8_vAiW9rw31FS_jw-4_AnbBphcuDbdh6SdpGFnS6FFowCDcpTrClV295R1q4RjAzkcayi8xGaCcTM8OihJOC-Cjyj8GW19cbT6PJdskWdixBHW7rHtQ?key=lxjGOMoRy8aXFNM_LTsRQxoa)

**Variable 3: Name Granularity**

The purpose of this variable is to ensure the correct CAGG table is selected based on the granularity. Since our CAGGs follow a naming convention (e.g., cagg_energy_hourly, cagg_energy_daily), we can replace the suffix of the table name based on the selected granularity.

* Type: Query
* Name: name_granularity
* Description: Returns the appropriate suffix for the CAGG table name based on the selected granularity.
* Query:

```sql
SELECT
    CASE '$query_granularity'::INTERVAL
        WHEN '1 hour'::INTERVAL THEN 'hourly'
        ELSE 'daily'
    END;
```
This query will return:

* `hourly` if the granularity is set to ‘1 hour’
* `daily` for other time intervals (you can extend it to include monthly or annual CAGGs if needed).

Now that we have the necessary variables (granularity, query_granularity, and name_granularity), we can create a new dynamic visualization in Grafana.

## Grid visualization

This visualization shows the amount of energy purchased and injected. In basics, it’s almost the same as the previous visualization we did.

![|602x227](https://lh7-rt.googleusercontent.com/docsz/AD_4nXeSEQiC2frGlHPRGgGjHpHV8mtD9Zy_eDCQ_YoYXclr4c1A1wsgPdzkUNGILqZECrfLrYAatCZb20Tsl77t3TNRtVV3zNsoTlZE6AiLC3cYW4ra9KMTuQCntYW0uT16i5Agr_uY?key=lxjGOMoRy8aXFNM_LTsRQxoa)

The main difference is in the query, which uses the recently prepared variables

```sql
SELECT
    time_bucket_gapfill
    (
        '$query_granularity'::interval,      
        "bucket",
        'Europe/Prague'
    ) AS timeb,
    entity_id,
    SUM(value) AS value
FROM cagg_energy_${name_granularity}
WHERE entity_id  IN ('sensor.energy_injected_hourly', 'sensor.energy_purchased_hourly')
AND $__timeFilter("bucket")
GROUP BY timeb, entity_id
```

Rename transformation with regular expression set to .*(injected|purchased).* returned clean names of series.

To separate injected and purchased data, I decided to show injected energy on the negative side of the graph. To achieve that, I used the Override feature of Grafana. You can find it on the very bottom of the visualization properties.

![|371x273](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfuVgxubHt7mte15MV_yyAdxmpwdGKJXXvhwg6od_wrPRMp7dvg_7x0f3qcOP8CDVF2Axu7ZrnpEAefYA257BCILZ_XYJXVbdL-do-kMWm9Td4fcCNBtVvQNMMGyV3Jz0cTnrxVGg?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Once it works, make adjustments to the Consumption panel SQL query, making use of prepared variables.

## Energy Usage

The last graph from the energy category is something similar to what Home Assistant shows on its Energy page. It will collect all energy data presented in previous charts into a bit different view.

![|602x228](https://lh7-rt.googleusercontent.com/docsz/AD_4nXe4J4iKTc-xehoNHBiTUcnLN9d4du135_P6OU4jQ6GEnt7UTuMGbiTcY30PNBHbEeqdyxNRq6yntTIEaGpR9VPHlRBJas7XRw2XZUeNUn1lGXEtjHzw5BOje0vgkiwzmtEh5fZcNA?key=lxjGOMoRy8aXFNM_LTsRQxoa)

The expected result is to get an overview of energy used vs superfluous

Let’s show the used energy above the X-axis. In my case, it consists of consumed solar and purchased energy.
Superfluous energy is energy injected into the grid as well as stored in the battery.

For this task, we need the following query:

```sql
SELECT
    time_bucket_gapfill
    (
        '$query_granularity'::interval,      
        "bucket", 'Europe/Prague'
    ) AS timeb,
    CASE WHEN entity_id ~ 'injected' THEN 'Injected'
        WHEN entity_id ~ 'purchased' THEN 'Purchased'
        WHEN entity_id ~ 'pv[12]' THEN 'PV'
        WHEN entity_id ~ '_charged' THEN 'Charged'
        WHEN entity_id ~ '_discharged' THEN 'Discharged'
    END AS entityid2,
    SUM(value) AS value
FROM cagg_energy_${name_granularity}
WHERE entity_id  IN (
'sensor.energy_injected_hourly',
'sensor.energy_purchased_hourly',
'sensor.wattsonic_pv1_input_energy_2_hourly',
'sensor.wattsonic_pv2_input_energy_2_hourly',
'sensor.energy_charged_to_battery_hourly',
'sensor.energy_discharged_from_battery_hourly'
)
AND $__timeFilter("bucket")
GROUP BY timeb, entityid2
```

The query already sums energy from my two FV strings, and sets final names for time series.

Now we need to use several transformations:

Like in the previous example, partition data by entityid2 and clean up the series names![|602x129](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcUkJcfJXC2gcvbUh9nlMv_-cQ18h3S9aLqjIBBT2OolUYJLC6aFfmKtUcwP88gfTxbD7zGjH_fyLMyY-aIGEe3pOG7YYgI4PZBDDu1E0Ygxij2sG6dC6M2OqAPjAFXSeaeWID9YQ?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Perform additional math using the Add field from calculation transformation

![|602x179](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdQYEPchpvjEMi1hOHEVLs9EMSgPnz2YyUbwcRjDJgPuTwgnzRI4oovNxQm2IGy1fPPmA5cOJqAAsRqpKr_AIX1Wh3TJs_gXMMF9vEQIwRll3cSxeGE9lcl9J9OXBzNNUr2cPpJJA?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Then clean up unwanted data series, and rename what needs to be renamed

![|602x156](https://lh7-rt.googleusercontent.com/docsz/AD_4nXc2TEvX4NHgacPD8n5QRc_B5qKtfXTeXameUNLyzi6qtAO0mwJOxRx0Mlr0E2AS1_QjaudQWT51fMVOtl93g94mOpICxuMx5Wgk3NyJkHZK3iFF_PnyTDfWTzBXcdOqrJsSQAgT?key=lxjGOMoRy8aXFNM_LTsRQxoa)

The final steps are selecting colors to display data series and inverting `Charged` and `Injected` to show them below the X axis.

## Costs

Costs are probably the ultimate goal of everything we have done so far. Everyone wants to know the profits from having FVE. Or be able to predict (at least roughly) its investment return time.

Having experience with building the visualisation from data stored in TimescaleDB, displaying costs is just another variation. The difference is, we need the price of the energy unit. Moreover, it must track multiple prices (for purchasing and selling energy) in time.

At this point, the approach might vary depending on the contract with the energy provider. For example, prices might change every several months or every 15 minutes, being delivered by HA sensor.

I don’t operate on spot, having a semi-fixed price, being not updated frequently (and getting previous and new price with a billing). So I come with an approach suitable for me, while it can be easily adjusted to be synced with HA sensor.

I’ve introduced a table in Postgresql database, maintaining prices manually.

This approach has multiple benefits at the start.

* I can change prices retrospectively
* I can setup them up for the future
* It can maintain multiple types of charge (energy, distribution etc)
* Once I have a HA sensor with a variable price, I can easily fill this table with incoming values using the trigger.

Here is a structure of the table:
```sql
CREATE TABLE public.electricity_cost
(
    cost_type TEXT NOT NULL,
    cost_kind TEXT NULL,
    cost_range DATARANGE NOT NULL,
    cost_value NUMERIC NOT NULL,
    cost_unit TEXT NOT NULL,
    CONSTRAINT exc_electricitycost_costrange EXCLUDE USING gist (cost_type WITH =, cost_kind WITH =, cost_range WITH &&),
    CONSTRAINT pk_electricitycost PRIMARY KEY (cost_type, cost_range)
);
CREATE INDEX exc_electricitycost_costrange ON public.electricity_cost USING gist (cost_type, cost_kind, cost_range);
```

Note that exc_electricitycost_costrange, which constraint that prevents the creation of overlapping ranges for the same cost type and kind. It creates the index named the same way.

For this, it’s needed to have the `btree_gist` extension installed (see the beginning of this article).

The table is populated with data (example from my installation). Units are informative (but might be used for recalculation if one needs that). Note, pricelists usually contain prices for MWh. If you maintain energy in kWh like me, those numbers need to be divided by 1000.
```
cost_type|cost_range             |cost_value|cost_unit|cost_kind   |
---------+-----------------------+----------+---------+------------+
purchase |[2023-08-29,2023-11-01)|   4.69834|kWh      |energy      |
purchase |[2023-11-01,2024-03-20)|   4.28925|kWh      |energy      |
purchase |[2024-03-20,2024-08-01)|   3.75537|kWh      |energy      |
purchase |[2024-08-01,2024-08-08)|     3.618|kWh      |energy      |
purchase |[2024-08-08,2026-01-01)|   2.99008|kWh      |energy      |
purchase |[2023-08-29,2024-01-01)|     1.611|kWh      |distribution|
purchase |[2024-01-01,2024-08-06)|   2.01566|kWh      |distribution|
purchase |[2024-08-06,2025-01-01)|   2.01566|kWh      |distribution|
purchase |[2025-01-01,infinity]  |   2.09963|kWh      |distribution|
sale     |[2024-08-08,2025-04-01)|       1.4|kWh      |energy      |
sale     |[2025-04-01,infinity)  |      1.21|kWh      |energy      |
```


Thanks to time stored as a range type (BTW another powerful PostgreSQL feature), it’s very easy to match particular records depending on measurement time.

I made a helper function that calculates the cost of the given energy and time.

```sql
CREATE OR REPLACE FUNCTION public.calculate_cost
(
   _cost_type  TEXT,
   _time       TIMESTAMPTZ,
   _value      NUMERIC
)
RETURNS NUMERIC
LANGUAGE sql
IMMUTABLE
AS $function$
   SELECT SUM(cost_value * _value)
   FROM electricity_cost
   WHERE _time::DATE <@ cost_range
     AND cost_type = _cost_type
$function$;
```

Let’s do some graphs now.

## Price evolution

![|602x227](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcL0vUEvOsxwsl_lXX4gsCsJ4E29Y8llT8s1t0qoB0fSZp4R-gRr0iIBknvDsxNkis9vJpNffaVfjqlqFM36ma_4nh6DOZAVsZE-oyLqrV6W3Ui2wtCuG--hWXU4WSdMC6mx00S_w?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Since the table with prices contains ranges, not points in time, these must be generated within SQL query. For this task generate_series() function is the best choice. note, using the Grafana $query_granularity variable, which command generate_series to generate datapoints in hourly or daily resolution.

```sql
SELECT time, cost_type, cost_kind, cost_value
FROM electricity_cost AS ec
JOIN generate_series
     (
          to_timestamp($__from/1000)::DATE::TIMESTAMPTZ,
          to_timestamp($__to/1000)::TIMESTAMPTZ,
          '$query_granularity'::interval
     ) AS x(time) ON TRUE
WHERE x.time::DATE <@ cost_range
ORDER BY time, cost_type, cost_kind
```
Having data, let’s clean up series names:

![|602x116](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdG9lC3kyLSYbRrKGfTkM5o1bNL6SDwbm6biPmHRULhmvkt_keE-G1HvCh9vA4ks--N8cA3HjEs89woxhliKq5-Py11He7IvQArsyINicXYkAbjPZRql7EzS-97nB04XP5lZQyocg?key=lxjGOMoRy8aXFNM_LTsRQxoa)

## Energy Price

It gets more interesting now. Let’s create a chart showing prices of purchased, injected and avoided energy. The first two are obvious. The latter is the energy that could have been purchased having an FVA installed. At the end, injected and avoided energy both contribute to the ROI.

![|602x228](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdhGigI-d2MloVMLeqpBxQ04bazPrKBNJOt16fzO7KW3ExlAmzis33Kvs1sZdy8qf7PMDWL6YT3V89aA9krWyMRGevpYPNvcqYN1mO-zDg_oMlPHVObSNE91rEZYtiDRzzoBqUqqw?key=lxjGOMoRy8aXFNM_LTsRQxoa)

SQL Query:

```sql
SELECT
    time_bucket_gapfill
    (
        '$query_granularity'::INTERVAL,      
        "bucket", 'Europe/Prague'
    ) AS timeb,
    CASE WHEN entity_id ~ 'injected' THEN 'Injected'
        WHEN entity_id ~ 'purchased' THEN 'Purchased'
        WHEN entity_id ~ 'cube|mainhouse' THEN 'Consumption'
        ELSE entity_id
    END AS entityid2,
    SUM
    (
        calculate_cost
        (
            CASE WHEN entity_id ~ 'injected' THEN 'sale'
                 WHEN entity_id ~ 'purchased' THEN 'purchase'
                 WHEN entity_id ~ 'cube|mainhouse' THEN 'purchase'
            END,
            bucket, value::NUMERIC
        )
    ) AS value
FROM cagg_energy_${name_granularity}
WHERE entity_id IN
(
    'sensor.energy_injected_hourly',
    'sensor.energy_purchased_hourly',
    'sensor.pg_mainhouse_total_energy_energy_hourly',
    'sensor.pg_cube_total_energy_energy_hourly'
)
AND $__timeFilter("bucket")
GROUP BY timeb, entityid2
```

See, we use the calculate_cost() function to get prices. It’s required to get prices before summing the data for the selected time granularity. Assuming that prices are not changing more often than once a day. We can still use hourly and daily CAGGs. If prices change hourly, daily CAGG cannot be used for obvious reasons (unless we agree with calculation divergence.

In case of prices changing every 15 minutes, I would probably extend CAGG with pre-calculated prices

As usual, partitioning and initial series name cleanup:

![|602x131](https://lh7-rt.googleusercontent.com/docsz/AD_4nXeTUqQjnr9i3juZWUBFrzwPpmcRlLW-Oa-UmsJIkAwHMc5S1SqgQxbkBK6UqHwOXZJbPKRojTlDdb1YR59dqdFdrAsCQk57uAtV5ak6yh23ZKEflKRh-EP1U5pfTIrdK0kI43y4qQ?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Reduce the measured consumed energy by purchased one

![|602x88](https://lh7-rt.googleusercontent.com/docsz/AD_4nXf2wblglI_nWGdWULEvIf-lSQ51BaOARccbAAEwcb3hrWoYn_HT3gZ60mGkFmVEU7E_YTt_Lksv6dpskakEo7c9rETL_aZruVy9_KBx6oF2EaKVQdMkH56iTgI0nsyIukHWwF-tjA?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Hide unneeded ones:

![|602x103](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfSaW8L36ZjhXk8V65TdOUQiOiXckF6zK9kKlcEo3bvEjLIscwsihlIgjAAtlkJkvbu_FIg8EFN5rWb3pIfMsh8RQpgMkXBYjSWGkmghXyMKnv8yqYKyQuwL5meYC54fvoe0hTM6w?key=lxjGOMoRy8aXFNM_LTsRQxoa)

And flip the purchasing data series to be on the negative side.

## Returned value

![|602x229](https://lh7-rt.googleusercontent.com/docsz/AD_4nXd8Eh2qUHe3yznPWMqDbHVEqPhGsqru6SwClybUK_7hGHOxvV2TeQzQMBKUh5hQzK0ctEYYBPjWiW7nLfXRP42Gy0UtzUD7qEQdVmpG_Eme5LTYwGcW3Paf9WDt5Dx0-5qZurrqzQ?key=lxjGOMoRy8aXFNM_LTsRQxoa)

The task is easy: calculate a cumulative sum of the energy price from existing energy records. It turns out into the most complex SQL query presented in this article. To achieve it, we need the window function. It can be written using several ways, but I prefer CTE for readability.

```sql
WITH
src AS
(
    SELECT
    time_bucket_gapfill
    (
        '$query_granularity'::INTERVAL,      
        "bucket", 'Europe/Prague'
    ) AS timeb,
    CASE WHEN entity_id ~ 'injected' THEN 'Injected'
        WHEN entity_id ~ 'purchased' THEN 'Purchased'
        WHEN entity_id ~ 'cube|mainhouse' THEN 'Consumption'
        ELSE entity_id
    END AS entityid2,
    SUM(CASE
            WHEN bucket < '2024-08-08' THEN 0
            ELSE calculate_cost
                (
                    CASE WHEN entity_id ~ 'injected' THEN 'sale'
                        WHEN entity_id ~ 'purchased' THEN 'purchase'
                        WHEN entity_id ~ 'cube|mainhouse' THEN 'purchase'
                    END,
                    bucket,
                    value::NUMERIC
                )
        END) AS value
    FROM cagg_energy_${name_granularity}
    WHERE entity_id  IN (
                            'sensor.energy_injected_hourly',
                            'sensor.energy_purchased_hourly',
                            'sensor.pg_mainhouse_total_energy_energy_hourly',
                            'sensor.pg_cube_total_energy_energy_hourly'
                        )
    AND $__timeFilter("bucket")
    GROUP BY timeb, entityid2
)
SELECT
    timeb,
    entityid2,
    SUM(value) OVER (
        PARTITION BY entityid2
        ORDER BY timeb::DATE
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS value
FROM src
```

To avoid unexpected presentation results, I zeroed collected data before FVE installation (`WHEN bucket < '2024-08-08' THEN`).

Transformations:

![|602x132](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdVkTGtLDlczlpPbF5ns6KexImYxgwg7dESH4ynJDskF3NLTqMRJyGqg4-UJ3dqrzlK24XOcNfMO77dSrmTMCuP237d9BQgZ65ghDF1XPZ0hmr4Pu7nx5h3w0499rSeel6pv6JK?key=lxjGOMoRy8aXFNM_LTsRQxoa)

![|602x181](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfcR6cVfSL0SInUZljG-fLDFOqc1rCPwVZdKkd1FS6FOSIcYdJpcJ5pvREWyxNYyu06IM2t9U47GpXxZIKS6QG_tGeI3R-NTfsicH4BMdVI3Uye_TjEBbIOuIoOcQ7Z_aT3BMPFUw?key=lxjGOMoRy8aXFNM_LTsRQxoa)

![|602x123](https://lh7-rt.googleusercontent.com/docsz/AD_4nXf0c1OcbalMq339nSXfX_3xB2w7C-XO5CDQFMX3BhL7wQElJRVr6Md81x4x3wmgtveFH4o9Od4__TGieUpcEgh-EzauTvyZlhkH6GBdfFiBwy1zT-BG-N8zYF7Q38T7wzt8qp8fuA?key=lxjGOMoRy8aXFNM_LTsRQxoa)
