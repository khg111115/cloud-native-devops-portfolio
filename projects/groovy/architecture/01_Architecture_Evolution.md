# Groovy Architecture Evolution

## 1. Overview

Groovy 프로젝트는 Docker Compose 기반의 초기 환경에서 시작하여 Kubernetes와 Helm을 도입하고, 이후 도메인 기반 MSA와 AWS EKS/RDS 환경으로 확장했습니다.

이 과정에서 단순히 기술을 추가하는 것이 아니라, 서비스 구조의 변화와 운영 환경의 요구사항에 따라 인프라와 Database Architecture를 단계적으로 재설계했습니다.

특히 MSA 전환 이후에는 서비스별 Database 분리, AWS RDS 구성 방식, Database 가용성 및 Connection Capacity와 같은 문제를 검토하며 실제 테스트 결과와 비용·운영 제약을 기반으로 아키텍처를 조정했습니다.

이 문서에서는 Groovy의 초기 Docker Compose 환경부터 최종 Cloud Native Architecture까지의 변화 과정과 각 단계에서의 주요 기술적 의사결정을 정리합니다.

## 2. Docker Compose 기반 초기 구조

프로젝트 초기에는 Frontend, Backend, MySQL, Redis를 Docker Compose 기반으로 구성하여 하나의 환경에서 서비스를 실행했습니다.

Docker Compose를 통해 여러 컨테이너의 실행과 서비스 간 연결을 관리할 수 있었지만, 프로젝트가 확장되면서 컨테이너 장애 시 복구, 서비스별 배포 및 확장, 리소스 관리와 같은 운영 관점의 요구사항을 함께 고려할 필요가 있었습니다.

또한 이후 AWS 환경으로의 확장을 계획하면서 특정 로컬 환경에 의존하는 구성보다 Kubernetes의 표준 리소스를 기반으로 애플리케이션 실행 환경을 구성하는 방향으로 전환했습니다.

이에 따라 기존 Docker Compose 환경을 Kubernetes로 이전하고, 이후 반복 가능한 배포와 설정 관리를 위해 Helm을 도입하는 것을 다음 단계로 결정했습니다.

## 3. Kubernetes & Helm으로의 전환

기존 Docker Compose 환경을 Kubernetes로 이전하기 위해 먼저 Kompose를 활용하여 Kubernetes Manifest의 초안을 생성했습니다.

자동 변환된 Manifest를 그대로 사용하지 않고 Deployment, StatefulSet, Service, PVC, ConfigMap 등의 Kubernetes 리소스 구조를 검토하고 프로젝트 환경에 맞게 수정했습니다. 이후 Mini PC의 Minikube 환경에서 서비스 간 연결과 애플리케이션 동작을 검증했습니다.

Raw Kubernetes Manifest로 서비스 실행을 확인한 이후에는 반복되는 설정을 템플릿화하고 이미지, 포트, Storage 등의 설정을 일관되게 관리하기 위해 Helm Chart 기반의 배포 구조로 전환했습니다.

이 과정에서 구성한 Kubernetes와 Helm 구조는 이후 AWS EKS 환경으로 서비스를 이전하는 기반으로 활용했습니다.

> 상세 구현 및 검증 과정은 [Kubernetes & Helm Migration](../infrastructure/01_Kubernetes_Helm_Migration.md)에서 정리합니다.

## 4. Domain-based MSA 전환

프로젝트가 고도화되면서 기존 Monolithic Backend를 Identity, Study, Content, Calendar, Notification의 5개 도메인 서비스로 분리하는 MSA 전환을 진행했습니다.

각 서비스가 독립적으로 배포되고 변경될 수 있는 구조로 전환되면서 애플리케이션뿐만 아니라 Database 구조 역시 기존 형태를 그대로 유지하기보다 서비스 경계에 맞게 재설계할 필요가 있었습니다.

이에 따라 각 서비스가 독립적인 Database를 사용하도록 Identity, Study, Content, Calendar, Notification Database를 분리했습니다.

이러한 변화는 이후 AWS 환경으로 이전하는 과정에서 각 서비스의 Database를 어떤 형태로 RDS에 구성할 것인지, 서비스 간 독립성과 인프라 비용 사이의 균형을 어떻게 맞출 것인지에 대한 추가적인 Architecture 의사결정으로 이어졌습니다.

## 5. Database Architecture 변화

MSA 전환 이후에는 서비스별 Database의 독립성을 인프라 계층에서도 유지하기 위해 Identity, Study, Content, Calendar, Notification 서비스마다 별도의 RDS Instance를 사용하는 구조를 우선 설계했습니다.

초기에는 서비스별 RDS Instance를 구성하는 방향으로 실제 인프라 작업을 진행했지만, 5개의 RDS Instance를 지속적으로 운영하는 것은 프로젝트 환경에서 비용 부담이 크다고 판단했습니다.

이에 따라 서비스별 RDS Instance를 유지하는 대신 하나의 RDS Instance를 공유하면서 내부 Database를 서비스별로 분리하는 구조로 재설계했습니다.

```text
초기 설계
Identity       → RDS
Study          → RDS
Content        → RDS
Calendar       → RDS
Notification   → RDS

                ↓ 비용 및 운영 조건 검토

최종 방향
                    ┌─ identity
                    ├─ study
Single RDS Instance ├─ content
                    ├─ calendar
                    └─ notification
```

이를 통해 서비스별 Database의 논리적 분리는 유지하면서 프로젝트 환경에서의 인프라 비용을 줄일 수 있었습니다.

다만 여러 서비스의 Database가 하나의 RDS Instance에 집중되면서 해당 Instance의 장애가 전체 서비스에 영향을 줄 수 있는 새로운 가용성 문제가 발생했습니다.

따라서 이후에는 RDS Multi-AZ를 적용하고 Failover 상황에서의 서비스 영향을 검증하는 방향으로 Database 가용성 구성을 확장했습니다.

Database 구조 변경 과정과 의사결정은 별도의 MSA Database Architecture 문서에서 상세히 정리합니다.

## 6. AWS Cloud Native Architecture로 확장

Kubernetes와 Helm 기반으로 구성한 애플리케이션 실행 환경을 바탕으로 최종적으로 AWS EKS와 RDS를 사용하는 Cloud Native 환경으로 확장했습니다.

AWS 환경에서는 앞서 결정한 Database Architecture를 기준으로 RDS의 가용성과 Connection Capacity를 실제 테스트를 통해 검증했습니다.

### RDS 가용성 검증

하나의 RDS Instance에 여러 서비스의 Database가 구성된 만큼, Database 장애에 대비하기 위해 Multi-AZ를 적용하고 강제 Failover 테스트를 수행했습니다.

테스트 과정에서 Primary DB의 Availability Zone 전환과 Failover 전후의 Database Connection 영향을 확인하고, 애플리케이션의 연결 복구 여부를 검증했습니다.

이를 통해 통합된 RDS 구조에서 Multi-AZ가 Database 가용성을 보완할 수 있는지 확인했습니다.

### RDS Connection Capacity 검증

여러 서비스가 하나의 RDS Instance의 Connection을 공유하기 때문에 실제 서비스의 Connection 수요를 기준으로 Instance Capacity를 검증했습니다.

서비스별 부하 테스트를 통해 Connection 사용량을 측정하고 HikariCP Pool을 조정했지만, `db.t4g.small` 환경에서는 약 137개의 가용 Connection 한계로 인해 애플리케이션 설정 조정만으로 필요한 Connection 수요를 안정적으로 수용하기 어렵다고 판단했습니다.

이에 따라 문제를 HikariCP 설정만의 문제가 아닌 RDS Instance 자체의 Capacity 문제로 판단하고 Scale-up을 검토했으며, 팀의 최종 운영 구성으로 `db.t4g.medium`을 선택했습니다.

이 과정에서 실제 서비스의 Connection 수요, Application Connection Pool, Infrastructure Capacity를 함께 검증하여 RDS Instance 규모를 결정했습니다.

## 7. Final Architecture

앞선 Architecture Evolution을 거쳐 Groovy는 Docker Compose 기반의 초기 환경에서 Kubernetes와 AWS를 기반으로 하는 Cloud Native Architecture로 확장했습니다.

![Groovy Final Service Architecture](./images/01_final_service_architecture.png)

최종적으로 도메인 기준으로 분리된 서비스는 AWS EKS 환경에서 운영하고, Database는 하나의 RDS Instance 내부에서 서비스별로 논리적으로 분리했습니다.

또한 RDS Multi-AZ와 Connection Capacity 검증을 통해 최종 Database 운영 구성을 결정했습니다.

전체 Architecture의 변화 과정은 다음과 같습니다.

```text
Docker Compose
      ↓
Kubernetes
      ↓
Helm
      ↓
Domain-based MSA
      ↓
Service-level Database Separation
      ↓
RDS 5 Instances 설계
      ↓
Cost Trade-off
      ↓
RDS 1 Instance + 5 Databases
      ↓
Multi-AZ / Failover Validation
      ↓
Connection Capacity Validation
      ↓
AWS Cloud Native Architecture
```
각 단계는 독립적인 기술 도입이 아니라, 이전 단계에서 확인한 운영 요구사항과 제약을 다음 Architecture 의사결정에 반영하며 발전시킨 결과입니다.

## 8. Architecture Evolution에서 얻은 점

Groovy의 아키텍처를 단계적으로 확장하면서 새로운 기술을 도입하는 것보다 현재 구조의 문제와 제약을 확인하고, 이를 다음 의사결정의 근거로 활용하는 과정이 중요하다는 점을 경험했습니다.

### 기술 선택에는 Trade-off가 존재합니다

서비스별 RDS 구성은 높은 독립성을 확보할 수 있었지만 프로젝트 환경에서는 비용 부담이 컸습니다.

이에 서비스별 데이터 경계는 유지하면서 RDS Instance를 통합했고, 그 과정에서 새롭게 발생한 가용성 요구사항은 Multi-AZ를 통해 보완했습니다.

이를 통해 하나의 요구사항만을 기준으로 구조를 결정하기보다 독립성, 비용, 가용성과 같은 여러 조건을 함께 고려해야 한다는 점을 경험했습니다.

### 설정이 아닌 검증 결과를 의사결정의 근거로 활용했습니다

Database Connection 문제에서는 HikariCP 설정 변경만으로 접근하지 않고 실제 부하 테스트를 통해 서비스별 Connection 수요와 RDS Capacity를 확인했습니다.

그 결과 애플리케이션 설정만으로 해결하기 어려운 인프라 Capacity의 한계를 확인했고, 이를 RDS Scale-up을 결정하는 근거로 활용했습니다.

### Application과 Infrastructure는 함께 변화합니다

Kubernetes 전환, MSA 분리, Database 구조 변경은 서로 독립적인 작업이 아니었습니다.

애플리케이션 구조의 변화는 Database의 가용성과 Connection 요구사항에 영향을 주었고, 반대로 인프라의 비용과 Capacity 제약은 Database Architecture의 결정에 영향을 주었습니다.

이를 통해 Cloud Native 환경에서는 개별 기술뿐만 아니라 Application, Database, Infrastructure 사이의 영향을 함께 고려하여 구조를 설계하고 검증해야 한다는 점을 배웠습니다.