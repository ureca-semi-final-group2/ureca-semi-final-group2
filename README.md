# 🐙 말 많은 무너팀 (유레카 3기 대면 백엔드)

<div align="center">

  <img src="https://github.com/user-attachments/assets/9a54a772-af78-4451-bc60-0528c480b4d6" width="470" alt="말많은무너팀 로고" />

  <h3>"말은 많지만 결과는 확실한, 유레카 대면 종합 프로젝트"</h3>

  <p>대용량 통신 요금 명세서 및 알림 발송 시스템</p>

  [![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](#)
  [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

</div>

<br/>

## 📌 목차

1. [프로젝트 소개](#-프로젝트-소개)
2. [주요 기능](#-주요-기능)
3. [아키텍처 및 전체 파이프라인 흐름](#-아키텍처-및-전체-파이프라인-흐름)
4. [기술적 특이점 및 도전 과제](#-기술적-특이점-및-도전-과제)
5. [데이터베이스 설계 (Single Source of Truth)](#-데이터베이스-설계-single-source-of-truth)
6. [발송 프로세스 & Kafka 로직](#-발송-프로세스--kafka-로직)
7. [Billing 테이블 설계](#-billing-테이블-설계)
8. [기술 스택](#-기술-스택)
9. [시작 가이드](#-시작-가이드)
10. [팀원 소개](#-팀원-소개)

<br>

## 📝 1. 프로젝트 소개

* **개발 기간**: 2026.01.06 ~ 2026.01.27

* **서비스 명**: 대용량 통신 요금 명세서 및 알림 발송 시스템

* **핵심 목표**: 통신사의 핵심 업무인 대규모 정산과 알림 발송을 안정적으로 처리하기 위한 시스템입니다. 매월 수백만 건에 달하는 요금 청구 데이터를 배치 기반으로 정확하게 정산하고, 정산 결과를 기반으로 요금 청구서를 고객에게 이메일과 SMS를 통해 발송합니다.

* **배포 주소**: [🚀 서비스 바로가기 링크]()

<br>

## ✨ 2. 주요 기능

### ✅ 요금 정산 및 발송 배치 시스템

* **대용량 데이터 정산**: 월별 약 100만 건의 청구서와 500만 건의 세부 내역 데이터를 **PostgreSQL Range Partitioning** 기반 테이블에 안정적으로 적재합니다.

* **배치 성능 최적화**: 
    - **Chunk(1,000건) 단위 처리**: 대량 데이터를 청크 단위로 분할하여 메모리 효율성을 확보합니다.
    - **Preload Listener 적용**: `ItemReadListener`를 통해 ID를 선수집하고 `IN` 절로 일괄 조회하여 **N+1 쿼리 문제를 해결**, DB I/O 성능을 극대화했습니다.

* **부하 분산 발송**: 모든 데이터를 한 번에 발송하지 않고, 고객의 급여일/결제일을 고려한 분산 **발송 전략**(15일, 21일)을 적용하여 시스템 피크 부하를 물리적으로 분산했습니다.

### ✅ 이벤트 기반 스마트 알림 발송 (DND & DLT)

* **비동기 논블로킹 발송**:
    - **전용 스레드 풀(EmailExecutor)**: `corePoolSize(50)`, `maxPoolSize(100)` 설정을 통해 초당 50~100건의 병렬 발송 능력을 확보했습니다.
    - **Backpressure 관리**: 스레드 풀 큐가 꽉 찰 경우 `CallerRunsPolicy`를 적용하여 유입 속도를 조절, 시스템 가용성을 유지합니다.

* **정교한 금칙 시간(DND) 제어**: 
    - **QuietHourService**: 자정을 경과하는 시간 범위(예: 22:00~02:00)까지 완벽히 판별하는 논리 로직을 구현했습니다.
    - **상태 기반 재처리**: 금칙 시간에 걸린 발송 건은 DB 상태를 `IN_QUIET_HOUR`로 변경하고, **매시간 구동되는 재발송 배치**를 통해 자동으로 Kafka에 재투입합니다.

* **장애 복원력(Resilience)**:
    - **멱등성 보장**: Kafka 재시도 시에도 DB의 `SendStatus`를 우선 확인하여 중복 발송을 원천 차단합니다.
    - **DLT(Dead Letter Topic)**: 최종 실패 메시지는 DLT로 격리하고, `EmailFailLog` 테이블에 페이로드를 저장하여 관리자가 사후 대응할 수 있게 설계했습니다.

### ✅ 실시간 모니터링 및 관리자 관제

* **메트릭 수집**: **Prometheus**를 통해 배치 처리량, Kafka Lag, 스레드 풀 상태, 시스템 리소스를 실시간 수집합니다.

* **시각화 대시보드**: **Grafana**를 활용하여 정산 성공률, 발송 진행 현황, 에러율을 시각화하여 모니터링합니다.

* **관리자 대응 기능**: 유저 검색을 통한 **강제 발송** 및 실패 로그 기반의 **수동 SMS 전환 발송** 기능을 제공합니다.

<br>

## 🏗 3. 아키텍처 및 전체 파이프라인 흐름

### 3.1 시스템 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────┐
│                    물리적 인프라 (EC2 단일 인스턴스)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐ │
│  │  Billing Batch  │  │  Sending Batch   │  │ Kafka Consumer │ │
│  │   (Profile:     │  │   (Profile:     │  │  (Profile:      │ │
│  │ billing-batch)  │  │   producer)     │  │   consumer)     │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘ │
│           │                     │                     │         │
│           │                     │                     │         │
│           ▼                     ▼                     ▼         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              PostgreSQL Database                         │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │  billing_p (Range Partition by billing_date)     │   │  │
│  │  │  - billing_p_2025_01                             │   │  │
│  │  │  - billing_p_2025_02                             │   │  │
│  │  │  - ...                                            │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                     │                     │         │
│           │                     │                     │         │
│           └─────────────────────┴─────────────────────┘         │
│                                 │                               │
│                                 ▼                               │
│                    ┌─────────────────────────┐                 │
│                    │   Kafka Cluster         │                 │
│                    │   (KRaft Mode)          │                 │
│                    │   - 3 Partitions        │                 │
│                    │   - DLT Topic           │                 │
│                    └─────────────────────────┘                 │
│                                 │                               │
│                                 ▼                               │
│                    ┌─────────────────────────┐                 │
│                    │   Email Service          │                 │
│                    │   (Handlebars Template) │                 │
│                    └─────────────────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 논리적 분산 환경 (Spring Profile 기반)

물리적으로는 단일 EC2 인스턴스를 사용하지만, **Spring Profile**을 통해 논리적으로 완전히 분리된 환경을 구축했습니다:

- **`billing-batch` 프로파일**: 월별 정산 배치 작업만 실행
- **`producer` 프로파일**: 발송 배치 작업 및 Kafka Producer 활성화
- **`consumer` 프로파일**: Kafka Consumer 및 이메일 발송 처리

이를 통해 각 프로세스가 독립적으로 동작하며, 필요에 따라 선택적으로 활성화할 수 있습니다.

### 3.3 전체 파이프라인 흐름


<img width="8192" height="3178" alt="Billing Kafka Data Pipeline-2026-01-26-133835" src="https://github.com/user-attachments/assets/72f95eba-12fa-4500-adc1-378579386717" />

#### 1단계: 월간 정산 (매월 1일 자정)

```
[BillingSchedule] 
  └─> @Scheduled(cron = "0 0 0 1 * *")
      └─> JobLauncher.run("billingJob")
          └─> [Master Step] 4개 파티션으로 분할
              └─> [Worker Step] 각 파티션 병렬 처리
                  └─> Reader: JdbcPagingItemReader (Chunk 1,000)
                      └─> Processor: 할인 정책 계산
                          └─> Writer: Billing 저장 (SendStatus.CREATED)
```

**처리 결과**:
- 약 100만 건의 청구서 생성
- 약 500만 건의 세부 내역 데이터 생성
- 모든 데이터는 `SendStatus.CREATED` 상태로 저장
- `billing_date` 기준으로 Range Partition 테이블에 분산 저장

#### 2단계: 발송 대상 확정 및 발행 (매월 15일/21일 자정)

```
[SendBillingScheduler]
  └─> @Scheduled(cron = "0 0 0 15,21 * *")
      └─> JobLauncher.run("sendingJob")
          └─> Reader: CREATED 상태 + targetDay(15 or 21) 조회
              └─> MemberPreloadListener: 회원 정보 사전 로딩
                  └─> Processor: BillingProducerMessageDto 변환
                      └─> Writer: Kafka Producer로 메시지 발행
                          └─> DB 상태 업데이트: CREATED → SEND_PENDING
```

**처리 결과**:
- `targetDay`와 `CREATED` 상태를 만족하는 대상 조회
- Kafka 토픽(`queuing.billing.email.send`)으로 메시지 발행
- DB 상태를 `SEND_PENDING`으로 변경
- 3개 파티션에 분산하여 발행 (부하 분산)

#### 3단계: 실시간 필터링 및 금칙 시간(DND) 판단

```
[KafkaConsumerListener]
  └─> @KafkaListener(topics = "queuing.billing.email.send")
      └─> CompletableFuture.runAsync(() -> {
            └─> BillingDispatchService.process()
                └─> processInternal() [트랜잭션 내부]
                    └─> Billing 조회 (복합키: id + billingDate)
                        └─> 상태 체크:
                            ├─> COMPLETED? → Skip (멱등성 보장)
                            └─> QuietHourService.isDndTime() 체크
                                ├─> 금칙 시간? → IN_QUIET_HOUR로 변경
                                └─> 금칙 시간 아님 → 이메일 발송 진행
```

**금칙 시간 판단 로직**:
```java
// 자정 경과 시간대 처리 (예: 22:00 ~ 02:00)
if (startDndTime.isBefore(endDndTime)) {
    // 같은 날 내 범위 (예: 10:00 ~ 12:00)
    return now >= start && now < end;
} else {
    // 자정 경과 범위 (예: 22:00 ~ 02:00)
    return now >= start || now < end;
}
```

#### 4단계: 비동기 발송 및 트랜잭션 분리

```
[EmailExecutor Thread Pool]
  └─> sendEmailAfterTransaction() [트랜잭션 외부]
      └─> EmailService.sendBillingEmail()
          └─> Handlebars 템플릿 렌더링
              └─> 외부 이메일 API 호출
                  ├─> 성공 → updateStatusToCompleted()
                  │       └─> SEND_PENDING → COMPLETED
                  └─> 실패 → updateStatusToFailed()
                          └─> SEND_PENDING → FAILED
                              └─> DLT로 전송
```

**트랜잭션 분리 전략**:
- **트랜잭션 1**: DB 조회 및 상태 업데이트 (IN_QUIET_HOUR)
- **트랜잭션 외부**: 이메일 발송 (외부 API 호출)
- **트랜잭션 2**: 성공 시 COMPLETED 업데이트
- **트랜잭션 3**: 실패 시 FAILED 업데이트

이를 통해 외부 API 호출이 DB 트랜잭션을 블로킹하지 않습니다.

#### 5단계: 재발송 및 복구 (매시간)

```
[SendBillingScheduler]
  └─> @Scheduled(cron = "0 0 * 15,21 * *")  // 매시간 실행
      └─> JobLauncher.run("sendingJob")
          └─> Reader: IN_QUIET_HOUR 상태 조회
              └─> Processor: 금칙 시간 해제 여부 확인
                  └─> Writer: Kafka로 재투입
                      └─> 상태 업데이트: IN_QUIET_HOUR → SEND_PENDING
```

**재발송 로직**:
- 매시간 `IN_QUIET_HOUR` 상태인 데이터를 조회
- 현재 시간이 금칙 시간이 아니면 Kafka로 재투입
- 멱등성 보장을 위해 상태를 먼저 확인

#### 6단계: DLT 처리 및 실패 로그 저장

```
[KafkaConsumerListener]
  └─> @DltHandler
      └─> handleDlt()
          └─> EmailFailLog 저장
              └─> ParseStatus.SUCCESS or FAIL
                  └─> 관리자 대시보드에서 조회 가능
```

**DLT 처리 흐름**:
- 재시도 실패 또는 특정 예외 발생 시 자동으로 DLT 전송
- `@DltHandler` 메서드에서 `EmailFailLog` 테이블에 저장
- 원본 메시지 페이로드를 JSON으로 저장하여 사후 분석 가능

<br>

## 🚀 4. 기술적 특이점 및 도전 과제

### 4.1 대용량 데이터 처리 전략

#### 파티셔닝 기반 병렬 처리

**Master-Worker 패턴**을 통해 대용량 데이터를 효율적으로 처리합니다:

```java
// 파티션 설정
private static final int GRID_SIZE = 4;  // 동시 실행 파티션 수
private static final int CHUNK_SIZE = 1000;  // 청크 크기

// 파티션 키: publicInfoId의 해시값
WHERE mod(abs(hashtext(t.sort_id::text)), :gridSize) = :partition
```

**성능 향상 효과**:
- 4개 파티션으로 병렬 처리하여 처리 시간 약 75% 단축
- 각 파티션은 독립적으로 실행되어 장애 격리 가능

#### N+1 쿼리 문제 해결

**Preload Listener 패턴**을 통해 N+1 쿼리 문제를 해결했습니다:

```java
@BeforeChunk
public void beforeChunk(ChunkContext context) {
    // 청크 시작 전 ID 선수집
    List<String> publicInfoIds = extractPublicInfoIds();
    
    // IN 절로 일괄 조회
    Map<String, MemberCredential> members = 
        memberRepository.findByIdIn(publicInfoIds);
    
    // 캐시에 저장
    preloadHolder.setMembers(members);
}
```

**효과**:
- 청크당 1,000건 처리 시 쿼리 수: 1,000회 → 1회로 감소
- DB I/O 부하 99% 감소

### 4.2 비동기 논블로킹 발송 시스템

#### Thread Pool 기반 병렬 처리

```java
@Bean(name = "emailExecutor")
public Executor emailExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(50);      // 기본 스레드 수
    executor.setMaxPoolSize(100);       // 최대 스레드 수
    executor.setQueueCapacity(1000);   // 대기 큐
    executor.setRejectedExecutionHandler(
        new ThreadPoolExecutor.CallerRunsPolicy()  // Backpressure
    );
    return executor;
}
```

**처리 능력**:
- 초당 50~100건의 병렬 발송 가능
- 큐가 꽉 차면 `CallerRunsPolicy`로 유입 속도 자동 조절

#### 트랜잭션 분리 전략

외부 API 호출과 DB 업데이트를 완전히 분리하여 시스템 안정성을 확보했습니다:

```java
// 1. 트랜잭션 내부: DB 조회 및 상태 업데이트
@Transactional
public ProcessResult processInternal(BillingProducerMessageDto dto) {
    // DB 작업만 수행
}

// 2. 트랜잭션 외부: 이메일 발송
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void sendEmailAfterTransaction(...) {
    // 외부 API 호출
}

// 3. 트랜잭션 내부: 성공/실패 상태 업데이트
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void updateStatusToCompleted(...) {
    // 상태 업데이트
}
```

### 4.3 정교한 금칙 시간(DND) 처리

#### 자정 경과 시간대 처리

자정을 넘어가는 금칙 시간 범위(예: 22:00 ~ 02:00)를 정확히 처리합니다:

```java
public boolean isDndTime(LocalTime startDndTime, LocalTime endDndTime, LocalTime now) {
    if (startDndTime.isBefore(endDndTime)) {
        // 같은 날 내 범위 (예: 10:00 ~ 12:00)
        return now >= start && now < end;
    } else {
        // 자정 경과 범위 (예: 22:00 ~ 02:00)
        return now >= start || now < end;
    }
}
```

**처리 시나리오**:
- 22:00 ~ 02:00 설정 시:
  - 23:00 → 금칙 시간 ✅
  - 01:00 → 금칙 시간 ✅
  - 03:00 → 금칙 시간 아님 ❌

### 4.4 멱등성 보장 및 중복 발송 방지

Kafka 재시도 시에도 중복 발송을 방지하기 위해 DB 상태를 우선 확인합니다:

```java
// 상태 체크를 통한 멱등성 보장
if (billing.getSendStatus() == SendStatus.COMPLETED) {
    log.info("이미 완료된 건입니다. 스킵합니다.");
    return ProcessResult.skip();
}
```

**보장 사항**:
- 동일 메시지가 여러 번 수신되어도 한 번만 발송
- 네트워크 장애로 인한 재시도 시에도 안전

<br>

## 🗄 5. 데이터베이스 설계 (Single Source of Truth)

### 5.1 Billing 테이블 설계

#### Primary Key (복합키)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | BIGINT | 사용자 청구 ID (TSID 전략) |
| `billing_date` | TIMESTAMP | 청구 기준 월/일 (파티셔닝 키) |

**복합키 사용 이유**:
- 동일 사용자의 여러 달 청구서를 구분
- Range Partitioning과 자연스럽게 연동
- `billing_date` 기준으로 파티션 자동 분할

#### 주요 컬럼

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `public_info_id` | VARCHAR | 회원 공개 정보 ID |
| `usage_id` | BIGINT | 사용량 ID |
| `billing_fee` | INTEGER | 청구 금액 |
| `status` | ENUM | 결제 상태 (PayStatus) |
| `send_status` | ENUM | 발송 상태 (SendStatus) |
| `paid_date` | TIMESTAMP | 결제 완료 시점 |
| `billing_details` | JSONB | 청구 상세 내역 (할인 정보 포함) |

#### SendStatus 상태 정의

```java
public enum SendStatus {
    // 1. 정산 단계
    CREATED,              // 정산 배치가 완료되어 청구 데이터 생성 완료
    
    // 2. 발송 발행 단계
    SEND_PENDING,         // 발송 배치(15, 21일)가 실행되어 카프카에 메시지 프로듀서 시작
    
    // 3. 발송 처리 단계 (Consumer)
    IN_QUIET_HOUR,       // 금칙시간에 걸려 대기 상태
    SENDING,             // 금칙시간 x 전송 완료 (실제로는 사용 안 함)
    
    // 4. 최종 결과
    COMPLETED,           // 이메일 발송 완료
    FAILED               // 발송 실패 (이메일 주소 오류, API 장애 등)
}
```

### 5.2 Range Partitioning 전략

PostgreSQL의 Range Partitioning을 활용하여 월별로 테이블을 분할합니다:

```sql
-- 파티션 테이블 생성
CREATE TABLE billing_p (
    id BIGINT,
    billing_date TIMESTAMP,
    ...
) PARTITION BY RANGE (billing_date);

-- 월별 파티션 생성
CREATE TABLE billing_p_2025_01 PARTITION OF billing_p
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE billing_p_2025_02 PARTITION OF billing_p
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

**장점**:
- 월별 데이터가 물리적으로 분리되어 조회 성능 향상
- 오래된 파티션은 별도 스토리지로 이동 가능 (아카이빙)
- 파티션 단위 백업 및 복구 가능

### 5.3 JSONB 컬럼 활용

`billing_details`는 JSONB 타입으로 할인 내역을 저장합니다:

```json
{
  "discounts": [
    {
      "type": "FAMILY",
      "amount": 5000,
      "description": "가족 할인"
    },
    {
      "type": "LOYALTY",
      "amount": 3000,
      "description": "로열티 할인"
    }
  ],
  "totalDiscount": 8000,
  "finalAmount": 42000
}
```

**장점**:
- 스키마 변경 없이 유연한 데이터 저장
- PostgreSQL JSONB 인덱스로 빠른 조회 가능
- 할인 정책 추가 시 기존 데이터 구조 유지

<br>

## 📤 6. 발송 프로세스 & Kafka 로직

### 6.1 Kafka 토픽 구조

#### 메인 토픽

```
토픽명: queuing.billing.email.send
파티션: 3개
복제본: 1개 (개발), 3개 권장 (운영)
Consumer Group: cg-billing-email-send
```

**파티션 분산 전략**:
- 3개 파티션으로 부하 분산
- 동일 사용자의 메시지는 동일 파티션으로 라우팅 (키 기반)
- 파티션별 독립적인 처리로 장애 격리

#### DLT (Dead Letter Topic)

```
토픽명: queuing.billing.email.send.dlt
파티션: 3개
복제본: 1개 (개발)
```

**DLT 역할**:
- 재시도 실패 메시지 저장
- `EmailFailLog` 테이블에 자동 저장
- 관리자가 사후 분석 및 수동 처리 가능

### 6.2 Kafka Consumer 설정

#### Ack Mode: MANUAL_IMMEDIATE

```java
factory.getContainerProperties().setAckMode(
    ContainerProperties.AckMode.MANUAL_IMMEDIATE
);
```

**동작 방식**:
- `ack.acknowledge()` 호출 시 즉시 오프셋 커밋
- 비동기 작업 완료 후 `whenComplete`에서 Ack 호출
- Zombie Consumer 방지 (항상 Ack 호출 보장)

#### 재시도 전략

```java
@RetryableTopic(
    attempts = "1",  // 최대 재시도 1회
    dltTopicSuffix = ".dlt",
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
```

**재시도 흐름**:
1. 첫 시도 실패 → 자동 재시도 1회
2. 재시도 실패 → DLT로 전송
3. DLT Handler에서 `EmailFailLog` 저장

### 6.3 상태 전환 흐름도

```
┌──────────┐
│ CREATED  │  ← 정산 배치 완료 (매월 1일)
└────┬─────┘
     │
     │ 발송 배치 실행 (매월 15일/21일)
     ▼
┌──────────────┐
│SEND_PENDING  │  ← Kafka Producer 발행
└──────┬───────┘
       │
       │ Kafka Consumer 수신
       ▼
   ┌───────────────┐
   │ 상태 체크      │
   └───┬───────────┘
       │
       ├─> COMPLETED? → Skip (멱등성)
       │
       ├─> 금칙 시간? → IN_QUIET_HOUR
       │                │
       │                └─> 재발송 배치 (매시간)
       │                    └─> SEND_PENDING (재투입)
       │
       └─> 금칙 시간 아님 → 이메일 발송
           │
           ├─> 성공 → COMPLETED
           │
           └─> 실패 → FAILED → DLT
```

### 6.4 비동기 처리 흐름

```java
@KafkaListener(topics = TOPIC_NAME, groupId = GROUP_ID)
public void consume(BillingProducerMessageDto messageDto, Acknowledgment ack) {
    CompletableFuture.runAsync(() -> {
        try {
            billingDispatchService.process(messageDto);
        } catch (Exception e) {
            // DLT로 전송
            kafkaTemplate.send(DLT_TOPIC, messageDto);
        }
    }, emailExecutor)
    .whenComplete((result, ex) -> {
        // 항상 Ack 호출 (Zombie Consumer 방지)
        ack.acknowledge();
    });
}
```

**처리 흐름**:
1. Kafka에서 메시지 수신
2. `emailExecutor` 스레드 풀에 작업 제출
3. 비동기로 이메일 발송 처리
4. 성공/실패와 관계없이 `ack.acknowledge()` 호출
5. 오프셋 커밋으로 다음 메시지 처리 가능

### 6.5 DLT 처리 로직

```java
@DltHandler
@Transactional
public void handleDlt(
    BillingProducerMessageDto messageDto,
    Acknowledgment ack,
    @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
    @Header(KafkaHeaders.OFFSET) Long offset
) {
    // EmailFailLog 저장
    String payloadJson = objectMapper.writeValueAsString(messageDto);
    saveFailLog(publicInfoId, payloadJson, ParseStatus.SUCCESS);
    
    // Ack 호출
    ack.acknowledge();
}
```

**DLT 저장 정보**:
- `publicInfoId`: 회원 공개 정보 ID
- `payload`: 원본 메시지 JSON (암호화 상태 유지)
- `parseStatus`: 파싱 성공 여부

<br>

## 🧾 7. Billing 테이블 설계

### 7.1 테이블 스키마

```sql
CREATE TABLE billing_p (
    id BIGINT NOT NULL,
    billing_date TIMESTAMP NOT NULL,
    public_info_id VARCHAR(255),
    usage_id BIGINT,
    billing_fee INTEGER,
    status VARCHAR(20),
    send_status VARCHAR(20),
    paid_date TIMESTAMP,
    billing_details JSONB,
    
    PRIMARY KEY (id, billing_date)
) PARTITION BY RANGE (billing_date);
```

### 7.2 인덱스 전략

```sql
-- 복합키 인덱스 (자동 생성)
CREATE INDEX idx_billing_id_date ON billing_p (id, billing_date);

-- 발송 상태 조회용 인덱스
CREATE INDEX idx_billing_send_status ON billing_p (send_status, billing_date);

-- 회원별 조회용 인덱스
CREATE INDEX idx_billing_public_info ON billing_p (public_info_id, billing_date);
```

### 7.3 상태별 데이터 분포 예상

| 상태 | 설명 | 데이터 분포 |
|---|---|---|
| `CREATED` | 정산 완료 | 매월 1일 ~ 14일/20일 |
| `SEND_PENDING` | 발송 대기 | 발송 배치 실행 중 |
| `IN_QUIET_HOUR` | 금칙 시간 대기 | 금칙 시간 설정 사용자 |
| `COMPLETED` | 발송 완료 | 대부분의 최종 상태 |
| `FAILED` | 발송 실패 | 소수 (에러 로그로 관리) |

### 7.4 데이터 일관성 보장

**트랜잭션 전략**:
- 배치 작업: 청크 단위 트랜잭션 (1,000건)
- 상태 업데이트: `REQUIRES_NEW` 전파로 독립 트랜잭션
- 외부 API 호출: 트랜잭션 외부에서 처리

**멱등성 보장**:
- 상태 체크를 통한 중복 발송 방지
- Kafka 메시지 키 기반 파티션 라우팅
- DB 상태를 Single Source of Truth로 사용

<br>

## 🛠 8. 기술 스택

### Environment

<p>
  <img src="https://img.shields.io/badge/git-%23F05033.svg?style=for-the-badge&logo=git&logoColor=white">
  <img src="https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white">
  <a href="https://shadow-lychee-a03.notion.site/2e0ab48eeb4b8001bb42f7c35e987cd8?source=copy_link">
    <img src="https://img.shields.io/badge/Notion-%23000000.svg?style=for-the-badge&logo=notion&logoColor=white">
  </a>
  <a href="https://jack36140.atlassian.net/jira/software/projects/MOONU/summary" target="_blank">
    <img src="https://img.shields.io/badge/jira-0052CC?style=for-the-badge&logo=jira&logoColor=white">
  </a>
</p>

### Development

<p>
  <img src="https://img.shields.io/badge/java-%23ED8B00.svg?style=for-the-badge&logo=openjdk&logoColor=white">
  <img src="https://img.shields.io/badge/spring%20boot-%236DB33F.svg?style=for-the-badge&logo=springboot&logoColor=white">
  <img src="https://img.shields.io/badge/postgresql-%234479A1.svg?style=for-the-badge&logo=postgresql&logoColor=white">
  <img src="https://img.shields.io/badge/apache%20kafka-%23231F20.svg?style=for-the-badge&logo=apache-kafka&logoColor=white">
</p>

**주요 라이브러리**:
- **Spring Boot**: 3.4.1
- **Spring Batch**: 대용량 배치 처리
- **Spring Kafka**: 메시지 큐 통합
- **PostgreSQL**: 18 (Range Partitioning 지원)
- **Kafka**: 3.5 (KRaft 모드)
- **Handlebars**: 4.3.1 (이메일 템플릿)
- **BouncyCastle**: 1.78.1 (PGP 암호화)
- **Micrometer + Prometheus**: 메트릭 수집

노션 링크에 저희 팀의 문서화가 자세히 되어있습니다.

---

## 🚀 9. 시작 가이드

### 요구 사항 (Prerequisites)

* Java 17+
* Spring Boot 3.4.1
* PostgreSQL 18
* Kafka 3.5 (Confluent 7.5.0)
* Docker & Docker Compose

### 환경 설정

#### 1. PostgreSQL 설정

```bash
# PostgreSQL 컨테이너 실행
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=moono \
  -p 5432:5432 \
  postgres:18
```

#### 2. Kafka 실행

```bash
# 프로젝트 루트에서 실행
docker-compose up -d
```

이 명령어는 다음 서비스를 시작합니다:
- **Kafka**: 포트 9092 (KRaft 모드)
- **Kafka UI**: 포트 8989 (웹 인터페이스)
- **Kafka Exporter**: 포트 9308 (메트릭 수집)

#### 3. 애플리케이션 설정

`src/main/resources/application.yml` 파일을 생성:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/moono
    username: postgres
    password: password
  
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: cg-billing-email-send
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

# 프로파일 설정
spring.profiles.active: billing-batch,producer,consumer
```

#### 4. 빌드 및 실행

```bash
# 빌드
./gradlew build

# 실행
./gradlew bootRun
```

---

## 👥 10. 팀원 소개

| 🐙 무너 1호 | 🐙 무너 2호 | 🐙 무너 3호 | 🐙 무너 4호 | 🐙 무너 5호 | 🐙 무너 6호 |
| :---: | :---: | :---: | :---: | :---: | :---: |
| <img src="https://github.com/hoonseok0710.png" width="100"/> | <img src="https://github.com/ashmin-yoon.png" width="100"/> | <img src="https://github.com/yubin012.png" width="100"/> | <img src="https://github.com/LeeGyeongYoon-BE.png" width="100"/> | <img src="https://github.com/dnwldla.png" width="100"/> | <img src="https://github.com/rettooo.png" width="100"/> |
| **최훈석 (팀장)** | **윤재민** | **박유빈** | **이경윤** | **임지우** | **최하영** |
| [@hoonseok0710](https://github.com/hoonseok0710) | [@ashmin-yoon](https://github.com/ashmin-yoon) | [@yubin012](https://github.com/yubin012) | [@LeeGyeongYoon-BE](https://github.com/LeeGyeongYoon-BE) | [@dnwldla](https://github.com/dnwldla) | [@rettooo](https://github.com/rettooo) |
| Backend / Infra | Backend / Infra | Backend / Kafka | Backend / DB | Backend / DB | Backend / Kafka |

### 🐙 역할

* **최훈석**: "모니터링 툴 작업, 이벤트 알림"
* **윤재민**: "배치 작업, 약정 및 생일 이벤트 할인"
* **박유빈**: "발송 배치 작업 후 Kafka 처리"
* **이경윤**: "부가 서비스 등급제 할인, 장기 고객 할인"
* **임지우**: "정산작업처리"
* **최하영**: "대용량 데이터 카프카 처리"

---


