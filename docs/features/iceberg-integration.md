# Iceberg Integration

NeorunBase integrates with Apache Iceberg, enabling automatic synchronization of transactional data to an open lakehouse format. This allows downstream analytics engines such as Apache Spark, Trino, and Hive to query NeorunBase data directly.

## Automatic Data Synchronization

NeorunBase automatically syncs table data to Iceberg tables in the background:

- **Initial sync**: A full snapshot of the table is exported as Parquet files to S3-compatible object storage and registered in the Iceberg catalog.
- **Incremental sync**: After the initial sync, only the changes (inserts, updates, deletes) are synchronized incrementally, minimizing the overhead.

## Iceberg Catalog Support

NeorunBase connects to any Iceberg REST catalog (e.g., Polaris, Nessie) with support for:

- OAuth2 client credentials authentication
- Static bearer token authentication

## Open Lakehouse Analytics

Once data is synced to Iceberg, it can be queried by any engine that supports the Iceberg table format:

- **Apache Spark**: Batch and streaming analytics
- **Trino**: Interactive SQL queries
- **Apache Hive**: Data warehousing workloads
- **Apache Flink**: Stream processing

## External Iceberg Table Queries

NeorunBase can also read data from external Iceberg tables. This allows you to query data stored in Iceberg (Parquet, ORC, Avro formats) directly from NeorunBase using standard SQL, bridging the gap between the transactional and analytical worlds.

## S3-Compatible Storage

Iceberg data files are stored in S3-compatible object storage, supporting AWS S3, MinIO, and other S3-compatible services.
