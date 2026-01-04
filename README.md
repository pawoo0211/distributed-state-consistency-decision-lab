# Distributed State & Data Consistency Decision Lab

Kafka / Flink 기반 분산 스트리밍 환경에서  
**중복, 재처리, 재시작, 지연 이벤트로 인해 발생하는 데이터 정합성 문제를  
다중 저장소(RDB / MongoDB / Elasticsearch)와 CDC 기반 NRT 동기화 전략을 통해 명세화하는 프로젝트**입니다.

본 프로젝트는 정합성을  
**“모든 데이터가 항상 동일해야 한다”가 아니라,  
“데이터의 목적과 조회 특성에 따라 어떤 수준의 일관성을 보장할 것인가”의 문제**로 정의합니다.

---

## Background

분산 스트리밍 시스템에서는 다음과 같은 이유로 데이터 정합성이 쉽게 깨집니다.

- at-least-once 처리로 인한 중복 이벤트
- Checkpoint 실패 및 재시작으로 인한 재처리
- Rebalance에 따른 이벤트 순서 변경
- Late Event 및 State TTL 만료
- 저장소 간 반영 시점 차이로 인한 조회 불일치

특히 서비스 환경에서는 다음 문제가 발생합니다.

- RDB와 조회용 저장소(MongoDB / Elasticsearch)의 데이터 시점이 다름
- 실시간 처리 결과와 조회 결과 간 시간차 발생
- 어떤 차이가 버그인지, 허용 가능한 NRT 지연인지 판단하기 어려움

본 프로젝트는 다음 질문에 답하는 것을 목표로 합니다.

- 데이터 경계(boundary)별로 어떤 정합성 기준이 필요한가?
- NRT 환경에서 어느 시점까지를 “정상”으로 볼 것인가?
- CDC를 활용해 저장소 간 정합성을 어떻게 보조할 것인가?

---

## Architecture

본 프로젝트는  
**분산 처리 → 정합성 판단 → 기준 저장소 반영 → CDC 기반 NRT 동기화 → 조회 일관성 보장**의 흐름으로 구성됩니다.

### Data Flow

Kafka Producer  
→ Kafka Topic  
→ Flink Job (Keyed State, Checkpoint 기반 처리)  
→ Result Topic  
→ Backend API (Consistency Decision Layer)  
→ Relational Database (Source of Truth)  
→ CDC Pipeline  
→ MongoDB / Elasticsearch (NRT Read Models)

---

## Architecture 설명

- Kafka Producer는 중복 및 out-of-order 이벤트를 의도적으로 생성합니다.
- Flink Job은 상태(state)와 checkpoint를 사용해 이벤트를 처리합니다.
- Backend API는 **정합성 판단을 수행하는 유일한 계층**입니다.
- Relational DB는 최종 정합성을 보장하는 **Source of Truth** 역할을 합니다.
- CDC 파이프라인은 RDB 변경 사항을 기반으로 MongoDB와 Elasticsearch를 NRT로 동기화합니다.
- MongoDB와 Elasticsearch는 조회 목적에 맞는 Read Model로 사용됩니다.

---

## Technology Stack

### Streaming / Distributed System
- Apache Kafka
- Apache Flink (Keyed State, Checkpoint)

### Backend
- Spring Boot
- REST API (Consistency Decision Layer)

### Databases
- PostgreSQL (Strong Consistency, Source of Truth)
- MongoDB (Eventually Consistent Read Model)
- Elasticsearch (Search / Aggregation, Near Real-Time)

### CDC / NRT
- Debezium (CDC)
- Kafka Connect
- Kafka Topics for Change Events

### Infrastructure
- Docker
- Docker Compose

---

## Data Consistency Strategy

본 프로젝트에서는 **저장소의 목적에 따라 정합성 전략을 분리**합니다.

### 1. Relational Database (PostgreSQL)

**역할**
- 비즈니스 기준 데이터의 최종 정합성 보장
- 재처리 및 보정의 기준점(Source of Truth)

**정합성 정책**
- 중복 이벤트 → Upsert
- Checkpoint rollback → 저장 거부
- 정합성 위반 → 재처리 대상 표시

---

### 2. CDC Pipeline (NRT Synchronization)

**역할**
- RDB 변경 사항을 기준으로 조회용 저장소 동기화
- 스트리밍 처리 결과와 조회 결과 간 시간차 최소화

**정합성 정책**
- RDB 커밋 이후 변경만 전파
- 순서 보장(Key 기반)
- 지연 허용 범위(SLA) 명시

---

### 3. MongoDB (Read Model)

**역할**
- 서비스 조회용 데이터
- 빠른 읽기와 유연한 스키마 제공

**정합성 정책**
- CDC 기반 NRT 반영
- Eventually Consistent 상태 명시
- confidence_level 기반 조회 판단

---

### 4. Elasticsearch

**역할**
- 검색 및 집계 중심 데이터
- Near Real-Time 조회

**정합성 정책**
- CDC 기반 비동기 반영
- 정확한 수치보다 탐색 가능성 우선
- NRT 지연 허용

---

## Read Model Schema (공통 개념)

각 저장소는 다음 메타데이터를 공통으로 포함합니다.

- business_key
- calculated_value
- confidence_level (HIGH / EVENTUALLY_CONSISTENT / LOW)
- consistency_scope (STRONG / EVENTUAL / SEARCH)
- processing_version
- last_checkpoint_time
- updated_at

정합성은 값 자체가 아니라  
**값 + 신뢰도 + 반영 시점의 조합**으로 표현됩니다.

---

## Consistency Decision Table

| Scenario | Cause | PostgreSQL | MongoDB | Elasticsearch | Service Decision |
|--------|------|-----------|---------|---------------|------------------|
| Restart duplication | at-least-once | Upsert | CDC 반영 | CDC 반영 | confidence=LOW |
| Checkpoint rollback | failure | Reject | No-op | No-op | Reprocess |
| Late event | out-of-order | Update | CDC 반영 | CDC 반영 | Stale 표시 |
| CDC 지연 | NRT lag | Stable | Lagging | Lagging | EVENTUAL 표시 |

---

## Backend Contract

Backend는 분산 시스템과 NRT 환경의 불확실성을  
**명시적인 서비스 계약(contract)**으로 노출합니다.

### Response Semantics

- value: 계산된 결과 값
- confidence: 데이터 신뢰도
- consistency_scope: STRONG / EVENTUAL / SEARCH
- last_checkpoint_time: 처리 기준 시점
- last_cdc_sync_time: 조회 기준 시점

클라이언트는 consistency_scope와 CDC 지연 상태를 기준으로  
즉시 사용, 지연 허용, fallback 여부를 결정합니다.

---

## Failure Injection

다음 장애 상황을 주입해 정합성 및 NRT 동기화 판단을 검증합니다.

- Flink Job 재시작
- TaskManager 강제 종료
- Checkpoint timeout 발생
- CDC 커넥터 일시 중단

각 상황에서 다음을 기록합니다.

- Flink 상태 변화
- RDB와 Read Model 간 시점 차이
- 서비스 응답의 신뢰도 변화

---

## Conclusion

분산 시스템과 NRT 환경에서 데이터 정합성은  
**모든 저장소를 동일하게 맞추는 문제가 아니라,  
각 저장소의 목적에 맞는 일관성 수준을 정의하는 문제**입니다.

본 프로젝트는  
분산 처리, 저장 전략, CDC, 서비스 계약을 하나의 정합성 명세로 통합하는 것을 목표로 합니다.
