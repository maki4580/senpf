# Cumulocity 設定定義書 — 設定項目・設定手段・エクスポート/インポート方法

| 版 | 日付 | 内容 |
|----|------|------|
| 初版 | 2026-08-04 | クラウド版テナント設定の6領域・設定手段・エクスポート/インポート調査（deep-research: 情報源19件・主張92件抽出・25件検証・確定21/棄却4） |
| 第2版 | 2026-08-04 | Cumulocity Edge での適用可否（検証済み）と、初期コンテンツ注入段階（デバイス/ルール/ダッシュボード等）を追記（deep-research 2回目: 情報源21件・主張99件抽出・25件検証・確定23/棄却2。※コンテンツ注入系の主張は検証枠外となったため journal から救出し「未検証」マーク付きで記載） |
| 第3版 | 2026-08-04 | 付録A「カテゴリ別 設定項目一覧」を追加。公式ドキュメント10ページを直接取得し、個別設定項目をカテゴリごとに列挙 |
| 第4版 | 2026-08-04 | 網羅性検証のdeep-research（3回目: 確定24主張・棄却1）を反映。漏れていたカテゴリ（SSO / Data broker / Usage statistics・課金 / アラームマッピング / デバイス可用性 / ユーザー管理設定 / c8yedge CLI・Edge運用）を付録Aに追加し、既定値・許容値などの具体値を全カテゴリで補強 |
| **第5版** | **2026-08-05** | **[§10 第5版追補](#10-第5版追補--openapi-一次仕様ファイルの直読により確定した事項2026-08-05) を追加**（deep-research 4回目: 情報源26件・主張129件抽出・25件検証・確定17/棄却8＋一次仕様ファイル直読）。**Redoc SPA 問題により過去にEdge関連の主張が誤って棄却されていたことを訂正**し、Edge専用REST API全18本を仕様から確定。EPLアプリのREST API、`c8y-analytics` によるAnalytics Builder自動化、go-c8y-cliのカバレッジ、authConfig完全スキーマ、**ラウンドトリップ不可能な設定（`<<Encrypted>>` / `editable=false`）**を追加。§7未解決事項12件のうち3件解消・4件部分解消 |

情報源: cumulocity.com 公式ドキュメント（2024/2025/2026リリース版）、OpenAPI仕様（**第5版で生仕様ファイルを直接取得**）、Cumulocity-IoT 公式GitHubリポジトリ、コミュニティフォーラム

**検証状態の凡例**: ✅ = 反証検証を通過した確定事項（票数併記） / ⚠️未検証 = 一次ソースから引用付きで抽出済みだが反証検証を未実施（採用前に実機確認を推奨） / 📄 = 公式ドキュメント直接参照（2026-08-04取得。付録Aで使用）

> ⚠️ **第4版までの読者への注意**: §3.7（旧VM版Edgeの構成チャネル）と §4.3〜4.4（スマートルール／Analytics Builder／EPL）は、**§10 の追補で内容が大幅に更新・訂正されている**。該当箇所を参照する際は必ず §10 を併読すること。

> **個別の設定項目レベルの一覧は [付録A: カテゴリ別 設定項目一覧](#付録a-カテゴリ別-設定項目一覧) を参照。**
>
> **設定単位・エクスポート/インポート可否・変更時の影響まで含む一覧表は、別冊の [cumulocity-config-item-list.md](cumulocity-config-item-list.md) にまとめている。**

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
| 7 | SSO / Single sign-on（第4版追加。認証の一部） | ○ Settings > Authentication > Single sign-onタブ | ○ `/tenant/loginOptions` | ✗ | ✗ | ✅高 |
| 8 | Data broker（第4版追加） | ○ Data brokerアプリ | 未確認（要 feature-broker サブスクリプション） | ✗ | ✗ | ✅高 |
| 9 | Usage statistics / 課金（第4版追加） | ○ Administration > Usage statistics | △（billingModeはマイクロサービスマニフェスト） | △（CSVエクスポートは統計データのみ） | ✗ | ✅高 |
| 10 | アラームマッピング（第4版追加） | ○ Business Rules > Alarm mapping | 未確認（権限はOption management） | ✗ | ✗ | ⚠️ |
| 11 | デバイス可用性・ユーザー管理設定（第4版追加） | ○ Device Management / Administration > Users | ○（Inventory / User API） | ✗ | ✗ | ⚠️ |

> 7〜11は第4版の網羅性検証で発見された追加カテゴリ。詳細は付録A（A-1.13〜A-1.17、A-2.5）を参照。

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
| （第4版）SSOプロバイダテンプレートは Custom（既定）＋Azure AD / Keycloak 12.0.0+ / ADFS 3.0 の検証済みテンプレート | 2024版Docs由来の記載で、現行版でのテンプレート一覧としては棄却。実機/現行Docsで要再確認 | 1-2で棄却 |

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
9. **（第4版）テナントオプション/システムオプションの既定値カタログ**: 文書化された網羅一覧表は3回の調査でも発見できず。実テナントで `GET /tenant/system/options` / `GET /tenant/options` を実行して現物採取するのが確実。
10. **（第4版）SSOプロバイダテンプレートの現行一覧**: 2026版Docsでのテンプレート（Custom / Azure AD / Keycloak / ADFS等）と対応バージョン（当該主張は1-2で棄却されたため要再調査）。
11. **（第4版）OAI-Secureのトークン有効期間系オプション**: アクセストークン/リフレッシュトークン有効期間、パスワードポリシー（強度・履歴）関連テナントオプションのcategory/keyと既定値の文書化有無。
12. **（第4版）Notifications 2.0 / Storage quota警告閾値 / Edge管理テナント側設定**: 候補のまま未検証。公式Docs上の粒度の確認が必要。

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
- https://cumulocity.com/docs/sector/platform_administration/ — 管理カテゴリの全体像（第4版・網羅性検証の起点）
- https://cumulocity.com/docs/2026/authentication/sso/ 、https://cumulocity.com/docs/2024/authentication/sso/ — SSO構成（バージョン間差分の確認用）
- https://cumulocity.com/docs/standard-tenant/managing-users/ — ユーザー管理設定
- https://cumulocity.com/docs/data-broker/data-broker-application/ 、https://cumulocity.com/docs/data-broker/ms-data-broker/ — Data broker
- https://cumulocity.com/docs/enterprise-tenant/usage-and-billing/ — Usage statistics・課金
- https://cumulocity.com/docs/microservice-sdk/general-aspects/ — billingMode・リソース計測
- https://cumulocity.com/docs/standard-tenant/alarm-mapping/ — アラームマッピング
- https://cumulocity.com/docs/device-management-application/monitoring-and-controlling-devices/ — デバイス可用性
- https://cumulocity.com/docs/2025/edge-kubernetes/edge-operations/ — Edge運用（監視・バックアップ・アップグレード）

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

### 一次仕様ファイル（第5版で直接取得・2026-08-05）
- **https://cumulocity.com/api/core/dist/c8y-oas.yml** — Core Platform OpenAPI 生仕様（3.0.3 / `info.version: Latest` / 約1.4MB・33,598行）※ `https://cumulocity.com/api/core/` はRedoc SPAシェルのみを返すため必ずこちらを取得すること
- **https://cumulocity.com/api/edge/10.18.0/dist/c8y-edge-oas.json** — Cumulocity Edge OpenAPI 生仕様（3.0.3 / `Release 10.18.0` / 約128KB）
- https://cumulocity.com/api/ — 仕様インデックス（Core / DataHub / Data Preparation / DTM / Edge の5カテゴリ）

### 第5版で追加した出典
- https://cumulocity.com/docs/streaming-analytics/epl-apps/ — EPLアプリ（`/service/cep/eplfiles`）
- https://github.com/Cumulocity-IoT/apama-eplapps-tools — `eplapp.py` CLI・PySysテスト・CI/CD（Apache 2.0・as-is）
- https://github.com/Cumulocity-IoT/c8y-analytics — go-c8y-cli 拡張（Analytics Builder / EPL / Apama管理）
- https://github.com/Cumulocity-IoT/cumulocity-analytics-management — AB拡張（ブロック）の管理・GitHubからの動的ビルド
- https://github.com/Cumulocity-IoT/cumulocity-provision-sso — 親→子テナントのSSO設定複製マイクロサービス
- https://cumulocity.com/docs/streaming-analytics/analytics-customization/ — `streaminganalytics` テナントオプション・Apama-ctrl必要ロール
- https://cumulocity.com/docs/cockpit/smart-rules-collection/ — 組み込みスマートルールテンプレート11種・変数一覧
- https://cumulocity.com/docs/streaming-analytics/smart-rules-plugin/ — Smart rules (NEW) プラグイン
- http://resources.cumulocity.com/documentation/javasdk/1007.5.0/com/cumulocity/rest/representation/cep/SmartRuleRepresentation.html — `c8y_SmartRule` フラグメント構造
- https://api.github.com/repos/reubenmiller/go-c8y-cli/contents/api/spec/yaml — go-c8y-cli コマンド仕様ファイル全65件
- https://api.github.com/repos/reubenmiller/go-c8y-cli/releases/tags/v2.54.0 — SSO（Authorization Code / Device Flow / Device Certificate）対応
- https://goc8ycli.netlify.app/docs/concepts/paging/ — `--includeAll` の挙動
- https://github.com/AdeelChohan/Cumulocity-Postman-V2-Collection — Postman V2 collection（コミュニティ製・公式は保守終了）

---

## 10. 第5版追補 — OpenAPI 一次仕様ファイルの直読により確定した事項（2026-08-05）

> 本節は 4回目の deep-research（情報源26件・主張129件抽出・25件検証・確定17/棄却8）と、その後に実施した**一次仕様ファイルの直接取得**に基づく。第4版までで「未確定」「棄却」とされていた事項のうち、いくつかは**検証手法の欠陥による誤った棄却**だったことが判明したため、本節で訂正・確定させる。

### 10.1 【最重要】検証手法の落とし穴 — Redoc SPA 問題

`https://cumulocity.com/api/core/` および `https://cumulocity.com/api/edge/10.18.0/` は **Redoc の SPA シェル**であり、HTTPフェッチしても約4KBのローダHTMLしか返らない。ブラウザ上でJavaScriptが仕様ファイルを読み込んで初めて内容が描画される。

**このため「URLを開いて確認した」という検証は成立せず、第4版までの Edge 固有API に関する主張が3票検証で軒並み棄却された（0-3 / 1-2）。棄却は「そのAPIが存在しない」ことの証明ではなく、「提示URLから再現できなかった」だけだった。**

仕様の検証には**必ず生ファイルを直接取得**すること:

| 対象 | 生仕様ファイルURL | 形式・規模 |
|------|-----------------|-----------|
| Core Platform | `https://cumulocity.com/api/core/dist/c8y-oas.yml` | OpenAPI 3.0.3 / `info.version: Latest` / 約1.4MB・33,598行 |
| Edge（VMアプライアンス世代） | `https://cumulocity.com/api/edge/10.18.0/dist/c8y-edge-oas.json` | OpenAPI 3.0.3 / `info.version: Release 10.18.0` / 約128KB |

（Redocの `Redoc.init("dist/c8y-edge-oas.json", …)` 呼び出しがHTMLソース中にあり、そこから相対パスを特定できる。）

`https://cumulocity.com/api/` のインデックスは5カテゴリ構成 — Core platform（`./core`, `./core/2026`, `./core/2025`）、DataHub、Data Preparation、DTM、**Edge（`./edge/10.18.0` のみ。年次併記なし＝Edgeは別バージョン軸）**。✅ 3-0

---

### 10.2 Edge 専用 REST API — 全18エンドポイントを仕様から確定（§3.7 の全面差し替え）

第4版 §3.7 では「旧VM版Edge固有の構成チャネル」として概略のみ記載していたが、生仕様の直読により**全エンドポイント・全リクエストスキーマ・認証モデル・非同期実行モデルが確定した**。

> **適用範囲の注意**: この `/edge/...` API は **Release 10.18.0（VMアプライアンス世代）** の仕様である。K8sネイティブ世代（2025/2026）については Edge CR が主たる注入層であり（§3.3）、`/edge/...` API の継承有無は本仕様ファイルからは判断できない。**採用前に、導入対象のEdge世代で該当エンドポイントが生きているか実機確認すること。**

#### 10.2.1 エンドポイント一覧（全18）

| パス | メソッド | 用途 | 認証 |
|------|---------|------|------|
| `/edge/install` | POST | 初期インストール実行 | **不要** |
| `/edge/tasks/latest-installation` | GET | 直近インストールの情報取得 | **不要** |
| `/edge/configuration/domain` | GET / POST | ドメイン名の取得・変更（GETはインストール成功後のみ） | **不要** |
| `/edge/configuration/network` | GET / POST | ネットワーク構成 | 要（※インストール前は不要） |
| `/edge/tasks/{id}` | GET | タスク進捗の取得 | 要（※インストール前は不要） |
| `/edge/tasks/{id}/log` | GET | タスクログの取得 | 要（※インストール前は不要） |
| `/edge/version` | GET | 現在のEdgeバージョン（インストール成功後のみ） | 要 |
| `/edge/update` | POST | Edgeのバージョン更新 | 要 |
| `/edge/reboot` | POST | アプライアンス再起動 | 要 |
| `/edge/expand-disk` | POST | インストールディスク／データディスクの拡張 | 要 |
| `/edge/configuration/hostname` | GET / POST | ホスト名 | 要 |
| `/edge/configuration/time-sync` | GET / POST | 時刻同期（NTP） | 要 |
| `/edge/configuration/microservices` | GET / POST | マイクロサービスホスティングの有効化 | 要（**"Tenant Manager" ロール必須**） |
| `/edge/configuration/remote-connectivity` | GET / POST | リモートデバイス管理 | 要 |
| `/edge/configuration/certificate` | GET / POST | SSL証明書の有効性確認・更新 | 要 |
| `/edge/configuration/security` | GET / POST | OS／Kubernetes セキュリティ設定 | 要 |
| `/edge/diagnostics` | POST | 診断レポート作成の要求 | 要 |
| `/edge/diagnostics/{id}` | GET | 診断レポートのダウンロードURL取得 | 要 |

サーバURL: `https://<MY_OWN_DOMAIN>`（インストール前は自己署名証明書がIPアドレス向けのため、URLにはIPアドレスを使う。例 `https://192.168.66.10/edge/tasks/latest-installation`。インストール中に証明書がドメイン名向けに切り替わる）

#### 10.2.2 認証モデル

Management テナントによる **Basic認証**。ユーザー名に `management/` を前置する:

```bash
echo -n management/admin:password | base64
# → Authorization: Basic bWFuYWdlbWVudC9hZG1pbjpwYXNzd29yZA==
```

インストール前に限り、`/edge/tasks/latest-installation` `/edge/install` `/edge/configuration/network` `/edge/tasks/{id}` `/edge/tasks/{id}/log` は認証不要。`/edge/configuration/domain` は常に認証不要。

#### 10.2.3 非同期タスクモデル（自動投入スクリプトの設計制約）

- **POSTはすべてバックグラウンドタスクを起動し、タスクIDを返す**。進捗は `GET /edge/tasks/{id}`、詳細ログは `GET /edge/tasks/{id}/log` でポーリングする
- **タスクの同時実行は不可。実行中に別タスクを投げると HTTP 409 が返る**
  → **バッチ投入スクリプトは「逐次実行＋完了ポーリング」が必須**。並列化してはならない
- 上記2点は仕様書の Introduction に `>**Important:** You cannot run two tasks at the same time. If you attempt to run a task when another task is in progress, then you will get a HTTP status 409.` として明記

#### 10.2.4 ファイルアップロードの規則（ライセンス・証明書・更新アーカイブ）

- アップロード先URLは**タスク作成レスポンスの `uploads` 配列から読み取る**。`The URL layout is not static and can change anytime.` `You must not construct the URL.` — **URLを自前で組み立ててはならない**
- ヘッダ: `Content-Type: application/octet-stream` / `Content-Disposition: attachment; filename="<filename>"`（**ファイル名のみ。パスを含めてはならない**）
- **タイムアウト10秒**（タスク作成時、または最後にバイトを受信した時点から）。超過すると HTTP 404
  → **大きなファイル（`/edge/update` のアーカイブ等）はHTTPクライアントが全体をメモリにロードしてから送る実装だとタイムアウトする**。ファイルから直接ストリームすること。これができない場合は先にメモリへ読み込んでからエンドポイントを呼ぶ
- 応答: 成功201 / 同一タスクで再アップロードまたは `Content-Disposition` 不正は400 / タイムアウト超過は404

#### 10.2.5 リクエストJSONスキーマ（設定項目の実体）

**`POST /edge/install`** — インストール自体をJSONで自動化できる

```json
{
  "admin":        { "username": "…", "password": "…" },
  "root_password": "…",
  "edge_admin":   { "username": "…", "password": "…", "email": "…" },
  "domain_name":  "myown.iot.com",
  "certificate":  "generate"          // generate | upload
}
```
レスポンスはタスクIDと、ライセンス／証明書ファイルのアップロード先URL配列を返す。**このPOSTだけではインストールは始まらず、続けてライセンスと証明書をアップロードして初めて開始する**。インストール成功後、このエンドポイントは利用不可になる（失敗した場合は残るので再試行できる）。

**`POST /edge/configuration/network`**
```json
{ "address": "…", "netmask": "…", "gateway": "…", "dns": "…", "ip_range": "…" }
```
⚠️ `10.244.0.0` と `10.96.0.0` はEdge内部予約のため使用不可。

**`POST /edge/configuration/hostname`** → `{ "hostname": "…" }`

**`POST /edge/configuration/domain`** → `{ "domain_name": "…", "certificate": "generate" | "upload" }`

**`POST /edge/configuration/time-sync`** → `{ "enabled": bool, "interval_power_of_two": number, "servers": [ … ] }`

**`POST /edge/configuration/microservices`** → `{ "enabled": bool }`（"Tenant Manager" ロール必須。実行中アプライアンスは一時停止）

**`POST /edge/configuration/remote-connectivity`** → `{ "enabled": bool, "remote_tenant_url": "…" }`

**`POST /edge/configuration/certificate`** → `{ "renewal_type": "generate" | "upload" }`

**`POST /edge/update`** → `{ "type": "…" }`

**`POST /edge/configuration/security`** — クラウド版に相当物がないOS/K8sレベル設定

```json
{
  "OS": {
    "login_banner": "…",
    "login_sessions_inactivity_timeout_seconds": 600,
    "rsyslog":  { "server": "…", "port": 514, "protocol": "TCP" },   // TCP | UDP
    "audisp":   { "server": "…", "port": 60 },
    "ssh_enabled": true,
    "selinux_mode": "permissive",        // permissive | enforcing
    "audit_logging_enabled": false
  },
  "kubernetes": {
    "audit_policy": {
      "level": "None",                   // None | Metadata | Request | RequestResponse
      "max_size": 0, "max_backup": 0, "max_age": 0
    }
  }
}
```
> ⚠️ **`audit_logging_enabled` は不可逆**。`Once enabled, you cannot disable the audit logging configuration.` — 設定定義ファイルの既定値設計時に要注意。

出典: `https://cumulocity.com/api/edge/10.18.0/dist/c8y-edge-oas.json`（2026-08-05取得・全パス／全requestBodyスキーマを機械的に列挙して確認）

---

### 10.3 Core API — 設定投入に関わるエンドポイント目録（仕様から機械抽出・全149パス中の該当分）

`c8y-oas.yml` の `paths:` を全件抽出した結果。**設定注入スクリプトが叩く先はこの範囲で尽きている**（スマートルール・Analytics Builder・Apama EPL は Core 仕様に**含まれない** → §10.6〜10.8）。

| 領域 | エンドポイント |
|------|--------------|
| テナント設定 | `/tenant/options`、`/tenant/options/{category}`、`/tenant/options/{category}/{key}`、`/tenant/options/{category}/{key}/editable`、`/tenant/system/options`、`/tenant/system/options/{category}/{key}` |
| テナント本体 | `/tenant/tenants`、`/tenant/currentTenant`、`/tenant/tenants/{tenantId}`、`/tenant/tenants/{tenantId}/applications`、同 `/{applicationId}`、`/tenant/tenants/{tenantId}/applications/restricted-roles`（+`/{roleId}`） |
| 認証・SSO | `/tenant/loginOptions`、`/tenant/loginOptions/{typeOrId}`、`…/accessMappings`（+`/{id}`）、`…/inventoryAccessMappings`（+`/{id}`）、`…/restrict`、`/tenant/oauth`、`/tenant/oauth/token`、`/tenant/oauth/certificate` |
| TFA | `/tenant/tenants/{tenantId}/tfa`、`/user/{tenantId}/users/{userId}/tfa`、`/user/currentUser/totpSecret`（+`/activity`、`/verify`） |
| 証明書 | `/tenant/tenants/{tenantId}/trusted-certificates`（+`/bulk`、`/{fingerprint}`）、`/tenant/tenants/{tenantId}/trusted-certificates-pop/{fingerprint}/{pop\|confirmed\|verification-code}`、`/tenant/trusted-certificates/verify-cert-chain`、`/tenant/trusted-certificates/settings/crl`、`/certificate-authority`、`/certificate-authority/renew`、`/.well-known/est/simpleenroll`、`/.well-known/est/simplereenroll` |
| ユーザー・ロール | `/user/roles`、`/user/roles/{name}`、`/user/inventoryroles`（+`/{id}`）、`/user/devicePermissions/{id}`、`/user/{tenantId}/users`（+`/{userId}`）、`/user/{tenantId}/groups`（+`/{groupId}`、`/{groupId}/roles`、`/{groupId}/roles/{roleId}`、`/{groupId}/users`）、`/user/{tenantId}/users/{userId}/{groups\|roles\|roles/inventory}`、`/user/{tenantId}/userByName/{username}`、`/user/{tenantId}/groupByName/{groupName}`、`/user/logout`、`/user/logout/{tenantId}/allUsers`、`/user/currentUser`、`/user/currentUser/password` |
| データ保持 | `/retention/retentions`、`/retention/retentions/{id}` |
| 通知 | `/notification2/subscriptions`（+`/{id}`）、`/notification2/token`、`/notification2/unsubscribe`、`/notification/realtime` |
| フィーチャートグル | `/features`、`/features/{featureKey}`、`/features/{featureKey}/by-tenant`（+`/{tenantId}`） |
| アプリケーション | `/application/applications`（+`/{id}`、`/{id}/binaries`、`/{id}/versions`、`/{id}/clone`、`/{id}/bootstrapUser`）、`/application/applicationsBy{Name\|Tenant\|Owner\|User}/…`、`/application/currentApplication`（+`/settings`、`/subscriptions`） |
| 統計・課金 | `/tenant/statistics`（+`/summary`、`/allTenantsSummary`、`/files`、`/files/{id}`、`/files/latest/{month}`）、`/tenant/statistics/device/{tenantId}/{monthly\|daily}/{date}` |
| デバイス登録・資格情報 | `/devicecontrol/deviceCredentials`、`/devicecontrol/newDeviceRequests`（+`/{requestId}`）、`/devicecontrol/bulkNewDeviceRequests`、`/devicecontrol/deviceAccessToken` |
| その他（設定の器） | `/inventory/managedObjects`（ダッシュボード・スマートルール・ABモデル等の実体）、`/identity/…`、`/audit/auditRecords`、`/alarm/alarms`、`/event/events`、`/measurement/measurements` |

出典: `https://cumulocity.com/api/core/dist/c8y-oas.yml`（2026-08-05取得・`paths:` を全件抽出）

---

### 10.4 テナントオプション API の仕様上の確定事項

- **モデル**: `Options are category-key-value tuples which store tenant configurations. Some categories of options allow the creation of new ones, while others are limited to predefined set of keys.` — カテゴリによっては新規キー作成が可能、カテゴリによっては定義済みキー限定 ✅ 3-0
- **必要ロール**: 更新 `ROLE_OPTION_MANAGEMENT_ADMIN` / 参照 `ROLE_OPTION_MANAGEMENT_READ`
- **一括更新**: `PUT /tenant/options/{category}` でカテゴリ単位の一括更新（`updateOptionsByCategory`）。**設定定義ファイルをカテゴリ単位のJSONで保持する設計と直接対応する**
- 仕様書に例示されている既定カテゴリ: `access.control`（キー `allow.origin`、既定値 `*`、定義済みキー限定）、`alarm.type.mapping`（`<ALARM_TYPE>` をキーに `severity|text` を上書き、新規キー作成可）

---

### 10.5 【設計上の重要制約】ラウンドトリップ不可能な設定 — 「全設定をJSONダンプして別テナントに完全再現」は原理的に不可能

「GUIから設定値を抽出してJSON化し、それを注入する」という本プロジェクトの構想に対する**明確な限界**が2つ、仕様上確定した。

#### (1) `credentials.` プレフィックス付きオプションは復号できない

- キーに `credentials.` を付けると値は**暗号化して保存**される
- 所有マイクロサービス以外が GET すると、値は固定文字列 **`"<<Encrypted>>"`** で返る
  ```json
  { "category": "microservice2", "key": "credentials.secret", "value": "<<Encrypted>>" }
  ```
- 所有者判定はマイクロサービスマニフェストの `settingsCategory` → コンテキストパス → マイクロサービス名 の優先順。復号は**オーナーのシステムユーザー（サービスユーザー／ブートストラップユーザー）に対してのみ**行われる
- `GET /tenant/system/options` も同様に難読化される

**含意**: SSOのクライアントシークレット、SMTPパスワード、コネクティビティ資格情報などの**秘匿値はエクスポートで採取できない**。設定定義ファイルには「キーの存在」だけを持たせ、**実値は別管理（Vault／CI/CDのSecret）とし、投入スクリプトが注入時に埋める**設計が必須。✅ 2-1

#### (2) management テナントが `editable=false` にしたオプションは403で上書きできない

- `Any option of any tenant can be defined as "non-editable" by the "management" tenant; once done, any PUT or DELETE requests made on that option by the tenant owner will result in a 403 error.`
- 設定は `PUT /tenant/options/{category}/{key}/editable?targetTenant={targetTenant}`（`ROLE_OPTION_MANAGEMENT_ADMIN` かつ management テナントであること）
- リテンションルールにも同様の `editable` フラグがあり「Management テナントのみが更新可能」と定義されている

**含意**: 投入スクリプトは403を「異常終了」ではなく「意図的にロックされた項目」として扱えるようにし、**どの項目がロック済みかを可視化する**必要がある。

#### (3) テナントポリシーのオプションは暗号化されない

- `Tenant options specified in a tenant policy are **not encrypted**.` かつ `You should not specify or overwrite tenant options here with a 'credentials.' prefix, since the platform expects those options to be encrypted with data that will appear after the tenant has been created.`
- これは推奨ではなく**仕様書自身の明示的な禁止事項**。第4版 §5「方式B」の注記の根拠が確定した ✅ 3-0
- 併せて確定: テナントポリシーは**作成時コピー**方式（`The options and rules are copied into the tenant. Editing the policy has no effect on tenants that have already been created.`）。**ポリシーを後から編集しても既存テナントには反映されない** ✅ 3-0
- なお、**テナントポリシーそのものを操作するREST エンドポイント（`/tenant/policies` 等）は Core 仕様にも公式ドキュメントにも存在が確認できなかった** → §7-2 は「GUI専用機能である可能性が高い」に更新

---

### 10.6 【§7-5 解消】EPL アプリ（Apama）の REST API — `/service/cep/eplfiles`

第4版 §4.4 で「EPLアプリはUIからのインポートのみ」としていたが、**完全なCRUD REST APIが存在する**。

| メソッド | パス | 用途 |
|---------|------|------|
| GET | `/service/cep/eplfiles` | 全EPLアプリの一覧 |
| GET | `/service/cep/eplfiles?contents=true` | **EPLソース本文を含めて取得＝設定ダンプの起点** |
| POST | `/service/cep/eplfiles` | 新規デプロイ |
| PUT | `/service/cep/eplfiles/{id}` | 更新（`{"state":"inactive"}` だけ送って無効化も可） |
| DELETE | `/service/cep/eplfiles/{id}` | 削除 |

**投入JSONスキーマ**（4フィールド）:
```json
{
  "name": "MyRule",
  "description": "",
  "state": "active",          // active | inactive
  "contents": "<*.monファイルの中身をそのまま文字列で>"
}
```
レスポンスには加えて `eplPackageName` / `errors` / `warnings` が含まれる。アクティベート時に `*.mon` へ一意のパッケージ名が割り当てられる。

- **公式CLI**: `eplapp.py`（`Cumulocity-IoT/apama-eplapps-tools`、Apache 2.0・as-is無保証・Python 3.7+）が `list` / `deploy` / `delete` / `update` を提供。`-r` で既存アプリの上書き再デプロイ、`-i` で非アクティブ状態デプロイ。認証は `--cumulocity_url` / `--username` / `--password`
- **CI/CD対応が公式に明記**: `As well as running tests locally during development, you can script deployment and testing of your EPL apps for CI/CD purposes.`（環境変数 `CUMULOCITY_SERVER_URL` / `CUMULOCITY_USERNAME` / `CUMULOCITY_PASSWORD`）
- **権限設計上の注意**: EPL Appsページの利用には CEP Manager ロールが必要で、公式ドキュメントが `A user who has ADMIN permission for 'CEP management' is able to create and activate EPL apps and thus also has almost full control over the current tenant.` と警告している。**自動投入用サービスアカウントにこの権限を与えることは事実上テナント全権の付与に等しい**

出典: `https://cumulocity.com/docs/streaming-analytics/epl-apps/`、`https://github.com/Cumulocity-IoT/apama-eplapps-tools`

---

### 10.7 【§7-5 部分解消】スマートルールの実体 — `c8y_SmartRule` マネージドオブジェクト

専用RESTエンドポイントの公式ドキュメントは**依然として発見できない**（Cockpit のドキュメントはGUI手順のみで、エンドポイント・マネージドオブジェクト型・JSONペイロード・エクスポート/インポート・CLIのいずれにも言及なし）。ただし **Java SDK の `SmartRuleRepresentation`（`c8y.SmartRuleRepresentation` フラグメントを表す）から構造が判明**した:

| フィールド | 型 | 意味 |
|-----------|-----|------|
| `ruleTemplateName` | String | スマートルールテンプレート名（シナリオ名） |
| `name` | String | ルール名 |
| `type` | String | 型 |
| `config` | Map<String,Object> | テンプレートごとのパラメータ（閾値、宛先メールアドレス等） |
| `enabledSources` | List<String> | 適用対象アセットのID群 |
| `disabledSources` | List<String> | 除外アセットのID群 |
| `enabled` | Boolean | 有効／無効 |
| `cepModuleId` | GId | Esperモジュール managed object ID または Apama シナリオインスタンスID |
| `id` | GId | スマートルール cep module の managed object ID |
| `body` | String | Esperモジュールのパース済みボディ |

**設計含意**: スマートルールは Inventory 上のマネージドオブジェクトであるため、`GET /inventory/managedObjects?fragmentType=c8y_SmartRule` でのダンプと、Inventory API での再作成が**原理的には**可能。ただし `cepModuleId` / `id` は環境固有のIDであり、テナント間移送時は再マッピングが必要。また `enabledSources` もデバイスIDを含むため、コンテンツ注入順序（デバイス→スマートルール）に依存する。**この経路は公式にドキュメント化されていないため、実機検証が必須**。

**前提条件（テナント側の事前設定）**: テナントが **Smartrule マイクロサービスと Apama-ctrl マイクロサービスの両方にサブスクライブ**されていること。

**権限**: グローバルスマートルールの閲覧は「Global smart rules」または「CEP management」のREAD＋「Inventory」のREAD、作成/編集/削除はADMIN。ローカルスマートルールはインベントリ権限（またはインベントリロール）のみで制御され、専用権限は不要。

**組み込みテンプレート（11種）**: On alarm send SMS / On alarm send email / On alarm escalate it / On alarm duration increase severity / On geofence create alarm / On geofence send email / Calculate energy consumption / On missing measurements create alarm / On alarm execute operation / On measurement threshold create alarm / On measurement explicit threshold create alarm。メッセージ内の変数は `#{id}` `#{type}` `#{source}` `#{time}` `#{text}`、アラーム系は `#{status}` `#{severity}` `#{count}`、計測値系は `#{valueFragment}` `#{valueSeries}` `#{value}` `#{unit}`、ネストは `#{X.Y}` / `#{X.Y.Z}`。

> **「Smart rules (NEW)」プラグインは別物**: Analytics Builder のモデルインスタンスをデバイス／グループ文脈から作成・管理する新方式で、実体は **Analytics Builder のモデル＋インスタンス**。権限はグローバルロール配下の「Smart rule instances」（READ / ADMIN）。前提はStreaming Analyticsマイクロサービスのサブスクリプション＋対象アプリへのプラグイン導入。**従来のスマートルールとは投入経路が異なるため、どちらを標準採用するか設計判断が必要。**

出典: `https://cumulocity.com/docs/cockpit/smart-rules/`、`https://cumulocity.com/docs/cockpit/smart-rules-collection/`、`https://cumulocity.com/docs/streaming-analytics/smart-rules-plugin/`、`http://resources.cumulocity.com/documentation/javasdk/1007.5.0/com/cumulocity/rest/representation/cep/SmartRuleRepresentation.html`

---

### 10.8 【§7 追加解消】Analytics Builder / Streaming Analytics の自動投入 — `c8y-analytics` 拡張

Analytics Builder の公式REST APIドキュメントは存在しない（コミュニティフォーラムでも「Dev toolsでUIの通信を見るしかない」との回答）が、**Cumulocity-IoT 公式GitHub org が go-c8y-cli 拡張 `c8y-analytics` を公開しており、これが実質的な自動投入経路**となる。

導入: `c8y extension install Cumulocity-IoT/c8y-analytics`

| 対象 | コマンド例 |
|------|-----------|
| ABモデル一覧 | `c8y analytics ab list` |
| ABモデルのJSONダンプ | `c8y analytics ab get --id 5379731810 --outputFileRaw iir.json` |
| ABモデルの投入 | `c8y analytics ab create --name iir2 --template iir.json` |
| ABモデルの状態変更 | `c8y analytics ab update --id 1579830237 --state ACTIVE` |
| ABモデル削除 | `c8y analytics ab delete --id 1579830237` |
| EPLアプリ一覧 | `c8y analytics epl list --includeAll` |
| テンプレートモデルのインスタンス | `c8y analytics instances list --id 2579780855` / `update` / `delete` |
| Apamaマイクロサービス管理 | `c8y analytics management restart` / `diagnosticsEnhanced` |
| Streaming Analyticsのテナントオプション | `c8y analytics configuration update --key analytics.builder/timedelay_secs --value 180` |
| AB拡張（ブロック）の入出力 | `c8y analytics extensions {list\|get\|delete\|download\|upload}`（例: `--outputFileRaw MyBlocks.zip` / `--file MyBlocks.zip --name MyBlocks`） |

**モデルの `mode`（DRAFT / TEST / SIMULATION / PRODUCTION）と `state`（ACTIVE / INACTIVE）もCLIから制御できる** — 第4版 §4.4 の「インポート後は常に非アクティブ → 別途アクティベーションが必要」という制約が、CLIで自動化可能になる。

**AB拡張（ブロック）の注意**: アップロードした拡張は即時反映されず、**Streaming Analytics エンジンの再起動（"Restart to deploy extension" ボタン／`c8y analytics management restart`）が必要**。自動投入手順にこのステップを組み込むこと。

**Apama-ctrl マイクロサービスの必要ロール**（サービスユーザー権限設計の参考）: `ROLE_APPLICATION_MANAGEMENT_READ, ROLE_INVENTORY_READ/ADMIN/CREATE, ROLE_MEASUREMENT_READ/ADMIN, ROLE_EVENT_READ/ADMIN, ROLE_ALARM_READ/ADMIN, ROLE_DEVICE_CONTROL_READ/ADMIN, ROLE_IDENTITY_READ/ADMIN, ROLE_CEP_MANAGEMENT_READ/ADMIN, ROLE_OPTION_MANAGEMENT_READ, ROLE_SMS_ADMIN, ROLE_AUDIT_READ/ADMIN, ROLE_USER_MANAGEMENT_READ, ROLE_USER_MANAGEMENT_OWN_READ, ROLE_TENANT_MANAGEMENT_READ, ROLE_BULK_OPERATION_READ/ADMIN, ROLE_NOTIFICATION_2_ADMIN`

出典: `https://github.com/Cumulocity-IoT/c8y-analytics`、`https://cumulocity.com/docs/streaming-analytics/analytics-customization/`、`https://cumulocity.com/docs/streaming-analytics/analytics-builder/`

---

### 10.9 【§7-4 解消】go-c8y-cli のカバレッジ確定

`api/spec/yaml/` の全65ファイルを GitHub Contents API で列挙（master、2026-08-05時点）。**コマンドはこのYAML仕様から自動生成**される（`api/spec/yaml/` に仕様を足して `task build` すれば未対応エンドポイント向けコマンドを自作できる）。

**カバーされている（生成済みサブコマンドあり）**:
`tenantOptions` `systemOptions` `currentTenant` `tenants` `tenantApplications` `tenantsTFA` `tenantStatistics` `userRoles` `userGroups` `users` `userReferences` `currentUser` `retentionRules` `notification2` `notification2Subscriptions` `notification2Tokens` `microservices` `microservicesLoglevels` `applications` `applicationVersions` `currentApplication` `identity` `features` `featuresByTenant` `certificateAuthority` `devicecertificates` `deviceCredentials` `deviceProfile` `remoteAccessConfigurations` `databroker` `datahub*` `uiPlugins` `smartgroups` `inventory` 系 ほか

**カバーされていない（＝汎用 `c8y api` か別手段で補う必要がある）**:
- **`loginOptions`（SSO/OAuth2）** — 専用仕様ファイルなし。`tenants.yaml`（`getTenantCollection` / `newTenant` / `getTenant` / `deleteTenant` / `updateTenant` / `enableApplicationOnTenant` / `disableApplicationFromTenant` / `getApplicationReferenceCollection` のみ）にも `currentTenant.yaml`（`currentTenant` / `listApplications` / `getTenantVersion` のみ）にも `/tenant/loginOptions` の参照はない
- **`smartRules`** — アルファベット順で `smartgroups.yaml` の直後が `software.yaml`。`smartrules` 相当が欠落
- **EPL / Apama / Analytics Builder** — `c8y-analytics` **拡張**で補う（§10.8）

✅ 3-0

**IaC運用に使える主要フラグ（グローバル）**:

| フラグ | 用途 |
|-------|------|
| `--outputFile` | select/view 適用後のJSONをファイル保存＝**設定ダンプ** |
| `--outputFileRaw` | 生レスポンスを保存 |
| `--includeAll` | ページングを自動反復して全件取得（**pageSize を自動的に最大値2000にするため `--pageSize` の明示は不要**） |
| `-d, --data` | JSON または shorthand JSON（`--data 'value1=1,my.nested.value=100'`）。**`@file.json` のファイル指定は不可** |
| `--template` / `--templateVars` | jsonnet テンプレートによるボディ生成。**ファイルでJSONを保持するのはこちらの役割** |
| `--dry` / `--dryFormat` | 送信せずドライラン。`json` / `dump` / `markdown`（既定） / **`curl`** — **等価なcurlコマンドを抽出できるため、CLI非依存の投入スクリプトへの変換にも使える** |
| `-o/--output` `--select` `--flatten` | table/json/csv/csvheader 出力、プロパティ抽出、ネストJSONのドット記法平坦化 |
| `--silentStatusCodes 409` | 重複エラーを黙殺し、実質的に冪等な再実行を実現（第4版 §4.1 で既出） |

**パイプライン一括投入**: `c8y tenantoptions create --category … --key …` の `--key` は `accepts pipeline` のため、Plain text / JSON Lines / CSV を標準入力から流し込める。テンプレート内で `input.value` を参照すると行ごとに異なる key/value を投入できる（各行が個別にテンプレート処理される）。
※ `c8y tenantoptions update` は category が URLパスパラメータのため、**1回のパイプ実行は実質1カテゴリに限定**される。

**認証（CI/CD での重要事項）**: v2.53.0（2026-01-29）で Device Flow、**v2.54.0（2026-03-18）で Authorization Code（ブラウザ）フローと Device Certificate(x509) ログイン**に対応。`c8y sessions create --loginType BROWSER` / `--loginType DEVICE`。
- 前提: テナント側SSO設定で「Redirect to the user interface application」が有効／IdP側で `http://localhost:5001/callback`（`--browserCallback` で変更可）が許可されていること
- **Authorization Code / Device Flow はいずれも初回ログインが対話的**。**完全無人のCI/CDには Device Certificate(x509) か、非SSOのサービスユーザーが必要** ✅ 2-1

**位置づけの注意**: 元来 Reuben Miller 個人プロジェクト（MIT、"Unofficial Cumulocity IoT Command Line Tool"）で、現在は Cumulocity GitHub org 配下にあるが**Cumulocity製品ラインの正式提供物ではない**。またプラットフォーム仕様との乖離が実在する（例: `c8y retentionrules create` の `dataType` 許容値は `ALARM/AUDIT/EVENT/MEASUREMENT/OPERATION/*` で **`BULK_OPERATION` が欠落**。プラットフォーム側の Core 仕様には存在する）。**CLIのフラグ検証を仕様の正としてはいけない。**

出典: `https://api.github.com/repos/reubenmiller/go-c8y-cli/contents/api/spec/yaml`、`https://goc8ycli.netlify.app/docs/cli/c8y/c8y/`、`https://goc8ycli.netlify.app/docs/cli/c8y/tenantoptions/…`、`https://goc8ycli.netlify.app/docs/concepts/paging/`、`https://api.github.com/repos/reubenmiller/go-c8y-cli/releases/tags/v2.54.0`

---

### 10.10 `loginOptions` の `authConfig` 完全スキーマ（SSO設定のJSON化）

`POST /tenant/loginOptions`
- Content-Type: **`application/vnd.com.nsn.cumulocity.authconfig+json`**
- 必要ロール: `ROLE_TENANT_ADMIN` **または** `ROLE_TENANT_MANAGEMENT_ADMIN`
- required: `type`, `providerName`

| フィールド | 内容 |
|-----------|------|
| `type` | `BASIC` / `OAUTH2` / `OAUTH2_INTERNAL`（大文字小文字非依存） |
| `providerName` | プロバイダ表示名 |
| `userManagementSource` | `INTERNAL` / `REMOTE` |
| `grantType` | `AUTHORIZATION_CODE` / `PASSWORD` |
| `sessionConfiguration` | `absoluteTimeoutMillis` / `maximumNumberOfParallelSessions` / `renewalTimeoutMillis` / `userAgentValidationRequired` |
| `authorizationRequest` / `tokenRequest` / `refreshRequest` / `logoutRequest` | 各リクエストのURL・メソッド・ヘッダ・ボディテンプレート |
| `signatureVerificationConfig` | `aad` / `adfsManifest` / `jwks` / `manual` の4方式 |
| `userIdConfig` / `accessTokenToUserDataMapping` | トークンクレーム→ユーザー属性のマッピング |
| `onNewUser.dynamicMapping` | 新規ユーザーへのグループ／アプリ／インベントリロールの動的マッピング |
| `externalTokenConfig` | 外部トークン検証。`validationMethod`: `INTROSPECTION` / `USERINFO` |
| `authenticationRestrictions` | 認証制限 |
| `onlyManagementTenantAccess` | managementテナントのみアクセス許可 |

付随エンドポイント: `/tenant/loginOptions/{typeOrId}`（GET/PUT/DELETE）、`…/accessMappings`（+`/{id}`）、`…/inventoryAccessMappings`（+`/{id}`）、`…/restrict`（PUT）。**トップレベルの loginOptions 系パスはちょうど7本**。

- **SAMLは非対応**。CumulocityのSSOは **OAuth2 の authorization code grant のみ**をサポートする。SSO設定の自動投入では SAML2 の loginOption 定義は考慮不要
- GUI上の場所: Administration > **Settings > Authentication > Single sign-on タブ**（SSOアクセス権を持つテナントにのみ表示）
- **エクスポート**: `GET /tenant/loginOptions/OAUTH2` で現行設定を読み出し、別テナントへ `POST` で投入する運用が成立する（ただしクライアントシークレット等は §10.5(1) の制約を受ける）
- **公式のプロビジョニング実装例**: `Cumulocity-IoT/cumulocity-provision-sso`（Java製マイクロサービス）が親テナントのSSO設定を子テナントへ複製する。**マイクロサービスのサブスクライブをトリガー**に、親の現行SSO設定がサブテナントへ書き込まれる。既知の制約として**アンサブスクライブ時の設定削除と、親側変更時の自動再配布は未実装**

✅ 3-0

---

### 10.11 追加で判明したテナントオプションの category/key（§7-9 の一部を埋める）

| category | key | 内容 |
|----------|-----|------|
| `measurement.series.latestvalue` | `<fragment>.<series>` 形式（例 `c8y_Humidity.H`、`c8y_Temperature.*`、全有効化は `*`） | 最新計測値機能。**GUIに設定UIがなく、`PUT /tenant/options/measurement.series.latestvalue` でのみ設定**。値は空文字列（将来予約）。プロパティ名に空白および `$ & + , / : ; = ? @ " < > # % { } \| \ ^ ~ [ ] \`` を含めてはならない。**ワイルドカード `*` が機能するのは Release 2026 以降**（同リリースで `c8y_LatestMeasurements` フラグメントによる自動永続化がpreview→GA） ✅ 3-0 |
| `access.control` | `allow.origin` | CORS許可オリジン。既定 `*`。定義済みキー限定 |
| `alarm.type.mapping` | `<ALARM_TYPE>` | `severity\|text` 形式で上書き（例 key=`temp_too_high`, value=`CRITICAL\|temperature too high`）。新規キー作成可 |
| `password` | `enforce.strength` | パスワード強度の強制 |
| `configuration` | `default.tenant.applications` / `default.tenant.microservices` / `on-update.tenant.*` | テナント作成／更新時のデフォルトサブスクリプション。**サブテナント個別にはテナントポリシー経由の追加オプションで上書き可** ✅ 3-0 |
| `configuration` | `measurement.series.previousvalue.enabled` | 計測値の前値保持 |
| `streaminganalytics` | `client.numClients` | Apama接続の並列数。既定は複数接続、`"1"` で完全直列 |
| `streaminganalytics` | `applicationAccess` | `"role"` を設定すると特定権限を持つユーザーにのみ表示 |
| `analytics.builder`（`c8y analytics configuration` 経由） | `timedelay_secs` | ABの遅延許容秒数（例 180） |
| マイクロサービス個別（既定は contextPath、`settingsCategory` で上書き可） | `email.protocol` / `email.connection.encrypted` ほか | メールサーバ設定。SMTP(暗号化なし)=`smtp`+`false` / SMTP(STARTTLS)=`smtp`+`true` / SMTPS(SSL/TLS)=`smtps`+`true` |
| — | `assets.permission.mode` | 2026-04-14にAPIドキュメントが改善された旨のアナウンスあり（要現物確認） |

> **依然として、テナントオプション／システムオプションの網羅カタログは公式に存在しない。** 実テナントで `GET /tenant/options` と `GET /tenant/system/options` を叩いて現物採取するのが唯一確実な方法（→ §7-9 は未解消のまま）。

---

### 10.12 Enterprise Configuration（メール・テンプレート）で設定できる項目

Administration > Configuration タブ（必要権限: 参照=Options management の READ / 変更=同 ADMIN）:

- **メールテンプレート**: 二要素認証SMSテンプレート、パスワードリセットメール（既知アドレス／未知アドレスの2種）、パスワード変更確認、招待メール、データエクスポートの件名・本文、テナント停止通知、ストレージ上限の事前警告／削除後通知
- **プレースホルダ**: `{host}` / `{tenant-domain}` / `{token}` / `{email}` / `{username}` / `{binaryId}` / `{exportApi}` / `{size}`
- **メールサーバ**: プロトコル/暗号化（SMTP / SMTP+STARTTLS / SMTPS）、ホスト、ポート、ユーザー名、パスワード、送信元アドレス。**空パスワード構成もサポート**
- **サポートリンク**: 「Request support」の設定。`false` で無効化可能

ドメイン名管理（`sslmanagement` マイクロサービスのサブスクリプションが前提。必要権限: Inventory / Options management / Application management の READ+ADMIN）:
- SSL証明書は有効期限内のX509形式・**確立された認証局発行（自己署名不可）**・**ワイルドカード形式**（例 `*.iot.mycompany.com`）。PKCS#12でチェーンと秘密鍵を結合。DNSはワイルドカードCNAMEを推奨。証明書更新時はCommon Nameが同一である必要がある

ブランディング（`feature-branding` アプリのサブスクリプションが前提。必要権限: Application management の READ+ADMIN）:
- Generic（タイトル / ファビコンICO / ダークテーマ切替 / フォントスタック / Cookieバナー）、Light・Darkテーマ別のロゴ・ブランドカラー・ステータスカラー・汎用カラー・アクションバー・メインヘッダ・ナビゲータ・右ドロワー、Custom CSS、**Advanced branding タブでのJSON直接編集（Web SDK の ApplicationOptions）**

出典: `https://cumulocity.com/docs/enterprise-tenant/customization/`

---

### 10.13 §7 未解決事項の更新（第5版時点）

| # | 項目 | 第5版での状態 |
|---|------|--------------|
| 1 | Settingsメニュー各項目↔テナントオプションの対応表 | **未解消**。ただしGUIカテゴリ（Application / Properties library / SMS provider / Connectivity / Localization / Feature toggles）は現行版ドキュメントで再確認済み（§10.12・付録A）。対応表は実機のネットワーク観察が必要 |
| 2 | テナントポリシーのREST API | **ほぼ解消（否定的結論）**。Core仕様にも公式Docsにも `/tenant/policies` 等のエンドポイントは存在しない → **GUI専用機能とみなし、ポリシー定義そのもののGit管理は断念して方式Aに寄せるのが妥当** |
| 3 | ブランディングパッケージの形式・手順 | 未解消 |
| 4 | go-c8y-cli の実用評価 | **解消**（§10.9）。カバレッジ・フラグ・認証・限界まで確定 |
| 5 | スマートルールのREST作成可否 | **部分解消**（§10.7）。マネージドオブジェクト構造は判明、公式RESTエンドポイントは非公開のまま。**EPLアプリ側は完全解消**（§10.6） |
| 6 | `c8y_Dashboard` の構造検証 | 未解消（実機確認事項） |
| 7 | Edge上でのTenant API挙動 | 未解消（実機確認事項）。ただしVM版EdgeのアプライアンスAPIは完全確定（§10.2） |
| 8 | Migration Toolの現状 | 未解消（採用非推奨の結論は不変） |
| 9 | テナントオプション/システムオプションの既定値カタログ | **未解消**（§10.11 で一部のkeyを追加。網羅カタログは公式に存在しない） |
| 10 | SSOプロバイダテンプレートの現行一覧 | 未解消。ただし**SAML非対応・OAuth2 authorization code grantのみ**は確定（§10.10） |
| 11 | OAI-Secureのトークン有効期間系オプション | **部分解消**。`sessionConfiguration`（`absoluteTimeoutMillis` / `renewalTimeoutMillis` / `maximumNumberOfParallelSessions` / `userAgentValidationRequired`）が authConfig 内に存在（§10.10）。パスワードポリシーは `password` / `enforce.strength` を確認 |
| 12 | Notifications 2.0 等 | **部分解消**。`/notification2/subscriptions`（+`/{id}`）、`/notification2/token`、`/notification2/unsubscribe`、`/notification/realtime` が Core 仕様に存在し、go-c8y-cli にも `notification2*.yaml` 3種のコマンド群がある（§10.3・§10.9） |
| **新** 13 | **K8sネイティブ世代Edgeにおける `/edge/...` API の継承有無** | 未確認。10.18.0 仕様はVM世代のもの。**導入対象世代での実機確認が必須** |
| **新** 14 | **Analytics Builder モデルの managed object フラグメント型** | 未確認。`c8y-analytics` CLI 経由なら不要だが、Inventory API 直叩きをするなら要特定 |
| **新** 15 | **Terraform provider / Ansible collection の有無** | **調査した限り公式・非公式とも存在しない**。Postman collection はコミュニティ製（`AdeelChohan/Cumulocity-Postman-V2-Collection`）で、公式は OpenAPI 仕様への移行を理由に v1 collection の保守を終了している。**IaC基盤は go-c8y-cli + jsonnet テンプレート、または OpenAPI からのクライアント生成で自作するのが現実解** |

---

### 10.14 第5版で訂正した第4版までの記述

| 箇所 | 第4版までの記述 | 第5版の訂正 |
|------|---------------|-----------|
| §3.7 | 旧VM版Edge向けに `/edge/configuration/{…}` が「ある」という概略のみ | **全18エンドポイント・全リクエストスキーマ・認証・非同期タスク（409排他）・アップロード規則まで確定**（§10.2）。過去の deep-research で0-3/1-2により棄却されたEdge関連主張は、**Redoc SPA 問題による誤った棄却**だった |
| §4.4 | EPLアプリは「Streaming Analyticsアプリでインポート・アクティベート」＝UI経路のみ | **`/service/cep/eplfiles` の完全なCRUD REST APIが存在**（§10.6）。`?contents=true` でソース込みダンプ可 |
| §4.4 | ABモデルは「デプロイはモデルマネージャUIのみ文書化」 | **`c8y-analytics` 拡張により list/get/create/update(state)/delete と mode 制御までCLI自動化可能**（§10.8） |
| §4.3 | スマートルールは「最も情報が薄い領域」 | マネージドオブジェクト構造（`c8y_SmartRule` / `SmartRuleRepresentation`）と組み込みテンプレート11種が判明（§10.7） |
| §2.2 / §5 | `credentials.` オプションはテナントポリシーに含めない | 加えて **`GET` でも `"<<Encrypted>>"` しか返らずエクスポート不能**であることが確定（§10.5）。**「GUIから設定値を抽出してJSON化」の構想には原理的な穴がある**ため、秘匿値は別管理前提で設計すること |
| §7-2 | テナントポリシーのREST APIの有無は「未確認」 | **存在が確認できない＝GUI専用機能とみなすのが妥当**（§10.13） |

---

## 付録A: カテゴリ別 設定項目一覧

> **本付録の出典**: 各カテゴリ見出しに記載の公式ドキュメントページを2026-08-04に直接取得して列挙（📄）。deep-researchの反証検証は経ていないため、実装前に該当ページ・実機での最終確認を推奨。既定値の「—」はドキュメントに記載がないことを示す。

### A-0. フェーズ0: Edge Custom Resource（Edgeのみ）

📄 出典: https://cumulocity.com/docs/2025/edge-kubernetes/edge-custom-resource-definition/

#### A-0.1 metadata / spec 直下（必須フィールド）

| フィールド | 型 | 必須 | 既定値 | 内容・制約 |
|-----------|-----|------|--------|-----------|
| `metadata.name` | String | ○ | — | Edge CR名 |
| `metadata.namespace` | String | ○ | — | Kubernetes名前空間 |
| `spec.version` | String | ○ | — | インストールするEdgeバージョン（`2025` または `2025.0.1` 形式）。**インストール時限定** |
| `spec.domain` | String | ○ | — | FQDN（例: myown.iot.com）。**インストール時限定**。ライセンスキーとドメインは紐づくため同時変更が必要 |
| `spec.licenseKey` | String | ○ | — | ドメインに対するライセンスキー（プロダクトサポートから取得）。**インストール時限定** |
| `spec.company` | String | ○ | — | edgeテナント名。**インストール時限定・変更不可** |
| `spec.email` | String | ○ | — | 管理者ユーザーのメールアドレス。**インストール時限定・変更不可** |
| `spec.cumulocityPasswordSecretName` | String | ○ | — | `INITIAL_C8Y_ADMIN_PASSWORD` キーを含むSecret名（8文字以上）。**インストール時限定**（インストール後のSecret変更は無視） |

#### A-0.2 spec 直下（任意フィールド）

| フィールド | 型 | 既定値 | 内容・制約 |
|-----------|-----|--------|-----------|
| `spec.tlsSecretName` | String | 自己署名証明書を自動生成 | `tls.key` / `tls.crt` を含むSecret。ドメイン本体・management・DataHubサブドメインをカバーする必要あり |
| `spec.storageClassName` | String | クラスタ既定のStorageClass | **インストール時限定・変更不可** |
| `spec.cloudTenant` | 構造体 | — | クラウドテナント経由のリモート管理構成（A-0.3） |
| `spec.core` | 構造体 | — | Cumulocity Core リソース設定（A-0.3） |
| `spec.messagingService` | 構造体 | 省略時は未インストール | Messaging Service（A-0.3） |
| `spec.mongodb` | 構造体 | — | MongoDB設定（A-0.3） |
| `spec.microservices` | 配列 | 既定マイクロサービスを全デプロイ | マイクロサービス単位のリソース割当（A-0.3） |
| `spec.dataHub` | 構造体 | 省略時は未インストール | DataHub（A-0.3） |

#### A-0.3 ネスト構造体

| フィールド | 型 | 必須 | 既定値 | 内容・制約 |
|-----------|-----|------|--------|-----------|
| `cloudTenant.domain` | String | ○（cloudTenant指定時） | — | クラウドテナントのドメイン（例: tenantid.cumulocity.com） |
| `cloudTenant.tlsSecretName` | String | — | 自己署名を自動生成 | MQTT X.509認証用TLS。中間CA発行なら `spec.tlsSecretName` を再利用可 |
| `core.resources.limits.cpu` | String | — | 3000m | Coreコンテナのリソース上限 |
| `core.resources.limits.memory` | String | — | 6GB | 同上 |
| `messagingService.enabled` | Boolean | ○ | — | 有効化には追加2 CPUコア/4 GB RAM、永続ボリューム3個。マイクロサービス版data broker・Notifications 2.0に必須 |
| `mongodb.credentialsSecretName` | String | — | `databaseAdmin` ユーザーで自動生成 | `MONGODB_DATABASE_ADMIN_USER` / `MONGODB_DATABASE_ADMIN_PASSWORD` キーを含むSecret |
| `mongodb.resources.limits.cpu` / `.memory` | String | — | 3000m / 6GB | MongoDBポッドのリソース上限 |
| `mongodb.resources.requests.storage` | String | — | 75 GB | データ用PVCサイズ。**インストール後は増加のみ可** |
| `microservices[].name` | String | ○ | — | 許容値: apama-ctrl / smartrule / opcua-mgmt-service / databroker-agent-server / datahub（リソース割当の対象指定。※ホスティング自体がこの5種に限定されるという主張は反証で棄却済み → §6） |
| `microservices[].resources.limits.cpu` / `.memory` | String | — | 1000m / 1 GB | マイクロサービス単位のリソース上限 |
| `dataHub.enabled` | String | ○（dataHub指定時） | — | 有効化には追加10 CPUコア/10 GB RAM、永続ボリューム5個 |
| `dataHub.dremioAdminCredentialsSecretName` | String | ○（dataHub指定時） | — | `DREMIO_ADMIN_USER` / `DREMIO_ADMIN_PASSWORD` を含むSecret（8文字以上・数字1+英字1以上） |
| `dataHub.resources.limits.cpu` | Integer | — | 2 | **単位なし整数のみ**（"2000m"不可） |
| `dataHub.resources.limits.memory` | String | — | 4096Mi | **Mi単位のみ**（"4Gi"不可） |

> ストレージ: Edge operatorはPVCを3つ作成する — `mongod-data` 75 GB（上記で変更可）/ `microservices-registry-data` 10 GB（マイクロサービスイメージ用プライベートレジストリ）/ `edge-logs` 5 GB。

#### A-0.4 c8yedge CLI・Edge運用の具体値（第4版追加・⚠️未検証、journal引用ベース）

**c8yedge CLI コマンド/フラグ**（`sudo c8yedge [command] [flags]`。取得元: download.cumulocity.com/Cumulocity-Edge/2025/c8yedge）

| コマンド | 内容 |
|---------|------|
| `c8yedge install` | 対話式インストール。`--version <番号>` / `-s <tarball>`（オフラインパッケージから） / `--cumulocity-password` |
| `c8yedge package` | オフライン（エアギャップ）用パッケージ作成 |
| `c8yedge upgrade` | アップグレード。`--version <番号>` / `-s "<オフラインパッケージ名>"` |
| `c8yedge config` | `--set` / `--set-file` で設定変更。キー: `domain` / `licenseKey` / `tlsSecret.tls.key` / `tlsSecret.tls.crt` / `company` / `email` |
| `sudo c8yedge uninstall` | アンインストール |

**Helm経路**: チャート `oci://registry.c8y.io/edge/helm-charts/cumulocity-iot-edge-operator`（`--version=2025`、名前空間 `c8yedge`）→ `kubectl apply -f c8yedge.yaml`

**サイジング・要件（バージョンにより異なる点に注意）**

| 項目 | Edge 2025（K8s版Docs） | Edge 2026（installing-edge） |
|------|----------------------|------------------------------|
| ベースライン | 6 CPUコア / 10 GB RAM / 100 GBディスク | 8 CPUコア / 16 GB RAM / 150 GBディスク |
| Messaging Service追加分 | +2コア / +4 GB RAM / +15 GBディスク | — |
| DataHub追加分 | +10コア / +16 GB RAM / +100 GBディスク | — |
| Kubernetes | 1.32.x（単一ノードのみ） | 1.34.x |
| 共通要件 | x86-64 AVX対応CPU（MongoDB要件）、Helm 3.x、LoadBalancerサービス、動的ボリュームプロビジョニング＋既定StorageClass、ポート80/443予約（他のIngress不可） | |

**運用系の具体値**

| 項目 | 値 |
|------|-----|
| ドメイン名形式 | 大小文字区別なし英数字をドット/ハイフンで連結、ドット1個以上、全体255文字以内、セグメント1〜63文字、TLD2〜6文字。国際化ドメインはPunycode表記 |
| 初期管理者 | ユーザー名 `admin`（management / edge 両テナント）。アクセスは `https://<domain>` / `https://management-<domain>` |
| 外部公開 | Kubernetesサービス `cumulocity-ontoplb`（名前空間c8yedge）をLoadBalancer化＋externalIPs設定 |
| プロキシ設定 | ConfigMap名は **`custom-environment-variables` 固定**（http_proxy / https_proxy / socks_proxy / no_proxy / ca.crt(PEM)） |
| 監視 | Prometheus互換エンドポイント `https://<domain>:3443/metrics`。クラウド送信メトリクスの間隔: ディスク10分毎(GB・小数2桁) / メモリ5秒毎(GiB) / CPU・ディスクI/O・ネットワークは5/60/600秒(%, KB/s) |
| バックアップ対象 | `/var/lib/rancher/k3s`（常に必須）、`/datahub`（DataHubデプロイ時のみ） |
| アップグレード方式 | recreate戦略（全置換・短時間のダウンタイムあり）。待機: `kubectl wait --timeout=1800s --namespace=c8yedge --for='jsonpath={.status.state}=Ready' edge/c8yedge` |
| ログ取得 | `kubectl get edge c8yedge -n c8yedge --output jsonpath='{.status.helpCommands.downloadLogs}' \| sh` |

### A-1. フェーズ1: テナント設定

#### A-1.1 Application（UI: Administration > Settings > Application）

📄 出典: https://cumulocity.com/docs/standard-tenant/changing-settings/

| 設定項目 | 内容・選択肢 | 既定値 |
|---------|-------------|--------|
| Default application | プラットフォームにアプリ指定なしでアクセスした際の全ユーザー共通ランディングアプリ（ドロップダウン） | — |
| Access control — CORS | Cumulocity APIのクロスオリジンリソース共有の有効/無効 | — |
| Allowed domain | REST APIと通信可能なJavaScriptアプリのドメイン。`*`（全ホスト）またはカンマ区切りドメインリスト | — |

#### A-1.2 Properties library（UI: Administration > Settings > Properties library）

📄 出典: 同上。対象オブジェクト: Inventory / Alarms / Events / Tenants

| 設定項目 | 内容・選択肢 | 既定値 |
|---------|-------------|--------|
| Name（識別子） | カスタムプロパティの一意な名前 | — |
| Label | 表示ラベル | — |
| Type | データ型（String / Boolean / Integer 等のドロップダウン） | — |
| Required | オブジェクト作成時の必須指定（チェックボックス） | 未チェック |
| Default value | 自動入力される既定値（**String型のみ**） | なし |
| Minimum / Maximum | 整数の下限/上限 | なし |
| Minimum length / Maximum length | 文字列の最小/最大文字数 | なし |
| Regular expression | 入力値の正規表現バリデーション | なし |

#### A-1.3 SMS provider（UI: Administration > Settings > SMS provider）

📄 出典: 同上

| 設定項目 | 内容・選択肢 | 既定値 |
|---------|-------------|--------|
| SMS provider | 利用可能なプロバイダからドロップダウン選択（フィルタ可） | — |
| Provider credentials | 選択プロバイダの認証情報（プロバイダ依存フィールド） | なし |
| Provider properties / Optional settings | プロバイダ固有の構成項目 | プロバイダ依存 |

#### A-1.4 Connectivity（UI: Administration > Settings > Connectivity）

📄 出典: 同上

| 設定項目 | 内容・選択肢 | 既定値 |
|---------|-------------|--------|
| Provider タブ | Actility LoRa / Sigfox / SIM | — |
| Provider URL | プロバイダのエンドポイントURL | なし |
| Provider credentials | プロバイダプラットフォームの認証情報 | なし |
| 追加フィールド | プロバイダ固有（各エージェントのドキュメント参照） | — |

#### A-1.5 Localization（UI: Administration > Settings > Localization）

📄 出典: 同上

| 設定項目 | 内容・選択肢 | 既定値 |
|---------|-------------|--------|
| Translation key | 新規翻訳キーの名前（一意） | — |
| 言語別翻訳フィールド | 対応言語ごとのカスタムUIテキスト | なし |

#### A-1.6 Feature toggles（UI: Administration > Settings > Feature toggles）

📄 出典: 同上

| 設定項目 | 内容・選択肢 | 既定値 |
|---------|-------------|--------|
| Feature name / Description / Key | 機能名・説明・一意キー（読み取り専用） | — |
| Phase | Generally available / Public Preview | 機能による |
| Status | 有効/無効トグル | Phaseに依存 |
| Strategy | カスタマイズ状態の表示（Resetボタンで既定に戻す） | 既定動作 |
| Subtenant feature toggles タブ | Managementテナントからサブテナントの機能を制御（Enterpriseテナントでは読み取り専用） | — |

#### A-1.7 テナントオプション（REST: `/tenant/options`、システム値の参照: `/tenant/system/options`）

⚠️ テナントオプションは **category / key / value の動的なキー空間**であり、全キーの静的な一覧は存在しない（マイクロサービスが自身の contextPath をcategoryとして保存する等）。調査で確認できた具体キーの例:

| Category | Key | 用途 | 出典 |
|----------|-----|------|------|
| configuration | default.tenant.applications | テナント作成時の既定Webアプリ（カンマ区切り） | managing-tenants |
| configuration | default.tenant.microservices | テナント作成時の既定マイクロサービス | 同上 |
| configuration | on-update.tenant.applications.enabled | アップグレード時の独自アプリリスト有効化（true/false） | 同上 |
| configuration | on-update.tenant.applications | アップグレード時の既定Webアプリ | 同上 |
| configuration | on-update.tenant.microservices.enabled | アップグレード時の独自マイクロサービスリスト有効化 | 同上 |
| configuration | on-update.tenant.microservices | アップグレード時の既定マイクロサービス | 同上 |
| measurement.series.latestvalue | シリーズ指定キー（例: `c8y_Humidity.H` / `c8y_Temperature.*` / `*`、値は空文字） | 最新メジャーメント値のシリーズ単位有効化。`PUT /tenant/options/measurement.series.latestvalue` で設定。任意の `strongConsistency` トグルで順序外れメジャーメントの扱いを制御 | managing-data |
| configuration | measurement.series.previousvalue.enabled | 直前値の取得。許容値 "true"/"false"（既定: 有効）。`POST /tenant/options/` で設定 | managing-data |
| sso | sso-redirect-default-application | Boolean。`false` を設定するとSSOログイン時にテナントドメインの使用を強制（baseUrlに `{tenantId}` を含む場合やSAN証明書利用時）。第4版追加 | authentication/sso |
| credentials.* | （プレフィックス） | 値が暗号化されて保存される。**テナントポリシー経由では暗号化されないため個別APIで設定** | managing-tenants |

> **既定値カタログの不在（第4版・3回目調査で確認）**: テナントオプション/システムオプション（`/tenant/system/options`）の「文書化された既定値の網羅一覧表」は、3回のdeep-research（公式Docs・OpenAPI仕様を対象）でも所在を確定できなかった。既定値は各機能ページに分散して記載されているのが実態であり、網羅的な把握には実テナントで `GET /tenant/system/options` / `GET /tenant/options` を実行して現物を採取するのが確実（→ §7）。

#### A-1.8 権限・ロール（UI: Administration > Roles、REST: `/user/*`）

📄 出典: https://cumulocity.com/docs/standard-tenant/managing-permissions/

**デフォルトグローバルロール**: admins / devices（登録デバイス用の標準構成） / CEP Manager / Cockpit User / Devicemanagement User / Global Manager（全デバイスデータ読み書き） / Global Reader（全デバイスデータ読み取り） / Global User Manager / Shared User Manager（要ユーザー階層サブスクリプション） / Tenant Manager / （レガシー: business, readers）

**権限レベル**: READ / CREATE / UPDATE / ADMIN

**権限カテゴリ（グローバルロール構成時の付与単位）**: Alarms / Application management / Audits / Bulk operations / CEP management / Data broker / Device control / Events / Global smart rules / Identity / Inventory / Measurements / Option management / Retention rules / Schedule reports / Simulator / SMS / Tenant management / Tenant statistics / User management / Own user management

**デフォルトインベントリロール**: Manager / Operations: All / Operations: Restart Device / Reader

**インベントリロール作成時のフィールド**: name / description / permissions（カテゴリ単位） / fragment types（Typeフィールド） / 権限レベル（READ / CHANGE / ALL）

#### A-1.9 データ保持・データ管理（UI: Administration > Management > Retention rules、REST: `/retention/retentions`）

📄 出典: https://cumulocity.com/docs/standard-tenant/managing-data/

| 設定項目 | 内容・選択肢 | 既定値 | 制約 |
|---------|-------------|--------|------|
| Data type | Alarms / Audits / Bulk operations / Events / Measurements / Operations | 必須選択 | ルールごとに1種別 |
| Fragment type | JSONプロパティ名 または `*` | `*` | 空白・特殊文字不可 |
| Type | オブジェクトの `type` 値 または `*` | `*` | — |
| Source | デバイスID または `*` | `*` | — |
| Maximum age | 1〜3,650日（10年） | 60日（システム設定で変更可） | 整数日のみ |

補足: アラームはstatus=CLEAREDのもののみ削除対象。ルールは通常1日1回逐次実行。**ファイルリポジトリには適用されない**。ファイルリポジトリの操作権限は Inventory の READ（閲覧）/ CREATE（アップロード）/ ADMIN（全管理）。アプリケーションアーカイブはこの画面から削除不可。

#### A-1.10 認証（UI: Administration > Settings > Authentication、REST: `/tenant/loginOptions`）

📄 出典: https://cumulocity.com/docs/authentication/basic-settings/

**ログイン設定**

| 設定項目 | 内容・選択肢 | 既定値 |
|---------|-------------|--------|
| Preferred login mode | OAI-Secure（推奨・新規テナント既定） / Basic Auth（レガシー） / Single sign-on redirect（SSO構成時のみ。選択するとBasic/OAI-Secureは除外） | OAI-Secure |
| Password validity limit | パスワード変更を要求するまでの日数（devicesロールのユーザーは対象外） | 0（無期限） |
| Password strength requirement | 全パスワードに「強」（green）を強制（チェックボックス）。⚠️未強制時の既定ポリシーは8文字以上のみ | 未チェック |
| Ignore case when logging in | ユーザー名/エイリアスの大文字小文字を無視（既存ユーザー間に衝突がないことが条件。テナント管理者のみ） | 無効 |

**Basic Auth 制限**

| 設定項目 | 内容 | 既定値 |
|---------|------|--------|
| Forbidden for web browsers | WebブラウザからのBasic認証を禁止 | — |
| Trusted user agents | 一致するUser-AgentヘッダのBasic認証リクエストを許可 | 空 |
| Forbidden user agents | 一致するUser-AgentヘッダのBasic認証リクエストを拒否 | 空 |

**二要素認証（TFA）**

| 設定項目 | 内容 | 既定値 |
|---------|------|--------|
| Allow two-factor authentication | TFAの許可（管理者のみ） | 未チェック |
| （SMS）Token validity limit | セッショントークンの有効期間（**分単位**）。SMS TFAには**SMSゲートウェイマイクロサービスの構成が必須** | — |
| （SMS）Verification code validity limit | SMS検証コードの有効期間（分単位） | — |
| （TOTP）Enforce TOTP on all users | 全ユーザーにログイン時のTOTPセットアップを強制（devicesロールは対象外） | — |

> **TOTP（および Enforce TOTP）は OAI-Secure ログインモード時のみ利用可能**（✅ 3-0）。SSO関連の設定は A-1.13 を参照。

#### A-1.11 ブランディング・Enterprise設定（UI: Administration > Settings > Branding ほか。Enterprise限定）

📄 出典: https://cumulocity.com/docs/enterprise-tenant/customization/

**Branding — Generic タブ**: Title（ブラウザ表示） / Favicon（ICO形式のみ） / ダークテーマ有効化 / タイポグラフィ（Base・Headings・Navigatorのフォントスタック、リモートフォントリンク） / Cookieバナー（有効化・Title・Text・プライバシーポリシーURL・ポリシーバージョン）

**Branding — Light / Dark テーマタブ**（両テーマ同一項目セット、色はHEX/RGB/RGBA）:

| グループ | 項目 |
|---------|------|
| Logos | Brand logo（PNG/SVG/JPG）とその高さ、Navigator logoとその高さ |
| Brand colors | Primary / Light / Dark ＋ 8段階のカラーシェード（自動生成可） |
| Status colors | Info / Warning / Danger / Success 各 default・light・dark |
| Generic | 背景色、テキスト色、Text muted、リンク色、リンクホバー色、ボタンborder-radius |
| Action bar | 背景・テキスト・アイコン・ボタン・ボタンホバー各色 |
| Main header | 背景・テキスト・ボタンホバー各色 |
| Navigator | 背景・テキスト/ボタン・セパレータ・ヘッダ背景・タイトル・アクティブ背景/枠/テキスト各色 |
| Right drawer | 背景・テキスト・Text muted・セパレータ・リンク・リンクホバー各色 |

**Branding — その他タブ**: Custom CSS（CSSエディタ） / Advanced（ブランディングJSONオブジェクトの直接編集）

**Domain name**: SSL証明書アップロード（PKCS #12） / ドメイン有効化 / 証明書更新・無効化

**Configuration タブ（メール・運用テンプレート等）**: TFAのSMSテンプレート / サポートリンクURL（"false"で非表示） / パスワードリセット（未知アドレスへの送信許可、既知/未知アドレス用テンプレート、件名、変更確認テンプレート、招待テンプレート） / メールサーバ（プロトコル: SMTP / SMTP STARTTLS / SMTPS SSL/TLS、ホスト、ポート、ユーザー名、パスワード、送信元アドレス） / データエクスポート（件名・テンプレート・権限エラーメッセージ） / ストレージ上限（警告・超過の件名/テンプレート） / テナントサスペンド（サスペンド管理者への送信、追加受信者、件名/テンプレート）

#### A-1.12 サブテナント・テナントポリシー・デフォルトサブスクリプション（Enterprise限定。REST: `POST /tenant/tenants` ほか）

📄 出典: https://cumulocity.com/docs/enterprise-tenant/managing-tenants/

**サブテナント作成フィールド**

| 設定項目 | 必須 | 内容・制約 |
|---------|------|-----------|
| Domain/URL | ○ | サブドメイン（小文字英数字とハイフンのみ、先頭は英字、2文字以上、ハイフンは中間のみ、1階層のみ） |
| Name | ○ | テナント名 |
| Administrator's email | ○ | パスワードリセットに使用 |
| Administrator's username | ○ | 管理者アカウント名 |
| Contact phone | ○ | 連絡先電話番号（第4版の検証で必須と確定 ✅） |
| Contact name | — | 連絡先氏名 |
| Send password reset link as email | — | **既定: 選択済み**。解除時はパスワード入力＋確認が必須 |
| Tenant policy | — | 適用するポリシーをドロップダウン選択 |

> サスペンドされたサブテナントは、Cumulocityパブリッククラウドでは**60日後に自動削除**される（⚠️）。

**テナントポリシーの構成**: Name（必須） / Description / Retention rules（**最低1件必須**） / Tenant options（category+key+value形式）。各ルール・オプションに「サブテナントによる変更を許可」チェックボックスあり（既定: 未チェック）。ポリシーは暗号化されないため `credentials.` プレフィックスのオプションは含めないこと。

**デフォルトサブスクリプション**: テナント「作成時」と「プラットフォームアップグレード時」の2リストを個別管理。テナントオプション（A-1.7の configuration カテゴリ6キー）で上書き可能。

**Limits タブ（サブテナント単位の制限）**: Limit HTTP queue / Limit HTTP requests（毎秒） / Limit stream queue（MQTT） / Limit stream requests（MQTT毎秒） / デバイス数上限（同時ルートまたは全デバイス） / External reference / Gainsightトラッキング有効化

#### A-1.13 Single sign-on（SSO / OAuth2）（第4版追加・✅主要項目は3-0検証済み）

UI: Administration > Settings > Authentication > **Single sign-on タブ**（SSO構成アクセスが有効なテナントにのみ表示。Managementテナント専用にも構成可）。REST: `/tenant/loginOptions`。
📄 出典: https://cumulocity.com/docs/authentication/sso/ 、https://cumulocity.com/docs/2026/authentication/sso/ 、https://cumulocity.com/docs/2024/authentication/sso/

**Basic設定フィールド** ✅: Redirect URI / Redirect to the user interface application（2026版のみの新フィールド） / Client ID / Token issuer / Button name / Provider name / Audience（JWTの`aud`期待値） / Visible on Login screen

**リクエスト定義** ✅: Authorize（GET） / Token（POST） / Refresh（POST） / Logout（任意、front-channel single logout。OIDCでは`id_token_hint`）。プレースホルダ: `${clientId}`, `${redirectUri}`, `${code}`, `${refreshToken}`

**Dynamic access mapping** ✅（3モード・既定は(3)）:
1. Use dynamic access mapping only on user creation（ユーザー作成時のみ）
2. 毎ログインでルール選択ロールを再割当てし、他は変更しない
3. **毎ログインで再割当てし、他はクリア（既定）** — Cumulocity内での手動ロール変更は次回ログインで上書きされる

**外部トークン検証** ✅: introspection / user-infoエンドポイント検証に対応。Access token validation frequency の**既定値は1分**（期間内は検証結果を内部キャッシュ）

**署名検証** ⚠️: 4方式（Azure AD証明書ディスカバリ / ADFSマニフェスト / 手動公開鍵 / JWKS URI）。**RSA鍵のみ対応**（"n"/"e"パラメータ対 or x5c証明書チェーン。楕円曲線鍵は非対応）

**Azure AD構成時の要点** ⚠️: Redirect URIはテナントアドレス＋`/tenant/oauth`。JWTには一意ユーザーID＋`iss`/`aud`/`exp`が必要。SSOユーザーには最低限「Own user management」のREAD権限が必要

**注意（棄却済み）**: 「プロバイダテンプレートは Custom（既定）＋Azure AD / Keycloak 12.0.0+ / ADFS 3.0 の検証済みテンプレート」という主張は**1-2で反証**された（2024版Docs由来の可能性）。現行2026版のテンプレート一覧は実機確認が必要。

#### A-1.14 Data broker（第4版追加・✅主要項目は3-0検証済み）

UI: Data brokerアプリ（Data connectors / Data subscriptions / Data filters）。**前提: テナントが `feature-broker` アプリケーションにサブスクライブしていること**。マイクロサービス版はさらに `databroker-agent-server` マイクロサービスのサブスクリプションが必要。
📄 出典: https://cumulocity.com/docs/data-broker/data-broker-application/ 、https://cumulocity.com/docs/data-broker/ms-data-broker/

**データコネクタのフィールド** ✅

| 設定項目 | 内容・制約 |
|---------|-----------|
| Title | コネクタ名 |
| Target URL | 転送先テナントURL。**保存後は編集不可** |
| Description | 任意 |
| Data filters | **最低1つ必須** |

**フィルタの5フィールド** ✅: Group or device（"All Objects"は後方互換のため残置・**非推奨予定で使用非推奨**） / API（alarms, events, measurements, managed objects=転送、operations=受信） / Fragments to filter / Fragments to copy（Copy all fragmentsオプションあり。未指定時は標準プロパティのみ転送 — アラーム作成時: type, text, time, severity, status） / Type filter

**コネクタのステータス** ⚠️: Active（転送有効） / Suspended（ソース側が転送無効化） / Pending（宛先側が転送無効化）

**その他** ⚠️: 転送されたデバイスは宛先テナントで通常デバイスとして課金される。コネクタ毎に永続メッセージバックログを保持（テナント毎クォータ・TTLは構成可能、数値既定はservice quotasドキュメント参照。クォータ到達時は転送対象APIリクエストがHTTP 500）。マイクロサービス版はMessaging Serviceに依存（パブリッククラウド10.13+で既定有効、専用/自己ホスト10.10+は明示有効化が必要）。**マイクロサービス版の有効化はManagementテナントから（運用サポート関与）で、テナントレベルのセルフサービス設定ではない**。

#### A-1.15 Usage statistics / 課金（第4版追加・✅3-0検証済み）

UI: Administration > Usage statistics（Management/Enterpriseテナント。**Tenant management のREAD権限が必要**）。
📄 出典: https://cumulocity.com/docs/enterprise-tenant/usage-and-billing/ 、https://cumulocity.com/docs/microservice-sdk/general-aspects/

| 項目 | 具体値 |
|------|--------|
| マイクロサービス課金モード | マニフェストの `billingMode` フィールド。**RESOURCES（既定）** / SUBSCRIPTION の2値 |
| 統計収集スケジュール | リクエスト数フラッシュ: **5分毎**。Used storage / Device count / Subscribed applications / Microservice resources: **9時・17時・EOD**（EOD値がその日の確定値。時刻フォーマット・タイムゾーンはDocsに明記なし） |
| リソース計測単位 | CPU: ミリコア（**1000m = 1 CPU**）、メモリ: MB。テナント毎に日次収集 |
| サスペンド中テナント | リクエスト数・マイクロサービスリソースは課金されず、テナント存在とストレージサイズのみカウント（⚠️） |
| 画面のサブテナント毎の列 | API requests / Device API requests / Storage (MB) / Peak storage (MB) / Devices / Peak devices / Endpoint devices / Subscribed applications / Alarms・Events・Inventories・Operations created-updated / Measurements created / Total inbound transfer / CPU (M) / Memory (MB) ほか（ID, Tenant, Root devices, Peak root devices, Creation time, Parent tenant, External reference） |
| CSVエクスポート | フィールド区切り文字・小数点記号・文字セットを指定可能 |

> 関連（⚠️）: Commit-to-Consume（CTC）契約者向けには使用量・消費を確認する **Console** アプリケーションが別途存在する。

#### A-1.16 アラームマッピング（第4版追加・⚠️未検証、journal引用ベース）

UI: Administration > **Business Rules > Alarm mapping**。権限: 閲覧は Option management のREAD、作成/編集/削除は同ADMIN（既定ロールでは Tenant Manager が保有）。
📄 出典: https://cumulocity.com/docs/standard-tenant/alarm-mapping/

| 設定項目 | 内容・制約 |
|---------|-----------|
| Alarm type | 必須。**プレフィックス（ワイルドカード）一致**で解釈され、1マッピングが同一プレフィックスの複数アラームタイプに適用される。**作成後は変更不可** |
| New description | 任意。空欄時は元のアラームテキストを維持 |
| Severity | ドロップダウン選択。新しい重大度、または **Drop**（アラーム表示の抑止） |

#### A-1.17 ユーザー管理の設定項目（第4版追加・⚠️未検証、journal引用ベース）

UI: Administration > Users。
📄 出典: https://cumulocity.com/docs/standard-tenant/managing-users/

| 項目 | 内容 |
|------|------|
| パスワード設定方法（ユーザー作成時） | 3択: Send password reset link as email / Set password that must be changed on the first login / Set password for the user (no change required) |
| ユーザー単位TFA | Two-factor authentication (SMS) / Two-factor authentication (TOTP) / Enforce TOTP setup for the user（初回ログイン時に設定強制）。テナントレベルでTFA有効時のみ |
| ユーザー階層 | `feature-user-hierarchy` アプリケーションサブスクリプションが必要（Shared User Managerロールも同依存） |
| SSOユーザーの制約 | 外部認可サーバー経由で作成されたユーザーは、Cumulocity側でのユーザー情報・グローバルロール・アプリケーションアクセス・インベントリロールの更新が**無効**。パスワードリセットも無効化 |
| Log out all users | OAI-Secure・SSOリダイレクト・JWTデバイストークンを無効化（Base64のBasic認証トークンは無効化されない） |

### A-2. フェーズ2: 初期コンテンツ

#### A-2.1 デバイス一括登録CSV（UI: Device Management > Registration）

📄 出典: https://cumulocity.com/docs/device-management-application/registering-devices/

**シンプル形式**

| 列 | 必須 | 内容 |
|----|------|------|
| ID | ○ | デバイス識別子（例: シリアル番号） |
| PATH | — | スラッシュ区切りのグループパス（**未存在のグループは自動作成**） |

**フル形式**

| 列 | 必須 | 内容・許容値 |
|----|------|-------------|
| ID | ○ | デバイス識別子 |
| CREDENTIALS | ○ | デバイスごとの一意なパスワード |
| TYPE | — | デバイスタイプ（例: c8y_Device） |
| NAME | — | 表示名 |
| ICCID | — | モバイルデバイス識別子 |
| IDTYPE | — | IDの種別（例: c8y_Serial） |
| PATH | — | グループ割り当てパス |
| SHELL | — | 1 / 0 |
| AUTH_TYPE | — | BASIC 等 |
| TENANT | — | 対象テナント（Enterprise限定） |

再実行時: 既存識別子のデバイスはCSV内容で**更新**される。

**単一デバイス登録**: Device ID（必須） / グループ割り当て / ワンタイムパスワード（クエリパラメータ `externalId` / `one-time-password` で事前入力可） / セキュリティトークン（ポリシー: IGNORED / OPTIONAL / REQUIRED）。カスタムマイクロサービスによる拡張登録フォームも可能。

#### A-2.2 ダッシュボード設定（UI: Cockpit）

📄 出典: https://cumulocity.com/docs/cockpit/working-with-dashboards/

| タブ/機能 | 設定項目 |
|----------|---------|
| General | Icon / Menu label（名前） / Description / Location（ナビゲータ内の位置: 5000〜-5000） |
| Availability | 閲覧可能なグローバルロールのチェックボックス（既定: 全ロール選択） |
| Dashboard template | 同一デバイスタイプの全デバイスへ共有するトグル（デバイスタイプ設定時のみ） |
| Appearance | Theme（Match UI / Light / Dark / Branded） / Widget header style（Regular / Border / Overlay / Hidden） / Widget gap / Layout mode（Grid=既定 / Responsive=列構成・ビューポートフィット） |
| その他 | Translate if possible（タイトル翻訳） / Version history（復元可） / **Import/Export（JSON編集・ファイルベース移送）** / Copy・Paste / Delete |

RESTでの実体は `c8y_Dashboard` フラグメント付きmanaged object（icon / priority / name / global / isFrozen / children=ウィジェット定義。§4.2参照・⚠️コミュニティ情報）。

#### A-2.3 Analytics Builder モデル（UI: Streaming Analytics > Analytics Builder）

📄 出典: https://cumulocity.com/docs/streaming-analytics/analytics-builder/

| 区分 | 設定項目 |
|------|---------|
| モデルのモード | Draft（新規モデルの既定） / Test（実データ処理・出力は保存のみ・単一デバイスのみ） / Simulation（履歴データ再生・出力は保存のみ・単一デバイスのみ） / Production（実データ処理・デバイスへ出力送信） |
| アクティベーション状態 | Active / Inactive（**JSONインポート直後は常にInactive**） |
| 基本メタデータ | 一意なモデル名 / Description / Tags（フィルタ用） |
| テンプレートパラメータ | パラメータ名（モデル内一意） / 型 / 既定値 / 必須・任意指定。インスタンスごとに値・モード・アクティベーションを独立設定可 |
| ブロック単位パラメータ | 入出力デバイス選択 / fragment・series マッピング / 閾値 / 期間・タイミング / メジャーメント種別 / アラーム種別・重大度 |
| Simulation設定 | 開始・終了タイムスタンプ / カレンダーベースの期間指定 |
| Import/Export | JSONダウンロード（ブラウザのダウンロード先へ） / JSONアップロード / クリップボード経由のJSONコピー&ペースト / サンプルからの新規作成 |

#### A-2.4 アプリケーション（UI: Administration > Ecosystem > Own applications）

⚠️ 今回の追加取得の対象外（§4.5参照）。WebアプリはZIPアップロード、マイクロサービスはZIP/レジストリ経由でデプロイし、go-c8y-cli の deploy コマンドで自動化可能。個別フィールドの列挙は実機確認時に補完すること。

#### A-2.5 デバイス可用性・監視設定（第4版追加・⚠️未検証、journal引用ベース）

UI: Device Management > 各デバイス。
📄 出典: https://cumulocity.com/docs/device-management-application/monitoring-and-controlling-devices/

| 設定項目 | 内容・具体値 |
|---------|-------------|
| Required interval | デバイスからの通信を期待する間隔。ユーザーが手動設定するか、デバイス自身が設定する（候補に挙がっていた「availability interval」の公式項目） |
| 接続断中の扱い | 既定では切断前の状態（in service / out of service）を維持するとみなされる |
| 可用性の算出 | パーセンテージ表示。期間は 24時間 / 7日 / 30日 |
| 不可用扱いのアラーム | 接続喪失なしの機能不全を不可用として扱うには該当アラームを **CRITICAL** 重大度にする必要がある（MAJORでは不可）。重大度は CRITICAL / MAJOR / MINOR / WARNING の4種 |
| 一覧の自動リフレッシュ | アラーム一覧・イベント一覧とも既定 **30秒** |
| メンテナンスモード | 接続監視カードでオン/オフ。メンテナンス中のアラーム生成を抑止 |
