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

- **Shard data**: All table data stored on Data Nodes is encrypted at rest.
- **Metadata**: Table schemas and shard maps maintained by the Coordinator are encrypted.
- **Write-Ahead Log (WAL)**: WAL segments are encrypted to protect data durability records.
- **Internal communication**: Data transmitted between Coordinators and Data Nodes is encrypted using AES.

## Transparent to Clients

Encryption is fully transparent to clients. No changes to queries, connection settings, or application code are required. Data is encrypted when written to disk and decrypted when read, all handled internally by NeorunBase.
