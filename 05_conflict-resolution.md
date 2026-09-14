# Conflict Resolution

## 1. 문서 목적

이 문서는 동기화 중 발생한 Conflict를 사용자가 어떻게 이해하고 해결하는지를 정의한다.

Conflict의 감지·cursor 전이 규칙은 [02 Synchronization Protocol](./02_synchronization-protocol.md), Conflict Record와 resolution operation의 논리 상태는 [06 Data Model](./06_data-model.md), 실제 API 요청은 [07 API Specification](./07_api-specification.md)을 따른다. 이 문서는 그 공통 규칙을 다시 정의하지 않고 사용자 경험과 resolution 선택만 정의한다.

Conflict는 단순한 동기화 오류가 아니다.

서버와 로컬에서 같은 기준 상태로부터 서로 다른 변경이 발생하여 어떤 결과를 최종 authoritative state로 사용할지 자동으로 결정할 수 없는 상태이다.

예:

```text
Base
 │
 ├── Server Change
 │
 └── Local Change
```

Conflict Resolution의 목적은 어느 한쪽 데이터를 자동으로 버리는 것이 아니라:

```text
Server State
     +
Local State
     +
User Decision
     │
     ▼
New Authoritative State
```

를 안전하게 만드는 것이다.

본 문서는 다음을 정의한다.

* Conflict의 사용자 관점 의미
* Conflict Lifecycle
* Conflict 목록과 상세 화면
* Conflict Type별 Resolution Action
* Server Version 선택
* Local Version 적용
* Manual Merge
* Delete Conflict
* Rename / Move Conflict
* Resolution 중 Server가 다시 변경된 경우
* Resolution Commit과 완료 조건
* Conflict History

자동 3-way merge는 초기 버전의 필수 기능으로 다루지 않는다.

---

## 2. 핵심 원칙

### 2.1 Conflict는 데이터 보존 상태다

Conflict가 발생하면 어느 한쪽 데이터를 자동으로 삭제하거나 덮어쓰지 않는다.

최소 다음 상태를 보존한다.

```text
Base State

Current Server State

Local State
```

초기 버전에서 Base Content 전체를 항상 보존할 필요는 없지만, Conflict를 이해하고 해결하는 데 필요한 정보는 유지해야 한다.

---

### 2.2 Server는 계속 Authoritative하다

Conflict가 발생했다고 Server Authority 원칙이 사라지지 않는다.

Server의 현재 상태는 여전히 전체 시스템의 authoritative state다.

Local Conflict State는:

```text
아직 Server가 받아들이지 않은
사용자의 대안 상태
```

로 취급한다.

---

### 2.3 Local 선택은 강제 덮어쓰기가 아니다

사용자가 Local Version을 선택했다고 기존 Conflict Operation을 강제로 Server에 적용하지 않는다.

예:

```text
Original Base:

Revision 100
Hash AAA
```

Conflict 발생:

```text
Server:

Revision 101
Hash BBB

Local:

Hash CCC
```

사용자가 Local을 최종 상태로 선택하면 새로운 Operation을 생성한다.

```text
New MODIFY

baseRevision = 101
baseHash = BBB

contentHash = CCC
```

즉:

```text
Conflict Resolution
    =
현재 Authoritative State를 기준으로
새로운 사용자 변경을 만드는 과정
```

이다.

---

### 2.4 Resolution은 항상 최신 Server State를 기준으로 한다

Conflict가 발생한 이후에도 다른 Client가 같은 파일을 수정할 수 있다.

예:

```text
100 Base

101 Server B
    ← Conflict 발생

102 Server C
    ← Conflict 해결 전 추가 변경
```

사용자가 Resolution을 수행하는 시점에는 Revision 101이 아니라 Revision 102를 기준으로 판단해야 한다.

따라서 Conflict 화면에 보이는 Server Version은 가능한 한 최신 상태를 반영한다.

---

### 2.5 기술 용어보다 결과를 보여준다

사용자에게 다음과 같은 표현을 기본 UI로 사용하지 않는다.

```text
baseRevision mismatch

412 conflict

hash mismatch
```

사용자에게는 결과를 중심으로 보여준다.

예:

```text
이 문서는 다른 기기에서도 변경되었습니다.

서버 버전
2026-09-13 00:03

이 기기의 버전
2026-09-13 00:05
```

Resolution Action 역시:

```text
Accept Server
Accept Local
```

보다:

```text
서버 버전 사용

이 기기의 버전을 서버에 적용

두 버전을 비교해 직접 합치기
```

처럼 실제 결과를 설명하는 것이 우선이다.

---

# 3. Conflict Lifecycle

Conflict는 다음 Lifecycle을 가진다.

```text
DETECTED
   │
   ▼
UNRESOLVED
   │
   ├─────────────┐
   │             │
   ▼             ▼
RESOLVING    SERVER_UPDATED
   │             │
   └──────┬──────┘
          │
          ▼
     COMMITTING
          │
     ┌────┴────┐
     │         │
     ▼         ▼
 RESOLVED   RECONFLICTED
```

---

## 3.1 DETECTED

Client가 다음과 같은 상황을 발견한다.

```text
Base = AAA

Server = BBB

Local = CCC
```

Conflict Store에 durable하게 기록한다.

---

## 3.2 UNRESOLVED

사용자 선택을 기다리는 상태다.

해당 Path는 자동 Push 및 destructive Remote Apply에서 격리된다.

다른 Path의 동기화는 계속 수행한다.

---

## 3.3 SERVER_UPDATED

Conflict가 해결되지 않은 동안 Server State가 다시 변경된 경우다.

Local Conflict Content는 그대로 유지하고 Conflict Record가 최신 Server State를 추적한다.

---

## 3.4 RESOLVING

사용자가 해결 Action을 선택했거나 Manual Merge 작업을 수행하고 있는 상태다.

아직 Server authoritative state가 변경된 것은 아니다.

---

## 3.5 COMMITTING

사용자 Resolution 결과를 새로운 Operation으로 Server에 제출하고 있다.

---

## 3.6 RESOLVED

Resolution 결과가 Server에 Commit되고 Client가 해당 Server Change를 정상적으로 통합한 상태다.

이 시점에서 Conflict Record를 active state에서 제거할 수 있다.

---

## 3.7 RECONFLICTED

Resolution을 Commit하려는 사이 Server State가 다시 변경되어 새로운 Base mismatch가 발생한 상태다.

사용자의 Resolution 결과를 버리지 않는다.

최신 Server State와 다시 비교할 수 있도록 Conflict 상태로 되돌린다.

---

# 4. Conflict Center

Conflict가 하나 이상 존재하면 사용자가 이를 한곳에서 확인할 수 있어야 한다.

예:

```text
Conflicts (3)

● notes/kubernetes.md
  이 기기와 서버에서 모두 수정됨

● projects/design.md
  서버에서 삭제됨 / 이 기기에서 수정됨

● inbox/draft.md
  같은 이름의 문서가 양쪽에서 생성됨
```

Conflict 목록은 최소 다음 정보를 제공한다.

```text
Path

Conflict Type

Local 변경 시각

Server 변경 시각

현재 Resolution 상태
```

Revision, Hash 등의 내부 정보는 기본 화면에 노출하지 않는다.

필요한 경우 진단 정보로 별도 제공할 수 있다.

---

# 5. Conflict Detail

사용자가 하나의 Conflict를 선택하면 해당 상황을 이해할 수 있어야 한다.

Modify Conflict 예:

```text
┌────────────────────────────────────┐
│ notes/kubernetes.md                │
│                                    │
│ 다른 기기에서도 수정되었습니다.     │
│                                    │
│ Server Version                     │
│ 2026-09-13 00:03                   │
│                                    │
│ Local Version                      │
│ 2026-09-13 00:05                   │
│                                    │
│ [차이 보기]                        │
│                                    │
│ [서버 버전 사용]                   │
│ [이 기기 버전을 서버에 적용]        │
│ [직접 합치기]                      │
└────────────────────────────────────┘
```

---

# 6. Resolution Action 모델

초기 버전은 다음 Resolution Action을 기본으로 지원한다.

```text
USE_SERVER

APPLY_LOCAL

MANUAL_MERGE

KEEP_DELETED

RESTORE_LOCAL

KEEP_BOTH
```

Conflict Type에 따라 사용할 수 있는 Action이 달라진다.

---

# 7. Modify vs Modify

가장 일반적인 Conflict다.

```text
Base = A

Server = B

Local = C
```

사용자에게 세 가지 기본 선택을 제공한다.

---

## 7.1 서버 버전 사용

사용자가 Server Version을 선택한다.

```text
Local C
    ↓ discard as active content

Server B
    ↓
Local Vault
```

처리 흐름:

```text
1. 최신 Server State 확인

2. Local Conflict Content 보존 여부 확인

3. Server Content를 Local Vault에 적용

4. Replica Index 갱신

5. Conflict 완료
```

이 Action은 Server mutation을 발생시키지 않는다.

Server가 이미 authoritative state이기 때문이다.

---

## 7.2 이 기기 버전을 서버에 적용

사용자가 Local Version을 최종 결과로 선택한다.

현재 Server:

```text
revision = 101
hash = B
```

Local:

```text
hash = C
```

새로운 Operation:

```text
MODIFY

baseRevision = 101
baseHash = B

contentHash = C
```

을 생성한다.

Commit 성공:

```text
Revision 102

Server = C
```

이후 Conflict를 완료한다.

---

## 7.3 직접 합치기

사용자가 Server Version과 Local Version을 비교하여 새로운 Content `D`를 만든다.

```text
Server B
       \
        → User Merge → D
       /
Local C
```

그 결과는 현재 Server State를 Base로 하는 새로운 Operation이다.

```text
MODIFY

baseRevision = current server revision
baseHash = current server hash

contentHash = D
```

Commit 이후:

```text
Server = D
```

가 된다.

---

# 8. Manual Merge UX

Markdown Conflict의 Manual Merge에서는 최소 다음 정보를 제공하는 것이 좋다.

```text
Server Version

Local Version

Diff
```

가능하면 Side-by-side 또는 unified diff를 제공한다.

예:

```text
Server                         Local

Kubernetes는 ...              Kubernetes는 ...

- 기존 설명                   - 수정한 설명
+ 서버에서 추가된 설명         + 내가 추가한 설명
```

사용자는 결과 문서를 직접 편집할 수 있다.

초기 버전에서는 완전한 IDE 수준 Merge Editor를 요구하지 않는다.

다음 정도면 충분하다.

```text
Server 확인

Local 확인

Diff 확인

Merged Result 편집
```

---

# 9. Delete vs Modify

예:

```text
Server:
A.md deleted

Local:
A.md modified
```

기술적으로는:

```text
DELETE vs MODIFY
```

지만 사용자에게는 다음 의미로 보여준다.

> 이 문서는 다른 기기에서 삭제되었지만 이 기기에서는 수정되었습니다.

기본 Action:

```text
삭제 유지

이 기기의 문서를 복원
```

---

## 9.1 삭제 유지

Server의 Delete를 받아들인다.

Local Modified Content를 active Vault에서 제거한다.

필요한 경우 Conflict Recovery 영역에 보존할 수 있다.

Conflict를 완료한다.

---

## 9.2 Local 문서 복원

삭제된 Server Path에 Local Content를 다시 생성한다.

이는 과거 Delete를 취소하는 것이 아니라 새로운 CREATE Operation이다.

```text
Server:

A.md = deleted
Revision = 101
```

Resolution:

```text
CREATE A.md

base = current deleted state

content = Local
```

Commit 성공:

```text
Revision 102

CREATE A.md
```

즉:

```text
Delete
  ↓
New Create
```

로 이력이 남는다.

---

# 10. Modify vs Delete

반대 상황도 존재할 수 있다.

예:

```text
Server:
A.md modified

Local:
A.md deleted
```

사용자에게:

> 이 문서는 다른 기기에서 수정되었지만 이 기기에서는 삭제되었습니다.

선택:

```text
서버 문서 유지

문서를 서버에서도 삭제
```

---

## 10.1 서버 문서 유지

현재 Server Content를 Local에 복원한다.

Server mutation은 필요 없다.

---

## 10.2 서버에서도 삭제

현재 Server State를 Base로 새로운 DELETE Operation을 제출한다.

```text
DELETE A.md

baseRevision = current
baseHash = current
```

Commit이 성공하면 Conflict가 해결된다.

---

# 11. Create vs Create

같은 Path에 서로 다른 파일이 생성된 경우다.

```text
Server:
A.md = B

Local:
A.md = C
```

Base가 존재하지 않았다는 점만 다르고 사용자 관점에서는 두 개의 서로 다른 문서다.

기본 Action:

```text
서버 버전 사용

이 기기 버전으로 교체

두 문서 모두 유지
```

---

## 11.1 두 문서 모두 유지

데이터 보존 측면에서 중요한 선택이다.

예:

```text
Server:
A.md

Local:
A.md
```

사용자가 Local Version의 새 이름을 결정한다.

```text
A.md

A-local.md
```

또는 자동 후보를 제시할 수 있다.

```text
A (conflict copy).md
```

하지만 자동 생성 이름은 사용자에게 변경할 기회를 제공하는 것이 좋다.

이후 Local Version은 새로운 Path의 CREATE Operation으로 제출한다.

---

# 12. Rename vs Modify

예:

```text
Server:

A.md → B.md

Local:

A.md modified
```

사용자 관점에서는:

> 이 문서는 다른 기기에서 `B.md`로 이름이 변경되었고, 이 기기에서는 이전 이름의 문서가 수정되었습니다.

기본 Resolution은 다음이 자연스럽다.

```text
B.md에 내 수정 내용 적용

서버의 B.md 유지하고 Local 버전을 별도 파일로 저장

Local 변경 포기
```

---

## 12.1 Rename된 문서에 Local Content 적용

Server의 현재 Path `B.md`를 기준으로 Local Content를 새로운 MODIFY Operation으로 제출한다.

```text
MODIFY B.md
```

즉 Local의 오래된 Path `A.md`를 다시 살리는 것이 아니다.

---

## 12.2 별도 문서로 유지

Local Content를 새로운 Path로 CREATE한다.

예:

```text
A-local.md
```

Server의 Rename 결과 `B.md`도 유지한다.

---

# 13. Rename vs Rename

예:

```text
Base:

A.md

Server:

A.md → B.md

Local:

A.md → C.md
```

사용자에게:

> 같은 문서의 이름이 두 기기에서 각각 다르게 변경되었습니다.

기본 Action:

```text
B.md 사용

C.md 사용

새 이름 지정
```

Local 이름을 선택하거나 새 이름을 선택하는 경우 현재 Server State에서 새로운 Rename Operation을 생성한다.

---

# 14. Rename Destination Conflict

다음 상황도 가능하다.

```text
Server:

A.md → B.md

Local:

B.md already exists
```

이 경우 Server Rename을 Local에 그대로 적용하면 기존 Local B.md를 덮어쓸 수 있다.

자동 overwrite하지 않는다.

사용자는:

```text
기존 Local B.md를 다른 이름으로 유지

Server B.md 사용

두 파일 이름 직접 지정
```

등의 선택을 할 수 있어야 한다.

---

# 15. Conflict 중 Server Update

Conflict가 unresolved 상태인 동안 Server가 또 변경될 수 있다.

예:

```text
Conflict 생성:

Server Revision 101 = B
Local = C

이후:

Server Revision 102 = D
```

Client는 Conflict를 제거하지 않는다.

대신:

```text
latestServerRevision = 102
latestServerState = D
```

로 갱신한다.

UI에도:

```text
서버 버전이 Conflict 발생 이후 다시 변경되었습니다.
```

를 표시할 수 있다.

---

# 16. Stale Resolution 방지

사용자가 Conflict 화면을 오랫동안 열어두고 Resolution을 선택할 수 있다.

예:

```text
화면에 표시된 Server = Revision 101

현재 실제 Server = Revision 105
```

이 경우 Revision 101을 Base로 Resolution Operation을 Commit하면 안 된다.

Resolution 실행 직전에 Server 최신 상태를 다시 확인한다.

---

## 16.1 Server 선택

사용자가 `서버 버전 사용`을 선택했다면 화면에 보이던 옛 버전이 아니라 기본적으로 **현재 Server Version**을 Local에 적용한다.

Server가 그 사이 변경되었다면 이를 사용자에게 알려줄 수 있다.

---

## 16.2 Local 또는 Manual Merge 선택

Resolution Operation은 최신 Server State를 Base로 생성한다.

단 사용자가 만든 Merge Result가 오래된 Server Version을 기반으로 작성되었다면 자동 Commit이 위험할 수 있다.

이 경우 다시 비교를 요청할 수 있다.

---

# 17. Resolution Commit Race

Resolution Action을 선택한 이후 Server가 다시 변경될 수 있다.

```text
1. Current Server = 105

2. User chooses Apply Local

3. New Operation base = 105

4. Other Client commits 106

5. Resolution Operation arrives
```

Server는 Base mismatch를 감지한다.

Resolution Operation을 강제로 적용하지 않는다.

Client는:

```text
RECONFLICTED
```

상태로 전환한다.

사용자가 선택한 Local 또는 Merge Result는 보존한다.

---

# 18. Resolution 결과 보존

Resolution Commit이 실패해도 사용자가 만든 결과를 잃어서는 안 된다.

특히 Manual Merge Result는 사용자가 직접 작성한 새로운 Content다.

따라서:

```text
Merged Result
```

는 Server Commit 성공 전까지 durable하게 보존한다.

---

# 19. Conflict Resolution과 Pending Operation

Conflict가 발생한 Original Pending Operation은 자동 Retry 대상에서 제외한다.

```text
Original Operation
     │
     ▼
CONFLICT
```

사용자가 Resolution을 선택하면 Original Operation을 수정해서 다시 보내는 대신 새로운 Operation을 생성한다.

```text
Original Conflict Operation
          │
          X
          │
   reused하지 않음

Resolution Decision
          │
          ▼
     New Operation
```

이 규칙을 통해 Operation Idempotency 의미를 유지한다.

---

# 20. Conflict 완료 조건

Conflict는 사용자가 버튼을 누른 순간 완료된 것이 아니다.

다음 조건이 모두 만족되어야 한다.

### Server를 선택한 경우

```text
현재 Server State가 Local에 안전하게 적용됨

Replica Index가 최신 Server State로 갱신됨
```

### Local / Merge 결과를 선택한 경우

```text
Resolution Operation이 Server에 Commit됨

Commit된 Revision을 Client가 정상적으로 통합함

Local State가 Commit Result와 일치함
```

그 후 Conflict를 `RESOLVED`로 처리한다.

---

# 21. Conflict History

Resolved Conflict를 즉시 완전히 삭제할 필요는 없다.

문제 진단과 사용자의 이해를 위해 최소한의 History를 유지할 수 있다.

예:

```text
2026-09-13 00:15

notes/a.md

Type:
MODIFY_MODIFY

Resolution:
APPLY_LOCAL

Result Revision:
1052
```

Conflict History에 파일 내용 전체를 영구 보관해야 하는 것은 아니다.

Content History는 별도 백업/버전 관리 정책과 구분한다.

---

# 22. Bulk Conflict

장기간 Offline Client가 복귀하면 많은 Conflict가 동시에 발생할 수 있다.

예:

```text
Conflicts (27)
```

사용자가 27개의 파일을 하나씩 처리해야 하는 UX는 좋지 않을 수 있다.

초기 버전에서는 최소한 Type별 또는 Folder별로 Conflict를 묶어 표시할 수 있다.

향후 다음과 같은 Bulk Action을 고려할 수 있다.

```text
선택한 파일 모두 Server 사용

선택한 파일 모두 Local 사용
```

하지만 Bulk Local Apply는 많은 Server 변경을 발생시키므로 반드시 사용자에게 영향 범위를 보여줘야 한다.

복잡한 Bulk Resolution은 MVP 필수 기능으로 하지 않는다.

---

# 23. Binary Conflict

Binary File은 Manual Merge를 제공하지 않는다.

예:

```text
image.png

Server Version
Local Version
```

기본 Action:

```text
서버 파일 사용

Local 파일로 교체

두 파일 모두 유지
```

두 파일 모두 유지 시 Local 파일에 새로운 이름을 지정하여 CREATE한다.

가능하다면 이미지 파일은 Preview를 제공할 수 있다.

---

# 24. Conflict와 Backup

Conflict Resolution은 Backup 기능이 아니다.

`서버 버전 사용`을 선택하여 Local Version이 더 이상 Active Vault에 존재하지 않게 되면 사용자가 이를 나중에 복구할 수 있다고 보장하지 않는다.

따라서 destructive Resolution 직전에 필요하다면:

```text
Local Conflict Recovery Copy
```

를 일정 기간 Client-local Storage에 유지할 수 있다.

구체적인 보존 기간은 구현 또는 운영 정책에서 정의한다.

---

# 25. Resolution UX 원칙

## 25.1 어느 쪽이 Server인지 명확히 표시한다

```text
Server Version

This Device Version
```

을 혼동하지 않도록 한다.

---

## 25.2 시간만으로 승자를 결정하지 않는다

다음과 같은 UI를 제공해서는 안 된다.

```text
Local이 더 최근이므로 Local 선택
```

Device Clock은 신뢰할 수 없고 최신 시각이 올바른 결과라는 보장도 없다.

Timestamp는 사용자 이해를 위한 참고 정보일 뿐 Resolution 기준이 아니다.

---

## 25.3 데이터가 사라지는 Action은 명확히 표시한다

예:

```text
서버 버전 사용
```

을 선택하면 현재 Local Version이 Active Vault에서 사라질 수 있다.

Action 결과를 사용자에게 설명해야 한다.

---

## 25.4 기본 Action은 데이터 보존 방향이어야 한다

가능하면 사용자의 실수 한 번으로 한쪽 Content가 복구 불가능하게 사라지지 않도록 한다.

특히 Create/Create 또는 Binary Conflict에서는 `두 파일 모두 유지`가 안전한 선택이 될 수 있다.

---

## 25.5 사용자가 전체 Revision 모델을 이해할 필요가 없어야 한다

Revision, Hash, Tombstone, Operation ID를 사용자가 알아야 Conflict를 해결할 수 있는 UX는 피한다.

---

# 26. Initial MVP Scope

초기 Conflict Resolution 기능은 다음 Type을 우선 지원한다.

```text
MODIFY vs MODIFY

DELETE vs MODIFY

MODIFY vs DELETE

CREATE vs CREATE

RENAME vs MODIFY
```

지원 Action:

```text
Use Server

Apply Local

Manual Merge for Markdown

Keep Deleted

Restore Local

Keep Both
```

다음 기능은 후순위로 둘 수 있다.

```text
Automatic 3-way Merge

Semantic Markdown Merge

Bulk Smart Resolution

Automatic Rename Conflict Resolution
```

---

# 27. Conflict Resolution Invariants

## Invariant 1 — No Silent Data Loss

Conflict Resolution 과정에서 Server 또는 Local Content를 명시적인 판단 없이 삭제하거나 덮어쓰지 않는다.

---

## Invariant 2 — Server Remains Authoritative

Conflict 상태에서도 Server Current State는 전체 시스템의 authoritative state다.

---

## Invariant 3 — Local Resolution Creates a New Change

사용자가 Local Version을 선택하면 과거의 실패한 Operation을 강제로 적용하지 않고 현재 Server State를 Base로 새로운 Operation을 생성한다.

---

## Invariant 4 — Manual Merge Creates a New Change

Merged Result 역시 현재 Server State를 Base로 하는 새로운 Operation이다.

---

## Invariant 5 — Resolution Uses Current Server State

Conflict 발생 시점의 Server Version이 아니라 Resolution 시점의 최신 Server State를 기준으로 Commit한다.

---

## Invariant 6 — Resolution Can Reconflict

Resolution Operation을 Commit하기 전에 Server State가 다시 변경되었다면 overwrite하지 않고 다시 Conflict로 처리한다.

---

## Invariant 7 — User-created Resolution Content Is Durable

Manual Merge 또는 사용자가 선택한 Local Result는 Server Commit 실패나 Client Crash 때문에 유실되어서는 안 된다.

---

## Invariant 8 — Conflict Resolution Is Path-isolated

하나의 Conflict를 해결하는 동안 다른 정상 Path의 동기화를 중단하지 않는다.

---

## Invariant 9 — Resolution Is Complete Only After Convergence

사용자 Action 선택만으로 Conflict를 완료 처리하지 않는다.

최종 Server Commit 및 Local Integration까지 완료되어야 한다.

---

## Invariant 10 — Timestamp Is Informational

Device Timestamp만으로 Conflict의 승자를 자동 결정하지 않는다.

---

# 28. 전체 Resolution Flow

```text
                  Conflict Detected
                         │
                         ▼
                 Conflict Store
                         │
                         ▼
                  Conflict Center
                         │
                         ▼
                 User opens Detail
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
         Use Server  Apply Local  Manual Merge
              │          │          │
              │          └────┬─────┘
              │               │
              │               ▼
              │        Refresh Server State
              │               │
              │               ▼
              │        New Resolution Operation
              │               │
              │               ▼
              │           Server Commit
              │               │
              │         ┌─────┴─────┐
              │         │           │
              │         ▼           ▼
              │      Success    Base Changed
              │         │           │
              │         │           ▼
              │         │      RECONFLICTED
              │         │
              ▼         ▼
         Local Integrate
              │
              ▼
         Replica Update
              │
              ▼
        Conflict RESOLVED
```

Conflict Resolution의 핵심은:

> **Conflict가 발생했을 당시의 두 버전 중 하나를 단순히 고르는 것이 아니라, 사용자가 현재 Server State를 기준으로 어떤 상태를 다음 authoritative state로 만들 것인지 명시적으로 결정하는 것**

이다.
