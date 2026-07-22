# 실습 2. Prometheus Target Down 장애 탐지 및 복구

## 실습 목적

Spring Boot Backend를 중단하여 Prometheus가 Metric을 수집하지 못하는 상황을 만들고,

- Grafana Dashboard의 변화
- Prometheus Target 상태
- Scrape Error Message
- Backend 복구 이후 모니터링 정상화

과정을 확인한다.

이를 통해 Metric 값이 `0`인 경우와 데이터가 존재하지 않는 `Gap`의 차이를 이해하고, 기본적인 장애 대응 절차를 학습한다.

---

## 실습 환경

| 항목 | 내용 |
|---|---|
| Backend | Spring Boot |
| Metric Endpoint | `/actuator/prometheus` |
| Metric Collector | Prometheus |
| Dashboard | Grafana |
| Container Management | Docker Compose |
| Prometheus Target | `backend:8080` |

---

## 장애 발생 명령

```bash
docker compose stop backend
```

위 명령을 통해 Spring Boot Backend 컨테이너를 중단하였다.

---

# 1. Grafana 관찰 결과

Backend 중단 후 다음 7개 패널에서 모두 데이터가 끊겼다.

- CPU
- JVM Live Threads
- Heap Memory
- Non-Heap Memory
- GC Count
- HTTP Requests
- Response Time

모든 그래프는 값이 `0`으로 내려간 것이 아니라 `Gap`으로 표시되었다.

---

## 값 0과 Gap의 차이

| 화면 상태 | 의미 |
|---|---|
| 값이 0 | Metric 수집에는 성공했으며 측정 결과가 0 |
| Gap | 해당 시간에 Metric 데이터를 수집하지 못함 |

따라서 모든 패널에서 동시에 Gap이 발생한 경우, 개별 Metric 문제가 아니라 애플리케이션 또는 Metric 수집 경로의 문제를 의심할 수 있다.

---

# 2. 초기 장애 추론

Grafana의 모든 패널에서 동시에 Gap이 발생했기 때문에 다음과 같이 추론하였다.

```text
모든 패널에서 Gap 발생
        ↓
특정 Metric만의 문제는 아님
        ↓
Prometheus가 Backend Metric을 수집하지 못하는 것으로 추정
        ↓
Prometheus Target 상태 확인
```

---

# 3. Prometheus Target 확인

Prometheus의 `Status → Target health` 화면에서 다음 상태를 확인하였다.

```text
groovy-backend

1 / 1 UP
    ↓
0 / 1 UP

State: DOWN
```

이를 통해 Grafana 자체의 문제가 아니라 Prometheus가 Backend Target을 Scrape하지 못하고 있음을 확인하였다.

---

# 4. Error Message 확인

Prometheus Target 화면에서 다음 오류를 확인하였다.

```text
Error scraping target:
Get "http://backend:8080/actuator/prometheus":
dial tcp: lookup backend on 127.0.0.11:53:
no such host
```

## 오류 해석

`no such host`는 Prometheus가 HTTP 요청을 보내기 전에 Docker DNS에서 `backend`라는 이름을 찾지 못했다는 의미이다.

Docker Compose 네트워크에서는 각 서비스가 서비스 이름으로 발견된다.

```text
Prometheus
    ↓
Docker 내부 DNS에서 backend 조회
    ↓
backend 이름을 찾지 못함
    ↓
Scrape 실패
    ↓
Target DOWN
```

이번 실습에서는 의도적으로 Backend 컨테이너를 중단했기 때문에 Backend 중단이 원인임을 확인할 수 있었다.

---

# 5. 장애 대응 흐름

실무에서는 Target DOWN만 보고 즉시 Backend 장애라고 단정하지 않는다.

다음 순서로 원인을 좁혀간다.

```text
Grafana Gap 확인
        ↓
Prometheus Target DOWN 확인
        ↓
Scrape Error Message 확인
        ↓
Docker Container 상태 확인
        ↓
Backend Log 확인
        ↓
네트워크 및 DNS 상태 확인
        ↓
원인 확정
```

이번 실습에서는 Error Message와 의도적으로 수행한 Backend 중단 작업을 통해 원인을 확정하였다.

---

# 6. Backend 복구

다음 명령으로 Backend 컨테이너를 다시 시작하였다.

```bash
docker compose start backend
```

실행 결과 Backend 컨테이너가 정상적으로 시작되었다.

---

# 7. 복구 결과 확인

## Prometheus

Backend 시작 후 Prometheus Target이 다음과 같이 변경되었다.

```text
DOWN
  ↓
UP
```

Prometheus가 `/actuator/prometheus` Endpoint를 다시 정상적으로 Scrape하는 것을 확인하였다.

## Grafana

Prometheus Scrape가 재개된 이후 Grafana의 7개 패널에서도 새로운 Metric이 다시 표시되었다.

```text
기존 데이터
     ↓
Backend 중단 구간: Gap
     ↓
Backend 복구 이후 새 데이터 표시
```

Prometheus는 수집하지 못했던 과거 데이터를 나중에 복원하지 않으므로, 장애 시간의 Gap은 그대로 남고 복구 이후 시점부터 새로운 그래프가 다시 생성된다.

---

# 8. 복구 후 Metric 상태

Backend 재시작 직후에는 JVM이 새로 시작되기 때문에 기존 값과 완전히 동일하지 않을 수 있다.

관찰 결과:

- CPU가 시작 과정에서 일시적으로 증가한 후 안정화
- Live Threads가 새 JVM 기준으로 다시 생성
- Heap Memory가 낮은 값부터 다시 증가
- Non-Heap Memory가 클래스 로딩에 따라 다시 증가
- GC Count가 새 JVM 프로세스 기준으로 다시 측정
- HTTP Requests가 새 요청부터 다시 기록
- Response Time이 정상 범위로 복귀

즉, Backend가 단순히 일시 정지되었다가 이어진 것이 아니라 JVM 프로세스가 다시 활성화되면서 Metric 시계열이 새롭게 시작되는 모습을 확인하였다.

---

# 9. 장애 대응 단계

이번 실습은 기본적인 Incident Response 과정을 포함한다.

| 단계 | 수행 내용 |
|---|---|
| Detection | Grafana에서 모든 패널의 Gap 발견 |
| Diagnosis | Prometheus Target DOWN 확인 |
| Error Analysis | `no such host` 오류 확인 |
| Root Cause | Backend 컨테이너 중단 확인 |
| Recovery | Backend 컨테이너 시작 |
| Verification | Target UP 및 Grafana 수집 재개 확인 |

---

# 실습 결론

Spring Boot Backend를 중단하자 Prometheus가 Metric Endpoint를 Scrape하지 못하면서 Target이 DOWN으로 변경되었다.

그 결과 Grafana의 모든 패널에서 값이 0이 아닌 Gap이 발생하였다.

Prometheus Error Message를 통해 Docker DNS가 `backend`라는 서비스 이름을 찾지 못한 것을 확인하였으며, Backend 컨테이너를 다시 시작한 후 Target이 UP으로 복구되고 Grafana에서 Metric 수집이 재개되는 것을 확인하였다.

이를 통해 다음 내용을 학습하였다.

- 값 0과 Gap은 서로 다른 상태이다.
- 모든 패널의 동시 Gap은 수집 경로 장애를 의심할 수 있는 신호이다.
- Grafana만으로 원인을 단정하지 않고 Prometheus Target과 Error Message를 확인해야 한다.
- Target DOWN에도 DNS, 연결 거부, Timeout, HTTP 오류 등 여러 원인이 존재한다.
- 복구 후에는 서비스 실행 여부뿐 아니라 Target UP과 Dashboard 정상화까지 검증해야 한다.