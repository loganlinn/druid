---
layout: doc_page
---

## Stream Push

If you have a program that generates a stream, then you can push that stream directly into Druid in
real-time. With this approach, [Tranquility](https://github.com/druid-io/tranquility) (a Druid-aware
client) is embedded in your data-producing application. Tranquility comes with bindings for the
Storm and Samza stream processors. It also has a direct API that can be used from any JVM-based
program, such as Spark Streaming or a Kafka consumer.

Tranquility handles partitioning, replication, service discovery, and schema rollover for you,
seamlessly and without downtime. You only have to define your Druid schema.

For examples and more information, please see the [Tranquility README](https://github.com/druid-io/tranquility) and the [Druid ingestion overview](http://druid.io/docs/latest/ingestion/overview.html).
