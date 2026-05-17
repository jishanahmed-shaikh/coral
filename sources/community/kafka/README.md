# Kafka

Query Apache Kafka cluster metadata — brokers, topics, partitions, consumer
groups, and per-partition offsets — using
[Kafka UI](https://github.com/provectus/kafka-ui) as an HTTP bridge.

Kafka speaks a custom binary protocol on port 9092 that HTTP clients cannot
use directly. Kafka UI wraps that protocol and exposes a REST API, which is
what this source queries.

## Authentication

No authentication is required by default. Kafka UI can be configured with
login/password protection; if you enable that, place a reverse proxy in front
and handle auth there before pointing `KAFKA_UI_URL` at it.

## Local Setup

### Quickstart (recommended — Docker network)

The simplest reliable setup runs both containers on a shared Docker network so
Kafka UI can reach Kafka by container name. This avoids the advertised-listener
problem where Kafka tells clients to reconnect to `localhost:9092`, which does
not resolve inside a Docker container.

```bash
# 1. Create a shared network
docker network create kafka-net

# 2. Start Kafka (KRaft mode, no ZooKeeper)
docker run -d \
  --network kafka-net \
  --name kafka \
  -p 9092:9092 \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_LISTENERS="PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093" \
  -e KAFKA_ADVERTISED_LISTENERS="PLAINTEXT://kafka:9092" \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP="CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT" \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS="1@kafka:9093" \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 \
  -e KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS=0 \
  apache/kafka:latest

# 3. Start Kafka UI pointing to the kafka container by name
docker run -d \
  --network kafka-net \
  --name kafka-ui \
  -p 8080:8080 \
  -e KAFKA_CLUSTERS_0_NAME=local \
  -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092 \
  provectuslabs/kafka-ui:latest
```

Kafka UI will be available at `http://localhost:8080`.

> **Why not `host.docker.internal:9092`?**
> When Kafka is started without explicit `KAFKA_ADVERTISED_LISTENERS`, it
> advertises itself as `localhost:9092`. A Kafka UI container that connects
> via `host.docker.internal:9092` gets redirected to `localhost:9092` — which
> resolves to the container itself, not the host. Setting
> `KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092` fixes this by making
> Kafka advertise the container name, which both containers can resolve on the
> shared network.

### Create a topic and produce messages (optional)

```bash
# Open a shell inside the Kafka container
docker exec --workdir /opt/kafka/bin/ -it kafka sh

# Create a topic
./kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test-topic

# Verify
./kafka-topics.sh --bootstrap-server localhost:9092 --list

# Produce messages (type messages, press Ctrl+C to stop)
./kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092

# Consume messages (in a second shell)
docker exec --workdir /opt/kafka/bin/ -it kafka sh
./kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server localhost:9092
```

## Configuration

| Input          | Kind     | Required | Description                                                      |
|----------------|----------|----------|------------------------------------------------------------------|
| `KAFKA_UI_URL` | variable | yes      | Base URL of your Kafka UI instance, e.g. `http://localhost:8080` |

The cluster name is not a global config — it is passed per query as the
`cluster_name` filter. Use `SELECT * FROM kafka.clusters` to discover the
names available in your Kafka UI instance.

## Schema

### `clusters`

One row per Kafka cluster registered in Kafka UI. Start here to discover
cluster names — the `cluster_name` value is required by all other tables.

### `brokers`

One row per broker in a cluster. Requires `cluster_name`. Use
`bytes_in_per_sec` and `bytes_out_per_sec` to spot throughput imbalances.

### `topics`

One row per topic. Requires `cluster_name`. Internal topics (e.g.
`__consumer_offsets`) are hidden by default; pass `show_internal = 'true'` to
include them. Use `search` for substring filtering on topic name.

### `topic_configs`

One row per configuration key for a specific topic. Requires `cluster_name`
and `topic_name`. Sensitive values are masked by Kafka UI.

### `consumer_groups`

One row per consumer group. Requires `cluster_name`. The `consumer_lag` column
is the total message lag across all assigned partitions; null means no offsets
have been committed yet.

### `consumer_group_offsets`

One row per topic-partition assignment for a specific consumer group. Requires
`cluster_name` and `group_id`. Use this to drill into per-partition lag after
identifying a lagging group in `consumer_groups`.

## Example Queries

```sql
-- List all clusters and their status
SELECT cluster_name, status, broker_count, topic_count FROM kafka.clusters;

-- List all topics in a cluster
SELECT name, partition_count, replication_factor, clean_up_policy
FROM kafka.topics
WHERE cluster_name = 'local';

-- Find under-replicated topics
SELECT name, partition_count, replication_factor, under_replicated_partitions
FROM kafka.topics
WHERE cluster_name = 'local'
  AND under_replicated_partitions > 0;

-- Inspect broker throughput
SELECT id, host, port, partitions_leader, bytes_in_per_sec, bytes_out_per_sec
FROM kafka.brokers
WHERE cluster_name = 'local';

-- Check retention config for a topic
SELECT name, value, source
FROM kafka.topic_configs
WHERE cluster_name = 'local'
  AND topic_name = 'test-topic'
  AND name IN ('retention.ms', 'cleanup.policy', 'max.message.bytes');

-- Find consumer groups with lag
SELECT group_id, state, members, consumer_lag
FROM kafka.consumer_groups
WHERE cluster_name = 'local'
  AND consumer_lag > 0
ORDER BY consumer_lag DESC;

-- Drill into per-partition lag for a specific group
SELECT topic, partition, current_offset, end_offset, consumer_lag
FROM kafka.consumer_group_offsets
WHERE cluster_name = 'local'
  AND group_id = 'my-consumer-group'
ORDER BY consumer_lag DESC NULLS LAST;
```
