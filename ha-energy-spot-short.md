## Preface

Originally intended as a simple extension to the previous article to support spot prices, the update revealed additional challenges and required more significant changes than expected. As a result, it turned into more advanced database project rather than tool for every-day Home Assistant user, especially if every move supported by respective description.

After a month or so of struggling to create easy-to-read short guide, I I decided to provide  "how-to-use" guide instead of "how to build" one. If you want to dig more, you are welcome to read full size text. At the same time, all technical aspects described in previous article applies to this one too.
If you already using a model inspired by previous part, you will find useful the migration path presented at the end of this text.

Let's start with overview of the solution, through examples of pricing data. The whole source code to be installeed is avaialble at the end of article.

## The overview

The diagram below illustrates involved components and objects to be created. Related tables and views are grouped within rectangles, and arrows indicate the direction of data flow.

```mermaid
flowchart LR
    subgraph Postgresql

        style Postgresql fill:lightgray
        style HA fill:lightgray

        t_prices@{ label: "electricity_prices" }
        t_feesp@{ label: "electricity_fees" }

        t_ltss@{ shape: rect, label: "public.ltss" }

        subgraph CAGGs
            v_energy_qh@{ label: "Energy Quarter-hourly" }
            v_energy_h@{ label: "Energy Hourly" }
            v_energy_d@{ label: "Energy Daily" }

            v_costs_qh@{ label: "Costs Quarter-hourly" }
            v_costs_h@{ label: "Costs Hourly" }
            v_costs_d@{ label: "Costs Daily" }
        end

    end

    subgraph HA [Home Assistant]
        p_ltss@{ shape: rect, label: "LTSS\nCustom Integration" }
    end

    p_ltss-->|insert|t_ltss


    v_energy_qh-->|fetch|t_ltss
    v_costs_qh-->|fetch|t_ltss

    v_energy_h-->|fetch|v_energy_qh
    v_energy_d-->|fetch|v_energy_h

    v_energy_qh-->|fetch|v_costs_qh
    v_costs_qh-->|"call calculate_fees()"|t_feesp
    v_costs_qh-->|"call calculate_costs()"|t_prices

    v_costs_h-->|fetch|v_costs_qh
    v_costs_d-->|fetch|v_costs_h

    t_ltss-->|"trigger(insert)"|t_prices
```

The overall approach remains similar to the previous article, rendering a few hierarchical CAGGs, based on provided prices. Despite of it, it introduces two important changes:

**1. Separate CAGGs for Energy and Costs**
While both energy and costs can be materialized by the single CAGG, separating them offers several advantages:

1. You can adjust prices retroactively and regenerate cost CAGGs, even if the original data is no longer present in the `ltss` table.
2. Additional energy-related CAGGs can be created later, leveraging the precalculated energy aggregates.
3. The energy CAGG can aggregate all energy sensors, while the cost CAGG can focus on a subset of sensors.
4. The codebase remains cleaner and easier to maintain.

**2. Net values and fees (ie Taxes)**

When trading on the spot market, prices are provided in net value. Depending on contract and other regulations some additional costs add added over the top of net price:
* a fixed price per energy unit (e.g., 250 CZK per 1 MWh). It might be handling fee for a trading company, costs of distribution or other ones defined by goverment
* a percentage of the energy price (e.g., 15% of the sold energy price). It might be taxes or another way of calculating the handling fee for a trading copany

To achieve that we will store all volume-based prices (ie 250CZK/MWh) in prices table, while percentage-based prices (ie 21% VAT) into rate table.

### Pricing tables

Becuase some names will apear in the source code, we need to estabilish some conventions:
1. All names starts with uppercase character. They might be all uppercase too. It's rather for eastetics and consiscy. It will influence visualizaiton in Grafana too.
1. `Purchase` and `Sale` are the only allowed values for `trade_type`
1. `Energy` is reserved name for price of power electricity alone.

Here is an example of my price setting. It represents purchasing and saling for fixed prices untill November, 11th. Since this time sale trading is changing to spot-based

| trade_type | price_name    | price_period                                      | price_value |dir|volume_unit |
|------------|---------------|---------------------------------------------------|-------------|---|------------|
|Purchase    |CEPS           |["2023-08-29 00:00:00+02","2024-01-01 00:00:00+01")|      0.11353|  1|kWh         |
|Purchase    |CEPS           |["2024-01-01 00:00:00+01","2025-01-01 00:00:00+01")|      0.21282|  1|kWh         |
|Purchase    |CEPS           |["2025-01-01 00:00:00+01",infinity)                |      0.31178|  1|kWh         |
|Purchase    |Dan Z Elektriny|["2023-08-29 00:00:00+02",infinity)                |       0.0283|  1|kWh         |
|Purchase    |Distribution   |["2023-08-29 00:00:00+02","2024-01-01 00:00:00+01")|        1.611|  1|kWh         |
|Purchase    |Distribution   |["2024-01-01 00:00:00+01","2024-08-06 00:00:00+02")|      2.01566|  1|kWh         |
|Purchase    |Distribution   |["2024-08-06 00:00:00+02","2025-01-01 00:00:00+01")|      2.01566|  1|kWh         |
|Purchase    |Distribution   |["2025-01-01 00:00:00+01",infinity)                |      2.09963|  1|kWh         |
|Purchase    |Energy         |["2023-08-29 00:00:00+02","2023-11-01 00:00:00+01")|      4.69834|  1|kWh         |
|Purchase    |Energy         |["2023-11-01 00:00:00+01","2024-03-20 00:00:00+01")|      4.28925|  1|kWh         |
|Purchase    |Energy         |["2024-03-20 00:00:00+01","2024-08-01 00:00:00+02")|      3.75537|  1|kWh         |
|Purchase    |Energy         |["2024-08-01 00:00:00+02","2026-01-01 00:00:00+01")|      2.99008|  1|kWh         |
|Purchase    |POZE           |["2023-12-31 23:00:00+01",infinity)                |        0.495|  1|kWh         |
|Sale        |Energy         |["2024-08-08 00:00:00+02","2025-04-01 00:00:00+02")|          1.4|  1|kWh         |
|Sale        |Energy         |["2025-04-01 00:00:00+02","2025-06-29 00:00:00+02")|            1|  1|kWh         |
|Sale        |Energy         |["2025-06-29 00:00:00+02","2025-11-01 00:00:00+02")|          0.5|  1|kWh         |
|Sale        |EnerSpot       |["2025-11-01 00:00:00+02",infinity)                |         0.25| -1|kWh         |
|Sale        |Energy         |["2025-11-01 00:00:00+02","2025-11-01 01:00:00+02")| ...         |  1|kWh         |
|Sale        |Energy         |["2025-11-01 00:01:00+02","2025-11-01 02:00:00+02")| ...         |  1|kWh         |
|Sale        |Energy         |["2025-11-01 00:02:00+02","2025-11-01 03:00:00+02")| ...         |  1|kWh         |
| ...        | ...           | ...                                               | ...         |...|kWh         |

You might notice a record named EnerSpot. It correlates with a breaking point when I started to to sell for spot prices. For this operation, the operator will charge me 250CZK / 1MWh. To be able to cover calculation of both: purchasing cost and income with use of a single formula, I estabilished the `dir` attribute. The math is as easy as it seems: `dir=1` adds value to, while `dir=-1` deducts values from the sum. The income is sold energy minus operator handling feel the `minus` in formula is secured by `dir=-1`. You get the point, right?

If you surf on the Spot, meaning you are purchasing and saling with spot prices, the prices have to be recorded into the table for sale and purchase. Which could look like this:

| trade_type | price_name    | price_period                                      | price_value |dir|volume_unit |
|------------|---------------|---------------------------------------------------|-------------|---|------------|
|Purchase    |Energy         |["2025-11-01 00:00:00+02","2025-11-01 01:00:00+02")| ...         |  1|kWh         |
|Sale        |Energy         |["2025-11-01 00:00:00+02","2025-11-01 01:00:00+02")| ...         |  1|kWh         |
|Purchase    |Energy         |["2025-11-01 00:01:00+02","2025-11-01 02:00:00+02")| ...         |  1|kWh         |
|Sale        |Energy         |["2025-11-01 00:01:00+02","2025-11-01 02:00:00+02")| ...         |  1|kWh         |
|Purchase    |Energy         |["2025-11-01 00:02:00+02","2025-11-01 03:00:00+02")| ...         |  1|kWh         |
|Sale        |Energy         |["2025-11-01 00:02:00+02","2025-11-01 03:00:00+02")| ...         |  1|kWh         |
| ...        | ...           | ...                                               | ...         |...|kWh         |

Despite it creates redundant records, is more flexible and future-proof. It supports a wider range of trading scenarios, including changes in energy trading practices over time, while keeping the source code straightforward.

Which operations spot prices are recorded for is determined by content of `trades_import`.
The trigger uses this table to determine which trade type should receive the spot price, based on the current settings.

For example:

| trade_type | trade_period                                 |
|------------|----------------------------------------------|
| Sale       | ["2025-11-01 00:00:00+01", infinity)         |
| Purchase   | ["2026-06-01 00:00:00+02", infinity)         |

This example above means that incoming spot prices will be stored in `electricity_prices` as Sale starting November 1st, and as Purchase beginning in June of the following year. Note that both Sale and Purchase records will continue to be stored from June onward, allowing you to track both trading directions simultaneously.

If the table is empty, or if the price time does not match any entry, the price will not be added to the prices table. Be careful when entering timestamps, as time offsets can be error-prone - especially in regions with daylight saving time. I advice to use named time zones like Europe/Prague, that makes database calculate proper time offset for the user.

#### Fees table

This table makes possible to deduct percentual fees from initial energy price calculated for a volume. It might be taxes such as VAT. Notably, there is relationship between (trade_type, price_name) pair to prices table. This relationship is not enforced by a foreign key constraint. It's intentional, to make the independent maintaince of prices and fees possible; Without need of making the data model overly complex and less intuitive.

As for example it's possible to use infinity values to setup VAT for forever using `(-infinity, infinity)` time range.

The ony drawback is, that a user has to take care about validity of names.

The following example demonstrates a configuration of the VAT applicable to price entries.

|trade_type  |price_name     |fee_name|fee_period          |fee_value|dir|
|------------|---------------|--------|--------------------|---------|---|
|Purchase    |CEPS           |VAT     |(-infinity,infinity)|     0.21|  1|
|Purchase    |Dan Z Elektriny|VAT     |(-infinity,infinity)|     0.21|  1|
|Purchase    |Distribution   |VAT     |(-infinity,infinity)|     0.21|  1|
|Purchase    |Energy         |VAT     |(-infinity,infinity)|     0.21|  1|
|Purchase    |POZE           |VAT     |(-infinity,infinity)|     0.21|  1|

It's possible to set many fees for single price item. These data are used to calculate final income or cost. For this purpose the order of operations does not matter.

## Populating prices

> :bulb: This part is tricky because it strongly depends on how you collect spot prices. Use it as a source of inspiration and adapt the approach to match your data provider, integration method, and automation tools.

When working with static energy prices, you typically update records only when contracts change. However, spot pricing requires frequent updates, as new price records must be entered continuously. The method you use to feed prices into your database depends on your data source and available tools. Options include custom Home Assistant (HA) integrations, Node-RED automations, external scripts, or even manual SQL queries for infrequently changing prices. The essential requirement is that prices are inserted into the `electricity_prices` table before energy data is aggregated into CAGGs.

If HA is your data source, the simplest solution would be to publish a sensor that provides the current price via LTSS to the database. However, this method has a significant drawback: any delay in delivering price to the database leads to skewing costs for those periods. This is generally unacceptable for cost tracking.

A more robust approach is to store prices in advance, especially since spot prices are available ahead of time. The exact solution will depend on your price provider’s API and your data processing workflow. While there is no universal method, the following example illustrates a practical approach.

Suppose you have an entity in HA, which provides the next day's prices as a JSON array in its attributes:

```json
{
    "period": "00:15:00",
    "data": [
        {"time": "time1", "price": 1.0},
        {"time": "time2", "price": 2.0},
        {"time": "time3", "price": 3.0}
        ...
    ]
}
```

To achieve this format, you will likely need to convert original data from your provided. Here is an example of Home Assistant template sensor, that converts data provided by Czech Spot Prices integration:

<details>
<summary>Click for the code</summary>

```yaml
template:
  - name: "Spot Electricity Prices"
    unique_id: "pv_ctrl_spot_electricity_prices"
    default_entity_id: sensor.pv_ctrl_spot_electricity_prices
    state: "{{ states('sensor.current_spot_electricity_price_15min') }}"
    unit_of_measurement: CZK/kWh
    attributes:
      interval: "00:15:00"
      data: >
          {% set data = namespace(prices=[]) %}
          {% for key, val in states.sensor.current_spot_electricity_price_15min.attributes.items() %}
          {% if key | as_datetime(0) != 0 %}
              {% set obj  = { "time" : key, "price" : val | round(7) } %}
              {% set data.prices = data.prices + [obj] %}
          {% endif %}
          {% endfor %}
          {{ data.prices }}
```

</details>

<br>Such sensor has to be published to our postgresql database using LTSS component. It will require adding it to the `LTSS` config and restarting the HA.

The last step is to offload price data from the `ltss` table inserting them to pricing dedicated table. This task is executed by the trigger that picks data from the JSON structure, nd puts it to the destination table.
See _'Example of trigger processing SPOT prices'_ in [SQL Code](#sql-code) section bellow.


## Presentation in Grafana

### Price Visualization

Once prices are in the table, you can visualize them.
![Grafana Year of Prices](images/grafana-year-of-prices-ote.png)

The graph above shows evolution of all price items by a year. As you can see, since November, the sale energy price changed drastically. It's because I've changed a fixed tarrif in favor of spot prices.

We need to provide the Grafana with data series. For this case We are going to tacle with two categories of price records:
* **Irregular, infrequently changing prices**: For visualization it's required to generate a continuous time series out of original price points
* **Spot prices**: These are already recorded as regular, ready to visualize, time series.

For improved performance, code clarity, and reusability, construct individual queries for each category and combine their results using `UNION`. By grouping the data by trade type, you will generate distinct series for sales and purchases, resulting in a chart similar to the example above.

<details>
<summary>SQL query to fetch prices to Grafana</summary>

```sql
SELECT time as time, 1000 * price_value * COALESCE(1+fee_value,1) AS price, format('%s (%s)', ec.trade_type, ec.price_name) as type
FROM ltss_energy_ote.electricity_prices AS ec
JOIN generate_series(to_timestamp($__from/1000)::DATE::TIMESTAMPTZ, to_timestamp($__to/1000)::TIMESTAMPTZ + '1d'::INTERVAL, '1 d'::INTERVAL) AS x(time) ON TRUE
LEFT JOIN ltss_energy_ote.electricity_fees AS efr ON efr.trade_type = ec.trade_type AND efr.price_name = ec.price_name AND ec.price_period <@ efr.fee_period
WHERE x.time <@ price_period
  AND NOT ((ec.trade_type, ec.price_name) IN (('Sale', 'Energy')) AND LOWER(ec.price_period) >= '2025-11-01 0:0'::TIMESTAMPTZ)

UNION

SELECT LOWER(price_period) AS time, SUM(1000 * price_value * COALESCE(1+fee_value,1)) AS price, format('%s (%s)', ec.trade_type, ec.price_name) AS type
FROM ltss_energy_ote.electricity_prices AS ec
LEFT JOIN ltss_energy_ote.electricity_fees AS efr ON efr.trade_type = ec.trade_type AND efr.price_name = ec.price_name AND ec.price_period <@ efr.fee_period
WHERE LOWER(price_period) BETWEEN to_timestamp($__from/1000)::DATE::TIMESTAMPTZ AND to_timestamp($__to/1000)::TIMESTAMPTZ
AND (ec.trade_type, ec.price_name) IN (('Sale', 'Energy')) AND LOWER(ec.price_period) >= '2025-11-01 0:0'::TIMESTAMPTZ
GROUP BY 1, 3

ORDER BY type DESC, time
```

The first query, with use of generate_series() function, generates a time points with daily frequency for given time period with values reflecting prices that change infrequently. It excludes energy sale prices since we expect them to be recorded hourly.

The second subquery specifically returns sale energy prices since November, 1st. These data assumed to be periodic (hourly) records.

Both subqueries join with price-related fees to present the final prices, rather than just the net values.

If you buy and sell energy on the spot market, you might consider including the `('Purchase', 'Energy')` pair to both query conditions.

If you do not use periodically recorded prices, you can omit the second subquery and remove the related condition from the first one. In this scenario, it might be reasonable to display all prices - including their individual components - within a single graph.

</details>
<br>
Having query showing all components, it's easy to achieve sale vs purchase prices trend. It can be done on Grafana side as well as by SQL query itself. SQL provide below provides data for the latter option.

![Grafana year of prices ote compacted](images/grafana-year-of-prices-ote2.png)

<details>
<summary>SQL for data of pricing compacted to sale and purchase</summary>


```sql
  SELECT time as time, 1000 * SUM(price_value * COALESCE(1+efr.fee_value,1)) AS price, ec.trade_type  as type
  FROM ltss_energy_ote.electricity_prices AS ec
  JOIN generate_series(to_timestamp($__from/1000)::DATE::TIMESTAMPTZ, to_timestamp($__to/1000)::TIMESTAMPTZ + '1d'::INTERVAL, '1 d'::INTERVAL) AS x(time) ON TRUE
  LEFT JOIN ltss_energy_ote.electricity_fees AS efr ON efr.trade_type = ec.trade_type AND efr.price_name = ec.price_name AND ec.price_period <@ efr.fee_period
  WHERE x.time <@ price_period
    AND NOT ((ec.trade_type, ec.price_name) IN (('Sale', 'Energy')) AND LOWER(ec.price_period) >= '2025-11-01 0:0'::TIMESTAMPTZ)
  GROUP by 1, 3

UNION

SELECT LOWER(ec.price_period) AS time, SUM(1000 * (ec.price_value * ec.dir + ec2.price_value * ec2.dir) * COALESCE(1+fee_value,1)) AS price, ec.trade_type AS type
FROM ltss_energy_ote.electricity_prices AS ec
LEFT JOIN ltss_energy_ote.electricity_fees AS efr ON efr.trade_type = ec.trade_type AND efr.price_name = ec.price_name AND ec.price_period <@ efr.fee_period
LEFT JOIN ltss_energy_ote.electricity_prices AS ec2 ON (ec.trade_type = ec2.trade_type) AND ec.price_name <> ec2.price_name AND ec.price_period <@ ec2.price_period
WHERE LOWER(ec.price_period) BETWEEN to_timestamp($__from/1000)::DATE::TIMESTAMPTZ AND to_timestamp($__to/1000)::TIMESTAMPTZ
AND (ec.trade_type, ec.price_name) = ('Sale', 'Energy') AND LOWER(ec.price_period) >= '2025-11-01 0:0'::TIMESTAMPTZ
GROUP BY 1, 3

ORDER BY 1, 3 DESC
```

</details>

In both examples, values are multiplied by 1000 to convert units to MWh. This scaling aligns the data with standard price sheets, making comparisons clearer and more intuitive.


## SQL code


<details>
<summary>SQL script creating pricing tables and functions</summary>

```sql
CREATE SCHEMA ltss_energy_ote;
GRANT USAGE ON SCHEMA ltss_energy_ote TO public;

CREATE TABLE IF NOT EXISTS ltss_energy_ote.electricity_prices
(
    trade_type  TEXT NOT NULL,
    price_name  TEXT NOT NULL,
    price_period TSTZRANGE NOT NULL,
    price_value NUMERIC NOT NULL,
    dir INTEGER NOT NULL,
    volume_unit  TEXT NOT NULL,
    CONSTRAINT pk_electricityprices PRIMARY KEY (trade_type, price_name, price_period),
    CONSTRAINT xc_electricityprices_unique EXCLUDE USING gist (trade_type WITH =, price_name WITH =, price_period WITH &&),
    CONSTRAINT ck_electricityprices_tradetype CHECK (trade_type IN ('Sale', 'Purchase')),
    CONSTRAINT ck_electricityprices_dir CHECK (dir IN (-1, 1))
);

COMMENT ON TABLE ltss_energy_ote.electricity_prices IS 'Net prices for an energy volume';
COMMENT ON COLUMN ltss_energy_ote.electricity_prices.trade_type IS $$Either 'Sale' or 'Purchase'$$;
COMMENT ON COLUMN ltss_energy_ote.electricity_prices.price_name IS $$Name assigned to the operation. The 'Energy' is reserved for electric energy. Other might be anything like 'Distribution', 'Maintenance' etc.$$;
COMMENT ON COLUMN ltss_energy_ote.electricity_prices.price_period IS 'Time period the price is valid for';
COMMENT ON COLUMN ltss_energy_ote.electricity_prices.price_value IS 'The value of price for the volume';
COMMENT ON COLUMN ltss_energy_ote.electricity_prices.dir IS 'Affects summing. 1 means the value adds to the total. -1 deducts from the total and might be used for handling fee for the operator';
COMMENT ON COLUMN ltss_energy_ote.electricity_prices.volume_unit IS 'kWh. Informative value just to emphasize that all values are for the same unit';

CREATE TABLE IF NOT EXISTS ltss_energy_ote.electricity_fees
(
    trade_type  TEXT NOT NULL,
    price_name  TEXT NOT NULL,
    fee_name    TEXT NOT NULL,
    fee_period  TSTZRANGE NOT NULL,
    fee_value   NUMERIC NOT NULL,
    dir         INTEGER NOT NULL,
    CONSTRAINT pk_electricityrates PRIMARY KEY (trade_type, price_name, fee_name, fee_period),
    CONSTRAINT xc_electricityrates_unique EXCLUDE USING gist (trade_type WITH =, price_name WITH =, fee_name WITH =, fee_period WITH &&),
    CONSTRAINT ck_electricityrates_tradetype CHECK (trade_type IN ('Sale', 'Purchase')),
    CONSTRAINT ck_electricityrates_dir CHECK (dir IN (-1, 1))
);

COMMENT ON TABLE ltss_energy_ote.electricity_fees IS 'Fees to be calculated from the net value of energy volume';
COMMENT ON COLUMN ltss_energy_ote.electricity_fees.trade_type IS $$Either 'Sale' or 'Purchase'$$;
COMMENT ON COLUMN ltss_energy_ote.electricity_fees.price_name IS 'Name assigned to the operation. Together with trade_type must match the pair from `electricity_prices` table.';
COMMENT ON COLUMN ltss_energy_ote.electricity_fees.fee_period IS 'Time period the price is valid for';
COMMENT ON COLUMN ltss_energy_ote.electricity_fees.fee_value IS 'The value of price for the volume';
COMMENT ON COLUMN ltss_energy_ote.electricity_fees.dir IS 'wip';

GRANT SELECT ON TABLE ltss_energy_ote.electricity_rates, ltss_energy_ote.electricity_prices TO public;

CREATE OR REPLACE AGGREGATE public.nmul(NUMERIC) (
	SFUNC = numeric_mul,
	STYPE = numeric
);

COMMENT ON FUNCTION public.nmul(NUMERIC) IS 'Implements product aggregation (multiplicative reduction).';

CREATE OR REPLACE FUNCTION ltss_energy_ote.calculate_cost
(
    _trade_type TEXT,
    _time       TIMESTAMPTZ,
    _value      NUMERIC,
    _incl_pname TEXT[] DEFAULT NULL,
    _excl_pname TEXT[] DEFAULT NULL
)
RETURNS NUMERIC
LANGUAGE sql
STABLE
AS $f$

    SELECT
        trim_scale(SUM(_value * price_value * dir)) AS val_total
    FROM ltss_energy_ote.electricity_prices AS p
    WHERE _time <@ price_period
      AND trade_type       = _trade_type
      AND p.price_name     = ANY(COALESCE(_incl_pname, Array[price_name]))
	  AND NOT p.price_name = ANY(COALESCE(_excl_pname, ARRAY[]::text[]))

$f$;

COMMENT ON FUNCTION ltss_energy_ote.calculate_cost(TEXT, TIMESTAMPTZ, NUMERIC, TEXT[], TEXT[]) IS 'Helper function for the first level costs CAGG. It calculates price of energy volume using prices valid at given time';


CREATE OR REPLACE FUNCTION ltss_energy_ote.calculate_fee
(
	_trade_type TEXT,
	_time       TIMESTAMPTZ,
	_value      NUMERIC,
	_incl_pname TEXT[] DEFAULT NULL,
	_excl_pname TEXT[] DEFAULT NULL,
	_incl_fname TEXT[] DEFAULT NULL,
	_excl_fname TEXT[] DEFAULT NULL
)
RETURNS NUMERIC
LANGUAGE sql
STABLE
AS $f$

    SELECT COALESCE(trim_scale(_value * SUM(value)), 0) AS value
    FROM
    (
        SELECT p.price_value * COALESCE((1 - public.nmul(1 - f.fee_value)), 1) AS value
        FROM ltss_energy_ote.electricity_prices AS p
        JOIN ltss_energy_ote.electricity_fees   AS f ON (p.trade_type, p.price_name) = (f.trade_type, f.price_name)
        WHERE _time <@ p.price_period
          AND _time <@ f.fee_period
          AND p.trade_type      = _trade_type
          AND p.price_name      = ANY(COALESCE(_incl_pname, ARRAY[p.price_name]))
          AND NOT p.price_name  = ANY(COALESCE(_excl_pname, ARRAY[]::TEXT[]))
          AND f.fee_name        = ANY(COALESCE(_incl_fname, ARRAY[f.fee_name]))
          AND NOT f.fee_name    = ANY(COALESCE(_excl_fname, ARRAY[]::TEXT[]))
        GROUP BY p.trade_type, p.price_name, p.price_period, p.price_value
    ) AS sub

$f$;

COMMENT ON FUNCTION ltss_energy_ote.calculate_fee(TEXT, TIMESTAMPTZ, NUMERIC, TEXT[], TEXT[], TEXT[], TEXT[]) IS 'Helper function for the first level costs CAGG. It calculates a fee deducted from energy volume price at given time';

```
</details>



<details>
<summary>Example of trigger processing SPOT prices</summary>

```sql
CREATE TABLE IF NOT EXISTS ltss_energy_ote.trades_import
(
    trade_type  TEXT NOT NULL,
    trade_period TSTZRANGE NOT NULL,
    CONSTRAINT pk_electricityprices PRIMARY KEY (trade_type, price_period),
    CONSTRAINT xc_electricityprices_unique EXCLUDE USING gist (trade_type WITH =, price_period WITH &&),
    CONSTRAINT ck_electricityprices_tradetype CHECK (trade_type IN ('Sale', 'Purchase'))
);

GRANT SELECT ON TABLE ltss_energy_ote.trades_import TO public;

CREATE OR REPLACE FUNCTION ltss_energy_ote.tr_ltss_oteprices()
    RETURNS trigger
    LANGUAGE 'plpgsql'
    SECURITY DEFINER
AS $BODY$

DECLARE
    err_msg    TEXT;
    err_code   TEXT;
	err_hint   TEXT;
	err_detail TEXT;
	err_ctx	   TEXT;
    ENTITYID CONSTANT TEXT = 'sensor.pv_ctrl_spot_electricity_prices';
BEGIN
    -- THIS TRIGGER FUNCTION IS USED on public.ltss table

    IF NEW.entity_id <> ENTITYID THEN
        RETURN NULL;
    END IF;

    INSERT INTO ltss_energy_ote.electricity_prices AS ep
    (
        trade_type,
        price_name,
        price_period,
        price_value,
        volume_unit,
		dir
    )
    SELECT DISTINCT
        t.trade_type,
        'Energy',
        tstzrange((j->>'time')::TIMESTAMPTZ, (j->>'time')::TIMESTAMPTZ + (NEW.attributes->>'period')::INTERVAL, '[)'),
        (j->'price')::NUMERIC,
        'kWh',
		1
    FROM jsonb_array_elements(NEW.attributes->'data') AS j,
         ltss_energy_ote.trade_import AS t
    WHERE (j->>'time')::TIMESTAMPTZ <@ trade_period
    ON CONFLICT ON CONSTRAINT pk_electricityprices
    DO UPDATE
    SET price_value = EXCLUDED.price_value
    WHERE ep.price_value <> EXCLUDED.price_value;

    RETURN NULL;

EXCEPTION WHEN others THEN
    GET STACKED DIAGNOSTICS err_msg    = MESSAGE_TEXT,
                            err_code   = RETURNED_SQLSTATE,
							err_hint   = PG_EXCEPTION_HINT,
							err_detail = PG_EXCEPTION_DETAIL,
							err_ctx    = PG_EXCEPTION_CONTEXT;

    RAISE WARNING USING
		MESSAGE = format('[%s], %s\nCONTEXT:\n%s', err_code, err_msg, err_ctx),
		DETAIL = err_detail,
		HINT = err_hint;
    RETURN NULL;
END;
$BODY$;

CREATE OR REPLACE TRIGGER tr_ltss_oteprices
AFTER INSERT
ON public.ltss
FOR EACH ROW
EXECUTE FUNCTION ltss_energy_ote.tr_ltss_oteprices();
```
</details>




<details>
<summary>CAGGs code</summary>

```sql
CREATE OR REPLACE FUNCTION ltss_energy_ote.get_entities_for_cagg_energy_qhourly()
RETURNS TEXT[]
LANGUAGE sql
IMMUTABLE
AS $f$


   SELECT ARRAY
       [
            -- replace sensor names with your own.
           'sensor.pg_mainhouse_total_energy_energy_quarter_hourly', -- consumption
           'sensor.pg_cube_total_energy_energy_quarter_hourly',      -- consumption
           'sensor.energy_injected_quarter_hourly',                  -- injected to grid
           'sensor.energy_purchased_quarter_hourly',                 -- purchased from grid
           'sensor.wattsonic_pv1_input_energy_2_quarter_hourly',     -- PV string 1 production
           'sensor.wattsonic_pv2_input_energy_2_quarter_hourly',     -- PV string 2 production
           'sensor.energy_discharged_from_battery_quarter_hourly',   -- Discharged from battery
           'sensor.energy_charged_to_battery_quarter_hourly'         -- Charged to battery
           -- Other energy sensors you want to aggregate
       ];
$f$;

CREATE OR REPLACE FUNCTION ltss_energy_ote.get_entities_for_cagg_costs_qhourly()
RETURNS TEXT[]
LANGUAGE sql
IMMUTABLE
AS $f$


   SELECT ARRAY
       [
            -- replace sensor names with your own.
           'sensor.pg_mainhouse_total_energy', -- consumption
           'sensor.pg_cube_total_energy',      -- consumption
           'sensor.energy_injected',           -- injected to grid
           'sensor.energy_purchased'           -- purchased from grid
       ];
$f$;


CREATE OR REPLACE FUNCTION ltss_energy_ote.normalize_energy_entity_name(entity_id TEXT)
RETURNS TEXT
LANGUAGE sql
IMMUTABLE
AS $BODY$

   SELECT replace(
            regexp_replace(entity_id, '(_(quarter_)?hourly)$', ''),
            '_energy_energy',
            '_energy'
           );
$BODY$;

COMMENT ON FUNCTION ltss_energy_ote.normalize_energy_entity_name(TEXT) IS 'Removes suffixes like `quarter_hourly` from entity name. These suffixes are no more relevant, since the names land in all level CAGGs (ie daily). Also it makes Grafana views nicer';


CREATE MATERIALIZED VIEW ltss_energy_ote.cagg_energy_qhourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('15m'::INTERVAL, "time", 'Europe/Prague') AS bucket,
    ltss_energy_ote.normalize_energy_entity_name(entity_id),
    delta(counter_agg("time", state::DOUBLE PRECISION))::NUMERIC AS energy
FROM ltss
WHERE entity_id = ANY (ltss_energy_ote.get_entities_for_cagg_energy_qhourly())
  AND state NOT IN ('unavailable', 'unknown')
GROUP BY 1, 2
WITH NO DATA;

-- create hourly CAGG based on quarter hourly one
CREATE MATERIALIZED VIEW ltss_energy_ote.cagg_energy_hourly
WITH (timescaledb.continuous) AS
SELECT
   time_bucket('1h'::INTERVAL, bucket, 'Europe/Prague') AS bucket,
   entity_id,
   SUM(energy) AS energy
FROM ltss_energy_ote.cagg_energy_qhourly
GROUP BY 1, 2
WITH NO DATA;

-- create daily CAGG based on hourly one
CREATE MATERIALIZED VIEW ltss_energy_ote.cagg_energy_daily
WITH (timescaledb.continuous) AS
SELECT
   time_bucket('1d'::INTERVAL, bucket, 'Europe/Prague') AS bucket,
   entity_id,
   SUM(energy) AS energy
FROM ltss_energy_ote.cagg_energy_hourly
GROUP BY 1, 2
WITH NO DATA;

CREATE MATERIALIZED VIEW ltss_energy_ote.cagg_costs_qhourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('15m'::INTERVAL, bucket, 'Europe/Prague') AS bucket,
    entity_id,
    ltss_energy_ote.calculate_cost('Purchase', MAX(bucket), SUM(energy)) + ltss_energy_ote.calculate_fee('Purchase', MAX(bucket), SUM(energy))
    AS purchase_cost,

    ltss_energy_ote.calculate_cost('Purchase', MAX(bucket), SUM(energy), _excl_pname => Array['Energy']) + ltss_energy_ote.calculate_fee('Purchase', MAX(bucket), SUM(energy), _excl_pname => Array['Energy'])
    AS purchase_trading_cost,

    ltss_energy_ote.calculate_cost('Purchase', MAX(bucket), SUM(energy), _incl_pname => Array['Energy'])
    AS purchase_energy_cost,

    ltss_energy_ote.calculate_cost('Sale', MAX(bucket), SUM(energy)) - ltss_energy_ote.calculate_fee('Sale', MAX(bucket), SUM(energy))
    AS sale_income, -- net income

    ltss_energy_ote.calculate_cost('Sale', MAX(bucket), SUM(energy), _excl_pname => Array['Energy']) + ltss_energy_ote.calculate_fee('Sale', MAX(bucket), SUM(energy), _excl_pname => Array['Energy'])
    AS sale_trading_cost, -- trading costs

    ltss_energy_ote.calculate_cost('Sale', MAX(bucket), SUM(energy), _incl_pname => Array['Energy'])
    AS sale_energy_cost -- net cost of sold energy

FROM ltss_energy_ote.cagg_energy_hourly
WHERE entity_id = ANY (ltss_energy_ote.get_entities_for_cagg_costs_qhourly())
GROUP BY 1, 2
WITH NO DATA;


CREATE MATERIALIZED VIEW ltss_energy_ote.cagg_costs_hourly
WITH (timescaledb.continuous) AS
SELECT
   time_bucket('1h'::INTERVAL, bucket, 'Europe/Prague') AS bucket,
   entity_id,
   SUM(purchase_cost)           AS purchase_cost,          -- total cost = spot+fee+tax
   SUM(purchase_trading_cost)   AS purchase_trading_cost,  -- totalcost-(energy*tax)
   SUM(purchase_energy_cost)    AS purchase_energy_cost,   -- net cost (ie spot)
   SUM(sale_income)             AS sale_income,            -- total cost = spot-fee (- potential taxes)
   SUM(sale_trading_cost)       AS sale_trading_cost,      -- totalcost-(energy*tax)
   SUM(sale_energy_cost)        AS sale_energy_cost        -- net cost (ie spot)
FROM ltss_energy_ote.cagg_energy_qhourly
GROUP BY 1, 2
WITH NO DATA;

CREATE MATERIALIZED VIEW ltss_energy_ote.cagg_costs_daily
WITH (timescaledb.continuous) AS
SELECT
   time_bucket('1d'::INTERVAL, bucket, 'Europe/Prague') AS bucket,
   entity_id,
   SUM(purchase_cost)           AS purchase_cost,          -- total cost = spot+fee+tax
   SUM(purchase_trading_cost)   AS purchase_trading_cost,  -- totalcost-(energy*tax)
   SUM(purchase_energy_cost)    AS purchase_energy_cost,   -- net cost (ie spot)
   SUM(sale_income)             AS sale_income,            -- total cost = spot-fee (- potential taxes)
   SUM(sale_trading_cost)       AS sale_trading_cost,      -- totalcost-(energy*tax)
   SUM(sale_energy_cost)        AS sale_energy_cost        -- net cost (ie spot)
FROM ltss_energy_ote.cagg_energy_hourly
GROUP BY 1, 2
WITH NO DATA;

-- Grant read access to everyone connected
GRANT SELECT ON TABLE
    ltss_energy_ote.cagg_energy_qhourly,
    ltss_energy_ote.cagg_energy_hourly,
    ltss_energy_ote.cagg_energy_daily,
    ltss_energy_ote.cagg_costs_qhourly,
    ltss_energy_ote.cagg_costs_hourly,
    ltss_energy_ote.cagg_costs_daily
    TO public;

-- make CAGGs real-time
ALTER MATERIALIZED VIEW ltss_energy_ote.cagg_energy_qhourly
SET (timescaledb.materialized_only = FALSE);

ALTER MATERIALIZED VIEW ltss_energy_ote.cagg_energy_hourly
SET (timescaledb.materialized_only = FALSE);

ALTER MATERIALIZED VIEW ltss_energy_ote.cagg_energy_daily
SET (timescaledb.materialized_only = FALSE);

ALTER MATERIALIZED VIEW ltss_energy_ote.cagg_costs_qhourly
SET (timescaledb.materialized_only = FALSE);

ALTER MATERIALIZED VIEW ltss_energy_ote.cagg_costs_hourly
SET (timescaledb.materialized_only = FALSE);

ALTER MATERIALIZED VIEW ltss_energy_ote.cagg_costs_daily
SET (timescaledb.materialized_only = FALSE);

-- start CAGGs refreshing automatically
SELECT add_continuous_aggregate_policy
(
   'ltss_energy_ote.cagg_energy_qhourly', '1h'::INTERVAL, '15m'::INTERVAL, '5m'::INTERVAL
);
SELECT add_continuous_aggregate_policy
(
   'ltss_energy_ote.cagg_energy_hourly', '3h'::INTERVAL, '1h'::INTERVAL, '30m'::INTERVAL
);
SELECT add_continuous_aggregate_policy
(
   'ltss_energy_ote.cagg_energy_daily', '3d'::INTERVAL, '1d'::INTERVAL, '12h'::INTERVAL
);

SELECT add_continuous_aggregate_policy
(
   'ltss_energy_ote.cagg_costs_qhourly', '1h'::INTERVAL, '15m'::INTERVAL, '5m'::INTERVAL
);
SELECT add_continuous_aggregate_policy
(
   'ltss_energy_ote.cagg_costs_hourly', '3h'::INTERVAL, '1h'::INTERVAL, '30m'::INTERVAL
)
SELECT add_continuous_aggregate_policy
(
   'ltss_energy_ote.cagg_costs_daily', '3d'::INTERVAL, '1d'::INTERVAL, '12h'::INTERVAL
);
```
</details>
