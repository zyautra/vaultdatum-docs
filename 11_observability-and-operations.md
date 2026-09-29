# Observability and Operations

## 1. 문서 목적

이 문서는 Sync System을 실제로 운영할 때 필요한 최소한의 상태 확인, 로그, 진단, 복구 절차를 정의한다.

이 문서는 운영 절차와 진단 기준의 단일 기준이다. 내부 recovery 구현은 [03 Server Architecture](./03_server-architecture.md)와 [08 Persistence Design](./08_persistence-design.md), health/status API의 wire contract는 [07 API Specification](./07_api-specification.md), 배포 접근 경계는 [09 Security and Deployment](./09_security-and-deployment.md)을 따른다.

초기 시스템은 개인 사용자를 위한 단일 Server를 전제로 한다.

따라서 대규모 Monitoring Platform을 구축하지 않는다.

운영 목표는 다음과 같다.

```text
문제가 발생했을 때
왜 동기화가 멈췄는지 알 수 있다.

Server가 안전하게 시작되었는지 알 수 있다.

Client가 어디까지 동기화되었는지 알 수 있다.

Recovery가 필요한 상태를 식별할 수 있다.
```

---

# 2. 운영 원칙

Observability는 Sync Correctness를 대신하지 않는다.

예:

```text
Notification 누락

Log 누락

Metric 누락
```

이 발생해도 동기화 자체가 잘못되어서는 안 된다.

Observability는:

```text
설명 가능성

진단 가능성

운영 편의성
```

을 위한 것이다.

---

# 3. 필요한 Observability 수준

다음 세 가지로 충분하다.

```text
Structured Log

Health Check

Current Sync Status
```

다음은 초기 필수 기능이 아니다.

```text
Prometheus

Grafana

Distributed Tracing

OpenTelemetry

Centralized Log Platform
```

필요해질 때 추가한다.

---

# 4. Server Log

Server는 주요 상태 전이를 Structured Log로 기록한다.

예:

```text
server_start

recovery_start

recovery_complete

operation_prepared

operation_committed

operation_conflict

external_drift_detected

reconciliation_start

reconciliation_complete

server_ready

authentication_rejected

token_rotated

realtime_ticket_rejected
```

authentication event에는 outcome만 기록할 수 있으며, Authorization header, Vault token,
realtime ticket, Vault content는 기록하지 않는다. wrong token은 enumeration을 막기 위해
raw token 또는 유사한 식별자를 남기지 않는다.

---

# 5. Mutation Log

Mutation 처리 시 최소 다음 정보를 남길 수 있다.

```text
operationId

clientId

operationType

path

resultRevision

result
```

예:

```text
operation=OP-123
client=C-1
type=MODIFY
path=notes/a.md
result=COMMITTED
revision=501
```

파일 Content 자체는 기록하지 않는다.

Vault token은 단일 Vault access secret이므로 audit correlation key로 쓰지 않는다.

---

# 6. Conflict Log

Conflict 발생 시:

```text
operationId

clientId

path

baseRevision

currentRevision

conflictType
```

정도를 기록한다.

예:

```text
path=notes/a.md

baseRevision=100

serverRevision=105

result=BASE_STATE_MISMATCH
```

Hash는 필요하면 진단 수준 Log에서 기록할 수 있다.

---

# 7. Recovery Log

Server Startup Recovery는 반드시 명확하게 보이도록 한다.

예:

```text
Recovery started

Found PREPARED operation OP-10

Filesystem state matches afterState

Finalized revision 200

Recovery completed
```

Recovery가 자동으로 완료되지 못하면:

```text
RECOVERY_REQUIRED
```

상태를 명확히 기록한다.

---

# 8. Client Log

Client 역시 중요한 Sync Event를 기록할 수 있다.

예:

```text
sync_start

pull_start

pull_complete

pending_push

push_committed

conflict_created

remote_apply

full_reconciliation

sync_complete
```

---

# 9. Client Log 수준

기본 사용자에게 너무 많은 Log를 노출하지 않는다.

개념적으로:

```text
INFO

DEBUG
```

정도면 충분하다.

기본은 INFO를 사용한다.

문제 진단 시 DEBUG를 활성화할 수 있다.

---

# 10. Client Sync Status

사용자는 현재 Client 상태를 쉽게 확인할 수 있어야 한다.

최소 상태:

```text
Synced

Syncing

Offline

Pending

Conflict

Error
```

이 상태는 기존 Client Architecture의 Sync State를 그대로 반영한다.

---

# 11. Status Detail

필요하면 상세 화면에서 다음 정보를 제공한다.

```text
Server Connection

Server Cursor

Server Current Revision

Pending Operation Count

Conflict Count

Last Successful Sync

Last Error
```

예:

```text
Status: Pending

Server Revision: 502
Client Cursor: 502

Pending: 2
Conflicts: 0

Last Sync:
2026-09-13 01:20
```

---

# 12. Cursor Lag

다음 값은 유용한 진단 정보다.

```text
serverCurrentRevision - serverCursor
```

예:

```text
Server = 1000

Client Cursor = 995

Lag = 5
```

이는 Client가 Server Change를 5개 아직 처리하지 않았다는 뜻이다.

하지만 Lag가 0이라고 반드시 Synced는 아니다.

```text
Pending

Conflict
```

가 있을 수 있기 때문이다.

---

# 13. Pending 상태

Pending Operation이 있다면 사용자에게 최소 개수 정도는 보여줄 수 있다.

예:

```text
2 local changes waiting to sync
```

각 Operation ID까지 기본 UI에서 보여줄 필요는 없다.

진단 화면에서는 확인할 수 있다.

---

# 14. Conflict 상태

Conflict가 하나 이상 존재하면 일반적인 Synced 상태로 표시하지 않는다.

예:

```text
Synced with conflicts
```

보다 명확하게:

```text
Conflict (2)
```

로 표시하는 편이 좋다.

다른 Path가 모두 정상 동기화되어도 Conflict가 해결되지 않았다는 사실은 사용자에게 보여야 한다.

---

# 15. Last Successful Sync

Client는 마지막으로 정상 Sync Cycle이 완료된 시각을 저장할 수 있다.

이는 correctness에 사용하지 않는다.

사용자 진단 용도다.

예:

```text
Last successful sync:
10 minutes ago
```

---

# 16. Last Error

Client는 마지막 Sync Error를 간단히 보존할 수 있다.

예:

```text
Network unavailable

Server unavailable

Authentication failed

Recovery required
```

Network Timeout 같은 일시적인 문제와 Conflict를 동일한 Error로 표현하지 않는다.

---

# 17. Health Endpoint

Server는 다음 Endpoint를 제공한다.

```text
GET /health/live

GET /health/ready
```

---

## 17.1 Liveness

Process 자체가 정상적으로 실행 중인지를 확인한다.

예:

```json
{
  "status": "ok"
}
```

---

## 17.2 Readiness

Server가 Sync 요청을 받을 준비가 되었는지를 나타낸다.

다음이 완료된 후 Ready다.

```text
Storage Open

Database Open

Migration

Recovery

State Validation
```

---

# 18. Not Ready 상태

다음 상태에서는 Server Process가 살아 있어도 Ready가 아니다.

```text
Startup Recovery 진행 중

Database Open 실패

Vault 접근 실패

RECOVERY_REQUIRED

Instance Lock 실패
```

---

# 19. Server Status Endpoint

개인 운영 편의를 위해 별도의 Status Endpoint를 둘 수 있다.

예:

```text
GET /api/v1/status
```

Response 예:

```json
{
  "vaultId": "V-123",
  "currentRevision": 502,
  "status": "READY"
}
```

복잡한 내부 정보를 모두 노출할 필요는 없다.

---

# 20. Server Startup

정상 Startup 흐름은 다음과 같다.

```text
Process Start
    ↓
Open Persistent Storage
    ↓
Schema Migration
    ↓
Acquire Instance Lock
    ↓
Recover PREPARED Operations
    ↓
Validate Vault State
    ↓
READY
```

각 주요 단계는 Log로 확인 가능해야 한다.

---

# 21. Server Shutdown

정상 종료에서는:

```text
새 Mutation 수락 중단

진행 중 Operation 정리

SQLite Transaction 완료

Database Close
```

를 수행한다.

하지만 강제 종료를 항상 피할 수 있는 것은 아니므로 Crash Recovery가 correctness의 핵심이다.

---

# 22. Server Update

Application Version Update는 다음 순서로 수행한다.

```text
Stop Server

Backup 확인

Start New Version

Schema Migration

Recovery

Readiness 확인
```

Container Image 교체가 `/data` 삭제를 의미해서는 안 된다.

---

# 23. Database Migration

Schema Migration 실패 시 Server는 Sync 요청을 받지 않는다.

예:

```text
Migration Failed
    ↓
NOT READY
```

Migration Error를 무시하고 이전 Schema로 계속 실행하지 않는다.

---

# 24. Server Vault 직접 수정

서버 Vault는 Sync API를 통해서만 변경한다. SSH, Editor, IDE Workspace, Script, Filesystem Sync Tool 등으로 Server가 관리하는 Vault를 직접 수정하는 것은 지원하지 않는다. 쓰기 경계는 [03 Server Architecture](./03_server-architecture.md)의 26절을 단일 기준으로 사용한다.

콘텐츠를 추가하거나 바꾸려면 Client나 API 기반 도구로 Operation을 제출한다. 서버 Vault 디렉터리를 Editor Workspace에 넣거나 파일을 직접 복사하는 방식은 운영 절차로 사용하지 않는다. 유일한 예외는 새 서버로 이전할 때의 Initial Vault Import이다.

## 24.1 다른 저장소에서 이전하기

기존 Vault를 VaultDatum으로 옮길 때는 서버 복제만으로 초기 상태를 만든다. 규칙은 [03 Server Architecture](./03_server-architecture.md)의 26.4절을 단일 기준으로 사용한다.

```text
1. 비어 있는 새 Data Root를 준비한다

2. 기존 Vault 내용을 /data/vault 아래에 복사한다
   (.obsidian/, .git/ 등 이름이 `.`으로 시작하는 항목, Symlink,
    최대 Content 크기를 넘는 파일은 제외한다)

3. VAULTDATUM_INITIAL_IMPORT=true로 서버를 시작한다

4. 시작이 거부되면 로그에 보고된 경로를 정리하고 3을 반복한다

5. 가져온 Change 수와 Current Revision을 로그와 /api/v1/vault로 확인한다

6. 플래그를 끄고 서버를 다시 시작한다

7. Client를 연결한다
```

가져오기는 하나의 Transaction이므로 실패하면 아무것도 기록되지 않는다. 플래그를 켠 채 이미 Revision이 있는 서버를 시작하면 시작이 거부되므로, 가져오기가 끝나면 반드시 플래그를 끈다.

기존 Local Vault를 가진 Client는 일반 Initial Bootstrap을 따른다. 서버와 같은 파일은 Replica로 기록되고, 내용이 다른 파일은 Conflict가 되며 어느 쪽도 덮어쓰지 않는다.

로그에는 경로, 사유, 개수, Revision만 기록하고 Content는 기록하지 않는다.

---

# 25. Integrity Scan

Integrity Scan은 최소 다음을 비교한다.

```text
Vault Filesystem

Path State
```

차이:

```text
Expected Present / Actual Missing

Expected Hash / Actual Hash mismatch

Unknown File discovered

Entry Type mismatch
```

등을 식별한다.

Integrity Scan은 감지와 보고만 한다. Journal, Path State, Vault Filesystem을 변경하지 않는다. Server Startup과 운영자 요청 시 수행하며, 주기 실행은 선택이다.

---

# 26. Vault Drift 처리

Drift를 발견하면 `external_drift_detected` Event로 경로와 차이 종류를 기록한다. Content는 기록하지 않는다.

Drift는 새 Server Change로 편입하지 않는다. Drift가 있는 경로를 대상으로 하는 Mutation은 PREPARED Operation을 만들기 전에 `RECOVERY_REQUIRED`로 거부된다. 다른 경로의 동기화는 계속된다.

운영자는 다음 중 하나로 해소한다.

```text
직접 수정이 실수였다
  → Filesystem을 Path State가 기록한 내용으로 되돌린다

직접 수정한 내용을 보존해야 한다
  → 내용을 Vault 밖으로 옮기고 Path State가 기록한 상태로 되돌린 뒤
    Sync API Operation으로 다시 제출한다
```

해소한 뒤 Integrity Scan을 다시 실행하여 Drift가 없는지 확인한다.

Backup 복원은 34절에 따라 Vault와 Sync State를 같은 시점의 Snapshot으로 함께 복원한다. Vault만 따로 복원하면 Drift가 생긴다.

---

# 27. Manual Reconciliation

운영자는 필요하면 Server Integrity Scan 또는 Client Full Reconciliation을 수동으로 실행할 수 있어야 한다.

Client:

```text
Sync Now

Full Reconciliation
```

Server:

```text
Integrity Scan
```

정도의 기능이면 충분하다.

---

# 28. Client Recovery

Client에서 Sync State가 이상하다고 판단되면 일반적인 순서는 다음과 같다.

```text
1. Local Vault 보존

2. Pending / Conflict 확인

3. Full Reconciliation

4. 필요하면 Sync State Reset
```

Local Vault를 먼저 삭제하거나 Server Vault로 무조건 덮어쓰지 않는다.

---

# 29. Client State Reset

Client Sync State Reset은 마지막 수단이다. 사용자가 보는 reset의 순서, 경고,
일반 reset과 connection settings reset의 구분은
[12 Client User Experience](./12_client-user-experience.md)를 따른다.

Reset 대상:

```text
Replica Index

Cursor

Apply Journal
```

등이다.

Pending 또는 Conflict가 존재하면 사용자의 변경을 잃을 수 있으므로 명시적인 경고 없이 Reset하지 않는다.

일반 reset은 Local Vault 또는 Server Vault의 파일을 삭제하거나 덮어쓰지 않고,
이후 Server-first Bootstrap으로 다시 분류해야 한다. 표준 사용자 절차가 browser
developer tools나 IndexedDB 이름을 요구해서는 안 된다.

---

# 30. Server Recovery Required

Server가 스스로 안전한 상태를 결정할 수 없는 경우 임의로 Write를 계속하지 않는다.

예:

```text
Operation Before Hash = AAA

Expected After Hash = BBB

Actual File Hash = CCC
```

이면:

```text
RECOVERY_REQUIRED
```

로 전환한다.

---

# 31. Recovery Required 상태

이 상태에서는:

```text
Readiness = false
```

로 두는 것이 기본이다.

운영자가 문제를 확인하기 전 새로운 Mutation을 받지 않는다.

---

# 32. Backup

Backup은 Sync System과 별도 기능이다.

최소 Backup 대상:

```text
/data/vault

/data/state
```

이다.

---

# 33. Backup 주기

개인 시스템에서는 복잡한 정책을 강제하지 않는다.

예:

```text
Daily

Weekly
```

등 사용자가 원하는 방식으로 운영할 수 있다.

중요한 것은 Sync가 Backup을 대체하지 않는다는 점이다.

---

# 34. Backup Consistency

실행 중 Backup을 수행한다면 Vault와 SQLite가 서로 다른 시점의 상태가 될 수 있음을 고려해야 한다.

가장 단순한 안전한 방법은:

```text
Server Stop

Backup

Server Start
```

이다.

---

# 35. Disk Usage

주요 Disk 사용처:

```text
Vault

SQLite

Staging

Recovery

Manifest

Client Conflict Artifact
```

개인용 Vault에서는 상세 Metric보다 Disk가 가득 차는 상황만 피하면 충분하다.

Server Host의 일반 Disk Monitoring을 활용할 수 있다.

---

# 36. Staging / Recovery Cleanup

정상 완료된 Operation의 Staging / Recovery Artifact는 정리한다.

Server Startup 시 Orphan Artifact도 검사할 수 있다.

단:

```text
정체를 알 수 없는 Artifact
```

를 즉시 삭제하지 않는다.

Recovery가 필요한 데이터일 수 있기 때문이다.

---

# 37. Manifest Cleanup

만료된 Manifest는 안전하게 제거할 수 있다.

Manifest는 temporary reconciliation metadata이며 Authoritative Change History가 아니다.

---

# 38. Log Rotation

Server Log를 File로 저장한다면 무한히 증가하지 않도록 Rotation을 사용한다.

Container 환경에서는 가능하면:

```text
stdout / stderr
```

로 출력하고 Container Runtime 또는 Host가 Log Rotation을 담당하게 할 수 있다.

---

# 39. User-facing Diagnostics

사용자가 문제를 신고하거나 스스로 진단할 때 최소 다음 정보를 복사할 수 있으면 유용하다.

```text
Client Version

Server Version

Vault ID

Client ID

Server Cursor

Server Current Revision

Pending Count

Conflict Count

Last Error
```

파일 Content는 포함하지 않는다.

---

# 40. Operations Checklist

정상 운영 시 다음 정도만 확인할 수 있으면 충분하다.

```text
Server가 Ready인가?

Client가 Server에 연결되는가?

Client Cursor가 진행하는가?

Pending이 계속 쌓이고 있지 않은가?

Conflict가 존재하는가?

Server Recovery Error가 있는가?

시작 로그에 external_drift_detected가 있는가?

Disk가 가득 차지 않았는가?
```

## 40.1 Public Token Operations

`public-token` instance를 Internet에 열기 전에는 다음을 확인한다.

```text
Server profile = public-token

Vault token이 read-only Secret file로 mount됨

Public Gateway의 TLS certificate와 hostname이 유효함

Server Service는 ClusterIP이며 Gateway 외 source가 NetworkPolicy로 차단됨

HTTP와 direct Server port는 public exposure에 없음

Gateway / Server log redaction이 Authorization과 ticket을 기록하지 않음
```

token rotation은 새 random token으로 Kubernetes Secret을 교체하고 Pod를 restart해
수행한다. token은 생성 시 한 번만 사용자에게 전달하며 ticket, Secret file, database
backup을 terminal log나 support bundle에 붙이지 않는다. rotation 뒤 이전 token으로 새
Client HTTP request가 `401`을 받는지 확인하고, 관련 장치의 Pending 상태를 확인한 뒤
새 token을 입력한다.

---

# 41. Observability Invariants

## Invariant 1 — Logs Are Not Required for Correctness

Log가 없어도 Sync Protocol은 정상 동작해야 한다.

---

## Invariant 2 — Content Is Not Logged

사용자의 파일 Content 전체를 일반 운영 Log에 기록하지 않는다.

---

## Invariant 3 — Ready Means Safe to Serve

Recovery가 완료되지 않은 Server를 Ready로 표시하지 않는다.

---

## Invariant 4 — Conflict Is Visible

Conflict가 존재하는 Client를 일반적인 Synced 상태로 표시하지 않는다.

---

## Invariant 5 — Cursor Is Diagnostic, Not Complete Sync Status

Cursor가 Server Revision과 같더라도 Pending 또는 Conflict가 있을 수 있다.

---

## Invariant 6 — Recovery Uncertainty Stops Mutation

Server가 Authoritative State를 안전하게 판단할 수 없으면 임의로 Mutation을 계속하지 않는다.

---

# 42. 최종 운영 모델

```text
                    Server

                 Structured Log
                       │
                       ▼
              ┌─────────────────┐
              │ Sync Server     │
              │                 │
              │ revision = 502  │
              │ READY           │
              └────────┬────────┘
                       │
                 /health/ready


                    Client

              ┌─────────────────┐
              │ Status          │
              │                 │
              │ Synced          │
              │ Cursor: 502     │
              │ Pending: 0      │
              │ Conflict: 0     │
              └─────────────────┘
```

이 운영 모델의 핵심은:

> **복잡한 Monitoring Infrastructure보다 현재 Sync 상태와 실패 원인을 사람이 빠르게 이해할 수 있도록 만드는 것**

이다.
