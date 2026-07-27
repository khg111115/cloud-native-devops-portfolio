# 05. MySQL Load Test

## 실습 목표

Spring Boot 애플리케이션을 수정하지 않고 Python을 이용하여 MySQL에 직접 부하를 발생시킨다.

이를 통해 Grafana에서 MySQL 관련 메트릭의 변화를 관찰하고,
DB 병목이 API 응답 성능에 어떤 영향을 미치는지 확인한다.

---

# 실습 환경

```text
Python
   │
   ▼
 MySQL
   │
mysqld-exporter
   │
Prometheus
   │
Grafana
```

추가 실험에서는 API 부하를 함께 발생시켜 Spring Boot와 HikariCP의 동작도 함께 확인하였다.

---

# 실습 전 상태 (Baseline)

실험 시작 전 Grafana의 기본 상태이다.

![Grafana Dashboard - Baseline](assets/db_dashboard_normal.png)

### 확인 내용

- MySQL UP
- Connections 약 11
- Threads Running 2
- Slow Queries 0
- API p95 약 16ms

모든 메트릭이 안정적인 기준값(Baseline)을 유지하는 것을 확인하였다.

---

# 실습 1. MySQL 연결 확인

Python에서 `mysql-connector-python`을 이용하여 MySQL 연결을 확인하였다.

확인 내용

- MySQL 연결
- Database 조회
- Table 조회
- users 데이터 조회

![MySQL Connection Test](assets/db_connection_test.png)

### 관찰 결과

- Connections 증가
- Threads Connected 증가
- Questions 증가

Python에서 MySQL에 직접 접근하여 Query가 정상적으로 수행되는 것을 확인하였다.

---

# 실습 2. Connection Storm

동시에 다수의 Connection을 생성하여 Connection Storm을 발생시켰다.

![Connection Storm](assets/db_connection_storm.png)

### 관찰 결과

- Connections 증가
- Threads Connected 증가
- Questions 약 45K
- Select Scan 증가

### 분석

Connection만 대량으로 생성하였기 때문에

- Threads Running 변화는 상대적으로 적었고
- Slow Query는 발생하지 않았다.

또한 Python이 MySQL에 직접 연결하기 때문에 Spring Boot(HikariCP)는 영향을 받지 않았다.

---

# 실습 3. Long Query Storm

동시에 다수의 Long Query를 실행하였다.

```sql
SELECT SLEEP(30);
```

![Long Query Peak](assets/db_long_query_peak.png)

### 관찰 결과

- Connections 증가
- Threads Connected 증가
- Threads Running 증가
- Slow Queries 증가

### 분석

Long Query가 장시간 실행되면서 MySQL 내부 Thread Activity가 증가하였다.

하지만 Python이 직접 MySQL에 연결하여 부하를 발생시켰기 때문에
Spring Boot와 HikariCP에는 거의 영향을 주지 않았으며,
API 응답시간도 정상 수준을 유지하였다.

---

# 실습 4. API Load + DB Load

Long Query를 실행한 상태에서 동시에 API 부하를 발생시켰다.

![API + DB Load](assets/db_api_load_hikari_exhaustion.png)

### 관찰 결과

- API p95 : 약 1.43s
- Hikari Active : 30
- Hikari Pending : 112
- Hikari Usage : 100%
- Questions 약 51K

### 분석

Connection Pool이 모두 사용되면서(Hikari Pool Exhaustion)

새로운 요청은 Pending 상태로 대기하게 되었고,

API 응답시간도 크게 증가하였다.

DB 병목이 Spring Boot 성능에도 영향을 주는 것을 확인하였다.

---

# 트러블 슈팅

## Python Exit Code 139

실험 중 Python이

```
Exit Code 139
Segmentation Fault
```

오류와 함께 종료되었다.

### 원인

mysql-connector-python의 C Extension 사용 중 충돌이 발생한 것으로 추정된다.

### 해결

Connection 생성 시

```python
use_pure=True
```

옵션을 적용하여
순수 Python Connector를 사용하였다.

이후 정상적으로 모든 부하 테스트를 수행할 수 있었다.

---

# Grafana에서 확인한 주요 메트릭

| Metric | 의미 |
|---------|------|
| API p95 | API 응답시간 |
| Current QPS | 초당 처리 요청 수 |
| Connections | 현재 MySQL 연결 수 |
| Threads Connected | 연결된 Thread 수 |
| Threads Running | 실행 중인 Thread 수 |
| Questions | 처리된 Query 수 |
| Slow Queries | 느린 Query 수 |
| Hikari Active | 사용 중인 Connection |
| Hikari Pending | Connection 대기 수 |
| Hikari Usage | Connection Pool 사용률 |

---

# 이번 실습에서 배운 점

- Python을 이용하면 Spring Boot를 수정하지 않고도 MySQL 부하를 재현할 수 있다.
- Connection 수 증가와 실제 Query 실행은 서로 다른 메트릭으로 관찰된다.
- Long Query는 Threads Running과 Slow Queries 증가에 직접적인 영향을 준다.
- API 부하와 DB 부하를 동시에 발생시키면 Hikari Connection Pool이 포화될 수 있다.
- 하나의 Metric보다 여러 Metric을 함께 분석해야 장애 원인을 정확하게 파악할 수 있다.

---

# 실습 결론

Python을 이용하여

- Connection Storm
- Long Query
- API Load

를 조합하여 MySQL 병목 상황을 재현하였다.

이를 통해

- Connections 증가
- Threads Running 증가
- Slow Queries 증가
- API p95 증가
- Hikari Pool 포화

등의 변화를 Grafana에서 직접 확인할 수 있었다.

Spring Boot를 수정하지 않고도 데이터베이스 장애 상황을 재현하고,
모니터링 지표를 분석하는 방법을 학습하였다.