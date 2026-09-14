# Data Model

## 1. 문서 목적

이 문서는 동기화 시스템이 correctness를 유지하기 위해 기억해야 하는 논리적 데이터를 정의한다.

이 문서의 목적은 SQL Schema를 먼저 설계하는 것이 아니다.

먼저 다음을 명확히 한다.

```text
무엇이 상태인가?

어떤 상태가 authoritative한가?

어떤 데이터가 서로 어떤 관계를 가지는가?

언제 생성되고 언제 제거되는가?
```

이를 바탕으로 이후 Server SQLite Schema, Client Storage 및 API Model을 설계한다.

이 문서는 용어와 논리 관계의 단일 기준이다. 상태를 전이하는 순서는 [02 Synchronization Protocol](./02_synchronization-protocol.md), wire 표현은 [07 API Specification](./07_api-specification.md), 물리 저장소와 transaction은 [08 Persistence Design](./08_persistence-design.md)에서 정의한다.

---

## 2. Data Model 설계 원칙

### 2.1 Content와 Metadata를 분리한다

Server의 실제 사용자 Content는 Vault Filesystem에 존재한다.

```text
Vault Filesystem
    =
Canonical Content
```

Sync Store는 Content 자체를 복제하여 원본으로 사용하지 않는다.

```text
Sync Store
    =
Synchronization Metadata
```

---

### 2.2 Current State와 History를 분리한다

Server는 두 가지 정보를 모두 가진다.

```text
Current State

현재 A.md가 존재하는가?
현재 Hash는 무엇인가?
현재 Revision은 무엇인가?
```

그리고:

```text
History

Revision 1000 이후
무엇이 변경되었는가?
```

따라서:

```text
Path State
    =
Current State

Change Journal
    =
History
```

로 역할을 분리한다.

---

### 2.3 Revision은 Content Version이 아니라 Commit Order다

Global Revision은 전체 Vault에서 발생한 committed change의 순서를 나타낸다.

예:

```text
1001 MODIFY notes/a.md
1002 CREATE notes/b.md
1003 DELETE notes/c.md
```

Revision 자체가 별도의 Document Version 객체를 의미하지 않는다.

---

### 2.4 Timestamp는 ordering 기준이 아니다

Server와 Client의 시계는 다를 수 있다.

따라서:

```text
createdAt
modifiedAt
committedAt
```

등의 Timestamp는 사용자 표시 및 진단 용도로 사용할 수 있지만 동기화 순서나 Conflict 판단의 기준으로 사용하지 않는다.

동기화 ordering은 Global Revision과 Base State를 사용한다.

---

# 3. 공통 Value Types

## 3.1 Vault ID

Server의 authoritative sync state를 식별하는 안정적인 ID다.

```text
vaultId
```

Client는 자신이 어떤 Server Vault와 동기화되고 있는지 기억한다.

예:

```text
Client State

vaultId = V-123
serverCursor = 5000
```

Server 연결 시 Vault ID가 예상과 다르면 기존 Cursor와 Replica Index를 그대로 사용해서는 안 된다.

```text
Expected Vault = V-123
Connected Vault = V-999

→ Rebootstrap Required
```

Sync Metadata를 완전히 재생성하여 기존 Revision lineage를 더 이상 보존할 수 없는 경우 새로운 Vault ID를 생성한다.

---

## 3.2 Client ID

각 Client 설치를 구분하는 안정적인 ID다.

```text
clientId
```

예:

```text
desktop = C-1

mobile = C-2

laptop = C-3
```

Plugin 재시작만으로 Client ID가 변경되어서는 안 된다.

---

## 3.3 Operation ID

하나의 Client Mutation Request를 식별하는 고유 ID다.

```text
operationId
```

같은 Operation ID의 Retry는 같은 논리적인 요청을 의미한다.

```text
same operationId
    =
same logical mutation
```

같은 Operation ID를 다른 Content나 다른 요청에 재사용해서는 안 된다.

---

## 3.4 Global Revision

Server에서 committed change의 순서를 나타내는 단조 증가 정수다.

```text
revision ∈ 1..N
```

Revision은 Server에서만 생성한다.

Client는 Revision을 생성하지 않는다.

---

## 3.5 Sync Path

모든 Path는 Vault Root 기준 상대 경로로 표현한다.

예:

```text
notes/kubernetes.md

attachments/image.png
```

다음과 같은 Path는 Sync Path가 될 수 없다.

```text
/data/vault/notes/a.md

../outside.md

.obsidian/plugins/x
```

논리적인 Path 표현은 플랫폼별 Filesystem Path와 분리한다.

기본 표현에서는 `/`를 separator로 사용한다.

---

## 3.6 Path Identity

Path는 synchronization identity의 중요한 일부이므로 플랫폼별 차이를 고려해야 한다.

Server는 최소한 다음 상황을 허용해서는 안 된다.

```text
지원하는 Client Platform 중 하나에서
동시에 표현할 수 없는 두 Path
```

예를 들어 case-sensitive Server에서:

```text
A.md
a.md
```

를 서로 다른 문서로 허용하면 case-insensitive Client에서 표현할 수 없을 수 있다.

따라서 Cross-platform Path Normalization과 이름 제한 정책이 필요하다.

정확한 normalization algorithm은 별도 구현 명세에서 정의하되 Data Model은 다음을 요구한다.

```text
모든 Sync Path는
일관된 Path Identity 규칙을 가져야 한다.
```

---

## 3.7 Content Hash

파일 Content의 동일성을 판정하는 opaque value다.

```text
contentHash
```

Hash Algorithm 자체는 구현에서 선택한다.

다음 목적으로 사용한다.

```text
Base Validation

Local Change Detection

Remote Apply Verification

Recovery

Reconciliation
```

Directory에는 Content Hash가 필요하지 않다.

---

## 3.8 Entry Type

동기화 대상은 최소 두 종류다.

```text
FILE

DIRECTORY
```

Markdown, Image, PDF 및 Attachment는 모두 `FILE`이다.

---

## 3.9 Change Type

초기 Change Type은 다음과 같다.

```text
CREATE

MODIFY

DELETE

RENAME

MOVE
```

RENAME과 MOVE는 사용자 의미가 다를 수 있으나 내부적으로 동일한 Path 이동 메커니즘을 공유할 수 있다.

---

# 4. Server Data Model

Server의 주요 논리 모델은 다음과 같다.

```text
Server State

├── Vault Metadata
├── Path State
├── Change Journal
└── Operation Store
```

Filesystem에는 별도로:

```text
Vault Content

Staging Artifact

Recovery Artifact
```

가 존재한다.

---

# 5. Vault Metadata

Vault Metadata는 Server 전체 Sync State를 나타낸다.

개념적으로:

```text
VaultMetadata

vaultId
currentRevision
schemaVersion
```

`currentRevision`은 현재 가장 최근에 commit된 Global Revision이다.

초기 상태:

```text
currentRevision = 0
```

첫 Commit 이후:

```text
currentRevision = 1
```

---

# 6. Revision은 별도 Entity가 아니다

초기 Data Model에서는 별도의 `Revision` 객체나 Revision Table을 만들 필요가 없다.

Revision은:

```text
ChangeRecord.revision
```

으로 존재한다.

그리고 Server 전체 현재 Revision은:

```text
VaultMetadata.currentRevision
```

으로 유지한다.

즉:

```text
Global Revision
    =
Change Journal ordering key
```

이다.

---

# 7. Path State

기존 Architecture에서 `File Index`라고 표현했던 Current State를 논리적으로 `Path State`로 일반화한다.

Directory도 표현할 수 있기 때문이다.

개념적으로:

```text
PathState

path
entryType
state
latestRevision
contentHash
size
```

`state`는:

```text
PRESENT

DELETED
```

중 하나다.

---

## 7.1 Present State

예:

```text
path = notes/a.md
entryType = FILE
state = PRESENT
latestRevision = 1200
contentHash = AAA
size = 5210
```

Filesystem에는 실제:

```text
vault/notes/a.md
```

가 존재한다.

---

## 7.2 Deleted State

파일이 삭제되어도 Path State를 즉시 제거하지 않는다.

예:

```text
path = notes/a.md
entryType = FILE
state = DELETED
latestRevision = 1205
```

Filesystem에는 파일이 존재하지 않는다.

이 상태 자체가 Tombstone이다.

---

# 8. Tombstone은 별도 Entity가 아니다

초기 모델에서는:

```text
PathState.state = DELETED
```

인 상태를 Tombstone으로 정의한다.

따라서 별도의:

```text
files table

+

tombstones table
```

구조를 논리적으로 요구하지 않는다.

```text
Path State

PRESENT
   │
   │ DELETE
   ▼
DELETED
```

로 표현한다.

이렇게 하면 같은 Path의 현재 상태를 항상 하나의 모델에서 조회할 수 있다.

---

## 8.1 Unknown과 Deleted는 다르다

매우 중요한 차이다.

Path State 자체가 존재하지 않는 경우:

```text
UNKNOWN
```

이다.

Path State가 존재하고:

```text
state = DELETED
```

인 경우는:

```text
KNOWN DELETED
```

이다.

즉:

```text
No PathState
    ≠
DELETED PathState
```

이다.

---

## 8.2 Deleted State의 이전 정보

Delete 이후에도 필요하다면 다음 정보를 보존할 수 있다.

```text
lastContentHash

previousEntryType
```

이는 다음 상황에서 유용하다.

```text
Stale File Detection

Recovery

Conflict Diagnosis
```

현재 Content는 존재하지 않으므로 `contentHash`와 이전 Content Hash는 논리적으로 구분한다.

---

# 9. Directory State

Directory 역시 Path State로 표현할 수 있다.

예:

```text
path = projects/archive
entryType = DIRECTORY
state = PRESENT
latestRevision = 3000
```

Directory에는:

```text
contentHash
size
```

가 필요하지 않다.

Directory 상태를 추적하면 Empty Directory도 동기화할 수 있다.

---

# 10. Change Record

Server에 Commit된 논리적 변경 하나는 하나의 Change Record를 생성한다.

개념적으로:

```text
ChangeRecord

revision
changeType
operationId
actor
committedAt
effects
```

`committedAt`은 정보용 Timestamp다.

Revision ordering을 대신하지 않는다.

---

# 11. Change Effect

하나의 Change는 하나 이상의 Path에 영향을 줄 수 있다.

따라서 Change를 단순히:

```text
revision
path
```

로만 모델링하지 않는다.

개념적으로:

```text
Change

    │
    ├── Path Effect
    └── Path Effect
```

를 허용한다.

---

## 11.1 CREATE

```text
CREATE A.md
```

Effect:

```text
A.md

UNKNOWN / DELETED
    →
PRESENT
```

---

## 11.2 MODIFY

```text
MODIFY A.md
```

Effect:

```text
A.md

PRESENT(old hash)
    →
PRESENT(new hash)
```

---

## 11.3 DELETE

```text
DELETE A.md
```

Effect:

```text
A.md

PRESENT
    →
DELETED
```

---

## 11.4 RENAME / MOVE

예:

```text
A.md → B.md
```

하나의 논리적인 Change지만 두 Path State를 변경한다.

```text
A.md
PRESENT → DELETED

B.md
ABSENT → PRESENT
```

두 Effect 모두 같은 Global Revision을 사용한다.

예:

```text
Revision 1050

RENAME A.md → B.md
```

결과:

```text
A.md
latestRevision = 1050
state = DELETED

B.md
latestRevision = 1050
state = PRESENT
```

---

# 12. Rename은 하나의 Revision이다

초기 모델에서 Rename을:

```text
DELETE A.md = Revision 100

CREATE B.md = Revision 101
```

로 분리하지 않는다.

대신:

```text
Revision 100
RENAME A.md → B.md
```

라는 하나의 Change로 표현한다.

이를 통해 Client가 Rename을 하나의 논리적 동작으로 관찰할 수 있다.

Filesystem 적용 역시 가능한 경우 atomic rename을 사용한다.

---

# 13. Operation Record

Operation은 Server가 받은 하나의 논리적인 Mutation Request 및 그 처리 결과를 나타낸다.

개념적으로:

```text
OperationRecord

operationId
actorClientId
operationType

requestDigest

status

baseConditions

stagingReference

resultRevision

createdAt
completedAt
```

---

# 14. Operation과 Change는 다른 개념이다

Operation은:

```text
Client Request / Idempotency Unit
```

이다.

Change는:

```text
Committed Server History
```

이다.

즉:

```text
Operation
     │
     │ successful commit
     ▼
Change
```

관계다.

Commit되지 않은 Operation은 Change를 만들지 않는다.

---

## 14.1 MVP 관계

초기 버전에서는 대부분:

```text
1 Operation
    →
1 Change
```

이다.

예:

```text
MODIFY request
    →
MODIFY change
```

향후 Multi-file Transaction을 지원하면 하나의 Operation 또는 Transaction이 여러 Change를 만들 수 있으므로 논리 모델 자체를 반드시 영구적인 1:1 관계로 제한하지 않는다.

---

# 15. Operation Request Digest

Operation ID 재사용 오류를 탐지하기 위해 Server는 요청의 identity를 확인할 수 있어야 한다.

예:

첫 요청:

```text
operationId = OP-1

MODIFY A.md

hash = AAA
```

Retry:

```text
operationId = OP-1

MODIFY A.md

hash = AAA
```

는 정상이다.

하지만:

```text
operationId = OP-1

DELETE B.md
```

가 오면 동일 Operation Retry로 취급해서는 안 된다.

이를 위해:

```text
requestDigest
```

또는 동일한 역할의 immutable request identity를 저장한다.

---

# 16. Base Condition

Operation은 자신이 어떤 Server State를 기준으로 만들어졌는지 표현한다.

하나의 Operation은 하나 이상의 Base Condition을 가질 수 있다.

개념적으로:

```text
BaseCondition

path
expectedState
expectedRevision
expectedHash
```

---

## 16.1 Existing File Base

MODIFY:

```text
path = A.md

expectedState = PRESENT
expectedRevision = 100
expectedHash = AAA
```

Server의 현재 A.md가 이 상태와 일치해야 Commit 가능하다.

---

## 16.2 Delete Base

DELETE 역시:

```text
expectedState = PRESENT
expectedRevision = 100
expectedHash = AAA
```

를 사용한다.

---

 ## 16.3 Create Base
 
CREATE는 Path의 과거 상태를 명시적으로 구분한다.
 
한 번도 존재한 적 없다고 Client가 알고 있는 새로운 Path를 생성하는 경우:
 
```text
expectedState = UNKNOWN
```
 
를 사용한다.

Server의 현재 Path State가 `PRESENT` 또는 `DELETED`라면 Base Condition이 일치하지 않으므로 Conflict다.

특히 `DELETED` 상태를 `UNKNOWN`과 구분함으로써 오래된 Client가 삭제된 파일을 새로운 파일로 오인하여 암묵적으로 부활시키는 것을 방지한다.

삭제된 Path를 사용자가 명시적으로 복원하는 경우에는:

```text
expectedState = DELETED
expectedRevision = <known delete revision>
```

을 사용한다.

예:

```text
Server:

A.md
state = DELETED
revision = 200
```

사용자가 해당 파일의 복원을 명시적으로 선택했다면:

```text
CREATE A.md

expectedState = DELETED
expectedRevision = 200
```

으로 표현한다.

따라서 CREATE에서:

```text
UNKNOWN
    =
새로운 Path를 생성하려는 의도

DELETED
    =
삭제 사실을 알고 해당 Path를 명시적으로 복원하려는 의도
```

를 구분한다.

## 16.4 Rename Base

Rename:

```text
A.md → B.md
```

에서는 최소:

```text
A.md
expectedState = PRESENT
expectedRevision = 100
expectedHash = AAA
```

그리고:

```text
B.md
expectedState = ABSENT
```

를 검증한다.

즉 Rename은 source와 destination 두 Path에 대한 조건을 가진다.

---

# 17. Operation Status

Server Operation은 다음 상태를 가질 수 있다.

```text
PREPARED

COMMITTED

REJECTED_CONFLICT

FAILED
```

---

## 17.1 PREPARED

Commit Intent가 durable하게 기록되었으며 Filesystem Apply 또는 Recovery가 아직 진행 중일 수 있다.

---

## 17.2 COMMITTED

Filesystem과 Sync State가 모두 확정되었다.

`resultRevision`을 가진다.

---

## 17.3 REJECTED_CONFLICT

Base Validation이 실패하여 Authoritative State에는 아무 변경도 발생하지 않았다.

동일 Operation ID가 Retry되는 경우 같은 요청이라는 것이 확인되면 기존 Conflict 결과를 반환할 수 있다.

---

## 17.4 FAILED

정상 자동 Recovery가 불가능한 terminal failure를 나타낼 수 있다.

Transient Network 또는 Upload 실패처럼 Server Commit Protocol에 진입하지 않은 실패는 반드시 durable FAILED Operation으로 기록할 필요가 없다.

---

# 18. Revision 할당 시점

Global Revision은 `PREPARED` 단계에서 할당하지 않는다.

Revision은 Filesystem Apply 이후 Sync State Finalization Transaction에서 할당한다.

```text
PREPARED
    │
Filesystem Apply
    │
    ▼
Finalize Transaction
    │
    ├── allocate revision
    ├── update Path State
    ├── append Change
    ├── mark Operation COMMITTED
    └── update currentRevision
```

따라서:

```text
Revision
    =
Committed Change Order
```

의 의미가 유지된다.

---

# 19. Server Actor

Change가 어디에서 발생했는지 진단할 수 있도록 Actor 정보를 가질 수 있다.

예:

```text
CLIENT

SERVER_EXTERNAL

SYSTEM
```

Client Mutation이면:

```text
actorType = CLIENT
actorClientId = C-1
```

외부 Filesystem Drift를 Server가 Journal에 편입했다면:

```text
actorType = SERVER_EXTERNAL
```

로 표현할 수 있다.

---

# 20. External Change도 Operation으로 표현한다

가능하면 Server 외부 변경을 Journal에 편입할 때도 Server-generated Operation ID를 생성한다.

```text
External Drift
      │
      ▼
Server-generated Operation
      │
      ▼
Committed Change
```

이를 통해 모든 committed mutation이 동일한 데이터 흐름을 가진다.

---

# 21. Server Recovery Data

Server Commit Protocol의 Recovery를 위해 Operation은 Filesystem상의 internal artifact와 연결될 수 있다.

예:

```text
OP-1

stagingReference
recoveryReference
```

실제 Content는 SQLite 안에 저장할 필요가 없다.

```text
Operation Metadata
       │
       ▼
Filesystem Artifact
```

관계를 가진다.

---

# 22. Server Logical Relationship

전체 관계를 단순화하면 다음과 같다.

```text
VaultMetadata
    │
    └── currentRevision
             │
             ▼
        Change Journal
             │
             │ updates
             ▼
         Path State


Operation
    │
    │ commit
    ▼
Change Record
    │
    └── Path Effects
            │
            ▼
        Path State
```

Filesystem:

```text
Path State
    │
    │ describes
    ▼
Vault Filesystem
```

---

# 23. Client Data Model

Client는 다음 주요 상태를 가진다.

```text
Client State

├── Client Metadata
├── Replica Index
├── Pending Operations
├── Apply Journal
├── Conflicts
└── Client Artifacts
```

---

# 24. Client Metadata

Client 전체 Sync 상태를 나타낸다.

개념적으로:

```text
ClientMetadata

clientId
vaultId
serverCursor
```

---

## 24.1 Server Cursor

`serverCursor`는 Client가 Server Change Stream을 연속적으로 어디까지 처리했는지를 나타낸다.

```text
serverCursor = 5000
```

의 의미:

> Revision 5000 이하의 모든 Server Change는 Local에 적용되었거나 안전하게 Conflict로 기록되었다.

---

## 24.2 Vault Binding

Client Metadata의 `vaultId`는 Cursor가 어느 Server Vault의 Revision인지 명시한다.

```text
vaultId = V-1
serverCursor = 5000
```

은 하나의 세트다.

Vault ID가 바뀌면 Cursor만 유지해서는 안 된다.

---

# 25. Replica Entry

Replica Index의 한 항목이다.

개념적으로:

```text
ReplicaEntry

path
entryType
serverState
serverRevision
serverHash
```

`serverState`:

```text
PRESENT

DELETED
```

---

## 25.1 Present Replica Entry

```text
path = A.md

serverState = PRESENT

serverRevision = 100

serverHash = AAA
```

의 의미:

> 이 Client가 마지막으로 처리한 Server의 A.md 상태는 Revision 100 / Hash AAA이다.

Local Vault의 실제 Hash가 반드시 AAA라는 뜻은 아니다.

---

## 25.2 Local Pending이 있는 경우

예:

```text
Replica:

A.md = AAA
```

Local:

```text
A.md = BBB
```

이면:

```text
AAA
    =
Base Server State

BBB
    =
Uncommitted Local State
```

이다.

Replica Index를 BBB로 갱신하면 안 된다.

---

# 26. Deleted Replica Entry

Server Delete를 처리한 후에도 Replica Entry를 즉시 제거하지 않는다.

예:

```text
path = A.md
serverState = DELETED
serverRevision = 105
```

이는:

```text
이 Client는
A.md가 Server에서 삭제되었다는 사실을 알고 있다
```

는 의미다.

---

## 26.1 Client에서도 Unknown과 Deleted를 구분한다

Replica Entry 없음:

```text
UNKNOWN
```

Replica Entry 존재:

```text
DELETED
```

는 서로 다르다.

이 구분은 오래된 파일의 잘못된 resurrection을 탐지하는 데 중요하다.

---

# 27. Pending Operation

아직 정상적인 Client Sync Lifecycle을 완료하지 않은 Local Mutation이다.

개념적으로:

```text
PendingOperation

operationId
operationType

path
previousPath

baseConditions

localState

status

createdAt
```

---

# 28. Pending Local State

파일 Operation이면 최소:

```text
localHash
```

를 가진다.

예:

```text
MODIFY A.md

baseHash = AAA
localHash = BBB
```

Server Upload 직전에 실제 Local File의 Hash가 BBB인지 다시 확인할 수 있다.

---

## 28.1 Local Content가 다시 변경된 경우

Pending:

```text
localHash = BBB
```

인데 실제 Local Vault:

```text
hash = CCC
```

라면 BBB Operation을 그대로 Upload하지 않는다.

Pending Operation을 재분석하거나 안전하게 coalesce한다.

---

# 29. Pending Operation Status

Client에서는 다음 상태를 사용할 수 있다.

```text
PENDING

SENDING

SERVER_COMMITTED

CONFLICT
```

---

## 29.1 PENDING

아직 Server Commit이 확인되지 않은 Local Operation이다.

---

## 29.2 SENDING

현재 전송 중일 수 있다.

이 상태 자체는 correctness를 위해 반드시 durable할 필요는 없다.

Operation ID와 Pending 내용이 durable한 것이 중요하다.

---

## 29.3 SERVER_COMMITTED

Server Success Response를 받아 Result Revision을 알고 있지만 Client Change Stream에는 아직 해당 Revision까지 통합되지 않은 상태다.

예:

```text
serverCursor = 100

own operation committed = 103
```

아직:

```text
101
102
```

를 처리하지 않았으므로 Cursor를 103으로 올리지 않는다.

Pending Operation은 자신의 Change가 정상적으로 Change Stream에 통합될 때까지 유지할 수 있다.

---

## 29.4 CONFLICT

Server가 Base mismatch를 반환했다.

자동 Retry 대상에서 제외한다.

Conflict Store와 연결된다.

---

# 30. Apply Record

Server Change를 Local Vault에 적용하는 도중 Client가 종료될 수 있으므로 Remote Apply Intent를 durable하게 표현한다.

개념적으로:

```text
ApplyRecord

revision
changeType
path

beforeState
afterState

artifactReference

status
```

---

## 30.1 Apply Lifecycle

기본 흐름:

```text
PREPARED

    ↓

Local Filesystem Apply

    ↓

Replica Update

    ↓

COMPLETED
```

완료 후 Apply Record는 제거할 수 있다.

---

# 31. Client Artifact

모든 Content를 metadata store에 직접 넣을 필요는 없다.

Conflict Snapshot이나 Download Temp File과 같은 큰 데이터는 별도 Client-local Artifact로 보존할 수 있다.

개념적으로:

```text
ClientArtifact

artifactId
artifactType
contentHash
storageReference
createdAt
```

종류 예:

```text
REMOTE_DOWNLOAD

CONFLICT_LOCAL_SNAPSHOT

CONFLICT_SERVER_SNAPSHOT

MERGE_RESULT

RECOVERY_COPY
```

---

# 32. Artifact는 Sync 대상이 아니다

Client Artifact는 Local Vault 사용자 Content가 아니다.

따라서:

```text
Client Artifact
    X
Server Sync
```

이다.

Obsidian Vault Change Detection에서도 제외해야 한다.

---

# 33. Conflict Record

Conflict의 durable state를 표현한다.

개념적으로:

```text
ConflictRecord

conflictId
conflictType

path
relatedPath

originalOperationId

baseState

latestServerState
localState

localArtifactReference
serverArtifactReference
mergeArtifactReference

status

resolutionOperationId

createdAt
resolvedAt
```

---

# 34. Conflict의 Base State

Conflict가 어떤 상태에서 갈라졌는지를 기록한다.

예:

```text
baseRevision = 100
baseHash = AAA
```

이는 Conflict 진단과 사용자 비교에 사용된다.

---

# 35. Latest Server State

Conflict가 unresolved 상태인 동안 Server는 계속 변경될 수 있다.

따라서 Conflict는 최초 Server State만 저장하는 것이 아니라:

```text
latestServerRevision
latestServerHash
```

를 추적한다.

필요하면 최신 Server Content를 Client Artifact로 보존한다.

---

# 36. Local Conflict State

Conflict 발생 당시 Local Content를 안전하게 보존해야 한다.

최소:

```text
localHash
```

를 기록한다.

데이터 보존을 강화하기 위해 Conflict 발생 시 Local Content Snapshot을 Client Artifact에 만들 수 있다.

```text
Conflict
    │
    └── localArtifactReference
```

이는 이후 사용자가 Local Vault를 계속 편집하더라도 최초 Conflict Content를 잃지 않게 한다.

---

# 37. Resolution Operation

사용자가:

```text
Apply Local

Manual Merge

Restore Local

Delete Server
```

등의 Resolution을 선택하면 기존 Operation을 재사용하지 않는다.

새로운:

```text
resolutionOperationId
```

를 생성한다.

Conflict Record는 해당 Operation과 연결된다.

```text
Conflict
    │
    ▼
Resolution Operation
    │
    ▼
Server Change
```

---

# 38. Conflict History

Conflict가 해결된 이후 Metadata 자체를 바로 삭제할 필요는 없다.

상태:

```text
RESOLVED
```

로 변경하고 최소한의 History만 유지할 수 있다.

예:

```text
conflictType

resolutionType

resultRevision

resolvedAt
```

대용량 Artifact는 별도 Retention Policy에 따라 제거할 수 있다.

---

# 39. Server와 Client State 대응

같은 파일의 상태는 다음처럼 연결된다.

정상 상태:

```text
Server PathState

A.md
revision = 100
hash = AAA
state = PRESENT


Client ReplicaEntry

A.md
revision = 100
hash = AAA
state = PRESENT


Local Vault

A.md = AAA
```

완전히 수렴한 상태다.

---

# 40. Local Pending 상태

```text
Server:

A.md = AAA
Revision 100


Replica:

A.md = AAA
Revision 100


Local:

A.md = BBB


Pending:

MODIFY
Base = 100 / AAA
Local = BBB
```

Server와 Replica는 아직 바뀌지 않는다.

---

# 41. Commit 이후 Pull 이전

Server Commit:

```text
Revision 101
A.md = BBB
```

Client:

```text
serverCursor = 100

Replica:
A.md = AAA

Pending:
SERVER_COMMITTED
resultRevision = 101

Local:
A.md = BBB
```

이 상태는 정상적인 transient state다.

Client가 Revision 101을 Pull하면:

```text
Replica:
revision = 101
hash = BBB

serverCursor = 101

Pending 제거
```

가 된다.

---

# 42. Conflict 상태

예:

```text
Base:

Revision 100
AAA
```

다른 Client Commit:

```text
Server:

Revision 101
BBB
```

Local:

```text
CCC
```

Client가 Revision 101을 처리하면서 Conflict를 안전하게 기록했다면 Client State는 다음과 같다.

```text
Replica:

101 / BBB

Local:

CCC

Pending:

CONFLICT

Conflict:

base = 100 / AAA
server = 101 / BBB
local = CCC

Server Cursor:

101
```

이 상태에서 각 정보의 의미는 다음과 같다.

```text
Replica
    =
Client가 현재 알고 있는 최신 Server State

Local Vault
    =
아직 Server에 반영되지 않은 사용자 Content

Conflict.base
    =
Local Change가 시작된 과거 Server State

Conflict.server
    =
Conflict 처리 시점에 알고 있는 Server State
```

Conflict가 발생했다고 Replica Index를 과거 Base State에 유지하지 않는다.

Revision 101을 안전하게 처리하여 Server Cursor를 101까지 진행했다면 Replica Index 역시 Revision 101의 Server State를 반영해야 한다.

과거 Base State:

```text
100 / AAA
```

는 Replica Index가 아니라:

```text
Conflict.baseState
```

에 보존한다.

따라서:

```text
Server Cursor = 101

Replica = 101 / BBB

Local = CCC
```

는 모순이 아니다.

이는 Client가 Server의 최신 상태 `BBB`를 알고 있지만 Local Vault에는 사용자의 Conflict Content `CCC`를 의도적으로 보존하고 있다는 의미다.

Conflict가 unresolved 상태인 동안 Server에서 같은 Path가 다시 변경되면 Replica와 Conflict의 latest Server State를 계속 갱신할 수 있다.

예:

```text
Revision 102
Server = DDD
```

를 처리한 이후:

```text
Replica:

102 / DDD

Local:

CCC

Conflict:

base = 100 / AAA
latestServer = 102 / DDD
local = CCC

Server Cursor:

102
```

가 될 수 있다.

Local Conflict Content는 사용자가 Resolution을 수행하기 전까지 자동으로 덮어쓰지 않는다.

---

# 43. Delete State

Server:

```text
Revision 200

DELETE A.md
```

Current Path State:

```text
A.md

state = DELETED
latestRevision = 200
```

Client가 처리한 이후:

```text
Replica:

A.md
state = DELETED
serverRevision = 200
```

Local Vault에는 A.md가 존재하지 않는다.

---

# 44. Rename State

Server:

```text
Revision 300

RENAME A.md → B.md
```

Server Current State:

```text
A.md

state = DELETED
latestRevision = 300
```

```text
B.md

state = PRESENT
latestRevision = 300
hash = BBB
```

Client Replica도 같은 두 상태를 표현한다.

이렇게 함으로써 과거 Path가 사라졌다는 사실을 잃지 않는다.

---

# 45. Server SQLite Logical Mapping

실제 SQLite Schema를 아직 고정하지 않지만 논리적으로 다음 구조로 매핑할 수 있다.

```text
vault_metadata

path_state

change_journal

change_effect

operations

operation_base_condition
```

Recovery Content 자체는:

```text
/data/staging

/data/recovery
```

등의 Filesystem에 존재한다.

정확한 Table Column, Index, Foreign Key 및 SQLite 설정은 구현 명세에서 정의한다.

---

# 46. Client Storage Logical Mapping

Client 역시 논리적으로 다음 저장 영역이 필요하다.

```text
client_metadata

replica_index

pending_operations

apply_journal

conflicts

conflict_history
```

그리고 대용량 또는 Content Snapshot:

```text
client_artifacts
```

을 위한 Client-local Storage가 필요하다.

정확한 기술은 이 문서에서 결정하지 않는다.

---

# 47. Metadata Store와 Artifact Store를 분리한다

특히 Client에서는 다음을 모두 하나의 거대한 JSON 문서에 넣는 것을 요구하지 않는다.

```text
Replica Index

Pending Metadata

Conflict Binary Content

Downloaded Attachments
```

개념적으로:

```text
Metadata Store
     +
Artifact Store
```

로 분리할 수 있어야 한다.

Metadata Store는 작은 structured state를 담당한다.

Artifact Store는 비교적 큰 Content Snapshot과 Temporary File을 담당한다.

---

# 48. Retention Policy

## 48.1 Server Path Tombstone

초기 버전에서는 삭제된 Path State를 장기간 유지한다.

개인 Vault 규모에서는 correctness를 우선한다.

---

## 48.2 Server Change Journal

초기 버전에서는 가능한 한 장기간 보존한다.

---

## 48.3 Server Operation Record

초기 버전에서는 COMMITTED Operation 역시 장기간 보존한다.

이유:

```text
Server Commit

Client response lost

Client offline for long period

same operationId retry
```

상황에서도 Idempotency를 유지하기 위해서다.

---

## 48.4 Client Replica Tombstone

초기 버전에서는 `DELETED` Replica Entry를 자동 제거하지 않는다.

오래된 파일이 다시 나타났을 때:

```text
Never Seen File
```

인지:

```text
Previously Deleted File
```

인지 구분할 수 있어야 하기 때문이다.

---

## 48.5 Completed Apply Record

Remote Apply가 완전히 Finalize된 Apply Record는 즉시 또는 짧은 기간 후 제거할 수 있다.

이미 Replica Index가 authoritative metadata를 가지고 있기 때문이다.

---

## 48.6 Resolved Conflict Artifact

Resolved Conflict의 Metadata는 History로 유지할 수 있다.

대용량 Local / Server Snapshot은 Retention Policy에 따라 제거할 수 있다.

---

## 48.7 Server Staging / Recovery Artifact

Operation이 COMMITTED되고 더 이상 Crash Recovery에 필요하지 않으면 Garbage Collection할 수 있다.

---

# 49. Garbage Collection 원칙

Metadata를 삭제하는 것은 단순한 Storage Optimization이 아니다.

특정 Metadata를 제거하면 Client가 과거 상태를 이해하는 능력이 사라질 수 있다.

따라서 GC는 다음 원칙을 따른다.

```text
Correctness
    >
Storage Saving
```

초기 개인용 시스템에서는 aggressive GC를 사용하지 않는다.

---

# 50. Data Model Invariants

## Invariant 1 — Path State Describes Current Server State

각 알려진 Sync Path는 Server에서 하나의 Current Path State를 가진다.

---

## Invariant 2 — Deleted Is State

삭제된 Path는 단순히 Metadata에서 사라지는 것이 아니라 `DELETED` 상태로 표현될 수 있어야 한다.

---

## Invariant 3 — Unknown Is Not Deleted

Path Metadata가 없는 것과 명시적으로 삭제된 것은 서로 다른 상태다.

---

## Invariant 4 — Revision Belongs to Committed Change

Global Revision은 PREPARED Operation이 아니라 committed Change에만 존재한다.

---

## Invariant 5 — Revision Orders Changes

Global Revision은 전체 Vault의 committed Change에 대한 하나의 순서를 제공한다.

---

## Invariant 6 — Rename Is One Logical Change

Rename / Move는 source와 destination 두 Path를 변경하지만 하나의 logical Change와 하나의 Global Revision으로 표현한다.

---

## Invariant 7 — Operation Identity Is Immutable

같은 Operation ID가 다른 Request를 의미해서는 안 된다.

---

## Invariant 8 — Operation and Change Are Distinct

Operation은 요청 및 Idempotency 단위이고 Change는 committed history 단위다.

---

## Invariant 9 — Replica Index Stores Server Knowledge

Client Replica Index는 Local Vault 현재 상태가 아니라 Client가 알고 있는 Server State를 표현한다.

---

## Invariant 10 — Pending Preserves Base State

Local Pending Operation은 자신이 어떤 Server State를 기반으로 생성되었는지 잃어서는 안 된다.

---

## Invariant 11 — Cursor Does Not Replace Replica State

Server Cursor와 Replica Index는 서로 다른 정보를 표현하며 하나가 다른 하나를 대체하지 않는다.

---

## Invariant 12 — Conflict Preserves User Alternatives

Conflict 상태는 Local 사용자의 대안 상태를 Server Commit 또는 명시적인 사용자 결정 전까지 보존해야 한다.

---

## Invariant 13 — Resolution Creates a New Operation

Conflict Resolution에서 과거 Conflict Operation을 변형하여 재사용하지 않는다.

---

## Invariant 14 — Client Artifacts Never Become Vault Content Implicitly

Conflict Snapshot, Recovery Copy, Download Temp 등의 Client Artifact가 자동으로 synchronized Vault Content가 되어서는 안 된다.

---

## Invariant 15 — Vault ID Scopes Revision Meaning

Revision과 Server Cursor는 해당 Vault ID 안에서만 의미가 있다.

다른 Vault ID에 기존 Cursor를 적용해서는 안 된다.

---

# 51. 전체 논리 모델

```text
                         SERVER

                    VaultMetadata
                         │
                         │ currentRevision
                         ▼
                   Change Journal
                         │
                  ┌──────┴──────┐
                  │             │
                  ▼             ▼
             Path Effect    Operation
                  │             │
                  ▼             │
               PathState ◄──────┘
                  │
                  │ describes
                  ▼
             Vault Filesystem


                         CLIENT

                   ClientMetadata
                  /      │       \
                 /       │        \
                ▼        ▼         ▼
         serverCursor  vaultId   clientId

                Replica Index
                     │
          ┌──────────┼───────────┐
          │          │           │
          ▼          ▼           ▼
       Pending     Apply       Conflict
      Operation   Journal       Store
          │                        │
          │                        ▼
          │                 Client Artifact
          │
          ▼
        Server
```

이 Data Model의 핵심은:

> **파일의 현재 모습만 저장하는 것이 아니라, 각 Client가 어떤 Server State를 알고 있고 어떤 Local Change가 그 상태를 기반으로 만들어졌는지를 명시적으로 데이터로 보존하는 것**

이다.
