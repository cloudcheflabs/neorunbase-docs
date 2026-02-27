# Replication & High Availability

NeorunBase provides fault tolerance and high availability through shard replication and automatic failure recovery.

## Shard Replication

Each shard can be configured with a replication factor. NeorunBase maintains multiple copies of each shard across different Data Nodes. This ensures that data remains available even if one or more Data Nodes go down.

## Automatic Failure Detection

NeorunBase continuously monitors the health of all Data Nodes. When a Data Node becomes unavailable, the system automatically detects the failure and initiates recovery actions.

## Automatic Shard Repair

When a Data Node failure is detected, NeorunBase automatically replicates the affected shards from surviving replicas to healthy Data Nodes, restoring the desired replication factor without manual intervention.

## Shard Rebalancing

When Data Nodes are added to or removed from the cluster, NeorunBase automatically rebalances shards across the cluster to ensure even data distribution and optimal resource utilization.

## Coordinator High Availability

Multiple Coordinators can run simultaneously. NeorunBase uses leader election to designate a primary Coordinator for metadata management, while all Coordinators can serve client queries. If the leader Coordinator fails, a new leader is automatically elected.
