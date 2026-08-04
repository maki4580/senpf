# Cumulocity 設定項目一覧

作成日: 2026-08-04 ／ 詳細な背景・注入アーキテクチャ・制約は [cumulocity-config-definition.md](cumulocity-config-definition.md) を参照

## 本書の読み方

### 列の定義

| 列 | 内容 |
|----|------|
| 設定項目 | 個別の設定名。末尾のマークは検証状態（凡例は下記） |
| 概要 | その設定が何を制御するか |
| 設定単位 | 設定値が効く範囲（テナント全体 / ユーザー / デバイス など） |
| 設定方法 | UI / REST API / Edge CR / CSV / CLI などの投入経路 |
| UIパス | Cumulocity UI 上の到達経路。UIから設定できないものは「—」 |
| 既定値・許容値 | 公式ドキュメントに記載のある具体値。記載がないものは「—」 |
| Export | 設定値をファイルへ出力できるか（凡例は下記） |
| Import | ファイル等から設定値を投入できるか（凡例は下記） |
| 変更時の影響 | 変更が及ぶ範囲、不可逆性、再起動/再ログインの要否など |

### Export / Import の記号

| 記号 | 意味 |
|------|------|
| ○ | **公式のファイル入出力機構がある**（UIのエクスポート/インポート、YAML/JSON/CSVファイル） |
| △ | 公式のファイル機構はないが、**REST APIで値を取得（GET）/ 投入（POST・PUT）できる** |
| ✗ | 手段が確認できていない（未検証を含む） |

### 検証状態の凡例

| マーク | 意味 |
|-------|------|
| ✅ | deep-research の反証検証（3票）を通過した確定事項 |
| 📄 | 公式ドキュメントを直接取得して確認（2026-08-04時点）。反証検証は未実施 |
| ⚠️ | 一次ソースからの引用付き抽出のみ。実機確認を推奨 |

> **「変更時の影響」列について**: 公式ドキュメントに明記のある挙動（変更不可・伝播しない・ダウンタイム等）を優先して記載し、明記がない場合はその設定の適用範囲から導かれる影響範囲を書いている。実運用前に実機で確認すること。

---

## 0. 全体サマリ（カテゴリ単位）

| # | カテゴリ | 設定単位 | 主な設定方法 | Export | Import | 適用環境 |
|---|---------|---------|-------------|--------|--------|---------|
| 1 | Edge Custom Resource | Edgeデプロイ全体 | YAML + kubectl | ○ | ○ | Edgeのみ |
| 2 | Application 設定 | テナント全体 | UI | △ | △ | 共通 |
| 3 | Properties library | テナント全体 | UI | △ | △ | 共通 |
| 4 | SMS provider | テナント全体 | UI | ✗ | ✗ | 共通 |
| 5 | Connectivity | テナント全体 | UI | ✗ | ✗ | 共通 |
| 6 | Localization | テナント全体 | UI | ✗ | ✗ | 共通 |
| 7 | Feature toggles | テナント / サブテナント | UI | ✗ | ✗ | 共通 |
| 8 | テナントオプション | テナント全体 | REST / UIプラグイン | △ | △ | 共通 |
| 9 | 権限・ロール | ロール単位 | UI / REST | △ | △ | 共通 |
| 10 | Retention rules | データ種別単位 | UI / REST | △ | △ | 共通 |
| 11 | 認証（Basic settings） | テナント全体 | UI / REST | △ | △ | 共通 |
| 12 | SSO / Single sign-on | テナント全体 | UI / REST | △ | △ | 共通 |
| 13 | ブランディング | テナント全体 | UI | **○** | **○** | クラウドEnterprise |
| 14 | Enterprise Configuration（メール等） | テナント全体 | UI | ✗ | ✗ | クラウドEnterprise |
| 15 | サブテナント / テナントポリシー | サブテナント | UI / REST | △ | △ | クラウドEnterprise |
| 16 | Data broker | コネクタ単位 | UI | ✗ | ✗ | 共通（要feature-broker） |
| 17 | Usage statistics / 課金 | テナント / マイクロサービス | 画面参照 / マニフェスト | △ | ✗ | クラウド中心 |
| 18 | アラームマッピング | アラームタイプ単位 | UI | ✗ | ✗ | 共通 |
| 19 | ユーザー管理 | ユーザー単位 | UI / REST | △ | △ | 共通 |
| 20 | デバイス一括登録 | デバイス単位 | **CSV** / REST | ✗ | **○** | 共通 |
| 21 | ダッシュボード | ダッシュボード単位 | UI / REST | **○** | **○** | 共通 |
| 22 | Analytics Builder モデル | モデル単位 | UI | **○** | **○** | 共通 |
| 23 | アプリケーション / マイクロサービス | アプリ単位 | ZIP / REST / CLI | ○ | ○ | 共通（Edgeは要オプトイン） |
| 24 | デバイス可用性・監視 | デバイス単位 | UI / REST | △ | △ | 共通 |

---

## 1. フェーズ0: Edge Custom Resource（Edgeデプロイ層）

**共通事項** — Export: `kubectl get --namespace=c8yedge edge/c8yedge -o yaml > edge.yaml` ／ Import: `kubectl apply -f edge.yaml`（Gitでのバージョン管理が可能／GitOps対応）。UIパスは全項目「—（UIなし）」。設定単位は全項目「Edgeデプロイ全体」。

### 1.1 spec 直下（必須）

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | Export | Import | 変更時の影響 |
|---------|------|---------|--------------|--------|--------|-------------|
| `version` ✅ | インストールするEdgeバージョン | CR / `c8yedge install --version` | `2025` または `2025.0.1` 形式 | ○ | ○ | パッチで変更するとアップグレード実行。recreate戦略のため**短時間のダウンタイム**が発生 |
| `domain` ✅ | EdgeのFQDN | CR / `c8yedge config --set domain=` | 英数字をドット/ハイフン連結・255文字以内・セグメント1〜63文字・TLD2〜6文字・Punycode必須 | ○ | ○ | **ライセンスキーと紐づくため同時変更が必須**。アクセスURLが変わる |
| `licenseKey` ✅ | ドメインに対するライセンス | CR / `c8yedge config --set-file licenseKey=` | プロダクトサポートから取得 | ○ | ○ | ドメイン不一致だとEdgeが起動しない |
| `company` ✅ | edgeテナント名 | CR | — | ○ | ○ | **インストール時限定・変更不可**（後からの変更はUI/REST側で別途） |
| `email` ✅ | 管理者ユーザーのメールアドレス | CR | — | ○ | ○ | **インストール時限定・変更不可** |
| `cumulocityPasswordSecretName` ✅ | 初期管理者パスワードを持つSecret名 | CR + K8s Secret | Secretキーは `INITIAL_C8Y_ADMIN_PASSWORD`。8文字以上 | ○ | ○ | **インストール後のSecret変更は無視される**。以後の変更はUI/REST |
| `metadata.name` / `metadata.namespace` 📄 | CR名 / 名前空間 | CR | 慣例: `c8yedge` / `c8yedge` | ○ | ○ | 参照するkubectlコマンドが変わる |

### 1.2 spec 直下（任意）

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | Export | Import | 変更時の影響 |
|---------|------|---------|--------------|--------|--------|-------------|
| `tlsSecretName` ✅ | TLS証明書のSecret | CR / `c8yedge config --set-file tlsSecret.tls.crt=` | 未指定時は**自己署名を自動生成**。ドメイン本体・management・DataHubサブドメインをカバーする必要あり | ○ | ○ | 証明書入れ替え。ブラウザ警告の有無に影響 |
| `storageClassName` 📄 | PVCのStorageClass | CR | 未指定時はクラスタ既定 | ○ | ○ | **インストール時限定・変更不可** |
| `core.resources.limits.cpu` / `.memory` 📄 | Coreコンテナのリソース上限 | CR | 3000m / 6GB | ○ | ○ | Podの再作成が発生 |
| `messagingService.enabled` ✅ | Messaging Serviceの有効化 | CR | 省略時は未インストール。有効化に**+2 CPUコア / +4 GB RAM / 永続ボリューム3個** | ○ | ○ | マイクロサービス版data broker・Notifications 2.0が利用可能になる |
| `mongodb.credentialsSecretName` 📄 | MongoDB資格情報のSecret | CR | 未指定時は `databaseAdmin` で自動生成。キーは `MONGODB_DATABASE_ADMIN_USER` / `..._PASSWORD` | ○ | ○ | DB接続情報の変更 |
| `mongodb.resources.limits.cpu` / `.memory` 📄 | MongoDBのリソース上限 | CR | 3000m / 6GB | ○ | ○ | Pod再作成 |
| `mongodb.resources.requests.storage` 📄 | MongoDBデータPVCサイズ | CR | 75 GB（PVC名 `mongod-data`） | ○ | ○ | **インストール後は増加のみ可**（縮小不可） |
| `microservices[].name` ✅ | リソース割当対象のマイクロサービス | CR | apama-ctrl / smartrule / opcua-mgmt-service / databroker-agent-server / datahub | ○ | ○ | 対象マイクロサービスのリソース枠が変わる |
| `microservices[].resources.limits` 📄 | マイクロサービス単位のリソース上限 | CR | CPU 1000m / メモリ 1 GB | ○ | ○ | 該当マイクロサービスの再デプロイ |
| `dataHub.enabled` ✅ | DataHubの有効化 | CR | 省略時は未インストール。**+10 CPUコア / +10 GB RAM / 永続ボリューム5個** | ○ | ○ | 大幅なリソース消費増。DataHub機能が利用可能に |
| `dataHub.dremioAdminCredentialsSecretName` 📄 | Dremio管理者資格情報 | CR + Secret | キーは `DREMIO_ADMIN_USER` / `DREMIO_ADMIN_PASSWORD`。8文字以上・数字1＋英字1以上 | ○ | ○ | DataHub管理者の変更 |
| `dataHub.resources.limits.cpu` / `.memory` 📄 | Dremioのリソース上限 | CR | 2（**単位なし整数のみ**）/ 4096Mi（**Mi単位のみ**） | ○ | ○ | 書式違反はバリデーションエラー |
| `cloudTenant.domain` 📄 | リモート管理元のクラウドテナント | CR | 例: tenantid.cumulocity.com | ○ | ○ | クラウドからのリモート管理が有効になる |
| `cloudTenant.tlsSecretName` 📄 | MQTT X.509認証用TLS | CR | 未指定時は自己署名を自動生成。中間CA発行なら `spec.tlsSecretName` 再利用可 | ○ | ○ | クラウド接続の認証に影響 |

### 1.3 c8yedge CLI・Edge運用 ⚠️

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | Export | Import | 変更時の影響 |
|---------|------|---------|--------------|--------|--------|-------------|
| インストール | Edgeの導入 | `sudo c8yedge install`（`--version` / `-s <tarball>` / `--cumulocity-password`） | 取得元 download.cumulocity.com/Cumulocity-Edge/2025/c8yedge | — | — | 新規構築。company/email/初期パスワードはこの時点でしか設定できない |
| Helm経路 | operatorのみHelm導入 | `oci://registry.c8y.io/edge/helm-charts/cumulocity-iot-edge-operator --version=2025 -n c8yedge` | — | — | — | 自己管理クラスタでの導入方式 |
| オフラインパッケージ | エアギャップ用 | `c8yedge package` / `install -s` / `upgrade -s` | — | ○ | ○ | インターネット非接続環境での導入が可能 |
| アップグレード | バージョン更新 | `c8yedge upgrade [--version]` または CR の `spec.version` パッチ | — | — | — | **recreate戦略のため短時間のダウンタイム** |
| 外部公開 | LoadBalancer化 | `kubectl patch service cumulocity-ontoplb -n c8yedge` | サービス名 `cumulocity-ontoplb` 固定 | — | — | 外部からのアクセス経路が変わる |
| プロキシ設定 | HTTP(S)プロキシ | ConfigMap | **名前は `custom-environment-variables` 固定**。キー: http_proxy / https_proxy / socks_proxy / no_proxy / ca.crt(PEM) | ○ | ○ | 外部通信経路が変わる |
| 監視エンドポイント | Prometheus互換メトリクス | 固定提供 | `https://<domain>:3443/metrics`。収集間隔: ディスク10分/メモリ5秒/CPU・I/O・NWは5・60・600秒 | — | — | 監視系の接続先 |
| バックアップ対象 | 保全すべきディレクトリ | 運用手順 | `/var/lib/rancher/k3s`（常に必須）、`/datahub`（DataHub導入時） | — | — | 復旧可否に直結 |
| ベースラインサイジング | 必要リソース | 前提条件 | 2025: 6コア/10GB/100GB、2026: 8コア/16GB/150GB。x86-64 AVX必須、Helm 3.x、ポート80/443予約 | — | — | 不足時はインストール不可 |

---

## 2. フェーズ1: テナント設定

### 2.1 Application 📄

UIパス: Administration > Settings > Application ／ 設定単位: テナント全体 ／ Export: △（REST GET） ／ Import: △（REST PUT）

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | 変更時の影響 |
|---------|------|---------|--------------|-------------|
| Default application | アプリ未指定でアクセスした際のランディングアプリ | UI（ドロップダウン） | 利用可能なアプリから選択 | **テナント内の全ユーザー**のログイン後の着地画面が変わる |
| Access control — CORS | Cumulocity APIのCORS有効化 | UI（トグル） | 有効/無効 | 無効化すると外部JSアプリからのAPI呼び出しが遮断される |
| Allowed domain | REST APIと通信可能なドメイン | UI（テキスト） | `*`（全ホスト）またはカンマ区切りドメイン | 範囲を狭めると既存の外部アプリが動作しなくなる可能性 |

### 2.2 Properties library 📄

UIパス: Administration > Settings > Properties library ／ 設定単位: オブジェクト種別ごとのカスタムプロパティ（対象: Inventory / Alarms / Events / Tenants） ／ Export: △ ／ Import: △

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | 変更時の影響 |
|---------|------|---------|--------------|-------------|
| Name（識別子） | プロパティの一意名 | UI | 一意であること | 既存データとのキー整合に影響 |
| Label | 表示ラベル | UI | — | UI表示のみ |
| Type | データ型 | UI | String / Boolean / Integer 等 | 既存値との型不整合が発生しうる |
| Required | 必須指定 | UI | 既定: 未チェック | **オブジェクト作成時に入力必須**となり、既存の作成フローに影響 |
| Default value | 既定値の自動入力 | UI | **String型のみ**利用可 | 新規作成時のみ適用 |
| Minimum / Maximum | 整数の下限/上限 | UI | — | 範囲外の入力が拒否される |
| Minimum length / Maximum length | 文字列長の下限/上限 | UI | — | 同上 |
| Regular expression | 正規表現バリデーション | UI | 任意のregex | 不正なパターンは全入力を拒否しうる |

### 2.3 SMS provider 📄

UIパス: Administration > Settings > SMS provider ／ 設定単位: テナント全体 ／ Export: ✗ ／ Import: ✗

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | 変更時の影響 |
|---------|------|---------|--------------|-------------|
| SMS provider | 利用するSMSプロバイダ | UI（ドロップダウン・フィルタ可） | 利用可能なプロバイダ一覧から選択 | **SMSベースの二要素認証・SMS送信全般**が影響を受ける |
| Provider credentials | プロバイダの認証情報 | UI | プロバイダ依存 | 誤設定でSMS送信が全面停止 |
| Provider properties / Optional settings | プロバイダ固有設定 | UI | プロバイダ依存 | 同上 |

### 2.4 Connectivity 📄

UIパス: Administration > Settings > Connectivity ／ 設定単位: プロバイダ単位 ／ Export: ✗ ／ Import: ✗

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | 変更時の影響 |
|---------|------|---------|--------------|-------------|
| Provider タブ | 接続プロバイダの種別 | UI（タブ） | Actility LoRa / Sigfox / SIM | 対象プロトコルのデバイス連携に影響 |
| Provider URL | プロバイダのエンドポイント | UI | URL形式 | 接続先が変わりデバイス通信が停止しうる |
| Provider credentials | プロバイダ認証情報 | UI | プロバイダ依存 | 同上 |

### 2.5 Localization 📄

UIパス: Administration > Settings > Localization ／ 設定単位: テナント全体 ／ Export: ✗ ／ Import: ✗

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | 変更時の影響 |
|---------|------|---------|--------------|-------------|
| Translation key | 翻訳キーの名前 | UI | 一意 | 既存キーの上書きに注意 |
| 言語別翻訳 | 言語ごとのカスタムUIテキスト | UI | 対応言語ごとのテキスト | **全ユーザーのUI表示文言**が変わる |

### 2.6 Feature toggles 📄

UIパス: Administration > Settings > Feature toggles ／ 設定単位: テナント（Managementからはサブテナント単位） ／ Export: ✗ ／ Import: ✗

| 設定項目 | 概要 | 設定方法 | 既定値・許容値 | 変更時の影響 |
|---------|------|---------|--------------|-------------|
| Feature name / Description / Key | 機能名・説明・一意キー | 読み取り専用 | — | — |
| Phase | 提供段階 | 読み取り専用 | Generally available / Public Preview | Preview機能は仕様変更の可能性 |
| Status | 機能の有効/無効 | UI（トグル） | 既定はPhaseに依存 | **該当機能がテナント全体で利用可否切替**。UIメニューの表示も変わる |
| Strategy | カスタマイズ状態 | 表示のみ（Resetボタン） | 既定動作 | Resetで既定へ戻る |
| Subtenant feature toggles | サブテナントの機能制御 | Managementテナントから | Enterpriseテナントでは読み取り専用 | 配下サブテナント全体に波及 |

### 2.7 テナントオプション ✅📄

UIパス: 公式UIプラグイン Tenant Option Management（プラグイン管理下のオプションのみ操作可） ／ 設定単位: テナント全体 ／ 設定方法: `GET/POST/PUT/DELETE /tenant/options`（システム値の参照は `/tenant/system/options`） ／ Export: △ ／ Import: △

> テナントオプションは category / key / value の**動的なキー空間**であり、全キーの静的一覧は存在しない。**文書化された既定値の網羅カタログは3回の調査でも発見できず**、既定値は各機能ページに分散している。網羅把握には実テナントで `GET /tenant/system/options` / `GET /tenant/options` を実行して現物採取するのが確実。

| category / key | 概要 | 既定値・許容値 | 変更時の影響 |
|---------------|------|--------------|-------------|
| configuration / `default.tenant.applications` ✅ | テナント作成時の既定Webアプリ | カンマ区切りのアプリ名 | **新規作成テナントのみ**に適用。既存テナントは変わらない |
| configuration / `default.tenant.microservices` ✅ | テナント作成時の既定マイクロサービス | カンマ区切り名 | 同上 |
| configuration / `on-update.tenant.applications` ✅ | アップグレード時の既定Webアプリ | カンマ区切り名 | プラットフォームアップグレード時に適用 |
| configuration / `on-update.tenant.microservices` ✅ | アップグレード時の既定マイクロサービス | カンマ区切り名 | 同上 |
| configuration / `on-update.tenant.applications.enabled` ✅ | アップグレード時の独自リスト有効化 | true / false（false・未設定時は `default.tenant.*` にフォールバック） | 有効化しないと上記2キーが効かない |
| configuration / `on-update.tenant.microservices.enabled` ✅ | 同上（マイクロサービス） | true / false | 同上 |
| `measurement.series.latestvalue` 📄 | 最新メジャーメント値のシリーズ単位有効化 | キー例 `c8y_Humidity.H` / `c8y_Temperature.*` / `*`（値は空文字）。`PUT /tenant/options/measurement.series.latestvalue` | 対象シリーズの最新値取得APIの挙動が変わる |
| （同上）`strongConsistency` 📄 | 順序外れメジャーメントの扱い | 任意トグル | 有効時は順序外れ値を最新として表示しない |
| configuration / `measurement.series.previousvalue.enabled` 📄 | 直前値の取得 | "true" / "false"（既定: 有効） | 直前値APIの可否 |
| sso / `sso-redirect-default-application` ✅ | SSOログイン時のリダイレクト先制御 | Boolean。`false` でテナントドメインの使用を強制 | baseUrlに `{tenantId}` を含む構成やSAN証明書利用時に必要 |
| `credentials.` プレフィックス ✅ | 値が暗号化されるオプション群 | — | **テナントポリシー経由では暗号化されない**ため、必ず個別APIで設定する |

### 2.8 権限・ロール 📄

UIパス: Administration > Roles ／ 設定単位: ロール単位（グローバルロール=システム権限、インベントリロール=デバイスグループ単位） ／ 設定方法: UI / REST（`/user/inventoryroles`、`/user/users/{username}/inventoryroles` 等） ／ Export: △ ／ Import: △

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| グローバルロール（既定） | システムレベル権限のセット | admins / devices / CEP Manager / Cockpit User / Devicemanagement User / Global Manager / Global Reader / Global User Manager / Shared User Manager / Tenant Manager（レガシー: business, readers） | 割り当て済みの全ユーザーの権限が即座に変わる |
| 権限レベル | 各権限タイプの粒度 | READ / CREATE / UPDATE / ADMIN | 過剰付与はセキュリティリスク、過少はAPI 403の原因 |
| 権限カテゴリ | 権限を付与する対象領域 | Alarms / Application management / Audits / Bulk operations / CEP management / Data broker / Device control / Events / Global smart rules / Identity / Inventory / Measurements / Option management / Retention rules / Schedule reports / Simulator / SMS / Tenant management / Tenant statistics / User management / Own user management（21種） | 各APIの利用可否に直結 |
| インベントリロール（既定） | デバイスグループ単位の権限 | Manager / Operations: All / Operations: Restart Device / Reader | 対象グループ配下のデバイス操作可否 |
| インベントリロール作成項目 | ロール定義のフィールド | name / description / permissions（カテゴリ単位）/ fragment types（Type欄）/ レベル（READ / CHANGE / ALL） | 割り当てユーザーのデバイスアクセス範囲 |
| Shared User Manager | サブユーザー管理ロール | `feature-user-hierarchy` サブスクリプションが必要 ⚠️ | サブスクリプションがないと機能しない |

### 2.9 Retention rules（データ保持）📄

UIパス: Administration > Management > Retention rules ／ 設定単位: データ種別＋フィルタ条件の組 ／ 設定方法: UI / REST `/retention/retentions` ／ Export: △ ／ Import: △ ／ 権限: Retention rules の ADMIN

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Data type | 削除対象のデータ種別 | Alarms / Audits / Bulk operations / Events / Measurements / Operations（1ルール1種別・必須） | 対象データが期限到来で**不可逆に削除**される |
| Fragment type | フラグメントでの絞り込み | 既定 `*`。空白・特殊文字不可 | 絞り込みを外すと対象が広がる |
| Type | typeフィールドでの絞り込み | 既定 `*` | 同上 |
| Source | デバイスIDでの絞り込み | 既定 `*` | 特定デバイスのみ対象化 |
| Maximum age | 最大保持日数 | 1〜3,650日（10年）。プラットフォーム既定は**60日** | **短縮すると次回実行時に既存データが削除される（復旧不可）** |
| （挙動）実行タイミング | ルールの適用 | 通常1日1回・逐次実行 | 即時ではなく日次で反映 |
| （挙動）対象外 | 適用されない領域 | **ファイルリポジトリには適用されない**。アラームはstatus=CLEAREDのみ削除 | ファイル容量は別途管理が必要 |

### 2.10 認証 — Basic settings ✅

UIパス: Administration > Settings > Authentication > Basic settings ／ 設定単位: テナント全体 ／ 設定方法: UI / REST `/tenant/loginOptions` ／ Export: △ ／ Import: △

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Preferred login mode | ログイン方式 | Basic Auth（互換目的のみ）/ **OAI-Secure（新規テナント既定・推奨）** / Single sign-on redirect（SSO構成時のみ選択可） | **SSO redirectを選ぶとBasic AuthとOAI-Secureのログインオプションが除去される**（切り戻し手段の確保が必要） |
| Password validity limit | パスワード有効期限 | 日数指定。**既定 0（無期限）**。devicesロールのユーザーは対象外 | 設定すると全ユーザーに期限到来時のパスワード変更を強制 |
| Password strength requirement | 強度の強制 | 既定: 未チェック（未強制時の既定ポリシーは**8文字以上のみ**） | 既存の弱いパスワードのユーザーに影響 |
| Ignore case when logging in | ユーザー名の大文字小文字を無視 | 既定: 無効。テナント管理者のみ設定可 | **既存ユーザー間に大小無視の衝突があると有効化できない** |
| Forbidden for web browsers | ブラウザからのBasic認証を禁止 | トグル | ブラウザ経由のBasic認証利用が停止 |
| Trusted user agents | Basic認証を許可するUA | **既定: 空** | リスト運用を誤ると正規クライアントが遮断される |
| Forbidden user agents | Basic認証を拒否するUA | **既定: 空** | 同上 |
| Allow two-factor authentication | TFAの許可 | 既定: 未チェック。管理者のみ | 有効化後にユーザー単位/全体のTFA設定が可能に |
| （SMS）Token validity limit | セッショントークン有効期間 | **分単位**。SMS TFAは**SMSゲートウェイマイクロサービスの構成が必須** | 短すぎると再認証が頻発 |
| （SMS）Verification code validity limit | SMS検証コード有効期間 | 分単位 | 短すぎるとコード入力が間に合わない |
| （TOTP）Enforce TOTP on all users | 全ユーザーにTOTP設定を強制 | **OAI-Secureモード時のみ利用可**。devicesロールは対象外 | 次回ログイン時に全ユーザーがTOTP設定を要求される |

### 2.11 SSO / Single sign-on ✅

UIパス: Administration > Settings > Authentication > **Single sign-on タブ**（SSO構成アクセスが有効なテナントにのみ表示。Management専用構成も可） ／ 設定単位: テナント全体 ／ 設定方法: UI / REST `/tenant/loginOptions` ／ Export: △ ／ Import: △

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Redirect URI | 認可サーバーからの戻り先 | Azure ADの場合はテナントアドレス＋`/tenant/oauth` ⚠️ | 誤設定でSSOログインが完全に失敗する |
| Redirect to the user interface application | UIアプリへのリダイレクト | 2026版のみの新フィールド（既定Disabledは推論） | リダイレクト挙動が変わる |
| Client ID | OAuth2クライアントID | 認可サーバー発行値 | 不一致で認証失敗 |
| Token issuer | トークン発行者（iss） | 認可サーバーの識別子 | 検証失敗の原因 |
| Button name / Provider name | ログイン画面の表示名 | — | UI表示のみ |
| Audience | JWTの`aud`期待値 | — | 不一致でトークン拒否 |
| Visible on Login screen | ログイン画面への表示 | トグル | 非表示にすると利用者が経路を選べない |
| Authorize / Token / Refresh / Logout リクエスト定義 | 各エンドポイントの呼び出し定義 | Authorize=GET / Token=POST / Refresh=POST / Logout=任意（front-channel single logout、OIDCでは`id_token_hint`）。プレースホルダ `${clientId}` `${redirectUri}` `${code}` `${refreshToken}` | 定義誤りで認証フローが途中で失敗 |
| Dynamic access mapping | トークン内容に基づくロール自動割当 | 3モード: ①ユーザー作成時のみ ②毎ログイン再割当（他は保持） ③**毎ログイン再割当＋他はクリア（既定）** | **既定モードではCumulocity側で手動変更したロールが次回ログインで上書きされる** |
| Access token validation frequency | 外部トークン検証のキャッシュ間隔 | **既定: 1分** | 短くすると認可サーバー負荷増、長くすると失効反映が遅延 |
| 署名検証方式 ⚠️ | トークン署名の検証手段 | Azure AD証明書ディスカバリ / ADFSマニフェスト / 手動公開鍵 / JWKS URI。**RSA鍵のみ対応**（"n"/"e"対 or x5c）。楕円曲線鍵は非対応 | EC鍵の認可サーバーは利用できない |
| SSOユーザーの権限前提 ⚠️ | 必要な最低権限 | 「Own user management」のREAD | 不足するとログイン後にエラー |

> **注意**: プロバイダテンプレート（Custom / Azure AD / Keycloak / ADFS）に関する主張は検証で棄却（1-2）されており、現行版の一覧は実機確認が必要。

### 2.12 ブランディング（クラウドEnterprise限定）✅📄

UIパス: Administration > Settings > Branding ／ 設定単位: テナント全体 ／ 設定方法: UI ／ **Export: ○（バリアントのエクスポート）** ／ **Import: ○（ブランディングパッケージのインポート。2026版ではブランディング未設定の新規テナントへ直接インポート可）**

| 設定項目グループ | 概要 | 既定値・許容値 | 変更時の影響 |
|----------------|------|--------------|-------------|
| Generic — Title / Favicon | ブラウザ表示のタイトルとアイコン | Faviconは**ICO形式のみ** | 全ユーザーのブラウザ表示 |
| Generic — Dark theme | ダークテーマ対応の有無 | トグル | ユーザーのテーマ選択肢が変わる |
| Generic — Typography | フォントスタック | Base / Headings / Navigator（base or headingsと同一）、リモートフォントリンク | 全画面のフォント |
| Generic — Cookie banner | Cookie同意バナー | 有効化 / Title / Text / プライバシーポリシーURL / ポリシーバージョン | **バージョンを上げると全ユーザーに再同意を要求** |
| Logos（Light/Dark各） | ロゴ画像と高さ | Brand logo / Navigator logo（PNG / SVG / JPG） | 全画面のロゴ表示 |
| Brand colors（同） | ブランド配色 | Primary / Light / Dark ＋ 8段階シェード（自動生成可）。HEX / RGB / RGBA | UI全体の配色 |
| Status colors（同） | 状態色 | Info / Warning / Danger / Success 各 default・light・dark | アラート等の視認性 |
| Generic colors（同） | 基本配色 | 背景 / テキスト / Text muted / リンク / リンクホバー / ボタンborder-radius | 全体の見た目 |
| Action bar / Main header / Navigator / Right drawer（同） | 各UI領域の配色 | 各領域ごとに背景・テキスト・ボタン等 | 該当UI領域の表示 |
| Custom CSS | 任意CSSの適用 | CSSエディタ | **記述ミスでUIが崩れる可能性** |
| Advanced | ブランディングJSONの直接編集 | JSONエディタ | 同上。最も自由度が高く危険度も高い |
| Domain name | 独自ドメインとSSL | 証明書は**PKCS #12**。有効化 / 更新 / 無効化 | アクセスURLと証明書の切り替え |

### 2.13 Enterprise Configuration（メール・テンプレート等）📄

UIパス: Administration > Settings > Configuration（Enterprise） ／ 設定単位: テナント全体 ／ Export: ✗ ／ Import: ✗

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Two-factor authentication SMSテンプレート | TFAのSMS文面 | テンプレート文字列 | 全TFA利用者へのSMS文面 |
| Support link | サポートリンクURL | URL または `false`（非表示） | UIのサポート導線 |
| Password reset | パスワードリセット関連 | 未知アドレスへの送信許可トグル / 既知・未知アドレス用テンプレート / 件名 / 変更確認テンプレート / 招待テンプレート | リセット運用全般 |
| Email server | メール送信サーバ | プロトコル: SMTP / SMTP STARTTLS / SMTPS SSL-TLS、ホスト、ポート、ユーザー名、パスワード、送信元アドレス | **誤設定で全通知メールが送信不能** |
| Data export | データエクスポート通知 | 件名 / テンプレート / 権限エラーメッセージ | エクスポート機能の通知 |
| Storage limit | ストレージ上限通知 | 警告・超過それぞれの件名/テンプレート | 容量監視の通知 |
| Suspending tenants | テナント停止通知 | サスペンド管理者への送信トグル / 追加受信者 / 件名 / テンプレート | サブテナント停止時の連絡 |

### 2.14 サブテナント / テナントポリシー / デフォルトサブスクリプション（クラウドEnterprise限定）✅

UIパス: Administration > Tenants ／ 設定単位: サブテナント ／ 設定方法: UI / REST `POST /tenant/tenants`（要 ROLE_TENANT_MANAGEMENT_ADMIN/CREATE） ／ Export: △ ／ Import: △

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Domain/URL（必須） | サブテナントのURL | 小文字英字・数字・ハイフンのみ、先頭は英字、ハイフンは中間のみ、**最短2文字**、1階層のみ | アクセスURLが確定する |
| Name（必須） | テナント名 | — | 表示名 |
| Administrator's email（必須） | 管理者メール | 有効なメールアドレス | パスワードリセットの宛先 |
| Administrator's username（必須） | 管理者ユーザー名 | — | 初期ログインアカウント |
| Contact phone（必須） | 連絡先電話番号 | — | 管理情報 |
| Contact name | 連絡先氏名 | 任意 | 管理情報 |
| Send password reset link as email | 初期パスワードの通知方法 | **既定: 選択済み**。解除時はパスワード入力＋確認が必須 | 解除するとメールが飛ばず手動連携が必要 |
| Tenant policy | 適用するポリシー | 任意（ドロップダウン） | **作成時に内容がコピーされる。以後ポリシーを編集しても既存テナントには伝播しない** |
| テナントポリシーの構成 | ポリシーの中身 | Name（必須）/ Description / **Retention rules（最低1件必須）** / Tenant options（category+key+value）。各項目に「サブテナントによる変更を許可」チェック（既定: 未チェック） | ポリシー編集は**新規作成テナントのみ**に効く |
| （制約）ポリシーの暗号化 | 値の保護 | **ポリシー内のテナントオプションは暗号化されない** | `credentials.` プレフィックスのキーを入れてはならない |
| デフォルトサブスクリプション | 既定で購読するアプリ | 「作成時」と「アップグレード時」の2リストを個別管理。テナントオプション（2.7）で上書き可 | 新規テナントの初期アプリ構成 |
| Limits — HTTPスロットリング | HTTPリクエスト制限 | Limit HTTP queue ＋ Limit HTTP requests（毎秒）。**両方設定時のみ有効** | 片方だけでは制限が効かない |
| Limits — streamスロットリング | MQTT制限 | Limit stream queue ＋ Limit stream requests（毎秒）。**両方必須** | 同上 |
| Limits — デバイス数上限 | 登録可能デバイス数 | 同時登録ルートデバイスのみ / 子デバイス含む全デバイスの2方式 | 上限到達で新規登録が失敗 |
| Limits — External reference | 外部参照キー | — | 課金・管理システムとの突合 |
| Limits — Gainsight tracking | 製品利用トラッキング | チェックボックス | 利用状況の外部送信 |
| （挙動）サスペンド | 停止時の扱い | パブリッククラウドでは**サスペンド後60日で自動削除** ⚠️ | データ消失の期限 |
| Custom properties | Properties libraryで定義した値 | 2.2で定義した項目 | テナント属性 |

### 2.15 Data broker ✅

UIパス: Data broker アプリ ／ 設定単位: データコネクタ単位 ／ **前提: `feature-broker` アプリケーションのサブスクリプション**（マイクロサービス版はさらに `databroker-agent-server`） ／ Export: ✗ ／ Import: ✗

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Title | コネクタ名 | — | 表示のみ |
| Target URL | 転送先テナントのURL | **保存後は編集不可** | 変更にはコネクタの作り直しが必要 |
| Description | 説明 | 任意 | — |
| Data filters | 転送対象の定義 | **最低1つ必須** | フィルタなしでは作成不可 |
| フィルタ — Group or device | 対象範囲 | グループ/デバイス選択。`All Objects` は後方互換で残置され**非推奨予定・使用非推奨** | 範囲拡大は転送量と課金に直結 |
| フィルタ — API | 対象API | alarms / events / measurements / managed objects（転送）、operations（受信） | 転送されるデータ種別 |
| フィルタ — Fragments to filter | フラグメント条件 | — | 対象データの絞り込み |
| フィルタ — Fragments to copy | 転送するフラグメント | `Copy all fragments` オプションあり。**未指定時は標準プロパティのみ**（アラーム作成時: type, text, time, severity, status） | 未指定だと想定より情報が欠落する |
| フィルタ — Type filter | typeでの絞り込み | — | 対象データの絞り込み |
| コネクタのステータス ⚠️ | 転送の状態 | Active（有効）/ Suspended（ソース側が無効化）/ Pending（宛先側が無効化） | 転送の停止/再開 |
| バックログ ⚠️ | 未配信メッセージの保持 | コネクタ毎に永続バックログ。テナント毎クォータ・TTLは構成可（数値既定はservice quotas文書参照） | **クォータ到達時は該当APIリクエストがHTTP 500** |
| マイクロサービス版の有効化 ⚠️ | 高機能版の利用 | **Managementテナントから運用サポート関与で有効化**（セルフサービス不可）。Messaging Service依存 | テナント管理者だけでは有効化できない |
| （課金）転送デバイス ⚠️ | 宛先での扱い | **宛先テナントで通常デバイスとして課金** | コスト増 |

### 2.16 Usage statistics / 課金 ✅

UIパス: Administration > Usage statistics（Management/Enterpriseテナント。**Tenant management の READ 権限が必要**） ／ 設定単位: テナント / マイクロサービス ／ Export: △（統計データのCSV） ／ Import: ✗

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| `billingMode` | マイクロサービスの課金方式 | マイクロサービスマニフェストのフィールド。**RESOURCES（既定）** / SUBSCRIPTION | 課金計算方式が変わる |
| 統計収集スケジュール | 値の採取タイミング | リクエスト数フラッシュ: **5分毎**。Used storage / Device count / Subscribed applications / Microservice resources: **9時・17時・EOD**（EODが確定値。TZ明記なし） | 数値の反映タイミング |
| リソース計測単位 | CPU・メモリの単位 | CPU: ミリコア（**1000m = 1 CPU**）、メモリ: MB。テナント毎日次収集 | 見積り換算 |
| Usage statistics 画面の列 | 表示項目 | API requests / Device API requests / Storage (MB) / Peak storage (MB) / Devices / Peak devices / Endpoint devices / Subscribed applications / Alarms・Events・Inventories・Operations created-updated / Measurements created / Total inbound transfer / CPU (M) / Memory (MB) ほか（ID, Tenant, Root devices, Peak root devices, Creation time, Parent tenant, External reference） | 請求根拠の確認 |
| CSVエクスポート | 統計のファイル出力 | フィールド区切り文字・小数点記号・文字セットを指定可 | ロケール差異による解析エラーの回避 |
| サスペンド中の課金 ⚠️ | 停止テナントの扱い | リクエスト数・マイクロサービスリソースは非課金。テナント存在とストレージのみカウント | コスト削減の手段 |

### 2.17 アラームマッピング ⚠️

UIパス: Administration > **Business Rules > Alarm mapping** ／ 設定単位: アラームタイプ（プレフィックス）単位 ／ 権限: 閲覧=Option management の READ、作成/編集/削除=同 ADMIN（既定では Tenant Manager が保有） ／ Export: ✗ ／ Import: ✗

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Alarm type（必須） | 対象アラームタイプ | **プレフィックス（ワイルドカード）一致**で解釈される | **1マッピングが同一プレフィックスの複数タイプに波及**。作成後は変更不可 |
| New description | 置き換える説明文 | 任意。空欄時は元のアラームテキストを維持 | 表示文言のみ |
| Severity | 新しい重大度 | ドロップダウン。または **Drop**（アラーム表示の抑止） | **Dropにすると該当アラームが表示されなくなる**（見落としリスク） |

### 2.18 ユーザー管理 ⚠️

UIパス: Administration > Users ／ 設定単位: ユーザー単位 ／ 設定方法: UI / REST（User API） ／ Export: △ ／ Import: △

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| パスワード設定方法 | 作成時の初期パスワード | 3択: Send password reset link as email / Set password that must be changed on the first login / Set password for the user (no change required) | 初回ログインの手順が変わる |
| Two-factor authentication (SMS) | ユーザー単位のSMS TFA | テナントレベルでTFA有効時のみ | 該当ユーザーのログイン手順 |
| Two-factor authentication (TOTP) | ユーザー単位のTOTP | 同上 | 同上 |
| Enforce TOTP setup for the user | 初回ログイン時のTOTP設定強制 | 同上 | 次回ログインで設定を要求 |
| ユーザー階層 | 階層的なユーザー管理 | `feature-user-hierarchy` サブスクリプションが必要 | 未購読では Shared User Manager も機能しない |
| SSOユーザーの制約 | 外部認可サーバー経由ユーザー | **ユーザー情報・グローバルロール・アプリケーションアクセス・インベントリロールのCumulocity側更新は無効**。パスワードリセットも無効 | ロール管理はIdP側で行う必要がある |
| Log out all users | 全ユーザーの強制ログアウト | OAI-Secure・SSOリダイレクト・JWTデバイストークンを無効化。**Base64のBasic認証トークンは無効化されない** | Basic認証利用時は完全な遮断にならない |

---

## 3. フェーズ2: 初期コンテンツ

### 3.1 デバイス一括登録 ⚠️

UIパス: Device Management > Devices > Registration ／ 設定単位: デバイス単位 ／ 設定方法: **CSVアップロード** / REST（New device requests API） ／ Export: ✗ ／ **Import: ○（CSV）**

| 設定項目（CSV列） | 概要 | 必須 | 既定値・許容値 | 変更時の影響 |
|-----------------|------|------|--------------|-------------|
| ID | デバイス識別子 | ○ | 例: シリアル番号 | **既存IDのデバイスはCSV内容で更新される**（再実行可能） |
| CREDENTIALS | デバイス毎のパスワード | ○（フル形式） | 一意な値 | デバイス認証に直結 |
| TYPE | デバイスタイプ | — | 例: c8y_Device | ダッシュボードテンプレート等の適用対象に影響 |
| NAME | 表示名 | — | — | UI表示 |
| ICCID | モバイル識別子 | — | — | SIM連携 |
| IDTYPE | IDの種別 | — | 例: c8y_Serial | 外部ID解決の方式 |
| PATH | グループ割り当てパス | — | スラッシュ区切り。**未存在のグループは自動作成** | グループ構造が自動生成される |
| SHELL | シェルアクセス | — | 1 / 0 | リモートコマンド可否 |
| AUTH_TYPE | 認証方式 | — | BASIC 等 | デバイス接続方式 |
| TENANT | 対象テナント | — | Enterprise限定 | 他テナントへの登録 |
| （単一登録）Device ID | 個別登録の識別子 | ○ | クエリパラメータ `externalId` で事前入力可 | — |
| （単一登録）One-time password | ワンタイムパスワード | — | クエリパラメータ `one-time-password` で事前入力可 | — |
| （単一登録）セキュリティトークン | トークンポリシー | — | IGNORED / OPTIONAL / REQUIRED | REQUIREDにすると全登録でトークンが必要 |

### 3.2 Cockpit ダッシュボード 📄⚠️

UIパス: Cockpit > 対象グループ/デバイス > Add dashboard ／ 設定単位: ダッシュボード単位 ／ 設定方法: UI / REST（Inventory API、`c8y_Dashboard` フラグメント付き managed object） ／ **Export: ○（JSONファイル）** ／ **Import: ○（JSONファイル。異なるアセット種別間・異なるテナント間でも共有可）**

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Icon | ナビゲータのアイコン | 選択式 | 表示のみ |
| Menu label | ダッシュボード名 | — | ナビゲータ表示 |
| Description | 説明 | — | 表示のみ |
| Location（位置） | ナビゲータ内の並び順 | 5000 〜 -5000 | メニューの並びが変わる |
| Availability（グローバルロール） | 閲覧可能なロール | **既定: 全ロール選択済み** | 絞ると対象外ユーザーから見えなくなる |
| Dashboard template | デバイスタイプ全体への共有 | デバイスタイプ設定時のみ有効 | **同一タイプの全デバイスに一括適用**（個別注入が不要になる） |
| Theme | 配色テーマ | Match UI / Light / Dark / Branded | 表示のみ |
| Widget header style | ウィジェット見出しの体裁 | Regular / Border / Overlay / Hidden | 表示のみ |
| Widget gap | ウィジェット間の余白 | — | レイアウト |
| Layout mode | レイアウト方式 | **Grid（既定・固定スナップ）** / Responsive（列構成・ビューポートフィット） | 既存配置の見え方が変わる |
| Translate if possible | タイトルの翻訳 | トグル | 多言語表示 |
| Version history | 変更履歴と復元 | — | 誤編集からの復旧が可能 |
| Copy / Paste dashboard | オブジェクト間の複製 | UI操作 | 複製後は**元と完全に切り離され継承関係はない** |
| `c8y_Dashboard` フラグメント ⚠️ | REST操作時の構造 | icon / priority / name / global / isFrozen / children（ウィジェット定義） | **テナント間移送では内部オブジェクトIDとカスタムウィジェットの再マッピングが必要** |

### 3.3 Analytics Builder モデル 📄

UIパス: Streaming Analytics > Analytics Builder（モデルマネージャ） ／ 設定単位: モデル単位 ／ **Export: ○（JSONダウンロード。公式にテナント間移送用途と明記）** ／ **Import: ○（JSONアップロード / クリップボード貼り付け）**

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| モデルのモード | 実行モード | **Draft（新規既定）** / Test（実データ・出力は保存のみ・単一デバイス） / Simulation（履歴再生・出力は保存のみ・単一デバイス） / Production（実データ・**デバイスへ出力送信**） | **Productionにすると実デバイスへ操作が送信される** |
| アクティベーション状態 | 配備の有無 | Active / Inactive。**JSONインポート直後は常にInactive** | 自動注入には**別途アクティベーション手順が必須** |
| モデル名 | 一意な識別名 | 一意であること | 重複不可 |
| Description / Tags | 説明・タグ | 任意（タグはフィルタ用） | 管理性 |
| テンプレートパラメータ | 再利用のためのパラメータ | 名前（モデル内一意）/ 型 / 既定値 / 必須・任意 | インスタンス毎に値・モード・アクティベーションを独立設定可 |
| ブロック単位パラメータ | 各ブロックの設定 | 入出力デバイス / fragment・series マッピング / 閾値 / 期間・タイミング / メジャーメント種別 / アラーム種別・重大度 | ロジックの挙動そのもの |
| Simulation設定 | 履歴再生の範囲 | 開始・終了タイムスタンプ（カレンダー選択） | 検証対象期間 |
| （関連）EPLアプリ ⚠️ | Apama `.mon` によるCEP | Streaming Analyticsでアクティベートするとデプロイ。既存 `.mon` のインポート可 | CEPロジックの追加 |

### 3.4 アプリケーション / マイクロサービス ⚠️

UIパス: Administration > Ecosystem > Own applications ／ 設定単位: アプリケーション単位 ／ 設定方法: **ZIPアップロード** / REST（Application API）/ go-c8y-cli ／ Export: ○（ZIP） ／ Import: ○（ZIP）

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Webアプリケーション | UIアプリの配置 | ZIPアーカイブ | サブスクライブしたテナントで利用可能に |
| マイクロサービス | サーバサイド処理 | ZIP / レジストリ経由。設定は**テナントオプション（category = contextPath が既定）**として保存 | Edgeでは**Microservice Hostingが有効な場合のみ**。VM版Edgeは有効化に10〜15分（その間Edge非稼働） |
| デプロイ自動化 | CI/CDからの投入 | go-c8y-cli の deploy コマンド | スクリプト化が可能 |
| （注記） | 個別フィールド | **今回の調査では未取得**。実機確認時に補完 | — |

### 3.5 デバイス可用性・監視 ⚠️

UIパス: Device Management > 対象デバイス ／ 設定単位: デバイス単位 ／ 設定方法: UI / REST（Inventory API） ／ Export: △ ／ Import: △

| 設定項目 | 概要 | 既定値・許容値 | 変更時の影響 |
|---------|------|--------------|-------------|
| Required interval | デバイスからの通信を期待する間隔 | ユーザーが手動設定するか、デバイス自身が設定 | 短すぎると誤検知の可用性アラームが多発 |
| 接続断中の扱い | オフライン時の状態判定 | **既定では切断前の状態（in service / out of service）を維持**とみなす | 可用性の算出結果に影響 |
| 可用性の算出期間 | パーセンテージの集計単位 | 24時間 / 7日 / 30日 | レポートの見え方 |
| 不可用扱いのアラーム重大度 | 機能不全の判定 | **CRITICAL でなければ不可用として扱われない**（MAJORでは不可）。重大度は CRITICAL / MAJOR / MINOR / WARNING | 重大度設計を誤ると可用性統計が実態と乖離 |
| 一覧の自動リフレッシュ間隔 | 画面更新頻度 | アラーム・イベントとも既定 **30秒** | 表示の即時性 |
| メンテナンスモード | 保守中のアラーム抑止 | 接続監視カードでオン/オフ | **メンテナンス中はアラームが生成されない**（戻し忘れに注意） |

---

## 4. 一覧から読み取れる要点

1. **ファイルベースのExport/Importが公式に存在するのは4カテゴリのみ** — Edge CR（YAML）、ブランディング、Cockpitダッシュボード（JSON）、Analytics Builderモデル（JSON）。加えてデバイス一括登録CSV（Import専用）とアプリケーションZIP。それ以外のテナント設定は **REST APIでのGET→JSON化→POST/PUTによる注入（△）が現実解**。
2. **インストール時にしか設定できない項目がある** — Edge CR の `company` / `email` / `version` / `domain` / `storageClassName` / 初期管理者パスワード。セットアップ手順の最初に確定させる必要がある。
3. **不可逆・伝播しない設定に注意** — Retention rules の期間短縮（データ削除は復旧不可）、テナントポリシー（作成時のみコピー・以後の編集は伝播しない）、Data broker の Target URL（保存後変更不可）、アラームマッピングの Alarm type（作成後変更不可）。
4. **ログインモードの変更は退路を断ちうる** — SSO redirect を選ぶと Basic Auth と OAI-Secure のログインオプションが除去される。
5. **既定値カタログは存在しない** — テナントオプション/システムオプションの網羅的な既定値一覧表は公式ドキュメントに見つからない。実テナントで `GET /tenant/system/options` / `GET /tenant/options` を実行して現物を採取すること。
