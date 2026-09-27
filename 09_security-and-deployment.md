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

`0.3.x`의 기본 원칙은 다음과 같다.

> **Sync Server를 Public Internet에 직접 노출하지 않고, 승인된 개인 Device만 접근할 수 있는 Private Network 안에서 운영한다.**

`0.4.0`은 이 private 배포 모델을 유지하면서, 단일 개인 Vault를 public Internet에서
사용할 수 있는 별도 access profile을 추가한다. public profile은 HTTPS와 Vault별
bearer token을 함께 사용하며, 익명 접근, 사용자 가입, password login, 여러
사용자 권한 모델을 추가하지 않는다.

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

따라서 `private-network` profile에서는 Sync Application 자체에서 별도의:

```text
User Account

Password Login

OAuth

OIDC

Device Provisioning System
```

을 구현하지 않는다. `public-token` profile의 Vault token은 VPN의 대체 network
boundary이며 사용자 계정 시스템은 아니다.

---

# 5. Client ID와 Authentication

Sync Protocol의:

```text
clientId
```

는 access token이 아니다.

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
Vault access token
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

# 7. Public Internet 직접 노출 조건

Plain HTTP Server, 인증 없는 Server, `private-network` profile Server를 public
Internet에 노출해서는 안 된다.

```text
Internet
    │
    X
    │
http://server:8080
```

`0.4.0`의 public 배포는 아래 구성을 모두 만족할 때만 지원한다.

```text
Internet
    │
    ▼
TLS-terminating Gateway
    │ HTTPS + rate limit + no token logging
    ▼
VaultDatum Server (public-token profile)
    │ Authorization: Bearer <Vault token>
    ▼
Authoritative Vault
```

Gateway, DNS, certificate만 추가하고 Server를 인증 없이 여는 것은 지원하지 않는다.
namespace, NetworkPolicy, 방화벽도 Vault token 인증의 대체 수단이 아니다.

---

# 8. Access Profile

각 Server instance는 시작할 때 하나의 access profile을 명시적으로 선택한다.
요청 출발 IP, URL의 hostname, Gateway header를 보고 Server가 public/private 여부를
추론해서는 안 된다.

| Profile | 허용 transport | Server access token | 사용처 |
| --- | --- | --- | --- |
| `private-network` | 보호된 VPN 안의 HTTP 또는 HTTPS | 없음 | 기존 home network 배포 |
| `public-token` | HTTPS만 | Vault bearer token | 단일 개인 Vault의 public access |

`private-network`이 default다. `public-token`은 non-empty Vault token file이 준비되지
않으면 Server가 ready 상태가 되지 않아야 한다.

Server 설정은 다음 논리 이름을 사용한다. 실제 token 값은 설정 파일이나 환경 변수에
넣지 않고, `token-file`이 가리키는 read-only Secret volume에만 둔다.

```yaml
vaultdatum:
  access-profile: private-network # 또는 public-token
  auth:
    token-file: /run/secrets/vaultdatum/access-token
```

---

# 9. 0.4.0 Vault Access Token

`public-token` profile은 user/password나 browser login 대신, operator가 장치에
out-of-band로 전달하는 Vault bearer token을 사용한다. token 하나는 해당
Server instance의 Vault 전체에 대한 read/write 권한을 가진다.

```text
token possession
        =
Vault access
```

따라서 Vault token은 다음 형식을 가진다.

```text
vd1_<256-bit-random-secret>
```

* secret은 cryptographically secure random source로 최소 256 bit를 생성한다.
* token plaintext는 deployment repository 밖의 Kubernetes Secret 또는 동등한
  protected file에만 둔다. Server는 시작 때 read-only file에서 읽어 constant-time
  비교에만 사용하며 SQLite, Vault, log, metric, error response, diagnostic에 저장하지
  않는다.
* Kubernetes Secret은 public source repository에 넣지 않으며, Secret을 읽을 수 있는
  RBAC principal을 Server workload와 필요한 operator로 제한한다.
* token은 URL query, request body, filename, WebSocket URL에 넣지 않는다.

HTTP Sync request는 정확히 한 개의 다음 header로 token을 전달한다.

```text
Authorization: Bearer vd1_...
```

HTTP bearer token은 TLS가 없으면 노출되므로 `public-token` client는 `https://`
URL만 받아들인다. Public Gateway는 인증된 sync request를 다른 origin으로 redirect해서는
안 된다. hostname을 바꾸는 migration은 새 URL과 새 token을 명시적으로 배포하는 절차다.

### Provision, rotation, revocation

operator가 token을 한 번 생성해 Secret volume으로 mount하고, 신뢰할 수 있는
out-of-band channel로 각 Obsidian 장치에 전달한다. public HTTP API에는 registration,
token creation, token listing, password reset endpoint가 없다.

`0.4.0` token은 Vault 전체에서 공유한다. 분실하거나 노출이 의심되면 새 token으로
Secret을 교체하고 Pod를 restart한다. 그 시점부터 이전 token은 모두 무효화되며 각
장치에는 새 token을 다시 입력해야 한다. 이는 장치별 권한 취소보다 단순한 개인 Vault
운영 모델이다. per-device token, 사용자/조직 권한, folder scope, anonymous share
link는 이번 버전에서 제공하지 않는다.

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

공개된 Container Image와 Kubernetes base는 Server container의 non-root 실행을
요구할 수 있지만, 다음 설치별 runtime 값을 고정해서는 안 된다.

```text
runAsUser

runAsGroup

fsGroup

hostPath

Host user name

Host node name
```

공용 base가 writable `/tmp` 같은 ephemeral volume을 준비하기 위해 제한된 root init
container를 쓰는 것은 가능하다. 단, 이 helper는 persistent `/data`를 mount하거나
owner를 바꾸지 않으며 Server runtime identity를 정하지 않는다.

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
* Log에는 콘텐츠나 Vault token을 남기지 않는다. 필요한 필드와 rotation은 [11 Observability and Operations](./11_observability-and-operations.md)을 따른다.
* Backup은 Vault Filesystem과 SQLite Sync State를 함께 다루며, 방법과 복구 절차는 [11 Observability and Operations](./11_observability-and-operations.md)을 따른다.
* 배포 설정은 최소 `DATA_ROOT`, listen address/port, maximum upload size, logging level을 별도 instance configuration으로 제공한다. 이는 동기화되거나 Vault에 저장되는 설정이 아니다.

이 문서는 해당 운영 절차를 중복 정의하지 않는다.

---

# 25. 권장 개인용 구성

`private-network` profile의 권장 구성은 다음과 같다.

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

이 환경에서는 Vault bearer token을 추가로 요구하지 않는다. 단, public endpoint와
같은 instance 또는 같은 Service를 공유해서는 안 된다.

---

# 26. Public Token 구성

`public-token` profile은 개인 Vault 하나를 Internet에서 사용할 때의 최소 구성이다.

```text
Obsidian Client
    │ HTTPS / WSS
    │ Authorization: Bearer Vault token
    ▼
Public Gateway
    │ Cluster-local HTTP
    ▼
VaultDatum Server
    │
    ▼
Dedicated PVC /data
```

public Gateway는 TLS를 종료하고, Server Service는 `ClusterIP`로 유지한다. Server의
8080 port, PVC, SQLite는 public Service로 노출하지 않는다. Gateway가
backend에 연결할 수 있는 source만 NetworkPolicy에서 허용하며, Client IP allow-list는
보조 제어일 뿐 authentication의 대체가 아니다.

공개 URL은 vault마다 독립된 hostname을 쓴다. 예를 들어 `second-brain` Vault는
`https://second-brain.example.com`처럼 하나의 stable origin을 사용한다. URL 변경은
새 Server instance 또는 다른 Vault로의 연결로 취급하며 Client가 기존 token을 자동
전달해서는 안 된다.

Gateway와 application log는 다음을 절대 기록하지 않는다.

```text
Authorization header

Sec-WebSocket-Protocol의 realtime ticket

Vault token plaintext
```

Gateway는 `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, 적절한
request body 제한을 제공한다. Sync API는 browser website용 API가 아니므로 CORS를
기본적으로 활성화하지 않는다.

### WebSocket authentication

browser-compatible WebSocket API는 HTTP Authorization header를 임의로 설정할 수
없다. bearer token을 query parameter로 보내면 URL logging 위험이 있으므로
사용하지 않는다.

Client는 인증된 HTTPS 요청으로 `POST /api/v1/realtime-tickets`를 호출해 one-time
ticket을 받고, 만료 전 한 번만 아래 subprotocol offer로 전달한다.

```text
Sec-WebSocket-Protocol: vaultdatum.v1, vaultdatum.ticket.<opaque-ticket>
```

Server는 정상 handshake에서 `vaultdatum.v1`만 선택하고 ticket을 echo하지 않는다.
ticket은 최소 128 bit random value, 60초 이하의 만료, single-use를 가져야 한다. ticket은
Server memory에만 존재하며 Pod restart와 token rotation 뒤에는 모두 무효다. ticket 발급
또는 WebSocket 연결 실패는 동기화 correctness를
바꾸지 않으며, Client는 기존 catch-up HTTP flow로 복구한다.

---

# 27. 0.4.0 범위 밖

다음 기능은 `0.4.0`에 포함하지 않는다.

```text
User Account

Password Login

OAuth / OIDC

Public Registration

Admin UI

장치별 token 또는 개별 token 취소

Server 자체의 certificate provisioning 및 renewal

Multi-user / organization authorization

Vault sharing 또는 folder-level permission

Anonymous public link
```


OIDC, mTLS, multi-user authorization처럼 다른 authentication mechanism을 더할 때도
Sync operation의 base/revision/idempotency 규칙은 바뀌지 않아야 한다.

public Gateway의 유효한 certificate는 0.4.0 배포의 필수 조건이다. 다만 certificate
발급·갱신 자동화는 VaultDatum Server의 기능이 아니라 cluster 또는 Gateway 운영 계층의
책임으로 남긴다.

---

# 28. Security Invariants

## Invariant 1 — Server Is Not Public by Default

MVP Sync Server는 Public Internet에 직접 노출하지 않는다.

---

## Invariant 2 — Network Layer Provides Access Control

기본 Deployment에서는 VPN이 Device Authentication과 Transport Encryption을 담당한다.

---

## Invariant 3 — Client ID Is Not an Access Token

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

## Invariant 10 — Public Access Is Explicitly Authenticated

`public-token` profile의 Sync HTTP와 notification handshake는 HTTPS/WSS 위에서 유효한
Vault token 또는 그것으로 발급한 one-time ticket을 검증한 뒤에만 처리한다.

---

## Invariant 11 — Tokens Never Enter URLs or Diagnostics

Vault token plaintext와 realtime ticket은 URL, synchronized state, Vault
content, log, metric, error response, diagnostic에 포함되지 않는다.

---

## Invariant 12 — Token Rotation Stops Old Access

Secret 교체와 Server restart가 완료된 뒤 이전 Vault token과 이전 realtime ticket은 새
HTTP 요청 또는 WebSocket handshake를 통과할 수 없다. 기존 WebSocket은 restart로 닫히며,
correctness는 HTTP catch-up으로 유지한다.

---

# 29. Private Profile 최종 구조

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

private profile의 핵심은:

> **개인용 Sync Application이 자체 인증 플랫폼을 만드는 대신 Private Network를 신뢰 경계로 사용하고, Application은 Vault Path와 Data Integrity를 안전하게 처리하는 데 집중하는 것**

이다.

public-token profile은 이 private model을 대체하지 않는다. public Vault마다 별도 Server
instance, PVC, hostname, token set을 두고 HTTPS와 application token을
추가하는 확장이다.
