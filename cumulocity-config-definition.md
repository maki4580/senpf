# Cumulocity 設定定義書 — 設定項目・設定手段・エクスポート/インポート方法

| 版 | 日付 | 内容 |
|----|------|------|
| 初版 | 2026-08-04 | クラウド版テナント設定の6領域・設定手段・エクスポート/インポート調査（deep-research: 情報源19件・主張92件抽出・25件検証・確定21/棄却4） |
| 第2版 | 2026-08-04 | Cumulocity Edge での適用可否（検証済み）と、初期コンテンツ注入段階（デバイス/ルール/ダッシュボード等）を追記（deep-research 2回目: 情報源21件・主張99件抽出・25件検証・確定23/棄却2。※コンテンツ注入系の主張は検証枠外となったため journal から救出し「未検証」マーク付きで記載） |

情報源: cumulocity.com 公式ドキュメント（2024/2025/2026リリース版）、OpenAPI仕様、Cumulocity-IoT 公式GitHubリポジトリ、コミュニティフォーラム

**検証状態の凡例**: ✅ = 反証検証を通過した確定事項（票数併記） / ⚠️未検証 = 一次ソースから引用付きで抽出済みだが反証検証を未実施（採用前に実機確認を推奨）

---

## 1. 総括（結論）

セットアップ自動化は次の **3層（フェーズ）** で構成するのが全体像として妥当:

| フェーズ | 対象 | 主な手段 | 適用環境 |
|---------|------|---------|---------|
| 0. デプロイ層 | Edge本体のインストール・インフラ構成（ライセンス/ドメイン/TLS/初期管理者/オプションコンポーネント） | **Edge Custom Resource (c8yedge.yaml) + kubectl apply**（GitOps対応） | Edgeのみ（クラウドに相当層なし） |
| 1. テナント設定 | 6設定領域（Settings/テナントオプション/ロール/Retention rules/認証/ブランディング） | **REST API による GET→JSON定義→POST/PUT**（go-c8y-cli でスクリプト化）。Enterpriseクラウドのみテナントポリシー併用可 | クラウド・Edge共通（Edgeは単一の edge テナントに直接注入） |
| 2. 初期コンテンツ | デバイス/グループ、ダッシュボード、スマートルール、Analytics Builderモデル、Webアプリ/マイクロサービス | 一括登録CSV、Cockpitダッシュボードの**JSONエクスポート/インポート**、モデルマネージャのJSONダウンロード/アップロード、go-c8y-cli、Inventory API | クラウド・Edge共通（要個別確認） |

- クラウド版の主要結論: 設定6領域はほぼすべてREST APIで注入可能。「UIで設定→エクスポート→インポート」の汎用機構は**テナント設定には存在しない**（ブランディングのみ例外）。✅
- **Edge**: 新Edge（2025/2026、Kubernetesネイティブ）はクラウドと同一ソフトウェア・同一アプリ構成のため、**フェーズ1のREST注入アプローチはEdgeでもそのまま有効**。ただし**厳格なシングルテナント**（management + edge の2固定テナントのみ）のため、**テナントポリシー/デフォルトサブスクリプションは使えない**。代わりにフェーズ0として **Edge CR というファイルベースの宣言的注入層**が加わる。✅
- **初期コンテンツ**: テナント設定と異なり、コンテンツ側には**公式のファイルベース入出力が複数存在**する（Cockpitダッシュボードの「JSONエクスポート/インポート」、Analytics BuilderモデルのJSONダウンロード/アップロード、デバイス一括登録CSV）。⚠️未検証（一次ソース引用あり、実機確認推奨）
- Migration Tool（テナント間コンテンツ移行ツール）は**2025-09-17にリポジトリがアーカイブ済み・非サポート**であり、本番セットアップ自動化の基盤には推奨しない。⚠️未検証

---

## 2. 設定領域カタログ（クラウド版・フェーズ1）

### 領域サマリ表

| # | 設定領域 | UI設定 | REST API | ファイルExport/Import | テナントポリシー注入 | 確信度 |
|---|---------|--------|----------|----------------------|---------------------|--------|
| 1 | Settingsメニュー（Application / Properties library / SMS provider / Connectivity / Localization / Feature toggles） | ○ Administration > Settings | △（対応するテナントオプションのキーは個別特定が未完了） | ✗ | —（テナントオプション化できる範囲で可） | ✅高 |
| 2 | テナントオプション | △（公式UIプラグイン、制約あり） | ○ `/tenant/options` | △（プラグイン管理下のオプションのみ） | ○ | ✅高 |
| 3 | 権限（グローバルロール / インベントリロール） | ○ Administration > Roles | ○ `/user/inventoryroles` ほか | ✗ | ✗ | ✅高 |
| 4 | データ保持（Retention rules） | ○ Management > Retention rules | ○ `/retention/retentions` | ✗ | ○ | ✅高 |
| 5 | 認証（ログインモード / パスワードポリシー / 2FA） | ○ Settings > Authentication | ○ `/tenant/loginOptions` | ✗ | ✗ | ✅高 |
| 6 | ブランディング（Enterprise限定） | ○ Settings > Branding | 未確認 | **○（テナント設定で唯一の公式Export/Import領域）** | ✗ | ✅中 |

### 2.1 Settingsメニュー（標準テナント）

**設定手段**: UI（Administration > Settings）。このドキュメントページにはエクスポート/インポート手段やREST API注入方法の記載はなく、UI操作手順のみ。

| カテゴリ | 設定内容 |
|---------|---------|
| Application | デフォルトアプリケーション（テナント内全ユーザー適用）、CORS等 |
| Properties library | インベントリオブジェクト・アラーム・イベント・テナントへのカスタムプロパティ定義。データ型 / 必須 / デフォルト値 / min-max / 正規表現の検証ルール付き |
| SMS provider | SMSプロバイダ設定 |
| Connectivity | 接続プロバイダ資格情報 |
| Localization | 言語・ローカライズ |
| Feature toggles | 機能の有効/無効切り替え |

> **注**: マイクロサービスの設定はテナントオプション（category = contextPath がデフォルト）として保存されるため、アプリケーションレベル設定の実体はテナントオプションである。Settingsメニュー各項目に1対1対応するテナントオプションのカテゴリ/キー名の特定は未完了（→ §7 未解決事項）。

出典: https://cumulocity.com/docs/standard-tenant/changing-settings/

### 2.2 テナントオプション

**設定手段**:
- REST API: `/tenant/options`（GET/POST/PUT/DELETE）、システムオプションは `/tenant/system/options`（読み取り）
- UIプラグイン: Tenant Option Management（現在は Cumulocity-IoT/cumulocity-ui-toolkit モノレポに移管。旧リポジトリ cumulocity-tenant-option-management-plugin は非推奨）

**UIプラグインの機能と制約**:
- テキスト/JSON形式のオプション作成、値の暗号化、一覧・検索・ソート・フィルタが可能
- **重大な制約**: プラグイン経由で作成（またはプラグインのインポート機能で取り込んだ）オプションしか表示・編集・削除できない。**UIの他画面で設定された任意の設定値を直接エクスポートすることはできない**
- 一括エクスポート/インポート機能の実用範囲はプラグイン管理下のオプションに限られる（「カテゴリ単位の一括JSON入出力で要件を満たせる」という主張は検証で棄却済み）

**セキュリティ注意**: `credentials.` プレフィックスのテナントオプションは、テナントポリシー経由ではポリシー値が暗号化されないため、**ポリシー経由で設定してはならない**（個別APIで直接設定する）。

出典: https://github.com/Cumulocity-IoT/cumulocity-ui-toolkit/tree/main/packages/tenant-option-management

### 2.3 権限（グローバルロール / インベントリロール）

**モデル**: 権限はユーザーへ直接付与せず、2種類のロールにグループ化して管理する。
- **グローバルロール**: システムレベル権限のセット
- **インベントリロール**: デバイスグループ単位の権限

**設定手段**:
- UI: Administration > Roles（※UIの「Copy inventory roles from another user」はテナント内ユーザー間コピーであり、ファイルエクスポートではない）
- REST API: OpenAPI仕様の Roles / Inventory Roles エンドポイント（`/user/inventoryroles`、`/user/users/{username}/inventoryroles` 等）

**エクスポート/インポート**: 公式には存在しない。設定値の抽出・再投入は **REST API経由が必須**。参考実装としてコミュニティ製 c8y-usergroup-migration（github.com/ButKor/c8y-usergroup-migration）がREST User API経由でユーザーグループ/ロールをバックアップ・コピーしている。

出典: https://cumulocity.com/docs/standard-tenant/managing-permissions/ 、https://cumulocity.com/api/core/#tag/Roles 、https://cumulocity.com/api/core/#tag/Inventory-Roles

### 2.4 データ保持（Retention rules）

**既定動作**: 履歴データは60日後に削除（既定値はプラットフォーム管理者がシステム設定で変更可能）。

**設定手段**:
- UI: Management > Retention rules
- REST API: `/retention/retentions`

**ルールのパラメータ**:

| パラメータ | 内容 |
|-----------|------|
| dataType | Alarms / Audits / Bulk operations / Events / Measurements / Operations |
| fragmentType | フラグメント種別で絞り込み |
| type | type で絞り込み |
| source | デバイスID で絞り込み |
| maximumAge | 最大保持日数（最大10年） |

**注意**: 保持ルールはファイルリポジトリには適用されない。作成には「Retention rules」のADMIN権限が必要。テナントポリシーに含めてサブテナント作成時に注入可能。

出典: https://cumulocity.com/docs/standard-tenant/managing-data/

### 2.5 認証

**設定手段**:
- UI: Settings > Authentication > Basic settings
- REST API: Tenant API `/tenant/loginOptions`（GET/POST/PUT）。SSOを含む全認証方式がこのAPIで構成される。2026年3月にはログインオプションマッピングのRESTエンドポイントも追加されており、APIカバレッジは拡大中。

**設定項目**:

| 項目 | 内容 |
|------|------|
| ログインモード | OAI-Secure（新規テナント既定・推奨）/ Basic Auth（互換性目的のみ）/ Single sign-on redirect（SSO構成時のみ） |
| パスワードポリシー | 有効期限・強度要件 |
| 二要素認証（TFA） | SMS / TOTP |

出典: https://cumulocity.com/docs/authentication/basic-settings/ 、https://cumulocity.com/docs/authentication/sso/

### 2.6 ブランディング（Enterpriseテナント限定）

**設定手段**: UI（Administration > Settings > Branding）

**設定項目**: ロゴ / ファビコン / タイトル、ブランドカラーと8シェード、ステータスカラー、タイポグラフィ、Cookieバナー、ライト/ダークテーマ、カスタムCSS、Advanced JSON

**エクスポート/インポート**: **テナント設定カテゴリ中、唯一公式にファイルベースのエクスポート/インポートが確認できた領域**。
- ブランディングバリアントはUIからエクスポート可能（ドキュメントに「export your existing variants beforehand」の記載）
- 2026リリース変更履歴に、ブランディング未設定の新規テナントへ既存ブランディングパッケージを直接インポートしてセットアップできることが明記
- ただしエクスポートの具体的手順・ファイル形式は現行ドキュメントに未記載（確信度: 中。検証票 2-1）

出典: https://cumulocity.com/docs/enterprise-tenant/customization/ 、https://cumulocity.com/docs/2026/change-logs/

### 2.7 テナント階層・テナントポリシー・デフォルトサブスクリプション（Enterpriseテナント限定）

- テナント階層は Standard / Enterprise / Management の3層。Enterpriseテナントのみサブテナントを作成でき、各サブテナントへのアプリケーション/機能のサブスクリプションを制御できる。
- **テナントポリシー** = テナントオプション＋保持ルールのセット定義。**サブテナント作成時に内容がテナントへコピーされる** — 基盤標準設定をセットアップ時に注入する純正機構として最有力。
  - **制約**: コピーは作成時の一回限り。ポリシーを後から編集しても既存テナントには伝播しない（既存テナントへの反映はManagementテナントからの明示的な更新操作が別途必要）。
- **デフォルトサブスクリプション**: 新規テナント作成時に既定でサブスクライブされるアプリ一覧。テナントポリシー/テナントオプションで上書き可能、Tenant APIで自動化可能。
- サブテナント構成（ドメイン、管理者資格情報、適用テナントポリシー、アプリケーションサブスクリプション、テナントオプション）は REST API（`POST /tenant/tenants` — domain/company/adminEmail/adminName/adminPass/customProperties、要 ROLE_TENANT_MANAGEMENT_ADMIN/CREATE）でプログラム管理できる。
- **注**: テナントポリシーの「適用」専用のパブリックRESTエンドポイントは今回の調査では引用確認できていない。ポリシー内容（オプション＋保持ルール）は個別APIで設定可能という位置づけ（→ §7 未解決事項）。

出典: https://cumulocity.com/docs/concepts/tenant-hierarchy/ 、https://cumulocity.com/docs/enterprise-tenant/managing-tenants/

---

## 3. Cumulocity Edge での適用（第2版追記・✅検証済み）

### 3.1 前提: 2つのEdge世代を混同しないこと

| 世代 | ドキュメント | セットアップ機構 |
|------|-------------|----------------|
| **新Edge（Kubernetesネイティブ、2025/2026）** | docs/2025/edge-kubernetes/、docs/2026/edge/ | Edge CR + operator（本節の主対象） |
| 旧Edge（VMアプライアンス、2024/10.x系） | docs/2024/edge/ | 専用Edge OpenAPI + SSHファイル編集（§3.6） |

### 3.2 新Edgeのインストール経路（✅ 3-0）

インストールは2経路のみ:
1. **c8yedge CLIツール** — 環境準備からインストールまでを自動化し、既存クラスタがない場合に **K3s**（軽量Kubernetes）上へEdgeをプロビジョニング
2. **自己管理Kubernetesクラスタへのデプロイ** — Edgeレジストリの **Helmチャート** で Edge operator をインストール（2026リリースは Kubernetes 1.34.x / Helm 3.x でテスト済み）

ライセンスファイルはプロダクトサポートから取得（会社名とドメイン名を提出。ドメインはFQDN不要）。コンテナレジストリ資格情報はライセンスと共に提供される。最小ハードウェア: 8 CPUコア / 16 GB RAM / x86-64 with AVX（MongoDB要件）/ 150 GB ディスク。オンライン・オフライン両インストール対応（クラウドはオンラインのみ）。

### 3.3 Edge Custom Resource — Edge固有のファイルベース宣言的注入層（✅ 3-0×5, 2-1×2）

**Edgeの主たるセットアップ時注入機構は Edge CR（c8yedge.yaml、kind: CumulocityIoTEdge、apiVersion: edge.cumulocity.com/v1）**。YAMLを編集して `kubectl apply -f` すると Edge operator がリコンサイルする（インストール / アップグレード・ダウングレード / スケーリング / ストレージ / バリデーション）。

- 必須フィールド: `version`、`domain`、`licenseKey`、`company`、`email`、`cumulocityPasswordSecretName`
- 公式ドキュメント自体が **GitOpsフレンドリー**（デプロイ状態全体をGitでバージョン管理可能）と位置づけ
- 運用中の変更: `kubectl get edge/c8yedge -n c8yedge -o yaml` → 編集 → 再apply。バージョンアップは `spec.version` のパッチで宣言的に実行
- インフラパラメータは `c8yedge config --set domain=<DOMAIN-NAME> --set-file licenseKey=<path/to/license.txt> --set-file tlsSecret.tls.key=... --set-file tlsSecret.tls.crt=...` でもスクリプト化可能
- **ライセンスキーはドメイン名に紐づく**ため、ドメイン変更とライセンス変更は同時に行う必要がある
- このファイルベースの宣言的チャネルは**クラウド版に相当物がない**（SaaSでは顧客がkubectl/CRにアクセスできない）

**⚠️ 重要なスコープ制約（複数の検証者が指摘）**: Edge CR がカバーするのは**デプロイ/インフラ構成のみ**（ライセンス、ドメイン、TLS、MongoDB、リソース制限、オプションコンポーネント）。**§2のテナントレベル設定（テナントオプション、ロール、保持ルール、Settingsメニュー項目）はCRに含まれず、Edge稼働後にREST API/UIで注入する必要がある**。つまりクラウド版のREST注入アプローチはEdgeでも必須であり、CRはその手前の追加層である。

### 3.4 インストール時限定の注入項目（✅ 3-0×3, 2-1×1）

初期 edge テナントと管理者IDは**インストール時にのみ**注入可能:

| 項目 | 手段 | 備考 |
|------|------|------|
| company（テナント名） | CR | 「インストール時のみ使用。既存インストールでは変更不可」と明記 |
| email（管理者ユーザー） | CR | 同上 |
| 初期管理者パスワード | Kubernetes Secret（`INITIAL_C8Y_ADMIN_PASSWORD` を含み `spec.cumulocityPasswordSecretName` で参照。デプロイ前に作成）または `c8yedge --cumulocity-password` フラグ | **インストール後のSecret変更は無視される** |

### 3.5 シングルテナント制約 — クラウド結論の適用可否（✅ 3-0×5）

**Edgeは厳格なシングルテナント**: 固定の2テナント（`management` と `edge`）のみで、サブテナント作成は不可（比較表に「Multi-tenancy: No; single tenant」）。

| クラウド版の結論 | Edgeでの適用 |
|----------------|-------------|
| 6設定領域のREST API注入 | **○ そのまま有効**（単一の edge テナントへ直接注入）。標準REST APIはEdgeでも利用可能 |
| テナントポリシー / デフォルトサブスクリプション | **✗ 適用不可**（サブテナント作成イベントが存在しない） |
| Managementテナントへのアクセス | **○ Edgeでは直接アクセス可**（`https://management-<domain>`）。パブリッククラウドのStandardテナントでは不可 — テナント階層機能の露出が両方向で異なる |
| アプリケーション構成（Cockpit / Device Management / Administration / Edge Agent / data broker / Streaming Analytics + Analytics Builder / OPC UA / マイクロサービスホスティング / Smart Rules） | **○ クラウドと同一ソフトウェア・同一アプリ**（"develop once, deploy anywhere"）。ただし「minor restrictions」の内訳はドキュメントに列挙されておらず、機能単位の完全パリティは個別検証が必要 |
| 水平スケール・ゼロダウンタイムアップグレード | ✗ なし（単一サーバ。サーバ障害はダウンタイム。目安 ~100 tps/CPUコア） |

**設計への含意**: Edgeでは標準設定の注入は edge テナントへREST APIで直接行う。テナントポリシーというショートカットはクラウドEnterprise限定。

### 3.6 オプションコンポーネント（✅ 3-0×2）

セットアップ自動化は**オプションコンポーネントの不在**を考慮する必要がある:

- **K8s版Edge**: Microservice Hosting / Machine Learning / DataHub / Messaging Service はオプトイン（Edge CRで有効化。例: DataHub は `spec.dataHub` を設定した場合のみインストールされ、追加で約10 CPUコア/10 GB RAMが必要。Messaging Service はマイクロサービス版 data broker と Notifications 2.0 に必須）
- カスタムマイクロサービスはK8s版Edgeでサポートされる（CRのMicroservicesセクションで構成。プライベートコンテナレジストリの構成が前提。最小要件に加えてリソースを明示的に割り当てる）
- **旧VM版Edge（2024）**: マイクロサービスホスティングはUIから明示的に有効化が必要で、有効化に10〜15分かかり（その間Edgeは非稼働）、ベースラインで4論理CPUコア/8 GB RAM以上が必要 — クラウドにない制約
- **注意（棄却済み主張）**: 「Edgeのマイクロサービスは CR 宣言の固定セット（apama-ctrl, smartrule, opcua-mgmt-service, databroker-agent-server）に限定される」という主張は **0-3で棄却**。固定セット限定とも完全に任意とも確定していない（→ §7）

### 3.7 旧VM版Edge（2024/10.x）固有の構成チャネル（✅ 3-0×2、レガシー対象時のみ関連）

- 専用の「Cumulocity Edge OpenAPI Specification」（cumulocity.com/api/edge/）があり、`/edge/configuration/{network, hostname, time-sync, microservices, security, domain, certificate}` というクラウドのテナントAPIとは別系統のRESTエンドポイントを持つ
- 一部設定はSSH経由の直接ファイル編集（Karaf setenv、ロギング、NGINX blocked-endpoints等）だが、**これらのファイル変更はEdgeアプライアンス更新で上書きされ、更新後に再適用が必要** — ファイルベース注入は永続的でない
- これらは新K8s版EdgeではCR/operatorモデルに置き換えられている

出典（§3全体）: https://cumulocity.com/docs/2025/edge-kubernetes/installing-edge-on-k8/ 、https://cumulocity.com/docs/2025/edge-kubernetes/edge-custom-resource-definition/ 、https://cumulocity.com/docs/2025/edge-kubernetes/manage-edge/ 、https://cumulocity.com/docs/2025/edge-kubernetes/k8-edge-introduction/ 、https://cumulocity.com/docs/2026/edge/installing-edge/ 、https://cumulocity.com/docs/2026/edge/edge-introduction/ 、https://cumulocity.com/docs/2024/edge/edge-configuration/ 、https://cumulocity.com/api/edge/ 、https://github.com/Cumulocity-IoT/edge-k8s-operator-docs（2024-09-13アーカイブ・情報陳腐化注意）

---

## 4. 初期コンテンツ（初期状態）の注入 — フェーズ2（第2版追記・⚠️大部分が未検証）

> **本節の確からしさについて**: 2回目調査ではコンテンツ注入系の主張が反証検証の上位25件枠から漏れたため、本節の大半は一次ソースからの引用付き抽出のみ（⚠️未検証）。**本節を実装根拠にする前に、実テナントでの動作確認を必ず行うこと**。検証済みなのは「これらのコンテンツ対象（ダッシュボード/Smart Rules/Analytics Builder/マイクロサービスホスティング）がEdge上にも存在する」という点のみ（✅ 3-0）。

### コンテンツ注入手段サマリ表（⚠️未検証含む）

| コンテンツ | ファイルベース入出力 | REST API | 備考 |
|-----------|-------------------|----------|------|
| デバイス/デバイスグループ | ○ 一括登録CSV | ○ Inventory API / New device requests API | CSVは再実行で既存デバイス更新（冪等に近い） |
| ダッシュボード | **○ CockpitのJSONエクスポート/インポート（公式UI機能）** | ○ Inventory API（c8y_Dashboard フラグメント） | テナント間・アセット種別間の共有可と明記 |
| スマートルール | ✗（Migration Toolは移行失敗バグ報告あり） | 未確認（/service/smartrule/smartrules は未検証） | Cockpitで作成・管理。apama-ctrl上で動作 |
| Analytics Builderモデル | **○ モデルマネージャのJSONダウンロード/アップロード（公式・テナント間移行用途と明記）** | 明示的なAPIドキュメントなし（実体はインベントリオブジェクト） | インポート後は常に非アクティブ → 有効化ステップが別途必要 |
| EPLアプリ（Apama .mon） | ○ .monファイルのインポート・アクティベート | — | Streaming Analyticsアプリから |
| Webアプリ/マイクロサービス | ○ ZIPアップロード（Administration > Own applications） | ○ Application API | go-c8y-cli が deploy をサポート |

### 4.1 デバイス・デバイスグループ（⚠️未検証）

- **一括登録CSV**（Device Management > Registering devices）: シンプルな2列形式（ID;PATH）では **PATHで参照されたグループがアップロード時に自動作成**される。フル形式は ID / CREDENTIALS に加え TYPE / NAME / ICCID / IDTYPE / PATH / SHELL / AUTH_TYPE 等の列をサポートし、各デバイスのmanaged object表現を作成する — 資格情報だけでなく初期デバイス状態の注入に使える
- **再実行時の挙動**: 既存の識別子を持つデバイスはCSVの内容で**更新**される（失敗しない）— 再実行可能なシーディングスクリプトに有利
- **REST**: 専用の New device requests API があり、UIなしで登録を自動化可能。未知の外部IDに遭遇した際の自動デバイス作成という経路もある
- **go-c8y-cli**: `c8y devicegroups create --name "..."` でグループ作成、`c8y inventory find --query "... and not(bygroupid(123456))" --includeAll | c8y devicegroups children assign --childType asset --id 123456 --workers 2 --progress` でパイプによる一括割り当て。`--silentStatusCodes 409` で重複エラーを無視すれば**実質的に冪等**な再実行可能スクリプトにできる。インベントリクエリ言語は `bygroupid(x)` 演算子をサポート

出典: https://cumulocity.com/docs/device-management-application/registering-devices/ 、https://goc8ycli.netlify.app/docs/examples/devicegroups/

### 4.2 Cockpitダッシュボード（⚠️未検証）

- **公式UIのJSONエクスポート/インポートが存在**: Cockpit公式ドキュメントに「ダッシュボードをJSONファイルへエクスポートし、JSONファイルからインポートできる。同一アセット種別間だけでなく、**異なるアセット種別間・異なるテナント間でも共有できる**」と明記。フェーズ2ではこれが「UIで設定→エクスポート→インポート」要望に最も直接応える機能（※テナント設定には同種機構がないのと対照的）
- UI内のコピー/ペースト（More… > Copy dashboard / Paste dashboard）でもオブジェクト間複製が可能
- **Dashboard template** オプション: あるデバイスタイプの全デバイスにダッシュボードを共有 — デバイスごとの注入が不要になる
- **RESTでの注入**: ダッシュボードはインベントリのmanaged objectであり、`c8y_Dashboard` フラグメント（icon / priority / name / global / isFrozen / children（ウィジェット定義のマップ））を持つ。Inventory API（POST）でプログラム作成可能。推奨ワークフローは「UIで手動作成 → JSONを確認 → API経由で再作成（一意識別フラグメント＋クエリフィルタで重複作成を防止）」（コミュニティ実装知見）
- **テナント間コピーの注意**: コピーは完全に切り離され、元ダッシュボードとの継承関係はない（親テナントをライブテンプレートにはできない）。**内部オブジェクトIDとカスタムウィジェット**が実務上の難所 — 対象テナントにウィジェットが存在し、IDを再マッピングしないと動作しない

出典: https://cumulocity.com/docs/cockpit/working-with-dashboards/ 、https://gist.github.com/confraria/3bf347750003310b72820321b6986172（コミュニティ）、https://community.cumulocity.com/t/copy-and-reuse-dashboards-in-subtenant-or-in-other-tenant/3091（コミュニティ）

### 4.3 スマートルール（⚠️未検証・最も情報が薄い領域）

- Cockpitアプリで作成・管理。Streaming Analyticsと同じ **apama-ctrl マイクロサービス**上で動作し、利用可能な分析機能はapama-ctrlのサブスクリプション階層に依存
- **REST APIでの作成可否は未確認**（`/service/smartrule/smartrules` エンドポイントの存在・ペイロード安定性は今回の調査では検証に至らず → §7）
- Migration Toolによるスマートルール移行は**失敗するというバグ報告**（GitHub Issue #21、2020-10、アセット割り当ての有無に関わらず失敗。修正確認なしでクローズ）

出典: https://cumulocity.com/docs/streaming-analytics/introduction-analytics/ 、https://github.com/SoftwareAG/cumulocity-migration-tool/issues/21

### 4.4 Analytics Builder モデル / EPLアプリ（⚠️未検証）

- **モデルマネージャからJSONフォーマットでダウンロード可能**で、ドキュメントが「**現在のテナントから別のテナントへモデルを移送する場合に有用**」と明示的に位置づけ。ダウンロード済みJSONはモデルマネージャからアップロード（インポート）可能 — JSONエクスポート/インポートが公式の移行経路
- **重要**: JSONファイルからインポートされたモデルは**常に非アクティブ状態**で取り込まれる → 自動注入には**別途アクティベーションステップが必須**
- モデルの実体はテナントのインベントリにJSON形式で保存される（原理的にはInventory REST APIで操作可能だが、公式ドキュメントにRESTエンドポイントやフラグメント構造の記載はない — デプロイはモデルマネージャUIのみ文書化）
- **EPLアプリ**（Apama .mon ベース）: Streaming Analyticsアプリでアクティベートするとデプロイされる。既存の *.mon ファイルをインポート可能 — CEPロジックのファイルベース注入経路
- Streaming Analytics（Analytics Builder含む）は**クラウドとEdge両方で利用可能** — このアプローチはEdgeにも通用する

出典: https://cumulocity.com/docs/streaming-analytics/analytics-builder/ 、https://cumulocity.com/docs/streaming-analytics/introduction-analytics/

### 4.5 Webアプリ / マイクロサービス（⚠️未検証）

- WebアプリはZIPとして Administration > Own applications からアップロード可能（Application APIでも可）
- **go-c8y-cli** が「deploy applications and microservices」をサポート — 新環境へのアプリ/マイクロサービス注入を自動化できる。またテンプレートからのオブジェクト作成（`create c8y items from templates`）、デバイス/ユーザー管理、コマンドチェーン/ワークフロー、セッション管理（MacOS/Linux/Windows対応）を備え、反復可能な一括セットアップスクリプトに適する

出典: https://goc8ycli.netlify.app/docs/introduction/

### 4.6 Cumulocity Migration Tool の評価（⚠️未検証、採用非推奨）

- README記載のスコープ: 「applications, dashboards, groups, devices, simulators, smart rules, images and managed objects をテナント間で移行するCumulocity Webアプリ」。ソースは3モード（現テナント / リモートテナント / **エクスポートファイル**）に対応し、移行前のオブジェクト確認・編集、create/updateモードを持つ — ファイルベースのゴールデンテンプレート運用が原理上は可能
- 必要権限: ソース側 Reader/Global Reader、宛先側 Administrator
- **しかし採用は非推奨**:
  - リポジトリは **2025-09-17 にアーカイブ済み**。「This tool is not further developed and it might break with upcoming Cumulocity releases. Use it at your own risk.」と明記
  - プリセールス/コミュニティ資産であり、製品サポート対象外（as-is・無保証）
  - スマートルール移行の失敗バグ（§4.3）、実環境でのCORSエラー報告（Allowed Domain "*" かつ両テナントadmin権限でも発生。v1016.0.256で観測）
  - 1回目調査では「Migration Toolがテナント設定のエクスポート/インポート要件を満たす」が0-3で棄却、2回目調査でもスコープ主張が0-3で棄却されている

出典: https://github.com/Cumulocity-IoT/cumulocity-migration-tool（アーカイブ済み）、https://tech.forums.softwareag.com/t/cumulocity-migration-tool/237289

### 4.7 対象外: Cockpitのデータエクスポート（⚠️未検証）

Cockpit組み込みのエクスポート機能（CSV/Excel、スケジュール実行、最大100万ドキュメント）がエクスポートするのは**運用データ（アラーム/イベント/メジャーメント/managed object）のみ**。ダッシュボード・スマートルール・テナント設定のエクスポート機構ではないため、環境シーディングには使えない。

出典: https://cumulocity.com/docs/cockpit/exports/

---

## 5. セットアップ時の標準設定注入 — 推奨アーキテクチャ

### フェーズ0: Edgeデプロイ（Edgeのみ）

- **Edge CR（c8yedge.yaml）をGit管理**し、`kubectl apply -f`（または c8yedge CLI）でインストール。ライセンス/ドメイン/TLS/初期管理者/オプションコンポーネント（Microservice Hosting等）をここで宣言
- company / email / 初期パスワードは**インストール時にしか注入できない**ため、CRとSecretの準備をセットアップ手順の最初に置く

### フェーズ1: テナント設定注入

**方式A: REST API による設定定義ファイル方式（クラウド・Edge共通の基本方式）**

1. リファレンステナントをUIで設定する
2. 各REST API（`/tenant/options`、`/user/inventoryroles`、`/retention/retentions`、`/tenant/loginOptions` 等）を **GET** して現行設定値を抽出
3. 抽出結果を設定定義ファイル（JSON）としてリポジトリ管理
4. セットアップスクリプトが各APIへ **POST/PUT** して新環境（Edgeでは単一の edge テナント）に注入

- CLIツール **go-c8y-cli**（https://goc8ycli.netlify.app/）がセッション管理・`c8y tenantoptions create`・JSON出力・冪等化（`--silentStatusCodes 409`）に対応しており、このワークフローのスクリプト化/CI-CD組み込みの最有力候補（※ツール自体の詳細評価は未実施 → §7）

**方式B: テナントポリシー＋デフォルトサブスクリプション（クラウドEnterprise限定。Edge不可）**

- テナントオプション＋保持ルールをテナントポリシーとして定義し、サブテナント作成時に自動コピー注入。デフォルトサブスクリプションで標準アプリ構成も自動化
- 適用範囲外の設定（ロール、認証、Properties library等）は方式Aで補完
- `credentials.` プレフィックスのオプションはポリシーに含めない（暗号化されないため）

**方式C: ブランディングパッケージ（クラウドEnterprise限定・ブランディングのみ）**

### フェーズ2: 初期コンテンツ注入（⚠️手段の大半が未検証 — 実機確認後に確定させる）

推奨候補順:
1. **デバイス/グループ**: 一括登録CSV（グループ自動作成・再実行で更新）または go-c8y-cli スクリプト
2. **ダッシュボード**: CockpitのJSONエクスポート/インポート（半自動）。完全自動化する場合は c8y_Dashboard managed object を Inventory API でPOST（内部ID・カスタムウィジェットの再マッピングに注意）。デバイスタイプ単位なら Dashboard template で注入回数を削減
3. **Analytics Builderモデル**: モデルマネージャのJSONダウンロード/アップロード＋アクティベーションステップ
4. **アプリ/マイクロサービス**: go-c8y-cli の deploy
5. **スマートルール**: 自動化手段が未確立（REST未確認・Migration Toolはバグ報告）。当面は手動構築またはREST実機調査後に判断
6. Migration Tool はアーカイブ済みのため基盤に組み込まない

---

## 6. できないこと・誤解しやすい点（検証で棄却された主張）

| 誤解 | 実際 | 検証票 |
|------|------|--------|
| Tenant Option Managementプラグインの一括JSON入出力で「UI設定→エクスポート→インポート」要件を満たせる | プラグイン管理下のオプションしか扱えず、他画面で設定された値はエクスポート不可 | 1-2で棄却 |
| 新規サブテナントは標準アプリ以外は空で始まる（＝全設定を作成後に注入する必要がある） | テナントポリシー・デフォルトサブスクリプションにより作成時点で注入可能 | 0-3で棄却 |
| Cumulocityの全外部通信はRESTである | MQTT等も存在。RESTの網羅性はあくまで設定管理面の話 | 0-3で棄却 |
| Cumulocity Migration Tool がテナント設定のエクスポート/インポート要件を満たす | 対象はアプリ/ダッシュボード/デバイス/スマートルール等のコンテンツであり、テナント設定（オプション、ロール、保持ルール、ブランディング等）は対象外。かつアーカイブ済み | 0-3で棄却 |
| （第2版）Edgeのマイクロサービスホスティングは CR 宣言の固定セット（apama-ctrl, smartrule, opcua-mgmt-service, databroker-agent-server）に限定される | 棄却。ただし「クラウド同様に完全任意」も未確定 — どちらの方向にも確定情報なし | 0-3で棄却 |
| （第2版）Migration Toolのスコープ列挙（apps/dashboards/groups/devices/simulators/smart rules/images/managed objects）が初期状態注入の要件を正確にカバーする | README記載のスコープ自体は存在するが、検証で棄却（アーカイブ・陳腐化・実動作の不一致が背景とみられる）。実用性は実機確認なしに主張できない | 0-3で棄却 |

---

## 7. 未解決事項（次の調査・検証タスク）

1. **Settingsメニュー各項目とテナントオプションの対応表**: Feature toggles・Localization・SMS provider・Connectivity資格情報などに1対1対応するREST API（テナントオプションのカテゴリ/キー名）の特定。実テナントでUI操作時のネットワークリクエストを観察して確定させるのが確実。
2. **テナントポリシーのREST API**: ポリシーそのもの（オプション＋保持ルールのセット定義）を作成・適用する公式エンドポイントの有無。
3. **ブランディングパッケージの実際の形式・手順**: 2026リリース変更後の実挙動、エクスポートファイル形式、API対応の有無。
4. **go-c8y-cli の実用評価**: GET→ファイル保存→別テナント適用のワークフローをどこまでカバーするか。実機評価が必要。
5. **（第2版）スマートルールのREST作成可否**: `/service/smartrule/smartrules` エンドポイントの存在・ペイロード形式の安定性・go-c8y-cliからのスクリプト化可否。
6. **（第2版）c8y_Dashboard の正確な構造検証**: フラグメント構造（`c8y_Dashboard!name!...` 等の命名を含む）、Inventory API POST/コピーで新テナント（Edge含む）に動作するダッシュボードが作れるかの実機確認。
7. **（第2版）Edge上でのTenant API挙動**: K8s版Edgeの edge テナントに対してクラウドと同一のTenant API（テナントオプション/ロール/保持ルール/アプリケーション）が同一挙動か。「minor restrictions」のうち設定注入に影響するもの（存在しない/読み取り専用のAdministration設定）の特定。
8. **（第2版）Migration Toolの現状**: アーカイブ後の実際の動作範囲・破損状況（フォークの有無を含む）。

---

## 8. 制約・注意事項（本書の限界）

- 本書はWeb上の公式ドキュメント・OpenAPI仕様・公式GitHubに基づく。**実機での動作検証は行っていない**。「REST APIで全設定を注入できる」は原理的可能性の確認。
- **§4（初期コンテンツ注入）は大部分が反証検証を経ていない**（検証枠の制約による）。一次ソース引用付きだが、設計判断の根拠にする前に実機確認すること。
- テナントポリシー・デフォルトサブスクリプション・ブランディングは **クラウドEnterpriseテナント限定**。単一標準テナント構成・**Edge**では使えない。
- **Edgeの2世代（VM版/K8s版）を混同しないこと**。§3.7はVM版のみに適用。K8s版の情報の一部はアーカイブ済みリポジトリ（edge-k8s-operator-docs、2024-09アーカイブ）由来で、現行CRDと軽微な差異がある（例: licenseKeyはインラインフィールド化）。
- Edgeアプリの「minor restrictions」はドキュメントに列挙がなく、機能単位のパリティ（特にAnalytics Builder/apama-ctrlのサイジング）は未検証。
- バージョン依存の数値（Kubernetes 1.34.x、リソース最小値等）は2025/2026リリース時点（2026-08現在）のもの。Edgeのリリースサイクルは速い。
- Tenant Option Managementプラグインの旧リポジトリは非推奨化済み。参照は ui-toolkit モノレポ側にすること。

---

## 9. 参照ソース一覧

### 公式ドキュメント（一次情報）— テナント設定
- https://cumulocity.com/docs/standard-tenant/changing-settings/ — Settingsメニュー全カテゴリ
- https://cumulocity.com/docs/standard-tenant/managing-permissions/ — 権限・ロール
- https://cumulocity.com/docs/standard-tenant/managing-data/ — Retention rules
- https://cumulocity.com/docs/authentication/basic-settings/ — 認証基本設定
- https://cumulocity.com/docs/authentication/sso/ — SSO・loginOptions API
- https://cumulocity.com/docs/concepts/tenant-hierarchy/ — テナント階層・テナントポリシー
- https://cumulocity.com/docs/enterprise-tenant/managing-tenants/ — サブテナント管理・デフォルトサブスクリプション
- https://cumulocity.com/docs/enterprise-tenant/customization/ — ブランディング
- https://cumulocity.com/docs/microservice-sdk/rest/ — マイクロサービス設定とテナントオプション
- https://cumulocity.com/docs/2026/change-logs/ — ブランディングパッケージインポート（2026）
- https://cumulocity.com/api/ — OpenAPI仕様ハブ（Tenant / Roles / Inventory Roles / Retention 等）

### 公式ドキュメント（一次情報）— Edge
- https://cumulocity.com/docs/2025/edge-kubernetes/installing-edge-on-k8/ — K8s版Edgeインストール
- https://cumulocity.com/docs/2025/edge-kubernetes/edge-custom-resource-definition/ — Edge CRリファレンス
- https://cumulocity.com/docs/2025/edge-kubernetes/manage-edge/ — Edge運用・c8yedge config
- https://cumulocity.com/docs/2025/edge-kubernetes/k8-edge-introduction/ — K8s版Edge概要・オプションコンポーネント
- https://cumulocity.com/docs/2026/edge/installing-edge/ 、https://cumulocity.com/docs/2026/edge/edge-introduction/ — 2026版Edge
- https://cumulocity.com/docs/2024/edge/edge-configuration/ 、https://cumulocity.com/api/edge/ — 旧VM版Edge構成・Edge OpenAPI
- https://github.com/Cumulocity-IoT/edge-k8s-operator-docs — Edge operator ドキュメント（2024-09-13アーカイブ）

### 公式ドキュメント（一次情報）— 初期コンテンツ
- https://cumulocity.com/docs/device-management-application/registering-devices/ — デバイス一括登録CSV
- https://cumulocity.com/docs/cockpit/working-with-dashboards/ — ダッシュボードJSONエクスポート/インポート・テンプレート
- https://cumulocity.com/docs/cockpit/exports/ — Cockpitデータエクスポート（運用データのみ）
- https://cumulocity.com/docs/streaming-analytics/analytics-builder/ — モデルのJSONダウンロード/アップロード
- https://cumulocity.com/docs/streaming-analytics/introduction-analytics/ — EPLアプリ・apama-ctrl・Edge対応

### 公式GitHub・ツール
- https://github.com/Cumulocity-IoT/cumulocity-ui-toolkit/tree/main/packages/tenant-option-management — Tenant Option Managementプラグイン（現行）
- https://github.com/Cumulocity-IoT/cumulocity-migration-tool — Migration Tool（**2025-09-17アーカイブ済み**・テナント設定は対象外）
- https://github.com/SoftwareAG/cumulocity-migration-tool/issues/21 — スマートルール移行失敗バグ
- https://github.com/reubenmiller/go-c8y-cli 、https://goc8ycli.netlify.app/ — go-c8y-cli（intro / devicegroups examples 含む）

### コミュニティ・裏付け
- https://community.cumulocity.com/t/exporting-branding-config/2289 — ブランディングエクスポート議論
- https://community.cumulocity.com/t/copy-and-reuse-dashboards-in-subtenant-or-in-other-tenant/3091 — テナント間ダッシュボードコピーの実務知見
- https://gist.github.com/confraria/3bf347750003310b72820321b6986172 — c8y_Dashboard フラグメント構造の実例
- https://tech.forums.softwareag.com/t/cumulocity-migration-tool/237289 — Migration Tool CORS問題
- https://developer.ntt.com/iot/docs/reference/retention-rules/ — NTT Things Cloud（OEM）のRetention rules APIリファレンス（裏付け）
- github.com/ButKor/c8y-usergroup-migration — ロール移行のコミュニティ実装例
