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

- **IAM Management**: Create and manage users, groups, and policies
- **Iceberg Sync**: Configure and monitor Iceberg synchronization
- **Kafka Ingestion**: Manage Kafka consumer groups and monitor ingestion pipelines
- **Shard Operations**: Monitor shard health, replication status, and repair progress

## REST API

All operations available in the Admin UI are also accessible via a REST API, enabling automation and integration with external monitoring and management tools.
