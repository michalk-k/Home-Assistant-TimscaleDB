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
* Refractory on entities replacement: In HA, it's difficult to maintain entity naming when making changes to the household systems, such as installing solar panels (FVE). Such changes lead to using different measurement devices, thus differently named sensors. Tracking energy before and after such events results in fragmented or incomplete views, without an option to visualise them as continuous data.

## Benefits of Using TimescaleDB

TimescaleDB, a time-series extension of PostgreSQL, offers a more robust solution for energy consumption data logging and analysis:

* Efficient time-series storage: Built specifically to handle large volumes of time-stamped data.
* Scalability: Handles months or years of high-resolution data with ease.
* Advanced queries: Supports SQL with time-series specific functions.
* Continuous Aggregates: Enables efficient pre-aggregation of data for fast queries.
* Flexible retention and compression: Easily drop or compress old data while preserving aggregate summaries (which can also be compressed)
* Changes to the data are up to your will. You can import data from other systems, rename entities. For example, you can import consumed energy data offered by the provider from time periods HA system did not measure house consumption, etc.
* On-the-fly data transformation like filtering, renaming, unit conversion, and more.

Bonus: Grafana can bring all this data to life with interactive dashboards right from the start.

All in one, with TimescaleDB, it’s possible to achieve energy analytics unavailable in HA, like the return value of FV installation collected over the years.

![|602x227](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdsORdqpQ79w_ul4NxFlKigsO4ZBhQlY0YYdkG_6epbxpn5dWOh3DA-kCZ8jGDQEyNhrUIIR5cG-gF6zBPbfAMXQ-34Lb2Z91lc70aqkEOuQC98tlqbclP9Ao6ry2mtA1XGMP5xbg?key=lxjGOMoRy8aXFNM_LTsRQxoa)

## Prerequisites

To get started, ensure the following components are installed and configured:

* Home Assistant sensors. Read below for the requirements for HA sensors
* Long-Term Statistics Store (LTSS) integration.
* TimescaleDB with TimescaleDB Toolkit extension enabled. It might be an add-on running in your HA Supervisor environment or as an external service.
* Grafana, like TimescaleDB, you can use the HA add-on or an independent installation of Grafana.

The next paragraphs go through key points of configuration.

### Home Assistant Sensors

Before we move on with the database, we need to acknowledge the character of data and make sure HA provides sensors we can use later.

In general, energy data tracked by Home Assistant sensors grows in time indefinitely. Optionally, it's allowed that from time to time the value is reset to zero and starts counting again. The reset can be triggered by a measurement device or by HA itself (utility sensors) at any time.

It's important to understand this concept because it makes common methods like sum or delta not applicable to data evolving this way. Managing it using common SQL might be pretty complex. 
TimescaleDB comes with ready-to-use tools helping with that.

There is one more issue to mention and solve: data changes are reported with finite frequency (granularity). It leads to "ignoring" some energy amount when attempting to calculate this data within intervals. Let's look at the diagram:
```
          0     5     12           15     23
time ─|───o─────o─────o─────|───────o─────o─────|────>
                      x  midnight   y
```
Suppose we want to calculate total energy usage for the day (midnight to midnight). With data distributed as in the picture, it's impossible to precisely determine:

* the energy used between x and midnight
* the energy used between midnight and y
* the daily energy

Simple grouping by day would result in 12 units during the first day and 8 units on the second day. 3 units are lost. Sometimes your energy readings will be frequent enough that the error is small, but you can’t always count on that.

We could try to interpolate (estimate) these missing parts, but for our case (explained later), interpolation isn't the way to go.

How to solve this?

We can use Home Assistant's Utility Meters. Based on my observations:
* Such sensors reset automatically at the start of each period (quarter-hourly, hourly, daily, monthly, etc.)
* The first recorded value for each period is always 0.
* The first value following the reset represents the energy collected since the last known value before zero-point.

In the result, the value at y represents the delta (the amount of energy used) between x and y.
This method causes energy from before the end of the period to be accounted for in the next one.
```
          0     5     12     0      3     7
time ─|───o─────o─────o─────|o──────o─────o─────|────>
                      x  midnight   y
```
Worth mentioning that utility sensors (regardless of reset time interval) based on the same source sensor are all updated at the same rate as the source sensor. It renders into conclusion that there is no need to publish daily sensors if there is hourly one. The shortest period utility sensor is enough to cover our needs, since we can make daily values out of hourly ones.

**Note:**
In this article, we assume hourly data as the baseline.
This could be considered enough for most use cases, unless you're dealing with other needs like quarter-hourly data, for often used for spot market energy trading.

### Consistent Units for Energy Sensors

Home Assistant can create energy sensors with different units — for example, Wh, kWh, or even MWh.
When you configure your utility meters, make sure all related sensors use the same unit.

If your sensors use mixed units, you have three options (some are messier than others):

* Convert the units when recording data into the database
* Convert the units later during data aggregation
* Unify units during visualization

Later in this article, you'll see that units are stored together with the data. However, unit conversion is not covered in the example queries — they assume everything is already consistent.

**My recommendation:**
Stick to `kWh` as your standard unit.
It’s the most common, easy to read, and fits well for hourly or daily energy tracking.

# Installation
## Timescale DB

TimescaleDB can be installed as a separate service or using a ready-to-use Home Assistant add-on. The installation process is described on the [add-on github pages](https://github.com/expaso/hassos-addon-timescaledb?tab=readme-ov-file#installation).

To execute actions described in this part of the article, you will need a DB client. The first already built-in option is a `psql` console. You can access it directly from HA console:
```
docker exec -it addon_77b2833f_timescaledb  psql -U <username> <databasename>
```

If you prefer a GUI client, you can use pgAdmin4 (also available as an HA add-on), or use other clients like DBeaver (my favorite). In both cases, enabling exposition of the PostgreSQL port in TimescaleDB Add-on settings is required.

After TimescaleDB installation, only the `postgres` login role is available. If you create your own, make it `SUPERUSER` - it might be needed. Then be careful, it gives a superpower ;)


### Privileges
Assuming TimescaleDB was installed via the add-on installer and the necessary databases were created, those databases are accessible to any role that can authenticate.

To operate LTSS and Grafana, we need to define dedicated login roles with minimal required privileges. Using the default postgres role is discouraged due to its broad and unrestricted permissions.

Let’s create roles specifically for LTSS and Grafana:
```sql
CREATE ROLE app_ltss LOGIN PASSWORD 'some_password';
CREATE ROLE app_grafana LOGIN PASSWORD 'another_password';
```

LTSS needs permission to create tables in the public schema:
```sql
GRANT CREATE ON SCHEMA public to app_ltss;
```

### Extensions
Next, install the required extensions that will be used later:
```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS timescaledb_toolkit;
CREATE EXTENSION IF NOT EXISTS btree_gist;
```

:bulb: Why install these manually?

* `postgis` – Installing it up front prevents the LTSS component from requesting superuser privileges to install it later.
* `timescaledb_toolkit` – Enables additional time-series features needed for handling continuously increasing values like energy consumption.
* `btree_gist` – Required later for advanced indexing when generating cost reports.

### Schema organization
Now let’s create a dedicated schema. This keeps our objects organized and avoids polluting the already-crowded public schema used by TimescaleDB:
```sql
CREATE SCHEMA ltss_energy;
GRANT USAGE ON SCHEMA ltss_energy to public;
```
By default, database users don't automatically have access to schemas unless explicitly granted. You can assign schema access either to specific roles or, if you want to allow access to all authenticated users, to the public pseudo-role, as shown above.

This same rule applies to objects created within the schema (except for functions).

### Timezone does matter!
Because we’re working with data grouped by days, it’s critical to ensure that the database uses the correct timezone. Otherwise, query results can be misaligned—for example, a "midnight" timestamp might fall on the wrong day depending on the timezone.

You can resolve this either by configuring the database appropriately or by explicitly passing the timezone in queries that rely on date grouping.

#### How PostgreSQL Handles Timezones
The active timezone used in a session is determined by several layers of configuration. The settings are applied in the following order:
```server defaults -> database -> client connection -> login role```

Some tools (e.g., DBeaver) automatically set the timezone at the start of the client connection, usually based on the system or application settings. Others, like psql or Grafana, do not override the session timezone, relying instead on whatever is configured at the database level.

This is important to understand, especially if you observe different query results depending on which tool you're using.

#### Check the Active Timezone
To check the current session timezone and its source, use:

```sql
SELECT setting, source FROM pg_settings WHERE name = 'TimeZone';
```
To examine both server and database-level timezone settings, run:
```sql
SELECT 'server' AS source, boot_val AS timezone FROM pg_Settings WHERE name = 'TimeZone'
UNION
SELECT 'database' AS source, split_part(cfg, '=', 2) AS timezone
FROM pg_db_role_setting s
JOIN pg_database d ON s.setdatabase = d.oid
JOIN  unnest(s.setconfig) AS cfg ON TRUE
WHERE datname = CURRENT_CATALOG AND split_part(cfg, '=', 1) = 'TimeZone';
```

* The server row shows the default timezone at the instance level (usually UTC when using TimescaleDB add-on).
* If a database row appears, its value overrides the server default for that specific database.

#### Set the Preferred Timezone

If the database timezone does not match your desired setting, it’s best to set it at the database level. This ensures consistency and also affects TimescaleDB background processes (like CAGG refresh policies).

To set the database timezone:

```sql
ALTER DATABASE <databasename> SET TimeZone = 'desired timezone';
```

Avoid abbreviations like `CET` or `PST`. Instead, use full names (e.g., `Europe/Prague`, `America/New_York`). These handle daylight saving time automatically and are much more reliable year-round.

You can find a list of all supported timezone names with:
```sql
SELECT * FROM pg_timezone_names;
```

## Configuring LTSS to Export Energy Sensors
LTSS is the Home Assistant's custom component. It can be installed by HACS. 
Installation instructions are available in [project Github](https://github.com/freol35241/ltss?tab=readme-ov-file#installation).

LTSS writes all data into a single table: `public.ltss` (without option to change). In the article, the table will be referenced just as `ltss` since schema `public` usually doesn't need to be explicitly referenced.

During start, the LTSS checks the existence of `ltss` table. If it doesn't exist. It also tries to add `postgis` extension to the database. For the latter operation, it requires superuser privileges. But as you remember, we have already installed it. 

Once LTSS is installed, configure it to publish all needed sensors to TimescaleDB.

> :warning: Changes to LTSS configuration require HA restart. 

Configuration of LTSS typically involves selecting the appropriate sensors in the integration’s configuration in configuration.yaml.

The example below makes LTSS component connected to `ha_ltss` database available in TimescaleDB installed as HA add-on with user `app_ltss`. It will publish specified Glances sensors (unimportant from pov of this article, just an example) and all sensors whose name ends with `_hourly` phrase.

```yaml
ltss:
  db_url: postgresql://app_ltss:some_password@77b2833f-timescaledb/ha_ltss
  include:
    entities:
      - sensor.glances_cpu_load
      - sensor.glances_cpu_used
      - sensor.glances_cpu_percent
      - sensor.glances_cpu_thermal_0_temperature
      - sensor.glances_ram_free
      - sensor.glances_ram_used
      - sensor.glances_ram_used_percent
      - sensor.glances_swap_free
      - sensor.glances_swap_used
      - sensor.glances_data_free
      - sensor.glances_swap_used
      - sensor.glances_data_used_percent
    entity_globs:
      - sensor.*_hourly
```
The configuration is similar to what the recorder offers, including globs. Unfortunately, it inherits the limitation that entities included by globs cannot be excluded anymore. In such a case, there are two options: replace globs with an explicit list of sensors, or let LTSS publish a broader set of data, to reject some entities with the use of a before trigger. While the second option seems tempting, it's easy to forget about such a trigger later on. Also, it still costs additional communication and processing. But sometimes knowing this way might be life-saving. 

Anyway, here is an example of such a trigger:

```sql
CREATE OR REPLACE FUNCTION public.ltss_exclude_entities()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Filter out unwanted entities before insertion
    IF NEW.entity_id NOT IN 
       (
           'sensor.unwanted_energy_consumption_hourly',
           'sensor.solar_production_hourly'
       )
    THEN
       RETURN NULL;  -- Skip insert
    END IF;
   
    RETURN NEW;
END;
$$;


CREATE TRIGGER ltss_exclude_entities
BEFORE INSERT ON public.ltss
FOR EACH ROW
EXECUTE FUNCTION public.ltss_exclude_entities();
```

HA sensors often include extensive and unnecessary metadata in their attributes field. These attributes are stored as JSONB in the database, which can significantly increase disk usage. If the additional metadata is not essential for your use case, it's wise to strip it away to optimize storage. Below is a trigger that retains only the unit_of_measurement attribute, which is typically the most relevant, thereby minimizing storage overhead caused by extraneous data:

```sql
CREATE OR REPLACE FUNCTION public.ltss_strip_attributes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
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
$$;


CREATE TRIGGER ltss_strip_attributes
BEFORE INSERT ON public.ltss
FOR EACH ROW
EXECUTE FUNCTION public.ltss_strip_attributes();
```

This trigger is especially useful when storing large amounts of sensor data where the attributes field may otherwise bloat the database.
Note, there is another tool to manage a large amount of data: compression (described later).

## Grafana

Like TimescaleDB, you have option to use Home Assistant add-on or external Grafana. 
Installation itself requires no notes.

Grafana natively supports postgresql. The only thing is to configure a datasource within Grafana. Provide `77b2833f-timescaledb:5432` as Host Url, then database name, user name and password as created earlier. Enable TimescaleDB switch - it will enable TimescaleDB specific features in SQL builder. Configuration of diagrams described in this article will not be using it though.

:bulb: Note, data source offers no option to set TimeZone.

# Working with TimescaleDB

Let's start with an ASCII art showing what we are going to achieve:
* Older `ltss` data will be compressed, and the even older one will be removed from the database
* We create the first level aggregation (hourly) being updated by recent data found in the `ltss` table
* We create the second-level aggregation (daily) built updated by data from the previous aggregation.

The diagram below depicts the time-wise dependencies between TimescaleDB objects accessing other ones. Note the fact, that CAGGs looks at source data limited by floating time window. It allows to delete old data from source objects.

```
      drop old data       compressed data 
ltss ─ ─ ─ ─ ─ ─ ─ ─ ─ ┴────────────────────┴─...────────┬─────┬───────┬────>
                                                         |     |      
                                                         |     |
                                                hourly CAGG refresh window
                                                         |     |
CAGG hourly ──────┬─────────────────────┬───...──────────┴─────┴───────┼────>
                  |                     |
                  |                     |
                 daily CAGG refresh window
                  |                     |
CAGG daily ───────┴─────────────────────┴────...───────────────────────┼────> time
                                                                      now
```

💡 Anticipating the facts: querying daily aggregation can result in real-time results, like querying ltss table! Keep reading!

## Hypertables
The TimescaleDB introduces a new table engine, the so-called hypertable. For a lot of use cases, it behaves like a traditional table. The underlying architecture makes it a special type of table designed for efficient time-series storage and querying.

**Hypertable Features:**

* Automatic partitioning: Data is split across time and optional space (e.g., entity_id) dimensions.
* Compression: Built-in support to compress historical chunks.
* Retention policies: You can automatically drop data older than a defined threshold.

The LTSS inserts data into the hypertable. Also, Continuous Aggregates (described below) store data into hyper tables, inheriting all their features. We will be back to this subject in this article later.

## Using Continuous Aggregates

Continuous Aggregates (CAGGs) are materialized views that maintain aggregated data over time. These views are automatically updated as new data arrives. They are useful for:

* Improving query performance by precomputing aggregated data
* Enabling efficient storage via summarized data
* Allowing old data deletion from the source table, without affecting reports
* Being based on hypertables, CAGGs data can be compressed (or removed after some time, which is not what we need though)

Earlier in the article, I mentioned that utility sensors refresh at the same rate as source sensors. It's not impossible that a single sensor generates 10MB of daily data if the changes are so frequent.

> :bulb: Thanks to CAGGs, we will transform that amount of data into a single value per defined period. 

### Defining Continuous Aggregates

Before we jump into CAGGs, let’s start with a helper function. Believe me or not, changes to processed entity names, renaming them, is something that just happens, especially at the beginning of the setup. For example, I found that it’s better to have a generic name for injected/purchased energy rather than use names dependent on a measuring device. I renamed them in HA, but then needed to reflect the change in a CAGG. Another situation was the need to cover pre-FV a FV eras together.

Anyway, I found out that it’s easier to add a new sensor name or manipulate its name when the list of sensors is provided by the utility function, rather than hardcoded within a CAGG. There is no option to update the statement of the materialized view. At the same time, the function can be replaced at any time. Notice the IMMUTABLE keyword in this function definition.

```sql
CREATE OR REPLACE FUNCTION ltss_energy.get_entities_for_cagg_energy()
RETURNS text[]
LANGUAGE sql
IMMUTABLE
AS $$
   SELECT ARRAY
       [
            -- replace sensor names with your ones.
           'sensor.pg_mainhouse_total_energy_energy_hourly',
           'sensor.pg_cube_total_energy_energy_hourly',
           'sensor.energy_injected_hourly',
           'sensor.energy_purchased_hourly',
           'sensor.wattsonic_pv1_input_energy_2_hourly',
           'sensor.wattsonic_pv2_input_energy_2_hourly',
           'sensor.energy_discharged_from_battery_hourly',
           'sensor.energy_charged_to_battery_hourly'
       ];
$$;
```

Now CAGG definition, making use of the function above. The CAGG makes use of a `counter_agg()` function. It comes from the timescaledb toolkit. And it's designed to handle data whose values grow indefinitely, being reset sometimes. Also, notice casting `state` value into a numeric data type (HA provides it as a text). To not make this cast fail, textual values like `'unavailable', 'unknown'` must be excluded.

```sql
CREATE MATERIALIZED VIEW ltss_energy.cagg_energy_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1h'::INTERVAL, "time", 'Europe/Prague'::text) AS bucket,
    entity_id,
    delta(counter_agg("time", state::::DOUBLE PRECISSION)) AS value
FROM ltss
WHERE entity_id = ANY (ltss_energy.get_entities_for_cagg_energy())
  AND state NOT IN ('unavailable', 'unknown')
GROUP BY bucket, entity_id
WITH NO DATA;

GRANT SELECT ON VIEW ltss_energy.cagg_energy_hourly TO public;
```

`GRANT` gives read-only access to all connected users.
TimescaleDB background processes don't need special privileges.
You, as a view creator (owner), are not limited by any privileges.

It might be useful to look at the existing CAGG in order to check out its definition:

```sql
SELECT * FROM timescaledb_information.continuous_aggregates;
```
The `WITH NO DATA` clause makes the view be created immediately, but without computing its content. Without that clause, the view creation might take some time depending on the amount of data in the source table. If the CAGG has been created with no initial data population, generating it can be requested with a query:

```sql
CALL refresh_continuous_aggregate('ltss_energy.cagg_energy_hourly', NULL, NOW()-'2h'::INTERVAL);
```

Note: This query will run a background process. The NULL passed to the second parameter means ‘take data from the past`. The 3rd argument adds a 2-hour buffer to not interfere with data being added in real-time.

There are several limitations when writing CAGGs. Most important to remember are:

* inability to use non-immutable functions in the view query projection (after select), in predicates (WHERE clause), as well as in GROUP BY
* Inability to use window functions
* The two above lead to inability to reference data from beyond of CAGG processing window (ie, direct preceding record)
* need of using `time_bucket()`
* Inability to use `time_bucket_fillgapp()` instead of time_bucket().

All those limitations make it impossible to interpolate or extrapolate missing data during CAGG calculations (are you recalling the issue described in the HA sensors paragraph?).

These limitations can be worked around, most effectively by handling such calculations later on, during data retrieval for front-end visualization or reporting. This allows for flexibility while keeping the continuous aggregate definitions simple and efficient.

Hopefully, this article provides ready-to-use solutions so the reader doesn't need to worry about that.

Ahh.. if there is a need to drop CAGG, dropping CAGG doesn’t drop the underlying hypertable. It has to be dropped manually.
With the use of the `continuous_aggregates` view, find the hypertable:

```sql
SELECT format('%I.%I', materialization_hypertable_schema, materialization_hypertable_name)
FROM timescaledb_information.continuous_aggregates
WHERE hypertable_name = '<cagg_name>';
```

Then drop the materialized view and the hypertable using common PostgreSQL syntax:

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
But for us, the most useful is to set the CAGGs update policy, which ensures automatic updates:

```sql
SELECT add_continuous_aggregate_policy(
   'ltss_energy.cagg_energy_hourly',
   start_offset => '1 hour'::INTERVAL,
   end_offset => '15 minutes'::INTERVAL,
   schedule_interval => '15 minutes'::INTERVAL
);
```

To improve performance, CAGG takes data for calculation only from the specified time range declared with two time offsets: `start_offset` and `end_offset`. Both are in the past, and relative to the moment of scheduled processing.

```
time —>———————|——————————————|———————————————|———————————|
         start offset   end offset     scheduled time   now
```

* `start_offset`: defines the lower boundary of how far back in time data should be refreshed. It also enables safe data deletion of older chunks from the origin hypertable. Otherwise, removing old data from the source table would be reflected in CAGG results.
* `end_offset`: defines the upper boundary of how far back in time data should be refreshed. It also prevents race conditions with incoming data. Aggressive end_offset values might miss performance expectations due to frequent refreshes.
* `schedule_interval`: How often should CAGG be refreshed

Data coming from HA is appending only (no updates nor deletes), thus the policy might be set like this:

* For hourly aggregates: `start_offset = 1 hour, end_offset = 15 minutes, schedule_interval = 15 minutes`
* For daily aggregates: `start_offset = 1 day, end_offset = 4 hours, schedule_interval = 2 hours`

### Real-Time Aggregates

One valid concern with the configuration we discussed earlier is that queries against Continuous Aggregates (CAGGs) might show accurate historical data, but may not reflect the most recent changes in real-time.

And let's be honest — everyone wants real-time data in Grafana, right?

Luckily, TimescaleDB provides a solution: **real-time aggregates**.\
When real-time aggregation is enabled, the database automatically blends:

* Precomputed (materialized) data
* The most recent raw records

Think of it like this: it’s as if TimescaleDB performs an automatic union between your CAGG and the source table for the missing interval.

This ensures that queries against CAGG always return up-to-date results, even if the continuous aggregate hasn't been refreshed yet.

By default, real-time aggregation is disabled, but you can easily turn it on (or off) at any time:

```sql
ALTER MATERIALIZED VIEW ltss_energy.cagg_energy_hourly
SET (timescaledb.materialized_only = false);
```

This way, you get the best of both worlds — high performance for historical data and freshness for new data.

### Hierarchical CAGGs

When working with large time ranges (like daily, monthly, or yearly views), it’s important to improve performance by aggregating at different levels.
One of the main reasons for using CAGGs is to eventually delete the original raw data, once it's safely summarized.

However, if you calculate monthly, quarterly, or annual aggregates directly from the original raw data, it wastes a lot of computing resources.
And if you’ve already deleted raw data (as planned), you wouldn't even be able to do it!

TimescaleDB offers a solution: Hierarchical Continuous Aggregates.
This means you can create a CAGG based on another, more detailed CAGG, without touching the raw data.

Here's a conceptual diagram (borrowed from the TimescaleDB blog):

![|191x250](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfqi-06JEio27didAnjeV7rrWtVvFhZuAAhGeNrJWTwTFg6laufqaxuyV6wWKz7H_uvEwMN25ovEvpA_SHNWFtAimqVQZyAoxOH8CWp7JHCt3tkUBb8iztw0BqJKIcVGuRihQoP?key=lxjGOMoRy8aXFNM_LTsRQxoa)

In our use case, we don’t need as complicated dependency chains as above.
Instead, we can define a daily CAGG based on the existing hourly CAGG with just a few lines of SQL:

```sql
CREATE MATERIALIZED VIEW ltss_energy.cagg_energy_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day'::interval, bucket) AS bucket,
    entity_id,
    SUM(value) AS value
FROM ltss_energy.cagg_energy_hourly
GROUP BY time_bucket('1 day'::interval, bucket), entity_id;

GRANT SELECT ON VIEW ltss_energy.cagg_energy_hourly TO public;
```

To keep it automatically updated, add a continuous aggregation policy:

```sql
SELECT add_continuous_aggregate_policy
(
   'ltss_energy.cagg_energy_daily',
   start_offset => '1 day::INTERVAL',
   end_offset => '4 hours'::INTERVAL,
   schedule_interval => '2 hours'::INTERVAL
);
```

And don’t forget to enable real-time aggregation if needed:

```sql
ALTER MATERIALIZED VIEW ltss_energy.cagg_energy_daily
SET (timescaledb.materialized_only = false);
```

## Data Compression

As mentioned earlier, TimescaleDB hypertables offer built-in support for data compression, and I can say from experience:

It’s a game changer.

You can expect huge storage savings.
For example, in my setup:

* One month of raw energy data in the `ltss` table took around 7 GB.
* After compression, the same data took just 126 MB!

That's a compression ratio of more than 50× — seriously impressive.

### How to set up compression

To maintain good query performance on compressed data, the compressed format must match your query patterns. Specifically, you need to carefully choose:

* Partitioning (how the data is grouped)
* Ordering (how the data is sorted inside compressed blocks)

In our case:

* Partition by: entity_id
* Order by: time

You can configure and automate compression for older chunks like this:

```sql
ALTER TABLE ltss
SET
(
   timescaledb.compress,
   timescaledb.compress_orderby='time',
   timescaledb.compress_segmentby='entity_id'
);
SELECT add_compression_policy('ltss', compress_after => '30d'::INTERVAL);
```

This setup will automatically compress data chunks older than 30 days.

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
The reason is that continuous aggregate policies and compression policies must cooperate — otherwise, you risk breaking updates or refreshes.

Instead, TimescaleDB provides a safe, dedicated method for CAGG compression:
```sql
ALTER MATERIALIZED VIEW <cagg_name> SET (timescaledb.compress = true);
SELECT add_compression_policy('<cagg_name>', compress_after=>'45 days'::INTERVAL);
```
This approach cleanly integrates compression into the automatic maintenance of your CAGGs.

⚠️ Reminder:\
Make sure the `compress_after` setting points further back in time than the `start_offset` you configured for `add_continuous_aggregate_policy()`.
Otherwise, compression could interfere with the automatic refresh of newer data.

## Data retention

⚠️  **Warning: The action described in this paragraph can permanently delete your collected data. Proceed with caution.**

Once your data has been aggregated into Continuous Aggregates (CAGGs), retaining the raw data in the source table (`ltss`) may no longer be necessary.

However, be aware that the `ltss` table might contain other sensor data not covered by this article (e.g., temperature or hardware metrics), which may not yet be aggregated. Data retention applies to all data.

TimescaleDB supports automatic data retention policies that delete old data by dropping entire chunks of the hypertable:

```sql
 SELECT add_retention_policy('ltss', INTERVAL '6 months');
```

The statement above configures TimescaleDB to retain raw data in the `ltss` table for at least one month. Once a data chunk is older than that threshold, it is automatically removed from the database.


> :warning: Important: To avoid losing necessary data for your aggregates, ensure that the CAGGs refresh window does not overlap with any data that might be deleted. For more information, refer to the [Timescale DB docs](https://docs.timescale.com/use-timescale/latest/data-retention/data-retention-with-continuous-aggregates/).

While it's technically possible to apply retention policies to CAGGs themselves, it's unlikely to be useful for the scenarios covered in this article.

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
    SUM(value) AS value
FROM ltss_energy.cagg_energy_hourly
WHERE entity_id  IN 
      (
          'sensor.pg_mainhouse_total_energy_energy_hourly',
          'sensor.pg_cube_total_energy_energy_hourly'
      )
  AND $__timeFilter("bucket")
GROUP BY timeb, entity_id
```

The `$__timeFilter("bucket")` is a Grafana macro that automatically adjusts the time range based on the dashboard’s time selection. Grafana internally converts it to an expression like:
```sql
"bucket" BETWEEN '2022-12-31T23:00:00Z' AND '2025-04-27T08:37:12.797Z'
```
This ensures that your query only pulls data for the selected time range.

**Transformations for Data Formatting**

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

Now it’s time to configure the visualization itself. In this case, a Bar Chart is the most suitable type of graph for displaying the consumption in time interval.

💡 If you’re visualizing data from multiple sensors, you might want to use the Stacking mode for the bars.

💡 Important Note: Stacking mode requires that all axes share the same time intervals. If the time intervals are different, the bars won’t stack correctly. This is why we use the time_bucket_gapfill() function in the query, which ensures consistent time intervals, even if data is missing for certain periods.

By following these steps, you can easily visualize the total energy consumption in your house and make the data more insightful in Grafana!

## Dynamic Data Source for Flexible Granularity

Previously, we visualized daily results based on hourly CAGGs. Now, let’s enhance the visualization with a dynamic data source selection and the ability to choose the aggregation level (hourly, daily, monthly, annually) based on user input or time range.

Understanding Grafana’s dynamic setup may take some time, but I’ll provide a ready-to-use solution to achieve this functionality.

**Define Variables in Grafana**

1. Go to Dashboard Settings → Variables
2. Add the following variables to control granularity and data source selection:

![|602x140](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfwQ_cGKG20v12kaybBqw3wCj5kmbCtmlRgceFIQtNKNeefUoOVtbS3rKjE8TCvIh7AdnPupphGFIGV9dBAVEMGu4__kVb6iYIMxBK3A5QOlKKEACh93KThlDlwHqfBChqsBdib?key=lxjGOMoRy8aXFNM_LTsRQxoa)

**Variable 1: Granularity**

* Type: Custom
* Name: granularity
* Description: List of available granularity types (auto, hourly, daily, etc.)
* Show on Dashboard: Label and Value
* Custom Options: `Auto, hour: 1 hour, day: 1 day, month: 1 month, year: 1 year`

This variable will allow the user to manually select the granularity he/she want to see on the graph. The `Auto` will cause the selection of the granularity automatically based on the selected time range.

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

The purpose of this variable is to ensure the most suitable CAGG is used for the requested granularity. Since our CAGGs follow a naming convention (e.g., cagg_energy_hourly, cagg_energy_daily), we can just replace the suffix of the CAGG name to achieve that.

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

Here is a query, applying newly created variables:

```sql
SELECT
    time_bucket_gapfill
    (
        '$query_granularity'::INTERVAL,      
        "bucket",
        'Europe/Prague'
    ) AS timeb,
    entity_id,
    SUM(value) AS value
FROM ltss_energy.cagg_energy_${name_granularity}
WHERE entity_id IN ('sensor.energy_injected_hourly', 'sensor.energy_purchased_hourly')
AND $__timeFilter("bucket")
GROUP BY timeb, entity_id
```

Rename transformation with regular expression set to .*(injected|purchased).* returned clean names of series.

To separate injected and purchased data, I decided to show injected energy on the negative side of the graph. To achieve that, I used the Override feature of Grafana. You can find it on the very bottom of the visualization properties.

![|371x273](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfuVgxubHt7mte15MV_yyAdxmpwdGKJXXvhwg6od_wrPRMp7dvg_7x0f3qcOP8CDVF2Axu7ZrnpEAefYA257BCILZ_XYJXVbdL-do-kMWm9Td4fcCNBtVvQNMMGyV3Jz0cTnrxVGg?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Once it works, make adjustments to the previously created Consumption graph, making use of dynamic variables.

## Energy Usage

The last graph from the energy category is something similar to what Home Assistant shows on its Energy page. It will show all energy data presented in previous charts into a bit different view.

![|602x228](https://lh7-rt.googleusercontent.com/docsz/AD_4nXe4J4iKTc-xehoNHBiTUcnLN9d4du135_P6OU4jQ6GEnt7UTuMGbiTcY30PNBHbEeqdyxNRq6yntTIEaGpR9VPHlRBJas7XRw2XZUeNUn1lGXEtjHzw5BOje0vgkiwzmtEh5fZcNA?key=lxjGOMoRy8aXFNM_LTsRQxoa)

The expected result is to get an overview of energy consumed vs superfluous.

Let’s show the consumed energy above the X-axis. In my case, it consists of consumed solar and purchased energy.
The energy injected into the grid as well as stored in the battery is considered superfluous.

For this task, we need the following query:

```sql
SELECT
    time_bucket_gapfill
    (
        '$query_granularity'::interval,      
        "bucket", 'Europe/Prague'
    ) AS timeb,
    CASE WHEN entity_id ~ 'injected'   THEN 'Injected'
        WHEN entity_id ~ 'purchased'   THEN 'Purchased'
        WHEN entity_id ~ 'pv[12]'      THEN 'PV'
        WHEN entity_id ~ '_charged'    THEN 'Charged'
        WHEN entity_id ~ '_discharged' THEN 'Discharged'
    END AS entityid2,
    SUM(value) AS value
FROM ltss_energy.cagg_energy_${name_granularity}
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

The query already sums energy from my two FV strings and sets final names for the time series.

Now we need to use several transformations:

Like in the previous example, partition data by entityid2 and clean up the series names![|602x129](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcUkJcfJXC2gcvbUh9nlMv_-cQ18h3S9aLqjIBBT2OolUYJLC6aFfmKtUcwP88gfTxbD7zGjH_fyLMyY-aIGEe3pOG7YYgI4PZBDDu1E0Ygxij2sG6dC6M2OqAPjAFXSeaeWID9YQ?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Perform additional math to achieve `Consumed Solar = PV - Injected - Charged`:

![|602x179](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdQYEPchpvjEMi1hOHEVLs9EMSgPnz2YyUbwcRjDJgPuTwgnzRI4oovNxQm2IGy1fPPmA5cOJqAAsRqpKr_AIX1Wh3TJs_gXMMF9vEQIwRll3cSxeGE9lcl9J9OXBzNNUr2cPpJJA?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Then clean up unwanted data series, and rename what needs to be renamed

![|602x156](https://lh7-rt.googleusercontent.com/docsz/AD_4nXc2TEvX4NHgacPD8n5QRc_B5qKtfXTeXameUNLyzi6qtAO0mwJOxRx0Mlr0E2AS1_QjaudQWT51fMVOtl93g94mOpICxuMx5Wgk3NyJkHZK3iFF_PnyTDfWTzBXcdOqrJsSQAgT?key=lxjGOMoRy8aXFNM_LTsRQxoa)

The final steps are selecting colors to display data series and inverting `Charged` and `Injected` to show them below the X axis.

## Costs

Costs are probably the ultimate goal of everything we have done so far. Everyone wants to know the profits from having FVE. Or be able to predict (at least roughly) its investment return time.

Having experience with building the visualisation from data stored in TimescaleDB, displaying costs is just another variation. The difference is, we need the price of the energy unit. Moreover, it must track multiple prices (for purchasing and selling energy) in time.

At this point, the approach might vary depending on the contract with the energy provider. For example, prices might change every several months or every 15 minutes, being delivered by an HA sensor.

I don’t operate on the spot, having a semi-fixed price, being not updated frequently (and getting the previous and new price with a billing). So I come with an approach suitable for me, while it can be easily adjusted to be synced with the HA sensor.

I’ve introduced a table in the PostgreSQL database, maintaining prices manually.

This approach has multiple benefits at the start.

* I can change prices retrospectively
* I can set them up for the future
* It can maintain multiple types of charge (energy, distribution, etc)
* Once I have a HA sensor with a variable price, I can easily fill this table with incoming values using the trigger.

Here is a structure of the table:
```sql
CREATE TABLE ltss_energy.electricity_cost
(
    cost_type TEXT NOT NULL,
    cost_kind TEXT NULL,
    cost_range DATARANGE NOT NULL,
    cost_value NUMERIC NOT NULL,
    cost_unit TEXT NOT NULL,
    CONSTRAINT xc_electricitycost_costrange EXCLUDE USING gist (cost_type WITH =, cost_kind WITH =, cost_range WITH &&),
    CONSTRAINT pk_electricitycost PRIMARY KEY (cost_type, cost_kind, cost_range)
);
```

Note that the `xc_electricitycost_costrange` constraint prevents the creation of overlapping ranges for the same cost type and kind. It creates the index named the same way.

Such of constraint requires `btree_gist` extension, which we already have installed (see the beginning of this article).

The table is populated with data (example from my installation). Units are informative (but might be used for recalculation if one needs that). Note, pricelists usually contain prices for `MWh`. If you maintain energy in `kWh` like me, those numbers need to be divided by 1000.


|cost_type|cost_kind   |cost_range             |cost_value|cost_unit|
|---------|------------|-----------------------|----------|---------|
|purchase |energy      |[2023-08-29,2023-11-01)|   4.69834|kWh      |
|purchase |energy      |[2023-11-01,2024-03-20)|   4.28925|kWh      |
|purchase |energy      |[2024-03-20,2024-08-01)|   3.75537|kWh      |
|purchase |energy      |[2024-08-01,2024-08-08)|     3.618|kWh      |
|purchase |energy      |[2024-08-08,2026-01-01)|   2.99008|kWh      |
|purchase |distribution|[2023-08-29,2024-01-01)|     1.611|kWh      |
|purchase |distribution|[2024-01-01,2024-08-06)|   2.01566|kWh      |
|purchase |distribution|[2024-08-06,2025-01-01)|   2.01566|kWh      |
|purchase |distribution|[2025-01-01,infinity]  |   2.09963|kWh      |
|sale     |energy      |[2024-08-08,2025-04-01)|       1.4|kWh      |
|sale     |energy      |[2025-04-01,infinity)  |      1.21|kWh      |


Thanks to time stored as a range type (BTW another powerful PostgreSQL feature), it’s very easy to match particular records depending on measurement time.

I made a helper function that calculates the cost of the energy at a certain time:

```sql
CREATE OR REPLACE FUNCTION ltss_energy.calculate_cost
(
   _cost_type  TEXT,
   _time       TIMESTAMPTZ,
   _value      NUMERIC
)
RETURNS NUMERIC
LANGUAGE sql
IMMUTABLE
AS $$
   SELECT SUM(cost_value * _value)
   FROM ltss_energy.electricity_cost
   WHERE _time::DATE <@ cost_range
     AND cost_type = _cost_type
$$;
```

Let’s do some graphs now.

## Price evolution

![|602x227](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcL0vUEvOsxwsl_lXX4gsCsJ4E29Y8llT8s1t0qoB0fSZp4R-gRr0iIBknvDsxNkis9vJpNffaVfjqlqFM36ma_4nh6DOZAVsZE-oyLqrV6W3Ui2wtCuG--hWXU4WSdMC6mx00S_w?key=lxjGOMoRy8aXFNM_LTsRQxoa)

Since the table with prices contains ranges (instead of points in time), we need to generate time series out of it within a SQL query. For this task generate_series() function is the best choice. Note, using the Grafana $query_granularity variable, which command generate_series to generate datapoints in hourly or daily resolution.

```sql
SELECT time, cost_type, cost_kind, cost_value
FROM ltss_energy.electricity_cost AS ec
JOIN generate_series
     (
          to_timestamp($__from/1000)::DATE::TIMESTAMPTZ,
          to_timestamp($__to/1000)::TIMESTAMPTZ,
          '$query_granularity'::interval
     ) AS x(time) ON TRUE
WHERE x.time::DATE <@ cost_range
ORDER BY time, cost_type, cost_kind
```

The `$__from` and `$__to` are Grafana macros. They are replaced during execution with numbers representing selected time boundaries.

Having data, let’s clean up series names:

![|602x116](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdG9lC3kyLSYbRrKGfTkM5o1bNL6SDwbm6biPmHRULhmvkt_keE-G1HvCh9vA4ks--N8cA3HjEs89woxhliKq5-Py11He7IvQArsyINicXYkAbjPZRql7EzS-97nB04XP5lZQyocg?key=lxjGOMoRy8aXFNM_LTsRQxoa)

## Energy Price

It gets more interesting now. Let’s create a chart showing prices of purchased, injected and avoided energy. The first two are obvious. The latter is the energy that could have been purchased with no FVA installed. At the end, injected and avoided energy both contribute to the ROI.

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
    SUM (
            ltss_energy.calculate_cost
            (
                CASE WHEN entity_id ~ 'injected' THEN 'sale'
                    WHEN entity_id ~ 'purchased' THEN 'purchase'
                    WHEN entity_id ~ 'cube|mainhouse' THEN 'purchase'
                END,
                bucket, value::NUMERIC
            )
        ) AS value
FROM ltss_energy.cagg_energy_${name_granularity}
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

The task sounds trivial: calculate a cumulative sum of the energy price (sold+avoided-bought) from existing energy records. It turns out into the most complex SQL query presented in this article. To achieve it, we need the window function. It can be written using several ways, but I prefer CTE for readability.

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
            ELSE ltss_energy.calculate_cost
                (
                    CASE WHEN entity_id ~ 'injected' THEN 'sale'
                        WHEN entity_id ~ 'purchased' THEN 'purchase'
                        WHEN entity_id ~ 'cube|mainhouse' THEN 'purchase'
                    END,
                    bucket,
                    value::NUMERIC
                )
        END) AS value
    FROM ltss_energy.cagg_energy_${name_granularity}
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

To avoid unexpected presentation results, I zeroed out collected data before FVE installation (`WHEN bucket < '2024-08-08' THEN 0`).

Transformations:

![|602x132](https://lh7-rt.googleusercontent.com/docsz/AD_4nXdVkTGtLDlczlpPbF5ns6KexImYxgwg7dESH4ynJDskF3NLTqMRJyGqg4-UJ3dqrzlK24XOcNfMO77dSrmTMCuP237d9BQgZ65ghDF1XPZ0hmr4Pu7nx5h3w0499rSeel6pv6JK?key=lxjGOMoRy8aXFNM_LTsRQxoa)

![|602x181](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfcR6cVfSL0SInUZljG-fLDFOqc1rCPwVZdKkd1FS6FOSIcYdJpcJ5pvREWyxNYyu06IM2t9U47GpXxZIKS6QG_tGeI3R-NTfsicH4BMdVI3Uye_TjEBbIOuIoOcQ7Z_aT3BMPFUw?key=lxjGOMoRy8aXFNM_LTsRQxoa)

![|602x123](https://lh7-rt.googleusercontent.com/docsz/AD_4nXf0c1OcbalMq339nSXfX_3xB2w7C-XO5CDQFMX3BhL7wQElJRVr6Md81x4x3wmgtveFH4o9Od4__TGieUpcEgh-EzauTvyZlhkH6GBdfFiBwy1zT-BG-N8zYF7Q38T7wzt8qp8fuA?key=lxjGOMoRy8aXFNM_LTsRQxoa)
