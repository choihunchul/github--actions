# lazyifconfig Homebrew Release 액션 적용 프롬프트

[choihunchul/lazyifconfig](https://github.com/choihunchul/lazyifconfig)에 `choihunchul/github--actions` Homebrew release 액션을 연결할 때 AI에게 붙여 넣는 프롬프트입니다.

---

## 복사용 프롬프트

````
choihunchul/lazyifconfig 저장소에 Homebrew tap 자동 배포를 적용해줘.
공통 액션 repo: choihunchul/github--actions

## 목표
- `v*` tag push → GitHub Release 빌드 완료 후 → choihunchul/homebrew-tap formula 자동 갱신
- 수동 workflow_dispatch도 유지 (버전 지정 재배포용)
- 기존 inline `publish-homebrew-tap.yml`을 github--actions reusable workflow / composite action으로 교체

## 프로젝트 정보
- 소스 repo: choihunchul/lazyifconfig (Rust CLI)
- tap repo: choihunchul/homebrew-tap
- formula: Formula/lazyifconfig.rb
- formula class: Lazyifconfig
- install: `brew install choihunchul/tap/lazyifconfig`
- secret: HOMEBREW_TAP_TOKEN (lazyifconfig repo에 등록됨, tap repo push 권한)

## Release asset (tag 예: v0.2.4)
GitHub Release workflow(`release.yml`)가 업로드하는 파일명 패턴:
- lazyifconfig-{tag}-x86_64-apple-darwin.tar.gz      (macOS Intel)
- lazyifconfig-{tag}-aarch64-apple-darwin.tar.gz     (macOS ARM)
- lazyifconfig-{tag}-x86_64-unknown-linux-gnu.tar.gz (Linux x86_64)
- lazyifconfig-{tag}-x86_64-pc-windows-msvc.zip      (Windows, formula 제외)

tar 안 실행 파일: lazyifconfig (루트)

## formula 구조 (multi-platform)
top-level url/sha256 한 쌍이 아님. 아래 블록 구조 유지:
```ruby
on_macos do
  on_intel do
    url "..."
    sha256 "..."
  end
  on_arm do
    url "..."
    sha256 "..."
  end
end
on_linux do
  on_intel do
    url "..."
    sha256 "..."
  end
end
```

## 중요 제약
github--actions의 `update-homebrew-formula`는 top-level url/sha256/version 한 쌍만 갱신함.
lazyifconfig는 macOS Intel/ARM + Linux 3개 asset이므로, 아래 중 하나로 처리:

1. (권장) github--actions에 multi-platform formula bump 액션 추가
   - 예: `update-homebrew-formula` 확장 또는 `publish-homebrew-formula` 신규 composite action
   - platform별 url/sha256 갱신 + version bump + tap push
2. lazyifconfig에만 inline workflow 유지하되 github--actions reusable workflow로 thin wrapper만 작성

단일 url formula용 `update-homebrew-formula`만 쓰면 안 됨.

## 기존 workflow (lazyifconfig)
- `.github/workflows/release.yml` — tag push 시 cross-platform 빌드 + GitHub Release
- `.github/workflows/publish-homebrew-tap.yml` — 수동 실행, formula 전체 생성 (현재 macOS Intel/ARM만, Linux 없음)
- `.github/workflows/create-release-tag.yml` — 버전 tag 생성
- `.github/workflows/publish-crate.yml` — crates.io 배포

## 구현 요구사항

### 1. github--actions (필요 시)
multi-platform formula publish composite action 작성:
- inputs: tap-repo, formula-name, github-token, version/tag, assets (platform → filename pattern)
- 동작: Release asset URL에서 sha256 계산 → formula의 각 on_* 블록 url/sha256/version 갱신 → tap commit & push
- reusable workflow: `.github/workflows/publish-homebrew-formula.yml`

### 2. lazyifconfig
새 workflow 또는 기존 `publish-homebrew-tap.yml` 교체:

```yaml
# tag push 후 release job 완료 뒤 homebrew 갱신 (workflow_run 또는 release 완료 후)
on:
  workflow_run:
    workflows: ["Release"]
    types: [completed]
    branches-ignore: []  # tag 이벤트는 workflow_run으로 연결
  workflow_dispatch:
    inputs:
      version: ...
```

또는 release.yml job에 homebrew step/job 추가 (needs: release).

workflow_dispatch 입력:
- version: 0.2.4 또는 v0.2.4

### 3. README 갱신 (lazyifconfig)
Install 섹션을 한 줄로:
```sh
brew install choihunchul/tap/lazyifconfig
```

Homebrew 배포 설명을 github--actions 기반으로 업데이트.

### 4. 검증
- tag v* push → Release → tap Formula/lazyifconfig.rb url/sha256/version 3플랫폼 모두 갱신
- workflow_dispatch로 특정 버전 재배포 가능
- formula에 Windows asset 포함하지 않음

## 참고 URL 패턴
```
https://github.com/choihunchul/lazyifconfig/releases/download/{tag}/lazyifconfig-{tag}-x86_64-apple-darwin.tar.gz
https://github.com/choihunchul/lazyifconfig/releases/download/{tag}/lazyifconfig-{tag}-aarch64-apple-darwin.tar.gz
https://github.com/choihunchul/lazyifconfig/releases/download/{tag}/lazyifconfig-{tag}-x86_64-unknown-linux-gnu.tar.gz
```

## 출력물
1. 변경/추가 파일 목록 (github--actions + lazyifconfig)
2. lazyifconfig에 추가할 workflow YAML 전체
3. github--actions composite action / reusable workflow (multi-platform 지원)
4. 필요 secret 및 권한 (permissions: contents: read)
5. 첫 테스트 방법 (tag push 또는 workflow_dispatch)
````

---

## 짧은 버전 (github--actions 확장 없이 wrapper만)

````
lazyifconfig의 publish-homebrew-tap.yml을 개선해줘.

- tap: choihunchul/homebrew-tap / Formula/lazyifconfig.rb
- Linux asset(lazyifconfig-{tag}-x86_64-unknown-linux-gnu.tar.gz) formula에 추가
- tag push 후 Release workflow 완료 시 자동 실행
- workflow_dispatch 유지
- secret: HOMEBREW_TAP_TOKEN
- install 명령: brew install choihunchul/tap/lazyifconfig
- README Install/Homebrew 섹션 반영
````

---

## github--actions만 먼저 확장할 때

````
choihunchul/github--actions에 multi-platform Homebrew formula publish 액션 추가해줘.

lazyifconfig(choihunchul/lazyifconfig)용 요구사항:
- tap: choihunchul/homebrew-tap
- formula: lazyifconfig (Formula/lazyifconfig.rb)
- platform assets:
  - macos-intel: lazyifconfig-{tag}-x86_64-apple-darwin.tar.gz
  - macos-arm: lazyifconfig-{tag}-aarch64-apple-darwin.tar.gz
  - linux-x86_64: lazyifconfig-{tag}-x86_64-unknown-linux-gnu.tar.gz
- formula on_macos/on_intel/on_arm, on_linux/on_intel 블록의 url/sha256/version 갱신
- composite action + reusable workflow + README + example YAML
- 기존 update-homebrew-formula(단일 url)와 역할 분리
````

---

## 체크리스트 (적용 후)

- [ ] `v*` tag push → Release → tap formula 자동 bump
- [ ] macOS Intel / ARM / Linux sha256 3개 모두 갱신
- [ ] `brew install choihunchul/tap/lazyifconfig` 동작
- [ ] workflow_dispatch로 수동 재배포 가능
- [ ] lazyifconfig README Install 섹션 한 줄 명령 반영
