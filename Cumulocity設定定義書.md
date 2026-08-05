# Cumulocity 設定定義書（Edge ベース IoT 基盤 セットアップ設定項目一覧）

作成日: 2026-08-06
対象: Cumulocity IoT Edge を標準セットアップした後、自社 IoT 基盤としての設定を投入する工程
出典: 原則 cumulocity.com/docs および cumulocity.com/api（一次情報）

---

## 0. 本書の読み方と前提

### 0.1 設定は「2層」に分かれる

Cumulocity Edge の設定対象は、**まったく別の API・別のツールで管理される 2 つの層**に分かれます。ここを混同すると自動化設計を誤ります。

| 層 | 対象 | 管理手段 | OpenAPI |
|---|---|---|---|
| **レイヤ0: インフラ層** | Edge アプライアンス／Kubernetes 基盤（ネットワーク、ホスト名、ドメイン、TLS 証明書、ライセンス、OS セキュリティ、マイクロサービスホスティング有効化） | `c8yedge` CLI / Edge Custom Resource (kubectl) / `/edge/*` REST API | [Cumulocity Edge OAS (Release 10.18.0)](https://cumulocity.com/api/edge/) |
| **レイヤ1: テナント業務設定層** | テナントオプション、認証/SSO、ユーザー・ロール、アプリ購読、リテンション、ルール、分析モデル、ブランディング | Cumulocity Core REST API / GUI / go-c8y-cli | [Cumulocity Core OAS](https://cumulocity.com/api/core/dist/c8y-oas.yml) |

**根拠**: Edge OpenAPI スペック（`info.title="Cumulocity Edge"`, `info.version="Release 10.18.0"`）の paths には `/edge/*` の 18 本しか定義されておらず、`/tenant/options`、`/user`、`/retention` 等は一切含まれません。逆に Core OAS 側にそれらが定義されます。

### 0.2 ⚠️ 最重要: バージョン断層を先に確定すること

**Edge は Release 2024 までと 2025/2026 で設定手段が根本的に異なります。** 導入対象バージョンを確定しないとレイヤ0の自動化方式が決まりません。

| | VM アプライアンス版（〜Release 2024 / 10.18.0） | Kubernetes 版（Release 2025 / 2026） |
|---|---|---|
| 基盤 | 単一 VM アプライアンス | K3s（軽量 Kubernetes） |
| インストール | `POST /edge/install` + GUI ウィザード | `c8yedge install` CLI |
| 設定投入 | `/edge/configuration/*` REST API（18 エンドポイント） | `c8yedge config --set` / Edge Custom Resource + `kubectl apply` |
| 現行ドキュメント | [/docs/2024/edge/edge-configuration/](https://cumulocity.com/docs/2024/edge/edge-configuration/) | [/docs/2026/edge/](https://cumulocity.com/docs/2026/edge/edge-introduction/) |

`/docs/2026/edge/edge-configuration/` は **404**（存在しない）であり、Edge の OpenAPI は `/api/edge/` 上 10.18.0 の 1 版のみで凍結されています。2026 版ドキュメントは `/edge/configuration/*` に言及せず、Edge CR + `kubectl apply` による管理を記述します。

ただし `microservices` と `remote-connectivity` の 2 つは 2026 版ドキュメントからも参照が続いており、廃止されていません。

### 0.3 Edge には「テナントが 2 つ」ある

インストール後、**`management` テナントと `edge` テナント**の 2 つが生成されます。設定投入スクリプトは投入先を明示的に区別する必要があります。

| テナント | URL | 用途 |
|---|---|---|
| Edge テナント | `https://<domain_name>`（ローカル時は `http://localhost`） | 業務データ・デバイス・ユーザーの本体 |
| Management テナント | `https://management-<domain_name>` | メールサーバー設定、パスワードリセットテンプレート、プラットフォームレベル設定 |

初期資格情報は両方とも `admin` ／ インストール時に指定したパスワード（`--cumulocity-password` フラグまたは Secret `spec.cumulocityPasswordSecretName`）。ハードコードされた既定パスワードは存在しません。両テナントのパスワードは独立して変更できます。

> **設計上の推奨**: 公式は自動投入に admin を使えとは書いていません。専用のサービスユーザー／API 資格情報を発行してください。

出典: [Installing Edge](https://cumulocity.com/docs/2026/edge/installing-edge/)

### 0.4 「Edge 対応可否」の判定方法

公式は「Edge はクラウド版と同じソフトウェアだが *minor restrictions* がある」としか書いておらず、**単一の権威ある「Edge 非サポート機能一覧」は存在しません**。したがって本書の Edge 対応可否列は、Edge vs Cloud 比較表（§1.0）と各機能ページの記述から個別に判定しています。

出典: [Edge introduction](https://cumulocity.com/docs/2026/edge/edge-introduction/)

### 0.5 凡例

- 自動化可否: **◎** = 公式 API/CLI で完全自動化可 / **○** = API はあるが一部手作業 / **△** = 回避策あり / **×** = GUI 手作業のみ
- エクスポート可否: **◎** = GET で完全に取得可 / **△** = 部分的（暗号化値等が欠落） / **○** = ファイル/JSON ダウンロード機能あり / **×** = 不可
- Edge: **◎** = 利用可 / **△** = 制約付き / **×** = 非対応・N/A

---

## 1. レイヤ0 ─ Edge インフラ層

### 1.0 Edge vs Cloud 機能比較（公式表・全行）

| 領域 | Edge | Cumulocity platform |
|---|---|---|
| Multi-tenancy | No; single tenant | Yes |
| Cluster | No; single server | Yes |
| High availability | HA は基盤仮想化に依存。サーバ障害でダウンタイムの可能性 | Full HA |
| Vertical scalability | Yes（CPU コアあたり約 100 tps に制限） | Yes（未使用） |
| Horizontal scalability | No | Yes |
| Upgrades with no downtime | No | Yes |
| Root access | Yes | Yes（顧客ホスティング時） |
| Installation | Online and offline | Online |
| Messaging Service | Included | Included |
| MQTT Service | Included | Included |
| Microservice-based data broker | **Included** | Optional |
| Microservice Hosting | **Included** | Optional |
| Streaming Analytics | **Included** | Optional |
| OPC UA | **Included** | Optional |
| Cloud Field Bus | **Included** | Optional |
| Data Hub | On request via Professional Services | Optional |

> **重要**: Streaming Analytics（Analytics Builder 含む）、マイクロサービスホスティング、OPC UA、Cloud Field Bus は Edge に**同梱**されています。「Edge だから使えない」機能は実際にはかなり狭く、中核は **Enterprise マルチテナンシー（サブテナント作成・テナント階層・サブテナント別ブランディング・テナント別課金）が無い**という点です。

出典: [Edge introduction](https://cumulocity.com/docs/2026/edge/edge-introduction/)

### 1.1 Kubernetes 版（2025/2026）— `c8yedge` CLI / Edge CR

| # | 設定項目名 | 説明 | GUI 上の場所 | 投入手段 | 自動化 | エクスポート | Edge |
|---|---|---|---|---|---|---|---|
| L0-01 | `domain` | Edge をホストする FQDN | なし（CLI のみ） | `c8yedge config --set domain=<FQDN>` | ◎ | ◎ (`kubectl get edge -o yaml`) | ◎ |
| L0-02 | `licenseKey` | ライセンスキー。**ドメイン名に紐づくため domain 変更と必ず同時に実施** | なし | `c8yedge config --set-file licenseKey=<path>` | ◎ | △ | ◎ |
| L0-03 | `tlsSecret.tls.crt` / `tls.key` | Edge 自身のサーバ TLS 証明書。未指定時は自己署名証明書を自動生成 | なし | `c8yedge config --set-file tlsSecret.tls.crt=<path> --set-file tlsSecret.tls.key=<path>` | ◎ | △ | ◎ |
| L0-04 | `company` | 組織名 | あり | `c8yedge config --set` / UI | ◎ | ◎ | ◎ |
| L0-05 | `email` | 管理者メールアドレス | あり | `c8yedge config --set` / UI | ◎ | ◎ | ◎ |
| L0-06 | `cumulocityPasswordSecretName` | admin パスワードを保持する K8s Secret 名 | なし | Edge CR | ◎ | × (Secret) | ◎ |
| L0-07 | `cloudTenant.*` | クラウドテナントへの接続設定（X.509 クライアント認証用 `cloudTenant.tlsSecret` を含む。L0-03 とは**別物**） | あり | Edge CR / CLI | ◎ | △ | ◎ |
| L0-08 | `version` / `storageClassName` / `core` / `messagingService` / `mongodb` / `microservices` / `dataHub` | 各コンポーネントのリソース・設定 | なし | Edge CR | ◎ | ◎ | ◎ |

**公式サンプルコマンド**（非対話・スクリプト化可）:

```bash
c8yedge config --set domain=<DOMAIN-NAME> --set-file licenseKey=<path> --set-file tlsSecret.tls.key=<path> --set-file tlsSecret.tls.crt=<path>
```

**設定可能項目の母集団**: 公式が *"an exhaustive listing of what could be changed"* と明記する [Edge custom resource](https://cumulocity.com/docs/2026/edge/edge-custom-resource-definition/) リファレンスを参照してください。これがレイヤ0 の設定項目一覧の正となります。

出典: [Manage Edge](https://cumulocity.com/docs/2025/edge-kubernetes/manage-edge/), [Edge CR definition](https://cumulocity.com/docs/2026/edge/edge-custom-resource-definition/), [Connecting Edge to cloud](https://cumulocity.com/docs/2026/edge/connecting-edge-to-cloud/)

### 1.2 VM アプライアンス版（〜2024）— `/edge/*` REST API 全 18 エンドポイント

| # | 設定項目名 | エンドポイント | 説明 | 自動化 | エクスポート |
|---|---|---|---|---|---|
| L0-11 | ネットワーク | `GET/POST /edge/configuration/network` | IP、DNS、ゲートウェイ | ◎ | ◎ |
| L0-12 | ホスト名 | `GET/POST /edge/configuration/hostname` | ホスト名 | ◎ | ◎ |
| L0-13 | ドメイン名 | `GET/POST /edge/configuration/domain` | カスタムドメイン | ◎ | ◎ |
| L0-14 | SSL 証明書 | `GET/POST /edge/configuration/certificate` | 証明書アップロード／自己署名生成 | ◎ | △ |
| L0-15 | 時刻同期 | `GET/POST /edge/configuration/time-sync` | NTP | ◎ | ◎ |
| L0-16 | OS/K8s セキュリティ | `GET/POST /edge/configuration/security` | §1.3 参照 | ◎ | ◎ |
| L0-17 | マイクロサービスホスティング | `GET/POST /edge/configuration/microservices` | §1.4 参照 | ◎ | ◎ |
| L0-18 | リモート接続 | `GET/POST /edge/configuration/remote-connectivity` | §1.4 参照 | ◎ | ◎ |
| L0-19 | インストール | `POST /edge/install` | 初期インストール | ○ | × |
| L0-20 | アップデート | `POST /edge/update` | アプライアンス更新 | ◎ | × |
| L0-21 | 再起動 | `POST /edge/reboot` | — | ◎ | × |
| L0-22 | ディスク拡張 | `POST /edge/expand-disk` | — | ◎ | × |
| L0-23 | 診断 | `POST /edge/diagnostics` / `GET /edge/diagnostics/{id}` | 診断レポート | ◎ | ○ |
| L0-24 | バージョン | `GET /edge/version` | — | ◎ | ◎ |
| L0-25 | タスク | `GET /edge/tasks/latest-installation`, `/edge/tasks/{id}`, `/edge/tasks/{id}/log` | 非同期タスク状態 | ◎ | ◎ |

出典: [Cumulocity Edge OAS](https://cumulocity.com/api/edge/) / [raw spec](https://cumulocity.com/api/edge/10.18.0/dist/c8y-edge-oas.json), [Edge configuration](https://cumulocity.com/docs/2024/edge/edge-configuration/)

### 1.3 ⭐ `POST /edge/configuration/security` — GUI から JSON を抽出して再投入できる

**ご要望の「GUI から設定値を抽出 → JSON で保持 → API 投入」が公式に成立する数少ない箇所です。**

> 公式記述: *"Click Download configuration to download a sample JSON syntax for the current configuration. You can use the same JSON file in the POST operation using the REST API."*

リクエストボディはトップレベルに `OS` と `kubernetes` の 2 プロパティのみを持つ単一 JSON:

| キー | 既定値 | 値域 |
|---|---|---|
| `OS.login_banner` | — | 文字列 |
| `OS.login_sessions_inactivity_timeout_seconds` | 600 | 整数（最小 0） |
| `OS.rsyslog.{server,port,protocol}` | 未設定 | protocol: `TCP` \| `UDP` |
| `OS.audisp.{server,port}` | 未設定 | — |
| `OS.ssh_enabled` | `true` | boolean |
| `OS.selinux_mode` | `permissive` | `permissive` \| `enforcing` |
| `OS.audit_logging_enabled` | `false` | boolean ⚠️ **一度有効化すると無効化不可** |
| `kubernetes.audit_policy.level` | `None` | `None` \| `Metadata` \| `Request` \| `RequestResponse` |
| `kubernetes.audit_policy.max_size` | 100 MB | — |
| `kubernetes.audit_policy.max_backup` | 10 | — |
| `kubernetes.audit_policy.max_age` | 30 日 | — |

> ⚠️ `OS.audit_logging_enabled` は公式に *"Once enabled, you cannot disable the audit logging configuration."* と明記された**不可逆設定**です。基盤テンプレートに含める前に方針を確定してください。

### 1.4 明示的に有効化が必要な Edge 固有項目

| 項目 | 既定 | 有効化手順 | 要件 |
|---|---|---|---|
| **マイクロサービスホスティング** | **無効** | `POST /edge/configuration/microservices` に `{"enabled": true}` / GUI: Administration > Edge > Microservices | `Tenant Manager` ロール、最低 4 vCPU / 8 GB RAM、`10.96.0.0/12` を予約、有効化に **10〜15 分** |
| **リモートデバイス管理** | **無効** | `POST /edge/configuration/remote-connectivity` に `{"enabled": ..., "remote_tenant_url": "https://..."}` / GUI: Edge > Remote Connectivity | クラウドテナント |

### 1.5 メールサーバー／パスワードリセット（インストール直後は未設定）

パスワードリセット機能は**インストール直後は使えません**。Management テナントにログインし、「reset password」テンプレートとメールサーバー設定（host / port / protocol / username / password / sender address）を構成する必要があります。

> 公式記述: *"To reset your password, you must first configure the \"reset password\" template and email server settings in Edge."*
> *"Configuring an email server enables you to receive email notifications about events, alarms, and also to reset your password."*

**投入先は Management テナント。** Administration > Settings 配下 ＝ tenant options 経由で API 自動投入が可能な領域です。

> ⚠️ `manage-edge` には**アプライアンス更新時に設定が上書きされ再適用が必要**との運用注記があります。この設定を外部 JSON で保持し再投入可能にしておく価値が高い項目です。

出典: [Installing Edge #how-to-reset-your-password](https://cumulocity.com/docs/2026/edge/installing-edge/), [Manage Edge #email-server](https://cumulocity.com/docs/2026/edge/manage-edge/)

---

## 2. レイヤ1 ─ テナントオプション（設定の中核）

### 2.1 ⭐ `PUT /tenant/options/{category}` — バルク投入の主力

**JSON ファイルで設定を保持してテナントへ流し込む IaC 運用の中心となるエンドポイントです。**

- リクエストボディ = 任意個の key-value ペアを持つ単純な JSON オブジェクト（`CategoryOptions` スキーマ）
  例: `{"temp_too_high":"120","temp_too_low":"0"}`
- 既存カテゴリの更新だけでなく**新規カテゴリの作成にも使える**（公式が `measurement.series.latestvalue` カテゴリを PUT で作る例を提示）
- 必要ロール: `ROLE_OPTION_MANAGEMENT_ADMIN`

| 操作 | エンドポイント |
|---|---|
| 全件取得 | `GET /tenant/options` |
| 単体作成 | `POST /tenant/options` |
| **カテゴリ単位バルク更新/作成** | **`PUT /tenant/options/{category}`** |
| カテゴリ単位取得 | `GET /tenant/options/{category}` |
| 単体取得/更新/削除 | `GET`/`PUT`/`DELETE /tenant/options/{category}/{key}` |
| 編集可否の制御（management テナント専用） | `PUT /tenant/options/{category}/{key}/editable` |

go-c8y-cli にも専用動詞があります: `c8y tenantoptions updateBulk --category <cat> --data '{...}'`

### 2.2 ⚠️ GET→PUT ラウンドトリップの落とし穴

「現行設定を GET でエクスポート → 編集 → PUT で再投入」は**無条件には成立しません**。

| リスク | 内容 | 対策 |
|---|---|---|
| **暗号化オプション** | キーに `credentials.` 接頭辞を付けると値は暗号化保存され、**読み戻すと `"<<Encrypted>>"` という定数が返る**。GET→PUT すると秘密情報が文字列 `<<Encrypted>>` で上書きされる | 秘密値は別管理（Vault/Secret）にし、投入時のみ注入。エクスポート対象から除外 |
| **継承値の実体化** | Enterprise テナントは management テナントからの継承オプションも GET で読める。素朴な GET→PUT は**継承値をローカルに実体化**させてしまう | Edge では Enterprise サブテナント階層が無いため影響は小。ただしクラウド側では注意 |
| **non-editable オプション** | management テナントが「非編集」に設定したオプションは PUT/DELETE が **403** | 事前に `editable` 状態を確認 |
| **定義済みキー限定カテゴリ** | 一部カテゴリは新規キーを自由に作れない | カテゴリごとに確認 |

> 復号は、**そのオプションのオーナーであるシステムユーザー（サービスユーザー／ブートストラップユーザー）に対してのみ**行われます。オーナー判定は「マイクロサービスマニフェストの `settingsCategory` → コンテキストパス → マイクロサービス名」の優先順です。

### 2.3 既知の標準カテゴリ／キー（公式スペックに明記されているもの）

| カテゴリ | キー | 既定値 | 説明 |
|---|---|---|---|
| `access.control` | `allow.origin` | `*` | CORS 許可ドメインのカンマ区切りリスト（ワイルドカード可: `*.cumulocity.com`） |
| `alarm.type.mapping` | `<ALARM_TYPE>` | — | 当該アラーム型の severity とテキストを上書き。形式 `<SEVERITY>\|<TEXT>`。severity が `NONE` ならアラームを抑止 |
| `configuration` | `acl.algorithm-version` | `OPTIMIZED` | インベントリロールベースアクセスの最適化。`LEGACY` \| `OPTIMIZED` |
| `configuration` | `acl.measurement.only-accessible-fragments` | — | `true` で、計測に含まれる全フラグメントへのアクセスを要求せず特定フラグメントのみ許可可能に |
| `measurement.series.latestvalue` | `strongConsistency` 他 | — | 最新値を `c8y_LatestMeasurements` に永続化 |
| `configuration` | `default.tenant.applications` | — | 新規テナントの既定購読 Web アプリ（カンマ区切り）※ **Enterprise サブテナント機能。Edge では N/A** |
| `configuration` | `default.tenant.microservices` | — | 同上（マイクロサービス）※ **Edge では N/A** |
| `configuration` | `on-update.tenant.applications[.enabled]` | — | プラットフォーム更新時の購読制御 ※ **Edge では N/A** |
| `configuration` | `on-update.tenant.microservices[.enabled]` | — | 同上 ※ **Edge では N/A** |

> ⚠️ **カテゴリ名に使えない文字**: 空白および `` $ & + , / : ; = ? @ " < > # % { } | \ ^ ~ [ ] ` ``

> ⚠️ **テナントオプションの網羅的な「全キー一覧」は公式に存在しません。** 上記は OpenAPI スペックに明記されたもののみです。実運用では、リファレンス環境で GUI から設定 → `GET /tenant/options` で差分を取る、という**リバースエンジニアリング方式**が現実的です（→ §9.4）。

出典: [Options API (Core OAS)](https://cumulocity.com/api/core/#tag/Options), [Managing data](https://cumulocity.com/docs/standard-tenant/managing-data/), [Managing permissions](https://cumulocity.com/docs/standard-tenant/managing-permissions/)

### 2.4 システムオプション（読み取り専用・全 33 キー公式一覧）

`GET /tenant/system/options` および `GET /tenant/system/options/{category}/{key}`。**テナント側からの投入・変更は不可**（プラットフォーム設定側で決まる）。`ROLE_OPTION_MANAGEMENT_ADMIN` が無い場合、secured 列が yes のものは `"<<Encrypted>>"` に難読化されます。

| カテゴリ | キー | secured |
|---|---|---|
| `password` | `green.min-length` | **yes** |
| `password` | `limit.validity` | no |
| `password` | `enforce.strength` | no |
| `two-factor-authentication` | `pin.validity` | **yes** |
| `two-factor-authentication` | `token.length` | **yes** |
| `two-factor-authentication` | `token.validity` | **yes** |
| `two-factor-authentication` | `enabled` | no |
| `two-factor-authentication` | `enforced` | no |
| `two-factor-authentication` | `enforced.group` | no |
| `two-factor-authentication` | `strategy` | no |
| `two-factor-authentication` | `pin.length` | no |
| `two-factor-authentication` | `tenant-scope-settings.enabled` | no |
| `two-factor-authentication` | `logout-on-browser-termination` | no |
| `authentication` | `badRequestCounter` | **yes** |
| `files` | `microservice.zipped.max.size` | **yes** |
| `files` | `microservice.unzipped.max.size` | **yes** |
| `files` | `webapp.zipped.max.size` | **yes** |
| `files` | `webapp.unzipped.max.size` | **yes** |
| `files` | `max.size` | no |
| `reportMailer` | `available` | no |
| `system` | `version` | no |
| `plugin` | `eventprocessing.enabled` | no |
| `connectivity` | `microservice.url` | no |
| `support-user` | `enabled` | no |
| `support` | `url` | no |
| `trackers` | `supported.models` | no |
| `encoding` | `test` | no |
| `data-broker` | `bootstrap.period` | no |
| `device-control` | `bulkoperation.creationramp` | no |
| `gainsight` | `api.key` | no |
| `cep` | `deprecation.alarm` | no |
| `remoteaccess` | `pass-through.enabled` | no |
| `device-registration` | `security-token.policy` | no |

出典: [System options tag, Core OAS](https://cumulocity.com/api/core/dist/c8y-oas.yml)

> **セットアップ手順での使い方**: これらは投入対象ではなく**検証対象**です。セットアップ完了後に `GET /tenant/system/options` を叩き、期待値（TFA 方針、パスワード方針、ファイルサイズ上限）になっているかをアサートするスモークテストに使ってください。

---

## 3. 認証・SSO・パスワードポリシー・TFA

### 3.1 基本認証設定（Administration > Settings > Authentication）

| # | 設定項目名 | 説明 | 既定値 | GUI 上の場所 | REST API | 自動化 | エクスポート | Edge |
|---|---|---|---|---|---|---|---|---|
| A-01 | Preferred login mode | `OAI-Secure` / `Basic Auth` / `Single sign-on redirect`。SSO を選ぶと Basic/OAI-Secure は選択肢から消える | OAI-Secure（新規テナント） | Settings > Authentication | `PUT /tenant/loginOptions/{typeOrId}` | ◎ | ◎ | ◎ |
| A-02 | Password validity limit | パスワード有効期限（日）。**0 = 無制限**。`devices` ロールのユーザーには適用されない | 0 | 同上 | 同上 | ◎ | ◎ | ◎ |
| A-03 | Enforce strong (green) password | 全パスワードに「強い（緑）」を強制。既定は 8 文字以上のみ | 無効 | 同上 | 同上 | ◎ | ◎ | ◎ |
| A-04 | Ignore case when logging in | ログイン時に大文字小文字を無視。**既存ユーザー間に名前衝突が無い場合のみ有効化可** | 無効 | 同上 | 同上 | ◎ | ◎ | ◎ |
| A-05 | Basic auth forbidden for web browsers | ブラウザからの Basic 認証を禁止 | — | 同上 | `authenticationRestrictions` | ◎ | ◎ | ◎ |
| A-06 | Trusted user agents | 許可する User-Agent リスト | 空 | 同上 | `authenticationRestrictions` | ◎ | ◎ | ◎ |
| A-07 | Forbidden user agents | 拒否する User-Agent リスト | 空 | 同上 | `authenticationRestrictions` | ◎ | ◎ | ◎ |
| A-08 | Allow two-factor authentication | TFA 有効化。**管理者に対してのみ設定可** | 無効 | 同上 | `PUT /tenant/tenants/{tenantId}/tfa` | ◎ | ◎ | ◎ |
| A-09 | SMS TFA: token validity limit | SMS トークン有効期限（分） | — | 同上 | 同上 | ◎ | ◎ | ◎ |
| A-10 | SMS TFA: verification code validity | 検証コード有効期限 | — | 同上 | 同上 | ◎ | ◎ | ◎ |
| A-11 | Enforce TOTP TFA on all users | 全ユーザーに TOTP を強制。**ログインモードが OAI-Secure の場合のみ利用可**。`devices` ロールには適用されない | 無効 | 同上 | 同上 | ◎ | ◎ | ◎ |
| A-12 | `onlyManagementTenantAccess` | この設定を management テナントからのみアクセス可に制限 | false | — | loginOptions フィールド | ◎ | ◎ | ◎ |

出典: [Basic settings](https://cumulocity.com/docs/authentication/basic-settings/)

### 3.2 ⭐ Login options API — 認証設定の自動投入の要

| 操作 | エンドポイント |
|---|---|
| 一覧取得 | `GET /tenant/loginOptions` |
| 作成 | `POST /tenant/loginOptions` |
| 個別取得 | `GET /tenant/loginOptions/{typeOrId}` |
| **更新** | **`PUT /tenant/loginOptions/{typeOrId}`**（operationId: `putAccessLoginOptionResource`） |
| 管理テナント制限 | `PUT /tenant/loginOptions/{typeOrId}/restrict` |
| アクセスマッピング | `GET/POST /tenant/loginOptions/{typeOrId}/accessMappings`, `GET/PUT/DELETE .../accessMappings/{id}` |
| インベントリアクセスマッピング | `GET/POST /tenant/loginOptions/{typeOrId}/inventoryAccessMappings`, `.../{id}` |
| OAuth トークン／証明書 | `POST /tenant/oauth`, `/tenant/oauth/token`, `GET /tenant/oauth/certificate` |

### 3.3 `authConfig` スキーマ全フィールド（SSO 設定 JSON の実体）

`type` と `providerName` が必須。**この JSON をそのまま設定ファイルとして保持し `PUT` すれば SSO 設定を自動投入できます。**

| フィールド | 型 / 値域 | 説明 |
|---|---|---|
| `type` **(必須)** | `BASIC` \| `OAUTH2` \| `OAUTH2_INTERNAL`（大小文字非依存） | 認証設定タイプ |
| `providerName` **(必須)** | string | プロバイダ名 |
| `grantType` | `AUTHORIZATION_CODE` \| `PASSWORD` | グラントタイプ |
| `userManagementSource` | `INTERNAL` \| `REMOTE` | ユーザーデータを内部管理か外部サーバー管理か |
| `clientId` | string | 外部認可サーバー上の Cumulocity テナント識別子 |
| `issuer` | URI | 外部トークン発行者 |
| `audience` | URI | トークンオーディエンス |
| `buttonName` | string | ログイン画面のボタン表示名 |
| `template` | string | UI が使うテンプレート名 |
| `visibleOnLoginPage` | boolean | ログイン画面に表示するか |
| `redirectToPlatform` | URI | プラットフォームへのリダイレクト URL。**空にすると SSO フローをクライアント(UI)側が制御** |
| `onlyManagementTenantAccess` | boolean | management テナント限定 |
| `useIdToken` | boolean | `true` でユーザーデータ/userId を **id_token** のクレームから取得（既定は access_token） |
| `authorizationRequest` | RequestRepresentation | 認可コード取得リクエスト |
| `tokenRequest` | RequestRepresentation | アクセストークン取得リクエスト |
| `refreshRequest` | RequestRepresentation | リフレッシュトークン取得リクエスト |
| `logoutRequest` | RequestRepresentation | ログアウトリクエスト（フロントチャネル SLO） |
| `sessionConfiguration` | OAuthSessionConfiguration | `absoluteTimeoutMillis`, `renewalTimeoutMillis`, `userAgentValidationRequired`, `maximumNumberOfParallelSessions` |
| `authenticationRestrictions` | BasicAuthenticationRestrictions | A-05〜A-07 の実体 |
| `userIdConfig.jwtField` | string | JWT のどのフィールドをユーザー名に使うか |
| `userIdConfig.useConstantValue` / `constantValue` | boolean / string(≤1000) | ⚠️ **非推奨**。全 SSO ユーザーが 1 アカウントを共有 |
| `accessTokenToUserDataMapping.{emailClaimName, firstNameClaimName, lastNameClaimName, phoneNumberClaimName}` | string | ユーザー属性のクレーム名マッピング |
| `signatureVerificationConfig.aad.publicKeyDiscoveryUrl` | URI | Azure AD 署名検証 |
| `signatureVerificationConfig.adfsManifest.manifestUrl` | URI | ADFS マニフェスト |
| `signatureVerificationConfig.jwks.jwksUrl` | URI | JWKS |
| `signatureVerificationConfig.manual.{certIdField, certIdFromField, certificates{alg(RSA\|PCKS), publicKey, validFrom, validTill}}` | — | X.509 手動指定 |
| `onNewUser.dynamicMapping.configuration.mapRolesOnlyForNewUser` | boolean | マッピングを初回ログイン時のみ評価するか |
| `onNewUser.dynamicMapping.configuration.manageRolesOnlyFromAccessMapping` | boolean | `true` で設定に列挙されたロール/アプリ/インベントリロールのみ動的管理、他は保持 |
| `onNewUser.dynamicMapping.configuration.mapFromIdToken` | boolean | 動的マッピングを id_token のクレームで行うか |
| `onNewUser.dynamicMapping.mappings[]` | accessMappingWithId[] | グローバルロール／アプリ割当ルール |
| `onNewUser.dynamicMapping.inventoryMappings[]` | inventoryAccessMappingWithId[] | インベントリロール割当ルール |
| `externalTokenConfig.enabled` | boolean | 外部アクセストークンでの認証を有効化 |
| `externalTokenConfig.validationRequired` | boolean | 認可サーバーへ introspection/userinfo で検証するか |
| `externalTokenConfig.validationMethod` | `INTROSPECTION` \| `USERINFO` | 検証方式 |
| `externalTokenConfig.tokenValidationRequest` | RequestRepresentation | 検証リクエスト |
| `externalTokenConfig.accessTokenValidityCheckIntervalInMinutes` | integer | 検証頻度（**推奨 1 分**） |
| `externalTokenConfig.userOrAppIdConfig.*` | — | 外部トークンのどのクレームをユーザー名にするか |

**公式スペックの example（OAuth Internal の実値）**:

```json
{
  "self": "https://<TENANT_DOMAIN>/tenant/loginOptions/924997e5-863c-4532-96f9-cbe6dc5f8902",
  "userManagementSource": "INTERNAL",
  "type": "OAUTH2_INTERNAL",
  "sessionConfiguration": {
    "absoluteTimeoutMillis": 7200000,
    "renewalTimeoutMillis": 3600000,
    "userAgentValidationRequired": false,
    "maximumNumberOfParallelSessions": 3
  },
  "id": "924997e5-863c-4532-96f9-cbe6dc5f8902",
  "providerName": "Cumulocity",
  "visibleOnLoginPage": true,
  "grantType": "PASSWORD",
  "onlyManagementTenantAccess": false
}
```

### 3.4 SSO（GUI: Administration > Authentication > Single sign-on）

> ⚠️ **重要: SAML はサポートされません。** 公式に *"SAML is not supported."* と明記。サポートされるのは **OAuth2 authorization code grant + JWT** のみです。当初の想定（SAML2 対応）は成り立ちません。

GUI で設定できる項目群（すべて §3.3 の `authConfig` にマップされます）:

- **リクエスト設定**: Authorize URL (GET) / Token URL (POST) / Refresh URL (POST) / Logout URL（任意、フロントチャネル SLO）、各リクエストのパラメータ・ヘッダ・ボディ
- **基本設定**: Redirect URI、Client ID、Token issuer、Button name、Provider name、Audience (aud)、mTLS 認証（PEM 形式の証明書と秘密鍵）、Visible on Login screen
- **アクセスマッピング**: ソース選択（access token / id token）、動的アクセスマッピング方針 3 種（①ユーザー作成時のみ割当 ②未マップのロールを保持して再割当 ③未マップのロールをクリアして完全再割当）、JWT クレームとグローバルロール／既定アプリ／インベントリロールのマッチングルール
- **ユーザーデータマッピング**: 姓・名・メール・電話のクレーム名、ユーザー ID クレーム（定数値オプション付き）
- **署名検証**: Azure AD 証明書ディスカバリ / ADFS マニフェストアドレス / 公開鍵手動アップロード / JWKS URI

> ⚠️ 公式注記: *"Placeholders are not validated for correctness."* — プレースホルダの綴り誤りは検証されず、そのまま未処理で残ります。SSO 設定 JSON はレビュー必須です。

**SSO タブは、SSO 設定アクセス権を持つテナントにのみ表示されます。**

出典: [SSO](https://cumulocity.com/docs/authentication/sso/), [Login options API](https://cumulocity.com/api/core/#operation/putAccessLoginOptionResource)

**参考実装**: [Cumulocity-IoT/cumulocity-provision-sso](https://github.com/Cumulocity-IoT/cumulocity-provision-sso)（SSO 設定のプロビジョニング例。公式組織リポジトリ）

---

## 4. ユーザー・ロール・権限

### 4.1 REST API エンドポイント全一覧（Core OAS より）

| 対象 | エンドポイント |
|---|---|
| ユーザー | `GET/POST /user/{tenantId}/users`, `GET/PUT/DELETE /user/{tenantId}/users/{userId}`, `GET /user/{tenantId}/userByName/{username}` |
| ユーザーのグローバルロール割当 | `GET/POST /user/{tenantId}/users/{userId}/roles`, `DELETE .../roles/{roleId}` |
| ユーザーのインベントリロール割当 | `GET/POST /user/{tenantId}/users/{userId}/roles/inventory`, `GET/PUT/DELETE .../roles/inventory/{id}` |
| ユーザーの所属グループ | `GET /user/{tenantId}/users/{userId}/groups` |
| ユーザー単位 TFA | `GET/DELETE /user/{tenantId}/users/{userId}/tfa` |
| グローバルロール（グループ） | `GET/POST /user/{tenantId}/groups`, `GET/PUT/DELETE /user/{tenantId}/groups/{groupId}`, `GET /user/{tenantId}/groupByName/{groupName}` |
| グループへのロール割当 | `GET/POST /user/{tenantId}/groups/{groupId}/roles`, `DELETE .../roles/{roleId}` |
| グループへのユーザー割当 | `GET/POST /user/{tenantId}/groups/{groupId}/users`, `DELETE .../users/{userId}` |
| ロール（権限）一覧 | `GET /user/roles`, `GET /user/roles/{name}` |
| **インベントリロール定義** | `GET/POST /user/inventoryroles`, `GET/PUT/DELETE /user/inventoryroles/{id}` |
| デバイス権限 | `GET/PUT /user/devicePermissions/{id}` |
| 現在ユーザー | `GET/PUT /user/currentUser`, `PUT /user/currentUser/password` |
| TOTP | `POST/DELETE /user/currentUser/totpSecret`, `GET/POST /user/currentUser/totpSecret/activity`, `POST /user/currentUser/totpSecret/verify` |
| ログアウト | `POST /user/logout`, `POST /user/logout/{tenantId}/allUsers` |

> **一括投入の設計**: 上記はすべて単体 CRUD です。「1 リクエストで N ユーザー」というバルクエンドポイントはありません。**JSON 配列をループして POST する**（go-c8y-cli のパイプ入力＋テンプレートが最適 → §9.2）方式になります。

### 4.2 標準グローバルロール（Administration > Accounts > Roles > Global roles）

| ロール名 | 用途 |
|---|---|
| `admins` | 管理権限。初期テナント管理者に割当 |
| `devices` | デバイスアカウント用標準セット。登録デバイスに自動割当 |
| `CEP Manager` | **スマートルールおよびイベント処理ルールへのアクセス**（EPL Apps 編集に必須） |
| `Cockpit User` | Cockpit アプリアクセス（別途デバイスアクセスロールが必要） |
| `Devicemanagement User` | Device Management アプリ、シミュレータ、バルクオペレーション |
| `Global Manager` | 全デバイスデータの読み書き |
| `Global Reader` | 全デバイスデータの読み取り専用 |
| `Global User Manager` | 全ユーザー管理 |
| `Shared User Manager` | サブユーザー管理（ユーザー階層サブスクリプションが必要） |
| `Tenant Manager` | **テナント全体の設定管理**（Edge のマイクロサービスホスティング有効化にも必要） |
| `business`（レガシー） | 管理権限なしのデバイスアクセス |
| `readers`（レガシー） | ユーザーを含む読み取り専用 |

**権限レベル**: `READ` / `CREATE` / `UPDATE`（READ を含まない） / `ADMIN`

**権限タイプ（全 21 種）**: Alarms, Application management, Audits, Bulk operations, CEP management, Data broker, Device control, Events, Global smart rules, Identity, Inventory, Measurements, Option management, Retention rules, Schedule reports, Simulator, SMS, Tenant management, Tenant statistics, User management, Own user management

### 4.3 標準インベントリロール（Administration > Accounts > Roles > Inventory roles）

| ロール名 | 内容 |
|---|---|
| `Manager` | 全アセットデータ読み取り＋インベントリ／ダッシュボード／アラーム管理 |
| `Operations: All` | オペレーション経由のリモートデバイス管理 |
| `Operations: Restart Device` | デバイス再起動のみ |
| `Reader` | アセットデータ読み取り専用 |

- **権限カテゴリ**: Alarms, Audits, Events, Inventory, Measurements, Device control, Full access
- **権限レベル**: `READ` / `CHANGE`（READ を含まない） / `ALL`
- **Type フィールド**: フラグメント型で制限可。既定は `*`（全型）
- **継承**: 親グループ → サブグループ → 内包デバイスへ継承
- 割当場所: Administration > Accounts > Users > [ユーザー] > Inventory roles タブ
- 「Copy inventory roles from another user」でリファレンスユーザーから複製可（マージ／置換を選択）

出典: [Managing permissions](https://cumulocity.com/docs/standard-tenant/managing-permissions/)

### 4.4 関連するアクセス制御チューニング

| 設定 | tenant option | 値 | 説明 |
|---|---|---|---|
| ACL アルゴリズム | `configuration` / `acl.algorithm-version` | `LEGACY` \| `OPTIMIZED`（既定 OPTIMIZED） | インベントリロールベースアクセスの性能最適化。プラットフォームレベルの設定ファイルでも指定可 |
| フラグメント単位許可 | `configuration` / `acl.measurement.only-accessible-fragments` | `true` | 計測内の全フラグメントへのアクセスを要求せず、特定フラグメントのみ許可可能に |

> OPTIMIZED モードが効くのは、マッチ件数が **2000 件未満**、または特定デバイスのデータ取得時のみ。LEGACY では `currentPage` がスキャンオフセットとして働き、空ページが返ることがあります（`statistics.next` で継続）。

**トラブルシュート**: 画面右上のユーザーアバター > 「Access denied requests」でアクセス拒否の詳細を確認できます。

---

## 5. アプリケーション・マイクロサービス

| # | 設定項目名 | 説明 | GUI 上の場所 | REST API | 自動化 | エクスポート | Edge |
|---|---|---|---|---|---|---|---|
| P-01 | アプリケーション一覧／作成 | Web アプリの登録 | Ecosystem > Applications | `GET/POST /application/applications`, `GET/PUT/DELETE /application/applications/{id}` | ◎ | ◎ | ◎ |
| P-02 | アプリケーションバイナリ | ZIP（`index.html` + `cumulocity.json`）アップロード。最大 500 MB | 同上 | `GET/POST /application/applications/{id}/binaries`, `.../binaries/{binaryId}` | ◎ | ○ | ◎ |
| P-03 | アプリケーションバージョン | バージョン管理 | 同上 | `GET/POST /application/applications/{id}/versions`, `.../versions/{version}` | ◎ | ◎ | ◎ |
| P-04 | アプリケーション複製 | 購読アプリのカスタムコピー作成（`Overrule subscribed application` トグルで元を置換） | 同上 | `POST /application/applications/{id}/clone` | ◎ | — | ◎ |
| P-05 | **テナントのアプリ購読** | テナントへのアプリ割当／解除 | Ecosystem | `GET/POST /tenant/tenants/{tenantId}/applications`, `DELETE .../applications/{applicationId}` | ◎ | ◎ | ◎ |
| P-06 | テナント別アプリ一覧 | 所有者／テナント／ユーザー別の絞り込み | — | `GET /application/applicationsByTenant/{tenantId}`, `ByOwner/{tenantId}`, `ByUser/{username}`, `ByName/{name}` | ◎ | ◎ | ◎ |
| P-07 | 制限ロール | アプリに紐づく制限ロール | — | `GET/POST /tenant/tenants/{tenantId}/applications/restricted-roles`, `.../{roleId}` | ◎ | ◎ | ◎ |
| P-08 | ブートストラップユーザー | マイクロサービスのブートストラップ資格情報 | — | `GET /application/applications/{id}/bootstrapUser` | ◎ | △（秘密） | ◎ |
| P-09 | **マイクロサービス設定（アプリケーションオプション）** | マイクロサービスが読む設定値 | Ecosystem > Microservices | `GET/PUT /application/currentApplication/settings`（自身から）、実体は `/tenant/options` の該当カテゴリ | ◎ | △ | ◎ |
| P-10 | 現アプリの購読者 | サブテナント購読一覧 | — | `GET /application/currentApplication/subscriptions` | ◎ | ◎ | △ |
| P-11 | **フィーチャートグル** | 機能フラグ | Settings > Feature toggles | `GET /features`, `GET /features/{featureKey}`, `GET/PUT/DELETE /features/{featureKey}/by-tenant`, `.../by-tenant/{tenantId}` | ◎ | ◎ | ◎ |

**マイクロサービス設定カテゴリの解決順**: マイクロサービスが `/tenant/options` から自分の設定を読むとき、カテゴリは以下の優先順で決まります。

1. マイクロサービスマニフェストの `settingsCategory`
2. マイクロサービスのコンテキストパス
3. マイクロサービス名

このカテゴリが一致するときのみ、`credentials.` 接頭辞付きの暗号化オプションが**そのマイクロサービスのサービスユーザーに対して復号**されます。一致しなければ `"<<Encrypted>>"` が返ります。

**必要権限**: 閲覧 = `Application management` の READ、管理 = 同 ADMIN

**自動生成されるアラーム**: `c8y_Application_Down`（インスタンスなし）、`c8y_Application_Unhealthy`（部分障害）

出典: [Ecosystem](https://cumulocity.com/docs/standard-tenant/ecosystem/), [Applications API](https://cumulocity.com/api/core/#tag/Applications)

---

## 6. ルール・ストリーミング分析

### 6.1 スマートルール

| 項目 | 内容 |
|---|---|
| **前提** | テナントが **Smartrule マイクロサービス**および **Apama-ctrl マイクロサービス**に購読していること |
| **必要ロール** | `CEP Manager`（＋権限タイプ `Global smart rules` / `CEP management`） |
| GUI | Cockpit > Smart rules（グローバル）、またはグループ／デバイス配下（ローカル） |
| 格納形態 | **インベントリ内の managed object**。Java SDK の `SmartRuleRepresentation` はフラグメント `c8y.SmartRuleRepresentation` を表し、ruleTemplateName、ID、CEP モジュール ID、configuration、有効/無効ソース等を持つ |
| グローバル vs ローカル | グローバル＝インベントリ全体を監視（作成時に対象を選ばなければ「全アセットで既定有効」、選べば「全アセットで既定無効」）。ローカル＝作成元のアセットのみ（設定により子アセットにも波及） |
| 自動化 | **○（要検証）** — 専用 REST エンドポイントは公式ドキュメントに明記されていない。managed object として `POST /inventory/managedObjects` 経由で投入する経路が現実的 |
| エクスポート | **△** — GUI のエクスポート機能は無い。`GET /inventory/managedObjects?fragmentType=...` で取得する方式 |
| Edge | **◎**（Streaming Analytics が Edge に同梱） |

**標準スマートルールテンプレート（全 11 種）**:

1. On alarm send SMS — アラーム発生時に指定番号へ SMS
2. On alarm send email — アラーム発生時に指定アドレスへメール
3. On alarm escalate it — 連鎖エスカレーション（メール/SMS）
4. On alarm duration increase severity — 指定時間継続で重大度を上げる
5. On geofence create alarm — ジオフェンス越境でアラーム
6. On geofence send email — ジオフェンス退出でメール
7. Calculate energy consumption — メーター値から消費量を算出
8. On missing measurements create alarm — 計測が途絶えたらアラーム
9. On alarm execute operation — アラームに応じてデバイスへオペレーション送信
10. On measurement threshold create alarm — red/yellow 閾値でアラーム
11. On measurement explicit threshold create alarm — 明示レンジでアラーム

ルールパラメータには変数 `#{fieldName}` 形式を使えます。

> ⚠️ 公式は *"this list might differ based on your installation."* と注記。**Edge 実機で実際のテンプレート一覧を確認してください。**

出典: [Smart rules](https://cumulocity.com/docs/cockpit/smart-rules/), [Smart rules collection](https://cumulocity.com/docs/cockpit/smart-rules-collection/), [Smart rules (NEW) plugin](https://cumulocity.com/docs/streaming-analytics/smart-rules-plugin/)

### 6.2 ⭐ Analytics Builder モデル — エクスポート／インポートが公式サポート

| 項目 | 内容 |
|---|---|
| 格納形態 | **Cumulocity インベントリに JSON 形式で格納** |
| GUI | Streaming Analytics > Model manager（Models / Samples タブ） |
| **エクスポート** | **◎** 各モデルを個別に JSON ダウンロード可。公式記述: *"You can download each model... if you want to transfer a model from the current Cumulocity tenant to a different tenant."* クリップボードコピーも可 |
| **インポート** | **◎** ダウンロード済み JSON のファイルアップロード、または JSON コードの直接ペースト |
| 状態 | Active（デプロイ済み） / Inactive |
| モード | Draft（開発） / Test（単一デバイス、出力は保存のみ） / Simulation（履歴データ再生、単一デバイス） / Production（実運用、出力をデバイスへ送信） |
| 自動化 | **○** — 専用 REST API は公式ドキュメントに明記されていないが、インベントリの managed object として扱える |
| Edge | **◎** 公式に *"Streaming Analytics engine for real-time local data analysis including the Cumulocity Analytics Builder"* が Edge に含まれると明記 |

> **これは「基盤として提供するデータ分析モデル」の配布に最適な経路です。** リファレンス環境で GUI からモデルを作成 → JSON ダウンロード → Git 管理 → 新規 Edge へアップロード、という半自動フローがそのまま成立します。

出典: [Analytics Builder](https://cumulocity.com/docs/streaming-analytics/analytics-builder/)

### 6.3 ⭐ EPL Apps — 完全な REST API あり

| 操作 | メソッド | エンドポイント |
|---|---|---|
| 一覧（メタデータ） | GET | `/service/cep/eplfiles` |
| 一覧（ソース込み） | GET | `/service/cep/eplfiles?contents=true` |
| 個別取得 | GET | `/service/cep/eplfiles/{id}` |
| **作成** | POST | `/service/cep/eplfiles` |
| **更新** | PUT | `/service/cep/eplfiles/{id}` |
| 削除 | DELETE | `/service/cep/eplfiles/{id}` |

全リクエストに `Accept: application/json` ヘッダと認証が必要。

| 項目 | 内容 |
|---|---|
| 格納形態 | 単一の `*.mon` ファイル。アクティブ化時に衝突回避のため一意のパッケージ名が割り当てられる |
| GUI | Streaming Analytics > EPL Apps（**`CEP Manager` 権限が必要**） |
| ローカル開発 | Apama Extension for VS Code + Git リポジトリテンプレート → 「Import EPL」ボタンで取り込み |
| エクスポート | **◎** アクションメニューから `*.mon` ファイルとしてダウンロード |
| 自動化 | **◎** 上記 REST API で完全自動化可 |
| Edge | **◎**（Streaming Analytics 同梱）※ページ上での明示的確認は無いため実機確認推奨 |

出典: [EPL apps](https://cumulocity.com/docs/streaming-analytics/epl-apps/)

---

## 7. データ保持・デバイスプロトコル

### 7.1 ⭐ リテンションルール — スキーマ確定済み

| 操作 | エンドポイント |
|---|---|
| 一覧／作成 | `GET/POST /retention/retentions` |
| 個別取得／更新／削除 | `GET/PUT/DELETE /retention/retentions/{id}` |

GUI: **Management メニュー > Retention rules**（権限タイプ `Retention rules`）

**`retentionRule` スキーマ（Core OAS より・完全）**:

| フィールド | 型 / 値域 | 既定 | 説明 |
|---|---|---|---|
| `dataType` | `ALARM` \| `AUDIT` \| `BULK_OPERATION` \| `EVENT` \| `MEASUREMENT` \| `OPERATION` \| `*` | `*` | 適用対象データ型 |
| `fragmentType` | string | `*` | 適用フラグメント型。**EVENT / MEASUREMENT / OPERATION / BULK_OPERATION でのみ使用** |
| `type` | string | `*` | 適用タイプ。**ALARM / AUDIT / EVENT / MEASUREMENT でのみ使用** |
| `source` | string | `*` | 適用ソース（デバイス ID）。全データ型で使用。**直下のデバイスデータのみに作用し、子デバイスやグループには波及しない** |
| `maximumAge` | integer | — | 最大保持日数（**上限 10 年**） |
| `editable` | boolean | `true` | 編集可否。**management テナントのみが変更可** |
| `id` | string (readOnly) | — | 一意識別子 |

**既定の保持期間**: 全履歴データは **60 日**後に削除（プラットフォーム管理者がシステム設定で変更可）。

**制約**:
- ⚠️ **アラームはステータスが `CLEARED` のもののみ削除されます。**
- リテンションルールは **files repository のファイルには適用されません。**

`{"dataType":"ALARM","fragmentType":"*","type":"*","source":"*","maximumAge":20,"editable":true}` のような JSON をそのまま `POST /retention/retentions` に流せます。**エクスポート ◎ / 自動化 ◎ / Edge ◎。**

出典: [Managing data](https://cumulocity.com/docs/standard-tenant/managing-data/), [Retention rules API](https://cumulocity.com/api/core/#tag/Retention-rules)

### 7.2 ファイルリポジトリ

GUI: Management メニュー > Files repository。権限は READ（自分のファイルの閲覧・削除）/ CREATE（アップロード）/ ADMIN（所有者を問わず全管理）。API は `GET/POST /inventory/binaries`, `GET/PUT/DELETE /inventory/binaries/{id}`。

> アプリケーションアーカイブはこの画面からは削除できません。

### 7.3 デバイスプロトコル／デバイスタイプ

GUI: ナビゲータ > **Device types > Device protocols**。プロトコル種別（Modbus、CANOpen、LoRa など）、デバイス名、リソース数が一覧表示されます。

| 機能 | 可否 |
|---|---|
| 追加（Add device protocol：名前＋任意の説明） | ◎ |
| **定義済みプロトコルからのインポート** | ◎ |
| **ファイルからのインポート** | ◎ |
| 編集 | ◎ |
| 削除 | ◎ |
| **ファイルシステムへのエクスポート** | **◎** |

> **これも「基盤として提供する設定」の配布に使える経路です。** リファレンス環境でプロトコルを定義 → ファイルにエクスポート → Git 管理 → 新規 Edge へインポート。

格納形態はデータベース（managed object）ですが、公式ページは managed object の型名や REST エンドポイントを明示していません。`GET /inventory/managedObjects` で実体を確認してください。

**Edge でのプロトコル対応**: Edge vs Cloud 比較表で **OPC UA** と **Cloud Field Bus**（Modbus 系）が「Included」。**MQTT / REST はネイティブ対応**。LWM2M・SNMP については Edge introduction ページに記載がなく、**別途確認が必要**です。

出典: [Managing device types](https://cumulocity.com/docs/device-management-application/managing-device-types/), [SNMP](https://cumulocity.com/docs/2024/protocol-integration/snmp/)

---

## 8. ブランディング・UI カスタマイズ・ダッシュボード

### 8.1 ブランディング

> ⚠️ **ブランディングは Enterprise テナントの機能です。** 公式は「Enterprise tenant の Administration アプリケーションで利用可能」と記載。**Edge はマルチテナンシー非対応（＝ Enterprise 機能なし）のため、Edge での利用可否は要確認**です。

GUI: Administration > Settings > Branding（Enterprise tenant）

| タブ | 設定項目 |
|---|---|
| **Generic** | タイトル、ファビコン（ICO）、ダークテーマ切替、タイポグラフィ（base font / headings font / navigator font / リモートフォントリンク）、Cookie バナー（タイトル、本文、プライバシーポリシーリンク、ポリシーバージョン） |
| **Light / Dark theme**（両者同一項目） | ロゴ（brand logo / navigator logo：PNG, SVG, JPG＋高さ指定）、ブランドカラー（primary / light / dark ＋生成される 8 階調、HEX/RGB/RGBA）、ステータスカラー（info / warning / danger / success 各 default/light/dark）、Generic（body background / text / link color / button border-radius）、Action bar / Main header / Navigator / Right drawer の背景・テキスト・アクセント色 |
| **Custom CSS** | 独自 CSS の提供 |
| **Advanced branding** | **標準フォームで扱えない `ApplicationOptions` の JSON 直接編集** ← 自動化の切り口 |

- **ブランディングバリアント**: 複数のバリアントを保持でき、1 つが「グローバルブランディング」として全アプリ・全サブテナントに既定適用される。アプリ個別のバリアントで上書き可
- **エクスポート**: 削除前に「既存バリアントをエクスポートできる」と記載あり。技術的な形式・API は公式に明記されていない → **要検証**
- 格納形態: *"Many settings configured in the UI, such as branding and feature toggles, are stored as key-value pairs and can be managed via the Tenant Options API."* → **tenant options として `GET /tenant/options` で抽出できる可能性が高い（要実機確認）**
- 「Open preview」でリアルタイムプレビュー可

出典: [Customizing your platform](https://cumulocity.com/docs/enterprise-tenant/customization/), [Application configuration](https://cumulocity.com/docs/web/application-configuration/)

### 8.2 ⭐ Cockpit ダッシュボード — JSON エクスポート／インポートが公式サポート

| 項目 | 内容 |
|---|---|
| 格納形態 | **managed object**（`c8y_Dashboard` フラグメント） |
| **エクスポート** | **◎** ダッシュボードを **JSON ファイル**にエクスポート（特定ウィジェットのインポートを支える付加データ込み） |
| **インポート** | **◎** JSON ファイルからインポート |
| 用途 | **同一テナント内の同種アセット間だけでなく、異なるアセット型・異なるテナント間でも共有可能**。ただし group ↔ device のように型が異なる場合はインポート後のレビュー推奨 |
| 高度編集 | 組み込みコードエディタで**ダッシュボードを JSON として直接編集可**（ContextDashboard とウィジェット設定インタフェースの知識が必要） |
| 自動化 | ◎（managed object として `/inventory/managedObjects` 経由） |
| Edge | ◎（Cockpit 同梱） |

> **これも基盤標準ダッシュボードの配布に直接使えます。** スキーマ検証とアセットマッピングを伴う import/export 機能が提供されています。

出典: [Working with dashboards](https://cumulocity.com/docs/cockpit/working-with-dashboards/)

---

## 9. 設定のエクスポートと自動注入の手段

### 9.1 手段の比較

| 手段 | 対象範囲 | エクスポート | 投入 | CI/CD 適性 | 備考 |
|---|---|---|---|---|---|
| **Core REST API 直叩き** | テナント業務設定全般 | ◎ | ◎ | ◎ | 最も確実。curl/PowerShell でスクリプト化 |
| **go-c8y-cli (`c8y`)** | 同上（ほぼ全リソースに専用動詞） | ◎ | ◎ | ◎ | **本命**。§9.2 |
| **`c8yedge` CLI** | Edge インフラ層のみ | ○ | ◎ | ◎ | 2025/2026 版 |
| **`kubectl` + Edge CR** | Edge インフラ層のみ | ◎ (`-o yaml`) | ◎ (`apply`) | ◎ | 2025/2026 版。宣言的 |
| **`/edge/*` REST API** | Edge インフラ層のみ | ◎ | ◎ | ◎ | VM アプライアンス版（〜2024）限定 |
| **GUI の Download configuration** | Edge security 設定 | ◎ | — | — | §1.3。JSON がそのまま POST に使える |
| **GUI の JSON ダウンロード** | Analytics Builder モデル、ダッシュボード | ◎ | ◎ | ○ | §6.2 / §8.2 |
| **ファイルエクスポート** | デバイスプロトコル、EPL apps (`*.mon`) | ◎ | ◎ | ○ | §7.3 / §6.3 |
| **cumulocity-migration-tool** | アプリ、ダッシュボード、グループ、デバイス、シミュレータ、**スマートルール**、画像、managed object | ○ | ○ | △ | [公式組織リポジトリ](https://github.com/Cumulocity-IoT/cumulocity-migration-tool)。テナント間移行用 |

### 9.2 ⭐ go-c8y-cli — 設定 bootstrap の本命

公式コミュニティツール（[goc8ycli.netlify.app](https://goc8ycli.netlify.app/docs/) / [reubenmiller/go-c8y-cli](https://github.com/reubenmiller/go-c8y-cli)）。

**確認済みコマンド名前空間**: `tenantoptions`, `users`, `userroles`, `usergroups`, `applications`, `microservices`, `retentionrules`, `tenants`, `systemoptions`, `databroker`, `sessions`, `inventory`, `api`
（`inventoryroles` と `smartrules` は index ページ上では未確認 → 汎用の `c8y api` で代替可能）

**jsonnet テンプレート + パイプ入力による一括投入**（＝「JSON で設定を保持して注入する」の実装形）:

```bash
# 変数付きテンプレートから作成
c8y inventory create --template "./device.jsonnet" --templateVars "type=macOS,fragment=customer_Agent"
```

```bash
# JSON ファイルの各行をテンプレート経由で一括投入
cat input.json | c8y inventory create --template "{ type: input.value.type, properties: input.value.ramSize }"
```

```bash
# テナントオプションのバルク更新
c8y tenantoptions updateBulk --category configuration --data '{"key1":"val1","key2":"val2"}'
```

- `--template`: jsonnet ファイルパスまたはインライン文字列。CLI パラメータは後からマージされる
- `--templateVars`: `var(name, [default])` で参照する実行時変数
- パイプ入力: `input.index`（イテレータ位置）、`input.value`（現在の要素）
- 組み込み関数: `_.Int()`, `_.Float()`, `_.Bool()`, `_.Name()`, `_.Password()`, `_.Now()`, `_.Date()`, `_.Duration()`, `_.Select()`, `_.SelectMerge()`, `_.UrlEncode()`
- 概念機能: 出力テンプレート、コマンドチェイン、**dry-run**、フィルタ／ページング、エラーハンドリング、拡張機能
- v2.54.0 以降で **SSO（Authorization Code / Device Flow）認証**に対応

**「既存設定を吸い出して再投入」の実装イディオム**（クローンパターン）:

```bash
c8y devices list --pageSize 1 --select '**,!id,!lastUpdated' | c8y devices create --name "myclone" --template "input.value"
```

`--select '**,!id,!lastUpdated'` で ID や更新日時などの再投入してはいけないフィールドを除外する — この考え方が**あらゆる設定エクスポートの基本形**になります。

出典: [go-c8y-cli docs](https://goc8ycli.netlify.app/docs/), [Templates](https://goc8ycli.netlify.app/docs/concepts/templates/), [c8y command index](https://goc8ycli.netlify.app/docs/cli/c8y/)

### 9.3 ⚠️ 「テナント丸ごとエクスポート／インポート」は存在しない

Cumulocity には **tenant 全体を 1 コマンドでエクスポート／インポートする公式機能はありません。** リソース種別ごとに上記の手段を組み合わせる必要があります。本書の一覧はそのための対象リストです。

### 9.4 推奨: リファレンステナント差分方式

テナントオプションの網羅的な全キー一覧が公式に存在しない以上、以下が最も現実的です。

1. まっさらな Edge をインストール → `GET /tenant/options`、`GET /tenant/loginOptions`、`GET /retention/retentions`、`GET /user/{t}/groups` 等を全部取得 → **ベースライン JSON** として保存
2. GUI で「基盤として提供したい設定」を人手で一通り設定
3. 同じ GET 群を再実行 → **差分を取る** → これが投入すべき設定の実体
4. 差分を jsonnet テンプレート化し、`credentials.*` と ID・タイムスタンプ系フィールドを除外
5. go-c8y-cli で新規 Edge へ投入 → 手順 1 と同じ GET で**投入後アサート**

**これで「手作業で設定用 JSON をいじる」必要がほぼ無くなります。** ご要望の「GUI から設定値を抽出」は、この差分方式で実現するのが最も確実です。

---

## 10. 推奨セットアップ手順（実行順）

| 段 | 工程 | 対象層 | 手段 | 冪等性 |
|---|---|---|---|---|
| 1 | Edge インストール | L0 | `c8yedge install`（2025/2026）または `POST /edge/install`（〜2024） | × |
| 2 | ドメイン・ライセンス・TLS | L0 | `c8yedge config --set/--set-file`（**ドメインとライセンスは必ず同時**） | ◎ |
| 3 | ネットワーク・ホスト名・時刻同期 | L0 | `/edge/configuration/*`（VM 版）または Edge CR | ◎ |
| 4 | OS/K8s セキュリティ | L0 | `POST /edge/configuration/security`（⚠️ `audit_logging_enabled` は不可逆） | ◎ |
| 5 | マイクロサービスホスティング有効化 | L0 | `POST /edge/configuration/microservices` `{"enabled":true}`（**10〜15 分かかる**） | ◎ |
| 6 | 【Management テナント】メールサーバー・パスワードリセットテンプレート | L1 | `/tenant/options`（Management テナント側） | ◎ |
| 7 | 【Edge テナント】テナントオプション一括 | L1 | `PUT /tenant/options/{category}` × カテゴリ数 | ◎ |
| 8 | 認証基本設定・パスワードポリシー・TFA | L1 | `PUT /tenant/loginOptions/{typeOrId}`, `PUT /tenant/tenants/{t}/tfa` | ◎ |
| 9 | SSO 設定 | L1 | `POST/PUT /tenant/loginOptions` + accessMappings + inventoryAccessMappings | ◎ |
| 10 | グローバルロール・インベントリロール定義 | L1 | `POST /user/{t}/groups`, `POST /user/inventoryroles` | ○（要 upsert 実装） |
| 11 | サービスユーザー・運用ユーザー作成＋ロール割当 | L1 | `POST /user/{t}/users` → `/roles` `/roles/inventory` | ○ |
| 12 | アプリ／マイクロサービス購読 | L1 | `POST /tenant/tenants/{t}/applications` | ◎ |
| 13 | フィーチャートグル | L1 | `PUT /features/{key}/by-tenant` | ◎ |
| 14 | リテンションルール | L1 | `POST /retention/retentions`（既存 60 日既定の扱いに注意） | ○ |
| 15 | デバイスプロトコル | L1 | ファイルインポート（GUI）または `/inventory/managedObjects` | ○ |
| 16 | EPL apps | L1 | `POST /service/cep/eplfiles` | ◎ |
| 17 | Analytics Builder モデル | L1 | JSON アップロード（GUI）または inventory | ○ |
| 18 | スマートルール | L1 | `/inventory/managedObjects`（要検証） | ○ |
| 19 | Cockpit ダッシュボード | L1 | JSON インポート（GUI）または inventory | ○ |
| 20 | ブランディング | L1 | tenant options（要検証）／Advanced branding の JSON | △ |
| 21 | **投入後アサート** | 両層 | 手順 7〜19 と同じ GET 群 ＋ `GET /tenant/system/options` | ◎ |

> **⚠️ アプライアンス更新後の再適用**: `manage-edge` に、Edge アプライアンス更新時に一部設定が上書きされ再適用が必要との運用注記があります。上記手順は**更新のたびに再実行できる冪等スクリプト**として実装してください。「冪等性 ×」の段のみ切り離す構成が安全です。

---

## 11. 未確定・要検証事項（実機での確認推奨）

| # | 項目 | なぜ未確定か | 確認方法 |
|---|---|---|---|
| V-01 | **導入する Edge のバージョン／導入経路** | VM アプライアンス版（〜2024）と Kubernetes 版（2025/2026）で L0 の自動化方式が根本的に異なる。**これが決まらないと §1 の設計が確定しない** | 調達仕様の確認。最優先 |
| V-02 | スマートルールの REST エンドポイント | 公式ドキュメントに専用エンドポイントの記載なし。`/service/smartrule/smartrules` の存在は未確認 | 実機で GUI からルール作成 → ブラウザ DevTools のネットワークタブで実 API を観測 |
| V-03 | Analytics Builder モデルの REST エンドポイント | 同上（JSON ダウンロード/アップロードは確認済み） | 同上 |
| V-04 | ブランディングの tenant options キー名 | 「tenant options として保存される」との記述はあるがキー名は非公開 | §9.4 の差分方式で特定 |
| V-05 | **Edge でのブランディング利用可否** | ブランディングは Enterprise 機能。Edge はマルチテナンシー非対応 | 実機で Administration > Settings にタブが出るか確認 |
| V-06 | Edge での LWM2M / SNMP 対応 | Edge vs Cloud 表に OPC UA と Cloud Field Bus はあるが LWM2M/SNMP の記載なし | 実機で Ecosystem のマイクロサービス一覧を確認 |
| V-07 | Edge の実際のスマートルールテンプレート一覧 | 公式が *"might differ based on your installation"* と注記 | 実機で Cockpit > Smart rules のテンプレート一覧を確認 |
| V-08 | Management テナント vs Edge テナントの設定管轄マッピング | メールサーバー = Management は確認済みだが、全項目の管轄は未整理 | 実機で両テナントの `GET /tenant/options` を比較 |
| V-09 | テナントオプションの完全なキー一覧 | 公式に網羅的一覧が存在しない | §9.4 の差分方式（これが唯一の実用解） |
| V-10 | EPL Apps の Edge 対応 | Streaming Analytics は同梱だが EPL Apps ページに Edge 明記なし | 実機で Streaming Analytics アプリに EPL Apps ページがあるか確認 |

---

## 付録A: 本書で確認した Core REST API エンドポイント全一覧

`https://cumulocity.com/api/core/dist/c8y-oas.yml`（`info.version: Latest`、2026-08-06 取得）から機械的に抽出。設定投入に関係するもののみ抜粋。

```
/tenant                                   /user
/tenant/currentTenant                     /user/currentUser
/tenant/tenants                           /user/currentUser/password
/tenant/tenants/{tenantId}                /user/currentUser/totpSecret
/tenant/tenants/{tenantId}/applications   /user/currentUser/totpSecret/activity
/tenant/tenants/{tenantId}/applications/{applicationId}
/tenant/tenants/{tenantId}/applications/restricted-roles
/tenant/tenants/{tenantId}/applications/restricted-roles/{roleId}
/tenant/tenants/{tenantId}/tfa            /user/currentUser/totpSecret/verify
/tenant/tenants/{tenantId}/trusted-certificates
/tenant/tenants/{tenantId}/trusted-certificates/bulk
/tenant/tenants/{tenantId}/trusted-certificates/{fingerprint}
/tenant/trusted-certificates/verify-cert-chain
/tenant/trusted-certificates/settings/crl
/tenant/options                           /user/roles
/tenant/options/{category}                /user/roles/{name}
/tenant/options/{category}/{key}          /user/inventoryroles
/tenant/options/{category}/{key}/editable /user/inventoryroles/{id}
/tenant/system/options                    /user/devicePermissions/{id}
/tenant/system/options/{category}/{key}   /user/{tenantId}/users
/tenant/loginOptions                      /user/{tenantId}/users/{userId}
/tenant/loginOptions/{typeOrId}           /user/{tenantId}/users/{userId}/roles
/tenant/loginOptions/{typeOrId}/restrict  /user/{tenantId}/users/{userId}/roles/{roleId}
/tenant/loginOptions/{typeOrId}/accessMappings
/tenant/loginOptions/{typeOrId}/accessMappings/{id}
/tenant/loginOptions/{typeOrId}/inventoryAccessMappings
/tenant/loginOptions/{typeOrId}/inventoryAccessMappings/{id}
/tenant/oauth                             /user/{tenantId}/users/{userId}/roles/inventory
/tenant/oauth/token                       /user/{tenantId}/users/{userId}/roles/inventory/{id}
/tenant/oauth/certificate                 /user/{tenantId}/users/{userId}/groups
/tenant/statistics                        /user/{tenantId}/users/{userId}/tfa
/tenant/statistics/summary                /user/{tenantId}/groups
/tenant/statistics/allTenantsSummary      /user/{tenantId}/groups/{groupId}
/tenant/statistics/files                  /user/{tenantId}/groups/{groupId}/roles
/tenant/statistics/device/{tenantId}/daily/{date}
/tenant/statistics/device/{tenantId}/monthly/{date}
                                          /user/{tenantId}/groups/{groupId}/roles/{roleId}
/retention/retentions                     /user/{tenantId}/groups/{groupId}/users
/retention/retentions/{id}                /user/{tenantId}/groups/{groupId}/users/{userId}
                                          /user/{tenantId}/groupByName/{groupName}
/features                                 /user/{tenantId}/userByName/{username}
/features/{featureKey}                    /user/logout
/features/{featureKey}/by-tenant          /user/logout/{tenantId}/allUsers
/features/{featureKey}/by-tenant/{tenantId}

/application                              /inventory
/application/applications                 /inventory/managedObjects
/application/applications/{id}            /inventory/managedObjects/{id}
/application/applications/{id}/binaries   /inventory/binaries
/application/applications/{id}/binaries/{binaryId}
/application/applications/{id}/bootstrapUser
/application/applications/{id}/clone      /identity/externalIds/{type}/{externalId}
/application/applications/{id}/versions   /identity/globalIds/{id}/externalIds
/application/applications/{id}/versions/{version}
/application/applicationsByName/{name}    /audit/auditRecords
/application/applicationsByOwner/{tenantId}
/application/applicationsByTenant/{tenantId}
/application/applicationsByUser/{username}
/application/currentApplication           /notification2/subscriptions
/application/currentApplication/settings  /notification2/token
/application/currentApplication/subscriptions

/devicecontrol/newDeviceRequests          /certificate-authority
/devicecontrol/bulkNewDeviceRequests      /certificate-authority/renew
/devicecontrol/deviceCredentials          /.well-known/est/simpleenroll
/devicecontrol/bulkoperations             /.well-known/est/simplereenroll
```

**Core OAS 以外のサービス API**:
- `GET/POST/PUT/DELETE /service/cep/eplfiles[/{id}]` — EPL Apps（§6.3）

## 付録B: 出典一覧

**Edge**
- [Edge introduction (2026)](https://cumulocity.com/docs/2026/edge/edge-introduction/)
- [Installing Edge (2026)](https://cumulocity.com/docs/2026/edge/installing-edge/)
- [Manage Edge (2026)](https://cumulocity.com/docs/2026/edge/manage-edge/) / [(2025, Kubernetes)](https://cumulocity.com/docs/2025/edge-kubernetes/manage-edge/)
- [Edge custom resource definition (2026)](https://cumulocity.com/docs/2026/edge/edge-custom-resource-definition/)
- [Connecting Edge to cloud (2026)](https://cumulocity.com/docs/2026/edge/connecting-edge-to-cloud/)
- [Edge configuration (2024, VM appliance)](https://cumulocity.com/docs/2024/edge/edge-configuration/)
- [Cumulocity Edge OpenAPI](https://cumulocity.com/api/edge/) / [raw spec](https://cumulocity.com/api/edge/10.18.0/dist/c8y-edge-oas.json)

**Core API**
- [Cumulocity OpenAPI Specification](https://cumulocity.com/api/core/) / [raw spec (c8y-oas.yml)](https://cumulocity.com/api/core/dist/c8y-oas.yml)
- [Options API](https://cumulocity.com/api/core/#tag/Options)
- [Login options API](https://cumulocity.com/api/core/#operation/putAccessLoginOptionResource)
- [Retention rules API](https://cumulocity.com/api/core/#tag/Retention-rules)
- [Applications API](https://cumulocity.com/api/core/#tag/Applications)

**認証**
- [Basic settings](https://cumulocity.com/docs/authentication/basic-settings/)
- [Single sign-on](https://cumulocity.com/docs/authentication/sso/)
- [Two-factor authentication](https://cumulocity.com/docs/authentication/tfa/)

**管理**
- [Managing permissions](https://cumulocity.com/docs/standard-tenant/managing-permissions/)
- [Managing data](https://cumulocity.com/docs/standard-tenant/managing-data/)
- [Ecosystem](https://cumulocity.com/docs/standard-tenant/ecosystem/)
- [Changing settings](https://cumulocity.com/docs/standard-tenant/changing-settings/)
- [Customizing your platform (Enterprise)](https://cumulocity.com/docs/enterprise-tenant/customization/)
- [Managing tenants (Enterprise)](https://cumulocity.com/docs/enterprise-tenant/managing-tenants/)

**ルール・分析**
- [Smart rules](https://cumulocity.com/docs/cockpit/smart-rules/) / [Smart rules collection](https://cumulocity.com/docs/cockpit/smart-rules-collection/) / [Smart rules (NEW) plugin](https://cumulocity.com/docs/streaming-analytics/smart-rules-plugin/)
- [Analytics Builder](https://cumulocity.com/docs/streaming-analytics/analytics-builder/)
- [EPL apps](https://cumulocity.com/docs/streaming-analytics/epl-apps/)
- [Working with dashboards](https://cumulocity.com/docs/cockpit/working-with-dashboards/)

**デバイス**
- [Managing device types](https://cumulocity.com/docs/device-management-application/managing-device-types/)
- [SNMP (2024)](https://cumulocity.com/docs/2024/protocol-integration/snmp/)

**ツール**
- [go-c8y-cli documentation](https://goc8ycli.netlify.app/docs/) / [Templates](https://goc8ycli.netlify.app/docs/concepts/templates/) / [Command index](https://goc8ycli.netlify.app/docs/cli/c8y/)
- [reubenmiller/go-c8y-cli (GitHub)](https://github.com/reubenmiller/go-c8y-cli)
- [Cumulocity-IoT/cumulocity-migration-tool](https://github.com/Cumulocity-IoT/cumulocity-migration-tool)
- [Cumulocity-IoT/cumulocity-provision-sso](https://github.com/Cumulocity-IoT/cumulocity-provision-sso)
