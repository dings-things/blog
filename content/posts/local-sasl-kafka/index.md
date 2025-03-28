---
date: "2025-03-28"
draft: false
author: dingyu
categories: ["eda"]
tags:
- kafka
- sasl
- bitnami
- kafka ui
- local test
title: '[EDA] Local SASL SCRAM Mechanism Kafka Docker compose 구성하기'
summary: E2E 테스트는 마지막 테스트로서 개발자가 애플리케이션을 잘 개발하였는지 확인하기 위해 꼭 필요한 테스트이다. Kafka를 로컬 환경에서 테스트하고자 한다면 어떻게 할까? SASL 인증 방식을 사용한다면? 
cover:
    image: img/kafka.png
    relative: true
comments: true
description: This post documents how to build a local Kafka cluster using Docker Compose that supports both SCRAM-SHA-256 and SCRAM-SHA-512 SASL authentication mechanisms, enabling secure, production-like testing for applications like event dispatchers—all without modifying code or relying on external infrastructure.
---
# Local SASL Mechanism Kafka Docker compose 구성하기
Kafka를 사용하는 서비스에서 인증 방식이 `SASL` 기반이라면, 로컬에서도 유사한 인증 환경을 구성할 수 있어야 한다. 이 글에서는 로컬 환경에서 `SCRAM-SHA-256`, `SCRAM-SHA-512`를 동시에 지원하는 Kafka 클러스터를 구성하고, 이를 기반으로 Event Dispatcher 애플리케이션을 테스트할 수 있는 인프라 환경을 구축한 내용을 정리했다.

`SASL`과 `TLS`의 경우 레퍼런스를 찾기 매우매우 힘들었다... 삽질하는 참에 추후 고생할 나에게 선물을 마련하고자 로컬 환경 SASL Kafka 구성하기를 기록한다

TLS는 인증서 Makefile 등 너무 복잡하여 추후에 시간이 될 때 포스팅하는것으로..

## 구성 목표

- Event Dispatcher는 source Kafka에서 이벤트를 consume하고, 이벤트 형태에 따라 분류하여 다른 dest Kafka로 produce하는 구조를 갖는다.
- source Kafka와 dest Kafka는 서로 다른 클러스터가 아닌, 동일한 클러스터 내에서 다른 SASL 인증 메커니즘을 통해 구분한다.
- source Kafka는 SCRAM-SHA-256 기반 인증을 사용하고, dest Kafka는 SCRAM-SHA-512 기반 인증을 사용한다.
- 테스트 환경에서도 코드 수정 없이 실제 운영 환경과 유사한 구조를 구축하는 것이 목표였다.
- ISR(in-sync replicas)을 2로 두기 위해 Kafka 브로커는 3노드로 구성했다.
- Kafka UI를 통해 produce 테스트가 가능해야 하며, 이를 통해 애플리케이션의 동작 확인도 가능하도록 한다.

## 로컬 환경 구성을 고려하게 된 이유

- 개발 서버에서 테스트 시 스택 트레이스 확인이 어려워 디버깅이 불편했으며, Bastion(보안용 게이트웨이) 사용을 위한 VPN 및 SSH 접근이 상대적으로 피로감을 느끼게 했다
- 사내 Kafka는 모두 SASL 인증 기반이므로 공통 설정을 만들어두면 다양한 프로젝트에 활용할 수 있다.
- 설정 값을 테스트할 때 코드 수정과 DEV 환경 배포 없이 변경 가능해야 했다. (트랜잭션 설정, exactly-once, ISR 등)
- 코드 레벨 수정 없이 테스트하려면 로컬 인프라 구성이 필요했다.

## Docker Compose 기반 환경 구성

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
  
  # user 정보가 broker에 저장이 되어야 정상적으로 시작될 수 있음
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

### 어쨰서 bitnami?
1. 환경 변수 기반의 간편한 사용자 등록
Bitnami Kafka는 다음과 같은 환경 변수만 설정하면 자동으로 SASL 사용자 등록을 수행
```env
KAFKA_CLIENT_USERS=user256,user512
KAFKA_CLIENT_PASSWORDS=pass256,pass512
```
이는 일반 Kafka 공식 이미지에서는 수동으로 kafka-configs.sh 스크립트를 실행하거나 custom entrypoint를 만들어야 가능!

ex.
```bash
bash -c '
/opt/bitnami/scripts/kafka/setup.sh &&
kafka-configs.sh --zookeeper zookeeper:2181 --alter \
    --add-config "SCRAM-SHA-512=[iterations=8192,password=pass]" \
    --entity-type users --entity-name user &&
/opt/bitnami/scripts/kafka/run.sh'
```

Bitnami는 컨테이너 기동 시점에 사용자 등록 로직을 자동 수행해줘서 훨씬 편리

2. SASL 설정 및 Zookeeper 연동이 기본 내장
Bitnami는 다음 설정들을 Docker 환경변수로 설정 가능

- KAFKA_CFG_SASL_ENABLED_MECHANISMS
- KAFKA_CFG_LISTENER_NAME_<listener>_SASL_ENABLED_MECHANISMS
- KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL

즉, 복잡한 server.properties 없이도 docker-compose.yml만으로 SASL 클러스터 구성 가능!

| 환경변수 | 설명 | 사용처 |
|----------|------|--------------|
| `KAFKA_CFG_ZOOKEEPER_CONNECT` | Kafka가 사용할 Zookeeper의 주소 (`host:port`) | Kafka가 Zookeeper 기반으로 클러스터 메타데이터를 저장/공유하기 위해 필요함 |
| `KAFKA_CFG_LISTENERS` | 브로커가 어떤 프로토콜과 포트로 외부 연결을 수신할지 설정 (ex: `SASL_PLAINTEXT://:9092`) | 클라이언트 또는 브로커 간 통신을 어떤 방식으로 받을지 정의함 |
| `KAFKA_CFG_ADVERTISED_LISTENERS` | 브로커가 클라이언트에게 자신을 알릴 때 사용하는 주소 (`host.docker.internal:${PORT}`) | Kafka 클라이언트가 브로커에 접속할 때 사용할 외부 IP 또는 호스트명과 포트 정보를 제공함 |
| `KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP` | 각 리스너 이름에 대해 사용할 보안 프로토콜을 매핑 (ex: `SASL_PLAINTEXT:SASL_PLAINTEXT`) | 리스너 이름과 보안 프로토콜을 매칭하여 SASL 인증을 활성화하기 위해 사용 |
| `KAFKA_CFG_INTER_BROKER_LISTENER_NAME` | 브로커 간 통신에 사용할 리스너 이름 | 클러스터 내의 브로커 간 통신을 어떤 리스너(인증 방식)로 할지 지정함 |
| `KAFKA_CFG_SASL_ENABLED_MECHANISMS` | Kafka에서 허용할 SASL 인증 메커니즘 목록 (ex: `SCRAM-SHA-512,SCRAM-SHA-256`) | Kafka 클라이언트와 브로커 간 인증에 사용할 수 있는 SASL 방식들을 정의함 |
| `KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL` | 브로커 간 통신에서 사용할 SASL 인증 메커니즘 | 브로커들끼리 통신 시 어떤 SCRAM 알고리즘으로 인증할지 설정함 |
| `KAFKA_CLIENT_USERS` | SASL 인증에 사용할 사용자 목록 (쉼표로 구분) | Kafka 사용자 등록을 위해 필요하며, 각 사용자마다 비밀번호가 필요함 |
| `KAFKA_CLIENT_PASSWORDS` | 사용자 목록에 대응되는 비밀번호 목록 (쉼표로 구분) | 위의 사용자 각각에 대응되는 비밀번호를 정의하여 Kafka 서버에 등록함 |
| `KAFKA_INTER_BROKER_USER` | 브로커 간 통신에서 사용할 사용자 이름 | 브로커들끼리 SCRAM 인증을 위해 사용하는 사용자 지정 |
| `KAFKA_INTER_BROKER_PASSWORD` | 브로커 간 통신에 사용할 사용자의 비밀번호 | 위의 사용자와 함께 브로커 간 인증을 위해 사용됨 |


해당 환경은 다음의 구성 요소로 이루어진다:

- Bitnami Kafka 3노드 클러스터 (SCRAM-SHA-256, SCRAM-SHA-512 동시 지원)
- Zookeeper
- Kafka UI (관리 및 테스트 용도)
- Event Dispatcher 애플리케이션 (Kafka Consumer/Producer)

`.env`와 파일을 구성한 후, 다음 명령어로 전체 인프라를 실행할 수 있다

```
docker compose --env-file .env up --build
```

종료 및 클러스터 초기화를 위해서는 아래 명령어를 사용한다:

```
docker compose down -v
```

`.env` example:
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

## 구성 세부 내용 요약

아래 다이어그램은 로컬 환경에서 Kafka 클러스터가 기동되고 Event Dispatcher와 Kafka UI가 상호 작용하는 흐름을 정리한 것이다. 내부적으로 어떤 초기화가 발생하고, 인증 및 통신이 어떤 순서로 진행되는지 이해할 수 있다. 각 단계는 실제 로그 흐름에 기반하여 순서대로 설명하였다.

### 1. Zookeeper 초기화
- Zookeeper 컨테이너가 standalone 모드로 기동되며 Kafka 브로커의 메타데이터 저장소로 동작한다.
- 사용자 정보 등록은 Zookeeper가 직접 수행하지 않으며, Kafka 브로커가 기동 시 `KAFKA_CLIENT_USERS`, `KAFKA_CLIENT_PASSWORDS` 환경변수를 통해 SCRAM 사용자 정보를 자동으로 등록한다.
- Zookeeper 컨테이너가 standalone 모드로 기동되며 사용자 정보를 등록할 준비를 한다.
- `user256` (SCRAM-SHA-256), `user512` (SCRAM-SHA-512)에 대한 사용자 정보가 Zookeeper에 등록된다.

### 2. Kafka 브로커 기동 및 Zookeeper 연결
- Kafka 브로커 3개(kafka-0, kafka-1, kafka-2)가 순차적으로 Zookeeper와 연결을 시도한다.
- 연결이 완료되면 각 브로커는 controller 선출에 참여하고, request 처리 준비가 완료되었음을 로그로 출력한다.

### 3. Controller 동작
- Kafka 클러스터 내에서 한 브로커가 controller로 선출된다.
- controller는 각 브로커의 상태를 확인하고, 토픽 메타데이터를 동기화하며, Partition 할당과 Leader 선출을 수행한다.
- 내부적으로 `LeaderAndIsr`, `UpdateMetadataRequest` 등의 메시지를 각 브로커에 전파하며 클러스터 상태를 안정화시킨다.

### 4. Kafka UI 연결
- Kafka UI는 SASL-SHA-512 사용자(user512)를 이용해 Kafka 브로커에 AdminClient로 연결한다.
- 인증 후 topic 목록 조회 및 메시지 produce 테스트를 수행한다.

### 5. Event Dispatcher 연결
- Event Dispatcher는 source Kafka에 SCRAM-SHA-256 방식(user256)으로 Consumer 연결을 시도하고 consume을 시작한다.
- 이후 dest Kafka에는 SCRAM-SHA-512 방식(user512)으로 Producer 연결하여 메시지를 produce한다.

### 전체 시퀀스 다이어그램

![](image.png)


로컬 환경에서 Kafka 클러스터가 기동되고 Event Dispatcher와 Kafka UI가 상호 작용하는 흐름은 다음과 같다

내부적으로 어떤 초기화가 발생하고, 인증 및 통신이 어떤 순서로 진행되는지 확인해보자!

- Kafka 클러스터는 `Zookeeper` 기반이며, 3개의 브로커로 구성
- SCRAM-SHA-256을 사용하는 `user256`과 SCRAM-SHA-512를 사용하는 `user512`가 생성
- Kafka UI를 통해 `user512` 인증으로 접근하여 메시지를 직접 produce 가능
- Event Dispatcher는 SASL 256/512 User/Password 설정을 기반으로 source/dest Kafka 각각 다른 인증 방식으로 접근하여 동작한다.

## 트러블슈팅 및 해결 과정

Kafka SASL 인증 기반 환경을 처음부터 구성하다 보면 여러 트러블을 겪게 된다. 실제 테스트 환경에서 마주쳤던 문제와 그에 대한 해결 과정을 아래에 정리했다.

### 1. KAFKA_CFG_ADVERTISED_LISTENERS 설정 오류

초기 구성 시 `KAFKA_CFG_ADVERTISED_LISTENERS` 값을 잘못 설정하여 Kafka UI 및 Event Dispatcher에서 브로커 접근에 실패했다. `localhost`로 설정할 경우 Docker 내부의 컨테이너 외부에서는 Kafka 브로커에 접근할 수 없었다.

**현상:**
- Kafka UI 또는 Event Dispatcher가 브로커와 통신할 수 없음
- `connection refused` 또는 `EOF` 에러 발생

**해결 방법:**
- `localhost` 대신 `host.docker.internal` 또는 실제 호스트 IP로 설정
- `.env` 또는 compose 파일 내 브로커 설정 확인

```
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://host.docker.internal:9092
```

### 2. 서로 다른 SASL 메커니즘을 가진 사용자 인증 실패

Kafka 클러스터 하나에서 `SCRAM-SHA-256`, `SCRAM-SHA-512`를 동시에 지원하게끔 설정했지만, 사용자 등록이 제대로 되지 않아 인증에 실패했다.

**현상:**
- Kafka UI는 인증에 성공하나, Event Dispatcher는 EOF 또는 인증 실패
- Kafka 로그에 `Failed to authenticate user` 메시지 출력

**해결 방법:**
- Kafka 초기화 스크립트에서 사용자 추가 시 `--mechanism` 플래그 명확히 지정
- 사용자 등록 확인을 위해 Kafka 실행 로그 확인 또는 `kafka-configs.sh` 이용해 확인

```
--entity-type users --entity-name user512 --alter --add-config 'SCRAM-SHA-512=[iterations=4096,password=pass512]'
```
- 추후에는 Broker에서 직접 실행할 필요가 없었음을 느끼고 `KAFKA_CLIENT_USERS`, `KAFKA_CLIENT_PASSWORDS` 설정!

### 3. Kafka UI를 통한 produce가 동작하지 않음

Kafka UI를 통해 메시지를 produce할 수 있어야 Event Dispatcher의 consume 확인이 가능하지만, 인증 실패나 메타데이터 조회 실패로 UI가 제대로 동작하지 않았다.

**현상:**
- UI 상에서 produce 시도했지만 메시지가 전송되지 않음
- 토픽 조회 실패 또는 사용자 인증 실패 로그

**해결 방법:**
- UI 클러스터 설정에 SCRAM 인증 정보를 추가하고 SASL 메커니즘을 정확히 지정
- 인증에 사용하는 사용자 계정이 SCRAM-SHA-512를 사용하고 있어야 함

### 4. ISR 설정과 클러스터 ID 충돌 문제

Kafka 브로커를 3개로 구성하여 ISR=2 설정을 테스트하던 중, 클러스터를 재기동하면 클러스터 ID 충돌로 인해 broker가 기동되지 않는 문제가 발생했다.

**현상:**
- Kafka broker가 기동 중단 또는 controller election 실패
- `Cluster ID mismatch` 또는 `log directory is not empty` 오류

**해결 방법:**
- 클러스터 재기동 시 아래 명령어로 볼륨까지 완전 제거

```
docker compose down -v
```

- 필요 시 `kafka_data_*` 디렉터리를 수동으로 삭제하여 클린 상태 유지

### 5. SASL 메커니즘에 맞는 Producer/Consumer 설정 누락

Event Dispatcher 설정 중 Kafka Producer와 Consumer 설정 시 SASL 인증 방식 지정이 빠져 인증 실패가 발생했다.

**현상:**
- `kafka: client has run out of available brokers to talk to` 에러
- 접속 시도는 하나 서버에서 연결을 끊음 (EOF)

**해결 방법:**
- `.env`에 SASL 관련 설정 추가하고, 라이브러리 설정에 반영
- Go 클라이언트 기준으로는 `Config.Net.SASL.Mechanism` 명시 필요

---

이러한 문제들을 하나씩 해결해가면서 현재와 같은 안정적인 로컬 테스트 환경을 구성할 수 있었다.

## 마무리

Kafka 환경이 점차 보안과 인증 요구사항을 갖게 되면서, SASL 기반 인증 환경을 로컬에서도 구축해보는 것은 매우 의미 있는 작업이었다. 

단일 클러스터에서 다중 SASL 메커니즘을 지원함으로써, 더 이상 클러스터를 이중으로 구성할 필요 없이 하나의 테스트 환경으로 다양한 인증 흐름을 재현할 수 있게 되었다.

특히... 감동받은 부분은 Kafka UI! GUI로 Message 생성하고 Consume 됨을 확인할 수 있음에 감격했다 

