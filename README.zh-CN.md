# AIuditor

**AI Vibe Coding 质量检测器** — Claude Code 自动化审计代理

AIuditor 是一个多代理编排系统，全面审计使用 AI Vibe Coding 构建的 Web 服务。它由两个独立的系统组成：

- **AIuditor-Code**：部署前的代码检查（源码安全、依赖 CVE、构建验证、配置验证、类型安全）
- **AIuditor-Live**：部署后的实时网站审计（安全、性能、无障碍、SEO、压力测试）

> [!NOTE]
> AIuditor 代理在 [Claude Code](https://claude.ai/claude-code) 中作为自定义代理（`~/.claude/agents/`）运行。

---

## 架构

### AIuditor-Code — 部署前代码检查（83 项测试）

通过文件分析、CLI 工具和本地服务器测试检查**本地代码库**。

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

| 代理 | 测试数 | 领域 |
|------|-------|------|
| aiuditor-code-secrets | 16 | 硬编码 API 密钥、密码、私钥、eval()、XSS 向量、SQL 注入模式 |
| aiuditor-code-deps | 10 | npm audit CVE、锁文件完整性、许可证兼容性、未使用依赖 |
| aiuditor-code-config | 14 | .env 完整性、DB 迁移安全性、CORS/CSP、认证中间件、速率限制 |
| aiuditor-code-types | 14 | TypeScript 编译、ESLint、console.log 残留、测试覆盖率、代码质量 |
| aiuditor-code-build | 12 | 构建成功、包大小、Source Map、死代码、图片优化 |
| aiuditor-code-localhost | 8 | 开发服务器启动、路由可访问性、API 冒烟测试、控制台错误 |

---

### AIuditor-Live — 部署后审计（151 项测试）

通过 HTTP 请求和 Playwright 浏览器自动化审计**已部署的实时 URL**。

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

| 代理 | 测试数 | 领域 |
|------|-------|------|
| aiuditor-live-render | 18 | 页面渲染 + 跨浏览器（Chromium、Firefox、WebKit） |
| aiuditor-live-security | 38 | OWASP Top 10、头部、认证、注入、访问控制 |
| aiuditor-live-infra | 11 | SSL/TLS 证书、DNS、CDN、SRI |
| aiuditor-live-quality | 23 | Core Web Vitals（13）+ WCAG AA 无障碍（10） |
| aiuditor-live-seo | 9 | Meta 标签、OG、结构化数据、站点地图、robots.txt |
| aiuditor-live-stress | 9 | 并发负载、持续流量、速率限制 |
| aiuditor-live-supabase * | 21 | RLS 绕过、API 密钥泄露、认证 Cookie、存储 |
| aiuditor-live-vercel * | 11 | 环境变量、中间件绕过（CVE-2025-29927）、Source Map |

\* 条件触发 — 仅在检测到 Supabase/Vercel 技术栈时执行。

---

## 为什么是两个系统？

| 领域 | AIuditor-Code（部署前） | AIuditor-Live（部署后） |
|------|----------------------|------------------------|
| 目标 | 本地源代码 | 已部署的 URL |
| 硬编码密钥 | 源代码 Grep（SAST） | 无法访问源码 |
| 依赖 CVE | npm audit | 无法访问 package.json |
| 构建失败 | 部署前捕获 | 已经部署 |
| 类型错误 | tsc --noEmit | 运行时类型信息丢失 |
| 配置遗漏 | .env 验证 | 仅通过运行时错误检测 |
| HTTP 头部 | 配置文件检查 | 实际头部验证 |
| 实际性能 | 包大小分析 | Core Web Vitals |
| 无障碍 | 静态分析局限 | 真实浏览器渲染 |

---

## 安装

```bash
# 克隆仓库
git clone https://github.com/visualic/aiuditor.git

# 安装所有代理（live + code）
cp aiuditor/agents/live/*.md ~/.claude/agents/
cp aiuditor/agents/code/*.md ~/.claude/agents/

# 仅安装一个系统
cp aiuditor/agents/live/*.md ~/.claude/agents/   # 仅实时审计
cp aiuditor/agents/code/*.md ~/.claude/agents/   # 仅代码检查
```

## 使用方法

```bash
# 部署前 — 检查本地代码库
"审计这段代码" → aiuditor-code（83 项测试）

# 部署后 — 审计实时站点
"审计 https://example.com" → aiuditor-live（最多 151 项测试）
```

### 选项

```bash
# 代码检查 — 特定阶段
"代码审计仅 secrets"
"代码审计跳过构建"

# 实时审计 — 特定阶段
"实时审计仅安全"
"实时审计 https://example.com --stress-level heavy"
```

---

## 输出

两个系统都在 `.audit/artifacts/` 生成统一的 JSON 产物，在 `.audit/reports/` 生成报告：

- **Markdown** — `.audit/reports/{code,live}-audit-report.md`
- **HTML** — 包含 Chart.js 可视化
- **PDF** — 通过 Playwright 生成 A4 格式

结果包含严重级别（Critical / High / Medium / Low）和可操作的建议。

---

## 要求

- [Claude Code](https://claude.ai/claude-code) CLI
- Node.js 18+
- Playwright（未安装时自动安装）

---

## 语言

[English](README.md) | [Español](README.es.md) | [Deutsch](README.de.md) | [Français](README.fr.md) | [한국어](README.ko.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

---

## 许可证

MIT
