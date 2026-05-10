# Getting Started

This page walks through a complete first session against a freshly started NeorunBase cluster: logging into the Admin UI, connecting with `psql`, and running a small but representative SQL workload that exercises sharding, joins, aggregation, updates, and deletes.

The instructions below assume you have already installed NeorunBase and started the example cluster as described on [Download & Install](installation.md):

```agsl
export NEORUNBASE_MASTER_KEY=test-master-key-for-integration-tests-12345
bin/start-example-servers.sh
```

## 1. Open the Admin UI

After a few seconds the Coordinator's admin HTTP server is reachable at:

```agsl
http://localhost:8080/admin
```

The initial admin user and password is `admin` / `admin`. You will be required to change the password on first login.

<img width="1200" src="../../images/getting-started/dashboard.png"/>

The Admin UI is the entry point for cluster topology, table and shard inspection, IAM (users, roles, policies, STS), encryption / KMS status, metrics, and log inspection. See the [Admin UI](../features/admin-ui.md) feature page for details.

## 2. Connect with psql

NeorunBase implements the PostgreSQL wire protocol, so any PostgreSQL-compatible client connects without modification. Using `psql`:

```agsl
PGPASSWORD="<your-admin-password>" psql -h localhost -p 5432 -U admin -d neorunbase
```

## 3. Run the Example Workload

The SQL below is also shipped under `examples/example.sql` in the release archive. It defines three sharded tables, inserts sample rows, then exercises filtering, ordering, joins, grouping, aggregation, and DML.

```agsl
-- =============================================================================
-- NeorunBase Quick Start Example
--
-- Connect:  psql -h localhost -p 5432 -U admin -d neorunbase
-- =============================================================================


-- ===================== 1. CREATE TABLES =====================

-- Customers table
CREATE TABLE customers (
    id         BIGINT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(255),
    city       VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
) SHARD KEY (id) SHARDS 4;

-- Products table
CREATE TABLE products (
    id       BIGINT PRIMARY KEY,
    name     VARCHAR(200) NOT NULL,
    category VARCHAR(50),
    price    FLOAT NOT NULL
) SHARD KEY (id) SHARDS 4;

-- Orders table
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    customer_id BIGINT,
    product_id  BIGINT,
    quantity    INT NOT NULL,
    total       FLOAT NOT NULL,
    status      VARCHAR(20),
    ordered_at  TIMESTAMP DEFAULT NOW()
) SHARD KEY (id) SHARDS 4;


-- ===================== 2. INSERT DATA =====================

-- Customers
INSERT INTO customers (id, name, email, city) VALUES (1, 'Alice Kim',    'alice@example.com',   'Seoul');
INSERT INTO customers (id, name, email, city) VALUES (2, 'Bob Park',     'bob@example.com',     'Busan');
INSERT INTO customers (id, name, email, city) VALUES (3, 'Charlie Lee',  'charlie@example.com', 'Seoul');
INSERT INTO customers (id, name, email, city) VALUES (4, 'Diana Choi',   'diana@example.com',   'Incheon');
INSERT INTO customers (id, name, email, city) VALUES (5, 'Eve Jung',     'eve@example.com',     'Busan');

-- Products
INSERT INTO products (id, name, category, price) VALUES (1, 'NeorunBase Enterprise License', 'software', 5000.00);
INSERT INTO products (id, name, category, price) VALUES (2, 'Premium Support (1 Year)',      'support',  2000.00);
INSERT INTO products (id, name, category, price) VALUES (3, 'Training Workshop',             'service',  800.00);
INSERT INTO products (id, name, category, price) VALUES (4, 'Consulting (Per Day)',           'service',  1500.00);
INSERT INTO products (id, name, category, price) VALUES (5, 'NeorunBase Starter License',    'software', 1000.00);

-- Orders
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (1,  1, 1, 1, 5000.00,  'completed');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (2,  1, 2, 1, 2000.00,  'completed');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (3,  2, 5, 2, 2000.00,  'completed');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (4,  2, 3, 1, 800.00,   'pending');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (5,  3, 1, 1, 5000.00,  'completed');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (6,  3, 4, 3, 4500.00,  'completed');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (7,  4, 2, 1, 2000.00,  'pending');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (8,  4, 3, 2, 1600.00,  'completed');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (9,  5, 5, 1, 1000.00,  'cancelled');
INSERT INTO orders (id, customer_id, product_id, quantity, total, status) VALUES (10, 5, 1, 1, 5000.00,  'completed');


-- ===================== 3. SELECT QUERIES =====================

-- 3-1. Basic SELECT
SELECT * FROM customers;

-- 3-2. WHERE clause
SELECT * FROM orders WHERE status = 'completed';

-- 3-3. ORDER BY with LIMIT
SELECT * FROM orders ORDER BY total DESC LIMIT 5;

-- 3-4. INNER JOIN - Order details with customer and product names
SELECT o.id AS order_id,
       c.name AS customer,
       p.name AS product,
       o.quantity,
       o.total,
       o.status
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
INNER JOIN products p  ON o.product_id  = p.id;

-- 3-5. LEFT JOIN - All customers with their order count (including those with no orders)
SELECT c.name, COUNT(o.id)
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;

-- 3-6. Aggregation - Total revenue by product category
SELECT p.category, SUM(o.total), COUNT(*)
FROM orders o
INNER JOIN products p ON o.product_id = p.id
WHERE o.status = 'completed'
GROUP BY p.category;

-- 3-7. Aggregation - Revenue per customer (top spenders)
SELECT c.name, SUM(o.total), COUNT(*)
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'completed'
GROUP BY c.name
ORDER BY SUM(o.total) DESC;

-- 3-8. HAVING - Customers who spent more than 5000
SELECT c.name, SUM(o.total)
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'completed'
GROUP BY c.name
HAVING SUM(o.total) > 5000;

-- 3-9. Aggregation - Revenue per city
SELECT c.city, SUM(o.total), COUNT(*)
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'completed'
GROUP BY c.city
ORDER BY SUM(o.total) DESC;

-- 3-10. Overall summary
SELECT COUNT(*) AS total_orders,
       SUM(total) AS total_revenue,
       AVG(total) AS avg_order_value,
       MIN(total) AS min_order,
       MAX(total) AS max_order
FROM orders
WHERE status = 'completed';


-- ===================== 4. UPDATE =====================

UPDATE orders SET status = 'completed' WHERE id = 4;

SELECT * FROM orders WHERE id = 4;


-- ===================== 5. DELETE =====================

DELETE FROM orders WHERE status = 'cancelled';

SELECT * FROM orders ORDER BY id;


-- ===================== 6. CLEANUP (optional) =====================
-- Uncomment below to drop all example tables:
-- DROP TABLE orders;
-- DROP TABLE products;
-- DROP TABLE customers;
```

The `SHARD KEY (id) SHARDS 4` clause is a NeorunBase extension to `CREATE TABLE`: it hash-partitions each table into four shards on `id` and places the shards across the available Data Nodes with the configured replication factor. Distributed joins, aggregations, sorts, and DML are executed transparently across shards by the Coordinator. See [Distributed Sharding](../features/distributed-sharding.md) and [Distributed Transactions](../features/transactions.md).

## 4. Stop the Cluster

When you are done evaluating, stop the example cluster:

```agsl
bin/stop-example-servers.sh
```

## Next Steps

- [Configuration](../configuration/configuration.md) — every tunable in `neorunbase.properties`.
- [Operations](../operations/operations.md) — running, monitoring, and shutting down a production cluster.
- [Vector Database](../features/vector-database.md) — pgvector-compatible `VECTOR` type and distributed HNSW search.
- [Full-Text Search](../features/full-text-search.md) — Lucene-backed BM25 with per-shard inverted indexes.
- [Graph Traversal & Analytics](../features/graph.md) — `GRAPH_NEIGHBORS`, `PAGERANK`, `PERSONALIZED_PAGERANK`, `GRAPH_PATH_EXISTS` TVFs and the CSR adjacency layer.
- [Hybrid Search](../features/hybrid-search.md) — `HYBRID_SEARCH` TVF blending BM25 + vector ANN in one SQL call.
- [Features overview](../features/postgresql-compatibility.md) — PostgreSQL compatibility, sharding, replication, encryption, transactions, Iceberg sync, Kafka ingestion, IAM, and the Admin UI.
