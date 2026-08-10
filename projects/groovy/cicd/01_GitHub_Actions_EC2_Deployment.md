# GitHub Actions 기반 Docker 이미지 빌드 및 EC2 자동 배포

## 1. 개요

Groovy 프로젝트의 개인 실습 환경에서 GitHub Actions와 Docker Hub를 활용하여 CI/CD 파이프라인을 구성하였다.

기존에는 코드 변경 후 Docker 이미지를 직접 빌드하고 EC2에 접속하여 컨테이너를 갱신해야 했다.

이를 개선하여 `dev` 브랜치에 코드가 Push되면 GitHub Actions가 자동으로 Docker 이미지를 빌드하여 Docker Hub에 Push하고, 이후 EC2에 SSH로 접속하여 최신 이미지를 Pull한 뒤 컨테이너를 재생성하도록 구성하였다.

최종적인 배포 흐름은 다음과 같다.

```text
Developer
    ↓
Git Push
    ↓
GitHub Repository
    ↓
GitHub Actions
    ├─ Frontend Image Build
    └─ Backend Image Build
    ↓
Docker Hub Push
    ↓
EC2 SSH 접속
    ↓
docker compose pull
    ↓
docker compose up -d
    ↓
Nginx Restart
    ↓
Health Check
```

---

## 2. 로컬 환경과 배포 환경 분리

개발 환경에서는 소스 코드를 직접 빌드할 수 있도록 `docker-compose.yml`을 사용하였다.

반면 EC2 배포 환경에서는 EC2 내부에서 애플리케이션을 다시 빌드하지 않고, CI 과정에서 미리 생성된 Docker 이미지를 가져와 실행하도록 `docker-compose.deploy.yml`을 별도로 구성하였다.

### Local

```yaml
backend:
  build:
    context: ./groovy
    dockerfile: Dockerfile
```

### Deploy

```yaml
backend:
  image: khg1115/groovy-backend:latest
```

즉, 두 Compose 파일의 역할을 다음과 같이 분리하였다.

```text
docker-compose.yml
→ 로컬 개발 및 직접 Build

docker-compose.deploy.yml
→ 배포 서버에서 Docker Hub Image 실행
```

![로컬 및 배포 Compose 구성 비교](./images/01_compose_local_vs_deploy.png)

이를 통해 개발 환경과 실제 배포 환경의 역할을 분리하고, EC2에서는 애플리케이션 빌드 작업 없이 배포에 필요한 이미지만 Pull하도록 구성하였다.

---

## 3. GitHub Actions를 통한 Docker 이미지 자동 빌드

CI 단계에서는 GitHub Actions를 이용하여 Frontend와 Backend Docker 이미지를 자동으로 빌드하도록 구성하였다.

Workflow는 다음 과정을 수행한다.

```text
Checkout Repository
        ↓
Set up Docker Buildx
        ↓
Login to Docker Hub
        ↓
Build Frontend Image
        ↓
Push Frontend Image
        ↓
Build Backend Image
        ↓
Push Backend Image
```

실제 GitHub Actions 실행 결과 Frontend와 Backend 이미지 모두 정상적으로 Build & Push가 완료되는 것을 확인하였다.

![GitHub Actions Docker Build 및 Push 성공](./images/02_github_actions_build_push.png)

이 구조를 통해 개발자가 직접 Docker 이미지를 빌드하고 Push하는 과정을 자동화하였다.

---

## 4. Docker Hub 이미지 저장

GitHub Actions에서 빌드된 이미지는 Docker Hub에 저장하도록 구성하였다.

사용한 Repository는 다음과 같다.

```text
khg1115/groovy-frontend
khg1115/groovy-backend
```

GitHub Actions 실행 후 두 Repository의 `Last Pushed` 시간이 갱신되는 것을 확인하여 이미지가 정상적으로 Push되었음을 검증하였다.

![Docker Hub 이미지 Push 결과](./images/03_dockerhub_images.png)

이후 EC2에서는 애플리케이션 소스 코드를 직접 빌드하는 대신 해당 이미지를 Pull하여 사용한다.

---

## 5. EC2 자동 배포

Docker 이미지 Build & Push가 성공하면 다음 단계에서 GitHub Actions가 EC2에 SSH로 접속하여 자동 배포를 수행하도록 구성하였다.

주요 배포 과정은 다음과 같다.

```bash
git pull origin dev

docker compose -f docker-compose.deploy.yml pull

docker compose -f docker-compose.deploy.yml up -d

docker compose -f docker-compose.deploy.yml restart nginx

sleep 15

docker compose -f docker-compose.deploy.yml ps

curl -fsS http://localhost/api/health
```

각 명령의 역할은 다음과 같다.

```text
git pull
→ EC2 Repository 최신화

docker compose pull
→ Docker Hub에서 최신 이미지 다운로드

docker compose up -d
→ 변경된 이미지 기반으로 컨테이너 재생성

restart nginx
→ Reverse Proxy 재시작

docker compose ps
→ 컨테이너 실행 상태 확인

curl /api/health
→ Backend 서비스 정상 여부 확인
```

GitHub Actions의 Deploy Job이 성공하면서 위 과정 전체가 자동으로 수행되는 것을 확인하였다.

![GitHub Actions EC2 자동 배포 성공](./images/04_github_actions_ec2_deploy.png)

---

## 6. 배포 결과 검증

GitHub Actions 성공 여부만으로 배포가 완료되었다고 판단하지 않고 EC2에 직접 접속하여 컨테이너 상태와 Backend Health Check를 추가로 검증하였다.

```bash
docker ps
```

확인 결과 다음 컨테이너가 정상적으로 실행되고 있었다.

```text
groovy-nginx
groovy-front
groovy-backend
groovy-mysql
```

특히 Backend와 MySQL은 `healthy` 상태임을 확인하였다.

추가로 Nginx를 통해 Backend Health API를 호출하였다.

```bash
curl -i http://localhost/api/health
```

결과:

```text
HTTP/1.1 200
```

응답 Body에서도 Backend가 정상적으로 동작하고 있음을 확인하였다.

![EC2 컨테이너 및 Health Check 검증](./images/05_ec2_deployment_health_check.png)

이를 통해 단순히 GitHub Actions Workflow가 성공한 것뿐만 아니라,

```text
Docker Image Build
→ Docker Hub Push
→ EC2 Image Pull
→ Container 실행
→ Nginx Proxy
→ Backend Health Check
```

까지 전체 배포 경로가 정상적으로 동작함을 검증하였다.

---

## 7. Troubleshooting - EC2 Docker 권한 문제

### 문제 상황

초기 자동 배포 과정에서는 Docker 이미지 Build & Push 단계는 정상적으로 완료되었지만 EC2 Deploy 단계에서 실패하였다.

GitHub Actions는 EC2에 SSH 접속한 후 다음 명령을 실행하도록 구성되어 있었다.

```bash
sudo docker compose -f docker-compose.deploy.yml pull
```

하지만 Deploy Job에서 Docker 명령이 정상적으로 처리되지 않으면서 배포가 중단되었다.

![EC2 Deploy 실패](./images/06_troubleshooting_sudo_compose.png)

### 원인 분석

초기에는 EC2에서 Docker 명령을 실행하기 위해 `sudo`를 사용하는 형태로 Workflow를 작성하였다.

그러나 GitHub Actions의 SSH 기반 비대화형 실행 환경에서 불필요하게 `sudo`를 사용하면서 Docker 실행 과정에 문제가 발생하였다.

EC2 사용자의 Docker 실행 권한을 확인한 뒤 별도의 `sudo` 없이 Docker 명령을 실행할 수 있도록 Workflow를 수정하였다.

### 수정

기존:

```bash
sudo docker compose -f docker-compose.deploy.yml pull
sudo docker compose -f docker-compose.deploy.yml up -d
sudo docker compose -f docker-compose.deploy.yml restart nginx
sudo docker compose -f docker-compose.deploy.yml ps
```

수정:

```bash
docker compose -f docker-compose.deploy.yml pull
docker compose -f docker-compose.deploy.yml up -d
docker compose -f docker-compose.deploy.yml restart nginx
docker compose -f docker-compose.deploy.yml ps
```

수정 후 GitHub Actions를 다시 실행하였으며 Deploy Job이 정상적으로 완료되었다.

최종적으로 Workflow에서 다음과 같이 성공 상태를 확인하였다.

```text
build-and-push  → Success
deploy          → Success
```

이후 EC2에서도 컨테이너 상태 및 `/api/health`의 HTTP 200 응답을 확인하여 실제 서비스까지 정상적으로 배포되었음을 재검증하였다.

---

## 8. 최종 구성

최종 CI/CD 구조는 다음과 같다.

```text
[Developer]
     │
     │ Git Push
     ▼
[GitHub Repository]
     │
     ▼
[GitHub Actions]
     │
     ├── Frontend Build
     ├── Backend Build
     │
     ▼
[Docker Hub]
     │
     │ Push / Pull
     ▼
[EC2]
     │
     ├── docker compose pull
     ├── docker compose up -d
     ├── nginx restart
     └── health check
```

이번 구성을 통해 코드 변경 이후 Docker 이미지 생성부터 EC2 컨테이너 갱신 및 Health Check까지 이어지는 자동 배포 흐름을 구현하였다.

기존의 수동 배포 과정을 자동화함으로써 반복적인 배포 작업을 줄였으며, 배포 마지막 단계에 Health Check를 포함하여 Workflow 성공 여부뿐만 아니라 실제 애플리케이션의 정상 동작까지 검증할 수 있도록 구성하였다.