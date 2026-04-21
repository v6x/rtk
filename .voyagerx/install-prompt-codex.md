# RTK 설치 지시서 (Codex CLI 전용)

> 사용법: Codex CLI 세션에서 이 파일을 읽혀 `이거 보고 설치해줘` 라고 전달하세요.

## 당신이 수행할 작업

사용자의 Mac에 **v6x/rtk** 사내 빌드를 설치하고, 지금 실행 중인 Codex (당신 자신) 에 `~/.codex/AGENTS.md` 기반 지시 방식으로 등록합니다.

Codex 연동은 Claude Code와 다릅니다:
- Claude Code는 **PreToolUse 후크**로 명령을 자동 재작성
- Codex는 **AGENTS.md 지시문**을 읽고 *명시적으로* `rtk <cmd>`를 사용하도록 학습

즉 설치 후 당신은 git/cargo/npm 등의 verbose 명령을 실행할 때 **자발적으로 `rtk` 접두사를 붙여** 실행해야 합니다.

## 전제 조건

```bash
uname -m            # arm64 여야 함 (Apple Silicon)
gh auth status      # v6x 접근 권한 필요
```

실패 시 사용자에게 보고하고 중단.

## Step 1 — 바이너리 다운로드 + 설치

```bash
mkdir -p ~/.local/bin
gh release download --repo v6x/rtk --pattern "*aarch64-apple-darwin.tar.gz" --output - \
  | tar -xz -C ~/.local/bin
chmod +x ~/.local/bin/rtk
xattr -d com.apple.quarantine ~/.local/bin/rtk 2>/dev/null || true
```

## Step 2 — PATH 등록 (없을 때만)

```bash
grep -q 'HOME/.local/bin' ~/.zshrc 2>/dev/null || \
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
export PATH="$HOME/.local/bin:$PATH"
```

## Step 3 — 설치 검증

```bash
~/.local/bin/rtk --version
~/.local/bin/rtk gain          # "no data yet" 또는 통계 — 에러면 중단
```

`command not found` 나오면 다른 `rtk` (Rust Type Kit) 충돌. `which -a rtk` 확인 후 사용자 보고.

## Step 4 — Codex 연동 설치

```bash
~/.local/bin/rtk init -g --codex
```

이 명령이 수행하는 일:
- `~/.codex/AGENTS.md` 생성 또는 추가 — Codex가 세션 시작 시 읽는 지시문
- `~/.codex/RTK.md` 생성 — RTK 메타 명령 레퍼런스
- 텔레메트리 consent는 `N`(거부)으로 응답

대화형 프롬프트가 나오면:
- `Telemetry consent?` → **N**
- 그 외 y/N 프롬프트 → 기본값 따라가기

## Step 5 — 방어적 런타임 가드

```bash
grep -q RTK_TELEMETRY_DISABLED ~/.zshrc 2>/dev/null || \
  echo 'export RTK_TELEMETRY_DISABLED=1' >> ~/.zshrc
```

## Step 6 — 설치 상태 확인

```bash
~/.local/bin/rtk init --show
ls -la ~/.codex/AGENTS.md ~/.codex/RTK.md
```

`AGENTS.md`와 `RTK.md`가 존재해야 함. 내용에 "Use rtk as a proxy" 같은 지시가 들어 있어야 함.

## Step 7 — 즉시 적용 확인 (Codex는 재시작 불필요할 수 있음)

Codex CLI 버전에 따라 AGENTS.md 반영 방식이 다릅니다:

- **현재 세션이 AGENTS.md를 이미 로드한 경우**: 새 codex 세션을 시작해야 반영됨
- **세션 시작 시 읽는 경우**: 현재 세션에서도 읽어서 반영될 수 있음

가장 확실한 방법: 사용자에게 다음과 같이 보고하고 턴 넘김:

> ✅ 설치가 완료되었습니다.
>
> RTK 지시문이 `~/.codex/AGENTS.md` 에 추가되었습니다. 확실한 반영을 위해 **현재 codex 세션을 종료하고 새 세션을 시작**해 주세요.
>
> 새 세션에서 저에게 **"rtk 동작 확인해줘"** 라고 말씀해 주시면 검증하겠습니다.

## Step 8 — (새 세션에서) 동작 검증 — 사용자가 재요청했을 때만

새 Codex 세션에서 사용자가 "동작 확인" 을 요청하면:

1. 먼저 현재 AGENTS.md가 로드됐는지 확인:
   ```bash
   head -20 ~/.codex/AGENTS.md
   ```
   "rtk" 또는 "Rust Token Killer" 언급이 있어야 함.

2. 의도적으로 `rtk` 접두사를 써서 명령 실행 — Codex는 AGENTS.md를 따라 자발적으로 rtk를 사용해야 함:
   ```bash
   rtk git status
   ```

3. 절감 통계 확인:
   ```bash
   rtk gain --history | head -5
   ```
   방금 실행된 `rtk git status`가 이력에 보이면 성공.

4. 향후 세션에서 당신이 git/cargo/npm 등을 쓸 때 자발적으로 `rtk` 접두사를 붙이는지 관찰. 그렇지 않다면 AGENTS.md 내용을 사용자에게 보여주고 상의.

## 실패 시 대응 매트릭스

| 증상 | 조치 |
|---|---|
| `gh auth status` 실패 | `gh auth login` 요청 후 중단 |
| `gh release download` 실패 | 네트워크 확인 + 1회 재시도 |
| `rtk gain`이 command not found | `which -a rtk` 로 다른 rtk 충돌 확인 |
| `~/.codex/AGENTS.md` 미생성 | `rtk init -g --codex` 재실행. 여전히 실패면 사용자 보고 |
| 새 세션에서도 AGENTS.md 무시 | Codex 버전 확인 (`codex --version`) 및 사용자 보고 |

## Claude Code와의 차이 요약

| 항목 | Claude Code | Codex |
|---|---|---|
| 연동 방식 | PreToolUse 후크 (자동 재작성) | AGENTS.md 지시문 (명시적 사용) |
| 설치 플래그 | `rtk init -g --auto-patch` | `rtk init -g --codex` |
| 수정 파일 | `~/.claude/settings.json`, `CLAUDE.md`, `RTK.md` | `~/.codex/AGENTS.md`, `RTK.md` |
| 반영 방법 | 앱 Cmd+Q 재시작 | 새 세션 시작 |
| 에이전트 행동 | 사용자 명령 가로채기 | 에이전트가 자발적 `rtk` 접두사 사용 |

## 금지사항

- `sudo` 사용 금지 — 모든 경로는 user-scope
- 자기 자신(Codex) 재시작/종료 시도 금지
- Rust/cargo 설치 시도 금지 — 바이너리 다운로드로 충분
- `~/.codex/AGENTS.md` 수동 편집 금지 — `rtk init -g --codex`가 병합 처리
