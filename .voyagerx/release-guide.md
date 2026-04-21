# RTK 사내 배포 가이드 — 빌드 · 릴리즈 · Claude Code 적용

> **대상 환경**: Apple Silicon Mac (`aarch64-apple-darwin`)
> **Fork 저장소**: https://github.com/v6x/rtk (public)
> **보안 전제**: 이번 빌드는 [security-audit-2026-04-21.md](./security-audit-2026-04-21.md) §6 **옵션 A** (telemetry env 미설정 자체 빌드)를 따름

이 문서는 두 역할로 나뉩니다:

1. **[Part 1 — 릴리즈 담당자](#part-1--릴리즈-담당자-빌드--업로드)**: 빌드하고 GitHub Release에 올리는 사람 (본인)
2. **[Part 2 — 사내 사용자](#part-2--사내-사용자-설치--claude-code-연동)**: 바이너리 받아서 Claude Code에 연결하는 사람 (동료)

---

## Part 1 — 릴리즈 담당자 (빌드 + 업로드)

### 1.1 사전 준비 (최초 1회만)

#### Rust 툴체인 설치

```bash
# rustup 설치 (권장 — 쉽게 업데이트/툴체인 관리 가능)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# 프롬프트에서 1 (default installation) 선택

# 새 셸 또는 source로 PATH 반영
source "$HOME/.cargo/env"

# 설치 확인
cargo --version   # cargo 1.xx.x
rustc --version   # rustc 1.xx.x

# Apple Silicon 타겟 추가 (이미 있으면 스킵됨)
rustup target add aarch64-apple-darwin
```

#### gh CLI 확인

```bash
gh --version             # 있어야 함 (현재 환경: 2.83.1)
gh auth status           # v6x org에 write 권한 있어야 함
```

### 1.2 빌드 (매 릴리즈마다)

```bash
# 1. fork 저장소에서 작업 — 최신 상태 확보
cd ~/Projects/rtk
git checkout master
git pull origin master

# 2. 릴리즈 빌드 — telemetry env 미설정 상태가 핵심
#    RTK_TELEMETRY_URL / RTK_TELEMETRY_TOKEN 가 export 되어 있지 않아야 함
env | grep -i RTK_TELEMETRY   # 출력이 비어 있어야 함. 있으면 unset 하고 다시 진행.

cargo build --release --target aarch64-apple-darwin

# 산출물 위치: target/aarch64-apple-darwin/release/rtk
ls -lh target/aarch64-apple-darwin/release/rtk
# 대략 3-5 MB
```

### 1.3 빌드 검증 (보안 감사 결과 재현)

```bash
# 1. telemetry URL 문자열이 바이너리에 없는지 확인 (옵션 A 검증)
strings target/aarch64-apple-darwin/release/rtk | grep -iE "telemetry|x-rtk-token" | head
# 출력에 http:// 또는 https://로 시작하는 외부 URL이 없어야 함
# "telemetry_marker_path" 같은 내부 함수/파일명은 있어도 정상

# 2. 네트워크 호출 없이 기본 동작 확인
./target/aarch64-apple-darwin/release/rtk --version
./target/aarch64-apple-darwin/release/rtk gain    # 비어 있어도 OK (DB 아직 없음)
./target/aarch64-apple-darwin/release/rtk git status

# 3. (선택) 설치해서 전역 테스트
cargo install --path . --force
rtk --version
```

### 1.4 패키징

```bash
# 버전 확인
VERSION=$(grep '^version' Cargo.toml | head -1 | cut -d'"' -f2)
echo "Version: $VERSION"   # 예: 0.37.2

# 스테이징 디렉토리에서 tar.gz 만들기 (경로가 깔끔하게 들어가도록)
TARGET_DIR=target/aarch64-apple-darwin/release
ARCHIVE="rtk-v${VERSION}-aarch64-apple-darwin.tar.gz"

tar -czvf "$ARCHIVE" -C "$TARGET_DIR" rtk

# SHA256 체크섬 생성 (동료들이 무결성 검증용)
shasum -a 256 "$ARCHIVE" > "${ARCHIVE}.sha256"

ls -lh "$ARCHIVE" "${ARCHIVE}.sha256"
cat "${ARCHIVE}.sha256"
```

### 1.5 Git 태그 + GitHub Release 생성

```bash
# 1. 태그 생성 — 사내 릴리즈는 v6x 접두사로 upstream과 구분
TAG="v${VERSION}-v6x"
git tag -a "$TAG" -m "v6x internal build: rtk v${VERSION} (telemetry-disabled, Apple Silicon)"
git push origin "$TAG"

# 2. Release 생성 + 바이너리 업로드 (gh CLI 사용)
gh release create "$TAG" \
  --repo v6x/rtk \
  --title "v6x internal build: rtk v${VERSION}" \
  --notes "$(cat <<EOF
사내 배포용 Apple Silicon 빌드.

## 빌드 구성
- Target: \`aarch64-apple-darwin\`
- Upstream: rtk-ai/rtk v${VERSION}
- Telemetry: **비활성화** (컴파일 타임에 \`RTK_TELEMETRY_URL\` 미설정)
- 보안 감사: \`.voyagerx/security-audit-2026-04-21.md\` 옵션 A 방식

## 설치 (Apple Silicon Mac)
\`\`\`bash
# 다운로드 + ~/.local/bin 에 설치
mkdir -p ~/.local/bin
curl -L https://github.com/v6x/rtk/releases/download/${TAG}/${ARCHIVE} \\
  | tar -xz -C ~/.local/bin

# PATH 확인 (없으면 ~/.zshrc 에 추가)
echo 'export PATH="\$HOME/.local/bin:\$PATH"' >> ~/.zshrc
source ~/.zshrc

# 검증
rtk --version
rtk gain  # 정상 출력 (처음엔 DB 비어 있음)
\`\`\`

## SHA256
\`\`\`
$(cat ${ARCHIVE}.sha256)
\`\`\`

## Claude Code 연동
\`rtk init -g\` → [사내 가이드](https://github.com/v6x/rtk/blob/master/.voyagerx/release-guide.md#part-2--사내-사용자-설치--claude-code-연동) 참조
EOF
)" \
  "$ARCHIVE" \
  "${ARCHIVE}.sha256"

# 3. 확인
gh release view "$TAG" --repo v6x/rtk
```

### 1.6 릴리즈 후 체크

```bash
# 동료 시점에서 한 번 테스트 (다른 디렉토리에서)
cd /tmp
curl -fL "https://github.com/v6x/rtk/releases/download/${TAG}/${ARCHIVE}" -o test.tar.gz
shasum -a 256 -c <(cat ~/Projects/rtk/${ARCHIVE}.sha256 | sed "s|${ARCHIVE}|test.tar.gz|")
tar -tzf test.tar.gz   # 내용 확인
rm test.tar.gz
```

---

## Part 2 — 사내 사용자 (설치 + Claude Code 연동)

> 동료에게 공유할 때 이 섹션 내용을 그대로 복붙해서 전달하면 됨.

### 2.1 요구사항

- Apple Silicon Mac (M1/M2/M3/M4)
- Claude Code 설치 및 로그인 완료

### 2.2 바이너리 설치 (30초)

```bash
# 1. 최신 릴리즈 다운로드 + 설치
mkdir -p ~/.local/bin
curl -L "https://github.com/v6x/rtk/releases/latest/download/rtk-v$(curl -s https://api.github.com/repos/v6x/rtk/releases/latest | grep tag_name | cut -d'"' -f4 | sed 's/-v6x//' | sed 's/^v//')-aarch64-apple-darwin.tar.gz" \
  | tar -xz -C ~/.local/bin

# 위 한 줄이 복잡하면, 릴리즈 페이지에서 파일명 확인하고 직접:
# curl -L https://github.com/v6x/rtk/releases/download/<TAG>/<FILE> | tar -xz -C ~/.local/bin

# 2. PATH 확인 (~/.local/bin 이 없으면 추가)
echo $PATH | tr ':' '\n' | grep -q "$HOME/.local/bin" || {
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
  source ~/.zshrc
}

# 3. 동작 확인
rtk --version      # 0.37.2 (또는 현재 릴리즈 버전)
rtk gain           # 토큰 절감 통계 (처음엔 비어 있음) — "command not found"가 나오면 잘못 설치된 것
```

> ⚠️ `rtk gain` 이 **반드시** 동작해야 합니다. `command not found` 또는 다른 에러가 나면 이건 Rust Type Kit(전혀 다른 프로젝트)이 잘못 설치된 것이니 알려주세요.

### 2.3 Claude Code 연동

```bash
rtk init -g
```

대화형 프롬프트가 뜹니다:

1. **Patch settings.json? [y/N]** → `y` 입력
2. 자동으로 백업 생성: `~/.claude/settings.json.bak`

### 2.3.1 `rtk init -g`가 실제로 하는 일

| 파일 | 동작 |
|---|---|
| `~/.claude/hooks/rtk-rewrite.sh` | (레거시 경로, 현재는 `rtk hook claude` 바이너리 명령으로 대체됨) |
| `~/.claude/settings.json` | `PreToolUse` 후크 엔트리 추가. 백업은 `.bak`로 저장 |
| `~/.claude/RTK.md` | 10줄짜리 context 파일 생성 |
| `~/.claude/CLAUDE.md` | 맨 위에 `@RTK.md` 참조 1줄 추가 |

후크 동작 흐름:

```
Claude Code         settings.json      rtk binary
     │                    │                 │
     │ "git status"       │                 │
     │───────────────────▶│                 │
     │                    │ PreToolUse      │
     │                    │────────────────▶│
     │                    │                 │ rewrite
     │                    │◀────────────────│ "rtk git status"
     │                    │                 │
     │ execute: rtk git status              │
     │─────────────────────────────────────▶│
     │                                      │ filter (60-90% 절감)
     │◀─────────────────────────────────────│
```

### 2.4 Claude Code 재시작 + 검증

1. **Claude Code 앱을 완전히 종료했다가 다시 실행** (후크 재로드 필수)
2. 새 세션에서 Claude에게 `git status` 같은 명령을 실행시켜 보기
3. 잠시 후:

```bash
rtk gain             # 사용 통계 확인 — 방금 실행된 명령이 나와야 함
rtk gain --history   # 상세 이력
```

### 2.5 텔레메트리 OFF 재확인 (방어적)

이번 빌드는 컴파일 타임에 telemetry가 제거되어 있지만, 런타임 가드도 걸어둡니다:

```bash
# ~/.zshrc 맨 아래에 추가
echo 'export RTK_TELEMETRY_DISABLED=1' >> ~/.zshrc
source ~/.zshrc
```

### 2.6 제거 방법

```bash
# Claude Code 연동만 제거 (바이너리는 유지)
rtk init -g --uninstall

# 바이너리까지 완전 제거
rm ~/.local/bin/rtk
rm -rf ~/.local/share/rtk   # SQLite 추적 DB + tee 파일

# 원복이 필요하면
cp ~/.claude/settings.json.bak ~/.claude/settings.json
```

---

## Part 3 — 업데이트 워크플로우 (릴리즈 담당자)

upstream에서 새 버전이 나왔을 때:

```bash
# 1. upstream fetch (최초 1회 remote 추가)
git remote add upstream https://github.com/rtk-ai/rtk.git 2>/dev/null || true
git fetch upstream

# 2. upstream의 새 태그 확인
git tag -l --sort=-version:refname | head -5

# 3. upstream/master 머지 (또는 특정 태그 체크아웃)
git checkout master
git merge upstream/master   # 또는: git reset --hard upstream/v0.38.0

# 4. 보안 감사 재실행 (최소한)
#    - 새 의존성 추가 여부 확인: `git diff upstream/master~..HEAD -- Cargo.toml`
#    - 새 ureq:: 호출지 확인: `grep -rn "ureq::" src/`
#    - 기존 2곳(telemetry.rs:142, telemetry_cmd.rs:173)에서 벗어나면 재감사

# 5. 메모리 업데이트: .voyagerx/security-audit-YYYY-MM-DD.md 새 파일로
#    (이전 감사 문서는 그대로 두고 날짜만 바꾼 새 파일 생성)

# 6. Part 1.2 ~ 1.5 반복
```

---

## Part 4 — 트러블슈팅

### 빌드가 실패하는 경우

```bash
# Rust 툴체인 업데이트
rustup update stable

# 클린 빌드
cargo clean
cargo build --release --target aarch64-apple-darwin
```

### 동료가 `command not found: rtk` 에러를 보고할 때

```bash
# PATH 확인
echo $PATH | tr ':' '\n' | grep local/bin

# 바이너리 존재 확인
ls -lh ~/.local/bin/rtk

# 실행 권한
chmod +x ~/.local/bin/rtk
```

### `rtk gain`이 "command not found"로 나올 때

다른 `rtk` (Rust Type Kit)가 PATH에서 우선순위가 높은 상태.

```bash
which rtk            # 첫 번째 rtk가 어디 있는지 확인
which -a rtk         # 모든 rtk 확인

# 예: /opt/homebrew/bin/rtk 가 먼저 잡힐 때
rm /opt/homebrew/bin/rtk   # 또는 PATH 순서 조정
```

### Claude Code가 후크를 안 타는 것 같을 때

```bash
# 설치 상태 점검
rtk init --show

# settings.json 직접 확인
cat ~/.claude/settings.json | grep -A5 rtk

# Claude Code 완전 종료 확인 (Cmd+Q) 후 재실행
```

### Apple의 Gatekeeper가 바이너리 실행을 차단할 때

로컬 빌드 바이너리는 서명이 없어서 macOS가 막을 수 있습니다.

```bash
# 특정 바이너리만 허용
xattr -d com.apple.quarantine ~/.local/bin/rtk 2>/dev/null || true

# 또는 설치 시점에 curl로 받으면 quarantine 속성이 안 붙음
# (Safari로 다운로드한 경우에만 붙음)
```

### 빌드 바이너리에 서명이 없어 배포가 불안정할 때 (선택 개선안)

```bash
# Apple Developer 인증서가 있다면 codesign
codesign --sign "Developer ID Application: VoyagerX (TEAMID)" \
  --options runtime \
  --timestamp \
  target/aarch64-apple-darwin/release/rtk

# 확인
codesign --verify --verbose target/aarch64-apple-darwin/release/rtk
```

> 사내 배포 규모/보안 요구에 따라 서명·공증(notarization)까지 할지 결정. 초기에는 생략해도 무방하지만, Gatekeeper 우회 절차가 매번 필요하면 서명 추가를 검토.

---

## 부록: 릴리즈 담당자용 원라이너

새 릴리즈를 올릴 때 한 방에:

```bash
#!/usr/bin/env bash
# .voyagerx/release.sh — 원라이너 (수동 실행용)
set -euo pipefail

cd ~/Projects/rtk
git checkout master && git pull origin master

# 보안 가드: telemetry env가 노출되어 있지 않은지 확인
if env | grep -qi RTK_TELEMETRY; then
  echo "ERROR: RTK_TELEMETRY_* env vars are set — unset them first" >&2
  exit 1
fi

cargo build --release --target aarch64-apple-darwin

VERSION=$(grep '^version' Cargo.toml | head -1 | cut -d'"' -f2)
TAG="v${VERSION}-v6x"
ARCHIVE="rtk-v${VERSION}-aarch64-apple-darwin.tar.gz"

tar -czvf "$ARCHIVE" -C target/aarch64-apple-darwin/release rtk
shasum -a 256 "$ARCHIVE" > "${ARCHIVE}.sha256"

git tag -a "$TAG" -m "v6x internal build: rtk v${VERSION}"
git push origin "$TAG"

gh release create "$TAG" \
  --repo v6x/rtk \
  --title "v6x internal build: rtk v${VERSION}" \
  --notes-file .voyagerx/release-notes-template.md \
  "$ARCHIVE" "${ARCHIVE}.sha256"

rm "$ARCHIVE" "${ARCHIVE}.sha256"
echo "Done: https://github.com/v6x/rtk/releases/tag/${TAG}"
```

(위 스크립트는 템플릿이니 필요시 `.voyagerx/release.sh`로 저장해서 사용)
