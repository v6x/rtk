# RTK 설치 지시서 (Claude Code 전용)

> 사용법: 이 파일을 Claude Code 세션에서 `@.voyagerx/install-prompt-claude.md 읽고 설치해줘` 라고 전달하세요.

## 당신이 수행할 작업

사용자의 Mac에 **v6x/rtk** 사내 빌드를 설치하고, 지금 실행 중인 Claude Code (당신 자신) 에 PreToolUse 후크로 등록합니다. 설치는 비대화형으로 진행하며, Claude Code 앱 재시작 직전에 사용자에게 턴을 넘깁니다.

## 전제 조건

실행 전 반드시 확인:

```bash
uname -m            # arm64 여야 함 (Apple Silicon)
gh auth status      # v6x 접근 권한 필요. 실패하면 사용자에게 "gh auth login 먼저" 요청
```

둘 중 하나라도 실패하면 사용자에게 보고하고 중단.

## Step 1 — 바이너리 다운로드 + 설치

```bash
mkdir -p ~/.local/bin
gh release download --repo v6x/rtk --pattern "*aarch64-apple-darwin.tar.gz" --output - \
  | tar -xz -C ~/.local/bin
chmod +x ~/.local/bin/rtk
xattr -d com.apple.quarantine ~/.local/bin/rtk 2>/dev/null || true
```

## Step 2 — PATH 등록 (없을 때만)

`~/.zshrc`에 `export PATH="$HOME/.local/bin:$PATH"` 라인이 이미 있으면 스킵. 없으면 추가.

```bash
grep -q 'HOME/.local/bin' ~/.zshrc 2>/dev/null || \
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
```

현재 셸에 반영:
```bash
export PATH="$HOME/.local/bin:$PATH"
```

## Step 3 — 설치 검증

```bash
~/.local/bin/rtk --version     # 버전 한 줄 출력되어야 함
~/.local/bin/rtk gain          # "no data yet" 또는 통계 — 에러면 Step 7 참고
```

`rtk gain`이 `command not found`로 나오면 **다른 rtk (Rust Type Kit)** 가 PATH 우선순위에 있는 것. `which -a rtk`로 확인 후 사용자에게 보고하고 중단.

## Step 4 — Claude Code 후크 설치 (비대화형)

```bash
~/.local/bin/rtk init -g --auto-patch
```

`--auto-patch` 플래그가 `Patch settings.json?` 프롬프트를 자동 승인합니다. 텔레메트리 consent 프롬프트가 나타나면 `N`(거부)으로 응답 — 이번 빌드는 telemetry가 컴파일 타임에 제거되어 있지만 명시적 거부가 안전합니다.

## Step 5 — 방어적 런타임 가드

```bash
grep -q RTK_TELEMETRY_DISABLED ~/.zshrc 2>/dev/null || \
  echo 'export RTK_TELEMETRY_DISABLED=1' >> ~/.zshrc
```

## Step 6 — 설치 상태 확인

```bash
~/.local/bin/rtk init --show
```

출력에 Hook / settings.json / RTK.md 항목이 모두 정상 표시되어야 함. 하나라도 누락이면 사용자에게 보고.

## Step 7 — 사용자에게 턴 넘기기 ⚠️ 중요

당신은 지금 Claude Code로 동작 중이므로, 후크가 반영되려면 사용자가 **Claude Code 앱을 완전 종료한 뒤 재실행**해야 합니다. 당신은 자기 자신을 재시작할 수 없습니다.

다음과 같이 사용자에게 정확히 보고하고 작업을 종료하세요:

> ✅ 설치가 완료되었습니다.
>
> 후크를 반영하려면 **Cmd+Q로 Claude Code를 완전 종료**한 뒤 다시 실행해주세요 (창만 닫으면 프로세스가 남아 있어 반영되지 않습니다).
>
> 재시작 후 저에게 **"rtk 동작 확인해줘"** 라고 말씀해 주시면, `git status` 같은 명령을 실행해서 후크가 정상 동작하는지 검증하겠습니다.

이 시점에서 더 이상 아무것도 실행하지 마세요.

## Step 8 — (재시작 후) 동작 검증 — 사용자가 재요청했을 때만

사용자가 재시작 후 "동작 확인" 을 요청하면:

```bash
# 아무 git 명령 실행
git status

# 후크가 탔다면 이 명령 결과가 절감 통계에 기록됨
rtk gain --history | head -5
```

`rtk gain --history`에 방금 실행된 명령이 보이면 **성공**. 비어 있으면:

- Claude Code가 정말 재시작 됐는지 확인
- `~/.claude/settings.json` 내용 확인 (PreToolUse에 rtk 엔트리가 있어야 함)
- 결과를 사용자에게 보고하고 수동 트러블슈팅 요청

## 실패 시 대응 매트릭스

| 증상 | 조치 |
|---|---|
| `gh auth status` 실패 | 사용자에게 `gh auth login` 요청 후 중단 |
| `gh release download` 실패 | 네트워크 확인, 1회 재시도. 재실패 시 사용자 보고 |
| `rtk gain`이 command not found | 다른 rtk 충돌 — `which -a rtk` 보고 후 중단 |
| `rtk init --show`에 Hook 미설치 | `rtk init -g --auto-patch` 재실행, 실패 시 `~/.claude/settings.json` 원본 상태 보고 |
| Step 8에서 `rtk gain`이 비어 있음 | Claude Code 재시작 여부 확인 후 사용자 보고 |

## 금지사항

- `sudo` 사용 금지 — 모든 경로는 user-scope (`~/.local/bin`, `~/.claude/`, `~/.zshrc`)
- 자기 자신(Claude Code) 재시작 시도 금지 — 블로킹됨
- Rust/cargo 설치 시도 금지 — 바이너리 다운로드로 충분
- `~/.claude/settings.json` 수동 편집 금지 — `rtk init -g` 가 백업과 함께 처리
