# CLAUDE.md — Shopify App Development (General Purpose)

## Overview
This is a Shopify embedded app project. Always follow Shopify's latest conventions and refer to shopify.dev documentation via the Dev MCP server before making assumptions.

## Tech Stack (as of API 2025-10)

### Core Framework
- **React Router v7** (`@shopify/shopify-app-react-router`)
- **TypeScript** (strict mode)
- **Prisma** (SQLite dev / PostgreSQL prod)
- **Node.js** v20+

### Frontend / UI
- **Polaris Web Components** (`<s-*>` prefix, CDN — NO npm install)
- **App Bridge** (CDN script, `shopify` global variable)
- Types: `@shopify/polaris-types`, `@shopify/app-bridge-types`

### UI Extensions (Customer Account)
- **Preact** (NOT React) — 64KB bundle limit
- Polaris Web Components (`<s-*>`) in extensions
- 必須依存: `preact`, `@preact/signals`, `@shopify/ui-extensions@2025.10.x`
- `shopify.query()` → **Storefront API**（商品・コレクション等）
- `fetch('shopify://customer-account/api/...')` → **Customer Account API**（顧客データ）
- Customer Account API に**ないもの**: `nodes(ids:)`, `product(id:)` → バックエンド経由で取得
- Extension から外部 API は **GET/POST のみ** 使用（DELETE/PUT/PATCH は CORS 制限あり）
- `.tsx` ファイルは必ず先頭に `/** @jsxImportSource preact */` を記述

### APIs
- **Admin GraphQL API** (primary) — cursor-based pagination, 1000 cost points/sec
- **Admin REST API** (legacy) — 40 requests/sec
- **Storefront API** (buyer-facing)
- Authentication: `shopify.authenticate.admin(request)` (React Router)

## Project Structure
```
app/
├── routes/           # URL → 画面/API のマッピング
├── models/           # データアクセス層 (*.server.ts)
├── shopify.server.ts # Shopify アプリ初期化
├── db.server.ts      # Prisma クライアント
extensions/           # UI Extensions
prisma/               # DB スキーマ・マイグレーション
shopify.app.toml      # アプリ設定
docs/                 # プロジェクト固有ドキュメント（要件定義・学習ノート等）
```

## Critical Rules

### バグ修正の手順（コード変更前に必ず実行）
1. **MCP で API スキーマを確認** — `introspect_graphql_schema` でフィールドの存在・型を検証してからコードを書く
2. **MCP でドキュメント確認** — `search_docs_chunks` / `fetch_full_docs` で公式の実装パターンを確認する
3. **1回のデプロイで検証可能な最小変更にする** — 複数の推測を同時に試さない
4. **動かない場合はデバッグUIを追加** — Extension 内に状態表示を入れて、どのステップで止まっているか特定する
5. **推測でコードを書き換えない** — エビデンス（エラーメッセージ、ログ、スキーマ検証結果）なしにコードを変更しない

### Embedded App Navigation
- Use `Link` from `react-router` or `@shopify/polaris`
- Use `redirect` helper from `authenticate.admin`
- NEVER use `<a>` tags, lowercase `<form>`, or `redirect` from `react-router` directly

### Authentication
- All `/app/*` routes must call `authenticate.admin(request)` in the loader
- NEVER store access tokens manually — use Prisma session storage
- Webhooks: respond with 200 within 5 seconds

### Customer Account Extension 必須設定
- `shopify.app.toml` に `customer_read_customers` スコープを追加
- Partner Dashboard → API access → Protected customer data を有効化
- Partner Dashboard → API access → Network access を許可

## Commands
- `shopify app dev` — ローカル開発（トンネル付き）
- `shopify app deploy` — Extensions デプロイ
- `fly deploy` — アプリ本体デプロイ（Fly.io）
- `npx prisma migrate dev` — DB マイグレーション
- `npx prisma studio` — DB GUI

## Dev MCP Server
`.mcp.json` で設定済み。以下のツールが使える:
- `search_docs_chunks` / `fetch_full_docs` — ドキュメント検索
- `introspect_graphql_schema` — GraphQL スキーマ検証
- `validate_component_codeblocks` — コンポーネントバリデーション
- `validate_graphql_codeblocks` — GraphQL バリデーション

## ドキュメント構成
| ファイル | 自動読み込み | 内容 |
|---------|:---:|------|
| `CLAUDE.md` | Yes | 開発ルール（本ファイル） |
| `.claude/docs/known-issues.md` | Yes | エラーナレッジベース |
| `docs/REQUIREMENTS.md` | No | プロジェクト要件定義（プロジェクト毎に作成） |
| `docs/LEARNING.md` | No | 学習ドキュメント |

## エラーナレッジベース
既知のエラーと解決方法は @.claude/docs/known-issues.md を参照。
新しいエラーを解決したら `/log-fix` で記録すること。
