# AIuditor

**AI Vibe Coding Qualitätsprüfer** — Automatisierte Audit-Agenten für Claude Code

AIuditor ist ein Multi-Agenten-Orchestrierungssystem, das Webdienste, die mit AI Vibe Coding erstellt wurden, umfassend prüft. Es besteht aus zwei unabhängigen Systemen:

- **AIuditor-Code**: Pre-Deployment-Code-Inspektion (Quellcode-Sicherheit, Dependency-CVEs, Build-Verifizierung, Konfigurationsvalidierung, Typsicherheit)
- **AIuditor-Live**: Post-Deployment-Audit von Live-Websites (Sicherheit, Performance, Barrierefreiheit, SEO, Stresstests)

> [!NOTE]
> AIuditor-Agenten laufen innerhalb von [Claude Code](https://claude.ai/claude-code) als benutzerdefinierte Agenten (`~/.claude/agents/`).

---

## Architektur

### AIuditor-Code — Pre-Deployment-Code-Inspektion (83 Tests)

Inspiziert die **lokale Codebasis** mittels Dateianalyse, CLI-Tools und lokalem Servertest.

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

| Agent | Tests | Fokus |
|-------|-------|-------|
| aiuditor-code-secrets | 16 | Hardcodierte API-Keys, Passwörter, Private Keys, eval(), XSS-Vektoren, SQL-Injection-Muster |
| aiuditor-code-deps | 10 | npm audit CVEs, Lockfile-Integrität, Lizenzkompatibilität, ungenutzte Dependencies |
| aiuditor-code-config | 14 | .env-Vollständigkeit, DB-Migrations-Sicherheit, CORS/CSP, Auth-Middleware, Rate Limiting |
| aiuditor-code-types | 14 | TypeScript-Kompilierung, ESLint, console.log-Rückstände, Testabdeckung, Codequalität |
| aiuditor-code-build | 12 | Build-Erfolg, Bundle-Größe, Source Maps, Dead Code, Bildoptimierung |
| aiuditor-code-localhost | 8 | Dev-Server-Start, Routen-Erreichbarkeit, API-Smoke-Test, Konsolenfehler |

---

### AIuditor-Live — Post-Deployment-Audit (151 Tests)

Prüft eine **bereitgestellte Live-URL** mittels HTTP-Anfragen und Playwright-Browser-Automatisierung.

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

| Agent | Tests | Fokus |
|-------|-------|-------|
| aiuditor-live-render | 18 | Seiten-Rendering + Cross-Browser (Chromium, Firefox, WebKit) |
| aiuditor-live-security | 38 | OWASP Top 10, Header, Authentifizierung, Injection, Zugriffskontrolle |
| aiuditor-live-infra | 11 | SSL/TLS-Zertifikate, DNS, CDN, SRI |
| aiuditor-live-quality | 23 | Core Web Vitals (13) + WCAG AA Barrierefreiheit (10) |
| aiuditor-live-seo | 9 | Meta-Tags, OG, strukturierte Daten, Sitemap, robots.txt |
| aiuditor-live-stress | 9 | Gleichzeitige Last, anhaltender Traffic, Rate Limiting |
| aiuditor-live-supabase * | 21 | RLS-Bypass, API-Key-Exposition, Auth-Cookies, Storage |
| aiuditor-live-vercel * | 11 | Umgebungsvariablen, Middleware-Bypass (CVE-2025-29927), Source Maps |

\* Bedingt — wird nur ausgeführt, wenn Supabase/Vercel-Stack erkannt wird.

---

## Warum zwei Systeme?

| Aspekt | AIuditor-Code (Pre-Deployment) | AIuditor-Live (Post-Deployment) |
|--------|-------------------------------|--------------------------------|
| Ziel | Lokaler Quellcode | Bereitgestellte URL |
| Hardcodierte Secrets | Quellcode-Grep (SAST) | Kein Zugriff auf Quellcode |
| Dependency-CVEs | npm audit | Kein Zugriff auf package.json |
| Build-Fehler | Erkennung vor Deployment | Bereits bereitgestellt |
| Typfehler | tsc --noEmit | Typinfo zur Laufzeit verloren |
| Fehlende Konfiguration | .env-Validierung | Nur Laufzeitfehler |
| HTTP-Header | Konfigurationsdatei-Prüfung | Tatsächliche Header-Verifizierung |
| Echte Performance | Bundle-Größen-Analyse | Core Web Vitals |
| Barrierefreiheit | Grenzen statischer Analyse | Echtes Browser-Rendering |

---

## Installation

```bash
# Repository klonen
git clone https://github.com/visualic/aiuditor.git

# Alle Agenten installieren (live + code)
cp aiuditor/agents/live/*.md ~/.claude/agents/
cp aiuditor/agents/code/*.md ~/.claude/agents/

# Nur ein System installieren
cp aiuditor/agents/live/*.md ~/.claude/agents/   # Nur Live-Audit
cp aiuditor/agents/code/*.md ~/.claude/agents/   # Nur Code-Audit
```

## Verwendung

```bash
# Pre-Deployment — lokale Codebasis inspizieren
"Audit this code" → aiuditor-code (83 Tests)

# Post-Deployment — Live-Site auditieren
"Audit https://example.com" → aiuditor-live (bis zu 151 Tests)
```

### Optionen

```bash
# Code-Audit — bestimmte Phasen
"Code audit secrets only"
"Code audit skip build"

# Live-Audit — bestimmte Phasen
"Live audit security only"
"Live audit https://example.com --stress-level heavy"
```

---

## Ausgabe

Beide Systeme erzeugen einheitliche JSON-Artefakte in `.audit/artifacts/` und Berichte in `.audit/reports/`:

- **Markdown** — `.audit/reports/{code,live}-audit-report.md`
- **HTML** — Mit Chart.js-Visualisierungen
- **PDF** — A4-Format via Playwright

Ergebnisse enthalten Schweregrade (Critical / High / Medium / Low) mit umsetzbaren Empfehlungen.

---

## Anforderungen

- [Claude Code](https://claude.ai/claude-code) CLI
- Node.js 18+
- Playwright (wird bei Bedarf automatisch installiert)

---

## Sprachen

[English](README.md) | [Español](README.es.md) | [Deutsch](README.de.md) | [Français](README.fr.md) | [한국어](README.ko.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

---

## Lizenz

MIT
