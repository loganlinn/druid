---
layout: doc_page
---
Real-time Node
==============
Realtime Node Configuration
==============================
For general Realtime Node information, see [here](../design/realtime.html).

Runtime Configuration
---------------------

The realtime node uses several of the global configs in [Configuration](../configuration/index.html) and has the following set of configurations as well:

### Node Config

|Property|Description|Default|
|--------|-----------|-------|
|`druid.host`|The host for the current node. This is used to advertise the current processes location as reachable from another node and should generally be specified such that `http://${druid.host}/` could actually talk to this process|InetAddress.getLocalHost().getCanonicalHostName()|
|`druid.port`|This is the port to actually listen on; unless port mapping is used, this will be the same port as is on `druid.host`|8084|
|`druid.service`|The name of the service. This is used as a dimension when emitting metrics and alerts to differentiate between the various services|druid/realtime|

### Realtime Operation

|Property|Description|Default|
|--------|-----------|-------|
|`druid.publish.type`|Where to publish segments. Choices are "noop" or "metadata".|metadata|
|`druid.realtime.specFile`|File location of realtime specFile.|none|

### Storing Intermediate Segments

|Property|Description|Default|
|--------|-----------|-------|
|`druid.segmentCache.locations`|Where intermediate segments are stored. The maxSize should always be zero.|none|


### Query Configs

#### Processing

|Property|Description|Default|
|--------|-----------|-------|
|`druid.processing.buffer.sizeBytes`|This specifies a buffer size for the storage of intermediate results. The computation engine in both the Historical and Realtime nodes will use a scratch buffer of this size to do all of their intermediate computations off-heap. Larger values allow for more aggregations in a single pass over the data while smaller values can require more passes depending on the query that is being executed.|1073741824 (1GB)|
|`druid.processing.formatString`|Realtime and historical nodes use this format string to name their processing threads.|processing-%s|
|`druid.processing.numThreads`|The number of processing threads to have available for parallel processing of segments. Our rule of thumb is `num_cores - 1`, which means that even under heavy load there will still be one core available to do background tasks like talking with ZooKeeper and pulling down segments. If only one core is available, this property defaults to the value `1`.|Number of cores - 1 (or 1)|
|`druid.processing.columnCache.sizeBytes`|Maximum size in bytes for the dimension value lookup cache. Any value greater than `0` enables the cache. It is currently disabled by default. Enabling the lookup cache can significantly improve the performance of aggregators operating on dimension values, such as the JavaScript aggregator, or cardinality aggregator, but can slow things down if the cache hit rate is low (i.e. dimensions with few repeating values). Enabling it may also require additional garbage collection tuning to avoid long GC pauses.|`0` (disabled)|

#### General Query Configuration

##### GroupBy Query Config

|Property|Description|Default|
|--------|-----------|-------|
|`druid.query.groupBy.singleThreaded`|Run single threaded group By queries.|false|
|`druid.query.groupBy.maxIntermediateRows`|Maximum number of intermediate rows.|50000|
|`druid.query.groupBy.maxResults`|Maximum number of results.|500000|

##### Search Query Config

|Property|Description|Default|
|--------|-----------|-------|
|`druid.query.search.maxSearchLimit`|Maximum number of search results to return.|1000|

### Caching

You can optionally configure caching to be enabled on the realtime node by setting caching configs here.

|Property|Possible Values|Description|Default|
|--------|---------------|-----------|-------|
|`druid.realtime.cache.useCache`|true, false|Enable the cache on the realtime.|false|
|`druid.realtime.cache.populateCache`|true, false|Populate the cache on the realtime.|false|
|`druid.realtime.cache.unCacheable`|All druid query types|All query types to not cache.|`["groupBy", "select"]`|

See [cache configuration](caching.html) for how to configure cache settings.


For Real-time Node Configuration, see [Realtime Configuration](../configuration/realtime.html).

For Real-time Ingestion, see [Realtime Ingestion](../ingestion/realtime-ingestion.html).

Realtime nodes provide a realtime index. Data indexed via these nodes is immediately available for querying. Realtime nodes will periodically build segments representing the data they’ve collected over some span of time and transfer these segments off to [Historical](../design/historical.html) nodes. They use ZooKeeper to monitor the transfer and the metadata storage to store metadata about the transferred segment. Once transfered, segments are forgotten by the Realtime nodes.

### Running

```
io.druid.cli.Main server realtime
```
Segment Propagation
-------------------

The segment propagation diagram for real-time data ingestion can be seen below:

![Segment Propagation](../../img/segmentPropagation.png "Segment Propagation")

You can read about the various components shown in this diagram under the Architecture section (see the menu on the right). Note that some of the names are now outdated.

### Firehose

See [Firehose](../ingestion/firehose.html).

### Plumber

See [Plumber](../design/plumber.html)

Extending the code
------------------

Realtime integration is intended to be extended in two ways:

1.  Connect to data streams from varied systems ([Firehose](https://github.com/druid-io/druid-api/blob/master/src/main/java/io/druid/data/input/FirehoseFactory.java))
2.  Adjust the publishing strategy to match your needs ([Plumber](https://github.com/druid-io/druid/blob/master/server/src/main/java/io/druid/segment/realtime/plumber/PlumberSchool.java))

The expectations are that the former will be very common and something that users of Druid will do on a fairly regular basis. Most users will probably never have to deal with the latter form of customization. Indeed, we hope that all potential use cases can be packaged up as part of Druid proper without requiring proprietary customization.

Given those expectations, adding a firehose is straightforward and completely encapsulated inside of the interface. Adding a plumber is more involved and requires understanding of how the system works to get right, it’s not impossible, but it’s not intended that individuals new to Druid will be able to do it immediately.

HTTP Endpoints
--------------

The real-time node exposes several HTTP endpoints for interactions.

### GET

* `/status`

Returns the Druid version, loaded extensions, memory used, total memory and other useful information about the node.


### Real-time Node

Run:

```
io.druid.cli.Main server realtime
```

Hardware (this is a little overkill):

```
r3.8xlarge (Cores: 32, Memory: 244 GB, SSD - this hardware is way overkill for the real-time node but we choose it for simplicity)
```

JVM Configuration:

```
-server
-Xmx13g
-Xms13g
-XX:NewSize=2g
-XX:MaxNewSize=2g
-XX:MaxDirectMemorySize=9g
-XX:+UseConcMarkSweepGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+HeapDumpOnOutOfMemoryError

-Duser.timezone=UTC
-Dfile.encoding=UTF-8
-Djava.util.logging.manager=org.apache.logging.log4j.jul.LogManager
-Djava.io.tmpdir=/mnt/tmp

-Dcom.sun.management.jmxremote.port=17071
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

Runtime.properties:

```
druid.host=#{IP_ADDR}
druid.port=8080
druid.service=druid/realtime

druid.processing.buffer.sizeBytes=1073741824
druid.processing.numThreads=7

druid.server.http.numThreads=50

druid.monitoring.monitors=["io.druid.segment.realtime.RealtimeMetricsMonitor", "com.metamx.metrics.JvmMonitor"]
```
