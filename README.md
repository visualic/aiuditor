# AIuditor

**AI Vibe Coding Quality Inspector** — Automated audit agents for Claude Code

AIuditor is a multi-agent orchestration system that comprehensively audits web services built with AI vibe coding. It consists of two independent systems:

- **AIuditor-Code**: Pre-deployment code inspection (source code security, dependency CVEs, build verification, config validation, type safety)
- **AIuditor-Live**: Post-deployment audit of live websites (security, performance, accessibility, SEO, stress testing)

> [!NOTE]
> AIuditor agents run inside [Claude Code](https://claude.ai/claude-code) as custom agents (`~/.claude/agents/`).

---

## Architecture

### AIuditor-Code — Pre-Deployment Code Inspection (83 tests)

Inspects the **local codebase** via file analysis, CLI tools, and local server testing.

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

| Agent | Tests | Focus |
|-------|-------|-------|
| aiuditor-code-secrets | 16 | Hardcoded API keys, passwords, private keys, eval(), XSS vectors, SQL injection patterns |
| aiuditor-code-deps | 10 | npm audit CVEs, lockfile integrity, license compliance, unused deps |
| aiuditor-code-config | 14 | .env completeness, DB migration safety, CORS/CSP, auth middleware, rate limiting |
| aiuditor-code-types | 14 | TypeScript compilation, ESLint, console.log residue, test coverage, code quality |
| aiuditor-code-build | 12 | Build success, bundle size, source maps, dead code, image optimization |
| aiuditor-code-localhost | 8 | Dev server startup, route accessibility, API smoke test, console errors |

---

### AIuditor-Live — Post-Deployment Audit (151 tests)

Audits a **deployed live URL** via HTTP requests and Playwright browser automation.

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

| Agent | Tests | Focus |
|-------|-------|-------|
| aiuditor-live-render | 18 | Page rendering + cross-browser (Chromium, Firefox, WebKit) |
| aiuditor-live-security | 38 | OWASP Top 10, headers, auth, injection, access control |
| aiuditor-live-infra | 11 | SSL/TLS certificates, DNS, CDN, SRI |
| aiuditor-live-quality | 23 | Core Web Vitals (13) + WCAG AA accessibility (10) |
| aiuditor-live-seo | 9 | Meta tags, OG, structured data, sitemap, robots.txt |
| aiuditor-live-stress | 9 | Concurrent load, sustained traffic, rate limiting |
| aiuditor-live-supabase * | 21 | RLS bypass, API key exposure, auth cookies, storage |
| aiuditor-live-vercel * | 11 | Env vars, middleware bypass (CVE-2025-29927), source maps |

\* Conditional — only runs when Supabase/Vercel stack is detected.

---

## Why Two Systems?

| Aspect | AIuditor-Code (Pre-deploy) | AIuditor-Live (Post-deploy) |
|--------|---------------------------|----------------------------|
| Target | Local source code | Deployed URL |
| Hardcoded secrets | Source code grep (SAST) | Cannot see source |
| Dependency CVEs | npm audit | Cannot access package.json |
| Build failures | Catches before deploy | Already deployed |
| Type errors | tsc --noEmit | Type info lost at runtime |
| Config gaps | .env validation | Only runtime errors |
| HTTP headers | Config file check | Actual header verification |
| Real performance | Bundle size analysis | Core Web Vitals |
| Accessibility | Static analysis limits | Real browser rendering |

---

## Installation

```bash
# Clone the repository
git clone https://github.com/visualic/aiuditor.git

# Install all agents (live + code)
cp aiuditor/agents/code/*.md ~/.claude/agents/
cp aiuditor/agents/live/*.md ~/.claude/agents/

# Or install only one system
cp aiuditor/agents/code/*.md ~/.claude/agents/   # Code audit only
cp aiuditor/agents/live/*.md ~/.claude/agents/   # Live audit only
```

## Usage

```bash
# Pre-deployment — inspect local codebase
"Audit this code" → aiuditor-code (83 tests)

# Post-deployment — audit live site
"Audit https://example.com" → aiuditor-live (up to 151 tests)
```

### Options

```bash
# Code audit — specific phases
"Code audit secrets only"
"Code audit skip build"

# Live audit — specific phases
"Live audit security only"
"Live audit https://example.com --stress-level heavy"
```

---

## Output

Both systems produce unified JSON artifacts in `.audit/artifacts/` and reports in `.audit/reports/`:

- **Markdown** — `.audit/reports/{code,live}-audit-report.md`
- **HTML** — With Chart.js visualizations
- **PDF** — A4 format via Playwright

Findings include severity levels (Critical / High / Medium / Low) with actionable recommendations.

---

## Requirements

- [Claude Code](https://claude.ai/claude-code) CLI
- Node.js 18+
- Playwright (auto-installed if missing)

---

## Languages

[English](README.md) | [Español](README.es.md) | [Deutsch](README.de.md) | [Français](README.fr.md) | [한국어](README.ko.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

---

## License

MIT
