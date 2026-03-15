# AIuditor

**Inspecteur de Qualité pour AI Vibe Coding** — Agents d'audit automatisés pour Claude Code

AIuditor est un système d'orchestration multi-agents qui audite de manière exhaustive les services web construits avec l'AI vibe coding. Il se compose de deux systèmes indépendants :

- **AIuditor-Code** : Inspection pré-déploiement du code (sécurité du code source, CVEs des dépendances, vérification du build, validation de configuration, sécurité des types)
- **AIuditor-Live** : Audit post-déploiement de sites web en production (sécurité, performance, accessibilité, SEO, tests de charge)

> [!NOTE]
> Les agents AIuditor s'exécutent dans [Claude Code](https://claude.ai/claude-code) en tant qu'agents personnalisés (`~/.claude/agents/`).

---

## Architecture

### AIuditor-Code — Inspection Pré-Déploiement (83 tests)

Inspecte le **code source local** via l'analyse de fichiers, les outils CLI et les tests sur serveur local.

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
| aiuditor-code-secrets | 16 | Clés API hardcodées, mots de passe, clés privées, eval(), vecteurs XSS, patterns d'injection SQL |
| aiuditor-code-deps | 10 | npm audit CVEs, intégrité du lockfile, compatibilité des licences, dépendances inutilisées |
| aiuditor-code-config | 14 | Complétude de .env, sécurité des migrations DB, CORS/CSP, middleware d'authentification, rate limiting |
| aiuditor-code-types | 14 | Compilation TypeScript, ESLint, résidus de console.log, couverture de tests, qualité du code |
| aiuditor-code-build | 12 | Succès du build, taille du bundle, source maps, code mort, optimisation d'images |
| aiuditor-code-localhost | 8 | Démarrage du serveur dev, accessibilité des routes, smoke test API, erreurs console |

---

### AIuditor-Live — Audit Post-Déploiement (151 tests)

Audite une **URL déployée en production** via des requêtes HTTP et l'automatisation du navigateur Playwright.

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
| aiuditor-live-render | 18 | Rendu de pages + cross-navigateur (Chromium, Firefox, WebKit) |
| aiuditor-live-security | 38 | OWASP Top 10, en-têtes, authentification, injection, contrôle d'accès |
| aiuditor-live-infra | 11 | Certificats SSL/TLS, DNS, CDN, SRI |
| aiuditor-live-quality | 23 | Core Web Vitals (13) + accessibilité WCAG AA (10) |
| aiuditor-live-seo | 9 | Balises meta, OG, données structurées, sitemap, robots.txt |
| aiuditor-live-stress | 9 | Charge concurrente, trafic soutenu, limitation de débit |
| aiuditor-live-supabase * | 21 | Contournement RLS, exposition de clés API, cookies d'auth, stockage |
| aiuditor-live-vercel * | 11 | Variables d'environnement, contournement middleware (CVE-2025-29927), source maps |

\* Conditionnel — exécuté uniquement lorsque le stack Supabase/Vercel est détecté.

---

## Pourquoi deux systèmes ?

| Aspect | AIuditor-Code (Pré-déploiement) | AIuditor-Live (Post-déploiement) |
|--------|--------------------------------|----------------------------------|
| Cible | Code source local | URL déployée |
| Secrets hardcodés | Grep du code source (SAST) | Pas d'accès au code |
| CVEs des dépendances | npm audit | Pas d'accès à package.json |
| Échecs de build | Détection avant déploiement | Déjà déployé |
| Erreurs de types | tsc --noEmit | Info de types perdue à l'exécution |
| Configuration manquante | Validation de .env | Erreurs d'exécution uniquement |
| En-têtes HTTP | Vérification des fichiers de config | Vérification réelle des en-têtes |
| Performance réelle | Analyse de la taille du bundle | Core Web Vitals |
| Accessibilité | Limites de l'analyse statique | Rendu réel du navigateur |

---

## Installation

```bash
# Cloner le dépôt
git clone https://github.com/visualic/aiuditor.git

# Installer tous les agents (live + code)
cp aiuditor/agents/live/*.md ~/.claude/agents/
cp aiuditor/agents/code/*.md ~/.claude/agents/

# Installer un seul système
cp aiuditor/agents/live/*.md ~/.claude/agents/   # Audit live uniquement
cp aiuditor/agents/code/*.md ~/.claude/agents/   # Audit code uniquement
```

## Utilisation

```bash
# Pré-déploiement — inspecter le code local
"Audit this code" → aiuditor-code (83 tests)

# Post-déploiement — auditer le site en production
"Audit https://example.com" → aiuditor-live (jusqu'à 151 tests)
```

### Options

```bash
# Audit de code — phases spécifiques
"Code audit secrets only"
"Code audit skip build"

# Audit live — phases spécifiques
"Live audit security only"
"Live audit https://example.com --stress-level heavy"
```

---

## Sortie

Les deux systèmes produisent des artefacts JSON unifiés dans `.audit/artifacts/` et des rapports dans `.audit/reports/` :

- **Markdown** — `.audit/reports/{code,live}-audit-report.md`
- **HTML** — Avec visualisations Chart.js
- **PDF** — Format A4 via Playwright

Les résultats incluent des niveaux de sévérité (Critical / High / Medium / Low) avec des recommandations actionnables.

---

## Prérequis

- [Claude Code](https://claude.ai/claude-code) CLI
- Node.js 18+
- Playwright (installé automatiquement si absent)

---

## Langues

[English](README.md) | [Español](README.es.md) | [Deutsch](README.de.md) | [Français](README.fr.md) | [한국어](README.ko.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

---

## Licence

MIT
