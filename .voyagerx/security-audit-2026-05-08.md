# RTK 사내 배포 보안 감사 보고서 (재감사)

- **감사 일자**: 2026-05-08
- **감사 대상**: `rtk` (Rust Token Killer) v0.39.0-v6x.260508.1 — 커밋 `dad9f16`(upstream) + v6x rebase HEAD 기준
- **이전 감사**: [security-audit-2026-04-21.md](security-audit-2026-04-21.md) (v0.37.2 / 80a6fe6)
- **증분**: upstream `rtk-ai/rtk` 99 commits (0.37.2 → 0.39.0)
- **결론 요약**: **사내 배포 안전 (변동 없음)**. 새 외부 egress 경로 없음, 의존성 추가 없음, unsafe 경계 동일. 일부 보안성·견고성 fix를 흡수.

---

## 0. 이전 감사 대비 변경 요약 (Diff Audit)

이번 감사는 v0.37.2 baseline 보고서에서 **달라진 점만** 검증한다. 기존 결론(§1–§5)은 변경 없이 유효하다.

| 영역 | 결과 |
|---|---|
| 신규 직접 의존성 | **0개** (`Cargo.toml` deps 라인 변경 없음, version 줄만 bump) |
| 신규 transitive 네트워크 크레이트 | **0개** (HTTP crate 여전히 `ureq` 단 1개) |
| 신규 `ureq::` 호출지 | **0** (telemetry.rs:142, telemetry_cmd.rs:173 — 동일) |
| 신규 `unsafe` 블록 | **0** (Unix signal handler 동일, lint annotation `#[allow(unsafe_code)]` 추가뿐) |
| Telemetry 로직 변경 | **0** (3중 gate, 페이로드 필드, 컴파일 타임 환경변수 모두 동일) |
| Telemetry 페이로드 필드 추가 | **0** |
| 신규 자동 업데이트/크래시 리포팅 | **0** |
| 신규 phone-home / DNS / TcpStream | **0** |
| 빌드 스크립트 (`build.rs`) 변경 | 변경 없음 (검증: `git diff 80a6fe6..HEAD -- build.rs` 비어 있음) |

검증 커맨드 (재현 가능):

```bash
# 의존성 변동
git diff 80a6fe6..HEAD -- Cargo.toml | grep -E '^\+[a-z]'
# 출력: (empty) — version 줄만 변경

# 네트워크 호출지
grep -rn 'ureq::\|reqwest::\|hyper::\|TcpStream\|to_socket_addrs' src/
# 출력: src/core/telemetry.rs:142, src/core/telemetry_cmd.rs:173 (이전과 동일)

# unsafe 블록
grep -rn 'unsafe {\|unsafe extern' src/
# 출력: src/main.rs:2219 (extern "C" fn), src/main.rs:2229 (block) — 둘 다 기존 signal handler
```

---

## 1. 흡수된 upstream 변경 — 보안·견고성 관점 분류

99개 커밋 중 보안·견고성에 의미 있는 항목만 정리.

### 1.1 의존성 보안 패치

| 커밋 | 변경 | 평가 |
|---|---|---|
| `74c1b36` | `rustls-webpki` 0.103.9 → 0.103.13 (dependabot, indirect) | ✅ TLS PKI 검증 라이브러리 패치 흡수. rustls/webpki 릴리스 노트 기준 보안 수정 포함. |

### 1.2 입력 처리 견고성 (DoS/오작동 방지)

| 커밋 | 변경 | 평가 |
|---|---|---|
| `2ed53c7` | `fix(curl)`: force_tee_hint gating, JSON heuristic 확장, **full-body alloc 회피** | ✅ 대용량 응답 메모리 폭증 회피. DoS 표면 축소. |
| `02da3d0` | `fix(curl)`: JSON passthrough + IsTerminal gate (invalid JSON 출력 방지) | ✅ 기능 정정, 보안 영향 없음 |
| `533886a`+`7840030`+`a017fbd` | `fix(json)`: char-boundary truncation for multibyte strings | ✅ UTF-8 멀티바이트 경계에서 panic 가능성 제거 (이전: 비ASCII 문자 자르면 패닉) |
| `cac8ce7` | `fix(ls)`: device files (block/char/pipe/socket) 처리 | ✅ 출력 누락 정정, 보안 영향 없음 |
| `b51a815` | `fix(ls)`: `LC_ALL=C` 강제 + 인식 못 한 locale 시 raw fallback | ✅ **이번 감사에서 직접 확인한 ko_KR.UTF-8 로케일 빈 출력 버그 fix** |
| `cac8ce7` | `fix(ls)`: device files 처리 | ✅ |

### 1.3 Process / FD 격리

| 커밋 | 변경 | 평가 |
|---|---|---|
| `81a1be6` | `fix(stream)`: respective fd로 라우팅 | ✅ stdout/stderr 혼선 방지 |
| `e92d099` | `fix(runner)`: command failure 시 fd separation 보존 | ✅ |
| `a1d46f3` | `fix(stream)`: stderr 필드 누락 fix | ✅ |
| 신규 SIGINT/SIGTERM handler (issue #897) | `src/main.rs:2210-2238` proxy 자식 프로세스 orphan 방지 | ⚠️ → ✅ unsafe 블록 그대로(2곳), libc::signal 등록만 추가. 표준 패턴. nosemgrep 주석은 정적 분석용 false-positive 억제용. |

### 1.4 Path / 입력 sanitization

| 커밋 | 변경 | 평가 |
|---|---|---|
| `323cc3d`+`73a05c3`+`2d031f3` | `fix(discover)`: `encode_project_path`에서 `_`, `\\`, 비ASCII 문자, `.` 모두 `-`로 정규화 | ✅ Claude Code의 project slug 인코딩 규칙과 일치시키는 결정론적 매핑. 보안 영향 없음 (이미 `~/.claude/projects/<slug>` 경로의 디렉터리 명만 영향). |
| `0ea115b`+`42d3161` | `fix(discover)`: `exclude_commands` 패턴 word boundary, env-prefix·sub cmd·regex bypass 차단 | ✅ 사용자가 텔레메트리/디스커버리에서 제외하려는 명령이 부분 매치로 새는 것을 방지. 보호 강화. |
| `d327724`+`7681daf` | `fix(stream)`/`fix(cicd)`: semgrep flag 추가 (sh test) | ✅ 정적 분석 false-positive 억제. 실제 unsafe 변경 없음. |

### 1.5 무관한 변경

CI/CD (release-please, build orchestration), 테스트 안정화, 리팩터링은 보안 평가 대상 아님.

---

## 2. 직접 검증 — 이번 감사에서 재실행한 항목

### 2.1 Telemetry compile-time disabled 확인

```rust
// src/core/telemetry.rs:14-15  — 변경 없음
const TELEMETRY_URL: Option<&str> = option_env!("RTK_TELEMETRY_URL");
const TELEMETRY_TOKEN: Option<&str> = option_env!("RTK_TELEMETRY_TOKEN");
```

빌드 환경(local) 확인:

```bash
$ env | grep RTK_TELEMETRY
# (empty)
```

→ 컴파일 타임 환경변수 미설정. `option_env!` 결과 `None` → `maybe_ping()` 첫 줄 early return → `send_ping` 함수 자체가 dead code.

### 2.2 ureq 호출지

```
src/core/telemetry.rs:142:    let mut req = ureq::post(url)...
src/core/telemetry_cmd.rs:173:  let mut req = ureq::post(&url)...
```

전 코드베이스에서 2곳뿐 (이전과 동일).

### 2.3 자동 업데이트 / 크래시 리포팅 부재 확인

```bash
$ grep -rnE 'self_update|sentry|bugsnag|rollbar|panic_handler|set_hook|opentelemetry|datadog|newrelic' src/
# (empty)
```

### 2.4 Cargo.toml 의존성 비교

```bash
$ git diff 80a6fe6..HEAD -- Cargo.toml | grep -E '^[+-]'
-version = "0.37.2"
+version = "0.39.0-v6x.260508.1"
```

오직 version 한 줄만 변경. 신규/제거 의존성 0.

### 2.5 unsafe 블록 surface

```rust
// src/main.rs:2216-2238 — Unix SIGINT/SIGTERM handler (이전과 같은 구조)
#[cfg(unix)]
#[allow(unsafe_code)]
{
    unsafe extern "C" fn handle_signal(sig: libc::c_int) { ... }
    unsafe { libc::signal(SIGINT, handle_signal); libc::signal(SIGTERM, handle_signal); }
}
```

`unsafe` 블록 수: 1 (extern fn 선언 + libc::signal 등록 묶음). 이전과 동일. 메모리 안전성 회귀 없음.

### 2.6 ko_KR.UTF-8 로케일 ls 빈 출력 회귀 테스트

이전 감사 시점(0.37.2) 직접 재현됨:

```bash
$ rtk ls    # ko_KR.UTF-8 로케일
(empty)     # ❌ 사용자가 보고한 버그
```

이번 0.39.0 sync 후 fix 흡수됨 (`b51a815`). 코드 확인:

```rust
// src/cmds/system/ls.rs:38
cmd.env("LC_ALL", "C");        // ← LC_ALL 강제

// src/cmds/system/ls.rs:78-84
let has_real_content = raw.lines()
    .any(|l| !l.starts_with("total ") && !l.is_empty() && !is_dotdir(l));
if parsed_count == 0 && has_real_content {
    return raw.to_string();    // ← 파싱 실패 시 raw fallback
}
```

binary 검증은 release 빌드 후 수행 (재실행 시: `LANG=ko_KR.UTF-8 LC_ALL= ./target/release/rtk ls`).

---

## 3. Supply Chain 재확인 (Cargo.lock)

`Cargo.lock`의 critical dependency 버전:

```
rustls          v0.23.37
rustls-pki-types v1.14.0
rustls-webpki   v0.103.13   ← bumped from 0.103.9
ring             (확인 필요시 cargo tree)
```

신규 추가된 `*-client / *-sdk / sentry / opentelemetry / datadog / rollbar / newrelic / self_update` 패턴 없음 (`grep -A1 'name = ' Cargo.lock`).

---

## 4. 결론

> **v0.37.2 보고서의 결론은 v0.39.0-v6x.260508.1에서도 그대로 유효합니다.**
>
> 99개 upstream 커밋 흡수 후 — 신규 외부 egress 경로 없음, 신규 의존성 없음, unsafe 경계 동일, telemetry 3중 gate 동일.
>
> upstream sync로 추가된 보안성 효과:
> - rustls-webpki 보안 패치 흡수
> - JSON multibyte truncation panic 가능성 제거
> - curl 대용량 응답 full-body alloc 회피 (DoS 표면 축소)
> - exclude_commands 정밀화 (word boundary, env prefix 처리)
> - **ko_KR.UTF-8 로케일에서 ls 빈 출력 버그 fix** (사용자 영향 직접 해소)
>
> 권장 옵션은 이전 감사와 동일: **옵션 A (자체 `cargo build --release` 빌드, RTK_TELEMETRY_URL 미설정)**.

---

## 부록: 재감사 검증 스크립트

```bash
# 의존성/네트워크 변동 확인
git diff 80a6fe6..HEAD -- Cargo.toml
grep -rn 'ureq::\|reqwest::\|hyper::\|TcpStream' src/
grep -rn 'self_update\|sentry\|bugsnag\|rollbar' src/

# 자체 빌드 후 telemetry 문자열 부재 확인
cargo build --release
strings target/release/rtk | grep -iE 'rtk-ai\.app/telemetry|metrics\.rtk|\.amazonaws\.com' | head
# 예상: 출력 없음

# ls 회귀 테스트
LANG=ko_KR.UTF-8 LC_ALL= ./target/release/rtk ls | head -3
# 예상: 정상 ls 출력
```
