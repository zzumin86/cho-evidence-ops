# CHO evidence 운영 승인 workflow

공개 저장소에는 workflow, `approved/backend_sha`, 이 문서만 둡니다. 비공개 backend 코드·history·원문·manifest 본문·DB 결과·개인정보·credentials를 복사하지 않습니다. 이 PR은 실행·발급·연구·P2 승인이나 merge·배포 승인이 아닙니다.

## 승인 경계

- 수동 `workflow_dispatch`만 가능합니다. main ref와 `expected_code_sha == github.sha`를 검사합니다. 실패하면 Environment job 전에 중단합니다.
- `approved/backend_sha`는 비공개 backend **운영 도구의 승인된 정확한 40자 commit**입니다. PR 리뷰로만 갱신합니다. runtime manifest의 `commit_sha`와 구분합니다.
- private backend는 Environment secret `CHO_BACKEND_READ_TOKEN`으로 승인 SHA만 checkout합니다. fine-grained token: **cho-trading-backend 한 저장소, Contents read-only**. broad PAT는 쓰지 않으며 checkout의 `persist-credentials:false`를 유지합니다.
- publisher/issuer는 다른 job·Environment입니다. issuer LOGIN DSN은 issuer에만, evidence write key는 publisher에만 있습니다. Render에는 접근하지 않습니다.
- 두 Environment는 main branch만, required reviewer `zzumin86`, admin bypass 비허용입니다. 사용자 직접 dispatch·승인 계획에 따라 self-review 방지 옵션은 미선택입니다. 실제 저장된 보호설정은 E-test로 별도 검증해야 합니다.

## E-test: secrets 등록 전 (아직 NOT_TESTED)

1. 이 PR의 코드·공개 출력·SHA를 Claude2 HQ 및 Codex가 리뷰합니다. GPT HQ의 동일 플랫폼 감사를 독립 감사로 계산하지 않습니다.
2. 별도 merge/실행 승인 후 main에서 `mode=env-guard`, main의 정확한 `expected_code_sha`로 수동 dispatch합니다. 두 Environment가 사용자 승인 전 **waiting**임을 확인하고 기록합니다. env-guard는 private checkout·키·DB·Storage·발급 도구를 실행하지 않습니다.
3. 비 main ref의 env-guard는 validate에서 거부되어 Environment job 실행0이어야 합니다. Environment 자체의 main 제한 설정도 함께 확인합니다. 코드 ref guard만으로 Environment 보호 저장을 증명하지 않습니다.
4. 실제 보호 확인 전 secrets 등록·다른 mode 실행은 보류합니다. 이 코딩 작업은 workflow를 dispatch하지 않았습니다.

## 후속 준비 (사용자/HQ 별도 승인)

두 Environment의 `CHO_BACKEND_READ_TOKEN`은 위 한 저장소 읽기 전용 token입니다. publisher Environment에만 `CHO_EVIDENCE_PUBLISHER_KEY`, `CHO_EVIDENCE_SOURCE_DSN`(현재 DB 읽기 전용 원천), `CHO_EVIDENCE_POLICY_DSN`(evidence DB 정책·grant·bucket metadata 읽기 전용)을 둡니다. issuer Environment에만 `CHO_RESEARCH_ISSUER_DSN`(전용 allowlisted LOGIN)을 둡니다. 두 Environment에 `CHO_RESEARCH_EVIDENCE_READ_KEY`(evidence legacy anon), `CHO_RESEARCH_ISSUER_STORAGE_READ_KEY`(현재 raw 프로젝트 읽기 원천)를 둡니다. 값은 공개 저장소·로그·채팅·대장에 쓰지 않습니다. 새 DB role/grant/secret 생성은 이 PR 범위에 없습니다.

Environment variables는 `CHO_RAW_PROJECT`, `CHO_RESEARCH_EVIDENCE_PROJECT`, `CHO_RESEARCH_TRUSTED_EVIDENCE_BUCKET`, `W5_RESEARCH_STORAGE_BUCKET`입니다. publisher에는 `CHO_SESSION_POOLER_HOST`, `CHO_EVIDENCE_OTHER_BUCKET_REFERENCE`(같은 evidence project의 **다른 private 버킷에 존재하는** 통제된 비민감 객체)를, issuer에는 `CHO_EXPECTED_ISSUER_PRINCIPAL`을 설정합니다. 없는 객체의 404를 버킷 접근 차단 증거로 쓰지 않습니다.

## 모드

| 모드 | 동작·쓰기 |
|---|---|
| env-guard | Environment 승인 경계만. 비밀·private checkout·외부 I/O 0 |
| reachability | Storage HTTPS / session pooler 5432 도달성 읽기. 발급/쓰기0 |
| permission-gates | 별도 승인 G2~G5: 테스트 객체 1개 create, reader POST/PUT/upsert/DELETE/move/copy 거부·불변성·G3/G4/G5·정책/grant 확인, publisher로 UUID namespace 정리. 이 모드는 **테스트 Storage 쓰기 있음** |
| publish | 독립 publisher read-only DB snapshot/count·전체 raw listing/hash → 새 UUID 경로 create-only 업로드·exact readback. 운영 발급0 |
| issuer-probe-db | 기존 issuer의 read-only principal/allowlist/원문 검사 |
| issuer-dry-run | 기존 issuer dry-run. 실제 DB 권한·분리 검증 완료를 주장하지 않음 |
| issuer-issue | 기존 issuer에 `confirm_manifest_hash == expected_manifest_hash` 재확인 후 수동 발급·exact readback. 별도 승인 필수 |

publish/issuer 모드의 manifest는 비공개 evidence Storage에 사전 준비한 canonical JSON입니다. 본문을 workflow input이나 artifact로 전달하지 않습니다. `manifest_reference`, `expected_manifest_hash`, `expected_runtime_code_sha`를 제공하며 publisher는 EXECUTION manifest와 `evidence_kind`를 사용합니다. BOOTSTRAP 원문에는 원 EXEC의 bootstrap reference 자체를 넣지 않으므로 승인자 준비 과정에서 ref/hash를 연결한 최종 EXEC manifest를 별도로 봉인합니다. 자동 승인·자동 발급은 없습니다.

원문 reference/sha256은 private tool의 반환 receipt와 새 `originals/<UUID>.json` 객체로 확인합니다. Actions 공개 stdout/summary에는 상태·검사ID·건수·hash만 출력하며 reference/path·원문·manifest·행·예외 상세는 출력하지 않습니다. 공개 artifact 업로드도 없습니다. 원문 위치를 공개 로그에서 복구하려 하지 말고 승인된 private Storage에서 해당 bytes/hash를 확인합니다.

publisher DB snapshot과 Storage listing은 원격 원자 snapshot이 아닙니다. 원문 진실성·권한 G1~G6·credential custody·E5·EXIT/overlap·Storage 용량·소규모 실행/P2는 별도 gate이며, 이 workflow 코드 준비만으로 종결하지 않습니다. 오류/응답분실 때 같은 경로를 덮거나 자동 재발급하지 않습니다. publisher 업로드 후 readback 실패는 commit unknown으로 취급해 private Storage에서 확인합니다.
