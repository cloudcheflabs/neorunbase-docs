# Encryption at Rest

NeorunBase provides built-in encryption at rest to protect data stored on disk, ensuring that sensitive data is always encrypted without any application-level changes.

## Envelope Encryption

NeorunBase uses envelope encryption, a widely adopted encryption approach used by major cloud providers. Each shard is encrypted with its own unique Data Encryption Key (DEK), and DEKs are encrypted with a master key managed by the built-in Key Management Service (KMS).

## Built-in KMS

NeorunBase includes a built-in Key Management Service that:

- Generates and manages encryption keys
- Distributes keys securely across the cluster
- Synchronizes keys between Coordinators and Data Nodes automatically

## What Is Encrypted

- **Row data**: Stored in append-only heap segment files on each Data Node, with AES-256-GCM applied per record using a per-segment Data Encryption Key.
- **RocksDB-resident metadata**: Row pointers, secondary indexes, and the change log are encrypted with the per-shard DEK.
- **Vector / ANN sidecars**: HNSW graph files use the same envelope-encryption pattern as row data.
- **Metadata store**: Table schemas and shard maps maintained by the Coordinator are encrypted.
- **Write-Ahead Log (WAL)**: WAL segments are encrypted record-by-record; the inner WAL record carries an additional CRC32 checksum so any tamper or partial decode is caught even before the GCM tag.
- **Internal communication**: Data transmitted between Coordinators and Data Nodes is encrypted using AES.

## Bitrot Detection

Every persistence layer carries strong on-read corruption detection out of the box:

- AES-GCM authentication tags reject any tampered or bit-rotted record on the heap and WAL paths.
- RocksDB's per-block CRC32C catches storage-side corruption on every read.
- An optional **bitrot scrubber** can be enabled per Data Node to proactively walk closed heap segments in the background, throttled to a configurable bytes-per-second budget so it never starves OLTP traffic. Findings are exposed via the Admin API for operator review.
- WAL recovery refuses to silently skip corrupted records: a structured corruption signal halts shard startup so the operator can investigate before any divergent state is created.

## Transparent to Clients

Encryption is fully transparent to clients. No changes to queries, connection settings, or application code are required. Data is encrypted when written to disk and decrypted when read, all handled internally by NeorunBase.
