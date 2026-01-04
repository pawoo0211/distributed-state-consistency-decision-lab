# Distributed State & Data Consistency Decision Lab

Kafka 기반 분산 이벤트 처리 환경에서  
**중복, 재처리, 지연, 복구로 인해 발생하는 데이터 정합성 문제를  
처리 엔진(Kafka-only / Kafka+Flink), 저장 전략(RDB / MongoDB / Elasticsearch),  
CDC 기반 NRT 동기화, 서비스 계약까지 포함해 명세화하는 프로젝트**입니다.

본 프로젝트는 동일한 정합성 명세를 기준으로,  
**Kafka-only 처리와 Kafka+Flink(stateful) 처리에서 정합성 이슈가 어떻게 다르게 발생하는지 비교하고,  
저장 및 서비스 계약 정책을 검증**하는 것을 목표로 합니다.

---

## Background

분산 이벤트 처리 시스템에서는 다음과 같은 이유로 데이터 정합성이 쉽게 깨집니다.

- at-least-once 처리로 인한 중복 이벤트
- 애플리케이션/잡 재시작으로 인한 재처리
- 파티션 리밸런스로 인한 이벤트 순서 변경
- Late Event 및 상태(State) TTL 만료
- 저장소 간 반영 시점 차이(NRT 지연)

이로 인해 다음과 같은 문제가 발생합니다.

- 저장소마다 데이터 값이 다름
- 조회 시점에 따라 결과가 달라짐
- 어떤 차이가 버그인지, 허용 가능한 결과인지 판단하기 어려움

본 프로젝트는 다음 질문에 답합니다.

- 데이터 정합성은 어디까지 보장해야 하는가?
- 처리 엔진의 차이는 정합성에 어떤 영향을 주는가?
- 분산 시스템의 불확실성을 저장 전략과 서비스 계약으로 어떻게 흡수할 것인가?

---

## Input Model – Event Generator

본 프로젝트의 Kafka Producer는  
**실제 데이터 소스를 대체하는 Event Generator**입니다.

단순한 페이커 데이터나 배치 결과가 아니라,  
**정합성 이슈가 발생할 수밖에 없는 이벤트 흐름을 의도적으로 생성**하는 것이 목적입니다.

### Event Generator의 역할

- 중복 이벤트 생성 (at-least-once 시뮬레이션)
- Out-of-order 이벤트 생성
- Late Event 생성 (event_time < processing_time)
- 재시작 시 동일 이벤트 재전송(replay)

### Event Time Model

각 이벤트는 다음 시간 개념을 포함합니다.

- `event_time`: 실제 비즈니스 이벤트 발생 시점
- `produce_time`: Kafka로 전송된 시점

이 두 시점의 불일치가  
정합성 문제의 핵심 원인이 됩니다.

---

## Architecture

본 프로젝트는  
**이벤트 생성 → 분산 처리 → 정합성 판단 → 기준 저장 → CDC 기반 NRT 동기화 → 조회 계약**의 흐름으로 구성됩니다.

### Data Flow

Event Generator  
→ Kafka Topic  
→ Processing Layer (Kafka-only / Kafka + Flink)  
→ Result Topic  
→ Backend API (Consistency Decision Layer)  
→ Relational Database (Source of Truth)  
→ CDC Pipeline  
→ MongoDB / Elasticsearch (NRT Read Models)

---

## Processing Modes

### Mode A. Kafka-only Processing

- Kafka Consumer 또는 Kafka Streams 기반 처리
- 상태 관리 및 중복 제거를 애플리케이션 레벨에서 직접 수행
- 장애 복구 시 오프셋 커밋 시점에 따라 중복/누락 발생 가능

**특징**
- 구조 단순
- 정합성 책임이 애플리케이션에 집중
- late event, 재처리 시 결과 해석이 어려움

---

### Mode B. Kafka + Flink Processing

- Flink Keyed State 기반 상태 관리
- Checkpoint를 통한 상태 + 오프셋 일관성 유지
- Event Time / Watermark 기반 처리

**특징**
- 분산 상태 머신에서 발생하는 정합성 이슈를 구조적으로 재현
- 재시작, rollback, late event로 인한 결과 변경을 명확히 관측 가능
- 정합성 판단을 위한 입력 데이터로 적합

---

## Architecture 설명

- Event Generator는 정합성 이슈를 유발하는 이벤트를 생성합니다.
- Processing Layer는 Kafka-only / Kafka+Flink 두 모드로 실행됩니다.
- Backend API는 **정합성 판단을 수행하는 유일한 계층**입니다.
- Relational DB는 최종 정합성을 보장하는 Source of Truth 역할을 합니다.
- CDC 파이프라인은 RDB 변경 사항을 기준으로 조회용 저장소를 NRT로 동기화합니다.
- MongoDB와 Elasticsearch는 조회 목적에 맞는 Read Model로 사용됩니다.

---

## Technology Stack

### Streaming / Distributed System
- Apache Kafka
- Apache Flink (Keyed State, Checkpoint, Event Time)

### Backend
- Spring Boot
- REST API (Consistency Decision Layer)

### Databases
- PostgreSQL (Strong Consistency, Source of Truth)
- MongoDB (Eventually Consistent Read Model)
- Elasticsearch (Search / Aggregation, Near Real-Time)

### CDC / NRT
- Debezium
- Kafka Connect

### Infrastructure
- Docker
- Docker Compose

---

## Data Consistency Strategy

저장소의 목적에 따라 정합성 전략을 분리합니다.

### PostgreSQL (Source of Truth)

- 중복 이벤트: Upsert
- Checkpoint rollback: 저장 거부
- 정합성 위반: 재처리 대상 표시

### MongoDB (Read Model)

- CDC 기반 NRT 반영
- Eventually Consistent 허용
- confidence_level 기반 조회 판단

### Elasticsearch

- CDC 기반 비동기 반영
- Near Real-Time 검색
- 정확한 수치보다 탐색 가능성 우선

---

## Read Model Metadata

모든 저장소는 다음 메타데이터를 공통으로 포함합니다.

- business_key
- calculated_value
- confidence_level (HIGH / EVENTUALLY_CONSISTENT / LOW)
- consistency_scope (STRONG / EVENTUAL / SEARCH)
- processing_engine (KAFKA_ONLY / FLINK)
- last_checkpoint_time
- last_cdc_sync_time
- updated_at

정합성은 값 자체가 아니라  
**값 + 처리 방식 + 반영 시점의 조합**으로 표현됩니다.

---

## Consistency Decision Table

| Scenario | Processing | PostgreSQL | MongoDB | Elasticsearch | Service Decision |
|--------|-----------|-----------|---------|---------------|------------------|
| Restart duplication | Kafka-only | Upsert | CDC 반영 | CDC 반영 | confidence=LOW |
| Restart duplication | Flink | Stable | CDC 반영 | CDC 반영 | confidence=HIGH |
| Late event | Kafka-only | Update | Update | Update | Stale 표시 |
| Late event | Flink | Update | Update | Update | Window 재계산 |
| CDC 지연 | All | Stable | Lagging | Lagging | EVENTUAL 표시 |

---

## Backend Contract

Backend는 분산 처리와 NRT 환경의 불확실성을  
**명시적인 서비스 계약(contract)**으로 노출합니다.

### Response Semantics

- value: 계산된 결과
- confidence: 데이터 신뢰도
- consistency_scope: STRONG / EVENTUAL / SEARCH
- processing_engine: KAFKA_ONLY / FLINK
- last_checkpoint_time
- last_cdc_sync_time

클라이언트는 위 정보를 기준으로  
즉시 사용, 지연 허용, fallback 여부를 결정합니다.

---

## Failure Injection

다음 장애 상황을 주입해 정합성 판단을 검증합니다.

- Consumer / Flink Job 재시작
- TaskManager 강제 종료
- Checkpoint timeout
- CDC 커넥터 일시 중단

각 상황에서 다음을 관측합니다.

- 처리 엔진별 데이터 변화
- 저장소 간 시점 차이
- 서비스 응답의 신뢰도 변화

---

## Conclusion

분산 시스템에서 데이터 정합성은  
**모든 데이터를 동일하게 맞추는 문제가 아니라,  
처리 방식과 저장 목적에 맞는 일관성 기준을 정의하는 문제**입니다.

본 프로젝트는  
이벤트 생성, 처리 엔진, 저장 전략, CDC, 서비스 계약을 하나의 정합성 명세로 통합하는 것을 목표로 합니다.
