# QSDB Documentation

QSDB, in its current alpha stage, comprises:
* a soft specification/schema of how to format and organize your personal data
* suggested tools for storing your data
* scripts and configuration files for deploying those tools


## Foundations

QSDB uses [InfluxDB](https://www.influxdata.com/) for data storage.

* Why InfluxDB?
  - [`influxdb`](https://github.com/influxdata/influxdb) is pure Go, open source, and very popular: ★ 9,582 / ⑂ 1,351
    + ⭐ Simple internal data model; easy-to-use HTTP API and query language ([InfluxQL](https://docs.influxdata.com/influxdb/v1.1/query_language/spec/))
    + 💯 Everything is timestamped
    + 😍 Space-efficient long-term storage
    + 🤑 Commercially-driven; but, beyond the enterprise support model, it's pretty clear what the value-add of the paid version is: distributed deployment
    + Plays nicely with other time-series protocols, e.g., [Graphite](https://graphiteapp.org/), [collectd](https://collectd.org/), [StatsD](https://github.com/etsy/statsd)
    + InfluxData has built their entire [`TICK` stack](https://www.influxdata.com/use-cases/introducing-the-tick-stack/) to work well in concert.
      * `T`elegraf, which is great for harvesting and ingesting data from system metrics and log files
      * `I`nfluxDB, the core data storage and retrieval mechanism
      * `C`hronograf shows promise, but Grafana is currently a better product
      * `K`apacitor is for post-processing (anomaly detection, alerting) time-series data, but I haven't used it for anything yet
* Why not ...?
  - [Prometheus](https://prometheus.io/)
    + [`prometheus`](https://github.com/prometheus/prometheus) is also pure Go, and quite popular: ★ 7,574 / ⑂ 742
    + ✨ No commercial component, so, no only-for-pay features
    + 😬 But, no commercial component also means continued maintenance and updates depend on the community
    + 😒 Designed for non-distributed architecture. So, no effective difference from the free distribution of InfluxDB, but also no option to pay for cluster scale-out if needed.
    + 😵 **Numeric data only**!
    + See [Comparison to alternatives](https://prometheus.io/docs/introduction/comparison/) on Prometheus's site.
  - [OpenTSDB](http://opentsdb.net/)
    + [`opentsdb`](https://github.com/OpenTSDB/opentsdb) is pure Java, and relatively popular: ★ 2,315 / ⑂ 756
    + It's built on Hadoop / HBase, so it's distributed from the ground up and designed to scale. The consensus seems to be that:
      * 😊 if you already maintain an HBase cluster, OpenTSDB is a great fit,
      * 😫 but otherwise, there's a lot of overhead, and it's **overly complex to set up for simple needs**


### [InfluxDB Data Types](https://docs.influxdata.com/influxdb/v1.1/write_protocols/line_protocol_reference/#data-types)

InfluxDB data model:

* A **database** is like a typical SQL database: data in one database usually has nothing to do with data in another database, you can only be "connected" to one database at a time, and all databases are stored separately in the underlying file system representations.
  - You must explicitly "CREATE DATABASE xyz" before using it.
* A **measurement** is like a typical SQL table; a database will contain multiple measurements, which are created/extended dynamically as needed.
* A **point** is the basic InfluxDB data structure, like a row in typical SQL.
  - A point belongs to a single measurement.
  - A point can have **tags**, which are key-value pairs; both the key and the value are stored as strings
    + Tag keys and values are stored as lookups in the corresponding measurement's metadata, so even long tag values are cheap to store, but they shouldn't be used for values with a lot of variance. They are like SQL's ENUM type, but with new values added dynamically as needed.
    + Tags are indexed, so they are useful for common `WHERE ... = "..."` clauses.
    + You cannot `GROUP BY` tag values.
  - A point has **fields**, which are key-value pairs.
    + It appears that a point must have at least one field (though this seems to be a limitation of the protocol rather than the internal representation, and I can imagine
    + Like tag keys, field keys are stored as strings, and are like typical SQL columns.
    + Field value types are inferred on first use; subsequent writes that use a different type will cause a `field type conflict` error.
  - A point always has a **timestamp**, natively stored as integer nanoseconds since the UNIX epoch
    + If no timestamp is given, the point will be assigned the server's internal time.
    + The required timestamp uses the special name "time" (which operates as if there were a integer-type `time` field).
    + When writing data to InfluxDB, you can specify a timestamp precision; this is purely to facilitate better compression, and has no semantics.
      I.e., you cannot retrieve the precision a point's timestamp was stored with.

Data is usually written to the InfluxDB server over HTTP in [Line Protocol](https://docs.influxdata.com/influxdb/v1.1/write_protocols/line_protocol_tutorial/) format. Basically, the format is (1) `,`-separated measurement + tags, (2) `,`-separated fields, and (3) optional timestamp, each separated by whitespace. The following examples are valid Line Protocol lines:

    msrmnt,tagx=xval,tagy=yval fieldk=true,fieldm=100i 1483983204980000000
    msrmnt fieldk=false 1483983204990000000
    msrmnt fieldm=-9i

InfluxDB **fields** can have one of the following values:
* `float`
  - IEEE-754 64-bit floating-point number
  - This is the default numerical type;
    number literals (in Line Protocol) without a trailing `i` are treated as `float`s,
    even if they are integers (no decimal anywhere)
* `integer`
  - Signed 64-bit integer (commonly/loosely known as `long` in other circles)
  - Range: -9223372036854775808 to 9223372036854775807
  - Must suffix the number literal with a trailing `i` to distinguish from the `float` type
* `string`
  - Length limit: 64KB
  - (That limit also applies to the other entities that have string types, namely, measurement names, tag keys and values, and field keys)
* `boolean`
  - Can be either TRUE or FALSE
  - TRUE literals in Line Protocol: `t`, `T`, `true`, `True`, `TRUE`
  - FALSE literals in Line Protocol: `f`, `F`, `false`, `False`, `FALSE`


# Database:

## `physical`

This is the database name I'll be using throughout. I also considered `health`, `self`, and `human`, among others, but "physical" seems the most descriptive of the lot, since I include a number of environmental factors in this database.


## Measurements:

### `heart`

Key      | Type     | Description
---------|----------|------------
`device` | string?  | **tag** the name of the electronic device reporting the measurement
`bpm`    | integer  | beats per minute (aka `rate`)
`rr`     | integer? | RR-interval in milliseconds (not 1/1024 of a second, as Bluetooth LE natively reports). It's empty on the first reported measurement in a given session

Example:

    heart,device=polarH7 bpm=64i,rr=99i

Note that tag values are never quoted

TODO:
* Maybe also include a `session_id=<string>` field, e.g., `session_id="2016-12-26a"`
  I.e., for doing `GROUP BY` aggregates?
  It has to be a field, since you cannot `GROUP BY` tag values.


### `location`

Key         | Type    | Description
------------|---------|------------
`device`    | string? | **tag** the name of the electronic device reporting the measurement
`longitude` | float   | Degrees easting (x-axis) in WGS 84 coordinate system
`latitude`  | float   | Degrees northing (y-axis) in WGS 84 coordinate system
`altitude`  | float?  | Altitude in meters, if known
`accuracy`  | float?  | Accuracy of geolocation in meters, if known (calculated from max of horizontal and vertical components if split)
`course`    | float?  | Compass direction currently heading, in non-negative degrees (modulus 360) clockwise from North (thus, 0.0 is due North, 90.0 is due East, 180.0 is due South, and 270.0 is due West)
`speed`     | float?  | Instantaneous speed, in meters / second

Example:

    heart,device=iPhone6S longitude=-97.7403,latitude=30.2747,accuracy=10.3,altitude=149,course=182.1,speed=3.32


## License

Copyright ©2017 Christopher Brown. Licensed [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
