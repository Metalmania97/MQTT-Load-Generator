# MQTT Load Generator

Small Java CLI tool that simulates many MQTT producers publishing JSON messages in parallel.

Each producer simulates a single MQTT device (for example, one sensor).

## What it does

At runtime, the app:

1. Parses CLI arguments into a `ProducerConfig`.
2. Builds a payload filler string (repeated `A`) sized to help reach a target message size.
3. Starts a fixed thread pool with one worker per producer.
4. Connects each worker to the MQTT broker with a unique client ID.
5. Publishes messages to `--topic/<producerId>` using QoS 0.
6. Disconnects and closes each MQTT client after sending.

Each message is JSON and includes:

- `message_id` (UUID)
- `timestamp` (UTC, nanosecond precision format)
- `temperature` (random double between 86 and 88, rounded to 2 decimals)
- `payload` (filler string)

## Requirements

- Java 17+
- Maven 3.8+ (or compatible)
- Reachable MQTT broker endpoint (for example `tcp://localhost:1883`)

## Build

```bash
mvn clean package
```

This creates a runnable shaded jar in `target/`.

Copy the created JAR from `target/` into the docker project `MqttLoadGenerator` folder, then build the Docker image:

```bash
docker build -t mqttloadgenerator .
```

Run the application container for Kafka:

```bash
docker run -it --rm --name mqttproducer /
  --network kafka-docker_kafka_network /
  mqttloadgenerator java -cp app.jar org.example.MqttLoadGenerator /
  --broker=tcp://mqtt-broker:1883 /
  --topic=mqtt/temp /
  --producer-count=250 /
  --messages-per-producer=1000 /
  --delay-between-messages=50 /
  --payload-size=512
```

Run the application container for Pulsar:

```bash
docker run -it --rm --name mqttproducer /
  --network pulsar-docker_pulsar_network /
  mqttloadgenerator java -cp app.jar org.example.MqttLoadGenerator /
  --broker=tcp://pulsar:1883 /
  --topic=persistent://public/default/temp /
  --producer-count=250 /
  --messages-per-producer=1000 /
  --delay-between-messages=50 /
  --payload-size=512
```

## CLI arguments

All arguments use `--name=value` format.


| Argument                   | Required | Default | Description                                                   |
| -------------------------- | -------- | ------- | ------------------------------------------------------------- |
| `--broker`                 | Yes      | none    | MQTT broker address. Example: `tcp://localhost:1883`          |
| `--topic`                  | Yes      | none    | Base topic. Each producer publishes to `<topic>/<producerId>` |
| `--producer-count`         | No       | `100`   | Number of concurrent producer workers                         |
| `--messages-per-producer`  | No       | `600`   | Number of messages each producer sends                        |
| `--delay-between-messages` | No       | `1000`  | Delay in milliseconds between messages for each producer      |
| `--payload-size`           | No       | `130`   | Target minimum size logic input for payload filler            |


Validation and clamping behavior:

- Missing `--broker` or `--topic` causes startup failure with an error message.
- `--producer-count` and `--messages-per-producer` are clamped to at least `1`.
- `--delay-between-messages` is clamped to at least `0`.
- `--payload-size` is clamped to at least `130`.

## Throughput and volume math

- Total producers = `producer-count`
- Messages per producer = `messages-per-producer`
- Total messages attempted = `producer-count * messages-per-producer`

Approximate per-producer duration:

- `random startup jitter (0-1499ms) + ((messages-per-producer - 1) * delay-between-messages)`

Approximate produced messages per second (steady state, ignoring startup jitter):

- `produced-messages-per-second = producer-count * (1000 / delay-between-messages)`

