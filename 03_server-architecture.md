# Server Architecture

## 1. 문서 목적

이 문서는 동기화 서버의 내부 아키텍처를 정의한다.

상위 authoritative-state 모델은 [01 Architecture Overview](./01_architecture-overview.md)을, 그 상태의 정확한 정의는 [06 Data Model](./06_data-model.md)을 따른다. 이 문서는 두 모델을 구현하는 Server 내부의 책임과 복구 경계만 다룬다.

서버 아키텍처의 핵심 문제는 canonical content와 sync metadata를 서로 다른 저장 매체에 보관하면서도 하나의 일관된 authoritative state처럼 동작하게 만드는 것이다.

이 문서는 특히 다음 문제를 다룬다.

* Vault Filesystem과 Sync State의 관계
* 서버 변경의 단일 진입점
* Global Revision 생성
* 변경의 durable commit
* 서버 Crash 중 부분 적용 방지
* Operation Retry
* Delete / Rename / Move 처리
* 서버 Vault 쓰기 경계와 불일치 감지
* 서버 시작 시 Recovery
* 동시 요청 처리
* Persistent Storage 계약

본 문서에서는 아직 다음을 확정하지 않는다.

* 구체적인 HTTP Endpoint
* Request / Response JSON
* 정확한 DB Table 정의
* Client 내부 구현
* UI
* VPN / TLS의 구체적인 구성

---

## 2. 핵심 설계 결정

### 2.1 콘텐츠의 원본은 Filesystem이다

Markdown과 Attachment의 실제 내용은 서버 Vault의 일반 파일로 저장한다.

```text
Canonical Content
        =
Vault Filesystem
```

Markdown 내용을 별도의 DB 레코드에 다시 저장하고 그것을 원본으로 사용하지 않는다.

즉:

```text
DB Document
     ↕
Filesystem Document
```

와 같은 이중 콘텐츠 모델을 만들지 않는다.

사용자가 백업하거나 서버에서 Vault를 직접 확인했을 때 일반적인 Obsidian Vault 구조를 그대로 볼 수 있어야 한다.

Filesystem이 콘텐츠의 원본이라는 것은 저장 형식에 대한 결정이다. 서버 Vault를 Sync API 밖에서 수정해도 된다는 의미가 아니다. 서버 Vault의 쓰기 경계는 아래 26절 Server Vault 쓰기 경계를 따른다.

### 2.2 Sync State는 별도의 durable metadata이다

Filesystem만으로는 다음 정보를 복원할 수 없다.

```text
이 파일이 언제 변경되었는가?

이 파일은 삭제된 것인가?

이 경로는 Rename된 것인가?

특정 Client Operation은 이미 처리되었는가?

Revision 1000 이후 무엇이 변경되었는가?
```

따라서 Server는 별도의 Sync State를 유지한다.

개념적으로:

```text
Sync State

├── Current Revision
├── File Index
├── Change Journal
├── Tombstones
├── Operations
└── Recovery State
```

이 정보 역시 authoritative state의 일부이므로 서버 재시작 이후에도 보존되어야 한다.

---

## 3. 저장소 선택

Sync State 저장소로 **SQLite**를 사용한다.

선택 이유:

* 단일 사용자
* 단일 Server Instance
* 별도 DB Server가 필요 없음
* Transaction 지원
* Crash Recovery 지원
* 작은 운영 복잡도
* 서버 Vault와 함께 쉽게 백업 가능

초기 시스템에서는 다음 구조를 가정한다.

```text
Server
│
├── Vault Filesystem
│
└── SQLite Sync Store
```

SQLite는 Markdown이나 Attachment의 본문을 저장하는 데이터베이스가 아니다.

```text
Filesystem
    =
Content Store

SQLite
    =
Synchronization Metadata Store
```

로 역할을 구분한다.

다중 서버와 High Availability가 필요해질 경우 Sync Store를 다른 시스템으로 교체할 수 있지만 초기 아키텍처에서는 고려하지 않는다.

---

## 4. Server 구성 요소

서버 내부를 개념적으로 다음과 같이 나눈다.

```text
                    Client
                      │
                      ▼
               ┌──────────────┐
               │  Sync API    │
               └──────┬───────┘
                      │
                      ▼
             ┌──────────────────┐
             │ Sync Coordinator │
             └────────┬─────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   Mutation       Read /        Reconciliation
    Engine        Manifest          Engine
        │
        ▼
   Commit Engine
        │
        ├───────────────┐
        ▼               ▼
  Vault Filesystem   Sync Store
                      SQLite
        │               │
        └───────┬───────┘
                ▼
        Recovery Engine
```

---

## 5. Sync API Layer

Sync API Layer는 Client와 통신하는 외부 경계이다.

책임:

* Client 요청 수신
* 입력 검증
* 인증 정보 전달
* Content Streaming
* Sync Coordinator 호출
* 결과 반환

Sync API 자체가 Vault 파일을 직접 수정해서는 안 된다.

잘못된 구조:

```text
HTTP Handler
    │
    ▼
Files.write()
```

정상 구조:

```text
HTTP Handler
    │
    ▼
Sync Coordinator
    │
    ▼
Mutation Engine
    │
    ▼
Commit Engine
```

모든 authoritative mutation은 동일한 Server Mutation Path를 거쳐야 한다.

---

## 6. Sync Coordinator

Sync Coordinator는 Server의 동기화 규칙을 적용하는 상위 서비스이다.

주요 책임:

* Client Base State 검증
* Operation ID 확인
* 충돌 판정
* Mutation 생성
* Commit Engine 호출
* Server Change 조회
* Current Server State 조회
* Full Reconciliation 지원

Sync Coordinator는 HTTP, WebSocket 등 특정 Transport에 종속되지 않는다.

---

## 7. File Index

Server는 현재 Vault 상태에 대한 metadata index를 가진다.

개념적으로 각 동기화 대상 파일은 다음 정보를 가진다.

```text
Path
Current Revision
Content Hash
Size
Existence
```

예:

```text
notes/a.md

revision = 1005
hash = ...
size = 4812
exists = true
```

삭제된 상태:

```text
notes/b.md

revision = 1006
exists = false
```

File Index의 목적은 Filesystem을 대체하는 것이 아니다.

다음과 같은 빠른 판단을 지원한다.

```text
Client Base와 현재 Server State가 같은가?

해당 파일은 현재 어떤 Revision인가?

Content가 실제로 바뀌었는가?

해당 경로는 삭제 상태인가?
```

---

## 8. Change Journal

서버에서 확정된 모든 논리적 변경은 Change Journal에 기록한다.

예:

```text
1001 CREATE  notes/a.md
1002 MODIFY  notes/a.md
1003 CREATE  notes/b.md
1004 DELETE  notes/a.md
1005 RENAME  notes/b.md -> archive/b.md
```

Change Journal은 Global Revision의 순서를 정의한다.

Client의:

```text
changes since N
```

요청은 이 Journal을 기준으로 처리한다.

### 8.1 Global Revision

Global Revision은 **서버에서 commit 완료된 변경**에 대해서만 외부에 노출한다.

다음 상태는 Revision으로 취급하지 않는다.

```text
uploading
validating
staging
prepared
applying
```

Revision은 authoritative state에 반영된 변경을 의미한다.

---

## 9. Operation Store

Client의 모든 mutation 요청에는 고유 Operation ID가 존재한다.

Server는 Operation ID와 처리 결과를 durable하게 저장한다.

개념적으로:

```text
operationId
status
resultRevision
resultState
```

Operation은 최소 다음 상태를 가진다.

```text
PREPARED
COMMITTED
FAILED
```

내부 구현에서는 추가 상태를 사용할 수 있다.

Operation Store를 통해 다음을 보장한다.

```text
same operationId
        │
        ├── first request  → Commit
        │
        └── retry          → Previous Result
```

Retry가 새로운 Revision을 생성해서는 안 된다.

---

## 10. Filesystem과 DB의 Transaction 문제

SQLite Transaction은 SQLite 내부 상태만 atomic하게 변경할 수 있다.

Filesystem 변경과 SQLite Transaction을 하나의 ACID Transaction으로 직접 묶을 수는 없다.

즉 다음 과정에서 Crash가 발생할 수 있다.

```text
Filesystem changed
        │
        X Crash
        │
DB not updated
```

또는 반대 방향의 위험도 존재한다.

```text
DB committed
        │
        X Crash
        │
Filesystem not changed
```

따라서 Server는 단순한:

```text
write file
update database
```

구조를 사용하지 않는다.

대신 **recoverable commit protocol**을 사용한다.

---

## 11. Server Commit Protocol

Server mutation은 기본적으로 다음 단계를 따른다.

```text
1. Validate

2. Stage

3. Prepare

4. Apply Filesystem

5. Finalize Sync State

6. Respond Success
```

---

## 12. Phase 1 — Validate

Server는 Client 요청을 authoritative current state와 비교한다.

예:

```text
Client:

path = notes/a.md
baseRevision = 1000
baseHash = AAA
```

Server:

```text
path = notes/a.md
revision = 1000
hash = AAA
```

이면 변경을 계속 진행할 수 있다.

다른 상태라면 Mutation을 시작하지 않고 Conflict를 반환한다.

Validate는 실제 Commit 직전에도 다시 확인해야 한다.

두 Client가 동시에 같은 Base를 보고 요청할 수 있기 때문이다.

---

## 13. Phase 2 — Stage

새로운 Content를 바로 Vault 파일에 덮어쓰지 않는다.

먼저 Server 관리 영역에 임시 파일로 저장한다.

```text
Incoming Content
       │
       ▼
Staging Area
```

예:

```text
/data/

├── vault/
├── state/
└── staging/
```

Upload가 큰 Binary 파일이라면 전체 파일을 메모리에 올리지 않고 Streaming으로 Staging File에 기록한다.

Staging 중 Content Hash를 계산할 수 있다.

Staging File은 durable하게 기록된 후 다음 단계로 진행한다.

---

## 14. Phase 3 — Prepare

Filesystem Mutation을 수행하기 전에 Operation 의도를 Sync Store에 기록한다.

개념적으로:

```text
Operation

id = OP-X
type = MODIFY
path = notes/a.md

before:
    revision = 1000
    hash = AAA

after:
    hash = BBB

stagedContent = ...

status = PREPARED
```

PREPARED 상태는 다음을 의미한다.

> Server가 이 Operation을 수행하기로 결정했으며 Crash가 발생해도 Recovery Engine이 의도를 복구할 수 있다.

PREPARED 기록은 Filesystem을 변경하기 전에 durable해야 한다.

---

## 15. Phase 4 — Filesystem Apply

PREPARED 이후 실제 Vault State를 변경한다.

가능한 경우 Filesystem의 atomic operation을 사용한다.

### 15.1 Create / Modify

새로운 파일은 먼저 Staging Area에 완성된다.

그 후 최종 경로로 atomic replace한다.

```text
staging/OP-X
     │
     │ atomic rename / replace
     ▼
vault/notes/a.md
```

동일 Filesystem에서의 atomic rename semantics를 활용할 수 있도록 Vault와 Staging Area를 배치해야 한다.

Modify 역시 대상 파일을 직접 truncate하면서 쓰지 않는다.

잘못된 방식:

```text
open A.md
truncate
write...
    X crash
```

권장 방식:

```text
write temp
fsync
atomic replace
fsync directory
```

이를 통해 부분적으로 기록된 파일이 Vault에 남는 것을 방지한다.

### 15.2 Delete

Delete 역시 가능한 경우 즉시 영구 삭제하지 않는다.

Commit이 확정될 때까지 복구 가능한 Server 관리 영역으로 이동시킬 수 있다.

```text
vault/A.md

    │ atomic move
    ▼

recovery/OP-X
```

이후 Sync State Commit이 완료되면 해당 파일은 더 이상 Vault State에 존재하지 않는 것으로 확정된다.

복구용 파일은 이후 Garbage Collection할 수 있다.

### 15.3 Rename / Move

동일 Filesystem 안에서 Rename/Move는 Filesystem의 atomic rename operation을 사용한다.

```text
vault/notes/a.md
        │
        ▼
vault/archive/a.md
```

적용 전에 다음 조건을 검사한다.

* source가 예상 상태인지
* destination이 예상 상태인지
* destination overwrite 위험이 없는지

Rename/Move는 source와 destination 두 경로 모두에 대한 mutation으로 취급한다.

---

## 16. Phase 5 — Finalize Sync State

Filesystem Apply가 성공한 이후 Sync Store에서 하나의 Transaction으로 다음 상태를 갱신한다.

개념적으로:

```text
File Index Update

Change Journal Insert

Global Revision Advance

Operation → COMMITTED
```

예:

```text
Revision 1001

MODIFY notes/a.md

hash = BBB

operation = OP-X
```

이 SQLite Transaction이 durable하게 Commit된 이후에만 Server mutation은 완료된 것으로 간주한다.

---

## 17. Phase 6 — Success Response

Client에게 성공을 반환하는 조건은:

```text
Filesystem State
        +
Sync State

both durable
```

이다.

즉:

```text
Success Response
        ⇒
Durable Authoritative State
```

가 항상 성립해야 한다.

---

## 18. Commit 과정의 Crash Recovery

PREPARED Operation이 존재하기 때문에 Server는 Crash 시 어느 단계에서 중단되었는지 복구할 수 있다.

### 18.1 Prepare 이전 Crash

```text
Operation record 없음
Filesystem 변경 없음
```

아무것도 복구할 필요가 없다.

Client는 동일 Operation을 다시 전송할 수 있다.

### 18.2 Prepare 이후 / Filesystem Apply 이전 Crash

```text
Operation = PREPARED

Vault = Before State

Staging = After State
```

Recovery Engine은 Operation을 다시 Apply할 수 있다.

### 18.3 Filesystem Apply 이후 / Sync State Finalize 이전 Crash

가장 중요한 상황이다.

```text
Operation = PREPARED

Vault = After State

Sync Store = Before State
```

Recovery Engine은 Vault의 실제 Hash와 Prepared Operation의 `after` 상태를 비교한다.

일치하면:

```text
Filesystem mutation already applied
```

로 판단하고 Sync State Finalize를 수행한다.

새로운 별도의 Client Operation으로 취급하지 않는다.

### 18.4 Finalize 이후 / Client Response 이전 Crash

```text
Operation = COMMITTED
Revision = 1001

Client did not receive response
```

Client는 동일 Operation ID로 Retry한다.

Server는:

```text
OP-X already committed

resultRevision = 1001
```

을 반환한다.

새로운 Revision을 생성하지 않는다.

---

## 19. Recovery Ambiguity

Recovery Engine이 다음 어느 상태와도 일치하지 않는 Filesystem State를 발견할 수 있다.

```text
Before State X

After State X

Observed State = Unknown
```

예:

```text
Prepared:

beforeHash = AAA
afterHash  = BBB

Filesystem:

hash = CCC
```

이 경우 Server가 임의로 파일을 덮어써서는 안 된다.

Server는 해당 경로를:

```text
RECOVERY_REQUIRED
```

상태로 두고 정상 mutation을 차단한다.

원인을 확인하거나 Full Reconciliation을 수행해야 한다.

**애매한 Recovery에서 데이터 보존을 자동 복구보다 우선한다.**

---

## 20. Single Commit Order

Global Revision은 전체 Server에 대한 하나의 순서를 가져야 한다.

Commit Finalization은 논리적으로 직렬화한다.

```text
Client A ─┐
          │
Client B ─┼──► Commit Coordinator
          │
Client C ─┘
                 │
                 ▼
             Revision
```

예:

```text
Client A MODIFY A.md
Client B CREATE B.md
Client C DELETE C.md
```

동시에 요청되더라도 최종적으로:

```text
1001 MODIFY A.md
1002 DELETE C.md
1003 CREATE B.md
```

와 같은 하나의 명확한 순서를 가진다.

어떤 순서가 선택되는지는 중요하지 않다.

**모든 Observer가 동일한 순서를 보는 것**이 중요하다.

---

## 21. Concurrency Model

모든 Upload나 Hash 계산까지 직렬화할 필요는 없다.

다음 작업은 병렬 수행할 수 있다.

```text
Network Receive
Content Streaming
Hash Calculation
Staging
Read
```

하지만 authoritative mutation을 확정하는 Commit Phase는 일관된 순서를 가져야 한다.

개념적으로:

```text
Parallel Work

Client A ── Stage ─┐
Client B ── Stage ─┼──► Serialized Commit
Client C ── Stage ─┘
```

초기 개인용 시스템에서는 복잡한 분산 Lock보다 단순한 Single Commit Coordinator를 우선한다.

---

## 22. Path Lock

동일 파일 또는 Rename의 source/destination에 동시에 여러 Operation이 적용되는 것을 방지해야 한다.

예:

```text
Client A → MODIFY A.md

Client B → DELETE A.md
```

두 Commit이 동시에 Filesystem에 적용되어서는 안 된다.

Server는 mutation 대상 경로에 대한 logical lock을 가진다.

Rename:

```text
A.md → B.md
```

의 경우:

```text
A.md
B.md
```

두 경로를 모두 보호해야 한다.

여러 경로를 Lock할 경우 고정된 Path Ordering을 사용하여 Deadlock을 방지한다.

---

## 23. Read Consistency

Filesystem Apply와 Sync State Finalize 사이에는 짧은 내부 중간 상태가 존재할 수 있다.

```text
Filesystem = After

DB = Before
```

이 상태가 외부 Client에게 정상적인 committed state로 노출되어서는 안 된다.

따라서 API의 Read / Manifest / Change 조회는 Commit Engine과 일관된 synchronization boundary를 가져야 한다.

Client가 관찰할 수 있는 상태는 항상:

```text
Committed State
```

여야 한다.

PREPARED 또는 APPLYING 상태는 내부 Recovery State이다.

---

## 24. Server Startup

Server는 API 요청을 받기 전에 Recovery를 완료해야 한다.

시작 순서는 개념적으로 다음과 같다.

```text
1. Server Instance Lock

2. Open Sync Store

3. Recover Incomplete Operations

4. Initial Vault Import (명시적으로 요청한 경우만)

5. Validate Vault vs File Index

6. Report Drift

7. Determine Current Revision

8. Start API

9. Start Notification Service
```

Recovery가 완료되지 않은 상태에서 Client mutation을 받아서는 안 된다.

---

## 25. Single Server Instance

하나의 Vault는 동시에 하나의 Server Process만 관리한다.

```text
Vault

  ▲

Server A   O
Server B   X
```

Server 시작 시 Instance Lock을 획득한다.

이미 다른 Server가 Vault를 관리하고 있다면 두 번째 Instance는 시작하지 않는다.

이는 다음 문제를 피한다.

* Revision 경쟁
* SQLite 동시 Server 관리
* Filesystem Mutation 경쟁
* Recovery 충돌

High Availability Server는 제품 목표가 아니다([00 Product Specification](./00_product-specification.md)의 비목표).

---

## 26. Server Vault 쓰기 경계

서버 Vault는 Sync API를 통해서만 변경한다. Server Mutation Engine이 서버 Vault를 쓰는 유일한 주체이다.

```text
Client / API-based Tool
   │
   ▼
Sync API
   │
   ▼
Mutation Engine
   │
   ▼
Vault
```

Server가 관리하는 Vault를 다음 방법으로 직접 수정하는 것은 지원하지 않는다. Server 실행 여부와 관계없다.

```text
SSH / Shell

Editor / IDE Workspace

Script

Filesystem Sync Tool

Administrator Manual Edit
```

직접 수정은 Base Revision 검증, Conflict 감지, Change Journal 기록, Idempotency를 모두 우회한다. 서버가 직접 수정을 새 변경으로 편입하면 Client의 Pending 변경을 Conflict 없이 덮어쓸 수 있고, Volume 미마운트나 부분 복원이 대량 DELETE로 전파될 수 있다. 따라서 서버는 직접 수정을 Journal에 편입하지 않는다. 유일한 예외는 Journal이 비어 있는 새 Vault에 기존 자료를 한 번 들여오는 26.4 Initial Vault Import이다.

운영자나 도구가 콘텐츠를 추가하거나 바꾸려면 Client처럼 Sync API의 Operation을 제출한다. Backup 복원은 Vault 내용을 Change로 편입하지 않는다. Data Root를 Backup으로 통째로 교체하는 시작 절차이며 33절을 따른다.

### 26.1 Vault Drift

File Index가 기록한 상태와 실제 Vault Filesystem이 다르면 이를 **Vault Drift**라고 한다.

예:

```text
File Index:

A.md hash = AAA

Filesystem:

A.md hash = BBB
```

Drift는 정상적인 동기화 상태가 아니라 Integrity 결함이다. 서버는 Drift를 다음 중 어느 방식으로도 자동 해소하지 않는다.

* Filesystem의 BBB를 AAA로 되돌리기
* BBB를 새 Revision으로 Journal에 편입하기
* 이후 Mutation으로 BBB를 덮어쓰기

### 26.2 Mutation 전 검증

Mutation Engine은 Operation을 PREPARED로 기록하기 전에 대상 경로의 Filesystem 상태가 File Index와 일치하는지 확인한다.

```text
CREATE        대상 경로에 Filesystem 항목이 없어야 한다
MODIFY        현재 파일 Hash가 File Index와 같아야 한다
DELETE        현재 파일 Hash가 File Index와 같아야 한다
RENAME / MOVE Source Hash가 같고 Destination이 없어야 한다
```

일치하지 않으면 PREPARED Operation을 만들지 않고 요청을 `RECOVERY_REQUIRED`로 거부한다. Drift 때문에 Recovery가 해석할 수 없는 PREPARED Operation이 생기거나, 서버 재시작이 실패해서는 안 된다. Client는 해당 변경을 Pending으로 보존하고 나중에 다시 시도한다.

### 26.3 Drift 해소

Drift는 운영자가 확인하여 해소한다.

* 직접 수정이 실수였다면 Filesystem을 File Index가 기록한 내용으로 되돌린다.
* 직접 수정한 내용을 보존해야 한다면 Vault 밖으로 옮겨 File Index가 기록한 상태로 되돌린 뒤, 그 내용을 Sync API Operation으로 다시 제출한다.

어느 경우에도 서버는 운영자 확인 없이 Content를 삭제하거나 덮어쓰지 않는다.

### 26.4 Initial Vault Import

다른 저장소에서 VaultDatum으로 이전할 때는 기존 Vault 파일을 새 Server Data Root의 Vault 디렉터리에 복사한 뒤, 서버가 이를 한 번에 Journal에 편입할 수 있다. 이것이 Filesystem 상태를 Change로 편입하는 유일한 경로이며, 결과 Change의 Actor는 `SERVER_EXTERNAL`이다.

**실행 조건**

* 운영자가 `VAULTDATUM_INITIAL_IMPORT=true`로 서버를 시작한 경우에만 수행한다. 자동으로 수행하지 않는다.
* Current Revision이 0이고 File Index와 Operation 기록이 비어 있어야 한다.
* 조건을 만족하지 않는데 플래그가 켜져 있으면 서버는 시작을 거부한다. 이미 사용 중인 Vault에서 플래그가 남아 있거나, Sync State를 잃은 뒤 운영자 확인 없이 새 계보로 다시 가져오는 일을 막기 위해서다.
* Vault 디렉터리가 비어 있으면 잘못된 Volume일 가능성이 높으므로 시작을 거부한다.

**사전 검사**

가져오기 전에 Vault 전체를 검사한다. Symlink는 따라가지 않는다. 다음 항목이 하나라도 있으면 아무것도 기록하지 않고 시작을 거부하며, 해당 경로와 사유를 보고한다.

```text
이름이 `.`으로 시작하는 파일 또는 디렉터리 (.obsidian/, .git/ 등)

Symlink, Socket 등 일반 파일이나 디렉터리가 아닌 항목

설정된 최대 Content 크기를 넘는 파일

Sync Path 규칙을 위반하는 경로

읽을 수 없는 항목
```

일부만 가져오면 남은 항목이 이후 Integrity Scan에서 계속 Drift로 보고되므로, 부분 가져오기는 하지 않는다. 운영자가 항목을 정리한 뒤 다시 시작한다.

**편입**

* 일반 파일은 경로 순서대로 각각 하나의 `CREATE` Change가 된다.
* 파일이 없는 빈 디렉터리는 Directory `CREATE` Change가 된다. 파일을 포함한 디렉터리는 암묵적 부모이므로 별도 Change를 만들지 않는다.
* Operation ID는 서버가 생성한다.
* 해시 계산 전후에 파일 크기와 수정 시각이 달라지면 가져오기를 중단한다.
* 전체 편입은 하나의 SQLite Transaction으로 Commit한다. 도중에 실패하거나 Process가 종료되면 아무것도 기록되지 않으며, 같은 조건으로 다시 실행할 수 있다.
* Vault Filesystem은 변경하지 않는다.

가져오기가 끝나면 운영자는 플래그를 끄고 서버를 다시 시작한다. 이후의 직접 수정은 26.1의 Drift로 취급한다. 운영 절차는 [11 Observability and Operations](./11_observability-and-operations.md)을 따른다.

---

## 27. Integrity Scan

Integrity Scan은 실제 Vault Filesystem과 File Index를 비교하여 Drift를 **감지하고 보고만** 한다. Journal, File Index, Vault Filesystem을 변경하지 않는다.

Server Startup에서는 빈 Implicit Parent 정리([06 Data Model](./06_data-model.md) 9.1절)를 마친 뒤 Integrity Scan을 실행한다. 따라서 삭제나 이동 뒤 남은 빈 부모 디렉터리는 Drift로 보고되지 않는다.

다음 시점에 수행할 수 있다.

```text
Server Startup

Operator Request

Periodic Check (선택)
```

감지 대상:

```text
Expected Present / Actual Missing

Expected Hash / Actual Hash mismatch

Unknown File discovered

Entry Type mismatch
```

File Watcher는 사용하지 않는다. Drift를 새 변경으로 편입하지 않으므로 빠른 감지가 필요하지 않고, 감지는 Integrity Scan과 26.2의 Mutation 전 검증으로 충분하다.

보고 형식과 운영 절차는 [11 Observability and Operations](./11_observability-and-operations.md)을 따른다.

---

## 28. Manifest

Server는 현재 synchronized Vault State를 표현하는 Manifest를 생성할 수 있어야 한다.

개념적으로:

```text
Server Revision = 5000

files:

notes/a.md
    revision = 4990
    hash = AAA

notes/b.md
    revision = 4995
    hash = BBB

archive/c.md
    revision = 5000
    hash = CCC
```

Manifest는 주로 다음 용도로 사용한다.

* Initial Sync
* Full Reconciliation
* Integrity Recovery
* Client State 손상 복구

일반적인 동기화에서는 Change Journal 기반 Incremental Pull을 우선한다.

---

## 29. Retention 경계와 Content History

Change Journal, Tombstone, completed operation, recovery artifact의 retention과 GC 기준은 [06 Data Model](./06_data-model.md) 및 [08 Persistence Design](./08_persistence-design.md)을 단일 기준으로 사용한다.

Server Architecture가 요구하는 결과는 하나다. incremental history를 사용할 수 없게 된 Client는 안전한 Full Reconciliation 또는 bootstrap으로 전환해야 하며, retention 때문에 오래된 파일을 새 CREATE로 오인해서는 안 된다.

### 29.1 Content History

Server는 MODIFY와 DELETE로 대체되거나 삭제된 파일의 이전 내용을 Content History로 보관한다. 사용자가 파일 하나를 과거 버전으로 되돌릴 수 있게 하기 위해서다.

```text
MODIFY / DELETE의 이전 내용
      │
      ▼
recovery artifact (commit 전까지 복구용)
      │ commit 완료
      ▼
Content History로 이동
```

* Content History는 동기화 correctness에 필요하지 않다. 보관 실패가 commit을 되돌리거나 Client 응답을 실패로 만들어서는 안 된다.
* commit 뒤 이동 전에 Process가 종료되면 Server Startup Recovery가 남은 recovery artifact를 Content History로 옮긴다. 옮기기 전에 Hash를 검증한다.
* Content History는 Vault 밖에 있으므로 동기화 대상도 Integrity Scan 대상도 아니다.
* Server는 이력을 읽는 API만 제공한다. 되돌리기는 Client가 기존 MODIFY 또는 CREATE Operation으로 제출하며, 이는 26절의 쓰기 경계를 그대로 따른다.

저장 형식과 GC는 [08 Persistence Design](./08_persistence-design.md), 읽기 API는 [07 API Specification](./07_api-specification.md)을 따른다.

---

## 30. Storage & Container Deployment Contract

Server는 Docker Container 또는 Kubernetes Pod와 같이 ephemeral한 실행 환경에서 동작할 수 있다.

Container의 writable layer는 authoritative data 저장소로 사용하지 않는다.

애플리케이션은 하나의 persistent filesystem을 **Data Root**로 제공받는 것을 기본 배포 모델로 한다.

기본 경로는 다음과 같이 정의할 수 있다.

```text
DATA_ROOT=/data
```

Server의 persistent data는 모두 Data Root 아래에 위치한다.

```text
/data/

├── vault/
├── state/
│   └── sync.db
├── staging/
├── recovery/
└── history/
```

Container Image는 실행 코드와 기본 설정만 포함하며 persistent state를 포함하지 않는다.

```text
Container

/app
    └── Server Binary

/data
    └── Persistent Filesystem
```

### 30.1 하나의 Persistent Filesystem

`vault`, `state`, `staging`, `recovery`, `history`는 기본적으로 하나의 persistent filesystem 아래에 배치한다. recovery artifact를 Content History로 atomic rename하기 때문이다.

```text
              Persistent Filesystem
                       │
                       ▼
                     /data
          ┌────────────┼────────────┐
          ▼            ▼            ▼
        vault        staging      state
                                   │
                                sync.db
```

특히:

```text
staging → vault
```

사이에서 atomic rename 또는 atomic replace를 사용할 수 있어야 하므로 서로 다른 filesystem에 분리하지 않는 것을 기본 계약으로 한다.

다음과 같은 구성은 권장하지 않는다.

```text
/data/vault      → Filesystem A
/data/staging    → Filesystem B
/data/state      → Filesystem C
```

### 30.2 Storage 요구사항

Data Root가 사용하는 filesystem은 최소 다음 특성을 제공해야 한다.

```text
Container / Process 재시작 이후 데이터 유지

Server 재배포 이후 데이터 유지

Atomic rename within DATA_ROOT

Durable file synchronization semantics

Normal filesystem locking semantics

Single writable Server ownership
```

Server가 성공 응답을 반환한 데이터는 Container 재생성만으로 유실되어서는 안 된다.

### 30.3 배포 프로필

Docker, Kubernetes, Volume/PVC 구성, Network Filesystem 지원 범위, `DATA_ROOT`의 실제 배치는 [09 Security and Deployment](./09_security-and-deployment.md)의 배포 계약을 단일 기준으로 사용한다.

Server 코드는 배포 기술을 인식하지 않고, 위 30.1~30.2의 filesystem 보장을 가진 persistent `DATA_ROOT`만 전제한다. 구체적인 SQLite 파일과 recovery artifact의 배치는 [08 Persistence Design](./08_persistence-design.md)을 따른다.

---

## 31. Internal Server Data

Server 관리 데이터는 동기화 대상 Vault 밖에 두며 Client-facing API로 노출하지 않는다. 실제 저장 레이아웃은 [08 Persistence Design](./08_persistence-design.md), `DATA_ROOT` 배치는 [09 Security and Deployment](./09_security-and-deployment.md)을 따른다. `.obsidian/`의 동기화 제외 규칙은 [02 Synchronization Protocol](./02_synchronization-protocol.md)이 기준이다.

---

## 32. Durability Boundary

Server가 durable commit이라고 판단하려면 최소 다음 조건이 만족되어야 한다.

```text
New file content durable

Filesystem metadata durable

Sync State durable
```

파일을 단순히 write하고 OS Page Cache에 남아 있는 상태만으로 성공 처리해서는 안 된다.

구체적인 `fsync` 호출 위치와 SQLite durability 설정은 구현 명세에서 정의하지만 Server Architecture는 다음 보장을 요구한다.

> Server가 성공을 반환한 변경은 정상적인 Process Crash나 OS Reboot 이후에도 복구 가능해야 한다.

---

## 33. Server Backup

Server는 마지막 정상 상태의 Backup 하나를 `/data/backups/current/`에 유지한다. Backup은 Vault, Sync State, Content History의 일관된 한 시점이다. 사용자 실수는 Content History로 되돌리고, Backup은 버그나 저장소 문제로 Vault나 Sync State 자체가 망가졌을 때 쓰는 마지막 수단이다. 운영 절차와 설정은 [11 Observability and Operations](./11_observability-and-operations.md) 32–33절을 따른다.

### 33.1 구성

```text
/data/backups/
  current/              유일한 Backup
    backup.json         vaultId, revision, createdAt, trigger, serverVersion,
                        schemaVersion, files, bytes, syncDbHash
    vault/  history/  recovery/  staging/
    state/sync.db
  .partial/             만드는 중. Backup으로 보지 않는다
  pre-restore-<time>/   복원 직전 상태
```

Server는 Vault 파일, Content History 객체, Recovery Artifact, Staged File을 제자리에서 고쳐 쓰지 않는다. 모든 변경은 새 파일을 만들어 원자적으로 옮긴다. 따라서 Backup은 이 파일들을 복사하지 않고 hard link로 만든다. 이후 원래 경로가 바뀌거나 지워져도 Backup의 inode는 그대로이고, 바뀌지 않은 파일은 디스크를 추가로 쓰지 않는다. hard link를 만들 수 없는 Filesystem에서는 복사한다. `sync.db`는 제자리에서 바뀌므로 link하지 않고 SQLite `VACUUM INTO`로 일관된 사본을 만든다.

일반 파일과 디렉터리가 아닌 항목은 원래 Drift이므로 건너뛰고 경로를 로그에 남긴다. Access Token은 Data Root 밖에 있으므로 Backup에 들어가지 않는다.

### 33.2 갱신

```text
1. MutationLock을 잡는다 (읽기와 Pull은 계속된다)
2. 정상 상태를 확인한다. 실패하면 기존 Backup을 유지하고 끝낸다
3. .partial/ 에 VACUUM INTO로 sync.db를 만들고
   vault, history, recovery, staging을 hard link로 복제한다
4. MutationLock을 푼다
5. 사본 DB를 열어 vaultId와 Revision이 같은지 확인하고 backup.json을 쓴다
6. current/ 를 .old/ 로, .partial/ 을 current/ 로 옮긴 뒤 .old/ 를 지운다
```

1–4단계를 Lock 안에서 하므로 Backup의 Path State와 Vault 트리는 정확히 일치한다. 6단계 중 Server가 멈추면 다음 시작에서 `.old/`를 `current/`로 되돌리고 `.partial/`을 지운다.

Backup이 하나뿐이므로 망가진 상태가 정상 Backup을 덮어쓰지 않도록, 기존 Backup이 있을 때는 대체하기 전에 다음을 확인한다. 기존 Backup이 없으면 지킬 것이 없으므로 이 검사 없이 만든다.

| 검사 | 조건 | 막는 상황 |
| --- | --- | --- |
| Vault 일치 | `vaultId`가 Backup과 같거나, Backup의 vaultId가 이 Vault의 이전 ID다 | 다른 Vault의 DB로 덮어쓰기 |
| Revision | 현재 Revision ≥ Backup의 Revision | 과거로 돌아간 DB로 덮어쓰기 |
| 변경 여부 | vaultId와 Revision이 Backup과 같으면 건너뛴다 | 같은 상태의 불필요한 갱신 |
| Vault 존재 | Path State에서 PRESENT인 모든 항목이 같은 타입으로 있고, 파일 크기가 기록과 같다 | 파일이 사라진 Vault로 덮어쓰기 |

주기 갱신은 PREPARED Operation이 있으면 건너뛴다. Vault 존재 검사는 해시를 계산하지 않으므로 큰 Vault에서도 빠르다. 모든 검사는 처음 Schema부터 있던 열만 읽으므로 Migration 전에도 실행할 수 있다.

갱신 시점:

* **주기**: 마지막 Backup의 `createdAt`부터 설정한 간격이 지나면 갱신한다.
* **Migration 직전**: 시작할 때 적용할 Migration이 있으면 Recovery와 Migration 전에 갱신한다. 이전 Server가 남긴 PREPARED Operation이 함께 들어갈 수 있으며, 복원한 뒤 시작하면 일반 Recovery가 처리한다. Backup을 만들 수 없으면 Schema를 바꾸지 않고 시작을 거부한다.

주기 갱신의 실패나 건너뜀은 동기화 correctness에 영향을 주지 않는다. 경고를 남기고 다음 간격에 다시 시도한다.

### 33.3 복원

복원은 `VAULTDATUM_RESTORE_BACKUP=true`로 시작할 때만 하며, DB를 열기 전에 수행한다.

```text
1. backup.json, sync.db Hash, Vault 파일 수와 크기를 확인한다
   → 맞지 않으면 아무것도 바꾸지 않고 시작을 거부한다
2. 현재 vault, state, history, recovery, staging을
   /data/backups/pre-restore-<time>/ 로 옮긴다 (지우지 않는다)
3. Backup을 Data Root에 hard link로 복제하고 sync.db는 복사한다
4. DB를 열고 필요한 Migration을 적용한다
5. 새 Vault ID를 발급하고 이전 Vault ID를 기록한다. Manifest를 비운다
6. 일반 시작 절차를 계속한다
```

2–3단계가 실패하면 옮긴 상태를 되돌리고 시작을 거부한다. 복원해도 Backup 자체는 바뀌지 않는다. 마지막으로 복원한 Backup과 같은 Backup으로 다시 복원하라는 요청은 거부한다. 플래그를 끄지 않고 재시작해 복원 뒤의 변경을 덮어쓰는 일을 막기 위해서다.

복원하면 Revision은 Backup 시점으로 돌아가지만 장치는 그 뒤 Revision까지 받았을 수 있다. Vault ID가 그대로이면 장치의 Cursor가 Server보다 앞서거나, 복원 뒤 다시 쓰인 Revision 번호를 장치가 이미 받은 것으로 보고 건너뛴다. 새 Vault ID를 발급하면 [06 Data Model](./06_data-model.md) Invariant 15에 따라 기존 Cursor와 Replica Index를 쓰지 않게 된다. 이전 Vault ID는 `GET /api/v1/vault`의 `previousVaultIds`로 알려 장치가 복원을 알아보고 다시 연결하게 한다([04 Client Architecture](./04_client-architecture.md) 49절).

---

## 34. Security Boundary

Server의 입력 검증과 Vault Root 경계는 [09 Security and Deployment](./09_security-and-deployment.md)을 단일 기준으로 사용한다. 이 아키텍처의 책임은 검증된 요청만 Mutation 경로로 전달하고, `.obsidian/`을 포함한 제외 경로를 authoritative content로 취급하지 않는 것이다.

HTTP/WebSocket boundary 앞에는 작은 `AccessAuthenticator`를 둔다.

```text
HTTP / WebSocket request
        │
        ▼
AccessAuthenticator
        │ AccessContext (profile)
        ▼
API Resource
        │
        ▼
Sync Coordinator
```

`AccessAuthenticator`만 Authorization header parsing, configured Vault token의
constant-time comparison, realtime ticket issue/consume을 수행한다. API Resource와 Sync
Coordinator가 header 문자열이나 token plaintext를 받지 않는다. authenticated
`AccessContext`는 profile 정보만 전달한다.

Server configuration은 `private-network` 또는 `public-token` profile을 명시적으로
선택한다. `public-token`은 non-empty Vault token file 없이는 ready가 될 수 없다. token
plaintext를 environment variable, application log, exception message에 넣지 않고 Secret
volume file에서 읽는다. health probe의 제한된 예외를 제외한 Sync API는 이 boundary를
우회할 수 없다.

---

## 35. Server Observability

Server는 PREPARED operation, Vault/File Index 불일치, recovery ambiguity, Sync Store corruption을 정상 상태와 구분해 외부에 진단 가능하게 해야 한다. 노출할 상태, 로그와 health/readiness 계약, 운영 절차는 [11 Observability and Operations](./11_observability-and-operations.md)을 단일 기준으로 사용한다.

---

## 36. Server Failure Policy

Server Architecture는 다음 우선순위를 가진다.

```text
1. 사용자 Content 보존

2. Authoritative State 일관성

3. Recovery 가능성

4. Availability

5. Performance
```

따라서 모호한 상태에서 Server가 계속 요청을 받아 잘못된 변경을 전파하는 것보다 일부 기능을 중단하고 Recovery를 요구하는 것을 선택한다.

---

## 37. Server Architecture Invariants

### Invariant 1 — Filesystem Is Canonical Content

사용자 Markdown과 Attachment의 현재 내용은 Vault Filesystem이 원본이다.

### Invariant 2 — Sync Metadata Is Durable

Revision, Change, Tombstone, Operation 등 동기화에 필요한 metadata는 Process Memory에만 존재해서는 안 된다.

### Invariant 3 — Single Mutation Path

정상적인 authoritative mutation은 반드시 Server Mutation Engine을 통과한다.

### Invariant 4 — Prepare Before Apply

Filesystem을 변경하기 전에 복구 가능한 Operation Intent가 durable하게 기록되어야 한다.

### Invariant 5 — Commit Before Success

Filesystem과 Sync State가 durable하게 확정되기 전에 Client에게 성공을 반환하지 않는다.

### Invariant 6 — Crash Is Recoverable

Commit 과정 어느 지점에서 Server가 종료되더라도 다음 실행에서 Operation의 상태를 판단할 수 있어야 한다.

### Invariant 7 — Revision Represents Committed Change

외부 Client에게 노출되는 Global Revision은 authoritative state에 확정된 변경을 의미한다.

### Invariant 8 — Retry Does Not Recommit

이미 Commit된 Operation을 Retry해도 새로운 Revision이나 중복 Filesystem Mutation을 생성하지 않는다.

### Invariant 9 — No Partial File Write

Client에게 보이는 Vault 파일은 완전히 이전 Content이거나 완전히 새로운 Content여야 한다.

부분적으로 기록된 Content가 정상 상태로 노출되어서는 안 된다.

### Invariant 10 — Vault Writes Only Through the Sync API

서버 Vault는 Sync API와 Mutation Engine을 통해서만 변경한다. Server가 알지 못하는 Vault 변경을 발견하면 되돌리거나, Journal에 편입하거나, 덮어쓰지 않고 Drift로 보고한다. 유일한 예외는 빈 Journal에 대한 명시적 Initial Vault Import이다.

### Invariant 11 — Read Only Committed State

Client-facing Server API는 내부 PREPARED/APPLYING 상태를 정상적인 authoritative state로 노출하지 않는다.

### Invariant 12 — One Server Owns One Vault

하나의 synchronized Vault는 동시에 하나의 Server Instance만 관리한다.

### Invariant 13 — Persistent Data Lives Outside Container Lifecycle

Vault, Sync State 및 Recovery에 필요한 데이터는 Container 또는 Pod의 lifecycle과 독립적으로 유지되어야 한다.

### Invariant 14 — Data Root Provides Atomic Filesystem Semantics

Commit Protocol이 의존하는 `vault`, `staging`, `recovery` 영역은 기본적으로 동일한 persistent filesystem의 atomic rename semantics를 사용할 수 있어야 한다.

---

## 38. 전체 Mutation Flow

서버의 mutation path를 요약하면 다음과 같다.

```text
                    Client
                      │
                      │ Operation
                      ▼
                Sync API
                      │
                      ▼
              Sync Coordinator
                      │
                Base Validation
                      │
                      ▼
                  Staging
                      │
                      ▼
               PREPARED Record
                      │
                      ▼
              Filesystem Apply
                      │
                      ▼
               Sync State Commit
                      │
                      ├── File Index
                      ├── Change Journal
                      ├── Revision
                      └── Operation COMMITTED
                      │
                      ▼
                Success Response
```

Crash가 발생하면:

```text
PREPARED Operation
        +
Filesystem Observation
        +
Staged / Recovery Data
        │
        ▼
   Recovery Engine
        │
        ▼
Authoritative State 복구
```
