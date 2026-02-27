# Kafka Integration

NeorunBase supports streaming data ingestion from Apache Kafka, enabling real-time data to flow directly into NeorunBase tables.

## Streaming Ingestion

NeorunBase can consume messages from Kafka topics and automatically insert them into designated tables. This provides a seamless pipeline from event streams to a queryable relational database.

## Multiple Consumer Groups

You can configure multiple independent Kafka consumer groups, each consuming from different topics and writing to different tables. This allows you to ingest data from various Kafka sources simultaneously.

## Configuration

Each Kafka consumer group can be configured with:

- Kafka broker addresses
- Topic name
- Consumer group ID
- Target NeorunBase table
- Batch size for efficient bulk inserts
- Auto offset reset policy
- Additional Kafka consumer properties

## JSON Message Format

NeorunBase consumes JSON-formatted messages from Kafka topics. Messages are parsed and batch-inserted into the target table, matching JSON fields to table columns.

## Management

Kafka consumer groups can be managed through the NeorunBase Admin API, allowing you to start, stop, and monitor ingestion pipelines at runtime.
