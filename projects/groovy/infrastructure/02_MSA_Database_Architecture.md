# Groovy MSA Database Architecture

## 1. Overview

Groovy 프로젝트는 Monolith Backend를 도메인 기준의 MSA 구조로 전환하면서 Database Architecture 역시 함께 재설계했습니다.

초기에는 하나의 MySQL Instance 내부에서 서비스별 Database와 User를 분리하여 데이터 소유권을 구분했고, 이후 단일 MySQL이 Connection, CPU/Memory, Storage 및 장애 영향 범위를 공유한다는 한계를 고려하여 서비스별 MySQL Instance를 분리했습니다.

서비스별 MySQL 구조는 Kubernetes 환경에서 StatefulSet, Service, PVC 및 DB 초기화 구조까지 분리하여 실제 동작을 검증했으며, AWS 이전 단계에서는 이를 서비스별 RDS 구성으로 확장하는 방안을 검토했습니다.

그러나 프로젝트 규모와 운영 비용을 다시 검토한 결과, 최종 AWS 환경에서는 하나의 RDS Instance 내부에 5개의 Database를 논리적으로 분리하는 구조로 재설계했습니다.

이 문서에서는 서비스 독립성, 장애 격리, 운영 비용 사이의 Trade-off를 고려하며 Database Architecture를 단계적으로 변경한 과정과 그 판단 기준을 정리합니다.

## 2. Monolith → MSA 전환과 데이터 경계 분리
초기 Groovy Backend는 하나의 애플리케이션과 하나의 `groovy` Database를 사용하는 구조였습니다.

Backend를 Identity, Study, Content, Calendar, Notification의 도메인 기준 MSA로 분리하면서 애플리케이션만 분리하고 Database를 그대로 공유할 경우, 서비스 간 데이터 경계와 소유권이 불명확해질 수 있다고 판단했습니다.

따라서 각 서비스가 자신의 데이터를 소유하도록 Database 경계를 함께 분리했습니다.

다른 서비스의 데이터를 직접 조회하기보다 해당 데이터를 소유한 서비스의 API를 통해 접근하는 것을 기본 원칙으로 두었습니다.

이 단계의 목표는 MySQL 서버 자체를 물리적으로 분리하는 것이 아니라, 먼저 **서비스 경계와 데이터 소유권을 일치시키는 것**이었습니다.


## 3. 1 MySQL / 5 Database 논리적 분리

### 서비스별 Database 분리
MSA 전환 초기에는 MySQL Instance 자체를 서비스별로 분리하지 않고, 하나의 MySQL 내부에 5개의 Database를 생성하는 방식으로 데이터 영역을 분리했습니다.

~~~text
MySQL Instance
├── identity_db
├── study_db
├── content_db
├── calendar_db
└── notification_db
~~~

각 Backend Service는 자신의 도메인에 해당하는 Database만 사용하도록 구성했습니다.

| Service | Database |
| --- | --- |
| identity-service | `identity_db` |
| study-service | `study_db` |
| content-service | `content_db` |
| calendar-service | `calendar_db` |
| notification-service | `notification_db` |

이 구조를 통해 하나의 MySQL Instance를 유지하면서도 서비스별 데이터 경계를 먼저 분리할 수 있었습니다.

다만 이 시점의 분리는 **논리적 분리**였습니다. Database는 서비스별로 나뉘었지만 실제 MySQL Process와 Connection Limit, CPU/Memory, Storage는 여전히 하나의 Instance를 공유하고 있었습니다.

### 서비스별 User / 권한 분리
Database 분리와 함께 각 서비스가 사용하는 MySQL User도 서비스별로 분리했습니다.

| Service | MySQL User | Database |
| --- | --- | --- |
| identity-service | `identity_service` | `identity_db` |
| study-service | `study_service` | `study_db` |
| content-service | `content_service` | `content_db` |
| calendar-service | `calendar_service` | `calendar_db` |
| notification-service | `notification_service` | `notification_db` |

각 User에는 자신이 담당하는 Database에 대한 권한만 부여하여, 하나의 MySQL Instance를 공유하더라도 서비스별 접근 범위를 구분했습니다.

~~~text
identity-service
      ↓
identity_service
      ↓
identity_db

study-service
      ↓
study_service
      ↓
study_db

...
~~~

이를 통해 **Service → User → Database**의 대응 관계를 명확하게 구성하고, 서비스가 다른 도메인의 Database에 직접 접근하는 범위를 제한했습니다.

### DB 초기화 자동화와 관리 책임 분리
초기 Kubernetes 환경에서는 MySQL 최초 기동 시 서비스별 Database와 User를 자동으로 생성하도록 DB 초기화 과정을 구성했습니다.

Infra Helm Chart에서 서비스별 초기화 SQL을 ConfigMap으로 관리하고, 이를 MySQL의 `/docker-entrypoint-initdb.d/`에 Mount하여 최초 초기화 시 Database 생성, User 생성 및 권한 설정이 수행되도록 했습니다.

~~~text
Infra Helm Chart
      ↓
Service별 Init SQL
      ↓
ConfigMap
      ↓
/docker-entrypoint-initdb.d/
      ↓
Database / User / Permission 생성
~~~

그러나 이 구조를 운영하면서 두 가지 한계를 확인했습니다.

첫째, `/docker-entrypoint-initdb.d/`의 초기화 Script는 MySQL Data Directory가 처음 생성될 때 실행되기 때문에, 이미 PVC가 초기화된 환경에서는 SQL을 변경하거나 새로운 Script를 추가해도 Pod 재시작만으로 다시 적용되지 않았습니다.

둘째, 실제 Database를 사용하는 주체는 각 Backend Service인데 Database 생성과 Schema 초기화 책임은 Infra Repository에 위치하여 **서비스의 데이터 소유권과 변경 책임이 일치하지 않았습니다.**

이에 따라 역할을 다음과 같이 재정의했습니다.

~~~text
Infrastructure
→ MySQL 실행 환경과 공통 Infrastructure 제공

Each Backend Service
→ 자신의 Database / Schema 초기화 및 변경 관리
~~~

이를 통해 MSA의 데이터 소유권을 단순히 Database 분리에만 적용하지 않고, **Database 변경에 대한 관리 책임까지 각 서비스 경계에 맞추는 방향으로 구조를 개선했습니다.**

## 4. 서비스별 MySQL Instance 물리적 분리

### 단일 MySQL 구조의 한계
논리적으로 Database와 User를 서비스별로 분리했지만, 모든 Database가 하나의 MySQL Instance에서 실행된다는 점은 그대로 남아 있었습니다.

~~~text
identity_db
study_db
content_db
calendar_db
notification_db
        ↓
Single MySQL Instance
        ↓
Connection / CPU / Memory / Storage 공유
~~~

이 구조에서는 특정 서비스의 Database 부하가 증가하더라도 실제로 사용하는 MySQL Process와 시스템 Resource가 동일하기 때문에 다른 서비스에도 영향을 줄 수 있습니다.

특히 다음 항목은 서비스별 Database 분리만으로 격리되지 않았습니다.

- MySQL Connection Limit
- CPU / Memory
- Storage 및 PVC
- MySQL Process 장애
- 장애 발생 시 영향 범위

기초 프로젝트에서 MySQL Connection Exhaustion을 재현하면서 하나의 MySQL Connection 한계가 Database를 사용하는 서비스와 Monitoring에도 영향을 줄 수 있음을 경험했기 때문에, MSA 전환 과정에서는 데이터의 논리적 소유권뿐 아니라 **Database 장애와 Resource의 영향 범위를 서비스 단위로 격리할 필요성**도 검토했습니다.

따라서 Database/User만 분리하는 1차 구조에서 한 단계 더 나아가, MySQL Instance 자체를 서비스별로 분리하는 방향으로 구조를 변경했습니다.

### 5 MySQL Instance 구조 설계
기존의 하나의 MySQL Instance를 서비스별로 독립된 5개의 MySQL Instance로 분리했습니다.

각 서비스의 MySQL은 독립적인 Kubernetes `StatefulSet`, `Service`, `PVC`를 갖도록 구성했습니다.

| Service | MySQL Service | PVC |
| --- | --- | --- |
| identity-service | `identity-mysql` | `identity-mysql-data` |
| study-service | `study-mysql` | `study-mysql-data` |
| content-service | `content-mysql` | `content-mysql-data` |
| calendar-service | `calendar-mysql` | `calendar-mysql-data` |
| notification-service | `notification-mysql` | `notification-mysql-data` |

Backend 역시 기존의 공용 `mysql` Service가 아니라 자신의 MySQL Service DNS를 사용하도록 변경했고, DB 초기화 작업과 관리자 Credential도 서비스별 MySQL을 대상으로 분리했습니다.

이를 통해 서비스별 데이터 경계를 Database/User 수준에서 MySQL 실행 환경과 Storage 수준까지 확장하여, 특정 MySQL의 장애나 Resource 사용 증가가 다른 서비스의 Database에 직접 미치는 영향 범위를 줄이는 구조로 변경했습니다.

### Helm 기반 Resource 분리
MySQL Instance가 5개로 증가하면서 서비스마다 StatefulSet, Service, PVC Manifest를 개별적으로 작성할 경우 동일한 Kubernetes Resource 정의가 반복되는 문제가 발생할 수 있었습니다.

이를 줄이기 위해 Helm `values.yaml`에 서비스별 MySQL Instance 정보를 정의하고, Template에서 이를 순회하여 필요한 Resource를 생성하도록 구성했습니다.

~~~yaml
mysql:
  instances:
    - name: identity
    - name: study
    - name: content
    - name: calendar
    - name: notification
~~~

~~~text
mysql.instances
      ↓
Helm Template
      ↓
서비스별 Resource 생성
      ├── StatefulSet
      ├── Service
      └── PVC
~~~

각 Backend의 DB Host 역시 공용 MySQL Service가 아닌 서비스별 Kubernetes Service DNS를 사용하도록 변경했습니다.

~~~text
identity-service      → identity-mysql
study-service         → study-mysql
content-service       → content-mysql
calendar-service      → calendar-mysql
notification-service  → notification-mysql
~~~

이를 통해 동일한 MySQL 배포 구조를 반복해서 작성하는 대신, 서비스별 차이는 Values로 관리하고 공통 Resource 구조는 Helm Template으로 재사용할 수 있도록 구성했습니다.

### Minikube 검증
서비스별 MySQL 분리 이후 기존 환경의 영향을 받지 않는 Clean Minikube 환경에서 전체 구조를 다시 배포하여 정상 동작 여부를 검증했습니다.

검증 결과 서비스별 MySQL Resource와 DB 초기화 작업이 모두 정상적으로 생성되었습니다.

| 검증 항목 | 결과 |
| --- | --- |
| MySQL StatefulSet | 5개 모두 Running |
| MySQL Service | 5개 생성 |
| PVC | 5개 모두 Bound |
| DB-init Job | 5개 모두 Complete |

각 MySQL Instance 내부에도 해당 서비스가 소유하는 Database가 생성되었는지 직접 확인했습니다.

~~~text
identity-mysql      → identity_db
study-mysql         → study_db
content-mysql       → content_db
calendar-mysql      → calendar_db
notification-mysql  → notification_db
~~~

이를 통해 하나의 MySQL을 논리적으로만 분리했던 구조에서 **서비스별 MySQL 실행 환경과 Storage가 독립된 구조로 전환되었으며, 이를 실제 Kubernetes 환경에서 배포 및 검증했습니다.**

다만 각 MySQL StatefulSet은 `replicaCount: 1`로 구성되어 있었기 때문에, 이 구조의 목적은 MySQL 자체의 고가용성을 확보하는 것이 아니라 **서비스별 Database의 Resource 및 장애 영향 범위를 분리하는 것**이었습니다.

## 5. AWS RDS Architecture 검토

### 서비스별 5 RDS 설계
Kubernetes 환경에서 서비스별 MySQL Instance 분리를 검증한 이후, AWS 환경에서도 동일한 서비스 경계를 유지하기 위해 서비스별 RDS 구성을 검토했습니다.

~~~text
api-gateway          → DB 없음

identity-service     → identity RDS
study-service        → study RDS
content-service      → content RDS
calendar-service     → calendar RDS
notification-service → notification RDS
~~~

초기 AWS Architecture에서는 5개의 Backend Service가 각각 독립된 RDS를 사용하도록 설계했습니다.

이 구조는 Kubernetes에서 검증했던 서비스별 Database 격리를 AWS Managed Database 환경까지 확장하여, 서비스마다 독립적인 DB Resource와 장애 영향 범위를 갖도록 하는 것이 목적이었습니다.

다만 RDS는 서비스별로 Instance를 분리할수록 운영 비용도 함께 증가하며, Multi-AZ 적용 여부에 따라서도 추가 비용이 발생합니다.

따라서 AWS 전환에서는 단순히 Kubernetes의 5 MySQL 구조를 그대로 옮기는 것이 아니라, **서비스 독립성과 장애 격리를 위해 필요한 비용을 프로젝트 규모에서 감당할 가치가 있는지** 함께 검토했습니다.

### 비용과 가용성 Trade-off
서비스별 5 RDS 구성을 전제로, 모든 Database에 동일한 가용성 수준을 적용하기보다 프로젝트 비용과 서비스별 장애 영향도를 함께 고려했습니다.

검토한 구성은 다음과 같습니다.

| 구성 | RDS 구성 | Multi-AZ | 특징 |
| --- | --- | --- | --- |
| A안 | 5 RDS | 적용하지 않음 | 비용 최소화, DB 장애 시 복구 필요 |
| B안 | 5 RDS | 핵심 DB에 선택 적용 | 비용과 가용성 절충 |
| C안 | 5 RDS | 전체 적용 | 높은 가용성, 가장 높은 비용 |

이 과정에서 Snapshot과 PITR은 장애 발생 시 즉시 서비스를 이어받는 Standby가 아니라 **Backup / Disaster Recovery 수단**으로 구분했습니다.

따라서 단순히 Backup이 존재하는지를 기준으로 고가용성을 판단하지 않고, 장애 발생 시 자동 Failover가 필요한 서비스인지와 그에 따른 추가 비용을 함께 고려했습니다.

~~~text
Service Importance
        +
Failure Impact
        +
Infrastructure Cost
        ↓
Multi-AZ 적용 범위 결정
~~~

모든 서비스에 동일한 Infrastructure 구성을 적용하는 대신, **서비스의 장애 영향도에 따라 가용성 수준을 다르게 가져갈 수 있는 구조**를 검토했습니다.

### 선택적 Multi-AZ 구성 검토
5개의 RDS를 모두 Multi-AZ로 구성할 경우 높은 가용성을 확보할 수 있지만, 프로젝트 규모에서 모든 Database에 동일한 수준의 HA를 적용하는 것은 비용 부담이 크다고 판단했습니다.

이에 따라 당시에는 서비스별 장애 영향도를 기준으로 핵심 Database부터 Multi-AZ를 적용하는 B안을 우선 선택했습니다.

우선 대상으로는 인증과 로그인을 담당하는 `identity-service`의 Database를 선정했습니다.

~~~text
identity RDS
├── Primary
└── Standby
    → Multi-AZ

study RDS
content RDS
calendar RDS
notification RDS
    → Single-AZ + Backup / PITR
~~~

Identity Database에 장애가 발생하면 다른 Backend Service가 정상적으로 동작하더라도 로그인과 인증 과정에 영향을 주어 전체 서비스 이용에 미치는 영향이 클 수 있다고 판단했습니다.

따라서 모든 Database에 동일한 HA 수준을 적용하기보다, **서비스 중요도와 장애 영향도를 기준으로 Multi-AZ 적용 범위를 결정하고 필요에 따라 확대하는 방식**을 선택했습니다.

이 결정은 5개의 RDS를 운영한다는 당시 Architecture를 전제로 한 절충안이었습니다. 이후 실제 AWS 이관 과정에서는 프로젝트 규모와 전체 RDS 운영 비용을 다시 검토하면서 **RDS Instance 수 자체를 줄이는 방향으로 Architecture를 다시 조정하게 되었습니다.**

## 6. 5 RDS → 1 RDS / 5 Database 재설계

### 프로젝트 규모와 운영 비용 재검토
서비스별 RDS 5개 구성은 각 서비스의 Database Resource와 장애 영향 범위를 물리적으로 분리할 수 있다는 장점이 있었습니다.

그러나 AWS 이관을 구체화하면서 프로젝트 규모와 실제 운영 비용을 다시 검토했고, 서비스별 RDS Instance를 각각 유지하는 구조는 현재 프로젝트에서 비용 부담이 크다고 판단했습니다.

특히 Database를 물리적으로 분리하면 다음과 같은 격리 효과를 얻을 수 있지만, 그만큼 서비스별 RDS Compute와 Storage를 각각 운영해야 했습니다.

이에 따라 **서비스별 물리적 Database 격리에서 얻는 이점과 이를 유지하기 위해 필요한 AWS 비용을 다시 비교**했습니다.

팀 논의 결과, 현재 프로젝트에서는 5개의 RDS Instance를 계속 운영하기보다 RDS Instance를 통합하여 비용을 줄이되, MSA 전환 과정에서 정의한 서비스별 데이터 경계는 유지하는 방향으로 Architecture를 재설계했습니다.

### 물리적 격리와 비용 사이의 Trade-off
5개의 RDS를 하나의 RDS Instance로 통합하면서 서비스별 Database의 물리적 격리는 포기하게 되었습니다.

~~~text
Before

identity-service      → identity RDS
study-service         → study RDS
content-service       → content RDS
calendar-service      → calendar RDS
notification-service  → notification RDS


After

                    ┌─ identity_db
                    ├─ study_db
5 Backend Services → RDS ─ content_db
                    ├─ calendar_db
                    └─ notification_db
~~~

통합 이후에는 다시 하나의 RDS Instance에서 Compute, Connection Capacity 및 장애 영향 범위를 공유하게 됩니다. 따라서 특정 서비스의 DB 사용량 증가나 RDS Instance 장애가 여러 서비스에 영향을 줄 수 있다는 Trade-off가 존재합니다.

반면 MSA 전환 과정에서 정의한 서비스별 데이터 경계 자체를 다시 합치지는 않았습니다.

각 서비스는 기존과 동일하게 자신의 Database를 사용하도록 유지하여 다음과 같이 **물리적 Infrastructure와 논리적 데이터 경계를 분리해서 판단**했습니다.

~~~text
Physical Infrastructure
→ 1 RDS Instance로 통합

Logical Data Boundary
→ 5 Service Database 유지
~~~

즉, 서비스별 RDS가 제공하는 물리적 격리보다 프로젝트 단계에서의 비용 효율을 우선하되, 향후 서비스 규모와 부하가 증가할 경우 다시 서비스별 RDS 분리 또는 개별 확장을 검토할 수 있도록 Database 경계는 유지했습니다.

또한 RDS 통합으로 장애 영향 범위가 다시 하나의 Instance로 집중되는 만큼, **단일 RDS의 가용성과 Connection Capacity를 별도의 운영 과제로 검증할 필요가 생겼습니다.**

이는 이후 `RDS Multi-AZ Failover`와 `RDS Connection Capacity / HikariCP` 검증으로 이어졌습니다.

### 최종 Database Architecture
최종 AWS 환경에서는 하나의 RDS Instance 내부에 서비스별 5개의 Database를 유지하는 구조를 선택했습니다.

최종 구조에서 유지한 원칙은 다음과 같습니다.

| 항목 | 최종 구성 |
| --- | --- |
| RDS Instance | 1개 |
| Service Database | 5개 |
| 데이터 소유권 | 서비스별 분리 유지 |
| 물리적 DB Resource | RDS Instance 공유 |
| Connection Capacity | RDS Instance 단위로 공유 |
| 장애 영향 범위 | RDS Instance에 집중 |
| 고가용성 | Multi-AZ 별도 검증 |
| Connection 한계 | 부하 테스트 및 HikariCP와 함께 별도 검증 |

이 구조는 서비스별 RDS의 물리적 격리를 유지하는 것보다 **현재 프로젝트 규모에서의 비용 효율을 우선한 결과**입니다.

동시에 RDS Instance를 통합함으로써 Connection Capacity와 Database 장애가 여러 서비스에 영향을 줄 수 있다는 새로운 운영 과제도 명확해졌습니다.

따라서 최종 Architecture를 단순히 구성하는 데서 끝내지 않고, 이후 단계에서 **Multi-AZ Failover를 통한 가용성 검증**과 **실제 부하를 기반으로 한 RDS Connection Capacity 및 HikariCP 검증**을 진행했습니다.

## 7. Architecture Evolution

Groovy의 Database Architecture는 MSA 전환과 AWS 이관 과정에서 서비스 경계, 장애 격리, 운영 비용을 기준으로 단계적으로 변경되었습니다.

~~~text
① Monolith
Backend
   ↓
Single MySQL / groovy DB

        ↓ MSA 전환

② Logical Separation
5 Backend Services
   ↓
1 MySQL / 5 Databases
   ↓
서비스별 Database / User 분리

        ↓ Resource 및 장애 영향 범위 격리

③ Physical Separation
5 Backend Services
   ↓
5 MySQL Instances
   ↓
5 StatefulSets / 5 Services / 5 PVCs
   ↓
Clean Minikube 검증

        ↓ AWS 이관 설계

④ Service-specific RDS
5 Backend Services
   ↓
5 RDS 검토
   ↓
비용 / 가용성 Trade-off 분석
   ↓
선택적 Multi-AZ 구성 검토

        ↓ 프로젝트 규모와 운영 비용 재검토

⑤ Final AWS Architecture
5 Backend Services
   ↓
1 RDS Instance
   ↓
5 Logical Databases
~~~

각 단계에서 모든 요구사항을 동시에 최대화하기보다 당시 환경에서 우선해야 할 기준을 선택했습니다.

| 단계 | 우선한 기준 | 남은 과제 |
| --- | --- | --- |
| 1 MySQL / 5 DB | 데이터 소유권 | Resource와 장애 범위 공유 |
| 5 MySQL | 장애 및 Resource 격리 | AWS 이관 시 서비스별 DB 운영 비용 검토 필요 |
| 5 RDS 검토 | 서비스별 Infrastructure 독립성 | AWS 운영 비용 증가 |
| 1 RDS / 5 DB | 비용 효율 + 논리적 경계 유지 | HA 및 Connection Capacity |

최종적으로 **서비스별 데이터 소유권은 유지하면서 물리적 Infrastructure는 프로젝트 규모에 맞게 통합**했습니다.

그리고 통합으로 인해 다시 중요해진 RDS의 장애 영향 범위와 Connection Capacity는 이후 별도의 고가용성 및 부하 검증 대상으로 이어졌습니다.

## 8. Result & Lessons Learned
Groovy의 MSA 전환 과정에서 Database Architecture를 단순히 서비스 수에 맞춰 분리하는 데 그치지 않고, 데이터 소유권부터 Resource 격리, 장애 영향 범위, AWS 운영 비용까지 단계적으로 검토했습니다.

### Result

- Monolith의 단일 Database를 서비스별 5개 Database로 분리하여 데이터 소유권을 명확히 했습니다.
- 서비스별 MySQL User와 권한을 분리하여 Service → User → Database의 경계를 구성했습니다.
- Kubernetes 환경에서는 5개의 MySQL StatefulSet, Service, PVC로 물리적 분리를 구현하고 Clean Minikube 환경에서 정상 동작을 검증했습니다.
- AWS 이관 과정에서 서비스별 5 RDS와 Multi-AZ 적용 범위를 검토하며 비용과 가용성의 Trade-off를 비교했습니다.
- 최종적으로 프로젝트 규모와 비용을 고려하여 1개의 RDS Instance와 5개의 논리적 Database 구조로 재설계했습니다.

### Lessons Learned

**1. MSA의 Database 분리는 Database 개수를 늘리는 것만으로 완성되지 않는다.**

서비스별 데이터 소유권뿐 아니라 User/Permission, Schema 변경 책임, Resource 공유 범위까지 함께 고려해야 서비스 경계를 Infrastructure에도 일관되게 적용할 수 있었습니다.

**2. 논리적 분리와 물리적 분리는 해결하는 문제가 다르다.**

Database/User 분리는 데이터 소유권을 구분할 수 있지만 MySQL Instance의 Connection, Compute, Storage와 장애 영향 범위까지 격리하지는 못했습니다. 반대로 물리적 분리는 더 강한 격리를 제공하지만 운영 Resource와 비용을 증가시켰습니다.

**3. 더 강한 격리가 항상 현재 환경의 최종 선택은 아니다.**

서비스별 MySQL과 RDS 구조가 제공하는 장점을 확인했지만, 프로젝트 규모에서는 이를 유지하는 비용도 Architecture Decision의 중요한 조건이었습니다. 따라서 최종 AWS 환경에서는 물리적 Infrastructure를 통합하면서 서비스별 논리적 데이터 경계는 유지했습니다.

**4. Architecture의 Trade-off는 다음 검증 과제가 된다.**

RDS 통합으로 비용을 줄이는 대신 장애 영향 범위와 Connection Capacity가 하나의 Instance에 집중되었습니다. 이를 설계상의 단점으로만 남겨두지 않고 Multi-AZ Failover와 Connection Capacity 검증의 후속 과제로 연결했습니다.
