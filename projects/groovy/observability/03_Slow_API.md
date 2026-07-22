# 실습 3. Slow API 응답 지연 모니터링

## 실습 목적

Spring Boot Backend에 의도적으로 응답 지연을 발생시키는 API를 추가하여 응답 시간이 증가하는 상황을 만든다.

이후 Grafana와 Prometheus를 통해 다음 항목을 관찰한다.

- Response Time 변화
- HTTP Requests 변화
- JVM Thread 변화
- CPU 및 Memory 변화
- Prometheus Target 상태

이를 통해 서비스는 정상적으로 동작하지만 응답 속도가 저하되는 상황과 실제 장애(Target DOWN)와의 차이를 이해한다.

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

# 1. 테스트 API 구현

응답을 의도적으로 지연시키기 위해 다음 API를 추가하였다.

```java
@RestController
@RequestMapping("/api/test")
public class SlowTestController {

    @GetMapping("/slow")
    public String slow() throws InterruptedException {
        Thread.sleep(5000);
        return "slow response completed";
    }
}
```

5초 동안 Thread를 대기시킨 후 응답하도록 구현하였다.

---

# 2. 보안 설정 수정

테스트 API는 인증 없이 호출할 수 있도록 `SecurityConfig`의 GET 허용 목록에 추가하였다.

```java
private static final String[] PERMIT_ALL_GET_PATTERNS = {
    "/api/studies",
    "/api/studies/{studyId}",
    "/api/tags",
    "/api/test/slow"
};
```

---

# 3. Baseline 확인

부하를 발생시키기 전 Grafana Dashboard를 확인하였다.

관찰 결과

- Response Time: 약 2~3ms
- HTTP Requests: 평상시 수준
- CPU: 낮은 사용률
- JVM Live Threads: 약 18개
- Heap / Non-Heap Memory: 안정적인 상태

Prometheus Target도 정상적으로 `UP` 상태를 유지하였다.

---

# 4. Slow API 호출

단일 요청 확인

```bash
time curl http://localhost:8080/api/test/slow
```

실행 결과 약 5초 후 응답이 반환되는 것을 확인하였다.

이후 동시 요청을 발생시켰다.

```bash
seq 20 | xargs -P10 -I{} curl -s http://localhost:8080/api/test/slow > /dev/null
```

20개의 요청을 생성하고 동시에 최대 10개의 요청을 수행하여 응답 지연 상황을 만들었다.

---

# 5. Grafana 관찰 결과

부하가 발생하는 동안 다음과 같은 변화를 확인하였다.

## Response Time

평상시 약 2~3ms 수준이던 응답 시간이 약 **4초**까지 증가하였다.

## HTTP Requests

동시 요청으로 인해 HTTP Request 수가 크게 증가하였다.

## JVM Live Threads

동시 요청을 처리하면서 Live Thread 수가 약 18개에서 25개 수준까지 증가하였다.

## CPU

CPU 사용률은 약간 증가하였으나 큰 변화는 발생하지 않았다.

## Heap Memory

Heap Memory는 평상시 범위 내에서 변동하였으며 급격한 증가나 메모리 부족 현상은 발생하지 않았다.

## Non-Heap Memory

큰 변화 없이 안정적으로 유지되었다.

## GC Count

GC Count 역시 큰 변화 없이 정상적으로 유지되었다.

---

# 6. Prometheus Target 확인

Prometheus Target은 실습 동안 계속 `UP` 상태를 유지하였다.

```text
Target

UP
 ↓
UP
```

즉,

Backend는 정상적으로 실행되고 있으며 Prometheus 역시 Metric을 정상적으로 수집하고 있음을 확인하였다.

이는 Target DOWN 장애와 가장 큰 차이점이다.

---

# 7. 부하 종료 후 복구

부하 종료 후 Grafana를 확인하였다.

관찰 결과

- Response Time이 약 2.37ms 수준으로 복구
- HTTP Requests가 평상시 수준으로 감소
- JVM Thread 수 정상화
- CPU 안정화

서비스 중단 없이 정상 상태로 복귀하는 것을 확인하였다.

---

# 8. Slow API와 Target DOWN 비교

| 항목 | Slow API | Target DOWN |
|---|---|---|
| Backend 실행 | O | X |
| Prometheus Target | UP | DOWN |
| Metric 수집 | 정상 | 실패 |
| Grafana | Response Time 증가 | Gap 발생 |
| 장애 유형 | 성능 저하 | 서비스 장애 |

---

# 실습 결론

Slow API를 통해 의도적으로 응답 시간을 증가시키자 Grafana에서는 Response Time이 약 4초까지 증가하는 것을 확인하였다.

동시에 HTTP Requests와 JVM Live Threads도 증가하였지만 Prometheus Target은 계속 `UP` 상태를 유지하였다.

즉, 서비스는 정상적으로 실행되고 있으며 Metric도 정상적으로 수집되고 있었지만 응답 속도만 저하된 상황이었다.

이를 통해 다음 내용을 확인하였다.

- Response Time 증가는 서비스 성능 저하를 의미한다.
- Target이 UP이라면 Metric 수집은 정상적으로 이루어지고 있다.
- Target DOWN과 Slow API는 서로 다른 장애 유형이다.
- 장애 판단 시 Target 상태와 Response Time을 함께 확인해야 한다.
- 부하 종료 후에는 Response Time과 서비스 상태가 정상적으로 복구되는 것을 확인하였다.