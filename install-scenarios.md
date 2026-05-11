# Pixel Horizon — MS Store Standard Install Scenarios

> **Use:** Partner Center "EXE/MSI app" 등록 시 요구되는 **표준 설치 시나리오 →
> 인스톨러 반환 코드** 매핑. 제출 폼에 그대로 복사할 수 있도록 영문 표를
> 별도 섹션으로 둔다.
>
> **Bundle reality (2026-05-11 기준):** `src-tauri/tauri.conf.json`의
> `bundle.targets = ["msi"]`. WiX/MSI가 생성하는 인스톨러는 Windows 표준
> **MSIEXEC 종료 코드**를 자동으로 반환하므로 Pixel Horizon 측에서는
> 별도 코드 작성 없이 표준 코드가 그대로 동작한다. 향후 NSIS로 전환하거나
> 커스텀 부트스트래퍼를 도입할 경우 이 문서 끝의 §Implementation Notes
> 절을 함께 갱신할 것.

---

## 1 · 시나리오 → 반환 코드 매핑 (한글 설명)

| # | MS Store 시나리오 | 의미 | 반환 코드 | 표준 출처 |
|---|------------------|------|-----------|-----------|
| 1 | 사용자가 설치를 취소함 | 사용자가 설치 마법사에서 취소 버튼 클릭 | **1602** | `ERROR_INSTALL_USEREXIT` |
| 2 | 애플리케이션이 이미 있음 | 동일/상위 버전이 이미 설치됨 | **1638** | `ERROR_PRODUCT_VERSION` |
| 3 | 설치가 이미 진행 중 | 다른 MSI 세션이 mutex 점유 중 | **1618** | `ERROR_INSTALL_ALREADY_RUNNING` |
| 4 | 디스크 공간 부족 | 대상 드라이브 용량 부족 | **1603** *(일반 실패)* / **112** *(디스크 풀)* | `ERROR_INSTALL_FAILURE` / `ERROR_DISK_FULL` |
| 5 | 재부팅 필요 | 설치 완료를 위해 OS 재시작 필요 | **3010** *(성공+재부팅)* / **1641** *(재부팅 시작됨)* | `ERROR_SUCCESS_REBOOT_REQUIRED` / `ERROR_SUCCESS_REBOOT_INITIATED` |
| 6 | 네트워크 오류 | 다운로드/검증 실패 등 (MSI는 로컬 패키지라 거의 발생 안 함; 부트스트래퍼만 해당) | **1612** *(소스 사용 불가)* + 커스텀 하위 코드 | `ERROR_INSTALL_SOURCE_ABSENT` |
| 7 | 보안 정책으로 패키지 거부 | WDAC / AppLocker / SmartScreen / 그룹 정책에 의한 차단 | **1625** *(정책에 의해 금지됨)* | `ERROR_INSTALL_PACKAGE_REJECTED` |
| 8 | 설치 성공 | 정상 완료 | **0** | `ERROR_SUCCESS` |

### 보조 코드 (Partner Center "더 추가" 입력란용)

| 보조 시나리오 | 코드 |
|--------------|------|
| 일반 설치 실패 (catch-all) | 1603 |
| 다른 인스턴스가 이미 실행 중 | 1500 (`ERROR_INSTALL_ALREADY_RUNNING` 변형) |
| 패키지 손상/서명 검증 실패 | 1330 |
| 관리자 권한 없음 | 1260 |
| 사용자 권한 부족 | 1314 |

---

## 2 · Partner Center 제출용 표 (영문 · 그대로 복사)

> Partner Center → Apps and games → *Pixel Horizon* → Packages →
> "Standard installer scenarios" 섹션에서 시나리오별 `EXE return code`
> 입력란에 붙여넣기.

| # | Scenario (Partner Center label) | Return code(s) | Notes for reviewer |
|---|---------------------------------|----------------|--------------------|
| 1 | The user cancelled the installation | `1602` | MSIEXEC standard `ERROR_INSTALL_USEREXIT`. Emitted when the user clicks Cancel in the Tauri/WiX MSI UI. |
| 2 | The application is already present | `1638` | `ERROR_PRODUCT_VERSION`. WiX upgrade table blocks downgrade; same/newer version installed returns this code. |
| 3 | Another installation is already in progress | `1618` | `ERROR_INSTALL_ALREADY_RUNNING`. Windows Installer global mutex held by another MSI. |
| 4 | The disk is full | `112`, `1603` | `ERROR_DISK_FULL` from the file-copy phase; otherwise wrapped as `ERROR_INSTALL_FAILURE` (1603). |
| 5 | A reboot is required to complete installation | `3010`, `1641` | `3010` = success, reboot pending. `1641` = installer initiated reboot. Treat both as success states. |
| 6 | Network error | `1612` (+ custom sub-codes if a bootstrapper is added) | The MSI itself is local (no network). Reserved for the future Store-managed downloader / bootstrapper. |
| 7 | The package was rejected during installation due to a security policy on the device | `1625` | `ERROR_INSTALL_PACKAGE_REJECTED`. Triggered by WDAC / AppLocker / SmartScreen Enterprise / Group Policy block. |
| 8 | Installation succeeded | `0` | `ERROR_SUCCESS`. |

---

## 3 · Verification checklist

제출 전에 아래 항목을 확인한다 (`docs/ms-store-sideload-checklist.md`와
중복되지 않는 인스톨러-종료-코드 전용 검증):

- [ ] **#1 cancel:** 설치 마법사 첫 화면에서 Cancel → `echo %ERRORLEVEL%` == `1602`.
- [ ] **#2 already-present:** 동일 버전 두 번째 설치 → `1638` 반환 (WiX `MajorUpgrade` 정책 확인).
- [ ] **#3 in-progress:** 두 개의 `msiexec /i` 동시 실행 → 두 번째가 `1618` 반환.
- [ ] **#4 disk-full:** 100MB 미만 잔여 드라이브에 설치 시도 → `112` 또는 `1603`.
- [ ] **#5 reboot:** `msiexec /i ... REBOOT=Force` → `3010` 반환 (자동 재부팅 안 함).
- [ ] **#6 network:** 향후 부트스트래퍼 추가 시에만 검증; 현재는 N/A 표기.
- [ ] **#7 policy:** 테스트 머신에 AppLocker 차단 규칙 추가 후 설치 → `1625`.
- [ ] **#8 success:** 깨끗한 머신에 정상 설치 → `0`.

테스트 결과는 `docs/msstore-publish/testing-notes.md`에 회차별로 기록한다.

---

## 4 · Implementation Notes

### 현재 (MSI / WiX, 추가 작업 없음)

- Tauri MSI 번들러는 WiX 3.x를 사용. WiX/MSIEXEC 기본 동작이 §1·§2 표를
  그대로 충족하므로 **별도 인스톨러 코드 추가 불필요**.
- 단, WiX 업그레이드 정책은 `MajorUpgrade` 규칙이 켜져 있어야 #2 가
  `1638`로 정확히 떨어진다. Tauri 기본 템플릿에서 활성 상태 — 변경 시
  본 문서와 §3 체크리스트 동시 갱신.

### 향후 (NSIS 또는 커스텀 부트스트래퍼 전환 시)

NSIS는 기본적으로 `0` / `1` / `2` 정도만 반환하므로, MS Store 표준 8개
시나리오를 만족하려면 커스텀 NSIS 템플릿에 분기를 직접 작성해야 한다.

- `src-tauri/tauri.conf.json` — `bundle.windows.nsis.template` 키로 커스텀
  템플릿 경로 지정.
- 커스텀 `.nsi` 파일 — 각 시나리오 분기에서 `SetErrorLevel <code>` 후 `Quit`.
  `<code>` 값은 §1·§2 표를 그대로 따른다 (호환성).
- 네트워크 오류(#6)용 하위 코드 묶음(예: `9001` 타임아웃 / `9002` DNS /
  `9003` 인증서)은 부트스트래퍼 도입 시점에 본 문서에 추가.
- 전환 PR에서는 `desktop-resident-stability` + `harness-workflow-control`
  스킬로 설치/언인스톨/재부팅/정책 차단 시나리오를 모두 재검증한다.

---

## 5 · 변경 이력

| 날짜 | 변경 | 근거 |
|------|------|------|
| 2026-05-11 | 초안 작성 — MSI 표준 코드 매핑 + Partner Center 제출 표 | MS Store EXE/MSI 등록 요구사항 정리 |
