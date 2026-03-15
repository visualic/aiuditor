# AIuditor

**AI Vibe Coding 品質檢測器** — Claude Code 自動化稽核代理

AIuditor 是一個多代理編排系統，全面稽核使用 AI Vibe Coding 建構的 Web 服務。它由兩個獨立的系統組成：

- **AIuditor-Code**：部署前的程式碼檢查（原始碼安全、相依性 CVE、建置驗證、設定驗證、型別安全）
- **AIuditor-Live**：部署後的即時網站稽核（安全、效能、無障礙、SEO、壓力測試）

> [!NOTE]
> AIuditor 代理在 [Claude Code](https://claude.ai/claude-code) 中作為自訂代理（`~/.claude/agents/`）執行。

---

## 架構

### AIuditor-Code — 部署前程式碼檢查（83 項測試）

透過檔案分析、CLI 工具和本機伺服器測試檢查**本機程式碼庫**。

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

| 代理 | 測試數 | 領域 |
|------|-------|------|
| aiuditor-code-secrets | 16 | 硬編碼 API 金鑰、密碼、私鑰、eval()、XSS 向量、SQL 注入模式 |
| aiuditor-code-deps | 10 | npm audit CVE、鎖定檔完整性、授權相容性、未使用相依性 |
| aiuditor-code-config | 14 | .env 完整性、DB 遷移安全性、CORS/CSP、驗證中介軟體、速率限制 |
| aiuditor-code-types | 14 | TypeScript 編譯、ESLint、console.log 殘留、測試覆蓋率、程式碼品質 |
| aiuditor-code-build | 12 | 建置成功、套件大小、Source Map、死碼、圖片最佳化 |
| aiuditor-code-localhost | 8 | 開發伺服器啟動、路由可存取性、API 冒煙測試、主控台錯誤 |

---

### AIuditor-Live — 部署後稽核（151 項測試）

透過 HTTP 請求和 Playwright 瀏覽器自動化稽核**已部署的即時 URL**。

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

| 代理 | 測試數 | 領域 |
|------|-------|------|
| aiuditor-live-render | 18 | 頁面渲染 + 跨瀏覽器（Chromium、Firefox、WebKit） |
| aiuditor-live-security | 38 | OWASP Top 10、標頭、驗證、注入、存取控制 |
| aiuditor-live-infra | 11 | SSL/TLS 憑證、DNS、CDN、SRI |
| aiuditor-live-quality | 23 | Core Web Vitals（13）+ WCAG AA 無障礙（10） |
| aiuditor-live-seo | 9 | Meta 標籤、OG、結構化資料、網站地圖、robots.txt |
| aiuditor-live-stress | 9 | 並行負載、持續流量、速率限制 |
| aiuditor-live-supabase * | 21 | RLS 繞過、API 金鑰洩露、驗證 Cookie、儲存 |
| aiuditor-live-vercel * | 11 | 環境變數、中介軟體繞過（CVE-2025-29927）、Source Map |

\* 條件觸發 — 僅在偵測到 Supabase/Vercel 技術棧時執行。

---

## 為什麼是兩個系統？

| 領域 | AIuditor-Code（部署前） | AIuditor-Live（部署後） |
|------|----------------------|------------------------|
| 目標 | 本機原始碼 | 已部署的 URL |
| 硬編碼密鑰 | 原始碼 Grep（SAST） | 無法存取原始碼 |
| 相依性 CVE | npm audit | 無法存取 package.json |
| 建置失敗 | 部署前捕獲 | 已經部署 |
| 型別錯誤 | tsc --noEmit | 執行時型別資訊遺失 |
| 設定遺漏 | .env 驗證 | 僅透過執行時錯誤偵測 |
| HTTP 標頭 | 設定檔檢查 | 實際標頭驗證 |
| 實際效能 | 套件大小分析 | Core Web Vitals |
| 無障礙 | 靜態分析侷限 | 真實瀏覽器渲染 |

---

## 安裝

```bash
# 複製儲存庫
git clone https://github.com/visualic/aiuditor.git

# 安裝所有代理（live + code）
cp aiuditor/agents/live/*.md ~/.claude/agents/
cp aiuditor/agents/code/*.md ~/.claude/agents/

# 僅安裝一個系統
cp aiuditor/agents/live/*.md ~/.claude/agents/   # 僅即時稽核
cp aiuditor/agents/code/*.md ~/.claude/agents/   # 僅程式碼檢查
```

## 使用方法

```bash
# 部署前 — 檢查本機程式碼庫
"稽核這段程式碼" → aiuditor-code（83 項測試）

# 部署後 — 稽核即時網站
"稽核 https://example.com" → aiuditor-live（最多 151 項測試）
```

### 選項

```bash
# 程式碼檢查 — 特定階段
"程式碼稽核僅 secrets"
"程式碼稽核跳過建置"

# 即時稽核 — 特定階段
"即時稽核僅安全"
"即時稽核 https://example.com --stress-level heavy"
```

---

## 輸出

兩個系統都在 `.audit/artifacts/` 產生統一的 JSON 產物，在 `.audit/reports/` 產生報告：

- **Markdown** — `.audit/reports/{code,live}-audit-report.md`
- **HTML** — 包含 Chart.js 視覺化
- **PDF** — 透過 Playwright 產生 A4 格式

結果包含嚴重等級（Critical / High / Medium / Low）和可執行的建議。

---

## 需求

- [Claude Code](https://claude.ai/claude-code) CLI
- Node.js 18+
- Playwright（未安裝時自動安裝）

---

## 語言

[English](README.md) | [Español](README.es.md) | [Deutsch](README.de.md) | [Français](README.fr.md) | [한국어](README.ko.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

---

## 授權

MIT
