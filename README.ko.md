# AIuditor

**AI 바이브 코딩 품질 검수기** — Claude Code용 자동화 검수 에이전트

AIuditor는 AI 바이브 코딩으로 만든 웹 서비스를 종합 검수하는 멀티 에이전트 오케스트레이션 시스템입니다. 두 개의 독립된 시스템으로 구성됩니다:

- **AIuditor-Code**: 배포 전 코드 검수 (소스코드 보안, 의존성 CVE, 빌드 검증, 설정 검증, 타입 안전성)
- **AIuditor-Live**: 배포 후 라이브 웹사이트 검수 (보안, 성능, 접근성, SEO, 스트레스 테스트)

> [!NOTE]
> AIuditor 에이전트는 [Claude Code](https://claude.ai/claude-code) 안에서 커스텀 에이전트(`~/.claude/agents/`)로 실행됩니다.

---

## 아키텍처

### AIuditor-Code — 배포 전 코드 검수 (83개 테스트)

**로컬 코드베이스**를 파일 분석, CLI 도구, 로컬 서버 테스트로 검수합니다.

```
                    ┌──────────────────────────┐
                    │      aiuditor-code       │
                    │   (Orchestrator, Opus)   │
                    └────────────┬─────────────┘
                                 │
              Phase 0: Codebase Recon (direct)
                                 │
          ┌──────────────────────┼─────────────────────┐
          │                      │                     │
┌─────────▼──────────┐ ┌─────────▼────────┐ ┌──────────▼─────────┐
│ aiuditor-code      │ │ aiuditor-code    │ │ aiuditor-code      │
│   -secrets         │ │   -deps          │ │   -config          │
│ SAST (16 tests)    │ │ CVE Audit (10)   │ │ Env/DB (14)        │
│     (Sonnet)       │ │     (Sonnet)     │ │     (Sonnet)       │
└─────────┬──────────┘ └─────────┬────────┘ └──────────┬─────────┘
          │              ┌───────▼──────┐              │
          │              │ aiuditor-code│              │
          │              │   -types     │              │
          │              │ TS + Quality │              │
          │              │  (14 tests)  │              │
          │              │   (Sonnet)   │              │
          │              └───────┬──────┘              │
          └──────────────────────┼─────────────────────┘
                   Batch 1: 4 agents in parallel
                        (read-only, lightweight)
                                 │
               ┌─────────────────┴─────────────────┐
               │                                   │
     ┌─────────▼──────────┐             ┌──────────▼─────────┐
     │ aiuditor-code      │             │ aiuditor-code      │
     │   -build           │             │   -localhost       │
     │ Bundle (12 tests)  │             │ Smoke (8 tests)    │
     │     (Sonnet)       │             │     (Sonnet)       │
     └─────────┬──────────┘             └──────────┬─────────┘
               └─────────────────┬─────────────────┘
                   Batch 2: 2 agents in parallel
                     (build + server side-effects)
                                 │
                    ┌────────────▼─────────────┐
                    │    Report Generation     │
                    │     MD + HTML + PDF      │
                    └──────────────────────────┘
```

| 에이전트 | 테스트 수 | 영역 |
|---------|----------|------|
| aiuditor-code-secrets | 16 | 하드코딩 API 키, 비밀번호, 개인키, eval(), XSS 벡터, SQL 인젝션 패턴 |
| aiuditor-code-deps | 10 | npm audit CVE, 잠금파일 무결성, 라이선스 호환성, 미사용 의존성 |
| aiuditor-code-config | 14 | .env 완전성, DB 마이그레이션 안전성, CORS/CSP, 인증 미들웨어, 레이트 리미팅 |
| aiuditor-code-types | 14 | TypeScript 컴파일, ESLint, console.log 잔존, 테스트 커버리지, 코드 품질 |
| aiuditor-code-build | 12 | 빌드 성공, 번들 사이즈, 소스맵, 데드코드, 이미지 최적화 |
| aiuditor-code-localhost | 8 | 개발 서버 기동, 라우트 접근성, API 스모크 테스트, 콘솔 에러 |

---

### AIuditor-Live — 배포 후 검수 (151개 테스트)

**배포된 라이브 URL**을 HTTP 요청과 Playwright 브라우저 자동화로 검수합니다.

```
                    ┌──────────────────────────┐
                    │      aiuditor-live       │
                    │   (Orchestrator, Opus)   │
                    └────────────┬─────────────┘
                                 │
              Phase 0: Reconnaissance (direct)
                                 │
          ┌──────────────────────┼─────────────────────┐
          │                      │                     │
┌─────────▼──────────┐ ┌─────────▼────────┐ ┌──────────▼─────────┐
│ aiuditor-live      │ │ aiuditor-live    │ │ aiuditor-live      │
│   -render          │ │   -security      │ │   -infra           │
│ Rendering          │ │ OWASP Top 10     │ │ SSL/TLS + DNS      │
│ + Cross-Browser    │ │ + Sessions       │ │ + Integrity        │
│     (Sonnet)       │ │     (Sonnet)     │ │     (Sonnet)       │
└─────────┬──────────┘ └─────────┬────────┘ └──────────┬─────────┘
          └──────────────────────┼─────────────────────┘
                                 │
               ┌─────────────────┴─────────────────┐
               │  * only when stack detected       │
     ┌─────────▼──────────┐             ┌──────────▼─────────┐
     │ aiuditor-live     *│             │ aiuditor-live     *│
     │   -supabase        │             │   -vercel          │
     │ RLS + Keys         │             │ Env + Middleware   │
     │     (Sonnet)       │             │     (Sonnet)       │
     └─────────┬──────────┘             └──────────┬─────────┘
               └─────────────────┬─────────────────┘
                  Batch 1: 3-5 agents in parallel
                                 │
          ┌──────────────────────┼─────────────────────┐
          │                      │                     │
┌─────────▼──────────┐ ┌─────────▼────────┐ ┌──────────▼─────────┐
│ aiuditor-live      │ │ aiuditor-live    │ │ aiuditor-live      │
│   -quality         │ │   -seo           │ │   -stress          │
│ Perf + A11y        │ │ SEO Analysis     │ │ Load Testing       │
│     (Sonnet)       │ │     (Sonnet)     │ │     (Sonnet)       │
└─────────┬──────────┘ └─────────┬────────┘ └──────────┬─────────┘
          └──────────────────────┼─────────────────────┘
                  Batch 2: 3 agents in parallel
                                 │
                    ┌────────────▼─────────────┐
                    │    Report Generation     │
                    │     MD + HTML + PDF      │
                    └──────────────────────────┘
```

| 에이전트 | 테스트 수 | 영역 |
|---------|----------|------|
| aiuditor-live-render | 18 | 페이지 렌더링 + 크로스브라우저 (Chromium, Firefox, WebKit) |
| aiuditor-live-security | 38 | OWASP Top 10, 헤더, 인증, 인젝션, 접근 제어 |
| aiuditor-live-infra | 11 | SSL/TLS 인증서, DNS, CDN, SRI |
| aiuditor-live-quality | 23 | Core Web Vitals (13) + WCAG AA 접근성 (10) |
| aiuditor-live-seo | 9 | 메타 태그, OG, 구조화 데이터, 사이트맵, robots.txt |
| aiuditor-live-stress | 9 | 동시 접속 부하, 지속 트래픽, 레이트 리미팅 |
| aiuditor-live-supabase * | 21 | RLS 우회, API 키 노출, 인증 쿠키, 스토리지 |
| aiuditor-live-vercel * | 11 | 환경변수, 미들웨어 우회 (CVE-2025-29927), 소스맵 |

\* 조건부 — Supabase/Vercel 스택 감지 시에만 실행.

---

## 왜 두 개의 시스템인가?

| 영역 | AIuditor-Code (배포 전) | AIuditor-Live (배포 후) |
|------|----------------------|------------------------|
| 대상 | 로컬 소스코드 | 배포된 URL |
| 하드코딩된 시크릿 | 소스코드 Grep (SAST) | 소스 접근 불가 |
| 의존성 CVE | npm audit | package.json 접근 불가 |
| 빌드 실패 | 배포 전 포착 | 이미 배포됨 |
| 타입 오류 | tsc --noEmit | 런타임에 타입 정보 소멸 |
| 설정 누락 | .env 검증 | 런타임 에러로만 감지 |
| HTTP 헤더 | 설정 파일 검사 | 실제 헤더 검증 |
| 실제 성능 | 번들 사이즈 분석 | Core Web Vitals |
| 접근성 | 정적 분석 한계 | 실제 브라우저 렌더링 |

---

## 설치

```bash
# 저장소 클론
git clone https://github.com/visualic/aiuditor.git

# 전체 에이전트 설치 (live + code)
cp aiuditor/agents/live/*.md ~/.claude/agents/
cp aiuditor/agents/code/*.md ~/.claude/agents/

# 하나만 설치
cp aiuditor/agents/live/*.md ~/.claude/agents/   # 라이브 검수만
cp aiuditor/agents/code/*.md ~/.claude/agents/   # 코드 검수만
```

## 사용법

```bash
# 배포 전 — 로컬 코드베이스 검수
"코드 검수해줘" → aiuditor-code (83개 테스트)

# 배포 후 — 라이브 사이트 검수
"https://example.com 검수해줘" → aiuditor-live (최대 151개 테스트)
```

### 옵션

```bash
# 코드 검수 — 특정 단계
"코드 검수 secrets만"
"코드 검수 빌드 제외"

# 라이브 검수 — 특정 단계
"라이브 검수 보안만"
"라이브 검수 https://example.com --stress-level heavy"
```

---

## 출력

두 시스템 모두 `.audit/artifacts/`에 통합 JSON 산출물을, `.audit/reports/`에 보고서를 생성합니다:

- **Markdown** — `.audit/reports/{code,live}-audit-report.md`
- **HTML** — Chart.js 시각화 포함
- **PDF** — Playwright를 통한 A4 형식

결과에는 심각도(Critical / High / Medium / Low)와 실행 가능한 권장사항이 포함됩니다.

---

## 요구사항

- [Claude Code](https://claude.ai/claude-code) CLI
- Node.js 18+
- Playwright (미설치 시 자동 설치)

---

## 언어

[English](README.md) | [Español](README.es.md) | [Deutsch](README.de.md) | [Français](README.fr.md) | [한국어](README.ko.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

---

## 라이선스

MIT
