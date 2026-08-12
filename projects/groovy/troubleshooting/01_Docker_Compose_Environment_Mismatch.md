# TS-001. 개발/운영 Docker Compose 혼용으로 인한 로컬 검증 실패

## 1. 문제 상황

Groovy 프로젝트에 MySQL Exporter를 적용한 뒤, 변경사항을 로컬 환경에서 검증하기 위해 운영 환경에서 사용하던 `docker-compose.prod.yml`을 실행하였다.

그러나 예상과 달리 수정한 코드가 정상적으로 반영되지 않았으며, 로컬 환경에서 정상적인 테스트를 진행할 수 없었다.

당시 확인된 주요 현상은 다음과 같았다.

```text
docker-compose.prod.yml 실행
        │
        ├─ Backend 컨테이너 정상 동작 실패
        ├─ Prometheus backend Target DOWN
        └─ MySQL Exporter Target UP
```

처음에는 동일한 애플리케이션을 실행하는 Compose 파일이므로 운영용 Compose도 로컬에서 그대로 사용할 수 있을 것이라고 판단하였다.

하지만 개발용과 운영용 Compose 파일의 구성 방식을 비교하면서 두 파일의 역할이 다르다는 것을 확인하였다.

---

## 2. Compose 구성 비교

### 2.1 개발 환경 - docker-compose.yml

개발 환경의 `docker-compose.yml`은 Backend와 Frontend를 현재 로컬 소스에서 직접 Build하도록 구성되어 있었다.

```yaml
backend:
  build:
    context: ./back

frontend:
  build:
    context: ./front
```

따라서 로컬 소스 코드를 수정한 뒤 Compose를 실행하면 현재 코드가 Docker 이미지에 반영된다.

```text
로컬 Source Code
        │
        ▼
docker compose build
        │
        ▼
현재 소스 기반 Image 생성
        │
        ▼
Container 실행
```

즉, 개발용 Compose는 **현재 작업 중인 코드를 로컬에서 빌드하고 검증하는 환경**이었다.

---

### 2.2 운영 환경 - docker-compose.prod.yml

반면 운영 환경의 `docker-compose.prod.yml`은 Backend와 Frontend를 직접 Build하지 않고 Docker Hub에 저장된 이미지를 사용하도록 구성되어 있었다.

```yaml
backend:
  image: bebeghi/groovy-backend:latest

frontend:
  image: bebeghi/groovy-frontend:latest
```

동작 구조는 다음과 같다.

```text
Docker Hub
        │
        ▼
배포용 Image Pull
        │
        ▼
Container 실행
```

즉, `docker-compose.prod.yml`을 실행한다고 해서 현재 Mac에서 수정한 로컬 소스가 자동으로 이미지에 반영되는 구조가 아니었다.

---

## 3. 원인 분석

### 3.1 build와 image의 역할 차이

문제의 핵심은 개발용과 운영용 Compose의 이미지 생성 방식이 서로 다르다는 점이었다.

```text
docker-compose.yml

Local Source
    ↓
build
    ↓
현재 코드가 반영된 Image
    ↓
Container
```

반면 운영 환경에서는 다음과 같이 동작하였다.

```text
docker-compose.prod.yml

Docker Hub
    ↓
기존 배포 Image Pull
    ↓
Container
```

따라서 로컬에서 MySQL Exporter 관련 코드나 설정을 수정했더라도 `docker-compose.prod.yml`이 참조하는 Docker Hub 이미지가 갱신되지 않았다면 해당 변경사항을 그대로 검증할 수 없었다.

즉,

> **로컬 소스를 검증하려는 목적과 이미 빌드된 배포 이미지를 실행하는 운영용 Compose의 목적이 일치하지 않았다.**

---

### 3.2 실행 환경의 CPU Architecture 차이

추가로 운영 환경과 개발 환경의 CPU Architecture도 서로 달랐다.

```text
운영 환경
Linux / amd64

개발 환경
macOS / Apple Silicon / arm64
```

운영 서버에서는 `amd64` 환경을 기준으로 이미지를 빌드하고 Docker Hub에 저장하여 사용하고 있었다.

반면 로컬 개발 환경은 Apple Silicon 기반의 `arm64` 환경이었다.

따라서 운영 환경을 기준으로 만들어진 이미지를 로컬에서 그대로 실행하는 방식은 이미지의 플랫폼 지원 여부도 함께 고려해야 했다.

이 문제를 통해 단순히 Compose 파일의 YAML 구성만 비교하는 것이 아니라, **해당 이미지가 어떤 환경을 기준으로 Build되었는지까지 확인해야 한다는 점**을 알게 되었다.

---

## 4. 해결

Compose 파일을 하나의 용도로 혼용하지 않고 **개발/테스트 환경과 운영 배포 환경의 역할을 분리**하였다.

### 개발 및 로컬 테스트

```text
docker-compose.yml
```

현재 로컬 소스를 직접 Build하여 실행한다.

```text
Local Source
    ↓
Build
    ↓
Local Image
    ↓
Container
```

따라서 코드나 설정 변경사항을 즉시 검증해야 할 때 사용한다.

### 운영 환경 배포

```text
docker-compose.prod.yml
```

Docker Hub에 Build/Push된 배포 이미지를 Pull하여 실행한다.

```text
Source
   ↓
CI/CD Build
   ↓
Docker Hub
   ↓
Production Pull
   ↓
Container
```

따라서 운영 서버에서는 서버 내부에서 애플리케이션 소스를 다시 Build하지 않고, 배포 파이프라인을 통해 생성된 이미지를 실행하도록 역할을 구분하였다.

---

## 5. 환경별 Compose 역할 정리

| 구분 | 개발 환경 | 운영 환경 |
|---|---|---|
| Compose | `docker-compose.yml` | `docker-compose.prod.yml` |
| 이미지 구성 | `build:` | `image:` |
| 이미지 생성 | 로컬 소스에서 직접 Build | CI/CD에서 Build된 이미지 사용 |
| 코드 변경 반영 | 로컬 Build 시 즉시 반영 | 새 이미지 Build/Push 필요 |
| 주요 목적 | 개발 및 기능 검증 | 운영 배포 |
| 실행 환경 | macOS / arm64 | Linux / amd64 |

결과적으로 두 Compose 파일은 같은 서비스를 실행하더라도 **서로 다른 목적과 배포 흐름을 가진 파일**이라는 것을 확인하였다.

---

## 6. 문제 흐름 정리

```text
MySQL Exporter 변경사항 로컬 검증 필요
        │
        ▼
docker-compose.prod.yml 실행
        │
        ▼
수정사항이 예상대로 반영되지 않음
        │
        ▼
개발/운영 Compose 비교
        │
        ├───────────────┐
        ▼               ▼
개발 Compose         운영 Compose
build:               image:
        │               │
        ▼               ▼
Local Source        Docker Hub Image
직접 Build          Pull 후 실행
        │               │
        └───────┬───────┘
                ▼
        역할 차이 확인
                │
                ▼
추가로 amd64 / arm64 환경 차이 확인
                │
                ▼
환경별 Compose 사용 목적 분리
                │
        ┌───────┴────────┐
        ▼                ▼
로컬 개발/검증         운영 배포
docker-compose.yml    docker-compose.prod.yml
```

---

## 7. 배운 점

이번 문제를 통해 Docker Compose 파일은 단순히 컨테이너를 실행하기 위한 설정 파일이 아니라 **어떤 소스와 이미지를 기준으로 어떤 환경을 구성할 것인지 정의하는 역할**을 한다는 점을 확인하였다.

특히 `build:`와 `image:`의 차이를 명확하게 이해하게 되었다.

```text
build:
→ 현재 소스를 이용하여 이미지를 생성

image:
→ 지정된 이미지를 이용하여 컨테이너 실행
```

따라서 로컬 코드 변경사항을 검증하려면 해당 코드가 실제 실행 이미지에 반영되는 경로를 먼저 확인해야 한다.

또한 개발 환경과 운영 환경의 CPU Architecture가 다를 경우 Docker 이미지의 플랫폼 호환성도 확인해야 한다.

이번 경험 이후 Compose 파일을 실행하기 전에 다음 항목을 먼저 확인하는 기준을 세웠다.

```text
1. 이 Compose 파일의 목적은 무엇인가?
2. build 방식인가, image 방식인가?
3. 현재 수정한 코드가 실행 이미지에 반영되는가?
4. 이미지가 어떤 Architecture를 대상으로 Build되었는가?
5. 현재 실행 환경과 이미지의 플랫폼이 호환되는가?
```

이를 통해 개발 환경에서는 현재 소스를 빠르게 검증하고, 운영 환경에서는 CI/CD를 통해 생성된 배포 이미지를 사용하는 방식으로 두 환경의 역할을 명확하게 분리할 수 있었다.