# サーバー（ログイン・API・設定）

## 機能説明: ログイン

- 全社共通の ID・パスワード1組でログインする。ログインしていないと、画面はログイン画面へ移動し、API は 401 を返す
- ログインに成功すると、署名付きの Cookie（JavaScript から読めない・HTTPS のみ・SameSite=Lax）を発行する。有効期限は `SESSION_DAYS`（既定 30日、1〜365日）
- Cookie の中身は「有効期限」と「ID・パスワードから作った目印」。パスワードを変えると目印が合わなくなり、全端末のログインが切れる
- ログイン失敗時は1秒待たせてから画面を戻す（総当たり対策）
- `/logout` を開くとログアウトする（画面にボタンはまだ無い）
- ログインしていないリクエストには、ストレージに触れずに小さな応答だけを返す（大量アクセスを受けても処理と通信量を小さく抑える）

ログイン不要なのは `/login`・`/logout`・`/api/admin/master`（トークンで認証）だけです。

## サーバー仕様（Hono）

サーバー本体（app.js）は Hono で作り、Cloudflare 固有のものを直接使いません。実行環境ごとの違いは、入口（index.js）が渡す3つの部品に閉じ込めています。

| 部品 | Cloudflare での中身 | 内容 |
| --- | --- | --- |
| `getStorage(c)` | storage/r2.js（R2 バインディング `MY_R2_BUCKET`） | `head`・`get`・`put`・`list`・`delete` の5つを持つストレージ部品 |
| `serveAsset(c)` | `env.ASSETS.fetch()` | 画面ファイル（HTML・JS・CSS・画像）を返す |
| `getEnv(c, name)` | `c.env[name]` | 設定値・シークレットを読む |

`c` は Hono がリクエストごとに作るコンテキストです。画面ファイルもログイン確認を通すため、wrangler.jsonc の assets で `run_worker_first: true` にしています（すべてのリクエストがサーバーを通る）。

| URL | ログイン | 内容 |
| --- | --- | --- |
| GET /login・POST /login・GET /logout | 不要 | ログイン画面・ログイン・ログアウト |
| PUT /api/admin/master | 不要（トークン） | 軽量化バッチからのマスタ受け取り |
| GET /api/master | 必要 | マスタ CSV をそのまま返す。ETag + `private, no-cache`、一致すれば 304 |
| GET /api/themes | 必要 | `themes/manifest.json` を JSON で返す（`private, no-cache`） |
| GET /api/theme-image/:filename | 必要 | `images/<filename>` の画像（`private, max-age=86400`） |
| 上記以外の GET | 必要 | 画面ファイル |

エラー時は理由を外に出さず「サーバーでエラーが発生しました」とだけ返し、詳細はログ（`console.error`）に出します。

### /api/master の処理

1. ストレージの `head` でマスタの etag だけを取得（本文は読まない）
2. ETag を `"v4-<etag>"` の形で作る（`v4` は app.js の `MASTER_FORMAT`）。`If-None-Match` と一致すれば 304（`W/` は無視して比較）
3. 一致しなければ、ストレージから本文を流すだけで返す（サーバーでは解析しない）。手順1と3の間に更新された場合に備え、ETag は実際に読んだ版のものを使う

以前のエッジキャッシュ（Cache API）と、サーバーでの文字コード変換・CSV 解析は、マスタを軽量化したことで不要になり廃止しました。キャッシュは IndexedDB＋ETag/304 の1層で、各端末が毎回マスタをダウンロードしないためのものです。

### PUT /api/admin/master の処理

1. `Authorization: Bearer <トークン>` を `UPLOAD_TOKEN` と比べる（ハッシュにしてから全バイトを比べ、処理時間で中身が推測されないようにする）。未設定なら 503、違えば 401
2. サイズ（上限 50MB）・UTF-8 か・1行目が見出し（`MASTER_HEADER`）と一致するか・2行目以降が I 行か P 行かを確認。問題があれば 400
3. 今のマスタを `masters/backup/pop-master-<日時>.csv` に退避し、新しい順に7件だけ残す
4. 新しいマスタを保存し、`{ ok: true, items, exceptions, bytes }` を返す

### /api/theme-image の安全対策

ファイル名に `/`・`\`・`..` が含まれる場合は 400 を返し、`images/` 以外のファイルを読めないようにしています。

## 設定値と開発環境

### サーバーの設定値

| 名前 | 種類 | 既定値 | 内容 |
| --- | --- | --- | --- |
| MY\_R2\_BUCKET | R2バインディング | なし（必須） | マスタ・テーマ・画像を置くバケット |
| ASSETS | 画面ファイルのバインディング | なし（必須） | wrangler.jsonc の assets（`run_worker_first: true`） |
| LOGIN\_USER | シークレット | なし（必須） | ログイン ID |
| LOGIN\_PASSWORD | シークレット | なし（必須） | パスワード。変えると全端末のログインが切れる |
| SESSION\_SECRET | シークレット | なし（必須） | ログイン Cookie の署名の鍵（長いランダムな文字列） |
| UPLOAD\_TOKEN | シークレット | なし | バッチのアップロード用トークン。未設定ならアップロードを受け付けない |
| SESSION\_DAYS | vars | 30 | ログインの有効日数（1〜365） |
| MASTER\_STORAGE\_KEY | vars | masters/pop-master.csv | マスタの置き場所 |

シークレットは `npx wrangler secret put <名前>` で登録します（値は登録後に誰も見られません）。LOGIN\_USER と LOGIN\_PASSWORD は、スタッフに伝える必要があるため社内のパスワード管理の決まりに沿って控えておきます。SESSION\_SECRET と UPLOAD\_TOKEN は忘れても作り直せば済みます。

ランダムな文字列は PowerShell の次の1行で作れます。

```powershell
$b = New-Object byte[] 32; [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); [Convert]::ToBase64String($b)
```

他のサービスに移るときは、同じ名前の設定値を移行先に登録し直し、入口（index.js 相当）で読み方を合わせます。

### 手元での開発（wrangler dev）

- 本番のシークレットは使われない。wrangler.jsonc と同じフォルダに `.dev.vars` を作り、`名前=値` で書く（LOGIN\_USER・LOGIN\_PASSWORD・SESSION\_SECRET は必須、UPLOAD\_TOKEN はアップロードを試すときだけ）。値は本番と違うテスト用にし、**.gitignore に入れる**
- ログイン Cookie は HTTPS 用の設定なので、確認は Chrome か Edge で行う（`http://localhost` を例外として扱うため）
- 既定では手元の空の R2 を使う。本番の R2 で確かめるときは `npx wrangler dev --remote`
- 普通の `wrangler dev` は Cloudflare の利用量に数えられない
