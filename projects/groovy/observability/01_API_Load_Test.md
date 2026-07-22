# 실습 1. ApacheBench를 이용한 API 부하 테스트

## 실습 목적

ApacheBench(ab)를 이용하여 다량의 API 요청을 발생시키고,
Prometheus와 Grafana를 통해 애플리케이션의 자원 사용량과 성능 변화를 관찰한다.

이를 통해 HTTP 요청 증가가 CPU, JVM Memory, GC, Thread, Response Time에 어떤 영향을 주는지 확인한다.

---

## 실습 환경

| 항목 | 내용 |
|------|------|
| Backend | Spring Boot |
| Monitoring | Prometheus |
| Dashboard | Grafana |
| Container | Docker Compose |
| Load Test Tool | ApacheBench(ab) |
| API | `/api/health` |

---

## 테스트 명령

```bash
ab -n 20000 -c 100 http://localhost:8080/api/health
```

### 옵션 설명

| 옵션 | 의미 |
|------|------|
| -n 20000 | 총 20,000개의 요청 수행 |
| -c 100 | 동시에 100개의 요청 전송 |

---

# 관찰 대상 Metric

이번 실습에서는 다음 Metric들을 관찰하였다.

- CPU Usage
- JVM Live Threads
- JVM Heap Memory
- JVM Non-Heap Memory
- GC Count
- HTTP Requests
- HTTP Response Time

---

# 실습 결과

## 1. HTTP Requests

### 관찰 결과

- 요청 수가 약 20,000건까지 급격히 증가하였다.

### 원인

ApacheBench가 짧은 시간 동안 대량의 HTTP 요청을 생성하였다.

---

## 2. CPU Usage

### 관찰 결과

- 약 0.x%
→ 최대 약 5~6%

### 원인

HTTP 요청을 처리하기 위해

- Controller 실행
- Security Filter
- Response 생성
- Micrometer Metric 기록

등의 작업이 수행되면서 CPU 사용량이 증가하였다.

다만 `/api/health`는 매우 단순한 API이므로
요청 수에 비해 CPU 사용률 증가는 크지 않았다.

---

## 3. JVM Live Threads

### 관찰 결과

약 30개
→ 약 33개

### 원인

요청 처리를 위한 JVM 내부 작업이 증가하면서
Live Thread 수가 소폭 증가하였다.

동시 요청이 100개라고 해서 Thread가 반드시 100개 생성되는 것은 아니다.

---

## 4. Heap Memory

### 관찰 결과

부하 테스트 중 Heap Memory가 크게 증가하였다.

이후 GC가 수행되면서 다시 감소하는 톱니(Saw-Tooth) 형태를 확인하였다.

### 원인

요청 처리 과정에서

- Request 객체
- Response 객체
- 문자열
- Metric 객체

등의 임시 객체가 생성되기 때문이다.

---

## 5. Non-Heap Memory

### 관찰 결과

소폭 증가하였다.

### 원인

JVM 내부에서 사용하는

- Class Metadata
- Method 정보
- JIT Compiler

등의 메모리 사용량이 증가한 것으로 판단된다.

Heap처럼 크게 변하지 않고
완만한 증가 형태를 보였다.

---

## 6. GC Count

### 관찰 결과

부하 테스트 동안 GC 발생 횟수가 증가하였다.

### 원인

Heap에 생성된 임시 객체가 많아지면서
Garbage Collector가 불필요한 객체를 더 자주 회수하였다.

GC Count는 메모리 사용량이 아니라

> 일정 시간 동안 GC가 몇 번 발생했는가

를 의미한다.

---

## 7. HTTP Response Time

### 관찰 결과

약 0.6ms
→ 약 40ms

### 원인

동시에 많은 요청이 발생하면서

- CPU
- Memory
- Thread

등의 자원을 공유하게 되었고,
각 요청의 평균 응답시간이 증가하였다.

---

# 전체 흐름

```text
ApacheBench 실행
        │
        ▼
HTTP 요청 급증
        │
        ▼
CPU 사용량 증가
        │
        ▼
임시 객체 생성
        │
        ▼
Heap Memory 증가
        │
        ▼
GC 수행 증가
        │
        ▼
응답시간 증가
```

---

# 실습 결론

ApacheBench를 이용하여 다량의 API 요청을 발생시킨 결과,

- HTTP Requests 증가
- CPU 사용량 증가
- Heap Memory 증가
- GC Count 증가
- Response Time 증가

를 확인하였다.

반면,

- Live Threads
- Non-Heap Memory

는 상대적으로 변화 폭이 작았다.

부하 종료 후에는 대부분의 Metric이 정상 수준으로 복구되었으며,
서비스 장애 없이 요청을 정상 처리하는 것을 확인하였다.

---

# 이번 실습에서 배운 점

- HTTP 요청 수가 증가하면 CPU, Memory, GC 등 여러 Metric이 함께 변화한다.
- Heap은 요청 처리 객체가 저장되는 영역이며, GC에 의해 주기적으로 회수된다.
- Non-Heap은 JVM 내부 실행 정보를 저장하는 영역으로 변화 폭이 상대적으로 작다.
- Response Time은 시스템 부하를 판단하는 중요한 지표이다.
- 하나의 Metric만 보는 것이 아니라 여러 Metric을 함께 분석해야 장애 원인을 정확하게 파악할 수 있다.

---

## 부하 종료 후(Baseline)

CPU
- 0%대로 복귀

Threads
- 약 31개 유지

Heap
- GC에 의해 안정적인 범위 유지

NonHeap
- 초기에는 증가하지만 점차 안정화

GC Count
- Heap 감소 시 증가하는 관계 확인

Response Time
- 약 1ms로 복귀

결론
- 애플리케이션이 부하 종료 후 정상 상태(Baseline)로 회복됨을 확인하였다.