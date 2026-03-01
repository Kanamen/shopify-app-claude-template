# Shopify App Claude Template

Shopify アプリ開発を Claude Code で進めるためのテンプレートリポジトリ。

## 含まれるもの

| ファイル | 内容 |
|---------|------|
| `CLAUDE.md` | 開発ルール・技術スタック（Claude Code が自動読み込み） |
| `.mcp.json` | Shopify Dev MCP サーバー設定 |
| `.claude/settings.local.json` | Claude Code の権限設定 |
| `.claude/docs/known-issues.md` | エラーナレッジベース（Claude Code が自動読み込み） |
| `.claude/skills/log-fix/SKILL.md` | エラー記録用スキル (`/log-fix`) |
| `docs/LEARNING.md` | 学習ドキュメント（手動読み込み） |
| `docs/REQUIREMENTS.md.example` | 要件定義テンプレート |

## 使い方

### 1. テンプレートから新規プロジェクトを作成

```bash
# テンプレートをクローン
git clone https://github.com/YOUR_USERNAME/shopify-app-claude-template.git my-new-app
cd my-new-app

# テンプレートの git 履歴を削除して新規リポジトリにする
rm -rf .git
git init
```

### 2. Shopify アプリをセットアップ

```bash
# Shopify CLI テンプレートのファイルを取得（app/, prisma/ 等）
# 方法A: shopify app init で対話的に作成
shopify app init

# 方法B: 公式テンプレートから必要なファイルをコピー
# https://github.com/Shopify/shopify-app-template-react-router

# 依存パッケージをインストール
npm install

# Prisma セットアップ
npx prisma generate
npx prisma migrate dev --name init
```

### 3. 要件定義を作成

```bash
cp docs/REQUIREMENTS.md.example docs/REQUIREMENTS.md
# REQUIREMENTS.md を編集してプロジェクトの要件を記述
```

### 4. 開発用リポジトリに push

```bash
git add .
git commit -m "Initial setup from template"
git remote add origin https://github.com/YOUR_USERNAME/my-new-app.git
git push -u origin main
```

### 5. Claude Code で開発開始

```bash
claude
# MCP サーバーは .mcp.json の設定により自動で利用可能
```

## 自動読み込みファイルについて

Claude Code は会話開始時に以下のファイルを自動で読み込みます:

- `CLAUDE.md` — 開発ルール（毎回のトークン消費あり）
- `.claude/docs/*.md` — ドキュメント（毎回のトークン消費あり）

`docs/` 配下のファイルは自動読み込み**されません**。必要なときだけ `@docs/LEARNING.md` で参照できます。

大きなドキュメント（LEARNING.md 等）は `docs/` に置くことでトークン消費を抑えられます。

## 前提条件

- Node.js v20+
- Shopify CLI (`npm install -g @shopify/cli`)
- Shopify Partners アカウント
