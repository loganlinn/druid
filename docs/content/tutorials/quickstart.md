---
layout: doc_page
---

# Druid Quickstart

In this quickstart, we will download Druid, set up it up on a single machine, load some data, and query the data.

## Prerequisites

You will need:

  * Java 7 or higher
  * Linux, Mac OS X, or other Unix-like OS (Windows is not supported)
  * 8G of RAM
  * 2 vCPUs

On Mac OS X, you can use [Oracle's JDK
8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) to install
Java.

On Linux, your OS package manager should be able to help for Java. If your Ubuntu-
based OS does not have a recent enough version of Java, WebUpd8 offers [packages for those
OSes](http://www.webupd8.org/2012/09/install-oracle-java-8-in-ubuntu-via-ppa.html).

## Getting started

To install Druid, issue the following commands in your terminal: 

```bash
curl -O http://static.druid.io/artifacts/releases/druid-0.9.0-bin.tar.gz
tar -xzf druid-0.9.0-bin.tar.gz
cd druid-0.9.0
```

In the package, you should find:

* `LICENSE` - the license files.
* `conf/*` - template configurations for a clustered setup.
* `conf-quickstart/*` - configurations for this quickstart.
* `extensions/*` - all Druid extensions.
* `hadoop-dependencies/*` - Druid Hadoop dependencies.
* `lib/*` - all included software packages for core Druid.
* `quickstart/*` - files useful for this quickstart.

## Start up Zookeeper

Druid currently has a dependency on [Apache ZooKeeper](http://zookeeper.apache.org/) for distributed coordination. You'll 
need to download and run Zookeeper.

```bash
curl http://www.gtlib.gatech.edu/pub/apache/zookeeper/zookeeper-3.4.7/zookeeper-3.4.7.tar.gz -o $zookeeper-3.4.7.tar.gz
tar xzf $zookeeper-3.4.7.tar.gz
cd zookeeper-3.4.7
cp conf/zoo_sample.cfg conf/zoo.cfg
./bin/zkServer.sh start
```

## Start up Druid services

With Zookeeper running, we can now start up the Druid processes.

```bash
java `cat conf-quickstart/historical/jvm.config | xargs` -cp conf-quickstart/_common:conf-quickstart/historical:lib/*.jar io.druid.cli.Main server historical > var/druid/log/historical.log
java `cat conf-quickstart/broker/jvm.config | xargs` -cp conf-quickstart/_common:conf-quickstart/broker:lib/*.jar io.druid.cli.Main server broker > var/druid/log/broker.log
java `cat conf-quickstart/coordinator/jvm.config | xargs` -cp conf-quickstart/_common:conf-quickstart/coordinator:lib/*.jar io.druid.cli.Main server coordinator > var/druid/log/coordinator.log
java `cat conf-quickstart/overlord/jvm.config | xargs` -cp conf-quickstart/_common:conf-quickstart/overlord:lib/*.jar io.druid.cli.Main server overlord > var/druid/log/overlord.log
java `cat conf-quickstart/middleManager/jvm.config | xargs` -cp conf-quickstart/_common:conf-quickstart/middleManager:lib/*.jar io.druid.cli.Main server middleManager > var/druid/log/middleManager.log
```

You should see a log message printed out for each service that starts up. You can view detailed logs
for any service by looking in the `var/log/` directory using another terminal.

Later on, if you'd like to stop the services, CTRL-C any of the running java processes. If you
want a clean start after stopping the services, simply remove the `var/` directory. 

Once every service has started, you are now ready to load data.

## Load example data

We've included a sample of Wikipedia edits from September 12, 2015 to get you started.

```note-info
This quickstart shows you how to load data in batches, but Druid also supports [ingesting
streams in real-time](ingestion.html). Druid's realtime ingestion can load data with virtually no
delay between events occurring and being available for queries.
```

The [dimensions](https://en.wikipedia.org/wiki/Dimension_%28data_warehouse%29) (attributes you can
filter and split on) in the Wikipedia dataset, other than time, are:

  * channel
  * cityName
  * comment
  * countryIsoCode
  * countryName
  * isAnonymous
  * isMinor
  * isNew
  * isRobot
  * isUnpatrolled
  * metroCode
  * namespace
  * page
  * regionIsoCode
  * regionName
  * user

The [measures](https://en.wikipedia.org/wiki/Measure_%28data_warehouse%29), or *metrics* as they are known in Druid (values you can aggregate)
in the Wikipedia dataset are:

  * count
  * added
  * deleted
  * delta
  * user_unique

To load this data into Druid, you can submit an *ingestion task* pointing to the file. We've included
a task that loads the `wikiticker-2015-09-12-sampled.json` file included in the archive. To submit
this task, POST it to Druid:

```bash
curl -X 'POST' -H 'Content-Type:application/json' -d @quickstart/wikiticker-2015-09-12-sampled.json localhost:8090/druid/indexer/v1/task
```

Which will print something the ID of the task if the submissions was successful:

```base
{"task":"index_hadoop_wikipedia_2013-10-09T21:30:32.802Z"}
```

To view the status of your ingestion task, go to your overlord console:
[http://localhost:8090/console.html](http://localhost:8090/console.html). You can refresh the console periodically, and after 
the task is successful, you should see a "SUCCESS" status for the task.

After your ingestion task finishes, the data will be loaded by historical nodes and available for
querying within a minute or two. You can monitor the progress of loading your data in the
coordinator console, by checking whether there is a datasource "wikiticker" with a blue circle
indicating "fully available": [http://localhost:8081/#/](http://localhost:8081/#/).

Once the data is fully available, you can immediately query it&mdash; to see how, skip to the [Query
data](#query-data) section below. Or, continue to the [Load your own data](#load-your-own-data)
section if you'd like to load a different dataset.

## Load your own data

You can easily load any timestamped dataset into Druid. For Druid batch loads, the most important
questions are:

  * What should the dataset be called? This is the "dataSource" field of the "dataSchema".
  * Where is the dataset located? The file paths belong in the "paths" of the "inputSpec". If you
want to load multiple files, you can provide them as a comma-separated string.
  * Which field should be treated as a timestamp? This belongs in the "column" of the "timestampSpec".
  * Which fields should be treated dimensions? This belongs in the "dimensions" of the "dimensionsSpec".
  * Which fields should be treated as metrics? This belongs in the "metricsSpec".
  * What time ranges (intervals) are being loaded? This belongs in the "intervals" of the "granularitySpec".

```note-info
If your data does not have a natural sense of time, you can tag each row with the current time.
You can also tag all rows with a fixed timestamp, like "2000-01-01T00:00:00.000Z".
```

Let's use this pageviews dataset as an example. Druid supports TSV, CSV, and JSON out of the box.
Note that nested JSON objects are not supported, so if you do use JSON, you should provide a file
containing flattened objects.

```json
{"time": "2015-09-01T00:00:00Z", "url": "/foo/bar", "user": "alice", "latencyMs": 32}
{"time": "2015-09-01T01:00:00Z", "url": "/", "user": "bob", "latencyMs": 11}
{"time": "2015-09-01T01:30:00Z", "url": "/foo/bar", "user": "bob", "latencyMs": 45}
```

If you save this to a file called "pageviews.json", then for this dataset:

  * Let's call the dataset "pageviews".
  * The data is located in "pageviews.json".
  * The timestamp is the "time" field.
  * Good choices for dimensions are the string fields "url" and "user".
  * Good choices for metrics are a count of pageviews, and the sum of "latencyMs". Collecting that
sum when we load the data will allow us to compute an average at query time as well.
  * The data covers the time range 2015-09-01 (inclusive) through 2015-09-02 (exclusive).

You can copy the existing `quickstart/wikiticker-index.json` indexing task to a new file:

```bash
cp quickstart/wikiticker-index.json quickstart/pageviews.json
```

And modify it by altering these sections:

```json
"dataSource": "pageviews"
```

```json
"inputSpec": {
  "type": "static",
  "paths": "pageviews.json"
}
```

```json
"timestampSpec": {
  "format": "auto",
  "column": "time"
}
```

```json
"dimensionsSpec": {
  "dimensions": ["url", "user"]
}
```

```json
"metricsSpec": [
  {"name": "views", "type": "count"},
  {"name": "latencyMs", "type": "doubleSum", "fieldName": "latencyMs"}
]
```

```json
"granularitySpec": {
  "type": "uniform",
  "segmentGranularity": "day",
  "queryGranularity": "none",
  "intervals": ["2015-09-01/2015-09-02"]
}
```

Finally, fire off the task and indexing will proceed!

```bash
curl -X 'POST' -H 'Content-Type:application/json' -d @quickstart/pageviews.json localhost:8090/druid/indexer/v1/task
```

If anything goes wrong with this task (e.g. it finishes with status FAILED), you can troubleshoot
by visiting the "Task log" URL given by the *post-index-task* program.

```note-info
Druid supports a wide variety of data formats, ingestion options, and configurations not
discussed here. For a full explanation of all available features, see the [ingestion
overview](http://druid.io/docs/latest/ingestion/index.html) and [schema
design](http://druid.io/docs/latest/ingestion/schema-design.html) sections of the Druid
documentation.
```

## Query data

### Direct Druid queries

Druid supports a rich [family of JSON-based
queries](http://druid.io/docs/latest/querying/querying.html). We've included an example topN query
in `quickstart/wikiticker-top-pages.json` that will find the most-edited articles in this dataset:

```bash
curl -L -H'Content-Type: application/json' -XPOST --data-binary @quickstart/wikiticker-top-pages.json http://localhost:8082/druid/v2/
```

## Visualizing data

Druid is ideal for power user-facing analytic applications. There are a number of different open source applications to 
visualize and explore data in Druid. You can find them on our [client libraries](../development/libraries.html) page.

### SQL and other query libraries

There are many more query tools for Druid than we've included here, including SQL
engines, and libraries for various languages like Python and Ruby. Please see [the list of
libraries](http://druid.io/docs/latest/development/libraries.html)for more information.

## Clustered setup

This quickstart sets you up with all services running on a single machine. The next step is to learn about 
[how to load streaming data](../tutorials/streaming.html) into Druid.
