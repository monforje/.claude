# RabbitMQ, Kafka, NATS

The recurring trap in all three: a broker tells clients where to reconnect, and
if it advertises a name that only resolves *inside* Docker, clients on the host
fail after a successful first connection.

## RabbitMQ

```yaml
  rabbitmq:
    image: rabbitmq:4-management-alpine
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: app
      RABBITMQ_DEFAULT_PASS: secret
    ports:
      - "5672:5672"      # AMQP
      - "15672:15672"    # management UI — http://localhost:15672
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "check_running", "&&",
             "rabbitmq-diagnostics", "-q", "check_local_alarms"]
      interval: 10s
      timeout: 10s
      retries: 10
      start_period: 30s
```

- Take the `-management` tag. The UI shows queue depth, unacked messages and
  bindings; guessing at those from application logs wastes hours.
- `start_period: 30s` — RabbitMQ genuinely takes that long. With a short one the
  container is killed and restarted forever.
- `rabbitmq-diagnostics ping` only says the Erlang VM is up. `check_running` plus
  `check_local_alarms` is the check that means "will accept publishes".
- The data volume holds the node's identity. Deleting it loses durable queues
  and definitions.
- URL from another service: `amqp://app:secret@rabbitmq:5672/`.

## Kafka — `apache/kafka`, KRaft, two listeners

ZooKeeper is gone (removed in Kafka 4.0). Any compose file with a `zookeeper`
service is obsolete — don't copy it.

```yaml
  kafka:
    image: apache/kafka:4.0.0
    restart: unless-stopped
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: INTERNAL://:19092,EXTERNAL://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:19092,EXTERNAL://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    volumes:
      - kafka-data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD-SHELL", "/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 15
      start_period: 20s
```

- **Two listeners is the whole point.** A client connects to a bootstrap
  address, then reconnects to whatever the broker *advertises*. One listener
  advertising `localhost:9092` breaks other containers; one advertising
  `kafka:9092` breaks the host. Containers use `kafka:19092`, host tools use
  `localhost:9092`.
- The three `..._REPLICATION_FACTOR: 1` / `MIN_ISR: 1` settings are mandatory
  for a single node — the defaults are 3 and the internal topics can't be created.
- `KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0` removes a 3-second pause on the
  first consumer join. Development only.
- The upstream Docker Hub example advertises `localhost:9092` only; it works for
  `docker run`, not inside a compose stack.

## NATS — with JetStream

Plain NATS is fire-and-forget: a subscriber that wasn't connected never sees the
message. Persistence, replay, queues and key/value all come from JetStream,
which is **off by default**.

```yaml
  nats:
    image: nats:2.11-alpine
    restart: unless-stopped
    command: ["-js", "-sd", "/data", "-m", "8222"]
    ports:
      - "4222:4222"      # clients
      - "8222:8222"      # monitoring
    volumes:
      - nats-data:/data
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8222/healthz | grep -q ok"]
      interval: 5s
      timeout: 3s
      retries: 10
```

- `-js` enables JetStream, `-sd /data` puts its store on the volume. Without
  `-sd` the store is in the container layer and every recreate loses the streams.
- `-m 8222` enables the monitoring endpoint, which is also the only sane
  healthcheck (`/healthz`). The alpine image has `wget`, not `curl`.
- URL from another service: `nats://nats:4222`.

## Shared

```yaml
volumes:
  rabbitmq-data:
  kafka-data:
  nats-data:
```

An app depending on a broker still needs its own reconnect logic:
`condition: service_healthy` gets the first connection right, not the broker
restarting at 3am.
