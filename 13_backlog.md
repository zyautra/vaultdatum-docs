# VaultDatum Backlog

> 이 문서는 아직 약속하지 않은 개선 후보를 모은다. 다른 설계 문서는 제품이 항상 만족해야 하는 목표 상태만 기술하므로, 여기의 항목은 요구사항이 아니다. 항목을 채택하면 규칙을 해당 기준 문서로 옮기고 이 문서에서 지운다.
>
> 후보는 [00 Product Specification](./00_product-specification.md)의 불변조건과 비목표를 바꾸지 않아야 한다.

## 1. 충돌 해결

- **Markdown 3-way 병합 제안**: Base, Server, Local을 비교해 병합 결과를 제안한다. 사용자 확인 없이 적용하지 않는다는 [05 Conflict Resolution](./05_conflict-resolution.md)의 원칙은 유지한다.
- **의미 단위 Markdown 병합**: heading, list 등 Markdown 구조를 고려한 병합 제안.
- **Rename Conflict 자동 해결**: 안전하게 판단 가능한 Rename vs Modify를 자동으로 합친다.
- **Bulk Conflict Action**: 선택한 여러 Conflict에 같은 Action을 적용한다. Bulk Local Apply는 많은 Server 변경을 만들므로 적용 전에 영향 범위를 보여 줘야 한다.

## 2. 동기화 모델

- **폴더·Vault 되돌리기**: 폴더나 Vault 전체를 특정 시점의 상태로 되돌린다. 현재는 단일 파일만 되돌릴 수 있다. 시점 지정 Manifest와, 원자적 적용을 위한 여러 파일의 논리적 변경 단위가 함께 필요하다.

- **여러 파일의 논리적 변경 단위**: 관련된 여러 파일 변경을 하나의 Transaction으로 commit한다. 하나의 Operation이 여러 Change를 만들 수 있도록 [06 Data Model](./06_data-model.md)의 Operation–Change 관계를 확장해야 한다.
- **Client 진행 상황 Acknowledgement API**: Server가 Client cursor를 관찰해 Change Journal, Tombstone, Operation Record의 GC 시점을 정한다.

## 3. 성능과 규모

- **대규모 Vault Local Scan 최적화**: 변경 후보만 해시하는 등 전체 Local Reconciliation 비용을 줄인다. Full Local Reconciliation 경로 자체는 유지한다.
- **대규모 Vault Server 성능**: Manifest 생성, Integrity Scan, Journal 조회 비용 측정과 개선.
- **Resumable Upload**: 큰 파일 Upload를 중단 지점부터 이어서 전송한다.
- **대용량 Artifact 저장소**: IndexedDB 대신 대용량 Blob에 적합한 Artifact backend를 ClientStore 경계 뒤에 둔다. Mobile Storage Quota, Crash Durability, 임시 Artifact GC를 함께 검증해야 한다.

## 4. 보안

- **공통 Secure Secret Storage**: Obsidian Desktop과 Mobile에 공통 secure storage API가 생기면 Vault token을 Plugin Settings에서 옮긴다.
- **범위가 있는 권한**: `403`을 사용하는 scope authorization. 개인 Vault 운영 모델과 [09 Security and Deployment](./09_security-and-deployment.md)의 제공하지 않는 접근 기능 목록과 충돌하지 않는 범위에서만 검토한다.

## 5. 운영

- **Online Consistent Backup**: Server를 멈추지 않고 Vault와 Sync State를 같은 시점으로 백업한다.
- **운영자 요청과 주기 Integrity Scan**: 현재 Integrity Scan은 Server 시작 시에만 실행한다. API에 노출하지 않는 운영자 전용 실행 경로와 선택적 주기 실행을 검토한다.

## 6. 테스트

시스템 규모와 복잡도가 커져 필요성이 생기면 다음 Test Infrastructure를 도입한다.

- Property-based Testing
- Model-based Testing
- Long-running Random Operation Test
- Large Vault Performance Test
- Protocol Compatibility Matrix
- Network Fault Simulation
- Multi-version Client / Server Test
