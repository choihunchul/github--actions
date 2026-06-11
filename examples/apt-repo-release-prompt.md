# APT Repository 생성·배포 프롬프트

Git-hosted APT repo(`choihunchul/apt-repo`)를 만들고, CLI 프로젝트에 `choihunchul/github--actions` APT release 액션을 연결할 때 AI에게 붙여 넣는 프롬프트입니다.

---

## 빠른 사용법

1. 아래 **「입력 템플릿」**의 `{...}` 를 채웁니다.
2. 목적에 맞는 **「프롬프트」** 섹션을 복사해 AI 채팅에 붙여 넣습니다.
3. 출력물을 각 repo에 commit합니다.
4. tag push 또는 workflow_dispatch로 첫 배포를 검증합니다.

---

## 입력 템플릿 (먼저 채울 것)

```
GitHub owner: choihunchul
소스 repo: choihunchul/my-cli
APT repo: choihunchul/apt-repo
패키지 이름(Debian): my-cli
codename: stable
component: main

Release tag 예: v1.0.0
Release .deb asset:
- my-cli_1.0.0_amd64.deb
- my-cli_1.0.0_arm64.deb

secret 이름: APT_REPO_TOKEN (소스 repo에 등록, apt-repo push 권한)
GitHub Pages URL: https://choihunchul.github.io/apt-repo

설치 명령:
  echo "deb [trusted=yes] https://choihunchul.github.io/apt-repo stable main" | sudo tee /etc/apt/sources.list.d/choihunchul.list
  sudo apt update
  sudo apt install my-cli
```

---

## 복사용 프롬프트 (전체 — APT repo + 소스 repo 연동)

````
APT repository를 처음부터 만들고 CLI 프로젝트에 자동 배포를 연결해줘.
공통 액션 repo: choihunchul/github--actions

## 목표
1. `choihunchul/apt-repo` APT 저장소 생성 (GitHub Pages 호스팅)
2. 소스 repo `{owner}/{project}` — `v*` tag push → GitHub Release `.deb` 업로드 → apt-repo 자동 갱신
3. workflow_dispatch로 수동 재배포 지원
4. `publish-apt-repo` composite action / reusable workflow 사용

## 프로젝트 정보
- GitHub owner: {choihunchul}
- 소스 repo: {owner}/{project} ({Rust/Go/Node CLI 등})
- APT repo: {owner}/apt-repo
- Debian package name: {my-cli}
- codename: {stable}
- component: {main}
- secret: APT_REPO_TOKEN (소스 repo에 등록, apt-repo push 권한)

## Release asset (tag 예: v1.0.0)
GitHub Release workflow가 업로드하는 `.deb` 파일명 패턴:
- {my-cli}_{version}_amd64.deb
- {my-cli}_{version}_arm64.deb
- (선택) {my-cli}_{version}_armhf.deb

`.deb` 안 바이너리: {my-cli} (PATH: /usr/bin/{my-cli})

## APT repo 구조 (GitHub Pages)
```text
apt-repo/
├── pool/
│   └── main/
│       └── {m}/                    # 패키지명 첫 글자
│           └── {my-cli}/
│               ├── {my-cli}_1.0.0_amd64.deb
│               └── {my-cli}_1.0.0_arm64.deb
└── dists/
    └── stable/
        ├── Release
        └── main/
            ├── binary-amd64/
            │   ├── Packages
            │   └── Packages.gz
            └── binary-arm64/
                ├── Packages
                └── Packages.gz
```

GitHub Pages URL: `https://{owner}.github.io/apt-repo/`

사용자 설치:
```bash
echo "deb [trusted=yes] https://{owner}.github.io/apt-repo stable main" | sudo tee /etc/apt/sources.list.d/{owner}.list
sudo apt update
sudo apt install {my-cli}
```

## 구현 요구사항

### 1. apt-repo (`{owner}/apt-repo`) — 신규 생성
- public repo 생성
- 초기 디렉터리 구조 commit:
  - `pool/main/`
  - `dists/stable/main/binary-amd64/`
  - `dists/stable/main/binary-arm64/`
  - `dists/stable/Release` (placeholder)
- README 작성:
  - repo 목적 (개인 APT mirror)
  - GitHub Pages 설정 방법 (Settings → Pages → branch `main` / root)
  - sources.list 한 줄 + `apt install` 예시
  - `[trusted=yes]` 설명 (GPG 서명 없음)
  - 포함 패키지 목록 표
- `.gitignore` 불필요 (`.deb`는 pool에 commit됨)

### 2. 소스 repo (`{owner}/{project}`) — Release + APT publish

#### A. `.deb` 빌드 (release workflow)
- `v*` tag push + workflow_dispatch
- amd64 / arm64 `.deb` 생성 (cargo-deb, nfpm, fpm, dpkg-deb 등 프로젝트에 맞게)
- GitHub Release에 `.deb` asset 업로드
- package name = `{my-cli}`, version = tag에서 `v` 제거

#### B. `publish-apt.yml` (신규)
```yaml
name: Publish APT Repository

on:
  push:
    tags:
      - "v*"
  workflow_run:
    workflows: ["Release"]          # release workflow name과 일치
    types: [completed]
  workflow_dispatch:
    inputs:
      version:
        description: "Release version, e.g. 1.0.0 or v1.0.0"
        required: true

permissions:
  contents: read

jobs:
  publish-apt:
    if: ${{ github.event_name != 'workflow_run' || github.event.workflow_run.conclusion == 'success' }}
    uses: choihunchul/github--actions/.github/workflows/publish-apt-repo.yml@main
    with:
      apt-repo: {owner}/apt-repo
      package-name: {my-cli}
      asset-amd64: {my-cli}_{version}_amd64.deb
      asset-arm64: {my-cli}_{version}_arm64.deb
      version: ${{ github.event_name == 'workflow_dispatch' && inputs.version || '' }}
      verify-tag: ${{ github.event_name != 'workflow_dispatch' }}
    secrets:
      APT_REPO_TOKEN: ${{ secrets.APT_REPO_TOKEN }}
```

또는 release job에 composite action step 추가 (`needs: release`).

#### C. README Install 섹션 (Linux)
```bash
echo "deb [trusted=yes] https://{owner}.github.io/apt-repo stable main" | sudo tee /etc/apt/sources.list.d/{owner}.list
sudo apt update
sudo apt install {my-cli}
```

Release 흐름 문서화:
```bash
git commit -am "Bump version to 1.0.0"
git tag v1.0.0
git push origin main
git push origin v1.0.0
```

### 3. Secret / 권한

| repo | APT_REPO_TOKEN | GITHUB_TOKEN |
|------|----------------|--------------|
| `{owner}/{project}` (소스) | ✅ workflow 실행 | ✅ Release 생성 (contents: write) |
| `{owner}/apt-repo` | ❌ | ❌ |
| `choihunchul/github--actions` | △ (테스트용만) | - |

PAT scope: `repo` (classic) 또는 fine-grained `apt-repo` Contents Read and write

### 4. GitHub Pages 설정 (apt-repo)
- Settings → Pages → Deploy from branch → `main` / `/ (root)`
- 반영 후 확인: `curl -I https://{owner}.github.io/apt-repo/dists/stable/Release`

## publish-apt-repo 액션 (이미 github--actions에 있음)
- composite: `choihunchul/github--actions/publish-apt-repo@main`
- reusable: `choihunchul/github--actions/.github/workflows/publish-apt-repo.yml@main`
- 동작: Release `.deb` download → pool/ 배치 → Packages/Release 재생성 → apt-repo push
- example: `examples/publish-apt-repo-my-cli.yml`

## 검증
- [ ] apt-repo GitHub Pages 200 응답
- [ ] tag v* push → Release `.deb` 업로드
- [ ] apt-repo에 pool/ + dists/ commit 확인
- [ ] Ubuntu/Debian VM에서 sources.list 추가 → `apt install {my-cli}` 성공
- [ ] workflow_dispatch로 특정 버전 재배포 가능

## 출력물
1. 변경/추가 파일 목록 (apt-repo + 소스 repo)
2. apt-repo README + 초기 디렉터리 구조
3. 소스 repo release workflow (`.deb` 빌드)
4. 소스 repo publish-apt workflow YAML 전체
5. 소스 repo README Install (Linux/APT) 섹션
6. secret 등록 안내 + 첫 테스트 방법
````

---

## APT repo만 먼저 만들 때

````
choihunchul/apt-repo APT repository를 처음부터 만들어줘.

## 목적
GitHub Pages로 호스팅하는 개인 APT mirror.
여러 CLI 프로젝트의 `.deb`를 `apt install`로 설치할 수 있게 한다.

## 요구사항
1. repo: choihunchul/apt-repo (public)
2. 초기 디렉터리 구조:
   - pool/main/
   - dists/stable/main/binary-amd64/
   - dists/stable/main/binary-arm64/
   - dists/stable/Release (placeholder)
3. README:
   - GitHub Pages 활성화 방법 (main / root)
   - sources.list 예시:
     `deb [trusted=yes] https://choihunchul.github.io/apt-repo stable main`
   - `[trusted=yes]` 설명
   - 패키지 추가 방법 (publish-apt-repo 액션 참조)
   - 포함 패키지 표 (초기엔 비어 있음)
4. GitHub Pages URL: https://choihunchul.github.io/apt-repo

## APT repo 디렉터리 설명 (README에 포함)
- pool/ — 실제 .deb 파일
- dists/stable/ — codename (sources.list의 `stable`)
- dists/stable/main/ — component (sources.list의 `main`)
- binary-amd64/Packages.gz — 아키텍처별 인덱스

## 출력물
- 초기 commit에 포함할 파일 목록
- README 전체
- GitHub Pages 설정 체크리스트
````

---

## 소스 repo에 APT publish만 연결할 때 (apt-repo는 이미 있음)

````
{owner}/{project} 저장소에 APT repo 자동 배포를 적용해줘.
공통 액션 repo: choihunchul/github--actions

## 목표
- `v*` tag push → GitHub Release `.deb` 업로드 → {owner}/apt-repo 자동 갱신
- workflow_dispatch 수동 재배포
- 기존 inline workflow가 있으면 publish-apt-repo reusable workflow로 교체

## 프로젝트 정보
- 소스 repo: {owner}/{project}
- APT repo: {owner}/apt-repo
- package name: {my-cli}
- secret: APT_REPO_TOKEN ({project} repo에 등록됨)

## Release .deb asset (tag 예: v1.0.0)
- {my-cli}_{version}_amd64.deb
- {my-cli}_{version}_arm64.deb

## 구현
1. `.github/workflows/publish-apt.yml` — publish-apt-repo reusable workflow
2. release workflow에 `.deb` 빌드 + GitHub Release 업로드 (없으면 추가)
3. README Linux Install:
   ```bash
   echo "deb [trusted=yes] https://{owner}.github.io/apt-repo stable main" | sudo tee /etc/apt/sources.list.d/{owner}.list
   sudo apt update && sudo apt install {my-cli}
   ```

## 참고
- example: choihunchul/github--actions/examples/publish-apt-repo-my-cli.yml
- permissions: contents: read (APT publish job)
````

---

## .deb 빌드 + Release + APT publish (한 번에)

````
{owner}/{project}에 Linux `.deb` Release + APT repo 자동 배포 pipeline을 만들어줘.

## 프로젝트
- repo: {owner}/{project} ({Rust/Go 등 — 언어 명시})
- APT repo: {owner}/apt-repo (이미 존재 / 새로 생성)
- package: {my-cli}
- tag: v*
- secret: APT_REPO_TOKEN

## pipeline
1. tag push → cross-build → `.deb` (amd64 + arm64)
2. GitHub Release upload
3. publish-apt-repo → apt-repo pool/ + dists/ 갱신
4. workflow_dispatch (version 입력)

## .deb 빌드 도구 (프로젝트에 맞게 선택)
- Rust: cargo-deb
- Go: nfpm / go releaser
- 범용: fpm

## debian/control 최소 필드
- Package: {my-cli}
- Version: {semver}
- Architecture: amd64 / arm64
- Depends: (필요 시)

## 출력물
- release.yml (빌드 + Release)
- publish-apt.yml
- debian/ 또는 nfpm.yaml (프로젝트에 맞게)
- README Install (APT)
````

---

## github--actions publish-apt-repo 액션 참고용

````
choihunchul/github--actions의 publish-apt-repo 액션 사용법을 {owner}/{project}에 적용해줘.

이미 있는 액션:
- composite: publish-apt-repo/action.yml
- reusable: .github/workflows/publish-apt-repo.yml
- example: examples/publish-apt-repo-my-cli.yml

{project} 설정:
- apt-repo: {owner}/apt-repo
- package-name: {my-cli}
- asset-amd64: {my-cli}_{version}_amd64.deb
- asset-arm64: {my-cli}_{version}_arm64.deb
- codename: stable (기본)
- APT_REPO_TOKEN secret

wrapper workflow YAML 전체 + README Install 섹션 출력.
````

---

## Homebrew tap vs APT repo (같은 owner)

| 프로젝트 | 배포 채널 | 저장소 | 설치 |
|----------|-----------|--------|------|
| lazyifconfig | Homebrew | choihunchul/homebrew-tap | `brew install choihunchul/tap/lazyifconfig` |
| {my-cli} | APT | choihunchul/apt-repo | `apt install {my-cli}` |

같은 CLI를 Homebrew + APT 둘 다 배포할 수 있음:
- macOS: Homebrew (tar.gz)
- Linux: APT (`.deb`)

---

## Secret 체크리스트

| repo | APT_REPO_TOKEN | 용도 |
|------|----------------|------|
| `{owner}/{project}` | ✅ | apt-repo push |
| `{owner}/apt-repo` | ❌ | push 대상 |
| `choihunchul/github--actions` | △ | 테스트용만 |

PAT 생성: GitHub Settings → Developer settings → PAT → `repo` scope  
등록 위치: 소스 repo → Settings → Secrets → Actions → `APT_REPO_TOKEN`

---

## 적용 후 체크리스트

- [ ] `choihunchul/apt-repo` repo 생성 + 초기 pool/dists 구조
- [ ] GitHub Pages 활성화 (`main` / root)
- [ ] `curl -I https://choihunchul.github.io/apt-repo/dists/stable/Release` → 200
- [ ] 소스 repo에 `APT_REPO_TOKEN` secret
- [ ] Release workflow → `.deb` asset 업로드
- [ ] publish-apt workflow → apt-repo commit 확인
- [ ] Ubuntu/Debian에서 sources.list + `apt install` 성공
- [ ] workflow_dispatch 수동 재배포 가능
- [ ] README Install (Linux) 섹션 반영

---

## 트러블슈팅 (프롬프트에 붙일 때 참고)

| 증상 | 원인 | 해결 |
|------|------|------|
| Pages 404 | Pages 미활성화 / 반영 대기 | Settings → Pages, 1~5분 대기 |
| `Release file not found` | codename/component 불일치 | sources.list `stable main` ↔ workflow `codename`/`component` 일치 |
| `Unable to locate package` | Packages.gz 미생성 / Pages 미반영 | publish workflow 로그, apt-repo commit 확인 |
| push 403 | PAT 권한 부족 | APT_REPO_TOKEN에 apt-repo write 권한 |
| wrong architecture | arm64 PC에 amd64 .deb만 있음 | asset-arm64 input + arm64 .deb 빌드 |
