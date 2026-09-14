# Testing Strategy

## 1. 문서 목적

이 문서는 Sync System의 correctness를 검증하기 위한 테스트 전략을 정의한다.

테스트의 기대 결과는 제품 요구사항·프로토콜·전문 설계 문서에서 가져온다. 본 문서는 이를 별도의 동작 명세로 다시 정의하지 않고, [00 Product Specification](./00_product-specification.md), [02 Synchronization Protocol](./02_synchronization-protocol.md), [05 Conflict Resolution](./05_conflict-resolution.md), [03 Server Architecture](./03_server-architecture.md), [04 Client Architecture](./04_client-architecture.md)의 규칙을 검증 가능한 시나리오로 매핑한다.

이 프로젝트에서 중요한 것은 개별 함수의 완벽한 테스트 커버리지가 아니다.

더 중요한 것은 실제 환경에서:

```text
Local File

Client Sync State

Network

Server API

SQLite

Server Vault
```

가 함께 동작했을 때 데이터가 안전하게 수렴하는지 확인하는 것이다.

따라서 테스트의 중심은:

> **End-to-End Scenario와 Failure Recovery**

에 둔다.

---

# 2. 테스트 철학

이 시스템은 정상적인 CRUD보다 실패 상황에서의 동작이 중요하다.

주요 검증 대상은 다음과 같다.

```text
Offline → Reconnect

Concurrent Change

Conflict

Network Response Loss

Server Crash

Client Crash

Delete Resurrection

Missed Notification

Full Reconciliation
```

테스트 코드 자체가 제품보다 복잡해지지 않도록 한다.

다음은 MVP에서 지양한다.

```text
과도한 Mock 기반 Unit Test

별도 Reference Sync Engine

대규모 Model-based Test Framework

무작위 Property-based Test Infrastructure

모든 내부 Class에 대한 개별 Test
```

---

# 3. 테스트 구조

MVP의 테스트는 세 Layer로 제한한다.

```text
1. Small Unit Tests

2. Integration Tests

3. End-to-End Tests
```

비중은 다음과 같다.

```text
Unit
    낮음

Integration
    중간

E2E
    높음
```

---

# 4. Small Unit Tests

Unit Test는 외부 상태와 관계없는 순수 로직에만 사용한다.

대상 예:

```text
Path Validation

Path Normalization

Base Condition Validation

Conflict Classification

Pending Operation Coalescing

Hash Comparison
```

예:

```text
Replica = AAA
Local   = BBB

→ MODIFY
```

또는:

```text
Server = DELETED / revision 10

CREATE
base = UNKNOWN

→ BASE_STATE_MISMATCH
```

같은 결정 로직이다.

---

## 4.1 Unit Test에서 하지 않는 것

다음과 같은 구조를 만들기 위해 많은 Mock을 사용하지 않는다.

```text
Mock Repository

Mock HTTP Client

Mock Filesystem

Mock Scheduler

Mock Sync Coordinator
```

이런 부분은 실제 Component를 연결한 Integration 또는 E2E Test에서 검증한다.

---

# 5. Integration Tests

Integration Test는 실제 Persistence와 API Boundary를 검증한다.

주요 대상은 두 가지다.

```text
Server Persistence

Client Persistence
```

---

## 5.1 Server Integration

실제:

```text
Temporary Vault

SQLite Database

Staging Directory

Recovery Directory
```

를 사용한다.

Mutation 후 다음 상태가 일치하는지 확인한다.

```text
Vault Filesystem

Path State

Change Journal

Operation State
```

예:

```text
MODIFY A.md
```

성공 후:

```text
Filesystem = BBB

Path State = BBB

Change Journal = new revision

Operation = COMMITTED
```

이어야 한다.

---

## 5.2 Client Integration

실제 Client Persistence Backend를 사용하여:

```text
Replica Index

Pending

Apply Journal

Conflict

Cursor
```

가 restart 이후에도 유지되는지 확인한다.

특히:

```text
Cursor Advance

+

Replica / Conflict Update
```

가 하나의 durable state transition으로 처리되는지 검증한다.

---

# 6. End-to-End Environment

E2E Test는 가능한 한 실제 구조를 사용한다.

```text
Client A
    │
    ▼
HTTP
    │
    ▼
Real Server
    │
    ├── SQLite
    └── Temporary Vault
```

Multi-client Test에서는:

```text
Client A

Client B

Client C
```

를 같은 Server에 연결한다.

Test마다 독립적인 Temporary Data Root를 사용한다.

---

# 7. Basic Sync Scenarios

가장 기본적인 동작을 검증한다.

```text
CREATE

MODIFY

DELETE

RENAME
```

예:

```text
Client A CREATE A.md
        ↓
Server Commit
        ↓
Client B Pull
        ↓
Client B A.md 생성
```

최종적으로:

```text
Server

Client A

Client B
```

가 같은 상태로 수렴해야 한다.

---

# 8. Offline Scenarios

Offline 사용은 정상적인 동작으로 취급한다.

반드시 다음을 검증한다.

---

## 8.1 Offline Modify

```text
Client A Offline

A.md 수정

App 종료

App 재시작

Reconnect
```

후 Local Change가 Server로 정상 Commit되어야 한다.

---

## 8.2 Long Offline Client

Client가 오래 Offline인 동안 Server에서 여러 Revision이 발생할 수 있다.

Reconnect 후:

```text
Incremental Sync

또는

Full Reconciliation
```

으로 정상 수렴해야 한다.

---

## 8.3 Notification Loss

Notification을 하나도 받지 못한 Client도:

```text
Reconnect

Resume

Manual Sync
```

시 Change Stream을 조회하여 정상 수렴해야 한다.

Notification은 correctness 조건이 아님을 검증한다.

---

# 9. Conflict Scenarios

Conflict는 반드시 E2E로 검증한다.

---

## 9.1 Modify vs Modify

초기:

```text
A.md = AAA
```

Client A:

```text
Offline

A.md = BBB
```

Client B:

```text
A.md = CCC

Server Commit
```

A Reconnect 결과:

```text
Server = CCC

Client A Local = BBB

Conflict 존재
```

이어야 한다.

어느 쪽도 자동으로 삭제되어서는 안 된다.

---

## 9.2 Delete vs Modify

Server에서:

```text
DELETE A.md
```

가 Commit된 동안 Offline Client가:

```text
MODIFY A.md
```

했다면 Reconnect 시:

```text
DELETE vs MODIFY Conflict
```

가 되어야 한다.

---

## 9.3 Create vs Create

두 Client가 Offline 상태에서 같은 Path를 서로 다른 Content로 생성한다.

먼저 Commit한 Client는 성공한다.

두 번째 Client는:

```text
CREATE
base = UNKNOWN
```

이 현재 Server PRESENT State와 충돌하면서 Conflict가 되어야 한다.

---

## 9.4 Rename Conflict

한 Client가:

```text
A.md → B.md
```

한 동안 다른 Offline Client가:

```text
A.md 수정
```

했다면 자동 overwrite하지 않는다.

Conflict로 처리한다.

---

## 9.5 Conflict Isolation

다음 Change가 있다고 하자.

```text
101 A.md Conflict

102 B.md Modify

103 C.md Create
```

A.md에서 Conflict가 발생해도 B.md와 C.md는 계속 처리되어야 한다.

결과:

```text
serverCursor = 103

A.md = Conflict
```

가 가능해야 한다.

---

# 10. Delete Resurrection Prevention

가장 중요한 E2E Scenario 중 하나다.

초기:

```text
Server = A.md

Client A = A.md

Client B = A.md
```

Client B가 Offline이 된다.

Client A가:

```text
DELETE A.md
```

를 Commit한다.

Server:

```text
A.md
state = DELETED
```

상태에서 Client B가 다시 접속한다.

B의 오래된 A.md가:

```text
CREATE A.md
```

로 Server에 자동 Commit되어서는 안 된다.

Local이 변경되지 않았다면 Server Delete를 적용한다.

Local이 변경되었다면 Conflict를 생성한다.

---

# 11. Retry와 Crash Scenarios

Failure Injection은 모든 코드 지점에 넣지 않는다.

Correctness Boundary 몇 곳만 검증한다.

---

## 11.1 Lost Commit Response

Client:

```text
POST Operation
```

Server:

```text
Commit 성공
```

하지만 Response가 Client에 도착하기 전에 Network가 끊긴다.

Client가 동일:

```text
operationId
```

로 Retry하면:

```text
같은 resultRevision
```

을 받아야 한다.

새 Revision이 생성되어서는 안 된다.

---

## 11.2 Server Crash After PREPARED

```text
Operation PREPARED

Crash

Restart
```

후 Operation을 안전하게 Recovery할 수 있어야 한다.

Local Content가 유실되거나 동일 Operation이 두 번 Commit되어서는 안 된다.

---

## 11.3 Server Crash After Filesystem Apply

상태:

```text
Filesystem = New State

SQLite Operation = PREPARED
```

에서 Server가 종료된다.

Restart 후 실제 Filesystem State가 예상된 `afterState`라면 Finalize해야 한다.

새로운 중복 Revision을 생성해서는 안 된다.

---

## 11.4 Server Crash After Commit Before Response

이미:

```text
COMMITTED
```

됐지만 Client가 결과를 모르는 상태다.

Retry는 기존 결과를 반환해야 한다.

---

## 11.5 Client Crash During Remote Apply

Client가 Server Content를 적용하다 종료되는 경우를 검증한다.

대표적으로:

```text
Apply PREPARED

Local File Write

Metadata Finalize
```

사이에서 Crash를 발생시킨다.

Restart 후:

```text
Local = before
    → Apply 다시 수행

Local = after
    → Metadata Finalize

Local = unexpected
    → Conflict / Recovery
```

중 하나로 안전하게 복구해야 한다.

---

# 12. Cursor Scenarios

Cursor 오류는 Change 누락으로 이어질 수 있기 때문에 별도로 검증한다.

---

## 12.1 Own Push Does Not Jump Cursor

현재:

```text
serverCursor = 100
```

다른 Client Change:

```text
101
102
```

자신의 Push:

```text
103
```

Server Response:

```text
resultRevision = 103
```

을 받았다고 해도:

```text
serverCursor = 103
```

으로 바로 이동하면 안 된다.

101, 102, 103을 Change Stream에서 처리한 이후에만 103으로 진행한다.

---

## 12.2 Conflict Cursor Advance

```text
101 A.md Conflict
102 B.md Modify
```

인 경우 A.md Conflict를 durable하게 저장한 후 Cursor는 101을 넘을 수 있다.

B.md 처리 후:

```text
cursor = 102
```

가 된다.

---

# 13. Initial Sync와 Reconciliation

---

## 13.1 Initial Sync

빈 Client가 Server에 연결한다.

검증 흐름:

```text
Manifest

Content Download

Replica 생성

Cursor 설정

Manifest 이후 Change Pull
```

이다.

---

## 13.2 Initial Sync Race

Manifest:

```text
snapshotRevision = 100
```

생성 후 Server에:

```text
101

102
```

가 Commit되어도 Client는 Manifest 적용 이후:

```text
GET changes after 100
```

으로 따라잡아야 한다.

---

## 13.3 Full Reconciliation

다음과 같은 상태를 만든다.

```text
Replica 일부 누락

Local File 존재

Server State 존재

Pending 존재
```

Full Reconciliation이 Local File을 단순히 Server 상태로 덮어쓰지 않는지 확인한다.

---

# 14. Conflict Resolution Scenarios

최소 다음 Resolution을 E2E로 검증한다.

```text
Use Server

Apply Local

Manual Merge

Keep Deleted

Restore Local

Keep Both
```

---

## 14.1 Apply Local

Conflict:

```text
Server = BBB

Local = CCC
```

사용자가 Local을 선택하면 과거 실패 Operation을 Retry하지 않는다.

현재 Server State를 Base로:

```text
New MODIFY

base = BBB

content = CCC
```

를 만들어 Commit한다.

---

## 14.2 Resolution Race

사용자가 본 상태:

```text
Server revision = 101
```

Resolution Commit 전 Server가:

```text
102
```

로 바뀌었다면:

```text
BASE_STATE_MISMATCH
```

가 발생해야 한다.

102를 자동 덮어써서는 안 된다.

---

## 14.3 Manual Merge Durability

사용자가 만든 Merge Result가 Server Commit 전에 Client 종료로 사라져서는 안 된다.

Restart 후 다시 복구할 수 있어야 한다.

---

# 15. Core Invariants

E2E Scenario마다 가능한 한 다음을 확인한다.

```text
Committed Server Revision은 중복되지 않는다.

Cursor는 처리하지 않은 Revision을 건너뛰지 않는다.

Pending Local Change는 Commit 확인 전에 사라지지 않는다.

Conflict Local Content는 자동 overwrite되지 않는다.

DELETED와 UNKNOWN은 구분된다.

동일 Operation Retry는 새로운 Revision을 만들지 않는다.

Conflict가 다른 Path Sync를 막지 않는다.

Conflict와 Pending이 없으면 결국 Client는 Server State로 수렴한다.
```

---

# 16. MVP Release Gate

MVP Release 전에 최소 다음 Scenario를 통과해야 한다.

```text
1. Basic Create / Modify / Delete

2. Rename

3. Offline Modify → Reconnect

4. Long Offline → Catch-up

5. Simultaneous Modify Conflict

6. Delete vs Modify Conflict

7. Create vs Create Conflict

8. Rename Conflict

9. Delete Resurrection Prevention

10. Conflict Does Not Block Other Paths

11. Lost Response → Same Operation Retry

12. Server Crash After PREPARED

13. Server Crash After Filesystem Apply

14. Client Crash During Remote Apply

15. Own Push Does Not Jump Cursor

16. Notification Loss Recovery

17. Initial Sync Race

18. Full Reconciliation

19. Conflict Resolution

20. `.obsidian/` Exclusion
```

이 Scenario들이 실제 Component를 사용해 안정적으로 반복 실행되면 MVP 수준의 Sync Correctness는 충분히 검증된 것으로 본다.

---

# 17. Future Testing

시스템 규모와 복잡도가 커졌을 때 다음을 추가할 수 있다.

```text
Property-based Testing

Model-based Testing

Long-running Random Operation Test

Large Vault Performance Test

Protocol Compatibility Matrix

Network Fault Simulation

Multi-version Client / Server Test
```

MVP에서는 이러한 Test Infrastructure를 먼저 만들지 않는다.

필요성이 실제로 발생했을 때 도입한다.

---

이 Testing Strategy의 핵심은:

> **개별 Class를 많이 테스트하는 것보다 실제 Client와 Server를 연결한 상태에서 중요한 실패 시나리오가 데이터 손실 없이 복구되고 최종적으로 수렴하는지를 검증하는 것**

이다.
