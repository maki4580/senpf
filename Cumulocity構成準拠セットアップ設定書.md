# Cumulocity 構成準拠 セットアップ設定書

**対象構成**: [cumulocity-iot-architecture.drawio](cumulocity-iot-architecture.drawio) —「全体構成(配置構成・レビュー反映)」タブ**のみ**
**版**: rev.4（2026-08-18）— 構成図側の修正（ダングリングエッジ2本への source 接続、`thin-edge.io(child)` → `REST/MQTT API` 経由への修正）を反映し、§2.3 注-3・注-4 と §7 図-5・図-6 を解消済みとして整理。rev.3 は構成図の通知方式変更（Webhook → **Notification 2.0**）とメール通知の不採用を反映。rev.2 で 3 観点のレビュー（製品仕様 / 構成整合性 / 運用リスク）を反映済み
**調査基準日**: 2026-08-17。本書の全 URL・仕様参照はこの時点の取得結果
**位置づけ**: 基盤提供側が、Cumulocity Edge の設置後に「この構成用の設定」と「基盤標準の初期設定値」を投入するための、設定定義と適用手順

---

## 0. 本書の位置づけと読み方

### 0.1 既存 3 文書の役割分担

| 文書 | 役割 | 本書との関係 |
|---|---|---|
| [Cumulocity設定定義書.md](Cumulocity設定定義書.md) | 設定項目の**網羅カタログ** | 本書は項目 ID を参照。ただし §0.2 の**訂正**が優先 |
| [Cumulocity設定エクスポート投入ガイド.md](Cumulocity設定エクスポート投入ガイド.md) | **投入の実装技法**（go-c8y-cli イディオム・冪等化パターン） | 本書は手順の中で使うパターンだけを指定。**ただし §6.2 のリポジトリ構成は本書が置き換える** |
| **本書** | **構成 → 設定のトレースと適用順序** | 構成図の各要素が要求する設定を確定し、フェーズに割り付ける |

### 0.2 ⚠️ 既存文書・前版に対する訂正

**rev.2 のレビューで、一次情報により次の誤りが判明しました。既存文書の該当記述より本書が優先します。**

| # | 訂正対象 | 誤 | 正 |
|---|---|---|---|
| 訂-1 | 設定定義書 §1.1 の注記 / 本書 rev.1 §3.1 | Edge CR ページが *"an exhaustive listing of what could be changed"* と明記 | **この文言は 2026・2025 いずれの CRD ページにも存在しない**（`grep -i exhaustive` で 0 件）。引用は削除 |
| 訂-2 | 設定定義書 §1.1 L0-08 / 本書 rev.1 E-05, E-06, §1 P-4 | Edge CR に `messagingService` / `microservices` / `core` / `dataHub` がある | **2026 の Edge CR `spec` は 12 項目のみ**（§3.1）。これらは **2025 版 CRD 限定** |
| 訂-3 | 設定定義書 §11 V-01 / 本書 rev.1 §1 P-3, §7 V-01 | ライセンス取得手順は確定情報なし【欠】 | **2026 の Prerequisites に逐語で記載あり**【確】。§1 P-3 |
| 訂-4 | 本書 rev.1 §1 P-1 | Edge の運用手順ページは `/docs/2024/edge/operating-edge/` にしか存在しない | **2026 に運用系ページ群が揃っている**（スラッグが変わっただけ）。§1 P-1 |
| 訂-5 | 本書 rev.1 §7 棄却リスト | 「Streaming Analytics / Microservice Hosting / data broker は Edge に既定で含まれる」は棄却 | **2026 の比較表に `Included` と逐語で記載**。棄却は誤り。§1 P-4 |
| 訂-6 | 本書 rev.1 §3.3 K-03, §2 行 20 | 構成配布は `c8y_Configuration` | **`c8y_Configuration` はテキスト設定の別系統で、thin-edge 標準マッパーは非対応。** 正しくは `c8y_SupportedConfigurations` + `c8y_DownloadConfigFile` / `c8y_UploadConfigFile`。§4.9.2 |
| 訂-7 | 本書 rev.1 §4.9.3 K-a | 流量制御は システムオプション `bulkoperation.creationramp` | **主たる制御は bulk operation リクエスト本文の `creationRamp` フィールド**（発行側が指定可能）。§4.9.3 |
| 訂-8 | 本書 rev.1 §4.9.1 | 大量取得だから `acl.algorithm-version` = OPTIMIZED を維持 | **因果が逆。OPTIMIZED は一致件数が閾値（既定 2000）未満のときだけ適用**され、超えると LEGACY に落ちる。`next` リンクで辿ることが実装規約。§4.9.1 |
| 訂-9 | 本書 rev.1 §2 行 9 | 紐づけ確認アプリは VM1 ホスト | **構成図では VM2**。`createHostedApplication` は使えず、EXTERNAL アプリ登録 + CORS が必要。§3.2 P-01 |
| 訂-10 | 本書 rev.1 §0.4, §2 行 8 | 現場管理端末が Cumulocity Web UI にアクセス | **現場管理端末 → 画像解析装置のフロントエンド**、**業務端末 → Cumulocity Web UI**。経路が別。§2 |
| 訂-11 | 本書 rev.1 §7 V-02, V-03, V-04, V-12, V-16 | SMTP / 公開ポート / バックアップ / ブランディング / Edge 監視は情報なし【欠】 | **いずれも 2026 ドキュメントに記載あり**【確】。§3.1・§4.10 |

**rev.3 での仕様変更**（誤りの訂正ではなく、前提の変更）:

| # | 変更 | 影響 |
|---|---|---|
| 変-1 | 構成図の通知が **Webhook → 通知 (Notification 2.0)** に変更 | §4.8 を全面改訂。**「ネイティブ webhook が無い」という制約は論点から消滅**。代わりに **§4.1 の RBAC バイパスが構成の中心問題に昇格** |
| 変-2 | **メール通知を採用しない** | メールサーバー設定（旧 N-01・旧 P1 1-3）を全面削除。スマートルールの「On alarm send email / SMS / escalate」も対象外。パスワードリセットへの影響は**限定的**（SSO ユーザーは Keycloak 管理・サービスアカウントは削除→再作成で復旧可）だが、**両テナント admin と break-glass だけはエスクローが必要**（§4.10） |
| 変-3 | Messaging Service の位置づけ | Notification 2.0 が通知の**唯一の手段**になったため、Messaging Service は「任意」ではなく**必須コンポーネント**。2026 Edge には `Included`（P-4）だが、**稼働確認が P1 の必須工程**になる |

> **なぜ前版で誤ったか**: rev.1 は敵対的検証つきの自動調査に依拠しましたが、その検証が**真の主張まで棄却**していました（訂-3・訂-5）。「棄却された ＝ 事実でない」ではありません。rev.2 では一次ドキュメントを直接読み直しています。

### 0.3 設定の 3 分類

| 分類 | 内容 | 変わるタイミング | 投入手段 |
|---|---|---|---|
| **L0 / 基盤インフラ層** | Edge そのもの（ドメイン・TLS・ライセンス・K8s リソース） | 環境ごとに必ず変わる | `c8yedge` CLI / `kubectl` + Edge CR |
| **L1-B / 基盤標準初期値** | 全案件で同一の「製品としての既定値」 | 基盤のバージョンアップ時のみ | go-c8y-cli（環境非依存） |
| **L1-C / 本構成固有設定** | この構成でのみ必要な設定 | 案件・環境ごとに変わる | go-c8y-cli（環境別 values） |

> **L0 は `config/edge/`、L1-B は `config/platform/`、L1-C は `config/site/` に分けます**（§6.2）。この分離ができているかが、2 案件目を `site/` だけの作業で立ち上げられるかを決めます。

### 0.4 確度の凡例

| 記号 | 意味 |
|---|---|
| **[確]** | 公式ドキュメントまたは OpenAPI 仕様の**逐語記述**で確認済み |
| **[確K]** | Cumulocity Tech Community の Knowledge Base 記事で確認（公式 docs ではない） |
| **[推]** | 一次情報からの演繹。直接の記述は無い |
| **[要]** | 未確認。実機検証またはベンダー確認が必要。§8 に一覧 |
| **[図外]** | **構成図に描かれておらず、口頭説明に基づく補完**。図の改訂要求として §7 に起票 |

### 0.5 対象構成

> ⚠️ **対象タブの限定**: 「全体構成(配置構成・レビュー反映)」タブ**のみ**が対象です。同ファイルの「全体構成(配置構成)」「全体構成(1拠点)のコピー」は旧版で、本書の根拠には使用していません。

旧版との差異（本書はすべて「レビュー反映」列に従う）:

| 論点 | 配置構成タブ | 1拠点コピータブ | レビュー反映タブ（本書の前提） |
|---|---|---|---|
| モデル登録先 | ファイルリポジトリ | — | **ソフトウェアリポジトリ** |
| 長期保存の配置 | 本社オブジェクトストレージ | 拠点内 Data Lake | **本社オブジェクトストレージ** |
| 拠点と Edge の関係 | 記載なし | 拠点ごとに基盤インスタンス | **単一 Edge・単一テナントに集約（D17）** |
| 通知 | なし | Webhook/メール（Edge→案件側アプリ） | **通知 (Notification 2.0)（イベント処理→他アセット）。メール通知は不採用** |
| メタ監視 | なし | メタ監視エージェント→集約（**拠点発 push, D9**） | **保守端末→VM1（pull）** ← §2 の注意点 |
| 業務端末 / 運用知識基盤 | なし | なし | **あり** |

```
現場(顧客拠点)×N               閉域網    本社サーバールーム(オンプレ)
──────────────────             ─────    ────────────────────────────
IPカメラ群(ONVIF/RTSP,〜数百台)          VM1(基盤インフラ層)
   ▲ ONVIF/ICMP死活確認                   Cumulocity Edge
   │ RTSP(映像)                             REST/MQTT API ─ Web UI ─ デバイス管理
サイトVMS(Genetec)                           イベント/アラーム処理 ─ Operational Store(〜90日)
   │ RTSP                                  Keycloak(IdP) / Otelインフラ / 運用知識基盤
画像解析装置(エッジ端末, Linux)             ベクトルDB / 特徴量エクスポートジョブ
  ├ Docker Compose                         外部Gateway = BOXアダプタ + thin-edge.io
  │  ├ フロントエンド ◄─ HTTPS ─ 現場管理端末   (全拠点分のBOXを集約)
  │  ├ バックエンド                        オブジェクトストレージ
  │  ├ ログ処理(Grafana Alloy) ─ OTLP ──►   ◄ ①C8Yオフロード ②GSC映像クリップ+サイドカーJSON
  │  └ 画像解析パイプライン
  └ thin-edge.io ── MQTT(イベント/計測)+REST(スナップショット添付) ──►
                                          VM2(アセット層)
画像センシングBOX ─ 独自形式イベント ──►    紐づけ確認アプリ / 生体認証SA / 他アセット
  (改造不可・バッファ+再送)                      ▲ アラーム閲覧 ─ 現場管理端末
                                          ※別筐体: Genetec Security Center(Windows Server要件)
業務端末(ブラウザ) ── HTTPS(閲覧・操作) ──► Cumulocity Web UI(Cockpit等)

閉域外: AIモデル改善サービス(クラウド) ◄ 特徴量データ送信(限定アウトバウンド) ─ 特徴量エクスポートジョブ
保守拠点: 保守端末(保守VPN) ── モデル登録・更新Operation発行 / メタ監視 / エクスポート承認
```

**設定に効く決定事項**（[design-decisions.md](../../IoTPlatform_cc/design-decisions.md)）:

- **D17**: 全拠点を単一 Cumulocity Edge・単一テナントに集約。拠点区分は device group + Inventory ロール
- **D9**: 拠点発アウトバウンドのみ。中央/自社側から拠点への着信接続は設けない
- **D16**: 標準エージェントのランタイムは thin-edge.io。カメラは child device として代理登録
- **D13**: VMS 連携は抽象レイヤー経由。映像バイト列は基盤を通さない
- **D14**: 保存対象は Alarm 昇格分の自動保存 + 手動指示分のみ
- **D15**: 映像解析 AI は別筐体。AI 側は基盤の標準ペイロード規約（**検知イベント形式、`modelVersion` 必須**）にのみ従う

> ⚠️ **[要] D6/D7/D15 の責務境界は D17 転換により再定義が必要**（SV-21）。レビュー反映タブの現場に「基盤アプライアンス／標準サイトサーバー」は存在せず、thin-edge.io は `画像解析装置` の内側にあります。**基盤チームの標準部品が、D15 で「別筐体」と切り離したはずの AI 筐体に同居する構成**です。本書は「基盤が画像解析装置を握れる」前提で書いていますが、その筐体の調達・OS 管理・構成管理の主体を確定してください。

---

## 1. 着手前に固定すべき前提

### P-0. ⚠️ ネットワーク前提 — インストール後に直せない項目群

**これらは `c8yedge install` の後では変更できません。P-1 以降に進む前に必ず潰してください。**

| # | 項目 | 内容 | 確度 |
|---|---|---|---|
| P-0-1 | **CIDR 衝突** | K3s の Service / Pod CIDR と社内網アドレス空間の重複確認。オンプレ閉域網は 10.0.0.0/8 が一般的で衝突しやすい。衝突すると Edge の Pod から社内 DNS / Keycloak / オブジェクトストレージ / Genetec の**一部にだけ**到達できず、「Web UI は動くが SSO だけ落ちる」のように散発的な症状が出る | [推] |
| P-0-2 | **公開ポート** | `cumulocity-ontoplb` (LoadBalancer): **443, 8443, 1883, 8883**。Edge operator メトリクス: **3443**。加えて *"Edge requires that your Kubernetes cluster does not have an Ingress provider (for example, Traefik) enabled on common ports that would block those used by Edge, such as ports 80 and 443."* | [確] |
| P-0-3 | **時刻同期（NTP）** | 社内 NTP を Edge ホスト・Keycloak・画像解析装置・外部Gateway に設定。ずれると TLS の notBefore/notAfter 判定、Keycloak の JWT `exp`/`iat` 検証、イベントと映像クリップの突合が**同時に**壊れ、エラーから原因に辿り着けない | [推] |
| P-0-4 | **到達性マトリクス** | Edge → 社内 DNS / Keycloak(JWKS) / オブジェクトストレージ / Genetec、拠点網 → Edge:8883,443、**案件アプリ(VM2) → Edge の WebSocket エンドポイント**（Notification 2.0・SV-05）、保守VPN → Edge | [推] |
| P-0-5 | **ハードウェア最小要件** | CPU **8 コア** / RAM **16 GB** / Disk **150 GB**。**MongoDB は AVX 命令 + x86-64-v3 以降が必須**（`lscpu` で確認） | [確] |
| P-0-6 | **`storageClassName` の確定** | *"Once the `storageClassName` field is configured in the Edge custom resource (CR), it cannot be changed."* | [確] |

### P-1. Edge のデプロイ形態 — 2026 は Kubernetes のみ [確]

| 選択肢 | 内容 | 本構成での評価 |
|---|---|---|
| **(a) `c8yedge` CLI（K3s 同梱）** | `sudo c8yedge install`。オフラインは `c8yedge package` → `sudo c8yedge install -s "<OFFLINE-PACKAGE-FILENAME>"` | **推奨**。閉域網でエアギャップ導入が可能。K8s 運用スキルが薄いチーム（D5）に適合 |
| (b) 持ち込み K8s + Helm | `helm upgrade --install c8yedge-operator oci://registry.c8y.io/edge/helm-charts/cumulocity-iot-edge-operator --version=2026 --namespace c8yedge` → Edge CR 作成 | 非推奨。**Kubernetes 1.34.x のみ**、**単一ノードクラスタのみサポート**（*"Edge is tested and supported on single-node Kubernetes clusters only."*）、LoadBalancer と動的ボリュームプロビジョニングが前提 |

> **訂正（訂-4）**: 2026 には運用系ページ群が揃っています。前版の「運用手順は 2024 にしかない」は誤りでした。

| 2026 URL | 内容 |
|---|---|
| `/docs/2026/edge/manage-edge/` | Modifying Edge / Upgrading / **メールサーバー設定** / **外部 IP 経由のアクセス** / registry credentials 変更 |
| `/docs/2026/edge/edge-operations/` | ログ / **バックアップとリカバリ** / **監視ツールの導入** / **メトリクス** |
| `/docs/2026/edge/benchmarks/` | 実測ベンチマーク |
| `/docs/2026/edge/using-edge/` | **ブランディング** / Web SDK |
| `/docs/2026/edge/troubleshooting-edge/` | トラブルシュート |

> ⚠️ **バージョン系の分断は残ります**: Edge の OpenAPI は「Release 10.18.0」の 1 版のみで、`/edge/*` の 18 エンドポイントは **VM アプライアンス系の遺産**です。K8s 形態では使えません。

### P-2. ドメイン 2 つと TLS 証明書 [確]

Edge は必ず `management` と `edge` の 2 テナントを持ちます。

| テナント | URL | 用途 |
|---|---|---|
| edge | `https://<domain_name>` | 業務データ・デバイス・ユーザーの本体 |
| management | `https://management-<domain_name>` | Edge プラットフォーム設定、**ブランディング**（メールサーバーは変-2 により設定しない） |

- 両テナントともユーザー名は **`admin`**。初期パスワードは `c8yedge --cumulocity-password` または Edge CR の `spec.cumulocityPasswordSecretName`（Secret のキー名は **`C8Y_ADMIN_PASSWORD`**、8 文字以上）。インストール後に独立して変更可能
- 閉域網 DNS に**両ホスト名の登録が必要**

> ⚠️⚠️ **TLS 証明書は両ホスト名をカバーすること。ワイルドカードは 1 ラベル階層しかカバーしない。** [確]
>
> *"If you have a wildcard certificate like `*.myown.iot.com`, then you must set the domain name to any single level subdomain of `myown.iot.com`, that is `sub.myown.iot.com`, but not `myown.iot.com` itself"*
>
> - `*.iot.com` は `myown.iot.com` と `management-myown.iot.com` の**両方をカバー**
> - `*.myown.iot.com` を買うと `management-myown.iot.com` は**カバーされない**
> - **推奨**: SAN に 2 つのホスト名を明示列挙した証明書を社内 CA で発行。PEM 形式・完全なチェーンを正しい順序で

**投入方式の分岐は「アプライアンス vs K8s」ではなく「`c8yedge` ツール vs 自前 K8s」です** [確]:

```bash
c8yedge config --set domain=<domain-name> --set-file licenseKey=<path/to/license.txt> --set-file tlsSecret.tls.key=<path/to/tls.key> --set-file tlsSecret.tls.crt=<path/to/tls.crt>
```

自前 K8s では `spec.tlsSecretName` で TLS Secret を参照します。

> ⚠️ **証明書・CA ローテーション手順が必要**（SV-22）。社内 CA ルートを更新すると **thin-edge 側の信頼ストア更新が必要**で、数百台×N 拠点が同時に接続不能になり得ます。しかも配布経路（Cumulocity 経由のソフトウェア更新）も同時に使えません。**新旧 CA を並行して信頼させる移行期間**を設計に入れてください。

### P-3. ライセンスと registry credentials [確] — 訂正項目

**前版で【欠】としましたが、2026 の Prerequisites に逐語で記載があります。**

> *"To request the license file for Edge, contact product support. In the email, you must include - Your company name, under which the license has been bought - The domain name (for example, myown.iot.com), where Edge will be reachable."*

- 同ページに専用節 **"Domain name validation for Edge license key generation"** があり、FQDN 不要／サブドメインを除外した場合はワイルドカード証明書必須／IDN は ASCII 変換必須／文字種・長さ規則が規定されています
- **ライセンスがドメインに紐づくことは確定**: *"the license key must always be valid for the domain name, so any change of domain name should be made simultaneously with a change of license key."*
- ⚠️ **前版が見落としていた必須入手物**: *"**The Edge registry credentials** — You will receive the Edge registry credentials along with the Edge license."* Helm 経路の `helm registry login registry.c8y.io` に必要です

**残るベンダー確認事項**（SV-01）: ①閉域網でのライセンス更新（オンライン検証の要否）②1 インスタンスで全拠点を収容する利用形態の商流・サポート可否。

### P-4. Edge CR のスキーマ版を確定する [確] — 前版 P-4 の差し替え

**前版は「Messaging Service を導入するか」を前提としていましたが、これは 2026 では成立しません。**

**2026 の Edge CR `spec` は次の 12 項目のみ**です:

```
spec.cloudTenant.{domain, otp, tlsSecretName}
spec.company                     spec.cumulocityPasswordSecretName
spec.domain                      spec.email
spec.licenseKey                  spec.storageClassName
spec.mongodb.{credentialsSecretName, resources.requests.storage}
spec.tlsSecretName               spec.version
```

`messagingService` / `microservices` / `core` / `dataHub` は **2025 版 CRD 限定**です。

**そして 2026 の比較表は、これらのコンポーネントを `Included` と明記しています** [確]（訂-5）:

| Area | Edge | Cumulocity platform |
|---|---|---|
| Multi-tenancy | **No; single tenant** | Yes |
| Vertical scalability | **Yes, limited to appr. 100 tps per CPU core** | Yes |
| **Messaging Service** | **Included** | Included |
| **MQTT Service** | **Included** | Included |
| **Microservice-based data broker** | **Included** | Optional |
| **Microservice Hosting** | **Included** | Optional |
| **Streaming Analytics** | **Included** | Optional |
| OPC UA / Cloud Field Bus | Included | Optional |
| Data Hub | **On request via Professional Services** | Optional |

> *"Since Edge is based on the same software as the cloud-based Cumulocity platform version, the included applications are the same in both versions, with minor restrictions."*

**帰結**: Notification 2.0 の前提となる Messaging Service は 2026 Edge に含まれます。**前版 P-4 の「導入するかを決める」という論点は消滅**しました。

> ⚠️ **さらに変-1 により、Messaging Service は「任意」から「必須」に変わりました。** Notification 2.0 が通知の唯一の手段になったため、これが動かなければ**案件アプリにアラームが一切届きません**。P1 1-5 で稼働確認を必須工程にしています。残る検証は実動作のみ（SV-05・**最優先**）。

---

## 2. 構成図 → 設定要求のトレース

構成図（レビュー反映タブ）の**全ノード 41・全エッジ 41** に対する判定です。「Cumulocity 側設定なし」も判定として明示します。

### 2.1 Cumulocity の設定を要求する要素

| # | 構成要素 | 要求される Cumulocity 設定 | 分類 | 項目 ID |
|---|---|---|---|---|
| 1 | Cumulocity Edge (VM1) | ドメイン 2 つ・TLS・ライセンス・K8s リソース・バックアップ | L0 | E-01〜E-11 |
| 2 | 2 テナント構造 | management / edge の管轄分離、admin パスワード分離、**両テナントの break-glass** | L0/L1-B | E-02, R-07 |
| 3 | 画像解析装置（thin-edge 内蔵） | デバイス登録方式・テナント CA・device type 規約・`c8y_RequiredAvailability`・supported operations | L1-C | D-01〜D-05 |
| 4 | 外部 Gateway（BOXアダプタ+thin-edge） | 同上 + child device としての BOX 登録 + external ID 命名規約 | L1-C | D-01, D-06 |
| 5 | IP カメラ（child device 代理登録） | child device 自動登録・死活計測の型・カメラ用 device type | L1-B/L1-C | D-04, D-07 |
| 6 | 拠点×N の区分 | **device group 階層** + **Inventory ロール** + グループの external ID | L1-C | G-01〜G-03 |
| 7 | Keycloak (IdP) | `loginOptions` / `authConfig`・JWKS・アクセスマッピング・ログインモード | L1-C | A-01〜A-05 |
| 8a | **保守端末**（保守VPN） | 保守用グローバルロール・ソフトウェアリポジトリ書込権限・Device Management アクセス | L1-B | R-01, M-01 |
| 8b | **現場管理端末** → 画像解析装置のフロントエンド／紐づけ確認アプリ | **Cumulocity Web UI へは直接アクセスしない**。紐づけ確認アプリ経由でのみ Cumulocity ユーザーが必要 | L1-B | R-02 |
| 8c | **業務端末** → Cumulocity Web UI (Cockpit等) | Cockpit アプリケーションアクセス + 拠点 Inventory ロール | L1-B | R-03 |
| 9 | 紐づけ確認アプリ（**VM2** ホスト） | **EXTERNAL アプリケーション登録 + テナント購読 + CORS オリジン追加** | L1-B/L1-C | P-01, P-02, T-02 |
| 10 | 生体認証SA / 他アセット (VM2) | **CORS**・サービスユーザー（マシン認証）・REST 権限 | L1-C | T-02, R-05 |
| 11 | イベント/アラーム処理 | イベント型/アラーム型分類・**ペイロード規約**・`alarm.type.mapping`・スマートルール | L1-B | S-01〜S-04, T-03 |
| 12 | **通知 (Notification 2.0)**（宛先＝**他アセット**） | サブスクリプション定義 + **トークン発行プロキシ**（`ROLE_NOTIFICATION_2_ADMIN` を案件アプリに渡さないため）+ Messaging Service の稼働 | L1-B/L1-C | N-01〜N-04 |
| 13 | スナップショット（イベント添付） | 添付バイナリのサイズ上限・リテンションとの関係 | L1-B | T-04, X-02 |
| 14 | Operational Store 〜90日 | リテンションルール（dataType 別） | L1-B | X-01 |
| 15 | オフロード → オブジェクトストレージ | **Data Hub は PS 経由（商流制約）** → 代替実装。**流入は 2 系統**（C8Y / GSC 映像クリップ） | L1-C | X-03, D-06 |
| 16 | AI モデル配布 | ソフトウェアリポジトリ・`c8y_SoftwareUpdate`・ファイルサイズ上限 | L1-B/L1-C | M-01〜M-03 |
| 17 | クリップ保存（exportClip） | 手動分は設定なし。**D14 の「Alarm 昇格分の自動保存」は Notification 2.0 のサブスクライバが受けて `exportClip` を叩く**（変-1 により EPL からの HTTP 送出は不要になった） | L1-B | N-01 |
| 18 | Otel インフラ / メタ監視 | 監査ログ・**Prometheus メトリクスエンドポイント** | L1-B | X-01, O-01 |
| 19 | **運用知識基盤**（VM1 内） | **Cumulocity の読み手かつ書き手**。拠点横断の読取 + 構成書込。**最高リスクの主体** | L1-B/L1-C | K-01〜K-08 |
| 20 | 閉域網（現場⇔本社） | Cumulocity 側設定なし。ただし公開ポート（P-0-2）と社内 CA 配布（P-2）が前提 | — | — |

### 2.2 Cumulocity の設定を要求しない要素（判定の明示）

| # | 構成要素 | 判定 | 留意点 |
|---|---|---|---|
| 21 | Docker Compose / フロントエンド / バックエンド | 設定なし | 現場管理端末の UI 入口。**Cumulocity ユーザーとは無関係** |
| 22 | ログ処理（Grafana Alloy） | 設定なし | 現場側テレメトリの唯一の供給源。運用知識基盤の入力（K-02） |
| 23 | 画像解析パイプライン | 設定なし | ただし**検知イベントのペイロード規約（S-04）に従う義務がある**（D15） |
| 24 | サイトVMS / Genetec Security Center | 設定なし | D13 の抽象レイヤー経由。映像バイト列は基盤を通さない（D1） |
| 25 | **ベクトルDB**（特徴量格納・イベントID/モデルver.紐づけ） | 設定なし | ⚠️ **紐づけキーが Cumulocity のイベント external ID**（D-06 の第 3 の利用者） |
| 26 | **特徴量エクスポートジョブ**（合意範囲の選択・監査ログ） | 設定なし | ジョブの監査ログを Cumulocity 監査ログに載せるかは設計判断 |
| 27 | **AIモデル改善サービス（クラウド）** | 設定なし | ⚠️ **図中で唯一、閉域網の外へ出るデータ経路。** D4 / D9 / D14 / H9（契約文言）が全部かかる。FW 例外（P-0-2）の対象 |
| 28 | 生体情報の取り扱い注記 | 設定なし | ⚠️ 生体情報は Cumulocity に入らない。ただし特徴量がクラウドへ出るため **external ID の仮名化要否**が論点（S-04） |

### 2.3 ⚠️ 構成図の記述に対する注意点

> **rev.4 で解消**: 旧注-3（`thin-edge.io(child)` の REST/MQTT API バイパス）・旧注-4（ダングリングエッジ2本）は構成図側の修正により解消。§7 図-5・図-6 参照。

| # | 内容 |
|---|---|
| 注-1 | **メタ監視の向きが D9 と逆**: 図は `保守端末 --[メタ監視(外部死活監視)(保守VPN)]--> VM1` すなわち自社側からの pull。D9「自社側から拠点への着信接続は設けない」と転換 3-3「各インスタンスが自社側に報告する」は push 型。旧タブには `保守VPN(拠点発のみ, D9)` と明記されていた。VPN 内なら D9 の例外に収まるが、**FW 審査と攻撃面の観点で設計判断として再確認が必要**（SV-23） |
| 注-2 | **生体認証SA への映像経路は図が「※経路未確定」と明記**。VM1/VM2 に映像ストリームを通す点は D1（映像は基盤の責務外）の帯域・ストレージ前提に触れる。経路確定後に再評価 |
| 注-3 | **運用知識基盤の読み書き経路のうち一部が図に存在しない**（テレメトリ入力・他アセットへの設定更新の2本は描画済み。Cumulocity Operational Store / オブジェクトストレージへの直接アクセス経路は未描画）→ §7 で図の改訂を要求 |

---

## 3. レイヤ別 設定項目定義

### 3.1 L0 — Edge 基盤層

| ID | 設定項目 | 目的 | 投入手段 | 確度 |
|---|---|---|---|---|
| E-01 | `spec.domain` | Edge の FQDN。**ライセンスと必ず同時に変更** | `c8yedge config --set domain=<FQDN>` | [確] |
| E-02 | `spec.cumulocityPasswordSecretName` | 両テナント admin の初期パスワード（Secret キー = `C8Y_ADMIN_PASSWORD`） | Edge CR + K8s Secret | [確] |
| E-03 | `spec.licenseKey` | ライセンスキー | `c8yedge config --set-file licenseKey=<path>` | [確] |
| E-04 | `spec.tlsSecretName` / `tlsSecret.tls.{crt,key}` | サーバ TLS 証明書（**両ホスト名を SAN に**） | `kubectl create secret tls` または `c8yedge config --set-file` | [確] |
| E-05 | `spec.storageClassName` | ストレージクラス。**設定後は変更不可** | Edge CR | [確] |
| E-06 | `spec.mongodb.resources.requests.storage` | MongoDB 容量。90 日保持 + オフロード前提でサイジング | Edge CR | [確] |
| E-07 | `spec.version` | Edge バージョン。**バックアップからの復元時は同一版が必須** | Edge CR / `c8yedge install --version` | [確] |
| E-08 | `spec.cloudTenant.*` | クラウドテナント接続（本構成では**未使用**） | Edge CR | [確] |
| E-09 | **バックアップ** | §4.10 | — | [確] |
| E-10 | **Edge registry credentials** | ライセンスと共に受領。Helm 経路の `registry.c8y.io` ログインに必要 | — | [確] |
| E-11 | **`c8yedge-operator-config` ConfigMap** | ⚠️ **閉域網で必須**。`ca.crt`（信頼する追加 CA の PEM バンドル）と `no_proxy`（**両テナントドメイン + Pod CIDR + Service CIDR を必ず含める**）。適用後は operator の再起動が必要 | `kubectl` | [確] |

**Edge CR のエクスポート**:

```bash
kubectl get --namespace=c8yedge edge/c8yedge -o yaml > config/edge/edge.yaml
```

> `status`, `metadata.{uid,resourceVersion,creationTimestamp}` を除去してから Git 管理してください。

### 3.2 L1-B — 基盤標準初期値

#### ロール・権限（R 系）

| ID | 設定項目 | 本構成での値 | API | 冪等化 |
|---|---|---|---|---|
| R-01 | グローバルロール「基盤運用者」 | Tenant Manager 相当 + Application management ADMIN + Retention rules ADMIN + CEP management ADMIN | `POST /user/{t}/groups` → `/roles` | 存在チェック |
| R-02 | 「拠点オペレーター」 | Alarms ADMIN / Events READ / Inventory READ / Device control READ + **Own user management READ** | 同上 | 同上 |
| R-03 | 「業務閲覧者」（業務端末） | Alarms READ / Events READ / Inventory READ + Own user management READ + **Cockpit アプリアクセス** | 同上 | 同上 |
| R-04 | Inventory ロール「拠点Manager」「拠点Reader」 | Alarms / Events / Inventory / Measurements の READ / CHANGE / ALL 組合せ | `POST /user/inventoryroles`（**専用 CLI サブコマンド無し → `c8y api`**） | 409 黙殺 |
| R-05 | サービスユーザー（VM2 アセット用） | ローカルユーザー + 最小ロール。**SSO 対象外** | `POST /user/{t}/users` → `/roles` | 存在チェック |
| R-06 | サービスユーザー（**運用知識基盤・読取用 / 書込用の 2 つ**） | 読取: 拠点横断 READ。書込: `ROLE_DEVICE_CONTROL_ADMIN` + `ROLE_INVENTORY_ADMIN` | 同上 | 同上 |
| R-07 | **break-glass アカウント（両テナント）** | SSO 対象外の緊急用ローカル管理者。TFA + 資格情報の封緘保管 + 使用時アラート | 同上 | 同上 |
| R-08 | 証明書アップロード用ローカルユーザー | `ROLE_TENANT_ADMIN` または `ROLE_TENANT_MANAGEMENT_ADMIN`。**SSO ユーザーでは `tedge cert upload` が実行できない**ため必須 | 同上 | 同上 |
| R-09 | **トークン発行プロキシ用サービスユーザー** | ⚠️ **`ROLE_NOTIFICATION_2_ADMIN` を持つ唯一の主体。** 案件アプリ・業務ユーザーには絶対に付与しない → §4.8.3 | 同上 | 同上 |

> ⚠️ **「Own user management」の READ が無いとログインできません。** SSO のアクセスマッピングで割り当てる全ロールに必ず含めてください。

> ⚠️ **API ロール名に `ROLE_INVENTORY_UPDATE` は存在しません** [確]。OAS に出現するのは `ROLE_INVENTORY_ADMIN` / `_CREATE` / `_READ` のみ。UI の権限レベル（READ/CREATE/UPDATE/ADMIN）と API ロール名は 1:1 対応しないため、投入スクリプトでハマります。

> ⚠️ **`ROLE_NOTIFICATION_2_ADMIN` は業務ユーザーへ絶対に渡さない** → §4.1。

#### テナントオプション（T 系）

| ID | カテゴリ / キー | 本構成での値 | 確度 |
|---|---|---|---|
| T-01 | `configuration` / `acl.algorithm-version` | `OPTIMIZED`（10.16+ の既定）。**ただし一致件数 2000 未満のときだけ適用される** → §4.9.1 | [確K] |
| T-02 | `access.control` / `allow.origin` | **既定 `*` からの絞り込み**。VM2 の全アプリ（**紐づけ確認アプリを含む**）のオリジンをスキーム+ホスト+ポートで列挙 | [確] |
| T-03 | `alarm.type.mapping` / `<ALARM_TYPE>` | `<SEVERITY>\|<TEXT>`。severity `NONE` で抑止 → §4.4 | [確] |
| T-04 | `configuration` / `acl.measurement.only-accessible-fragments` | `true` を検討 | [確] |

#### データ保持（X 系）

| ID | 設定項目 | 本構成での値 | 確度 |
|---|---|---|---|
| X-01 | リテンションルール | `EVENT`/`MEASUREMENT`/`ALARM`/`OPERATION` = 90 日、**`AUDIT` = 別基準（要決定・K-c）** | [確] |
| X-02 | イベント添付バイナリ | **[要]** リテンションが添付に及ぶかは公式に記載なし（SV-06） | [要] |
| X-03 | オフロード | §4.6 | — |

- 既定は**全履歴データ 60 日**、上限 10 年
- **アラームは `CLEARED` のもののみ削除**。自動クリアを設計しないと単調増加（S-03）
- **files repository には適用されない**

#### ルール・イベント規約（S 系）

| ID | 設定項目 | 本構成での役割 | 投入 |
|---|---|---|---|
| S-01 | スマートルール | **死活アラーム、閾値アラームの生成のみ**。⚠️ メール/SMS/エスカレーション系テンプレートは**変-2 により対象外** | ⚠️ **[要] managed object の `type`/`fragmentType` 値が未確定**（投入ガイド `U-01`）。型名確定前は投入不可 |
| S-02 | EPL apps | BOX 再送の重複排除、イベント→アラーム昇格 | `POST /service/cep/eplfiles` |
| S-03 | アラーム自動クリア | 未クリアアラームの蓄積防止 | スマートルールまたは EPL |
| S-04 | **検知イベントのペイロード規約** | ⚠️ **基盤標準初期値の最重要項目**。イベント型名・必須フラグメント・**`modelVersion`**（D15）・カメラ参照・external ID・確信度。将来のオブジェクトストレージ移行を見越した参照フィールドを初版から予約 | 規約文書 + バリデータ |

前提: **Smartrule / Apama-ctrl マイクロサービスへの購読**。

#### 通知（N 系）・アプリケーション（P 系）

| ID | 設定項目 | 内容 |
|---|---|---|
| N-01 | **Notification 2.0 サブスクリプション定義** | 拠点グループ単位（`context: mo` + `alarmsWithChildren`）。§4.8 |
| N-02 | **トークン発行プロキシ（token broker）** | ⚠️ **`ROLE_NOTIFICATION_2_ADMIN` を案件アプリに渡さないための基盤標準部品**。§4.8 |
| N-03 | Messaging Service の稼働確認 | Notification 2.0 の前提。2026 Edge には `Included` |
| N-04 | コンシューマのライフサイクル管理 | ⚠️ 切断してもサブスクライバは消えず**バックログが溜まり続ける**。§4.8 |
| — | ~~メール通知~~ | **不採用**（変-2）。§4.10 |
| P-01 | 紐づけ確認アプリ | ⚠️ **VM2 ホスト**のため `createHostedApplication` は不適。**`type: EXTERNAL` のアプリケーション登録**（アプリスイッチャーとアクセス制御に載せるため）+ **CORS**（T-02）。Cumulocity にバンドルを載せる案との二択は SV-24 |
| P-02 | テナントのアプリ購読 | `POST /tenant/tenants/{t}/applications` |
| P-03 | 標準ダッシュボード | Cockpit の JSON エクスポート/インポート。**Edge での可否は要検証**（SV-11） |
| P-04 | フィーチャートグル | `c8y features enable/disable` |

### 3.3 L1-C — 本構成固有設定

| ID | 設定項目 | 詳細 |
|---|---|---|
| G-01 | 拠点 device group 階層 | §4.1。**段階ロールアウトの単位でもある**（K-b） |
| G-02 | グループの external ID | 冪等投入の土台 |
| G-03 | Inventory ロール割当 | SSO の `inventoryMappings` で管理（手動割当は上書きされる） |
| A-01〜A-05 | Keycloak SSO | §4.3 |
| D-01〜D-07 | デバイスオンボーディング・型規約・死活 | §4.2 / §4.5 |
| M-01〜M-03 | モデル配布 | §4.7 |
| K-01〜K-08 | 運用知識基盤 | §4.9 |
| O-01 | Edge 自体の監視 | §4.10 |

---

## 4. 主要設計の詳細

### 4.1 ⚠️ 拠点分離 — Notification 2.0 が RBAC を貫通する

Edge の比較表は **"Multi-tenancy | No; single tenant"** と明記しています [確]。拠点ごとのサブテナント分離は不可能で、device group + Inventory ロールで代替する D17 の判断は正しい選択です。

> ただし「Edge はシングルテナント」と無条件に書くと誤りです。同じドキュメントが *"Edge has two tenants, management and edge"* と述べています。「シングルテナント」＝**サブテナントを作れない**の意です。

#### ハード制約: Notification 2.0 は Inventory ロール RBAC をバイパスする [確]

Core OpenAPI 仕様に逐語で:

> *"**⚠️ Caution:** If you assign Notification 2.0 roles or permissions to users, they can create Notification 2.0 subscriptions and receive notifications for any device, including those to which assigned inventory roles do not grant access, bypassing the inventory role RBAC."*

`POST /notification2/subscriptions` の必要ロールは**テナントレベルの `ROLE_NOTIFICATION_2_ADMIN` 単独**で、`source` によるスコープ指定がありません。対照的に Alarm API は *"...if the user has access to alarms through inventory roles, only those alarms are returned"* と述べており、**Notification 2.0 は意図的かつ文書化された例外**です。2025 / 2026 / Latest の 3 版でバイト一致。

#### 設計方針

| 決定 | 内容 |
|---|---|
| **D-a** | `ROLE_NOTIFICATION_2_ADMIN` は運用者サービスアカウントとマイクロサービスにのみ付与 |
| **D-b** | ⚠️ **rev.3 で更新**: 「拠点担当者に自拠点だけの通知購読権を **Cumulocity のロールで** 与える」ことは原理的に不能。ただし**サブスクリプション自体は拠点グループ単位に絞れる**（`context: mo` + `alarmsWithChildren`）。**基盤がサブスクリプションを定義し、トークン発行プロキシ（N-02）で配る**ことで実質的な拠点分離が成立する → §4.8 |
| **D-c** | 顧客間の分離が必要になった時点（2 顧客目）で単一テナント集約は成立しない。**顧客ごとに Edge を分ける**（拠点ごとではない） |
| **D-d** | **運用知識基盤も拠点分離の外側に立つ。ただし「常時付与」ではなく承認時の一時付与に絞れる** → §4.9.3 G-2 |
| **D-e** | **案件アプリ・基盤標準アプリの REST 呼び出しは原則エンドユーザーの OIDC トークンで行い、サービスユーザー代行を禁止する。** サービスユーザー代行は Notification 2.0 と同型の RBAC バイパスになり、拠点分離が無効化される |

#### グループ階層

```
拠点 (root group)
├── 拠点A
│   ├── 画像解析装置        ← thin-edge デバイス
│   │   └── (child) カメラ群
│   └── 画像センシングBOX   ← 外部Gateway の child device
├── 拠点B …
```

- **グループ階層は「望ましい」ではなく「必須基盤」です** [確]: bulk operation の `groupId` は *"Identifies the target group on which this operation should be performed."* で、**API 経由の一括オペレーションはグループ指定が事実上の前提**です
- 各グループに external ID を振ること。名前だけで参照すると再実行で重複グループが生えます
- Inventory ロールは親グループ → サブグループ → デバイスへ継承

### 4.2 デバイスオンボーディング

| 方式 | 内容 | 評価 |
|---|---|---|
| **(1) テナント CA + EST 登録** | `POST /certificate-authority` でテナント CA 作成 → デバイスが `/.well-known/est/simpleenroll` で取得（tenant + device serial + **one-time password** を BasicAuth） | **推奨**。thin-edge.io 公式も *"Recommended"* |
| (2) 自己署名 CA + 自動登録 | CA を `POST /tenant/tenants/{tenantId}/trusted-certificates` に `autoRegistrationEnabled: true` でアップロード | 次善。⚠️ **一括オンボーディング完了後は無効化すること**（有効なままだと承認なしにデバイスが自動登録される） |
| (3) 個別証明書アップロード | `tedge cert create` → `tedge cert upload c8y --user <user>` | 開発・検証用 |

必要ロール: **`ROLE_TENANT_ADMIN` または `ROLE_TENANT_MANAGEMENT_ADMIN`** [確]。**`tedge cert upload` は SSO ユーザーでは実行不可** → R-08 が必須。

> テナント CA は **毎年 10 月 2 日 02:00 に自動更新**されます [確]。⚠️ **これは日付が確定した障害リスクです**（SV-25）: 閉域網で `simplereenroll` が期待通り動かないと、深夜帯に全拠点のデバイス証明書が順次失効します。検証環境でシミュレートし、証明書有効期限をメタ監視に入れ、運用カレンダーに載せてください。

#### ⭐ 一括登録 CSV が拠点グループ階層も同時に作れる [確]

`POST /devicecontrol/bulkNewDeviceRequests`（`ROLE_DEVICE_CONTROL_ADMIN`）は CSV（区切り自動判別）で次を同時に投入できます。

1. **証明書エンロール**: `AUTH_TYPE=CERTIFICATES` + `ENROLLMENT_OTP`（`CREDENTIALS` は省略可）
2. **device group 階層の自動生成**: スラッシュ区切りの `PATH`。**存在しないグループは作成される**

**制約** [確]: ①`PATH` 使用時は `TYPE` と `NAME` 列も必要 ②`com_cumulocity_model_Agent.active` ヘッダ（値 `true`）が必要 ③**テナント CA の存在が前提** ④`ENROLLMENT_OTP` と `PATH` の併用例は仕様に無い（禁止記述も無い）→ SV-07 ⑤UI ウィザードは `ENROLLMENT_OTP` 非対応。

#### thin-edge.io との分界点 [確]

| 責務 | 側 | 具体 |
|---|---|---|
| テナント CA 作成 | Cumulocity | `POST /certificate-authority` |
| 登録 OTP 発行 | Cumulocity | Device Management → Devices → Registration |
| 証明書取得 | thin-edge | `sudo tedge cert download c8y --device-id "<id>"` |
| 接続先設定 | thin-edge | `tedge config set c8y.url` / `c8y.root_cert_path` / `c8y.http` / `c8y.mqtt` |
| 接続確立 | thin-edge | `sudo tedge connect c8y` |
| child device 登録 | thin-edge | child device API。**カメラ側に証明書は不要** |

> **閉域網での追加作業**: `c8y.root_cert_path` の信頼ストアに社内 CA ルートを配置する構成管理タスクを、画像解析装置の標準イメージに組み込んでください。**Edge operator 側にも同じ CA が必要**です（E-11）。

#### BOX 再送の重複排除と external ID の「三重の役割」

Cumulocity 側に重複排除機能はありません。外部 Gateway の BOX アダプタが冪等化を担い、Cumulocity 側で決めるのは規約だけです。

> ⚠️ **この external ID は三重の役割を持ちます。**
> 1. BOX 再送の**重複排除キー**
> 2. オブジェクトストレージ上で C8Y イベントと **GSC 映像クリップ + サイドカー JSON を突き合わせるキー**（§4.6）
> 3. **ベクトルDB の特徴量 ↔ イベント紐づけキー**（図のノード「特徴量格納: イベントID/モデルver.紐づけ」）
>
> 3 つ目はクラウドへ持ち出される特徴量に紐づくため、**採番規約に「業務 ID を仮名化するか」という個人情報上の論点が乗ります**。S-04 と併せて基盤標準として最初に確定してください。

### 4.3 Keycloak SSO

`POST /tenant/loginOptions`（メディアタイプ **`application/vnd.com.nsn.cumulocity.authconfig+json`**、小文字）で完全に API 投入できます [確]。必要ロールは `ROLE_TENANT_ADMIN` **OR** `ROLE_TENANT_MANAGEMENT_ADMIN`。

> ⚠️ **リクエストボディのスキーマでは `id` が `readOnly` にオーバーライドされます。** エクスポート JSON から `id` を必ず除去してください。

主要フィールド: `type: OAUTH2` / `providerName` / `grantType: AUTHORIZATION_CODE` / `userManagementSource: INTERNAL` / `issuer` / `clientId` / `audience` / `signatureVerificationConfig.jwks.jwksUrl` / `accessTokenToUserDataMapping.*` / `userIdConfig.jwtField` / `onNewUser.dynamicMapping.mappings[]`（ロール・アプリ）/ `onNewUser.dynamicMapping.inventoryMappings[]`（**拠点 Inventory ロール ＝ 拠点分離の実装点**）/ `sessionConfiguration`。

**Keycloak 側の要件** [確]: OAuth2 authorization code grant のみ（**SAML 非対応**）／署名鍵は **RSA のみ**／リダイレクト URI に Edge のドメイン／バックチャネルログアウトは `https://<domain>/user/logout/oidc`。閉域網では **Edge → JWKS 到達性**と**ブラウザ → Keycloak 到達性**の両方が必要。

#### ⚠️ 必須ガード

| # | ガード |
|---|---|
| 1 | ログインモードで SSO を選ぶと**ログイン画面から Basic Auth / OAI-Secure の選択肢が消えます**。management テナントはローカル admin を維持 |
| 2 | **edge テナントにも break-glass アカウント（R-07）を残す。** ガード 1 は management しか守りません。⚠️ **他人のローカルユーザーのパスワードは管理者でも変更できない**（§4.10）ため、break-glass は「作っておく」だけでなく**有効性を定期確認する**必要があります |
| 3 | 再割当ポリシーの既定は「毎ログインで全ロール再計算（ルール外はクリア）」。**手動編集は次回ログインで上書き**されます |
| 4 | どのルールにもマッチしないと `access denied`（デフォルト拒否）。**クレーム名の 1 文字違いで全ユーザーがログイン不可**になります |
| 5 | デバイス認証とサービスユーザーは **SSO 対象外** |
| 6 | **順序**: 先に Edge 側でロールを定義 → 対応するグループを Keycloak で作る |
| 7 | ⚠️ **[要] SSO からローカル認証へ戻す手順を、実機で検証して runbook 化すること**（SV-04）。UI から選択肢が消えても `PUT /tenant/loginOptions/{typeOrId}` が Basic 認証の API で叩けるかが鍵 |

### 4.4 イベント/アラーム型の標準化

`alarm.type.mapping` はアラーム型ごとに severity とテキストを `<SEVERITY>|<TEXT>` で上書きします。severity `NONE` で抑止できます [確]。

| 分類 | 標準 severity | 自動クリア条件（S-03） |
|---|---|---|
| デバイス死活 `c8y_UnavailabilityAlarm`（既定 MAJOR / *"No data received from the device within the required interval."*） | MAJOR | 通信再開時 |
| カメラ死活（ONVIF/ICMP 応答なし） | MAJOR | 応答再開時 |
| 検知イベントからの昇格アラーム | 案件による | **要決定** |
| モデル更新失敗 | CRITICAL | 再適用成功時 |
| Cumulocity マイクロサービス異常 `c8y_Application_Down` / `c8y_Application_Unhealthy`（自動生成） | — | プラットフォーム管理 |

> ⚠️ `c8y_Application_*` の対象は **Cumulocity のマイクロサービス**です。VM1/VM2 上の非 Cumulocity コンポーネントは対象外で、そちらは Otel 側の責務です。

> **イベント型・アラーム型・ペイロード規約（S-04）を最初に確定してください。** 型名は `alarm.type.mapping` のキー、リテンションの `type` フィルタ、スマートルールの条件、EPL のマッチング、オブジェクトストレージ上の突合キーの全ての結節点です。**ここが後から変わると全部壊れます。**

### 4.5 死活監視とスケール

- `c8y_RequiredAvailability.responseInterval` を設定すると、指定時間内に通信が無い場合にアラームが自動生成されます
- ⚠️ **値域は `-32768`〜`32767`。範囲外は境界に丸められます** [確]
- 対象は画像解析装置と外部 Gateway。**カメラは child device で通信主体ではない**ため、死活は thin-edge の ONVIF/ICMP プラグインが報告します
- **[要] 装置種別ごとの具体値を決めてください**（SV-26）。過大なら故障を検知できず、過小なら回線瞬断のたびに数百件のアラームが上がり、`CLEARED` にならない限り消えません

#### ⚠️ スケールの根拠 [確] — 前版の訂正

「約 100 tps/CPU コア」は **2026 のページにも記載**があります（「2025 版のみ・版依存」は誤り）。さらに 2026 には実測ベンチマークがあります。

| シナリオ | CPU threads / RAM | 到達値 |
|---|---|---|
| Narrow（10 クライアント） | 8 / 16GB | 25,000 measurement/s |
| Narrow | 16 / 32GB | 47,500 measurement/s |
| **Wide（各 1 measurement/s）** | **8 / 16GB** | **1,200 接続クライアント** |
| **Wide** | **16 / 32GB** | **2,200 接続クライアント** |

> ⚠️ **Wide シナリオが本構成に直撃します。** 「1 拠点あたり数百台 × N 拠点」の child device 設計に対し、8 CPU で 1,200 クライアントが上限です。**D17 成立確認条件(1) の議論にこの数値を必ず引いてください。** 全拠点合算相当の負荷試験（SV-14）は P7 の必須工程です。

### 4.6 リテンションとオフロード

#### リテンション [確]

新規 Edge には既定ルール（60 日）が先に存在するため、**「あるべき集合」を宣言的に適用**します。

> ⚠️ **前版の「全削除 → 再作成」は危険なので順序を反転します。**
>
> - 削除が成功して作成が失敗すると、**リテンションルールがゼロの状態で放置**され、誰も気づかないまま Operational Store が単調増加します
> - 保持日数を**短縮する方向**の変更は**不可逆**（削除されたデータは戻りません）
>
> **手順**: ①現行ルールを GET してタグ付き Git コミット ②**新ルールを先に作成** ③旧ルールを個別に削除 ④`--dry` で事前確認、全コマンドに `--session` を明示。短縮方向の変更は別承認フローを通す。

> ⚠️ **`AUDIT` の日数が未決のまま 6-1 を実行しないでください。** 既定ルールを消すと再作成すべき値が必要になります。K-c（設定変更の追跡要件）から逆算して決めてください。

#### オフロード

**Data Hub は Edge では "On request via Professional Services"** です。これは技術的制約ではなく**商流の制約**ですが、コスト・リードタイム的に本フェーズでは非現実的です。

| 選択肢 | 前提 | 評価 |
|---|---|---|
| **(A) Notification 2.0 で購読 → 独自コンシューマ** | Messaging Service は **2026 Edge に Included**（P-4）。⚠️ **変-1 により、通知のために Notification 2.0 はどのみち稼働させる**ので、追加コストがほぼ無くなった | **再評価の価値あり**。取りこぼしのない永続トピックで受けられる |
| **(B) REST ページングで定期エクスポート** | なし | **推奨（初期版）**。構成図の「定期エクスポート」とも整合。実装が単純で失敗モードが読みやすい |

> **変-1 による選択肢の変化**: rev.2 では「(A) は Messaging Service の導入コスト（CPU +2 / RAM +4GB / PV 3 本）が重い」ことが (B) を推す理由でしたが、**通知が Notification 2.0 になった以上そのコストは既に払っています**。ただし (A) は `measurements` を `tenant` 文脈で購読できない（§4.8.2）等の制約があるため、**オフロード対象に計測値を含めるなら (B) が必要**です。初期版 (B) の判断は維持しますが、理由が変わっています。

> ⚠️ **(B) の実装規約**: `acl.algorithm-version` の OPTIMIZED は**一致件数が閾値（既定 2000）未満のときだけ適用**され、超えると LEGACY に落ちます [確K]。日次一括エクスポートはまさに 2000 件超になるため、**`prev` / `next` リンクで辿ることを実装規約にしてください**（*"navigation links via 'prev' and 'next' will work properly and this should be the only way of iterating through multiple pages"*）。ページ番号を自前で加算する実装は壊れます。

> ⚠️ **オフロードは「アーカイブ」ではなく「運用知識基盤のデータソース」です**（§4.9.1）。**優先度を「あとで実装する保管機能」から「初期に成立させる機能」へ引き上げてください。** 稼働開始から 90 日以内に成立していないと、最初のリテンション削除で長期データが恒久的に失われます。

> **オブジェクトストレージへの流入は 2 系統**: ①C8Y のオフロード ②GSC の映像クリップ + サイドカー JSON（S3 互換）。突き合わせには **external ID の安定性**が前提です（§4.2）。

### 4.7 AI モデル配布

| 段階 | 主体 | 手段 |
|---|---|---|
| 1. モデル取得 | 保守端末 | クラウドから DL（保守 VPN） |
| 2. リポジトリ登録 | 保守端末 → Cumulocity | `c8y software create` / `c8y software versions create` |
| 3. 更新 Operation 発行 | 保守端末 → Cumulocity | `c8y operations create` / `c8y bulkoperations create` |
| 4. 配信 | Cumulocity → デバイス | **拠点発の既存 MQTT 接続**（D9 と整合） |
| 5. 適用 | thin-edge sm-plugin | 失敗時ロールバック |

**Cumulocity 側の必要設定**: デバイスの `c8y_SupportedOperations` に `c8y_SoftwareUpdate` が含まれること（thin-edge の c8y-mapper が申告）[確] ／ **[要] `files/max.size` の実値確認**（SV-13。AI モデルのサイズ上限になる）／ Operation のリテンション（X-01）。

### 4.8 通知 — Notification 2.0

構成図は `イベント/アラーム処理・通知 --[通知 (Notification 2.0)]--> 他アセット(案件側アプリ)` です。**Cumulocity の標準機能で成立します**（前版で問題にしていた「ネイティブ webhook が無い」制約は論点から外れました）。

代わりに、**§4.1 の RBAC バイパスが構成の中心問題に昇格します。** 通知の受け手が案件側アプリになったことで、拠点分離を保つ設計が必須になりました。

#### 4.8.1 モデル — push ではなく WebSocket 購読 [確]

Notification 2.0 は Cumulocity が外部エンドポイントを叩く push ではなく、**コンシューマ側から WebSocket で接続して読む**方式です。Messaging Service（N-03）がトピックにメッセージを永続化します。

```
① 基盤が サブスクリプション を定義   POST /notification2/subscriptions   [ROLE_NOTIFICATION_2_ADMIN]
                                       → Messaging Service にトピックが対応づく
② 案件アプリが トークン を取得       （基盤のトークン発行プロキシ経由。§4.8.3）
③ 案件アプリが WebSocket 接続        トークンで認証 → サブスクライバが生成される
④ メッセージを受信し ack
```

#### 4.8.2 ⭐ サブスクリプションは拠点グループ単位に絞れる [確]

**これが本構成にとって決定的です。** `NotificationSubscription` スキーマの主要フィールド:

| フィールド | 値 | 本構成での使い方 |
|---|---|---|
| `context` **(必須)** | `mo` \| `tenant` | **`mo`**（`tenant` はテナント全体で拠点分離ができない） |
| `source` | managed object のグローバル ID | **拠点グループの MO ID**（`context: mo` のとき必須） |
| `subscription` **(必須)** | トピック名。**パターン `^[a-zA-Z0-9]+$`（英数字のみ）** | ⚠️ 拠点名に日本語やハイフンは使えない。**拠点コードの英数字命名規約が必要**（G-01 と連動） |
| `subscriptionFilter.apis` | `mo` 文脈: `alarms` / **`alarmsWithChildren`** / `events` / **`eventsWithChildren`** / `managedobjects` / `measurements` / `operations` / `*` | **`alarmsWithChildren`, `eventsWithChildren`** |
| `subscriptionFilter.typeFilter` | 型の完全一致、または `or` の OData 式（例 `'c8y_Temperature' or 'c8y_Pressure'`） | 基盤標準アラーム型（S-04）で絞る |
| `fragmentsToCopy` | 指定した独自フラグメント**のみ**を含める | ⚠️ **生体情報・個人情報に関わるフラグメントを案件アプリへ渡さない**ために使える |
| `nonPersistent` | boolean | 既定 `false`（永続）。⚠️ `subscription` 名が同じでも `nonPersistent` が違えば**別トピック**になる |

> **`alarmsWithChildren` / `eventsWithChildren` は、`source.id` の managed object と、その配下の全 descendant managed object のアラーム/イベントを購読します** [確]。**拠点グループを `source` にすれば、その拠点配下の全デバイスが 1 サブスクリプションで covered** されます。

**context ごとの API 対応表** [確]:

| Context | MO Create | MO Update & Delete | Alarms | Events | Measurements | Operations |
|---|---|---|---|---|---|---|
| `mo` | **✗** | ✓ | ✓ | ✓ | ✓ | ✓ |
| `tenant` | ✓ | ✗ | ✓ | ✓ | **✗** | ✓ |

> ⚠️ **`mo` 文脈では「新規 managed object の作成」を受け取れません**（作成時点で ID が無いため）。新規デバイスの登録を検知したい場合は、**別途 `tenant` 文脈のサブスクリプションが必要**です。本構成では新拠点・新装置の追加を検知する用途で必要になる可能性があります → 基盤側の運用サービスが持ち、案件アプリには渡さないこと。

> ⚠️ **`tenant` 文脈では measurements を購読できません。**

#### 4.8.3 ⚠️ トークン発行プロキシ（N-02）— 拠点分離を保つ要

**問題**: サブスクリプション作成（`POST /notification2/subscriptions`）**もトークン取得（`POST /notification2/token`）も、どちらも `ROLE_NOTIFICATION_2_ADMIN` を要求します** [確]。このロールはテナントレベルの all-or-nothing で、付与された主体は**任意のデバイスのサブスクリプションを作れます**（§4.1）。

つまり、**案件アプリに直接このロールを与えると、拠点分離が即座に無効化されます。**

**解決**: 基盤標準部品として**トークン発行プロキシ**を置きます。

```
案件アプリ ──① OIDC トークンで認証 ──► トークン発行プロキシ（基盤標準・VM1）
                                          │ ② 呼び出し元が購読してよい拠点を判定
                                          │    （Keycloak のグループクレーム or Inventory ロール）
                                          │ ③ ROLE_NOTIFICATION_2_ADMIN を持つ
                                          │    基盤サービスユーザーとして
                                          ▼
                                      POST /notification2/token
                                      { subscriber, subscription: "<拠点コード>", expiresInMinutes }
案件アプリ ◄── ④ 当該トピックにスコープされたトークンのみ返す
```

- **発行されるトークンはトピックにスコープされます** [確]（JWT の `topic` クレーム。仕様の例: `"topic":"management/relnotif/testSubscription"`）。したがって、プロキシが渡すサブスクリプション名を絞れば、案件アプリは他拠点のトピックを読めません
- `ROLE_NOTIFICATION_2_ADMIN` を持つのは**プロキシのサービスユーザーだけ**（R-06 とは別に R-09 として定義）
- ⚠️ この設計は §4.1 **D-e**（案件アプリの REST 呼び出しはエンドユーザートークンで行う）と同じ思想です。**Notification 2.0 だけロールを直接渡す例外を作らないでください**

#### 4.8.4 ⚠️ コンシューマのライフサイクル（N-04）

| 事実 [確] | 帰結 |
|---|---|
| サブスクライバは**最初の WebSocket 接続時に生成**される | 事前作成は不要 |
| **一度作られたサブスクライバは、WebSocket が切断されても削除されない** | ⚠️ 停止した案件アプリの分のメッセージが**溜まり続ける** |
| Messaging Service は、消費されるか TTL に達するか**明示的に unsubscribe されるまで**メッセージを永続化する | ⚠️ **ディスク逼迫の経路**。単一 Edge に全拠点が集約されている本構成では影響が全拠点に及ぶ |
| 解除は `POST /notification2/unsubscribe?token=<token>` | 当該トピック・サブスクライバのトークンを作ってから呼ぶ |

**運用設計に必ず入れること**:

1. **廃止した案件アプリ・試験用サブスクライバの unsubscribe 手順**を runbook 化する
2. サブスクライバ数とバックログ量を**メタ監視の対象**にする（Prometheus エンドポイント §4.10 で取れるかは SV-33）
3. `expiresInMinutes` の既定は **1440 分（24 時間）** [確]。案件アプリ側にトークン再取得のロジックが必要
4. `shared: true` で共有コンシューマを作れる [確]。案件アプリを冗長化する場合に使う

#### 4.8.5 人への通知は案件アプリの責務になる

**変-2 によりメール通知を採用しないため、Cumulocity から人へ直接届く通知経路はありません。**

| 経路 | 状態 |
|---|---|
| スマートルール「On alarm send email / SMS / escalate」 | **不採用**（変-2） |
| Notification 2.0 → 案件アプリ → 人（アプリの通知機能） | **これが唯一の経路** |
| Cumulocity Web UI / 紐づけ確認アプリでの目視 | 能動的な通知ではない |

> ⚠️ **基盤自身の異常が誰にも届きません。** `c8y_Application_Down` / `c8y_Application_Unhealthy`（§4.4）や Edge 自体の障害は、メールが無い以上 **メタ監視（O-01）が唯一の検知経路**です。§4.10 の Prometheus エンドポイントを保守拠点側の監視に接続し、**アラート通知先を Cumulocity の外側に持つ**ことを設計に入れてください（SV-34）。

### 4.9 運用知識基盤 — Cumulocity の読み手であり書き手

**役割**: Otel インフラ経由のテレメトリ（画像解析装置＝Grafana Alloy 経由 / VM1・VM2 の各コンポーネント＝直接 OTLP）と、**Cumulocity Operational Store の格納値**をインプットに、各アセットのより良い設定を検討し、**設定の更新**を行う。

**配置**: 構成図上、**VM1 の内側**（Otel インフラの隣）です。ブラウザではなくサーバサイド常駐プロセスなので、**CORS（T-02）は不要**、資格情報は R-06 のローカルサービスユーザーになります。

> ⚠️⚠️ **[図外] 構成図に描かれているのは 2 本のエッジだけです**: `Otelインフラ --[テレメトリ]--> 運用知識基盤` と `運用知識基盤 --[設定の更新]--> 他アセット`。
>
> **本節が前提とする Cumulocity との読み書き経路 3 本は、いずれも図に存在しません。** ご説明（「Cumulocity Edge の Operational Store の格納値をインプットとして」「各アセットの設定を更新」）に基づく補完です。§7 に図の改訂要求として起票しています。

#### 4.9.1 読み取り経路

制約: 「Operational Store への直接アクセスは不可」（配置の要点）。REST API 経由に限られます。

| # | 用途 | 経路 | 確度 |
|---|---|---|---|
| 1 | テレメトリ（メトリクス/ログ/トレース） | **Otel バックエンドのクエリ API**。Cumulocity の設定対象外 | 図に有 |
| 2 | 直近の運用データ（〜90日） | `GET /event/events` / `/alarm/alarms` / `/measurement/measurements` / `/inventory/managedObjects`。**`next` リンクで辿る**（§4.6） | [図外] |
| 3 | 長期傾向 | **オブジェクトストレージ（オフロード済み）から読む** | [図外] |
| 4 | リアルタイム反応 | Notifications 2.0。⚠️ RBAC バイパス（§4.1） | [図外] |

> ⚠️ **リテンション設計との相互作用 — 設計判断の分岐点です。**
> 分析に必要なデータ期間が 90 日を超えるなら、**X-01 のリテンションを延ばすのは誤った解決**です。Operational Store（MongoDB）を長期保管に使うと §4.5 のスケール制約に直撃します。**オフロード（§4.6）を先に成立させ、長期分析はオフロード先を読む**構成にしてください。**必要期間の確定は SV-20。**

#### 4.9.2 書き込み経路

**まず対象を確定してください。** 構成図の「設定の更新」矢印は `他アセット` にのみ引かれていますが、ご説明では「各アセット」です。**SV-17 が本節全体の要否を決めます。**

| 対象 | Cumulocity のデバイスか | 配置 | 設定配布経路 |
|---|---|---|---|
| **画像解析装置** | ○ | 現場 | 構成管理オペレーション（thin-edge）。⚠️ **届くのは thin-edge が申告した設定タイプのみ。** `画像解析パイプライン` の設定を変えるには、その設定ファイルを `tedge-configuration-plugin.toml` に登録しておく必要がある |
| **外部 Gateway（thin-edge 部分）** | ○ | VM1 | 同上 |
| **外部 Gateway（BOXアダプタのマッピング定義）** | ✕（VM1 コンポーネント） | VM1 | Ansible 等。**[要] どちらで配るか未決**（SV-27） |
| **IP カメラ**（fps・解像度・検知感度・`responseInterval`） | ○（child device） | 現場 | thin-edge 経由の child device オペレーション **[要]**（SV-28） |
| **Cumulocity 自身**（スマートルール閾値 S-01 / `alarm.type.mapping` T-03 / `c8y_RequiredAvailability` D-04 / リテンション X-01） | — | VM1 | **構成管理ではなく Inventory UPDATE / テナントオプション PUT の世界**。R-06 の書込アカウントに `ROLE_INVENTORY_ADMIN` が必要 |
| 画像センシングBOX | ○（child device） | 現場 | **改造不可のため設定変更対象外** |
| VM1/VM2 の各コンポーネント | ✕ | VM1/VM2 | Ansible 等。Cumulocity の対象外 |
| サイトVMS / GSC | ✕ | 現場/本社 | D13 の最小 5 操作に設定変更は含まれない → 対象外 |
| 他アセット（図の矢印の宛先） | ✕ | VM2 | アプリ側 API |

#### ⚠️ 構成管理の正しい仕組み [確] — 前版の訂正（訂-6）

**前版は `c8y_Configuration` を構成配布の手段として挙げましたが、これは誤りです。** Cumulocity の設定管理は 3 系統に分かれます。

| 系統 | フラグメント | thin-edge 標準マッパー |
|---|---|---|
| **Typed file-based（推奨・設定リポジトリを使う）** | `c8y_SupportedConfigurations`（申告・SmartREST 119）/ `c8y_DownloadConfigFile` `{type, url}`（配信・524）/ `c8y_UploadConfigFile` `{type}`（吸い上げ・526） | **対応** |
| Legacy file-based | — | — |
| Text-based | `c8y_Configuration` `{config: string}`（SmartREST 113/513） | **非対応**（別途コミュニティプラグイン `c8y-textconfig-plugin` が必要） |

**thin-edge の c8y-mapper が申告する supported operations** [確]: `c8y_SoftwareUpdate`, **`c8y_UploadConfigFile`**, **`c8y_DownloadConfigFile`**, `c8y_Firmware`, `c8y_Restart`, `c8y_LogfileRequest`, **`c8y_DeviceProfile`**, `c8y_RemoteAccessConnect`, `c8y_Command`。設定管理は `tedge-agent` が既定で提供し、管理対象ファイルは `/etc/tedge/plugins/tedge-configuration-plugin.toml` で宣言します。

**Cumulocity 経由で設定を配る場合に必要なもの**:

| # | 必要なもの | 備考 |
|---|---|---|
| 1 | `c8y_SupportedConfigurations` の申告 | デバイス側（tedge-agent） |
| 2 | 設定リポジトリにスナップショットを登録（K-03） | `c8y configuration create` |
| 3 | 配信 | `c8y configuration send`（type + url） |
| 4 | ⚠️ **現在値の吸い上げ（`c8y_UploadConfigFile`）** | K-d のロールバック退避も K-04 の逸脱検出も**これが前提**。前版は配信側しか書いていなかった |
| 5 | ⚠️ **配信は URL フェッチ型** | デバイスが Edge の HTTP エンドポイント（thin-edge の `c8y.http`）に到達でき、**社内 CA を信頼している**必要がある。§4.2 の `c8y.root_cert_path` と直結 |
| 6 | デバイスプロファイル（K-04） | `c8y deviceprofiles create`。型ごとの「あるべき設定の束」 |
| 7 | 権限 | `ROLE_DEVICE_CONTROL_ADMIN` + **`ROLE_INVENTORY_ADMIN`**（設定リポジトリは managed object + バイナリ） |

#### 4.9.3 ⚠️ ガードレール — 技術的強制力のあるものを主にする

**運用知識基盤は、この構成で唯一「機械が判断して機械が書く」経路**です。方針ではなく**技術的に強制できる統制**を主にしてください。

| ID | ガードレール | 内容 | 強制力 |
|---|---|---|---|
| **G-1** | **キルスイッチ** | ①書込アカウントの即時無効化 ②一括オペレーションのキャンセル ③PENDING オペレーションの一括削除 — の 3 手順を runbook 化し、**P7 で実際に実行して止まることを確認する** | 強 |
| **G-2** | **ブラストレディウスの権限的制限** | ⚠️ **D-d の再検討結果**: 書込アカウントに拠点横断権限を**常時持たせない**。配信対象の拠点グループに対する Inventory ロールを**承認時に一時付与**する。これにより「拠点分離の外側」に立つ時間を最小化できる | 強 |
| **G-3** | **配信前の値検証** | 配信可能な設定キーの**ホワイトリストと値域**を設定リポジトリ側に定義し、範囲外は配信前に拒否。単位違い（秒 vs ミリ秒）や null でも thin-edge は「適用成功」を返すため、K-d のロールバックは発動しない | 強 |
| **G-4** | **`creationRamp` の必須指定** | ⚠️ **前版の訂正（訂-7）**: bulk operation リクエスト本文の **`creationRamp`**（*"Delay between every operation creation in seconds."*、例 `15`）が**発行側で指定できる主たる流量制御**です。CI 側で必須パラメータ化してください。システムオプション `device-control/bulkoperation.creationramp` は実在しますが**読み取り専用**で、意味（既定値か下限か）を述べた一次記述は見つかっていません **[推]** | 強 |
| **G-5** | **段階ロールアウトと打ち切り基準** | 拠点グループ（G-01）を配信単位にし、1 拠点 → 数拠点 → 全拠点。**各段階に「進行条件」を数値で置く**（適用後 N 時間、対象拠点の検知イベント数・アラーム数・オペレーション失敗率が基準内）。`failedParentId` で失敗分だけ再スケジュールできる | 中 |
| **G-6** | **入力データの鮮度・完全性ゲート** | オフロードが静かに失敗 → 「イベントが激減した」と誤解釈 → 検知しきい値を下げる配信 → 誤検知爆発、を防ぐ。分析実行前に入力データの期間・件数・拠点カバレッジを検査し、閾値未満なら**出力自体を抑止**。オフロードジョブの死活をメタ監視に入れる | 強 |
| **G-7** | **デバイスプロファイルとの排他** | K-04 が「逸脱」判定 → 準拠のため再適用 → 運用知識基盤が再配信、の**フラッピング**でオペレーションが無限生成される。プロファイル管理キーと自動最適化キーを**排他**にするか、運用知識基盤が**プロファイル自体を更新**してデバイスへ直接配信しない設計にする | 強 |
| **G-8** | **変更理由のトレーサビリティ** | 監査ログに残るのは「誰が何を書いたか」だけ。配信するオペレーション／設定に**相関 ID**（分析ジョブ ID・入力データ期間・モデルバージョン・承認者）をフラグメントとして埋める | 中 |
| **G-9** | **クールダウン** | 最適化ロジックが 2 値の間で振動し毎時書き換える事故を防ぐ。デバイス単位・キー単位で「同一キーは 24 時間に 1 回まで」等 | 中 |
| **G-10** | **除外タグ** | 「絶対に自動変更してはいけない」拠点・装置を managed object のフラグメント（例 `x_NoAutoConfig`）で保護。P7 で実際に除外されることを確認 | 中 |
| **G-11** | **変更ウィンドウと凍結期間** | 顧客拠点の稼働ピークやイベント当日の配信を防ぐ。配信可能時間帯をスケジューラで強制し、凍結日リストを持つ | 中 |
| **G-12** | **人手承認（初期版）** | **初期版は「提案のみ・適用は人手承認」。** 保守端末（既存の承認導線）を通す。完全自動化は運用実績を積んでから | 強 |
| **G-13** | **監査ログ保持（K-c）** | リテンションの `AUDIT` を**イベント 90 日とは別基準**に。設定変更の追跡・障害調査要件から逆算 | 中 |
| **G-14** | **資格情報管理（K-f）** | R-06 の資格情報は CI シークレットストア。`credentials.*` としてテナントオプションに置かない | 中 |

> **設計判断: アカウントを 2 つに分ける（R-06）**
> 分析処理は読み取り専用アカウントで常時稼働させ、設定適用のみ書き込み権を持つ別アカウントで承認を伴う経路からのみ実行する。単一アカウントだと**分析処理のバグが本番デバイスの設定を壊せる状態**になります。実装コストはユーザーを 1 つ増やすだけです。

### 4.10 Edge 自体の運用 — バックアップ・メール・監視・ブランディング [確]

**前版で【欠】としていた項目群です。2026 ドキュメントに記載があります（訂-11）。**

#### バックアップとリカバリ（E-09 / SV 解決済み）

| 形態 | 保全対象 | 復元手順 |
|---|---|---|
| **c8yedge（K3s）** | **`/var/lib/rancher/k3s`** | 同一（または互換）OS を用意 → データ復元（**パス・所有者・権限を保持**）→ `sudo c8yedge install --version <previous_version>`（オフラインは `-s c8yedge.tar`） |
| 自前 K8s | **PVC と Edge CR** | 同上の考え方 |

> ⚠️ *"Installing a different Edge version on top of a restored data set is unsupported and may fail the upgrade guard rails."* **復元時は同一バージョンが必須**です（E-07）。

> ⚠️ **D17 により全拠点が 1 台の Edge に集約されています。** ディスク障害・MongoDB 破損・誤配信のいずれでも全拠点が同時に停止します。**P1 の最後にスナップショット取得と「リストア試験の実施」を必須ゲートにし、これが完了するまで P2 以降に進まないでください。** RTO/RPO を数値で決め、BOX と thin-edge のバッファ保持時間を実測して「Edge 停止許容時間」として明記してください（SV-29）。

#### メールサーバーを設定しない判断とその帰結（変-2）

**メール通知を採用しないため、メールサーバーは設定しません。** 公式はメールサーバーが 2 つの機能を担うと述べています [確]:

> *"Configuring an email server enables you to receive email notifications about events, alarms, and also to reset your password. In case you forget the password, the Edge mails you the password reset link to reset your password."*

**ただし「パスワードリセットが使えない」影響は限定的です。** 本構成のユーザーの大半は Keycloak 管理の SSO ユーザーで、**Cumulocity 側にパスワードを持ちません**。ケース別に整理します。

| ケース | メール要否 | 根拠 |
|---|---|---|
| **SSO ユーザー（Keycloak 管理）** | **不要** | Cumulocity 側にパスワードが存在しない。*"password reset in Cumulocity is disabled for users created through an external authentication server."* **リセットは Keycloak の機構で行う** |
| ローカルユーザーの**新規作成時**にパスワードを設定 | **不要** | `POST /user/{t}/users` の `password` フィールド（writeOnly・6〜32 文字）で直接指定できる。UI にも「Set password for the user」がある |
| **自分自身**のパスワード変更 | **不要** | `PUT /user/currentUser/password`（`currentUserPassword` + `newPassword`） |
| **他人のローカルユーザーのパスワードをリセット** | ⚠️ **必要** | 下記 |

> ⚠️ **想定と違う制約**: 「運用側（管理者）が他ユーザーのパスワードをリセットすればよい」は、**現行の Cumulocity では API 上できません** [確]。
>
> - OpenAPI 仕様（`PUT /user/{tenantId}/users/{userId}`）: *"Note that you cannot update the password or email of another user, they can only be updated for the current user."*
> - go-c8y-cli `users resetUserPassword`: *"The password can be reset either by issuing a password reset email (default), or be specifying a new password."* に続けて *"In more recent Cumulocity versions, you can't set a fixed password for another user."*
>
> **つまり他人のローカルユーザーに対する唯一の正規リセット手段がメールです。**

**回避策（メール無しで成立する運用）**:

| # | 手段 | 適用対象 | 留意点 |
|---|---|---|---|
| 1 | **削除 → 同名で再作成**（`POST` はパスワードを直接指定できる） | サービスアカウント（R-05, R-06, R-08, R-09） | ユーザー ID が変わるため、**ロール・インベントリロール・アプリケーションアクセスの再割当が必要**。投入スクリプト（§6）が冪等なら再実行で復旧できる ＝ **P3 のスクリプト化がそのまま復旧手段になる** |
| 2 | **admin が複数居る状態を保つ** | management / edge の管理者 | ただし手段 1 と同じ制約（他人のパスワードは変えられない）を受けるため、**admin 自身の資格情報は削除・再作成では救えない** |
| 3 | **資格情報のエスクロー保管** | **両テナントの admin と break-glass（R-07）のみ** | 下記 |

> ⚠️ **残る唯一の復旧不能ケース**: **両テナントの admin と break-glass の資格情報を全て失った場合**です。ほかに管理者が居なければ、削除・再作成を実行する主体そのものが居ません。単一 Edge に全拠点が集約されている構成では影響が全拠点に及びます。
>
> 対象を絞ったうえで、次の統制は必須としてください:
> 1. **両テナントの admin と break-glass の資格情報を封緘してエスクロー保管**（保管場所・開封手順・開封記録）
> 2. **四半期ごとにログイン可能性を確認**する（必要になる前に有効性を確かめる）
> 3. **管理者アカウントのパスワード有効期限（A-05）は無期限にする**。期限切れは自分で変更できるので致命的ではないが、**期限切れに気づかないまま緊急時を迎える**事故を避けるため
>
> **サービスアカウントについてはエスクローは不要**です（手段 1 で復旧できるため）。CI シークレットストアでの管理を継続してください。

> **将来メール通知を採用する場合**: Management テナントにログインし、Password reset テンプレートと Email server（host / port / protocol / username / password / sender address）を設定します。完了条件は「**テストメールが実際に受信できること**」です。

#### 監視（O-01）

> *"The Edge operator exposes a **Prometheus-compatible metrics endpoint, `https://<domain>:3443/metrics`**, where the domain is the one you configured in the Edge CR."*

**OTLP ネイティブ出力は未記載ですが、Prometheus エンドポイントが公式提供されています。Otel Collector の prometheus receiver で取り込むのが正攻法**です。同ページに Prometheus / Grafana の導入節があります。

#### ブランディング（SV 解決済み）

Management テナント → Administration → **Settings > Enterprise tenant** → Branding。**「Edge はマルチテナンシー非対応だから Enterprise 機能は使えないのでは」という懸念は否定されます** [確]。

---

## 5. 適用手順

各フェーズは**独立して再実行可能（冪等）**であることを目標とします。冪等にできない工程（P0）は明確に分離します。

### P0. 設置 — 冪等化不可・1 回きり

> ⚠️ **P-0（ネットワーク前提）が全て潰れるまで 0-4 に進まないこと。** install 後には直せません。

| # | 工程 | 完了条件 |
|---|---|---|
| 0-1 | **検証環境 Edge の構築** | SV 群（§8）の実機検証をここで消化。**本番でリテンション検証をすると本番データが消えます** |
| 0-2 | ドメイン 2 つの DNS 登録 | `<domain>` と `management-<domain>` が名前解決される |
| 0-3 | TLS 証明書の発行（**SAN に両ホスト名**） | PEM・完全なチェーン・正しい順序 |
| 0-4 | **ライセンスと registry credentials の受領** | 両方が揃っている（P-3） |
| 0-5 | Edge インストール | **install 引数に最終ドメイン・パスワード・ライセンス・TLS を全て渡す。** 完了条件＝両 URL でログイン可能 |
| 0-6 | `c8yedge-operator-config` ConfigMap（`ca.crt` / `no_proxy`）+ operator 再起動 | 閉域網では必須（E-11） |
| 0-7 | 社内 CA ルート証明書の配布 | 全ブラウザ・thin-edge デバイス・**Edge operator** が Edge を信頼する |

> ⚠️ **ドメイン変更は P0 完了後に行わないでください。** TLS 証明書の再発行、Keycloak の redirect URI / backchannel logout URL、全 thin-edge の `c8y.url` と `c8y.mqtt`、全ブラウザの信頼設定を巻き込みます。デバイス接続開始後に気づくと拠点に人を出す作業になります。

### P1. Edge 基盤層の確定 — 冪等

| # | 工程 | 完了条件 |
|---|---|---|
| 1-1 | Edge CR のエクスポートと Git コミット | `status`/`uid` 等を除去済み |
| 1-2 | admin パスワードの変更（両テナント独立） | 変更後に両テナントでログイン可能 |
| 1-3 | **両テナント admin と break-glass の資格情報をエスクロー保管**（§4.10。サービスアカウントは対象外） | 封緘・保管場所・開封手順が文書化され、実際に保管されている |
| 1-4 | マイクロサービス配備の**確認と是正** | Apama-ctrl / Smartrule が稼働。無ければ CR を修正して適用（SV-09） |
| 1-5 | ⚠️ **Messaging Service の稼働確認（N-03）** | Notification 2.0 の前提。2026 Edge には `Included` だが**実際に動いていることを確認** |
| 1-6 | ⚠️ **スナップショット取得 + リストア試験** | **試験成功まで P2 に進まない** |

### P1.5 CLI セッションの準備 — 冪等

> ⚠️ **セッションモードを設定しないと投入コマンドが全て無効になります**（*"all commands which create, update and/or delete data are disabled by default"*）。**`--mode prod` は Create/Update/Delete が全て無効**で、スクリプトが**エラーも出さずに何も投入せず完走**します。

| # | 工程 |
|---|---|
| 1.5-1 | management / edge の 2 セッションを作成。**`C8Y_MODE=ci`（`prod` は使わない）** |
| 1.5-2 | ヘッドレス用に `c8y sessions login --from-env` を確立（セッションファイル暗号化のパスフレーズ要求を回避） |
| 1.5-3 | 投入用ブートストラップユーザーの資格情報を確定 |
| 1.5-4 | **本書の全コマンド例に `--session <name>` を付与する規約を徹底** |

### P2〜P6

> **各フェーズの先頭で「対象リソースの GET エクスポート + タグ付きコミット」を実施してください。** これが P7 の diff の基準（期待値 = Git 定義、現物 = 実機、差分ゼロが合格）になり、切り戻しの起点にもなります。

| フェーズ | # | 工程 | 依存 |
|---|---|---|---|
| **P2** テナント基盤 | 2-1 | テナントオプション（T-01, T-04） | — |
| | 2-2 | CORS（T-02）— **紐づけ確認アプリを含む全 VM2 オリジン** | オリジン一覧の確定 |
| | 2-3 | フィーチャートグル（P-04） | — |
| **P3** 権限 | 3-1 | グローバルロール定義（R-01〜R-03） | — |
| | 3-2 | ロールへの権限割当 | 3-1 |
| | 3-3 | Inventory ロール定義（R-04） | — |
| | 3-4 | サービスユーザー（R-05） | 3-1 |
| | 3-5 | **運用知識基盤アカウント 2 つ（R-06）** | SV-17 |
| | 3-6 | **break-glass（R-07・両テナント）** | 3-1 |
| | 3-7 | **証明書アップロード用ローカルユーザー（R-08）** | — |
| | 3-7b | **トークン発行プロキシ用サービスユーザー（R-09）** — `ROLE_NOTIFICATION_2_ADMIN` はここだけ | 3-1 |
| | 3-8 | **パスワードポリシー / TFA（A-05）** — SSO を迂回できる最強権限の経路を保護。⚠️ **管理者アカウントの有効期限は無期限に**（§4.10） | 3-4〜3-7 |
| | 3-9 | **サービスアカウント復旧手順の確認** — 削除 → 同名で再作成 → ロール再割当が投入スクリプトの再実行で完了することを検証環境で確認（§4.10） | 3-4〜3-7b |
| **P4** 拠点とデバイス | 4-1 | 拠点 device group 階層（G-01） | — |
| | 4-2 | グループの external ID（G-02） | 4-1 |
| | 4-3 | テナント CA 作成（D-01） | SV-08 |
| | 4-4 | trusted certificate 登録（方式(2)の場合） | — |
| | 4-5 | 一括登録 CSV 投入 | 4-3, SV-07 |
| | 4-6 | **`autoRegistrationEnabled` の無効化**（方式(2)を使った場合） | 4-5 |
| **P5** SSO | 5-1 | Keycloak 側クライアント設定 | 3-1〜3-3 |
| | 5-2 | `authConfig` 投入（A-01, A-02） | 5-1, 4-1 |
| | 5-3 | ⚠️ **切替前ゲート**: SSO ユーザー 1 名で実際にログインでき、期待するロールと Inventory ロールが付与されることを、**ログインモードを切り替えずに**確認 | 5-2 |
| | 5-4 | ログインモード切替（A-03） | **5-3 合格が必須** |
| | 5-5 | フォールバック確認（management ローカル admin + edge break-glass） | 3-6 |
| **P6** データ・アプリ | 6-1 | リテンション（X-01）— **新規作成 → 旧削除の順**。`AUDIT` 値の確定が前提 | G-13 |
| | 6-2 | `alarm.type.mapping`（T-03） | S-04 の確定 |
| | 6-3 | 紐づけ確認アプリ登録（P-01, P-02） | SV-24 |
| | 6-4 | EPL apps（S-02） | Apama-ctrl 購読 |
| | 6-5 | Analytics Builder モデル | 拡張のインストール（**閉域網では事前準備が必要**） |
| | 6-6 | スマートルール（S-01） | **投入ガイド `U-01` の型名実測が前提** |
| | 6-7 | **アラーム自動クリア（S-03）** | 6-2 |
| | 6-8 | 標準ダッシュボード（P-03） | 6-3, SV-11 |
| | 6-9 | ソフトウェアリポジトリ（M-01） | — |
| | 6-10 | **オフロードバッチの配備と初回実行（X-03）** | ⚠️ **稼働開始から 90 日以内が期限** |
| | 6-11 | 設定リポジトリ・デバイスプロファイル（K-03, K-04） | **SV-17 で対象確定後** |
| | 6-12 | **Notification 2.0 サブスクリプション定義（N-01）** — 拠点グループごとに `context: mo` + `source` = 拠点グループ MO + `alarmsWithChildren` | 4-1（グループ）, 4-2（external ID）, S-04（型規約） |
| | 6-13 | **トークン発行プロキシの配備（N-02）** | 3-7b, 6-12, 5-2（OIDC 検証のため） |
| | 6-14 | **コンシューマ棚卸し手順の整備（N-04）** — unsubscribe runbook + バックログ監視 | 6-12 |

### P7. 検証

| # | 工程 |
|---|---|
| 7-1 | **設定の往復差分**（Git 定義 vs 実機、差分ゼロが合格） |
| 7-2 | システムオプションのアサート（`GET /tenant/system/options`） |
| 7-3 | 実デバイス 1 台の疎通 → **正しい拠点グループに現れるか** |
| 7-4 | ⚠️ **合算規模の負荷試験**（§4.5 の Wide シナリオ 1,200 クライアントに対する実測）。合格基準を tps・E2E 遅延・CPU・MongoDB IO で数値化 |
| 7-5 | SSO ログイン（各ロール）— 期待通りの拠点だけが見えるか |
| 7-6 | ⚠️ **権限分離の否定テスト**: ①拠点 A のユーザーの資格情報で拠点 B のデータを **API 直叩き**し 403/404 になること ②`ROLE_NOTIFICATION_2_ADMIN` が **R-09 以外のどのユーザー・ロールにも付いていない**ことを**スクリプトで機械検査**（CI に載せる）③手動付与したロールが次回ログインで消えることの確認 |
| 7-6b | ⚠️ **通知の否定テスト**: 拠点 A の案件アプリの認証情報でトークン発行プロキシに**拠点 B のサブスクリプション**を要求し、拒否されること。また拠点 A 用トークンで拠点 B のトピックに WebSocket 接続できないこと |
| 7-6c | **通知の到達試験**: 拠点 A でアラームを発生させ、①拠点 A の案件アプリに届く ②拠点 B の案件アプリには**届かない** ③`fragmentsToCopy` で除外したフラグメントが**含まれない**ことを確認 |
| 7-6d | **コンシューマ棚卸し試験（N-04）**: 案件アプリを停止 → バックログが増えることを観測 → `POST /notification2/unsubscribe` で解除できることを確認 |
| 7-7 | ⚠️ **スマートルールの発火試験**: テストデバイスから条件を満たすデータを投入し、**アラームが実際に生成されること**を標準ルールごとに確認。「一覧に見える」を完了条件にしない |
| 7-8 | ⚠️ **死活監視の実効確認**: テスト装置の通信を止めて N 分後にアラームが上がり、復旧で自動クリアされること |
| 7-9 | **CORS 検証**: VM2 の各アプリから実際にブラウザ経由で Edge の REST を叩き 200 が返ること |
| 7-10 | **オフロード検証**: オフロード先の件数が Cumulocity 側と一致し、external ID が含まれ GSC サイドカー JSON と突合できること |
| 7-11 | **キルスイッチ試験（G-1）**: 実際に一括オペレーションを発行し、3 手順で止まることを確認 |
| 7-12 | **除外タグ試験（G-10）**: `x_NoAutoConfig` を付けたデバイスが配信対象から外れること |
| 7-13 | 証明書有効期限が監視対象に入っていること（P-2, §4.2） |

---

## 6. ツール選定と実装方式

### 6.1 ツールの評価

| ツール | 用途 | 採否 |
|---|---|---|
| **go-c8y-cli (`c8y`)** | L1-B / L1-C の全て | **採用（主）** |
| **`kubectl` + Edge CR** | L0 | **採用（従）**。宣言的で `apply` が冪等 |
| **`c8yedge` CLI** | L0 のインストールと設定 | **採用（従）**。ただし**設定の読み取りサブコマンドは文書化されていない** → 取得は `kubectl get -o yaml` |
| **c8y-tedge**（go-c8y-cli 拡張） | thin-edge デバイスのブートストラップ | **評価対象**。thin-edge 公式組織・MIT。SSH 経由で証明書を作成・アップロード。ただし star 0 / fork 2 と実績が薄い |
| **Terraform provider（community）** | テナント設定の IaC | **不採用**。`bjoernHeneka/cumulocity` は 30+ リソース・Registry 公開・MPL 2.0 だが**コミット 3 件・非公式**。将来の再評価対象 |
| **cumulocity-migration-tool** | テナント間移行 | **使用禁止**。2025-09-17 にアーカイブ済み |

> **公式の「テナント丸ごとエクスポート/インポート」機能は存在しません。**

**検証済みのコマンド**（go-c8y-cli sitemap の全列挙で確認）: `tenantoptions updateBulk`, `retentionrules`, `usergroups create`, `userroles addRoleToGroup`, `users create`, `applications createHostedApplication`, `features enable/disable`, `software create` / `software versions create`, `operations create`, `bulkoperations create`, `systemoptions list`, **`configuration create/send/list/update/delete`**, **`deviceprofiles create`**, `sessions login`。

`inventoryroles` はトップレベルに存在しないため `c8y api` で `/user/inventoryroles` を直接叩きます。`analytics` も存在せず拡張が必要です。

### 6.2 リポジトリ構成

> **本節は [投入ガイド](Cumulocity設定エクスポート投入ガイド.md) §6.1 のディレクトリ構成案を置き換えます**（`platform/` と `site/` の分離が本構成の要求のため）。

```
config/
  edge/                              # L0（環境ごと）
    edge.yaml                        # kubectl get edge -o yaml（status/uid 除去）
    operator-config.yaml             # ca.crt / no_proxy（E-11）
  platform/                          # L1-B: 基盤標準初期値（環境非依存・製品）
    tenantoptions.alarm-type-mapping.json
    usergroups.json  inventoryroles.json  features.json
    retentionrules.json
    event-payload-spec.md            # S-04（規約文書）
    epl/*.mon  analytics/ab/*.json  dashboards/*.json
  site/                              # L1-C: 構成固有（環境ごと）
    tenantoptions.access.control.json  # CORS: VM2 の全オリジン
    loginoptions.oauth2.json           # Keycloak（id 除去済み）
    devices.bulk.csv                   # 拠点グループ階層 + デバイス登録
    users.json  inventoryrole-assignments.json
    deviceprofiles/*.json              # K-04
  mgmt/                              # management テナント管轄
    mailserver.json  branding/
binaries/                            # ZIP は Git LFS または成果物リポジトリ
secrets/                             # 値は入れない。キー名の一覧のみ
scripts/
  export.sh  import.sh  assert.sh
```

### 6.3 冪等化パターンの割り当て

| パターン | 適用先 |
|---|---|
| A: 名前ベース参照 | ロール割当（`--group` / `--role` がワイルドカードを受ける） |
| B: PUT 先行 → 404 なら POST | `loginOptions`（SSO 設定） |
| C: 存在チェック → 分岐 | ユーザー、グローバルロール定義 |
| D: 宣言的な集合適用 | リテンション（**新規作成 → 旧削除の順**。§4.6） |
| E: 重複エラー黙殺 | Inventory ロール定義、グループ |

### 6.4 CI/CD 実行時の注意

- **`--mode prod` は使わない**（Create/Update/Delete が全て無効。無言で何も投入されない）
- ヘッドレスでは `c8y sessions login --from-env`
- management / edge の 2 セッションを `--session` で明示
- `--dry --dryFormat json` の出力に **`Authorization: Basic` ヘッダが含まれ得る**
- Windows/PowerShell では `eval "$(...)"` とシングルクォートによる `!` の保護が動作しない → [投入ガイド](Cumulocity設定エクスポート投入ガイド.md) §1.2 の PowerShell 版

---

## 7. 構成図への改訂要求

**§4.9 は構成図に存在しない経路を前提としています。** 図の改訂を提案します。

| # | 追加/修正すべきエッジ | 理由 |
|---|---|---|
| 図-1 | `運用知識基盤` → `REST / MQTT API`「運用データ参照（REST・ページング）」 | §4.9.1 の読み取り経路 2 |
| 図-2 | `オブジェクトストレージ` → `運用知識基盤`「長期傾向データ参照」 | §4.9.1 の読み取り経路 3。オフロードの位置づけが変わる |
| 図-3 | `運用知識基盤` → `REST / MQTT API`「構成管理オペレーション発行」 | §4.9.2（SV-17 が「含める」の場合） |
| 図-4 | `メタ監視` の向きを拠点発 push に変更、または D9 の例外である旨を凡例に明記 | §2 注-1 |

> **rev.4 で対応済み**（削除済みの要求）:
> - ~~図-5~~ ダングリングエッジ「映像入力」「イベント/アラーム(ローカルMQTT)」の source を接続 → `映像入力` は `サイトVMS → 画像センシングBOX`、`イベント/アラーム(ローカルMQTT)` は `画像解析パイプライン → thin-edge.io` として接続済み
> - ~~図-6~~ `thin-edge.io(child) → イベント/アラーム処理` を `REST/MQTT API` 経由に修正 → `thin-edge.io(child device管理)` の接続先を `REST/MQTT API` に変更済み

---

## 8. 未確定事項

> **ID を `SV-nn` に変更しました。** 既存 2 文書の `V-nn`（設定定義書）・`U-nn`（投入ガイド）と衝突していたためです。

| # | 項目 | 確認方法 | 優先度 | 既存文書 |
|---|---|---|---|---|
| **SV-17** | **運用知識基盤の「設定の更新」対象に画像解析装置・カメラを含めるか** | **設計判断（実機不要）**。含めないなら K-03〜K-05 / R-06 書込側が丸ごと不要になりリスクが激減 | **最高** | — |
| **SV-14** | 全拠点合算のスケール | PoC 実測。§4.5 の Wide 1,200 クライアントが基準 | **最高** | 定義書 V-01 相当 |
| **SV-01** | 閉域網でのライセンス更新 / 1 インスタンス全拠点収容の商流 | ベンダー照会 | 高 | 定義書 V-01 |
| **SV-04** | **SSO からローカル認証へ戻す手順** | 検証環境で実機確認 → runbook 化 | **最高** | — |
| **SV-29** | RTO/RPO、BOX・thin-edge のバッファ保持時間 | 実測 + 設計判断 | **最高** | — |
| **SV-05** | **Edge 上での Notification 2.0 の実動作** | 検証環境（Messaging Service は `Included` と確定済み）。⚠️ **変-1 により通知の唯一の手段になったため最優先**。WebSocket エンドポイントが Edge の LoadBalancer 経由で外部から到達できるか（P-0-2 のポートに含まれるか）も併せて確認 | **最高** | — |
| **SV-35** | **`alarmsWithChildren` が device group の `childAssets` を辿るか** | 仕様は *"all of its descendant managed objects"* とあるが、グループ配下（`childAssets`）とデバイス配下（`childDevices`）の両方を辿るかは未確認。**拠点グループ単位の購読が成立するかの根幹**。検証環境で拠点グループを `source` にして配下デバイスのアラームが届くか確認 | **最高** | — |
| SV-06 | イベント添付バイナリがリテンションの対象か | **検証環境で**短期リテンションを設定して観測 | 高 | — |
| SV-07 | 一括登録 CSV で `ENROLLMENT_OTP` と `PATH` を併用できるか | 検証環境で 1 行の CSV を投入 | 高 | — |
| SV-08 | Edge 版での device enrollment / テナント CA の可用性 | 検証環境で `POST /certificate-authority` | 高 | — |
| SV-09 | K8s 形態で実際に配備されるマイクロサービス | `kubectl get pods -n c8yedge` | 中 | — |
| SV-10 | Apama の HTTP クライアント接続プラグインが Edge で使えるか | ⚠️ **変-1 により優先度が下がりました。** webhook は不要になり、D14 の自動クリップ保存も Notification 2.0 のサブスクライバで実装できます。EPL から直接 HTTP を出したい場合にのみ必要 | 低 | — |
| SV-33 | Notification 2.0 のサブスクライバ数・バックログ量を監視できるか | Prometheus エンドポイント（§4.10）に該当メトリクスがあるか。**無い場合は棚卸し運用（N-04）が唯一の歯止めになる** | 高 | — |
| SV-34 | **基盤自身の異常のアラート通知先** | ⚠️ **変-2 によりメールが無いため、`c8y_Application_Down` 等が誰にも届かない。** メタ監視（O-01）の Prometheus を保守拠点側の監視基盤に接続し、**Cumulocity の外側に通知先を持つ**設計が必要 | **高** | — |
| SV-36 | 案件アプリのトークン再取得ロジック | `expiresInMinutes` の既定は 1440 分。案件側の実装要件として合意が必要 | 中 | — |
| SV-37 | **Edge のホスト側から admin パスワードを復旧する手段があるか** | ⚠️ **メール不採用（変-2）で残る唯一の復旧不能ケース**（§4.10）を潰せるか。初期パスワードは K8s Secret `cumulocityPasswordSecretName` 由来だが、インストール後に Secret を変更してもパスワードが変わるかは未確認。**検証環境で試す価値が高い**（成立すればエスクローの必要性が下がる） | **高** | — |
| SV-38 | UI から管理者が他ユーザーのパスワードを変更できるか | API は明示的に不可（§4.10）。UI に別経路があるかは公式ドキュメントが明示していない | 中 | — |
| SV-11 | Cockpit の Import/export が Edge で使えるか | 検証環境 | 中 | ガイド U-03 |
| SV-13 | `files/max.size` の実値 | `GET /tenant/system/options` | 中 | — |
| SV-18 | thin-edge 設定管理の Edge 上での実動作（オペレーション名は確定済み） | 検証環境 | 中 | — |
| SV-19 | `bulkoperation.creationramp` の意味（既定値か下限か） | ベンダー照会。**G-4 が主統制なので優先度は低い** | 低 | — |
| SV-20 | 運用知識基盤が必要とする分析データの期間 | **設計判断**。90 日超ならオフロードが唯一のデータ源 | 高 | — |
| SV-21 | D6/D7/D15 の責務境界（thin-edge を載せる筐体の所有・構成管理主体） | 設計判断 → `design-decisions.md` の改訂 | 高 | — |
| SV-22 | 証明書・社内 CA ローテーション手順（新旧並行の移行期間） | 設計 + 検証環境 | 高 | — |
| SV-23 | メタ監視の向き（D9 との整合） | 設計判断 | 中 | — |
| SV-24 | 紐づけ確認アプリを EXTERNAL 登録にするか Cumulocity にホストするか | 設計判断 | 高 | — |
| SV-25 | テナント CA 自動更新（毎年 10/2）時の `simplereenroll` 動作 | 検証環境でシミュレート | **高** | — |
| SV-26 | `responseInterval` の装置種別ごとの値 | 設計判断（値域 -32768〜32767） | 高 | — |
| SV-27 | BOX アダプタのマッピング定義を Cumulocity 経由で配るか Ansible か | 設計判断 | 中 | — |
| SV-28 | カメラ（child device）への構成オペレーションが thin-edge 経由で成立するか | 検証環境 | 中 | — |
| SV-30 | `PUT /tenant/options/{category}` のマージ／置換セマンティクス | 検証環境。置換なら再実行で設定が壊れる | 高 | — |
| SV-31 | ブランディングの tenant option キー名 | 差分方式で特定 | 低 | 定義書 V-04 |
| SV-32 | スマートルールの managed object の `type`/`fragmentType` | 検証環境で GUI 作成 → `c8y inventory find` で観測 | **高** | **ガイド U-01** |

### 着手前に必ず埋めるべき具体値

| 項目 | 該当 | なぜ最初か |
|---|---|---|
| **イベント型・アラーム型・ペイロード規約**（`modelVersion` 含む） | S-04, T-03 | `alarm.type.mapping` / リテンションの type フィルタ / スマートルール条件 / EPL マッチング / 突合キーの結節点。**後から変えると全部壊れる** |
| **external ID の採番規約**（仮名化要否を含む） | D-06 | 重複排除 / 映像クリップ突合 / 特徴量紐づけの**三重の役割** |
| ドメイン 2 つの FQDN と証明書 SAN | P-2 | P0 の後戻り不可 |
| Keycloak の realm / clientId / issuer / JWKS URL / **グループクレーム名** | A-01, A-02 | 1 文字違いで全ユーザーがログイン不可 |
| **拠点コードの英数字命名規約** | G-01, N-01 | Notification 2.0 の `subscription` 名は**パターン `^[a-zA-Z0-9]+$`**。日本語・ハイフン・アンダースコアが使えないため、拠点グループの命名と別に**トピック名用の拠点コード**が要る |
| リテンションの `AUDIT` 日数 | X-01, G-13 | 6-1 の実行に必須 |
| `responseInterval` の装置種別ごとの値 | D-04 | 過大＝検知漏れ、過小＝アラーム氾濫 |
| CORS で許可する VM2 オリジンの完全なリスト | T-02 | **紐づけ確認アプリを含む** |
| R-01〜R-08 の実際の `ROLE_*` 名 | §3.2 | `ROLE_INVENTORY_UPDATE` は存在しない |

---

## 9. 出典

### Edge 基盤層
- [Installing Edge (2026)](https://cumulocity.com/docs/2026/edge/installing-edge/) — インストール方式、2 テナント、TLS 要件、**ライセンス取得手順**、**registry credentials**、前提要件
- [Edge introduction (2026)](https://cumulocity.com/docs/2026/edge/edge-introduction/) — **機能比較表（Included の一覧）**、100 tps/CPU コア
- [Manage Edge (2026)](https://cumulocity.com/docs/2026/edge/manage-edge/) — **メールサーバー**、**外部 IP とポート一覧**、`c8yedge config` の実コマンド
- [Edge operations (2026)](https://cumulocity.com/docs/2026/edge/edge-operations/) — **バックアップとリカバリ**、**Prometheus メトリクスエンドポイント**
- [Benchmarks (2026)](https://cumulocity.com/docs/2026/edge/benchmarks/) — **実測スケール値**
- [Using Edge (2026)](https://cumulocity.com/docs/2026/edge/using-edge/) — **ブランディング**
- [Edge custom resource definition (2026)](https://cumulocity.com/docs/2026/edge/edge-custom-resource-definition/) — **spec 12 項目**
- [同 (2025)](https://cumulocity.com/docs/2025/edge-kubernetes/edge-custom-resource-definition/) — `messagingService` / `microservices` / `dataHub`（**2025 限定**）

### Core API・デバイス
- [Cumulocity Core OpenAPI](https://cumulocity.com/api/core/dist/c8y-oas.yml) — Notifications 2.0 の RBAC バイパス、`bulkNewDeviceRequests`、`bulkOperation.creationRamp` / `groupId` / `failedParentId`、`c8y_Configuration`、ロール名
- [Fragment library](https://cumulocity.com/docs/device-integration/fragment-library/) — `c8y_SupportedConfigurations` / `c8y_DownloadConfigFile` / `c8y_UploadConfigFile`
- [Managing device data](https://cumulocity.com/docs/device-management-application/managing-device-data/) — 設定管理 3 系統
- [Monitoring and controlling devices](https://cumulocity.com/docs/device-management-application/monitoring-and-controlling-devices/) — bulk operation ウィザード
- [Certificate authority](https://cumulocity.com/docs/device-certificate-authentication/certificate-authority/) / [Device certificates](https://cumulocity.com/docs/device-certificate-authentication/device-certificates/)
- [Inventory roles performance improvements (KB)](https://community.cumulocity.com/t/inventory-roles-performance-improvements/513) — **OPTIMIZED の 2000 件閾値**、`prev`/`next`

### 認証・データ・ルール
- [Single sign-on](https://cumulocity.com/docs/authentication/sso/) / [Basic settings](https://cumulocity.com/docs/authentication/basic-settings/)
- [Managing users](https://cumulocity.com/docs/standard-tenant/managing-users/) — *"password reset in Cumulocity is disabled for users created through an external authentication server."*、ユーザー作成時のパスワード直接指定
- [Cumulocity Core OpenAPI](https://cumulocity.com/api/core/dist/c8y-oas.yml) — `user` スキーマの `password`（writeOnly, 6〜32）/ `sendPasswordResetEmail` / `shouldResetPassword`、`PUT /user/{tenantId}/users/{userId}` の *"you cannot update the password or email of another user"*、`PUT /user/currentUser/password`
- [c8y users resetUserPassword](https://goc8ycli.netlify.app/docs/cli/c8y/users/c8y_users_resetuserpassword/) — *"In more recent Cumulocity versions, you can't set a fixed password for another user."*
- [Managing data](https://cumulocity.com/docs/standard-tenant/managing-data/) / [Ecosystem](https://cumulocity.com/docs/standard-tenant/ecosystem/)
- [Smart rules](https://cumulocity.com/docs/cockpit/smart-rules/) / [Alarm mapping](https://cumulocity.com/docs/standard-tenant/alarm-mapping/)

### Notification 2.0（rev.3 で追加）
- [Cumulocity Core OpenAPI](https://cumulocity.com/api/core/dist/c8y-oas.yml) — `NotificationSubscription` スキーマ（`context` / `source` / `subscription` のパターン `^[a-zA-Z0-9]+$` / `subscriptionFilter.apis` の `alarmsWithChildren` / `eventsWithChildren` / `typeFilter` / `fragmentsToCopy` / `nonPersistent`）、`NotificationTokenClaims`（`expiresInMinutes` 既定 1440 / `shared` / `signed`）、context × API 対応表、`POST /notification2/subscriptions` `/token` `/unsubscribe` の必要ロール `ROLE_NOTIFICATION_2_ADMIN`、**inventory role RBAC バイパスの警告**、コンシューマのバックログ・ライフサイクル
- [Monitoring (standard tenant)](https://cumulocity.com/docs/standard-tenant/monitoring/) — トピック名とサブスクライバ名の対応、サブスクライバの生成・非削除、unsubscribe 手順

### thin-edge.io・ツール
- [c8y-mapper reference](https://thin-edge.github.io/thin-edge.io/references/mappers/c8y-mapper/) — **supported operations の完全リスト**
- [Configuration management](https://thin-edge.github.io/thin-edge.io/operate/c8y/configuration-management/) / [Device profiles](https://thin-edge.github.io/thin-edge.io/operate/c8y/device-profiles/) / [Connect to Cumulocity](https://thin-edge.github.io/thin-edge.io/start/connect-c8y/)
- [go-c8y-cli](https://goc8ycli.netlify.app/docs/) / [thin-edge/c8y-tedge](https://github.com/thin-edge/c8y-tedge) / [bjoernHeneka/terraform-provider-cumulocity](https://github.com/bjoernHeneka/terraform-provider-cumulocity)

### 社内文書
- [Cumulocity設定定義書.md](Cumulocity設定定義書.md) — ⚠️ §0.2 の訂-1〜訂-3 が該当
- [Cumulocity設定エクスポート投入ガイド.md](Cumulocity設定エクスポート投入ガイド.md) — ⚠️ §6.1 は本書 §6.2 が置き換え
- [design-decisions.md](../../IoTPlatform_cc/design-decisions.md) — ⚠️ D17 のライセンス記述と D6/D7/D15 は改訂要（SV-21）
- [cumulocity-edge-sso-user-permission.md](../../IoTPlatform_cc/cumulocity-edge-sso-user-permission.md)
- [cumulocity-iot-architecture.drawio](cumulocity-iot-architecture.drawio) — **「全体構成(配置構成・レビュー反映)」タブのみ**
