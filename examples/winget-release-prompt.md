# WinGet Release 생성·배포 프롬프트

GitHub Release Windows asset → `microsoft/winget-pkgs` manifest PR 자동 제출.  
`choihunchul/github--actions`의 `publish-winget` 액션을 소스 repo에 연결할 때 AI에게 붙여 넣는 프롬프트입니다.

---

## 변수 (프로젝트마다 수정)

```
소스 repo: choihunchul/lazyifconfig
PackageIdentifier: Choihunchul.Lazyifconfig
Windows asset 패턴: lazyifconfig-.*-x86_64-pc-windows-msvc\.zip$
Release workflow 이름: Release
secret: WINGET_TOKEN (소스 repo에 등록)
fork 계정: choihunchul (microsoft/winget-pkgs fork 위치)
설치: winget install Choihunchul.Lazyifconfig
```

---

## 복사용 프롬프트 (전체 — 최초 등록 + 자동 PR)

````
WinGet 배포를 처음부터 설정하고 lazyifconfig에 자동 PR workflow를 연결해줘.
공통 액션 repo: choihunchul/github--actions

## 목표
1. microsoft/winget-pkgs에 lazyifconfig 최초 manifest PR (수동 1회, AI가 manifest 초안 작성)
2. 이후 Release workflow 완료 → winget-pkgs bump PR 자동 제출
3. publish-winget composite action / reusable workflow 사용

## 프로젝트 정보
- 소스 repo: choihunchul/lazyifconfig (Rust CLI)
- PackageIdentifier: Choihunchul.Lazyifconfig
- Publisher: Choihunchul
- PackageName: Lazyifconfig
- homepage: https://github.com/choihunchul/lazyifconfig
- license: MIT
- Windows asset: lazyifconfig-{tag}-x86_64-pc-windows-msvc.zip (zip 안 lazyifconfig.exe)
- Release workflow: `.github/workflows/release.yml` (이름: Release)
- secret: WINGET_TOKEN (classic PAT, public_repo scope)

## 사전 준비 (사용자가 직접)
1. https://github.com/microsoft/winget-pkgs fork → choihunchul/winget-pkgs
2. classic PAT 생성 (public_repo) → lazyifconfig repo secret `WINGET_TOKEN`
3. winget-pkgs에 Choihunchul.Lazyifconfig 최소 1버전 PR 머지 (자동 bump 전제)

## winget-pkgs 최초 manifest (수동 PR)
경로:
manifests/c/Choihunchul/Lazyifconfig/{version}/
- Choihunchul.Lazyifconfig.yaml
- Choihunchul.Lazyifconfig.installer.yaml
- Choihunchul.Lazyifconfig.locale.en-US.yaml

installer 타입: zip (portable 또는 archive — lazyifconfig.exe 경로 확인)
InstallerUrl: GitHub Release zip URL
PackageVersion: semver (v 접두사 없음, 예: 0.2.10)

## lazyifconfig workflow (신규)
`.github/workflows/publish-winget.yml`:

- trigger: workflow_run (Release completed) + workflow_dispatch(tag 입력)
- Release workflow와 병렬 tag push 금지 — asset 업로드 완료 후 실행
- resolve-release job으로 tag/version 추출 (publish-apt-repo.yml 패턴 참고)
- reusable workflow:
  uses: choihunchul/github--actions/.github/workflows/publish-winget.yml@main
  with:
    identifier: Choihunchul.Lazyifconfig
    tag / version: resolve job outputs
    verify-tag: false
    installers-regex: lazyifconfig-.*-x86_64-pc-windows-msvc\.zip$
  secrets:
    WINGET_TOKEN: ${{ secrets.WINGET_TOKEN }}

example: choihunchul/github--actions/examples/publish-winget-lazyifconfig.yml

## README Install (Windows) 추가
```powershell
winget install Choihunchul.Lazyifconfig
```

## 검증
- [ ] winget-pkgs에 Choihunchul.Lazyifconfig 존재
- [ ] Release 후 publish-winget workflow 성공
- [ ] winget-pkgs PR 생성 확인
- [ ] PR 머지 후 `winget install Choihunchul.Lazyifconfig` 동작

## 출력
1. winget-pkgs 최초 manifest 3파일 초안 (버전은 최신 Release 기준)
2. lazyifconfig publish-winget.yml 전체
3. README Windows 설치 섹션
4. 사용자가 직접 할 일 (fork, PAT, 최초 PR 제출)
````

---

## lazyifconfig에 workflow만 연결 (winget-pkgs 등록은 이미 됨)

````
choihunchul/lazyifconfig 저장소에 WinGet 자동 PR workflow를 추가해줘.
공통 액션: choihunchul/github--actions publish-winget

## 조건
- PackageIdentifier: Choihunchul.Lazyifconfig
- Release workflow("Release") 완료 후 workflow_run으로 실행
- workflow_dispatch: tag 입력 재배포
- Windows asset regex: lazyifconfig-.*-x86_64-pc-windows-msvc\.zip$
- secret: WINGET_TOKEN

## 참고
- example: examples/publish-winget-lazyifconfig.yml
- APT publish workflow와 같은 resolve-release 패턴
- README에 winget install 한 줄 추가

publish-winget.yml 전체 YAML과 README 변경을 출력해줘.
````

---

## 다른 CLI 프로젝트용 (짧은 버전)

````
{owner}/{project}에 WinGet 자동 PR workflow 추가.

- uses: choihunchul/github--actions/.github/workflows/publish-winget.yml@main
- identifier: {Publisher}.{PackageName}
- installers-regex: {windows-asset-regex}
- trigger: workflow_run after "{Release workflow name}" OR release: types: [released]
- secret: WINGET_TOKEN (public_repo classic PAT)
- fork: {owner}/winget-pkgs

example/prompt: choihunchul/github--actions/examples/publish-winget-lazyifconfig.yml
````

---

## Homebrew / APT / WinGet 비교 (lazyifconfig)

| 프로젝트 | 채널 | 저장소 | 설치 |
|----------|------|--------|------|
| lazyifconfig | Homebrew | choihunchul/homebrew-tap | `brew install choihunchul/tap/lazyifconfig` |
| lazyifconfig | APT | choihunchul/apt-repo | `apt install lazyifconfig` |
| lazyifconfig | WinGet | microsoft/winget-pkgs (PR) | `winget install Choihunchul.Lazyifconfig` |

---

## Secret / PAT

| Secret | PAT 종류 | Scope | fork |
|--------|----------|-------|------|
| `WINGET_TOKEN` | classic (fine-grained ❌) | `public_repo` | `{owner}/winget-pkgs` |

PAT 생성: https://github.com/settings/tokens/new?scopes=public_repo

---

## 흔한 실패

| 증상 | 원인 | 해결 |
|------|------|------|
| Package does not exist in winget-pkgs | 최초 manifest 없음 | 수동 PR 1회 |
| Release asset not found | Release workflow 전에 실행 | workflow_run 사용 |
| Failed to create branch | fork stale / token scope | fork sync, classic PAT 확인 |
| PR validation failed | zip portable 경로 오류 | installer.yaml RelativeFilePath 확인 |

---

## 관련 파일 (github--actions)

| 파일 | 설명 |
|------|------|
| `publish-winget/action.yml` | composite action (winget-releaser 래퍼) |
| `.github/workflows/publish-winget.yml` | reusable workflow |
| `examples/publish-winget-lazyifconfig.yml` | lazyifconfig 예시 |
