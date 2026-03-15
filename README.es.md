# AIuditor

**Inspector de Calidad para AI Vibe Coding** — Agentes de auditoría automatizados para Claude Code

AIuditor es un sistema de orquestación multi-agente que audita exhaustivamente servicios web construidos con AI vibe coding. Consta de dos sistemas independientes:

- **AIuditor-Code**: Inspección pre-despliegue del código (seguridad del código fuente, CVEs de dependencias, verificación de build, validación de configuración, seguridad de tipos)
- **AIuditor-Live**: Auditoría post-despliegue de sitios web en producción (seguridad, rendimiento, accesibilidad, SEO, pruebas de estrés)

> [!NOTE]
> Los agentes AIuditor se ejecutan dentro de [Claude Code](https://claude.ai/claude-code) como agentes personalizados (`~/.claude/agents/`).

---

## Arquitectura

### AIuditor-Code — Inspección Pre-Despliegue (83 pruebas)

Inspecciona el **código fuente local** mediante análisis de archivos, herramientas CLI y pruebas con servidor local.

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

| Agente | Pruebas | Enfoque |
|--------|---------|---------|
| aiuditor-code-secrets | 16 | Claves API hardcodeadas, contraseñas, claves privadas, eval(), vectores XSS, patrones de inyección SQL |
| aiuditor-code-deps | 10 | npm audit CVEs, integridad del lockfile, compatibilidad de licencias, dependencias sin usar |
| aiuditor-code-config | 14 | Completitud de .env, seguridad de migraciones DB, CORS/CSP, middleware de autenticación, rate limiting |
| aiuditor-code-types | 14 | Compilación TypeScript, ESLint, residuos de console.log, cobertura de tests, calidad de código |
| aiuditor-code-build | 12 | Éxito del build, tamaño del bundle, source maps, código muerto, optimización de imágenes |
| aiuditor-code-localhost | 8 | Inicio del servidor dev, accesibilidad de rutas, smoke test de API, errores de consola |

---

### AIuditor-Live — Auditoría Post-Despliegue (151 pruebas)

Audita una **URL desplegada en producción** mediante solicitudes HTTP y automatización de navegador con Playwright.

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

| Agente | Pruebas | Enfoque |
|--------|---------|---------|
| aiuditor-live-render | 18 | Renderizado de páginas + cross-browser (Chromium, Firefox, WebKit) |
| aiuditor-live-security | 38 | OWASP Top 10, headers, autenticación, inyección, control de acceso |
| aiuditor-live-infra | 11 | Certificados SSL/TLS, DNS, CDN, SRI |
| aiuditor-live-quality | 23 | Core Web Vitals (13) + accesibilidad WCAG AA (10) |
| aiuditor-live-seo | 9 | Meta tags, OG, datos estructurados, sitemap, robots.txt |
| aiuditor-live-stress | 9 | Carga concurrente, tráfico sostenido, limitación de velocidad |
| aiuditor-live-supabase * | 21 | Bypass RLS, exposición de claves API, cookies de autenticación, almacenamiento |
| aiuditor-live-vercel * | 11 | Variables de entorno, bypass de middleware (CVE-2025-29927), source maps |

\* Condicional — solo se ejecuta cuando se detecta el stack Supabase/Vercel.

---

## ¿Por qué dos sistemas?

| Aspecto | AIuditor-Code (Pre-despliegue) | AIuditor-Live (Post-despliegue) |
|---------|-------------------------------|--------------------------------|
| Objetivo | Código fuente local | URL desplegada |
| Secretos hardcodeados | Grep del código fuente (SAST) | Sin acceso al código |
| CVEs de dependencias | npm audit | Sin acceso a package.json |
| Fallos de build | Captura antes del despliegue | Ya desplegado |
| Errores de tipos | tsc --noEmit | Info de tipos perdida en runtime |
| Configuración faltante | Validación de .env | Solo errores en runtime |
| Headers HTTP | Verificación de archivos de config | Verificación real de headers |
| Rendimiento real | Análisis de tamaño de bundle | Core Web Vitals |
| Accesibilidad | Límites del análisis estático | Renderizado real del navegador |

---

## Instalación

```bash
# Clonar el repositorio
git clone https://github.com/visualic/aiuditor.git

# Instalar todos los agentes (live + code)
cp aiuditor/agents/live/*.md ~/.claude/agents/
cp aiuditor/agents/code/*.md ~/.claude/agents/

# Instalar solo un sistema
cp aiuditor/agents/live/*.md ~/.claude/agents/   # Solo auditoría live
cp aiuditor/agents/code/*.md ~/.claude/agents/   # Solo auditoría de código
```

## Uso

```bash
# Pre-despliegue — inspeccionar código local
"Audita este código" → aiuditor-code (83 pruebas)

# Post-despliegue — auditar sitio en producción
"Audita https://example.com" → aiuditor-live (hasta 151 pruebas)
```

### Opciones

```bash
# Auditoría de código — fases específicas
"Auditoría de código solo secrets"
"Auditoría de código saltar build"

# Auditoría live — fases específicas
"Auditoría live solo seguridad"
"Auditoría live https://example.com --stress-level heavy"
```

---

## Salida

Ambos sistemas producen artefactos JSON unificados en `.audit/artifacts/` y reportes en `.audit/reports/`:

- **Markdown** — `.audit/reports/{code,live}-audit-report.md`
- **HTML** — Con visualizaciones Chart.js
- **PDF** — Formato A4 vía Playwright

Los hallazgos incluyen niveles de severidad (Critical / High / Medium / Low) con recomendaciones accionables.

---

## Requisitos

- [Claude Code](https://claude.ai/claude-code) CLI
- Node.js 18+
- Playwright (se instala automáticamente si falta)

---

## Idiomas

[English](README.md) | [Español](README.es.md) | [Deutsch](README.de.md) | [Français](README.fr.md) | [한국어](README.ko.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

---

## Licencia

MIT
