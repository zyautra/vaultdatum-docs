# API Specification

## 1. 문서 목적

이 문서는 Server와 Client 사이의 Sync API 계약을 정의한다.

상위 문서에서 이미 다음을 정의했다.

```text
Server
    =
Authoritative State

Client
    =
Replica + Offline Workspace
```

API의 목적은 단순한 파일 업로드/다운로드가 아니다.

다음을 안전하게 수행할 수 있어야 한다.

```text
Server Change 조회

현재 Server State 조회

Content 다운로드

Local Operation Commit

Operation Retry

Conflict Detection

Initial Sync

Full Reconciliation

Realtime Notification
```

본 문서는 Sync Protocol의 외부 계약을 정의하며 Server 내부 SQLite Schema나 Client 내부 Storage 구현을 직접 노출하지 않는다.

상태 필드와 용어는 [06 Data Model](./06_data-model.md), 상태 전이와 sync 순서는 [02 Synchronization Protocol](./02_synchronization-protocol.md)의 단일 기준을 따른다. Server/Client 내부 구현과 영속성 세부 사항은 각각 [03 Server Architecture](./03_server-architecture.md), [04 Client Architecture](./04_client-architecture.md), [08 Persistence Design](./08_persistence-design.md)에 둔다.

---

# 2. API 설계 원칙

## 2.1 API는 Server Authority를 보존한다

Client는 Server State를 직접 지정하지 않는다.

예를 들어 Client가:

```text
revision = 1000
```

을 새 Revision으로 생성할 수 없다.

Client는 Operation을 제출하고 Server가 이를 검증한 뒤 새로운 Revision을 생성한다.

---

## 2.2 Mutation은 Base Condition을 가진다

Client Mutation은 항상 자신이 어떤 Server State를 기반으로 만들어졌는지 전달한다.

```text
Operation
    =
Intent
+
Base Condition
+
Result Content
```

Server는 Commit 직전에 Base Condition을 현재 Authoritative State와 비교한다.

---

## 2.3 Notification은 API Correctness를 담당하지 않는다

Realtime Notification은:

```text
새 Revision이 생겼다
```

는 사실을 알려주는 Trigger다.

실제 Sync는 항상 일반 API를 통해 수행한다.

---

## 2.4 Change Journal은 Content History가 아니다

Change Journal은:

```text
무엇이
어떤 순서로
변경되었는가
```

를 표현한다.

과거 Revision별 파일 Content 전체를 보존하는 Version Store가 아니다.

따라서 Client는:

```text
Revision 1001의 A.md Content 다운로드
Revision 1002의 A.md Content 다운로드
Revision 1003의 A.md Content 다운로드
```

와 같이 모든 과거 Content를 재생하려 해서는 안 된다.

Change Stream을 이용해 어떤 State 변화가 있었는지 판단하고, 실제 파일 Content가 필요하면 **현재 Authoritative Path State를 조건부로 다운로드**한다.

---

# 3. API Version

초기 API Prefix는 다음과 같다.

```text
/api/v1
```

예:

```text
GET /api/v1/vault
```

Breaking Change가 필요한 경우 새로운 Major Version을 사용한다.

```text
/api/v2
```

---

# 4. Content Type

일반 Metadata API는:

```text
application/json
```

을 사용한다.

파일 Content 다운로드는 해당 File의 Content Type 또는:

```text
application/octet-stream
```

을 사용할 수 있다.

CREATE / MODIFY Operation은 Metadata와 Binary Content를 함께 전달하기 위해:

```text
multipart/form-data
```

를 사용한다.

---

# 5. Encoding

JSON 문자열은 UTF-8을 사용한다.

Sync Path는 Vault Root 기준 논리 경로다.

예:

```text
notes/kubernetes.md
```

HTTP Query Parameter로 전달할 때 URL Encoding한다.

Filesystem Absolute Path는 API에 노출하지 않는다.

---

# 6. Path State Vocabulary

API에서는 Path 존재 상태를 다음 세 가지로 명시적으로 구분한다.

```text
UNKNOWN

PRESENT

DELETED
```

의미:

```text
UNKNOWN
    =
Server가 해당 Path에 대한 상태를 가지고 있지 않음

PRESENT
    =
현재 Server Vault에 존재함

DELETED
    =
이전에 알려진 Path이며 현재 삭제 상태임
```

따라서 모호한:

```text
ABSENT
```

라는 상태만으로 CREATE의 Base를 표현하지 않는다.

특히:

```text
UNKNOWN
    ≠
DELETED
```

이다.

---

# 7. Common Path State

API에서 Path State는 개념적으로 다음 형태를 사용한다.

```json
{
  "path": "notes/a.md",
  "entryType": "FILE",
  "state": "PRESENT",
  "revision": 1205,
  "contentHash": "sha256:...",
  "size": 5210
}
```

삭제 상태:

```json
{
  "path": "notes/a.md",
  "entryType": "FILE",
  "state": "DELETED",
  "revision": 1210
}
```

Unknown Path는 필요에 따라:

```json
{
  "path": "notes/a.md",
  "state": "UNKNOWN"
}
```

으로 표현한다.

---

# 8. Hash Representation

Protocol v1에서는 Client와 Server가 동일한 Content Hash Algorithm을 사용한다.

Vault 정보 API는 현재 Algorithm을 반환한다.

예:

```json
{
  "hashAlgorithm": "SHA-256"
}
```

Hash 문자열은 Algorithm을 식별할 수 있는 canonical representation을 사용한다.

예:

```text
sha256:<lowercase-hex>
```

Client는 Server가 지원한다고 선언한 Algorithm을 사용해야 한다.

---

# 9. Error Model

일반적인 API Error는 동일한 구조를 사용한다.

예:

```json
{
  "error": {
    "code": "BASE_STATE_MISMATCH",
    "message": "The server state no longer matches the operation base.",
    "details": {}
  }
}
```

`message`는 진단용이다.

Client Logic은 가능한 경우 `code`를 기준으로 동작한다.

---

# 10. 주요 Error Code

초기 Protocol은 최소 다음 오류를 구분할 수 있어야 한다.

```text
INVALID_REQUEST

INVALID_PATH

PATH_EXCLUDED

VAULT_MISMATCH

BASE_STATE_MISMATCH

OPERATION_ID_REUSED

CONTENT_HASH_MISMATCH

STATE_CHANGED

HISTORY_NOT_AVAILABLE

MANIFEST_EXPIRED

RECOVERY_REQUIRED

SERVER_NOT_READY

UNAUTHORIZED

FORBIDDEN
```

---

# 11. HTTP Status 기본 원칙

권장 의미:

```text
200
정상 조회 또는 Idempotent Operation 결과

400
잘못된 Request

401
인증 필요

403
접근 거부

404
Resource 자체를 찾을 수 없음

409
현재 Server State와 요청 상태가 충돌

413
Upload Size 제한 초과

422
형식은 유효하지만 의미적으로 처리 불가

503
Server Recovery 또는 Temporary Unavailable
```

Conflict는 정상적인 Sync 상황이므로 Client가 단순한 Network Error와 구분할 수 있어야 한다.

---

# 12. Vault Information

```text
GET /api/v1/vault
```

현재 연결된 Server Vault의 Identity와 Sync 정보를 반환한다.

Response 예:

```json
{
  "vaultId": "V-123",
  "currentRevision": 1205,
  "oldestRetainedRevision": 1,
  "protocolVersion": 1,
  "hashAlgorithm": "SHA-256"
}
```

---

## 12.1 Vault ID 확인

Client가 저장한:

```text
vaultId = V-123
```

과 Response의 `vaultId`가 다르면 기존:

```text
serverCursor

Replica Index
```

를 그대로 사용하지 않는다.

Client는 Rebootstrap 또는 Recovery 절차로 전환한다.

---

## 12.2 Current Revision

`currentRevision`은 현재 Server에서 가장 최근에 Commit된 Global Revision이다.

Client가:

```text
serverCursor == currentRevision
```

이라고 해서 반드시 Synced 상태인 것은 아니다.

추가로:

```text
Pending = 0

Conflict = 0
```

이어야 정상 Synced 상태다.

---

# 13. Change Stream 조회

```text
GET /api/v1/changes
```

Query:

```text
after=<revision>
limit=<n>
```

예:

```text
GET /api/v1/changes?after=1000&limit=500
```

의 의미:

> Revision 1000 이후 Commit된 Change를 순서대로 반환하라.

---

# 14. Change Stream Response

예:

```json
{
  "vaultId": "V-123",
  "fromExclusive": 1000,
  "toInclusive": 1003,
  "currentRevision": 1010,
  "hasMore": true,
  "changes": [
    {
      "revision": 1001,
      "type": "MODIFY",
      "operationId": "OP-A",
      "actor": {
        "type": "CLIENT",
        "clientId": "C-2"
      },
      "effects": [
        {
          "path": "notes/a.md",
          "entryType": "FILE",
          "state": "PRESENT",
          "contentHash": "sha256:...",
          "size": 500
        }
      ]
    }
  ]
}
```

Change는 반드시 Revision 오름차순으로 반환한다.

---

# 15. Change Pagination

Client는 `toInclusive`까지의 Change를 모두 안전하게 처리한 후에만 Cursor를 해당 Revision까지 진행시킨다.

예:

```text
Client Cursor = 1000

Response:
1001
1002
1003

toInclusive = 1003
```

모든 Change가:

```text
Applied

또는

Safely recorded as Conflict
```

된 경우:

```text
serverCursor = 1003
```

으로 진행할 수 있다.

---

# 16. History Retention 초과

Client Cursor가 Server가 보유한 Change Journal보다 오래된 경우:

```text
GET /changes?after=100
```

에 대해 Server는:

```text
HISTORY_NOT_AVAILABLE
```

을 반환한다.

예:

```json
{
  "error": {
    "code": "HISTORY_NOT_AVAILABLE",
    "details": {
      "requestedAfter": 100,
      "oldestRetainedRevision": 500
    }
  }
}
```

Client는 Full Reconciliation으로 전환한다.

---

# 17. Change Effect

Change 하나는 하나 이상의 Path Effect를 가진다.

CREATE:

```json
{
  "revision": 101,
  "type": "CREATE",
  "effects": [
    {
      "path": "A.md",
      "state": "PRESENT",
      "contentHash": "sha256:..."
    }
  ]
}
```

DELETE:

```json
{
  "revision": 102,
  "type": "DELETE",
  "effects": [
    {
      "path": "A.md",
      "state": "DELETED"
    }
  ]
}
```

---

# 18. Rename / Move Change

Rename은 하나의 Revision이다.

예:

```json
{
  "revision": 200,
  "type": "RENAME",
  "sourcePath": "A.md",
  "destinationPath": "archive/A.md",
  "effects": [
    {
      "path": "A.md",
      "state": "DELETED"
    },
    {
      "path": "archive/A.md",
      "state": "PRESENT",
      "contentHash": "sha256:..."
    }
  ]
}
```

Client는 이를 DELETE + CREATE 두 개의 독립적인 Revision으로 해석하지 않는다.

---

# 19. Intermediate Content를 재생하지 않는다

예:

```text
1001 MODIFY A.md → Hash B
1002 MODIFY A.md → Hash C
1003 MODIFY A.md → Hash D
```

Client가 1000에서 복귀했다고 해서:

```text
B 다운로드
C 다운로드
D 다운로드
```

를 수행할 필요가 없다.

Change Metadata를 순서대로 처리하면서 안전하다면 최종 Server State인 `D`만 Materialize할 수 있다.

다만 중간 Change가 Conflict 판단이나 Rename Tracking에 영향을 주는 경우 Metadata상의 순서는 반드시 처리한다.

---

# 20. Current Path State 조회

```text
GET /api/v1/path-state
```

Query:

```text
path=<sync-path>
```

예:

```text
GET /api/v1/path-state?path=notes/a.md
```

Response:

```json
{
  "path": "notes/a.md",
  "entryType": "FILE",
  "state": "PRESENT",
  "revision": 1205,
  "contentHash": "sha256:...",
  "size": 5210
}
```

이 API는 Conflict Resolution 직전 최신 Server State 확인에도 사용한다.

---

# 21. Unknown Path Response

Server가 해당 Path를 전혀 알지 못하면:

```json
{
  "path": "notes/new.md",
  "state": "UNKNOWN"
}
```

을 반환할 수 있다.

HTTP 404와 구분한다.

이 경우 API Resource는 정상적으로 조회되었고 Path State가 UNKNOWN인 것이기 때문이다.

---

# 22. File Content Download

```text
GET /api/v1/content
```

Query:

```text
path=<sync-path>
revision=<path-revision>
hash=<content-hash>
```

예:

```text
GET /api/v1/content
    ?path=notes/a.md
    &revision=1205
    &hash=sha256:...
```

Server는 현재 Path State가 Request 조건과 정확히 일치하는 경우에만 Content를 반환한다.

---

# 23. Conditional Content Download

Request:

```text
path = A.md
revision = 100
hash = AAA
```

Server Current State:

```text
A.md
revision = 100
hash = AAA
```

이면 Content를 Streaming한다.

반면 Content 다운로드 전에 다른 Client가 수정하여:

```text
revision = 101
hash = BBB
```

가 되었다면 과거 AAA를 반환하지 않는다.

Server는:

```text
409 STATE_CHANGED
```

를 반환한다.

---

# 24. STATE_CHANGED

예:

```json
{
  "error": {
    "code": "STATE_CHANGED",
    "details": {
      "currentState": {
        "path": "A.md",
        "state": "PRESENT",
        "revision": 101,
        "contentHash": "sha256:BBB"
      }
    }
  }
}
```

Client는 Change Stream을 다시 조회하고 Reconciliation을 계속한다.

이 구조를 통해 Server가 과거 Content Version Store를 유지하지 않아도 Race를 안전하게 처리할 수 있다.

---

# 25. Unrelated Revision은 Content Download를 막지 않는다

Content Request의 `revision`은 Global Current Revision이 아니라 **해당 Path의 latestRevision**이다.

예:

```text
A.md latestRevision = 100

Server currentRevision = 500
```

Revision 101~500이 모두 다른 파일에 대한 Change라면 A.md는 여전히 안전하게 다운로드할 수 있다.

---

# 26. Mutation API

Client Mutation은:

```text
POST /api/v1/operations
```

으로 제출한다.

Operation Type:

```text
CREATE

MODIFY

DELETE

RENAME

MOVE
```

---

# 27. JSON-only Operation

Content가 필요하지 않은 Operation:

```text
DELETE

RENAME

MOVE
```

는:

```text
Content-Type: application/json
```

을 사용한다.

---

# 28. Content Operation

Content를 전달해야 하는:

```text
CREATE

MODIFY
```

는:

```text
multipart/form-data
```

를 사용한다.

Multipart Part:

```text
operation
    application/json

content
    binary stream
```

Server는 전체 Content를 Memory에 올리지 않고 Staging Storage로 Streaming할 수 있어야 한다.

---

# 29. MODIFY Request

Operation Metadata 예:

```json
{
  "operationId": "OP-123",
  "clientId": "C-1",
  "type": "MODIFY",
  "path": "notes/a.md",
  "base": [
    {
      "path": "notes/a.md",
      "state": "PRESENT",
      "revision": 100,
      "contentHash": "sha256:AAA"
    }
  ],
  "content": {
    "contentHash": "sha256:BBB",
    "size": 5210
  }
}
```

Server는 Upload Content를 직접 Hash하여 Metadata의 `contentHash`와 일치하는지 검증한다.

---

# 30. DELETE Request

```json
{
  "operationId": "OP-124",
  "clientId": "C-1",
  "type": "DELETE",
  "path": "notes/a.md",
  "base": [
    {
      "path": "notes/a.md",
      "state": "PRESENT",
      "revision": 100,
      "contentHash": "sha256:AAA"
    }
  ]
}
```

---

# 31. New CREATE Request

한 번도 존재한 적 없다고 Client가 알고 있는 Path 생성:

```json
{
  "operationId": "OP-125",
  "clientId": "C-1",
  "type": "CREATE",
  "path": "notes/new.md",
  "base": [
    {
      "path": "notes/new.md",
      "state": "UNKNOWN"
    }
  ],
  "content": {
    "contentHash": "sha256:CCC",
    "size": 1000
  }
}
```

Server Path가:

```text
PRESENT
```

또는:

```text
DELETED
```

라면 이 Operation은 Base mismatch다.

특히 오래된 Client가 Tombstone을 모른 채 삭제된 파일을 암묵적으로 부활시키는 것을 방지한다.

---

# 32. Explicit Restore CREATE

사용자가 Conflict Resolution에서 삭제된 파일을 명시적으로 복원하는 경우는 다르다.

Server:

```text
A.md

state = DELETED
revision = 200
```

Client Operation:

```json
{
  "operationId": "OP-126",
  "clientId": "C-1",
  "type": "CREATE",
  "path": "A.md",
  "base": [
    {
      "path": "A.md",
      "state": "DELETED",
      "revision": 200
    }
  ],
  "content": {
    "contentHash": "sha256:DDD",
    "size": 1500
  }
}
```

이는 사용자의 명시적인 복원 의도를 나타낸다.

---

# 33. RENAME Request

```json
{
  "operationId": "OP-127",
  "clientId": "C-1",
  "type": "RENAME",
  "sourcePath": "A.md",
  "destinationPath": "B.md",
  "base": [
    {
      "path": "A.md",
      "state": "PRESENT",
      "revision": 300,
      "contentHash": "sha256:AAA"
    },
    {
      "path": "B.md",
      "state": "UNKNOWN"
    }
  ]
}
```

Source와 Destination 모두 검증한다.

---

# 34. Deleted Destination Rename

Destination이 과거에 삭제된 Path이고 Client가 그 사실을 알고 있으며 해당 경로를 의도적으로 재사용하려면 Base에:

```json
{
  "path": "B.md",
  "state": "DELETED",
  "revision": 250
}
```

를 명시할 수 있다.

Client가 Tombstone을 모르는 상태에서 이를 자동으로 수행해서는 안 된다.

---

# 35. Operation Success

새로운 Commit이 성공하면:

```json
{
  "operationId": "OP-123",
  "status": "COMMITTED",
  "resultRevision": 1206,
  "replayed": false
}
```

를 반환한다.

`resultRevision`을 받았다고 Client Cursor를 직접 1206으로 변경해서는 안 된다.

---

# 36. Operation Retry

Client가 Response를 받지 못해 같은 Operation ID를 재전송할 수 있다.

이미 COMMITTED라면:

```json
{
  "operationId": "OP-123",
  "status": "COMMITTED",
  "resultRevision": 1206,
  "replayed": true
}
```

를 반환한다.

새 Revision을 생성하지 않는다.

---

# 37. Operation ID Reuse Error

동일 Operation ID로 다른 Request가 들어오면:

```text
409 OPERATION_ID_REUSED
```

를 반환한다.

예:

기존:

```text
OP-1
MODIFY A.md
```

새 요청:

```text
OP-1
DELETE B.md
```

는 Retry가 아니다.

---

# 38. Base Conflict Response

현재 Server State가 Base Condition과 다르면:

```text
409 BASE_STATE_MISMATCH
```

를 반환한다.

예:

```json
{
  "operationId": "OP-123",
  "status": "REJECTED_CONFLICT",
  "error": {
    "code": "BASE_STATE_MISMATCH"
  },
  "currentStates": [
    {
      "path": "A.md",
      "state": "PRESENT",
      "revision": 105,
      "contentHash": "sha256:SERVER"
    }
  ]
}
```

Client는 Original Pending Operation을 자동 Retry하지 않고 Conflict 상태로 전환한다.

---

# 39. Request Validation 순서

Server는 개념적으로 다음 순서로 Mutation을 처리한다.

```text
1. Authentication / Access Check

2. Request Format Validation

3. Path Validation

4. Operation ID Lookup

5. Upload / Hash Validation

6. Base Validation

7. Prepare

8. Filesystem Apply

9. Sync State Finalize

10. Success
```

실제 내부 최적화로 일부 검증 순서는 조정할 수 있지만 외부 의미는 동일해야 한다.

---

# 40. Manifest

Initial Sync와 Full Reconciliation을 위해 Server는 Current Authoritative State의 Manifest를 제공한다.

Manifest는 일반 Change Stream과 다르게 **하나의 논리적으로 일관된 Snapshot**을 나타내야 한다.

---

# 41. Manifest Snapshot 생성

초기 API:

```text
POST /api/v1/manifests
```

Response:

```json
{
  "manifestId": "M-123",
  "vaultId": "V-123",
  "snapshotRevision": 5000,
  "expiresAt": "..."
}
```

Server는 해당 Manifest ID가 살아 있는 동안 하나의 일관된 Path State Snapshot을 제공한다.

---

# 42. Manifest Page 조회

```text
GET /api/v1/manifests/{manifestId}
```

Query:

```text
cursor=<opaque>
limit=<n>
```

Response:

```json
{
  "manifestId": "M-123",
  "snapshotRevision": 5000,
  "entries": [
    {
      "path": "notes/a.md",
      "entryType": "FILE",
      "state": "PRESENT",
      "revision": 4990,
      "contentHash": "sha256:AAA",
      "size": 500
    },
    {
      "path": "old.md",
      "entryType": "FILE",
      "state": "DELETED",
      "revision": 4500
    }
  ],
  "nextCursor": "...",
  "hasMore": true
}
```

Pagination Cursor는 Client가 해석하지 않는 opaque value다.

---

# 43. Manifest Consistency

Manifest Pagination 중 Server Revision이 증가해도 같은 `manifestId`의 내용은 변하지 않는다.

```text
Manifest snapshotRevision = 5000

Server later reaches 5010
```

이어도 Manifest는 Revision 5000 시점의 논리적 Snapshot을 표현한다.

Manifest 완료 후 Client는:

```text
changes after 5000
```

을 Pull하여 최신 상태로 따라잡는다.

---

# 44. Manifest Expiration

Manifest Snapshot은 영구 Resource가 아니다.

만료된 Manifest를 요청하면:

```text
MANIFEST_EXPIRED
```

를 반환한다.

Client는 새로운 Manifest를 생성하여 Full Reconciliation을 다시 시작할 수 있다.

---

# 45. Manifest와 Content Race

Manifest Entry:

```text
A.md
revision = 4990
hash = AAA
```

를 본 이후 A.md가 Server에서 변경될 수 있다.

Client가:

```text
GET /content
path=A.md
revision=4990
hash=AAA
```

를 요청했을 때 Server Current State가 바뀌었다면 `STATE_CHANGED`를 반환한다.

Client는 과거 Content를 요구하지 않고 Manifest 이후 Change Stream을 이용해 최신 상태로 수렴한다.

---

# 46. Initial Sync

신규 Client의 기본 흐름:

```text
GET /vault
    ↓
Create Manifest
    ↓
Read Manifest
    ↓
Materialize Server Content
    ↓
Set Replica Index
    ↓
Set serverCursor = snapshotRevision
    ↓
Pull changes after snapshotRevision
    ↓
Converge
```

Initial Sync 중 Local synchronized content에 예상치 못한 사용자 변경이 발견되면 이를 자동으로 덮어쓰지 않고 Client Bootstrap 정책에 따라 처리한다.

---

# 47. Full Reconciliation

Full Reconciliation도 동일한 Manifest API를 사용한다.

비교 대상:

```text
Manifest

Replica Index

Local Vault

Pending Operations

Conflicts
```

이다.

Server Manifest만 보고 Local Vault를 무조건 덮어쓰지 않는다.

---

# 48. Conflict Resolution에서 사용하는 API

Conflict Resolution은 별도 특수 Commit API를 필요로 하지 않는다.

다음 기존 API를 조합한다.

```text
GET /path-state

GET /content

POST /operations
```

예:

```text
Apply Local

1. GET current path-state
2. Create new MODIFY Operation using current state as Base
3. POST /operations
```

즉 Conflict Resolution도 일반 Mutation Protocol을 그대로 사용한다.

---

# 49. Use Server Resolution

사용자가 Server Version 사용을 선택하면:

```text
GET /path-state

GET /content
```

를 통해 현재 Authoritative State를 확인하고 Local에 적용한다.

Server Mutation은 발생하지 않는다.

---

# 50. Manual Merge Resolution

Manual Merge Result `D`를 Commit할 때:

```text
1. 현재 Server Path State 다시 조회
2. 현재 Server State를 Base로 사용
3. D를 새로운 MODIFY Operation으로 생성
4. POST /operations
```

한다.

Commit 중 Server State가 다시 변경되면 일반 `BASE_STATE_MISMATCH`를 반환하고 Conflict는 RECONFLICTED 상태가 된다.

---

# 51. Notification Channel

Realtime Notification은 WebSocket 기반 Channel을 사용할 수 있다.

Endpoint:

```text
/api/v1/notifications
```

연결 방식의 구체적인 Authentication Binding은 Security 문서에서 정의한다.

---

# 52. Revision Advanced Notification

Server에서 새로운 Commit이 발생하면 다음과 같은 Notification을 보낼 수 있다.

```json
{
  "type": "REVISION_ADVANCED",
  "currentRevision": 1206
}
```

Payload는 Change Content 자체가 아니다.

Client는 수신 후:

```text
GET /changes?after=<serverCursor>
```

를 수행한다.

---

# 53. Notification의 의미

Notification은 다음을 보장하지 않아도 된다.

```text
Exactly Once

Ordering

Durable Delivery
```

즉:

```text
duplicate

missing

late

out-of-order
```

Notification이 발생해도 correctness에 영향을 주지 않는다.

Client는 `currentRevision`을 힌트로만 사용한다.

---

# 54. Notification Disconnect

WebSocket이 끊어져도 특별한 Recovery Protocol은 필요하지 않다.

재연결 후:

```text
GET /vault

GET /changes?after=<serverCursor>
```

만 수행하면 된다.

---

# 55. Operation Content Size

Server는 Upload 제한이 있을 수 있다.

필요하면 `/vault` 또는 별도 Capability Field에서:

```text
maxOperationContentSize
```

를 제공할 수 있다.

제한을 초과한 경우:

```text
413
```

을 반환한다.

MVP에서 Resumable Multipart Upload는 필수 기능으로 하지 않는다.

---

# 56. Directory Operations

Directory CREATE / DELETE / RENAME / MOVE 역시 같은 Operation 모델을 사용할 수 있다.

Directory에는:

```text
contentHash
content payload
```

가 존재하지 않는다.

초기 버전의 Directory Operation은 비어 있는 Directory에만 적용한다. Recursive
Directory Delete처럼 다수 Path를 암묵적으로 변경하는 API는 지양한다.

Client는 필요한 경우 명시적인 Path Change를 생성한다.

---

# 57. `.obsidian/` 접근

다음 Path는 Sync API에서 허용하지 않는다.

```text
.obsidian

.obsidian/...

```

Client가 Mutation을 요청하면:

```text
PATH_EXCLUDED
```

를 반환한다.

Manifest와 Change Stream에도 포함하지 않는다.

---

# 58. Path Security

Server는 모든 Path에 대해:

```text
Normalization

Vault Root containment

Excluded Path

Cross-platform Path Rule
```

을 검증한다.

Client가 전달한 Path를 Filesystem Path로 단순 문자열 결합하지 않는다.

---

# 59. Authentication Boundary

모든 Sync API는 논리적으로 인증된 Client Context에서 실행되는 것을 전제로 한다.

```text
Authenticated Client
    │
    ├── clientId
    └── permissions
```

구체적인:

```text
VPN-only

Bearer Token

mTLS

Reverse Proxy Authentication
```

선택은 Security / Deployment 문서에서 정의한다.

Sync API의 데이터 모델은 특정 인증 기술에 종속되지 않는다.

---

# 60. Client ID 신뢰

Request Body의 `clientId`만으로 Identity를 신뢰해서는 안 된다.

인증 계층이 Client Identity를 제공하는 환경에서는:

```text
Authenticated Identity
```

와:

```text
Request clientId
```

가 일치하는지 확인한다.

구체적인 Identity Binding 방식은 Security 문서에서 정의한다.

---

# 61. Health Endpoint

Container 환경 운영을 위해 Sync API와 별도로 최소한 다음 Health 개념을 제공할 수 있다.

```text
Liveness

Readiness
```

예:

```text
GET /health/live

GET /health/ready
```

Readiness는 Server Startup Recovery가 완료되기 전에는 성공해서는 안 된다.

---

# 62. Readiness 의미

다음 상태에서는 Server가 Process로 살아 있어도 Ready가 아닐 수 있다.

```text
Startup Recovery 진행 중

RECOVERY_REQUIRED

Sync Store 사용 불가

Authoritative State 검증 실패
```

이 경우 Kubernetes 등은 해당 Pod에 Client Traffic을 전달하지 않을 수 있다.

---

# 63. API Idempotency

Read API는 일반적으로 idempotent하다.

Mutation API는 `operationId`를 통해 논리적 idempotency를 제공한다.

```text
POST Operation

Response Lost

POST Same Operation

→ Same Commit Result
```

이어야 한다.

---

# 64. API Ordering

여러 Client 요청이 동시에 Server에 도착할 수 있다.

HTTP 요청 도착 순서 자체는 Global Revision 순서를 보장하지 않는다.

실제 Revision Order는 Server Commit Coordinator가 결정한다.

Client는:

```text
Request sent first
    =
Revision assigned first
```

라고 가정하지 않는다.

---

# 65. Cursor는 Client가 Server에 Commit하지 않는다

`serverCursor`는 Client-local metadata다.

Server에:

```text
SET my cursor = 1000
```

과 같은 API를 필수로 두지 않는다.

Client가 자신의 Cursor를 durable하게 관리한다.

향후 Server가 Client Progress를 관찰하거나 Journal GC에 사용하고 싶다면 별도 optional acknowledgement API를 추가할 수 있다.

---

# 66. API Retry Policy

다음 Read 요청은 안전하게 Retry할 수 있다.

```text
GET /vault

GET /changes

GET /path-state

GET /content

GET Manifest Page
```

Mutation은 반드시 동일 `operationId`로 Retry한다.

새 Operation ID를 만들면 새로운 사용자 Intent로 취급될 수 있다.

---

# 67. Timeout 의미

Mutation Request Timeout은:

```text
Operation Failed
```

을 의미하지 않는다.

Client 입장에서는:

```text
Result Unknown
```

이다.

따라서 동일 Operation ID로 다시 조회 또는 Retry해야 한다.

---

# 68. API에서 노출하지 않는 내부 상태

Client는 다음 Server 내부 구현 세부 사항을 알 필요가 없다.

```text
SQLite Table

WAL Position

Staging Filename

Recovery Directory

Filesystem Absolute Path

Internal Lock
```

API는 논리적 Sync State만 노출한다.

---

# 69. 기본 Sync Sequence

일반적인 Client Sync:

```text
GET /vault
      │
      ▼
GET /changes?after=cursor
      │
      ▼
Process Server Changes
      │
      ▼
Download required current Content
      │
      ▼
POST Pending Operations
      │
      ▼
GET /changes?after=cursor
      │
      ▼
Convergence Check
```

---

# 70. Mutation Sequence

```text
Client Local Change
        │
        ▼
Pending Operation
        │
        ▼
POST /operations
        │
        ▼
Server Base Validation
        │
    ┌───┴───┐
    │       │
    ▼       ▼
 Match    Mismatch
    │       │
    ▼       ▼
Commit   Conflict
    │
    ▼
resultRevision
    │
    X
Cursor 직접 이동 금지
    │
    ▼
Pull Change Stream
```

---

# 71. Content Materialization Sequence

```text
Change says:

A.md
revision = 500
hash = AAA

        │
        ▼

GET /content
path=A.md
revision=500
hash=AAA

        │
   ┌────┴────┐
   │         │
   ▼         ▼
Match     Changed Again
   │         │
   ▼         ▼
Content   STATE_CHANGED
   │         │
   ▼         ▼
Apply     Pull Again
```

이 구조를 통해 Server는 파일별 Historical Content Store 없이도 안전한 Sync를 제공할 수 있다.

---

# 72. API Invariants

## Invariant 1 — Server Assigns Revision

Client는 Global Revision을 생성하거나 강제로 지정하지 않는다.

---

## Invariant 2 — Mutation Requires Base Validation

Server State를 변경하는 모든 Operation은 해당 변경이 의존하는 Base Condition을 검증한다.

---

## Invariant 3 — Unknown and Deleted Are Distinct

한 번도 알려지지 않은 Path와 명시적으로 삭제된 Path를 동일한 absent state로 취급하지 않는다.

---

## Invariant 4 — Content Download Is Conditional

Client가 예상한 Path Revision과 Hash가 현재 Server State와 일치할 때만 Content를 반환한다.

---

## Invariant 5 — No Historical Content Assumption

Client correctness는 Server가 모든 과거 Revision의 File Content를 보존한다고 가정하지 않는다.

---

## Invariant 6 — Change Stream Is Ordered

Change API는 Global Revision 순서로 Change를 반환한다.

---

## Invariant 7 — Push Result Does Not Advance Cursor

Operation Commit Response의 `resultRevision`만으로 Client Cursor를 이동시키지 않는다.

---

## Invariant 8 — Retry Uses Same Operation ID

결과가 불명확한 Mutation은 동일 Operation ID로 Retry한다.

---

## Invariant 9 — Operation ID Is Immutable

동일 Operation ID는 항상 동일한 Request Identity를 의미해야 한다.

---

## Invariant 10 — Manifest Is Snapshot-consistent

하나의 Manifest ID는 하나의 `snapshotRevision` 기준 State를 표현한다.

---

## Invariant 11 — Notification Is Advisory

Notification의 손실, 중복 또는 순서 변경이 Sync Correctness에 영향을 주지 않는다.

---

## Invariant 12 — Conflict Uses Normal Mutation Protocol

Conflict Resolution 결과도 일반적인 현재 Base State 기반 Operation으로 Commit한다.

---

## Invariant 13 — Internal Storage Is Hidden

Client API는 SQLite, Staging, Recovery 등의 Server 내부 Storage 구조에 의존하지 않는다.

---

# 73. 전체 API Surface

초기 Sync API는 다음 정도로 구성할 수 있다.

```text
Vault

GET  /api/v1/vault


Incremental Sync

GET  /api/v1/changes


Current State

GET  /api/v1/path-state

GET  /api/v1/content


Mutation

POST /api/v1/operations


Full Reconciliation

POST /api/v1/manifests

GET  /api/v1/manifests/{manifestId}


Realtime Hint

WS   /api/v1/notifications


Operations

GET  /health/live

GET  /health/ready
```

핵심 Sync Surface는 의도적으로 작게 유지한다.

이 API 모델의 핵심은:

> **Change Journal을 파일 버전 저장소로 만들지 않고, ordered change metadata와 조건부 current-content download를 조합하여 Client가 항상 현재 Authoritative State로 수렴하게 하는 것**

이다.
