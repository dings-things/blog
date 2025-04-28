---
title: "[EDA] Running a Local Kafka Cluster with SASL SCRAM Authentication (Docker Compose)"
date: "2025-03-28"
draft: false
author: dingyu
categories: ["eda"]
tags: ["kafka", "sasl", "bitnami", "kafka ui", "local test"]
summary: E2E testing is critical for validating production-ready applications. But how do you simulate a secure Kafka environment locally—especially with SASL authentication? This post demonstrates how to spin up a Kafka cluster with both SCRAM-SHA-256 and SCRAM-SHA-512 using Docker Compose.
cover:
  image: img/kafka.png
  relative: true
comments: true
description: This post documents how to build a local Kafka cluster using Docker Compose that supports both SCRAM-SHA-256 and SCRAM-SHA-512 SASL authentication mechanisms, enabling secure, production-like testing for applications like event dispatchers—all without modifying code or relying on external infrastructure.
---


# Setting Up Local SASL Mechanism Kafka Cluster with Docker Compose

When using Kafka with SASL-based authentication, it's important to replicate a similar authentication environment locally for testing.  
This post documents how to build a Kafka cluster in a local environment that simultaneously supports both `SCRAM-SHA-256` and `SCRAM-SHA-512`, and how to test the Event Dispatcher application on top of it.

It was extremely difficult to find references for setting up `SASL` and `TLS` locally...  
So while struggling through the setup, I decided to leave this guide as a gift for my future self.

(*TLS setup with Makefile for certificates was a bit too complex, so I'll cover that in a separate post later.*)

---

## Goals

- The Event Dispatcher consumes events from a **source Kafka** and produces to a **destination Kafka** depending on event types.
- **Source** and **Destination** Kafka are on the **same cluster**, but differentiated by **different SASL mechanisms**.
- **Source Kafka** uses **SCRAM-SHA-256**, **Destination Kafka** uses **SCRAM-SHA-512**.
- The goal is to replicate the production environment structure without modifying the application code even during tests.
- Kafka cluster has 3 brokers to set ISR (in-sync replicas) to 2.
- Kafka UI should allow manual produce tests to verify application behavior.

---

## Why Set Up a Local Environment?

- Debugging on a shared dev server was inconvenient because of limited stack trace access and the need to go through VPN/Bastion gateways.
- All internal Kafka clusters use SASL, so creating a reusable setup would benefit many projects.
- Configuration should be modifiable for testing (like transaction settings, exactly-once semantics, ISR tuning) **without code changes** or deployments.
- A local infrastructure was necessary to maintain identical codebases during testing.

---

## Docker Compose Configuration

```yaml
version: '3.9'

networks:
  kafka_network:

volumes:
  kafka_data_0:
  kafka_data_1:
  kafka_data_2:

services:
  zookeeper:
    image: bitnami/zookeeper:3.8.1
    container_name: zookeeper
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
    ports:
      - '2181:2181'
    networks:
      - kafka_network

  kafka-0:
    image: bitnami/kafka:3.7.0
    container_name: kafka-0
    depends_on:
      - zookeeper
    ports:
      - '${KAFKA_BROKER_0_PORT}:9092'
    environment:
      KAFKA_CFG_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_CFG_LISTENERS: SASL_PLAINTEXT://:9092
      KAFKA_CFG_ADVERTISED_LISTENERS: SASL_PLAINTEXT://host.docker.internal:${KAFKA_BROKER_0_PORT}
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: SASL_PLAINTEXT:SASL_PLAINTEXT
      KAFKA_CFG_INTER_BROKER_LISTENER_NAME: SASL_PLAINTEXT
      KAFKA_CFG_SASL_ENABLED_MECHANISMS: SCRAM-SHA-512,SCRAM-SHA-256
      KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL: SCRAM-SHA-512
      KAFKA_CLIENT_USERS: ${512_SASL_USER},${256_SASL_USER}
      KAFKA_CLIENT_PASSWORDS: ${512_SASL_PASSWORD},${256_SASL_PASSWORD}
      KAFKA_INTER_BROKER_USER: ${512_SASL_USER}
      KAFKA_INTER_BROKER_PASSWORD: ${512_SASL_PASSWORD}
    volumes:
      - kafka_data_0:/bitnami/kafka
    networks:
      - kafka_network
    hostname: kafka

  kafka-1:
    image: bitnami/kafka:3.7.0
    container_name: kafka-1
    depends_on:
      - zookeeper
    ports:
      - '${KAFKA_BROKER_1_PORT}:9092'
    environment:
      KAFKA_CFG_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_CFG_LISTENERS: SASL_PLAINTEXT://:9092
      KAFKA_CFG_ADVERTISED_LISTENERS: SASL_PLAINTEXT://host.docker.internal:${KAFKA_BROKER_1_PORT}
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: SASL_PLAINTEXT:SASL_PLAINTEXT
      KAFKA_CFG_INTER_BROKER_LISTENER_NAME: SASL_PLAINTEXT
      KAFKA_CFG_SASL_ENABLED_MECHANISMS: SCRAM-SHA-512,SCRAM-SHA-256
      KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL: SCRAM-SHA-512
      KAFKA_CLIENT_USERS: ${512_SASL_USER},${256_SASL_USER}
      KAFKA_CLIENT_PASSWORDS: ${512_SASL_PASSWORD},${256_SASL_PASSWORD}
      KAFKA_INTER_BROKER_USER: ${512_SASL_USER}
      KAFKA_INTER_BROKER_PASSWORD: ${512_SASL_PASSWORD}
    volumes:
      - kafka_data_1:/bitnami/kafka
    networks:
      - kafka_network
    hostname: kafka-1

  kafka-2:
    image: bitnami/kafka:3.7.0
    container_name: kafka-2
    depends_on:
      - zookeeper
    ports:
      - '${KAFKA_BROKER_2_PORT}:9092'
    environment:
      KAFKA_CFG_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_CFG_LISTENERS: SASL_PLAINTEXT://:9092
      KAFKA_CFG_ADVERTISED_LISTENERS: SASL_PLAINTEXT://host.docker.internal:${KAFKA_BROKER_2_PORT}
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: SASL_PLAINTEXT:SASL_PLAINTEXT
      KAFKA_CFG_INTER_BROKER_LISTENER_NAME: SASL_PLAINTEXT
      KAFKA_CFG_SASL_ENABLED_MECHANISMS: SCRAM-SHA-512,SCRAM-SHA-256
      KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL: SCRAM-SHA-512
      KAFKA_CLIENT_USERS: ${512_SASL_USER},${256_SASL_USER}
      KAFKA_CLIENT_PASSWORDS: ${512_SASL_PASSWORD},${256_SASL_PASSWORD}
      KAFKA_INTER_BROKER_USER: ${512_SASL_USER}
      KAFKA_INTER_BROKER_PASSWORD: ${512_SASL_PASSWORD}
    volumes:
      - kafka_data_2:/bitnami/kafka
    networks:
      - kafka_network
    hostname: kafka-2

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka-0
    ports:
      - '8080:8080'
    environment:
      KAFKA_CLUSTERS_0_NAME: Local-Zookeeper-Cluster
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: host.docker.internal:${KAFKA_BROKER_0_PORT},host.docker.internal:${KAFKA_BROKER_1_PORT},host.docker.internal:${KAFKA_BROKER_2_PORT}
      KAFKA_CLUSTERS_0_PROPERTIES_SECURITY_PROTOCOL: SASL_PLAINTEXT
      KAFKA_CLUSTERS_0_PROPERTIES_SASL_MECHANISM: SCRAM-SHA-512
      KAFKA_CLUSTERS_0_PROPERTIES_SASL_JAAS_CONFIG: org.apache.kafka.common.security.scram.ScramLoginModule required username="${512_SASL_USER}" password="${512_SASL_PASSWORD}";
    networks:
      - kafka_network

  your-app:
    env_file:
      - .env
    build:
      context: .
      dockerfile: dev.Dockerfile
      args:
        - VERSION=dev
    environment:
      - BOOTSTRAP_SERVERS_256=host.docker.internal:${KAFKA_BROKER_0_PORT},host.docker.internal:${KAFKA_BROKER_1_PORT},host.docker.internal:${KAFKA_BROKER_2_PORT}
      - BOOTSTRAP_SERVERS_512=host.docker.internal:${KAFKA_BROKER_0_PORT},host.docker.internal:${KAFKA_BROKER_1_PORT},host.docker.internal:${KAFKA_BROKER_2_PORT}
    image: your-app
    container_name: your-app
    networks:
      - kafka_network
    restart: always
    depends_on:
      - kafka-0
      - kafka-1
      - kafka-2
```

---

## Why Bitnami?

1. **Simple User Registration with Environment Variables**

Bitnami Kafka allows automatic SASL user registration simply by setting the following environment variables:

```env
KAFKA_CLIENT_USERS=user256,user512
KAFKA_CLIENT_PASSWORDS=pass256,pass512
```

In contrast, the official Kafka image requires manually running `kafka-configs.sh` or creating a custom entrypoint.

Example with official Kafka:

```bash
bash -c '
/opt/bitnami/scripts/kafka/setup.sh &&
kafka-configs.sh --zookeeper zookeeper:2181 --alter \
    --add-config "SCRAM-SHA-512=[iterations=8192,password=pass]" \
    --entity-type users --entity-name user &&
/opt/bitnami/scripts/kafka/run.sh'
```

Bitnami handles user registration automatically during container startup.

2. **Built-in SASL and Zookeeper Integration**

Bitnami allows setting SASL and Zookeeper configurations through environment variables without editing `server.properties`:

- `KAFKA_CFG_SASL_ENABLED_MECHANISMS`
- `KAFKA_CFG_LISTENER_NAME_<listener>_SASL_ENABLED_MECHANISMS`
- `KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL`

| Environment Variable | Description | Purpose |
|-----------------------|-------------|---------|
| `KAFKA_CFG_ZOOKEEPER_CONNECT` | Address of Zookeeper (`host:port`) | Required for Kafka to store/share cluster metadata |
| `KAFKA_CFG_LISTENERS` | Protocol and port for external connections | Defines how clients and brokers communicate |
| `KAFKA_CFG_ADVERTISED_LISTENERS` | Address advertised to clients | Provides connection info to Kafka clients |
| `KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP` | Maps listener names to security protocols | Activates SASL authentication |
| `KAFKA_CFG_INTER_BROKER_LISTENER_NAME` | Listener used for broker-to-broker communication | Sets which listener (auth method) is used internally |
| `KAFKA_CFG_SASL_ENABLED_MECHANISMS` | List of allowed SASL mechanisms | Defines available SASL mechanisms |
| `KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL` | SASL mechanism for inter-broker communication | Selects SCRAM algorithm for internal authentication |
| `KAFKA_CLIENT_USERS` | Comma-separated list of users | Registers users in Kafka |
| `KAFKA_CLIENT_PASSWORDS` | Comma-separated list of passwords | Associates passwords with users |
| `KAFKA_INTER_BROKER_USER` | User for inter-broker authentication | Specifies the user for broker-to-broker SASL authentication |
| `KAFKA_INTER_BROKER_PASSWORD` | Password for inter-broker user | Password for the above user |

The environment includes:

- 3-node Bitnami Kafka cluster (supporting both SCRAM-SHA-256 and SCRAM-SHA-512)
- Zookeeper
- Kafka UI for management and testing
- Event Dispatcher application (Kafka Consumer/Producer)

After setting up the `.env` file, start the infrastructure with:

```bash
docker compose --env-file .env up --build
```

To stop and clean up the cluster (including volumes):

```bash
docker compose down -v
```

Example `.env` file:

```env
256_SASL_USER=user256
256_SASL_PASSWORD=pass256
512_SASL_USER=user512
512_SASL_PASSWORD=pass512

# Kafka Settings
KAFKA_BROKER_0_PORT=9092
KAFKA_BROKER_1_PORT=9093
KAFKA_BROKER_2_PORT=9094
```

---

## Detailed Component Overview

The diagram below summarizes how the Kafka cluster, Event Dispatcher, and Kafka UI interact within the local environment. It describes the initialization and authentication sequences based on actual startup logs.

### 1. Zookeeper Initialization

- Zookeeper container starts in standalone mode, acting as metadata storage for Kafka brokers.
- Kafka brokers automatically register SCRAM users via `KAFKA_CLIENT_USERS` and `KAFKA_CLIENT_PASSWORDS`.
- `user256` (SCRAM-SHA-256) and `user512` (SCRAM-SHA-512) are registered in Zookeeper.

### 2. Kafka Broker Startup and Zookeeper Connection

- Brokers (kafka-0, kafka-1, kafka-2) connect sequentially to Zookeeper.
- Once connected, brokers participate in controller election and become ready to serve requests.

### 3. Controller Election

- One broker is elected as the controller.
- The controller synchronizes broker states, partition metadata, and leader elections.
- Messages like `LeaderAndIsr` and `UpdateMetadataRequest` are exchanged to stabilize the cluster.

### 4. Kafka UI Connection

- Kafka UI connects to the brokers using `user512` via SCRAM-SHA-512 authentication.
- After authentication, topics can be browsed, and test messages can be produced.

### 5. Event Dispatcher Connection

- Event Dispatcher connects to the source Kafka using SCRAM-SHA-256 (`user256`) and starts consuming.
- It connects to the destination Kafka using SCRAM-SHA-512 (`user512`) and produces processed messages.

### Full Sequence Diagram

![Kafka Cluster Interaction](image.png)

---

## Troubleshooting and Solutions

### 1. Incorrect `KAFKA_CFG_ADVERTISED_LISTENERS`

Initially, setting `localhost` caused connection failures from Kafka UI and Event Dispatcher.

**Symptoms:**
- Kafka UI or Event Dispatcher couldn't connect.
- `connection refused` or `EOF` errors.

**Solution:**
- Use `host.docker.internal` or the actual host IP instead of `localhost`.

```env
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://host.docker.internal:9092
```

### 2. Authentication Failures with Different SASL Mechanisms

User registrations weren't properly set for both SCRAM mechanisms.

**Symptoms:**
- Kafka UI succeeds, Event Dispatcher fails with `EOF` or auth error.
- Kafka logs show `Failed to authenticate user`.

**Solution:**
- Ensure users are registered properly using `--mechanism` option.
- Verify using logs or `kafka-configs.sh`.

```bash
--entity-type users --entity-name user512 --alter --add-config 'SCRAM-SHA-512=[iterations=4096,password=pass512]'
```

Eventually switched to setting `KAFKA_CLIENT_USERS` and `KAFKA_CLIENT_PASSWORDS`.

### 3. Produce Failures in Kafka UI

Kafka UI failed to produce messages due to auth or metadata issues.

**Symptoms:**
- Produce attempts failed.
- Topic browse failed or auth errors in logs.

**Solution:**
- Correctly configure SCRAM authentication in Kafka UI settings.
- Ensure the user uses SCRAM-SHA-512.

### 4. ISR and Cluster ID Conflicts

Restarting the cluster caused cluster ID mismatch errors.

**Symptoms:**
- Broker failed to start or controller election failed.
- Errors like `Cluster ID mismatch` or `log directory is not empty`.

**Solution:**
- Tear down volumes completely during reset:

```bash
docker compose down -v
```

- If necessary, manually delete `kafka_data_*` directories.

### 5. Missing SASL Config in Producers/Consumers

Producers/Consumers were missing SASL configs.

**Symptoms:**
- `kafka: client has run out of available brokers to talk to`
- Connection attempts ended with `EOF`.

**Solution:**
- Add SASL settings in `.env` and client library configs.
- (e.g., for Go clients: set `Config.Net.SASL.Mechanism` properly)

---

## Conclusion

As security and authentication become essential in Kafka environments, setting up a SASL-authenticated local cluster is highly valuable.

By supporting multiple SASL mechanisms within a single cluster, you can replicate production authentication flows without maintaining separate clusters.

The Kafka UI was particularly impressive — being able to produce and consume messages via a GUI interface was a huge productivity boost.

---

