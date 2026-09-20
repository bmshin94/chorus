# Chorus 전수조사 & 활용·수익화 분석 (한국어)

> 이 문서는 `chorus` 저장소를 전수조사한 결과와, 설치·활용법·수익화 아이디어까지
> 논의한 내용을 정리한 기록입니다.

## 관련 링크

| 구분 | 주소 |
|---|---|
| 원본 저장소 (upstream) | <https://github.com/chorus-codes/chorus> |
| 이 저장소 (fork) | <https://github.com/bmshin94/chorus> |
| 공식 웹사이트 | <https://chorus.codes> |
| npm 패키지 | <https://www.npmjs.com/package/chorus-codes> |
| 이슈 트래커 | <https://github.com/chorus-codes/chorus/issues> |
| 디스커션 | <https://github.com/chorus-codes/chorus/discussions> |
| X (트위터) | <https://x.com/ChorusCodes> |

---

## 1. Chorus란 무엇인가

### 한 줄 정의

> **AI가 쓴 코드를, 다른 회사 AI 2~3개에게 동시에 교차 검수시키는 로컬 오케스트레이터.**

`package.json`의 공식 설명:

> *"Driver-agnostic multi-LLM peer review for code decisions. Bring your own CLI;
> Chorus convenes 2-4 other LLMs to review the work before you ship."*

### 기본 정보

| 항목 | 내용 |
|---|---|
| 이름 | Chorus (npm: `chorus-codes`) |
| 조사 시점 버전 | v0.8.65 |
| 제작 | 99x Agency |
| 라이선스 | Apache-2.0 (상업적 이용 가능) |
| 언어 | TypeScript |
| 요구 런타임 | Node.js 20+ |
| 기술 스택 | Next.js 16 · React 19 · Fastify 5 · libsql(SQLite) · Zod · Tailwind 4 |

### GitHub 지표 (2026-09 조사 시점)

| 지표 | 값 |
|---|---|
| Stars | 533 |
| Forks | 56 |
| 저장소 생성일 | 2026-04-30 |
| 최근 푸시 | 2026-08-13 |
| 열린 이슈 | 1 |
| 토픽 | `claude-code`, `codex`, `llm` |

> 저장소 생성 약 4개월 반 만에 500스타를 넘겼고, 열린 이슈가 1개뿐일 만큼
> 유지보수가 활발합니다.

---

## 2. 해결하려는 문제

README가 제시하는 문제의식은 세 가지입니다.

1. **AI 코딩 도구는 자신만만하게 틀린다.** 약 5% 확률로 미묘하게 틀리는데,
   프로덕션에 올라가기 전까지 발견되지 않는 경우가 많다.
2. **코드를 쓴 모델은 자기 사각지대를 못 본다.** GPT가 쓴 코드를 GPT에게
   리뷰시키는 것은 연극이다 — 같은 학습 데이터, 같은 편향.
3. **API 키로 멀티 LLM 리뷰를 돌리면 비용이 빠르게 커진다.**
   diff × 리뷰어 수 × 토큰 과금.

### Chorus의 해법

- **다른 벤더끼리 교차 검증** — Claude가 작성하고 GPT·Gemini가 검수. 서로의
  사각지대를 덮는다. 의견 불일치 자체가 머지 전 경고 신호가 된다.
- **이미 결제 중인 구독을 재활용** — Claude Pro / ChatGPT Plus / Gemini Advanced의
  CLI를 헤드리스로 구동하므로 리뷰 추가 비용이 사실상 0원.
- **완전 로컬 실행** — 새로운 제3자 벤더로 코드가 전송되지 않는다.

---

## 3. 아키텍처

```
사용자 → Cockpit(:5050 웹 UI) ↔ Daemon(:7707 로컬 서버) → SQLite(~/.chorus/chorus.db)
                                      │
                                      ├─ spawn → Claude Code  (writer)
                                      ├─ spawn → Codex / GPT   (reviewer)
                                      └─ spawn → Gemini CLI    (reviewer)

  다른 AI 툴 ──(MCP stdio)──→ Daemon
```

핵심은 **Chorus 자신이 AI가 아니라는 점**입니다. AI들을 불러 모아 회의를
진행하는 사회자 역할이며, 그래서 이름이 "Chorus(합창단)"입니다.

### 데이터 흐름 상세

1. 웹 UI에서 제출 → `POST /chats` → `chats` 테이블에 행 INSERT
2. `runner/doer.ts` 시동 — `prompt-builder.ts`가 [페르소나 + 작업 + 첨부]를 합성,
   임시 작업 폴더에 프롬프트 파일 기록, `agents/<벤더>.ts`가 실행 명령을 조립,
   `headless.ts`가 실제 subprocess를 spawn
3. CLI가 stream-JSON을 stdout으로 방출 → `agents/parsers/`가 실시간 파싱 →
   SSE로 웹 UI 스트리밍 + `phase_events` 테이블에 출력·비용·토큰 기록
4. 작성 완료 → `runner/reviewer.ts`가 리뷰어들을 병렬 spawn
   (`cli-semaphore.ts`가 동시 실행 수 제한, `run-with-fallback.ts`가 실패 시 대체)
5. `runner/verdict.ts`가 템플릿의 정족수 규칙과 대조해 판정
6. 통과 → `ship.ts`(브랜치·커밋·푸시·`gh pr create`) / 거부 → 피드백 정리 후
   `maxRounds`까지 재시도 / 한도 초과 → `blocked`

---

## 4. 폴더 구조 분석

| 경로 | 역할 |
|---|---|
| `src/daemon/` | Fastify 로컬 서버(:7707). 모든 실행과 상태 관리의 중심 |
| `src/daemon/agents/` | **CLI 어댑터(shim)** — `claude.ts`, `codex.ts`, `gemini.ts`, `kimi.ts`, `grok.ts`, `opencode.ts`, `antigravity.ts`, `openrouter.ts`, `local.ts` |
| `src/daemon/agents/parsers/` | 벤더별 stream-JSON을 공통 `AgentEvent`로 변환 |
| `src/daemon/agents/sandbox-guard.ts` | 권한 경계 강제 |
| `src/daemon/orchestrators/` | Claude Code·Codex·Gemini·OpenCode·Kimi·Cursor·Windsurf 설정 파일에 Chorus를 MCP 서버로 자동 등록 |
| `src/daemon/runner/` | 실행 엔진 — `doer.ts`, `reviewer.ts`, `verdict.ts`, `prompt-builder.ts`, `run-with-fallback.ts`, `fallback-registry.ts`, `prior-round.ts` |
| `src/daemon/ship.ts` | 승인 후 브랜치·커밋·푸시·PR 생성 (자동 머지는 의도적으로 하지 않음) |
| `src/daemon/tmux.ts` | 대화형 CLI를 tmux 세션에 띄워 화면을 스크레이핑하는 모드 |
| `src/daemon/cli-semaphore.ts` | 동시 실행 개수 제한 |
| `src/daemon/reaper.ts` | 타임아웃·고아 프로세스 정리 |
| `src/daemon/error-detector.ts` | 쿼터 초과·인증 실패·레이트리밋 패턴 인식 |
| `src/daemon/routes/` | REST API (chats, templates, personas, voices, settings, stats, openrouter) |
| `src/app/` | Next.js 16 기반 Cockpit 웹 UI(:5050) |
| `src/components/` | React 19 컴포넌트 (`live-run-real/`, `run-viewer/`, `template-dialog/`, `persona-dialog/`, `phase-editor/`, `ui/`) |
| `src/mcp/` | MCP stdio(JSON-RPC) 서버 — 툴 9종 |
| `src/cli/commands/` | `init`, `start`, `stop`, `status`, `doctor`, `diagnose`, `update`, `audit`, `quickstart` |
| `src/lib/db/` | SQLite 스키마 및 접근 계층 |
| `src/lib/` | `cli-detect.ts`, `cli-precheck.ts`, `cli-paths.ts`, `voices.ts`, `model-pricing.ts`, `atomic-write.ts`, `telemetry.ts` 등 |
| `templates/` | 리뷰 패턴 YAML 6종 |
| `prompts/personas/` | 리뷰어 페르소나 마크다운 10종 |
| `tests/` | vitest 테스트 40개 이상 (커버리지 목표 80%) |
| `docs/` | v0.5 스펙 문서, 이미지, `integrating-a-new-cli.md` |

---

## 5. 핵심 개념 5가지

### ① Lineage (혈통)

어느 벤더의 AI인지를 나타내는 식별자:
`anthropic` · `openai` · `google` · `moonshot` · `grok` · `opencode` ·
`antigravity` · `openrouter` · `local` · `any`

템플릿의 `crossLineage: true` 옵션이 **"리뷰어는 작성자와 다른 벤더여야 한다"**를
강제하며, 이것이 Chorus 철학의 핵심입니다.

### ② Voice (보이스)

`lineage + 모델명`의 조합 = 사용 가능한 AI 한 명.
`src/lib/voices.ts`가 부팅 시 호스트를 스캔해 목록을 자동 구성하고,
설치되지 않은 CLI는 자동 비활성화합니다.

### ③ Template (템플릿)

회의 진행 시나리오를 담은 YAML. 예시:

```yaml
doer:
  lineage: anthropic
  models: [claude-opus-5]
reviewer:
  require: 2                # 정족수
  crossLineage: true        # 교차 벤더 강제
  candidates:
    - { lineage: openai, models: [gpt-5.5] }
    - { lineage: google, models: [gemini-3.1-pro-preview] }
iterate:
  maxRounds: 2
  onDisagreement: continue
```

내장 템플릿:

| 템플릿 | 구성 |
|---|---|
| `code-review` | 작성자 1 + 리뷰어 2, 2/2 만장일치 |
| `tri-review` | 작성자 1 + 리뷰어 3, 2/3 정족수 |
| `architect-review` | 설계안을 3개 벤더가 비평 |
| `bug-diagnose` | 한 쪽이 가설, 다른 쪽이 반박 |
| `red-green` | TDD — 테스트 작성자는 코드를 보지 못함 |
| `review-only` | 작성자 없이 diff만 검토 |

사용자 템플릿은 `~/.chorus/templates/`에 YAML을 두면 됩니다.

### ④ Persona (페르소나)

리뷰어에게 씌우는 "모자". 프롬프트 앞에 붙는 마크다운 파일입니다.

| 페르소나 | 초점 |
|---|---|
| Sentinel | 보안 — 시크릿 노출, 인젝션, 인증 우회, 공급망 |
| Cartographer | 크로스플랫폼 (Windows vs macOS, 브라우저) |
| Accountant | 비용 회귀 (추가 DB 쿼리, API 호출) |
| Profiler | 성능 회귀 |
| Inspector / Quartermaster / Concierge / Conservator / Librarian / Translator | 각기 다른 관점 |

Sentinel 프롬프트는 "심각도 / 파일·라인 / 공격 시나리오 한 문장 / 구체적 수정 코드"를
강제하고, 마지막에 **"보안 이슈가 없으면 한 문장으로 말하고 멈춰라. 없는 문제를
지어내지 마라"**고 못 박아 두어 오탐을 억제합니다.

### ⑤ Quorum (정족수)

- `require: 2` / 후보 2명 → 만장일치 (엄격)
- `require: 2` / 후보 3명 → 2/3 다수결 (현실적)
- `agreementThreshold: 0.66` → 66% 이상 동의

---

## 6. 설치 및 사용법

### 설치

```bash
node -v                  # v20 이상 필수 (bin/chorus.mjs가 하드 게이트)
npm i -g chorus-codes    # sudo 금지
chorus init              # CLI 탐지 + MCP 자동 등록 + 템플릿 시드
chorus start --ui        # http://localhost:5050
chorus quickstart        # 30초 자가진단 (권장)
```

`sudo npm install -g`를 피해야 하는 이유: nvm/fnm/asdf 사용 시 sudo는 root의
npm prefix에 설치하지만 PATH는 사용자 prefix를 보므로, 설치는 성공해도 옛 버전이
계속 실행됩니다. EACCES가 발생하면:

```bash
npm config set prefix ~/.npm-global
export PATH="$HOME/.npm-global/bin:$PATH"
```

### 소스에서 개발

```bash
git clone https://github.com/chorus-codes/chorus.git
cd chorus && pnpm install
pnpm dev:daemon   # 데몬 :7707
pnpm dev          # 코크핏 :5050
pnpm test         # 테스트 전체
```

### `chorus init`이 수행하는 일

1. `detectAllClis()`로 claude / codex / gemini / opencode / kimi / grok /
   antigravity 탐지
2. `autoConnectAll()`이 각 에디터 설정 파일에 Chorus를 MCP 서버로 등록
   (Claude Code, Codex, Gemini CLI, OpenCode, Kimi, Cursor, Windsurf)
3. 내장 템플릿을 사용자가 보유한 CLI에 맞춰 적응시켜 DB에 시드
   (`is_complete` 플래그로 미완성 슬롯 관리)
4. CLI가 0개면 경고 — 코크핏은 정상으로 보이지만 모든 실행이 멈추기 때문

### `chorus quickstart`

소스 주석에 따르면, 런칭 후 텔레메트리에서 **설치자의 78%가 한 번도 실행하지
않았다**는 점이 확인되어 만들어진 명령입니다. 의도적으로 off-by-one 버그가 심어진
샘플 코드를 리뷰시켜 설치 상태를 즉시 검증합니다.

```js
function average(numbers) {
  let sum = 0;
  for (let i = 0; i <= numbers.length; i++) {   // off-by-one
    sum += numbers[i];
  }
  return sum / numbers.length;
}
```

### 사용 방법 3가지

**① 웹 UI** — `localhost:5050`에서 템플릿 선택 후 작업 제출, 실시간 스트리밍 관전

**② 다른 AI CLI 안에서 자연어로** (가장 편리)

```
> chorus로 main 대비 staged diff 리뷰해줘
> chorus한테 src/payments/*.ts에 architect-review 템플릿 돌려줘
```

**③ MCP 툴 직접 호출**

```jsonc
// tool: chorus.create_chat
{ "template": "code-review", "work": "main 대비 staged diff 리뷰. 경쟁 상태와 누락된 테스트 위주." }
// → { "chatId": "abc123", "url": "http://localhost:5050/runs/abc123", "status": "reviewing" }
```

### CLI 명령어

| 명령 | 역할 |
|---|---|
| `chorus init` | 1회성 초기 설정 |
| `chorus start --ui` | 데몬 + 웹 UI 시작 |
| `chorus quickstart` | 30초 자가진단 |
| `chorus stop` / `status` | 종료 / 상태 확인 |
| `chorus doctor` | "내 터미널이 보는 PATH" vs "데몬이 보는 PATH" 차이 진단 |
| `chorus diagnose` | 버그 리포트용 정보 번들 출력 (경로·키 마스킹) |
| `chorus update` | 실행 중인 바이너리 위치를 찾아 정확히 그 자리에 업데이트 |
| `chorus audit` | 감사 모드 |

### 문제 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 실행이 계속 멈춤 | CLI 미탐지 | `chorus doctor` |
| 버전이 안 바뀜 | sudo 설치 | npm prefix 재설정 |
| `codex exec`에서 MCP 차단 | Codex 헤드리스 제약 | `--dangerously-bypass-approvals-and-sandbox` (upstream 이슈 #16) |
| 부팅 중 크래시 | Node/Windows 조합 이슈 | `~/.chorus/crashes/<시각>.log` 첨부해 이슈 등록 |
| 업그레이드 후 이상 동작 | 데몬이 구버전 | `chorus stop && chorus start` (`diagnose`가 VERSION MISMATCH 표시) |

---

## 7. 플러그인인가, 스킬인가, MCP인가

**정답: MCP 서버를 내장한 독립 실행 애플리케이션.**

| 분류 | 해당 여부 | 설명 |
|---|:---:|---|
| 독립 앱 | O (주된 정체) | npm 전역 패키지. 데몬 + 웹 UI + SQLite로 단독 동작 |
| MCP 서버 | O (두 번째 얼굴) | `src/mcp/`에 stdio JSON-RPC 서버, 툴 9종 노출 |
| MCP 클라이언트 | X | 다른 도구를 MCP로 부르지 않고 **직접 subprocess로 spawn** |
| 플러그인 | 부분적 | Claude Code 플러그인 형식은 아니지만, `chorus init`이 각 에디터 설정에 자기를 등록해 플러그인처럼 느껴지게 함 |
| 스킬 | X | `SKILL.md` 형식이 전혀 아님 |

경계를 그리면:

- **에디터 ↔ Chorus 구간** = MCP
- **Chorus ↔ AI CLI 구간** = `child_process.spawn()`

`src/daemon/agents/claude.ts`는 실제로 문자열 명령을 조립합니다:

```ts
let cmd = `cd ${quotePath(opts.cwd)} && claude`;
if (opts.unsandboxed || opts.sandbox === 'full') cmd += ` --dangerously-skip-permissions`;
if (opts.model) cmd += ` --model ${quoteValue(opts.model)}`;
```

이 방식은 MCP보다 범용적입니다. 대상 CLI가 MCP를 지원하든 말든 조종할 수 있기
때문입니다.

### MCP 툴 9종

| 툴 | 역할 |
|---|---|
| `create_chat` | 리뷰 시작 (chatId + URL 반환) |
| `wait_for_chat` | 종료 상태까지 블로킹 대기 |
| `get_chat_status` | 논블로킹 폴링 |
| `cancel_chat` / `resume_chat` | 중단 / 재개 |
| `list_templates` / `list_personas` | 사용 가능 목록 조회 |
| `invoke_persona` | 단일 페르소나 실행 (멀티 리뷰어 팬아웃 생략) |
| `list_blocked` | 사람의 개입이 필요한 실행 목록 |

---

## 8. API 토큰이 필요한가

**필수 아님.** 세 가지 경로가 있습니다.

| | 구독 CLI (기본) | OpenRouter | 로컬 LLM |
|---|:---:|:---:|:---:|
| API 키 필요 | X | O | X |
| 리뷰 1회 비용 | $0 | $0.30 ~ $1.50 | $0 (전기료) |
| 모델 다양성 | 3~7 | 200+ | GPU 사양에 따름 |
| 쿼터 제한 | 있음 | 없음 | 없음 |
| 오프라인 | X | X | O |

### 경로 ① 구독 CLI — 토큰 불필요

Claude Pro / ChatGPT Plus / Gemini Advanced에 이미 로그인된 CLI를 실행할 뿐이므로,
Chorus는 API 키를 알 필요도 볼 일도 없습니다. 추가 비용 0원.

### 경로 ② OpenRouter — 토큰 필요

`src/daemon/agents/openrouter.ts`가 DB의 `secrets` 테이블에서 키를 읽어
`Authorization: Bearer <key>`로 HTTP 호출합니다. 키는 로컬
`~/.chorus/chorus.db`에만 저장됩니다.

### 경로 ③ 로컬 LLM — 토큰 불필요

모델 ID가 `local:`로 시작하면 `src/daemon/agents/local.ts`로 라우팅됩니다.
Ollama / LM Studio / vLLM 등 OpenAI 호환 엔드포인트 연결용이며, 정식 지원은
로드맵 v1.0 항목입니다.

### 유의사항

- 구독 경로는 금전 지출이 없지만 **구독 쿼터는 리뷰어 수만큼 빨리 소모**됩니다.
- 구독 CLI 자동화는 벤더 약관상 회색지대입니다. 개인 개발용은 무난하나,
  상업 서비스로 돌릴 때는 별도 검토가 필요합니다.
- 텔레메트리는 기본 ON이지만 내용물(프롬프트·경로·모델명·키)은 전송되지 않습니다.
  끄는 방법: `export CHORUS_TELEMETRY=0` 또는 `touch ~/.chorus/no-telemetry`.

---

## 9. 권한 모델

| 모드 | 코드 읽기 | 코드 쓰기 | 네트워크 | 용도 |
|---|:---:|:---:|:---:|---|
| Strict | O | X | X | 신뢰하지 않는 diff 검토 |
| Workspace (기본) | O | 작업 디렉터리 내부만 | X | 일상 사용 |
| Full | O | 어디든 | O | 본인 소유 개인 머신 |

`chorus doctor`로 각 AI 도구에 설정한 샌드박스가 실제로 적용되었는지 확인할 수
있습니다.

---

## 10. GitHub에서 주목받는 이유

1. **타이밍** — 모두가 AI로 코드를 쓰지만 검증 방법은 정립되지 않은 시점
2. **한 줄 피치의 힘** — "AI 구독료 이미 내고 있으니, 그걸로 서로 검수시키면 공짜"
3. **구조적 모순을 정확히 지적** — "같은 모델이 자기 코드를 리뷰하는 건 연극"
4. **실제로 무료** — 경쟁 제품이 월 $15~30인데 기존 구독으로 0원
5. **README 완성도** — 로고, 히어로 GIF, 스크린샷, Mermaid 다이어그램, 구체적
   사고 시나리오, 경쟁 제품 비교표, 솔직한 비용 공개
6. **유통 전략** — 새 도구를 배우게 하지 않고 Claude Code·Cursor·Windsurf·Codex·
   Gemini CLI·OpenCode·Kimi에 스며듦. MCP 확산 흐름과도 정확히 맞음
7. **프라이버시 + 오픈소스** — 코드가 새 벤더로 나가지 않음, Apache-2.0, 로컬 우선
8. **품질 신호** — 테스트 40개+, 열린 이슈 1개, `doctor`/`diagnose` 같은 운영 도구,
   "PR을 Chorus로 리뷰한다(we dogfood)"는 문구

부가적으로 **"AI들끼리 논쟁시킨다"**는 컨셉 자체가 공유되기 좋은 비주얼을 만들며,
전용 도메인(chorus.codes)과 X 계정 운영 등 마케팅 실행력도 뒷받침됩니다.

---

## 11. 로컬 에이전트 구축에 도움이 되는가

**크게 도움이 됩니다.** 두 가지 활용 방식이 있습니다.

### 방식 1: 부품으로 사용

Chorus를 띄워두고 MCP 또는 REST(`POST localhost:7707/chats`)로 호출해
"여러 AI에게 묻고 합의를 뽑는" 기능을 그대로 빌려 씁니다.

### 방식 2: 레퍼런스로 학습 (권장)

로컬 에이전트를 직접 만들 때 부딪히는 문제 대부분의 실전 해답이 들어 있습니다.

| 만들고 싶은 기능 | 참고 파일 | 배울 내용 |
|---|---|---|
| AI CLI를 코드로 실행 | `src/daemon/agents/claude.ts` | 명령 조립, 샌드박스 플래그, root 예외 처리 |
| 벤더 공통 인터페이스 | `src/daemon/agents/types.ts` | `AgentShim` 추상화로 9개 벤더 통합 |
| 헤드리스 + 스트리밍 | `src/daemon/headless.ts` | `--print --output-format stream-json`, stdin 파이프 |
| 벤더별 출력 파싱 | `src/daemon/agents/parsers/` | Anthropic/OpenAI/Google 포맷 차이 흡수 |
| CLI 설치 탐지 | `src/lib/cli-detect.ts`, `cli-paths.ts` | PATH 탐색, 폴백 경로, 수동 오버라이드 |
| 사전 인증 확인 | `src/lib/cli-precheck.ts` | 실행 전 로그인 확인으로 쿼터 낭비 방지 |
| MCP 서버 구현 | `src/mcp/index.ts`, `tools.ts` | MCP SDK + Zod 스키마 실전 예제 |
| 에디터 MCP 자동 등록 | `src/daemon/orchestrators/` | 각 에디터 설정 파일 위치·포맷 |
| 멀티 에이전트 조율 | `src/daemon/runner/` | 작성자/리뷰어 분리, 정족수, 라운드 반복 |
| 실패 시 대체 모델 | `run-with-fallback.ts`, `fallback-registry.ts` | 폴백 체인 설계 |
| 동시 실행 제한 | `src/daemon/cli-semaphore.ts` | 세마포어 |
| 좀비 프로세스 정리 | `src/daemon/reaper.ts` | 타임아웃·고아 프로세스 회수 |
| 에러 자동 분류 | `src/daemon/error-detector.ts` | 쿼터/인증/레이트리밋 패턴 인식 |
| 권한 샌드박스 | `agents/sandbox-guard.ts`, `lib/settings/permissions.ts` | 3단계 권한 구현 |
| 실시간 스트리밍 UI | `routes/chats-stream.ts` + `components/live-run-real/` | SSE 서버 + React 소비 |
| 상태 저장 설계 | `src/lib/db/schema.sql` | 에이전트 실행 기록 스키마 |
| 원자적 파일 쓰기 | `src/lib/atomic-write.ts` | 중단되어도 파일이 깨지지 않게 |
| 비용 추적 | `src/lib/model-pricing.ts` | 토큰 → 달러 환산 |
| 대화형 CLI 제어 | `src/daemon/tmux.ts` | tmux 세션 스크레이핑 |

### 주목할 만한 프로덕션 디테일

- root로 실행 중이면 `--dangerously-skip-permissions`를 제거 (Claude Code가 root에서
  해당 플래그를 거부하기 때문)
- `process.cwd()`가 ENOENT를 던질 때 homedir로 폴백 (작업 폴더가 삭제된 채
  장시간 떠 있는 MCP 서버 대응)
- 모든 subprocess에 `windowsHide` 적용 (Windows 콘솔 창 방지)
- stderr에서 홈 경로를 `~`로 마스킹하고 ssh 키 언급 줄은 제거 후 DB 저장
- `template_snapshot` 컬럼 — 템플릿을 나중에 수정해도 과거 실행 기록은 당시 설정
  그대로 재현
- Node 20 하드 게이트를 **모든 import 이전에** 배치 (npm은 `engines`를 경고만 하므로)

### 설계 원칙 3가지 (그대로 차용할 가치가 있음)

1. **어댑터 패턴** — 새 AI 도구 지원 = 파일 1개 추가
   (`docs/integrating-a-new-cli.md`에 가이드 존재)
2. **설정을 데이터로** — 동작 변경에 코드 수정이 불필요 (YAML 템플릿 + MD 페르소나)
3. **프로세스 격리** — AI마다 별도 subprocess. 하나가 죽어도 전체는 유지

---

## 12. React / PHP로 만들 수 있는가

### React — 이미 React로 만들어져 있음

`src/app/`과 `src/components/`가 **Next.js 16 + React 19** 기반입니다
(`radix-ui`, `tailwindcss 4`, `lucide-react`, shadcn/ui 규약의 `components.json`).
따라서 UI 수정·교체가 바로 가능하며, 데몬 REST API 위에 완전히 새로운 프런트엔드를
얹는 것도 가능합니다.

### PHP — 가능하지만 아키텍처 조정 필요

| 부분 | PHP 대안 | 난이도 |
|---|---|---|
| 웹 UI | Laravel + Blade / Livewire / Inertia | 쉬움 |
| REST API | Laravel 라우트 | 쉬움 |
| DB | Eloquent + SQLite/MySQL | 쉬움 |
| 설정 관리 | PHP 배열 또는 YAML 파서 | 쉬움 |
| CLI 실행 | `proc_open()`, Symfony Process | 보통 |
| 병렬 실행 | Laravel Queue + 다중 워커 | 보통 |
| 실시간 스트리밍 | SSE 또는 Laravel Reverb | 보통 |

**장벽 3가지**

1. **MCP SDK 부재** — 공식 SDK는 TypeScript·Python·Java·Kotlin·C#·Go·Rust 중심이며
   PHP는 1급 지원이 아님. 대안: (a) MCP 계층만 얇은 Node로 분리, (b) MCP를 포기하고
   REST만 제공, (c) JSON-RPC 2.0 over stdio를 직접 구현
2. **장시간 프로세스 관리** — PHP의 요청-응답 모델은 수 분간 subprocess를 붙잡고
   출력을 읽는 작업에 불리. 대안: Laravel Octane(Swoole/RoadRunner), ReactPHP/Amp,
   또는 큐 워커를 데몬처럼 상주
3. **스트리밍 파싱** — `proc_open` + non-blocking stream으로 가능하나 Node 대비
   코드가 복잡해짐

**권장 하이브리드 구조**

```
PHP (Laravel)                    Node 워커 (얇게)
- 웹 UI                    HTTP  - AI CLI spawn
- 사용자/팀/과금      ───────→   - stream-JSON 파싱
- 실행 이력 DB           큐      - MCP 서버
- REST API
```

**신규 구축 시 언어 추천 순위**

| 순위 | 언어 | 근거 |
|:--:|---|---|
| 1 | TypeScript/Node | MCP SDK 1급, AI CLI 대부분이 Node, 비동기 스트리밍 네이티브, Chorus 코드 직접 참고 가능 |
| 2 | Python | MCP SDK 1급, AI 생태계 최강, asyncio |
| 3 | Go | 동시성 우수, 단일 바이너리 배포, MCP SDK 존재 |
| 4 | PHP | 웹은 강하나 프로세스·스트리밍이 약함 — 하이브리드 권장 |

React는 UI 계층이므로 어떤 백엔드와도 결합 가능합니다.

---

## 13. 수익화 아이디어

### 전제 3가지

1. **라이선스는 문제없음** — Apache-2.0은 상업적 이용·수정·재배포·비공개 파생물을
   허용. 조건은 라이선스 사본 포함과 변경 파일 표시. 단, **상표권은 부여되지 않으므로
   "Chorus"라는 이름으로 판매할 수 없고 별도 브랜드가 필요**합니다.
2. **진짜 리스크는 구독 CLI 자동화** — 구독 약관은 개인 사용을 전제로 하는 경우가
   많습니다. 타인의 코드를 대신 리뷰해주는 유료 서비스에 본인 구독을 사용하면
   계정 정지 위험이 있습니다. **해법은 BYOK/BYOS**: "고객 본인 머신에서 고객 본인
   구독/키로 실행되며, 우리는 소프트웨어와 부가 서비스만 판매한다"는 구조.
3. **단순 재판매는 성립하지 않음** — 누구나 무료로 받을 수 있으므로,
   **Chorus에 없는 것**을 팔아야 합니다.

### A. SaaS / 제품

| # | 아이디어 | 핵심 | 가격 예시 |
|---|---|---|---|
| A-1 | **팀 대시보드** | Chorus는 철저히 1인용(`~/.chorus/chorus.db`)이며 팀 기능이 0 — 가장 큰 빈칸. 실행 이력 동기화, 팀 통계, 표준 템플릿 배포, 감사 로그 | 시트당 월 $12~20 |
| A-2 | **GitHub App PR 리뷰봇** | `ship.ts`는 PR 생성만 하고 댓글 봇은 없음. 3개 모델이 모두 지적한 항목만 경고 → **오탐 대폭 감소**가 차별점 | 리포당 월 $20 / 조직 $99~299 |
| A-3 | **한국 시장 특화 버전** | 한국어 UI·페르소나, 전자정부 표준프레임워크·개인정보보호법·금융보안 가이드 검사관, HyperCLOVA X/Solar 어댑터, 망분리 환경 지원 | 온프레미스 연 2,000만~5,000만원 |
| A-4 | **도메인 특화 에디션** | 엔진은 그대로, 페르소나·템플릿만 교체 — 보안팀용, 핀테크용, 의료용, 게임용, 임베디드(MISRA-C)용 | 월 $200~500/팀 |
| A-5 | **매니지드 호스팅(BYOK)** | 고객 키로 호스팅 실행. 단, "로컬 우선"이라는 원래 매력을 일부 포기 | 월 $29 / $99 + 사용량 |

### B. 콘텐츠 / 마켓플레이스

| # | 아이디어 | 핵심 | 가격 예시 |
|---|---|---|---|
| B-1 | **페르소나·템플릿 팩** | 페르소나가 단순 마크다운이라는 점을 활용. Frontend/Backend/Security/Architecture/Korea 팩 | $29~99, All Access $149 |
| B-2 | **강의·정보 상품** | Chorus 코드를 교재로 "AI 에이전트 실전 구축" 강의 | ₩99,000 × 300명 = 약 ₩2,970만원 |
| B-3 | **유료 뉴스레터/커뮤니티** | AI 코드 품질 주제, 모델별 성능 비교 | 월 $10 × 500명 |

### C. 통합 / 확장

| # | 아이디어 | 핵심 |
|---|---|---|
| C-1 | **GitHub Actions 통합** | PR마다 다중 AI 리뷰, 합의된 심각 이슈 시 CI 실패. 오픈소스 무료 / 프라이빗 유료 |
| C-2 | **IDE 확장** | VS Code / JetBrains, 우클릭 리뷰 + 인라인 진단 |
| C-3 | **신규 벤더 어댑터 팩** | HyperCLOVA X, Solar, A.X, Qwen, DeepSeek, Azure OpenAI, AWS Bedrock, Ollama 최적화 |
| C-4 | **Slack/Discord 봇** | `/review <PR URL>` → 채널에 합의 요약. 비개발 직군 참여 유도 |
| C-5 | **모바일 앱** | 이동 중 리뷰 결과 확인·승인 |

### D. 서비스 / 에이전시 (현금화 속도 최상)

| # | 아이디어 | 구성 | 단가 |
|---|---|---|---|
| D-1 | **기업 도입 컨설팅** | 현황 진단 → 맞춤 페르소나 설계 → CI 통합 → 교육 + 3개월 유지보수 | 500만~2,000만원 |
| D-2 | **일회성 코드 감사** | 레거시 코드베이스를 페르소나 10종으로 스캔, 종합 리포트 + 우선순위 로드맵 | 300만~1,500만원 |
| D-3 | **기업 교육** | "AI 시대의 코드 품질 관리" 1~2일 과정 | 회당 300만~800만원 |
| D-4 | **맞춤 개발(SI)** | Chorus 기반 사내 시스템 구축 | 3,000만~1억원 |

### E. 창의적 확장

| # | 아이디어 | 핵심 |
|---|---|---|
| E-1 | **코드 외 대상 검토** | Chorus 엔진은 "여러 AI에게 같은 대상을 검토시키고 합의를 뽑는 범용 기계". 계약서(독소조항), 마케팅 카피, 사업계획서, 논문, 이력서, 설계 문서 등에 적용 가능 — **합의 검증의 가치는 코드보다 오히려 이쪽에서 더 큼** |
| E-2 | **"AI 판정단" API** | LLM 출력 검증(할루시네이션 방지)을 API로 제공. `judge({content, judges, criteria, quorum})` → `{verdict, confidence, dissent}` |
| E-3 | **모델 벤치마크 서비스** | 동일 작업을 여러 모델에 돌리므로 비교 데이터가 자연히 축적. 무료 리포트 → 트래픽 → 유료 전환 |
| E-4 | **교육용 자동 채점** | 학생 제출 코드를 3개 AI가 채점·합의. 편향 감소 + 근거 제시 |

### 단계별 실행 로드맵

| 시기 | 실행 항목 | 기대 |
|---|---|---|
| **0~1개월** | Chorus 직접 사용 → 한국어 페르소나 5~10개 제작 → 블로그/영상 콘텐츠 → 페르소나 팩 판매 | 투자 0원, 월 10~50만원 + 인지도 |
| **~3개월** | 강의 제작, 컨설팅 1~2건 수주, GitHub Action 오픈소스 공개(바이럴) | 월 300~1,000만원 |
| **6~12개월** | 팀 대시보드 또는 PR 리뷰봇 MVP, 한국 특화 버전, 컨설팅 고객의 SaaS 전환 | 월 1,000만원+ |

### 권고 사항

**해야 할 것**

1. 오픈소스 코어 + 유료 애드온 전략 (Chorus 자신이 이미 이 전략)
2. BYOK/BYOS 고수 — 구독을 대신 돌려주지 말 것
3. 한국 시장 우선 — 로컬라이징과 규제 대응이 해자
4. 원저작자 존중 — Apache-2.0 표시 준수, 가능하면 업스트림 기여
5. 작게 시작해 반응을 보고 확장

**하지 말 것**

1. "Chorus" 이름 그대로 사용 (상표권 미포함)
2. "우리 구독으로 대신 실행" 모델
3. 거의 수정 없는 단순 재판매
4. 처음부터 대규모 SaaS 착수
5. 라이선스 표시 누락

**단일 추천 경로**

> 한국어 페르소나 팩 + 강의로 시작 → 그 고객군을 컨설팅으로 전환 →
> 축적된 노하우로 한국 특화 SaaS 구축.

초기 투자가 거의 없고, 피드백이 빠르며, 한국어라는 진입장벽이 글로벌 경쟁자를
차단하는 계단식 성장 경로입니다.

---

## 14. 알아둘 제약과 주의사항

- **구독 약관 회색지대** — 구독 CLI 자동화는 개인 사용 범위를 넘으면 문제될 수 있음
- **쿼터 소모** — 금전 지출은 없어도 사용량 한도는 리뷰어 수만큼 빨리 소진
- **아직 v0.8** — v1.0 이전이며 안정화가 진행 중
- **텔레메트리 기본 활성** — 설치 ID·버전·OS·업타임·24시간 실행 수만 전송,
  내용물은 전송되지 않음. `CHORUS_TELEMETRY=0`으로 비활성화 가능
- **Node 20+ 필수**, tmux 모드는 Unix 계열에서 더 원활
- **Next.js 16 최신 버전 사용** — 저장소 `AGENTS.md`에 "당신이 아는 Next.js가
  아니므로 `node_modules/next/dist/docs/`의 가이드를 먼저 읽으라"는 경고가 있음

---

## 15. 로드맵 (upstream 기준)

- [x] v0.5 — 데몬 + 코크핏 + 4개 벤더
- [x] v0.6 — MCP 서버, 페르소나 시스템
- [x] v0.7 — OpenRouter 연동, voices 테이블, 실시간 사이드바
- [x] v0.8 — 공개 런칭 + 안정화 (diagnose, crash hook, 폴백 dedup)
- [ ] v0.9 — 다단계 리뷰 (작성 → 리뷰 → 수정 → 재리뷰)
- [ ] v0.10 — 보이스별 페르소나 오버라이드, 반복 실패 시 자동 비활성화
- [ ] v1.0 — 로컬 LLM 어댑터 (Ollama / LM Studio / vLLM)

---

## 16. 요약

| 질문 | 답 |
|---|---|
| 무엇인가 | 여러 벤더의 AI CLI를 조율해 코드를 교차 검증시키는 로컬 오케스트레이터 |
| 언제 쓰나 | 머지 전 점검, 리팩터링 검증, 아키텍처 결정, 대형 PR 리뷰, TDD, 플래키 버그 추적 |
| 플러그인/스킬/MCP | MCP 서버를 내장한 독립 애플리케이션 |
| API 토큰 | 불필요 (구독 CLI 기본). OpenRouter 사용 시에만 필요 |
| 유명한 이유 | 타이밍 + 명확한 피치 + 실제 무료 + README 완성도 + 기존 도구 침투 전략 |
| 로컬 에이전트에 도움? | 매우 큼 — 멀티 에이전트 오케스트레이션의 실전 레퍼런스 |
| React/PHP 가능? | React는 이미 사용 중. PHP는 하이브리드 구조 권장 |
| 수익화 | 팀 SaaS, PR 리뷰봇, 한국 특화 버전, 페르소나 팩, 강의, 컨설팅, 코드 외 영역 확장 |
