# Shopify App — 学習ドキュメント

> このドキュメントは、実装を進めながら登場する概念・用語・技術のつながりを
> やさしく解説するものです。「なぜこうなっているのか」を理解するための辞書として使ってください。
>
> **注**: コード例は Wishlist（お気に入り）アプリの実装を元にしています。
> 自分のプロジェクトに合わせて読み替えてください。

---

## 目次

1. [プロジェクト全体像 — 登場人物マップ](#1-プロジェクト全体像--登場人物マップ)
2. [Git — コードのバージョン管理](#2-git--コードのバージョン管理)
3. [React Router v7 — アプリの骨格](#3-react-router-v7--アプリの骨格)
4. [Prisma — データベースとの会話係](#4-prisma--データベースとの会話係)
5. [Shopify App の仕組み — 認証からAPIまで](#5-shopify-app-の仕組み--認証からapiまで)
6. [Theme App Extension — ストアフロントに飛び出すアプリ](#6-theme-app-extension--ストアフロントに飛び出すアプリ)
7. [Customer Account UI Extension — マイページを拡張する](#7-customer-account-ui-extension--マイページを拡張する)
8. [用語集](#8-用語集)
9. [MCP（Model Context Protocol）— Claude Code の拡張機能](#9-mcpmodel-context-protocol-claude-code-の拡張機能)
10. [ブラウザ開発者ツール — エラーの見つけ方](#10-ブラウザ開発者ツール--エラーの見つけ方)
11. [Claude Code Skills（カスタムコマンド）— 繰り返し作業を自動化する](#11-claude-code-skillsカスタムコマンド-繰り返し作業を自動化する)
12. [サーバーとトンネル — テーマ開発とアプリ開発の決定的な違い](#12-サーバーとトンネル--テーマ開発とアプリ開発の決定的な違い)
13. [開発ワークフロー — ローカル開発からデプロイまで](#13-開発ワークフロー--ローカル開発からデプロイまで)

---

## 1. プロジェクト全体像 — 登場人物マップ

このアプリには、大きく分けて **4つの世界** があります。

```
┌─────────────────────────────────────────────────────────────┐
│                    Shopify プラットフォーム                    │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ ストアフロント │  │ マイページ   │  │ 管理画面（Admin）    │ │
│  │ (お客さんが  │  │ (お客さんの  │  │ (ストアオーナーが    │ │
│  │  見る画面)   │  │  個人ページ) │  │  使う画面)          │ │
│  │             │  │             │  │                     │ │
│  │ Theme App   │  │ Customer    │  │ React Router v7     │ │
│  │ Extension   │  │ Account UI  │  │ + Polaris           │ │
│  │ (Liquid)    │  │ Extension   │  │                     │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
│         │                │                     │            │
│         └────────┬───────┘                     │            │
│                  │                             │            │
│                  ▼                             │            │
│  ┌──────────────────────────────┐              │            │
│  │  アプリサーバー               │◄─────────────┘            │
│  │  (React Router v7)          │                           │
│  │  ┌─────────────────────┐    │                           │
│  │  │ API エンドポイント    │    │                           │
│  │  │ (api.wishlist.ts)   │    │                           │
│  │  └──────────┬──────────┘    │                           │
│  │             │               │                           │
│  │  ┌──────────▼──────────┐    │                           │
│  │  │ データアクセス層      │    │                           │
│  │  │ (wishlist.server.ts)│    │                           │
│  │  └──────────┬──────────┘    │                           │
│  └─────────────┼───────────────┘                           │
│                │                                           │
│       ┌────────▼────────┐                                  │
│       │   データベース    │                                  │
│       │  (SQLite/PG)    │                                  │
│       └─────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
```

### ポイント

- **ストアフロント**（お客さんが商品を見る画面）と**管理画面**（ストアオーナーが設定する画面）は **完全に別の世界**
- アプリサーバーが**中間に立って**、両方の世界からのリクエストを受け取る
- データベースはアプリサーバーの中にあり、**Prisma** を通じてアクセスする

---

## 2. Git — コードのバージョン管理

### Git とは？

> コードの変更履歴を記録・管理する仕組み

ファイルを保存するたびに「いつ・誰が・何を変えたか」を記録できます。
間違えても過去の状態に戻せます。

### commit と push の違い（最重要）

```
自分のPC（ローカル）                          GitHub（リモート）
┌─────────────────────┐                    ┌─────────────────────┐
│                     │                    │                     │
│  作業ディレクトリ     │                    │  github.com/        │
│  （ファイルを編集）   │                    │  Kanamen/           │
│         │           │                    │  shopify-wishlist   │
│         ▼           │                    │                     │
│    git add          │                    │                     │
│  （変更をステージ）   │                    │                     │
│         │           │                    │                     │
│         ▼           │    git push        │                     │
│    git commit ──────┼──────────────────► │  ここに反映される     │
│  （ローカルに保存）   │                    │                     │
│                     │    git pull        │                     │
│                     │ ◄──────────────────┼  他の人の変更を取得   │
│                     │                    │                     │
└─────────────────────┘                    └─────────────────────┘
```

| コマンド | 何をする | どこに影響 |
|---------|---------|-----------|
| `git add ファイル名` | 「この変更を次のcommitに含める」と印をつける | ローカルのみ |
| `git commit -m "メッセージ"` | 印をつけた変更をローカルに保存（スナップショット） | **ローカルのみ** |
| `git push` | ローカルのcommitをGitHubにアップロード | **GitHubに反映** |
| `git pull` | GitHubの変更をローカルにダウンロード | ローカルに反映 |

### つまり

- **commit = セーブ**（自分のPCの中だけ。他の人には見えない）
- **push = アップロード**（GitHubに送る。他の人にも見える）
- commit しただけでは GitHub には何も起きない
- push して初めて GitHub のリポジトリに反映される

### 日常の流れ

```bash
# 1. コードを書く・編集する

# 2. 変更したファイルを確認
git status

# 3. 変更をステージに追加（commitに含めるファイルを選ぶ）
git add app/models/wishlist.server.ts
git add prisma/schema.prisma
# まとめて追加したい場合（ただし意図しないファイルに注意）:
# git add .

# 4. ローカルに保存（commit）
git commit -m "Wishlist モデルを追加"

# 5. GitHub にアップロード（push）
git push

# ※ 初回のみ、push先のブランチを指定する必要がある:
git push -u origin main
```

### よくある疑問

**Q: commit せずに push できる？**
→ No。push は commit の中身を送るだけ。commit がなければ送るものがない。

**Q: commit したけど push し忘れた。どうなる？**
→ ローカルには保存されている。次に push すれば、溜まった commit がまとめて GitHub に送られる。

**Q: 間違えて commit した。取り消せる？**
→ できる。ただし push 前と push 後で難易度が違う。push 前なら簡単。push 後は他の人に影響するので慎重に。

---

## 3. React Router v7 — アプリの骨格

### React Router v7 とは？

> 「URLに応じて、どの画面を表示するか」を決める仕組み

以前は **Remix** という名前のフレームワークでしたが、2024年に **React Router v7** に統合されました。
Shopifyの公式テンプレートも Remix → React Router v7 に移行しています。

### ファイルベースルーティング

```
app/routes/
├── app._index.tsx          → /app           （管理画面のトップページ）
├── app.additional.tsx       → /app/additional （管理画面の追加ページ）
├── app.tsx                 → /app 以下の共通レイアウト
├── api.wishlist.ts         → /api/wishlist    （WishlistのREST API）
├── auth.$.tsx              → /auth/*          （認証処理）
├── auth.login/route.tsx    → /auth/login      （ログインページ）
├── webhooks.app.uninstalled.tsx → /webhooks/app/uninstalled
└── _index/route.tsx        → /               （ルートページ）
```

**命名ルール:**
- ファイル名の `.` がURLの `/` に変換される
  - `app._index.tsx` → `/app` のインデックスページ
  - `api.wishlist.ts` → `/api/wishlist`
- `_index` は「そのパスそのもの」を表す（URLには現れない）
- `$` はワイルドカード（何でもマッチ）
- フォルダ名の `/` もルートセグメントになる

### loader と action — データの流れ

React Router v7 では、データの流れが明確に分かれています。

```
ブラウザ ──── GET リクエスト ────► loader（データ取得）──► 画面表示
             （ページを開く）

ブラウザ ──── POST リクエスト ───► action（データ変更）──► リダイレクト or 再表示
             （ボタンを押す）
```

```typescript
// loader: ページを開いたとき（GET）に実行される
// 「このページに必要なデータを取ってくる係」
export const loader = async ({ request }: LoaderFunctionArgs) => {
  const { admin } = await authenticate.admin(request);
  // DBからデータを取得して返す
  return { wishlistItems: await getWishlist(shop, customerId) };
};

// action: フォーム送信やボタンクリック（POST）で実行される
// 「データを変更する係」
export const action = async ({ request }: ActionFunctionArgs) => {
  const { admin } = await authenticate.admin(request);
  const formData = await request.formData();
  // DBにデータを追加/削除する
  await addToWishlist(shop, customerId, productId);
  return { success: true };
};
```

### `.server.ts` サフィックスの意味

```
wishlist.server.ts   ← サーバーでのみ実行される（ブラウザには送られない）
db.server.ts         ← サーバーでのみ実行される
shopify.server.ts    ← サーバーでのみ実行される
```

`.server.ts` をつけることで、**このファイルのコードはブラウザに送信されない**ことが保証されます。
データベース接続やAPIキーなど、秘密の情報を扱うファイルにはこのサフィックスをつけます。

---

## 4. Prisma — データベースとの会話係

### Prisma とは？

> データベースを **TypeScript のオブジェクトのように操作できる** ツール（ORM）

SQLを直接書く代わりに、TypeScriptの関数呼び出しでデータベースを操作できます。

### 3つの構成要素

```
┌─────────────────────────┐
│  prisma/schema.prisma   │ ← 1. スキーマ（設計図）
│  「どんなテーブルがある？  │    データベースの構造を定義
│   どんなカラムがある？」   │
└────────────┬────────────┘
             │
             ▼ prisma migrate dev（マイグレーション）
┌─────────────────────────┐
│  prisma/dev.sqlite      │ ← 2. データベース（実体）
│  （実際のデータが入る箱）  │    スキーマ通りにテーブルが作られる
└────────────┬────────────┘
             │
             ▼ prisma generate
┌─────────────────────────┐
│  @prisma/client         │ ← 3. クライアント（操作ツール）
│  （TypeScriptから操作）   │    TypeScript用の型付きAPIが生成される
└─────────────────────────┘
```

### スキーマの読み方

```prisma
model Wishlist {
  // ↓ カラム名    ↓ 型         ↓ 属性（オプション）
  id         String   @id @default(cuid())  // 主キー。自動でユニークIDを生成
  shop       String                          // どのストアのデータか
  customerId String                          // どの顧客のお気に入りか
  productId  String                          // どの商品か
  variantId  String?                         // ? = null許可（バリアントは任意）
  addedAt    DateTime @default(now())        // 追加日時。自動で現在時刻

  @@unique([shop, customerId, productId, variantId])  // この組み合わせは重複不可
  @@index([shop, customerId])                          // 検索を速くするインデックス
  @@index([shop, productId])                           // 検索を速くするインデックス
}
```

### よく使う操作

```typescript
// 作成（INSERT）
await prisma.wishlist.create({
  data: { shop, customerId, productId, variantId }
});

// 取得（SELECT）
const items = await prisma.wishlist.findMany({
  where: { shop, customerId }
});

// 削除（DELETE）
await prisma.wishlist.delete({
  where: { id: "xxx" }
});

// カウント（COUNT）
const count = await prisma.wishlist.count({
  where: { shop, productId }
});
```

### マイグレーションとは？

> スキーマの変更を **バージョン管理して、順番に適用する仕組み**

```bash
# スキーマを変更した後に実行
npx prisma migrate dev --name add_wishlist_table
#                              ↑ 変更の名前（git commitのメッセージのようなもの）
```

これを実行すると:
1. `prisma/migrations/` フォルダに SQL ファイルが生成される
2. そのSQLがデータベースに適用される
3. Prisma Client が再生成される（新しいテーブルの型が使えるようになる）

---

## 5. Shopify App の仕組み — 認証からAPIまで

### Shopify のアカウント体系（超重要・ハマりポイント）

Shopify には **3つの別々の管理画面** があり、それぞれ役割が違います。
名前が似ているので非常に混乱しやすいです。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Shopify のアカウント体系                       │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Shopify ID（メールアドレス）                               │  │
│  │  = すべてのShopifyサービスに共通のログインアカウント          │  │
│  └────────────┬──────────────┬──────────────┬────────────────┘  │
│               │              │              │                   │
│    ┌──────────▼──────────┐ ┌─▼────────────┐ │                   │
│    │  Partners ダッシュ   │ │ Dev Dashboard│ │                   │
│    │  ボード              │ │ (開発者用)   │ │                   │
│    │  partners.shopify   │ │              │ │                   │
│    │  .com               │ │              │ │                   │
│    │                     │ │              │ │                   │
│    │ ・公開アプリの管理   │ │ ・CLIが使う  │ │                   │
│    │ ・Partnerストア管理  │ │ ・開発ストア │ │                   │
│    │ ・収益レポート       │ │ ・アプリ管理 │ │                   │
│    └─────────────────────┘ └──────────────┘ │                   │
│                                     │       │                   │
│                              ┌──────▼───────▼────────────────┐  │
│                              │  ストア管理画面                │  │
│                              │  admin.shopify.com/store/xxx  │  │
│                              │                               │  │
│                              │ ・商品登録・注文管理           │  │
│                              │ ・テーマ編集                  │  │
│                              │ ・アプリのインストール・使用   │  │
│                              └───────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### Partners ダッシュボード vs Dev Dashboard

| | Partners ダッシュボード | Dev Dashboard |
|---|---|---|
| URL | `partners.shopify.com` | CLI が内部的に使う |
| 主な用途 | 公開アプリの管理・収益管理 | アプリ開発・テスト |
| ストア作成 | Partner用の開発ストア | CLI用の開発ストア |
| **CLIとの連携** | **連携しない** | **CLIはここを見る** |

#### 最大の罠

> **Partners ダッシュボードで作った開発ストア** と **Dev Dashboard の開発ストア** は **別物**。
> `shopify app dev` コマンドは **Dev Dashboard のストアしか認識しない**。

Partners ダッシュボードで開発ストアを作っても、CLI の `shopify app dev` からは見えません。

#### 正しい手順

```
1. shopify app config link  → CLIでアプリを作成/リンク（Dev Dashboard に登録される）
2. shopify app dev          → 「開発ストアがない」と言われたら y を押す
3. Dev Dashboard でストア作成 → CLIが認識するストアができる
4. shopify app dev --store xxx.myshopify.com → 起動成功
```

**ストアを先に作りたい場合** も、Dev Dashboard から作る必要があります。
Partners ダッシュボードから作ったストアは CLI 連携に使えません。

---

### アプリの認証フロー

Shopifyアプリは **OAuth** という仕組みで認証します。
ただし、Shopify CLIが大部分を自動処理してくれるので、自分で書くコードは少ないです。

```
ストアオーナー                  Shopify                    アプリサーバー
    │                           │                           │
    │── アプリをインストール ────►│                           │
    │                           │── 認証リクエスト ─────────►│
    │                           │                           │── アクセストークン保存
    │                           │◄── 認証成功 ──────────────│
    │◄── インストール完了 ───────│                           │
    │                           │                           │
    │── 管理画面でアプリを開く ──►│                           │
    │                           │── セッション検証 ─────────►│
    │                           │                           │── DBからセッション取得
    │                           │◄── OK ────────────────────│
    │◄── アプリ画面表示 ─────────│                           │
```

### `shopify.server.ts` — アプリの心臓部

```typescript
const shopify = shopifyApp({
  apiKey: process.env.SHOPIFY_API_KEY,        // アプリの識別子
  apiSecretKey: process.env.SHOPIFY_API_SECRET, // 秘密鍵
  apiVersion: ApiVersion.October25,            // 使うAPIのバージョン
  scopes: process.env.SCOPES?.split(","),      // アプリが要求する権限
  appUrl: process.env.SHOPIFY_APP_URL,         // アプリのURL
  sessionStorage: new PrismaSessionStorage(prisma), // セッションをDBに保存
  distribution: AppDistribution.ShopifyAdmin,   // カスタムアプリ
});
```

このファイルから `authenticate` 関数がエクスポートされ、各ルートで使われます。

### スコープ（permissions）

```toml
# shopify.app.toml
scopes = "read_products,read_customers,write_customers"
```

- `read_products` — 商品情報を読み取れる（お気に入り一覧で商品名・画像を表示するため）
- `read_customers` — 顧客情報を読み取れる（誰がお気に入りしたか特定するため）
- `write_customers` — 顧客のメタフィールドを書き込める（将来的な拡張用）

### App Proxy — ストアフロントからアプリへの橋渡し

```
お客さんのブラウザ
    │
    │── GET https://your-store.com/apps/wishlist?customerId=123
    │
    ▼
Shopify（プロキシサーバー）
    │
    │── GET https://your-app.com/api/wishlist?customerId=123
    │   + HMAC署名（リクエストが改ざんされていない証明）
    │
    ▼
アプリサーバー
    │── HMAC署名を検証
    │── データを返す
```

**なぜApp Proxyが必要？**
- ストアフロント（お客さんが見る画面）から直接アプリサーバーにアクセスすると **CORS エラー** になる
- App Proxyを通すことで、ストアのドメイン (`your-store.com/apps/wishlist`) 経由でアクセスできる
- Shopifyが自動でHMAC署名を追加してくれるので、**リクエストの正当性を検証できる**

---

## 6. Theme App Extension — ストアフロントに飛び出すアプリ

### Theme App Extension とは？

> ストアのテーマ（見た目）に **アプリの部品を埋め込む** 仕組み

テーマのコードを直接編集せずに、アプリから部品（ブロック）を提供できます。
ストアオーナーがテーマエディタから配置場所を選べます。

```
extensions/wishlist-button/
├── shopify.extension.toml    ← Extension の設定ファイル
├── blocks/
│   └── wishlist-button.liquid ← Liquid テンプレート（HTMLの設計図）
└── assets/
    ├── wishlist-button.js     ← JavaScript（ボタンの動作）
    └── wishlist-button.css    ← スタイル（見た目）
```

### App Block の仕組み

```liquid
{% comment %} wishlist-button.liquid {% endcomment %}

{% if customer %}
  {%- comment -%} customer はShopifyが自動で注入する変数。ログイン中の顧客情報 {%- endcomment -%}
  <div id="wishlist-button"
    data-product-id="{{ product.id }}"
    data-customer-id="{{ customer.id }}">
    <button class="wishlist-btn">♡ お気に入り</button>
  </div>
{% else %}
  <a href="/account/login">ログインしてお気に入りに追加</a>
{% endif %}
```

**Liquid** はShopifyのテンプレート言語です。
`{{ }}` で変数を表示、`{% %}` でロジック（if文など）を書きます。

---

## 7. Customer Account UI Extension — マイページを拡張する

### Customer Account UI Extension とは？

> 顧客のマイページ（注文履歴やアカウント設定の画面）に **アプリ独自のページを追加する** 仕組み

2024年から導入された新しいタイプの Extension です。
従来の「App Proxyで一覧ページを作る」方式よりも、ネイティブな体験を提供できます。

### 3つの Extension タイプ

```
extensions/
├── wishlist-full-page/              ← 1. フルページ Extension
│   │                                   マイページに「お気に入り」という新しいページを追加
│   └── src/FullPageExtension.tsx
│
├── wishlist-profile-block/          ← 2. プロフィールブロック Extension
│   │                                   プロフィールページに「お気に入りを見る」バナーを追加
│   └── src/ProfileBlockExtension.tsx
│
└── wishlist-editor-collection/      ← 3. エディタ Extension コレクション
                                        上の2つをグループ化（管理画面でまとめて表示）
```

### Preact + Polaris Web Components

Customer Account UI Extension では、**React ではなく Preact** を使います。

- **Preact** = React の軽量版（API はほぼ同じ、サイズが1/10）
- **Polaris Web Components** = `<s-button>`, `<s-page>` のような HTML タグ型のUI部品
  - 通常の React用 Polaris (`<Button>`, `<Page>`) とは別物
  - Web Components なので、React/Preact/素のHTMLどこでも使える

```tsx
// Customer Account UI Extension のコード例
import "@shopify/ui-extensions/preact";
import { render } from "preact";
import { useState } from "preact/hooks";

export default async () => {
  render(<WishlistPage />, document.body);
};

function WishlistPage() {
  const [items, setItems] = useState([]);

  // shopify グローバルオブジェクトで API にアクセス
  // shopify.sessionToken.get()  — セッショントークン取得
  // shopify.query()             — Storefront API クエリ
  // shopify.settings.value      — マーチャント設定

  return (
    <s-page heading="お気に入り">
      <s-section>
        {items.map(item => (
          <s-stack key={item.id} direction="block" gap="base">
            <s-text>{item.title}</s-text>
          </s-stack>
        ))}
      </s-section>
    </s-page>
  );
}
```

---

## 8. 用語集

| 用語 | 説明 |
|------|------|
| **Partners ダッシュボード** | `partners.shopify.com`。公開アプリの管理・収益管理用。CLIとは連携しない |
| **Dev Dashboard** | Shopify CLIが内部的に使う開発者向けダッシュボード。`shopify app dev` はここのストアしか認識しない |
| **Shopify ID** | メールアドレスで管理される共通アカウント。Partners・Dev Dashboard・ストアすべてに使う |
| **開発ストア** | テスト用の無料ストア。Partners版とDev Dashboard版があり、**CLIが使えるのはDev Dashboard版のみ** |
| **ORM** | Object-Relational Mapping。DBのテーブルをプログラミング言語のオブジェクトとして扱う仕組み。Prisma がこれにあたる |
| **マイグレーション** | DBスキーマの変更を管理する仕組み。git の commit に似ている |
| **スキーマ** | データの構造を定義したもの。「このテーブルにはこういうカラムがある」という設計図 |
| **OAuth** | 「あなたの代わりにこのアプリが操作していいですか？」を許可する仕組み |
| **セッション** | 「この人はログイン済みである」という状態を保持する仕組み |
| **App Proxy** | Shopifyがストアのドメイン経由でアプリサーバーにリクエストを転送する仕組み |
| **HMAC** | リクエストが改ざんされていないことを証明するハッシュ値 |
| **CORS** | ブラウザが「異なるドメインへのリクエストをブロックする」セキュリティ機能 |
| **Extension** | アプリからShopifyプラットフォームに機能を追加する仕組みの総称 |
| **App Block** | Theme App Extension で提供する「テーマに埋め込める部品」 |
| **loader** | React Router v7 でページを開くとき（GET）にサーバーで実行される関数 |
| **action** | React Router v7 でフォーム送信など（POST）のときにサーバーで実行される関数 |
| **Polaris** | Shopify が提供する UI コンポーネントライブラリ |
| **Polaris Web Components** | Polaris を Web Components (`<s-*>`) として使えるようにしたもの |
| **Preact** | React の軽量代替ライブラリ。API互換で、サイズが約3KB |
| **GraphQL** | APIのクエリ言語。「欲しいデータだけ指定して取得」できる |
| **Storefront API** | 顧客側（ストアフロント）から使えるShopify API |
| **Admin API** | 管理者側から使えるShopify API（より多くの操作が可能） |
| **TOML** | 設定ファイルの形式（`shopify.app.toml` など） |
| **cuid** | 衝突しにくいユニークIDの生成方式。UUIDより短く、ソート可能 |
| **git commit** | コードの変更をローカルPCに保存する。GitHubには反映されない |
| **git push** | ローカルのcommitをGitHubにアップロードする。これで初めてGitHubに反映 |
| **git pull** | GitHubの変更をローカルにダウンロードする |
| **git add** | 次のcommitに含めるファイルを選択する（ステージング） |
| **ローカル** | 自分のPC上のリポジトリ。commit はここに保存される |
| **リモート（origin）** | GitHub上のリポジトリ。push すると反映される |
| **MCP** | Model Context Protocol。Claude Code に外部ツール（APIドキュメント検索など）を接続する仕組み。設定は `.mcp.json` に保存 |

---

## 9. MCP（Model Context Protocol）— Claude Code の拡張機能

### MCP とは？

> Claude Code に **外部の知識やツールを接続する** 仕組み

このプロジェクトでは `@shopify/dev-mcp` を使って、Claude Code から Shopify の公式ドキュメント検索・GraphQL スキーマ参照・コンポーネントバリデーションができるようになっています。

### 設定ファイル

```
プロジェクトルート/
└── .mcp.json   ← MCP サーバーの設定（git に含まれる）
```

```json
{
  "mcpServers": {
    "shopify-dev-mcp": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@shopify/dev-mcp@latest"]
    }
  }
}
```

### PC を変えたらどうなる？

```
┌─────────────────────────┐     ┌─────────────────────────┐
│  旧 PC                   │     │  新 PC                   │
│                          │     │                          │
│  .mcp.json ──── git ────────►  .mcp.json（自動で来る）   │
│  npx キャッシュ  ✗ 消える│     │  npx キャッシュ ← 自動DL │
│  Node.js         ✗ 消える│     │  Node.js  ← 要インストール│
└─────────────────────────┘     └─────────────────────────┘
```

| 項目 | 引き継がれる？ | 理由 |
|------|:---:|------|
| `.mcp.json` | ✅ | git リポジトリに含まれる |
| MCP サーバー本体 | ❌ → 自動復元 | `npx -y` が初回起動時に自動ダウンロード |
| Node.js / npx | ❌ → 手動 | OS に依存するため、新 PC に別途インストールが必要 |

**結論**: Node.js さえ入っていれば、`git clone` → Claude Code を開くだけで MCP は動く。手動設定は不要。

---

## 10. ブラウザ開発者ツール — エラーの見つけ方

### 開き方

Mac: `Cmd + Option + I` / Windows: `F12` または `Ctrl + Shift + I`

### 主要なタブと「いつ見るか」

```
┌──────────┬──────────────────────────────────────────────────────┐
│ タブ      │ いつ・何を見る？                                       │
├──────────┼──────────────────────────────────────────────────────┤
│ Console  │ JavaScript エラー・警告・ログを確認                      │
│          │ 赤い文字 = エラー、黄色 = 警告                          │
│          │ 「Extension が表示されない」→ まずここを見る              │
├──────────┼──────────────────────────────────────────────────────┤
│ Network  │ HTTP リクエスト/レスポンスを確認                         │
│          │ 「ファイルが読み込めない」「API が失敗」→ ここを見る       │
│          │ Status 200=成功, 4xx=クライアントエラー, 5xx=サーバーエラー│
├──────────┼──────────────────────────────────────────────────────┤
│ Elements │ HTML/CSS の構造を確認                                   │
│          │ 「見た目がおかしい」→ ここを見る                         │
├──────────┼──────────────────────────────────────────────────────┤
│ Sources  │ JavaScript のソースコードを確認                          │
│          │ ブレークポイントを設定してステップ実行                     │
└──────────┴──────────────────────────────────────────────────────┘
```

### Console タブの読み方

```
🔴 赤アイコン = エラー（要対応）
🟡 黄アイコン = 警告（大体は無視してOK）
🔵 青アイコン = 情報ログ
⚪ グレー    = デバッグログ (console.log)
```

**よくあるパターン**:

| エラーメッセージ | 意味 | 対応 |
|---|---|---|
| `net::ERR_BLOCKED_BY_CLIENT` | 広告ブロッカーがリクエストをブロック | 無視 or ブロッカーを無効化 |
| `Uncaught ReferenceError: X is not defined` | 変数 X が存在しない | コードのスペルミスか import 忘れ |
| `Uncaught TypeError: Cannot read properties of null` | null のプロパティにアクセス | 要素が見つからない or データが未取得 |
| `Failed to fetch` | HTTP リクエストが失敗 | Network タブで詳細を確認 |
| `CORS error` | クロスオリジンリクエストがブロック | サーバー側の CORS 設定を確認 |

**Console のフレーム切り替え** (重要):

Extension は Web Worker/iframe 内で動くため、**コンソール上部のドロップダウン（デフォルト「top」）** を切り替えないと Extension のエラーが見えないことがある。

### Network タブの読み方

```
┌───────────────────────────────────────────────────────┐
│  Name         Status   Type       Size    Time        │
│  ─────────────────────────────────────────────────    │
│  wishlist.js  200 OK   script     24KB    120ms  ✅   │
│  api/data     404      fetch      0B      50ms   ❌   │
│  metrics      (blocked) xhr       0B      0ms    🚫   │
└───────────────────────────────────────────────────────┘
```

**チェックポイント**:
1. **Status Code**: 200=成功, 404=見つからない, 500=サーバーエラー
2. **Type**: script=JS, fetch/xhr=API呼び出し, document=HTML
3. **Size**: 異常に大きい → 不要な依存が混入している可能性
4. **フィルタ**: URL でフィルタして自分のリクエストだけ表示

### Shopify Extension デバッグのフローチャート

```
Extension が表示されない
    │
    ├─ shopify app dev のターミナルにエラーは？
    │   └─ YES → ビルドエラー。エラーメッセージを読んで修正
    │
    ├─ Network タブで .js ファイルは 200 OK で読み込めてる？
    │   └─ NO → トンネル URL か Extension 設定の問題
    │
    ├─ .js のファイルサイズは正常？ (Preact なら ~25KB 前後)
    │   └─ 異常に大きい (60KB+) → 不要な依存混入 (React 等)
    │
    ├─ Console にエラーは？
    │   ├─ remote-ui エラー → Extension のランタイムクラッシュ
    │   ├─ ReferenceError → 未定義変数・import 忘れ
    │   └─ ERR_BLOCKED_BY_CLIENT → 広告ブロッカー（通常は無害）
    │
    └─ Console のフレームを切り替えた？ (top → extension の Worker)
        └─ Extension 固有のエラーはそこに出る
```

### 実例：今回の TSX + React JSX ランタイム問題

1. **Console**: `XHR request errored` (remote-ui_async) — Extension がクラッシュ
2. **Network**: `wishlist-full-page.js` → 200 OK (読み込みは成功)
3. **バンドルサイズ**: 64,245 bytes — Preact 版にしては異常に大きい
4. **バンドル内容**: `var t="18.3.1"` — React 18.3.1 が混入していた
5. **原因**: `.tsx` で `@jsxImportSource` 未指定 → React JSX ランタイムが使われた
6. **修正**: `/** @jsxImportSource preact */` を先頭に追加 → 24,071 bytes に激減

### トラブルシューティング

MCP 接続に失敗した場合は npx キャッシュ破損の可能性がある。
詳細は `.claude/docs/known-issues.md` の「npx キャッシュ破損」を参照。

---

## 11. Claude Code Skills（カスタムコマンド）— 繰り返し作業を自動化する

### Skills とは？

> Claude Code に **独自のコマンド（`/コマンド名`）を追加する** 仕組み

よくある操作をテンプレート化して、毎回同じ指示を書かなくて済むようにします。
例: エラーを解決したら `/log-fix` で記録する、など。

### ディレクトリ構成

```
.claude/
└── skills/
    └── log-fix/          ← スキル名がフォルダ名
        └── SKILL.md      ← スキルの定義ファイル（必須）
```

### SKILL.md の書き方

```markdown
---
name: log-fix
description: 直前のエラーとその解決方法を known-issues.md に記録する。
---

ここに Claude Code への指示を書く。
会話の中で `/log-fix` と入力すると、この指示が実行される。

1. 直前の会話からエラー内容と解決方法を要約
2. `.claude/docs/known-issues.md` に追記
3. 同じパターンが既にあれば重複せず更新
```

**ポイント:**
- `---` で囲まれた部分が **フロントマター**（メタデータ）
- `name` — コマンド名（`/` の後に入力する文字列）
- `description` — スキルの説明（Claude Code が候補を表示するときに使う）
- フロントマターの下が **実際の指示内容**（プロンプト）

### 新しいスキルを追加する手順

```
1. .claude/skills/ の下にフォルダを作る
   └─ フォルダ名 = コマンド名（例: review-code/）

2. フォルダ内に SKILL.md を作成
   └─ フロントマター（name, description）+ 指示内容

3. Claude Code を再起動（またはセッションを新しく開始）
   └─ 次のセッションから /コマンド名 で使えるようになる
```

### 実用例

| コマンド | 用途 |
|---------|------|
| `/log-fix` | エラーと解決方法を known-issues.md に記録 |
| `/review-code` | コード変更をレビューして改善点を提案（自作例） |
| `/deploy-check` | デプロイ前のチェックリストを実行（自作例） |

### 注意点

- スキルは **同じセッション内では反映されない** ことがある → 新しいセッションで確認
- SKILL.md のフロントマターに構文エラーがあると認識されない
- 指示内容は具体的に書くほど正確に動作する（曖昧だと毎回異なる結果になる）
- `.claude/skills/` ディレクトリは git に含めることで、チームで共有できる

---

## 12. サーバーとトンネル — テーマ開発とアプリ開発の決定的な違い

### テーマ開発 vs アプリ開発 — 根本的に違うもの

テーマ開発の経験があると「Shopify にファイルをアップロードすれば動く」という感覚がある。
アプリ開発はそれとは **根本的に仕組みが違う**。

```
テーマ開発:
  自分のPC → shopify theme push → Shopify のサーバーにファイルが置かれる → 終わり
  Shopify がすべてホスティング（サーバーの心配なし）

アプリ開発:
  自分のPC → 自分でサーバーを用意 → Shopify が「あのサーバーにいるアプリ」として認識
  自分のコードを動かすサーバーが別途必要
```

### なぜアプリにはサーバーが必要なのか

テーマは **見た目のテンプレート** なので Shopify が表示してくれる。
アプリは **独自のロジックを実行するプログラム** なので、そのプログラムを動かす場所（サーバー）が要る。

```
┌──────────────────────────────────────────────────────────────┐
│ テーマ開発                                                    │
│                                                              │
│   Liquid ファイル                                             │
│       ↓ shopify theme push                                   │
│   Shopify のサーバー ← Shopify が全部やってくれる               │
│       ↓                                                      │
│   お客さんのブラウザに表示                                      │
│                                                              │
│   ★ 自分のサーバーは不要。Shopifyに預けるだけ。                  │
├──────────────────────────────────────────────────────────────┤
│ アプリ開発                                                    │
│                                                              │
│   React Router + Prisma                                      │
│       ↓ fly deploy                                           │
│   Fly.io のサーバー ← 自分で用意するサーバー                    │
│       ↑                                                      │
│   Shopify が「このURLにアプリがいる」と認識                      │
│       ↑                                                      │
│   ストアオーナー/お客さんがアプリを使う                          │
│                                                              │
│   ★ 自分のサーバーが必要。コードを実行する場所を自分で管理。      │
└──────────────────────────────────────────────────────────────┘
```

**テーマと対比すると:**
| | テーマ | アプリ |
|---|---|---|
| コードの置き場所 | Shopify のサーバー | **自分のサーバー**（Fly.io 等） |
| データベース | Shopify が管理（商品、注文等） | **自分で用意**（PostgreSQL 等） |
| デプロイ方法 | `shopify theme push` | `fly deploy` + `shopify app deploy` |
| 月額コスト | 無料（テーマはストアの一部） | サーバー代（Fly.io 無料枠あり） |
| ダウンタイムの責任 | Shopify | **自分** |

### サーバーとは？

> アプリのコードを **24時間365日、インターネット上で動かし続ける** コンピューター

テーマ開発では Shopify がすべてホスティングしてくれるので意識しない。
アプリ開発では、Wishlist データの読み書きやAPIの処理を行う「自分のコンピューター」が必要。

```
「サーバー」のイメージ:

テーマ開発の世界:
  「Shopifyさん、このLiquidファイルを表示してください」
  → Shopifyが自分のサーバーで処理してくれる（あなたはサーバーを意識しない）

アプリ開発の世界:
  「Fly.io さん、この Node.js アプリを動かし続けてください」
  → Fly.io がサーバーを貸してくれる（あなたがコードとDBを管理する）
  → Shopify は「https://your-app.fly.dev にアプリがいる」と覚えるだけ
```

**サーバーの選択肢:**
| サービス | 特徴 | 無料枠 |
|---------|------|--------|
| **Fly.io** | Shopify 公式推奨。Docker ベース | あり（小規模なら無料） |
| Railway | GUI が使いやすい。GitHub 連携 | あり |
| Render | シンプル。自動デプロイ | あり |
| Heroku | 老舗。情報が多い | なし（有料のみ） |

### トンネルとは？

> 開発中に **自分のPCを一時的にサーバーに見せかける** 仕組み

テーマ開発では `shopify theme dev` で自分のPCをプレビューサーバーにする。
アプリ開発でも似た仕組みがあるが、もう一段複雑。

```
テーマ開発の shopify theme dev:
┌──────────┐                    ┌──────────┐
│ 自分のPC  │ ← localhost:9292  │ ブラウザ  │
│ (テーマ)  │ ──────────────── → │ (プレビュー)│
└──────────┘   直接接続         └──────────┘

  ★ ブラウザが直接 localhost にアクセスする（シンプル）

アプリ開発の shopify app dev:
┌──────────┐                    ┌──────────┐      ┌──────────┐
│ 自分のPC  │ ← トンネル →      │ Shopify  │ ←── │ ブラウザ  │
│ (アプリ)  │ xxxx.trycloudflare│ のサーバー│      │ (管理画面)│
│ localhost │ .com              │          │      │          │
│ :3000     │                   │          │      │          │
└──────────┘                    └──────────┘      └──────────┘

  ★ Shopify が「xxxx.trycloudflare.com にアプリがいる」と認識
  ★ 実際にはそのURLが自分のPCの localhost:3000 に転送されている
  ★ トンネル = インターネット上の一時的な入り口を作る仕組み
```

**なぜトンネルが必要？**

テーマ開発: ブラウザが `localhost` に直接アクセスできる（同じPCだから）。
アプリ開発: Shopify のサーバーが自分のPCにアクセスする必要がある。
でも `localhost` はインターネットから見えない。
→ **トンネルがインターネット上に一時的な入り口を作り、Shopifyからのリクエストを自分のPCに転送する**。

```
トンネルの仕組み（もう少し詳しく）:

1. shopify app dev を実行
2. Cloudflare Tunnel が起動（自動）
3. ランダムなURL が生成される
   例: https://abc123.trycloudflare.com
4. Shopify に「このURLにアプリがいます」と登録
5. Shopify → abc123.trycloudflare.com → 自分のPC (localhost:3000) に転送
6. 開発中はすべてのリクエストがこの経路を通る
```

**テーマ開発と比較:**
| | テーマ (`theme dev`) | アプリ (`app dev`) |
|---|---|---|
| ブラウザのアクセス先 | `localhost:9292` | `admin.shopify.com/store/xxx/apps/...` |
| Shopify の関与 | プレビューデータ提供のみ | **リクエスト転送の中継** |
| トンネルの必要性 | 不要 | **必須**（Shopifyがアクセスするため） |
| URLの安定性 | 常に localhost | **毎回変わる**（一時的なトンネルURL） |
| 本番との差 | `theme push` で同じファイル | サーバー環境が完全に異なる |

### 本番デプロイ = トンネルの代わりに固定サーバーを置く

開発中はトンネル（一時的なURL）を使うが、本番では**固定のサーバー**を使う。

```
開発中:
  Shopify → xxxx.trycloudflare.com (毎回変わる) → 自分のPC
                                                   ↑ PCを閉じたら動かない

本番:
  Shopify → your-app.fly.dev (固定URL) → Fly.io のサーバー
                                          ↑ 24時間365日動き続ける
```

**テーマとの対比:**
- テーマ開発: `shopify theme push` → Shopify のサーバーにファイルが置かれる → 完了
- アプリ開発: `fly deploy` → Fly.io にアプリが配置される → Shopify にURLを登録 → 完了

どちらも「本番の場所にコードを配置する」点は同じ。
違いは、テーマは Shopify がホスト、アプリは自分のサーバー（Fly.io）がホストする点。

### Extension はどこで動く？

ここが混乱しやすい点。アプリは3つのパーツに分かれる。

```
┌───────────────────────────────────────────────────────────────┐
│ Shopify Wishlist App の3つのパーツ                              │
│                                                               │
│  ① アプリ本体（サーバーサイド）                                  │
│     ├─ React Router + Prisma                                  │
│     ├─ API エンドポイント、管理画面                              │
│     ├─ デプロイ先: Fly.io（自分のサーバー）                      │
│     └─ テーマで例えると: 存在しない概念（テーマにはサーバーがない）│
│                                                               │
│  ② Theme App Extension（♡ボタン）                              │
│     ├─ Liquid + JS + CSS                                      │
│     ├─ デプロイ先: Shopify CDN（Shopifyが管理）                 │
│     └─ テーマで例えると: セクション/スニペット と同じ感覚         │
│                                                               │
│  ③ Customer Account Extension（マイページ一覧）                │
│     ├─ Preact バンドル (.js)                                   │
│     ├─ デプロイ先: Shopify CDN（Shopifyが管理）                 │
│     └─ テーマで例えると: theme push で Shopify に置く感覚       │
│                                                               │
│  ★ ②③ は shopify app deploy で Shopify に送る（theme push に近い）│
│  ★ ① だけ自分のサーバーに置く必要がある                          │
└───────────────────────────────────────────────────────────────┘
```

### 環境変数とは？

> サーバーに **秘密の設定値を渡す仕組み**

テーマ開発では `.env` や環境変数をほぼ使わない（Shopify が管理するため）。
アプリ開発では、APIキーやDB接続先などの秘密情報をサーバーに渡す必要がある。

```
テーマ開発:
  Shopify が秘密情報を管理 → 開発者は意識しない

アプリ開発:
  .env ファイル（ローカル開発用）:
    SHOPIFY_API_KEY=abc123
    SHOPIFY_API_SECRET=xyz789
    DATABASE_URL=postgresql://...

  fly secrets set（本番サーバー用）:
    fly secrets set SHOPIFY_API_KEY=abc123
    fly secrets set DATABASE_URL=postgresql://...

  ★ .env は git に入れてはいけない（秘密情報が漏れる）
  ★ 本番は fly secrets set でサーバーに直接設定する
```

### Docker とは？（Fly.io が使う）

> アプリの実行環境を **箱詰め** にする仕組み

テーマ開発では全く登場しない概念。アプリ開発では、Fly.io が Docker を使って
「Node.js + アプリコード + 設定」をまるごとパッケージして配置する。

```
Docker のイメージ:

引っ越しに例えると:
  テーマ = 家具だけ送ればいい（Shopify が家を用意してくれる）
  アプリ = 家具 + 家 + 電気水道 をまるごと箱詰めして送る

Dockerfile の中身（概要）:
  1. Node.js をインストール
  2. npm install で依存パッケージをインストール
  3. npm run build でアプリをビルド
  4. npm start でアプリを起動

  → この手順書通りにFly.ioが自動でセットアップしてくれる
```

**覚えること:** Dockerfile はアプリの「セットアップ手順書」。Fly.io がそれを読んで自動で環境を作る。

### 本番構成の全体像 — 3つのサービスが URL でつながる

テーマ開発はすべてが Shopify の中で完結するが、アプリ開発は **3つの独立したサービスを URL で接続する**。

```
テーマ開発（全部 Shopify の中）:
┌──────────────────────────────────────────────┐
│ Shopify                                      │
│                                              │
│  Liquid ファイル → Shopify サーバー → 表示    │
│  商品データ     → Shopify DB                  │
│                                              │
│  ★ 全部 Shopify がやってくれる                 │
└──────────────────────────────────────────────┘

アプリ開発（3つのサービスを URL で接続）:
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Shopify      │   │ Fly.io       │   │ Neon         │
│              │   │ (サーバー)    │   │ (データベース) │
│ Extensions   │   │              │   │              │
│ ・♡ボタン    │   │ React Router │   │ PostgreSQL   │
│ ・マイページ  │   │ + Prisma     │   │ Wishlist     │
│              │   │              │   │ Session      │
│ ストア管理画面│   │ API処理      │   │ テーブル      │
│ (Admin)     │   │ 認証処理      │   │              │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       │    HTTPリクエスト  │   DATABASE_URL   │
       │◄────────────────►│◄────────────────►│
       │                  │                  │
       │  APP_URL で接続    │  DATABASE_URL    │
       │  (shopify.app.toml│  で接続           │
       │   に記載)         │  (fly secrets)   │
```

### サービス間をつなぐ「アダプタ」= URL（接続文字列）

テーマ開発ではすべて Shopify 内にあるので接続を意識しない。
アプリ開発では **URL** がサービスをつなぐアダプタの役割を果たす。

| 接続 | つなぐもの | アダプタ（接続方法） | どこに設定する |
|------|-----------|-------------------|-------------|
| Shopify → Fly.io | ストアとアプリ | **APP_URL** | `shopify.app.toml` |
| Fly.io → Neon | アプリとDB | **DATABASE_URL** | `fly secrets set` |
| Fly.io → Shopify | アプリからAPI呼び出し | **API_KEY + API_SECRET** | `fly secrets set` |

```
具体的な設定例:

① Shopify → Fly.io をつなぐ
   shopify.app.toml:
     application_url = "https://shopify-wishlist-red-breeze-4526.fly.dev"
   → Shopify が「このURLにアプリがいる」と認識

② Fly.io → Neon をつなぐ
   fly secrets set DATABASE_URL="postgresql://user:pass@xxx.neon.tech/dbname"
   → Prisma が「このURLにDBがある」と認識

③ Fly.io → Shopify をつなぐ
   fly secrets set SHOPIFY_API_KEY="xxx" SHOPIFY_API_SECRET="yyy"
   → アプリが Shopify API を呼べるようになる
```

**テーマ開発に例えると:**
- **APP_URL** = テーマのプレビューURL（どこにアプリがあるか）
- **DATABASE_URL** = 商品データの在り処（テーマでは Shopify が自動管理するが、アプリでは自分で指定）
- **API_KEY** = ストアへのアクセス権（テーマでは Shopify 内なので不要だが、アプリは「外」にいるので鍵が必要）

### サーバーの選択肢 — なぜ Fly.io か

| サービス | 無料枠 | Shopify 公式推奨 | 特徴 |
|---------|:---:|:---:|------|
| **Fly.io** | あり（256MB x 3台） | **テンプレートに Dockerfile 同梱** | CLI で `fly deploy` |
| Railway | 月$5クレジット | No | GitHub 連携で自動デプロイ |
| Render | あり（750時間/月） | No | GUI が簡単 |
| Koyeb | あり（1インスタンス） | No | GitHub 連携 |

**Fly.io を選ぶ理由:**
1. プロジェクトの `Dockerfile` がそのまま使える（Shopify テンプレートが Fly.io 前提で作られている）
2. `fly launch` が Shopify アプリを自動検出して設定してくれる
3. 東京リージョン (`nrt`) がある

他のサービスでもデプロイ可能だが、Dockerfile の調整が必要になる場合がある。

### DB の選択肢 — なぜ Neon か

| サービス | 無料枠 | 特徴 |
|---------|------|------|
| Fly.io Managed Postgres | **月$38〜（無料枠なし）** | Fly 内で完結するが高い |
| **Neon** | **0.5GB まで無料** | サーバーレス PostgreSQL。URLを設定するだけ |
| Supabase | 500MB まで無料 | PostgreSQL + 管理画面付き |
| PlanetScale | MySQL のみ | Prisma は使えるが MySQL |

**Neon を選ぶ理由:**
1. **無料** — 学習プロジェクトにコストをかけない
2. **PostgreSQL** — Prisma でそのまま使える
3. **サーバーレス** — 使わないときはコストゼロ
4. **接続が簡単** — `DATABASE_URL` を設定するだけ

### Fly Launch の設定値の意味

今回設定した値の解説:

| 項目 | 設定値 | テーマ開発で例えると |
|------|--------|-------------------|
| App name | `shopify-wishlist-red-breeze-4526` | テーマ名（ストアで識別するための名前） |
| Region | `nrt - Tokyo, Japan` | — (テーマにリージョンの概念はない) |
| Internal port | `3000` | — (テーマにポートの概念はない) |
| VM Sizes | `shared-cpu-1x` | — (テーマに CPU の概念はない) |
| VM Memory | `256MB` | — (テーマにメモリの概念はない) |
| Postgres | `none` | Shopify DB は自動提供。アプリは自分で用意 |
| Tigris | `Disabled` | 画像ストレージ。テーマ画像は Shopify CDN に自動配置 |

**ポイント:** テーマ開発で意識しない「リージョン」「CPU」「メモリ」「ポート」は、
すべて **Shopify が裏で自動管理している**。アプリ開発では自分で選ぶ必要がある。

---

## 13. 開発ワークフロー — ローカル開発からデプロイまで

### Shopify アプリ開発の全体サイクル

> **ローカルで開発 → デプロイで公開 → またローカルで開発** を繰り返す

```
┌────────────────────────────────────────────────────────────────┐
│                    開発サイクル（繰り返し）                       │
│                                                                │
│   ① ローカル開発         ② デプロイ          ③ 本番確認         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ shopify app  │    │ shopify app  │    │ 本番ストアで  │      │
│  │ dev          │──► │ deploy       │──► │ 動作確認      │      │
│  │              │    │              │    │              │      │
│  │ コード編集    │    │ Extensions   │    │ 問題なし？    │      │
│  │ ブラウザ確認  │    │ + アプリ本体  │    │  → ①に戻る   │      │
│  │ エラー修正    │    │ を本番反映    │    │  → 次の機能へ │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                                │
│  ※ ①と②は独立。デプロイ後もローカル開発はそのまま継続できる       │
└────────────────────────────────────────────────────────────────┘
```

### ① ローカル開発（`shopify app dev`）

日常的な開発はすべてローカルで行う。

```bash
# 開発サーバーを起動（トンネル経由で Shopify に接続）
shopify app dev

# やること:
# - コードを書く・編集する
# - ブラウザでリアルタイム確認（HMR で自動反映）
# - 開発ストアでテスト
# - エラーが出たら修正 → /log-fix で記録
```

**ポイント:**
- `shopify app dev` は毎回異なるトンネルURLを生成する（一時的）
- Extension の `backend_url` 設定は毎回変わる可能性がある
- **本番デプロイすると固定URLになる** → Extension 設定が安定する

### ② デプロイ（`shopify app deploy`）

機能が完成したら本番に反映する。

```bash
# アプリ本体 + 全 Extension を本番にデプロイ
shopify app deploy
```

**デプロイされるもの:**

| 対象 | 内容 |
|------|------|
| アプリ本体 | サーバーコード（Fly.io 等にホスト） |
| Theme App Extension | ♡ボタン（Liquid + JS + CSS） |
| Customer Account Extension | マイページ一覧（Preact バンドル） |
| Extension 設定 | `shopify.extension.toml` の内容 |

### ③ デプロイ後もローカル開発を継続する

```
デプロイ後の状態:

本番:    fly.io (固定URL)     ← 顧客・ストアオーナーが使う
ローカル: shopify app dev     ← 開発者が使う（独立して動く）

→ 本番に影響を与えずにローカルで新機能を開発できる
→ 完成したら再度 shopify app deploy で本番に反映
```

### 本番デプロイに必要なもの

| 項目 | 内容 | 選択肢 |
|------|------|--------|
| サーバー | アプリのホスト先 | Fly.io（無料枠あり）、Railway、Render |
| データベース | Prisma の接続先 | PostgreSQL（Fly.io Postgres / Neon / Supabase） |
| ドメイン | アプリのURL | 不要（`xxx.fly.dev` が自動付与） |
| 環境変数 | APIキー等 | サーバー側で設定（`fly secrets set`） |

### Prisma: SQLite → PostgreSQL の切り替え

```prisma
// 開発（SQLite）
datasource db {
  provider = "sqlite"
  url      = "file:dev.sqlite"
}

// 本番（PostgreSQL）
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")  // 環境変数から読み込む
}
```

切り替え手順:
1. `prisma/schema.prisma` の `provider` を変更
2. `DATABASE_URL` 環境変数を設定
3. `npx prisma migrate deploy` で本番DBにスキーマを適用

### よくある疑問

**Q: デプロイしたらローカル開発できなくなる？**
→ No。`shopify app dev` は独立して動く。本番とローカルは別環境。

**Q: デプロイは毎回全部やり直す？**
→ No。`shopify app deploy` は差分だけを反映する（変更があった部分のみ）。

**Q: Extension だけデプロイできる？**
→ Yes。`shopify app deploy` で Extension のみの更新も可能。アプリ本体は別途サーバーにデプロイ。

**Q: いつデプロイすべき？**
→ 機能がひとまとまり完成したタイミング。Git の commit と同じ感覚で「区切り」ごとに。

---


## 実装ログ

> 以下は開発を進めるなかで追記されるメモです。
> Phase ごとにハマったポイントと教訓を記録してください。

<!-- ### YYYY-MM-DD: Phase X — タイトル -->
