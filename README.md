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

### 1. Shopify アプリを作成（ターミナルで対話実行）

```bash
cd ~/Documents
shopify app init
#   → "React Router app" を選択
#   → アプリ名を入力（例: my-new-app）
#   → ~/Documents/my-new-app/ が自動作成される
#   → npm install と Prisma セットアップも自動で実行される
```

### 2. テンプレートファイルをコピー

```bash
cd ~/Documents/my-new-app

# テンプレートの原本からファイルをコピー
cp -r ~/Documents/shopify-app-claude-template/.claude .
cp ~/Documents/shopify-app-claude-template/.mcp.json .
cp ~/Documents/shopify-app-claude-template/CLAUDE.md .
cp -r ~/Documents/shopify-app-claude-template/docs .
```

### 3. アプリを Shopify に登録（ターミナルで対話実行）

```bash
shopify app config link
#   → Organization を選択
#   → 新しいアプリとして作成
#   → shopify.app.toml に client_id が書き込まれる
```

### 4. 要件定義を作成

```bash
cp docs/REQUIREMENTS.md.example docs/REQUIREMENTS.md
# REQUIREMENTS.md を編集してプロジェクトの要件を記述
```

### 5. GitHub リポジトリを作成して push

```bash
git add -A
git commit -m "Initial setup"
gh repo create YOUR_USERNAME/my-new-app --private --source=. --push
```

### 6. Claude Code で開発開始

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
