# github--actions

공통 GitHub Actions 모음입니다.

## Actions

| Action | 경로 | 설명 |
| --- | --- | --- |
| Publish VS Code Extension | [`publish-vscode-extension/`](publish-vscode-extension/) | Open VSX + VS Code Marketplace 배포 |
| Release Obsidian Plugin | [`release-obsidian-plugin/`](release-obsidian-plugin/) | Obsidian 플러그인 GitHub Release 배포 |
| Update Homebrew Formula | [`update-homebrew-formula/`](update-homebrew-formula/) | Homebrew tap formula url/sha256/version 갱신 (단일 asset) |
| Publish Homebrew Formula | [`publish-homebrew-formula/`](publish-homebrew-formula/) | multi-platform Formula 생성·배포 |
| Publish Homebrew Cask | [`publish-homebrew-cask/`](publish-homebrew-cask/) | macOS Cask 생성·배포 |

---

# Release Obsidian Plugin

Obsidian 플러그인을 빌드한 뒤 **GitHub Releases**에 `main.js`, `manifest.json`, `styles.css`를 업로드하는 공통 GitHub Action입니다. [Obsidian 공식 가이드](https://docs.obsidian.md/Plugins/Releasing/Release+your+plugin+with+GitHub+Actions) 흐름을 따릅니다.

## 사전 준비

- repo **Settings → Actions → General → Workflow permissions**에서 **Read and write permissions** 활성화
- `manifest.json`의 `version`과 Git tag 이름이 일치해야 합니다 (예: tag `1.0.0`)
- Community plugin 등록은 별도로 [obsidian-releases](https://github.com/obsidianmd/obsidian-releases) PR이 필요합니다

## 사용 방법

### 1. Composite Action (권장)

```yaml
name: Release Obsidian plugin

on:
  push:
    tags:
      - "*"

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Release plugin
        uses: choihunchul/github--actions/release-obsidian-plugin@main
        with:
          build-command: npm run build
```

### 2. Reusable Workflow

```yaml
name: Release Obsidian plugin

on:
  push:
    tags:
      - "*"

permissions:
  contents: write

jobs:
  release:
    uses: choihunchul/github--actions/.github/workflows/release-obsidian-plugin.yml@main
    with:
      build-command: npm run build
```

더 많은 예시는 [`examples/release-obsidian-plugin-on-tag.yml`](examples/release-obsidian-plugin-on-tag.yml)을 참고하세요.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `github-token` | No | `github.token` | Release 생성용 토큰 |
| `working-directory` | No | `.` | 플러그인 루트 디렉터리 |
| `node-version` | No | `20` | Node.js 버전 |
| `package-manager` | No | `npm` | `npm`, `yarn`, `pnpm` |
| `install-command` | No | - | 커스텀 설치 명령 |
| `build-command` | No | `npm run build` | 빌드 명령 |
| `verify-tag` | No | `true` | tag ↔ manifest version 검증 |
| `tag-prefix` | No | - | tag 비교 전 제거할 접두사 (예: `v`) |
| `main-path` | No | `main.js` | 빌드 결과 main 스크립트 |
| `manifest-path` | No | `manifest.json` | manifest 경로 |
| `styles-path` | No | `styles.css` | styles 경로 |
| `include-styles` | No | `auto` | `auto` / `true` / `false` |
| `include-zip` | No | `false` | zip 아카이브 추가 |
| `draft` | No | `true` | draft release 생성 |
| `prerelease` | No | `false` | pre-release 표시 |
| `generate-release-notes` | No | `false` | release notes 자동 생성 |
| `release-title` | No | tag 이름 | release 제목 |
| `extra-assets` | No | - | 추가 asset 경로 (공백 구분) |

## Outputs

| Output | Description |
| --- | --- |
| `tag` | Git tag 이름 |
| `version` | `manifest.json` version |
| `release-url` | 생성된 GitHub Release URL |

## 동작 방식

1. 의존성 설치
2. `build-command` 실행 (기본 `npm run build`)
3. `main.js`, `manifest.json` 존재 확인
4. tag ↔ manifest version 검증
5. `styles.css`가 있으면 포함 (`include-styles: auto`)
6. `gh release create`로 GitHub Release 생성 (기본 draft)

## 태그로 릴리즈

```bash
git tag -a 1.0.0 -m "1.0.0"
git push origin 1.0.0
```

`v` prefix tag를 쓰면 `tag-prefix: v`를 설정하세요.

---

# Update Homebrew Formula

GitHub Release 이후 **Homebrew tap** formula의 `url`, `sha256`, `version`을 자동으로 갱신하는 공통 GitHub Action입니다.

## 사전 준비

- tap repo에 formula가 미리 있어야 합니다 (예: `Formula/my-cli.rb`)
- formula에 `url`, `sha256` 필드가 있어야 합니다 (`version`은 선택)
- tap repo에 push할 PAT가 필요합니다 (`repo` scope)

```ruby
class MyCli < Formula
  desc "My CLI tool"
  homepage "https://github.com/choihunchul/my-cli"
  url "https://github.com/choihunchul/my-cli/releases/download/v1.0.0/my-cli-1.0.0-darwin-arm64.tar.gz"
  sha256 "..."
  version "1.0.0"

  def install
    bin.install "my-cli"
  end
end
```

tap 등록:

```bash
brew tap choihunchul/tap
brew install my-cli
```

## 사용 방법

### 1. Composite Action (권장)

```yaml
name: Update Homebrew formula

on:
  push:
    tags:
      - "v*"

permissions:
  contents: read

jobs:
  homebrew:
    runs-on: ubuntu-latest
    steps:
      - name: Update formula
        uses: choihunchul/github--actions/update-homebrew-formula@main
        with:
          github-token: ${{ secrets.HOMEBREW_TAP_TOKEN }}
          tap-repo: choihunchul/homebrew-tap
          formula-name: my-cli
          download-url: https://github.com/choihunchul/my-cli/releases/download/{tag}/my-cli-{version}-darwin-arm64.tar.gz
          tag-prefix: v
```

### 2. Reusable Workflow

```yaml
name: Update Homebrew formula

on:
  push:
    tags:
      - "v*"

permissions:
  contents: read

jobs:
  homebrew:
    uses: choihunchul/github--actions/.github/workflows/update-homebrew-formula.yml@main
    with:
      tap-repo: choihunchul/homebrew-tap
      formula-name: my-cli
      asset-name: my-cli-darwin-arm64.tar.gz
      tag-prefix: v
    secrets:
      HOMEBREW_TAP_TOKEN: ${{ secrets.HOMEBREW_TAP_TOKEN }}
```

더 많은 예시는 [`examples/update-homebrew-formula-on-tag.yml`](examples/update-homebrew-formula-on-tag.yml)을 참고하세요.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `github-token` | No | `github.token` | tap repo push/PR용 토큰 |
| `tap-repo` | Yes | - | tap 저장소 (`owner/homebrew-tap`) |
| `formula-name` | Yes | - | formula 이름 (`.rb` 제외) |
| `formula-path` | No | `Formula/{name}.rb` | tap 내 formula 경로 |
| `tap-branch` | No | `main` | 갱신할 tap branch |
| `download-url` | No* | - | 다운로드 URL (`{tag}`, `{version}`, `{owner}`, `{repo}` 치환) |
| `asset-name` | No* | - | Release asset 파일명으로 URL 자동 조회 |
| `version` | No | tag에서 추출 | formula version |
| `tag-prefix` | No | `v` | tag에서 제거할 접두사 |
| `sha256` | No | 자동 계산 | SHA256 checksum |
| `verify-tag` | No | `true` | tag push 필수 여부 |
| `commit-message` | No | `Bump {formula} to {version}` | 커밋 메시지 |
| `create-pr` | No | `false` | 직접 push 대신 PR 생성 |
| `pr-base-branch` | No | `main` | PR base branch |
| `pr-branch-prefix` | No | `bump-` | PR branch 접두사 |

\* `download-url` 또는 `asset-name` 중 하나는 필수입니다.

## Outputs

| Output | Description |
| --- | --- |
| `version` | 갱신된 version |
| `tag` | Git tag 이름 |
| `download-url` | 사용된 다운로드 URL |
| `sha256` | SHA256 checksum |
| `formula-path` | tap 내 formula 경로 |
| `commit-sha` | tap repo 커밋 SHA (`create-pr: false`) |
| `pull-request-url` | 생성된 PR URL (`create-pr: true`) |

## 동작 방식

1. tag에서 version 추출 (`v1.0.0` → `1.0.0`)
2. `download-url` 또는 Release asset에서 URL 결정
3. URL에서 SHA256 자동 계산
4. tap repo clone → formula `url` / `sha256` / `version` 갱신
5. tap repo에 commit & push (또는 PR 생성)

## 태그로 formula 갱신

```bash
git tag v1.0.0
git push origin v1.0.0
```

---

# Publish Homebrew Formula

multi-platform CLI용 **Homebrew tap Formula**를 Release asset에서 생성·갱신합니다.  
macOS Intel/ARM, Linux 등 플랫폼별 `on_*` 블록을 지원합니다.

## 언제 쓰나

| Action | 용도 |
| --- | --- |
| `update-homebrew-formula` | 기존 formula, **url/sha256 한 쌍** bump |
| `publish-homebrew-formula` | **multi-platform** formula 전체 생성·갱신 |

## 사전 준비

- tap repo: `choihunchul/homebrew-tap`
- secret: `HOMEBREW_TAP_TOKEN` (workflow 실행 repo에 등록, tap push 권한)
- GitHub Release에 플랫폼별 asset 업로드 완료

## 사용 방법 (lazyifconfig 예시)

```yaml
name: Publish Homebrew Tap

on:
  push:
    tags:
      - "v*"
  workflow_dispatch:
    inputs:
      version:
        required: true
        type: string

permissions:
  contents: read

jobs:
  publish-tap:
    uses: choihunchul/github--actions/.github/workflows/publish-homebrew-formula.yml@main
    with:
      tap-repo: choihunchul/homebrew-tap
      formula-name: lazyifconfig
      desc: Terminal UI for inspecting local network state
      homepage: https://github.com/choihunchul/lazyifconfig
      binary-name: lazyifconfig
      asset-macos-intel: lazyifconfig-{tag}-x86_64-apple-darwin.tar.gz
      asset-macos-arm: lazyifconfig-{tag}-aarch64-apple-darwin.tar.gz
      asset-linux-intel: lazyifconfig-{tag}-x86_64-unknown-linux-gnu.tar.gz
      version: ${{ github.event_name == 'workflow_dispatch' && inputs.version || '' }}
      verify-tag: ${{ github.event_name != 'workflow_dispatch' }}
    secrets:
      HOMEBREW_TAP_TOKEN: ${{ secrets.HOMEBREW_TAP_TOKEN }}
```

설치:

```bash
brew install choihunchul/tap/lazyifconfig
```

예시: [`examples/publish-homebrew-formula-lazyifconfig.yml`](examples/publish-homebrew-formula-lazyifconfig.yml)

## 주요 Inputs

| Input | Required | Description |
| --- | --- | --- |
| `tap-repo` | Yes | tap 저장소 |
| `formula-name` | Yes | formula 이름 |
| `desc`, `homepage`, `binary-name` | Yes | formula 메타데이터 |
| `asset-macos-intel/arm`, `asset-linux-intel/arm` | No* | 플랫폼 asset (`{tag}`, `{version}` 치환) |
| `version` / `tag` | No | tag push 시 자동 추출 |
| `verify-tag` | No | tag push 필수 여부 |

\* 플랫폼 asset 중 하나 이상 필수.

---

# Publish Homebrew Cask

macOS GUI 앱용 **Homebrew Cask**를 DMG Release asset에서 생성·갱신합니다.

## 사용 방법 (codex-menu-bar 예시)

```yaml
name: Publish Homebrew Cask

on:
  push:
    tags:
      - "v*"
  workflow_dispatch:
    inputs:
      version:
        required: true
        type: string

permissions:
  contents: read

jobs:
  publish-cask:
    uses: choihunchul/github--actions/.github/workflows/publish-homebrew-cask.yml@main
    with:
      tap-repo: choihunchul/homebrew-tap
      cask-name: codex-menu-bar
      name: Codex Menu Bar
      desc: macOS menu bar companion for Codex, Cursor, and Antigravity
      homepage: https://github.com/choihunchul/codex-menu-bar
      app-name: CodexMenuBar.app
      asset-name: CodexMenuBar.dmg
      zap-trash: ~/.codex-menu-bar
      version: ${{ github.event_name == 'workflow_dispatch' && inputs.version || '' }}
      verify-tag: ${{ github.event_name != 'workflow_dispatch' }}
    secrets:
      HOMEBREW_TAP_TOKEN: ${{ secrets.HOMEBREW_TAP_TOKEN }}
```

설치:

```bash
brew install --cask choihunchul/tap/codex-menu-bar
```

예시: [`examples/publish-homebrew-cask-codex-menu-bar.yml`](examples/publish-homebrew-cask-codex-menu-bar.yml)

## 주요 Inputs

| Input | Required | Description |
| --- | --- | --- |
| `tap-repo` | Yes | tap 저장소 |
| `cask-name` | Yes | cask 이름 |
| `name`, `desc`, `homepage`, `app-name` | Yes | cask 메타데이터 |
| `asset-name` / `download-url` | No* | Release asset |
| `zap-trash` | No | uninstall 시 삭제 경로 (공백 구분) |

\* `asset-name` 또는 `download-url` 중 하나 필수.

---

# Publish VS Code Extension

VS Code 확장을 **Open VSX Registry**와 **VS Code Marketplace**에 한 번 패키징한 뒤 동일한 `.vsix`로 배포하는 공통 GitHub Action입니다.

## 사전 준비

### Secrets

| Secret | 설명 |
| --- | --- |
| `VSCE_PAT` | [VS Code Marketplace PAT](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#get-a-personal-access-token) (Azure DevOps, `Marketplace` → `Manage` 권한) |
| `OVSX_PAT` | [Open VSX PAT](https://open-vsx.org/user-settings/tokens) |

### Open VSX

- 확장의 `publisher` 네임스페이스가 Open VSX에 등록되어 있어야 합니다.
- `package.json`에 `license` 필드가 있어야 합니다.

## 사용 방법

### 1. Composite Action (권장)

```yaml
name: Release Extension

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Publish to Open VSX & VS Code Marketplace
        uses: choihunchul/github--actions/publish-vscode-extension@main
        with:
          vsce-pat: ${{ secrets.VSCE_PAT }}
          ovsx-pat: ${{ secrets.OVSX_PAT }}
          build-command: npm run compile
```

### 2. Reusable Workflow

```yaml
name: Release Extension

on:
  release:
    types: [published]

jobs:
  publish:
    uses: choihunchul/github--actions/.github/workflows/publish-vscode-extension.yml@main
    with:
      build-command: npm run compile
    secrets:
      VSCE_PAT: ${{ secrets.VSCE_PAT }}
      OVSX_PAT: ${{ secrets.OVSX_PAT }}
```

Reusable workflow 내부 composite action은 `choihunchul/github--actions/publish-vscode-extension@main`처럼 **절대 경로**를 씁니다. `./publish-vscode-extension`은 호출자 repo 기준으로 해석되어 실패합니다.

더 많은 예시는 [`examples/publish-on-release.yml`](examples/publish-on-release.yml)을 참고하세요.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `vsce-pat` | No | - | VS Code Marketplace PAT |
| `ovsx-pat` | No | - | Open VSX Registry PAT |
| `working-directory` | No | `.` | 확장 `package.json`이 있는 디렉터리 |
| `node-version` | No | `20` | Node.js 버전 |
| `package-manager` | No | `npm` | `npm`, `yarn`, `pnpm` |
| `install-command` | No | - | 커스텀 설치 명령 (package-manager 대체) |
| `build-command` | No | - | 패키징 전 실행할 명령 (예: `npm run compile`) |
| `publish-vscode` | No | `true` | VS Code Marketplace 배포 여부 |
| `publish-openvsx` | No | `true` | Open VSX 배포 여부 |
| `vsce-package-args` | No | - | `vsce package`에 전달할 추가 인자 |
| `vsce-publish-args` | No | - | `vsce publish`에 전달할 추가 인자 |
| `ovsx-args` | No | - | `ovsx`에 전달할 추가 인자 |
| `no-verify` | No | `false` | Marketplace 검증 건너뛰기 (`enableProposedApi` 등) |
| `pre-release` | No | `false` | pre-release 버전으로 배포 |

PAT가 비어 있으면 해당 레지스트리 배포 단계는 자동으로 건너뜁니다.

## Outputs

| Output | Description |
| --- | --- |
| `vsix-path` | 생성된 `.vsix` 파일 경로 |
| `vsix-name` | `.vsix` 파일명 (`extension.vsix`) |

## 동작 방식

1. 의존성 설치 (`npm ci` / `yarn install --frozen-lockfile` / `pnpm install --frozen-lockfile`)
2. 선택적 빌드 (`build-command`)
3. `@vscode/vsce`로 `extension.vsix` 패키징 (한 번만)
4. `ovsx publish extension.vsix` → Open VSX
5. `vsce publish extension.vsix` → VS Code Marketplace

동일한 `.vsix`를 두 마켓플레이스에 올리므로 버전/내용 불일치를 방지합니다.

## pnpm / yarn 예시

```yaml
- uses: choihunchul/github--actions/publish-vscode-extension@main
  with:
    vsce-pat: ${{ secrets.VSCE_PAT }}
    ovsx-pat: ${{ secrets.OVSX_PAT }}
    package-manager: pnpm
    build-command: pnpm run compile
```

## Open VSX만 / Marketplace만 배포

```yaml
# Open VSX만
- uses: choihunchul/github--actions/publish-vscode-extension@main
  with:
    ovsx-pat: ${{ secrets.OVSX_PAT }}
    publish-vscode: false

# VS Code Marketplace만
- uses: choihunchul/github--actions/publish-vscode-extension@main
  with:
    vsce-pat: ${{ secrets.VSCE_PAT }}
    publish-openvsx: false
```
