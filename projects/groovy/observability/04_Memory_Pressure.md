# 실습 4. Memory Pressure 모니터링

## 실습 목적

Spring Boot Backend에서 의도적으로 Heap Memory를 점유하는 API를 구현하여 메모리 사용량 증가 상황을 재현하였다.

Grafana와 Prometheus를 이용하여 다음 항목을 관찰하였다.

- Heap Memory 변화
- Non-Heap Memory 변화
- CPU 사용률
- JVM Live Threads
- GC Count
- HTTP Requests
- Response Time

또한 실험 종료 후 테스트 코드를 제거하고 Backend를 재배포하여 Heap Memory가 정상 상태로 복구되는 과정까지 확인하였다.

---

## 실습 환경

| 항목 | 내용 |
|---|---|
| Backend | Spring Boot |
| Metric Endpoint | `/actuator/prometheus` |
| Metric Collector | Prometheus |
| Dashboard | Grafana |
| Container | Docker Compose |
| JVM | Java 21 |

---

# 1. 테스트 API 구현

메모리를 지속적으로 점유하기 위해 다음 Controller를 추가하였다.

```java
@RestController
@RequestMapping("/api/test")
public class MemoryTestController {

    private static final List<byte[]> MEMORY_HOLDER = new ArrayList<>();

    @GetMapping("/memory")
    public String consumeMemory() {
        MEMORY_HOLDER.add(new byte[10 * 1024 * 1024]);

        return "allocated memory : "
                + (MEMORY_HOLDER.size() * 10)
                + " MB";
    }
}
```

호출될 때마다 약 10MB의 Heap Memory를 할당하며, 정적 List가 참조를 유지하므로 GC가 즉시 회수하지 못하는 구조를 만들었다.

---

# 2. Security 설정

테스트 API 호출을 위해 SecurityConfig에 GET 허용 경로를 추가하였다.

```java
"/api/test/memory"
```

---

# 3. Baseline 확인

부하를 발생시키기 전 Grafana Dashboard를 확인하였다.

관찰 결과

- Heap Memory : 약 40~100MB 범위
- CPU : 낮은 사용률
- JVM Live Threads : 약 20~22개
- Response Time : 약 3~4ms
- HTTP Requests : 평상시 수준

Prometheus Target도 정상적으로 UP 상태를 유지하였다.

---

# 4. Memory Pressure 발생

메모리 점유를 위해 API를 여러 차례 호출하였다.

```bash
for i in {1..10}; do
  curl http://localhost:8080/api/test/memory
done
```

실행 결과

```text
allocated memory : 10 MB
allocated memory : 20 MB
...
allocated memory : 100 MB
```

---

# 5. Grafana 관찰 결과

## Heap Memory

Heap Memory가 기존 약 40~100MB 수준에서 약 250MB까지 증가하였다.

GC가 수행되더라도 기존 수준으로 즉시 감소하지 않고 높은 Heap 사용량을 유지하였다.

---

## CPU

메모리 할당 시 CPU 사용률이 순간적으로 증가하였으나 이후 다시 정상 수준으로 회복되었다.

---

## HTTP Requests

API 호출 시 HTTP Request 수가 일시적으로 증가하였다.

---

## JVM Live Threads

큰 변화 없이 안정적인 수준을 유지하였다.

---

## Non-Heap Memory

큰 변화 없이 안정적인 수준을 유지하였다.

---

## Response Time

약 7~8ms 수준으로 소폭 증가하였으나 서비스 성능에는 큰 영향이 없었다.

---

## GC Count

Minor GC는 계속 수행되었지만 Heap Memory는 이전 수준까지 감소하지 않았다.

이는 객체에 대한 참조가 계속 유지되고 있기 때문이다.

---

# 6. 복구 확인

실험 종료 후

- MemoryTestController 삭제
- SecurityConfig 원복
- Backend 재빌드

를 수행하였다.

```bash
docker compose up -d --build backend
```

재배포 후 Grafana를 확인한 결과

Heap Memory가 다시 기존 40~100MB 수준으로 복구되는 것을 확인하였다.

---

# 7. 결과 분석

이번 실험에서는 의도적으로 Heap Memory를 점유하여 메모리 압박 상황을 재현하였다.

Prometheus는 정상적으로 Metric을 수집하였으며 Target 역시 계속 UP 상태를 유지하였다.

메모리 사용량은 크게 증가하였지만

- CPU는 순간적으로만 증가
- Response Time은 큰 변화 없음
- 서비스는 정상 동작

하는 것을 확인하였다.

또한 테스트 코드를 제거하고 Backend를 재배포하자 Heap Memory가 다시 정상 수준으로 복구되는 것을 확인하였다.

---

# 실습 결론

이번 실습을 통해 메모리 사용량 증가 상황에서도 서비스는 정상적으로 동작할 수 있으며, Grafana를 통해 Heap Memory 증가를 실시간으로 확인할 수 있었다.

또한 객체 참조가 유지되는 경우 GC가 메모리를 즉시 회수하지 못한다는 점을 확인하였다.

실험 종료 후 테스트 코드를 제거하고 Backend를 재배포함으로써 Heap Memory가 정상 상태로 복구되는 과정까지 확인하였다.