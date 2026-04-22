# 🚀 Heartbit
> **LMAX Disruptor와 커널 튜닝을 통해 단일 엔진 700,000 TPS를 달성한 초고속 체결 시스템**

---

## 1. 프로젝트 소개
- **설명**: 대규모 트래픽 환경에서 **0.1s 이내의 응답 속도**를 보장하는 가상자산 체결 엔진입니다.
- **기획 의도**: 자바 기반 환경의 동시성 병목을 해결하고, 실제 금융권 수준의 가용성과 처리량을 확보하는 데 집중했습니다.

## 2. Tech Stack
| Category | Tech Stack |
| :--- | :--- |
| **Backend** | Java 17, Spring Boot, JPA, Security (JWT), **LMAX Disruptor** |
| **Database** | **PostgreSQL 16**, Redis 7.2 |
| **Infra** | AWS (EC2, RDS, Route 53, ALB), Docker, GitHub Actions |
| **Monitoring** | k6, Prometheus, Grafana |

## 3. 주요 기능
* **Matching Engine**: Lock-free 기반 초고속 매수/매도 주문 체결
* **Asset Management**: 체결 결과 비동기 저장 및 실시간 자산 업데이트
* **Infra Optimization**: AWS Graviton3 인스턴스 및 리전 최적화

## 4. 담당 역할 (Key Contributions)
* **체결 엔진 코어**: Disruptor 기반 매칭 로직 설계 및 구현 (유진 님 담당)
* **데이터 파이프라인**: 주문 수집부터 비동기 저장까지의 Full-flow 설계
* **인프라 총괄**: AWS 리전 마이그레이션 및 인스턴스 사양 최적화

---

## 5. 트러블 슈팅 & 성능 최적화 (Critical Issues)

### ① 동시성 병목 해결: PriorityQueue → LMAX Disruptor
- **문제**: 초기 테스트 시 `ConcurrentModificationException` 발생 및 에러율 60% 기록.
- **원인**: Blocking 방식의 큐 사용으로 인한 쓰레드 경합 및 자원 낭비.
- **해결**: **Lock-free Ring Buffer** 구조인 Disruptor 도입.
- **결과**: **최대 처리량 약 1,666% 향상 (300건 → 500,000건)** 달성.

### ② OS 커널 튜닝: somaxconn 최적화
- **문제**: 30만 TPS 도달 후 네트워크 Backlog 한계로 인한 성능 정체.
- **해결**: Linux 커널 파라미터 `somaxconn` 조정을 통해 동시 연결 수용량 확장.
- **최종 성과**: 단일 엔진 기준 **700,000 TPS** 돌파.

---

## 6. 성능 테스트 결과 (Load Test)

| 지표 (Metric) | AS-IS (Traditional) | TO-BE (Disruptor) | 결과 (Result) |
| :--- | :--- | :--- | :--- |
| **최대 처리량** | 300 TPS | **700,000 TPS** | **약 2,333배 향상** |
| **성공률** | 40.86% | **99.9%** | **시스템 안정성 확보** |
| **데이터 구조** | PriorityQueue | **Ring Buffer** | **CME 에러 원천 차단** |

---

## 7. 인프라 아키텍처 및 전략

### [인프라 구조도]
<img width="807" height="485" alt="스크린샷 2026-04-22 오후 6 29 02" src="https://github.com/user-attachments/assets/fe5460db-3004-45e6-9782-f484ce761873" />

* **Core Engine (c7g.xlarge)**: ARM 기반 Graviton3 인스턴스로 가성비 및 연산 성능 극대화.
* **Memory Node (r7i.large)**: Redis 최적화를 위한 Memory-Optimized 인스턴스 사용.
* **Region Migration**: 시드니 → 서울 리전 이동으로 **네트워크 레이턴시 80% 개선**.
