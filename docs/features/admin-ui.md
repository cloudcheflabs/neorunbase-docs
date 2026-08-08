# Admin UI

NeorunBase includes a built-in web-based Admin UI for monitoring and managing the cluster.

## Cluster Overview

The Admin UI provides a visual overview of the cluster, including:

- List of active Coordinators and Data Nodes
- Node status and health information
- Shard distribution across Data Nodes

## Monitoring

Real-time metrics and monitoring capabilities:

- Query throughput and latency metrics
- Storage usage per node and per table
- Cluster-wide performance dashboards
- Time-series metrics with configurable retention

## Management

The Admin UI allows administrators to perform operational tasks:

- **IAM Management**: Create and manage users, groups, policies, access keys, and STS sessions. In federated mode this page is read-only and hosts the standalone ↔ federated mode toggle. See [Identity and Access Management](iam.md).
- **Security & KMS**: List the KMS key hierarchy, inspect a key's versions, create new keys, and rotate a key to a new version. See [Encryption at Rest](encryption.md).
- **pg-wire TLS**: Upload, rotate, and remove the cluster-wide PostgreSQL wire protocol certificate. Activation propagates to every Coordinator with no restart and existing connections are unaffected. See [pg-wire TLS](pg-wire-tls.md).
- **Catalogs**: List, create, edit, and drop catalogs (the built-in `lakebase` plus external Iceberg catalogs). Drop is refused for a non-empty catalog and `lakebase` cannot be removed. See [Catalogs](catalogs.md).
- **Iceberg**: Configure the Apache Polaris connection (URI, OAuth client credentials, S3 access/secret key) and monitor sync status. See [Iceberg Integration](iceberg-integration.md).
- **S3 Backup**: Configure scheduled cluster backups, view backup history, and trigger restore. See [Backup & Restore](backup-restore.md).
- **Site Replication**: Configure disaster-recovery peer sites and monitor the leader-only WAL streamer that ships committed writes to them.
- **Kafka Ingestion**: Manage Kafka consumer groups and monitor ingestion pipelines
- **Query Runner**: Execute SQL against the cluster directly from the browser, with a local query history.
- **Shard Operations**: Monitor shard distribution and health, and drive shard/disk repair progress

## REST API

All operations available in the Admin UI are also accessible via a REST API, enabling automation and integration with external monitoring and management tools.
