# Client Architecture

## 1. 문서 목적

이 문서는 Obsidian에서 실행되는 동기화 Client의 내부 아키텍처를 정의한다.

상위 역할 분리는 [01 Architecture Overview](./01_architecture-overview.md), 동기화 순서는 [02 Synchronization Protocol](./02_synchronization-protocol.md)을 따른다. Client가 보존하는 논리 상태는 [06 Data Model](./06_data-model.md), 그 durable storage와 transaction은 [08 Persistence Design](./08_persistence-design.md)을 단일 기준으로 사용한다.

Client는 단순히 서버의 파일을 다운로드하는 프로그램이 아니다.

다음 네 종류의 상태를 동시에 관리한다.

```text
Client State

├── Local Vault
├── Replica Index
├── Pending Operations
└── Conflict State
```

Client Architecture의 핵심 책임은 다음과 같다.

- Obsidian의 Local Vault 변경 감지
- Offline 변경의 durable 보존
- 서버 변경 Pull 및 Local Vault 반영
- Replica Index 관리
- Pending Operation 관리
- Local / Remote 동시 변경 충돌 감지
- Remote Apply에 의한 Sync Loop 방지
- Client Crash 이후 복구
- Event 유실 이후 복구
- 모바일 환경 지원
- Reconciliation Scheduling

본 문서에서는 구체적인 HTTP Endpoint와 UI 구성을 확정하지 않는다. 연결 설정,
첫 동기화, 상태 표시, conflict 진입, 진단과 reset의 사용자 경험은
[12 Client User Experience](./12_client-user-experience.md)를 따른다. 정확한
Client Storage 구현 기술은 [08 Persistence Design](./08_persistence-design.md)을
따른다.

---

## 2. 핵심 설계 원칙

### 2.1 Local Vault는 사용자 작업 공간이다

Obsidian은 항상 일반 Local Vault를 직접 읽고 수정한다.

```text
Obsidian
    │
    ▼
Local Vault
```

동기화 Client가 별도의 Virtual Filesystem을 제공하지 않는다.

Server 연결 여부와 관계없이 사용자는 Local Vault를 사용할 수 있다.

---

### 2.2 Local Vault는 Authoritative State가 아니다

Local Vault는 사용자 작업 공간이며 Server Vault의 Replica이다.

```text
Local Vault
    =
Replica
+
Offline Workspace
```

Local 변경은 Server에 Commit되기 전까지 전체 시스템의 authoritative state가 아니다.

---

### 2.3 File Event는 correctness mechanism이 아니다

Obsidian에서 발생하는:

```text
create
modify
delete
rename
```

이벤트는 Local 변경을 빠르게 감지하기 위한 수단이다.

Client는 이벤트가 항상 전달된다고 가정하지 않는다.

예를 들어:

```text
File modified
    │
    X
App killed before event processing
```

상황에서도 다음 실행 시 Local Vault와 Replica Index를 비교하여 변경을 다시 발견할 수 있어야 한다.

따라서:

```text
Vault Event
    =
Notification / Optimization

Local Reconciliation
    =
Correctness
```

로 정의한다.

---

### 2.4 Background Execution에 의존하지 않는다

모바일 OS는 언제든 Obsidian Process를 suspend하거나 종료할 수 있다.

따라서 다음을 전제로 하지 않는다.

```text
항상 실행되는 background worker

항상 유지되는 WebSocket

종료 직전 cleanup 실행

일정 시간 안에 Queue flush 완료
```

대신 모든 중요한 상태를 durable하게 저장하고 다음 실행 또는 foreground 복귀 시 이어서 처리한다.

---

### 2.5 Mobile-compatible Core

Client Core는 Desktop에서만 제공되는 기능에 의존하지 않는다.

동기화 핵심 기능은 다음에 의존해서는 안 된다.

```text
Node.js filesystem API
Electron API
Shell Process
Native Git
Desktop-only FileSystem API
```

Desktop과 Mobile에서 동일한 Sync Core를 사용하는 것을 기본 구조로 한다.

플랫폼별 기능이 필요한 경우 Adapter 계층 뒤로 격리한다.

---

## 3. Client 구성 요소

Client는 개념적으로 다음 컴포넌트로 구성한다.

```text
                 Obsidian
                    │
                    ▼
           ┌─────────────────┐
           │ Vault Observer  │
           └────────┬────────┘
                    │
                    ▼
           ┌─────────────────┐
           │ Change Analyzer │
           └────────┬────────┘
                    │
                    ▼
           ┌─────────────────┐
           │ Pending Store   │
           └────────┬────────┘
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
   Sync Scheduler       Local Reconciler
          │
          ▼
   Transport Client
          │
          ▼
      Server
          │
          ▼
   Remote Apply Engine
          │
          ▼
      Local Vault

Shared State:

Replica Index
Conflict Store
Apply Journal
Client State Store
```

---

## 4. Vault Observer

Vault Observer는 Obsidian의 Vault 변경 이벤트를 관찰한다.

관심 이벤트는 최소 다음과 같다.

```text
CREATE
MODIFY
DELETE
RENAME
```

Vault Observer 자체는 Network 요청을 수행하지 않는다.

잘못된 구조:

```text
modify event
    │
    ▼
HTTP PUT
```

정상 구조:

```text
modify event
    │
    ▼
Change Analyzer
    │
    ▼
Persistent Pending State
    │
    ▼
Sync Scheduler
```

Network 상태와 Local File Event 처리를 분리한다.

---

## 5. Event Debounce

Obsidian Editor의 한 번의 사용자 편집이 여러 파일 이벤트를 발생시킬 수 있다.

예:

```text
modify
modify
modify
modify
```

짧은 시간 동안 동일 파일에서 발생한 이벤트는 debounce할 수 있다.

목적은 불필요한 Hash 계산과 Queue 갱신을 줄이는 것이다.

하지만 debounce 상태 자체는 correctness를 담당하지 않는다.

Debounce가 완료되기 전에 Process가 종료되더라도 다음 Local Reconciliation에서 변경을 다시 발견할 수 있어야 한다.

---

## 6. Ignore Paths

다음 영역은 Local Change Detection과 Reconciliation에서 제외한다.

```text
.obsidian/
```

또한 Client 내부 데이터가 Vault 내부에 존재하는 구현을 선택하는 경우 해당 내부 경로 역시 반드시 제외해야 한다.

그러나 기본적으로 Client Sync State는 synchronized content와 분리하여 저장한다.

---

## 7. Change Analyzer

Change Analyzer는 Local Vault의 실제 상태와 Replica Index를 비교한다.

예:

```text
Replica Index:

notes/a.md
hash = AAA
exists = true
```

Local:

```text
notes/a.md
hash = BBB
```

이면:

```text
MODIFY notes/a.md
```

로 판단할 수 있다.

반면:

```text
Replica:
A.md exists

Local:
A.md absent
```

이면:

```text
DELETE A.md
```

후보이다.

Replica에 존재하지 않지만 Local에 존재하면:

```text
CREATE
```

후보가 된다.

Vault Event는 Change Analyzer를 실행시키는 Trigger일 뿐 실제 Operation 유형의 최종 판단은 현재 상태와 Replica Index를 기준으로 한다.

---

## 8. Client Sync State 경계

Client는 Replica Index, Server Cursor, Pending Operation, Apply Journal, Conflict State를 함께 관리한다. 각 필드와 상태값의 의미는 [06 Data Model](./06_data-model.md), durable 저장 순서와 store 분리는 [08 Persistence Design](./08_persistence-design.md)을 따른다.

이 아키텍처에서 중요한 사용 규칙은 다음과 같다.

- Replica Index는 Local Vault의 현재 내용이 아니라 Client가 처리한 Server 지식이다.
- Cursor는 처리한 Change Stream 범위를, Replica Index는 path별 Server 상태를 나타내므로 서로 대체할 수 없다.
- Pending Operation은 Server Commit 확인 전까지 제거하지 않으며, 같은 Operation ID로 retry할 수 있어야 한다.
- Upload 직전 Local Hash가 기록된 Pending 상태와 다르면, Change Analyzer가 현재 Local 상태를 다시 판단한다.

---

## 9. Safe Operation Coalescing

Server에 아직 전송되지 않은 연속 Local Operation은 안전한 경우 합칠 수 있다.

예:

```text
Base AAA

MODIFY → BBB
MODIFY → CCC
MODIFY → DDD
```

를 다음 하나로 표현할 수 있다.

```text
MODIFY

baseHash = AAA
localHash = DDD
```

그러나 Operation 의미가 달라지는 경우 무조건 합치지 않는다.

예:

```text
MODIFY
RENAME
DELETE
```

는 순서에 따라 의미가 달라질 수 있다.

초기 구현에서는 복잡한 최적화보다 정확한 Operation Ordering을 우선한다.

---

## 10. Operation Ordering

동일 Path에 대한 Pending Operation은 인과 순서를 유지해야 한다.

예:

```text
1. MODIFY A.md
2. RENAME A.md → B.md
3. MODIFY B.md
```

Server는 이 변경을 임의의 순서로 받을 수 없다.

따라서 Client는 동일한 파일 계보에 속한 Operation을 순차적으로 Commit한다.

이전 Operation이 Server에서 확정된 뒤 후속 Operation의 Base State를 갱신할 수 있다.

서로 독립적인 파일은 병렬 전송을 허용할 수 있지만 초기 구현에서는 단순한 Queue 처리를 사용할 수 있다.

---

## 11. Local Reconciliation

File Event 유실을 복구하기 위해 Client는 Local Vault와 Replica Index를 비교할 수 있어야 한다.

개념적으로:

```text
Local Vault
      ↕
Replica Index
```

비교 결과:

```text
Unchanged
Local Created
Local Modified
Local Deleted
Unknown / Conflict
```

을 식별한다.

Local Reconciliation은 최소 다음 시점에서 수행할 수 있다.

```text
Plugin Startup

App Resume

Unexpected State

Explicit Sync

Recovery
```

대규모 Vault에서는 전체 Scan 비용이 있으므로 향후 최적화할 수 있지만 correctness를 위해 Full Local Reconciliation 경로는 항상 존재해야 한다.

---

## 12. Client Startup

Client 시작 절차는 다음 순서를 기본으로 한다.

```text
1. Client State Store Open

2. Vault Event Listener 활성화
   └── Initial Buffer Mode

3. Incomplete Remote Apply Recovery

4. Pending / Conflict State 복구

5. Sync Scheduler 시작

6. Initial Bootstrap이 미완료인 경우
   └── Server Manifest 통합 → Local 분류 → CREATE 후보 durable queue

7. Local Reconciliation

8. Buffered Event 재평가

9. Normal Operation
```

초기화 중 발생하는 Vault Event를 잃지 않도록 Listener를 먼저 등록하고 Buffer한다.

Bootstrap이 미완료인 Client는 Server `UNKNOWN` Local 경로를 Local scan이나 buffered event만으로 CREATE하지 않는다. 다만 기존 Replica와 비교해 발견한 Local drift는 offline Pending으로 먼저 durable하게 복구할 수 있다. 기존 Pending이 있으면 Change Journal에서 own operation recovery를 먼저 수행하고, 이어서 fresh Server Manifest를 통합하여 path별 Server State를 확정한 후 `UNKNOWN` Local 경로를 분류한다. Bootstrap 완료 Client는 일반 Local Reconciliation을 수행한 뒤 Buffered Event를 다시 평가한다.

---

## 13. Sync Scheduler

Sync Scheduler는 다음 사건을 동기화 Trigger로 사용한다.

```text
Local Pending 발생

Plugin Startup

App Foreground / Resume

Network Online

Server Notification

Manual Sync

Retry Timer
```

여러 Trigger가 동시에 발생해도 실제 Sync Cycle을 중복 실행하지 않는다.

개념적으로:

```text
Many Triggers
     │
     ▼
Sync Scheduler
     │
     ▼
Single Logical Sync Cycle
```

Local create, modify, delete, rename 이벤트는 Pending Operation을 durable하게 기록한 뒤 자동으로 Scheduler를 깨워야 한다. 따라서 사용자가 **Sync now**를 누르는 것은 정상 동기화의 전제 조건이 아니라, 즉시 재시도를 원하는 경우의 수동 Trigger다.

Scheduler는 최소 다음 상태를 구분한다.

```text
IDLE
  │ trigger
  ▼
SCHEDULED
  │
  ▼
RUNNING ── trigger during run ──► RUNNING + FOLLOW_UP flag
  │
  ├── retryable failure ──► RETRY_WAIT
  │
  └── completed ──► IDLE 또는 SCHEDULED (FOLLOW_UP)
```

실행 중 새 Trigger가 생기면 별도 병렬 Sync를 시작하지 않고, 현재 Cycle 종료 후 한 번 더 실행한다. retry timer는 foreground에서 실행 중인 Client를 위한 최적화일 뿐이며, background 실행이나 영구적인 WebSocket 연결을 전제로 하지 않는다.

Client는 상태 표시를 통해 자동 동기화가 실제로 수행되었는지 알 수 있어야 한다. 최소한 `Syncing`, `Up to date`, `Pending`, `Offline`, `Conflict`, `Error`와 마지막 성공 시각을 제공한다. 자동 실행 실패는 상태에 남기고, 사용자 조치가 필요한 Conflict와 지속 Error만 눈에 띄게 알린다. 표시 상태의 우선순위, 문구, 진입점과 Action은 [12 Client User Experience](./12_client-user-experience.md)를 따른다.

---

## 14. Sync Cycle

기본 Sync Cycle은 Synchronization Protocol에서 정의한 순서를 따른다.

```text
1. Pull

2. Integrate Remote Changes

3. Push Pending

4. Pull Again

5. Convergence Check
```

한 Cycle 중 새로운 Local Event가 발생할 수 있으므로 종료 시 Pending 및 Server Revision 상태를 다시 확인한다.

필요하면 다음 Cycle을 즉시 수행한다.

---

## 15. Remote Apply Engine

Server에서 받은 변경을 Local Vault에 적용하는 책임은 Remote Apply Engine이 가진다.

Remote Apply Engine은 먼저 다음을 확인한다.

```text
현재 Local State가
Replica Index가 예상한 상태와 같은가?
```

예:

```text
Replica:
A.md = AAA

Local:
A.md = AAA

Remote:
A.md = BBB
```

이면 안전하게 Remote Change를 적용할 수 있다.

반면:

```text
Replica:
A.md = AAA

Local:
A.md = LOCAL

Remote:
A.md = SERVER
```

이면 Local 변경을 덮어쓰지 않는다.

Conflict로 전환한다.

---

## 16. Remote Apply Journal

Remote Apply는 Local Vault write 전 durable Apply Intent를 기록하고, write 성공 후 Replica Index를 갱신해 intent를 완료한다. Apply Record의 필드와 lifecycle은 [06 Data Model](./06_data-model.md), prepare/finalize transaction은 [08 Persistence Design](./08_persistence-design.md)을 단일 기준으로 사용한다.

---

## 17. Remote Apply Crash Recovery

Client Startup 시 미완료 Apply Intent가 존재하면 Local State를 확인한다.

### Before State와 동일

```text
Local = beforeHash
```

Remote Apply가 아직 적용되지 않은 것으로 판단할 수 있다.

Remote Change를 다시 적용한다.

### After State와 동일

```text
Local = afterHash
```

Remote Apply는 완료되었으나 metadata finalize 전에 종료된 것으로 판단할 수 있다.

Replica Index를 갱신하고 Apply Intent를 완료한다.

### 어느 쪽과도 다름

```text
Local != before
Local != after
```

임의로 덮어쓰지 않는다.

```text
RECOVERY_REQUIRED
```

또는 Conflict 상태로 전환하여 Local Content를 보존한다.

---

## 18. Remote Apply Loop 방지

Remote Change를 Local Vault에 기록하면 Obsidian은 이를 다시 Local Modify Event로 전달할 수 있다.

잘못 처리하면:

```text
Server
   ↓
Client Pull
   ↓
Local Write
   ↓
Modify Event
   ↓
Client Push
   ↓
Server
```

라는 Loop가 발생한다.

Remote Apply 중에는 단순히:

```text
ignore next event for A.md
```

같은 방식으로 이벤트를 무시하지 않는다.

이 방식은 사용자의 실제 변경까지 잘못 무시할 수 있다.

대신 Apply Journal의 expected state를 이용한다.

```text
Observed Event
      │
      ▼
현재 Local Hash
      │
      ├── expected Remote Hash와 동일
      │       → Remote Apply Event
      │
      └── 다름
              → 실제 Local Change 후보
```

즉 **Path가 아니라 State를 기준으로 Loop를 방지한다.**

---

## 19. Own Change Integration

Client가 Push한 Operation도 이후 Server Change Stream에서 다시 관찰할 수 있다.

해당 Change의 Operation ID가 자신이 Commit한 Operation과 같고 Local State가 기대한 결과와 일치하면 Local File을 다시 작성하지 않는다.

대신:

```text
Replica Index Update

Server Cursor Advance

Committed Pending Cleanup
```

만 수행한다.

---

## 20. Conflict Detection

Remote Change가 Local Pending과 같은 File State에 영향을 미치면 Conflict가 발생한다.

예:

```text
Base = AAA

Server = BBB

Local = CCC
```

Client는 어느 한쪽을 Local Vault에 강제로 적용하지 않는다.

```text
        AAA
       /   \
    BBB     CCC

   Server   Local
```

---

## 21. Conflict Store

Conflict는 Pending Operation의 단순 실패 상태가 아니다.

별도의 durable state로 보존한다.

개념적으로:

```text
conflictId
path

baseRevision
baseHash

serverRevision
serverHash

localHash

originalOperationId
state
```

필요한 경우 Local Conflict Content 자체도 Client-local recovery storage에 별도로 보존할 수 있다.

Conflict Artifact는 synchronized Vault의 일반 문서로 자동 생성하지 않는다.

그렇게 하면 다시 동기화 대상이 되기 때문이다.

---

## 22. Conflict 중 Server Cursor 진행

Conflict가 발생해도 전체 Vault 동기화를 중단하지 않는다.

예:

```text
1201 MODIFY A.md  → Conflict
1202 MODIFY B.md
1203 CREATE C.md
```

A.md Conflict를 durable하게 기록한 후:

```text
1202
1203
```

을 계속 처리할 수 있다.

결과:

```text
serverCursor = 1203

Conflict:
A.md
```

---

## 23. Conflicted Path

Conflict가 존재하는 Path는 자동 Remote Apply 및 Local Push 대상으로 사용하지 않는다.

```text
A.md = CONFLICTED
```

상태에서 새로운 Server Change가 A.md에 발생할 수 있다.

이 경우 Client는 새로운 Server State를 계속 관찰하고 Replica Index에는 최신 Server State를 반영한다.

하지만 Local Vault의 A.md를 자동으로 덮어쓰지 않는다.

Conflict Record는 최신 Server State를 가리킬 수 있도록 갱신된다.

```text
Local Conflict Content
        │
        │
        ├────────── latest Server State
        │
        └────────── Base State
```

Conflict가 해결될 때까지 해당 Path는 격리된다.

---

## 24. Local Change During Conflict

사용자는 Conflict 상태의 Local File을 계속 편집할 수 있다.

Client는 이를 금지하지 않는다.

그러나 해당 변경을 자동으로 Server에 Push하지 않는다.

Conflict 해결 시점의 최신 Local Content를 사용자가 선택할 수 있도록 현재 Local State를 추적한다.

필요한 경우 최초 Conflict 발생 시점의 Local Content를 별도 recovery artifact로 보존할 수 있다.

---

## 25. Delete Apply

Server에서 Delete Change를 Pull한 경우 다음을 확인한다.

```text
Replica:
A.md = AAA

Local:
A.md = AAA

Remote:
DELETE A.md
```

이면 Local File을 제거할 수 있다.

반면:

```text
Replica:
A.md = AAA

Local:
A.md = BBB

Remote:
DELETE A.md
```

이면 Local 사용자가 변경한 파일을 삭제하지 않는다.

```text
DELETE vs MODIFY
        │
        ▼
     Conflict
```

---

## 26. Local Delete

사용자가 Local File을 삭제하면 Change Analyzer는 Replica Index를 확인한다.

Replica에 존재했던 파일이면:

```text
DELETE
```

Pending Operation을 생성한다.

Replica에 존재한 적 없는 Local-only File이 생성되었다가 Server Sync 전에 삭제된 경우 안전하다면 Pending Create를 제거하여 No-op으로 만들 수 있다.

---

## 27. Rename / Move

Obsidian에서 Rename Event가 제공되는 경우 source와 destination을 하나의 논리적인 Local Change로 인식한다.

```text
A.md
 ↓
B.md
```

Pending Operation:

```text
RENAME

from = A.md
to = B.md
```

로 표현할 수 있다.

Rename 전에 MODIFY가 Pending 상태라면 Operation Ordering을 보존한다.

```text
MODIFY A.md
      ↓
RENAME A.md → B.md
```

무리하게 하나의 Operation으로 합치지 않는다.

---

## 28. Rename Event 유실

Client가 Rename Event를 놓치고 이후 Reconciliation만 수행하는 경우:

```text
Replica:
A.md exists

Local:
A.md absent
B.md exists
```

이라는 상태만으로 `A → B` Rename을 항상 확실하게 판정할 수 있는 것은 아니다.

이 경우 Hash 등으로 높은 신뢰도의 Rename을 추론할 수 있지만 correctness를 위해 반드시 Rename으로 복구할 필요는 없다.

안전한 경우:

```text
DELETE A
CREATE B
```

형태로 표현할 수도 있다.

단, 이 과정이 Server State와 충돌한다면 Conflict로 처리한다.

Rename 보존은 유용한 metadata이지만 데이터 보존보다 우선하지 않는다.

---

## 29. Local Integrity Scan

Client는 Replica Index와 Local Vault의 동기화 대상 파일을 비교할 수 있다.

검사 대상:

```text
Path

Existence

Content Hash
```

mtime은 Scan 최적화에 사용할 수 있지만 최종 동일성 판정 기준으로 사용하지 않는다.

Full Hash 계산 비용이 큰 경우 mtime/size 등을 이용해 Hash가 필요한 파일을 좁힐 수 있다.

최종 내용 동일성은 Content Hash로 판단한다.

---

## 30. Initial Sync

새 Client는 Server를 Source of Truth로 사용한다.

빈 Local Vault의 기본 초기화는:

```text
Server Manifest
      │
      ▼
Local Synchronized Content
```

방향으로 수행한다.

`.obsidian/`과 Client-local configuration은 변경하지 않는다.

---

## 31. Initial Sync와 기존 Local Content

처음 등록되는 Client의 기존 Local Content 처리 규칙은 [02 Synchronization Protocol](./02_synchronization-protocol.md)의 안전한 Initial Bootstrap을 따른다. Server Manifest를 먼저 통합한 뒤, Server가 `UNKNOWN`으로 확인한 Local 경로만 CREATE 후보로 만든다. 동일 경로의 Server PRESENT, Server DELETED, 타입 불일치는 모두 Conflict로 보존하며 자동 upload 또는 overwrite하지 않는다.

Initial Bootstrap은 다음 상태로 재개 가능해야 한다.

```text
BOOTSTRAP_REQUIRED
        │ fresh manifest integrated
        ▼
CLASSIFYING_LOCAL_STATE
        │ every path durably classified
        ▼
BOOTSTRAP_COMPLETE
```

Client 종료, network failure, 또는 manifest 만료가 발생하면 `BOOTSTRAP_COMPLETE`를 기록하지 않는다. 다음 실행은 새 manifest로 처음 두 단계를 다시 수행한다. 이미 durable한 Pending과 Conflict는 유지하고 중복 생성하지 않는다.

---

## 32. Client State Store

Client Sync Metadata는 synchronized content 및 사용자 설정과 분리해 durable하게 보존한다. 정확한 저장 대상, IndexedDB mapping, transaction과 settings 분리 규칙은 [08 Persistence Design](./08_persistence-design.md)을 단일 기준으로 사용한다. 설정 초기화가 Sync State의 무조건적인 삭제로 이어져서는 안 된다.

---

## 33. Client Identity

각 Client는 재시작에도 변하지 않는 Client Identity를 가진다. 이는 Vault access token이 아니며 데이터 모델과 API 경계의 정의는 [06 Data Model](./06_data-model.md), 접근 제어와의 구분은 [09 Security and Deployment](./09_security-and-deployment.md)을 따른다.

---

## 33.1 Public Vault Token

`public-token` Server에 연결하는 Client는 Server URL과 별도로 Vault token을
가진다. token은 `.obsidian/`의 plugin-local settings에만 보관하며 VaultDatum
sync state, IndexedDB replica metadata, Pending Operation, diagnostic, clipboard helper에
복제하지 않는다.

Obsidian Mobile과 Desktop에 공통으로 쓸 수 있는 OS secure-storage API가 없으므로,
plugin-local persistence가 device-at-rest encryption을 보장한다고 주장해서는 안 된다.
device를 잃었거나 token 노출이 의심되면 Server의 Vault token을 교체하고 모든 장치에
새 token을 다시 입력하는 것이 0.4.0의 복구 경계다.

Transport는 token을 정규화한 configured HTTPS origin에만 `Authorization: Bearer`
header로 붙인다. `http://` public URL, token을 URL 또는 request body에 넣는 동작은
거부한다. Public Gateway route는 인증된 sync request를 다른 origin으로 redirect해서는
안 된다. 사용자가 URL을 다른 origin으로 바꾸면 Client는 기존 token field를 비우고,
새 token을 명시적으로 입력받아야 한다.

---

## 34. Transport Client

Transport Client는 Server와 통신하는 책임만 가진다.

역할:

```text
Server Status

Manifest

Change Pull

Content Download

Operation Push

Notification Connection
```

Transport는 Sync Algorithm을 소유하지 않는다.

```text
Transport Failure
    │
    ▼
Sync Scheduler Retry
```

로 처리한다.

`public-token` transport는 모든 protected HTTP request에 bearer header를 붙이고,
401을 retryable network failure로 취급하지 않는다. token 입력 또는 rotation이
완료되기 전에는 Pending Operation을 durable하게 유지한다. header, token,
realtime ticket은 log와 error object에 담지 않는다.

---

## 35. Notification Channel

Notification Channel은:

```text
Server changed
latestRevision = N
```

과 같은 이벤트를 받을 수 있다.

Notification 수신 시 Sync Scheduler를 깨운다.

Notification 자체를 Local Vault에 바로 적용하지 않는다.

연결이 끊어져도 Client correctness에는 영향을 주지 않는다.

Notification payload와 reconnect 계약은 [07 API Specification](./07_api-specification.md), correctness 규칙은 [02 Synchronization Protocol](./02_synchronization-protocol.md)을 따른다.

`public-token` profile에서는 먼저 authenticated HTTP로 one-time ticket을 받고
`vaultdatum.v1`과 ticket subprotocol offer로 WebSocket을 연다. browser WebSocket API에
임의 Authorization header를 붙이거나 URL query에 token을 넣지 않는다. ticket
발급 또는 socket 연결 실패는 다음 HTTP catch-up을 막지 않는다.

---

## 36. Network Failure

Network Timeout이 발생했을 때 Client는 해당 Operation이 Server에서 Commit되었는지 알 수 없을 수 있다.

따라서:

```text
Timeout
    ≠
Operation Failed
```

로 본다.

Pending Operation을 유지하고 동일 Operation ID로 Retry한다.

---

## 37. Offline Mode

Server에 접근할 수 없으면 Client는 Offline 상태가 된다.

Offline 상태에서도:

```text
Read

Create

Modify

Delete

Rename
```

Local 작업은 계속 가능하다.

변경은 Pending Store에 누적된다.

Server 연결이 복구되면 정상 Sync Cycle을 실행한다.

---

## 38. Mobile Lifecycle

모바일 Client는 App Lifecycle 변화를 Sync Trigger로 활용할 수 있다.

예:

```text
App Startup

Foreground Resume

Network Restored
```

Foreground로 돌아올 때 항상 다음을 가정한다.

```text
내가 Offline인 동안
Server도 변경되었을 수 있다.
```

따라서 Notification 수신 여부와 관계없이 Server Reconciliation을 수행할 수 있어야 한다.

---

## 39. Client State Corruption

Replica Index 또는 Server Cursor를 신뢰할 수 없으면 Incremental Sync를 중단하고 Full Reconciliation으로 전환한다. Pending과 Conflict를 보존하는 storage recovery 절차는 [08 Persistence Design](./08_persistence-design.md), 사용자가 실행하는 reset 절차는 [11 Observability and Operations](./11_observability-and-operations.md)을 따른다.

---

## 40. Server History Gap

Client Cursor가 Change Journal 보존 범위보다 오래되면 Full Reconciliation으로 전환하고 Local Pending을 유지한다. history-gap 응답 및 reconciliation 절차는 [02 Synchronization Protocol](./02_synchronization-protocol.md)과 [07 API Specification](./07_api-specification.md)을 따른다.

---

## 41. Hash Strategy

Content Hash는 다음 상황에서 사용한다.

```text
Local Change Detection

Base State Validation

Remote Apply Verification

Crash Recovery

Full Reconciliation

Loop Prevention
```

전체 Vault를 모든 이벤트마다 다시 Hash하지 않는다.

변경 후보 파일 또는 Reconciliation 대상 파일에 대해서만 필요한 Hash를 계산한다.

대용량 Binary File은 Streaming Hash를 사용할 수 있다.

---

## 42. Sync Scheduling과 Backpressure

사용자가 대량의 파일을 변경했을 때 Event마다 독립적인 Network Request를 즉시 발생시키지 않는다.

```text
1000 File Events
      │
      ▼
Pending Store
      │
      ▼
Sync Scheduler
```

Scheduler는 Queue를 조절하며 Server와 동기화한다.

Network가 느린 경우에도 Local Event Producer가 Network Consumer 속도에 직접 종속되지 않는다.

---

## 43. Large Attachment

대용량 Attachment를 처리할 때 전체 파일을 JavaScript Memory에 동시에 올리는 구현을 피한다.

가능한 Transport 및 Platform API 범위에서 Streaming 또는 bounded memory 처리를 사용한다.

Upload 중 App이 종료되면 해당 Pending Operation을 유지하고 다음 실행에서 재시도할 수 있어야 한다.

부분 Upload 자체가 Server Commit으로 간주되어서는 안 된다.

---

## 44. Client-local Recovery Data

Pending metadata, conflict artifact, apply journal, replica index, temporary download와 recovery state는 synchronized content와 분리한다. 이들이 Vault Content로 업로드되지 않도록 하는 store layout과 GC는 [08 Persistence Design](./08_persistence-design.md)을 따른다.

---

## 45. Client Failure Policy

Client는 다음 우선순위를 가진다.

```text
1. 사용자의 Local Content 보존

2. Pending Change 보존

3. Server와의 정합성

4. Recovery 가능성

5. 자동화

6. 성능
```

따라서 Local State가 애매한 경우:

```text
Overwrite
```

보다:

```text
Conflict / Recovery Required
```

를 선택한다.

---

## 46. Client Architecture Invariants

### Invariant 1 — Local Changes Survive Restart

Server가 아직 확정하지 않은 Local Change는 Client Process 종료만으로 사라져서는 안 된다.

### Invariant 2 — Events Are Advisory

Vault Event를 놓쳐도 Local Reconciliation으로 Local Change를 발견할 수 있어야 한다.

### Invariant 3 — Remote Apply Never Blindly Overwrites Local Drift

Local State가 Replica Index에서 벗어난 상태라면 Remote Change를 무조건 적용하지 않는다.

### Invariant 4 — Remote Apply Is Recoverable

Local Vault를 Remote State로 변경하기 전에 복구 가능한 Apply Intent를 보존한다.

### Invariant 5 — Loop Suppression Is State-based

Remote Apply Event 여부는 단순 Path 또는 시간 기준이 아니라 expected state와 실제 state를 비교하여 판정한다.

### Invariant 6 — Push Requires Durable Pending State

Client는 Pending Operation을 durable하게 보존하기 전에 Network Push를 authoritative workflow로 시작하지 않는다.

### Invariant 7 — Server Success Before Pending Removal

Server Commit이 확인되기 전에 Pending Operation을 제거하지 않는다.

### Invariant 8 — Conflict Does Not Stop Global Synchronization

하나의 Path에 Conflict가 존재하더라도 다른 Server Change는 계속 처리할 수 있어야 한다.

### Invariant 9 — Conflicted Path Is Quarantined

Conflict가 해결되기 전까지 해당 Path에 대해 자동 Push 또는 destructive Remote Apply를 수행하지 않는다.

### Invariant 10 — Cursor Is Contiguous

중간 Server Change를 처리하지 않은 상태로 Server Cursor를 건너뛰지 않는다.

### Invariant 11 — Client-local State Never Syncs

`.obsidian/` 및 Client Sync Metadata는 synchronized content에 포함하지 않는다.

### Invariant 12 — Background Execution Is Optional

Client correctness는 지속적인 background execution이나 persistent WebSocket 연결에 의존하지 않는다.

### Invariant 13 — Mobile Is a First-class Client

Core Sync Algorithm은 Desktop-only Runtime 기능을 필요로 하지 않는다.

---

## 47. 전체 Client Flow

```text
                       User
                        │
                        ▼
                   Local Vault
                        │
                        │ Vault Event
                        ▼
                 Change Analyzer
                        │
              Replica Index 비교
                        │
                        ▼
                 Pending Store
                        │
                        ▼
                  Sync Scheduler
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
        Pull Changes          Push Pending
             │                     │
             ▼                     ▼
      Remote Apply Engine        Server
             │
       ┌─────┴─────┐
       │           │
       ▼           ▼
     Apply      Conflict
       │           │
       ▼           ▼
 Local Vault   Conflict Store
       │
       ▼
 Replica Index
       │
       ▼
 Server Cursor
```

정상적인 Client 동작의 핵심은:

```text
Vault Event를 신뢰하는 것
```

이 아니라:

```text
Local Vault
     +
Replica Index
     +
Pending Operations
     +
Server State

를 지속적으로 비교하여
설명되지 않는 상태 차이를 남기지 않는 것
```

이다.
