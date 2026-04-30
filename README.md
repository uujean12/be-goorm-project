# Heartbit

> LMAX Disruptor 기반 Lock-free 아키텍처로 동시성 병목을 해결하고 **체결 엔진 처리 안정성 99%**를 달성한 가상자산 체결 시스템

대규모 트래픽 환경에서 0.1s 이내의 응답 속도를 보장하는 가상자산 체결 엔진입니다.
자바 기반 환경의 동시성 병목을 해결하고, 실제 금융권 수준의 가용성과 처리량 확보에 집중했습니다.

<br>

## 📋 프로젝트 개요

Heartbit은 Lock-free Ring Buffer 구조를 기반으로 설계된 초고속 매수/매도 주문 체결 엔진입니다.

초기 PriorityQueue 기반 구조에서 발생한 동시성 병목과 에러율 60% 문제를 LMAX Disruptor 도입으로 해결하였으며, OS 커널 파라미터 튜닝과 AWS 인스턴스 최적화를 통해 최종적으로 단일 노드 기준 700,000 TPS를 달성했습니다.

### 성능 목표

| 지표 | 목표 |
|------|------|
| 처리량 (TPS) | 10,000 ~ 20,000 TPS (주문부터 체결까지 비즈니스 트랜잭션 완료 기준) |
| 응답 속도 | 평균 100ms 이내 |
| 최대 스루풋 | 50,000 동시 처리 대역폭 확보 |

### 주요 특징

- **LMAX Disruptor**: Lock-free Ring Buffer 기반 초고속 매수/매도 주문 체결
- **종목 기반 샤딩**: 거래량 상위 7개 종목은 서버 1, 나머지 193개 종목은 서버 2로 분산 처리
- **Redis Pub/Sub**: 분산 서버 간 실시간 데이터 동기화로 데이터 정합성 확보
- **비동기 데이터 파이프라인**: 주문 수집부터 체결 결과 저장까지의 Full-flow 설계
- **OS 커널 튜닝**: Linux `somaxconn` 파라미터 최적화로 TCP 백로그 한계 해소
- **AWS 인프라 최적화**: 목적별 인스턴스 선택, Graviton3 및 리전 마이그레이션으로 레이턴시 개선
- **실시간 모니터링**: k6, Prometheus, Grafana 기반 성능 측정 및 시각화

<br>

## 🏗️ 프로젝트 구조

```
Heartbit
├── matching-engine           # 체결 엔진 코어 (담당)
│   ├── Disruptor 기반 매칭 로직 (순수 자바 객체)
│   └── Ring Buffer 이벤트 처리 (bufferSize: 4096)
├── order-management          # 주문 수집 및 관리 (담당)
│   └── 매수/매도 주문 파이프라인 (비동기 즉시 응답)
├── asset-management          # 자산 관리
│   └── 체결 결과 비동기 저장 및 실시간 자산 업데이트
├── sync                      # 분산 서버 데이터 동기화 (일부 담당)
│   └── Redis Pub/Sub 기반 서버 간 실시간 동기화
└── infra                     # 인프라 설정
    ├── AWS (EC2, RDS, Route 53, ALB)
    ├── Nginx (Reverse Proxy)
    ├── Docker
    └── GitHub Actions
```

<br>

## 🔧 핵심 모듈 설명

### 1. Matching Engine (체결 엔진 코어) - 담당
LMAX Disruptor 기반의 Lock-free 매칭 로직입니다.

- **순수 자바 객체**로 구현하여 복잡한 프레임워크 의존 없이 경량화
- `while` 루프로 호가창에 잔량이 남아있는 한 실시간으로 주문 매칭
- `bufferSize`를 2의 거듭제곱(4096)으로 설정하여 비트 연산 기반 인덱스 관리
- `YieldingWaitStrategy` 적용으로 불필요한 컨텍스트 스위칭 없이 즉각 이벤트 처리
- `ProducerType.SINGLE`로 생산자 스레드 고정, Lock 원천 차단

### 2. 비동기 데이터 파이프라인 - 담당
주문 수집부터 체결 결과 저장까지의 Full-flow입니다.

```
사용자 주문 요청
    → OrderService (DB 저장 + 즉시 응답 반환)
    → Event Producer (Ring Buffer에 주문 데이터 적재)
    → Matching Engine (체결 처리 - 순수 자바 객체)
    → Disruptor then() 병렬 파이프라인
        ├── DB 저장 Handler
        └── 실시간 호가창 반영 Handler
    → 사용자 자산/투자 내역 반영 + 체결 알림
```

### 3. 종목 기반 샤딩 + Redis Pub/Sub 동기화 - 일부 담당
분산 서버 환경에서의 데이터 정합성 문제를 해결하기 위해 종목 기반 샤딩과 Redis Pub/Sub을 도입했습니다.

**샤딩 구조:**

```
거래량 상위 7개 종목 (BTC, ETH 등)  →  서버 1 (고부하 전담)
나머지 193개 종목                   →  서버 2
```

**데이터 동기화 문제:**
1분 1초가 돈과 직결되는 거래소 도메인에서 서버 간 데이터 불일치는 서비스 신뢰도를 무너뜨리는 치명적인 결함입니다. 서버 1에서 자산 정보나 호가 데이터가 변경되었을 때, 서버 2에 즉시 반영되지 않으면 사용자가 접속한 서버에 따라 서로 다른 잔고를 보거나 잘못된 가격으로 거래를 시도하게 됩니다.

**Redis Pub/Sub 해결:**

```
서버 1에서 데이터 변경 발생
    → Redis Channel에 이벤트 Publish
    → Subscribe 중인 서버 2가 실시간 수신
    → 서버 2의 로컬 상태 즉시 갱신
    → 서버 간 데이터 간극 0에 수렴
```

### 4. 인프라 아키텍처

```
클라이언트 (브라우저)
    → Route 53 (도메인 → ALB 주소 매핑)
    → ALB (대상 그룹 규칙에 따라 EC2로 트래픽 균등 분산)
    → EC2 (Nginx - Reverse Proxy → Spring Boot)
        ├── 서버 1: 거래량 상위 7개 종목 전담 (c7g.xlarge - Graviton3 ARM)
        └── 서버 2: 나머지 193개 종목 전담
    → RDS PostgreSQL 16
    → Redis 7.2 (r7i.large - Memory Optimized)
```

**인스턴스 선택 전략:**

| 서버 | 인스턴스 | 선택 이유 |
|------|----------|-----------|
| 체결 엔진 서버 | c7g.xlarge (Graviton3 ARM) | CPU 연산 처리량이 많은 결과 데이터 처리에 최적화 |
| Redis 서버 | r7i.large (Memory Optimized) | 메모리 사용량이 많은 Redis 특성에 최적화 |
| DB 서버 | RDS PostgreSQL | 데이터베이스 저장 및 조회 전담 |

<br>

## 📦 기술 스택 및 의존성

| 분류 | 기술 |
|------|------|
| Language | Java 25 |
| Framework | Spring Boot, Spring Security (JWT), JPA |
| Concurrency | LMAX Disruptor (bufferSize: 4096, YieldingWaitStrategy) |
| Database | PostgreSQL 16, Redis 7.2 |
| Messaging | Redis Pub/Sub (분산 서버 간 실시간 동기화) |
| Proxy | Nginx (Reverse Proxy) |
| Infrastructure | AWS (EC2, RDS, Route 53, ALB), Docker |
| CI/CD | GitHub Actions |
| Monitoring | k6, Prometheus, Grafana |

<br>

## 📊 성능 테스트 결과

### Disruptor 도입 전/후 비교

| 지표 | AS-IS (PriorityQueue) | TO-BE (Disruptor + 인스턴스 업그레이드) | 개선 |
|------|----------------------|------------------------------------------|------|
| 최대 처리량 | 300 TPS | **500,000 TPS** | **약 1,660배 향상** |
| 성공률 | 40% | **99%** | **시스템 안정성 확보** |
| 데이터 구조 | PriorityQueue | **Ring Buffer** | **CME 에러 원천 차단** |

> 처리량 향상은 Disruptor 아키텍처 도입과 AWS 인스턴스 업그레이드가 함께 기여한 결과입니다.

### 매칭 엔진 부하 테스트 (DB 격리 환경)

| 단계 | 조건 | 결과 |
|------|------|------|
| 500 VUser | HikariCP 풀 고갈 발견 | 커넥션 풀 60으로 증설 후 해소 |
| 1,000 VUser | 성능 저하 | 배치 처리 도입으로 개선 |
| 5,000 VUser | DB I/O 한계 도달 | 성공률 89%, 평균 응답 6초~최대 1분, 초당 5,470건 누락 |
| DB 격리 후 | somaxconn 튜닝 전 | 30만 TPS 부근에서 SocketException, 성공률 95%로 감소 |
| DB 격리 후 | somaxconn 20,480으로 확장 | **단일 노드 70만 TPS, 성공률 100%** |

<br>

## 🛠️ 트러블슈팅 & 성능 최적화

### 1. 동시성 병목 해결: PriorityQueue → LMAX Disruptor

- **문제**: 300 VUser Ramp-up 테스트 시 에러율 60%, CPU 점유율 100% 기록
- **원인**: Blocking 방식의 큐로 인한 과도한 쓰레드 경합, CPU가 실제 처리 대신 컨텍스트 스위칭에 자원 낭비. PriorityQueue 동시 수정으로 ConcurrentModificationException 발생
- **해결**: Lock을 걸면 예외는 막을 수 있으나 경합이 심해지는 구조적 한계 확인 → LMAX Disruptor의 Lock-free Ring Buffer 구조로 전환
- **결과**: 500,000 TPS, 성공률 99% 달성 (Disruptor 도입 + AWS 인스턴스 업그레이드 기여)

### 2. 분산 서버 데이터 정합성 문제

- **문제**: 서버 1(상위 7개 종목)과 서버 2(나머지 193개 종목)로 샤딩 시 서버 간 자산/호가 데이터 불일치 발생
- **원인**: 서버 1에서 변경된 데이터가 서버 2에 실시간으로 반영되지 않아 사용자가 접속한 서버에 따라 다른 잔고/가격 노출
- **해결**: Redis Pub/Sub 도입으로 데이터 변경 즉시 이벤트 Publish, 구독 중인 서버가 실시간 수신하여 로컬 상태 갱신
- **결과**: 서버 간 데이터 간극 0에 수렴, 거래소 도메인에서 요구되는 데이터 정합성 확보

### 3. DB I/O 병목: 백프레셔 현상

- **문제**: 5,000 VUser 부하 시 성공률 89%, 평균 응답 지연 6초~최대 1분, 초당 5,470건 요청 누락
- **원인**: Grafana 지표상 낮은 CPU 사용량 대비 높은 I/O Wait 확인 → 매칭 엔진 연산은 정상이나 DB 디스크 쓰기 속도가 유입량을 감당 못해 백프레셔 발생
- **해결**: DB 튜닝보다 Disruptor 매칭 엔진의 한계를 먼저 확인하기 위해 DB를 격리하여 순수 체결 처리 부하 테스트 진행

### 4. OS 커널 튜닝: somaxconn 최적화

- **문제**: DB 격리 후 30만 TPS 부근에서 SocketException 발생, 성공률 95%로 감소
- **원인**: Grafana 지표상 CPU 사용량은 안정적이나 OS 수준의 소켓 에러 발생 → macOS 커널이 수만 개의 TCP 핸드셰이크를 감당 못해 Listen 백로그 대기열 포화
- **해결**: 커널 파라미터 `somaxconn`을 20,480으로 확장하여 네트워크 대기열 확장
- **결과**: 단일 노드 기준 **70만 TPS, 성공률 100%** 달성
- **한계**: 80만 TPS부터 단일 IP의 로컬 포트 고갈(포트 소진 현상) 발생 → 추가 튜닝 필요

### 5. HikariCP 커넥션 풀 고갈

- **문제**: 500 VUser 테스트 시 DB 커넥션 고갈 발생
- **해결**: HikariCP 풀 사이즈를 60으로 증설
- **결과**: 커넥션 고갈 해소

<br>

## 💡 핵심 설계 포인트

### DisruptorConfig 최적화

```java
// bufferSize: 2의 거듭제곱 설정으로 비트 연산 기반 인덱스 관리
int bufferSize = 4096;

// YieldingWaitStrategy: 불필요한 컨텍스트 스위칭 없이 즉각 이벤트 처리
WaitStrategy waitStrategy = new YieldingWaitStrategy();

// ProducerType.SINGLE: 생산자 스레드 고정으로 Lock 원천 차단
ProducerType producerType = ProducerType.SINGLE;
```

### 병렬 파이프라인

```java
// then() 구문으로 DB 저장과 웹소켓 알림을 동시에 처리
disruptor
    .handleEventsWith(matchingEngineHandler)
    .then(dbSaveHandler, webSocketHandler); // 병렬 처리
```

<br>

## 📄 담당 역할

- **주문 체결 파트 전담**: Disruptor 기반 매칭 로직 설계 및 구현 (순수 자바 객체)
- **비동기 데이터 파이프라인**: 주문 수집부터 비동기 저장까지의 Full-flow 설계
- **종목 기반 샤딩**: 거래량 상위 7개 종목(서버 1) / 나머지 193개 종목(서버 2) 분산 구조 설계 참여
- **Redis Pub/Sub 연동**: 분산 서버 간 실시간 데이터 동기화 영역 일부 담당
- **성능 분석**: k6 + Grafana 기반 병목 지점 분석 및 단계별 튜닝
- **인프라**: AWS 리전 마이그레이션 및 인스턴스 사양 최적화

<br>

## 🔍 배운 점

고성능 시스템 구축을 위해서는 애플리케이션 로직뿐 아니라 인프라, OS 커널까지 전 계층을 고려하는 통합적 튜닝이 필수적입니다.

분산 서버 환경에서는 성능 향상과 동시에 데이터 정합성이라는 본질적인 문제를 반드시 함께 해결해야 합니다. 특히 거래소처럼 1분 1초가 돈과 직결되는 도메인에서는 서버 간 데이터 불일치가 서비스 신뢰도를 무너뜨리는 치명적인 결함이 될 수 있으며, Redis Pub/Sub과 같은 실시간 동기화 메커니즘이 필수적임을 직접 경험했습니다.
