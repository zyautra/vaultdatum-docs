# VaultDatum 제품 사양서

> 이 문서는 제품 요구사항과 전역 불변조건의 기준 문서다. 구현·프로토콜·저장소·운영의 세부 규칙은 아래 전문 문서에서만 정의하며, 이 문서에 같은 규칙을 다시 쓰지 않는다.
>
> 모든 설계 문서는 제품이 항상 만족해야 하는 목표 상태만 기술한다. 버전별 범위와 변경 이력은 릴리즈 태그와 커밋에서 관리하고, 아직 약속하지 않은 개선 후보는 [13 Backlog](./13_backlog.md)에 둔다.

| 관심사                                            | 기준 문서                                                                                                                                                                                                                                    |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 상위 시스템 구조                                  | [01 Architecture Overview](./01_architecture-overview.md)                                                                                                                                                                                    |
| 동기화 동작                                       | [02 Synchronization Protocol](./02_synchronization-protocol.md)                                                                                                                                                                              |
| Server / Client 내부 구조                         | [03 Server Architecture](./03_server-architecture.md), [04 Client Architecture](./04_client-architecture.md)                                                                                                                                 |
| Conflict UX, 논리 상태, HTTP 계약                 | [05 Conflict Resolution](./05_conflict-resolution.md), [06 Data Model](./06_data-model.md), [07 API Specification](./07_api-specification.md)                                                                                                |
| Client 설정, 상태 표시, 첫 동기화, 사용자 복구 UX | [12 Client User Experience](./12_client-user-experience.md)                                                                                                                                                                                  |
| 영속성, 배포 보안, 검증, 운영                     | [08 Persistence Design](./08_persistence-design.md), [09 Security and Deployment](./09_security-and-deployment.md), [10 Testing Strategy](./10_testing-strategy.md), [11 Observability and Operations](./11_observability-and-operations.md) |
| 약속하지 않은 개선 후보                           | [13 Backlog](./13_backlog.md)                                                                                                                                                                                                                |

## 1. 제품 개요

### 1.1 제품명

**VaultDatum**

Obsidian 클라이언트용 플러그인은 **VaultDatum for Obsidian**으로 제공한다.

### 1.2 한 줄 정의

> **VaultDatum는 여러 장치의 로컬 Obsidian Vault를 하나의 중앙 Vault에 지속적으로 수렴시키는 개인용 동기화 시스템이다.**

### 1.3 제품 목적

사용자는 데스크탑, 스마트폰, 노트북 등 여러 장치에서 각각 로컬 Obsidian Vault를 사용한다.

VaultDatum는 홈서버에 위치한 Vault를 모든 장치가 공유하는 **절대적인 기준 상태(Authoritative Vault)** 로 사용하고, 각 장치의 Vault가 네트워크 단절이나 앱 종료와 관계없이 최종적으로 해당 상태와 일치하도록 만든다.

VaultDatum의 목적은 원격 파일시스템을 제공하는 것이 아니라 다음을 보장하는 것이다.

- 각 장치에서는 항상 로컬 Vault를 사용한다.
- 서버 Vault가 전체 시스템의 유일한 기준 상태가 된다.
- 오프라인에서도 문서를 읽고 수정할 수 있다.
- 네트워크 복구 후 변경사항이 자동으로 서버에 반영되고 다른 클라이언트로 전달된다.
- 서로 다른 장치에서 동시에 수정된 문서를 무조건 덮어쓰지 않는다.
- 클라이언트가 오랫동안 연결되지 않았더라도 다시 접속하면 서버의 최신 상태로 수렴한다.
- 서버 연결이나 실시간 알림이 일시적으로 끊어져도 데이터 정합성이 깨지지 않는다.

---

## 2. 해결하려는 문제

Obsidian은 로컬 파일을 중심으로 동작한다.

이 특성은 빠르고 단순하지만 여러 장치에서 하나의 Vault를 사용하려면 별도의 동기화 수단이 필요하다.

특히 모바일 환경에서는 다음 상황을 안정적으로 처리해야 한다.

- 앱이 백그라운드에서 종료된다.
- 네트워크가 Wi-Fi와 모바일 데이터 사이에서 전환된다.
- 장시간 오프라인 상태로 문서를 수정한다.
- PC와 모바일에서 같은 문서를 동시에 수정한다.
- 한 장치에서 문서를 삭제하거나 이동한 동안 다른 장치가 오래된 상태를 유지한다.
- 실시간 연결이 끊겨 변경 알림을 받지 못한다.

VaultDatum는 실시간 연결 자체를 신뢰하지 않고 **서버 Vault의 현재 상태와 변경 이력을 기준으로 클라이언트를 다시 수렴시킴으로써** 이러한 문제를 해결한다.

---

## 3. 제품 철학

### 3.1 Local First

사용자가 실제로 읽고 편집하는 파일은 각 장치의 로컬 Vault에 존재한다.

서버 연결 여부가 Obsidian의 기본적인 읽기와 편집을 방해해서는 안 된다.

Local First는 각 클라이언트가 독립적인 원본이라는 의미가 아니다.

클라이언트는 로컬에서 자유롭게 작업할 수 있지만 동기화 시스템 전체의 최종 기준 상태는 VaultDatum Server가 가진다.

### 3.2 Server Authoritative

VaultDatum Server의 Vault는 전체 시스템의 **유일한 Authoritative Vault** 이다.

클라이언트의 로컬 Vault는 서버 Vault의 Replica이자 오프라인 작업 공간으로 취급한다.

```text
Desktop Vault ───┐
                 │
Mobile Vault ────┼──► VaultDatum Server
                 │         │
Laptop Vault ────┘         ▼
                   Authoritative Vault
```

클라이언트 간에 직접 상태를 합의하거나 동기화하지 않는다.

모든 정상적인 동기화 흐름은 VaultDatum Server를 기준으로 이루어진다.

클라이언트에서 발생한 변경 역시 서버가 받아들인 이후에 전체 시스템의 확정된 상태가 된다.

### 3.3 Eventual Consistency

모든 장치가 항상 즉시 같은 상태일 필요는 없다.

대신 다음 조건이 만족되면 모든 클라이언트는 최종적으로 VaultDatum Server와 동일한 상태에 도달해야 한다.

- 네트워크 연결이 가능하다.
- 충분한 동기화 시간이 주어진다.
- 해결되지 않은 충돌이 존재하지 않는다.
- 새로운 변경이 계속 발생하지 않는다.

### 3.4 Offline First

네트워크가 없는 상태에서도 사용자는 정상적으로 Vault를 편집할 수 있어야 한다.

오프라인 동안 발생한 변경은 앱 종료나 장치 재부팅 이후에도 유실되어서는 안 된다.

오프라인 변경은 서버에 반영되기 전까지 해당 클라이언트의 미확정 변경으로 취급한다.

### 3.5 Notification Is Not Correctness

실시간 알림은 빠른 동기화를 위한 기능일 뿐 정합성의 전제 조건이 아니다.

클라이언트가 모든 실시간 알림을 놓치더라도 이후 서버와 다시 통신하면 정상 상태를 복구할 수 있어야 한다.

---

## 4. 제품 구성

VaultDatum는 사용자 관점에서 다음 두 구성요소를 가진다.

### VaultDatum Server

홈서버에서 실행되며 Authoritative Vault를 보유한다.

역할:

- Authoritative Vault 보관
- 클라이언트 변경 수신 및 확정
- 변경 이력 제공
- 클라이언트 간 변경 전달
- 충돌 감지
- 동기화 상태 관리
- 배포 profile에 따른 클라이언트 인증 및 접근 통제

네트워크 접근 통제와 통신 보안은 배포 방식에 따라 WireGuard 등의 외부 VPN에 위임할 수 있다.

### VaultDatum for Obsidian

각 Obsidian 환경에서 실행되는 클라이언트 플러그인이다.

역할:

- 로컬 Vault 변경 감지
- 서버에 로컬 변경 전달
- 서버 변경 수신
- 오프라인 변경 보관
- 동기화 상태 표시
- 충돌 표시 및 해결 지원

VaultDatum Server 자체는 Obsidian UI를 제공하지 않는다.

---

## 5. 핵심 사용자 시나리오

### 5.1 일반적인 장치 간 동기화

사용자가 데스크탑 Obsidian에서 문서를 수정한다.

VaultDatum가 해당 변경을 Authoritative Vault에 반영한 후 모바일 클라이언트가 변경사항을 받아 자신의 로컬 Vault에 적용한다.

사용자는 어느 장치에서 작업했는지를 의식하지 않고 같은 문서를 계속 사용할 수 있어야 한다.

### 5.2 모바일 오프라인 편집

사용자가 네트워크가 없는 상태에서 모바일 Obsidian으로 여러 문서를 수정한다.

변경사항은 장치에 안전하게 보존된다.

네트워크가 복구되면 VaultDatum가 변경사항을 서버와 동기화한다.

앱이 중간에 종료되거나 장치가 재부팅되어도 변경사항이 사라져서는 안 된다.

### 5.3 장기간 연결되지 않은 장치

노트북을 한 달 동안 사용하지 않는 동안 다른 장치에서 수백 개의 변경이 발생할 수 있다.

노트북을 다시 실행하면 사용자가 전체 Vault를 수동으로 복사하지 않아도 VaultDatum가 필요한 변경사항을 확인하고 서버의 최신 상태로 수렴시킨다.

### 5.4 동시 수정

데스크탑과 모바일이 동일한 문서의 같은 이전 상태에서 각각 다른 내용을 수정할 수 있다.

VaultDatum는 나중에 업로드된 파일을 단순히 최신 파일이라는 이유로 덮어써서는 안 된다.

VaultDatum는 이러한 상황을 **충돌로 감지**하고 두 내용을 모두 보존한다. 어떤 내용을 사용할지는 사용자가 결정한다.

### 5.5 삭제와 오프라인 장치

데스크탑에서 문서를 삭제한 동안 모바일 장치가 오프라인이었다고 가정한다.

모바일 장치가 다시 연결되었을 때 오래된 파일을 가지고 있다는 이유로 삭제된 문서가 자동으로 부활해서는 안 된다.

### 5.6 Rename 및 Move

사용자가 한 장치에서 문서 이름을 변경하거나 다른 폴더로 이동하면 다른 장치에서도 동일한 구조로 반영되어야 한다.

가능한 경우 단순한 "삭제 후 새로운 파일 생성"과 실제 이동을 구분해야 한다.

---

## 6. 동기화 대상

VaultDatum의 동기화 단위는 Obsidian Vault 안의 사용자 콘텐츠 파일과 디렉터리이다.

지원 대상:

- Markdown 문서
- 디렉터리
- 이미지
- PDF
- 기타 Obsidian Attachment

### 6.1 `.obsidian` 제외

`.obsidian/` 디렉터리는 VaultDatum의 동기화 대상에서 제외한다.

Obsidian 설정과 플러그인 구성은 장치별로 다를 수 있기 때문이다.

예를 들어:

- 모바일에서만 사용하는 플러그인
- 데스크탑에서만 사용하는 플러그인
- 장치별 UI 설정
- 장치별 Workspace 상태
- 플랫폼에 따라 다른 플러그인 설정

등이 존재할 수 있다.

따라서 VaultDatum는 다음과 같이 구분한다.

```text
Vault/
├── notes/            ← Sync
├── attachments/      ← Sync
├── projects/         ← Sync
│
└── .obsidian/        ← Do Not Sync
```

`.obsidian/`은 각 클라이언트의 로컬 상태로 취급하며 VaultDatum Server의 Authoritative Vault 개념에서도 제외한다.

---

## 7. 변경 유형

VaultDatum는 최소 다음 변경을 구분할 수 있어야 한다.

- Create
- Modify
- Delete
- Rename
- Move

각 변경은 파일 또는 빈 디렉터리 하나를 단위로 안전하게 동기화한다. Rename과 Move는 원본과 대상 경로를 하나의 변경으로 확정한다.

---

## 8. 충돌 정책

### 8.1 기본 원칙

VaultDatum는 서버에 존재하는 새로운 변경을 오래된 클라이언트가 무조건 덮어쓰는 것을 허용하지 않는다.

**충돌을 잘못 자동 해결하는 것보다 충돌을 명확하게 감지하는 것을 우선한다.**

### 8.2 Markdown 충돌

충돌이 발생하면 서버 버전과 로컬 버전이 모두 유실되지 않아야 한다.

사용자는 충돌 사실을 확인하고 서버 버전, 로컬 버전, 두 파일 모두 보존, 또는 수동 병합 중에서 최종 내용을 결정할 수 있어야 한다.

VaultDatum는 사용자 확인 없이 Markdown 내용을 자동 병합하지 않는다.

### 8.3 Binary 충돌

이미지나 PDF 등의 Binary 파일은 내용 병합을 시도하지 않는다.

충돌한 두 파일을 모두 보존하거나 사용자에게 선택을 요청한다.

### 8.4 삭제 충돌

다음 상황은 명시적으로 처리한다.

- 한 장치에서 삭제하고 다른 장치에서 수정
- 한 장치에서 이동하고 다른 장치에서 수정
- 삭제된 문서를 오래된 오프라인 장치가 다시 업로드
- 서로 다른 장치에서 동일 파일을 서로 다른 위치로 이동

데이터를 조용히 유실시키는 자동 해결보다 충돌로 남기는 것을 우선한다.

---

## 9. 동기화 상태

사용자는 현재 VaultDatum의 상태를 쉽게 확인할 수 있어야 한다.

최소 다음 사용자 상태를 표현한다.

```text
Setup Required
First Sync
Up to Date
Syncing
  └─ 필요하면 Uploading / Downloading 단계를 보조로 설명
Paused
Offline
Pending Changes
Conflict
Error
Recovery Required
```

Obsidian의 Status Bar 또는 전용 Sync 화면에서 다음 정보를 확인할 수 있어야 한다.

- 서버 연결 상태
- 동기화 여부
- 업로드 대기 변경
- 다운로드 대기 변경
- 충돌 여부
- 마지막 성공 동기화 시각

일반적인 사용 중에는 사용자가 Sync 화면을 지속적으로 확인할 필요가 없어야 한다.

Client가 이 상태를 어떻게 설명하고, 첫 연결·오류·복구를 어떤 화면과 Action으로
제공하는지는 [12 Client User Experience](./12_client-user-experience.md)를 따른다.

---

## 10. 자동 동기화

VaultDatum는 기본적으로 사용자의 수동 조작 없이 동작한다.

다음 상황에서 자동으로 동기화를 시도한다.

- 로컬 파일이 변경됨
- 서버의 새로운 변경을 감지함
- 네트워크가 복구됨
- Obsidian이 다시 실행됨
- 일정 시간 동안 동기화되지 않음

사용자는 필요할 경우 **Sync Now** 명령을 통해 즉시 동기화를 요청할 수 있어야 한다.

수동 요청은 정상 동기화의 전제 조건이 아니다. 연결 설정과 상태 표시의 사용자
경험은 [12 Client User Experience](./12_client-user-experience.md)를 따른다.

---

## 11. 데이터 안전성

VaultDatum에서 데이터 유실 방지는 동기화 속도보다 우선한다.

다음 조건을 만족해야 한다.

### 로컬 변경 보존

서버가 변경을 정상적으로 받아들였다는 사실이 확인되기 전에 로컬 변경을 완료된 것으로 처리해서는 안 된다.

### 서버 확정

클라이언트에서 발생한 변경은 VaultDatum Server가 정상적으로 받아들여 Authoritative Vault에 반영한 이후에 전체 시스템의 확정된 변경으로 간주한다.

### 재시도 안전성

네트워크 Timeout 등으로 동일한 변경이 여러 번 전송되더라도 같은 변경이 중복 적용되어서는 안 된다.

### 부분 동기화 복구

동기화 도중 앱이나 서버가 종료되어도 다음 실행에서 중단된 상태를 식별하고 복구할 수 있어야 한다.

### 삭제 보존

삭제 사실 역시 하나의 변경이다.

오프라인 클라이언트가 나중에 돌아왔을 때 삭제 여부를 판단할 수 있을 만큼 충분히 보존되어야 한다.

### 서버 장애

VaultDatum Server가 재시작되더라도 이미 확정된 Vault 상태와 동기화 상태가 손실되어서는 안 된다.

---

## 12. 보안 요구사항

VaultDatum는 Vault 전체에 접근할 수 있으므로 승인되지 않은 사용자가 VaultDatum Server에 접근할 수 없어야 한다.

다만 VaultDatum가 모든 네트워크 보안 기능을 자체적으로 구현해야 하는 것은 아니다.

### 12.1 보안 계층 분리

VaultDatum는 네트워크 보안과 Vault 동기화를 별개의 책임으로 취급한다.

VaultDatum Server는 다음과 같은 환경에서 운영할 수 있다.

```text
Internet
   │
   X
   │
WireGuard / VPN
   │
   ▼
Private Network
   │
   ├── VaultDatum Server
   ├── Desktop
   └── Mobile
```

WireGuard와 같은 VPN을 사용하는 경우 다음 기능을 외부 네트워크 계층에 위임할 수 있다.

- 외부 네트워크에서 VaultDatum Server 접근 차단
- 장치 인증
- 장치별 접근 폐기
- 통신 암호화
- 서버와 클라이언트 사이의 사설 네트워크 구성

### 12.2 VPN 사용

VPN 사용은 VaultDatum의 필수 요구사항이 아니다.

사용자는 자신의 환경에 따라 다음과 같은 배포 모델을 선택할 수 있다.

**VPN 기반**

```text
Client
   │
WireGuard
   │
Private Network
   │
VaultDatum Server
```

네트워크 접근 제어와 통신 보호를 VPN에 위임한다.

**VaultDatum 직접 보호**

```text
Client
   │
Secure Transport
   │
Authentication
   │
VaultDatum Server
```

VPN을 사용하지 않는 환경에서는 VaultDatum 또는 VaultDatum 앞단의 별도 구성요소가 적절한 통신 보호와 접근 통제를 제공해야 한다. 개인용 public
profile은 HTTPS와 운영자가 provision한 Vault bearer token을 사용한다. 이것은 여러
사용자 계정 또는 공개 가입 기능을 뜻하지 않는다.

구체적인 인증 방식, TLS 구성, VPN 구성은 제품 사양이 아닌 배포 및 아키텍처 문서에서 정의한다.

### 12.3 접근 범위

네트워크 보안 방식과 관계없이 VaultDatum를 통해 Vault Root 외부의 서버 파일에 접근할 수 없어야 한다.

### 12.4 최소 권한

VaultDatum Server는 Vault 동기화에 필요하지 않은 시스템 자원에 대한 권한을 요구하지 않아야 한다.

---

## 13. 장애 상황

VaultDatum는 다음 상황을 정상적인 동작 환경의 일부로 간주한다.

- 인터넷 단절
- Wi-Fi ↔ 모바일 네트워크 전환
- VPN 연결 단절 및 복구
- 실시간 연결 끊김
- HTTP Timeout
- 모바일 앱 강제 종료
- OS에 의한 Background Process 종료
- PC 절전
- 클라이언트 Crash
- 서버 재시작
- 동일 요청 재전송
- 장기간 오프라인 상태

이러한 사건 자체가 Vault 손상이나 파일 유실을 발생시켜서는 안 된다.

---

## 14. 백업과 동기화

동기화는 백업이 아니다.

VaultDatum의 목적은 여러 장치가 Authoritative Vault의 현재 상태로 수렴하게 만드는 것이다. 따라서 사용자의 실수나 잘못된 변경도 모든 장치에 동기화될 수 있다.

잘못 바꾸거나 지운 파일은 파일 기록에서 되돌린다. 서버 버그나 저장소 문제로 Vault나 Sync State 자체가 망가지는 경우에 대비해, 서버는 **마지막 정상 상태의 Backup 하나**를 유지하고 그 Backup으로 서버 전체를 복원할 수 있다. 여러 시점의 Backup을 보관하지 않는다.

Backup은 서버와 같은 저장소에 있다. 디스크 자체의 손실에 대비해 Backup을 다른 장치에 보관하는 것은 제품 범위에 포함하지 않으며 운영자가 맡는다.

---

## 15. 대상 사용자

VaultDatum의 대상은 다음 사용자이다.

> 한 명의 사용자가 개인 홈서버를 운영하면서 데스크탑, 노트북, 모바일 등의 여러 장치에서 하나의 Obsidian Vault를 사용하고 싶은 경우.

VaultDatum는 조직용 협업 도구를 목표로 하지 않는다.

---

## 16. 제품 기능

VaultDatum는 다음 기능을 제공한다. 각 기능의 세부 규칙은 기준 문서를 따른다.

**구성요소와 플랫폼**

- VaultDatum Server
- VaultDatum for Obsidian (Desktop, Android)

**동기화**

- 파일과 빈 디렉터리의 Create, Modify, Delete, Rename, Move
- Markdown, 이미지, PDF 등 모든 일반 파일
- 자동 동기화와 수동 동기화
- 오프라인 변경 보존과 재연결 후 Catch-up
- Change Journal 기반 증분 동기화와 Manifest 기반 전체 재조정
- 실시간 변경 알림 (정확성에는 필요하지 않음)
- `.obsidian/` 제외

**충돌**

- 수정, 삭제, Rename/Move 충돌 감지와 양쪽 데이터 보존
- 서버 버전 사용, 로컬 버전 적용, 삭제 유지, 로컬 복원, 두 파일 모두 보존, Markdown 수동 병합

**안전성과 복구**

- 재시도해도 중복 적용되지 않는 변경
- 서버 장애 중 부분 적용 방지와 재시작 복구
- 서버 Vault 무결성 검사와 Drift 보고
- 기존 Vault를 새 서버로 옮기는 일회성 초기 가져오기
- 파일 기록 조회와 단일 파일 되돌리기 (삭제된 파일 복원 포함)
- 마지막 정상 상태의 서버 Backup 하나와 전체 복원, 복원된 서버에 장치 다시 연결

**사용자 경험**

- 동기화 상태, 마지막 동기화 시각, Pending/Conflict 수 표시
- 연결 확인, 일시 정지, 동기화 추적 초기화, 진단 정보 복사

**배포와 접근**

- VPN 등 사설 네트워크 배포
- HTTPS와 운영자가 발급한 Vault token을 사용하는 개인용 public 배포

---

## 17. 비목표 (Non-goals)

VaultDatum는 다음을 목표로 하지 않는다.

- 여러 사용자의 공동 편집
- Google Docs 수준의 실시간 동시 편집
- Character 단위 실시간 Merge
- CRDT 기반 협업
- 서버 Cluster 및 High Availability
- 여러 VaultDatum Server 간 Federation
- 범용 원격 파일시스템
- Git Client 대체
- Obsidian 이외 애플리케이션의 범용 파일 동기화
- `.obsidian/` 동기화
- Obsidian Plugin 설정 동기화
- AI Agent

AI Agent는 VaultDatum와 독립적인 별도 제품 또는 확장 기능으로 다룬다.

---

## 18. 핵심 제품 불변조건

VaultDatum 구현 방식과 관계없이 다음 조건은 항상 만족해야 한다.

### Invariant 1 — Server Authority

VaultDatum Server의 Vault가 전체 시스템의 유일한 Authoritative Vault이다.

클라이언트에서 발생한 변경은 서버가 받아들인 이후에만 전체 시스템의 확정된 상태가 된다.

### Invariant 2 — Client Convergence

정상적으로 동기화가 완료된 모든 클라이언트는 VaultDatum Server의 동기화 대상 Vault와 동일한 상태에 도달해야 한다.

`.obsidian/` 등 명시적으로 제외된 로컬 상태는 이 조건에 포함하지 않는다.

### Invariant 3 — Offline Durability

오프라인 상태에서 사용자가 저장한 변경은 앱 종료나 장치 재부팅 때문에 사라져서는 안 된다.

### Invariant 4 — Recoverability

실시간 변경 알림을 하나도 받지 못하더라도 클라이언트는 서버와 다시 통신하여 최신 상태로 복구할 수 있어야 한다.

### Invariant 5 — No Silent Overwrite

서버와 로컬이 서로 다른 변경을 가지고 있는 상황을 인지하지 못한 채 한쪽 데이터가 조용히 덮어써져서는 안 된다.

### Invariant 6 — Retry Safety

이미 성공한 변경을 다시 전송해도 동일 변경이 중복 적용되어서는 안 된다.

### Invariant 7 — Deletion Is State

파일이 존재하지 않는다는 사실도 동기화해야 하는 상태로 취급해야 한다.

### Invariant 8 — Local Usability

VaultDatum Server가 일시적으로 사용할 수 없더라도 사용자는 자신의 로컬 Obsidian Vault를 계속 읽고 수정할 수 있어야 한다.

### Invariant 9 — Client-local Configuration

`.obsidian/`을 비롯한 VaultDatum의 동기화 제외 영역은 클라이언트별 로컬 상태로 유지되어야 한다.

VaultDatum가 해당 상태를 다른 클라이언트의 상태로 덮어써서는 안 된다.

### Invariant 10 — Public Access Requires a Vault Token

public deployment의 요청은 HTTPS/WSS와 유효한 Vault token을 통해서만
Authoritative Vault에 접근할 수 있어야 한다. token은 동기화되는 콘텐츠나 URL에
포함되어서는 안 된다. 세부 규칙은 [09 Security and
Deployment](./09_security-and-deployment.md)를 따른다.

---

## 19. 제품 성공 기준

VaultDatum의 첫 번째 성공 기준은 많은 기능을 제공하는 것이 아니다.

다음 상황을 사용자가 걱정하지 않아도 되는 상태가 목표다.

> 데스크탑에서 작성하고 휴대폰에서 이어서 작성한다.

> 지하철에서 오프라인으로 수정하고 나중에 자동으로 반영된다.

> 며칠 사용하지 않은 장치를 켜도 다시 서버의 최신 상태가 된다.

> 앱이나 서버가 중간에 종료되어도 작성한 문서가 사라지지 않는다.

> 두 장치에서 동시에 수정했을 때 어느 한쪽 내용이 몰래 사라지지 않는다.

> 모바일과 데스크탑이 서로 다른 Obsidian 플러그인 구성을 자유롭게 유지한다.

> 홈서버를 VPN 안에 두었다면 VaultDatum 자체에 불필요한 보안 계층을 중복해서 구성하지 않아도 된다.

VaultDatum의 핵심 가치는 **빠른 동기화보다 신뢰할 수 있는 동기화**이다.
