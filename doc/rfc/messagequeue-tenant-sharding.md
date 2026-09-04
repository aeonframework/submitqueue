# RFC: Per-Tenant Vitess Sharding for the MySQL Message Queue

## Metadata

| Field | Value |
|-------|-------|
| **Author** | Preetam Dwivedi |
| **Status** | In Review |
| **Created** | 2026-09-03 |

## Summary

The platform MySQL message queue gains a domain-agnostic `tenant` column on every table. Vitess routes on `tenant`; every primary key and hot-path query leads with it. SubmitQueue, Stovepipe, and Runway map their `queueName` onto `tenant` at the wiring boundary. `partition_key` remains the ordering unit within a tenant and is not the vindex.

## Background

Domain storage (`request`, `batch`, `counter`, …) already shards by a leading `queue` column. The message queue backend was deliberately excluded from `tool/linter/queueshard` because it keyed rows by `(consumer_group, topic, partition_key)` with a global `AUTO_INCREMENT offset` — fine for a single MySQL instance, not for Vitess.

SubmitQueue's business queue name already flows through publish metadata (`queue_name`) and consumer context (`WithQueueName`). Several pipeline stages use a different `partition_key` (build ID, request ID) so those deliveries serialize independently while still belonging to one business queue. Sharding on `partition_key` would split one queue across shards; the MQ needs a dedicated isolation column.

## Naming

| Layer | Column / field | Meaning |
|-------|----------------|---------|
| Platform MQ schema | `tenant` | Vitess vindex; opaque to the backend |
| Platform `Message` | `Tenant` | Persisted shard identity |
| SubmitQueue domain | `queue` / `queueName` | Same string as `tenant` at wiring |
| Platform MQ schema | `partition_key` | Ordering unit within `(tenant, topic)` |

The MQ schema does not use `queue` — that word is overloaded (SubmitQueue domain, `Queue` interface, `queue_*` table prefix).

## Schema

Every table's primary key leads with `tenant`. Secondary indexes that do not lead with `tenant` are removed.

### `queue_messages`

- PK: `(tenant, topic, partition_key, offset)`
- Unique: `(tenant, topic, partition_key, id)`
- `offset` is per-partition, not global; fetch is `WHERE tenant=? AND topic=? AND partition_key=? AND offset>? ORDER BY offset`

### `queue_delivery_state`

- PK: `(tenant, consumer_group, topic, partition_key, message_offset)`

### `queue_offsets`

- PK: `(tenant, consumer_group, topic, partition_key)`
- Drop `idx_topic`

### `queue_partition_leases`

- PK: `(tenant, consumer_group, topic, partition_key)`
- Drop `idx_lease_renewed`; purge is scoped to `(tenant, consumer_group, topic)`

### `queue_subscriber_heartbeats`

- PK: `(tenant, consumer_group, topic, subscriber_name)`

DLQ moves rewrite `topic` to `original + suffix` and keep `tenant` + `partition_key` on the same shard.

## Subscriber discovery

Today partition discovery runs `SELECT DISTINCT partition_key FROM queue_messages WHERE topic=?`, which scatter-gathers across all Vitess shards.

The subscriber takes a configured tenant list (SubmitQueue: queue names from YAML). Discovery becomes:

```sql
SELECT DISTINCT partition_key FROM queue_messages
WHERE tenant = ? AND topic = ?
ORDER BY partition_key
```

Fair-share, orphan sweep, and idle-lease release run per `(tenant, topic)`, not across all tenants on a topic.

## Publish

`platform/publish` stamps `Message.Tenant` from context metadata (`queue_name`). Empty tenant on publish is rejected. `PartitionKey` is unchanged.

## Wiring

One `extqueue.Queue` and one VTGate DSN per service. `NewQueue` / subscriber `Params` carry `Tenants []string`. Service `main.go` fills that from configured queue names.

## Out of scope

Live migration of existing Stovepipe prod queue databases (expand/contract, backfill, dual-write). This RFC describes a breaking greenfield schema; prod cutover is a separate exercise.

## Related

- [SQL-Based Distributed Queue](sql-queue-rfc.md)
- [Modular Queue Wiring](submitqueue/modular-queue-wiring.md)
