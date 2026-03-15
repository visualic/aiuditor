# AIuditor

**AI バイブコーディング品質検査ツール** — Claude Code用自動監査エージェント

AIuditorは、AIバイブコーディングで構築されたWebサービスを包括的に監査するマルチエージェントオーケストレーションシステムです。2つの独立したシステムで構成されています：

- **AIuditor-Code**: デプロイ前のコード検査（ソースコードセキュリティ、依存関係CVE、ビルド検証、設定検証、型安全性）
- **AIuditor-Live**: デプロイ後のライブWebサイト監査（セキュリティ、パフォーマンス、アクセシビリティ、SEO、ストレステスト）

> [!NOTE]
> AIuditorエージェントは[Claude Code](https://claude.ai/claude-code)内でカスタムエージェント（`~/.claude/agents/`）として実行されます。

---

## アーキテクチャ

### AIuditor-Code — デプロイ前コード検査（83テスト）

ファイル分析、CLIツール、ローカルサーバーテストで**ローカルコードベース**を検査します。

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

| エージェント | テスト数 | 対象 |
|------------|---------|------|
| aiuditor-code-secrets | 16 | ハードコードAPIキー、パスワード、秘密鍵、eval()、XSSベクター、SQLインジェクションパターン |
| aiuditor-code-deps | 10 | npm audit CVE、ロックファイル整合性、ライセンス互換性、未使用依存関係 |
| aiuditor-code-config | 14 | .env完全性、DBマイグレーション安全性、CORS/CSP、認証ミドルウェア、レート制限 |
| aiuditor-code-types | 14 | TypeScriptコンパイル、ESLint、console.log残存、テストカバレッジ、コード品質 |
| aiuditor-code-build | 12 | ビルド成功、バンドルサイズ、ソースマップ、デッドコード、画像最適化 |
| aiuditor-code-localhost | 8 | 開発サーバー起動、ルートアクセス、APIスモークテスト、コンソールエラー |

---

### AIuditor-Live — デプロイ後監査（151テスト）

HTTPリクエストとPlaywrightブラウザ自動化で**デプロイ済みのライブURL**を監査します。

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

| エージェント | テスト数 | 対象 |
|------------|---------|------|
| aiuditor-live-render | 18 | ページレンダリング + クロスブラウザ（Chromium, Firefox, WebKit） |
| aiuditor-live-security | 38 | OWASP Top 10、ヘッダー、認証、インジェクション、アクセス制御 |
| aiuditor-live-infra | 11 | SSL/TLS証明書、DNS、CDN、SRI |
| aiuditor-live-quality | 23 | Core Web Vitals（13）+ WCAG AAアクセシビリティ（10） |
| aiuditor-live-seo | 9 | メタタグ、OG、構造化データ、サイトマップ、robots.txt |
| aiuditor-live-stress | 9 | 同時接続負荷、持続トラフィック、レート制限 |
| aiuditor-live-supabase * | 21 | RLSバイパス、APIキー露出、認証Cookie、ストレージ |
| aiuditor-live-vercel * | 11 | 環境変数、ミドルウェアバイパス（CVE-2025-29927）、ソースマップ |

\* 条件付き — Supabase/Vercelスタック検出時のみ実行。

---

## なぜ2つのシステムなのか？

| 領域 | AIuditor-Code（デプロイ前） | AIuditor-Live（デプロイ後） |
|------|---------------------------|----------------------------|
| 対象 | ローカルソースコード | デプロイ済みURL |
| ハードコードシークレット | ソースコードGrep（SAST） | ソースアクセス不可 |
| 依存関係CVE | npm audit | package.jsonアクセス不可 |
| ビルド失敗 | デプロイ前に検出 | すでにデプロイ済み |
| 型エラー | tsc --noEmit | ランタイムで型情報消失 |
| 設定漏れ | .env検証 | ランタイムエラーでのみ検出 |
| HTTPヘッダー | 設定ファイル検査 | 実際のヘッダー検証 |
| 実パフォーマンス | バンドルサイズ分析 | Core Web Vitals |
| アクセシビリティ | 静的分析の限界 | 実ブラウザレンダリング |

---

## インストール

```bash
# リポジトリをクローン
git clone https://github.com/visualic/aiuditor.git

# 全エージェントをインストール（live + code）
cp aiuditor/agents/live/*.md ~/.claude/agents/
cp aiuditor/agents/code/*.md ~/.claude/agents/

# 片方のみインストール
cp aiuditor/agents/live/*.md ~/.claude/agents/   # ライブ監査のみ
cp aiuditor/agents/code/*.md ~/.claude/agents/   # コード検査のみ
```

## 使い方

```bash
# デプロイ前 — ローカルコードベース検査
"コードを検査して" → aiuditor-code（83テスト）

# デプロイ後 — ライブサイト監査
"https://example.com を監査して" → aiuditor-live（最大151テスト）
```

### オプション

```bash
# コード検査 — 特定フェーズ
"コード検査 secretsのみ"
"コード検査 ビルドスキップ"

# ライブ監査 — 特定フェーズ
"ライブ監査 セキュリティのみ"
"ライブ監査 https://example.com --stress-level heavy"
```

---

## 出力

両システムとも`.audit/artifacts/`に統合JSONアーティファクトを、`.audit/reports/`にレポートを生成します：

- **Markdown** — `.audit/reports/{code,live}-audit-report.md`
- **HTML** — Chart.js可視化付き
- **PDF** — PlaywrightによるA4形式

結果には重大度（Critical / High / Medium / Low）と実行可能な推奨事項が含まれます。

---

## 要件

- [Claude Code](https://claude.ai/claude-code) CLI
- Node.js 18+
- Playwright（未インストール時は自動インストール）

---

## 言語

[English](README.md) | [Español](README.es.md) | [Deutsch](README.de.md) | [Français](README.fr.md) | [한국어](README.ko.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

---

## ライセンス

MIT
