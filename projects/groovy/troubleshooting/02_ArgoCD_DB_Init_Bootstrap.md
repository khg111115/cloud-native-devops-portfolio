# Argo CD PreSync DB-init Bootstrap Failure

## 1. Overview

Groovy 프로젝트에서 서비스별 MySQL Database와 Application User를 자동으로 생성하기 위해 DB-init Job을 구성했습니다.
> **Note:** 이 사례는 Groovy 프로젝트의 Kubernetes/Argo CD 기반 배포 구조를 구성하던 중 발생한 트러블슈팅 기록입니다. 이후 프로젝트 아키텍처가 변경되면서 해당 DB-init 방식은 최종 구성에서 제거되었으며, 본 문서는 당시 Bootstrap 의존성 문제를 발견하고 해결한 과정을 다룹니다.

DB-init Job에는 Argo CD `PreSync` Hook이 적용되어 있었지만, 최초 Bootstrap 구조를 검토하는 과정에서 **DB-init이 의존하는 MySQL과 Secret보다 먼저 실행되는 의존성 문제가 발생할 수 있음**을 발견했습니다.

실제 Argo CD 환경에서 최초 Bootstrap을 재현한 결과 DB-init Pod에서 다음 오류가 발생했습니다.

~~~text
FailedMount:
secret "db-init-sql" not found
~~~

동시에 DB-init이 필요로 하는 MySQL StatefulSet과 Service 역시 아직 생성되지 않은 상태였습니다.

원인을 분석한 결과 DB-init 내부의 MySQL Connection Retry 문제가 아니라, **Argo CD Hook 실행 단계와 Kubernetes Resource의 실제 의존관계가 일치하지 않는 Deployment Ordering 문제**였습니다.

이에 DB-init을 `PreSync`에서 제거하고 Sync Wave를 이용하여 선행 리소스 이후 처리되도록 변경했습니다.

~~~text
Before

PreSync
└─ DB-init
      ↓
   MySQL / Secret 필요
      ↓
   아직 Sync되지 않음


After

Sync wave 0
├─ MySQL
├─ Secret
├─ Service
└─ PVC
      ↓
Sync wave 1
└─ DB-init
      ↓
MySQL Connection Retry
      ↓
DB 초기화
~~~

---

## 2. DB-init Architecture

MSA 전환 이후 Identity, Study, Content, Calendar, Notification 서비스별 MySQL을 구성하면서 각 서비스에서 사용할 Database와 Application User를 자동으로 생성하기 위해 5개의 DB-init Job을 사용했습니다.

~~~text
identity-service-db-init
study-service-db-init
content-service-db-init
calendar-service-db-init
notification-service-db-init
~~~

DB 초기화의 기본 흐름은 다음과 같습니다.

~~~text
MySQL Resource 생성
        ↓
MySQL Process 시작
        ↓
Provisioner 계정 준비
        ↓
DB-init Job 실행
        ↓
서비스별 Database 생성
        ↓
Application User 생성
~~~

DB-init Job에는 MySQL 프로세스가 실제 Connection을 받을 준비가 되기 전에 SQL을 실행하지 않도록 Connection Retry 로직도 포함되어 있었습니다.

~~~text
MySQL Connection 확인
        ↓
실패
        ↓
3초 대기
        ↓
재시도
        ↓
최대 60회
        ↓
Connection 성공
        ↓
DB-init SQL 실행
~~~

따라서 MySQL 프로세스의 시작이 늦어지더라도 약 3분 동안 준비 상태를 기다릴 수 있는 구조였습니다.

---

## 3. Problem: PreSync와 Resource Dependency 충돌

기존 DB-init Job에는 다음과 같은 Argo CD Hook이 설정되어 있었습니다.

~~~yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
~~~

`PreSync` DB-init을 먼저 실행한 뒤 Application을 배포한다는 의도였지만, DB-init 자체가 실행되기 위해서는 MySQL과 관련 Secret이 먼저 필요했습니다.

반면 해당 리소스들은 일반 Sync 단계에서 처리되는 구조였습니다.

~~~text
Argo CD Sync 시작
        ↓
PreSync
        ↓
DB-init Job
   ┌────┴────┐
   ↓         ↓
MySQL 필요   Secret 필요
   ↓         ↓
아직 없음    아직 없음

MySQL / Secret
        ↑
일반 Sync 단계
        ↑
PreSync 완료 필요
~~~

즉 DB-init은 MySQL과 Secret을 필요로 하지만, MySQL과 Secret이 처리되는 일반 Sync 단계로 진행하기 위해서는 먼저 PreSync가 완료되어야 하는 의존성 충돌이 존재했습니다.

---

## 4. Why Connection Retry Could Not Solve It

처음에는 DB-init 내부에 이미 MySQL Connection Retry가 구현되어 있었기 때문에 MySQL이 늦게 시작되더라도 문제가 없을 수 있다고 판단할 수 있었습니다.

하지만 분석 과정에서 **Process Readiness와 Resource Creation은 서로 다른 문제**라는 점을 구분했습니다.

Retry가 해결할 수 있는 상황은 다음과 같습니다.

~~~text
MySQL Resource 존재
        ↓
MySQL Pod 시작
        ↓
MySQL Process 초기화 중
        ↓
Connection 실패
        ↓
DB-init Retry
        ↓
MySQL Ready
        ↓
Connection 성공
~~~

반면 PreSync 문제에서는 MySQL이 느리게 시작되고 있는 것이 아니라, 일반 Sync 단계가 시작되지 않아 **MySQL Resource 자체가 아직 처리되지 않은 상태**였습니다.

~~~text
PreSync DB-init 실행
        ↓
MySQL Connection 대기
        ↓
MySQL은 일반 Sync 단계
        ↓
PreSync가 완료되지 않음
        ↓
일반 Sync 진행 불가
        ↓
MySQL Resource 미생성
~~~

따라서 Retry 횟수나 대기 시간을 늘리는 것으로 해결할 수 있는 문제가 아니라고 판단했습니다.

---

## 5. Failure Reproduction

설정만 보고 추측한 상태에서 바로 변경하지 않고, 별도의 테스트용 Argo CD Application과 Namespace를 구성하여 최초 Bootstrap 상황을 재현했습니다.

Sync를 수행한 결과 DB-init Job은 생성되었지만 Pod가 정상적으로 초기화되지 않았습니다.

~~~text
DB-init Jobs
→ 생성

DB-init Pods
→ Init:0/1

MySQL StatefulSets
→ 생성되지 않음

MySQL Services
→ 생성되지 않음
~~~

DB-init Pod의 상태를 확인한 결과 다음 오류가 발생하고 있었습니다.

~~~text
FailedMount:
secret "db-init-sql" not found
~~~

DB-init Job은 PreSync 단계에서 이미 처리되고 있었지만, Job이 필요로 하는 Secret은 아직 일반 Sync 단계에서 처리되지 않은 상태였습니다.

~~~text
PreSync
        ↓
DB-init Job
        ↓
db-init-sql Secret 필요
        ↓
Secret 없음
        ↓
FailedMount

동시에

MySQL / Secret
        ↓
일반 Sync 단계 대기
~~~

실제 재현 결과가 앞서 분석한 Resource Dependency 구조와 일치함을 확인했습니다.

---

## 6. Deployment Ordering Redesign

문제를 해결하기 위해 DB-init의 실제 Resource Dependency를 기준으로 배포 순서를 다시 설계했습니다.

DB-init이 실행되기 위해서는 먼저 다음 리소스가 처리되어 있어야 합니다.

~~~text
Secret
Service
PVC
StatefulSet
        ↓
MySQL Resource
        ↓
DB-init Job
~~~

따라서 DB-init을 일반 리소스보다 먼저 처리하는 `PreSync` 구조에서 제거하고, **동일한 Sync 단계에서 Sync Wave를 이용해 처리 순서를 분리**했습니다.

~~~text
Before

PreSync
└─ DB-init

Sync
├─ MySQL
├─ Service
├─ PVC
└─ Secret


After

Sync wave 0
├─ MySQL
├─ Service
├─ PVC
└─ Secret

        ↓

Sync wave 1
└─ DB-init
~~~

서비스별 DB-init Job의 Annotation도 이에 맞게 변경했습니다.

### Before

~~~yaml
annotations:
  argocd.argoproj.io/hook: PreSync
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
~~~

### After

~~~yaml
annotations:
  argocd.argoproj.io/hook: Sync
  argocd.argoproj.io/sync-wave: "1"
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
~~~

동일한 변경을 5개 서비스의 DB-init Job에 적용했습니다.

---

## 7. Sync Wave와 Connection Retry의 역할 분리

Deployment Ordering을 수정한 뒤에도 기존 MySQL Connection Retry 로직은 유지했습니다.

Sync Wave를 통해 Kubernetes Resource의 처리 순서를 제어하더라도, **MySQL Resource가 처리되는 시점과 MySQL Process가 실제 Connection을 받을 수 있는 시점은 동일하지 않을 수 있기 때문**입니다.

따라서 두 장치의 역할을 다음과 같이 분리했습니다.

| Mechanism | Responsibility |
| --- | --- |
| Argo CD Sync Wave | Kubernetes Resource 처리 순서 제어 |
| DB-init Connection Retry | MySQL Process의 실제 Connection 준비 상태 대기 |

최종 구조는 다음과 같습니다.

~~~text
Sync wave 0
MySQL / Secret / Service / PVC
        ↓
Sync wave 1
DB-init Job
        ↓
MySQL Connection 가능?
   ├─ NO → 3초 후 재시도
   └─ YES
        ↓
DB-init SQL 실행
~~~

즉, **Resource Dependency와 Runtime Readiness를 서로 다른 문제로 구분하여 각각의 메커니즘으로 처리**했습니다.

---

## 8. Validation

변경 후 최초 Bootstrap 상황을 다시 확인했습니다.

Sync 시작 후에는 기존과 달리 DB-init보다 MySQL 관련 선행 리소스가 먼저 처리되는 것을 확인했습니다.

~~~text
MySQL StatefulSet
→ 생성

MySQL Service
→ 생성

PVC
→ 생성

DB-init Job
→ 아직 생성되지 않음
~~~

이후 별도의 로컬 기능 검증에서 서비스별 MySQL과 DB-init의 최종 상태를 확인했습니다.

~~~text
MySQL 5개
        ↓
Running

PVC 5개
        ↓
Bound

DB-init Job 5개
        ↓
Complete

서비스별 Database
        ↓
생성 확인
~~~

이를 통해 Argo CD 환경에서 선행 Resource의 처리 순서가 개선된 것을 확인했으며, 별도의 로컬 기능 검증을 통해 변경된 구조에서도 DB 초기화 흐름이 정상적으로 완료되는 것을 확인했습니다.

---

## 9. Result & Lessons Learned

### Result

이번 문제의 원인은 MySQL의 시작 속도나 DB-init Retry 횟수가 아니라 **Argo CD Hook 실행 단계와 DB-init의 실제 Resource Dependency가 일치하지 않는 Bootstrap Ordering 문제**였습니다.

~~~text
[기존]

DB-init PreSync
        ↓
MySQL / Secret 필요
        ↓
일반 Sync Resource
        ↓
PreSync 완료 필요
        ↓
Bootstrap 진행 불가


[개선]

Sync wave 0
MySQL / Secret / Service / PVC
        ↓
Sync wave 1
DB-init
        ↓
MySQL Runtime Readiness 확인
        ↓
DB 초기화
~~~

### Lessons Learned

#### 1. Retry는 모든 Dependency 문제를 해결하지 않는다

Retry는 이미 존재하는 서비스가 아직 준비되지 않은 상황에는 사용할 수 있지만, 의존하는 Resource 자체가 생성되지 않는 Deployment Ordering 문제를 해결할 수는 없습니다.

따라서 단순히 Timeout이나 Retry 횟수를 늘리기 전에 **대상 Resource가 존재하는지, 존재하지만 아직 Ready하지 않은지를 먼저 구분**해야 한다는 점을 확인했습니다.

#### 2. 선언된 배포 순서보다 실제 Resource Dependency를 기준으로 설계해야 한다

DB-init을 Application보다 먼저 수행한다는 목적만 보면 PreSync가 자연스러워 보일 수 있지만, DB-init 역시 MySQL과 Secret이라는 선행 Resource를 필요로 했습니다.

따라서 GitOps 배포 순서는 작업의 이름이나 목적이 아니라 **실제 Resource Dependency를 기준으로 구성해야 한다**는 점을 확인했습니다.

#### 3. Resource Ordering과 Runtime Readiness는 별도로 처리해야 한다

이번 수정에서는 Sync Wave만 적용하고 기존 Retry를 제거하지 않았습니다.

Sync Wave는 Resource 처리 순서를 제어하고, Connection Retry는 MySQL Process의 Runtime Readiness를 확인하도록 각각의 책임을 분리했습니다.

이를 통해 하나의 메커니즘으로 모든 문제를 해결하려 하기보다 **Deployment Ordering과 Runtime Readiness를 서로 다른 계층의 문제로 분리하여 처리**할 수 있었습니다.