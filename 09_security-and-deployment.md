# Security and Deployment

## 1. 문서 목적

이 문서는 개인 사용자를 대상으로 하는 Sync Server의 최소한의 Security Boundary와 Deployment 원칙을 정의한다.

이 문서는 접근 경계와 배포 계약의 기준 문서다. Server 내부 mutation/recovery 구조는 [03 Server Architecture](./03_server-architecture.md), API로 노출되는 보안·health 계약은 [07 API Specification](./07_api-specification.md), 시작·백업·로그·상태 점검 절차는 [11 Observability and Operations](./11_observability-and-operations.md)에서만 정의한다.

초기 사용 환경은 다음을 전제로 한다.

```text
Single User

Home Server

Desktop / Laptop / Mobile Clients
```

따라서 별도의 사용자 계정 시스템이나 복잡한 인증 인프라를 구축하지 않는다.

기본 원칙은:

> **Sync Server를 Public Internet에 직접 노출하지 않고, 승인된 개인 Device만 접근할 수 있는 Private Network 안에서 운영한다.**

이다.

---

# 2. Security 범위

초기 버전에서 보호해야 하는 것은 다음 정도다.

```text
승인되지 않은 외부 접근 방지

Network 구간의 Vault Content 보호

Vault Root 외부 File 접근 방지

잘못된 Upload로 인한 Data Corruption 방지

Persistent Data 보호
```

다음까지 Application이 직접 해결하지 않는다.

```text
Server Host 자체가 탈취된 경우

Client Device 자체가 탈취된 경우

Operating System 전체가 compromise된 경우
```

---

# 3. 기본 Deployment Model

권장 구조는 다음과 같다.

```text
Desktop ─┐
Laptop  ─┼── Private VPN ─── Sync Server
Phone   ─┘                       │
                                ▼
                              /data
```

Private VPN은 WireGuard 또는 동등한 역할의 Network를 사용할 수 있다.

Sync Protocol 자체는 특정 VPN 제품에 종속되지 않는다.

---

# 4. VPN의 역할

VPN Layer는 다음을 담당한다.

```text
Device Access Control

Peer Authentication

Network Encryption
```

따라서 Sync Application 자체에서 별도의:

```text
User Account

Password Login

OAuth

OIDC

Device Provisioning System
```

을 MVP에 구현하지 않는다.

---

# 5. Client ID와 Authentication

Sync Protocol의:

```text
clientId
```

는 Security Credential이 아니다.

용도는 다음과 같다.

```text
어떤 Client가 Change를 만들었는지 식별

Operation Actor 기록

진단
```

VPN이 Client의 Network Access를 이미 제한한다고 가정한다.

따라서:

```text
clientId
    ≠
authentication credential
```

이다.

---

# 6. VPN 내부 Transport

VPN 자체에서 Encryption을 제공하므로 MVP에서는 VPN 내부에서:

```text
HTTP
```

를 사용할 수 있다.

예:

```text
http://10.0.0.10:8080
```

이 구성은 다음 조건을 전제로 한다.

```text
Sync API가 VPN Interface 또는 Trusted Private Network에서만 접근 가능

Public Internet에서 직접 접근 불가능
```

---

# 7. Public Internet 직접 노출 금지

다음 구성은 지원하지 않는다.

```text
Internet
    │
    ▼
http://server:8080
    │
    ▼
Sync Server
```

Application Level Authentication이 없기 때문에 Public Internet에 Plain HTTP Server를 직접 열어서는 안 된다.

---

# 8. Public Access가 필요한 경우

VPN을 사용할 수 없는 환경에서는 선택적으로:

```text
Internet
    │
    ▼
HTTPS Reverse Proxy
    │
    ▼
Sync Server
```

구조를 사용할 수 있다.

이 경우 최소한:

```text
HTTPS

+

간단한 Authentication
```

이 필요하다.

구체적인 Public Authentication 방식은 MVP의 핵심 기능으로 정의하지 않는다.

예를 들어 Reverse Proxy에서 Authentication을 처리하거나 단일 API Token을 사용할 수 있다.

---

# 9. Application Authentication은 Optional

기본 Private VPN Deployment에서는 Sync Server에 별도의 Login System을 구현할 필요가 없다.

따라서 MVP의 Sync API는:

```text
Trusted Private Network 내부 호출
```

을 기본 Security Boundary로 간주한다.

향후 Public Service 또는 Multi-user Service로 확장할 경우 Authentication Layer를 추가한다.

---

# 10. `.obsidian/` 격리

`.obsidian/`은 Client마다 독립적인 설정 영역이다.

따라서 Server와 Client 모두 다음 Path를 Sync 대상에서 제외한다.

```text
.obsidian

.obsidian/**
```

Client Observer가 무시하는 것만으로는 충분하지 않다.

Server도 Mutation Path를 독립적으로 검증한다.

---

# 11. Path Traversal 방지

Client가 전달하는 Path는 항상 Vault Root 기준 Relative Path다.

예:

```text
notes/a.md
```

다음 요청은 허용하지 않는다.

```text
../outside.md

../../etc/passwd

/data/other.md

C:\outside
```

Filesystem Operation 전 반드시 Path Validation을 수행한다.

---

# 12. Vault Root Escape 방지

Resolved Filesystem Path는 항상:

```text
Vault Root 내부
```

여야 한다.

즉:

```text
resolve(VaultRoot, SyncPath)
```

결과가 Vault Root 외부를 가리킬 수 없어야 한다.

Client가 전달한 문자열을 그대로 Filesystem Path에 연결하지 않는다.

---

# 13. Symbolic Link

MVP에서는 Symbolic Link를 Sync 대상으로 지원하지 않는다.

이유:

```text
Vault Root Escape 가능성

Platform별 동작 차이

External Filesystem 접근 가능성
```

때문이다.

Server가 Symlink를 발견하면 따라가지 않는다.

---

# 14. 지원 Filesystem Entry

초기 버전은 다음만 지원한다.

```text
Regular File

Directory
```

다음은 Sync 대상이 아니다.

```text
Symbolic Link

Socket

Named Pipe

Device File
```

---

# 15. Upload 검증

Server는 Client가 전달한:

```text
contentHash

size
```

를 그대로 신뢰하지 않는다.

Upload Stream을 실제로 읽으면서 Server가 직접:

```text
Hash 계산

Size 계산
```

을 수행한다.

선언된 값과 다르면 Commit하지 않는다.

---

# 16. Upload 제한

실수 또는 비정상 요청으로 하나의 Operation이 무제한 데이터를 사용할 수 없도록:

```text
maxUploadSize
```

를 둘 수 있다.

정확한 기본값은 구현 단계에서 결정한다.

---

# 17. Operation Retry 안전성

Operation retry의 idempotency는 보안 인증 수단이 아니라 data-integrity 요구사항이다. `operationId`와 request digest의 의미 및 검증 순서는 [02 Synchronization Protocol](./02_synchronization-protocol.md)과 [07 API Specification](./07_api-specification.md)을 따른다.

---

# 18. Server Process 권한

Server Process는 가능하면 root가 아닌 전용 User로 실행한다.

해당 Process는 필요한:

```text
/data
```

영역에만 Write 권한을 가진다.

---

# 19. Persistent Data Root

Container 내부 Persistent Data는:

```text
/data/
├── vault/
├── state/
├── staging/
└── recovery/
```

에 둔다.

Container Image 자체에 사용자 Vault를 저장하지 않는다.

---

# 20. Docker Deployment

가장 단순한 기본 배포는:

```text
Host

/srv/sync/
    │
    │ bind mount
    ▼

Container

/data/
```

형태다.

Container가 삭제되거나 Update되어도 Persistent Data는 유지된다.

---

# 21. Kubernetes Deployment

Kubernetes를 사용하는 경우에도 구조는 동일하다.

```text
PVC
 │
 ▼
/data
```

에 Persistent Volume을 Mount한다.

초기 Server는:

```text
replicas = 1
```

을 사용한다.

---

## 21.1 Runtime Identity와 Host-backed Volume

Server Process는 root가 아닌 User로 실행한다. 그러나 Container의 numeric UID/GID는
hostPath, local volume, bind mount를 사용할 때 Host의 같은 numeric identity로
그대로 보인다.

```text
Container UID 1001
        ↓
Host UID 1001
        ↓
Host의 /etc/passwd에 등록된 계정 이름
```

따라서 Image 안의 일반적인 non-root fallback UID를 Host의 특정 사용자 계정이라고
가정해서는 안 된다. 어떤 Host에서는 같은 UID가 개인 사용자 또는 전혀 다른 서비스
계정에 대응할 수 있다.

### 공용 배포 설정의 경계

공개된 Container Image와 Kubernetes base는 non-root 실행을 요구할 수 있지만, 다음
설치별 값을 고정해서는 안 된다.

```text
runAsUser

runAsGroup

fsGroup

hostPath

Host user name

Host node name
```

특히 `fsGroup`은 group 접근을 조정할 뿐 파일 owner를 원하는 Host 계정으로 바꾸지
않는다. Server가 content와 staging file을 owner-only로 생성할 수 있으므로, Host
파일 owner를 정하는 문제의 대체 수단도 아니다.

### 설치별 전용 Service Account

host-backed storage를 쓰는 운영자는 Host에 VaultDatum 전용 service account를
만들고, 해당 UID/GID를 비공개 deployment overlay 또는 접근 제어된 deployment
repository에서 선택한다. 개인 login 계정이나 다른 Application 계정을 재사용하지
않는다.

```text
Host service account: vaultdatum
        ↓
Host Vault directory owner: vaultdatum:vaultdatum
        ↓
Pod runAsUser/runAsGroup: vaultdatum UID/GID
        ↓
Server-created files: vaultdatum:vaultdatum
```

Pod security context의 `runAsUser`, `runAsGroup`, 필요할 때의 `fsGroup`은 같은
설치별 identity와 일치해야 한다. 해당 값을 담은 overlay, Host 경로, 실제 UID/GID,
node selector는 public source repository에 넣지 않는다.

### Volume 준비와 Migration

새 Volume은 Pod를 시작하기 전에 운영자가 전용 account owner로 준비한다. 일반
runtime Pod가 root init container로 매 startup마다 `/data` 전체를 재귀 `chown`하지
않는다. 이렇게 하면 기존 Vault의 owner를 예기치 않게 바꾸지 않고 runtime 권한도
최소화할 수 있다.

기존 Volume을 전용 identity로 옮길 때는 다음 순서를 따른다.

```text
1. Server를 중지하여 단일 writer를 보장한다.
2. /data 전체와 SQLite WAL을 crash-consistent 방식으로 backup한다.
3. 운영자가 Host에서 전용 account owner와 필요한 directory mode를 설정한다.
4. private overlay의 Pod identity를 같은 UID/GID로 설정한다.
5. Pod를 시작하고 /health/ready, SQLite open, Vault read/write를 확인한다.
```

Server가 생성한 Vault content는 기본적으로 service account만 읽을 수 있는 mode를
유지할 수 있다. Host 관리자가 내용을 읽어야 하면 root 권한 또는 명시적으로 부여한
운영 권한을 사용하며, 편의를 위해 모든 Host 사용자에게 읽기 권한을 주지 않는다.

검증은 적어도 다음을 포함한다.

```text
Pod process UID/GID = private overlay의 설정값

새 Vault file owner = 전용 Host service account

허가되지 않은 Host login account는 Vault content를 읽지 못함

Pod 재시작 뒤에도 owner와 /data 내용이 유지됨
```

---

# 22. Single Writer

하나의 Vault에는 하나의 Server Instance만 Write해야 한다.

```text
One Vault
    =
One Active Server Writer
```

이다.

Docker에서는 동일 Data Directory를 두 Container가 동시에 사용하지 않는다.

Kubernetes에서는 기본적으로:

```text
replicas = 1
```

과 Single-writer Storage를 사용한다.

---

# 23. Network Filesystem

초기 버전에서는 NFS 같은 Network Filesystem을 기본 지원 Storage로 간주하지 않는다.

Server Persistence는 다음 semantics를 필요로 한다.

```text
Atomic Rename

Reliable fsync

File Locking

SQLite Locking
```

개인 Home Server에서는 Local Disk 또는 일반적인 Persistent Block Storage를 우선한다.

---

# 24. 운영 경계

배포 구성은 다음 운영 계약을 충족해야 한다.

* Server는 startup recovery가 끝나기 전 Sync 요청을 받지 않는다.
* `/health/live`와 `/health/ready`의 의미는 [07 API Specification](./07_api-specification.md)과 [11 Observability and Operations](./11_observability-and-operations.md)을 따른다.
* Log에는 콘텐츠나 credential을 남기지 않는다. 필요한 필드와 rotation은 [11 Observability and Operations](./11_observability-and-operations.md)을 따른다.
* Backup은 Vault Filesystem과 SQLite Sync State를 함께 다루며, 방법과 복구 절차는 [11 Observability and Operations](./11_observability-and-operations.md)을 따른다.
* 배포 설정은 최소 `DATA_ROOT`, listen address/port, maximum upload size, logging level을 별도 instance configuration으로 제공한다. 이는 동기화되거나 Vault에 저장되는 설정이 아니다.

이 문서는 해당 운영 절차를 중복 정의하지 않는다.

---

# 25. 권장 개인용 구성

초기 버전의 권장 구성은 다음 하나로 단순화한다.

```text
Home Server

Docker
    │
    └── Persistent /data

WireGuard
    │
    └── 개인 Device만 Peer 등록

Sync Server
    │
    └── VPN Network에서만 접근
```

이 환경에서는 다음 기능을 구현하지 않는다.

```text
User Account

Password Login

OAuth / OIDC

Device Token 관리

Token Rotation

Public Registration

Admin UI

Certificate Management
```

---

# 26. Security 확장은 나중에 추가한다

향후 다음 요구가 생기면 Security Layer를 확장할 수 있다.

```text
Public Internet Service

Multiple Users

Vault Sharing

Hosted SaaS

Organization Deployment
```

이 경우:

```text
HTTPS

Device Authentication

User Authentication

Authorization

Credential Revocation
```

등을 별도 Architecture로 추가한다.

Sync Protocol 자체는 이런 인증 방식에 의존하지 않도록 유지한다.

---

# 27. Security Invariants

## Invariant 1 — Server Is Not Public by Default

MVP Sync Server는 Public Internet에 직접 노출하지 않는다.

---

## Invariant 2 — Network Layer Provides Access Control

기본 Deployment에서는 VPN이 Device Authentication과 Transport Encryption을 담당한다.

---

## Invariant 3 — Client ID Is Not a Credential

`clientId`만 알고 있다고 Server 접근 권한이 생기지 않는다.

---

## Invariant 4 — Paths Cannot Escape Vault Root

어떤 Client Request도 Vault Root 외부 Filesystem에 접근할 수 없다.

---

## Invariant 5 — Symlinks Are Not Followed

MVP에서는 Symlink를 통해 외부 Filesystem에 접근하지 않는다.

---

## Invariant 6 — Server Verifies Uploaded Content

Server는 Upload Content의 Hash와 Size를 직접 검증한다.

---

## Invariant 7 — One Vault Has One Active Writer

동일 Vault를 두 Server Instance가 동시에 수정하지 않는다.

---

## Invariant 8 — Persistent Data Outlives Container

Container 또는 Pod 교체로 Vault와 Sync Metadata가 삭제되어서는 안 된다.

---

## Invariant 9 — Host-backed Storage Uses an Installation-specific Service Identity

hostPath, local volume, bind mount의 `/data` owner는 공용 Image의 fallback UID가
아니라 설치별 전용 service account여야 한다. Pod runtime identity와 Volume
provisioning identity는 일치하며, 개인 Host 계정이나 해당 설치의 numeric UID/GID는
공용 배포 설정에 기록하지 않는다.

---

# 28. 최종 구조

```text
Laptop ──────┐
Desktop ─────┼──── VPN ───── Home Server
Phone ───────┘                    │
                                  ▼
                            Sync Container
                                  │
                                  ▼
                                /data
                         ┌────────┼────────┐
                         │        │        │
                       vault    state   recovery
                                  │
                               sync.db
```

이 Security / Deployment 모델의 핵심은:

> **개인용 Sync Application이 자체 인증 플랫폼을 만드는 대신 Private Network를 신뢰 경계로 사용하고, Application은 Vault Path와 Data Integrity를 안전하게 처리하는 데 집중하는 것**

이다.
