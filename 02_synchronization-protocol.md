# VaultDatum Synchronization Protocol

## 1. 문서 목적

이 문서는 VaultDatum Server와 VaultDatum for Obsidian 사이의 동기화 동작을 정의한다.

구체적인 HTTP API나 데이터 직렬화 형식보다 상위 수준에서 다음을 정의한다.

* 서버 변경의 순서를 어떻게 식별하는가
* 클라이언트가 서버의 어느 상태까지 알고 있는지 어떻게 표현하는가
* 로컬 변경을 언제 서버에 제출할 수 있는가
* 서버 변경을 언제 로컬 Vault에 적용할 수 있는가
* 충돌을 어떻게 판단하는가
* Offline Client가 어떻게 복구되는가
* Delete/Rename/Move를 어떻게 처리하는가
* Retry와 Crash 이후에도 어떻게 정합성을 유지하는가

이 문서의 규칙은 이후 API와 데이터 모델이 반드시 만족해야 하는 프로토콜 요구사항이다.

상위 컴포넌트 책임은 [01 Architecture Overview](./01_architecture-overview.md)을, 용어와 상태 필드의 정의는 [06 Data Model](./06_data-model.md)을, HTTP·notification wire contract는 [07 API Specification](./07_api-specification.md)을 따른다. 이 문서는 그 상태들이 시간에 따라 전이하는 규칙만 정의한다.

---

# 2. 사용하는 상태 용어

Global Revision, Server Revision, Server Cursor, Replica Index, Pending Operation, Base State, Operation ID의 필드와 의미는 [06 Data Model](./06_data-model.md)의 공통·Client value type을 단일 기준으로 사용한다.

이 문서에서 이 용어들은 다음 동작 규칙으로 사용된다.

* Global Revision은 서버에서 확정된 Change의 전체 순서다.
* Cursor는 처리 또는 보존한 Change Stream의 연속 범위이며, 단순한 local write 완료 지점이 아니다.
* Replica Index는 마지막으로 처리한 path별 Server State로서 local change와 concurrent change를 판정한다.
* Pending Operation은 Commit 전까지 durable하게 유지하고 Base State와 Operation ID로 conflict·retry를 판정한다.

---

# 3. Client Sync State

VaultDatum Client는 개념적으로 다음 상태를 유지한다.

```text
Client Sync State

├── serverCursor
│
├── Replica Index
│   ├── a.md → revision/hash/existence
│   ├── b.md → revision/hash/existence
│   └── ...
│
├── Pending Operations
│
└── Conflicts
```

Local Vault 자체와 Sync State는 구분한다.

```text
Local Vault
    =
사용자가 실제로 보는 파일

Sync State
    =
VaultDatum가 Local Vault를 서버와 수렴시키기 위해
기억하는 메타데이터
```

`.obsidian/`은 Replica Index에도 포함하지 않는다.

---

# 4. Server Change Model

서버에 확정된 모든 변경은 하나의 Change로 표현할 수 있어야 한다.

최소한 다음 정보를 논리적으로 가진다.

```text
revision
operation type
affected path
previous path        # rename/move인 경우
result state
actor/client
operation id
```

파일 내용 전체를 Change Journal에 반드시 저장해야 한다는 의미는 아니다.

Change Journal은 최소한 클라이언트가 변경 사실을 이해하고 필요한 현재 데이터를 서버에서 조회할 수 있을 정도의 정보를 제공해야 한다.

---

# 5. 기본 Sync Cycle

VaultDatum Client의 동기화는 하나의 일관된 Sync Cycle로 생각한다.

```text
        ┌──────────────────┐
        │ Connect / Wakeup │
        └────────┬─────────┘
                 │
                 ▼
        Pull Server Changes
                 │
                 ▼
        Integrate / Conflict
                 │
                 ▼
        Push Local Pending
                 │
                 ▼
        Pull Again
                 │
                 ▼
       Check Convergence
```

기본적으로 **서버 변경 확인을 먼저 수행한 뒤 로컬 변경을 Push**한다.

이렇게 하면 오래된 클라이언트가 서버 상태를 확인하지 않은 채 변경을 제출하는 상황을 줄일 수 있다.

하지만 이것만으로 충돌이 완전히 방지되는 것은 아니다.

Pull 이후 Push 직전에도 다른 클라이언트가 서버를 변경할 수 있기 때문에 서버는 Push 시점에 반드시 Base State를 다시 검증해야 한다.

---

# 6. Pull Protocol

## 6.1 Incremental Pull

클라이언트가 다음 상태라고 하자.

```text
serverCursor = 1000
```

서버는:

```text
serverRevision = 1010
```

이다.

클라이언트는 의미상 다음 정보를 요청한다.

```text
Revision 1000 이후의 변경을 알려줘.
```

서버는:

```text
1001 MODIFY A.md
1002 CREATE B.md
1003 DELETE C.md
...
1010 MODIFY D.md
```

를 반환한다.

클라이언트는 Revision 순서대로 변경을 처리한다.

---

## 6.2 Remote Change 적용

해당 파일에 로컬 Pending Operation이 없고 Local State가 Replica Index와 일치한다면 서버 변경을 Local Vault에 적용한다.

예:

```text
Replica Index:

A.md @ revision 1000
hash = AAA

Local:

A.md
hash = AAA

Server Change:

1001 MODIFY A.md
hash = BBB
```

로컬에서 별도 변경이 없으므로 안전하게:

```text
Local A.md = BBB
Replica Index = revision 1001 / hash BBB
```

로 갱신한다.

---

# 7. Local Change Protocol

사용자가 Local Vault의 파일을 수정한다.

```text
Replica Index:

A.md @ revision 1000
hash = AAA
```

사용자가 내용을 수정하여:

```text
Local:

A.md
hash = LOCAL
```

이 되면 VaultDatum Client는 Pending Operation을 생성한다.

```text
MODIFY A.md

baseRevision = 1000
baseHash = AAA

contentHash = LOCAL
```

사용자가 같은 파일을 여러 번 수정하더라도 서버에 아직 제출되지 않았다면 안전한 범위에서 하나의 최종 변경으로 합칠 수 있다.

단, 최초 Base State는 유지해야 한다.

예:

```text
Server base
AAA

Local edit 1
BBB

Local edit 2
CCC
```

최종 Pending은:

```text
baseHash = AAA
contentHash = CCC
```

가 될 수 있다.

---

# 8. Push Protocol

클라이언트가 Pending Operation을 서버에 제출하면 서버는 먼저 Base State를 검증한다.

## 8.1 Base State 일치

```text
Client Base:

A.md revision 1000
hash AAA

Server Current:

A.md revision 1000
hash AAA
```

서버 상태가 일치하므로 변경을 적용할 수 있다.

```text
Server:

Revision 1001
MODIFY A.md
```

서버는 authoritative Vault와 Sync State에 변경을 확정한 후에만 성공을 반환한다.

클라이언트는 성공이 확인된 이후 해당 Pending Operation을 완료 처리할 수 있다.

---

## 8.2 Base State 불일치

다음 상태에서는 자동으로 덮어쓰지 않는다.

```text
Client Base:

A.md revision 1000
hash AAA

Server Current:

A.md revision 1005
hash SERVER
```

클라이언트의 변경은 이미 오래된 상태를 기반으로 만들어졌다.

따라서 서버는 변경을 거부하고 Conflict 정보를 반환한다.

```text
Client Change
      │
      X
      │
Server

→ Conflict
```

클라이언트의 로컬 내용은 그대로 보존한다.

---

# 9. Pull 중 Local Pending이 존재하는 경우

이 경우가 VaultDatum에서 가장 중요한 상황 중 하나다.

클라이언트:

```text
A.md

Replica = revision 1000
Local   = modified
Pending = MODIFY A.md
```

서버 변경:

```text
1001 MODIFY A.md
```

이 서버 변경을 그대로 Local Vault에 쓰면 사용자의 미전송 변경을 잃게 된다.

따라서:

```text
Remote change affects pending path
              │
              ▼
          Conflict
```

로 처리한다.

다음 데이터를 최소한 보존한다.

```text
Local version
Server version/state
Base information
```

서버 버전이 Local Vault를 조용히 덮어써서는 안 된다.

---

# 10. Conflict와 Server Cursor

Conflict가 발생했다고 해서 전체 서버 동기화를 멈출 필요는 없다.

예:

```text
1001 MODIFY A.md   ← Conflict
1002 MODIFY B.md
1003 CREATE C.md
```

A.md가 충돌했다고 B.md와 C.md까지 영원히 동기화되지 않는 것은 바람직하지 않다.

따라서 클라이언트가 Revision 1001을 Conflict State로 안전하게 기록했다면 이를 **처리된 Revision**으로 간주할 수 있다.

이후:

```text
1002 MODIFY B.md
1003 CREATE C.md
```

를 계속 처리한다.

최종적으로:

```text
serverCursor = 1003

Conflict:
A.md
```

가 가능하다.

따라서:

```text
Synced
```

와

```text
Caught Up
```

은 완전히 같은 의미가 아니다.

서버 변경 이력은 모두 처리했지만 Conflict가 남아 있을 수 있다.

이 경우 UI 상태는 `Conflict`로 표시한다.

---

# 11. Push 이후 Cursor 처리

Push 성공 응답으로 새로운 Revision을 받았다고 해서 클라이언트의 `serverCursor`를 그 Revision으로 바로 이동시켜서는 안 된다.

예:

```text
Client Cursor = 1000

다른 클라이언트:
1001
1002

내 Push:
1003
```

내 변경이 Revision 1003으로 확정되었다고 해서 클라이언트가 1001, 1002를 이미 처리한 것은 아니다.

따라서:

```text
Push Revision
    ≠
Server Cursor
```

이다.

Push 성공은 해당 Pending Operation의 서버 확정을 의미한다.

Global Cursor는 누락된 이전 Revision을 정상적으로 처리한 후 순차적으로 진행한다.

---

# 12. Own Change Pull

클라이언트가 서버에 성공적으로 Push한 변경도 Change Journal에는 존재한다.

따라서 이후 Pull에서 자신의 변경을 다시 볼 수 있다.

Change가 자신의 `operationId`와 동일하다는 것을 확인할 수 있다면 Local Vault에 같은 내용을 다시 쓸 필요는 없다.

대신:

```text
Replica Index update
Server Cursor update
```

만 수행할 수 있다.

이를 통해 불필요한 파일 write와 변경 이벤트 loop를 줄일 수 있다.

---

# 13. Delete Protocol

## 13.1 일반 Delete

클라이언트의 기준 상태:

```text
A.md

revision = 1000
hash = AAA
```

사용자가 파일을 삭제한다.

Pending:

```text
DELETE A.md

baseRevision = 1000
baseHash = AAA
```

서버 상태가 Base와 일치하면:

```text
1001 DELETE A.md
```

로 확정한다.

---

## 13.2 Offline Client의 오래된 파일

Client B가 Revision 1000에서 오프라인이 되었다.

Client A가:

```text
1001 DELETE A.md
```

를 발생시킨다.

Client B가 다시 연결되면 Delete Change를 Pull하여 자신의 오래된 `A.md`를 제거한다.

---

## 13.3 Delete vs Offline Modify

Client B가 오프라인 동안 A.md를 수정했다고 하자.

```text
Client B:

Base A.md @ 1000
Local modified
```

서버에서는:

```text
1001 DELETE A.md
```

가 발생했다.

Client B가 Pull하면:

```text
Remote DELETE
+
Local Pending MODIFY
```

가 발견된다.

이 경우 A.md를 삭제하거나 서버에 다시 올려서는 안 된다.

```text
DELETE vs MODIFY
       │
       ▼
    Conflict
```

Local Content는 보존한다.

서버 상태는 `deleted`로 유지한다.

따라서 오래된 Offline Client 때문에 파일이 자동으로 부활하지 않는다.

---

# 14. Create Conflict

클라이언트가 오프라인에서 새로운 파일을 생성했다.

```text
CREATE A.md
```

그러나 그 사이 다른 클라이언트도 서버에 동일 경로를 생성했다.

```text
Server:
A.md exists
```

이 경우 이름이 같다는 이유만으로 어느 한쪽을 덮어쓰지 않는다.

```text
CREATE vs CREATE
       │
       ▼
    Conflict
```

로 처리한다.

---

# 15. Rename / Move Protocol

Rename과 Move는 가능한 경우 독립적인 Change로 표현한다.

```text
RENAME

from: notes/a.md
to: network/a.md
```

또는:

```text
MOVE

from: scratch/a.md
to: archive/a.md
```

Rename/Move는 최소 두 경로에 영향을 준다고 본다.

```text
source path
destination path
```

따라서 두 경로 중 어느 하나에 충돌 가능성이 있다면 자동 적용하지 않는다.

예:

```text
Server:

A.md -> B.md

Offline Client:

A.md modified
```

이 경우:

```text
RENAME vs MODIFY
       │
       ▼
    Conflict
```

로 처리한다.

VaultDatum는 Rename Conflict를 자동 해결하지 않는다.

---

# 16. Reconnect Protocol

Offline Client가 다시 연결되면 다음 순서로 동기화를 수행한다.

```text
1. Server 연결

2. 현재 serverRevision 확인

3. serverCursor 이후 변경 Pull

4. 서버 변경 통합
   ├── safe → Local Apply
   └── unsafe → Conflict

5. Push 가능한 Pending Operation 전송

6. Push 도중 새로 발생한 Server Change Pull

7. 반복

8. Convergence 확인
```

이 과정에서 네트워크가 다시 끊겨도 다음 연결에서 같은 절차를 재개한다.

---

# 17. Retry Protocol

네트워크 오류 때문에 서버가 요청을 처리했는지 클라이언트가 알 수 없는 상황이 발생할 수 있다.

```text
Client
   │
   │ Operation X
   ▼
Server
   │
   │ commit
   ▼
Response

   X network lost
```

클라이언트는 Operation X를 완료 처리하지 않는다.

다음 연결에서 동일한 `operationId`로 Retry한다.

서버는:

```text
operationId already committed
```

임을 확인하고 기존 결과를 반환한다.

새로운 Revision을 생성해서는 안 된다.

---

# 18. Client Crash Recovery

Pending Operation과 Sync State는 Client Process의 메모리에만 존재해서는 안 된다.

다음 상황을 가정한다.

```text
Local file modified
       │
Pending created
       │
       X
Obsidian killed
```

다음 실행에서 VaultDatum Client는:

```text
Pending Operation
Replica Index
Server Cursor
Conflict State
```

를 복구할 수 있어야 한다.

그 후 일반 Reconnect Protocol을 수행한다.

---

# 19. Server Crash Recovery

서버가 클라이언트에게 성공을 반환했다면 해당 변경은 재시작 이후에도 존재해야 한다.

즉:

```text
Success Response
      ⇒
Durable Commit
```

이어야 한다.

Vault Content와 Sync State 중 하나만 반영된 상태가 영구적으로 남아서는 안 된다.

실제 filesystem과 metadata store 사이의 원자성 및 Crash Recovery 방식은 Server Architecture에서 별도로 정의한다.

---

# 20. Initial Synchronization

VaultDatum는 서버를 absolute source of truth로 사용한다.

따라서 새로운 클라이언트의 기본 초기화 방향은:

```text
VaultDatum Server
      │
      ▼
New Client
```

이다.

새로운 클라이언트가 처음 연결되면 서버의 현재 synchronized content를 기준으로 Local Vault를 구성한다.

`.obsidian/`은 영향을 받지 않는다.

---

## 20.1 Existing Local Vault

이미 파일이 존재하는 Local Vault를 등록할 때도 서버를 먼저 동기화한다. 다만 서버에 전혀 알려지지 않은 Local 경로는 안전한 `CREATE` 후보로 분류하여 자동으로 업로드할 수 있다. 이는 양쪽 내용을 임의로 합치는 것이 아니라, Server Manifest를 기준으로 각 경로를 명시적으로 분류하는 **안전한 초기 Bootstrap**이다.

초기 Bootstrap은 항상 다음 순서로 실행한다.

```text
0. 기존 Replica가 있는 Local drift만 Pending으로 복구
   └── Server UNKNOWN 경로는 CREATE 후보로 만들지 않음

1. 기존 Pending이 있으면 Change Journal로 이미 확정된 own operation을 먼저 복구

2. Server Manifest snapshot 생성 및 검증

3. Manifest를 Local Vault와 대조하여 적용 또는 Conflict 기록

4. 최신 Local Vault를 다시 scan

5. Server가 UNKNOWN인 경로만 durable CREATE 후보로 기록

6. CREATE 후보 Push

7. Push 이후 Change Journal을 Pull하여 수렴 확인
```

초기 scan이나 Local Event만을 근거로 Server `UNKNOWN` CREATE를 보내서는 안 된다. Server Manifest를 성공적으로 통합할 수 없으면 Bootstrap은 완료되지 않으며, 기존 Replica와 비교해 복구한 Pending 외의 Local Content는 upload하지 않는다.

이미 동기화 이력이 있는 Client는 manifest 전에 Replica Index와 일치하지 않는 Local 변경을 Pending으로 복구할 수 있다. 이는 기존 Server state를 Base로 하는 offline 변경의 복구이며, 기존 Local Vault를 import하는 동작이 아니다. 이미 저장된 Pending이 있다면 먼저 Change Journal을 처리해 동일 Operation ID의 Server commit을 복구한다. 그래야 응답을 잃은 own operation이 최신 Manifest에서 단순한 충돌로 오인되지 않는다. 이 사전 recovery 뒤에도 fresh Manifest는 반드시 통합한다. Manifest snapshot의 해당 path가 Pending의 Base State와 동일하면 Server가 변경되지 않은 것이므로 Pending을 유지한다. Base State가 다르면 Conflict로 기록한다.

### 경로별 분류

Bootstrap snapshot에서의 Server State와 현재 Local State를 다음처럼 처리한다.

| Server State | Local State | 결과 |
| --- | --- | --- |
| PRESENT | 없음 | Server 내용을 Local에 적용 |
| PRESENT file | 같은 hash의 file | Replica로 기록, upload하지 않음 |
| PRESENT directory | directory 존재 | Replica로 기록, upload하지 않음 |
| PRESENT | 내용·타입이 다름 | Conflict. 어느 쪽도 덮어쓰지 않음 |
| DELETED | 없음 | 삭제 Replica로 기록 |
| DELETED | file 또는 directory 존재 | Conflict. 오래된 Local 항목을 자동 복원하지 않음 |
| UNKNOWN | file 존재 | `CREATE` 후보로 durable queue |
| UNKNOWN | 빈 directory 존재 | directory `CREATE` 후보로 durable queue |

비어 있지 않은 Local directory는 그 자체의 directory CREATE가 아니라 포함된 파일과 빈 하위 directory의 후보를 통해 서버 구조에 반영된다. `.obsidian/`, 지원하지 않는 경로, 동기화 크기 제한을 넘는 파일은 후보에서 제외하고 사용자에게 상태를 보여준다.

`UNKNOWN`은 해당 경로가 Server에 존재한 기록이 없다는 의미이고, `DELETED`와 다르다. 따라서 Server가 삭제한 경로를 Local에 가지고 있다는 이유만으로 초기 Bootstrap에서 되살릴 수 없다.

### Bootstrap race와 재시도

Manifest snapshot 이후 다른 Client가 같은 `UNKNOWN` 경로를 생성할 수 있다. Initial CREATE는 `UNKNOWN` Base Condition을 사용하고 Server가 현재 Path State를 다시 검증한다. 이 검증이 실패하면 Client는 Server 내용을 덮어쓰지 않고 Create Conflict로 기록한다.

Bootstrap 도중 Client가 종료되거나 네트워크가 끊겨도, 이미 기록한 Pending Operation은 유지한다. Bootstrap 완료 metadata가 durable하게 기록되기 전에는 다음 실행에서 새 Manifest로 다시 분류한다. 재실행은 기존 Pending Operation을 재사용하거나 아직 기록되지 않은 후보만 추가해야 하며, 중복된 CREATE를 만들면 안 된다.

Bootstrap 완료는 다음 조건이 모두 만족된 뒤에만 기록한다.

```text
fresh Server Manifest integrated
AND
every in-scope Local path classified as
  Replica / Conflict / durable Pending / explicitly skipped
```

Pending CREATE가 아직 Server에 commit되지 않았더라도 Bootstrap 분류 자체는 완료될 수 있다. Commit과 최종 수렴은 이후 일반 Sync Cycle이 담당한다.

---

# 21. Full Reconciliation

일반적인 동기화는 Global Revision을 이용한 Incremental Pull을 사용한다.

하지만 다음 상황에서는 전체 비교가 필요할 수 있다.

* Client Sync State 손상
* Change Journal 보존 범위를 벗어난 장기 Offline
* 서버 복구 이후 정합성 검사
* 사용자가 명시적인 Reconcile을 요청
* 예상하지 못한 filesystem 변경 발견

이 경우 Server Manifest와 Replica Index 및 Local Vault를 비교한다.

---

## 21.1 Full Reconciliation 판단

예를 들어:

```text
Server:
A.md = SERVER

Replica Index:
A.md = BASE

Local:
A.md = BASE
```

Local은 변경되지 않았으므로 Server 상태를 적용할 수 있다.

반면:

```text
Server:
A.md = SERVER

Replica Index:
A.md = BASE

Local:
A.md = LOCAL
```

이라면 양쪽 모두 Base 이후 변경되었다.

```text
       BASE
      /    \
 SERVER    LOCAL
```

따라서 Conflict다.

---

## 21.2 Server에 없는 Local File

Replica Index를 이용하면 다음 두 상황을 구분할 수 있다.

### 서버에 존재했던 파일

```text
Replica Index:
A.md existed

Server:
A.md absent
```

서버에서 삭제된 상태일 가능성이 있다.

Local이 Base에서 변경되지 않았다면 서버의 삭제 상태로 수렴시킨다.

Local이 수정되었다면 Delete vs Modify Conflict다.

### 서버에 존재한 기록이 없는 파일

```text
Replica Index:
A.md unknown

Server:
A.md absent

Local:
A.md exists
```

이는 새로운 Local Create 후보로 판단할 수 있다.

따라서 Full Reconciliation에서도 단순히:

```text
Server에 없으니 Local 삭제
```

와 같은 규칙을 사용해서는 안 된다.

---

# 22. Notification Protocol

실시간 Notification은 다음 정도의 정보만 전달해도 된다.

```text
Server state changed.

latestRevision = N
```

클라이언트는 이 메시지를 받은 뒤 실제 변경 내용을 Sync Protocol을 통해 조회한다.

Notification 자체에 전달된 데이터를 authoritative하게 적용하지 않는다.

```text
Notification
    │
    ▼
Trigger Reconciliation
```

이 구조를 통해 Notification 유실, 중복, 순서 변경이 발생하더라도 정합성에는 영향을 주지 않는다.

---

# 23. Sync State 판정

## Synced

```text
serverCursor == current serverRevision
Pending = 0
Conflict = 0
```

## Pending

```text
Pending > 0
Conflict = 0
```

## Catching Up

```text
serverCursor < serverRevision
```

## Conflict

```text
Conflict > 0
```

## Offline

서버에 접근할 수 없지만 Local Vault 사용은 가능하다.

## Error

자동 Retry 또는 Reconciliation으로 정상 진행할 수 없는 오류가 존재한다.

---

# 24. Protocol Invariants

## Protocol Invariant 1 — Ordered Server History

서버에서 확정된 변경은 Global Revision을 통해 일관된 순서를 가져야 한다.

## Protocol Invariant 2 — Base Validation

서버는 Client가 제출한 변경의 Base State가 현재 서버 상태와 호환되는지 검증해야 한다.

## Protocol Invariant 3 — Durable Pending

서버에서 확정되지 않은 Local Change는 Client Crash 이후에도 복구할 수 있어야 한다.

## Protocol Invariant 4 — No Blind Overwrite

Remote Change와 Local Pending이 동일한 상태에 영향을 미칠 경우 명시적인 안전성 판단 없이 Local 또는 Remote 데이터를 덮어써서는 안 된다.

## Protocol Invariant 5 — Cursor Means Processed

`serverCursor=N`은 N 이하의 모든 Server Change가 정상 적용되었거나 명시적인 Conflict 상태로 보존되었음을 의미한다.

## Protocol Invariant 6 — Cursor Is Contiguous

클라이언트는 중간 Revision을 알지 못한 채 Cursor를 건너뛸 수 없다.

## Protocol Invariant 7 — Push Does Not Advance Cursor

Push 성공으로 받은 Revision만으로 Global Server Cursor를 앞으로 이동시키지 않는다.

## Protocol Invariant 8 — Delete Cannot Resurrect Implicitly

서버에서 확정된 Delete는 오래된 클라이언트의 존재만으로 취소될 수 없다.

## Protocol Invariant 9 — Retry Is Idempotent

동일 Operation의 Retry는 새로운 논리적 변경을 생성해서는 안 된다.

## Protocol Invariant 10 — Notification Is Advisory

실시간 Notification은 정합성 판정의 근거로 사용하지 않는다.

## Protocol Invariant 11 — Server Commit Before Success

서버는 Authoritative State가 durable하게 확정되기 전에 Client에게 성공을 반환해서는 안 된다.

## Protocol Invariant 12 — Local Configuration Isolation

`.obsidian/`은 Sync Protocol의 모든 Scan, Manifest, Change, Replica Index 및 Reconciliation에서 제외한다.

## Protocol Invariant 13 — Initial Bootstrap Is Server-First

기존 Local Vault를 등록할 때도 Server Manifest를 먼저 통합한다. Server가 `UNKNOWN`으로 확인한 경로만 CREATE 후보가 될 수 있으며, `DELETED` 경로는 Local 존재만으로 복원되지 않는다.

---

# 25. Synchronization Loop

전체 프로토콜을 단순화하면 다음과 같다.

```text
                 VaultDatum Client

                      │
                      │
             Local Vault Change
                      │
                      ▼
               Pending Queue
                      │
                      │
       ┌──────────────┴──────────────┐
       │                             │
       │        Sync Cycle           │
       │                             │
       │   1. Pull Changes           │
       │                             │
       │   2. Integrate              │
       │      ├─ Apply               │
       │      └─ Conflict            │
       │                             │
       │   3. Push Pending           │
       │      ├─ Commit              │
       │      └─ Conflict            │
       │                             │
       │   4. Pull Again             │
       │                             │
       │   5. Check Convergence      │
       │                             │
       └──────────────┬──────────────┘
                      │
                      ▼
                 Local Replica


                 VaultDatum Server

                      │
              Authoritative Vault
                      +
                Change Journal
```

VaultDatum 동기화의 핵심은 **파일을 가능한 빨리 복사하는 것**이 아니라,

> **클라이언트가 어떤 서버 상태를 기준으로 로컬 변경을 만들었는지를 보존하고, 그 기준이 아직 유효한지 확인하면서 모든 Replica를 Authoritative State에 수렴시키는 것**

이다.
