# Cumulocity 設定 エクスポート／投入 実装ガイド

作成日: 2026-08-06
対象: リファレンス環境の GUI で設定した設定値群を**取得（エクスポート）→ Git でテンプレート保持 → 新規 Edge へ投入（インポート）**する半自動パイプラインの実装
関連: [Cumulocity設定定義書.md](Cumulocity設定定義書.md) の §9.4「リファレンステナント差分方式」の実装編

---

## 0. 全体設計

### 0.1 パイプライン

```
┌─ リファレンス Edge ─┐      ┌── Git リポジトリ ──┐      ┌─ 新規 Edge ─┐
│  GUI で設定を投入   │ ──►  │ config/*.json      │ ──►  │  投入        │
│  (人手・1回きり)    │ 取得 │ (id/timestamp除去) │ 投入 │             │
└─────────────────────┘      │ secrets/ は別管理  │      └─────────────┘
                             └────────────────────┘
        c8y ... --select        git commit              c8y ... --template
        c8y api GET             (レビュー可能)           c8y api PUT/POST
        kubectl get -o yaml                              c8yedge config --set
```

### 0.2 ⚠️ 素朴な「GET して Git に置いて PUT」が必ず破綻する 3 点

この 3 つを設計に織り込まないと、パイプラインは静かに壊れます。

| # | 破綻ポイント | 内容 | 対策 |
|---|---|---|---|
| **B-1** | **暗号化オプション** | キーに `credentials.` 接頭辞を付けたテナントオプションは、オーナー（当該マイクロサービスの system user / bootstrap user）以外が GET すると固定文字列 `<<Encrypted>>` が返る。管理者セッションでのエクスポートは**必ず**このセンチネルを掴む | エクスポート時に `credentials.*` を除外し、投入時は CI のシークレットストアから別経路で PUT（§5） |
| **B-2** | **`--select` は空オブジェクトのフラグメントを黙って落とす** | 公式の `--select "**"` 例では `"c8y_IsDevice": {}` や `"company_Example": {}` といった**値が空オブジェクトのマーカーフラグメントが出力から消えている**。`--select` ベースのエクスポートはロッシー | マーカーフラグメントを持つオブジェクト（デバイス、スマートグループ等）は `--select` を使わず `--raw` で取得し、jq/jsonnet 側で除去（§2.2） |
| **B-3** | **継承値の実体化** | `GET /tenant/options/{category}` は Enterprise テナントでは management テナントから**継承した値も返す**。それをそのまま PUT すると、継承していただけの値が自テナントにハードコードされる | Edge ではサブテナント階層が無いため影響は小さいが、**management テナントと edge テナントで別々にエクスポートし、差分を取る**（§3.1） |

さらに `GET /tenant/options/{category}` ↔ `PUT /tenant/options/{category}` は**厳密な逆写像ではありません**。公式スキーマ説明は "an arbitrary number of **existing** options" と書いており、POST の説明にも *"Some categories of options allow the creation of new ones, while others are limited to predefined set of keys."* とあります。全カテゴリで任意キーを新規作成できるわけではない点に注意。

出典: [Options API (Core OAS)](https://cumulocity.com/api/core/dist/c8y-oas.yml), [select parameter](https://goc8ycli.netlify.app/docs/concepts/select-parameter/)

---

## 1. 前提セットアップ

### 1.1 セッション（management / edge の 2 本を用意）

Edge には `management` と `edge` の 2 テナントがあるため、**セッションを 2 つ作り `--session` で切り替えます。**

```bash
c8y sessions create --host "https://edge.example.com" --username "admin" --tenant "edge" --name "edge-ref"
```

```bash
c8y sessions create --host "https://management-edge.example.com" --username "admin" --tenant "management" --name "mgmt-ref"
```

コマンド単位の切替（セッション名は `~/.cumulocity/` 配下のファイル名）:

```bash
c8y tenantoptions list --session edge-ref
```

セッションの複製（新規 Edge 用に使い回す）:

```bash
c8y sessions clone --newName "edge-prod" --mode prod
```

> ⚠️ `c8y sessions clone --type` は **deprecated** です。`--mode` を使ってください。

### 1.2 CI/CD 非対話認証

環境変数からセッションを起動します（公式が *"the recommended approach for CI pipelines (GitHub Actions, GitLab CI, etc.)"* と位置づける方式）。

```bash
export C8Y_HOST="https://edge.example.com"
export C8Y_TENANT="edge"
export C8Y_USER="svc_bootstrap"
export C8Y_PASSWORD="********"
export C8Y_MODE="dev"
eval "$( c8y sessions login --from-env )"
```

PowerShell（**`eval "$(...)"` は使えません**）:

```powershell
$env:C8Y_HOST = "https://edge.example.com"; $env:C8Y_TENANT = "edge"; $env:C8Y_USER = "svc_bootstrap"; $env:C8Y_PASSWORD = "********"; $env:C8Y_MODE = "dev"; c8y sessions login --from-env | Out-String | Invoke-Expression
```

**認識される環境変数**:

| 変数 | 別名 | 用途 |
|---|---|---|
| `C8Y_HOST` | `C8Y_URL`, `C8Y_BASEURL` | 接続先 |
| `C8Y_TENANT` | — | テナント ID |
| `C8Y_USER` | `C8Y_USERNAME` | ユーザー名 |
| `C8Y_PASSWORD` | — | パスワード |
| `C8Y_TOKEN` | — | 事前取得トークン |
| `C8Y_SETTINGS_LOGIN_TYPE` | — | ログインタイプ上書き |
| `C8Y_CERTIFICATE` / `C8Y_CERTIFICATE_KEY` | — | mTLS |
| `C8Y_MODE` | — | セッションモード（下記） |
| `C8Y_SETTINGS_CI` | — | 全プロンプト無効（v2.18.0 以降は CI 環境を自動検出） |

### 1.3 ⚠️ セッションモード — これを設定しないと投入コマンドが全部無効

公式に *"all commands which create, update and/or delete data are disabled by default"* と明記されています。**投入自動化には必ずモード設定が必要です。**

| モード | 有効なコマンド |
|---|---|
| `ci` | 全コマンド有効 |
| `dev` | 全コマンド有効 |
| `qual` | Delete のみ無効 |
| `prod` | Create / Update / Delete すべて無効 |

有効化の経路は 5 つあります（`C8Y_MODE` は必須ではありません）:

1. `export C8Y_MODE=dev`
2. `c8y sessions set --mode dev`
3. `c8y settings update mode dev`
4. コマンド単位の `--sessionMode dev`
5. `C8Y_SETTINGS_MODE_ENABLE{CREATE,UPDATE,DELETE}`

> ⚠️ `session.mode=ci`（全コマンド有効）と `C8Y_SETTINGS_CI`（全プロンプト無効）は**別物**です。混同しないでください。

> ⚠️ **CI での落とし穴**: セッションファイルは既定で暗号化され、パスフレーズを対話要求します。ヘッドレス環境では使えないため、**環境変数直渡し（`--from-env`）か暗号化の無効化**が必要です。

出典: [Sessions](https://goc8ycli.netlify.app/docs/concepts/sessions/), [Settings](https://goc8ycli.netlify.app/docs/configuration/settings/)

### 1.4 拡張機能のインストール（Streaming Analytics 用）

Analytics Builder モデルと EPL apps を CLI で扱うには、公式組織の拡張が必要です。

```bash
c8y extension install Cumulocity-IoT/c8y-analytics
```

出典: [c8y-analytics](https://github.com/Cumulocity-IoT/c8y-analytics), [Extensions](https://goc8ycli.netlify.app/docs/concepts/extensions/)（extensions は v2.30.0 以降）

---

## 2. 共通イディオム（全設定項目で使う道具）

### 2.1 `--select` によるフィールド除外

`--select` はグローバルフラグで、**glob / globstar と `!` による除外**を受け付けます。

```bash
c8y devices list --select '**,!*parents.*,!child*.*'
```

再投入してはいけないフィールドを落とす基本形:

```bash
c8y retentionrules list --includeAll --select '**,!id,!self,!lastUpdated,!creationTime' -o json --outputFile config/retentionrules.json
```

> ⚠️ **`!` は必ずシングルクォートで囲む**こと。bash / zsh の history expansion に食われます（公式も caution で明記）。
> ⚠️ ドット記法パスは **case-insensitive** なので、`!id` は `ID` という名のフラグメントも落とします。
> ⚠️ **B-2**（空オブジェクトのフラグメントが消える）に該当する対象では `--select` を避けてください。

### 2.2 `--select` を避けるべきときの代替

マーカーフラグメント（`c8y_IsDevice: {}` 等）を保持したい場合は生レスポンスを取り、除去は後段で行います。

```bash
c8y inventory find --query "has(c8y_Dashboard)" --includeAll --raw -o json --outputFileRaw config/dashboards.raw.json
```

### 2.3 `--template` / `input.value` — 投入側の要

**ここが最重要**: go-c8y-cli のパイプライン入力は、コマンドごとに**ヘルプで `(accepts pipeline)` と記された 1 つのパラメータにしか束縛されません**（`--id` / `--name` / `--key` など）。**リクエストボディ全体には束縛されません**（`--data` にマーカーは付いていない）。

したがって素朴な `list | create` では全フィールドが運ばれません。**忠実な export→import には `--template "input.value"` が必須です。**

```bash
cat config/retentionrules.json | c8y retentionrules create --template "input.value"
```

テンプレート変数を使う場合:

```bash
c8y inventory create --template "./mo.jsonnet" --templateVars "type=custom_Type,env=prod"
```

組み込み関数（`_.Now()`, `_.Int()`, `_.Name()`, `_.Password()` 等）も使えます。

### 2.4 `--outputTemplate` — 取得側の整形（v2.32.0 以降）

`--template` と同じ jsonnet エンジンで、**出力を Git 保管用の正規化された形状に変換**できます。

```bash
c8y tenantoptions getForCategory --category configuration --outputTemplate "{category: 'configuration', options: output}"
```

参照できる変数: `input.value`（現在のパイプラインアイテム）、`input.index`（1 始まり）、`output`、`request`、`response`、`flags`

> ⚠️ `--outputTemplate` はレスポンスデータ上でのみ動作します。**暗号化された `credentials.*` の復元はできません**（B-1 の解決にはならない）。

### 2.5 `--dry` / `--dryFormat` — 投入前検証と curl 生成

```bash
cat config/retentionrules.json | c8y retentionrules create --template "input.value" --dry --dryFormat json
```

curl 相当コマンドを生成（CLI が入らない環境向けの手順書生成に有用）:

```bash
c8y tenantoptions updateBulk --category configuration --data "@config/tenantoptions.configuration.json" --dry --dryFormat curl
```

出力形式: `json` / `dump` / `markdown`（既定） / `curl`

> ⚠️ `--dry` は**リクエスト構築（URL / メソッド / ヘッダ / ボディ）の検証のみ**です。`kubectl --dry-run=server` のようなサーバ側バリデーションは行われません。
> ⚠️ **dry 出力には `Authorization: Basic` ヘッダが含まれ得ます。** Git にコミットする運用では要注意。
> ⚠️ `curl` 形式は beta。multipart/form-data（アプリ／マイクロサービスのバイナリ投入）では *"might not be 100% correct"* と公式注意あり。
> ⚠️ チェインの場合、`--dry` を付けたコマンドだけが dry になります。**上流の `list` / `find` は実際に GET を発行します。**

### 2.6 `c8y api` — 専用サブコマンドが無いエンドポイント

```bash
c8y api GET /tenant/loginOptions
```

```bash
c8y api POST /tenant/loginOptions --template "input.value"
```

URL のパイプライン置換（`%s` が入力行に置換される）:

```bash
echo "12345" | c8y api PUT "/service/example/%s" --template "{id: input.value}"
```

主なフラグ: `--method`（既定 GET） / `--url`（`%s` 置換、パイプライン受付） / `-d,--data`（JSON またはショートハンド `a.b.c=1`） / `--file`（バイナリ） / `--formdata` / `--template` / `--templateVars` / `--contentType` / `--accept` / `--host` / `-H,--header` / `--keepProperties`（**既定 true**）

> `--keepProperties` は既定 true なので明示は冗長です。剥がしたいときだけ `--keepProperties=false`。これは `api` サブコマンド固有でグローバルフラグではありません。

### 2.7 その他の実用グローバルフラグ

| フラグ | 用途 |
|---|---|
| `--includeAll` | 全ページ取得（pageSize を最大 2000 に強制）。**全件をメモリに載せる**ので超大規模では非推奨 |
| `-f, --force` | 確認プロンプト抑止（`--confirm` 併用時は無視） |
| `-o json` / `-o csv` | 出力形式 |
| `--outputFile` / `--outputFileRaw` | ファイル保存（前者は select/view 適用後、後者は生レスポンス） |
| `--silentStatusCodes 409` | 特定ステータスをエラー表示しない（べき等投入で有用） |
| `--silentExit` | サイレントステータスを終了コードに反映しない |
| `--allowEmptyPipe` | 空入力で失敗しない（CI で有用） |
| `-n, --nullInput` | stdin を読まない（シェルのループ内で使うとき必須） |
| `--abortOnErrors N` | バッチ中断閾値（既定 10） |
| `--workers N` / `--delay` | 並列度とレート制御 |
| `--filter` | クライアント側フィルタ |

出典: [c8y root command](https://goc8ycli.netlify.app/docs/cli/c8y/c8y/), [Chaining commands](https://goc8ycli.netlify.app/docs/concepts/chaining-commands/), [Templates](https://goc8ycli.netlify.app/docs/concepts/templates/), [Output templates](https://goc8ycli.netlify.app/docs/concepts/output-templates/), [Dry run](https://goc8ycli.netlify.app/docs/concepts/dryrun/), [c8y api](https://goc8ycli.netlify.app/docs/cli/c8y/api/c8y_api/)

---

## 3. 設定項目別 取得／投入 一覧表

### 凡例

- **べき等性**: **◎** = 何度実行しても同じ結果 / **○** = 存在チェック併用で実現可 / **△** = 重複エラーの捕捉が必要 / **×** = 手動

### 3.1 テナント基本設定

| 設定項目 | 取得コマンド | 保持する JSON | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| テナントオプション（カテゴリ単位） | `c8y tenantoptions getForCategory --category <cat>` | key-value マップ | `credentials.*` | `c8y tenantoptions updateBulk --category <cat> --data @file` | ◎ | **B-1 / B-3 に該当** |
| テナントオプション（全件） | `c8y tenantoptions list --includeAll` | `{category,key,value}[]` | `credentials.*`, `self` | `c8y tenantoptions create` / `update` | ◎ | 全件からカテゴリ別に分解 |
| システムオプション | `c8y systemoptions list` | 参照用 | — | **投入不可（読み取り専用）** | — | **投入後の検証（アサート）に使う** |
| フィーチャートグル | `c8y features list --includeAll` | `{key,active}[]` | — | `c8y features enable --key <k>` / `disable` | ◎ | |

#### レシピ

```bash
# 取得（カテゴリ単位、credentials.* を除外）
c8y tenantoptions getForCategory --category configuration --session edge-ref -o json --select '**,!credentials*' --outputFile config/tenantoptions.configuration.json
```

```bash
# 取得（全カテゴリを一覧してループ）
c8y tenantoptions list --includeAll --session edge-ref -o json --select 'category' | sort -u | c8y tenantoptions getForCategory
```

```bash
# 投入
c8y tenantoptions updateBulk --category configuration --data "$(cat config/tenantoptions.configuration.json)" --session edge-new --force
```

```bash
# 投入後アサート（差分が無いことを確認）
diff <(c8y tenantoptions getForCategory --category configuration --session edge-new -o json --select '**,!credentials*') config/tenantoptions.configuration.json
```

> ⚠️ `PUT /tenant/options/{category}` は `ROLE_OPTION_MANAGEMENT_ADMIN` が必要。management テナントが「非編集」に設定したオプションは **403** で落ちます。ボディ不正時は **422**。

> **管轄の切り分け**: メールサーバー設定・パスワードリセットテンプレートは **management テナント**側です。`--session mgmt-ref` / `--session mgmt-new` で取得・投入してください。

### 3.2 認証・SSO

| 設定項目 | 取得コマンド | 保持する JSON | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| ログインオプション全件 | `c8y api GET /tenant/loginOptions` | `authConfig[]` | `id`, `self` | `c8y api POST /tenant/loginOptions --template "input.value"` | **△** | 専用サブコマンド無し |
| ログインオプション個別更新 | `c8y api GET /tenant/loginOptions/<typeOrId>` | `authConfig` | `id`, `self` | `c8y api PUT /tenant/loginOptions/<typeOrId> --template "input.value"` | ◎ | **再投入はこちら** |
| SSO アクセスマッピング | `c8y api GET /tenant/loginOptions/<id>/accessMappings` | 配列 | `id` | `c8y api POST /tenant/loginOptions/<id>/accessMappings` | △ | |
| SSO インベントリマッピング | `c8y api GET /tenant/loginOptions/<id>/inventoryAccessMappings` | 配列 | `id` | `c8y api POST .../inventoryAccessMappings` | △ | |
| TFA 設定 | `c8y api GET /tenant/tenants/<tid>/tfa` | オブジェクト | — | `c8y api PUT /tenant/tenants/<tid>/tfa --data ...` | ◎ | |

#### レシピ

```bash
# 取得（id を除外）
c8y api GET /tenant/loginOptions --session edge-ref -o json --select '**,!id,!self' --outputFile config/loginoptions.json
```

```bash
# 投入（更新優先 = べき等）
c8y api PUT "/tenant/loginOptions/OAUTH2" --template "input.value" --session edge-new < config/loginoptions.oauth2.json
```

> ⚠️ **`POST /tenant/loginOptions` は非冪等です。** 重複すると **400 "Duplicated – The login option already exists."** を返します。
> ⚠️ POST のリクエストボディスキーマでは `allOf` により **`id` が `readOnly` に上書き**されています。エクスポート JSON から `id` を必ず除去してください。
> ⚠️ 必要ロールは `ROLE_TENANT_ADMIN` **OR** `ROLE_TENANT_MANAGEMENT_ADMIN`（AND ではない）。
> メディアタイプは `application/vnd.com.nsn.cumulocity.authconfig+json`。

**べき等化パターン**（PUT 先行、404 なら POST）:

```bash
c8y api PUT "/tenant/loginOptions/OAUTH2" --template "input.value" --silentStatusCodes 404 --silentExit < config/loginoptions.oauth2.json || c8y api POST /tenant/loginOptions --template "input.value" < config/loginoptions.oauth2.json
```

### 3.3 ユーザー・ロール・権限

| 設定項目 | 取得コマンド | 保持する JSON | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| グローバルロール（グループ）定義 | `c8y usergroups list --includeAll` | `{name,description,...}[]` | `id`, `self`, `roles`, `users` | `c8y usergroups create --template "input.value"` | ○ | `getByName` で存在確認 |
| グループへのロール割当 | `c8y userroles getRoleReferenceCollectionFromGroup --group <g>` | ロール ID 配列 | `self` | `c8y userroles addRoleToGroup --group <g> --role <r>` | ◎ | **`--role "*ALARM*"` のワイルドカード指定可** |
| ロール一覧（参照用） | `c8y userroles list --includeAll` | 参照用 | — | 投入不可（プラットフォーム定義） | — | |
| ユーザー | `c8y users list --includeAll` | `{userName,email,...}[]` | `id`, `self`, `password`, `lastPasswordChange` | `c8y users create --template "input.value"` | ○ | `getUserByName` で存在確認 |
| ユーザーのグループ所属 | `c8y userreferences listGroupMembership --user <u>` | 配列 | `self` | `c8y userreferences addUserToGroup --group <g> --user <u>` | ◎ | **パイプライン対応** |
| ユーザーのロール割当 | `c8y userroles getRoleReferenceCollectionFromUser --user <u>` | 配列 | `self` | `c8y userroles addRoleToUser --user <u> --role <r>` | ◎ | |
| **インベントリロール定義** | `c8y api GET /user/inventoryroles` | `{name,description,permissions[]}[]` | `id`, `self` | `c8y api POST /user/inventoryroles --template "input.value"` | △ | **⚠️ 専用サブコマンド無し** |
| ユーザーのインベントリロール割当 | `c8y users listInventoryRoles --id <u>` | 配列 | `id`, `self` | `c8y api POST /user/<tid>/users/<uid>/roles/inventory` | △ | |

#### レシピ

```bash
# グローバルロール定義の取得
c8y usergroups list --includeAll --session edge-ref -o json --select '**,!id,!self,!roles,!users' --outputFile config/usergroups.json
```

```bash
# グローバルロール定義の投入
cat config/usergroups.json | c8y usergroups create --template "input.value" --session edge-new --force --silentStatusCodes 409
```

```bash
# ロール割当（グループ名とロール名のワイルドカードで指定できるのでID解決が不要）
c8y userroles addRoleToGroup --group "業務管理者" --role "*ALARM*" --session edge-new --force
```

```bash
# ユーザーを複数グループへ一括所属（公式のパイプライン例）
c8y users list --session edge-new | c8y userreferences addUserToGroup --group business | c8y userreferences addUserToGroup --group admins
```

> ⚠️ **`c8y` に `inventoryroles` 名前空間は存在しません。** インベントリロール**定義**の CRUD は `c8y api` で `/user/inventoryroles` を直接叩いてください。`c8y users listInventoryRoles` / `getInventoryRole` はユーザーへの**割当**の参照のみです。

> 💡 `--group` と `--role` が**名前・ワイルドカードを受け付ける**ため、ID を解決せずにスクリプトを書けます。これがべき等性を大幅に楽にします。

### 3.4 アプリケーション・マイクロサービス

| 設定項目 | 取得コマンド | 保持する JSON | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| アプリ一覧 | `c8y applications list --includeAll` | `{name,key,type,...}[]` | `id`, `self`, `owner` | `c8y applications create --template "input.value"` | ○ | |
| アプリバイナリ | `c8y applications listApplicationBinaries --id <a>` | — | — | `c8y applications createBinary --id <a> --file <zip>` | × | **ZIP は Git LFS か成果物リポジトリで管理** |
| ホステッドアプリ | — | — | — | `c8y applications createHostedApplication --file <zip>` | × | |
| テナントのアプリ購読 | `c8y api GET /tenant/tenants/<tid>/applications` | 配列 | `id`, `self` | `c8y api POST /tenant/tenants/<tid>/applications --data "application.id=<id>"` | △ | |
| マイクロサービス一覧 | `c8y microservices list --includeAll` | 同上 | `id`, `self`, `owner` | `c8y microservices create` | ○ | |
| マイクロサービス有効化 | `c8y microservices getStatus --id <m>` | — | — | `c8y microservices enable --id <m>` / `disable` | ◎ | |
| マイクロサービス設定 | `c8y tenantoptions getForCategory --category <settingsCategory>` | key-value | `credentials.*` | `c8y tenantoptions updateBulk --category <settingsCategory>` | ◎ | カテゴリ名は manifest の `settingsCategory` → context path → 名前の順で決まる |

### 3.5 データ保持

| 設定項目 | 取得コマンド | 保持する JSON | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| リテンションルール | `c8y retentionrules list --includeAll` | `{dataType,fragmentType,type,source,maximumAge,editable}[]` | `id`, `self` | `c8y retentionrules create --template "input.value"` | △ | 重複作成される可能性あり |

#### レシピ

```bash
# 取得
c8y retentionrules list --includeAll --session edge-ref -o json --select '**,!id,!self' --outputFile config/retentionrules.json
```

```bash
# 投入（既存を全削除してから作り直す = 疑似べき等）
c8y retentionrules list --includeAll --session edge-new --select id | c8y retentionrules delete --force
cat config/retentionrules.json | c8y retentionrules create --template "input.value" --session edge-new --force
```

```bash
# 単発作成（フラグ指定版）
c8y retentionrules create --dataType ALARM --maximumAge 180 --session edge-new --force
```

`--dataType` の値域: `ALARM` / `AUDIT` / `EVENT` / `MEASUREMENT` / `OPERATION` / `*`（`BULK_OPERATION` は API スキーマにはあるが CLI フラグのヘルプには非掲載）

> ⚠️ **既定で全履歴データが 60 日後に削除**されます。新規 Edge でこの既定ルールが先に存在するため、「全削除 → 再作成」方式が最も確実です。
> ⚠️ `editable: false` のルールは management テナントからしか変更できません。

### 3.6 ルール・分析モデル

| 設定項目 | 取得コマンド | 保持する JSON | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| **EPL apps** | `c8y analytics epl list --includeAll` | `{name,contents,state}[]` | `id` | `c8y analytics epl create --name <n> --template <f>` | ○ | 拡張が必要 |
| EPL apps（拡張なし） | `c8y api GET "/service/cep/eplfiles?contents=true"` | 同上 | `id` | `c8y api POST /service/cep/eplfiles --template "input.value"` | △ | |
| **Analytics Builder モデル** | `c8y analytics ab get --id <id> --outputFileRaw <f>` | モデル JSON | `id` | `c8y analytics ab create --name <n> --template <f>` | ○ | 拡張が必要 |
| Analytics Builder 状態 | `c8y analytics ab list` | — | — | `c8y analytics ab update --id <id> --state ACTIVE` | ◎ | |
| Analytics Builder インスタンス | `c8y analytics instances list --id <id>` | — | — | `c8y analytics instances update --id <id> --instanceId <i> --mode PRODUCTION` | ◎ | |
| Analytics 拡張機能 | `c8y analytics extensions list` / `download` | — | — | `c8y analytics extensions upload` | ○ | |
| Apama 設定 | — | — | — | `c8y analytics configuration update --key <k> --value <v>` | ◎ | |
| Apama 再起動 | — | — | — | `c8y analytics management restart` | — | 投入後に必要な場合あり |
| **スマートルール** | `c8y inventory find --query "..."` | managed object | `id`, `self`, `lastUpdated`, `creationTime`, `cepModuleId` | `c8y inventory create --template "input.value"` | △ | **⚠️ フラグメント名 要検証（§7）** |
| スマートグループ | `c8y smartgroups list --includeAll` | 同上 | `id`, `self` | `c8y smartgroups create --template "input.value"` | ○ | 専用サブコマンドあり |

#### レシピ

```bash
# EPL apps を丸ごとエクスポート（ソースコード込み）
c8y api GET "/service/cep/eplfiles?contents=true" --session edge-ref -o json --outputFile config/eplfiles.json
```

```bash
# Analytics Builder モデルを個別ダウンロード
c8y analytics ab list --session edge-ref --select id | while read -r id; do c8y analytics ab get --id "$id" --outputFileRaw "config/ab/$id.json" --session edge-ref; done
```

```bash
# Analytics Builder モデルを投入
for f in config/ab/*.json; do c8y analytics ab create --name "$(basename "$f" .json)" --template "$f" --session edge-new --force; done
```

> ⚠️ スマートルールは **Smartrule マイクロサービス**と **Apama-ctrl マイクロサービス**への購読が前提です。投入前に §3.4 でマイクロサービスを有効化してください。
> ⚠️ スマートルールの managed object には `cepModuleId`（Apama シナリオインスタンス ID）が含まれます。**これは環境固有なので必ず除外**してください。

### 3.7 UI・ダッシュボード

| 設定項目 | 取得コマンド | 保持する JSON | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| **Cockpit ダッシュボード** | **GUI: ダッシュボード設定 → Import/export タブ** | ダッシュボード JSON | （GUI が処理） | **GUI: 同タブからインポート** | × | **⚠️ Edge 対応 要検証（§7）** |
| ダッシュボード（API 経由） | `c8y inventory find --query "has(c8y_Dashboard)" --includeAll --raw` | managed object | `id`, `self`, `lastUpdated`, `owner` | `c8y inventory create --template "input.value"` | △ | 画像バイナリ参照が dangling になる |
| UI アプリケーション | `c8y ui applications list` | — | `id` | `c8y ui applications ...` | ○ | |
| UI プラグイン | `c8y ui plugins list` | — | `id` | `c8y ui plugins ...` | ○ | |
| ブランディング | `c8y tenantoptions getForCategory --category <要検証>` | key-value | — | `c8y tenantoptions updateBulk` | ◎ | **⚠️ カテゴリ名 要検証（§7）** |

> ⚠️ **Cockpit の Import/export は 2026-02-19 に GA 化**（build 2025.373.0）しましたが、**GA 展開先として列挙されているのはパブリッククラウド（eu/apj/jp/us.latest.cumulocity.com）のみで、Cumulocity IoT Edge への言及がありません。** 対象 Edge 実機で Import/export タブの有無を必ず確認してください。
> ⚠️ エクスポート JSON は**素の managed object ではありません**。ウィジェットのデバイス候補提示やアップロード画像の扱いを支援する付加データを含みます。生 MO をコピーする方式とは等価ではなく、**付加データの正確なスキーマは非公開**です。
> ⚠️ ウィジェット画像は binaries API 上の別オブジェクトを ID 参照するため、**生 MO コピーでは dangling 参照**になります。

### 3.8 デバイス関連

| 設定項目 | 取得コマンド | 保持する JSON | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| デバイスプロトコル | **GUI: Device types → Device protocols → エクスポート** | ファイル | — | **GUI: インポート** | × | GUI が公式手段 |
| デバイスプロファイル | `c8y deviceprofiles list --includeAll` | 同上 | `id`, `self` | `c8y deviceprofiles create --template "input.value"` | ○ | |
| ファームウェア定義 | `c8y firmware list --includeAll` | 同上 | `id`, `self` | `c8y firmware create` | ○ | |
| ソフトウェア定義 | `c8y software list --includeAll` | 同上 | `id`, `self` | `c8y software create` | ○ | |

### 3.9 Edge インフラ層

| 設定項目 | 取得コマンド | 保持する形式 | 除外フィールド | 投入コマンド | べき等性 | 備考 |
|---|---|---|---|---|---|---|
| **Edge CR 全体（K8s 版）** | `kubectl get --namespace=c8yedge edge/c8yedge -o yaml > edge.yaml` | YAML | `status`, `metadata.uid`, `metadata.resourceVersion`, `metadata.creationTimestamp` | `kubectl apply -f edge.yaml` | ◎ | **宣言的・最も IaC 的** |
| ドメイン / ライセンス / TLS | **⚠️ 取得系サブコマンド無し** | — | — | `c8yedge config --set domain=<d> --set-file licenseKey=<f> --set-file tlsSecret.tls.crt=<f> --set-file tlsSecret.tls.key=<f>` | ◎ | ドメインとライセンスは**必ず同時に**変更 |
| OS/K8s セキュリティ（VM 版） | `GET /edge/configuration/security` または **GUI「Download configuration」** | JSON | — | `POST /edge/configuration/security` | ◎ | **公式に GET→POST 往復が保証された唯一の箇所** |
| ネットワーク等（VM 版） | `GET /edge/configuration/{network,hostname,domain,time-sync,certificate}` | JSON | — | 同 POST | ◎ | VM アプライアンス版のみ |
| マイクロサービスホスティング | `GET /edge/configuration/microservices` | JSON | — | `POST /edge/configuration/microservices` `{"enabled":true}` | ◎ | 有効化に 10〜15 分 |
| リモート接続 | `GET /edge/configuration/remote-connectivity` | JSON | — | `POST /edge/configuration/remote-connectivity` | ◎ | |

#### レシピ

```bash
# Edge CR のエクスポート（Kubernetes 版）
kubectl get --namespace=c8yedge edge/c8yedge -o yaml > config/edge/edge.yaml
```

```bash
# Edge CR の投入
kubectl apply -f config/edge/edge.yaml
```

```bash
# Edge セキュリティ設定（VM 版）— GUI の Download configuration で得た JSON をそのまま投入
c8y api POST /edge/configuration/security --template "input.value" --session edge-new < config/edge/security.json
```

> ⚠️ **`c8yedge` に設定の読み取りサブコマンドは文書化されていません。** ドキュメント化されているのは `c8yedge config`（`--set` / `--set-file`）、`c8yedge upgrade`、`c8yedge uninstall`、`c8yedge install`、`c8yedge package` のみです。**現行設定の取得は `kubectl get ... -o yaml` で行ってください。**
> ⚠️ Edge の VM アプライアンス版（〜Release 2024 / 10.18.0）と Kubernetes 版（2025/2026）で手段が根本的に異なります。詳細は [Cumulocity設定定義書.md](Cumulocity設定定義書.md) §0.2。

出典: [Manage Edge (2026)](https://cumulocity.com/docs/2026/edge/manage-edge/), [Manage Edge (2025, K8s)](https://cumulocity.com/docs/2025/edge-kubernetes/manage-edge/), [Edge configuration (2024)](https://cumulocity.com/docs/2024/edge/edge-configuration/)

---

## 4. べき等性の実装パターン

go-c8y-cli に **upsert 相当の公式コマンドはありません。** 以下のいずれかを実装側で選びます。

### パターン A: 名前ベース参照（最も簡単・推奨）

一部のコマンドは **ID ではなく名前やワイルドカードを受け付ける**ため、そもそも ID 解決が不要です。

```bash
c8y userroles addRoleToGroup --group "admins" --role "*ALARM*" --force
```

該当: `c8y userroles addRoleToGroup/addRoleToUser`（`--group` / `--role`）、`c8y userreferences addUserToGroup`（`--group`）

### パターン B: PUT 先行 → 失敗したら POST

更新エンドポイントが存在するリソース（loginOptions、tenant options）向け。

```bash
c8y api PUT "/tenant/loginOptions/OAUTH2" --template "input.value" --silentStatusCodes 404 --silentExit < config/loginoptions.oauth2.json || c8y api POST /tenant/loginOptions --template "input.value" < config/loginoptions.oauth2.json
```

### パターン C: 存在チェック → 分岐

`getByName` 系がある場合。

```bash
c8y usergroups getByName --name "業務管理者" --silentStatusCodes 404 --silentExit >/dev/null 2>&1 && echo "skip" || c8y usergroups create --name "業務管理者" --force
```

### パターン D: 全削除 → 再作成（宣言的）

リテンションルールのように「あるべき集合」を定義したい場合。**新規 Edge の初期セットアップでは最も確実です。**

```bash
c8y retentionrules list --includeAll --select id | c8y retentionrules delete --force
cat config/retentionrules.json | c8y retentionrules create --template "input.value" --force
```

### パターン E: 重複エラーを黙殺

```bash
cat config/usergroups.json | c8y usergroups create --template "input.value" --force --silentStatusCodes 409 --silentExit
```

> 💡 **投入スクリプトは必ず `--dry --dryFormat json` で先に流し、レビューしてから本実行**してください。`--dry` はリクエスト構築の検証のみでサーバ側検証はされない点に留意。

---

## 5. `credentials.*`（シークレット）の扱い

### 5.1 なぜ特別扱いが必要か

公式 OpenAPI の `POST /tenant/options` 記述より:

> *"Adding a `credentials.` prefix to the key causes the value of the option to be stored in an encrypted form. When the option is retrieved from a microservice, the `credentials.` prefix is removed, and the value is decrypted **only if the microservice is the owner** of the option."*

オーナーでない場合の戻り値:

```json
{ "category": "microservice2", "key": "credentials.secret", "value": "<<Encrypted>>" }
```

管理者セッションでのエクスポートは**常にこのセンチネルを取得**します。そのまま PUT すると、文字列 `<<Encrypted>>` が新しいシークレットとして保存されます。

> ※ 「`<<Encrypted>>` を書き戻すとその文字列が保存される」ことを明示した公式記述はありません。書き込み側のガードが仕様に存在しないことからの演繹です（反証も見つかりませんでした）。**いずれにせよ安全側に倒し、除外してください。**

さらに Enterprise ドキュメントは逐語で警告しています:

> *"You should not specify or overwrite tenant options here with a 'credentials.' prefix, since the platform expects those options to be encrypted"*（テナントポリシー経由のテナントオプションは暗号化されない、とも明記）

### 5.2 実装パターン

**エクスポート側 — 必ず除外**:

```bash
c8y tenantoptions getForCategory --category myservice --select '**,!credentials*' -o json --outputFile config/tenantoptions.myservice.json
```

**Git 側 — プレースホルダのみ管理**:

```
config/
  tenantoptions.myservice.json        # 非機密のみ
  secrets/
    tenantoptions.myservice.keys      # キー名の一覧だけ（値は入れない）
```

**投入側 — CI シークレットストアから別途注入**:

```bash
c8y tenantoptions update --category myservice --key "credentials.apiKey" --value "$MYSERVICE_API_KEY" --force
```

複数キーをまとめて:

```bash
c8y tenantoptions updateBulk --category myservice --data "{\"credentials.apiKey\":\"$MYSERVICE_API_KEY\",\"credentials.dbPassword\":\"$MYSERVICE_DB_PW\"}" --force
```

> ⚠️ `--outputTemplate` や jsonnet では復号できません。**シークレットの供給元は必ず CLI の外側**（GitHub Actions Secrets、Vault 等）にしてください。
> ⚠️ **`--dry` の出力には `Authorization: Basic` ヘッダが含まれ得ます。** dry 出力をログやアーティファクトに残す運用では要注意。

出典: [Options API (Core OAS)](https://cumulocity.com/api/core/dist/c8y-oas.yml), [Microservice SDK general aspects](https://cumulocity.com/docs/microservice-sdk/general-aspects/), [Managing tenants](https://cumulocity.com/docs/enterprise-tenant/managing-tenants/)

---

## 6. リポジトリ構成とスクリプト骨子

### 6.1 ディレクトリ構成案

```
config/
  edge/
    edge.yaml                          # kubectl get edge -o yaml
    security.json                      # GUI Download configuration
  mgmt/                                # management テナント管轄
    tenantoptions.*.json
  tenant/                              # edge テナント管轄
    tenantoptions.configuration.json
    tenantoptions.access.control.json
    loginoptions.oauth2.json
    usergroups.json
    users.json
    inventoryroles.json
    retentionrules.json
    features.json
  analytics/
    ab/*.json                          # Analytics Builder モデル
    epl/*.mon                          # EPL apps
  binaries/                            # ZIP は Git LFS
scripts/
  export.sh
  import.sh
  assert.sh
```

### 6.2 エクスポートスクリプト骨子

```bash
#!/usr/bin/env bash
set -euo pipefail
S="${1:?session name required}"
mkdir -p config/tenant config/analytics/ab

# テナントオプション（カテゴリごと）
c8y tenantoptions list --includeAll --session "$S" -o json --select category \
  | sort -u \
  | while read -r cat; do
      c8y tenantoptions getForCategory --category "$cat" --session "$S" \
        -o json --select '**,!credentials*' \
        --outputFile "config/tenant/tenantoptions.${cat}.json"
    done

# 認証・SSO
c8y api GET /tenant/loginOptions --session "$S" -o json \
  --select '**,!id,!self' --outputFile config/tenant/loginoptions.json

# ロール・ユーザー
c8y usergroups list --includeAll --session "$S" -o json \
  --select '**,!id,!self,!roles,!users' --outputFile config/tenant/usergroups.json
c8y api GET /user/inventoryroles --session "$S" -o json \
  --select '**,!id,!self' --outputFile config/tenant/inventoryroles.json

# リテンション
c8y retentionrules list --includeAll --session "$S" -o json \
  --select '**,!id,!self' --outputFile config/tenant/retentionrules.json

# EPL apps
c8y api GET "/service/cep/eplfiles?contents=true" --session "$S" \
  -o json --outputFile config/analytics/eplfiles.json
```

### 6.3 投入スクリプト骨子

```bash
#!/usr/bin/env bash
set -euo pipefail
S="${1:?session name required}"
DRY="${DRY:-}"   # DRY="--dry --dryFormat json" で試走

for f in config/tenant/tenantoptions.*.json; do
  cat="${f#config/tenant/tenantoptions.}"; cat="${cat%.json}"
  c8y tenantoptions updateBulk --category "$cat" --data "$(cat "$f")" \
    --session "$S" --force $DRY
done

c8y api PUT "/tenant/loginOptions/OAUTH2" --template "input.value" \
  --session "$S" $DRY < config/tenant/loginoptions.oauth2.json

cat config/tenant/usergroups.json \
  | c8y usergroups create --template "input.value" \
      --session "$S" --force --silentStatusCodes 409 --silentExit $DRY

c8y retentionrules list --includeAll --session "$S" --select id \
  | c8y retentionrules delete --session "$S" --force $DRY
cat config/tenant/retentionrules.json \
  | c8y retentionrules create --template "input.value" --session "$S" --force $DRY
```

### 6.4 投入後アサート

```bash
#!/usr/bin/env bash
set -euo pipefail
S="${1:?session name required}"

# システムオプションで方針が期待どおりか検証（読み取り専用の事実確認）
c8y systemoptions list --session "$S" -o json

# 設定の往復差分
for f in config/tenant/tenantoptions.*.json; do
  cat="${f#config/tenant/tenantoptions.}"; cat="${cat%.json}"
  diff <(c8y tenantoptions getForCategory --category "$cat" --session "$S" \
           -o json --select '**,!credentials*') "$f" \
    || echo "DRIFT: $cat"
done
```

> ⚠️ **PowerShell 環境の注意**: 上記は POSIX シェル前提です。`eval "$(...)"`、シングルクォートによる `!` の保護、`echo -e` はいずれも PowerShell では動作しません。Windows で回す場合は §1.2 の PowerShell 版を参照し、`--outputFile` ベースに書き換えてください。

---

## 7. ⚠️ 未確定事項（実機検証が必要）

本ガイドで**裏付けが取れなかった**項目です。実装前に必ず実機で確認してください。

| # | 項目 | 状況 | 確認方法 |
|---|---|---|---|
| U-01 | **スマートルールの managed object の `type` / `fragmentType` 値** | Java SDK の `SmartRuleRepresentation` はフラグメント `c8y.SmartRuleRepresentation` を表すとあるが、**`type` フィールドの値は文書化されていない** | GUI でルールを 1 つ作成 → `c8y inventory find --query "creationTime.date gt '<直前時刻>'" --raw` で実体を観測 |
| U-02 | **Analytics Builder モデルの managed object 型名** | 公式・コミュニティとも未記載。`c8y analytics ab` 拡張を使えば型名を知らなくても済む | 同上、または `c8y analytics ab get --id <id> --outputFileRaw -` |
| U-03 | **Cockpit ダッシュボードの Import/export が Edge で使えるか** | GA 展開先はパブリッククラウドのみ記載。Edge への言及なし | 対象 Edge の Cockpit でダッシュボード設定に Import/export タブが出るか確認 |
| U-04 | **ダッシュボードエクスポート JSON の正確なスキーマ** | 「付加データを含む」とあるが**スキーマは非公開** | 1 枚エクスポートして中身を実測 |
| U-05 | **ブランディングの tenant option カテゴリ／キー名** | 「tenant options として保存される」との記述はあるがキー名は非公開 | ベースライン取得 → GUI で設定 → 再取得 → 差分 |
| U-06 | **`c8yedge` の設定読み取りサブコマンド** | 文書化されていない | 実機で `c8yedge --help` / `c8yedge config --help` |
| U-07 | **VM 版 `/edge/configuration/*` の GET→POST 完全往復可否** | `security` については公式に保証。**他のエンドポイントは未確認** | 各エンドポイントで GET → そのまま POST を試験 |
| U-08 | **`c8y analytics` 拡張のメンテナンス状況** | 明示なし | リポジトリの最終コミットと Issue を確認 |
| U-09 | **`c8y ui` サブコマンドの詳細** | `applications` / `plugins` の 2 グループがあることのみ確認 | 実機で `c8y ui --help` |

### ⛔ 使ってはいけないもの

| ツール | 理由 |
|---|---|
| **cumulocity-migration-tool** | **2025-09-17 にリポジトリがアーカイブされ read-only。** 公式に *"This tool is not further developed and it might break with upcoming Cumulocity releases. Use it at your own risk."* と明記。ホステッド Web アプリで CLI ではないため CI にも不向き |
| **cumulocity-tenant-option-management-plugin** | deprecated |

出典: [cumulocity-migration-tool](https://github.com/Cumulocity-IoT/cumulocity-migration-tool)

---

## 8. go-c8y-cli コマンド名前空間 全一覧（v2 ブランチ実測）

リポジトリの `docs/cli/c8y/` 配下を機械的に列挙した結果（52 件）。**`inventoryroles` は存在しません。**

```
activitylog  agents        alarms         alias        api           applications
assert       auditrecords  binaries       bulkoperations  cache      cli
completion   configuration currentapplication  currenttenant  currentuser
databroker   datahub       devicegroups   devicemanagement  deviceprofiles
deviceregistration  devices  events       extensions   features      firmware
identity     inventory     measurements   microservices  notification2
operations   realtime      remoteaccess   retentionrules  scripts    sessions
settings     smartgroups   software       systemoptions  template    tenantoptions
tenants      tenantstatistics  ui         usergroups   userreferences
userroles    users         util           version
```

### 本ガイドで使用する主要サブコマンド

| 名前空間 | サブコマンド |
|---|---|
| `tenantoptions` | `create` `delete` `get` **`getForCategory`** `list` `update` **`updateBulk`** `updateEdit` |
| `systemoptions` | `get` `list` |
| `retentionrules` | `create` `delete` `get` `list` `update` |
| `usergroups` | `create` `delete` `get` **`getByName`** `list` `update` |
| `userroles` | **`addRoleToGroup`** `addRoleToUser` `deleteRoleFromGroup` `deleteRoleFromUser` `getRoleReferenceCollectionFromGroup` `getRoleReferenceCollectionFromUser` `list` |
| `userreferences` | **`addUserToGroup`** `deleteUserFromGroup` `listGroupMembership` |
| `users` | `create` `delete` `get` `getInventoryRole` `getUserByName` `list` `listInventoryRoles` `listUserMembership` `resetUserPassword` `revokeTOTPSecret` `update` |
| `applications` | `copy` `create` `createBinary` `createHostedApplication` `delete` `deleteApplicationBinary` `get` `list` `listApplicationBinaries` `open` `update` |
| `microservices` | `create` `createBinary` `delete` **`disable`** **`enable`** `get` `getBootstrapUser` `getStatus` `list` `update` |
| `inventory` | `count` `create` `delete` **`find`** `findByText` `get` `list` `subscribe` `update` `wait` |
| `features` | `delete` **`disable`** **`enable`** `get` `list` `update` |
| `smartgroups` | `create` `delete` `get` `list` `update` |
| `template` | `execute` |
| `assert` | `text` |
| `ui` | `applications` `plugins` |

出典: [go-c8y-cli v2 リポジトリ `docs/cli/c8y/`](https://github.com/reubenmiller/go-c8y-cli/tree/v2/docs/go-c8y-cli/docs/cli/c8y)（2026-08-06 時点、GitHub Contents API で列挙）

---

## 9. 出典一覧

**go-c8y-cli**
- [go-c8y-cli ドキュメント](https://goc8ycli.netlify.app/docs/) / [reubenmiller/go-c8y-cli (GitHub)](https://github.com/reubenmiller/go-c8y-cli)
- [ルートコマンド／グローバルフラグ](https://goc8ycli.netlify.app/docs/cli/c8y/c8y/)
- [select parameter](https://goc8ycli.netlify.app/docs/concepts/select-parameter/)
- [Templates](https://goc8ycli.netlify.app/docs/concepts/templates/) / [Output templates](https://goc8ycli.netlify.app/docs/concepts/output-templates/)
- [Chaining commands](https://goc8ycli.netlify.app/docs/concepts/chaining-commands/) / [Paging](https://goc8ycli.netlify.app/docs/concepts/paging/)
- [Dry run](https://goc8ycli.netlify.app/docs/concepts/dryrun/)
- [Sessions](https://goc8ycli.netlify.app/docs/concepts/sessions/) / [Settings](https://goc8ycli.netlify.app/docs/configuration/settings/)
- [Extensions](https://goc8ycli.netlify.app/docs/concepts/extensions/)
- [c8y api](https://goc8ycli.netlify.app/docs/cli/c8y/api/c8y_api/)
- [c8y sessions login](https://goc8ycli.netlify.app/docs/cli/c8y/sessions/c8y_sessions_login/) / [c8y sessions clone](https://goc8ycli.netlify.app/docs/cli/c8y/sessions/c8y_sessions_clone/)
- [Cumulocity-IoT/c8y-analytics](https://github.com/Cumulocity-IoT/c8y-analytics)

**Cumulocity API**
- [Cumulocity OpenAPI Specification](https://cumulocity.com/api/core/) / [raw spec](https://cumulocity.com/api/core/dist/c8y-oas.yml)
- [Cumulocity Edge OpenAPI](https://cumulocity.com/api/edge/)

**Cumulocity ドキュメント**
- [Working with dashboards](https://cumulocity.com/docs/cockpit/working-with-dashboards/)
- [Smart rules collection](https://cumulocity.com/docs/cockpit/smart-rules-collection/)
- [SSO](https://cumulocity.com/docs/authentication/sso/)
- [Microservice SDK general aspects](https://cumulocity.com/docs/microservice-sdk/general-aspects/)
- [Managing tenants](https://cumulocity.com/docs/enterprise-tenant/managing-tenants/)
- [Manage Edge (2026)](https://cumulocity.com/docs/2026/edge/manage-edge/) / [(2025 K8s)](https://cumulocity.com/docs/2025/edge-kubernetes/manage-edge/)
- [Edge configuration (2024)](https://cumulocity.com/docs/2024/edge/edge-configuration/)
- [Dashboard import/export GA 告知 (2026-02-19)](https://community.cumulocity.com/t/february-19-2026-dashboard-import-export-feature-moved-to-general-availability/14302)
- [SmartRuleRepresentation (Java SDK)](http://resources.cumulocity.com/documentation/javasdk/1007.5.0/com/cumulocity/rest/representation/cep/SmartRuleRepresentation.html)
