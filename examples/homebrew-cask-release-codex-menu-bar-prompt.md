# codex-menu-bar Homebrew Cask Release 프롬프트

[choihunchul/codex-menu-bar](https://github.com/choihunchul/codex-menu-bar)에 GitHub Release + Homebrew Cask 자동 배포 workflow를 만들 때 AI에게 붙여 넣는 프롬프트입니다.

---

## 복사용 프롬프트 (전체)

````
choihunchul/codex-menu-bar 저장소에 GitHub Release + Homebrew Cask 자동 배포 workflow를 만들어줘.

## 목표
1. `v*` tag push → macOS 앱 빌드 → DMG GitHub Release 업로드 (기존 release.yml 개선)
2. Release 완료 후 → choihunchul/homebrew-tap Cask 자동 갱신
3. workflow_dispatch로 수동 Release / Cask 재배포 지원
4. (선택) choihunchul/github--actions에 reusable cask publish action 추가

## 프로젝트 정보
- repo: choihunchul/codex-menu-bar
- 종류: macOS 메뉴바 GUI 앱 (Swift 6, macOS 13+)
- tap repo: choihunchul/homebrew-tap
- cask 경로: Casks/codex-menu-bar.rb
- install: brew install --cask choihunchul/tap/codex-menu-bar
- secret: HOMEBREW_TAP_TOKEN (codex-menu-bar repo에 org secret 접근 필요, tap push 권한)

## 기존 workflow (release.yml)
- trigger: tag v*, workflow_dispatch
- runner: macos-15, Xcode 16
- 빌드: node scripts/package-app.mjs → create-dmg → CodexMenuBar.dmg
- Release: softprops/action-gh-release@v2, asset: dist/release/CodexMenuBar.dmg
- workflow_dispatch 시 version 하드코딩 1.0.0 → Package.swift / tag와 연동하도록 개선 필요

## Release asset
| tag | 파일 |
|-----|------|
| v1.0.7 | CodexMenuBar.dmg |

URL 패턴:
https://github.com/choihunchul/codex-menu-bar/releases/download/{tag}/CodexMenuBar.dmg

dmg 안 app 이름: CodexMenuBar.app (package-app.mjs / create-dmg 설정 기준, 실제 확인 후 cask에 반영)

## Cask 예시 (v1.0.7)
```ruby
cask "codex-menu-bar" do
  version "1.0.7"
  sha256 "71f8ff9544072fba3c57ebc7f47d03620857a5515b9c7b1026970201e55aec1d"

  url "https://github.com/choihunchul/codex-menu-bar/releases/download/v1.0.7/CodexMenuBar.dmg"
  name "Codex Menu Bar"
  desc "macOS menu bar companion for Codex, Cursor, and Antigravity"
  homepage "https://github.com/choihunchul/codex-menu-bar"

  app "CodexMenuBar.app"

  zap trash: [
    "~/.codex-menu-bar",
  ]
end
```

## 구현 요구사항

### A. release.yml 개선
- workflow_dispatch input: version (예: 1.0.8 또는 v1.0.8)
- tag ↔ Package.swift / Info.plist version 검증 (있으면)
- job id/name 유지 또는 publish-homebrew-cask 연동에 맞게 정리

### B. publish-homebrew-cask.yml (신규)
```yaml
name: Publish Homebrew Cask

on:
  workflow_run:
    workflows: ["Build and Release"]
    types: [completed]
  workflow_dispatch:
    inputs:
      version:
        description: "Release version, e.g. 1.0.7 or v1.0.7"
        required: true
      tap_repository:
        default: choihunchul/homebrew-tap
```

동작:
1. HOMEBREW_TAP_TOKEN 확인
2. version/tag 정규화 (v 접두사 처리)
3. DMG URL에서 sha256 계산: curl -fsSL URL | shasum -a 256
4. tap repo clone → Casks/codex-menu-bar.rb 생성 또는 url/version/sha256 갱신
5. commit & push
6. GitHub Step Summary에 install 명령 출력

permissions: contents: read

### C. homebrew-tap
- Casks/codex-menu-bar.rb 추가 (최초) 또는 bump
- README Casks 표 업데이트
- (선택) Casks/_template.rb + docs/cask-guide.md 추가 (Formula/_template.rb 패턴 참고)

### D. codex-menu-bar README
Install 섹션 추가:
```sh
brew install --cask choihunchul/tap/codex-menu-bar
```

Release 섹션에 tag push → Release → Cask 자동 bump 설명

### E. github--actions (선택, 공통화)
publish-homebrew-cask composite action:
- inputs: tap-repo, cask-name, cask-path, download-url, version, github-token
- Cask 파일의 url, sha256, version 갱신 (cask "name" do ... end 블록)
- reusable workflow: .github/workflows/publish-homebrew-cask.yml

Formula용 update-homebrew-formula와 역할 분리 (Cask는 --cask, Casks/ 경로)

## workflow_run 주의
- Build and Release workflow name과 정확히 일치해야 함
- tag push 이벤트는 workflow_run으로 연결 (release job 성공 시에만 cask publish)
- workflow_run 실패 시 cask publish 스킵

## Secret / 권한
- codex-menu-bar repo: HOMEBREW_TAP_TOKEN (org secret, codex-menu-bar 선택)
- homebrew-tap repo: secret 불필요 (push 대상)
- GITHUB_TOKEN: Release 생성용 (release.yml, contents: write)

## 릴리즈 흐름 (문서화)
```bash
# 버전 bump 후
git commit -am "Bump version to 1.0.8"
git tag v1.0.8
git push origin main
git push origin v1.0.8
# → Build and Release → Cask publish 자동
```

수동:
- Actions → Build and Release → Run workflow
- Actions → Publish Homebrew Cask → Run workflow (version 입력)

## 검증
- brew audit --cask --new Casks/codex-menu-bar.rb (tap repo 로컬)
- brew install --cask choihunchul/tap/codex-menu-bar
- 앱이 메뉴바에 표시되는지 확인

## 출력물
1. 변경/추가 파일 목록 (codex-menu-bar + homebrew-tap + github--actions)
2. workflow YAML 전체
3. Casks/codex-menu-bar.rb
4. README diff
5. 첫 테스트 방법
````

---

## 짧은 버전 (codex-menu-bar만)

````
codex-menu-bar에 Homebrew Cask 자동 배포 workflow 추가해줘.

- repo: choihunchul/codex-menu-bar (macOS 메뉴바 앱)
- 기존 release.yml: tag v* → CodexMenuBar.dmg Release
- 신규 publish-homebrew-cask.yml: Release 완료 후 choihunchul/homebrew-tap Casks/codex-menu-bar.rb bump
- secret: HOMEBREW_TAP_TOKEN
- install: brew install --cask choihunchul/tap/codex-menu-bar
- workflow_dispatch 버전 입력 지원
- release.yml workflow_dispatch version 하드코딩(1.0.0) 수정
- README Install/Release 섹션 반영
````

---

## homebrew-tap Cask만 먼저

````
choihunchul/homebrew-tap에 codex-menu-bar Cask 추가해줘.

- Casks/codex-menu-bar.rb
- version 1.0.7, dmg sha256: 71f8ff9544072fba3c57ebc7f47d03620857a5515b9c7b1026970201e55aec1d
- url: https://github.com/choihunchul/codex-menu-bar/releases/download/v1.0.7/CodexMenuBar.dmg
- app: CodexMenuBar.app
- README Casks 표 업데이트
- Casks/_template.rb + docs/cask-guide.md (Formula 가이드 패턴 참고)
````

---

## github--actions 공통 Cask 액션만

````
choihunchul/github--actions에 publish-homebrew-cask composite action 추가해줘.

- tap repo Casks/*.rb 의 url, sha256, version 갱신
- inputs: tap-repo, cask-name, download-url, version, github-token, tag-prefix
- reusable workflow + README + example
- update-homebrew-formula(Formula)와 분리
- codex-menu-bar가 첫 consumer (CodexMenuBar.dmg)
````

---

## Secret 체크리스트

| repo | HOMEBREW_TAP_TOKEN |
|------|-------------------|
| codex-menu-bar | ✅ (workflow 실행) |
| lazyifconfig | ✅ (Formula bump) |
| homebrew-tap | ❌ |
| github--actions | △ (테스트용만) |

---

## Formula vs Cask (같은 tap)

| 프로젝트 | 타입 | 경로 | 설치 |
|----------|------|------|------|
| lazyifconfig | Formula | Formula/lazyifconfig.rb | `brew install choihunchul/tap/lazyifconfig` |
| codex-menu-bar | Cask | Casks/codex-menu-bar.rb | `brew install --cask choihunchul/tap/codex-menu-bar` |
