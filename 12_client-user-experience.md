# VaultDatum Client User Experience

> 상태: `0.3.0`을 위한 설계 초안이며, `0.3.1`의 Markdown 수동 병합 작업 공간을
> 포함한다. 이 문서는 Obsidian Client가 사용자에게 동기화 상태와 복구 동작을
> 어떻게 보여 주는지를 정의한다.

## 1. 문서 목적

VaultDatum의 동기화 규칙은 안전하지만, 사용자는 revision, cursor, manifest,
IndexedDB를 이해하지 않아도 자신의 문서가 안전하게 동기화되는지 알 수 있어야
한다.

이 문서는 다음 사용자 여정을 하나의 흐름으로 정의한다.

```text
연결 설정
    → 첫 동기화
    → 일상적인 자동 동기화
    → 주의가 필요한 상태
    → 진단과 안전한 복구
```

동기화의 authoritative state, conflict 판정, persistence와 HTTP 계약은 여기서
재정의하지 않는다. 각각 다음 문서가 단일 기준이다.

| 관심사                                  | 기준 문서                                                                                                                  |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 제품 원칙과 동기화 상태의 의미          | [00 Product Specification](./00_product-specification.md), [02 Synchronization Protocol](./02_synchronization-protocol.md) |
| Client lifecycle 및 scheduler           | [04 Client Architecture](./04_client-architecture.md)                                                                      |
| Conflict의 상세 화면과 resolution       | [05 Conflict Resolution](./05_conflict-resolution.md)                                                                      |
| Client state와 reset의 persistence 제약 | [08 Persistence Design](./08_persistence-design.md)                                                                        |
| 진단 정보와 운영 복구                   | [11 Observability and Operations](./11_observability-and-operations.md)                                                    |

이 문서의 UI 상태는 protocol의 durable state가 아니다. 여러 protocol 상태를
사용자가 이해할 수 있는 하나의 화면 상태와 다음 행동으로 투영한 것이다.

---

## 2. UX 목표와 경계

### 2.1 사용자가 해야 할 일은 최소여야 한다

일반적인 사용자는 Server URL을 연결한 뒤 노트를 평소처럼 편집한다. 다음은
사용자 행동의 전제 조건이 되어서는 안 된다.

```text
매번 Sync now 누르기

Sync 화면을 열어 둔 채 진행 상황 감시하기

앱을 종료하기 전에 수동으로 flush하기
```

자동 동기화가 성공했을 때는 조용해야 한다. 반대로 사용자의 판단이 필요한
상태는 놓치지 않게 보여야 한다.

### 2.2 안전성은 간단한 문구보다 우선한다

UX는 다음을 단순화해서는 안 된다.

```text
서버가 유일한 authoritative Vault라는 사실

서로 다른 두 내용을 자동으로 덮어쓰지 않는다는 원칙

삭제도 동기화 상태라는 사실

Pending 또는 conflict를 지우는 복구가 위험할 수 있다는 사실
```

따라서 `Upload all`, `Force sync`, `Overwrite server`, `Reset everything`처럼
결과를 숨기는 기본 Action을 제공하지 않는다. 데이터에 영향을 주는 Action은
무엇을 보존하고 무엇을 다시 분류하는지 먼저 설명해야 한다.

### 2.3 Mobile과 Desktop은 같은 결과를 제공한다

Desktop Status Bar는 빠른 진입점일 뿐이다. 설정, 첫 동기화 결과, conflict,
진단과 복구는 Android를 포함한 모든 지원 플랫폼에서 사용할 수 있어야 한다.

모바일의 background 실행은 보장하지 않는다. 화면은 이를 숨기거나 "항상
백그라운드에서 동기화됨"이라고 약속해서는 안 된다. 앱이 foreground가 되었을
때 확인하고 동기화한다는 사실을 상태 설명에 반영한다.

### 2.4 기술 세부 사항은 기본 화면에서 숨긴다

기본 화면에는 `revision`, `cursor`, `manifest`, `operation ID`, HTTP status,
IndexedDB 같은 용어를 노출하지 않는다. 사용자가 문제를 진단할 때에만 별도의
상세 정보와 복사 가능한 진단 정보를 제공한다.

파일 이름도 사적인 정보일 수 있다. 기본 알림에는 파일 내용을 절대 넣지 않고,
경로 목록은 사용자가 상세 화면을 열었을 때에만 표시한다.

### 2.5 접근성과 언어는 기능의 일부다

상태는 색만으로 구분하지 않는다. 아이콘, 짧은 텍스트, 스크린 리더용 설명을
함께 제공한다. 모든 Action은 키보드와 터치로 실행할 수 있어야 하며, 긴 번역문
또는 작은 모바일 화면에서 잘리지 않아야 한다. 사용자에게 보이는 문구는
번역 가능한 한 곳의 문자열 집합으로 관리한다.

---

## 3. 정보 구조와 진입점

Client는 하나의 **VaultDatum Sync Overview**를 동기화 정보의 기준 화면으로
제공한다. 구현은 Obsidian의 설정 탭, 전용 view, modal 중 플랫폼에 알맞은
형태를 선택할 수 있지만, 같은 정보와 Action을 중복된 여러 화면에 다르게
구현해서는 안 된다.

### 3.1 진입점

| 진입점             | 목적                                                  | 지원 플랫폼     |
| ------------------ | ----------------------------------------------------- | --------------- |
| Desktop Status Bar | 현재 상태를 한눈에 보이고 Overview 열기               | Desktop         |
| VaultDatum 설정    | 연결 설정과 Overview의 항상 사용 가능한 진입점        | Desktop, Mobile |
| Command Palette    | `Sync now`, `Open sync overview`, `Resolve conflicts` | Desktop, Mobile |
| 주의 알림          | 새 conflict 또는 지속 오류에서 해당 Overview 열기     | Desktop, Mobile |

Status Bar를 클릭했다고 새 동기화를 무조건 시작하지 않는다. 사용자는 먼저
현재 상태와 마지막 결과를 확인할 수 있어야 한다. 즉시 확인은 Overview의
명시적인 `Sync now` Action 또는 Command Palette 명령으로 수행한다.

### 3.2 Overview의 고정 구성

Overview는 상태가 무엇이든 다음 순서를 유지한다.

```text
1. 상태 요약과 다음 행동
2. 마지막 동기화 결과와 대기 작업 수
3. 상태별 세부 정보
4. 일반 Action
5. Advanced diagnostics and recovery
```

항상 표시하는 최소 정보는 다음과 같다.

| 항목               | 표시 원칙                                                  |
| ------------------ | ---------------------------------------------------------- |
| 현재 상태          | 결과 중심의 짧은 문장과 보조 설명                          |
| 마지막 성공 동기화 | 상대 시간과 접근 가능한 절대 시간                          |
| 대기 변경 수       | 알 수 있을 때만 개수; 정확하지 않은 진행률을 추측하지 않음 |
| conflict 수        | 0이 아니면 항상 눈에 띄는 Action과 함께 표시               |
| 연결 대상          | Server URL의 안전한 표시와 Vault identity 확인 결과        |
| 다음 행동          | 현재 상태에서 가장 안전한 한 가지 Action을 우선 제시       |

`Up to date`는 Pending과 active conflict가 모두 없을 때만 사용할 수 있다.
Cursor가 같다는 사실만으로 이 문구를 표시해서는 안 된다.

---

## 4. 사용자에게 보이는 상태

### 4.1 표시 상태와 우선순위

동시에 여러 내부 상태가 존재할 수 있다. 화면은 다음 우선순위로 하나의 주
상태를 정하고, 나머지는 보조 정보로 보여 준다.

| 우선순위 | 표시 상태           | 사용자 문구 예                                                    | 기본 Action        |
| -------- | ------------------- | ----------------------------------------------------------------- | ------------------ |
| 1        | Recovery required   | `동기화 정보를 안전하게 확인할 수 없습니다`                       | `Open recovery`    |
| 2        | Conflict            | `3개의 변경에 확인이 필요합니다`                                  | `Review conflicts` |
| 3        | Setup required      | `서버를 연결하면 동기화가 시작됩니다`                             | `Connect server`   |
| 4        | First sync          | `첫 동기화를 준비하고 있습니다`                                   | `View progress`    |
| 5        | Syncing             | `변경 사항을 확인하고 있습니다`                                   | `View progress`    |
| 6        | Paused              | `동기화가 일시 중지되었습니다`                                    | `Resume sync`      |
| 7        | Offline or retrying | `서버에 연결할 수 없습니다. 변경은 이 기기에 안전하게 보관됩니다` | `Retry now`        |
| 8        | Pending             | `2개의 로컬 변경이 동기화를 기다리고 있습니다`                    | `Sync now`         |
| 9        | Up to date          | `모든 변경 사항이 동기화되었습니다`                               | `Sync now`         |

여기서 `Paused`는 사용자가 명시적으로 일시 중지한 경우에만 사용한다. network
failure를 `Paused`라고 표시하면 안 된다.

Recovery required와 conflict는 다른 정상 경로의 동기화가 계속 진행될 수 있어도
숨기지 않는다. 예를 들어 한 파일의 conflict가 남아 있으면 `Up to date` 대신
`1개의 변경에 확인이 필요합니다`를 주 상태로 표시한다.

### 4.2 동기화 중의 단계

진행률 percentage는 전체 작업량을 신뢰성 있게 알 수 있을 때만 표시한다. 그
외에는 다음처럼 의미 있는 단계만 표시한다.

```text
서버에 연결하는 중

서버 변경 사항을 확인하는 중

이 기기의 변경 사항을 보내는 중

결과를 확인하는 중
```

첫 동기화에는 `서버의 현재 Vault를 확인하는 중`과 `이 기기의 기존 파일을
안전하게 분류하는 중`을 구분해 보여 준다. 이 단계는 사용자에게 서버 우선
원칙을 설명할 기회이며, 단순한 spinner로 감추지 않는다.

### 4.3 성공, 오류와 알림의 빈도

다음 상황에서는 방해성 알림을 표시하지 않는다.

```text
정상적인 자동 동기화 시작과 완료

동일한 retryable network failure의 반복

동일한 상태의 단순 갱신
```

다음 전이에서만 눈에 띄는 알림을 사용한다.

```text
첫 동기화가 완료됨

새 conflict가 발생했거나 conflict 수가 증가함

자동 retry 후에도 사용자의 확인이 필요한 오류가 지속됨

Recovery required가 됨
```

알림은 짧은 결과와 Overview로 이동하는 Action을 제공한다. 성공한 파일의
내용, access token, authorization header는 어떤 알림이나 기본 log에도 포함하지
않는다.

---

## 5. 연결 설정과 Server 확인

### 5.1 설정 화면

Server URL은 입력 중인 값과 저장된 값을 구분한다. 현재 문자열이 바뀔 때마다
network 요청을 보내거나 error 상태로 바꾸지 않는다.

설정 화면은 최소 다음을 제공한다.

```text
Server URL field

Vault access token field (optional; required by a public Vault)

Test connection

Save and start sync

현재 연결 대상과 Vault identity
```

`Test connection`은 Server health와 Vault 정보를 확인할 수 있지만 Local Vault를
변경하거나 operation을 전송하지 않는다. public Server가 Vault token을 요구하면 Test는
입력된 token으로만 요청하며 token value를 notice나 diagnostic에 되돌려 보여 주지
않는다. `Save and start sync`는 URL과 필요한 Vault token을 durable하게 저장하고 normal
scheduler를 깨운다. 서버가 일시적으로 닿지 않아도 URL은 저장할 수 있어야 하며, 이
경우 상태는 `Offline or retrying`으로 명확히 표시한다.

`저장됨`과 `연결됨`은 같은 상태가 아니다. URL이 저장된 뒤에도 Server가 닿지
않을 수 있으므로 화면은 최소 다음을 구분한다.

| 표시 | 의미 |
| --- | --- |
| `Saved` | URL이 Client 설정에 durable하게 저장됨 |
| `Connected` | 마지막 확인에서 Server와 Vault identity를 읽음 |
| `Saved — server unavailable` | URL은 저장됐지만 현재 Server 확인에는 실패함 |
| `Unverified` | URL을 저장했지만 아직 성공적인 Server 확인이 없음 |

저장된 URL과 정규화한 입력값이 같으면 저장 Action은 `Saved`로 비활성화한다.
유효한 다른 URL을 입력했을 때만 다시 `Save and start sync`를 활성화한다. 저장
요청이 진행 중이면 중복 요청을 막고 `Saving…` 상태를 보여 준다.

### 5.2 Public Vault token

`public-token` Server는 Vault access token 없이는 연결할 수 없다. Client는 private
network 설치에서도 같은 설정 화면을 쓸 수 있도록 다음 field를 항상 제공하되, token이
없는 Server에는 비워 둘 수 있게 한다. Server가 `401` Bearer challenge를 반환하면 field를
강조하고 `Authentication required` 상태를 표시한다.

```text
Vault access token
••••••••••••••••
```

field는 기본적으로 mask하고, 사용자가 누르는 동안만 값을 보이는 reveal control을
제공한다. token을 URL, QR code가 아닌 화면 screenshot, status bar, notification,
diagnostic에 넣지 않는다. Client는 token을 해당 Obsidian Vault의 plugin-local settings에만
보관하며 동기화하지 않는다.

사용자에게 다음 경계를 설명한다.

> 이 token을 가진 사람은 이 Vault에 접근할 수 있습니다. 기기를 잃었거나 token이
> 노출되었다고 의심되면 Server 관리자에게 Vault token 교체를 요청하세요. 교체 뒤에는
> 모든 연결 장치에 새 token을 다시 입력해야 합니다.

token이 없는 public Server는 `Authentication required` 상태로 표시하고, 자동 retry를
계속하지 않는다. 사용자가 유효한 token을 저장하면 scheduler를 즉시 다시 시작한다.

#### 최초 연결 절차

`0.4.0`은 사용자 가입이나 로그인 화면을 제공하지 않는다. Vault 소유자(운영자)가
배포 시 Vault token을 생성하고 Secret으로 Server에 mount한 뒤, URL과 token을 신뢰할 수
있는 비공개 채널(예: password manager 또는 end-to-end encrypted messenger)로 장치
사용자에게 전달한다. token을 URL query나 QR code에 넣지 않는다.

장치 사용자는 다음 순서로 최초 연결한다.

1. VaultDatum settings에서 public `https://` Server URL을 입력한다.
2. `Test connection`이 `Authentication required`를 표시하면 `Vault access token` field에 받은 token을 붙여 넣는다. token을 미리 받았다면 Test 전에 입력해도 된다.
3. 다시 `Test connection`을 눌러 Vault identity를 확인한다. 이 단계는 파일을 올리거나 내려받지 않는다.
4. `Save and start sync`를 누른다. URL과 token은 해당 Obsidian Vault의 plugin-local settings에만 저장되고 첫 동기화가 시작된다.

device 분실이나 token 노출 뒤 Server 관리자가 token을 교체하면, 각 장치는 다시
`Authentication required`가 된다. 사용자는 새 token을 입력하고 저장하면 되며, 기존
Pending Operation과 Local file을 지우거나 Vault를 다시 만들 필요는 없다.

### 5.3 URL과 private network 안내

Client는 명시적인 `https://` 또는 `http://` URL만 받아들인다. `http://`를
기술적으로 막지는 않는다. WireGuard 같은 보호된 private network에서 운영하는
설치를 지원해야 하기 때문이다.

다만 URL이 `http://`이면 다음 안내를 보여 준다.

> HTTP 연결은 보호된 private network에서만 사용하세요. 공개 네트워크에서는
> HTTPS와 적절한 접근 제어가 필요합니다.

이 안내는 private VPN 운영을 오류로 취급하지 않으며, URL이 localhost인지
public address인지로 네트워크 보안을 잘못 판단하지 않는다.

### 5.4 다른 Server Vault로의 변경

이미 Vault identity가 기록된 Client가 다른 Vault identity를 반환하는 URL을
입력하면, URL만 바꾸어 기존 sync state를 재사용해서는 안 된다.

화면은 다음을 설명해야 한다.

```text
이 서버는 현재 이 기기에 연결된 Vault와 다릅니다.

기존 동기화 정보를 유지한 채 서버를 바꾸면 변경을 안전하게 비교할 수 없습니다.
```

사용자는 이전 설정으로 돌아가거나, Advanced recovery의 보호된 절차를 통해
새로운 Client state를 시작할 수 있다. 자동으로 다른 Server를 신뢰하거나
기존 Pending을 전송하지 않는다.

HTTPS public URL의 origin이 달라지는 경우 기존 Vault access token을 새 URL로 자동
전달해서는 안 된다. 새 origin에는 token을 다시 명시적으로 입력하고 Test connection을
통과해야 한다.

---

## 6. 첫 동기화와 기존 Local Vault

### 6.1 시작 전 설명

처음 Server URL을 연결하면 Client는 자동 동기화를 시작한다. 시작 화면은 다음
세 가지를 짧게 설명한다.

```text
서버 Vault가 기준 상태입니다.

서버의 현재 파일을 먼저 확인합니다.

이 기기에만 있고 서버와 충돌하지 않는 파일만 동기화 후보가 됩니다.
```

사용자에게 `Download all` 또는 `Upload all` 중 하나를 고르게 하지 않는다. 두
선택지는 같은 경로의 서로 다른 Local 파일과 server tombstone을 안전하게
표현하지 못한다.

### 6.2 진행과 결과

Server-first Bootstrap이 끝나면 Overview는 다음 범주의 결과를 보여 준다.

| 범주              | 의미                                                 | 사용자에게 보이는 결과              |
| ----------------- | ---------------------------------------------------- | ----------------------------------- |
| 서버 파일 적용    | Local에 없던 Server content                          | Local Vault에 추가됨                |
| 이미 일치         | 같은 경로와 같은 content                             | 변경 없음                           |
| 이 기기 전용 항목 | Server가 `UNKNOWN`으로 확인한 항목                   | 안전한 업로드 대기 또는 업로드 완료 |
| 확인 필요         | 같은 경로의 다른 content, tombstone 또는 타입 불일치 | conflict로 보존됨                   |
| 제외됨            | `.obsidian/`, 지원하지 않는 경로 또는 크기 제한 초과 | 동기화하지 않았음과 이유            |

일반 화면에는 범주별 개수만 표시한다. 사용자가 상세 보기를 요청했을 때에만
경로 목록을 제공한다. 파일 내용을 미리 보기로 노출하지 않는다.

첫 동기화가 끝난 직후에는 일반적인 마지막 결과와 구별해 `First sync result`를
표시한다. 적어도 Server Vault를 먼저 확인했다는 사실, Server가 받아들인 local
변경 수, 남은 conflict 수와 제외된 파일 수를 함께 요약한다.

첫 동기화 완료는 "모든 파일이 즉시 같아졌다"는 뜻이 아니다. 안전하게 분류된
Pending이 남아 있으면 Overview는 이를 분명히 표시하고 scheduler가 정상적인
push와 확인을 계속하도록 한다.

### 6.3 중단과 재개

앱 종료, mobile suspend, network 단절 또는 manifest 만료로 첫 동기화가
중단되면 완료로 표시하지 않는다. 다음 실행 또는 연결 복구 시 새 Server
manifest로 안전하게 다시 시작한다.

사용자에게는 다음처럼 설명한다.

> 첫 동기화가 아직 완료되지 않았습니다. 연결되면 서버 상태를 다시 확인한 뒤
> 계속합니다.

중단 때문에 사용자가 다시 URL을 입력하거나 파일을 수동 복사할 필요는 없다.
`Sync now`를 누른 경우에도 실행 중인 Bootstrap에 합류할 뿐, 병렬 Bootstrap을
만들지 않는다.

---

## 7. 일상적인 자동 동기화

### 7.1 자동 Trigger의 사용자 의미

Local 변경, plugin 시작, foreground 복귀, network 복구, Server notification과
retry timer는 scheduler의 Trigger다. 사용자는 이를 개별 설정으로 관리할 필요가
없다.

Overview에는 다음 한 문장으로 설명한다.

> 변경 사항은 자동으로 동기화됩니다. 오프라인에서 만든 변경은 연결되면
> 안전하게 계속됩니다.

`Sync now`는 자동 동기화가 꺼져 있거나 실패했다는 뜻이 아니다. 사용자가 지금
즉시 Server 상태를 확인하고 싶을 때의 수동 Trigger다.

### 7.2 일반 Action

| Action          | 사용자에게 설명할 결과                                       | 안전성 규칙                                              |
| --------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| Sync now        | 지금 Server 변경을 확인하고 로컬 Pending을 처리              | 기존 sync cycle과 동일한 base validation을 사용          |
| Pause sync      | network sync만 일시 중지; Local 변경은 계속 durable하게 기록 | Local Vault를 읽거나 쓰지 않으며 Pending을 삭제하지 않음 |
| Resume sync     | 보류한 scheduler를 즉시 다시 시작                            | 정상 pull → push → verification 순서를 사용              |
| Check all files | Server manifest와 Local 상태를 다시 비교                     | Local content를 무조건 삭제하거나 Server로 덮어쓰지 않음 |

주 화면에서는 `Sync now`만 보조 Action으로 제공한다. `Check all files`는
Advanced recovery에 두고, 시간이 걸릴 수 있지만 Local 변경을 버리지 않는다는
설명을 함께 표시한다. 내부 명령 이름인 `Full reconciliation`은 Command Palette
호환성을 위해 남길 수 있으나 기본 UI 문구로 쓰지 않는다.

`Pause sync`는 명시적인 사용자 선택일 때만 제공한다. plugin 비활성화, 앱 종료,
network failure를 사용자가 선택한 pause와 혼동하지 않는다.

Overview의 주 Action은 현재 상태에 따라 하나만 우선 표시한다.

| 상태 | 주 Action |
| --- | --- |
| Setup required 또는 Vault mismatch | `Connect server` 또는 `Review connection` |
| First sync 또는 Syncing | `Syncing…` (비활성화) |
| Paused | `Resume sync` |
| Offline 또는 retryable error | `Retry now` |
| Conflict | `Review conflicts` |
| Pending 또는 Up to date | `Sync now` |

`Review conflicts`는 conflict 수가 0일 때 비활성화할 수 있다. 동기화 중의
`Sync now`는 별도 병렬 작업을 만든다는 인상을 주지 않도록 `Syncing…`으로
표시하고 비활성화한다.

### 7.3 마지막 결과

일상 상태에서는 다음 정도면 충분하다.

```text
모든 변경 사항이 동기화되었습니다.
마지막 성공: 2분 전
```

Pending이면 대기 수와 이유를 보인다.

```text
3개의 로컬 변경이 연결을 기다리고 있습니다.
마지막 시도: 방금 전
```

서버 revision이나 cursor는 기본 화면의 성공 여부를 판단하는 값으로 쓰지
않는다. 필요할 때 진단 영역에서만 제공한다.

---

## 8. Offline, 오류와 사용자의 다음 행동

### 8.1 Offline은 오류가 아니다

Server에 연결할 수 없을 때 Client는 Local Vault를 계속 사용할 수 있다. 화면은
다음 두 사실을 함께 전달한다.

```text
현재 서버에 연결할 수 없습니다.

이 기기의 변경은 안전하게 보관되며 연결되면 자동으로 다시 시도합니다.
```

VPN이 사용되는 설치에서는 VPN 연결을 확인하라는 추천 Action을 제시할 수
있다. Client가 VPN 상태를 실제로 검사할 수 없다면 `VPN이 꺼져 있습니다`라고
단정하지 않는다.

### 8.2 오류 분류

사용자에게 보여 주는 오류는 원인과 다음 행동을 묶는다.

| 분류                 | 사용자 문구 예                                        | 기본 Action                  |
| -------------------- | ----------------------------------------------------- | ---------------------------- |
| URL 형식 오류        | `서버 주소를 확인하세요`                              | `Edit server URL`            |
| 연결 불가            | `서버에 연결할 수 없습니다`                           | `Retry now`                  |
| Server 응답 오류     | `서버가 동기화를 처리할 수 없습니다`                  | `Retry now`와 진단 보기      |
| 지원하지 않는 Server | `이 Server 버전과 호환되지 않습니다`                  | `View compatibility details` |
| 인증 필요            | `이 Vault access token을 입력하세요`                  | `Enter access token`         |
| 인증 거부 또는 token 교체 | `이 Vault access token을 업데이트하세요`             | `Update access token`        |
| local recovery 문제  | `이 기기의 동기화 상태를 안전하게 확인할 수 없습니다` | `Open recovery`              |

raw error code와 stack trace는 기본 문구로 사용하지 않는다. 원문은 사용자가
`Copy diagnostic details`를 선택했을 때에만 포함할 수 있다.

### 8.3 Retry 표시

retryable failure는 scheduler가 정한 backoff로 재시도한다. 화면은 다음 retry가
예약되었음을 알려 줄 수 있지만, mobile background에서 정확한 시각에 실행된다고
약속해서는 안 된다.

```text
다음 연결 기회에 자동으로 다시 시도합니다.
```

사용자가 `Retry now`를 선택하면 기존 실행과 병렬 요청을 만들지 않고 scheduler에
하나의 follow-up trigger를 전달한다.

---

## 9. Conflict와 사용자 주의 상태

Conflict는 일반 오류나 단순한 "동기화 실패"가 아니다. 두 버전이 모두 보존된
상태이며 사용자 판단이 필요하다는 것을 먼저 설명한다.

Overview는 다음을 제공한다.

```text
확인이 필요한 변경 수

영향을 받는 경로의 목록으로 가는 진입점

다른 경로는 계속 동기화되고 있다는 설명

Conflict Center 열기
```

Conflict Center의 목록, 서버/이 기기 버전 비교, `서버 버전 사용`, `이 기기의
버전을 서버에 적용`, `두 버전 모두 유지`, manual merge의 정확한 의미는
[05 Conflict Resolution](./05_conflict-resolution.md)을 따른다.

Conflict Center는 Command Palette가 없어도 완결된 경로여야 한다. 목록은 경로와
사람이 이해할 수 있는 짧은 이유를 보이고, 선택한 conflict의 실제 상태에 맞는
해소 방법만 보여 준다. 예를 들어 양쪽 파일이 있으면 `서버 버전 사용`, `이
기기의 버전 적용`, `둘 다 유지`, Markdown의 경우 `수동 병합`을 제공한다. local
삭제 또는 Server 삭제 conflict에는 각각 삭제 유지 또는 복원 Action을 제공한다.
해소 Action 뒤에는 남은 conflict 목록과 count로 돌아갈 수 있어야 한다. 해소
operation이 Server에 확정되기 전에는 성공으로 단정하지 않는다.

Overview와 알림에서 다음 표현을 사용하지 않는다.

```text
412 conflict

base revision mismatch

last write wins
```

사용자가 conflict Action을 선택한 직후에도 "해결됨"이라고 표시해서는 안 된다.
새 operation이 Server에 commit되고 Client가 다시 수렴한 뒤에만 완료로 표시한다.

### 9.1 Markdown 수동 병합 작업 공간

수동 병합은 conflict를 감추거나 자동으로 해결하는 기능이 아니다. Server와 이
기기의 내용을 보존한 채, 사용자가 새 결과를 만들 수 있게 하는 비교·편집
작업 공간이다. conflict 판정, durable resolution, Server commit의 의미는 계속
[05 Conflict Resolution](./05_conflict-resolution.md)을 따른다.

`0.3.0`의 단순한 세 텍스트 영역은 두 내용을 안전하게 보였지만 비교하기에는
불편했다. `0.3.1`은 사용자가 다음 세 가지를 바로 알 수 있게 한다.

```text
Server 버전에 무엇이 있었는가?

이 기기 버전에 무엇이 있었는가?

어떤 내용이 새 결과로 저장되는가?
```

이 작업 공간은 자동 3-way merge, last-write-wins, protocol·Server API·persistence
형식 변경을 도입하지 않는다. Markdown 렌더링이나 실행 가능한 embed도 병합
화면에서 실행하지 않는다.

#### Desktop 비교 화면

모달 제목은 `Resolve conflict`이고, 그 아래에 영향받은 경로를 표시한다. Server
버전은 새 변경이 받아들여질 때까지 authoritative state라는 점을 설명한다.

Desktop에서는 두 읽기 전용 버전을 줄 단위로 나란히 비교하고, 그 아래에 별도의
편집 가능한 결과를 둔다. 각 변경 행에 `Use Server`와 `Use this device` 버튼을
반복해서 놓지 않는다. 긴 문서에서 버튼이 내용보다 더 눈에 띄거나 줄의 폭을
빼앗기 때문이다.

```text
Resolve conflict
notes/meeting.md

┌────────────────────────┬────────────────────────┐
│ Server version     │   │ This device's version  │
│ 12 - 삭제된 줄     │ ✓ │ 12 + 대체된 줄         │
│ 13   공통 줄       │   │ 13   공통 줄           │
└────────────────────────┴────────────────────────┘

Merged result
편집 가능한 텍스트 영역

[Cancel]                                  [Save merged result]
```

연속된 삽입·삭제·교체는 하나의 변경 묶음(hunk)으로 묶어 표시하지만, 묶음 안의
각 변경 행은 독립적으로 선택한다. Desktop에서 사용자는 해당 행의 Server 또는 이
기기 열을 직접 클릭하거나 키보드로 선택한다. 가운데의 좁은 gutter는 선택된 쪽을
표시하며, 행 또는 열을 hover·focus했을 때 선택 가능하다는 affordance를 드러낸다.
행마다 넓은 Action을 반복하지 않는다.

Server 전용 줄과 이 기기 전용 줄은 서로 다른 색, `-` / `+` 표식, 각 버전의 줄
번호를 함께 사용해 표시한다. 따라서 색을 구분하지 못해도 의미를 알 수 있다.
Desktop의 선택 가능한 각 열은 현재 선택 여부를 접근 가능한 이름과 pressed 상태로
전달한다.
긴 줄은 비교 의미가 흐려지지 않도록 줄바꿈하지 않고 가로로 스크롤한다.

#### Mobile 비교 화면

Mobile은 두 열을 무리하게 좁히지 않는다. 비교와 결과를 접근 가능한 탭 또는
segmented control로 전환한다.

```text
[Changes] [Merged result]

Server version
- 삭제된 줄

This device's version
+ 대체된 줄

선택한 변경 행: 1
[Use Server] [Use this device]

[Save merged result]
```

변경 묶음에서는 양쪽 레이블과 각 변경 행의 내용을 모두 보이며, 탭을 옮겨도 아직
저장하지 않은 결과 텍스트를 버리지 않는다. Mobile에서는 변경 행을 먼저 탭해
선택하고 Changes 화면 하단의 고정 Action bar에서 `Use Server` 또는 `Use this
device`를 고른다. 이 bar는 선택한 행이 없을 때 무엇을 먼저 해야 하는지 설명하고
두 Action을 비활성화한다. 따라서 줄마다 두 개의 버튼을 두지 않아도 작은 화면에서
명확한 터치 대상을 제공한다. 저장 Action은 desktop 전용 단축키 없이 터치로도
실행할 수 있어야 한다.

#### 줄 단위 선택과 결과 편집

초기 결과는 이 기기 버전으로 시작한다. 이는 모든 변경 행이 처음에는 `Use this
device`로 선택된 것과 같다. 화면은 `Result starts with this device's version.`처럼
그 사실을 명시한다.

각 변경 행에서 사용자는 다음 중 하나를 선택한다.

```text
Use Server

Use this device
```

선택은 그 행만 결과에 반영하며, 어느 읽기 전용 원본이나 Server Vault를 바꾸지
않는다. Desktop에서는 선택하려는 버전의 행을 직접 선택하고, Mobile에서는 행을
선택한 뒤 하단 Action bar에서 버전을 고른다. 같은 변경 묶음 안에서도 첫 줄은
Server, 다음 줄은 이 기기처럼 섞어 선택할 수 있다. 행 수가 서로 다른 삽입·삭제
에서는 빈 반대쪽을 선택해 해당 줄을 제외하거나 포함한다. 결과는 그 뒤에도 일반
텍스트로 직접 편집할 수 있다.

서로 다른 선택 행이 이어질 때 Client는 두 논리 행을 한 행으로 붙이지 않도록 줄
경계를 유지한다. 단, 선택 결과의 마지막 논리 행에 줄 끝 문자가 없었던 경우에는
그 줄 끝 문자를 임의로 추가하지 않는다.

이것은 두 버전을 비교해 사용자가 결정하도록 돕는 기능이지, 숨겨진 base version을
이용한 자동 병합이 아니다. 같은 줄을 양쪽에서 다르게 고쳤다면 사용자가 한쪽을
선택하거나 직접 조합해야 한다.

사용자가 결과를 직접 편집하면 결과는 custom unsaved result가 된다. 이 상태에서
다른 변경 행이나 전체 Server/이 기기 버전을 선택하려면 확인을 요구한다. 확인은
현재 결과가 선택된 행으로 다시 구성되며, 바뀌는 것은 저장 전 결과뿐이고 원본과
Server는 바뀌지 않는다고 설명한다.

Client는 큰 노트에서 비교 계산 때문에 mobile 앱이 멈추지 않도록 작업량을
제한한다. 안전하게 줄 단위로 나눌 수 없으면 그 사실을 표시하고 양쪽의 전체
문서 선택과 결과 편집만 제공한다. 불완전한 줄 diff를 완전한 것처럼 표시해서는
안 된다.

결과 편집기는 plain text다. Markdown preview를 병합 화면에 표시하지 않으며,
입력한 내용과 줄 바꿈을 임의로 변환하지 않는다.

#### 저장, 접근성, 개인정보

`Save merged result`는 사용자가 작성한 결과를 durable하게 기록하고 Local
replica에 적용한 뒤, 최신 Server 상태를 base로 하는 정상 MODIFY operation을
queue한다는 뜻이다. Server가 이미 받아들였다는 뜻은 아니다.

저장 중에는 다음처럼 중복 Action을 막는다.

```text
Save merged result → Saving… (disabled)
```

성공하면 결과가 동기화 대기열에 들어갔음을 알리고 Conflict Center에서 열었다면
남은 conflict 목록으로 돌아간다. 실패하면 modal과 편집한 결과를 유지하고,
어느 원본도 버려지지 않았음을 설명한다. 저장 전에 modal을 닫아도 conflict나
원본은 바뀌지 않는다. 기존 durable manual-merge artifact는 앱 종료·재시작의
복구 기준으로 계속 사용한다.

- `Server version`, `This device's version`, `+` / `-`, 줄 번호를 텍스트로 함께
  표시한다. 색만으로 상태를 구분하지 않는다.
- 탭, 변경 행 Action, 편집기, 저장·취소 Action은 키보드와 터치로 사용할 수
  있어야 한다.
- 긴 경로는 안전하게 줄바꿈하거나 생략하고, 접근 가능한 전체 값을 제공한다.
- note 내용은 사용자가 특정 conflict를 열었을 때에만 표시한다. notice, log,
  diagnostic에는 note 내용이나 Vault access token을 넣지 않는다.

다음 시나리오를 검증한다.

| 시나리오 | 기대 결과 |
| --- | --- |
| Desktop Markdown conflict | 색·표식·줄 번호가 있는 양쪽 비교와 별도 결과 편집기가 보인다. |
| Mobile Markdown conflict | 변경 행을 선택한 뒤 하단 Action bar에서 버전을 적용하고, 결과를 편집·저장할 수 있으며 가로 overflow가 없다. |
| 같은 묶음의 행을 다르게 선택 | 첫 행은 Server, 다음 행은 이 기기로 결과를 조합하며 원본은 바뀌지 않는다. |
| 직접 편집 후 행 선택 | 결과 재구성 전에 확인하며 원본은 바뀌지 않는다. |
| 저장 성공 | 결과를 Server 확정으로 잘못 표시하지 않고 durable queue에 넣는다. |
| 저장 실패 또는 앱 종료 | `0.3.0`의 durable resolution 규칙으로 결과를 복구할 수 있다. |
| Markdown 이외 또는 binary conflict | 수동 병합을 제공하지 않고 상태에 맞는 해소 Action만 제공한다. |

---

## 10. 진단과 안전한 Client State 재설정

### 10.1 복구 Action의 순서

재설정은 일반적인 동기화 문제의 첫 번째 해법이 아니다. Overview의 Advanced
recovery는 다음 순서로 Action을 제시한다.

```text
1. Retry now

2. Check all files

3. Review pending changes and conflicts

4. Reset sync tracking
```

각 Action은 local note와 Server Vault에 미치는 영향을 먼저 설명한다. 표준
사용자 흐름에서 파일 탐색기, browser developer tools 또는 IndexedDB 이름을
알도록 요구하지 않는다.

### 10.2 Reset sync tracking

`Reset sync tracking`은 Server URL과 Local Vault 파일을 삭제하지 않는다. Replica
index, cursor, apply journal과 bootstrap metadata 같은 Client sync state를 새로
만들고 Server-first Bootstrap을 다시 수행하는 마지막 수단이다.

실행 전 화면은 다음을 명확히 보여 준다.

```text
이 작업은 이 기기의 동기화 기록을 새로 만듭니다.

로컬 노트와 서버의 파일은 자동으로 삭제하지 않습니다.

다음 연결에서 서버 상태를 먼저 확인하고 모든 파일을 다시 안전하게 분류합니다.
```

Pending 또는 unresolved conflict가 없을 때만 일반 reset을 활성화한다. Pending
또는 conflict가 있으면 그 개수와 위험을 보여 주고 Conflict Center 또는 pending
상태로 이동시킨다. 사용자가 아직 Server에 반영되지 않은 변경을 모르고 버리는
reset을 한 번의 확인으로 수행하게 해서는 안 된다.

관리 또는 장애 복구를 위해 active Pending/Conflict가 있는 state를 정말
초기화해야 한다면, 구현은 먼저 local recovery artifact와 redacted diagnostic을
durable하게 보존하고, 사용자가 해당 backup 위치와 영향 범위를 확인한 별도의
고위험 절차를 요구해야 한다. 이 예외 절차는 normal setup UI에 노출하지 않는다.

### 10.3 설정 초기화와 state reset의 구분

다음 두 Action을 같은 "Reset" 버튼으로 합치지 않는다.

| Action                    | 바뀌는 것                                                    | 유지되는 것                          |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------ |
| Reset connection settings | Server URL와 pause preference                                | Client sync state와 Local Vault      |
| Reset sync tracking       | replica, cursor, pending이 없는 sync metadata, apply journal | Local Vault, Server Vault, 연결 설정 |

`Reset connection settings`도 Server URL과 Vault access token을 즉시 잃게 하므로 확인 modal을 거친다.
확인 뒤에는 입력 필드와 상태 요약을 저장된 빈 설정으로 즉시 다시 그려, 화면에
이전 URL이 남아 있는 상태와 실제 저장 상태가 달라지지 않게 한다.

plugin을 제거하거나 Obsidian app data를 지우는 것은 모든 플랫폼에서 같은 결과를
보장하지 않는다. 이는 일반 사용자용 reset 절차가 될 수 없다.

### 10.4 Copy diagnostic details

사용자가 지원 또는 self-diagnosis를 위해 복사할 수 있는 진단 정보는 다음을
포함할 수 있다.

```text
Client version and platform

Connected Server Vault identity, when known

Current display state

Pending and conflict counts

Last successful sync and last error category

Cursor and revision, when available
```

기본 diagnostic에는 note content, file content hash, authorization header,
Vault access token, recovery artifact content를 포함하지 않는다. 경로가 필요한 경우에도
사용자가 별도로 포함을 허용해야 한다.

---

## 11. Desktop과 Mobile의 표현

### 11.1 Desktop

Desktop Status Bar는 다음처럼 짧고 읽을 수 있는 문구를 사용한다.

```text
VaultDatum: Up to date

VaultDatum: Syncing

VaultDatum: 2 changes pending

VaultDatum: Review conflicts (2)
```

아이콘만으로 상태를 표현하지 않는다. tooltip 또는 accessibility label에는
마지막 성공 시각, 대기 수, 다음 추천 Action을 포함할 수 있다.

### 11.2 Mobile

모바일에는 Status Bar가 없어도 VaultDatum 설정에서 같은 Overview와 Action을
열 수 있어야 한다. touch target, 긴 상태 문구, conflict 목록과 modal은 작은
화면에서 사용 가능해야 한다.

모바일에서 앱이 background로 전환된 동안 동기화가 완료된 것처럼 표시하지
않는다. 다음 foreground 또는 network opportunity에서 확인을 재개한 뒤 실제
결과를 갱신한다.

### 11.3 시간과 지역화

기본 화면에는 `방금 전`, `2분 전` 같은 상대 시간을 사용하되, 보조 설명에는
사용자 locale과 timezone에 맞는 절대 시간을 제공한다. 상태, 버튼, 오류 설명은
모두 번역 가능해야 하며, 날짜 형식이나 영어 기술 용어에 의존하지 않는다.

---

## 12. `0.3.0` 범위와 비범위

### 12.1 필수 범위

`0.3.0` Client UX는 최소 다음을 제공한다.

```text
단일 Sync Overview와 Mobile 접근 경로

입력 중 요청을 보내지 않는 연결 설정 및 Test connection

Server-first 첫 동기화의 단계·결과·중단 후 재개 표시

상태 우선순위, 마지막 성공 시각, Pending/Conflict 수

사용자 행동이 필요한 오류의 원인별 설명과 추천 Action

일상 동기화와 구분된 Sync now 및 Check all files

Conflict Center 진입과 새로운 conflict 알림

파일을 건드리지 않는 보호된 Reset sync tracking 흐름

redacted Copy diagnostic details

색에만 의존하지 않는 Desktop/Mobile 접근성
```

이 범위는 현재 HTTP protocol을 바꾸지 않는 Client UX 개선을 우선한다. 새 API가
필요해지면 [07 API Specification](./07_api-specification.md)과 OpenAPI contract를
별도 변경으로 검토한다.

### 12.2 이번 범위에서 제외하는 것

다음은 UX가 좋아 보여도 `0.3.0`의 약속으로 만들지 않는다.

```text
Markdown 자동 3-way merge

mobile의 항상 실행되는 background sync

서버 또는 VPN의 인증 체계 변경

파일 내용 또는 정확한 전체 byte progress를 포함한 기본 알림

서버 backup 관리 화면

여러 사용자의 협업 또는 presence UI
```

---

## 13. UX 수용 시나리오

구현은 기존 protocol과 persistence 검증에 더해 다음 end-to-end 사용자 결과를
검증한다.

| 시나리오                                      | 기대 UX 결과                                                                |
| --------------------------------------------- | --------------------------------------------------------------------------- |
| 빈 Local Vault를 처음 연결                    | Server-first 단계가 보이고 Server 파일을 받은 뒤 `Up to date`가 됨          |
| 기존 Local 파일과 Server 파일을 처음 연결     | 자동 upload 전에 분류 결과를 보이며, 서로 다른 같은 경로는 conflict로 남음  |
| URL 입력 중                                   | 부분 URL마다 network 요청이나 오류 notice가 발생하지 않음                   |
| Server가 닿지 않는 URL 저장                   | 설정은 보존되고 Offline 설명과 retry Action이 보임                          |
| 저장된 URL을 다시 열기                        | `Saved`와 현재 Server 연결 상태를 혼동하지 않음                             |
| 일반 Local edit                               | 별도 Sync now 없이 pending → syncing → up-to-date로 전이함                  |
| mobile suspend 뒤 foreground 복귀             | 완료를 거짓 표시하지 않고 sync를 다시 확인함                                |
| 새 conflict                                   | 눈에 띄는 count와 Conflict Center 진입점이 생기며 다른 경로의 sync는 계속됨 |
| 여러 conflict 해소                            | Command Palette 없이 목록에서 상태별 Action을 연속으로 선택할 수 있음       |
| Pending 또는 conflict가 없는 state reset      | Local/Server 파일을 삭제하지 않고 Server-first Bootstrap으로 재시작함       |
| Pending 또는 conflict가 있는 state reset 시도 | 위험 설명과 review 경로를 제공하고 일반 reset을 실행하지 않음               |
| diagnostic 복사                               | note content, Vault access token, authorization header가 포함되지 않음     |

이 시나리오는 [10 Testing Strategy](./10_testing-strategy.md)의 Client integration 및
E2E test에 반영한다. UI 문구만 확인하는 test에 그치지 않고, 각 화면 상태가
실제 durable sync state와 일치하는지도 검증해야 한다.

---

## 14. UX 불변조건

### Invariant 1 — Automatic Sync Is Visible but Not Required

사용자는 자동 동기화가 실제로 동작하는지 알 수 있어야 하지만, 정상 동기화를
위해 매번 수동 버튼을 누를 필요는 없어야 한다.

### Invariant 2 — No Misleading Success

Pending, conflict, incomplete bootstrap 또는 recovery uncertainty가 있으면
`Up to date` 또는 동등한 성공 문구를 표시하지 않는다.

### Invariant 3 — User Actions Preserve Protocol Safety

UX의 `Sync now`, retry, pause/resume, full check, conflict resolution, reset은
Server-first bootstrap, base validation, idempotency, no-silent-overwrite
규칙을 우회하지 않는다.

### Invariant 4 — Reset Never Deletes User Content Automatically

일반 reset은 Local Vault와 Server Vault의 파일을 삭제하거나 덮어쓰지 않는다.
active Pending 또는 conflict가 있을 때에는 명시적인 보호 절차 없이 sync state를
삭제하지 않는다.

### Invariant 5 — Diagnostics Are Useful Without Leaking Content

사용자와 운영자가 상태를 진단할 수 있어야 하지만, 기본 UI, 알림, diagnostic,
log에는 Vault content와 Vault access token을 포함하지 않는다.
