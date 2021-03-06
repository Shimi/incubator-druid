<!--
  ~ Licensed to the Apache Software Foundation (ASF) under one
  ~ or more contributor license agreements.  See the NOTICE file
  ~ distributed with this work for additional information
  ~ regarding copyright ownership.  The ASF licenses this file
  ~ to you under the Apache License, Version 2.0 (the
  ~ "License"); you may not use this file except in compliance
  ~ with the License.  You may obtain a copy of the License at
  ~
  ~   http://www.apache.org/licenses/LICENSE-2.0
  ~
  ~ Unless required by applicable law or agreed to in writing,
  ~ software distributed under the License is distributed on an
  ~ "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  ~ KIND, either express or implied.  See the License for the
  ~ specific language governing permissions and limitations
  ~ under the License.
  -->

---
layout: doc_page
---

# Schema Design

This page is meant to assist users in designing a schema for data to be ingested in Druid. Druid intakes denormalized data 
and columns are one of three types: a timestamp, a dimension, or a measure (or a metric/aggregator as they are 
known in Druid). This follows the [standard naming convention](https://en.wikipedia.org/wiki/Online_analytical_processing#Overview_of_OLAP_systems) 
of OLAP data.

For more detailed information:

* Every row in Druid must have a timestamp. Data is always partitioned by time, and every query has a time filter. Query results can also be broken down by time buckets like minutes, hours, days, and so on.
* Dimensions are fields that can be filtered on or grouped by. They are always single Strings, arrays of Strings, single Longs, single Doubles or single Floats.
* Metrics are fields that can be aggregated. They are often stored as numbers (integers or floats) but can also be stored as complex objects like HyperLogLog sketches or approximate histogram sketches.

Typical production tables (or datasources as they are known in Druid) have fewer than 100 dimensions and fewer 
than 100 metrics, although, based on user testimony, datasources with thousands of dimensions have been created.

Below, we outline some best practices with schema design:

## Numeric dimensions

If the user wishes to ingest a column as a numeric-typed dimension (Long, Double or Float), it is necessary to specify the type of the column in the `dimensions` section of the `dimensionsSpec`. If the type is omitted, Druid will ingest a column as the default String type.

There are performance tradeoffs between string and numeric columns. Numeric columns are generally faster to group on
than string columns. But unlike string columns, numeric columns don't have indexes, so they are generally slower to
filter on.

See [Dimension Schema](../ingestion/index.html#dimension-schema) for more information.

## High cardinality dimensions (e.g. unique IDs)

In practice, we see that exact counts for unique IDs are often not required. Storing unique IDs as a column will kill 
[roll-up](../ingestion/index.html#rollup), and impact compression. Instead, storing a sketch of the number of the unique IDs seen, and using that 
sketch as part of aggregations, will greatly improve performance (up to orders of magnitude performance improvement), and significantly reduce storage. 
Druid's `hyperUnique` aggregator is based off of Hyperloglog and can be used for unique counts on a high cardinality dimension. 
For more information, see [here](https://www.youtube.com/watch?v=Hpd3f_MLdXo).

## Nested dimensions

At the time of this writing, Druid does not support nested dimensions. Nested dimensions need to be flattened. For example, 
if you have data of the following form:
 
```
{"foo":{"bar": 3}}
```
 
then before indexing it, you should transform it to:

```
{"foo_bar": 3}
```

Druid is capable of flattening JSON input data, please see [Flatten JSON](../ingestion/flatten-json.html) for more details.

## Counting the number of ingested events

A count aggregator at ingestion time can be used to count the number of events ingested. However, it is important to note 
that when you query for this metric, you should use a `longSum` aggregator. A `count` aggregator at query time will return 
the number of Druid rows for the time interval, which can be used to determine what the roll-up ratio was.

To clarify with an example, if your ingestion spec contains:

```
...
"metricsSpec" : [
      {
        "type" : "count",
        "name" : "count"
      },
...
```

You should query for the number of ingested rows with:

```
...
"aggregations": [
    { "type": "longSum", "name": "numIngestedEvents", "fieldName": "count" },
...
```

## Schema-less dimensions

If the `dimensions` field is left empty in your ingestion spec, Druid will treat every column that is not the timestamp column, 
a dimension that has been excluded, or a metric column as a dimension.

Note that when using schema-less ingestion, all dimensions will be ingested as String-typed dimensions.

## Including the same column as a dimension and a metric

One workflow with unique IDs is to be able to filter on a particular ID, while still being able to do fast unique counts on the ID column. 
If you are not using schema-less dimensions, this use case is supported by setting the `name` of the metric to something different than the dimension. 
If you are using schema-less dimensions, the best practice here is to include the same column twice, once as a dimension, and as a `hyperUnique` metric. This may involve 
some work at ETL time.

As an example, for schema-less dimensions, repeat the same column:

```
{"device_id_dim":123, "device_id_met":123}
```

and in your `metricsSpec`, include:
 
```
{ "type" : "hyperUnique", "name" : "devices", "fieldName" : "device_id_met" }
```

`device_id_dim` should automatically get picked up as a dimension.
