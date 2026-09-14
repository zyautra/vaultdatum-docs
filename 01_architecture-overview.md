# VaultDatum Architecture

## 1. 문서 목적

이 문서는 VaultDatum 제품 사양을 구현하기 위한 상위 수준의 시스템 아키텍처를 정의한다.

제품 사양이 **VaultDatum가 무엇을 보장해야 하는가**를 정의한다면, 본 문서는 다음 질문에 답한다.

* VaultDatum의 authoritative state는 무엇인가?
* 서버와 클라이언트는 각각 어떤 상태를 소유하는가?
* 변경은 어떤 흐름으로 확정되는가?
* 오프라인 클라이언트는 어떻게 다시 서버 상태로 수렴하는가?
* 실시간 이벤트를 놓쳐도 어떻게 정합성을 회복하는가?
* 삭제, 충돌, 재시도 등의 상태를 어떻게 모델링할 것인가?

본 문서에서는 아직 다음 사항을 확정하지 않는다.

* 구체적인 REST API 경로
* DB 제품 및 스키마
* 프로그래밍 언어 및 Framework
* HTTP/WebSocket 라이브러리
* 배포 방식
* VPN 및 TLS의 구체적인 구성

이들은 상위 아키텍처가 확정된 이후 별도 문서에서 정의한다.

제품 요구사항과 전역 불변조건은 [00 Product Specification](./00_product-specification.md)을 기준으로 한다. 본 문서는 이를 반복하지 않고, 그 요구사항이 Server·Client 상태와 수렴 구조에 만드는 결과만 정의한다. 동작 순서는 [02 Synchronization Protocol](./02_synchronization-protocol.md), 상태의 정확한 의미는 [06 Data Model](./06_data-model.md)을 따른다.

---

# 2. 제품 요구사항이 만드는 아키텍처 제약

Server authority, local-first, eventual convergence, notification independence, no-silent-overwrite는 [00 Product Specification](./00_product-specification.md)의 제품 요구사항이다. 이 요구사항은 다음 아키텍처 제약으로만 여기에서 사용한다.

* Client는 서로 직접 동기화하거나 상태를 합의하지 않고 Server를 통해서만 수렴한다.
* Local Vault와 Server authoritative state 사이의 차이는 Client의 durable sync state로 관리한다.
* 알림은 동기화를 깨우는 최적화이며, correctness는 reconciliation이 담당한다.
* 안전하게 판정할 수 없는 동시 변경은 자동 결론 대신 Conflict로 보존한다.

---

# 3. System Context

VaultDatum는 크게 두 구성요소로 이루어진다.

```text
┌─────────────────────────────────────┐
│            VaultDatum Server            │
│                                     │
│   Authoritative Vault               │
│   Sync State                        │
│   Change History                    │
│   Reconciliation                    │
│                                     │
└──────────────────┬──────────────────┘
                   │
             VaultDatum Protocol
                   │
       ┌───────────┼───────────┐
       │           │           │
       ▼           ▼           ▼
   Desktop       Mobile      Laptop

 VaultDatum for    VaultDatum for   VaultDatum for
 Obsidian      Obsidian     Obsidian

 Local Vault   Local Vault  Local Vault
```

---

## 3.1 VaultDatum Server

VaultDatum Server는 다음 책임을 가진다.

### Authoritative State 관리

전체 시스템에서 확정된 Vault 상태를 보유한다.

### 변경 확정

클라이언트에서 전달된 변경이 현재 서버 상태에 적용 가능한지 판단하고, 성공적으로 적용된 변경을 새로운 authoritative state로 확정한다.

### 변경 이력 관리

오프라인이었던 클라이언트가 과거 시점 이후 발생한 변경을 다시 확인할 수 있도록 충분한 변경 정보를 유지한다.

### 충돌 감지

클라이언트가 변경을 시작한 기준 상태와 현재 서버 상태가 달라졌는지를 판단한다.

### Reconciliation 지원

클라이언트가 자신의 상태와 서버 상태의 차이를 확인하고 다시 수렴할 수 있도록 필요한 정보를 제공한다.

---

## 3.2 VaultDatum for Obsidian

VaultDatum for Obsidian은 다음 책임을 가진다.

### Local Vault 감시

Obsidian에서 발생하는 파일 변경을 감지한다.

### Pending Change 보존

서버에 아직 확정되지 않은 로컬 변경을 안전하게 보존한다.

### Push

Pending Change를 서버에 전달한다.

### Pull

서버에서 확정된 변경을 Local Vault에 적용한다.

### Reconciliation

현재 Local State와 VaultDatum Server의 상태를 비교하여 누락된 변경을 복구한다.

### Conflict Handling

서버가 충돌로 판단한 변경을 사용자가 확인할 수 있는 상태로 보존한다.

---

## 3.3 Client-local State

다음 데이터는 VaultDatum의 동기화 대상이 아니다.

```text
.obsidian/
```

이는 각 장치에 독립적으로 존재한다.

따라서 하나의 클라이언트 Vault는 개념적으로 다음 두 영역으로 나뉜다.

```text
Local Vault

├── Synchronized Content
│
│   ├── notes/
│   ├── projects/
│   ├── attachments/
│   └── ...
│
└── Client-local State
    │
    └── .obsidian/
```

VaultDatum의 convergence는 **Synchronized Content 영역에 대해서만** 정의한다.

---

# 4. Authoritative State Model

## 4.1 Vault만으로는 Authoritative State가 완성되지 않는다

VaultDatum Server의 실제 파일시스템이 다음 상태라고 하자.

```text
Vault/

├── A.md
└── B.md
```

현재 상태만 보면 `A.md`, `B.md`가 존재한다는 사실은 알 수 있다.

하지만 다음 사실은 알 수 없다.

```text
C.md가 이전에는 존재했지만 삭제되었다.
```

오랫동안 오프라인이었던 클라이언트가 아직 `C.md`를 가지고 있다면 서버의 현재 파일 목록만으로는 다음 두 상황을 구분하기 어렵다.

```text
1. C.md는 서버에 아직 업로드되지 않은 새로운 파일

2. C.md는 서버에서 이미 삭제된 오래된 파일
```

따라서 VaultDatum에서 authoritative state는 Vault Filesystem만으로 구성되지 않는다.

---

## 4.2 VaultDatum Authoritative State

VaultDatum의 authoritative state는 크게 두 부분으로 구성한다.

```text
         VaultDatum Authoritative State

        ┌──────────────────────────┐
        │                          │
        │    Vault State           │
        │                          │
        │ Markdown                 │
        │ Attachments              │
        │ Directory Structure      │
        │                          │
        └────────────┬─────────────┘
                     │
                     │
        ┌────────────▼─────────────┐
        │                          │
        │    Sync State            │
        │                          │
        │ Revision                 │
        │ Change History           │
        │ Deletion State           │
        │ Operation State          │
        │                          │
        └──────────────────────────┘
```

두 상태는 역할이 다르지만 모두 VaultDatum Server의 정합성을 유지하는 데 필요하다.

---

## 4.3 Vault State

Vault State는 사용자가 실제로 사용하는 콘텐츠의 현재 상태를 의미한다.

포함:

```text
Markdown
Directory
Attachment
Filename
Path
```

제외:

```text
.obsidian/
```

Vault State의 실제 파일 내용은 VaultDatum Server의 Vault가 원본이다.

즉 Markdown 원문을 별도의 데이터 모델로 만들어 그쪽을 원본으로 취급하지 않는다.

```text
Canonical Markdown
        =
Filesystem File
```

---

## 4.4 Sync State

Sync State는 분산된 클라이언트를 Vault State에 수렴시키기 위해 필요한 상태이다.

최소한 다음 개념을 표현해야 한다.

### Revision

Authoritative State가 변경될 때 그 순서를 식별할 수 있는 값.

예:

```text
1001  A.md MODIFY
1002  B.md CREATE
1003  C.md DELETE
1004  notes/x.md RENAME notes/y.md
```

구체적인 번호 생성 방식은 이후 설계에서 결정한다.

### Change

특정 Revision에서 무엇이 변경되었는지를 나타낸다.

최소한 다음 유형을 표현할 수 있어야 한다.

```text
CREATE
MODIFY
DELETE
RENAME
MOVE
```

### Deletion State

삭제는 단순히 파일이 없는 상태가 아니다.

```text
DELETE C.md
```

라는 사실 자체가 일정 기간 authoritative state로 유지되어야 한다.

이 상태를 통해 오래된 클라이언트가 삭제된 파일을 다시 생성하는 것을 방지한다.

### Operation State

클라이언트가 동일한 변경을 Retry했을 때 이미 성공한 요청인지 판단할 수 있는 정보가 필요하다.

이를 통해 다음 상황을 방지한다.

```text
Client
  │
  │ request
  ▼
Server
  │
  │ success
  ▼
Response lost

Client
  │
  │ retry
  ▼
Server

→ duplicate change 금지
```

구체적인 Idempotency 모델은 이후 문서에서 정의한다.

---

# 5. Client State Model

클라이언트도 서버와 별개의 상태를 가진다.

```text
Client State

├── Local Vault
├── Last Known Server State
├── Pending Operations
└── Conflict State
```

---

## 5.1 Local Vault

Obsidian이 직접 읽고 수정하는 로컬 파일이다.

VaultDatum는 Obsidian이 사용하는 로컬 파일을 별도의 가상 filesystem으로 대체하지 않는다.

---

## 5.2 Last Known Server State

클라이언트는 자신이 서버의 어느 상태까지 정상적으로 적용했는지를 기억해야 한다.

개념적으로:

```text
lastAppliedRevision = N
```

과 같은 상태이다.

이를 이용하여 서버와 다시 연결되었을 때:

```text
N 이후 무슨 일이 발생했는가?
```

를 확인할 수 있다.

구체적인 저장 방식은 이후 클라이언트 아키텍처에서 결정한다.

---

## 5.3 Pending Operations

사용자가 Local Vault를 변경했지만 서버가 아직 확정하지 않은 변경이다.

```text
Local file modified
        │
        ▼
Pending Operation
        │
        │ network available
        ▼
Server
```

Pending Operation은 메모리에만 존재해서는 안 된다.

다음 사건 이후에도 복구되어야 한다.

```text
Obsidian 종료
Android Process Kill
Device Reboot
Network Change
```

---

## 5.4 Conflict State

서버에 제출한 변경을 안전하게 적용할 수 없는 경우 해당 Pending Operation을 단순 삭제하지 않는다.

```text
Pending
   │
   ▼
Server detects conflict
   │
   ▼
Conflict
```

Conflict가 해결될 때까지 필요한 Local Data를 보존한다.

---

# 6. Reconciliation Model

VaultDatum의 핵심 동작은 파일 복사가 아니라 **Reconciliation**이다.

Reconciliation의 목표는 클라이언트에게 다음 질문에 답하는 것이다.

```text
내가 알고 있는 서버 상태와
현재 서버 상태의 차이는 무엇인가?
```

---

## 6.1 Normal State

정상적으로 동기화된 상태:

```text
Server Revision = 1000
Client Applied   = 1000

Pending = 0
Conflict = 0
```

이 상태를 `Synced`라고 한다.

---

## 6.2 Server Ahead

클라이언트가 오프라인인 동안 서버에 변경이 발생한다.

```text
Server = 1050
Client = 1000
```

클라이언트는 서버에 다음 의미의 요청을 할 수 있어야 한다.

```text
1000 이후의 변경을 알려줘.
```

그 결과:

```text
1001 MODIFY A.md
1002 CREATE B.md
...
1050 DELETE C.md
```

를 순차적으로 적용한다.

모든 변경 적용에 성공한 후에만:

```text
lastAppliedRevision = 1050
```

으로 갱신한다.

---

## 6.3 Client Ahead

클라이언트에서 로컬 변경이 발생한 경우:

```text
Server Revision = 1000
Client Applied   = 1000

Pending:
MODIFY A.md
```

클라이언트는 변경을 서버에 제출한다.

서버가 안전하게 적용할 수 있으면:

```text
Revision 1001
MODIFY A.md
```

로 확정한다.

클라이언트는 서버의 성공 응답을 받은 후 해당 Pending Operation을 완료 처리한다.

---

## 6.4 Both Changed

다음 상황을 가정한다.

```text
Client last known:

A.md @ Revision 1000

Server:
A.md changed at Revision 1001

Client:
A.md changed offline
```

클라이언트의 변경은 Revision 1000을 기준으로 만들어졌지만 서버에는 이미 새로운 버전이 존재한다.

```text
Base
   │
   ├── Server Change
   │
   └── Client Change
```

이 상태를 충돌 가능 상태로 판단한다.

초기 VaultDatum는 자동 병합을 필수로 하지 않는다.

따라서 기본 동작은:

```text
Conflict detected

Server version preserved
Local version preserved
```

이다.

---

## 6.5 Missed Notification

클라이언트가 실시간 이벤트를 전부 놓친 경우:

```text
Server = Revision 1200

Client = Revision 1100

Notification:
all lost
```

아무 문제도 없어야 한다.

클라이언트가 다음 reconciliation을 수행하면:

```text
changes since 1100
```

을 통해 1101~1200의 변경을 다시 확인할 수 있다.

이 때문에 Notification Channel은 authoritative channel이 아니다.

---

## 6.6 Full Reconciliation

클라이언트가 가진 동기화 상태를 신뢰할 수 없거나 변경 이력만으로 복구할 수 없는 경우 전체 상태를 비교할 수 있어야 한다.

개념적으로:

```text
Server Manifest
        ↕
Client Manifest
```

비교를 통해:

```text
Missing
Modified
Deleted
Unknown
```

파일을 식별한다.

일반적인 동기화에서는 Incremental Reconciliation을 사용하고 Full Reconciliation은 복구 경로로 사용한다.

---

# 7. 아키텍처 경계 불변조건

제품 전역 불변조건은 [00 Product Specification — 핵심 제품 불변조건](./00_product-specification.md)을 단일 기준으로 사용한다. 이 문서에서 추가하는 아키텍처 고유의 조건은 다음 하나다.

## Architecture Invariant — Vault State와 Sync State는 함께 해석한다

Vault의 현재 내용만으로 동기화 정합성을 판단하지 않는다. 삭제와 변경 순서를 포함하는 Sync State가 함께 존재해야 한다.

이 상태의 정확한 모델은 [06 Data Model](./06_data-model.md), 상태를 전이하는 순서는 [02 Synchronization Protocol](./02_synchronization-protocol.md)에 정의한다.

---
