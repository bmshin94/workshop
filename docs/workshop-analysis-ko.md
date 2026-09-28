# Raindrop Workshop 분석 정리 (대화 요약)

> 작성일: 2026-09-28
> 대상 저장소
> - 원본(Upstream): https://github.com/raindrop-ai/workshop
> - 내 저장소(Fork/복사본): https://github.com/bmshin94/workshop
> - 공식 사이트 / 문서: https://raindrop.ai , https://raindrop.ai/docs
> - 클라우드 제품: https://app.raindrop.ai

이 문서는 "이 저장소가 뭔지, 어디에 쓰는지, 나한테 어떤 도움이 되는지"를 전수조사한 대화 내용을 정리한 것입니다.
대화는 5단계(1. 분석 → 2. 쉬운 설명 → 3. Q&A → 4. 수익화 아이디어 → 5. 이 문서 저장)로 진행됐습니다.

---

## 1. 전수조사 결과: 이게 뭐야?

### 한 줄 요약
**AI 에이전트 전용 "로컬 디버거(블랙박스)"** 입니다.
내가 만든 AI 에이전트(챗봇, 자동화 봇 등)가 실행될 때 **모든 토큰, 모든 툴 호출, 모든 판단 과정**을 실시간으로 녹화해서 브라우저 화면(`http://localhost:5899`)에 보여주고,
Claude Code 같은 코딩 에이전트가 그 기록을 읽고 **버그를 찾고 → 평가(eval)를 쓰고 → 코드를 고치게** 해주는 도구입니다.

- 만든 곳: Raindrop (Invisible Tools, Inc.) — AI 제품 모니터링 스타트업
- 라이선스: **MIT** (상업적 사용·수정·재배포 자유, 저작권 표기만 유지)
- 배포 형태: `raindrop` 라는 **단일 실행 파일(CLI 바이너리)**. 현재 버전 v0.1.21 (`latest.json`)
- 지원 OS: macOS(arm64/x64), Linux(x64/arm64), Windows(x64)

### 구성 요소 (폴더별)

| 폴더/파일 | 역할 |
| --- | --- |
| `src/index.ts` | `raindrop` CLI 진입점. `workshop`, `setup`, `sync`, `update`, `uninstall`, `replay register`, `login`, `cloud setup` 등 명령어 |
| `src/server.ts` | 로컬 서버(Express, 포트 5899). 트레이스 수집(`/v1/traces`, OTLP), 실시간 이벤트(`/v1/live`), UI용 API, WebSocket 브로드캐스트 |
| `src/db.ts`, `src/db/schema.ts`, `drizzle/` | 로컬 **SQLite** DB(`~/.raindrop/raindrop_workshop.db`). 테이블: `runs`(실행 1회), `spans`(LLM 호출/툴 호출 단위), `live_events`(스트리밍 토큰), 저장된 실행, 주석 등 |
| `src/spans/` | 여러 SDK(Vercel AI SDK, Claude Agent SDK, LiveKit, Traceloop 등)의 트레이스 형식을 하나의 공통 형식으로 **정규화** |
| `src/otlp-protobuf.ts` | OpenTelemetry(OTLP) protobuf 형식 파싱 → 표준 텔레메트리도 받을 수 있음 |
| `src/mcp/` | **MCP 서버**. 코딩 에이전트가 쓰는 도구 11개: `get_current_run`, `query_traces`(SQL 조회), `get_span_payload`, `get_run_outline`, `search_run`, `get_span_context`, `annotate`, `ask_agent`, `replay_run`, `import_cloud_trace`, `show_in_ui` |
| `skills/instrument-agent` | **스킬** `/instrument-agent`: 내 에이전트 코드에 Raindrop 추적 코드를 자동으로 심어주는 작업 지침 |
| `skills/setup-agent-replay` | **스킬** `/setup-agent-replay`: 기록된 트레이스를 내 실제 에이전트 코드로 **재실행(Replay)** 할 수 있게 서버를 만들어주는 지침 |
| `src/install/` | Claude Code, Cursor, Codex, OpenCode, Amp, Windsurf 코딩 에이전트를 감지해서 MCP + 스킬을 자동 설치 |
| `src/replay.ts`, `src/agents-config.ts` | Replay 기능 (`.raindrop/agents.yaml`, 포트 61020~61044 replay 서버 호출) |
| `src/claude-cli-chat.ts`, `src/codex-cli-chat.ts`, `src/agent-chat.ts` | Workshop 화면 안에서 Claude Code / Codex 와 대화하며 트레이스 분석 |
| `src/cloud/`, `src/auth/` | Raindrop Cloud(유료 호스팅 서비스) 연결, OAuth 로그인, `RAINDROP_WRITE_KEY` 관리 |
| `src/secret-store.ts` | UI에서 입력한 OpenAI/Anthropic API 키 보관 |
| `src/drip.ts` | 굿즈(모자·우산·스티커) 신청 이벤트 터미널 UI (재미 요소) |
| `app/` | 웹 UI. **React 19 + Vite + Tailwind + React Query**. 페이지: Runs(실행 목록), Search, Saved, Settings. 컴포넌트: 플레임 타임라인, 스팬 트리, 채팅 흐름, 리플레이 뷰, 주석 칩 등 |
| `examples/` | 11개 데모 챗봇: OpenAI, Anthropic, Vercel AI SDK(v1/otel v2), Claude Agent SDK, 브라우저, Python, Go, Rust, OpenCode 플러그인, Pi agent |
| `scripts/` | 빌드(bun compile), 설치 스크립트, 데모 트레이스 시드, 마이그레이션 임베드 |

### 기술 스택
- 런타임: **Bun** (≥1.3.13), 언어: **TypeScript**
- 서버: Express + ws(WebSocket) + SQLite(drizzle-orm)
- UI: React 19 + Vite + Tailwind + shadcn 스타일 컴포넌트
- 프로토콜: OpenTelemetry(OTLP JSON/protobuf), MCP(Model Context Protocol)

### 동작 흐름
```
[내 AI 에이전트] --(Raindrop SDK / OTel 트레이스)--> [raindrop workshop 로컬 서버 :5899]
                                                          │  SQLite 저장
                                                          ├─> [브라우저 UI] 실시간 타임라인/스팬 트리
                                                          └─> [MCP] Claude Code/Cursor가 트레이스 읽고 분석·수정·재실행
```

### 언제 쓰나?
- 에이전트가 이상한 답을 했는데 **왜 그랬는지** 모를 때
- 툴 호출이 실패하거나 무한 루프를 돌 때
- 토큰/비용이 어디서 새는지 보고 싶을 때
- 프롬프트·모델을 바꿔서 **같은 상황을 재현(Replay)** 해보고 싶을 때
- Claude Code에게 "이 트레이스 보고 버그 고쳐줘"를 시키고 싶을 때

### 나한테 무슨 도움?
- LLM 앱/에이전트를 만든다면: 디버깅 시간 단축, 동작 과정 가시화
- Claude Code 사용자라면: 트레이스 기반 **자가 치유 루프**(eval 작성 → 실행 → 실패 확인 → 수정 → 재실행)
- 학습자라면: 에이전트 내부가 어떻게 동작하는지 눈으로 보며 배우기 좋음
- 개발자라면: MIT 라이선스 + React 코드 → 참고·포크·확장 가능한 좋은 교재

---

## 2. 더 쉽게 설명하면

- **자동차 블랙박스**: 에이전트가 달리는 동안 모든 걸 녹화 → 사고(버그) 나면 돌려보기
- **요리 CCTV**: 결과(맛없는 요리)만 보는 게 아니라, 재료를 언제 뭘 얼마나 넣었는지(툴 호출, 프롬프트) 다 보임
- **재방송(Replay)**: 같은 손님 주문(입력)으로 레시피(프롬프트/모델)만 바꿔 다시 요리해보기
- **AI 조수에게 CCTV 보여주기**: Claude Code가 녹화본을 보고 "여기서 소금을 너무 넣었네요" 하고 레시피(코드)까지 고쳐줌
- 전부 **내 컴퓨터 안에서** 동작(로컬). 데이터가 밖으로 안 나감 (Cloud는 선택 사항)

---

## 3. Q&A

### 3-1. 설치 및 사용법
```bash
# 1) 설치 (한 줄이면 끝, 소스 클론/빌드 불필요)
curl -fsSL https://raindrop.sh/install | bash

# 2) Workshop 실행 → 브라우저 http://localhost:5899 열림
raindrop workshop

# 3) 내 에이전트 프로젝트에서 코딩 에이전트(Claude Code 등)에게
/instrument-agent        # 추적 코드 자동 삽입

# 4) (선택) 리플레이 설정
/setup-agent-replay
```
- 에이전트 쪽 환경변수: `RAINDROP_LOCAL_DEBUGGER=http://localhost:5899/v1/` (`raindrop workshop setup` 이 `.env`에 써줌)
- 주요 명령: `raindrop workshop status | stop | reset`, `raindrop update`, `raindrop sync`, `raindrop uninstall`
- 설정 환경변수: `RAINDROP_WORKSHOP_PORT`(기본 5899), `RAINDROP_WORKSHOP_DB_PATH`
- 소스 개발(기여자용): `bun install && bun run dev` (daemon :5899, Vite UI :5900)
- 예제: `cd examples/openai-chat && bun install && bun run dev` (각 예제는 해당 LLM API 키 필요)

### 3-2. 플러그인? 스킬? MCP?
**정답: 독립 실행형 CLI 앱(로컬 서버 + 웹 UI)이고, 설치하면 MCP 서버와 스킬을 함께 깔아주는 "패키지"** 입니다.
- 본체: `raindrop` 실행 파일 (로컬 데몬 + 웹 UI)
- MCP: `raindrop workshop mcp` (stdio, 서버 이름 `workshop`) — 코딩 에이전트가 트레이스를 조회하는 통로
- 스킬: `/instrument-agent`, `/setup-agent-replay` (+ 클라우드용 `raindrop-setup`, `raindrop-investigate`)
- Claude Code "플러그인(마켓플레이스)" 형태는 아님. `raindrop setup` 이 여러 에이전트(Claude Code, Cursor, Codex, OpenCode, Amp, Windsurf)에 MCP·스킬을 직접 설치하는 방식

### 3-3. API 토큰이 필요해?
| 용도 | 필요 여부 |
| --- | --- |
| Workshop 로컬 사용 (트레이스 수집/조회) | **불필요** (계정·키 없이 무료) |
| 내 에이전트 자체 실행 | 원래 쓰던 LLM 키 필요 (OpenAI/Anthropic 등) — Workshop 때문에 추가로 필요한 건 아님 |
| Workshop 안의 AI 기능 (ask_agent, 요약, UI 채팅) | Anthropic/OpenAI 키 (Settings에 입력 또는 `ANTHROPIC_API_KEY`/`OPENAI_API_KEY`), 또는 로그인된 Claude Code / Codex CLI 활용 |
| 예제 앱 실행 | 해당 제공자 API 키 |
| Raindrop Cloud | `raindrop login`(OAuth) + `RAINDROP_WRITE_KEY` (유료 서비스) |

### 3-4. 왜 깃허브에서 유명할까? (추정)
1. **AI 에이전트 관측성(Observability)** 은 지금 가장 뜨거운 문제 — 에이전트는 블랙박스라 디버깅이 어려움
2. **설치 한 줄, 로컬, 무료, MIT** — 진입장벽이 거의 없음
3. **Claude Code/Cursor와 MCP로 연결** — "AI가 AI를 디버깅"하는 자가 치유 루프라는 신선한 콘셉트
4. **호환성**: TS/Python/Go/Rust, Vercel AI SDK, OpenAI Agents, LangChain, LangGraph, CrewAI, Mastra 등 대부분 프레임워크 지원
5. 실시간 스트리밍 UI, 리플레이 등 개발자 경험(DX)이 좋음
6. 회사(Raindrop)의 유료 Cloud로 가는 오픈소스 퍼널 전략 → 마케팅·커뮤니티 활동 활발

### 3-5. 로컬 에이전트 구축에 도움이 될까?
**많이 됩니다. 단, "만드는 도구"가 아니라 "디버깅/검증 도구"** 입니다.
- 에이전트를 만드는 건 AI SDK, LangGraph, Claude Agent SDK 등으로 하고, Workshop은 그 과정을 들여다보는 역할
- 로컬 LLM(Ollama, LM Studio 등 OpenAI 호환 엔드포인트)을 Vercel AI SDK·OpenAI SDK로 호출하면 동일하게 추적 가능
- 데이터가 로컬 SQLite에만 저장 → 사내/개인 데이터 보안에 유리
- 리플레이로 프롬프트·모델 교체 실험을 빠르게 반복 가능

### 3-6. 수익화 아이디어?
→ 4장에서 자세히

### 3-7. React나 PHP로 만들 수 있어?
- **React: 가능 (사실 이미 React)** — `app/` UI가 React 19 + Vite + Tailwind. 그대로 참고·커스터마이징 가능
- **PHP: 백엔드는 가능, 조건부**
  - 수집 API(`POST /v1/traces` JSON 수신 → DB 저장) → Laravel/Slim + MySQL/SQLite로 충분히 구현 가능
  - 실시간 스트리밍: PHP는 WebSocket이 약함 → Laravel Reverb, Swoole, Ratchet 또는 SSE/폴링으로 대체
  - OTLP protobuf 파싱: `google/protobuf` PHP 패키지 필요 (JSON만 받으면 간단)
  - MCP 서버: PHP용 SDK가 있긴 하나 TS/Python 대비 생태계가 작음
- **추천 조합**: 화면은 React, 서버는 기존 PHP(Laravel) 혹은 원본 TS 서버 재사용
- **PHP 에이전트를 추적하고 싶다면**: 공식 SDK는 TS/Python/Go/Rust + **HTTP API** → PHP에서는 HTTP API로 직접 이벤트 전송 가능 (PHP SDK를 만드는 것 자체가 기회)
- MIT 라이선스라 포크·상업적 활용 가능. 단 "Raindrop" 이름·로고(상표)는 쓰면 안 됨

---

## 4. 수익화 아이디어 상세

> 가격·시장 수치는 예시이며 실제 검증이 필요합니다.

### ① 에이전트 "설치·진단" 대행 서비스 (가장 빠르게 시작)
- **대상**: 챗봇/AI 기능을 도입했지만 품질 문제를 겪는 중소기업·스타트업
- **내용**: Workshop 연결 → 실제 트레이스 분석 → 문제점·비용 누수 리포트 → 개선안 적용
- **수익 모델**: 진단 1회 패키지 + 월 유지보수 계약
- **장점**: 개발 없이 바로 시작, 포트폴리오 쌓기 좋음

### ② PHP / Laravel용 에이전트 추적 SDK + 대시보드
- **배경**: 공식 SDK에 PHP가 없음. 국내에는 PHP(그누보드, 워드프레스, Laravel) 기반 서비스가 많음
- **내용**: `composer require` 한 줄로 LLM 호출·툴 호출을 추적하는 패키지 + Workshop/OTel 호환 전송
- **수익 모델**: 오픈소스(무료) + Pro 기능(팀 대시보드, 알림, 보관 기간) 유료, 또는 기업 기술지원 계약
- **장점**: 틈새 시장 선점, 우리 역량(PHP+React)과 정확히 일치

### ③ 한국어 특화 에이전트 품질 평가(Eval) SaaS
- **내용**: 트레이스를 모아 한국어 답변 품질(존댓말, 맞춤법, 환각, 금칙어, 개인정보 노출) 자동 채점 + 회귀 테스트
- **수익 모델**: 월 구독(트레이스 수/평가 횟수 기준 요금제)
- **차별점**: 글로벌 도구는 한국어 기준 평가가 약함

### ④ 개인정보(PII) 마스킹 · 규제 대응 레이어
- **내용**: 트레이스에 섞인 주민번호, 전화번호, 계좌번호, 주소 등을 저장 전에 자동 마스킹 + 감사 로그
- **대상**: 금융·의료·공공 등 개인정보보호법 준수가 필요한 곳
- **수익 모델**: 온프레미스 라이선스(설치형) + 연간 유지보수

### ⑤ AI 비용(토큰) 모니터링 대시보드
- **내용**: 모델별·기능별·고객별 토큰 사용량과 비용을 시각화, 이상 급증 알림, 저렴한 모델 전환 제안
- **수익 모델**: SaaS 구독, 또는 절감액의 일정 % 성과보수

### ⑥ 교육 콘텐츠 / 강의
- **내용**: "AI 에이전트 만들고 디버깅하기" 한국어 강의, 유튜브, 블로그, 전자책, 기업 교육
- **수익 모델**: 강의 판매, 기업 출강, 광고, 후원
- **장점**: 한국어 자료가 적어 선점 효과, ①~⑤의 마케팅 채널 역할도 함

### ⑦ 에이전트 개발 외주 + "품질 증빙" 패키지
- **내용**: 외주로 에이전트를 만들고, 납품 시 Workshop 트레이스·eval 결과를 품질 보증서처럼 제공
- **효과**: 단가 인상 근거, 유지보수 계약으로 연결

### 추천 실행 순서
1. 먼저 직접 써보기 (예제 1개 + 내 프로젝트 1개 연결) → 블로그/영상으로 기록 (⑥)
2. 그 경험으로 진단 대행 시작 (①) → 고객 문제 수집
3. 반복되는 문제를 제품화 (② PHP SDK 또는 ④ PII 마스킹, ⑤ 비용 대시보드)
4. 한국어 평가 기능을 붙여 SaaS 구독화 (③)

### 주의할 점
- Raindrop Cloud와 정면 경쟁보다는 **한국·PHP·규제·한국어** 같은 틈새에 집중
- MIT 라이선스 고지 유지, "Raindrop" 상표·로고 사용 금지
- 고객 트레이스에는 민감 정보가 많음 → 보안·계약(비밀유지) 필수

---

## 5. 이 문서
- 1~4단계 대화 내용을 정리해 `docs/workshop-analysis-ko.md` 로 저장하고 `main` 브랜치에 머지함
- 원본: https://github.com/raindrop-ai/workshop
- 내 저장소: https://github.com/bmshin94/workshop
