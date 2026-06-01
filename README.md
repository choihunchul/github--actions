# github--actions

공통 GitHub Actions 모음입니다.

## Actions

| Action | 경로 | 설명 |
| --- | --- | --- |
| Publish VS Code Extension | [`publish-vscode-extension/`](publish-vscode-extension/) | Open VSX + VS Code Marketplace 배포 |

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
