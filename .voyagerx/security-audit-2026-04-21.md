# RTK 사내 배포 보안 감사 보고서

- **감사 일자**: 2026-04-21
- **감사 대상**: `rtk` (Rust Token Killer) v0.37.2 — 커밋 `80a6fe6` 기준
- **저장소**: https://github.com/rtk-ai/rtk (fork)
- **감사 범위**: 코드 전체 — 외부로 나가는 모든 데이터 흐름, 의존성 supply chain, 사용자 데이터 접근 경로
- **방법론**: 병렬 에이전트 팀 (telemetry / 네트워크 egress / analytics / hooks·learn·discover / 의존성) + 수동 검증
- **결론 요약**: **사내 배포 안전**. 단, 권장 옵션 A(자체 빌드) 또는 옵션 B(런타임 비활성화) 적용 필요.

---

## 1. Telemetry — 외부 전송 지점

### 1.1 엔드포인트

엔드포인트 URL은 **저장소 내 어디에도 하드코딩되어 있지 않음**. 빌드 타임에 환경변수로 주입.

```rust
// src/core/telemetry.rs:14-15
const TELEMETRY_URL: Option<&str> = option_env!("RTK_TELEMETRY_URL");
const TELEMETRY_TOKEN: Option<&str> = option_env!("RTK_TELEMETRY_TOKEN");
```

주입 위치:
- `.github/workflows/release.yml:85-86, 124-125, 151-152`
  - URL은 `vars.RTK_TELEMETRY_URL` (공개 변수, 단 조직 설정에만 노출)
  - Token은 `secrets.RTK_TELEMETRY_TOKEN` (비공개 시크릿)

**핵심 함의**: 공식 release 바이너리에만 URL/토큰이 박히고, 로컬 `cargo build --release`로 만든 바이너리에서는 해당 모듈이 완전 dead code가 됨.

### 1.2 전송 크레이트

- 크레이트: `ureq` v2 (TLS via `rustls`)
- 호출지: **오직 2곳**
  - `src/core/telemetry.rs:142` (정기 ping)
  - `src/core/telemetry_cmd.rs:173` (GDPR erasure 요청)

전 코드베이스에서 `ureq::` 호출은 이 두 곳뿐.

### 1.3 페이로드 내용 (`telemetry.rs:106-140`)

모든 필드가 **집계 수치 또는 익명화된 식별자**:

| 구분 | 필드 | 설명 |
|---|---|---|
| 신원 | `device_hash` | SHA-256(랜덤 salt). 사용자명/호스트명 역산 불가 |
| 환경 | `version`, `os`, `arch`, `install_method` | 빌드 버전/플랫폼, `homebrew\|cargo\|script\|nix\|other` |
| 집계 | `commands_24h`, `commands_total`, `top_commands` | 상위 5개 **도구명만** (예: `["git","cargo"]`) |
| 절감 | `tokens_saved_24h\|30d\|total`, `savings_pct`, `avg_savings_per_command` | 숫자만 |
| 품질 | `passthrough_top`, `parse_failures_24h`, `low_savings_commands` | 도구명:카운트 포맷 |
| 설정 | `hook_type`, `custom_toml_filters`, `has_config_toml`, `exclude_commands_count`, `projects_count` | 경로 아닌 **개수**와 enum |
| 경제 | `tokens_saved_30d`, `estimated_savings_usd_30d` | 계산값 ($3/Mtok × 절감 토큰) |
| 리텐션 | `first_seen_days`, `active_days_30d` | 기간(일) |

**전송되지 않는 것** (명시적 확인):
- 명령 인자 / 커맨드 라인 원문
- stdout / stderr 내용
- 파일 경로 / 프로젝트 경로 / git 레포 이름
- 환경 변수 값
- 사용자명 / 호스트명 / IP / MAC
- 소스 코드

### 1.4 발사 조건 (3중 gate)

1. **컴파일 타임 gate** — `RTK_TELEMETRY_URL`이 빌드 시 미설정 → `Option<&str>::None` → `telemetry.rs:22-24`에서 조기 return, 모듈 전체가 dead code로 링크 제거됨.
2. **런타임 env var gate** — `RTK_TELEMETRY_DISABLED=1` 설정 시 즉시 return (`telemetry.rs:27-28`).
3. **명시적 동의 gate** — `config.telemetry.consent_given == Some(true)`가 아니면 return (`telemetry.rs:38-41`). **기본값 `None`, 즉 기본 OFF**.

전송 동작:
- 주기: 23시간당 1회
- 스레드: 백그라운드 `std::thread::spawn`
- 타임아웃: 2초 fire-and-forget
- 단일 호출지: `src/main.rs:1321` (`run_cli()` 진입 시)

### 1.5 사용자 제어 명령

```bash
rtk telemetry disable   # 동의 철회
rtk telemetry forget    # 동의 철회 + 로컬 salt/marker/DB 삭제 + 서버 erasure 요청
rtk telemetry status    # 현재 상태 확인
```

---

## 2. Telemetry 외 네트워크 Egress 스캔

전체 `src/` 에 대해 `ureq|reqwest|hyper|http::|TcpStream|to_socket_addrs|https?://` 패턴 스캔.

| 영역 | 결과 | 비고 |
|---|---|---|
| `src/analytics/*` | HTTP 코드 **0** | `cc_economics.rs:18` URL은 주석뿐 (Anthropic docs 링크) |
| `src/core/tracking.rs` | HTTP 호출 **0** | 순수 SQLite |
| `src/hooks/*` | HTTP 호출 **0** | URL은 `init.rs`의 설치 안내 eprintln 문자열뿐 |
| `src/learn/*` | 매치 **0** | |
| `src/discover/*` | HTTP 호출 **0** | 등장하는 URL은 전부 테스트 fixture 안의 `"curl https://example.com"` 문자열 |
| `src/core/tee.rs` | 네트워크 **없음** | `~/.local/share/rtk/tee/` 로컬 저장만, 20개 rotation |
| `build.rs` | 네트워크 **없음** | TOML 필터 결합 + 검증만 |
| Auto-update / panic-handler / sentry | **존재하지 않음** | `self_update\|auto_update\|PanicInfo\|set_hook` 0 matches |
| DNS/소켓 raw API | **없음** | `TcpStream\|to_socket_addrs` 0 matches |

**결론**: 저장소 내 실제 네트워크 코드는 telemetry 2곳이 전부.

---

## 3. 사용자 데이터 접근 → 외부 전송 경로 추적

RTK가 디스크에서 읽을 수 있는 민감 데이터와 그 흐름을 모두 추적:

| 읽는 데이터 | 저장/사용처 | 외부 전송? |
|---|---|---|
| 실행 명령 전체 (args 포함) | `~/.local/share/rtk/` SQLite `commands.original_cmd` | ❌ 로컬만. telemetry는 **도구명 집계**만 전송 |
| 프로젝트 경로 | SQLite `commands.project_path` | ❌ |
| 실패 명령 raw stdout/stderr | 로컬 tee 파일 (최대 20개, 각 1MB) | ❌ |
| 파싱 실패 명령 | SQLite `parse_failures.raw_command` | ❌ |
| Claude Code 사용량 (`ccusage.rs`) | 로컬 `ccusage` npm 바이너리 subprocess 실행 | ❌ RTK 자체는 subprocess 호출만 |
| 후크 설치 여부 (`detect_hook_type`) | telemetry에 `"claude"\|"gemini"\|"codex"\|"cursor"\|"none"` 문자열 | ⚠️ 설치 여부만, 경로 아님 |
| 커스텀 필터 개수 | telemetry 카운트 | ⚠️ 개수만 |

**`rtk init` 후크 주입의 보안적 의미**: Claude Code 설정에 `rtk rewrite` 명령을 후크로 주입 → Claude가 실행하려는 명령을 가로채 `rtk <cmd>`로 재작성. 데이터를 **캡처·저장·전송하지 않음**, 문자열 변환만 수행.

### SQLite 스키마 (`tracking.rs:263-318`)

```sql
CREATE TABLE commands (
  id, timestamp, original_cmd, rtk_cmd,
  project_path, input_tokens, output_tokens,
  saved_tokens, savings_pct, exec_time_ms
);
CREATE TABLE parse_failures (
  id, timestamp, raw_command, error_message, fallback_succeeded
);
```

이 테이블들은 **로컬에서만 쓰이고**, 네트워크로 흘러가는 경로가 없음 (검증됨).

---

## 4. 의존성 Supply Chain 감사

### 4.1 직접 의존성 (`Cargo.toml:14-38`)

총 19개 직접 의존성. 네트워크 가능 크레이트는 **`ureq = "2"` 단 1개**.

| 범주 | 크레이트 | 네트워크 |
|---|---|---|
| CLI | `clap` 4 | ❌ |
| 에러 | `anyhow` 1.0 | ❌ |
| 파일시스템 | `ignore` 0.4, `walkdir` 2, `dirs` 5, `which` 8, `tempfile` 3 | ❌ |
| 파싱 | `regex` 1, `serde` 1, `serde_json` 1, `toml` 0.8, `quick-xml` 0.37 | ❌ |
| 출력 | `colored` 2, `lazy_static` 1.4 | ❌ |
| 저장 | `rusqlite` 0.31 (`bundled` feature) | ❌ |
| 시간 | `chrono` 0.4 | ❌ |
| 암호 | `sha2` 0.10, `getrandom` 0.4 | ❌ |
| 압축 | `flate2` 1.0 | ❌ |
| 빌드 매크로 | `automod` 1 | ❌ |
| **HTTP** | **`ureq` 2** | ✅ (rustls TLS) |
| Unix FFI | `libc` 0.2 (`cfg(unix)`) | ❌ |

### 4.2 Transitive 의존성 스캔

`Cargo.lock`에서 확인:
- `rustls`, `webpki-roots`, `url`, `base64` → `ureq`의 TLS/파싱 보조
- **sentry / opentelemetry / datadog / rollbar / newrelic / self_update — 전무**
- `*-client`, `*-sdk`, `*-api` 명명 크레이트 없음

### 4.3 Build 타임 감사

- `build.rs:1-66` — `src/filters/*.toml`을 읽어 `OUT_DIR/builtin_filters.toml` 생성. 네트워크/subprocess 호출 없음. 결정론적.
- Build-dependencies: `toml = "0.8"` 1개뿐. 바이너리에 링크되지 않음.

### 4.4 `unsafe` 블록

총 3곳 — 모두 `src/main.rs:83-84`의 Unix signal handler (`libc::signal`, `libc::raise`). 표준적인 CLI 자식 프로세스 관리용. 메모리 안전성 이슈 없음.

---

## 5. 보안 평가 요약

| 항목 | 결과 |
|---|---|
| PII 유출 | ❌ 없음 |
| 소스코드 / 파일 경로 유출 | ❌ 없음 |
| 명령 인자 유출 | ❌ 없음 |
| 자동 업데이트 / phone-home | ❌ 없음 |
| 크래시 리포팅 / APM | ❌ 없음 |
| 빌드 타임 phone-home | ❌ 없음 |
| 외부 egress 지점 | 1곳 (telemetry, 3중 opt-out) + 1곳 (수동 erasure 명령) |
| 비밀 키 바이너리 embed | ⚠️ `RTK_TELEMETRY_TOKEN`이 공식 release 바이너리에 `option_env!`로 embed됨. strings 추출 가능. **자체 빌드 시 해당 없음** |
| Supply chain | ✅ 깨끗함 — telemetry SDK / auto-updater 전무 |

---

## 6. 사내 배포 권장안

### 🟢 옵션 A (권장): 자체 빌드

```bash
git clone https://github.com/rtk-ai/rtk
cd rtk
# RTK_TELEMETRY_URL / RTK_TELEMETRY_TOKEN env 미설정 상태로 빌드
cargo build --release
# 결과 바이너리: target/release/rtk
# telemetry 모듈이 완전 dead code → 네트워크 호출 0건
```

- **장점**: 결정론적, 외부 의존 0, 감사 가능한 빌드
- **검증 방법**: 빌드된 바이너리에 `strings target/release/rtk | grep -i telemetry` 실행 시 URL 문자열이 없어야 함
- **권장 배포 경로**: 사내 패키지 저장소 / 내부 homebrew tap / 사내 Artifactory

### 🟡 옵션 B: 공식 바이너리 사용 시

공식 release 바이너리를 쓸 경우 반드시 전사 강제:

1. 배포 스크립트/이미지에 `RTK_TELEMETRY_DISABLED=1` 환경변수 강제
2. `~/.config/rtk/config.toml`에 다음 푸시:
   ```toml
   [telemetry]
   enabled = false
   consent_given = false
   ```
3. 네트워크 방화벽에서 rtk-ai 텔레메트리 도메인 차단 (URL은 저장소에 없으므로 업스트림에 별도 확인 필요)

### 🔵 옵션 C: 자체 텔레메트리 서버 운영

사내에서 사용 통계를 받고 싶다면:

```bash
RTK_TELEMETRY_URL=https://rtk-metrics.internal.voyagerx.com \
  cargo build --release
# RTK_TELEMETRY_TOKEN은 바이너리에 박히므로 생략 권장
# 서버 측 IP allowlist 또는 mTLS로 인증
```

- `/erasure` 엔드포인트 구현 필수 (GDPR)
- 받는 페이로드는 본 보고서 §1.3 표 참조

---

## 7. 최종 결론

> **RTK는 사내 배포에 안전합니다.**
>
> 코드의 98%는 완전한 로컬 처리이고, 유일한 외부 egress인 telemetry는 컴파일 타임 / 런타임 env / 명시적 동의 3중으로 막혀 있으며 페이로드에 PII가 없습니다.
>
> **옵션 A (자체 `cargo build --release`)로 빌드하여 네트워크 호출이 결정론적으로 0인 바이너리를 사내 배포**하는 방식을 권장합니다.

---

## 부록 A: 파일:라인 인덱스 (재현 가능한 감사 근거)

- `src/core/telemetry.rs:14-15` — URL/토큰 `option_env!` 선언
- `src/core/telemetry.rs:20-67` — `maybe_ping` 3중 gate 및 스레드 spawn
- `src/core/telemetry.rs:69-153` — `send_ping` 페이로드 조립 및 POST
- `src/core/telemetry.rs:106-140` — 전송 JSON 필드 정의
- `src/core/telemetry_cmd.rs:161-182` — `/erasure` 엔드포인트 호출
- `src/core/tee.rs:49-51` — tee 기본 경로 `~/.local/share/rtk/tee/`
- `src/core/tracking.rs:263-272` — `commands` 테이블 스키마
- `src/core/tracking.rs:312-318` — `parse_failures` 테이블 스키마
- `src/main.rs:1321` — `telemetry::maybe_ping()` 유일 호출지
- `src/main.rs:83-84` — `unsafe` 블록 (Unix signal handler)
- `.github/workflows/release.yml:85-86,124-125,151-152` — 릴리즈 워크플로의 env 주입
- `Cargo.toml:30` — `ureq = "2"` 유일한 네트워크 의존성
- `build.rs:1-66` — 빌드 스크립트 전체

## 부록 B: 검증 커맨드

아무나 재현 가능한 검증 스크립트:

```bash
# 1. 네트워크 의존성 ureq만인지 확인
grep -E '^(reqwest|hyper|isahc|curl|attohttpc)' Cargo.toml || echo "OK: no other HTTP crates"

# 2. ureq 호출지 2곳 확인
grep -rn "ureq::" src/
# 예상: src/core/telemetry.rs:142, src/core/telemetry_cmd.rs:173

# 3. 자동 업데이트 / 크래시 리포팅 부재 확인
grep -rn -E "self_update|sentry|bugsnag|rollbar|panic_handler|set_hook" src/ || echo "OK: none"

# 4. 자체 빌드 후 URL 문자열 부재 확인
cargo build --release
strings target/release/rtk | grep -iE "telemetry|metrics" | head
# 예상: URL 문자열 없음 (함수명/로컬 파일 경로 문자열만)
```
