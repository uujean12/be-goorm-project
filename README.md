# Heartbit

> LMAX Disruptor 기반 Lock-free 아키텍처로 동시성 병목을 해결하고 체결 엔진 처리 안정성 99%를 달성한 가상자산 모의 투자 거래 플랫폼

<br>

## 📋 프로젝트 개요

Heartbit은 실시간 거래 데이터를 활용한 모의 투자 거래 플랫폼입니다.

사용자는 실제 거래소와 동일한 환경에서 매수/매도 주문을 경험할 수 있으며, 캔들 차트 시각화, 종목별 채팅 커뮤니티, AI 투자 분석 등 다양한 기능을 통해 전략적인 투자 의사결정을 지원합니다.

<br>

<img width="1453" height="833" alt="Group 19" src="https://github.com/user-attachments/assets/2550d25e-6f1f-448f-b93d-2e60cc7ea0df" />

<br>

### 주요 서비스

- **주문**: 원하는 종목을 매수/매도하고 체결 결과를 실시간 알림으로 확인
- **캔들 차트**: 체결된 거래 데이터를 캔들 차트로 시각화하여 투자 상황을 한눈에 파악
- **종목별 채팅**: 사용자 간 실시간 소통 및 투자 정보 교환 커뮤니티
- **AI 투자 분석**: 유명 종목에 대한 종합적인 AI 분석 결과 제공

### 성능 목표

| 지표 | 목표 |
|------|------|
| 처리량 (TPS) | 10,000 ~ 20,000 TPS (주문부터 체결까지 비즈니스 트랜잭션 완료 기준) |
| 응답 속도 | 평균 100ms 이내 |
| 최대 스루풋 | 50,000 동시 처리 대역폭 확보 |

<br>

## 🏗️ 시스템 아키텍처

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

## 📦 기술 스택

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

## 👩‍💻 본인 담당 역할

> **주문 데이터 관리 / 체결 엔진 / 서버 배포 관리**

### 1. 체결 엔진 설계 및 구현

PriorityQueue 기반 구조에서 발생한 동시성 병목과 에러율 60% 문제를 LMAX Disruptor 도입으로 해결했습니다.

**Disruptor 파이프라인 구조:**

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

**핵심 설계 포인트:**
- `bufferSize` 4096 (2의 거듭제곱)으로 비트 연산 기반 인덱스 관리
- `YieldingWaitStrategy` 적용으로 불필요한 컨텍스트 스위칭 제거
- `ProducerType.SINGLE`로 생산자 스레드 고정, Lock 원천 차단
- 순수 자바 객체로 구현하여 프레임워크 의존 없이 경량화

### 2. 주문 데이터 관리 및 종목 기반 샤딩

분산 서버 환경에서의 데이터 정합성 문제를 해결하기 위해 종목 기반 샤딩을 설계하고 Redis Pub/Sub 연동을 담당했습니다.

**샤딩 구조:**

```
거래량 상위 7개 종목 (BTC, ETH 등)  →  서버 1 (고부하 전담)
나머지 약 200개 종목                   →  서버 2
```

**데이터 동기화:**
1분 1초가 돈과 직결되는 거래소 도메인에서 서버 간 데이터 불일치는 치명적인 결함입니다. Redis Pub/Sub을 활용하여 데이터 변경 즉시 이벤트를 Publish하고 구독 중인 서버가 실시간으로 수신하여 로컬 상태를 갱신하도록 설계했습니다.

```
서버 1에서 데이터 변경 발생
    → Redis Channel에 이벤트 Publish
    → Subscribe 중인 서버 2가 실시간 수신
    → 서버 2의 로컬 상태 즉시 갱신
    → 서버 간 데이터 간극 0에 수렴
```

### 3. 서버 배포 관리

- **AWS 인프라 구성**: Route 53 → ALB → EC2(Nginx → Spring Boot) → RDS/Redis 구조 구성
- **목적별 인스턴스 선택**: 연산 집약적 체결 엔진에 Graviton3(c7g.xlarge), Redis에 Memory Optimized(r7i.large) 적용
- **리전 마이그레이션**: 시드니 → 서울 리전 이동으로 네트워크 레이턴시 개선
- **Docker 컨테이너화**: 팀 전체 동일 개발 환경 확보
- **GitHub Actions CI/CD**: 자동 빌드 및 배포 파이프라인 구성

<br>

## 📊 성능 테스트 결과

### Disruptor 도입 전/후 비교

| 지표 | AS-IS (PriorityQueue) | TO-BE (Disruptor + 인스턴스 업그레이드) | 개선 |
|------|----------------------|------------------------------------------|------|
| 최대 처리량 | 300 TPS | **500,000 TPS** | 대폭 향상 |
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

## 🛠️ 트러블슈팅

### 1. 동시성 병목 해결: PriorityQueue → LMAX Disruptor

- **문제**: 300 VUser Ramp-up 테스트 시 에러율 60%, CPU 점유율 100% 기록
- **원인**: Blocking 방식의 큐로 인한 과도한 쓰레드 경합. PriorityQueue 동시 수정으로 ConcurrentModificationException 발생
- **해결**: Lock을 걸면 예외는 막을 수 있으나 경합이 심해지는 구조적 한계 확인 → LMAX Disruptor의 Lock-free Ring Buffer 구조로 전환
- **결과**: 성공률 99% 달성

### 2. 분산 서버 데이터 정합성 문제

- **문제**: 샤딩 구조에서 서버 간 자산/호가 데이터 불일치 발생
- **원인**: 서버 1의 데이터 변경이 서버 2에 실시간으로 반영되지 않아 사용자가 접속한 서버에 따라 다른 잔고/가격 노출
- **해결**: Redis Pub/Sub 도입으로 데이터 변경 즉시 이벤트 Publish, 구독 중인 서버가 실시간 수신하여 로컬 상태 갱신
- **결과**: 서버 간 데이터 간극 0에 수렴

### 3. OS 커널 튜닝: somaxconn 최적화

- **문제**: DB 격리 후 30만 TPS 부근에서 SocketException 발생, 성공률 95%로 감소
- **원인**: macOS 커널이 수만 개의 TCP 핸드셰이크를 감당 못해 Listen 백로그 대기열 포화
- **해결**: 커널 파라미터 `somaxconn`을 20,480으로 확장
- **결과**: 단일 노드 기준 **70만 TPS, 성공률 100%** 달성
- **한계**: 80만 TPS부터 단일 IP 포트 고갈 현상 발생 → 추가 튜닝 필요

### 4. DB I/O 병목: 백프레셔 현상

- **문제**: 5,000 VUser 부하 시 성공률 89%, 평균 응답 6초~최대 1분, 초당 5,470건 누락
- **원인**: 낮은 CPU 사용량 대비 높은 I/O Wait → DB 디스크 쓰기 속도가 유입량을 감당 못해 백프레셔 발생
- **해결**: DB 격리 후 순수 체결 처리 부하 테스트로 엔진 한계 검증

### 5. HikariCP 커넥션 풀 고갈

- **문제**: 500 VUser 테스트 시 DB 커넥션 고갈 발생
- **해결**: HikariCP 풀 사이즈를 60으로 증설
- **결과**: 커넥션 고갈 해소

<br>

## 🔍 배운 점

고성능 시스템 구축을 위해서는 애플리케이션 로직뿐 아니라 인프라, OS 커널까지 전 계층을 고려하는 통합적 튜닝이 필수적입니다.

분산 서버 환경에서는 성능 향상과 동시에 데이터 정합성이라는 본질적인 문제를 반드시 함께 해결해야 하며, 거래소처럼 1분 1초가 돈과 직결되는 도메인에서는 Redis Pub/Sub과 같은 실시간 동기화 메커니즘이 필수적임을 직접 경험했습니다.
