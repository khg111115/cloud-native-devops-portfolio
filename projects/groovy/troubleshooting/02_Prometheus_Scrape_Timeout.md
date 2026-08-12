# TS-002. Docker Exporter 응답 지연으로 인한 Prometheus Target DOWN

## 1. 문제 상황

부하 테스트 및 Observability 환경을 구성한 뒤 Prometheus Target 상태를 확인하였다.

확인 결과 `docker-exporter`가 `DOWN` 상태였으며 다음 오류가 발생하고 있었다.

```text
job: docker-exporter
scrapeUrl: http://docker-exporter:8080/metrics
lastError: context deadline exceeded
health: down
scrapeInterval: 15s
scrapeTimeout: 10s
```

즉, Prometheus가 Docker Exporter의 `/metrics` 엔드포인트에 접근하고 있었지만 제한 시간 내 응답을 받지 못하고 있었다.

---

## 2. 원인 분석

Docker Exporter 자체의 응답시간을 확인하기 위해 `/metrics` 요청에 걸리는 시간을 직접 측정하였다.

```bash
time curl -s http://localhost:8082/metrics > /dev/null
```

측정 결과 약 **17.2초**가 소요되었다.

![Docker Exporter slow response](./images/01_docker_exporter_slow_response.png)

당시 Prometheus의 설정은 다음과 같았다.

```yaml
scrape_interval: 15s
scrape_timeout: 10s
```

Docker Exporter가 메트릭을 생성하는 데 약 17.2초가 걸리는 반면 Prometheus는 10초까지만 응답을 기다리고 있었다.

따라서 Docker Exporter가 응답을 완료하기 전에 Prometheus의 `scrape_timeout`이 먼저 발생하면서 다음 오류와 함께 Target이 `DOWN` 상태로 전환된 것으로 판단하였다.

```text
context deadline exceeded
```

Docker Exporter 로그에서 반복적으로 확인된 `BrokenPipeError` 역시 Prometheus가 timeout으로 연결을 먼저 종료한 뒤 Exporter가 응답을 쓰려고 하면서 발생한 현상으로 판단하였다.

---

## 3. 1차 조치 - Prometheus Scrape 설정 조정

### 3.1 대응 방향

우선 Prometheus의 메트릭 수집을 정상화하기 위해 Docker Exporter의 실제 응답시간보다 충분한 `scrape_timeout`을 확보하는 방향으로 설정을 변경하기로 하였다.

Docker Exporter의 응답시간이 약 17.2초였기 때문에 기존 10초의 timeout을 25초로 증가시키고자 하였다.

### 3.2 첫 번째 설정 변경 실패

그러나 `scrape_timeout`만 25초로 증가시키는 과정에서 Prometheus가 정상적으로 시작되지 않았으며 다음 오류가 발생하였다.

```text
Error loading config
scrape timeout greater than scrape interval
```

당시 `scrape_interval`은 15초였기 때문에 timeout을 25초로 설정할 수 없었다.

Prometheus에서는 `scrape_timeout`이 `scrape_interval`보다 길어질 수 없으므로 timeout뿐만 아니라 interval도 함께 조정해야 했다.

### 3.3 Scrape Interval / Timeout 재조정

Docker Exporter의 약 17.2초 응답시간을 고려하여 해당 Job의 설정을 다음과 같이 변경하였다.

```yaml
- job_name: 'docker-exporter'
  scrape_interval: 30s
  scrape_timeout: 25s
  static_configs:
    - targets: ['docker-exporter:8080']
```

![Prometheus scrape configuration](./images/02_prometheus_scrape_timeout_config.png)

이를 통해 Docker Exporter가 응답을 완료할 수 있는 시간을 확보하면서,

```text
scrape_timeout < scrape_interval
```

조건도 만족하도록 구성하였다.

### 3.4 1차 조치 재검증

설정 수정 후 Prometheus Target 상태를 다시 확인하였다.

```bash
curl -s localhost:9090/api/v1/targets \
| grep -o '"health":"[^"]*"' \
| sort | uniq -c
```

결과:

```text
5 "health":"up"
```

![Prometheus targets recovered](./images/03_prometheus_targets_recovered.png)

전체 5개 Target이 모두 `UP` 상태로 정상화된 것을 확인하였다.

따라서 Prometheus의 메트릭 수집 실패는 우선 해결할 수 있었다.

---

## 4. 1차 조치의 한계

Prometheus의 `scrape_interval`과 `scrape_timeout`을 조정하면서 Docker Exporter Target은 다시 `UP` 상태로 정상화되었다.

하지만 이 조치는 **Prometheus가 Docker Exporter의 응답을 더 오래 기다릴 수 있도록 변경한 것**이다.

즉,

```text
Docker Exporter /metrics 응답시간
약 17.2초
```

라는 현상 자체가 개선된 것은 아니다.

현재 상태를 정리하면 다음과 같다.

```text
Docker Exporter /metrics
약 17.2초
        │
        ▼
Prometheus scrape_timeout 10s
        │
        ▼
Timeout
        │
        ▼
Target DOWN

        ↓ 1차 조치

scrape_interval 30s
scrape_timeout 25s
        │
        ▼
17.2초 응답 수집 가능
        │
        ▼
Target UP
```

따라서 1차 조치는 **모니터링 수집을 우선 정상화하기 위한 대응**으로 보고, Docker Exporter의 높은 응답시간 자체는 추가로 개선할 필요가 있다고 판단하였다.

---

## 5. 2차 조치 예정 - Docker Exporter 응답시간 최적화

다음 단계에서는 Prometheus의 대기시간을 늘리는 방식이 아니라 **Docker Exporter의 `/metrics` 응답시간 자체를 단축하는 것**을 목표로 한다.

현재 확인된 기준 응답시간은 다음과 같다.

```text
Before: 약 17.2초
After : 측정 예정
```

먼저 Docker Exporter가 메트릭을 생성하는 과정에서 어느 구간에 시간이 소요되는지 측정하여 병목 지점을 특정할 예정이다.

이후 확인된 원인에 따라 수집 방식을 개선하고 다음 항목을 비교한다.

```text
1. /metrics 응답시간
2. Prometheus Target 상태
3. 메트릭 정상 수집 여부
4. 개선 전/후 응답시간
```

최적화 이후에도 동일한 메트릭이 정상적으로 수집되는지 확인하고, 필요할 경우 Prometheus의 `scrape_interval`과 `scrape_timeout`도 다시 조정할 예정이다.

> **Status:** 2차 최적화 실험 예정

---

## 6. 현재까지의 장애 흐름

```text
docker-exporter Target DOWN
        │
        ▼
context deadline exceeded 확인
        │
        ▼
/metrics 직접 호출
        │
        ▼
응답시간 약 17.2초 측정
        │
        ▼
Prometheus 설정 확인
scrape_timeout = 10s
        │
        ▼
응답시간 > timeout
        │
        ▼
원인 특정
        │
        ▼
[1차 조치]
scrape_timeout 증가 시도
        │
        ▼
scrape timeout greater than scrape interval
        │
        ▼
interval 30s / timeout 25s로 조정
        │
        ▼
Prometheus Targets 5/5 UP
        │
        ▼
모니터링 수집 정상화
        │
        ▼
그러나 /metrics는 여전히 약 17.2초
        │
        ▼
[2차 조치 예정]
Docker Exporter 응답시간 최적화
```

---

## 7. 현재까지의 배운 점

이번 문제를 통해 Prometheus Target이 `DOWN`이라고 해서 반드시 Exporter 프로세스 자체가 중단된 것은 아니라는 점을 확인하였다.

Exporter가 실행 중이더라도 `/metrics` 응답시간이 Prometheus의 `scrape_timeout`을 초과하면 수집 실패로 인해 Target이 `DOWN` 상태가 될 수 있다.

또한 단순히 timeout 값만 증가시키는 것이 아니라 `scrape_timeout`과 `scrape_interval`의 관계를 함께 고려해야 한다는 점을 확인하였다.

장애 분석 과정에서는 다음 순서로 원인을 좁혀가는 것이 효과적이었다.

```text
Target 상태 확인
→ lastError 확인
→ Exporter 로그 확인
→ /metrics 직접 호출
→ 실제 응답시간 측정
→ Prometheus scrape 설정 확인
→ 1차 설정 조정
→ Target 상태 재검증
```

다만 **Target을 다시 `UP` 상태로 만드는 것과 응답 지연의 원인을 제거하는 것은 별개의 문제**라는 점도 확인하였다.

현재 1차 조치를 통해 모니터링 수집은 정상화하였으며, 다음 단계에서는 Docker Exporter의 약 17.2초 응답 지연 원인을 분석하여 응답시간 자체를 줄이는 방향으로 추가 개선을 진행할 예정이다.
