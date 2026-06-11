# Homebrew Formula 생성 프롬프트

`choihunchul/homebrew-tap`에 formula를 추가할 때 AI에게 붙여 넣어 사용하는 프롬프트입니다.

---

## 빠른 사용법

1. 아래 **「입력 템플릿」**의 `{...}` 를 채웁니다.
2. **「프롬프트」** 전체를 복사해 AI 채팅에 붙여 넣습니다.
3. 출력된 `Formula/{name}.rb`를 tap repo에 commit합니다.
4. 로컬에서 `brew audit --strict`로 검증합니다.

---

## 입력 템플릿 (먼저 채울 것)

```
프로젝트 이름: lazyifconfig
formula 이름: lazyifconfig
GitHub repo: choihunchul/lazyifconfig
homepage: https://github.com/choihunchul/lazyifconfig
license: MIT
version: 0.2.4
tag: v0.2.4
설명(desc): Terminal UI for inspecting local network state

바이너리 이름: lazyifconfig
압축 해제 후 실행 파일 경로: lazyifconfig (tar 루트)

Release asset:
- lazyifconfig-v0.2.4-x86_64-apple-darwin.tar.gz
- lazyifconfig-v0.2.4-aarch64-apple-darwin.tar.gz
- lazyifconfig-v0.2.4-x86_64-unknown-linux-gnu.tar.gz

지원 OS: macOS (Intel + Apple Silicon) / Linux / macOS only / 단일 universal
tap repo: choihunchul/homebrew-tap
tap 경로: Formula/lazyifconfig.rb
```

---

## 프롬프트 (복사용)

````
Homebrew formula Ruby 파일을 작성해줘.

## 목적
아래 프로젝트를 `choihunchul/homebrew-tap` tap에 등록한다.
사용자는 `brew tap choihunchul/tap && brew install {formula-name}` 로 설치한다.

## 프로젝트 정보
- 프로젝트 이름: {프로젝트 이름}
- formula 이름(class명 PascalCase): {formula-name → Lazyifconfig}
- GitHub repo: {owner/repo}
- homepage: {URL}
- license: {MIT 등}
- version: {0.2.4}
- tag: {v0.2.4}
- desc: {한 줄 설명}

## Release asset
Release base URL: https://github.com/{owner}/{repo}/releases/download/{tag}/

| 파일명 | 플랫폼 |
|--------|--------|
| {asset-1} | {macOS Intel} |
| {asset-2} | {macOS ARM} |
| {asset-3} | {Linux x86_64} |

## 바이너리
- tar/zip 안 실행 파일 이름: `{binary-name}`
- install 블록: `bin.install "{binary-name}"`

## 작성 규칙
1. 출력은 `Formula/{formula-name}.rb` 내용만 (코드 블록 하나).
2. class 이름은 formula 이름의 PascalCase (예: `lazyifconfig` → `Lazyifconfig`).
3. `url`과 `sha256`은 실제 Release URL 기준으로 작성.
   - sha256은 아직 모르면 placeholder 대신 아래 curl 명령을 주석으로 함께 제공:
     `curl -fsSL "<url>" | shasum -a 256`
4. 플랫폼별 asset이 다르면 Homebrew 관용구 사용:
   - macOS Intel/ARM 분리: `on_macos do on_intel / on_arm`
   - Linux 추가: `on_linux do`
   - 단일 asset이면 top-level `url` + `sha256` 한 쌍
5. `test do` 블록 포함 (최소 `assert_predicate bin/"{binary-name}", :exist?`).
6. macOS-only 도구면 Linux 블록 넣지 마.
7. Rust/Go CLI 단일 바이너리 tar.gz 패턴을 가정 (압축 해제 시 루트에 바이너리 1개).
8. Homebrew Formula Cookbook 스타일, 불필요한 주석 없이.

## 참고 예시 (multi-arch macOS)
```ruby
class Lazyifconfig < Formula
  desc "Terminal UI for inspecting local network state"
  homepage "https://github.com/choihunchul/lazyifconfig"
  license "MIT"
  version "0.2.4"

  on_macos do
    on_intel do
      url "https://github.com/choihunchul/lazyifconfig/releases/download/v0.2.4/lazyifconfig-v0.2.4-x86_64-apple-darwin.tar.gz"
      sha256 "INTEL_SHA256_HERE"
    end

    on_arm do
      url "https://github.com/choihunchul/lazyifconfig/releases/download/v0.2.4/lazyifconfig-v0.2.4-aarch64-apple-darwin.tar.gz"
      sha256 "ARM_SHA256_HERE"
    end
  end

  def install
    bin.install "lazyifconfig"
  end

  test do
    assert_predicate bin/"lazyifconfig", :exist?
  end
end
```

## 추가로 알려줄 것
- sha256 계산용 curl 명령 (asset마다)
- `brew audit --strict Formula/{name}.rb` 로컬 검증 명령
- tap README Formulae 표에 추가할 한 줄 (| Formula | Description | Version |)
````

---

## 패턴별 프롬프트 한 줄 추가 지시

formula 유형에 따라 프롬프트 맨 아래에 한 줄 추가:

| 유형 | 추가 문장 |
|------|-----------|
| macOS Intel + ARM | `macOS Intel/ARM asset 각각 on_intel/on_arm 블록으로 작성해.` |
| macOS + Linux | `macOS(on_intel/on_arm)와 on_linux 블록을 모두 작성해.` |
| 단일 universal | `top-level url/sha256 한 쌍만 사용해.` |
| GitHub Actions 자동 bump | `url/sha256/version 필드는 update-homebrew-formula 액션이 갱신할 수 있게 단순 구조로 작성해.` |

---

## lazyifconfig 실전 예시 (채워진 프롬프트)

````
Homebrew formula Ruby 파일을 작성해줘.

## 목적
choihunchul/homebrew-tap tap에 lazyifconfig 등록.
brew tap choihunchul/tap && brew install lazyifconfig

## 프로젝트 정보
- formula: lazyifconfig (class Lazyifconfig)
- repo: choihunchul/lazyifconfig
- homepage: https://github.com/choihunchul/lazyifconfig
- license: MIT
- version: 0.2.4 / tag: v0.2.4
- desc: Terminal UI for inspecting local network state

## Release asset (macOS only)
- lazyifconfig-v0.2.4-x86_64-apple-darwin.tar.gz
- lazyifconfig-v0.2.4-aarch64-apple-darwin.tar.gz

tar 안 실행 파일: lazyifconfig (루트)

macOS Intel/ARM on_intel/on_arm 블록으로 작성해.
sha256은 curl로 계산하는 명령도 함께 알려줘.
````

---

## 검증 체크리스트

formula 생성 후:

```bash
# sha256 계산
curl -fsSL "https://github.com/OWNER/REPO/releases/download/vX.Y.Z/asset.tar.gz" | shasum -a 256

# tap clone
git clone https://github.com/choihunchul/homebrew-tap
cd homebrew-tap

# formula 저장 후
brew audit --strict Formula/my-tool.rb
brew install --build-from-source ./Formula/my-tool.rb
brew test ./Formula/my-tool.rb
my-tool --version
```

---

## tap README 갱신 프롬프트 (선택)

formula 작성 후 README 표 업데이트용:

````
choihunchul/homebrew-tap README의 Formulae 표에 아래 formula 한 줄 추가해줘.
- formula: lazyifconfig
- description: Terminal UI for inspecting local network state
- version: 0.2.4
- _(coming soon)_ 행은 제거
````
