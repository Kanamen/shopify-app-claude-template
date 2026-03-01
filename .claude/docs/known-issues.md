# 既知のエラーと解決方法

> エラーを解決したら `/project:log-fix` コマンドで追記すること。
> 同じパターンが既にあれば重複せず既存エントリを更新する。

---

## 2026-03-01 Customer Account Extension で shopify.query() が Customer Account API ではない

- **症状**: Extension からウィッシュリストデータが取得できない。バックエンドのログにリクエストが一切来ない。Extension は「0件のお気に入り」を表示
- **原因**: `shopify.query()` は **Storefront API** を叩く。`customer { id }` は Storefront API に存在しないため失敗し、顧客 ID が取得できず fetch がスキップされていた
- **解決**: Customer Account API には `fetch('shopify://customer-account/api/2025-10/graphql.json')` でアクセスする
  ```typescript
  const res = await fetch(
    "shopify://customer-account/api/2025-10/graphql.json",
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ query: "query { customer { id } }" }),
    },
  );
  const json = await res.json();
  const customerId = json?.data?.customer?.id;
  ```
- **教訓**:
  - `shopify.query()` = Storefront API（商品・コレクション等）
  - `fetch('shopify://customer-account/api/...')` = Customer Account API（顧客データ・shop情報）
  - Customer Account API にしかないクエリ: `customer { id }`, `shop { url }`, etc.
  - Customer Account API に**ないもの**: `nodes(ids:)`, `product(id:)` → 商品データはバックエンド経由で取得

---

## 2026-03-01 Customer Account API の Protected Customer Data 未有効化

- **症状**: `fetch('shopify://customer-account/api/...')` で `customer { id }` を取得しようとすると、HTTP 200 だがレスポンスに `"Level 1 protected customer data access is required to use the Customer Account API"` エラー
- **原因**: Customer Account API を使うには、**Partner Dashboard で Protected Customer Data アクセスを有効化** する必要がある。コードが正しくても、この設定がないと API がエラーを返す
- **解決**:
  1. **Partner Dashboard** (https://partners.shopify.com) → アプリ → （アプリ名） → 「**API access**」
  2. 「**Protected customer data**」セクション → 「**Request access**」
  3. Step 1: データの使用目的を選択（「アプリの機能」等）
  4. Step 2: 保護の詳細を入力
  5. 開発ストアの場合、審査不要で即有効化
- **教訓**:
  - Customer Account API はコードだけでなく **ダッシュボード設定** も必要
  - エラーメッセージが HTTP 200 で返るため、ステータスコードだけでは気づけない
  - 設定場所は **Partner Dashboard**（Dev Dashboard ではない）

---

## 2026-03-01 Customer Account API の customer_read_customers スコープ不足

- **症状**: Protected Customer Data を有効化した後も `"Access denied for customer field. Required access: \`customer_read_customers\` access scope"` エラー
- **原因**: `shopify.app.toml` の `scopes` に `customer_read_customers` が含まれていなかった。`read_customers`（Admin API 用）とは別のスコープ
- **解決**:
  1. `shopify.app.toml` の `scopes` に `customer_read_customers` を追加:
     ```toml
     scopes = "read_customers,read_products,write_customers,customer_read_customers"
     ```
  2. `shopify app deploy` で反映
- **教訓**:
  - `read_customers` = Admin API 用（サーバーサイドで顧客データ取得）
  - `customer_read_customers` = Customer Account API 用（Extension から顧客データ取得）
  - **両方必要**。名前が似ているが別物
  - Customer Account Extension を作る場合、`shopify.app.toml` に `customer_read_customers` を忘れずに追加

---

## 2026-03-01 Customer Account Extension 64KB バンドルサイズ超過

- **症状**: `Your script size is 75 KB which exceeds the 64 KB limit` でビルドは成功するが app preview が起動しない
- **原因**: `@shopify/ui-extensions-react` + `react` が extension バンドルに含まれ、React ランタイム (~50KB) がサイズ超過の主因
- **解決**:
  1. **React → Preact に移行** (`~3KB`)
  2. **`@shopify/ui-extensions-react` → Polaris Web Components** (`<s-page>`, `<s-section>`, `<s-text>` 等)
  3. **`@shopify/ui-extensions` を `2025.7.x` → `2025.10.x`** にアップグレード（`/preact` エントリが必要）
  4. `@preact/signals` を依存に追加（`@shopify/ui-extensions/preact` が内部で import する）
  5. `reactExtension()` + `useApi()` → `export default async` + グローバル `shopify` オブジェクトに移行
- **教訓**:
  - API version `2025-10` 以降の Customer Account Extension は **Preact がデフォルト**
  - React パターンは旧バージョン (`2025-07` 以前) 用。新規作成時は最初から Preact で書く
  - `peerDependencies` に変更するだけでは外部化されない（Shopify CLI の esbuild は peerDep を見ない）
- **コンポーネント対応表**:

  | React (旧) | Polaris Web Component (新) |
  |---|---|
  | `Page` | `<s-page heading="...">` |
  | `Card` / `ResourceItem` | `<s-section>` |
  | `Text` | `<s-text>` (color="subdued", type="strong") |
  | `Heading` | `<s-heading>` |
  | `Button` | `<s-button>` (variant, tone, href, onClick) |
  | `Image` | `<s-image>` (src, alt) |
  | `Link` | `<s-link>` (href) |
  | `Banner` | `<s-banner>` (tone, **title は存在しない**) |
  | `BlockStack` | `<s-stack direction="block">` |
  | `InlineStack` | `<s-stack direction="inline">` |
  | `Grid` / `GridItem` | `<s-grid gridTemplateColumns="...">` |
  | `Spinner` | `<s-spinner />` |
  | `Divider` | `<s-divider />` |

- **API 対応表**:

  | React (旧) | Preact (新) |
  |---|---|
  | `import { reactExtension, useApi, useSettings, ... } from "@shopify/ui-extensions-react/customer-account"` | `import "@shopify/ui-extensions/preact"; import { render } from "preact";` |
  | `export default reactExtension("target", () => <App />)` | `export default async () => { render(<App />, document.body); }` |
  | `const { sessionToken, query } = useApi()` | `shopify.sessionToken.get()` / `shopify.query()` |
  | `const settings = useSettings()` | `shopify.settings.value` |
  | `useState` / `useEffect` from `"react"` | `useState` / `useEffect` from `"preact/hooks"` |

- **extension の package.json テンプレート** (2025-10):
  ```json
  {
    "name": "my-extension",
    "private": true,
    "dependencies": {
      "preact": "^10.25.0",
      "@preact/signals": "^1.3.0",
      "@shopify/ui-extensions": "2025.10.x"
    }
  }
  ```

- **`s-stack` の注意点**:
  - `inlineAlignment` プロパティは存在しない → `alignItems` を使う
  - `gap` の有効値: `small-300` | `small-200` | `small` | `base` | `large` | `large-200` | `large-300`
  - `tight` / `loose` は無効 → `small` / `large` を使う

- **`s-banner` の注意点**:
  - `title` プロパティは存在しない（子要素としてテキストを渡す）
  - `status` プロパティは存在しない → `tone` を使う (`critical`, `success`, `info`)

---

## 2026-03-01 react-reconciler ビルドエラー

- **症状**: `@shopify/ui-extensions-react` → `@remote-ui/react` → peer dep `react-reconciler >=0.26.0 <0.30.0` が見つからずビルド失敗
- **原因**: Customer Account UI Extension は個別の `package.json` が必要で、react-reconciler のバージョン範囲が厳しい
- **解決**: root `package.json` に `react-reconciler@0.29.2` を明示インストール
- **教訓**: **そもそも React を使わない方が正解**（上記の 64KB 問題参照）。新規 extension は最初から Preact で作る

---

## 2026-03-01 npx キャッシュ破損 (shopify-dev-mcp)

- **症状**: `Cannot find module 'ohm-js/extras'` で MCP サーバー接続失敗
- **原因**: `~/.npm/_npx/` 配下のキャッシュが壊れている
- **解決**:
  ```bash
  # キャッシュディレクトリを特定して削除
  rm -rf ~/.npm/_npx/8f00c70bd5114901
  # 再実行で自動再インストール
  npx -y @shopify/dev-mcp@latest --version
  ```
- **教訓**: npx キャッシュ破損は `rm -rf ~/.npm/_npx/<hash>` で解決。hash はエラーの Require stack から特定できる

---

## 2026-03-01 @preact/signals 未インストール

- **症状**: `Could not resolve "@preact/signals"` でビルド失敗
- **原因**: `@shopify/ui-extensions@2025.10.x` の `/preact` エントリが `@preact/signals` を import するが、依存に含まれていない
- **解決**: extension の `package.json` に `"@preact/signals": "^1.3.0"` を追加
- **教訓**: Preact 移行時は `preact` と `@preact/signals` の **両方** が必要

---

## 2026-03-01 TSX ファイルで React JSX ランタイムが混入（Extension クラッシュ）

- **症状**: Customer Account Extension のページが「There's a problem loading this page」で表示されない。ビルド自体は成功。コンソールに `Error: XHR request errored` (remote-ui_async) が出る
- **原因**: `.tsx` ファイルに `@jsxImportSource` が未指定 → esbuild がデフォルトの **React JSX ランタイム** (`react/jsx-runtime`) でコンパイル → React 18.3.1 の完全なランタイム (~40KB) がバンドルに混入 → Remote DOM 環境で React と Preact が衝突してクラッシュ
- **発見方法**:
  1. **Network タブ**: Extension の `.js` ファイルが 200 OK でロードされていることを確認（スクリプトの読み込み自体は成功）
  2. **dist/ のバンドルサイズ確認**: `wc -c extensions/*/dist/*.js` で異常に大きいバンドル (64KB近く) を発見
  3. **バンドル内容の確認**: `head` でバンドル冒頭を読み、`var t="18.3.1"` (React バージョン) を発見
  4. **JSX コンパイル結果の確認**: `_n(Lt()).jsx(...)` — React の `jsx()` が使われていることを確認
- **解決**: `.tsx` ファイルの先頭に `/** @jsxImportSource preact */` プラグマを追加
  ```tsx
  /** @jsxImportSource preact */
  import "@shopify/ui-extensions/preact";
  import { render } from "preact";
  ```
- **効果**: バンドルサイズ 64,245 bytes → **24,071 bytes** (-63%)
- **教訓**:
  - `.jsx` ファイルと `.tsx` ファイルでは esbuild の JSX 処理が異なる
  - Shopify 公式チュートリアルは `.jsx` を使用（React JSX ランタイム問題が起きない）
  - `.tsx` を使う場合は **必ず `/** @jsxImportSource preact */` を先頭に記述**
  - バンドルサイズが異常に大きい場合は、意図しない依存の混入を疑う
- **デバッグの手順** (Extension が「ページ読み込みエラー」の場合):
  1. Network タブで `.js` ファイルの Status Code を確認 (200 OK ならスクリプトは読み込めている)
  2. `dist/` ディレクトリのバンドルサイズを確認
  3. バンドルの内容を `head` / `grep` で確認（React 文字列の有無）
  4. コンソールの `top` 以外のフレーム/Worker も確認

---

## 2026-03-01 Prisma P3019: migration_lock.toml の provider 不一致

- **症状**: `fly deploy` 後にアプリがクラッシュループ。ログに `Error: P3019 — The datasource provider 'postgresql' specified in your schema does not match the one specified in the migration_lock.toml, 'sqlite'`
- **原因**: ローカル開発で SQLite を使っていた `prisma/migrations/` がそのまま残っており、`migration_lock.toml` が `provider = "sqlite"` のまま。本番用に `schema.prisma` を `postgresql` に変更しても、既存のマイグレーション履歴と矛盾する
- **解決**:
  1. `.env` に Neon の `DATABASE_URL` を設定（ローカルから Neon に直接接続）
  2. `rm -rf prisma/migrations` で旧 SQLite マイグレーションを削除
  3. `npx prisma migrate dev --name init` で PostgreSQL 用の新しいマイグレーションを生成
  4. `fly deploy` で再デプロイ → `migration_lock.toml` が `postgresql` になり成功
- **教訓**:
  - SQLite → PostgreSQL に切り替える時は、**マイグレーション履歴をリセット**する必要がある
  - `migration_lock.toml` は自動生成されるファイルだが、provider の変更は手動対応が必要
  - ローカルの `.env` に本番 DB の URL を設定すると、マイグレーション生成と適用を同時に行える

---

## 2026-03-01 Customer Account Extension の Network Access 未承認

- **症状**: `shopify app deploy` でバージョンは作成されるが、`New version created, but not released` と表示。エラーメッセージ: `Network access must be requested and approved in order for the wishlist-full-page extension to be published`
- **原因**: Customer Account UI Extension がバックエンド API（Fly.io サーバー）に HTTP リクエストを送る場合、Shopify の **Network Access 承認** が必要
- **解決**:
  1. **Partner Dashboard** (https://partners.shopify.com) を開く（Dev Dashboard ではない！）
  2. アプリ → （アプリ名） → 「**API access**」をクリック
  3. 「**Allow network access in checkout and account UI extensions**」セクションで「**Allow network access**」をクリック
  4. 即座に自動承認される
  5. 再度 `shopify app deploy` を実行 → `New version released to users.` で成功
- **教訓**:
  - Customer Account Extension が外部 API にアクセスするには Network Access の承認が必要
  - Theme App Extension（♡ボタン）は App Proxy 経由のため不要
  - **承認は Dev Dashboard ではなく Partner Dashboard で行う**（これがわかりにくい）
  - カスタムアプリの場合は即承認（審査なし）

---

## 2026-03-01 本番デプロイの正しい手順（初回）

以下は、このプロジェクトのデプロイで実際に行った手順を「正解の順序」でまとめたもの。

### 前提
- ローカル開発（SQLite）で Phase 0〜4 が完了済み
- `shopify app dev` で動作確認済み

### Step 1: Fly.io アカウント作成 + アプリ作成

```bash
# 1. Fly CLI インストール
brew install flyctl

# 2. ログイン（ブラウザが開く）
fly auth login

# 3. アプリ作成（プロジェクトルートで実行）
fly launch
#   → Dockerfile を自動検出
#   → アプリ名を自動生成（例: shopify-wishlist-little-thunder-3464）
#   → リージョン選択: nrt（東京）
#   → PostgreSQL: None を選択（Neon を使うため）
#   → Redis: None
#   → 初回デプロイは NO（先に DB 設定が必要）
```

**注意**: Fly.io Managed PostgreSQL は無料プランなし（最低 $38/月）。代わりに Neon を使う。

### Step 2: Neon アカウント作成 + DB 作成

```
1. https://neon.tech にアクセス → GitHub でサインアップ
2. プロジェクト作成:
   - Name: shopify-wishlist（任意）
   - Region: Asia Pacific (Singapore) ※東京はない
   - Database: neondb（デフォルト）
3. 接続文字列をコピー（postgresql://neondb_owner:xxx@xxx.neon.tech/neondb?sslmode=require）
```

### Step 3: Prisma を PostgreSQL に切り替え

```bash
# 1. schema.prisma の provider を変更
#    provider = "sqlite"  →  provider = "postgresql"
#    url = env("DATABASE_URL")  ← 既存のまま

# 2. ローカルに .env を作成（.gitignore 済み）
echo 'DATABASE_URL="postgresql://neondb_owner:xxx@xxx.neon.tech/neondb?sslmode=require"' > .env

# 3. 旧 SQLite マイグレーションを削除
rm -rf prisma/migrations

# 4. PostgreSQL 用の新しいマイグレーションを生成（Neon に直接適用される）
npx prisma migrate dev --name init
```

### Step 4: Fly.io に環境変数を設定

```bash
# DATABASE_URL をシークレットとして設定
fly secrets set DATABASE_URL="postgresql://neondb_owner:xxx@xxx.neon.tech/neondb?sslmode=require"

# SHOPIFY_API_SECRET は fly launch 時に自動設定済み
# 確認: fly secrets list
```

### Step 5: fly.toml を確認・修正

```toml
# 不要な設定を削除:
# - [[mounts]] セクション（SQLite ボリューム）→ 削除
# - dbsetup.js 参照 → 削除

[processes]
  app = 'npm run docker-start'  # prisma generate && prisma migrate deploy && start
```

### Step 6: デプロイ

```bash
# アプリ本体をデプロイ
fly deploy

# 動作確認
curl -s -o /dev/null -w "%{http_code}" https://your-app.fly.dev/
# → 302（Shopify OAuth リダイレクト）なら成功

# Extensions をデプロイ
shopify app deploy
# → Network Access 承認が必要な場合、表示されるリンクから申請
```

### Step 7: shopify.app.toml の URL を確認

`fly launch` 実行時に `shopify.app.toml` が自動更新される:
- `application_url` → Fly.io の URL
- `auth.redirect_urls` → Fly.io の URL
- `app_proxy.url` → Fly.io の URL

手動で更新が必要な場合は `shopify.app.toml` を直接編集。

### よくあるハマりポイント

| 問題 | 原因 | 解決 |
|------|------|------|
| P3019 エラー | migration_lock.toml が sqlite のまま | `rm -rf prisma/migrations` → `prisma migrate dev` |
| fly deploy 後にクラッシュ | DATABASE_URL 未設定 or 間違い | `fly secrets set DATABASE_URL=...` |
| Extensions が未リリース | Network Access 未承認 | Partner Dashboard から申請 |
| fly launch でデプロイ失敗 | SQLite の [[mounts]] が残っている | fly.toml から削除 |
| アプリページが空白 | `application_url` のパスが不正 | パスを削除してベース URL のみにする |
| トンネル URL が残る | `shopify app dev` 停止後も URL が戻らない | ストアからアプリを再インストール |

---

## 2026-03-01 application_url の不正パスでアプリページが空白

- **症状**: Shopify 管理画面でアプリを開くと完全に空白。エラーメッセージは表示されない
- **原因**: `shopify.app.toml` の `application_url` が `https://xxx.fly.dev/apps/default-app-home` になっていたが、アプリに `/apps/default-app-home` というルートは存在しない（404 を返す）。`fly launch` が自動設定した `/apps/default-app-home` パスは React Router テンプレートのルート構成と一致しない
- **解決**:
  1. `shopify.app.toml` の `application_url` からパスを削除し、ベース URL のみにする
     - 変更前: `https://xxx.fly.dev/apps/default-app-home`
     - 変更後: `https://xxx.fly.dev`
  2. `shopify app deploy --force` で新バージョンをリリース
  3. 管理画面をハードリロード
- **教訓**:
  - `fly launch` が `application_url` に `/apps/default-app-home` を自動追加することがある
  - React Router テンプレートのルートは `/` → `/app` にリダイレクトする構成。`/apps/default-app-home` は存在しない
  - `application_url` はパスなしのベース URL（`https://xxx.fly.dev`）が正しい
  - アプリが空白の場合、`curl` で `application_url` のレスポンスコードを確認する（302 なら正常、404 なら URL が間違っている）

---

## 2026-03-01 shopify app dev 停止後にトンネル URL が残り続ける

- **症状**: `shopify app dev` を停止した後も、Shopify 管理画面でアプリを開くと旧トンネル URL（`xxx.trycloudflare.com`）に接続しようとして「接続が拒否されました」エラー
- **原因**: `shopify app dev` 実行中は `application_url` が Cloudflare トンネル URL に一時的に上書きされる。正常に停止（Ctrl+C）すれば元の URL に戻るが、ターミナルを直接閉じたり異常終了すると URL が戻らないことがある
- **解決**:
  1. `shopify app deploy --force` で再デプロイ → 効果なしの場合あり
  2. `shopify app dev` を再度起動して Ctrl+C で正常停止 → 効果なしの場合あり
  3. **最終手段**: ストアからアプリをアンインストール → 再インストール → URL がリセットされる
- **教訓**:
  - `shopify app dev` は必ず **Ctrl+C で正常停止** する（ターミナルを直接閉じない）
  - URL が戻らない場合、アプリの再インストールが最も確実な解決策
  - `shopify app deploy` だけでは `application_url` が更新されないケースがある

---

## 2026-03-01 Customer Account Extension から DELETE リクエストが送信されない

- **症状**: お気に入り一覧の「削除」ボタンを押しても商品が消えない。Fly.io ログに OPTIONS（204）のみが記録され、DELETE 本リクエストが一切来ない
- **原因**: Customer Account Extension から外部サーバーへの `fetch` で `method: "DELETE"` を使うと、CORS プリフライト（OPTIONS）は通るが、本リクエストが送信されない。Extension の実行環境（Remote DOM / Web Worker）が DELETE メソッドを制限している可能性がある
- **解決**: DELETE の代わりに POST + `{ action: "delete" }` を使う
  ```tsx
  // Extension 側
  fetch(url, {
    method: "POST",  // DELETE ではなく POST
    headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
    body: JSON.stringify({ action: "delete", productId: numericId }),
  });
  ```
  ```typescript
  // バックエンド側（action ハンドラ）
  const body = await request.json();
  const { action, productId, variantId } = body;
  if (action === "delete") {
    await removeFromWishlist(shop, customerId, productId, variantId);
    return cors(Response.json({ success: true }));
  }
  ```
- **教訓**:
  - Customer Account Extension から外部 API を呼ぶ場合、**GET と POST のみ使う**（DELETE / PUT / PATCH は避ける）
  - 操作の種類は HTTP メソッドではなく **リクエストボディの `action` フィールド** で区別する
  - ログで OPTIONS だけが記録され本リクエストが来ない場合、メソッド制限を疑う
