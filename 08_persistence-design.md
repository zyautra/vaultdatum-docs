# Persistence Design

## 1. 문서 목적

이 문서는 논리적 Data Model을 Server와 Client의 실제 Persistent Storage 구조로 매핑한다.

Persistence Layer의 목적은 단순히 데이터를 저장하는 것이 아니다.

다음을 보장해야 한다.

```text
Server

Committed Authoritative State 보존
Crash Recovery
Operation Idempotency
Change History
Tombstone
Manifest Snapshot


Client

Offline Change 보존
Replica State 보존
Cursor 보존
Remote Apply Recovery
Conflict 보존
App Kill 이후 복구
```

Persistence 구현은 상위 Synchronization Protocol의 correctness를 변경해서는 안 된다.

논리 entity와 필드는 [06 Data Model](./06_data-model.md), 상태 전이 요구사항은 [02 Synchronization Protocol](./02_synchronization-protocol.md)의 단일 기준을 따른다. 이 문서는 이를 storage schema와 atomic transaction으로 옮기는 방법만 정의한다.

---

# 2. Persistence 구성

전체 구조는 다음과 같다.

```text
SERVER

/data/
├── vault/
├── state/
│   └── sync.db
├── staging/
└── recovery/


CLIENT

Obsidian Vault
    =
사용자 Content

Client Sync Store
    =
IndexedDB

Plugin Settings
    =
Obsidian Plugin Data

Client Artifact Store
    =
Client-local Storage
```

Server와 Client는 서로 다른 persistence 기술을 사용하지만 동일한 논리적 Data Model을 표현한다.

---

# 3. Server Persistence

Server Persistence는 두 영역으로 나눈다.

```text
Vault Filesystem

+

SQLite Sync Store
```

역할:

```text
Vault Filesystem
    =
Canonical User Content

SQLite
    =
Authoritative Sync Metadata
```

둘을 하나의 recoverable authoritative state로 관리한다.

---

# 4. SQLite 위치

기본 위치:

```text
/data/state/sync.db
```

SQLite Database와 Vault는 동일한 persistent Data Root 아래 존재한다.

```text
/data/

├── vault/
├── state/
├── staging/
└── recovery/
```

---

# 5. SQLite 기본 설정

초기 Server에서는 다음 설정을 사용한다.

```text
journal_mode = WAL

synchronous = FULL

foreign_keys = ON
```

Server는 데이터 손실 방지를 성능보다 우선한다.

따라서 Sync Metadata Commit은 OS Crash 또는 갑작스러운 전원 손실까지 고려한 durability 설정을 기본값으로 한다.

---

## 5.1 WAL의 목적

SQLite WAL은 다음을 제공한다.

```text
Read / Write concurrency

Atomic Transaction

Crash Recovery
```

하지만 SQLite WAL과 Server Architecture의:

```text
PREPARED Operation
```

은 다른 개념이다.

```text
SQLite WAL
    =
SQLite 내부 Transaction Recovery

Server PREPARED Operation
    =
Filesystem + SQLite 사이의
Cross-resource Recovery Protocol
```

둘을 혼동하지 않는다.

---

# 6. Single Writer

Authoritative Mutation Finalization은 하나의 logical writer를 통해 수행한다.

```text
Commit Coordinator
        │
        ▼
SQLite Write Transaction
```

Read Transaction은 병렬로 수행할 수 있다.

이를 통해:

```text
Revision Allocation

Path State Update

Change Journal Append
```

순서를 단순하게 유지한다.

---

# 7. Schema Migration

Database Schema는 명시적인 Version을 가진다.

```text
schema_migrations
```

개념 구조:

```text
version
applied_at
```

Server Startup 시:

```text
Open Database
      │
      ▼
Read Schema Version
      │
      ▼
Apply Required Migrations
      │
      ▼
Recovery
```

순서로 진행한다.

지원하지 않는 미래 Schema Version을 발견하면 Server를 시작하지 않는다.

---

# 8. vault_metadata

Server Vault 전체 Metadata를 저장한다.

논리 구조:

```text
vault_metadata

id
vault_id
current_revision
```

하나의 Server Vault에서는 하나의 row만 존재한다.

예:

```text
id = 1
vault_id = V-123
current_revision = 1205
```

---

# 9. path_state

현재 Server Path State를 저장한다.

논리 구조:

```text
path_state

path
entry_type
state
latest_revision
content_hash
size
last_content_hash
```

Primary Key:

```text
path
```

---

## 9.1 PRESENT File

예:

```text
path = notes/a.md
entry_type = FILE
state = PRESENT
latest_revision = 100
content_hash = AAA
size = 5210
```

---

## 9.2 DELETED File

예:

```text
path = notes/a.md
entry_type = FILE
state = DELETED
latest_revision = 105
content_hash = NULL
last_content_hash = AAA
```

`last_content_hash`는 optional recovery / diagnosis information이다.

---

## 9.3 Directory

Directory:

```text
entry_type = DIRECTORY
```

인 경우:

```text
content_hash = NULL
size = NULL
```

이다.

---

# 10. change_journal

Committed logical Change를 저장한다.

구조:

```text
change_journal

revision
change_type
operation_id
actor_type
actor_client_id
source_path
destination_path
committed_at
```

Primary Key:

```text
revision
```

Revision은 Global Commit Order다.

---

# 11. change_effect

하나의 Change가 여러 Path에 미치는 결과를 저장한다.

구조:

```text
change_effect

revision
ordinal
path
entry_type
state
content_hash
size
```

Primary Key:

```text
revision + ordinal
```

Foreign Key:

```text
revision
    →
change_journal.revision
```

---

## 11.1 MODIFY

```text
Revision 100

Effect 0

A.md
PRESENT
hash = BBB
```

---

## 11.2 RENAME

```text
Revision 101

Effect 0
A.md
DELETED

Effect 1
B.md
PRESENT
hash = BBB
```

두 Effect가 하나의 Change Revision에 속한다.

---

# 12. operations

Operation Idempotency와 Recovery 상태를 저장한다.

구조:

```text
operations

operation_id
actor_client_id
operation_type
request_digest

status

staging_reference
recovery_reference

result_revision

created_at
completed_at
```

Primary Key:

```text
operation_id
```

---

# 13. operation_base_condition

Operation이 의존한 Server State를 저장한다.

구조:

```text
operation_base_condition

operation_id
ordinal

path

expected_state
expected_revision
expected_hash
```

Primary Key:

```text
operation_id + ordinal
```

Foreign Key:

```text
operation_id
    →
operations.operation_id
```

Rename과 같이 여러 Path를 검증하는 Operation은 여러 row를 가진다.

---

# 14. Operation Request Identity

`operation_id`만 저장하지 않는다.

다음도 함께 저장한다.

```text
request_digest
```

Retry Request가 들어왔을 때:

```text
operationId same
requestDigest same
```

이면 동일 Operation Retry다.

반면:

```text
operationId same
requestDigest different
```

이면:

```text
OPERATION_ID_REUSED
```

오류다.

---

# 15. PREPARED Transaction

Server Commit의 Prepare 단계에서는 하나의 SQLite Transaction으로 최소 다음을 기록한다.

```text
operations

operationId
requestDigest
baseConditions
stagingReference
status = PREPARED
```

Commit 후에만 Filesystem Apply를 시작한다.

즉:

```text
PREPARED metadata durable
          │
          ▼
Filesystem mutation
```

순서를 유지한다.

---

# 16. Finalization Transaction

Filesystem Apply가 완료되면 하나의 SQLite Write Transaction에서 다음을 수행한다.

```text
1. currentRevision 읽기

2. newRevision = currentRevision + 1

3. change_journal Insert

4. change_effect Insert

5. path_state Update

6. operation status = COMMITTED

7. operation resultRevision 설정

8. vault_metadata.currentRevision Update

9. Commit
```

이 Transaction 전체가 성공하거나 전체가 실패한다.

---

# 17. Revision Allocation

Revision은 Finalization Transaction 안에서 생성한다.

```text
PREPARED
    X
Revision 없음
```

Filesystem Apply 이후:

```text
Finalize
    │
    ▼
Revision 생성
```

한다.

Single Commit Coordinator와 SQLite Transaction을 이용하여 두 Operation이 같은 Revision을 얻는 것을 방지한다.

---

# 18. Conflict Operation

Base Validation에 실패한 Operation은 Authoritative Change를 만들지 않는다.

필요한 경우:

```text
status = REJECTED_CONFLICT
```

로 Operation Result를 보존할 수 있다.

하지만:

```text
change_journal

path_state

current_revision
```

은 변경하지 않는다.

---

# 19. Manifest Persistence

API에서 Manifest는 여러 HTTP 요청에 걸쳐 같은 Snapshot을 제공해야 한다.

단순히 `path_state`를 page별로 직접 조회하면 Manifest 생성 중 Server 변경이 발생하여 서로 다른 시점의 State가 섞일 수 있다.

따라서 Manifest는 materialize한다.

---

# 20. manifests

Manifest Metadata:

```text
manifests

manifest_id
snapshot_revision
created_at
expires_at
```

---

# 21. manifest_entry

Manifest 생성 시 현재 `path_state`를 Snapshot으로 복제한다.

구조:

```text
manifest_entry

manifest_id
path

entry_type
state
revision
content_hash
size
```

Primary Key:

```text
manifest_id + path
```

---

# 22. Manifest 생성 Transaction

Manifest는 하나의 SQLite Read/Write Transaction 안에서 생성한다.

개념적으로:

```text
BEGIN

snapshotRevision
    =
vault_metadata.current_revision

INSERT manifest

INSERT manifest_entry
    SELECT current path_state

COMMIT
```

이후 Server Revision이 증가하더라도 해당 Manifest 내용은 바뀌지 않는다.

---

# 23. Manifest GC

Manifest는 temporary metadata다.

```text
expires_at < now
```

인 Manifest는 Garbage Collection할 수 있다.

삭제:

```text
manifest_entry

then

manifest
```

Manifest GC는 Authoritative Sync History에 영향을 주지 않는다.

---

# 24. SQLite Index

최소 다음 Access Pattern을 효율적으로 지원해야 한다.

```text
Path State by Path

Changes after Revision

Operation by Operation ID

Prepared Operations

Manifest Entries by Manifest + Path Order
```

따라서 Primary Key 외에 다음과 같은 Logical Index가 필요하다.

```text
operations(status)

change_journal(operation_id)

manifest_entry(manifest_id, path)
```

정확한 Index 정의는 실제 Query Plan 측정 후 조정할 수 있다.

---

# 25. Staging Artifact

CREATE / MODIFY Content는 SQLite에 저장하지 않는다.

Upload는:

```text
/data/staging/<operation-id>
```

형태의 temporary artifact로 기록할 수 있다.

실제 이름은 implementation detail이다.

Staging Artifact는 Operation Metadata와 연결된다.

```text
operations.staging_reference
```

---

# 26. Recovery Artifact

DELETE 또는 Replace Recovery를 위해:

```text
/data/recovery/
```

에 temporary backup을 둘 수 있다.

예:

```text
OP-123
    →
old A.md
```

Operation이 완전히 COMMITTED된 후 더 이상 필요하지 않으면 GC한다.

---

# 27. Orphan Artifact Recovery

Server Startup 시:

```text
staging/

recovery/
```

안의 Artifact와 `operations` 상태를 비교한다.

예:

```text
Artifact exists
Operation exists
status = PREPARED

→ Recovery 대상
```

반면:

```text
Artifact exists
No Operation
```

이면 orphan candidate다.

즉시 삭제하기보다 안전한 GC 정책을 적용한다.

---

# 28. SQLite 파일 자체의 취급

WAL Mode에서는:

```text
sync.db

sync.db-wal

sync.db-shm
```

이 존재할 수 있다.

따라서 실행 중:

```text
sync.db
```

파일 하나만 단순 복사하여 완전한 Backup이라고 간주하지 않는다.

Backup은 SQLite가 보장하는 일관된 Snapshot 방식 또는 Server가 정지된 상태에서 수행해야 한다.

---

# 29. Client Persistence 목표

Client Persistence에서 가장 중요한 것은 다음이다.

```text
Process 종료

Mobile Background Kill

Obsidian Reload

Network Failure
```

후에도 다음 상태를 복원할 수 있는 것이다.

```text
serverCursor

Replica Index

Pending Operations

Apply Journal

Conflicts

Initial Bootstrap State
```

---

# 30. Client Persistence 분리

Client에서는 사용자 설정과 Sync State를 분리한다.

```text
Plugin Settings

≠

Sync Engine State
```

---

# 31. Plugin Settings

다음과 같은 사용자 설정은 Obsidian Plugin Data API를 사용한다.

예:

```text
Server URL

Vault access token (public-token profile only)

Sync Enabled

UI Preferences

Optional Device Display Name
```

논리적으로:

```text
Plugin.loadData()

Plugin.saveData()
```

를 사용한다.

이 영역은 correctness-critical Sync Database로 사용하지 않는다.

---

# 32. Client Sync Store

초기 Client Sync Store는 **IndexedDB**를 사용한다.

선택 이유:

```text
Desktop / Mobile 공통 사용 가능

Node.js 불필요

Transactional Update 지원

Indexed Object Store 지원

Binary / Blob 저장 가능

Process Memory보다 durable
```

Client Core는 IndexedDB API 자체에 직접 강하게 결합하지 않고 Repository Interface 뒤에 둔다.

```text
Sync Engine
     │
     ▼
ClientStore Interface
     │
     ▼
IndexedDB
```

Storage Backend를 바꾸더라도 Sync Algorithm이 바뀌지 않도록 한다.

---

# 33. Client Database Identity

Database 이름은 Plugin과 Vault를 구분할 수 있어야 한다.

개념적으로:

```text
<plugin-id>:<vault-identity>
```

를 사용할 수 있다.

실제 이름 규칙은 구현 단계에서 결정한다.

서로 다른 Local Vault가 같은 Client Sync State를 공유해서는 안 된다.

---

# 34. Client Schema Version

IndexedDB 자체의 Version Migration을 사용한다.

```text
Client DB Version

1
2
3
...
```

Upgrade 중 Object Store와 Index를 Migration한다.

지원하지 않는 Client State를 발견한 경우 기존 Pending / Conflict Content를 무조건 삭제하고 새로 시작해서는 안 된다.

---

# 35. Client Object Stores

논리적으로 다음 Object Store를 사용한다.

```text
metadata

replica

pending

apply

conflict

artifact
```

Resolved Conflict History는 `conflict` 내부 상태로 유지하거나 별도 Store로 분리할 수 있다.

---

# 36. metadata Store

Client Metadata singleton:

```text
clientId

vaultId

serverCursor

schemaVersion

initialBootstrap
```

등을 보존한다.

### 36.1 Initial Bootstrap Metadata

`initialBootstrap`은 [06 Data Model](./06_data-model.md)의 `policyVersion`, `vaultId`, `complete`를 보존한다. 이 metadata가 없거나 `complete = false`이면 Client는 Incremental Pull만으로 초기화를 끝내서는 안 되며 fresh Server Manifest로 Server-first Bootstrap을 다시 시작해야 한다.

`complete = true`는 Pending Operation의 commit 완료가 아니라 Local 경로 분류 완료를 뜻한다. 따라서 Pending Store와 독립적으로 저장하되, 해당 `vaultId`와 함께 검증한다.

---

# 37. replica Store

Key:

```text
path
```

Value:

```text
path
entryType

serverState
serverRevision
serverHash
```

Server Change를 처리할 때 Path별 Server Knowledge를 갱신한다.

---

# 38. pending Store

Key:

```text
operationId
```

Value:

```text
operationType

path
previousPath

baseConditions

localHash

status

resultRevision

createdAt
```

Index 후보:

```text
status

path
```

---

# 39. apply Store

Remote Apply Crash Recovery를 위한 Store다.

Key는 안정적인 Apply ID 또는 Revision 기반 ID를 사용할 수 있다.

Value:

```text
revision

changeType

beforeState

afterState

artifactId

status
```

정상 Apply 완료 후 제거할 수 있다.

---

# 40. conflict Store

Key:

```text
conflictId
```

Value:

```text
conflictType

path

originalOperationId

baseState

latestServerState

localState

localArtifactId

serverArtifactId

mergeArtifactId

status

resolutionOperationId
```

Index 후보:

```text
status

path
```

---

# 41. artifact Store

초기 Markdown Sync에서 Conflict Snapshot, Merge Result 및 Downloaded Content를 보존한다.

개념적으로:

```text
artifactId
artifactType
contentHash
content
createdAt
```

Text는 String 또는 Blob으로 저장할 수 있다.

Attachment Support가 추가되면 Binary Blob을 저장할 수 있도록 Model을 유지한다.

---

# 42. Large Binary Artifact

대용량 Attachment를 IndexedDB에 장기간 복제하지 않는다. Artifact는 전송과 적용에 필요한 동안만 보관한다.

Artifact 저장은 다음을 만족해야 한다.

```text
Mobile Storage Quota

Large Blob Performance

Crash Durability

Temporary Artifact GC
```

Artifact 저장은 ClientStore 경계 뒤에 두어 Sync Algorithm과 분리한다.

---

# 43. Local Change Transaction

Local Change를 Pending으로 만들 때:

```text
Local Vault changed
        │
        ▼
Hash / Analyze
        │
        ▼
IndexedDB Transaction
        │
        └── pending Insert / Update
```

Transaction이 성공한 이후에만 해당 Operation이 Network Push 대상이 된다.

### 43.1 Initial Bootstrap Classification

Initial Bootstrap은 Server Manifest 통합 이후 Local Vault를 scan한다. Server가 `UNKNOWN`으로 확인한 Local 항목은 일반 Local Change와 같은 방식으로 `pending`과 필요한 `artifact`를 durable하게 기록한다. Server `PRESENT` 또는 `DELETED`와 겹치는 Local 항목은 `conflict`와 `replica`의 normal conflict transaction으로 기록한다.

Vault 전체를 하나의 IndexedDB transaction에 넣을 필요는 없다. 다만 다음 순서는 지켜야 한다.

```text
각 CREATE 후보: pending + artifact durable commit
        │
        ▼
모든 Local 경로가 분류됨
        │
        ▼
initialBootstrap.complete = true durable commit
```

Client가 중간에 종료되면 `complete`는 false로 남는다. 다음 실행은 fresh manifest를 읽고, 이미 존재하는 Pending / Conflict를 인식하여 재사용하고 아직 분류되지 않은 항목만 처리한다. 완료 metadata를 먼저 기록하거나, Pending 없이 CREATE를 Network로 보내면 안 된다.

---

# 44. Pending Metadata가 유실된 경우

Client Store는 durable해야 하지만 Local Vault 자체도 하나의 복구 신호다.

예:

```text
Replica:
AAA

Local:
BBB

Pending:
missing
```

인 상태가 발견되면 Local Reconciliation은:

```text
AAA → BBB
```

차이를 다시 Local Change로 발견할 수 있다.

따라서 Client Metadata 손상이 곧 사용자 Content 손실을 의미해서는 안 된다.

---

# 45. Remote Apply Prepare Transaction

Remote Content를 Local Vault에 적용하기 전에:

```text
artifact 저장

+

ApplyRecord PREPARED 저장
```

을 같은 Client Store Transaction에서 가능한 한 함께 수행한다.

```text
Remote Content
      │
      ▼
IndexedDB Transaction

├── Artifact
└── ApplyRecord PREPARED
```

그 후 Local Vault를 변경한다.

---

# 46. Remote Apply Finalize Transaction

Local Vault Apply 성공 후 하나의 Client Transaction에서 다음을 수행한다.

```text
Replica Entry 갱신

Server Cursor 갱신

ApplyRecord 제거

필요한 Pending 상태 갱신
```

한다.

중요한 원칙:

> **Cursor 진행과 해당 Revision의 durable processing state는 하나의 Client Transaction으로 Commit되어야 한다.**

---

# 47. Conflict 처리 Transaction

Server Revision이 Local Pending과 충돌한 경우 하나의 Transaction에서:

```text
Conflict Record 생성

Local Snapshot Artifact 연결

Latest Server State 저장

Replica Entry를 최신 Server State로 갱신

Original Pending → CONFLICT

Server Cursor 진행
```

을 수행한다.

예:

```text
Before:

Replica = 100 / AAA
Local   = CCC
Cursor  = 100
```

Server Change:

```text
101 / BBB
```

처리 후:

```text
Replica = 101 / BBB

Local Vault = CCC

Conflict:

base   = 100 / AAA
server = 101 / BBB
local  = CCC

Cursor = 101
```

즉:

```text
Replica Index
    =
현재 Client가 알고 있는 최신 Server State

Local Vault
    =
Conflict 상태의 사용자 Content
```

로 역할을 분리한다.

---

# 48. 왜 Conflict에서도 Replica를 갱신하는가

Cursor가 Revision 101을 처리했다고 기록했는데 Replica가 Revision 100을 계속 가리키면:

```text
Cursor
    =
101까지 처리됨

Replica
    =
100 상태
```

라는 내부 모순이 생긴다.

Conflict Content는 별도 Conflict Record와 Artifact가 보존하므로 Replica Index가 과거 Base State를 계속 들고 있을 필요가 없다.

Base State는:

```text
Conflict.baseState
```

에 보존한다.

---

# 49. Own Change Integration

Server에서 자신의 Operation Change를 Pull한 경우 하나의 Transaction에서:

```text
Replica Update

Cursor Advance

Pending 제거
```

를 수행한다.

Client가 이미 같은 Content를 Local에 가지고 있다면 Local Vault Write는 하지 않는다.

---

# 50. Cursor Atomicity

다음과 같은 순서는 금지한다.

```text
Cursor = 101 저장

        X Crash

Replica / Conflict 저장 안 됨
```

왜냐하면 재시작 후 Client는 Revision 101을 다시 Pull하지 않기 때문이다.

따라서:

```text
Revision Processing State
        +
Cursor Advance
```

는 반드시 하나의 durable transaction boundary 안에 있어야 한다.

---

# 51. Client Apply Crash

ApplyRecord가:

```text
PREPARED
```

상태인데 Client가 재시작되면 Local Vault Hash를 검사한다.

```text
Local = beforeState
    → Apply 다시 수행

Local = afterState
    → Metadata Finalize

Local = neither
    → Conflict / Recovery Required
```

로 처리한다.

---

# 52. Conflict Artifact Durability

사용자의 Local Conflict Content를 active Vault에서 제거해야 하는 Resolution Action을 수행하기 전에 Recovery Artifact를 durable하게 보존할 수 있다.

특히:

```text
Use Server
```

Action에서 Local Content가 사라지기 전 Snapshot을 저장할 수 있다.

Artifact 저장 실패 시 destructive Resolution을 계속 진행하지 않는 것이 안전하다.

---

# 53. Manual Merge Draft

Manual Merge Result는 새로운 사용자 Content다.

Server Commit 이전까지:

```text
MERGE_RESULT
```

Artifact로 durable하게 저장한다.

Client가 종료되더라도 Merge Result를 다시 불러올 수 있어야 한다.

---

# 54. Client Settings와 Secret

일반 설정은 Plugin Settings에 저장한다. Vault access token도 Desktop과
Mobile에서 공통으로 동작해야 하므로 해당 Obsidian Vault의 Plugin Settings에
client-local로 저장한다. token은 IndexedDB, replica/pending/artifact store, sync
payload에 복제하지 않는다.

Obsidian Plugin Data API가 모든 지원 platform에서 OS-level secret store를 보장하지
않으므로, 이 저장 방식은 device-at-rest protection을 보장한다고 주장하지 않는다.
이는 이미 Local Vault 자체를 읽을 수 있는 attacker에 대한 새로운 보호 경계가 아니다.
device 분실 또는 token 노출은 Server-side token rotation으로 처리한다.

token plaintext는 log, diagnostic, notification,
URL, clipboard helper에 저장하거나 출력하지 않는다. 구체적인 access lifecycle은
[09 Security and Deployment](./09_security-and-deployment.md)를 따른다.

---

# 55. Client Database 손상

IndexedDB Open 또는 Migration 실패로 Client Sync State를 신뢰할 수 없는 경우:

```text
기존 Local Vault
    X
자동 삭제
```

하지 않는다.

Client는:

```text
RECOVERY_REQUIRED
```

또는 Full Reconciliation으로 전환한다.

Local Vault가 Replica와 다른 파일은 사용자 변경 가능성이 있으므로 보존한다.

---

# 56. Client Store 초기화

Client Store reset은 destructive operation이다. reset 대상, Pending/Conflict가 있을 때의 사용자 경고와 실제 복구 순서는 [11 Observability and Operations](./11_observability-and-operations.md)을 단일 기준으로 사용한다. 이 persistence layer의 요구사항은 reset이 Local Vault를 자동 삭제하거나 overwrite하지 않아야 한다는 것이다.

---

# 57. Storage GC

Client GC 대상:

```text
Completed Apply Records

Expired Download Artifacts

Resolved Conflict Snapshots

Obsolete Recovery Copies
```

Server GC 대상:

```text
Expired Manifest

Committed Staging Artifact

Committed Recovery Artifact
```

다음은 aggressive GC하지 않는다.

```text
Server Change Journal

Server Tombstone

Committed Operation Record

Client Replica Tombstone
```

---

# 58. Persistence Backup Boundary

Server Backup은 최소:

```text
Vault Filesystem

+

SQLite Sync Store
```

를 하나의 논리적 단위로 취급한다.

Client Sync Store는 중요한 Recovery Metadata를 가지고 있지만 사용자의 주 Content Backup을 대체하지 않는다.

```text
Client IndexedDB
    ≠
Vault Backup
```

이다.

---

# 59. Persistence Invariants

## Invariant 1 — Canonical Content Lives in Vault

Server SQLite는 User Content의 원본이 아니다.

---

## Invariant 2 — Sync Metadata Is Durable

Revision, Path State, Operation 및 Change Journal은 Server Memory에만 존재해서는 안 된다.

---

## Invariant 3 — SQLite Commit Is Not Filesystem Commit

SQLite Transaction만으로 Vault Filesystem 변경이 완료되었다고 간주하지 않는다.

Cross-resource Commit은 PREPARED Recovery Protocol을 사용한다.

---

## Invariant 4 — Revision Is Allocated During Finalize

PREPARED Operation에는 외부에 노출되는 Revision을 할당하지 않는다.

---

## Invariant 5 — Manifest Is Snapshot-consistent

Manifest의 여러 Page가 서로 다른 Current State를 섞지 않는다.

---

## Invariant 6 — Client Pending Is Durable Before Push

Local Operation은 durable Client Store에 기록되기 전에 Server Push 대상으로 사용하지 않는다.

---

## Invariant 7 — Cursor Advance Is Transactional

Server Cursor는 해당 Revision의 Replica / Conflict / Apply 처리 결과와 같은 Client Transaction에서 진행한다.

---

## Invariant 8 — Conflict Keeps Local and Server State Separate

Conflict 발생 시 Replica Index는 최신 Server State를 표현하고 Local 사용자 Content는 Local Vault와 Conflict Store가 보존한다.

---

## Invariant 9 — Client Store Loss Must Not Imply Content Loss

Client Metadata 일부를 잃더라도 Local Reconciliation을 통해 Local Vault Change를 다시 발견할 수 있어야 한다.

---

## Invariant 10 — Settings Are Not Sync Database

사용자 Plugin Settings와 correctness-critical Sync State를 같은 lifecycle로 취급하지 않는다.

---

## Invariant 11 — Artifact Is Not Vault Content

Conflict Snapshot, Merge Result 및 Temporary Download는 명시적인 사용자 Resolution 없이는 synchronized Vault Content가 되지 않는다.

---

## Invariant 12 — Persistence Backend Is Replaceable

Server Sync Algorithm은 SQLite Schema Detail에, Client Sync Algorithm은 IndexedDB Detail에 직접 의존하지 않는다.

각 Storage는 Persistence Interface 뒤에 격리한다.
