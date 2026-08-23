# Cumulocity 利用設計書

**— 本構成における Cumulocity Edge の使い方の定義と、その設定への落とし込み —**

**対象構成**: [cumulocity-iot-architecture.drawio](cumulocity-iot-architecture.drawio) —「全体構成(配置構成・レビュー反映)」タブ**のみ**
**版**: **rev.4（2026-08-23）** — **バックアップ設計の完成**（CU-33 / CU-30 の推奨設計・取得/復元 runbook・K3s / CA の一次情報補強 — §12.6）と、**運用設計の新設**（§12.10 — 3 文書横断のカバレッジ調査で判明した欠落: Edge アップグレード・計画停止・ホスト OS・しきい値・棚卸し・監査レビュー・Keycloak 依存ほか）。rev.3 は個人情報を含むスナップショット画像のアクセス監査（前提 9 / CU-18）、rev.2 は一次情報照合（ファクトチェック）の反映。[レビュー結果](Cumulocity利用設計書_ファクトチェック結果.md) / [詳細レポート](factcheck/)
**位置づけ**: **Cumulocity 側設計の「正」**。本構成において Cumulocity Edge を「何のためにどう使うか」を Cumulocity の概念軸（グループ / デバイス / データ種別 / ルール / 通知 / ロール / テナントオプション）で定義し、その定義を Cumulocity Edge の**設定項目・設定手段・設定値**へ落とすところまでを一冊で担う
**ベース文書**: [IoT連携チーム_担当範囲設計書.md](IoT連携チーム_担当範囲設計書.md)（以下 **[担当範囲]**）
**前提**: 外部Gateway の配置は**案 β**（拠点ごとに thin-edge インスタンスを分ける・[担当範囲] §2.5）。案 α を採る場合の差分は**付録 A**

> **読み方のショートカット**: 初読は §0 →§1 →§2 →§6 を推奨します。**設定を投入する立場の方は §13（設定設計）から読み、そこから各定義章へ遡ってください。** §13.2 が定義と設定項目の対応表です。

---

## 0. 本書の位置づけと前提

### 0.1 目的とスコープ

本書は次の 4 つの問いに答えます。

| # | 問い | 答える章 |
|---|---|---|
| Q1 | **Cumulocity 上でデバイスをどう表現し、どう管理するか** | §1〜§5 |
| Q2 | **そのデバイスが扱うデータをどう定義するか** | §6〜§7 |
| Q3 | **そのデータをどう処理し、誰にどう届け、誰に何を見せるか** | §8〜§11 |
| Q4 | **それらを実現するために Cumulocity Edge に何をどう設定するか** | §12〜§13 |

**スコープ内**:

- Cumulocity Edge 上の設計（デバイスモデル・グループ・データ型・ルール・通知・ロール・テナントオプション・リテンション）
- Cumulocity Edge の設置と設定投入の設計（設定項目・手段・値・順序・冪等化方針）
- 上記を検証するための観点

**スコープ外**（他文書が正）:

- thin-edge.io 側の実装設計（プラグイン・`tedge config`・mosquitto バッファ）→ **[担当範囲] §5〜§7**
- 誰が何を担当するか、いつやるか → **[担当範囲] §1・§3・§8**
- go-c8y-cli の実装技法（テンプレート・`--select`・シークレット） → **[投入ガイド]**
- BOXアダプタ・画像解析パイプラインの実装 → 各担当チーム

### 0.2 既存文書との関係と「正」の所在

**本構成には既に 6 つの文書があります。同じ設計が複数箇所に書かれると保守が破綻するため、「どの文書が正か」を先に固定します。**

| 文書 | 何の正か | 本書との関係 |
|---|---|---|
| **本書** | **Cumulocity 側の設計と設定値** | — |
| [IoT連携チーム_担当範囲設計書.md](IoT連携チーム_担当範囲設計書.md)（**[担当範囲]**） | **担当範囲・thin-edge 側実装・WBS・Ready for Service** | 本書 §1〜§11 の Cumulocity 側定義を [担当範囲] が参照する。**thin-edge の設定値・プラグイン設計は [担当範囲] が正** |
| [Cumulocity構成準拠セットアップ設定書.md](Cumulocity構成準拠セットアップ設定書.md)（**[設定書]**） | **適用手順（P0〜P7）と冪等化の運用** | ⚠️ **[設定書] §4「主要設計の詳細」は本書 §1〜§11 へ移管**（§0.3）。[設定書] は**手順とフェーズ割付に純化**する |
| [Cumulocity設定定義書.md](Cumulocity設定定義書.md)（**[定義書]**） | 設定項目の網羅カタログ・API エンドポイント一覧 | 参照のみ。**[設定書] §0.2 の訂正が優先**。本書は §13 で「本構成で実際に使う項目」だけを抜き出す |
| [Cumulocity設定エクスポート投入ガイド.md](Cumulocity設定エクスポート投入ガイド.md)（**[投入ガイド]**） | go-c8y-cli の実装技法・冪等化パターン A〜E | 本書 §13.5 が参照する。**リポジトリ構成は [設定書] §6.2 → 本書 §13.6 が置き換える** |
| [../../IoTPlatform_cc/data-integration-spec_new.md](../../IoTPlatform_cc/data-integration-spec_new.md)（**[連携仕様]**） | IF-nn 単位のデータ連携仕様 | 本書 §6 のデータ型定義が [連携仕様] §4.3 を統合・拡張する |
| [../../IoTPlatform_cc/architecture-camera-monitoring.md](../../IoTPlatform_cc/architecture-camera-monitoring.md)（**[カメラ構成]**） | デバイスモデル・ペイロード規約の原典 | 本書 §6 の原典 |
| [../../IoTPlatform_cc/design-decisions.md](../../IoTPlatform_cc/design-decisions.md)（**[判断記録]**） | D1〜D17 の設計判断 | §0.5 で引き継ぐ |

### 0.3 ⚠️ [設定書] §4 からの移管対象

**本書を「正」とする決定に伴い、[設定書] の次の記述は本書へ移管します。[設定書] 側は本書への参照に置き換えてください。**

| [設定書] の節 | 内容 | 本書での受け先 |
|---|---|---|
| §4.1 拠点分離 | Notification 2.0 の RBAC バイパス・グループ階層・D-a〜D-e | **§1.6 / §10.3 / §11.7** |
| §4.2 デバイスオンボーディング | 登録方式 3 種・一括登録 CSV・thin-edge との分界点・external ID の三重の役割 | **§2.2 / §3.3〜§3.5** |
| §4.3 Keycloak SSO | `authConfig` の主要フィールド・必須ガード 7 点 | **§11.6** |
| §4.4 イベント/アラーム型の標準化 | `alarm.type.mapping`・分類表 | **§9** |
| §4.5 死活監視とスケール | `c8y_RequiredAvailability`・ベンチマーク値 | **§5.2 / §7.7** |
| §4.6 リテンションとオフロード | リテンション適用順序・オフロード方式 (A)/(B)・ページング規約 | **§7.2 / §7.5** |
| §4.7 AI モデル配布 | 5 段階の配布フロー | **§4.4** |
| §4.8 通知 (Notification 2.0) | モデル・サブスクリプション・トークン発行プロキシ・ライフサイクル | **§10** |
| §4.9 運用知識基盤 | 読み書き経路・ガードレール G-1〜G-14 | **§11.1 / §11.9 / §4.10** |
| §4.10 Edge 自体の運用 | バックアップ・メール不採用・監視・ブランディング | **§12.6〜§12.8** |

**[設定書] に残るもの**: §0（訂正記録）・§1（着手前前提 P-0〜P-4）・§2（構成図トレース）・§3（レイヤ別項目 ID の定義）・§5（適用手順 P0〜P7）・§6（ツール選定）・§7（構成図改訂要求）・§8（未確定事項 SV-nn）。

> ⚠️ **本書の作成をもって [設定書] が自動的に改訂されるわけではありません。** [設定書] 側の改訂（§4 の削除と本書への参照置換）を別作業として起票してください。**それまでは §4 の記述と本書が二重に存在します。齟齬があった場合は本書が優先です。**

### 0.4 確度の凡例

[設定書] §0.4 / [担当範囲] §0.3 を継承します。

| 記号 | 意味 |
|---|---|
| **[確]** | 公式ドキュメントまたは OpenAPI 仕様の逐語記述で確認済み |
| **[確K]** | Cumulocity Tech Community の Knowledge Base 記事で確認（公式 docs ではない） |
| **[確S]** | thin-edge.io のソースコードで確認 |
| **[推]** | 一次情報からの演繹。直接の記述は無い |
| **[要]** | 未確認。実機検証またはベンダー確認が必要。§15 に一覧 |
| **[判]** | 技術的制約ではなく**設計判断**。本書で提案し、合意を求める |

### 0.5 引き継ぐ設計判断

本書は次を所与とします。ここを覆すと本書の広範囲が影響を受けます。

| # | 判断 | 本書への効き方 |
|---|---|---|
| **D9** | 拠点発アウトバウンドのみ。中央から拠点への着信接続なし | §4.7（リモートアクセス封じ込め）・§4.4（モデル配布は既存の拠点発 MQTT 接続に乗る）・§12.7（メタ監視の向き） |
| **D13** | VMS 連携は抽象レイヤー経由。映像バイト列は基盤を通さない | §6.8（添付はスナップショットのみ。映像クリップは Cumulocity に入らない） |
| **D14** | 保存対象は Alarm 昇格分の自動保存 + 手動指示分のみ | §8.2（昇格ルール）・§10.2（購読の先で `exportClip` を叩く） |
| **D15** | 映像解析 AI は別筐体。AI 側は標準ペイロード規約にのみ従う | §6.5（ペイロード規約が本チームの提供物になる） |
| **D16** | 標準エージェントのランタイムは thin-edge.io。カメラは child device として代理登録 | §3.1（デバイス階層モデル）の前提そのもの |
| **D17** | 全拠点を単一 Cumulocity Edge・単一テナントに集約。拠点区分は device group + Inventory ロール | **§1 の前提そのもの**。§11.3・§10.2 と一体 |
| **変-1** | 通知は Notification 2.0 | §10。**拠点グループ単位のサブスクリプションが成立することが §1 のグループ設計の要件になる** |
| **変-2** | メール通知は不採用 | §10.6・§12.8。**基盤自身の異常の通知先を Cumulocity の外に持つ必要がある** |

### 0.6 用語の定義

**本書で最も誤解を生みやすい 4 組を先に固定します。**

| 用語 | 定義 | 混同しやすいもの |
|---|---|---|
| **資産階層**（`childAssets`） | グループとデバイスで作る**業務上の所属関係**。Inventory ロールの継承経路 | 接続階層 |
| **接続階層**（`childDevices`） | thin-edge の main device と child device が作る**通信上の親子関係** | 資産階層。**thin-edge の child 登録は資産階層を作りません**（§1.3） |
| **main device** | thin-edge 1 インスタンスが代表する Cumulocity 上の 1 デバイス。証明書 1 枚・MQTT 接続 1 本 | ホスト・筐体。**画像解析装置という筐体と、その上の thin-edge の main device は別概念** |
| **child device** | main device 経由で代理登録されるデバイス。証明書を持たない | 独立デバイス |
| **拠点（サイト）** | 顧客の物理拠点。Cumulocity 上では**グループ**として表現する | Cumulocity のテナント |
| **テナント** | Cumulocity の管轄単位。Edge は **`management` と `edge` の 2 つ**を持つ | 拠点。**拠点ごとのテナント分離はできません**（Edge はシングルテナント） |
| **`edge` テナント** | 業務データ・デバイス・ユーザーの本体。`https://<domain>` | `management` テナント |
| **`management` テナント** | Edge プラットフォーム設定・ブランディング。`https://management-<domain>` | `edge` テナント |

### 0.7 本書が前提とする未決事項

**本書は次を「そうなる」と仮定して書いています。仮定が崩れた場合の影響範囲を明示します。**

| # | 前提 | 崩れたときの影響 | 対応する未確定事項 |
|---|---|---|---|
| **前提 1** | 外部Gateway は**案 β**（拠点ごとに thin-edge インスタンス） | §1.2 の階層図・§6.2 のデータ経路・§10.2 の購読設計が変わる → **付録 A** に差分を用意 | [担当範囲] W0-10 |
| **前提 2** | `alarmsWithChildren` が**グループの `childAssets` を辿る** | **§1 の拠点分離方式と §10 の購読設計が同時に破綻する。D17 の実現方式そのものを再設計** | **SV-35 / TE-12（最優先）** |
| **前提 3** | Cumulocity の **certificate-authority（EST）が Edge で使える** | §3.3 の登録方式が「自前 CA を trusted certificate に登録する従来方式」に落ち、§3.6 の証明書自動更新を別途設計する必要がある | **SV-08 / TE-8（最優先）** |
| **前提 4** | Edge 上で **Notification 2.0 が実動作する**（WebSocket が LoadBalancer 経由で到達できる）。**機能として使えること自体は [確]**（`Messaging Service: Included`） | 到達できなければ §10 の経路が成立しない。**変-2 と併せて、通知経路がゼロになる** | **SV-05** |
| **前提 5** | イベント/アラーム型・external ID 採番規約（§2.2・§6.4）が**関係チームと合意される** | §9 のマッピング・§7.2 の type フィルタ・§8 のルール条件・§10.2 の `typeFilter` が全部やり直し | [担当範囲] C-1 / C-2 |
| **前提 6** 〈rev.2〉 | Edge 同梱の Apama-ctrl で **EPL Apps が使える** | **§8 の RL-b・RU-2〜RU-5 が全て EPL 前提。§8 の過半が成立しない** | **SV-45（最優先）** |
| **前提 7** 〈rev.2〉 | Inventory ロールが **`childDevices`（カメラ・BOX）まで継承される** | **拠点 Manager がカメラ個別のアラーム・インベントリを見られず §11.3 が崩れる**。前提 2 と同型 | **SV-47（最優先）** |
| **前提 8** 〈rev.2〉 | リテンションの「既定 60 日」が**削除可能なルールとして存在する** | **システム設定側の値なら、90 日ルールを入れても 60 日で消える。§7.2 が崩れる** | **SV-43（最優先）** |
| **前提 9** 〈rev.3〉 | スナップショット画像を **Cumulocity イベント添付に置いたままでよい**（＝画像アクセスの監査証跡が要件にならない） | **画像が個人情報を含み監査証跡が要件になる場合、参照は監査ログに残らず（§6.8 制約 5）権限分離もできない（制約 6）。§6.8 の初期版設計・§11.3 のロール割当・§13 の投入対象・構成図が変わる** | **CU-18**（⚠️ 実機不要なのは (A)/(A′)/(C) のみ。**(D)/(B) の評価には CT-5c が要る**） |

> ⚠️ **前提 2・3・6・7・8 は「検証環境で最初に潰す」項目です。** 本書の設計を本番へ投入する前に、[担当範囲] W1-2 / W1-3 / W1-10 を完了させてください。

> ⚠️ **前提 9 は性質が違います** 〈rev.3〉。技術的な未確認ではなく **案件・法務との合意事項**です。ただし **5 案のうち (D)/(B) は CT-5c（Edge 導入後の検証環境）の結果が無いと評価できない**ため、「合意だけ先に取る」ことができません。**W0 で (A)/(A′)/(C) の可否を詰め、(D)/(B) を残すなら CT-5c を §12.4 直後に前倒ししてください**（§13.4）。

> ⚠️⚠️ **rev.2 で前提が 3 つ増えました。** いずれも rev.1 が [確] としていた（あるいは疑問にすら上がっていなかった）事項が、一次情報照合で未確認と判明したものです。**前提 2 と前提 7 は「グループやデバイスの配下をどこまで辿るか」という同一の疑問**であり、**同じ階層構成で同時に検証してください**（§14 CT-4）。

### 0.8 本書の構造

```
定義（何をどう表現するか）
  §1  デバイス管理グループ ── 拠点分離の器
  §2  デバイス識別・命名規約 ── 全章の結節点
  §3  デバイス管理方式 ── 登録・証明書・ライフサイクル
  §4  デバイスオペレーション管理方式 ── Cumulocity → デバイスの制御
  §5  死活監視方式
      ↓
  §6  データモデル ── デバイスが出すデータの定義（MQTT プレフィックス含む）
  §7  データ保持・オフロード・容量
      ↓
  §8  条件一致メッセージの処理 ── ルール
  §9  アラームマッピング定義
  §10 通知（Notification 2.0）── 処理結果の配信
      ↓
  §11 ロール・アクセス制御 ── 誰に何を見せるか

実装（どう設定するか）
  §12 Cumulocity Edge セットアップ・運用設計（バックアップ §12.6 / 監視 §12.7 / 定常運用 §12.10）
  §13 Cumulocity 設定設計 ← §1〜§11 の定義を設定項目に落とす

確認
  §14 検証観点   §15 未確定事項   付録A 案α差分   §16 出典
```

---

## 1. デバイス管理グループの定義

### 1.1 グループが担う 5 つの役割

**Cumulocity の device group は「見やすくするための入れ物」ではありません。本構成では 5 つの機能の実装基盤です。** どれか 1 つでも欠けると設計が成立しません。

| # | 役割 | 根拠 | 効く章 |
|---|---|---|---|
| **1** | **拠点分離の単位** | D17。Edge はシングルテナントで拠点ごとのサブテナントを作れない [確]。グループ + Inventory ロールが唯一の代替 | §11.3 |
| **2** | **Inventory ロールの割当単位** | ロールは親グループ → サブグループ → **それらのグループに属するデバイス**へ継承される [確]。⚠️ **その先の `childDevices`（カメラ・BOX）まで届くかは未確認**（SV-47・§11.3） | §11.3 |
| **3** | **Notification 2.0 サブスクリプションの `source`** | `context: mo` + `source` = 拠点グループ MO + `alarmsWithChildren` で拠点単位の購読になる **[推]**。⚠️ **「descendant」がどの関係を辿るかは公式に記述が無い**（前提 2 / SV-35・§10.2） | §10.2 |
| **4** | **一括オペレーションの対象指定** | bulk operation の `groupId` は *"Identifies the target group on which this operation should be performed."* [確]。**API 経由の一括オペレーションはグループ指定が事実上の前提** | §4.8 |
| **5** | **段階ロールアウトの単位** | AI モデル配布・設定配布を 1 拠点 → 数拠点 → 全拠点で進める単位 | §4.9 |

> ⚠️⚠️ **役割 2 と 3 は、いずれも「デバイス配下の子デバイスまで届くか」という同一の未確認事項に依存しています。**
> - 役割 3（通知）＝ 前提 2 / SV-35 → §10.2
> - 役割 2（Inventory ロール）＝ SV-47 → §11.3
>
> **rev.1 は役割 3 を [確] としていましたが、7 経路の探索の結果「公式のどこにも書かれていない」ことが確定したため [推] に格下げしました**（§10.2）。§1.6 の [要] 表記と整合させています。**両方を同じ階層構成で同時に実機検証してください**（§14 CT-4）。成立しない場合の代替は §1.6 に置きます。

### 1.2 階層設計（案 β 採用時の最終形）

```
[group] 拠点 (root)                      extID: sites
├── [group] 拠点A                        extID: site-001    ★Inventory ロールの割当単位
│   │                                                        ★Notification 2.0 の source
│   │                                                        ★bulk operation の groupId
│   │
│   ├── [device] 画像解析装置             extID: site-001:ANLZ-<serial>
│   │   │                                 type:  x_ImageAnalyzer   ← thin-edge A の main device
│   │   │                                 ★資産階層に直接所属（プロビジョニングで 1 回）
│   │   └── [child] IPカメラ ×N           extID: site-001:CAM-<serial>
│   │                                     type:  x_Camera
│   │                                     ★接続階層で自然に配下に入る（追加作業なし）
│   │
│   └── [device] BOXゲートウェイ          extID: site-001:GW-BOX
│       │                                 type:  x_BoxGateway      ← thin-edge B の main device
│       │                                 ★資産階層に直接所属（プロビジョニングで 1 回）
│       └── [child] 画像センシングBOX ×M  extID: site-001:BOX-<serial>
│                                         type:  x_SensingBox
│                                         ★接続階層で自然に配下に入る（追加作業なし）
│
├── [group] 拠点B …
└── [group] 共通                          extID: common
    └── （どの拠点にも属さない管理対象があれば置く。案 β では原則不要）
```

**案 β の効果**: main device が 2 つとも拠点グループに直接所属するため、**拠点あたり 2 回の `childAssets` 追加で拠点全体が資産階層に載ります**。カメラ・BOX は接続階層で自動的に配下に入るため、台数分の編成作業が不要です。

> 案 α（単一 thin-edge が全拠点の BOX を収容）を採ると、**BOX 全台を個別に `childAssets` へ追加する工程**と、GW 本体の可視性の手当てが必要になります → **付録 A**。

### 1.3 ⚠️ 資産階層と接続階層は別物

**本書で最も間違えやすい箇所です。**

```
資産階層（childAssets・業務上の所属）        接続階層（childDevices・通信上の親子）
─────────────────────────────────         ──────────────────────────────────
拠点A (group)                              画像解析装置 (main device)
 ├── 画像解析装置  ←─ 本チームが明示追加     └── カメラ ×N  ←─ thin-edge が自動生成
 └── BOXゲートウェイ ←─ 本チームが明示追加
                                            BOXゲートウェイ (main device)
Inventory ロールの継承経路                    └── BOX ×M    ←─ thin-edge が自動生成
```

| 事実 | 確度 |
|---|---|
| 1 つの MO を「main device の `childDevices`」かつ「グループの `childAssets`」に**両方所属させられる** | [確] |
| **thin-edge の child device 登録は `childAssets`（グループ所属）を作らない** | [確] |
| したがって **main device を拠点グループへ入れる工程は、本チームのプロビジョニングツールが明示的に行う必要がある** | [判] |

**[判] 規約**: 拠点グループへの編成は **main device に対してのみ**行い、child device は接続階層に任せます。理由は 3 つです。

1. 台数分の編成作業が不要になる（案 β なら拠点あたり 2 回で済む）
2. 編成漏れの検査対象が main device だけになる（機械検査が単純になる・§2.8）
3. child を個別に `childAssets` へ入れると、装置交換時に**接続階層と資産階層が食い違ったまま残る**

> ⚠️ **この規約は前提 2（`alarmsWithChildren` が `childAssets` を辿る）に依存します。** 辿らない場合、child のアラームが拠点グループ購読に乗りません → §1.6。

### 1.4 グループの命名と external ID

| 項目 | 規約 | 理由 |
|---|---|---|
| **グループ表示名** | 日本語可（例: 「〇〇工場 東棟」） | 運用者が読む |
| **グループの同定キー** 〈rev.2 改訂〉 | **`x_Site.siteId` フラグメント**（例: `site001`）。**external ID は振らない**（下記） | 冪等投入の土台。名前だけで参照すると再実行で重複グループが生える |
| **拠点コード** | **`^[a-zA-Z0-9]+$` に適合すること** | Notification 2.0 の `subscription` 名がこのパターン [確]。**ハイフン・アンダースコア・日本語は不可** |

> ⚠️⚠️ **`site-001` はハイフンを含むため、Notification 2.0 の `subscription` 名にはそのまま使えません** [確]（`pattern: '^[a-zA-Z0-9]+$'`）。次のいずれかを §2.5 で確定してください。
>
> - **(a) 拠点コードを最初から英数字のみにする**（例: `site001`）。1 つの値で通せる → **[判] 推奨**
> - (b) 拠点コード（`site-001`）とトピック名用コード（`site001`）を別に持ち、対応表を維持する
>
> **(b) は対応表が second source of truth になり、拠点追加のたびに同期漏れのリスクを負います。(a) を推奨します。**

> **[確] この制約が効くのは `subscription` 名の 1 用途だけです。** rev.1 は「external ID・グループ名・トピック名を 1 つの値で通す」ために英数字制約を全体へ広げていましたが、**仕様上そうしなければならない理由はありません**:
>
> - `subscription` は購読の名前であって MO の external ID ではない（*"Several subscriptions can share the same name"*）
> - サブスクリプションが MO を指すのは `source.id` = **内部 ID** であり、external ID ではない（例: `id: '251982'`）
> - サブスクリプションの一意キーは (`source`, `context`, `subscription`) の三つ組
> - なお **`^[a-zA-Z0-9]+$` は Core OpenAPI 全体で唯一の `pattern:` 宣言**で、他の識別子に文字種制約はありません
>
> **したがって (a) は「そうしないと動かない」ではなく「1 値で通すと運用が単純になる」という [判] です。** 実務上は (a) を推奨しますが、**拠点コードにハイフンを使いたい業務上の理由があるなら (b) も成立します**（対応表の維持コストとの比較）。

**[判] 本書は以降 `site001` 形式（英数字のみ）を採用した表記を併記しますが、実値の確定は §2.5 に委ねます。**

#### ⭐ グループの external ID は不要です 〈rev.2〉

**[判] 拠点グループに external ID を振るのをやめ、`x_Site.siteId` フラグメントを同定キーにします。**

| 観点 | 結論 |
|---|---|
| **Cumulocity 上の必要性** | **無い。** managed object は内部 ID だけで成立し、`managedObject` スキーマに必須項目は 1 つもありません。external ID は「デバイスが自分の MO を見つける」ための追加機構です |
| **冪等投入の土台になるか** | **external ID でなくても同じことができます。** `?query=$filter=(has(c8y_IsDeviceGroup) and x_Site.siteId eq 'site001')` で一意に引けます（§1.5 で `x_Site` は既に必須）。**API 呼び出し回数も「クエリ → 分岐」で変わりません** |
| **Cumulocity 自身はどうしているか** | **名前ベースで冪等生成しています。** bulk device registration の `PATH` 列で自動生成されるグループに **external ID は付きません** |
| **go-c8y-cli はどうしているか** | 内部のグループ解決も `has(c8y_IsDeviceGroup) and name eq '<name>'` のクエリです |

**やめる利点**: ①同じ実体に「external ID」と「`x_Site.siteId`」という 2 つの同定キーが並ぶ状態（second source of truth）を解消できます。両方あると、片方だけ直した投入スクリプトが静かに別グループを作ります。②**投入が 1 ステップで済み、孤児 MO が発生しません**（§13.5）。

> ⚠️ **external ID を振っても「重複が防げる」わけではありません。** Edge（Release 2026 API）には**原子的な「external ID 付き MO 作成」がありません**。`POST /inventory/managedObjects`（重複チェック無し）→ `POST /identity/globalIds/{id}/externalIds`（409 あり）の **2 ステップ**になるため、**①が成功して②が 409 で落ちると external ID を持たない孤児 MO が残ります**（§13.5）。**「external ID があるから冪等」は成立しません。**

> ⚠️ **グループ名は一意ではありません。** 「名前だけで参照すると再実行で重複グループが生える」という rev.1 の懸念自体は正しく、**同名グループは作れてしまいます**（`POST /inventory/managedObjects` は 409 を返しません → §13.5）。**したがって同定キーは必要で、それを `x_Site.siteId` にする**という変更です。「同定キーを無くす」ではありません。

> ⚠️ **デバイスの external ID は話が別です。やめられません** → §2.2。

### 1.5 グループに持たせるフラグメント

| フラグメント | 対象 | 内容 | 用途 |
|---|---|---|---|
| `c8y_IsDeviceGroup` | 全グループ | Cumulocity 標準 | グループとして認識される |
| **`type`** | 全グループ | root = **`c8y_DeviceGroup`** / サブグループ = **`c8y_DeviceSubGroup`** | ⭐ **rev.2 で追加。** go-c8y-cli の `--excludeRootGroup` が `not(type eq 'c8y_DeviceGroup')` を使うなど、**ツール側がこの値を前提にしています**。省略すると絞り込みが効きません |
| `x_Site` | 拠点グループ | `{ "siteId": "site001", "siteName": "〇〇工場 東棟" }` | ⭐ **拠点グループの同定キー**（§1.4）。冪等投入・拠点コードの正引き。**デバイス側の `x_Site` と同じキーを使う**（§6.4） |

> **[判] `x_Site.siteId` は拠点グループの同定キーです。必須項目として扱ってください。** rev.1 では「あると便利なフラグメント」でしたが、rev.2 で **external ID を廃止した結果、これが唯一の同定キー**になりました（§1.4）。**未設定のグループを作らないこと。** §13.7 のアサートに「全拠点グループに `x_Site.siteId` があり、値が重複していないこと」を追加してください。

### 1.6 ⚠️ 拠点分離の成立条件と、不成立時の代替

**拠点分離は「グループを作れば成立する」ものではありません。3 つの条件が同時に必要です。**

| # | 条件 | 状態 | 不成立時に壊れるもの |
|---|---|---|---|
| **1** | Inventory ロールがグループ配下へ継承される | **[確]** グループ内デバイスまでは成立。⚠️ **`childDevices`（カメラ・BOX）まで届くかは [要 SV-47]** | 拠点 Manager がカメラ個別のアラーム・インベントリを見られない |
| **2** | `alarmsWithChildren` / `eventsWithChildren` が **`childAssets` を辿る** | **[要] 未確認（SV-35 / TE-12）**。公式は *"all of its descendant managed objects"* としか述べず、**関係の種別を特定していない**（§10.2） | **拠点グループ単位の通知購読が全滅** |
| **3** | `ROLE_NOTIFICATION_2_ADMIN` が拠点分離をバイパスしない運用が確立している | **[確] バイパスする**。§10.4 のトークン発行プロキシで封じる | 案件アプリが全拠点のアラームを購読できてしまう |

#### 条件 2 が不成立だった場合の代替案

| 案 | 内容 | 評価 |
|---|---|---|
| **(i) 拠点ごとに main device を `source` にする** | `context: mo` + `source` = 各 main device + `alarmsWithChildren`（`childDevices` は辿る前提） | **拠点あたり 2 サブスクリプション**（画像解析装置・BOXゲートウェイ）になる。拠点数 × 2 の管理。**案 β なら成立する** |
| (ii) `context: tenant` + `typeFilter` | テナント全体を購読し、コンシューマ側で拠点を振り分ける | ⚠️ **拠点分離が Cumulocity 側で成立しない**。案件アプリが全拠点のアラームを受け取る。D17 の分離要件に反する |
| (iii) 拠点ごとに EPL でロールアップアラームを生成し、それを購読する | 部品が増える | 最終手段 |

> **[判] 条件 2 が NG なら (i) を採ってください。** 案 β を採用していれば main device が拠点と 1:1 に対応するため、(i) は追加設計なしで成立します。**これは案 β を推奨するもう 1 つの理由です。**

### 1.7 グループ設計の規約

| # | 規約 | 理由 |
|---|---|---|
| **G-a** 〈rev.2 改訂〉 | **すべての拠点グループに `x_Site.siteId` を振り、これを同定キーにする**（external ID は振らない・§1.4） | グループ名は一意ではなく、`POST /inventory/managedObjects` は 409 を返さないため、**同定キーが無いと再実行で重複グループが黙って生える**（§13.3.3） |
| **G-b** | **拠点コードは英数字のみ**（§1.4） | Notification 2.0 の `subscription` 名がパターン `^[a-zA-Z0-9]+$` [確] |
| **G-c** | **Inventory ロールは拠点グループに割り当てる。デバイス個別に割り当てない** | 継承される。**個別割当は SSO の `inventoryMappings` に次回ログインで上書きされる** [確] |
| **G-d** | **段階ロールアウトの単位も拠点グループ** | bulk operation の `groupId` がグループ指定を前提とする [確] |
| **G-e** | **グループ階層の深さは 2 段（root → 拠点）まで** | ⚠️ **[判]** 深くすると Inventory ロールの継承経路と `alarmsWithChildren` の探索範囲が読みにくくなる。拠点内の区分が必要なら**フラグメント（`x_Site` の拡張）で表現**し、グループを増やさない |
| **G-f** | **グループの作成・編成はプロビジョニングツールが行う。UI からの手作業を禁止** | D10。手作業のグループは external ID を持たず、再実行で重複する |

---

## 2. デバイス識別・命名規約

### 2.1 なぜ独立した章にするか

**識別子と型名は、本書のほぼ全章の結節点です。** 後から変えると次がすべて壊れます。

```
external ID ──┬─► BOX 再送の重複排除キー（§8.5）
              ├─► 映像クリップとの突合キー（オブジェクトストレージ・§7.5）
              ├─► ベクトルDB の特徴量紐づけキー（クラウドへ出る・§2.2 の法務論点）
              ├─► プロビジョニングの冪等キー（§3.5）
              └─► デバイス交換時の同一性の担保（§3.9）

device type ──┬─► リテンションの type フィルタ（§7.2）
              ├─► スマートルールの対象条件（§8.2）
              ├─► Notification 2.0 の typeFilter（§10.2）
              └─► デバイスプロファイルの適用対象（§4.5）

アラーム型 ───┬─► alarm.type.mapping のキー（§9）
              ├─► リテンションの type フィルタ（§7.2）
              ├─► EPL のマッチング（§8）
              └─► Notification 2.0 の typeFilter（§10.2）
```

> **[判] この章の内容は、Cumulocity への設定投入より前に「規約文書 + バリデータ」として確定させてください。** [担当範囲] C-1〜C-5 に対応します。**本チームの遅れがそのまま画像解析側・BOXアダプタ側・案件アプリ側の遅れになります。**

### 2.2 external ID 採番規約

**[判] 形式**: `{siteId}:{機器種別}-{シリアル}`

> ⚠️⚠️ **「external ID を採番しない」という選択肢はデバイスには存在しません** [確]。thin-edge / Cumulocity が**必ず自動生成**します。したがって本節が決めているのは「external ID を持つかどうか」ではなく **「自動生成に任せるか、意味のある値を与えるか」** です。
>
> | 対象 | 何もしないとどうなるか |
> |---|---|
> | **main device** | **証明書の CN がそのまま external ID になります**（`type: c8y_Serial`）。`tedge cert create --device-id <X>` の `<X>` = 証明書 CN = MQTT ClientId = external ID = MO の `name` が**すべて同一値に固定**されます |
> | **child device** | thin-edge が **`<main-device-id>:device:<topic-id>`** を自動生成します（`/` は `:` に変換）。`@id` を明示すればその値が verbatim で使われます |
>
> **グループは別です。グループに external ID は不要で、rev.2 で廃止しました** → §1.4。

| 対象 | external ID | 例 |
|---|---|---|
| 拠点グループ | ⚠️ **振らない**（`x_Site.siteId` で同定・§1.4） | — |
| 画像解析装置（main device） | **`{siteId}-ANLZ-{シリアル}`** | `site001-ANLZ-SN00123` |
| IP カメラ（child device） | `{siteId}:CAM-{シリアル}` | `site001:CAM-SN12345` |
| BOXゲートウェイ（main device） | **`{siteId}-GW-BOX`** | `site001-GW-BOX` |
| 画像センシングBOX（child device） | `{siteId}:BOX-{シリアル}` | `site001:BOX-SN98765` |

> ⚠️⚠️ **rev.2 で main device の区切りをコロンからハイフンに変更しました。コロンは使えません** [確]:
>
> > *"the following format should be used for the ClientId: `connectionType:deviceIdentifier:defaultTemplateIdentifier`"*
> > *"**Important** — The colon character has a special meaning in Cumulocity. Hence, it must not be used in the `deviceIdentifier`."*
> > *"Set CLIENT_ID to the common name of the device certificate. **The certificate common name must not contain `:`**."*
> > — [MQTT device integration](https://cumulocity.com/docs/device-integration/mqtt/)
>
> main device では次の連鎖が確定しており、**external ID を決める手段は証明書 CN しかありません**:
>
> ```
> tedge cert create --device-id <X>   →   証明書 CN = <X>
>   → device.id = <X>（tedge.toml より証明書が優先）
>   → MQTT ClientId = <X>（CN と一致必須）
>   → Cumulocity external ID = <X>（type: c8y_Serial・接続時に自動生成）
>   → MO の name = <X>（thin-edge が 100,<device.id>,<device.type> を送る）
>   → EST の CSR も CN = <X>（*"CSR must be a valid PKCS#10 with deviceID as Common Name (CN)"*）
> ```
>
> **`site001:ANLZ-SN00123` のままでは、`connectionType=site001` / `deviceIdentifier=ANLZ-SN00123` と誤解釈されうるうえ、証明書 CN として不正です。**
>
> **child device は `:` のままで問題ありません。** 証明書も MQTT 接続も持たず、SmartREST 101 経由で作られるためです（thin-edge の自動生成値 `<main>:device:<child>` 自体がコロンを含みます）。

**規約**:

| # | 規約 | 理由 |
|---|---|---|
| **X-a** | **`{siteId}` プレフィックスを必ず付ける** | 全拠点が単一テナントに集約されるため（D17）、シリアルだけでは衝突し得る。**thin-edge 自身が同じ理由で child に main の ID を prefix しています**。また案 α では 1 thin-edge インスタンス内での一意性も必要 |
| **X-b** | **機器種別は `ANLZ` / `CAM` / `GW` / `BOX` の 4 語に固定** | 型からの逆引きと機械検査を単純にする |
| **X-c** 〈rev.2 改訂〉 | **main device は区切りをすべて `-` にする。child device は `:`（拠点）と `-`（種別とシリアル）** | main はコロン禁止（上記）。child は thin-edge の自動生成形式に合わせる |
| **X-d** 〈rev.2 改訂〉 | **シリアルは機器の物理シリアルを使う。連番を振らない** | 資産管理台帳と突合できる。⚠️ **rev.1 の理由「交換時に同一性を追跡できる」は §3.9 の判断（交換時は新 external ID を振る）と矛盾していたため差し替えました** |
| **X-e** | ⚠️ **一度登録した external ID は変更できない** [確] | thin-edge の `@id` は登録後変更不可（main device は *"Updating the external id of {0} is not supported"* で明示的に拒否）。**間違えたら削除して作り直すしかない**（§2.6 E-e）。⚠️ なお **Cumulocity 側は DELETE + POST で付け替え自体は可能**です（Identity API に PUT/PATCH が無いだけ）。制約は thin-edge 側 |
| **X-f** 〈rev.2 で追加〉 | ⚠️⚠️ **main device の external ID に `:` を使わない** | 証明書 CN・MQTT ClientId と同一値になるため。**§2.8 CK-1 で機械検査する** |

> ⚠️⚠️ **この external ID は 3 つ目の役割でクラウドへ出ます。** ベクトルDB に格納された特徴量は AI モデル改善サービス（クラウド）へ送信されます。**シリアル番号や拠点コードが顧客の業務 ID を含む場合、仮名化の要否が個人情報保護上の論点になります。**
>
> **[要] 仮名化の要否は法務／顧客合意で確定してください。** 仮名化する場合、①Cumulocity 上の external ID を仮名にするか ②クラウドへ出す時点で変換するか、の 2 案があり、①なら本節の形式そのものが変わります。**§13 の設定投入より前に決着が必要です。**

### 2.3 device type 規約

**[判] `type` フィールドの値を 4 つに固定します。**

| MO 種別 | `type` | 表す実体 |
|---|---|---|
| 画像解析装置 | **`x_ImageAnalyzer`** | thin-edge A の main device |
| IP カメラ | **`x_Camera`** | child device（証明書なし） |
| BOXゲートウェイ | **`x_BoxGateway`** | thin-edge B の main device |
| 画像センシングBOX | **`x_SensingBox`** | child device（証明書なし） |

| # | 規約 | 理由 |
|---|---|---|
| **T-a** | **`x_` プレフィックス + PascalCase** | Cumulocity 標準の `c8y_` と衝突させない |
| **T-b** | **type は「機器の種類」だけを表す。拠点・世代・案件を含めない** | type はリテンション・ルール・購読フィルタのキー。ここに可変要素を入れると、拠点追加のたびにルールが増える |
| **T-c** | **拠点は `x_Site` フラグメント、世代は `x_SensingBox.fwVersion` 等の個別フラグメントで表す** | 同上 |

### 2.4 カスタムフラグメント命名規約

**[判] すべて `x_` プレフィックス + PascalCase に統一します。**

| フラグメント | 対象 MO | 内容 | 出典 |
|---|---|---|---|
| `x_Site` | 拠点グループ / main device | `{ siteId, siteName }` | **本書で新規定義** |
| `x_Camera` | IP カメラ | `{ vmsCameraId, location }` | [連携仕様] §4.3 |
| `x_SensingBox` | 画像センシングBOX | `{ model, fwVersion, channels }` | **本書で新規定義** |
| `x_Detection` | 検知イベント | `{ eventUuid, modelVersion, aiProduct, confidence, boundingBoxes, clipHint }` | [カメラ構成] §4.3 |
| `x_NoAutoConfig` | 任意の MO | 自動設定変更の除外タグ（値は `{}` でよい） | [設定書] G-10 |

> ⚠️ **`x_SensingBox` と `x_Site` は本書が新たに定義する項目です**（[連携仕様] には未収録）。**[連携仕様] の改訂として反映してください。**

> ⚠️ **`x_Camera.vmsCameraId` は必須です。** これは**イベント → 映像の紐づけを成立させる唯一の結合点**です（[カメラ構成] §4.1）。**未設定はプロビジョニングツールがエラーにしてください。**

> ⚠️ **`x_NoAutoConfig` は「運用知識基盤の自動設定変更からデバイスを除外するタグ」であり、他のフラグメントとは性質が異なります**（データではなく制御フラグ）。§4.10 で扱います。

### 2.5 拠点コードの確定

**[判] 拠点コードは英数字のみとし、1 つの値で 4 用途を通します。**

| 用途 | 使われ方 | 制約 |
|---|---|---|
| 拠点グループの external ID | `site001` | 特になし |
| external ID のプレフィックス | `site001:CAM-SN12345` | 特になし |
| Notification 2.0 の `subscription` 名 | `site001` | **`^[a-zA-Z0-9]+$`** [確] |
| thin-edge の topic id の一部 | `device/site001-cam-SN12345//` | **英数字・`-`・`_` に限定** [判] |

**採番方針の候補**（**[要] 案件チームと確定**）:

| 案 | 例 | 評価 |
|---|---|---|
| **連番** | `site001` `site002` … | **[判] 推奨**。衝突しない。桁数を先に決める（3 桁で 999 拠点） |
| 顧客略号 + 連番 | `acme001` | 2 顧客目で Edge を分ける方針（[設定書] D-c）なので不要 |
| 拠点名のローマ字 | `tokyoeast` | 表記ゆれが起きる。**非推奨** |

> ⚠️ **拠点コードは表示名ではありません。** 運用者が読むのはグループの表示名（日本語可）です。拠点コードは機械が使う識別子と割り切ってください。

### 2.6 thin-edge のエンティティ登録規約

**Cumulocity 側の external ID を規約どおりに保つには、thin-edge 側の登録方法を統制する必要があります。**

#### 問題

thin-edge は既定で**自動登録**が有効で、external ID を自分の規則 **`<main-device-id>:device:<child-id>`** で生成します [確]。これは本規約 `{siteId}:{機器種別}-{シリアル}` と**一致しません**。放置すると規約外の MO が生え、プロビジョニングツールが登録した MO と**二重に存在**します。

公式も明示しています [確]:

> *"For any complex deployments requiring external id customizations or with nested child devices, auto-registration **must be disabled**."*

#### **[判] 規約**

| # | 規約 | 具体 |
|---|---|---|
| **E-a** | **自動登録を必ず無効化する** | `c8y.entity_store.auto_register = false` および `agent.entity_store.auto_register = false`。**変更後は `tedge-agent` の再起動が必要** [確] |
| **E-b** | **すべてのエンティティを明示登録し、`@id` で external ID を直接指定する** | `@id` を指定すると **prefix なしでそのまま external ID になる** [確] |
| **E-c** | **登録は HTTP API（`POST /te/v1/entities`）を優先する** | MQTT は成功/失敗のフィードバックが無い [確] |
| **E-d** | **child の topic id には `/` `+` `#` を使わない** | topic id は MQTT トピックの一部になる。**英数字・`-`・`_` に限定** [判] |
| **E-e** | ⚠️ **登録後に `@type` と `@id` は変更できない** | *"Updates are limited to the `@parent` and `@health` properties only"* [確] |
| **E-f** | 削除は同トピックへの**空の retained メッセージ** | 子孫も一緒に解除される [確] |

**登録メッセージの例**（カメラ）:

```bash
curl -X POST http://127.0.0.1:8000/te/v1/entities -H 'Content-Type: application/json' -d '{
  "@topic-id": "device/site001-cam-SN12345//",
  "@type": "child-device",
  "@id": "site001:CAM-SN12345",
  "name": "東門カメラ",
  "type": "x_Camera",
  "x_Camera": { "vmsCameraId": "gsc-cam-8842", "location": "東門" },
  "x_Site": { "siteId": "site001" }
}'
```

> **`@type` / `@id` 以外の追加フィールドは twin データとして扱われ、Cumulocity の MO フラグメントになります** [確]。`x_Camera` / `x_Site` はこの仕組みで載ります。**`@` プレフィックスは thin-edge の予約**なので独自キーに使わないこと。

> ⚠️ **thin-edge 側の設定手順の正は [担当範囲] §5.5 です。** 本節は Cumulocity 側から見た「守られるべき結果」を定義しています。

### 2.7 文字種・長さの制約一覧

| 対象 | 制約 | 確度 |
|---|---|---|
| Notification 2.0 の `subscription` 名 | **`^[a-zA-Z0-9]+$`** / `minLength: 1` / **`maxLength` の定義なし**。⚠️ **Core OpenAPI 全体で唯一の `pattern:` 宣言** | [確] |
| thin-edge の topic id | 4 セグメント固定（`device/<id>/service/<id>`）で **`/` は区切りとして必ず含む**。**セグメント内**の `/` `+` `#` が不可。external ID 生成時に `/` は `:` に変換される | [確] |
| オペレーション定義ファイル名 | **英数字と `_` のみ**（ハイフン不可。使うと定義が無視される） | [確] |
| `c8y_RequiredAvailability.responseInterval` | `-32768`〜`32767`（範囲外は境界に丸められる）。**0 以下はメンテナンスモード**（§5.2） | [確] |
| **external ID（Identity API 側）** | **無制約。** `externalId` スキーマに `pattern` / `minLength` / `maxLength` の定義が**一切ありません** | **[確]** 〈rev.2 で確定〉 |
| **`device.id` = 証明書 CN = MQTT ClientId（main device）** | ⚠️⚠️ **`:` 不可**（*"The colon character … must not be used in the `deviceIdentifier`"* / *"The certificate common name must not contain `:`"*）。`c8y deviceregistration register-ca --id` は最大 1000 文字 | **[確]** 〈rev.2 で確定〉 |
| **フラグメント名** | **空白と `. , * [ ] ( ) @ $ / '` が不可**（*"Names used for fragments must not contain whitespaces nor the special characters …"*） | **[確]** 〈rev.2 で追加〉 |
| managed object 1 件の JSON サイズ | 上限 **16 MiB**、**1 MiB 未満を推奨** | [確] 〈rev.2 で追加〉 |
| 1 グループあたりのサブアセット数 | **1000 未満を推奨** | [確] 〈rev.2 で追加〉 |

> ⭐ **rev.2 で TE-6 は解消しました。** rev.1 は「external ID の文字種・長さは未確認」としていましたが、**Identity API 側は無制約であることが OpenAPI スキーマから確定**しました。**実際に制約が効くのは main device の経路だけ**（証明書 CN / MQTT ClientId のコロン禁止）で、これも公式に逐語記述があります。**実機確認は不要です。**

> ⚠️ **MQTT 3.1 を使う場合は ClientId の長さにも注意してください。** 24 文字を超えるとエラーになる実装があります（MQTT 3.1.1 以降なら制約なし）。

### 2.8 規約準拠の機械検査

**[判] 規約は「文書にする」だけでは守られません。CI で機械検査してください。**

| # | 検査 | 実装 | 不合格時 |
|---|---|---|---|
| **CK-1a** | **main device** の external ID が **`^[a-z0-9]+-(ANLZ\|GW)-.+$`** に適合（**コロンを含まない**こと・X-f） | ⚠️ **`GET /identity/externalIds` は存在しません**（下記）。`GET /inventory/managedObjects?fragmentType=c8y_IsDevice` で MO を列挙 → 各 MO に `GET /identity/globalIds/{id}/externalIds` | CI 失敗 |
| **CK-1b** | **child device** の external ID が **`^[a-z0-9]+:(CAM\|BOX)-.+$`** に適合 | 同上 | CI 失敗 |
| **CK-1c** 〈rev.2〉 | **全拠点グループに `x_Site.siteId` があり、値が重複していない**（external ID を廃止したため、これが同定キー・§1.4） | `GET /inventory/managedObjects?fragmentType=c8y_IsDeviceGroup` | CI 失敗 |
| **CK-1d** 〈rev.2〉 | ⭐ **孤児 MO の検出** — `c8y_IsDevice` を持つ MO で **external ID が 1 つも紐づいていないものが無い**こと | 同上（CK-1a/b と同じ走査で判定できる） | CI 失敗。**REST 経由の 2 ステップ登録が途中で失敗した痕跡**（§13.5） |
| **CK-2** | 全デバイスの `type` が 4 種のいずれか | `GET /inventory/managedObjects?fragmentType=c8y_IsDevice` | CI 失敗 |
| **CK-3** | 全 `x_Camera` に `vmsCameraId` が設定されている | 同上 | CI 失敗 |
| **CK-4** | 全 main device が**いずれかの拠点グループの `childAssets`** に入っている | グループを走査して差集合を取る | CI 失敗 |
| **CK-5** | external ID のプレフィックス `{siteId}` と、実際に所属する拠点グループが一致 | 同上 | ⚠️ **BOXアダプタの誤配送を検出できる唯一の手段**（§6.2） |
| **CK-6** | `ROLE_NOTIFICATION_2_ADMIN` が §11.4 の指定ユーザー以外に付いていない | `GET /user/{t}/groups` + `/users` | CI 失敗 |
| **CK-8** 〈rev.3〉 ⚠️ **CU-18 で案 (D) を採る場合のみ有効** | ⭐ **スナップショットのバイナリ MO の隔離検査**（§15.3 案 (D) の成立条件を維持する）<br>**(a)** **`type` が画像用の値**（§13.3.9 の名前空間規約）のバイナリ MO が、**どの拠点グループの `childAssets` にも入っていない**こと<br>**(b)** ⚠️ **グローバルの `ROLE_BINARY_READ` / `ROLE_BINARY_ADMIN` / `ROLE_INVENTORY_READ` / `ROLE_INVENTORY_ADMIN` を持つ主体が、R-14 と（実体確認後の）R-01 だけであること** — 検査対象は **R-01 / R-02 / R-03 / R-05 / R-06a / R-06b / R-08 / R-10 / R-11 / R-12**〈rev.3 レビューで対象拡大〉 | **(a)** 拠点グループを走査して `childAssets` に画像バイナリ MO が無いことを確認（CK-4 の逆向き）<br>**(b)** `GET /user/{t}/groups` + `GET /user/roles` でロールの実体を展開して照合 | CI 失敗。⚠️ **どちらか一方でも崩れると「イベントは見えるが画像は 403」が静かに破れます** |

> ⚠️⚠️ **`GET /identity/externalIds`（コレクション取得）は存在しません** [確]。Identity API のパスは 4 本だけです — `/identity`、`/identity/search`、`/identity/globalIds/{id}/externalIds`、`/identity/externalIds/{type}/{externalId}`。**rev.1 の CK-1 は実装不能でした。** さらに inventory クエリの `has()` は `externalIds` を明示的に除外しているため、**「全 MO 列挙 → 各 MO に N+1 で問い合わせ」以外の実装方法がありません。** MO 数が多いので、CI での実行時間を見積もってください。

> ⚠️ **CK-6 は §11.7 AX-1 と同じ検査です。** 併せて **AS-4（`ROLE_REMOTE_ACCESS_ADMIN` が誰にも付いていないこと）**・**AS-13（`ROLE_AUDIT_ADMIN` が人間ユーザーに付いていないこと・§11.2）**・**CK-7（アラーム型が他のマッピングキーの前方一致に該当しないこと・§9.2 AM-a）** も CI に載せてください。

> ⚠️ **CK-8(a) が実際に守る相手は「admin 級のロールを持つ主体」です** 〈rev.3 で追加〉。
> `POST /inventory/managedObjects/{id}/childAssets` の Required roles は *"`ROLE_INVENTORY_ADMIN` **OR** `ROLE_MANAGED_OBJECT_ADMIN` **OR** ((owner of the source **OR** `MANAGE_OBJECT_ADMIN` permission on the source) **AND** (owner of the child **OR** `MANAGE_OBJECT_ADMIN` permission on the child))"* [確]。
>
> **child（バイナリ MO）側の条件を拠点 Manager は満たせない**ため、**「拠点 Manager が自力で画像を自グループにぶら下げる」経路は塞がっています。** CK-8(a) が想定すべき相手は **R-01 / R-06b / R-08 / R-10** です。**この AND 条件を知らないと、CK-8(a) の重みと (b) の深刻さを取り違えます。**

> ⚠️ **CK-1〜CK-3 は「壊れても長期間気づけない」種類の破綻です。** 手で 1 台登録すると、その拠点だけ external ID の規約が崩れ、重複排除・映像突合・特徴量紐づけの三重の役割が**静かに**壊れます。**検査を CI に載せることが唯一の歯止めです。**

---

## 3. デバイス管理方式

### 3.1 デバイス階層モデル

**thin-edge.io は「1 インスタンス = Cumulocity 上の 1 台の main device + その配下の child device 群」というモデルです**（D16）。したがって **thin-edge のインスタンス境界が、そのまま Cumulocity 上のデバイス階層の境界**になります。

| 実体 | Cumulocity 上の表現 | 証明書 | MQTT 接続 | `com_cumulocity_model_Agent` |
|---|---|---|---|---|
| 画像解析装置 | main device（`x_ImageAnalyzer`） | **1 枚** | **1 本** | **あり** |
| IP カメラ | child device（`x_Camera`） | なし | なし（代理） | **なし** |
| BOXゲートウェイ | main device（`x_BoxGateway`） | **1 枚** | **1 本** | **あり** |
| 画像センシングBOX | child device（`x_SensingBox`） | なし | なし（代理） | **なし** |

> ⚠️⚠️ **`com_cumulocity_model_Agent` を child device に付けないでください。** 付けるとオペレーションの宛先候補になり、実行できないオペレーションが PENDING のまま滞留します（§4.11）。

**MO 種別ごとのフラグメント定義**:

| MO 種別 | `com_cumulocity_model_Agent` | `c8y_SupportedOperations` | `c8y_RequiredAvailability` | 必須カスタムフラグメント |
|---|---|---|---|---|
| 画像解析装置 | **あり** | フルセット（§4.2） | **あり**（§5.5） | `x_Site{siteId}` |
| IP カメラ（child） | なし | **なし** | **`responseInterval: 32767`**（§5.4）。⚠️ **`0` は不可** | `x_Camera{vmsCameraId, location}` ← **`vmsCameraId` 必須** |
| BOXゲートウェイ | **あり** | **縮退**（§4.2） | **あり** | `x_Site{siteId}` |
| 画像センシングBOX（child） | なし | **なし** | **`responseInterval: 32767`**。⚠️ **`0` は不可** | `x_SensingBox{model, fwVersion, channels}` |

### 3.2 デバイス種別ごとの登録方式

| 種別 | 登録主体 | 認証 | 登録タイミング |
|---|---|---|---|
| 画像解析装置 | thin-edge（EST エンロール）+ プロビジョニングツール（グループ編成・フラグメント設定） | **X.509 デバイス証明書** | 拠点プロビジョニング時 |
| BOXゲートウェイ | 同上 | **X.509 デバイス証明書** | 同上 |
| IP カメラ | **プロビジョニングツールが明示登録**（thin-edge の HTTP API 経由） | なし（main device の接続に相乗り） | 同上 |
| 画像センシングBOX | 同上 | なし | 同上 |

> ⚠️ **child device に証明書は不要です** [確]。main device の MQTT 接続を通じて代理でデータが上がります。**これは「BOX やカメラを個別に認証しない」ことを意味します。** 装置内での成りすまし防止は、ローカル MQTT の到達範囲の統制（[担当範囲] §5.10）に依存します。

### 3.3 オンボーディング方式の選択

**[判] 方式 (1)「テナント CA + 証明書エンロール」を採用します。**

| 方式 | 内容 | 採否 |
|---|---|---|
| **(1) テナント CA + EST 登録** | `POST /certificate-authority` でテナント CA 作成 → デバイスが `/.well-known/est/simpleenroll` で証明書を取得（tenant + device serial + **one-time password** を BasicAuth） | **✅ 採用**。thin-edge.io 公式も *"Recommended"*。**証明書の自動更新（§3.6）が同じ仕組みに乗る** |
| (2) 自己署名 CA + 自動登録 | CA を `POST /tenant/tenants/{t}/trusted-certificates` に `autoRegistrationEnabled: true` でアップロード | **次善**。前提 3 が NG の場合のフォールバック。⚠️ **一括オンボーディング完了後は必ず `autoRegistrationEnabled` を無効化**（有効なままだと承認なしにデバイスが自動登録される） |
| (3) 個別証明書アップロード | `tedge cert create` → `tedge cert upload c8y --user <user>` | 開発・検証用のみ |

> ⚠️⚠️ **前提 3（Cumulocity の certificate-authority が Edge で使えるか）は未確認です**（SV-08 / TE-8。**同機能のドキュメントに Edge への言及がない**）。**W1 の最優先検証項目**です。NG の場合は方式 (2) に落ちますが、**その場合の証明書更新の自動化方式を別途設計する必要があります**（§3.6 の自動更新は EST `simplereenroll` に依存しているため）。

#### 一括登録 CSV — グループ階層も同時に作れる [確]

`POST /devicecontrol/bulkNewDeviceRequests`（`ROLE_DEVICE_CONTROL_ADMIN`）は CSV で次を同時に投入できます。

1. **証明書エンロール**: `AUTH_TYPE=CERTIFICATES` + `ENROLLMENT_OTP`
2. **device group 階層の自動生成**: スラッシュ区切りの `PATH`。**存在しないグループは作成される**

**制約** [確]: ①`PATH` 使用時は `TYPE` と `NAME` 列も必要 ②`com_cumulocity_model_Agent.active` ヘッダ（値 `true`）が必要 ③**テナント CA の存在が前提** ④**`ENROLLMENT_OTP` と `PATH` は併用可**（CSV ヘッダ表に**両方が独立した任意列**として並記されている。排他は `CREDENTIALS` × `AUTH_TYPE` のみ）→ **SV-07 は [推] へ格下げ** ⑤UI の**一括**登録ウィザードは `ENROLLMENT_OTP` 非対応（⚠️ ただし**単体**登録 UI には OTP 欄があり、`?externalId=…&one-time-password=…` のクエリで事前入力もできる）。

> ⚠️ **`PATH` で自動生成されるグループに external ID は付きません。** Cumulocity 自身がグループを**名前ベースで冪等生成**しています。これは §1.4 で「グループに external ID は不要」と判断した根拠の 1 つです。

> **[判] 一括登録 CSV は「main device の初期投入」にのみ使い、child device（カメラ・BOX）には使いません。** child は thin-edge の entity API で `@id` を指定して登録する必要があるため（§2.6 E-b）、CSV では規約を満たせません。

### 3.4 登録主体と必要ロール

| # | 規約 | 理由 |
|---|---|---|
| **P-a** | **登録はすべてプロビジョニングツール（構成コード）が行う。UI からの手作業登録を禁止** | D10。手で 1 台登録すると、その拠点だけ external ID の規約が崩れる（§2.8） |
| **P-b** | ⚠️ **証明書アップロード／CA 操作用に、SSO 対象外のローカルユーザーを 1 つ用意する** | **SSO ユーザーでは `tedge cert upload` が実行できない** [確]。§11.4 の R-08 |
| **P-c** | 必要ロールは **`ROLE_TENANT_ADMIN` または `ROLE_TENANT_MANAGEMENT_ADMIN`** | [確] |
| **P-d** | 一括登録には **`ROLE_DEVICE_CONTROL_ADMIN`** | [確] |

> ⚠️ **R-08（証明書アップロード用ローカル管理者）のライフサイクル管理を運用手順に入れてください。** パスワードローテーション・保管方法・ロックアウト時のブレークグラス手順が必要です（§11.5・[担当範囲] RB-13）。**メール不採用のため、このアカウントのパスワードを失うと管理者でもリセットできません**（§12.8）。

### 3.5 child device の代理登録

**プロビジョニングツールが実行する手順**（拠点あたり 1 回）:

| # | 手順 | 完了条件 | 冪等 |
|---|---|---|---|
| 1 | `siteId` 採番・構成コードへの拠点定義追加 | 拠点定義が存在する | ○ |
| 2 | main device のデバイス証明書取得（§3.6） | main device が Cumulocity に現れる | ○ |
| 3 | **main device を拠点グループの `childAssets` へ追加** | 拠点グループ配下に表示される | ○（409 黙殺・[確]） |
| 4 | **child device の明示登録**（`@id` 指定・§2.6） | 全 child が規約どおりの external ID で登録済み | ○ |
| 5 | **`x_Camera.vmsCameraId` の設定** | **未設定はツールがエラー** | ○ |
| 6 | child の `c8y_RequiredAvailability = {"responseInterval": 32767}` 設定 | 実質無効化（§5.4）。⚠️ **`0` にしない** | ○ |
| 7 | `x_Site` / `x_SensingBox` フラグメントの設定 | §2.4 のとおり | ○ |
| 8 | supported operations の確認 | **child には何も出ていない**（§4.3） | — |

> ⚠️⚠️ **手順 2〜4 で MO を作る箇所は、必ず「作成前に存在確認」してください。** `POST /inventory/managedObjects` には重複チェックがなく、`POST /identity/globalIds/{id}/externalIds` だけが 409 を返すため、**順に叩くと external ID を持たない孤児 MO が残ります**（§13.5）。なお **main device は thin-edge の接続時に Cumulocity 側が find-or-create するため、プロビジョニングツールが MO を作る必要はありません**（SmartREST 100）。

> ⚠️⚠️ **手順 6 は「手順 4 の直後・thin-edge の初回接続より前」に実行してください。** SmartREST 117 は既存値を上書きしませんが、**先に入っていた値が勝つ**ため、thin-edge が先に 60 分を書いてしまうと以後どれだけ設定しても変わりません（§5.5）。**プロビジョニングツールの実行順序が結果を決めます。**

> ⚠️ **手順 6 に `0`（メンテナンスモード）を使ってはいけません。** そのデバイスを `source` とする全アラームが抑止され、`x_CameraDown` / `x_BoxSilent` が 1 件も上がらなくなります（§5.2）。

> **冪等化パターン**: 手順 3 は **409 黙殺**（[投入ガイド] パターン E）、手順 4〜7 は **存在チェック → 分岐**（パターン C）→ §13.5
>
> ⭐ **[確] 手順 3 の 409 黙殺は妥当です**（SV-38 解消）。OpenAPI の応答定義には 409 がありませんが、go-c8y-cli の公式例が明示しています — *"The `silentStatusCodes` parameter is used to **silence 409 (duplicate) errors, as we don't care if the device is already assigned to the group**."*
>
> ⚠️ **ただしグループ作成（G-01）に 409 黙殺は使えません**（§13.3.3）。**`childAssets` への追加は 409 を返すが、MO の作成は返さない** — この非対称に注意してください。

### 3.6 デバイス証明書のライフサイクル

**登録フロー** [確]:

```bash
# (1) テナント側の事前準備（1 回きり・§13.3.5）
c8y features enable --key certificate-authority
c8y devicemanagement certificate-authority create

# (2) オペレータ端末: デバイス ID とワンタイムパスワードを登録
c8y deviceregistration register-ca --id "$DEVICE_ID" --one-time-password "$OTP"

# (3) デバイス上: CSR 生成 → EST simpleenroll → 証明書取得
sudo tedge config set c8y.url "$C8Y_URL"
sudo tedge cert download c8y --device-id "$DEVICE_ID" --one-time-password "$OTP"
sudo tedge connect c8y
```

> **上記 3 コマンドの実在は確認済み** [確]（go-c8y-cli の既定ブランチは `master` ではなく **`v2`**。`master` を見ると「存在しない」と誤判定するので注意）。ただし `c8y devicemanagement certificate-authority create` は docs 上 **「(PREVIEW FEATURE)」** 表記です。
>
> ⚠️ **`certificate-authority` フィーチャーは既定で有効**（*"By default, it is enabled"*）なので、手順 (1) の `c8y features enable` は冪等な確認として実行する位置づけになります。
>
> ⚠️⚠️ **[要 SV-39] Public Preview → GA の移行に注意。** 公式に *"**Migration from Public Preview to General Availability - action required.** … you need to remove all the devices you have registered under Public Preview and re-register them."* との告知があります。**PoC を Preview 期間中に実施した場合、GA 移行時に全デバイスの再登録が必要になりえます。** 現在の提供ステータスを確認し、PoC と本番の間に移行が挟まらない計画にしてください。

**自動更新** [確]: `tedge-mapper` パッケージが `tedge-cert-renewer@c8y.timer` / `.service` を提供し、**毎時**（`OnCalendar=hourly` + `RandomizedDelaySec=5m`）判定します。判定基準は `certificate.validity.minimum_duration`（**既定 30 日**）[確S]。更新は EST **`simplereenroll`** で行われます。

**Cumulocity 側から見た論点**:

| # | 論点 | 内容 | 対処 |
|---|---|---|---|
| **1** | **更新は c8y-proxy 経由。Cumulocity に繋がっていることが前提** | mapper が動いていて Edge に到達できなければ更新できない [確S] | 長期オフラインを許さない運用、または 2 |
| **2** | **失効したら現地作業になる** | *"If for any reason the certificate expires, and the device can no longer communicate with Cumulocity, then the device will need to be registered again."* [確] | **`certificate.validity.minimum_duration` を「想定最大オフライン期間 × 3 以上」に設定**（既定 30 日では 10 日以上の断で危険） |
| **3** | **Edge は自己署名証明書を使う** | *"Updating the local certificate store is notably required to connect Cumulocity Edge, as this distribution of Cumulocity uses self-signed certificates"* [確] | 社内 CA / Edge の CA をデバイス側の信頼ストアへ（§12.5） |
| **4** | **有効期限をメタ監視の対象にする** | 期限切れは静かに起きる | §12.7 |
| **5** | **失効手順（拠点撤収・機器盗難）を runbook 化する** | **監査ログ対象**（[連携仕様] §4.7） | §3.9 |

### 3.7 テナント CA の毎年 10/2 自動更新 — 一斉失効は起きない

**テナント CA は毎年 10 月 2 日 02:00 に自動更新されます** [確]。

> *"Tenant Certificate Authority (CA) is **automatically renewed on 2 October at 02:00 AM every year**."*

**ただし、この更新でデバイス証明書が失効することはありません** [確]:

> *"**The renewal process ensures that existing device certificates remain valid until their expiration.**"*
> *"**All CA metadata, private keys, and public keys remain unchanged, ensuring a seamless renewal process. Only NotAfter and NotBefore will be changed.**"*
> *"Device certificates issued by the CA continue to have 1 year validity from issuance date"*
> — [Certificate authority](https://cumulocity.com/docs/device-certificate-authentication/certificate-authority/)

**同じ鍵ペア・同じ Subject での再署名**なので、既存のデバイス証明書の署名検証は成立し続けます。影響は接続の張り直し程度です:

> *"**The connected devices may need a moment to reconnect after the renewal**, but it prevents sudden authentication failures"*

> ⚠️⚠️ **旧版（rev.1）は「10/2 深夜に全拠点のデバイス証明書が順次失効し、全拠点が同時に接続不能になる」としていましたが、これは誤りでした。**
> 「日付が確定した一斉障害」というリスク分析の型そのものは有効ですが、**対象を取り違えていました**。本書はこの型を、次項の「初回エンロール日の集中」に付け替えます。

#### 本当の輻輳リスク — 初回エンロール日の集中

デバイス証明書の更新契機は **CA の更新日ではなく、各デバイスの発行日**です（発行から 1 年・期限 30 日前に `simplereenroll`、§3.6）。したがって:

- **1 拠点を 1 日で一括プロビジョニングすると、その拠点の全デバイスが 11 か月後の同じ日に更新期を迎えます**
- **複数拠点を同じ週に立ち上げると、その週に更新が集中します**

**[判] 必須対応**:

| # | 対応 | 実施時期 |
|---|---|---|
| 1 | **証明書有効期限をメタ監視の対象にする**（発行日の分布も可視化する） | §12.7 |
| 2 | **プロビジョニング計画の段階で、拠点の立ち上げ日を意図的に分散させる**。やむを得ず集中する場合は `certificate.validity.minimum_duration` を拠点ごとにずらして更新期をばらす | 設計・運用開始前 |
| 3 | **更新の輻輳を検証する**（CT-32 を「CA 更新日のシミュレーション」から「**同一日発行のデバイス群が一斉に更新期を迎える**シミュレーション」へ付け替え）。複数デバイス同時実行のケースを含めること | W1 |
| 4 | **失効時の現地対応手順を runbook 化する**（§3.6 論点 2・5） | runbook |

> **thin-edge には既にジッターがあります** [確]: `tedge-cert-renewer@c8y.timer` は `OnCalendar=hourly` + `RandomizedDelaySec=5m`（ソース上のコメントは *"Add jitter to prevent a thundering herd of simultaneous certificate renewals"*）。**5 分幅で足りるかは上記 3 の実測で判断してください。**

> ⚠️ **[要 SV-25b] 閉域網で `simplereenroll` が成立すること自体は、CA 更新とは独立に検証が必要です。** 旧版はこの基礎検証を CA 更新シミュレーション（CT-32）に含めていましたが、両者は別物です。**初回 `simpleenroll` と更新 `simplereenroll` を、それぞれ独立したテストケースとして §14 に置いてください。**

### 3.8 デバイスツイン（フラグメント）の管理方針

**[判] 各フラグメントの「唯一の情報源」を 1 つに固定します。二重管理すると「なぜか設定が戻る」という切り分けの難しい事象になります。**

> **用語**: Cumulocity に「デバイスツイン」という機能名はありません（managed object のフラグメント更新を指す一般語です）。**`twin` は thin-edge 側の一級概念**（`te/<topic-id>/twin/<key>`）で、本節はその両方を扱います。Cumulocity の別製品 Digital Twin Manager（`/service/dtm/`）とは無関係です。

| フラグメント | 唯一の情報源 | Cumulocity 側から書いてよいか |
|---|---|---|
| `c8y_RequiredAvailability`（main device） | **§13 の設定投入（Cumulocity 側）** | ✅ **ここが正**。thin-edge の `c8y.availability.interval` は初回接続時の初期値にすぎない（§5.5） |
| `c8y_RequiredAvailability`（child device） | **プロビジョニングツール**（`responseInterval: 32767`） | ✅ プロビジョニングツールのみ。**初回接続より前に入れること**（§3.5 手順 6） |
| `c8y_SupportedOperations` | **thin-edge の capability メッセージ** | ❌ **書かない**。PUT で減らすことは可能だが、次回の 114 再申告で元に戻る（§4.2） |
| `x_Site` / `x_Camera` / `x_SensingBox` | **プロビジョニングツール**（entity 登録時の twin） | ✅ プロビジョニングツールのみ |
| `x_NoAutoConfig` | **運用者の判断**（手動または構成コード） | ✅ |
| `name` | **プロビジョニングツール** | ✅ |
| `type` | **プロビジョニングツール**（entity 登録時） | ⚠️ **[要 TE-23]** 変更可否は §2.6 E-e を要確認（下記） |

> ⚠️⚠️ **`c8y_RequiredAvailability` の「正」の向きを rev.2 で反転しました。**
> 旧版は「thin-edge が唯一の情報源。Cumulocity 側から書くと次回接続時に上書きされる」としていましたが、**事実は逆**です。SmartREST 117 は *"This will only set the value if it does not exist. Values entered, for example, through the UI, are not overwritten."* [確] であり、**先に入っている値が勝ちます**。
>
> したがって「二重管理するな」という原則は変わりませんが、**寄せる先が thin-edge 側ではなく Cumulocity 側**になります。詳細と移行手順は §5.5 を参照してください。

> ⚠️ **[要 TE-23] `type` の変更可否は未確定です。** thin-edge の REST API リファレンスにある *"Updates are limited to the `@parent` and `@health` properties only, so other properties like `@type` and `@id` cannot be updated after the registration"* は、**エンティティ種別 `@type` の話**であって、Cumulocity の MO `type` の話ではない可能性があります。MO の `type` は twin キーとして `PUT /te/v1/entities/{id}/twin/{key}` の対象になりうるため、**実機で確認してください。** §2.6 E-e の断定もこの確認結果に合わせて調整が必要です。

### 3.9 デバイスの追加・交換・撤去

| 事象 | 手順 | 論点 |
|---|---|---|
| **拠点追加** | 構成コードに `sites/site-00N.yaml` を追加 → プロビジョニング実行（§3.5） | **[判] 2 拠点目が「`sites/` の追加だけ」で立つことが設計品質の判定基準**（[担当範囲] §2.6） |
| **カメラ / BOX の増設** | child 登録（§3.5 手順 4〜7）を追加分だけ実行 | 冪等なので全件再実行でよい |
| **カメラ / BOX の撤去** | thin-edge の entity 削除（空 retained）→ Cumulocity の MO を削除するか論理削除するか | ⚠️ **[要]** 過去のイベント・アラームの source が消える。**[判] MO は残し、`x_Retired` フラグメントで論理削除**を推奨 |
| **装置の交換**（同一 siteId・同一役割） | ⚠️ **シリアルが変わるため external ID が変わる**（§2.2 X-d）。新 external ID で登録 → 旧 MO を論理削除 | **[判] external ID を交換前後で同じにしない。** 同じにすると「いつ交換したか」が追跡できなくなる |
| **拠点撤収** | ①デバイス証明書の失効 ②MO の論理削除 ③Notification 2.0 サブスクリプションの unsubscribe（§10.5）④Inventory ロール割当の解除 | ⚠️⚠️ **③を忘れると、その拠点のアラーム・イベントの POST が 500 で失敗するようになります**（§10.5・バックログ 25 MiB の Hard quota） |

> ⚠️ **撤去・交換の手順は runbook 化してください**（[担当範囲] RB-2 / RB-3 / RB-12）。**特に「MO を物理削除するか論理削除するか」は先に決めてください。** 物理削除すると過去のイベントの `source` が欠損し、オフロード済みデータとの突合ができなくなります。

**物理削除を選ぶ場合の API 仕様** [確] — `DELETE /inventory/managedObjects/{id}` のクエリパラメータ:

| パラメータ | 既定 | 意味 |
|---|---|---|
| `cascade` | **`false`** | *"When set to `true` and the managed object is a device or group, all the hierarchy will be deleted."* |
| `forceCascade` | — | 同上（強制） |
| `withDeviceUser` | — | *"it deletes the associated device user (credentials)"* |

> ⚠️ **撤収 runbook には `withDeviceUser` を明記してください。** MO だけ消してデバイスユーザーを残すと、失効済み証明書とは別に**認証情報が孤児として残ります**。また **削除は非同期**なので、完了確認を伴わない一括削除は「消えたつもり」を生みます。

### 3.10 インベントリ整合性の維持

**[判] インベントリは「投入したら終わり」ではなく、継続的に検査する対象です。**

| # | 検査 | 頻度 | §2.8 の ID |
|---|---|---|---|
| 1 | external ID の規約準拠 | CI（毎回） | CK-1 |
| 2 | device type の規約準拠 | CI | CK-2 |
| 3 | `vmsCameraId` の設定 | CI | CK-3 |
| 4 | main device の拠点グループ所属 | CI | CK-4 |
| 5 | **external ID のプレフィックスと実所属拠点の一致** | **日次** | CK-5 |
| 6 | child device に `c8y_SupportedOperations` が付いていないこと | CI | §4.3 |
| 7 | child device に `com_cumulocity_model_Agent` が付いていないこと | CI | §3.1 |

> ⚠️ **検査 5 は、BOXアダプタが誤った拠点の thin-edge に publish した場合を検出する唯一の手段です。** BOXアダプタは本チームの担当外（TB-2）ですが、**結果として本チーム所有の Cumulocity にデータが着地する**ため、検出だけは持っておく価値があります。**「アダプタの誤配送への防御は範囲外」と明示的にスコープを切る場合も、その判断を記録してください。**

---

## 4. デバイスオペレーション管理方式

### 4.1 オペレーションの全体像

**§3 が「デバイス → Cumulocity」の登録だったのに対し、本章は「Cumulocity → デバイス」の制御です。** 構成図の「デバイス管理(メイン機能) 機器登録・死活・構成管理」に相当します。

```
保守端末 / 運用知識基盤 / 基盤運用者
     │  オペレーション発行（単発 or bulk）
     ▼
Cumulocity Edge ── デバイスが接続していないと PENDING で滞留
     │  既存の拠点発 MQTT 接続で配信（D9 と整合）
     ▼
thin-edge (main device) ── supported operations に申告済みのものだけ受け付ける
     │
     ├─ c8y_SoftwareUpdate  → sm-plugin（AI モデル適用・§4.4）
     ├─ c8y_DownloadConfigFile / c8y_UploadConfigFile → 設定配布・吸上（§4.5）
     ├─ c8y_LogfileRequest  → ログ取得（§4.6）
     └─ c8y_Restart         → 再起動
```

**本構成でのオペレーションの利用方針**:

| 利用者 | 発行するオペレーション | 経路 | 権限 |
|---|---|---|---|
| **保守端末** | `c8y_SoftwareUpdate`（AI モデル配布） | 保守 VPN → Cumulocity UI / CLI | §11.2 の保守ロール |
| **基盤運用者** | `c8y_LogfileRequest` / `c8y_Restart` / 設定配布 | 業務端末 → Cumulocity UI | §11.2 |
| **運用知識基盤** | 設定配布（`c8y_DownloadConfigFile`） | サーバサイド常駐 → REST | §11.4 の書込アカウント + §4.10 のガードレール |

### 4.2 supported operations の設計

#### 仕組み [確]

- `c8y_SupportedOperations`（SmartREST **114**）は**固定リストではなく、`/etc/tedge/operations/c8y` 配下のファイル名の集合**から動的に生成される
- ファイルは **MQTT capability メッセージ**（`te/<topic-id>/cmd/<operation>` に retained publish）を受けてマッパーが自動生成する
- child device 用は `/etc/tedge/operations/c8y/<child-device-xid>/`
- 追加で **SmartREST 118 = supported logs / 119 = supported configs** が申告される

> ⚠️ **supported operations は「デバイス側（thin-edge）が申告するもの」です。** したがって本節の「決定」は thin-edge の構成コードに落ちる要件になります（実装は [担当範囲] §5.8）。
>
> 厳密には `c8y_SupportedOperations` は通常の MO フラグメントなので **Cumulocity 側から PUT で減らすこと自体は可能**ですが、**次回の 114 再申告で元に戻ります**。「減らせない」のではなく「**戻る**」ため、Cumulocity 側で消す運用は取らないでください。

#### **[判] 取捨の決定**

| オペレーション | 画像解析装置 | BOXゲートウェイ | 制御方法（thin-edge 側） |
|---|---|---|---|
| `c8y_SoftwareUpdate` | **✅ 必須**（AI モデル配布・§4.4） | ❌ | `c8y.enable.software_update` |
| `c8y_DownloadConfigFile`（config_update） | **✅**（パイプライン設定・死活間隔） | ✅（GW 自身のみ） | `c8y.enable.config_update` + `agent.enable.config_update` |
| `c8y_UploadConfigFile`（config_snapshot） | **✅**（現在値の吸い上げ） | ✅ | `c8y.enable.config_snapshot` |
| `c8y_LogfileRequest` | **✅**（現地に行かずログを取る唯一の手段） | ✅ | `c8y.enable.log_upload` |
| `c8y_Restart` | ✅（**変更ウィンドウ制約つき**） | ✅ | `c8y.enable.device_restart` |
| `c8y_Firmware` | **❌ 無効化** | **❌** | `c8y.enable.firmware_update = false` |
| `c8y_DeviceProfile` | △（§4.5 のデバイスプロファイルを採るなら ✅） | ❌ | `c8y.enable.device_profile` |
| **`c8y_RemoteAccessConnect`** | **❌ 無効化**（§4.7） | 運用判断 | **設定フラグは存在しない**。§4.7 の手順 |
| `c8y_Command` | **❌**（任意コマンド実行経路を作らない） | ❌ | オペレーション定義ファイルを置かない |
| **child（カメラ / BOX）の全オペレーション** | **❌ 一切申告しない** | **❌** | capability メッセージを publish しない |

**BOXゲートウェイの申告は 4 種のみになります**:

```
114,c8y_DownloadConfigFile,c8y_LogfileRequest,c8y_Restart,c8y_UploadConfigFile
```

> **[確] 再同期の手段**: 設定を変えたのに Cumulocity 側の一覧が古い場合は `tedge mqtt pub te/device/main/service/tedge-mapper-c8y/signal/sync '{}'` で main とすべての child の supported operations を再送できます。

### 4.3 ⚠️ child device には一切申告しない

**これは「設定を減らす」以上の意味があります。**

| # | 理由 |
|---|---|
| 1 | **カメラも BOX（改造不可）もオペレーションを実行できません。** 申告すると Cumulocity の UI に実行ボタンが出て、運用者が実行してしまいます |
| 2 | 実行されたオペレーションは **PENDING のまま滞留**し、オペレーションのリテンション（§7.2）を圧迫します |
| 3 | ⚠️ **child device の supported operations は「動的に」削除できません** [確]（*"Dynamic removal of an operation from the supported operation list is not supported for child devices."*）。取り消しにはマッパー再起動を伴う手順が要ります（下記） |

> ⚠️ **rev.1 の「一度申告すると消せない」は過度な一般化でした。** 公式の警告は **動的**削除（capability メッセージの取り下げによる自動反映）が効かないという意味で、**取り消し自体は可能**です。
>
> 1. `/etc/tedge/operations/c8y/<child-device-xid>/` から該当ファイルを削除
> 2. **`tedge-mapper-c8y` を再起動**
> 3. `tedge mqtt pub te/device/main/service/tedge-mapper-c8y/signal/sync '{}'` で再送（*"This command republishes the supported operations for the main device and **and all its child devices**"*）
>
> SmartREST 114 側も *"If you want to remove an item from the supported operations list, send a new 114 request with the updated list"* と明記しています。**「後戻りできない」ことを理由に判断を硬直させないでください**（SV-17 / SV-28 の評価に影響します）。ただし**運用中の全拠点でマッパー再起動が必要**なので、コストが高いことに変わりはありません。

> **[判] 検証項目として明示的に置いてください**（§14 CT-16）: 「カメラ・BOX の詳細画面にオペレーションの実行 UI が出ないこと」。**取り消しには全拠点のマッパー再起動が要るため、パイロット拠点で必ず確認してください。**

### 4.4 ソフトウェア更新（AI モデル配布）

#### 全体フロー

| 段階 | 主体 | 手段 | Cumulocity 側の設定 |
|---|---|---|---|
| 1. モデル取得 | 保守端末 | クラウドから DL（保守 VPN） | — |
| 2. リポジトリ登録 | 保守端末 → Cumulocity | `c8y software create` / `c8y software versions create` | **ソフトウェアリポジトリ**（§13.3.9） |
| 3. 更新 Operation 発行 | 保守端末 → Cumulocity | `c8y operations create` / `c8y bulkoperations create` | 保守ロール（§11.2） |
| 4. 配信 | Cumulocity → デバイス | **拠点発の既存 MQTT 接続**（D9 と整合） | `c8y_SoftwareUpdate` が supported operations にあること |
| 5. 適用 | thin-edge sm-plugin | 失敗時ロールバック（実装は [担当範囲] §6.3） | — |
| 6. 結果の記録 | thin-edge → Cumulocity | `x_ModelApplied` イベント / `x_ModelUpdateFailed` アラーム | §6.4・§9.2 |

**[判] `softwareType` は `aimodel` に固定します。** thin-edge 側の sm-plugin 名と一致させる必要があります。

#### Cumulocity 側の論点

| # | 論点 | 内容 |
|---|---|---|
| **1** | ⚠️ **[要 SV-13 / TE-16] `files/max.size` の実値** | AI モデルのサイズ上限になる。`GET /tenant/system/options` で確認 |
| **2** | ⚠️ **[要 TE-20] 旧世代の削除運用** | ソフトウェアリポジトリは MongoDB 格納のため肥大化する。**保持世代数と削除運用を決める** |
| **3** | **段階ロールアウト** | 拠点グループ単位の bulk operation + **`creationRamp` の必須指定**（§4.8） |
| **4** | **リテンション** | オペレーションは **本書が §7.2 で設定する 90 日**で削除。⚠️ **90 日は Cumulocity の既定ではありません**（既定は 60 日 — *"By default, all historical data is deleted after 60 days"*）。**設定漏れがあると 60 日で消えます。** **失敗した Operation の記録も消える**ため、結果は `x_ModelApplied` / `x_ModelUpdateFailed` として残す |
| **5** | **ロールバックは Cumulocity 側の機能ではない** | sm-plugin API は *"in case the plugin is able to manage rollbacks"* と述べており、**ロールバックはプラグイン実装者の責務**（[担当範囲] §6.3）。プラットフォームが肩代わりすることはない [確] |
| **6** | ⚠️⚠️ **sm-plugin の 5 分タイムアウト** | *"If the command fails to return within **5 minutes**, the sm-agent reports a timeout error: `4`: timeout."* [確]。**AI モデルのインストールが 5 分を超えると失敗扱いになります。** モデルサイズ・展開時間の実測が必要（**[要 SV-40]**）。超える場合は、プラグイン側で「起動して即座に戻り、進捗は別途報告する」形にできるかを [担当範囲] §6.3 と調整すること |

### 4.5 設定配布

#### ⚠️ Cumulocity の設定管理は 3 系統ある — 選択を間違えると動かない

| 系統 | フラグメント | thin-edge 標準マッパー | 採否 |
|---|---|---|---|
| **Typed file-based（設定リポジトリを使う）** | `c8y_SupportedConfigurations`（申告・SmartREST 119）/ `c8y_DownloadConfigFile` `{type, url}`（配信・524）/ `c8y_UploadConfigFile` `{type}`（吸い上げ・526） | **対応** | **✅ 採用** |
| Legacy file-based | — | — | ❌ |
| Text-based | `c8y_Configuration` `{config: string}`（SmartREST 113/513） | **非対応** | ❌ |

> ⚠️ **`c8y_Configuration` は使えません。** thin-edge 標準マッパーが非対応で、別途コミュニティプラグイン（`c8y-textconfig-plugin`）が必要になります（[設定書] 訂-6 を継承）。

#### 配布に必要なもの

| # | 必要なもの | 主体 |
|---|---|---|
| 1 | `c8y_SupportedConfigurations` の申告 | デバイス側（`tedge-configuration-plugin.toml` に登録） |
| 2 | 設定リポジトリにスナップショットを登録 | `c8y configuration create` |
| 3 | 配信 | `c8y configuration send`（type + url） |
| 4 | ⚠️ **現在値の吸い上げ（`c8y_UploadConfigFile`）** | **逸脱検出もロールバック退避も、これが前提**。必ず有効にする |
| 5 | ⚠️ **配信は URL フェッチ型。ただし URL はマッパーが書き換える** | Cumulocity が渡す URL は、マッパーによって**ローカルの C8Y プロキシ**（既定 `127.0.0.1:8001`）に書き換えられる [確]（*"rewritten by the mapper to a `<c8y-proxy-url>` — a locally accessible URL served by the C8Y HTTP proxy"*）。したがって **main device 自身が Edge の HTTPS に直接到達する必要はありません**。論点になるのは **child 上で agent を動かす場合**で、その child からプロキシに到達できること（既定は HTTP・未認証。HTTPS 化するなら child の OS トラストストアに証明書）が要件（§12.5） |
| 6 | 権限 | `ROLE_DEVICE_CONTROL_ADMIN` + **`ROLE_INVENTORY_ADMIN`**（設定リポジトリは managed object + バイナリ） |

#### ⚠️ 配布対象の確定が先

| 対象 | Cumulocity のデバイスか | 配布経路 | 状態 |
|---|---|---|---|
| **画像解析装置**（thin-edge 自身の設定） | ○ | `c8y_DownloadConfigFile` | ✅ |
| **画像解析パイプラインの設定** | ○（装置経由） | 同上。⚠️ **そのファイルを `tedge-configuration-plugin.toml` に登録することについて画像解析側の合意が必要** | **[要 SV-17]** |
| **BOXゲートウェイ**（thin-edge 自身） | ○ | 同上 | ✅ |
| **BOXアダプタのマッピング定義** | ✕（VM1 コンポーネント） | Cumulocity 経由か Ansible か未決 | **[要 SV-27]** |
| **IP カメラ**（fps・解像度・検知感度） | ○（child device） | thin-edge 経由の child device オペレーション | **[要 SV-28]**。⚠️ §4.3 で child に何も申告しない方針のため、**現状の設計では配布経路がありません** |
| **画像センシングBOX** | ○（child） | **改造不可のため対象外** | — |

> ⚠️ **§4.3（child に何も申告しない）と「カメラへの設定配布」は両立しません。** カメラへの設定配布が要件になる場合、**child に `c8y_DownloadConfigFile` だけを申告する**という選択になります。取り消しは可能ですが**全拠点のマッパー再起動を伴います**（§4.3）。**SV-17 / SV-28 の決着まで、child には何も申告しない方針を維持してください。**

#### ⚠️ 設定配布の落とし穴

| # | 落とし穴 | 対処 |
|---|---|---|
| 1 | **file プラグイン経由の設定更新にはロールバックが無い** [確]（*"…powered by the file plugin, **no rollback to the old configuration is attempted** to maintain backward compatibility."*）。⚠️ **一方、typed file-based のプラグイン実装側にはロールバック機構がある**（agent が *"calls the plugin's rollback command … to restore the previous configuration"* を行う。**thin-edge 2.0 で追加**） | 独自 config plugin を書くか、配信前の値検証で守る（§4.10 G-3）。**どちらの経路を使うかで前提が変わる**ため [担当範囲] §6.4 と経路を明示的に合わせること |
| 2 | **単位違い・null でも thin-edge は「適用成功」を返す** | **ホワイトリストと値域を配信側で検証する**（§4.10 G-3） |
| 3 | 届くのは toml に登録した設定タイプだけ | 対象ファイルの所有権が移ることについて相手チームの合意が必要 |

### 4.6 ログ取得

**`c8y_LogfileRequest` は「現地に行かずにログを取る唯一の手段」です。** 閉域網・D9（拠点への着信接続なし）の構成では特に重要です。

| # | 論点 | 内容 |
|---|---|---|
| 1 | **取得できるログの種類** | `tedge-log-plugin.toml` で宣言したもの + journald（実装は [担当範囲] §6.5） |
| 2 | ⚠️ **[要 TE-18] ログマスキング規約** | **`c8y_LogfileRequest` で吸い上げたログは Cumulocity（＝拠点横断で見える場所）に載ります。** 個人情報・業務データを含めない規約を本チームが定義する必要がある |
| 3 | **取得したログの保持** | ログはイベント添付として保存される。リテンションが添付に及ぶかは **[要 SV-06]**（§7.4） |
| 4 | **権限** | `ROLE_DEVICE_CONTROL_ADMIN`。§11.2 の基盤運用者ロールに含める |

### 4.7 リモートアクセスの封じ込め

**`c8y_RemoteAccessConnect` は Cumulocity から装置への SSH / VNC / Telnet を張る機能です。** D9（拠点への着信接続を設けない）の趣旨に照らし、**画像解析装置では無効化します**。

> ⚠️⚠️ **リスクは SSH / VNC にとどまりません。** thin-edge の remote access は **`PASSTHROUGH` タイプで任意の TCP 接続**を通せます [確]（*"arbitrary TCP connections"*）。**封じ込めの必要性は rev.1 の想定より強い**と考えてください。

> ⚠️ **無効化のための thin-edge 設定フラグは存在しません** [確]。次の組み合わせで封じます。

| # | 手段 | 主体 | 確度 |
|---|---|---|---|
| 1 | **`c8y-remote-access-plugin` パッケージを導入しない**（`tedge-full` を使わない） | thin-edge 側 | [判] 最も確実 |
| 2 | `/etc/tedge/operations/c8y/c8y_RemoteAccessConnect` を削除 → 114 の再送で Cumulocity 側の一覧からも消える | thin-edge 側 | [確] |
| 3 | `c8y-remote-access-plugin.socket` を disable | thin-edge 側 | **[確]**（socket 必須であることは公式に明記） |
| 4 | ⭐ **Cumulocity テナント側で Cloud Remote Access のロールを誰にも割り当てない** | **Cumulocity 側（本書の担当）** | [判] |

**Remote access のタブが UI に出る条件は 3 つすべてが揃ったときです** [確]:

1. Cloud Remote Access マイクロサービスがテナントに subscribe されている
2. ユーザーに **`ROLE_REMOTE_ACCESS_ADMIN`** が割り当てられている（**既定ではどのグループ・ユーザーにも未割当**）
3. デバイスが `c8y_RemoteAccessConnect` を申告している

> ⚠️ **2 のロール名を §11.2 R-13 と §13.7 AS-4 に明記してください。** rev.1 はロール名を書いていなかったため、**CI 検査（「誰にも割り当てられていないこと」の確認）が実装できない**状態でした。

> ⚠️ **2 だけではパッケージの再インストール・更新で復活し得ます。** **1 と 4 を主、2・3 を従**としてください。

> ⚠️ **[要 SV-41] そもそも Edge で Cloud Remote Access マイクロサービスが利用可能かが未確認です。** Edge 2026 の機能比較表・Edge OpenAPI・WebSearch の 3 経路で決定的な記述を発見できませんでした。**未サブスクライブなら条件 1 が成立せず、本節の懸念自体が消えます。** §12.4 の導入直後に確認してください。

> **検証**（§14 CT-17）: 「Cumulocity UI に Remote access のボタンが出ないこと」「`114` の一覧に `c8y_RemoteAccessConnect` が無いこと」「**`ROLE_REMOTE_ACCESS_ADMIN` がどのグローバルロールにも含まれないこと**」。

### 4.8 一括オペレーションと流量制御

| 項目 | 内容 | 確度 |
|---|---|---|
| **対象指定** | bulk operation の `groupId`（拠点グループ） | [確] |
| ⭐ **流量制御** | リクエスト本文の **`creationRamp`**（*"Delay between every operation creation in seconds."*、例 `15`）。**発行側で指定できる主たる制御**。⚠️ **go-c8y-cli のフラグ名は `--creationRampSec`**（`creationRamp` は API のボディフィールド名であってフラグ名ではない） | [確] |
| 失敗分の再実行 | `failedParentId` で失敗分だけ再スケジュールできる。⚠️ **`groupId` とは排他**（*"`groupId` and `failedParentId` are mutually exclusive."*） | [確] |
| システムオプション | `device-control/bulkoperation.creationramp` は実在するが**読み取り専用**で、意味（既定値か下限か）を述べた一次記述は無い | **[推 / 要 SV-19]** |

> **[判] `creationRamp` を CI 側で必須パラメータ化してください。** 指定しないと、拠点数 × 装置数のオペレーションが一斉に生成され、Edge のスループット上限（§7.7）に直撃します。
> **これは本書の判断であると同時に API の要求でもあります** [確]: *"**It is required to specify the delay between the creation of subsequent operations.**"*

> ⚠️⚠️ **bulk operation は精密なスケジューラではありません** [確]:
>
> > *"Bulk operations are an asynchronous, **best-effort** mechanism … **not a precision scheduling tool**"*
>
> **§4.9 R-c（変更ウィンドウ内に収める）を bulk operation の `startDate` だけで担保しないでください。** ウィンドウ超過を検知する側（§4.10 のキルスイッチ）を必ず併設してください。

### 4.9 段階ロールアウト

**[判] 配信は必ず「1 拠点 → 数拠点 → 全拠点」で進めます。単位は拠点グループです（§1.7 G-d）。**

| # | 規約 | 内容 |
|---|---|---|
| **R-a** | **各段階に「進行条件」を数値で置く** | 適用後 N 時間、対象拠点の検知イベント数・アラーム数・オペレーション失敗率が基準内 |
| **R-b** | **打ち切り基準を先に決める** | 基準を超えたら次段階へ進まない。誰が判断するかも決める |
| **R-c** | **変更ウィンドウと凍結期間** | 顧客拠点の稼働ピーク・イベント当日の配信を防ぐ。配信可能時間帯をスケジューラで強制し、凍結日リストを持つ |
| **R-d** | **除外タグ** | `x_NoAutoConfig` を持つ MO は配信対象から外す。**検証で実際に除外されることを確認**（§14 CT-25） |

### 4.10 キルスイッチとガードレール

**運用知識基盤は、この構成で唯一「機械が判断して機械が書く」経路です。** 方針ではなく**技術的に強制できる統制**を主にします。

| ID | ガードレール | 内容 | 強制力 |
|---|---|---|---|
| **G-1** | ⭐ **キルスイッチ** | ①書込アカウントの即時無効化 ②一括オペレーションのキャンセル ③PENDING オペレーションの一括削除 — の 3 手順を runbook 化し、**実際に実行して止まることを確認する**（§14 CT-24） | 強 |
| **G-2** | **ブラストレディウスの権限的制限** | 書込アカウントに拠点横断権限を**常時持たせない**。配信対象の拠点グループに対する Inventory ロールを**承認時に一時付与**する | 強 |
| **G-3** | **配信前の値検証** | 配信可能な設定キーの**ホワイトリストと値域**を定義し、範囲外は配信前に拒否。単位違い・null でも thin-edge は「適用成功」を返す | 強 |
| **G-4** | **`creationRamp` の必須指定** | §4.8 | 強 |
| **G-5** | **段階ロールアウトと打ち切り基準** | §4.9 | 中 |
| **G-6** | **入力データの鮮度・完全性ゲート** | オフロードが静かに失敗 → 「イベントが激減した」と誤解釈 → 閾値を下げる配信 → 誤検知爆発、を防ぐ。入力データの期間・件数・拠点カバレッジを検査し、閾値未満なら**出力自体を抑止** | 強 |
| **G-7** | **デバイスプロファイルとの排他** | 「逸脱」判定 → 準拠のため再適用 → 運用知識基盤が再配信、の**フラッピング**を防ぐ。プロファイル管理キーと自動最適化キーを**排他**にする | 強 |
| **G-8** | **変更理由のトレーサビリティ** | 配信するオペレーション／設定に**相関 ID**（分析ジョブ ID・入力データ期間・モデルバージョン・承認者）をフラグメントとして埋める | 中 |
| **G-9** | **クールダウン** | デバイス単位・キー単位で「同一キーは 24 時間に 1 回まで」等 | 中 |
| **G-10** | **除外タグ** | `x_NoAutoConfig`（§4.9 R-d） | 中 |
| **G-11** | **変更ウィンドウと凍結期間** | §4.9 R-c | 中 |
| **G-12** | **人手承認（初期版）** | **初期版は「提案のみ・適用は人手承認」。** 完全自動化は運用実績を積んでから | 強 |
| **G-13** | **監査ログ保持** | リテンションの `AUDIT` をイベント 90 日とは別基準に（§7.2） | 中 |
| **G-14** | **資格情報管理** | 書込アカウントの資格情報は CI シークレットストア。`credentials.*` としてテナントオプションに置かない | 中 |

> **[判] 読取アカウントと書込アカウントを分けてください**（§11.4）。単一アカウントだと**分析処理のバグが本番デバイスの設定を壊せる状態**になります。実装コストはユーザーを 1 つ増やすだけです。

### 4.11 オペレーションのリテンションと PENDING 滞留

| 事実 | 帰結 |
|---|---|
| オペレーションはリテンション対象（`OPERATION` = 90 日・§7.2） | 古いオペレーション記録は消える |
| **デバイスが接続していないオペレーションは PENDING のまま残る** | child に申告すると溜まり続ける（§4.3） |
| **PENDING の一括削除はキルスイッチの 3 手順目**（G-1） | 緊急時の手段として runbook 化 |

> **[判] PENDING の件数を監視対象にしてください。** 増え続けている場合、①申告してはいけないオペレーションを申告している ②デバイスが接続できていない ③一括オペレーションが流量制御なしで発行された、のいずれかです。

---

## 5. デバイスの死活監視方式

### 5.1 死活検知の一次責務の一元化

**[判] 死活検知の一次責務を対象ごとに 1 箇所へ寄せます。** 二重に検知させると、同じ事象で 2 種類のアラームが出て運用が混乱し、片方だけクリアされて残ります。

| 対象 | 一次検知の担当 | アラーム型 | `c8y_RequiredAvailability` |
|---|---|---|---|
| **画像解析装置** | **Cumulocity**（thin-edge のハートビート） | `c8y_UnavailabilityAlarm` | **設定する（15 分）** |
| **BOXゲートウェイ** | **Cumulocity**（同上） | `c8y_UnavailabilityAlarm` | **設定する（10 分）** |
| **IP カメラ** | **thin-edge A の ONVIF 監視サービス** | `x_CameraDown` | **`responseInterval: 32767`（実質無効化）**。⚠️ **`0` は不可**（§5.4） |
| **画像センシングBOX** | **BOXアダプタ**（無通信タイムアウト） | `x_BoxSilent` | **`responseInterval: 32767`（実質無効化）**。⚠️ **`0` は不可**（§5.4） |
| **Cumulocity のマイクロサービス** | **Cumulocity**（自動生成） | `c8y_Application_Down` / `c8y_Application_Unhealthy` | — |
| **Edge 自体** | **メタ監視**（Prometheus・§12.7） | Cumulocity の外 | — |

> ⚠️ **`c8y_Application_*` の対象は Cumulocity のマイクロサービスです。** VM1/VM2 上の非 Cumulocity コンポーネント（Keycloak・Otel・BOXアダプタ等）は対象外で、そちらは Otel 側の責務です。

### 5.2 `c8y_RequiredAvailability` の仕組み

| 項目 | 内容 | 確度 |
|---|---|---|
| 機能 | `responseInterval` を設定すると、指定時間内に通信が無い場合にアラームが自動生成される | [確] |
| 生成されるアラーム | `c8y_UnavailabilityAlarm`（既定 MAJOR / *"No data received from the device within the required interval."*） | [確] |
| **値域** | **`-32768`〜`32767`（分）。範囲外は境界に丸められる** | [確] |
| **メンテナンスモード** | **`responseInterval` が `0` 以下**でメンテナンスモードに入る（`0` だけでなく負値も含む） | [確] |
| 自動クリア | 通信再開時に Cumulocity が自動でクリア | [確] |
| ⚠️⚠️ **メンテナンスモードの副作用** | **そのデバイスを `source` とする「すべての」アラームが抑止される。** `c8y_UnavailabilityAlarm` だけでなく `x_CameraDown` も EPL・スマートルールも発火しなくなる | [確] |

> ⚠️⚠️ **メンテナンスモードは「可用性監視の無効化」ではなく「そのデバイスのアラーム全停止」です。** 公式に逐語で明記されています。
>
> > *"**Alarm suppression** — If the source device is in maintenance mode, **the alarm is not created and not reported to the Cumulocity event processing engine.**"*（Core OpenAPI `POST /alarm/alarms`）
> >
> > *"**While a device is being maintained, no alarms for that device are raised.**"*（[Monitoring and controlling devices](https://cumulocity.com/docs/device-management-application/monitoring-and-controlling-devices/)）
>
> **しかも POST は成功扱いで返り、レスポンスの `self` が `https://<TENANT_DOMAIN>/alarm/alarms/null` になるだけです。** 呼び出し側はエラーを受け取らないため、「アラームを送っているのに 1 件も登録されていない」ことに気づくのが極端に遅れます。**§14 に「メンテナンス中の child へ `x_CameraDown` を POST し、実際に登録されるか」の否定テストを必ず置いてください（CT-9b）。**

### 5.3 ⚠️⚠️ 落とし穴 — thin-edge は既定で全 child device に 60 分を設定する

thin-edge.io の公式ドキュメントに逐語で [確]:

> *"%%te%% main and child devices set their required interval during their first connection. Availability monitoring is enabled by default with a default required interval of 1 hour."*

child device への送出はソースでも確認できます [確S]：`crates/extensions/c8y_mapper_ext/src/availability/actor.rs` は child device の新規登録時に `send_smartrest_set_required_availability_for_child_device` を呼び、**main と同じ `c8y.availability.interval` の値**を SmartREST **117** で送ります。

**つまり何もしないと、代理登録した全カメラ・全 BOX に `c8y_RequiredAvailability = 60 分` が付きます。** カメラ・BOX は自分でハートビートを送らないため（既定のヘルス判定元はそのデバイス上の `tedge-agent` サービスで、代理登録した child には存在しない）、**接続から 1 時間後に全 child が一斉にアラームを上げると推測されます** [推]。

**数百台 × N 拠点の規模では、初回接続の 1 時間後に数百〜数千件のアラームが同時発生することを意味します。しかもアラームは `CLEARED` にならない限り消えません**（§7.3）。

### 5.4 **[判] 対処方針**

| # | 対処 | 内容 |
|---|---|---|
| **1** | **ゲートウェイの死活は thin-edge のハートビート機構を使う** | `c8y.availability.interval` を設定。**1 分未満または 0 を設定すると無効化される** [確]。⚠️ **ただし §5.5 の注意（117 は既存値を上書きしない）を必ず読むこと** |
| **2** | ⭐ **child device は `responseInterval` に十分長い正値を入れる** | `c8y_RequiredAvailability = {"responseInterval": 32767}`（上限値・約 22.7 日）を twin で設定（§3.5 手順 6）。**`0` にしてはいけない** — メンテナンスモードに入り `x_CameraDown` ごと抑止される（§5.2）**[要 TE-5: 実機で child への twin 反映を確認]** |
| 3 | 代替案: `c8y.availability.enable false` | *"When disabled, the required availability interval and periodic heartbeat messages aren't sent to Cumulocity"* [確]。**child に 117 が一切飛ばなくなる**ので最も確実。ただし **main / child を分離できない全体スイッチ**で、main のハートビートも止まる。採る場合は main の `c8y_RequiredAvailability` を §13 の設定投入で明示的に入れること |
| 4 | 代替案: `@health` を使う | child 登録メッセージの `@health` に監視サービスの topic id を指定すると、そのサービスが `up` の間だけハートビートが送られる [確]。**公式機構だが、アラーム型が `c8y_UnavailabilityAlarm` になり `x_CameraDown` 規約と食い違う**ため、**本書は 2 を採る** |

> ⚠️⚠️ **旧版（rev.1）は対処 2 を「メンテナンスモード（`responseInterval: 0`）へ落とす」としていましたが、これは誤りでした。**
> メンテナンスモードは可用性監視だけでなく **そのデバイスを `source` とする全アラームを抑止**します（§5.2）。§5.6 で同じカメラに `x_CameraDown` を上げる設計と真正面から衝突し、**「死活監視を設計したのに死活アラームが 1 件も上がらない」**状態になります。

> ⚠️ **対処 2 の残存リスク**: `responseInterval: 32767` は「無効化」ではなく「約 22.7 日で発報」です。カメラ・BOX が §5.6 要件 4（正常時も定期的に `x_CameraHealth` を送る）を守っていれば発報しませんが、**§5.9 の削減方針 2（正常時は送らない）を採ると 22.7 日後に全 child が一斉に `c8y_UnavailabilityAlarm` を上げます。** 方針 2 を採る場合は対処 3（`c8y.availability.enable false`）に切り替えてください。**[判] この組み合わせ制約を §5.9 の決定（TE-15）と同時に確定させること。**

> ⚠️⚠️ **設定の「正」の向きに注意 — SmartREST 117 は既存値を上書きしません。** 旧版はここを逆に書いていました。
>
> > *"Set required availability (117) … **This will only set the value if it does not exist. Values entered, for example, through the UI, are not overwritten.**"*（[SmartREST static templates](https://cumulocity.com/docs/smartrest/mqtt-static-templates/)）
> >
> > *"This template can be sent in a fire-and-forget approach during device startup because **it doesn't override already existing required availability configuration**"*（[Fragment library](https://cumulocity.com/docs/device-integration/fragment-library/)）
>
> **帰結**: 「Cumulocity 側で手で書くと thin-edge の値に戻される」のではなく、**逆に Cumulocity 側に一度値が入ると thin-edge の `c8y.availability.interval` が二度と反映されなくなります**。実務上の影響は §5.5 を参照してください。

### 5.5 `responseInterval` の設計値

| 対象 | **[判] 提案値** | 設定箇所 | 根拠 |
|---|---|---|---|
| **画像解析装置** | **15 分** | thin-edge `c8y.availability.interval = 15m` | 死活計測が 1〜5 分間隔。WAN の瞬断・再接続で誤報しない余裕を取る。**過小だと回線瞬断のたびに拠点数ぶんのアラームが上がる** |
| **BOXゲートウェイ** | **10 分** | thin-edge `c8y.availability.interval = 10m` | 同一サーバールーム内で WAN 断の影響を受けないため短くできる |
| **IP カメラ / BOX** | **32767（実質無効）** | プロビジョニングツール（twin） | 死活は `x_CameraDown` / `x_BoxSilent` に一元化。**`0` は不可**（§5.2・§5.4） |

> ⚠️⚠️ **この設計値は「初回接続より前」に入れないと反映されません。**
> SmartREST 117 は既存値を上書きしませんが、**逆に言えば先に入っていた値が勝ちます**（§5.4）。thin-edge は初回接続時に 117 を送るため、順序によって結果が変わります。
>
> | 順序 | 結果 |
> |---|---|
> | ① Cumulocity 側に先に値を入れる → ② thin-edge 初回接続 | **① の値が残る**（117 は無視される）→ **狙いどおり** |
> | ① thin-edge 初回接続（60 分が入る） → ② `tedge config set c8y.availability.interval 15m` | **60 分のまま。15 分は永久に反映されない** |
>
> **既に接続済みの装置に対しては `tedge config set` だけでは値が変わりません。** 次のいずれかで既存フラグメントを置き換えてください。
>
> - Cumulocity 側の MO を直接 PUT して `c8y_RequiredAvailability` を書き換える（**推奨**。§13 の設定投入に載せれば冪等・版管理できる）
> - SmartREST `107` でフラグメントを削除してから 117 を再送させる
>
> **[判] 本書は「main device の `c8y_RequiredAvailability` は §13 の設定投入（Cumulocity 側）を正とする」に改めます。** thin-edge の `c8y.availability.interval` は「初回接続時の初期値」としてのみ機能し、運用中の変更手段にはなりません。**[要 TE-5b] 移行手順（既存拠点の値の一括置換）を runbook 化してください。**

> ⚠️ **[要 SV-26] 上記は提案値です。** 確定には「許容できる検知遅れ」と「実測された WAN 瞬断の頻度・長さ」が要ります。**検証環境で瞬断を再現し、パイロット拠点で観測期間を定めて実測（例: 2 週間）してから固定してください。** 合格条件は「その期間中の誤報件数が合意した許容件数以下であること」です。

### 5.6 カメラ・BOX の死活

| 対象 | 検知主体 | 手段 | アラーム | クリア |
|---|---|---|---|---|
| **IP カメラ** | thin-edge A の ONVIF/ICMP 監視サービス（自作・[担当範囲] §6.2） | ONVIF `GetSystemDateAndTime` / ICMP。間隔 1〜5 分 | **無応答 N 回連続（既定 3 回）** で `x_CameraDown`（MAJOR） | 応答再開時に監視サービスがクリア |
| **画像センシングBOX** | BOXアダプタ（担当外・TB-2） | 無通信タイムアウト | `x_BoxSilent`（MAJOR） | 受信再開時にアダプタがクリア |

**Cumulocity 側から見た要件**:

| # | 要件 | 理由 |
|---|---|---|
| 1 | **ヒステリシス（N 回連続）を必ず入れる** | フラッピングすると同一アラームの ACTIVE/CLEARED が往復し、通知が乱発される |
| 2 | **監視サービスの起動時にアラーム状態を全カメラぶん再評価（reconcile）する** | 監視サービスが死んでいた間のアラームが古いまま残る |
| 3 | **クリアは必ず実装する** | アラームは `CLEARED` にならない限り消えず、リテンションでも削除されない（§7.3） |
| 4 | **正常時も定期的に `x_CameraHealth` 計測を送る** | 「監視が動いているか」を Cumulocity 側から判定する手段になる。ただし §5.9 の送信量に直結 |

### 5.7 サービスの死活

**thin-edge のサービス（mapper・agent・自作の監視サービス等）は Cumulocity 上に「サービス」として表示されます。**

| 項目 | 内容 | 確度 |
|---|---|---|
| 仕組み | `te/<service-topic-id>/status/health` に `{"pid":…, "status":"up"/"down", "time":…}` を **retained** で publish。Cumulocity には SmartREST **102**（サービス作成）/ **104**（状態更新）として反映 | [確] |
| **LWT** | *"The services are also expected to register an MQTT Last Will and Testament (LWT) message"*。**自作サービスには必ず LWT を実装する** | [確] |
| ウォッチドッグ | `tedge-watchdog` + systemd の `WatchdogSec`。**unit ファイルに `After=tedge-watchdog.service` / `WatchdogSec=30` / `Restart=always` を追加**しないと有効にならない | [確] |

> **[判] 「監視サービスの死」を検知できるようにしてください。** カメラ死活監視サービスが死ぬと、①カメラのアラームが古いまま残り ②新規の断も検知できません。**LWT + watchdog + 起動時 reconcile の 3 点セットが要件です**（§14 CT-10）。

### 5.8 アラームストームの抑止

**[判] 死活設計における最大のリスクは「1 つの事象で数百〜数千件のアラームが同時発生すること」です。**

| 発生シナリオ | 件数 | 抑止策 |
|---|---|---|
| **初回接続の 1 時間後**（§5.3 の落とし穴） | 全 child（数百〜数千） | **§5.4 の対処 2**（登録時に `responseInterval: 32767`）。**初回接続より前に入れること**（§5.5） |
| **WAN 瞬断**（`responseInterval` が過小） | 拠点数ぶんの main device | **§5.5 の値の実測確定** |
| **Edge 停止 → 復旧** | 全 main device | Cumulocity 側の自動クリアで解消。ただし通知が復旧時刻に集中する → §8.4 |
| **GW 停止**（案 α のみ） | 全拠点の BOX | 案 β では発生しない → 付録 A |
| **網断復旧時の一括再送** | 断の長さに比例 | **リプレイ抑止ガード**（§8.4） |

> ⚠️ **アラームは `CLEARED` にならない限り削除されません** [確]。ストームが 1 回起きると、手動クリアするまで Operational Store に残り続けます。**「起きてから消す」ではなく「起こさない」設計が必要です。**

**Cumulocity 標準の重複排除は、ここでは効きません** [確]:

> *"**Alarm de-duplication** — If an **ACTIVE or ACKNOWLEDGED** alarm with the same source and type exists, no new alarm is created. Instead, the existing alarm is updated by incrementing the `count` property; the `time` property is also updated. **Any other changes are ignored**, and the audit log is not created. Alarms with status CLEARED are not de-duplicated."*（Core OpenAPI `POST /alarm/alarms`）

- 重複排除の単位は **(`source`, `type`)** です。上表のストームは **`source` が全 child ぶん異なる**ため 1 件に畳まれません
- **`CLEARED` を挟むフラッピングも畳まれません**。§5.6 要件 1（ヒステリシス）は必須のままです
- ⚠️ **重複排除は ACTIVE だけでなく ACKNOWLEDGED でも働きます。** 運用者がアラームを「確認済み」にした後も、同じ (`source`, `type`) の新規アラームは立たず `count` が増えるだけです。**「確認済みにしたら次の異常が見えなくなる」わけではありませんが、新着として通知されない**点は運用手順（§10.6）で織り込んでください

### 5.9 死活計測の送信量とスケールへの影響

**死活計測は Edge の受信量の支配項になります。**

```
[連携仕様] §3.5 の試算 : 50 拠点 × 300 台 ÷ 60 秒 ≒ 250 msg/s（死活のみ）
Edge 比較表の目安      : 約 100 tps / CPU コア                      [確]
Narrow ベンチマーク    : 8 CPU スレッド / 16GB で 25,000 measurement/s [確]
Wide ベンチマーク      : 8 CPU スレッド / 16GB で 1,200 接続クライアント [確]
```

> ⚠️⚠️ **公式ドキュメント自身が非整合です。** 比較表の「約 100 tps / CPU コア」（[Edge introduction](https://cumulocity.com/docs/2026/edge/edge-introduction/) の `Vertical scalability` 行に *"Yes, limited to appr. 100 tps per CPU core"* と逐語）と、ベンチマークの Narrow 値（8 スレッドで 25,000 measurement/s ≒ 3,100/スレッド）は **30 倍以上乖離**します。ベンチマークページには *"for illustrative purposes only"* の Caution があり、単位も「CPU **threads**」です。
>
> **どちらも公式なので、どちらか一方をサイジングの根拠にしないでください。** [判断記録] D17 の「PoC 実測を正とする」という指示がそのまま有効で、§14 SV-14（負荷試験）の必要性はむしろ強まります。

> ⚠️ **旧版（rev.1）は「Wide シナリオが本構成に直撃する」としていましたが、これは誤りでした。**
> Wide の律速は **同時接続 MQTT クライアント数**です。
>
> > *"These end-to-end test scenarios drive a number of **MQTT clients** … The clients are connecting to the MQTT service and sending measurements using the JSON via MQTT protocol."*（[Edge benchmarks](https://cumulocity.com/docs/2026/edge/benchmarks/)）
>
> **代理登録された child device は独自の MQTT 接続を持ちません**（拠点あたり thin-edge の 1 本に集約される）。したがって接続クライアント数は **child 台数ではなく拠点数のオーダー**であり、1,200 という上限には遠く及びません。**律速は Narrow 側（メッセージ毎秒）で評価してください。**
>
> **ただし child device 台数は別軸の制約として残ります** — MO 総数、デバイス課金、inventory クエリ性能（§11.9 の 2000 件閾値）。**[要 SV-14b] これらを台数軸で別途評価すること。**

**[判] 送信量の削減方針（[要 TE-15] いずれを採るか未決）**:

| # | 方針 | 効果 | 副作用 |
|---|---|---|---|
| 1 | **死活間隔を 5 分側に寄せる** | 5 倍の削減 | 検知遅れが最大 5 分 + N 回分 |
| 2 | **正常時は計測を送らず、rtt の変化・閾値超過時のみ送る** | 大幅削減 | 「監視が動いているか」が Cumulocity から分からなくなる。サービス死活（§5.7）で代替する必要 |
| 3 | thin-edge の `flows` の `onInterval` で集約する | 中程度 | 実装コスト |
| 4 | transient トピック（`t/us`）の利用 | 中程度 | **[要] 保証レベルの差が未確認**（V14-4） |

> **[判] まず 1 を採り、実測（§14 CT-29）で不足なら 2 を検討してください。** 2 を採る場合、**「計測が来ないこと」と「監視サービスが死んでいること」を区別する手段**（§5.7 のサービス死活）を先に成立させてください。

---

## 6. データモデルの概念

> **本章が [連携仕様] §4.3 / [カメラ構成] §4.2〜4.3 / [担当範囲] §4.2 を統合し、本構成のデータ定義の「正」になります。**

### 6.1 Cumulocity のデータ種別と本構成での使い分け

**Cumulocity は 7 種類のデータを扱います。「何をどれで表現するか」を先に固定しないと、同じ事象が Event と Alarm の両方に出たり、状態を Measurement で送ってしまったりします。**

| 種別 | Cumulocity での意味 | **[判] 本構成での使い方** | 使わないもの |
|---|---|---|---|
| **Inventory**（Managed Object） | デバイス・グループ・設定リポジトリ等の「もの」 | デバイス階層（§3.1）・拠点グループ（§1）・ソフトウェア/設定リポジトリ | — |
| **Measurement** | **数値の時系列**。集計・グラフ化の対象 | **死活の定期記録のみ**（`x_CameraHealth`） | ⚠️ **検知結果を Measurement で送らない**（数値でないうえ、リテンションと突合の設計が変わる） |
| **Event** | **時点で起きた出来事**。任意の JSON フラグメントを載せられる | **AI 検知**（`x_Detection_*`）・**モデル適用完了**（`x_ModelApplied`） | — |
| **Alarm** | **継続する異常状態**。ACTIVE / ACKNOWLEDGED / CLEARED の状態を持つ | 死活異常・処理失敗・検知イベントからの昇格 | ⚠️ **一過性の出来事を Alarm にしない**（クリアされずに溜まる） |
| **Operation** | Cumulocity → デバイスの制御指示 | §4 | — |
| **Audit log** | 誰が何を変更したか | 設定変更の追跡（§7.2 の `AUDIT`） | — |
| **Binary**（添付） | イベントに添付されるファイル | **スナップショット画像**（§6.8）・ログファイル | ⚠️ **映像クリップは入れない**（D13/D1。オブジェクトストレージへ） |

> ⚠️ **Event と Alarm の使い分けを規約として明文化してください。**
>
> - **Event**: 「起きた」で完結するもの。クリアの概念がない。例: 侵入を検知した、モデルを適用した
> - **Alarm**: 「異常が続いている」もの。**必ずクリア条件とセットで定義する**（§9.6）
> - **検知イベントを直接 Alarm にしない**理由: 検知は次々起きるため、Alarm にすると同一 source + type で重複判定が働き、件数が正しく残らない。**「イベントとして残し、条件に一致したものだけルールでアラームに昇格させる」**（§8.2）のが本構成の設計です

### 6.2 構成図の各デバイスが出すデータ

**構成図（レビュー反映タブ）の「現場 → 閉域網 → 本社」の各経路が、Cumulocity 上でどのデータになるかの一覧です。**

| # | 送出元（構成図の要素） | 経路 | Cumulocity 上の `source` | データ種別 | 型 |
|---|---|---|---|---|---|
| 1 | 画像解析パイプライン | ローカル MQTT（IF-S04）→ thin-edge A | **対象カメラ**（child） | **Event** | `x_Detection_<種別>` |
| 2 | 画像解析パイプライン | c8y-proxy → REST（添付） | **対象カメラ**（child） | **Event + Binary** | `x_Detection_<種別>` + スナップショット |
| 3 | ONVIF 死活監視サービス | ローカル MQTT → thin-edge A | **対象カメラ**（child） | **Measurement** | `x_CameraHealth` |
| 4 | ONVIF 死活監視サービス | 同上 | **対象カメラ**（child） | **Alarm** | `x_CameraDown` |
| 5 | sm-plugin `aimodel` | 同上 | **画像解析装置**（main） | **Event** | `x_ModelApplied` |
| 6 | sm-plugin `aimodel` | 同上 | **画像解析装置**（main） | **Alarm** | `x_ModelUpdateFailed` |
| 7 | thin-edge A の各サービス | 同上 | **画像解析装置**配下のサービス | **Inventory**（サービス） | SmartREST 102 / 104 |
| 8 | **Cumulocity 自身** | `c8y_RequiredAvailability` の判定 | **画像解析装置 / BOXゲートウェイ**（main） | **Alarm** | `c8y_UnavailabilityAlarm` |
| 9 | BOXアダプタ | ローカル MQTT（IF-H01）→ thin-edge B | **対象 BOX**（child） | **Event** | `x_Detection_<種別>` |
| 10 | BOXアダプタ | 同上 | **対象 BOX**（child） | **Alarm** | `x_BoxSilent` |
| 11 | BOXアダプタ | 同上 | **BOXゲートウェイ**（main） | **Alarm** | `x_BoxParseError` |
| 12 | **Smart Rules / EPL** | Cumulocity 内部 | **対象カメラ / BOX** | **Alarm** | `x_Alarm_<種別>` |
| 13 | Cumulocity マイクロサービス | Cumulocity 内部 | 該当アプリケーション | **Alarm** | `c8y_Application_Down` / `_Unhealthy` |

> ⚠️ **`source` は「そのデータが誰に帰属するか」です。** 検知イベントの `source` は**画像解析装置ではなく対象カメラ**です。これが Inventory ロール（§11.3）と Notification 2.0 の購読範囲（§10.2）の両方に効きます。**装置を source にすると「カメラ単位で見せる／見せない」ができなくなります。**

> ⚠️ **BOX 経由の検知イベント（#9）も `source` は BOX（child）です。** ただし [連携仕様] N2（BOX の送信仕様）が未確認のため、**BOX が「どのカメラの検知か」を出せるかは未確定**です（TE-9）。出せない場合、BOX 単位の粒度が上限になります。

### 6.3 ⭐ MQTT トピック体系とデータタイプのプレフィックス定義

#### 6.3.1 2 つの MQTT レイヤ

**本構成には MQTT が 2 層あります。混同すると設計が破綻します。**

```
┌─ 拠点（画像解析装置 / 本社の外部Gatewayホスト）─────────────────┐
│                                                                  │
│  画像解析パイプライン / BOXアダプタ                              │
│         │                                                        │
│         │  ① ローカル MQTT（thin-edge の te/ トピック体系）      │
│         │     ※ 本構成が定義する IF 契約はここ                    │
│         ▼                                                        │
│  mosquitto（ローカル） ── tedge-mapper c8y ─┐                   │
└──────────────────────────────────────────────┼──────────────────┘
                                               │
                          ② Cumulocity MQTT（SmartREST の s/us 等）
                             ※ thin-edge が代行。本構成は直接使わない
                                               │
                                               ▼
                                      Cumulocity Edge :8883
```

| 層 | トピック体系 | 誰が使うか | 本構成での位置づけ |
|---|---|---|---|
| **① ローカル MQTT** | **`te/...`**（thin-edge の MQTT API） | 画像解析パイプライン・BOXアダプタ・自作サービス | **本書と [担当範囲] §5.6 が定義する IF 契約** |
| **② Cumulocity MQTT** | `s/us` `s/ds` `s/uat` `s/dat` `t/us` 等（SmartREST） | **thin-edge の c8y-mapper のみ** | **アプリケーションは直接使わない。** thin-edge が ① を ② に変換する |

> ⚠️ **`source` はトピックの `<child-id>` が決めます。ペイロードの `source.id` は使われません** [確]。Cumulocity REST 形式の JSON（`source: {id: ...}`）と混同しないでください。

#### 6.3.2 ローカル MQTT のトピック体系

**thin-edge の MQTT API のトピック構造** [確]:

```
te / <identifier>          / <channel> / <channel-detail>
     ^^^^^^^^^^^^            ^^^^^^^^^   ^^^^^^^^^^^^^^^^
     エンティティを指す      種別        型名など
     4 セグメント固定
```

**エンティティ識別子（4 セグメント）**:

| エンティティ | identifier | 例 |
|---|---|---|
| main device | `device/main//` | `te/device/main///m/...` |
| child device | `device/<child-id>//` | `te/device/site001-cam-SN12345///e/...` |
| main device のサービス | `device/main/service/<svc>` | `te/device/main/service/onvif-monitor/status/health` |

**チャネル（データ種別のプレフィックス）**:

| チャネル | 記号 | 完全形 | データ種別 | retain | QoS |
|---|---|---|---|---|---|
| **計測** | **`m`** | `te/device/<id>///m/<measurement-type>` | Measurement | **禁止** [確] | 0 or 1 |
| **イベント** | **`e`** | `te/device/<id>///e/<event-type>` | Event | **禁止** [確] | 1 |
| **アラーム** | **`a`** | `te/device/<id>///a/<alarm-type>` | Alarm | **必須** [確] | **2**（`>1` が要件）[確] |
| **アラームのクリア** | `a` | 同一トピックに**空メッセージ** | Alarm → CLEARED | **必須** | 2 |
| **ツイン** | `twin` | `te/device/<id>///twin/<fragment>` | Inventory のフラグメント | 必須 | — |
| **サービス死活** | `status/health` | `te/device/main/service/<svc>/status/health` | Inventory（サービス） | 必須 | — |
| **オペレーション申告** | `cmd` | `te/<topic-id>/cmd/<operation>` | supported operations | 必須 | — |
| **エラー通知** | — | `te/errors` | thin-edge の内部エラー | — | — |

> ⚠️ **`te/errors` を監視し、件数をメトリクス化してください。** 自動登録を無効化した状態（§2.6 E-a）で未登録エンティティのデータが来ると、**マッパーはそのデータを無視し、`te/errors` にエラーを出します** [確]。**ここを見ないと「送ったのに Cumulocity に出ない」の切り分けができません。**

**具体例**:

```
te/device/site001-cam-SN12345///m/x_CameraHealth      ← カメラの死活計測
te/device/site001-cam-SN12345///e/x_Detection_Intrusion ← カメラの侵入検知イベント
te/device/site001-cam-SN12345///a/x_CameraDown        ← カメラ無応答アラーム
te/device/main///e/x_ModelApplied                     ← 装置のモデル適用完了
te/device/site001-box-SN98765///a/x_BoxSilent         ← BOX 無通信アラーム
te/device/main/service/onvif-monitor/status/health    ← 監視サービスの死活
```

#### 6.3.3 ⭐ 型名のプレフィックス定義

**[判] 本構成の型名は次の規則に従います。これが本章の中核です。**

| プレフィックス | 意味 | 誰が定義するか | 例 |
|---|---|---|---|
| **`c8y_`** | **Cumulocity 標準**。製品が生成・解釈する | Cumulocity（変更不可） | `c8y_UnavailabilityAlarm` / `c8y_Application_Down` / `c8y_SoftwareUpdate` |
| **`x_`** | **本構成のカスタム**。基盤が定義する | **本チーム** | `x_CameraHealth` / `x_CameraDown` / `x_ModelApplied` |
| **`x_Detection_`** | **AI 検知イベント**の型。`<種別>` が続く | **文法は本チーム / 種別の列挙は案件側** | `x_Detection_Intrusion` / `x_Detection_Loitering` |
| **`x_Alarm_`** | **検知イベントから昇格したアラーム**の型 | 同上 | `x_Alarm_Intrusion` |

**命名規則**:

| # | 規則 | 理由 |
|---|---|---|
| **N-a** | **カスタム型は必ず `x_` で始める** | Cumulocity 標準（`c8y_`）と衝突させない。将来の製品アップデートで標準型が増えても影響を受けない |
| **N-b** | **`x_` の後は PascalCase**（`x_CameraDown`） | 表記ゆれを防ぐ |
| **N-c** | ⚠️ **`<種別>` の列挙は基盤側で固定しない** | **本チームが規約とするのは「`x_Detection_` プレフィックス + PascalCase + 必須フラグメント」だけ**。種別の列挙は案件側から受領する。**列挙を基盤側で固定すると 2 案件目で壊れます** |
| **N-d** | **`x_Detection_<種別>` と `x_Alarm_<種別>` の `<種別>` は同じ語を使う** | 昇格の対応関係が型名から読める |
| **N-e** | **型名に拠点・装置・世代を含めない** | 型はリテンション・ルール・購読フィルタのキー。可変要素を入れるとフィルタが増殖する（§2.3 T-b と同じ理由） |

#### 6.3.4 フラグメント名のプレフィックス定義

**[判] カスタムフラグメントも `x_` + PascalCase に統一します。**

| フラグメント | 載る先 | 内容 | 定義元 |
|---|---|---|---|
| `x_Detection` | 検知 Event | `{ eventUuid, modelVersion, aiProduct, confidence, boundingBoxes, clipHint }` | [カメラ構成] §4.3 |
| `x_Camera` | カメラ MO | `{ vmsCameraId, location }` | [連携仕様] §4.3 |
| `x_SensingBox` | BOX MO | `{ model, fwVersion, channels }` | **本書で新規定義** |
| `x_Site` | 拠点グループ / main device MO | `{ siteId, siteName }` | **本書で新規定義** |
| `x_NoAutoConfig` | 任意の MO | 自動設定変更の除外タグ | [設定書] G-10 |

> ⚠️ **`@` プレフィックスは thin-edge の予約です**（`@id` / `@type` / `@topic-id` / `@parent` / `@health`）。独自キーに使わないでください [確]。

#### 6.3.5 SmartREST テンプレート番号（参考）

**thin-edge が Cumulocity へ変換する際に使う番号です。Cumulocity 側でログや挙動を追うときに必要になります。**

| 番号 | 方向 | 意味 | 本構成で関係する箇所 |
|---|---|---|---|
| **102** | デバイス → C8Y | サービスの作成 | §5.7 |
| **104** | デバイス → C8Y | サービスの状態更新 | §5.7 |
| **113 / 513** | 双方向 | **Text-based 設定**（`c8y_Configuration`） | ⚠️ **使わない**（§4.5） |
| **114** | デバイス → C8Y | **supported operations の申告** | §4.2・§4.3 |
| **118** | デバイス → C8Y | supported logs の申告 | §4.6 |
| **119** | デバイス → C8Y | supported configs の申告 | §4.5 |
| **524** | C8Y → デバイス | `c8y_DownloadConfigFile`（設定配信） | §4.5 |
| **526** | C8Y → デバイス | `c8y_UploadConfigFile`（現在値吸い上げ） | §4.5 |

### 6.4 型カタログ

**[判] 本構成で使用する全型の定義です。この表が [連携仕様] §4.3 の改訂版になります。**

#### Measurement

| 型名 | `source` | 生成元 | フィールド | 用途 |
|---|---|---|---|---|
| `x_CameraHealth` | **カメラ**（child） | thin-edge A の ONVIF 監視サービス | `rtt`（ms・数値）/ `reachable`（0 or 1） | 死活の定期記録 |

#### Event

| 型名 | `source` | 生成元 | 必須フラグメント | 用途 |
|---|---|---|---|---|
| `x_Detection_<種別>` | **対象カメラ / 対象 BOX**（child） | 画像解析パイプライン（IF-S04）／ BOXアダプタ（IF-H01） | **`x_Detection`**（`eventUuid` / `modelVersion` / `aiProduct` / `confidence` / `boundingBoxes` / `clipHint`） | AI 検知 |
| `x_ModelApplied` | **画像解析装置**（main） | thin-edge A の sm-plugin | `modelVersion` | モデル適用完了。**切替時刻の追跡**（IF-S07） |

#### Alarm

| 型名 | `source` | 生成元 | severity | クリア条件 |
|---|---|---|---|---|
| `c8y_UnavailabilityAlarm` | 画像解析装置 / BOXゲートウェイ | **Cumulocity**（`c8y_RequiredAvailability`） | MAJOR | 通信再開時（**Cumulocity が自動**） |
| `x_CameraDown` | **カメラ**（child） | thin-edge A の ONVIF 監視サービス | MAJOR | 応答再開時（**監視サービスがクリア**） |
| `x_BoxSilent` | **BOX**（child） | BOXアダプタ | MAJOR | 受信再開時（**アダプタがクリア**） |
| `x_BoxParseError` | **BOXゲートウェイ**（main） | BOXアダプタ | WARNING | **[要決定]**（§9.6） |
| `x_ModelUpdateFailed` | **画像解析装置**（main） | thin-edge A の sm-plugin | CRITICAL | 再適用成功時 |
| `x_Alarm_<種別>` | **対象カメラ / BOX** | **Smart Rules / EPL** | **案件側が決定** | **[要決定]**（§9.6） |
| `c8y_Application_Down` / `_Unhealthy` | Cumulocity のマイクロサービス | **Cumulocity**（自動生成） | — | プラットフォーム管理 |

> ⚠️ **`x_Detection_<種別>` の `<種別>` 一覧は案件要件です。** 本チームは**型名の文法と必須フラグメントだけ**を規約とし、種別の列挙は案件側から受領します（§6.3.3 N-c）。

### 6.5 ペイロード規約

**[判] ローカル MQTT に publish する側（画像解析パイプライン・BOXアダプタ）が守るべき規約です。本チームの提供物になります。**

#### 計測 [確]

| # | 規約 |
|---|---|
| 1 | キーは**英数字と `_` のみ。先頭 `_` は不可** |
| 2 | 値は**数値のみ**。文字列・真偽値・オブジェクトは不可 |
| 3 | **ネストは 1 段まで** |
| 4 | 自由形式の付帯情報は **`properties` サブオブジェクト**に入れる |
| 5 | `time` はルート直下のみ。**ISO 8601 文字列 または unix 秒** |

```json
{ "time": "2026-08-20T10:00:00+09:00", "rtt": 12.4, "reachable": 1 }
```

#### イベント [確]

| # | 規約 |
|---|---|
| 1 | `text` と `time` は任意。**それ以外のフィールドはすべてカスタムフラグメント**として Cumulocity に載る |
| 2 | 本構成の必須フラグメント: **`x_Detection`**（§6.4） |
| 3 | ⚠️ **1 イベントのペイロードは 16KB 未満**（§6.7） |

```json
{
  "time": "2026-08-20T10:00:00+09:00",
  "text": "侵入検知: 東門エリア",
  "x_Detection": {
    "eventUuid": "0198c1d4-....",
    "modelVersion": "2026.06-r3",
    "aiProduct": "intrusion-v2",
    "confidence": 0.92,
    "boundingBoxes": [ { "x": 120, "y": 240, "w": 80, "h": 160 } ],
    "clipHint": "gsc-cam-8842@2026-08-20T10:00:00+09:00"
  }
}
```

#### アラーム [確]

| # | 規約 |
|---|---|
| 1 | `severity`: `critical` / `major` / `minor` / `warning` |
| 2 | `text` と `time` を必ず入れる |
| 3 | **retain 必須・QoS 2** |
| 4 | クリアは**同一トピックへの空の retained メッセージ**。retain を付けないとクリアされない [確] |

> ⚠️ **thin-edge 側は「型 + severity」を一意のアラームとして扱います** [確]。*"Every alarm is uniquely identified by its type and severity."* **severity を変えると別のアラームになる**ため、昇格させる場合は**旧 severity のアラームを別途クリアする**必要があります → §9.4

### 6.6 retain / QoS 規約（まとめ）

| データ種別 | retain | QoS | 誤ると何が起きるか |
|---|---|---|---|
| 計測 | **禁止**（[判]・下記） | 0 or 1 | retain すると再接続のたびに同じ計測が再送される |
| イベント | **禁止**（[判]・下記） | 1 | 同上 |
| **アラーム** | **必須** | **2** | **retain しないとアラームが上がらない／クリアできない** |
| **アラームのクリア** | **必須**（空メッセージ） | **2** | **クリアされずアラームが残り続ける** |
| ツイン | 必須 | — | 反映されない |
| サービス死活 | 必須 | — | 反映されない |

> ⚠️ **アラームの retain 要件は、実装者が最も間違えやすい点です**（[担当範囲] §7.4）。**IF 仕様書に太字で書き、疎通試験の必須項目にしてください。**

> **確度の内訳**: アラームの `retain` 必須と `QOS > 1` は公式に逐語で明記されています [確]（*"…with **QOS > 1** to ensure guaranteed processing"* — [Raise an alarm](https://thin-edge.github.io/thin-edge.io/start/raise-alarm/)）。
> 一方、**計測・イベントの retain「禁止」を明文で述べた一次情報は見つかりませんでした**（thin-edge の MQTT API・設定ファイル・FAQ を全文検索）。retain の意味論から導かれる **[判]** として扱ってください。規約としては維持すべきですが、「公式が禁止している」とは書かないでください。

### 6.7 ⚠️ サイズ制約 — 16KB の壁

| 経路 | 上限 | 超えたときの挙動 | 確度 |
|---|---|---|---|
| **ローカル MQTT（計測）** | `c8y.mapper.mqtt.max_payload_size` = **16,184 バイト** | **拒否される（破棄される）。ただし `te/errors` に出る** | **[確]** |
| **ローカル MQTT（イベント）** | 同上 | **HTTP に自動で切り替わる** | [確] |
| **ローカル MQTT（アラーム）** | 同上 | **拒否される** | **[確S]** |
| **REST（添付バイナリ）** | 既定 **50MiB**（チャンク 5MiB） | エラー | [確] |

**16,184 バイトは Cumulocity 側の制限です** [確]:

> *"For all Core MQTT connections to the platform, the maximum accepted payload size is **16184 bytes (16KiB)**, which includes both message header and body."*

thin-edge 側もこれに合わせています（`tedge_config.rs` の `C8Y_MQTT_PAYLOAD_LIMIT = 16184`）。rev.1 は [確S]（ソースのみ）としていましたが、**公式ドキュメントに逐語根拠があるため [確] に格上げ**しました。

> ⚠️⚠️ **判定は「変換後」のサイズです** [確]: *"Any message with a larger payload (**after transformation**) will be rejected."*
> thin-edge JSON → Cumulocity JSON の変換で**ペイロードは数倍に膨らみます**（例: `{"temperature":25}` → `{"temperature":{"temperature":{"value":25.0}}}`）。**ローカルでの文字列長チェックだけでは不十分**です（S-d を参照）。

> ⚠️ **アラームにも 16KB 制限が適用されます** [確S]（2.0.1 の `flows.rs` の `alarms_flow()` にも `limit-payload-size` ステップがある）。rev.1 の表はアラーム行を欠いていました。

> ⚠️⚠️ **16KB 制約と 50MiB 制約は別レイヤの話です。** 16KB は **MQTT トピック投入経路にのみ適用**され、REST（`tedge upload c8y` / c8y-proxy 直叩き）経路には及びません。混同しないでください。

> ⚠️ **イベントが 16KB を超えると HTTP 経路に切り替わり、ストア&フォワードの対象外になります** [確S]。**網断中はそのイベントが失われます。**（「HTTP はストア&フォワード対象外」を明文で述べた docs は無く、根拠は `events.rs` のコメントです。）

> **拒否は検知できます。** 既定フローの `measurements.toml` に `[errors.mqtt] topic = "te/errors"` が定義されており、**サイズ超過で捨てられたメッセージは `te/errors` に出ます**。rev.1 の「気づけない」は誤りでした。**[判] `te/errors` の購読を監視要件に入れてください**（§12.7）。

**[判] 規約**:

| # | 規約 |
|---|---|
| **S-a** | **1 イベントのペイロードは 16KB 未満とする** |
| **S-b** | **`boundingBoxes` の件数上限を定める**（例: 32 件）。混雑シーンで検知数が増えると容易に超える |
| **S-c** | **超えた場合の挙動を規約で決める**（切り詰めるか、別イベントに分けるか） |
| **S-d** | **発行側でサイズを検証してから publish する。ただし「変換後」のサイズで判定すること。** ローカルの thin-edge JSON の文字列長では過小評価になる。**併せて `te/errors` を購読し、実際に拒否されたら検知できるようにする**（二重の防御） |

### 6.8 添付（スナップショット）の扱い

**構成図の「スナップショット: 初期版は Cumulocity イベント添付のみ」に対応します。**

| 項目 | 内容 |
|---|---|
| **保存先** | Cumulocity のイベント添付（**Operational Store と同一 MongoDB・同一保持期間で消える**） |
| **経路** | c8y-proxy 経由の REST。**MQTT ではない** |
| **推奨手段** | **`tedge upload c8y`**（イベント作成 + 添付 + イベント ID 取得を 1 コマンドで行う）[確] |
| **制約 1** | ⚠️ **これは `tedge upload c8y` の制約であって、プラットフォームの制約ではありません。** `tedge upload c8y` は必ず新規イベントを作りますが、Cumulocity 側には **`POST /event/events/{id}/binaries`**（*"**Attach a file to a specific event** — Upload a file (binary) as an attachment of a specific event by a given ID."*）があり、**c8y-proxy 経由で既存イベントへの後付け添付は可能**です [確] |
| **制約 2** | **1 イベント 1 バイナリ**（2 件目は HTTP 409） |
| ⚠️ **制約 3** | **網断耐性なし**。HTTP 経路は**ストア&フォワードの対象外**。**発行側のローカルスプールが必須** |
| ⚠️ **制約 4** | c8y-proxy は *"requests to the proxy API are occasionally spuriously rejected with a `401 Not Authorized` status code"* [確] → **リトライを実装規約に入れる** |
| ⚠️⚠️ **制約 5** 〈rev.3〉 | **添付画像の「参照・ダウンロード」は監査ログに残りません** [確]。Cumulocity の監査ログが記録するのは *"security-relevant operations a user has processed"* で、**その対象操作は公式ドキュメントの「Audit log types」表に列挙されており、イベント添付・バイナリの取得はそこに含まれていません**。⚠️ **「参照はいっさい監査されない」わけではありません** — 同じ表に `Report` → *"Test tenant statistics accessed"* / *"Real tenant statistics accessed"* があり、**Cumulocity は一部の参照操作を監査しています**。**画像取得 API がその一覧に無い**、というのが制約 5 の正確な内容です。**Edge 固有の制限ではなくプラットフォーム共通です**（共通 docs に Edge 例外の記述が無いことによる） |
| ⚠️⚠️ **制約 6** 〈rev.3〉 | **添付の取得権限をイベント閲覧権限から分離できません** [確]。`GET /event/events/{id}/binaries` の Required roles は *"ROLE_EVENT_READ **OR** EVENT_READ permission on the source"* で、**イベント本体の参照とまったく同じ**です。**§11.3 の R-04a / R-04b（Events READ）を持つ利用者は、UI を経由せず REST で直接スナップショットを取得できます**（→ **CT-5b**） |
| **将来** | オブジェクトストレージ本体 + サムネイル添付へ移行（構成図の配置の要点）。⚠️ **この「将来」は CU-18 案 (A) と同じではありません** 〈rev.3〉 — **サムネイルが Cumulocity に残る構成**なので、**サムネイルに顔が写るなら個人情報は Cumulocity 内に残り、制約 5・6 は解消しません**。マスクを伴う案 (A′) が「将来」＋マスクに相当します |

> ⭐ **[判] 制約 1 が外れたことで、網断耐性の設計が 1 段良くなります。**
> rev.1 は「イベントと添付は不可分」という前提から、**添付が失敗するとイベントごと失われる**設計になっていました。後付け添付が可能である以上、次の分離が取れます。
>
> 1. **イベント本体は MQTT で送る**（ストア&フォワードが効く。網断中もキューに載る）
> 2. **スナップショットはローカルにスプールし、復旧後に `POST /event/events/{id}/binaries` で後付けする**
>
> これにより「網断中に検知イベントそのものが消える」ことがなくなります。**[要 SV-42] この方式を採るか（＝発行側にイベント ID とファイルの対応を保持させるか）を [担当範囲] §6.6 と決めてください。**

> **[確K] リテンションは添付バイナリにも及びます。** Cumulocity Tech Community に *"Yes, that is one of the main reasons why binaries should be attached to events…"*（[Event file binary attachment retention rules](https://community.cumulocity.com/t/event-file-binary-attachment-retention-rules/14387)）との回答があります。rev.1 は SV-06 を「公式に記載がない」としていましたが、**上表の「保存先」欄が既に「同一保持期間で消える」と断定しており、文書内で矛盾していました。**
>
> ⚠️ ただし **`files` リポジトリ（ソフトウェアリポジトリ等）にはリテンションが及びません** [確]（*"Retention rules do not apply to files stored in the files repository."*）。**イベント添付とファイルリポジトリを混同しないでください**（§7.4）。

> **[判] 添付を出すのは画像解析パイプラインのみです。** BOX はスナップショットを出しません（**[要 TE-9] N2 で確認**）。

#### ⚠️⚠️ スナップショットが個人情報を含む場合 〈rev.3 で追加〉

**スナップショット画像・サムネイル画像に人の顔が判別できる形で写り込む場合、上の制約 5 と制約 6 が同時に効きます。**

> **「Cumulocity Edge 上で画像を表示・ダウンロードした事実を証跡として残す」は、素の Cumulocity では実現できません。**
>
> - **制約 5** により、参照は監査ログに記録されない
> - **制約 6** により、「イベントは見せるが画像は見せない」という権限分離もできない
>
> この 2 つが重なるため、**UI 側にログ出力を仕込んでも REST の直叩きで迂回されます。迂回できるものは監査証跡になりません。**
>
> ⚠️ **EPL / Streaming Analytics・スマートルール・Notification 2.0 のいずれも代替になりません。** これらが扱うのは Measurement / Event / Alarm / Operation の**発生**であって、**API の読み取りをフックする仕組みは持ちません**（§8 の仕組みでは代替不能）。

> ⭐ **[確] ただし「書き込み側」は開いています。**
>
> Audit API には **`POST /audit/auditRecords`** があり（Required roles: *"ROLE_AUDIT_ADMIN **OR** ROLE_SYSTEM **OR** AUDIT_ADMIN permission on the resource"*）、**自前の監査レコードを投入できます**。監査レコードは *"persistently stored and **cannot be externally modified after being written**"* [確] なので、証跡としての性質（改竄不能）は満たします。
>
> 加えて **Edge は `Microservice Hosting` が `Included`** [確]（§12.4）。**したがって「画像へのアクセスを自作経路に一本化し、そこで監査レコードを書く」なら実現可能です。**

> ⭐ **[確] 制約 6 は「イベント添付」に固有です。Cumulocity 全体の性質ではありません** 〈rev.3 で追加〉。
>
> **`/inventory/binaries`（ファイルリポジトリ）に格納すれば、標準の権限モデルのまま「イベントは見えるが画像は 403」を作れます。** 必要ロールの構造が違うためです:
>
> - `GET /event/events/{id}/binaries` → `ROLE_EVENT_READ` **OR** `EVENT_READ` permission **on the source**（＝ **カメラ**。拠点グループ配下なので Inventory ロールで届いてしまう）
> - `GET /inventory/binaries/{id}` → `ROLE_INVENTORY_READ` **OR** `ROLE_BINARY_READ` **OR** **owner of the resource** **OR** `MANAGE_OBJECT_READ` permission **on the resource**（＝ **バイナリ MO 自身**。**拠点グループ配下に置かなければ届かない**）
>
> ⭐ **さらに「1 イベント 1 バイナリ」（制約 2）も外れる**ため、**原本とサムネイルの 2 本**を同じイベントに紐づけられます。
>
> ⚠️⚠️ **ただし、ファイルリポジトリにはリテンションが及びません** [確]（§7.4）。**個人情報を含む画像が無期限に残るため、明示的な削除バッチが必須になります。** また**バイナリ 1 件 ＝ MO 1 件**なので §7.7 の MO 総数（SV-14b）に直撃します。
>
> **この案は §15.3 CU-18 の 案 (D) として比較表に入れています。403 になることは仕様からの推論なので、CT-5c / CU-23 で実機確認してください。**

> ⚠️ **格納先を変えても制約 5 は解消しません。** `/inventory/binaries` に移しても**参照・DL が監査ログに残らないことは変わりません**。**監査レコードを自前で書く必要性は、どの案を採っても消えません。**

> ⚠️ **ただし、そのためには「画像の置き場所」そのものを決め直す必要があります。**
> 選択肢は「オブジェクトストレージへ全部出す（(A)）」「マスク済みサムネイルだけ残す（(A′)）」「`/inventory/binaries` に移す（(D)）」「暗号化して添付に残す（(B)）」「監査を諦めて人を絞る（(C)）」の 5 つです。**[要決定] CU-18（§15.3）** として起票しました。**この決定が済むまで、§11.4 のサービスユーザーと §13 の投入対象は確定できません。**
>
> ⚠️ **本節の「将来」欄は (A) ではなく (A′) に近い**ことに注意してください（サムネイルが Cumulocity に残るため）。

> **否定の探索経路**（制約 5 の根拠）〈rev.3 レビューで修正〉:
>
> 1. 公式 docs の Audit logs ページ（[2025 版](https://cumulocity.com/docs/2025/standard-tenant/audit-logs/) / [2024 版](https://cumulocity.com/docs/2024/standard-tenant/audit-logs/)）の **「Audit log types」表を全 type・全 action 逐一確認** → イベント添付・バイナリの取得に該当する項目は無い。⚠️ **ただし `Report` → *"…statistics accessed"* という参照系の項目は存在する**（＝「read 系は 1 つも無い」は誤り）
> 2. Core OpenAPI 全文（`https://cumulocity.com/api/core/dist/c8y-oas.yml`・2026-08-23 取得）で `auditRecord` スキーマと `/audit/auditRecords` を検査 → **読み取りをフックする API は存在しない**
> 3. Notification 2.0 が購読できる API を確認 → *"alarms, alarmsWithChildren, events, eventsWithChildren, managedobjects, measurements and operations"* の **7 種のみ。binaries も読み取りフックも無い**（制約 5 の代替手段が無いことの spec 上の裏付け）
> 4. WebSearch で docs / Tech Community を「audit log + read / download / GDPR」で探索 → 該当記述なし
>
> **この 4 経路をもって「画像の参照・DL は監査ログに残らない」と判断しています。** ⚠️ **rev.3 初出時は「`auditRecord.type` の enum に read 系の値が無いから」という理由付けをしていましたが、これは誤りでした**（`Report` が反例）。**理由は「画像取得 API が監査対象操作の一覧に含まれていないから」です。**

### 6.9 時刻の規約

| # | 規約 | 理由 |
|---|---|---|
| **TM-a** | **`time` は「事象の発生時刻」。受信時刻ではない** | 再送時も同一値であることが重複判定の前提 |
| **TM-b** | **ISO 8601（タイムゾーン付き）または unix 秒** | [確] |
| **TM-c** | ⚠️ **全ノードで NTP を必須にする**（Edge ホスト・画像解析装置・外部Gateway・Keycloak） | **時刻ずれは「TLS の有効期限判定」「JWT の `exp`/`iat` 検証」「イベントと映像クリップの突合」を同時に壊し、エラーから原因に辿り着けなくなる**（IF-P03 / P-0-3） |
| **TM-d** | **`time` が無い場合は thin-edge が「ローカル受信時刻」を付ける** | 補完された時刻は**発行側の意図した事象時刻とは限らない**。**発行側が必ず `time` を入れること** |

> ⚠️⚠️ **rev.1 の TM-d は主体を取り違えていました。** 時刻を補うのは Cumulocity ではなく **thin-edge** で、しかも**クラウドへの送信時ではなくローカルでの受信時**です [確]:
>
> > *"When thin-edge.io receives a measurement, it will **add a timestamp to it before any further processing**."* / *"when not provided, thin-edge.io uses the current system time"*（[Thin Edge JSON](https://thin-edge.github.io/thin-edge.io/understand/thin-edge-json/)）
> >
> > *"The default value for the time fragment will be the timestamp in utc time that is **added by the tedge-mapper-c8y**"*（[Raise an alarm](https://thin-edge.github.io/thin-edge.io/start/raise-alarm/)）
>
> 既定フローの第 1 ステップが `{ builtin = "add-timestamp", … }` であることからも確認できます。
>
> **したがって「網断復旧後の一括再送で全イベントが復旧時刻になる」という失敗モードは、本構成（thin-edge 経由）では起きません。** 時刻はローカルで確定するため、キューに滞留しても値は変わりません。

> **それでも `time` を必須にする理由は変わりません**:
>
> 1. **thin-edge が付けるのは「thin-edge が受け取った時刻」**であり、カメラ・BOX が事象を検知した時刻ではない。BOXアダプタの遅延・バッチ送信ぶんだけずれる
> 2. TM-a のとおり **再送時に同一値であること**が重複判定の前提になる（`time` が無いと再送のたびに値が変わりうる）
> 3. **thin-edge を経由しない経路**（REST 直叩き・将来の別経路）では Cumulocity 側の受信時刻になる
>
> **IF 仕様書で `time` を必須項目にしてください。** ただし理由は「Cumulocity が復旧時刻を付けるから」ではなく上記 1〜3 です。**理由が誤ったままだと、レビューで規約ごと否定されます。**

### 6.10 データフロー（まとめ図）

```
拠点                          閉域網            本社 VM1 / Cumulocity Edge
──────────────────────       ──────           ─────────────────────────────

IPカメラ ──ONVIF/ICMP──┐
                       │
画像解析パイプライン ──┤
  検知 + スナップ      │  ① te/device/<cam>///e/x_Detection_*   ┐
                       │  ① te/device/<cam>///m/x_CameraHealth  ├─ MQTT 8883 ─┐
ONVIF死活監視 ─────────┤  ① te/device/<cam>///a/x_CameraDown    ┘             │
                       │                                                       ▼
sm-plugin ─────────────┘  ① te/device/main///e/x_ModelApplied         REST/MQTT API
                       ▼                                                       │
              thin-edge A（main: x_ImageAnalyzer）                             ▼
                       │  ② SmartREST（s/us 等）                       Operational Store
                       └── REST（c8y-proxy）── スナップショット添付 ───►（〜90日・§7）
                                                                               │
                                                                               ├─► Smart Rules / EPL（§8）
画像センシングBOX ──独自形式──► BOXアダプタ                                    │      │
                                  │ ① te/device/<box>///e/x_Detection_*        │      ▼
                                  ▼ ① te/device/<box>///a/x_BoxSilent          │  x_Alarm_<種別>
                        thin-edge B（main: x_BoxGateway）─── MQTT 8883 ────────┤      │
                                                                               │      ▼
                                                                               │  Notification 2.0（§10）
                                                                               │      │
                                                                               ▼      ▼
                                                                        オフロード  案件アプリ（VM2）
                                                                        （§7.5）
```

---

## 7. データ保持・オフロードと容量設計

### 7.1 Operational Store の位置づけ

**構成図の配置の要点に明記されています**: 「Operational Store = 短期運用データ（〜90 日目安）／オブジェクトストレージ = 長期保存。オフロードはその移送」。

| # | 原則 | 理由 |
|---|---|---|
| **DS-a** | **Operational Store は短期運用データのみ。長期保管には使わない** | MongoDB を長期保管に使うと §7.7 のスケール制約に直撃する |
| **DS-b** | ⚠️ **分析に 90 日超が必要でも、リテンションを延ばすのは誤った解決** | **オフロードを先に成立させ、長期分析はオフロード先を読む** |
| **DS-c** | **Operational Store への直接アクセスは不可。REST API 経由のみ** | 構成図の配置の要点 |

### 7.2 リテンション設計

| データ種別 | **[判] 保持日数** | 根拠 |
|---|---|---|
| `EVENT` | **90 日** | 構成図の「〜90 日目安」 |
| `MEASUREMENT` | **90 日** | 同上 |
| `ALARM` | **90 日** | 同上。⚠️ **`CLEARED` のもののみ削除される**（§7.3） |
| `OPERATION` | **90 日** | 同上 |
| **`AUDIT`** | **[要決定]**（イベントとは別基準） | **①設定変更の追跡・②障害調査要件**（§4.10 G-13）、および 〈rev.3 で追加〉⭐ **③個人情報アクセス記録の保存義務期間**（§6.8）から逆算する。⚠️ **③が最も長くなる可能性が高く、支配的になります** |

**Cumulocity の既定と制約** [確]:

| 事実 | 帰結 | 確度 |
|---|---|---|
| **既定では全履歴データが 60 日で削除される** — *"By default, all historical data is deleted after 60 days (**configurable in the system settings by the platform administrator**)"* | 90 日にするには明示的な設定が要る。**設定漏れ＝60 日で消える** | [確] |
| ⚠️⚠️ **その 60 日が「削除できるリテンションルールのオブジェクト」として存在するかは未確認** | **§13.3.6 の手順③（旧ルールを個別に削除）が空振りする可能性がある** | **[要 SV-43]** |
| 上限は 10 年 | — | [確] |
| **`files` リポジトリには適用されない** | ソフトウェアリポジトリは別途削除運用が要る（§4.4 論点 2） | [確] |
| **`ALARM` には `fragmentType` による絞り込みが効かない** | アラームはフラグメント単位で保持期間を変えられない | [確] |

> ⚠️⚠️ **[要 SV-43] リテンション設計の前提が 1 つ未確認のまま残っています。**
> 「60 日」という既定値は公式に明記されていますが、**それが `GET /retention/retentionRules` で列挙・削除できるオブジェクトなのか、それともシステム設定側の値なのかを述べた一次記述が見つかりません**（docs / OpenAPI / Tech Community の 3 経路で探索）。
>
> **もしシステム設定側の値なら**、90 日のルールを追加しても**システム設定の 60 日が先に効いて 90 日目のデータが存在しない**可能性があります。**W1 の最優先で実機確認してください。** 確認方法:
>
> 1. `c8y retentionrules list` で既定ルールが列挙されるか
> 2. されない場合、`GET /tenant/system/options` で保持日数に相当するキーを探す
> 3. 検証環境で短い保持日数を設定し、**実際にどちらが効くか**を観測する

#### ⚠️ 適用順序 — 「全削除 → 再作成」は危険

> **[判] 手順を反転します。**
>
> 1. 現行ルールを **GET してタグ付き Git コミット**
> 2. **新ルールを先に作成**
> 3. 旧ルールを個別に削除
> 4. `--dry` で事前確認、全コマンドに `--session` を明示
>
> **理由**: 削除が成功して作成が失敗すると、**リテンションルールがゼロの状態で放置**され、誰も気づかないまま Operational Store が単調増加します。

> ⚠️ **保持日数を短縮する方向の変更は不可逆です**（削除されたデータは戻りません）。**別承認フローを通してください。**

> ⚠️ **`AUDIT` の日数が未決のままリテンション投入を実行しないでください。** 既定ルールを消すと再作成すべき値が必要になります。

> ⭐ **[判] `AUDIT` の日数を決める入力に「個人情報アクセス記録の保存義務期間」を追加してください** 〈rev.3 で追加〉。
>
> スナップショット画像が個人情報を含む場合、**「その画像を誰がいつ見たか」を自前の監査レコードとして残す**設計になり得ます（§6.8 / **CU-18**）。**この記録も `auditRecords` に入るため `AUDIT` リテンションの対象**であり、**障害調査要件から逆算した日数では足りない可能性があります。**
>
> **CU-5（§15.2）の決定主体に「法務」を追加してください。** 保存義務期間は社内規程・契約・適用法令から決まるもので、基盤側では決められません。
>
> ⭐ **[確] 「画像アクセス記録だけ別の保持期間にする」ことは可能です** 〈rev.3 レビューで判明。CU-22 は解消〉。
>
> `retentionRule` スキーマに絞り込みキーの適用範囲が明記されていました:
>
> | キー | 適用される dataType | `AUDIT` で使えるか |
> |---|---|---|
> | `fragmentType` | *"Used by the data types **EVENT, MEASUREMENT, OPERATION and BULK_OPERATION**"* | ❌ **使えない** |
> | **`type`** | *"Used by the data types **ALARM, AUDIT, EVENT and MEASUREMENT**"* | ⭐ **使える** |
> | **`source`** | *"Used by **all data types**"* | ⭐ **使える** |
>
> **したがって、自前の監査レコードに専用の `type` 値を割り当てれば、`dataType=AUDIT` + `type=<画像アクセス用>` で長期、`dataType=AUDIT` + `type=*` で短期、という 2 本立てにできます。** 「個人情報の保存義務期間が `AUDIT` 全体を支配する」という懸念は回避できます。
>
> ⚠️ **これは監査レコードの `type` の選び方に跳ね返ります**（§15.3）。`auditRecord.type` は 20 値の固定 enum なので、**Cumulocity 自身が使っている値（`Event` など）を流用すると、この分割ができません。**

> ⚠️⚠️ **`AUDIT` のリテンションルールは「証跡を狙い撃ちで消す」経路にもなります** 〈rev.3 レビューで発見〉。
>
> §11.2 の **R-01 基盤運用者は Retention rules ADMIN を持ちます**。`dataType=AUDIT` + `type=<画像アクセス用>` + `maximumAge=1` のルールを 1 本入れれば、**画像アクセスの証跡だけを消せます**（`source` 指定で特定カメラ分だけも可能）。**「Audit API に `PUT`/`DELETE` が無いので改竄不能」という性質は、リテンション経路を勘定に入れていませんでした。**
>
> ⭐ **対策が spec 側にあります**: `retentionRule.editable` は *"Indicates whether the rule is editable or not. **It can be updated only by the Management tenant.**"* [確]。**`AUDIT` のルールを `management` テナントから `editable: false` で作成すれば、`edge` テナントの R-01 では変更できません。** §13.3.6 の投入手順に反映してください。あわせて **AS-11b** で「`dataType=AUDIT` のルールが想定どおりの本数・`maximumAge` であること」を CI 検査します。

### 7.3 ⚠️ アラームは `CLEARED` のものしか削除されない

**これは本構成で最も見落とされやすい制約です** [確]。

```
アラームが ACTIVE のまま        → リテンション 90 日を過ぎても削除されない → 単調増加
アラームが CLEARED になっている → 90 日で削除される
```

**[判] したがって、全アラーム型についてクリア条件を定義することが必須です**（§9.6）。クリア条件を持たないアラーム型を作らないでください。

| アラーム型 | クリアの主体 | 状態 |
|---|---|---|
| `c8y_UnavailabilityAlarm` | **Cumulocity（自動）** | ✅ |
| `x_CameraDown` | thin-edge の監視サービス | ✅ |
| `x_BoxSilent` | BOXアダプタ | ✅ |
| `x_ModelUpdateFailed` | thin-edge の sm-plugin（再適用成功時） | ✅ |
| **`x_BoxParseError`** | **[要決定]** | ⚠️ デッドレター処理後に手動クリアか、時限自動クリアか。**⚠️ 時限自動クリアに使える組み込みスマートルールは存在しません**（§8.2）。EPL で実装する前提で決めること |
| **`x_Alarm_<種別>`** | **[要決定]** | ⚠️ 案件側が決定 |

> ⚠️ **`ACKNOWLEDGED` も削除されません。** 削除されるのは `CLEARED` のみです（*"Alarms are only removed if they have a status of CLEARED."*）。運用者が「確認済み」にしただけでは Operational Store から消えないため、**運用手順で「確認したらクリアまで行う」ことを明示してください**（§10.6）。

> ⚠️ **リテンションルールの `fragmentType` は `ALARM` には効きません。** アラームの削除条件は `type` と status のみで絞られます。**フラグメント単位で保持期間を変えたい場合、アラームでは実現できません**（§7.2 の設計に反映済みか確認すること）。

### 7.4 添付バイナリ

| 項目 | 状態 |
|---|---|
| リテンションが**イベント添付**に及ぶか | **及ぶ** [確K]（§6.8）。Tech Community に *"Yes, that is one of the main reasons why binaries should be attached to events…"*。**SV-06 は解消** |
| リテンションが **`files` リポジトリ**に及ぶか | ⚠️ **及ばない** [確]（*"Retention rules do not apply to files stored in the files repository."*）。**ソフトウェアリポジトリ（AI モデル）は自動削除されない** → §4.4 論点 2 の世代管理が必須 |
| サイズ | 既定 50MiB（チャンク 5MiB） |

> ⚠️ **「添付」と「ファイルリポジトリ」で挙動が正反対です。** スナップショット（イベント添付）は放っておけば消えますが、**AI モデル（ファイルリポジトリ）は放っておくと永久に残ります**。§4.4 論点 2（[要 TE-20] 旧世代の削除運用）は、この非対称ゆえに必須の運用項目です。

> ⚠️⚠️ **CU-18 で案 (D)（スナップショットを `/inventory/binaries` に格納）を採る場合、この非対称が画像にも適用されます** 〈rev.3 で追加〉。
>
> **格納先をイベント添付からファイルリポジトリに移した瞬間、「90 日で自動的に消える」という前提が失われます。** 個人情報を含む画像が**無期限に残る**ことになるため、**明示的な削除バッチが必須**です。
>
> ⚠️ **これは §4.4 論点 2（AI モデルの世代管理・TE-20）と同型の問題ですが、件数が桁違いです**（検知件数 × 2 本 ＝ 原本 + サムネイル）。**個人情報の保存期間を守る唯一の手段が削除バッチになる**ため、**「バッチが止まったことを検知する監視」までを設計に含めてください**（**§12.7 のメタ監視**・SV-23 と併せて検討）。
>
> ⚠️ **監視条件は「バッチの成否」ではなく「最古バイナリの年齢」にしてください** 〈rev.3〉。バッチが「成功して 0 件削除」でも異常は起こりえます。**`GET /inventory/binaries?type=<画像用 type>` の最古 `creationTime` が保存期間を超えていないこと**を直接の判定条件にしてください（§12.7 ⑪）。
>
> ⚠️⚠️ **バックアップには及びません。** 削除バッチで消しても、§12.6 の Edge バックアップ（`/var/lib/rancher/k3s` 丸ごと）には画像が残ります。**保存期間の遵守をバックアップまで含めて主張するなら、§12.6 の保持世代・破棄手順が前提になります**（§12.6 の [要決定]）。
>
> **判断材料**: 案 (D) は**権限分離という利点**と**リテンションが効かないという欠点**を同時に持ちます。§15.3 CU-18 の比較表で並べています。

### 7.5 オフロード（長期保存）

**Data Hub は Edge では *"On request via Professional Services"*** です。これは技術的制約ではなく**商流の制約**ですが、コスト・リードタイム的に本フェーズでは非現実的です。

| 選択肢 | 前提 | **[判] 評価** |
|---|---|---|
| **(A) Notification 2.0 で購読 → 独自コンシューマ** | Messaging Service は 2026 Edge に `Included` | **通知のために Notification 2.0 はどのみち稼働させる**ので追加コストは小さい。ただし **`measurements` を `tenant` 文脈で購読できない**（§10.2） |
| **(B) REST ページングで定期エクスポート** | なし | **✅ 推奨（初期版）**。構成図の「定期エクスポート」とも整合。実装が単純で失敗モードが読みやすい。**計測値を含めるなら (B) が必要** |

#### ⚠️ (B) の実装規約

> **`acl.algorithm-version` の `OPTIMIZED` は、一致件数が閾値（既定 2000）未満のときだけ適用**され、超えると `LEGACY` に落ちます [確K]。日次一括エクスポートはまさに 2000 件超になるため、**`prev` / `next` リンクで辿ることを実装規約にしてください**（*"navigation links via 'prev' and 'next' will work properly and this should be the only way of iterating through multiple pages"*）。**ページ番号を自前で加算する実装は壊れます。**

#### ⚠️ オフロードは「あとで実装する保管機能」ではない

> **オフロードは「アーカイブ」ではなく「運用知識基盤のデータソース」です。優先度を「初期に成立させる機能」へ引き上げてください。**
>
> **稼働開始から 90 日以内に成立していないと、最初のリテンション削除で長期データが恒久的に失われます。**

**オブジェクトストレージへの流入は 2 系統**: ①Cumulocity のオフロード ②GSC の映像クリップ + サイドカー JSON（S3 互換）。**突き合わせには external ID の安定性が前提**です（§2.2）。

### 7.6 容量サイジング

> ⚠️ **install 後に変更できないのは `storageClassName` だけです** [確]。**MongoDB 容量は「増やす」ことができます**:
>
> > `spec.mongodb.resources.requests.storage`: *"If not provided, it defaults to 75GB. **Once Edge is installed, you can only increase this value, but cannot reduce.**"*
> > `spec.storageClassName`: *"This value is used only during the Edge installation and **can't be changed for existing installations**."*
> > — [Edge custom resource definition](https://cumulocity.com/docs/2026/edge/edge-custom-resource-definition/)
>
> **rev.1 は MongoDB 容量も変更不可としていたため、TE-22（サイジング手法の確立）を不必要にクリティカルパスへ載せていました。** 初期値は余裕を持たせつつ、**「足りなければ増やせる／減らせない」という非対称を前提に、やや小さめから始めて増設する**運用も選べます。**縮小はできない**点だけ守ってください。

**[判] 算出手順**（**[要 TE-22]** 手法の確立が必要）:

```
① 受信レートの試算
   死活計測  : 拠点数 × カメラ台数 ÷ 死活間隔[秒]           （§5.9）
   検知イベント: 拠点数 × 想定検知数/日 ÷ 86400              （案件要件）
   ────────────────────────────────────────
② 1 レコードの平均バイト数
   計測      : 数百バイト
   イベント  : 最大 16KB（§6.7）
   添付      : スナップショット 1 枚あたりのサイズ × 検知件数
   ────────────────────────────────────────
③ 必要容量 ≒ ① × ② × リテンション日数 × 安全率
   ＋ 監査ログ（AUDIT の保持日数 × 変更件数）
   ＋ ソフトウェアリポジトリ（モデルサイズ × 保持世代数）
   ＋ Messaging Service のバックログ（§10.5）
```

**確定に必要な入力値**（すべて **[要]**）:

| 入力 | 対応する未確定事項 |
|---|---|
| 拠点数・カメラ台数・BOX 台数 | 案件要件 |
| 死活間隔の確定値 | SV-26（§5.5） |
| 検知イベントのレート | 案件要件 |
| スナップショットの平均サイズと保存方針 | §6.8 / SV-06 |
| リテンション日数（特に `AUDIT`） | §7.2 |
| モデルの保持世代数 | TE-20 |

### 7.7 スケール上限

| 指標 | 値 | 確度 |
|---|---|---|
| 比較表の目安 | **約 100 tps / CPU コア** | [確]（[Edge introduction](https://cumulocity.com/docs/2026/edge/edge-introduction/) の `Vertical scalability` 行に *"Yes, limited to appr. 100 tps per CPU core"*） |
| Narrow（10 クライアント）8 CPU スレッド / 16GB | 25,000 measurement/s | [確] |
| Narrow 16 CPU スレッド / 32GB | 47,500 measurement/s | [確] |
| **Wide（各 1 measurement/s）8 CPU スレッド / 16GB** | **1,200 接続クライアント** | [確] |
| **Wide 16 CPU スレッド / 32GB** | **2,200 接続クライアント** | [確] |
| ハードウェア最小要件 | CPU 8 コア / RAM 16GB / Disk 150GB。**MongoDB は AVX 命令 + x86-64-v3 以降が必須** | [確] |

> ⚠️⚠️ **公式の 2 つの数値は 30 倍以上食い違っています。** 比較表の「100 tps / CPU コア」に対し、Narrow ベンチマークは 8 スレッドで 25,000 measurement/s（≒ 3,100 / スレッド）です。**両方とも公式なので、どちらか一方をサイジングの根拠にしないでください。** ベンチマークページには *"for illustrative purposes only"* の Caution があり、単位も「CPU **threads**」（コアではない）です。[判断記録] D17 の「PoC 実測を正とする」がそのまま有効です。

> ⚠️ **rev.1 の「Wide シナリオが本構成に直撃する」は誤りでした。** Wide の律速は **同時接続 MQTT クライアント数**です（*"These end-to-end test scenarios drive a number of **MQTT clients** …"*）。
>
> 本構成で MQTT 接続を張るのは **main device（thin-edge インスタンス）だけ**で、child device（カメラ・BOX）は独自の接続を持ちません。**接続数は「拠点数 × 2」（案 β）のオーダー**であり、1,200 という上限には遠く及びません。**律速は Narrow 側（メッセージ毎秒）で評価してください。**
>
> **ただし child device 台数は別軸の制約として残ります**（**[要 SV-14b]**）: ①MO 総数 ②デバイス課金 ③inventory クエリ性能（§11.9 の 2000 件閾値）。**「接続数の問題ではない」ことと「台数が無制限」は別です。**
>
> **[要 SV-14] 全拠点合算の負荷試験は必須です**（§14 CT-28）。**「レート」「MO 総数」の両面で実測してください。**

> ⚠️ **案 α を採ると、外部Gateway の thin-edge が 1 プロセスで全拠点分のイベントを処理します**（**[要 TE-13]** child device 数の上限はドキュメントに記載なし）→ 付録 A。

---

## 8. 条件一致メッセージの処理

### 8.1 処理系の選択肢と使い分け

**Cumulocity Edge には 3 つの処理系があります。** 2026 Edge の機能比較表で `Included` と明記されているのは **`Streaming Analytics`** の 1 行です [確]（EPL Apps と Analytics Builder はその配下の機能で、スマートルールは Cockpit の機能）。

> ⚠️⚠️ **[要 SV-45] Edge が同梱する Apama-ctrl のバリアントが未確認です。これは本章の前提を丸ごと左右します。**
>
> Cumulocity には複数の Apama-ctrl バリアントがあり、**どれが入るかで使える処理系が変わります** [確]（[Streaming Analytics introduction](https://cumulocity.com/docs/streaming-analytics/introduction-analytics/)）:
>
> | バリアント | 使えないもの |
> |---|---|
> | `Apama-ctrl-starter` | *"The EPL Apps page is not available"* |
> | `Apama-ctrl-smartrules` | *"The Analytics Builder and EPL Apps pages are not available"* |
>
> **Edge 2026 のドキュメント全ページを `apama` / `epl` / `cep` で検索しても、どのバリアントが同梱されるかの記述はありません。**
>
> **本章の RL-b・RU-2〜RU-5 はすべて EPL 前提です。** EPL Apps が使えないバリアントだった場合、**§8 の設計の過半が成立しません**。§12.4（インストールと初期確認）で最優先に確認し、使えない場合の代替（スマートルール + マイクロサービス、または Analytics Builder）を先に決めてください。

| 処理系 | 実体 | 表現力 | 保守のしやすさ | **[判] 本構成での用途** |
|---|---|---|---|---|
| **スマートルール** | GUI で定義。テンプレート化された条件 → アクション | 低（単一イベント/計測に対する閾値・存在判定） | **高**（案件担当者が触れる） | **✅ 単純な閾値・存在判定によるアラーム生成と自動クリア** |
| **EPL Apps** | Apama の EPL。`POST /service/cep/eplfiles` で投入 | **高**（時間窓・相関・状態保持） | 低（コード） | **✅ 時間窓を使う判定（重複排除・拠点集約・リプレイ抑止）** |
| **Analytics Builder** | GUI のブロックエディタ。JSON でエクスポート/インポート | 中 | 中 | △ **初期版では使わない**。⚠️ **閉域網では拡張のインストールに事前準備が必要** |

**[判] 使い分けの規約**:

| # | 規約 | 理由 |
|---|---|---|
| **RL-a** | **単一メッセージで判定できるものはスマートルール** | 案件側が保守できる（D7） |
| **RL-b** | **時間窓・相関・状態保持が要るものだけ EPL** | EPL はコードなのでレビューと版管理が要る |
| **RL-c** | **メール／SMS／エスカレーション系のスマートルールテンプレートは使わない** | **変-2 によりメール不採用**。人への通知は案件アプリの責務（§10.6） |
| **RL-d** | **Analytics Builder は初期版のスコープ外** | 閉域網での拡張導入コストに対し、EPL とスマートルールで足りる |

### 8.2 本構成で置くルールの一覧

**[判] 基盤が雛形を提供し、判定ロジックは案件側が保守します（D7）。**

| # | ルール | 処理系 | 入力 | 出力 | 保守主体 |
|---|---|---|---|---|---|
| **RU-1** | **検知イベント → アラーム昇格** | スマートルール（単純な場合）／ EPL（時間窓を使う場合） | `x_Detection_<種別>`（`confidence` 等） | `x_Alarm_<種別>` | **案件側**（雛形は基盤） |
| **RU-2** | **アラームの自動クリア** | ⚠️ **EPL**（時限クリアに使える組み込みスマートルールは存在しない・§8.3） | 各アラーム型 | CLEARED | **基盤**（§8.3） |
| **RU-3** | ⚠️ **リプレイ抑止ガード** | EPL | 全イベント | 抑止 | **基盤**（§8.4・**雛形に必須で組み込む**） |
| **RU-4** | **BOX 再送の重複排除の補助** | EPL | `x_Detection_*` の `eventUuid` | 重複の除外／検知 | ⚠️ **一次責務は BOXアダプタ**（§8.5） |
| **RU-5** | **拠点単位のロールアップアラーム** | EPL | `x_CameraDown` / `x_BoxSilent` の多発 | 拠点集約アラーム | 基盤（**案 α では必須**・付録 A） |
| **RU-6** | **D14 の自動クリップ保存** | — | `x_Alarm_<種別>` | `exportClip` 呼び出し | ⚠️ **ルールではなく Notification 2.0 のサブスクライバが実装**（§10.2） |

> ⚠️ **RU-6 は変-1 により設計が変わりました。** 以前は「EPL から HTTP を出す」案でしたが、**Notification 2.0 のサブスクライバが受けて `exportClip` を叩く**方式になり、Apama の HTTP クライアントプラグイン（SV-10）の可否は論点から外れました。

### 8.3 アラームの自動クリア

**§7.3 のとおり、リテンションは `CLEARED` のアラームしか削除しません。自動クリアを設計しないとアラームが単調増加します。**

| アラーム型 | クリア主体 | 実装 |
|---|---|---|
| `c8y_UnavailabilityAlarm` | **Cumulocity（自動）** | `c8y_RequiredAvailability` の設定が前提（§5.2）。設定していないデバイスでは発報もクリアも起きない |
| `x_CameraDown` | thin-edge の監視サービス | 空 retained メッセージ（§6.6） |
| `x_BoxSilent` | BOXアダプタ | 同上 |
| `x_ModelUpdateFailed` | thin-edge の sm-plugin | 同上 |
| **`x_BoxParseError`** | **[要決定]** | デッドレター処理後の手動クリアか、**EPL による時限自動クリア**か |
| **`x_Alarm_<種別>`** | **[要決定]** | 案件側が決定。**時限自動クリア（例: 24 時間）を既定にすることを推奨。実装は EPL** |

> ⚠️⚠️ **「スマートルールによる時限自動クリア」は存在しません。** rev.1 は §8.2 RU-2 と本節でこれを前提にしていましたが、**組み込みのスマートルールテンプレート全 11 種に、時間経過でアラームをクリアするものはありません**（[Smart rules collection](https://cumulocity.com/docs/cockpit/smart-rules-collection/) を全件確認）。
>
> クリアを行うテンプレートは閾値系・ジオフェンス系だけで、しかも「**自分が作ったアラームを条件反転で戻す**」動作です（*"Measurement outside of yellow and red range: If there is an active alarm of given type for the object, clear the CRITICAL and/or MINOR alarm."*）。時間経過は条件になりません。
>
> **[判] 時限自動クリアは EPL（RL-b）で実装してください。** §8.2 の RU-2 も EPL 側へ移してあります。これは §8.1 の使い分け規約（時間窓が要るものは EPL）とも整合します。

> **[判] 「クリア条件が決まっていないアラーム型は投入しない」を規約にしてください。** 型を先に作ってクリアを後回しにすると、本番でアラーム一覧が使い物にならなくなります。

### 8.4 ⚠️ リプレイ抑止 — 網断復旧時の一括再送

**本構成に固有の最重要ガードです。**

```
WAN 断（例: 30 分）
   → thin-edge のストア&フォワードに数千件が滞留
   → 復旧
   → 数千件が一気に Cumulocity へ到達
   → ルールが一斉に発火
   → 数千件のアラーム生成 + 数千件の通知
```

**[判] 対処**:

| # | 対処 | 実装 |
|---|---|---|
| **1** | **ルール雛形にリプレイ抑止ガードを必須で組み込む** | イベントの `time` と現在時刻の差が閾値（例: 5 分）を超える場合は昇格しない、または集約する |
| **2** | **`time` を必須項目にする**（§6.9 TM-d） | ガード 1 の判定に `time` が必要 |
| **3** | **復旧時の一括通知を抑止する** | Notification 2.0 の購読側でも時刻ベースのフィルタを持つ |
| **4** | **検証する** | §14 CT-15: 「一括再送で通知が復旧時刻にまとめて発火しないこと」 |

> ⚠️ **ガード 1 は「古いイベントを捨てる」ことではありません。** イベント自体は Operational Store に残し（後から突合できる）、**アラーム昇格と通知だけを抑止**します。捨ててしまうと欠落と区別がつかなくなります。

> ⚠️⚠️ **[要 SV-44] そもそも「一括再送される」という前提自体が未検証です。**
> thin-edge の公式ドキュメントに**ストア&フォワード／オフラインバッファの容量規定が見つかりません**（MQTT API・設定ファイル・overview・FAQ を全文検索）。加えて、thin-edge が生成する mosquitto ブリッジ設定には **`max_queued_messages` の指定がありません**（`crates/core/tedge/src/bridge/config.rs`）。
>
> **mosquitto の既定は 1000 件**なので、これを超えた分は「あとで再送される」のではなく **その場で捨てられる（欠落する）**可能性があります。
>
> **つまり本節が想定する失敗モード（大量再送）と、逆向きの失敗モード（大量欠落）の両方がありえます。** §14 CT-15 は「一括再送で通知が集中しないこと」だけでなく、**「30 分の網断で送信したメッセージ数と、復旧後に Cumulocity に到達した数が一致すること」**を必ず含めてください。欠落が起きるなら、リプレイ抑止ガードより先に**キュー容量のチューニング**が必要になります。

### 8.5 重複排除の責務

| 事実 | 帰結 |
|---|---|
| **Cumulocity のアラーム de-duplication は「本構成が必要とする重複排除」には使えません** | 対象がアラームのみ・判定キーが (`source`, `type`) のみ・`CLEARED` は対象外（下記）。**検知イベントの重複には効かない** |
| **イベントには重複排除の仕組みがありません** | 責務を一元化する必要がある |
| **thin-edge にも（イベントの）重複排除機能はありません** | 同上。⚠️ ただしアラームは retained 上書きにより「最新のみ保持」になる |
| BOX は「バッファ + 再送機能あり想定」（構成図） | **再送による重複が必ず発生する** |

**[判] 一次責務は BOXアダプタです**（[連携仕様] §4.5）。

| # | 規約 | 内容 |
|---|---|---|
| **DD-a** | **重複排除は BOXアダプタが行う**（複合キー + 永続ストア・仮 72h 窓） | 責務の一元化 |
| **DD-b** | **重複判定キーは external ID + `eventUuid` + `time`** | **`time` は BOX 側のイベント発生時刻**。再送時も同一値であることが前提（§6.9 TM-a） |
| **DD-c** | **Cumulocity 側（EPL）は「検知」に留め、除外の責務は持たない** | 二重に排除すると、どちらが落としたか分からなくなる |
| **DD-d** | ⚠️ **[要 TE-9]** BOX の送信仕様（プロトコル・シーケンス ID の有無・再送間隔・バッファ容量）が未確認 | **重複判定キーと保持ウィンドウ（仮 72h）が確定できない** |

> ⚠️ **Cumulocity サーバー側の de-duplicate の正確な仕様** [確] — Core OpenAPI `POST /alarm/alarms`:
>
> > *"**Alarm de-duplication** — If an **ACTIVE or ACKNOWLEDGED** alarm with the same source and type exists, no new alarm is created. Instead, the existing alarm is updated by incrementing the `count` property; the `time` property is also updated. **Any other changes are ignored**, and the audit log is not created. **Alarms with status CLEARED are not de-duplicated.**"*
>
> - ⚠️ **対象は ACTIVE だけでなく ACKNOWLEDGED も含みます。** rev.1 は「ACTIVE 中の」としていましたが不正確でした。**運用者が「確認済み」にした後も、同じ (`source`, `type`) の新規アラームは立たず `count` が増えるだけ**です
> - 判定キーは (`source`, `type`) のみで **severity は含まれません**（§9.4 で詳述）
> - **アラームにのみ働く仕組み**で、イベントの重複排除には使えません（§6.5 の thin-edge 側「型 + severity」規則とは別レイヤ）
> - **`CLEARED` を挟むフラッピングは畳まれません** → §5.6 要件 1（ヒステリシス）は必須

### 8.6 保守主体の分界

| 対象 | 起案 | 保守 | 根拠 |
|---|---|---|---|
| **ルールの雛形**（リプレイ抑止ガード込み） | **基盤（本チーム）** | 基盤 | D7 |
| **判定ロジック**（閾値・条件式・`<種別>` の列挙） | 案件側 | **案件側** | D7 |
| **アラーム自動クリア** | **基盤** | 基盤 | §8.3 |
| **EPL のコード** | 基盤 | 基盤 | コードなのでレビューと版管理が要る |

> ⚠️ **雛形を渡す時点で「リプレイ抑止ガードを外さないこと」を明示してください。** 案件側が条件式を書き換える際に、ガードごと消してしまうのが典型的な事故です。**ガードは別の EPL / 別のルールに分離し、案件側が触る部分と物理的に分けることを推奨します。**

### 8.7 ルールの版管理と投入

| 項目 | 内容 |
|---|---|
| **スマートルール** | ⚠️ **[要 SV-32]** managed object の `type` / `fragmentType` の実値が未確定。**GUI で 1 つ作成 → `c8y inventory find` で観測してから投入スクリプトを書く**（[投入ガイド] U-01） |
| **EPL Apps** | `POST /service/cep/eplfiles`。**ソースコードごとエクスポート/インポートできる** [確]。Git 管理する |
| **前提** | **Apama-ctrl / Smartrule マイクロサービスへの購読が必要** [確]。§12.4 で稼働確認 |
| **投入順序** | **型名（§6.4）の確定が前提**。型が決まる前にルールは投入できない |

### 8.8 ルール設計の禁則

| # | 禁則 | 理由 |
|---|---|---|
| **RX-1** | **クリア条件のないアラーム型を作らない** | §7.3 |
| **RX-2** | **メール／SMS／エスカレーション系テンプレートを使わない** | 変-2 |
| **RX-3** | **リプレイ抑止ガードのないルールを本番投入しない** | §8.4 |
| **RX-4** | **ルールから直接 HTTP を出さない** | 変-1 により Notification 2.0 のサブスクライバで実装する（RU-6）。EPL からの HTTP は Apama の HTTP クライアントプラグインの Edge 可否（SV-10）に依存する |
| **RX-5** | **型名を条件式にハードコードした個別ルールを拠点ごとに作らない** | 拠点追加のたびにルールが増える。**拠点は `source` のスコープで表現する** |

---

## 9. アラームマッピング定義

### 9.1 `alarm.type.mapping` の仕組み

| 項目 | 内容 | 確度 |
|---|---|---|
| **場所** | テナントオプション。カテゴリ `alarm.type.mapping` / キー `<ALARM_TYPE>` | [確] |
| **キーの解釈** | ⭐ **前方一致（`<type-prefix>*`）**。完全一致ではない（§9.2） | [確] |
| **値の形式** | **`<SEVERITY>\|<TEXT>`**（パイプ区切り）。片方が空なら上書きしない | [確] |
| **効果** | アラーム型ごとに **severity とテキストを上書き**する | [確] |
| **抑止** | severity に **`NONE`** を指定するとそのアラームを抑止できる | [確] |
| **適用タイミング** | アラーム生成時 | [確] |

> ⚠️ **出典に注意 — `alarm.type.mapping` は docs サイトのどのページにも記載がありません。**
> 逐語根拠は **Core OpenAPI の `POST /tenant/options` の "Default option categories" 表**にのみ存在します:
>
> > *"The severity and text are specified as `<ALARM_SEVERITY>|<ALARM_TEXT>`. If either part is empty, the value will not be overridden. **If the severity is NONE, the alarm will be suppressed.** Example: `"CRITICAL|temperature too high"`"*
> > — `https://cumulocity.com/api/core/dist/c8y-oas.yml`
>
> **docs サイトだけを検索した後任が「根拠なし」と誤判定しやすい箇所です。** §16 にこの URL を明記してあります。なお **前方一致の挙動だけは [Alarm mapping](https://cumulocity.com/docs/standard-tenant/alarm-mapping/) 側に記載**があり、2 ページを合わせて読む必要があります。

> **なぜ必要か**: デバイス側が送る severity は実装者に委ねられます。`alarm.type.mapping` を置くことで、**「型ごとの severity を Cumulocity 側で一元的に決められる」**ようになり、デバイス実装の差が運用に漏れなくなります。

### 9.2 マッピング定義表

**[判] 本構成の全アラーム型に対するマッピングです。**

| キー（アラーム型） | 値 | severity の根拠 | 自動クリア条件 |
|---|---|---|---|
| `c8y_UnavailabilityAlarm` | `MAJOR\|ゲートウェイ応答なし` | ゲートウェイ断は拠点全体の停止を意味するが、復旧が自動なので CRITICAL にはしない | 通信再開時（**Cumulocity が自動**） |
| `x_CameraDown` | `MAJOR\|カメラ応答なし` | 1 台の断は監視の欠損。**台数が多いので CRITICAL にすると埋もれる** | 応答再開時（監視サービスがクリア） |
| `x_BoxSilent` | `MAJOR\|BOX無通信` | 同上 | 受信再開時（BOXアダプタがクリア） |
| `x_BoxParseError` | `WARNING\|BOXペイロード変換不能` | 個別のペイロード不良。**デッドレターに退避されるためデータは失われない** | **[要決定]**（§9.6） |
| `x_ModelUpdateFailed` | `CRITICAL\|モデル適用失敗` | **検知精度に直結**し、放置すると古いモデルで運用が続く | 再適用成功時 |
| `x_Alarm_<種別>` | **案件側が決定** | 業務要件 | **[要決定]**（§9.6） |
| `c8y_Application_Down` | **[判] 変更しない**（既定のまま） | Cumulocity の自動生成。**プラットフォーム管理の領域** | プラットフォーム管理 |
| `c8y_Application_Unhealthy` | 同上 | 同上 | 同上 |

> ⭐⭐ **アラーム型は「前方一致」で解釈されます** [確]:
>
> > *"The alarm type provided as an alarm mapping is **interpreted as alarm type prefix: `"<type-prefix>*"`**. If you create, for example, an alarm mapping to address alarms of type "crit-alarm", the mapping is effective for any type of alarm that starts with this value, for example, "crit-alarm-1", "crit-alarm-2", or "crit-alarm-xyz"."*
> > — [Alarm mapping](https://cumulocity.com/docs/standard-tenant/alarm-mapping/)
>
> **rev.1 は「`<種別>` ごとに個別のキーが要る／種別追加のたびにマッピング追加が必要」としていましたが、誤りでした。**
>
> **良い面**: `x_Alarm_` という 1 キーで全種別に効きます。**種別追加のたびの運用手順は不要**になります。
>
> ⚠️⚠️ **危険な面**: 短いキーを置くと、**意図しない型まで severity が上書きされたり抑止されたりします**。特に `NONE`（抑止）と組み合わせると **「アラームが静かに消える」** 最悪ケースになります。例えば `x_` というキーを置くと、`x_` で始まる全アラームが巻き添えになります。
>
> **[判] 型名規約に次を追加してください**（§6.4 に反映・§2.8 の CI 検査 CK-6 として機械検査すること）:
>
> | # | 規約 |
> |---|---|
> | **AM-a** | **どのアラーム型も、他のマッピングキーの前方一致に該当してはならない**（例: `x_CameraDown` と `x_Camera` を同時にキーとして持たない） |
> | **AM-b** | **`NONE` を指定するキーは、必ず完全な型名を書く。** プレフィックスとしての `NONE` 指定を禁止する |
> | **AM-c** | **`x_Alarm_` のような総称キーを使う場合、その配下の型がすべて同じ severity でよいことを確認する。** 個別に変えたい型があれば、より長い（＝より具体的な）キーを追加する |

### 9.3 severity 設計の考え方

**[判] 本構成では 3 段階に整理します。**

| severity | 意味 | 該当 | 期待される対応 |
|---|---|---|---|
| **CRITICAL** | **放置すると業務が成立しなくなる** | `x_ModelUpdateFailed` | 即時対応 |
| **MAJOR** | **監視の一部が欠けている** | `c8y_UnavailabilityAlarm` / `x_CameraDown` / `x_BoxSilent` | 当日中に対応 |
| **WARNING** | **記録として残すが即時対応は不要** | `x_BoxParseError` | 定期棚卸し |
| MINOR | **使わない** | — | MAJOR / WARNING の 2 段で足りる |
| NONE | 抑止（アラームを生成しない） | 現時点で該当なし | — |

| # | 規約 | 理由 |
|---|---|---|
| **SV-a** | **MINOR は使わない** | 段階を増やすと運用者が使い分けられなくなる |
| **SV-b** | **「台数が多いもの」を CRITICAL にしない** | 数百台のカメラ断を CRITICAL にすると、本当に致命的なアラームが埋もれる |
| **SV-c** | **severity は「対応の緊急度」を表す。異常の技術的な深刻さではない** | 運用者が優先順位を判断するための情報 |

### 9.4 ⚠️ severity 変更時の挙動 — 2 層の一意性規則

**アラームの一意性には 2 つの層があり、規則が異なります。**

| 層 | 規則 | 確度 |
|---|---|---|
| **thin-edge のローカル MQTT** | **「型 + severity」で一意**。*"Every alarm is uniquely identified by its type and severity. That is, for a given alarm type, alarms of varying severities are treated as independent alarms and hence, must be acted upon separately."* | [確] |
| **Cumulocity サーバー側の de-duplicate** | **ACTIVE または ACKNOWLEDGED の同一 `source` + `type`** が対象。**severity は条件に含まれない** | **[確]** |

**帰結**:

- thin-edge 側で severity を変えて再 publish すると、**thin-edge から見れば別のアラーム**になります。**旧 severity のアラームを別途クリアしないと 2 つ残ります**
- **Cumulocity 側では、severity 変更は「無視」されます**（下記）

> ⭐ **[確] rev.1 が「未検証」としていた相互作用には、公式の答えがあります。** Core OpenAPI `POST /alarm/alarms`:
>
> > *"If an ACTIVE or ACKNOWLEDGED alarm with the same source and type exists, no new alarm is created. Instead, the existing alarm is updated by incrementing the `count` property; the `time` property is also updated. **Any other changes are ignored**, and the audit log is not created."*
>
> **「Any other changes are ignored」＝ severity を変えて再送しても、Cumulocity 側のアラームの severity は変わりません。** `count` と `time` だけが更新されます。
>
> 傍証として、SmartREST の **305（Update severity of existing alarm）** と **306（Clear existing alarm）** はいずれも **type のみでアラームを識別**します（`306,c8y_TemperatureAlarm`）。severity を変えたければ、新規 POST ではなく 305 のような明示的な更新経路を使う必要があります。
>
> **したがって「severity を上げて再送すれば昇格する」は成立しません。** §14 CT-13 は「観測して確かめる」ではなく「**予測どおりに無視されることを 1 ケースで確認する**」に縮小できます。**型規約（§6.4）の確定を CT-13 の完了まで待つ必要はありません。**

**[判] 暫定規約**: **アラームの severity を運用中に変えない設計にします。** severity を変えたい場合は、`alarm.type.mapping` で型ごとに固定し、デバイス側は常に同じ severity を送ります。**「軽度 → 重度への昇格」が必要な場合は、別の型のアラームとして扱ってください。**

### 9.5 抑止（`NONE`）の使いどころ

| ケース | `NONE` を使うか | 代替 |
|---|---|---|
| 検証中に大量発生するアラームを一時的に止めたい | ⚠️ **使わない** | 原因を直す。`NONE` にすると「直したつもりで実は抑止していただけ」になる |
| 特定のデバイスだけ止めたい | ❌ **使えない**（型単位の設定なのでデバイス指定できない） | メンテナンスモード（`responseInterval` を **0 以下**）。⚠️ **ただしそのデバイスの全アラームが止まります**（§5.2）。「1 つの型だけ止める」用途には使えません |
| 製品が自動生成するアラームで本構成では不要なもの | ✅ **候補** | 現時点で該当なし |

> **[判] `NONE` の使用は「恒久的に不要と判断した型」に限定し、使う場合は理由をコメントとして構成コードに残してください。** 抑止は「アラームが出ない」ことを意味し、切り分け時に極めて追いにくい設定です。

> ⚠️⚠️ **`NONE` は前方一致で効きます。** §9.2 の AM-b のとおり、`NONE` を指定するキーには**必ず完全な型名**を書いてください。プレフィックスとして指定すると、**まだ存在しない将来の型まで巻き添えで抑止されます**。これは「後から型を追加したら、なぜかアラームが出ない」という最も追いにくい事故になります。

### 9.6 クリア条件の定義（必須項目）

**[判] アラーム型を定義するときは、必ず次の 5 項目をセットで決めてください。片方でも欠けたら投入しません。**

| # | 決めること | 例（`x_CameraDown`） |
|---|---|---|
| 1 | **型名** | `x_CameraDown` |
| 2 | **`source`**（どの MO に紐づくか） | カメラ（child device） |
| 3 | **生成条件と生成主体** | 無応答 3 回連続 / thin-edge の ONVIF 監視サービス |
| 4 | **severity とテキスト**（`alarm.type.mapping`） | `MAJOR\|カメラ応答なし` |
| 5 | ⭐ **クリア条件とクリア主体** | 応答再開時 / 同じ監視サービスが空 retained を publish |

**未決定のもの（着手前に確定が必要）**:

| 型 | 欠けている項目 | 決定主体 |
|---|---|---|
| `x_BoxParseError` | **5（クリア条件）** | 基盤 × BOXアダプタ担当 |
| `x_Alarm_<種別>` | **3・4・5** | 案件側 |

---

## 10. 通知（Notification 2.0）設計

> ⚠️ **変-1 により、Notification 2.0 は本構成における「Cumulocity から外部へアラームを届ける唯一の手段」になりました。** ここが動かないと、案件アプリにアラームが一切届きません。

### 10.1 モデル — push ではなく WebSocket 購読

**Notification 2.0 は Cumulocity が外部エンドポイントを叩く push ではなく、コンシューマ側から WebSocket で接続して読む方式です** [確]。Messaging Service がトピックにメッセージを永続化します。

```
① 基盤が サブスクリプション を定義   POST /notification2/subscriptions   [ROLE_NOTIFICATION_2_ADMIN]
                                       → Messaging Service にトピックが対応づく
② 案件アプリが トークン を取得       （基盤のトークン発行プロキシ経由・§10.4）
③ 案件アプリが WebSocket 接続        トークンで認証 → サブスクライバが生成される
④ メッセージを受信し ack
```

| 前提 | 状態 |
|---|---|
| **Messaging Service が稼働していること** | 2026 Edge には `Included` [確]。**ただし実際に動いていることの確認が必須**（§12.4） |
| **WebSocket が LoadBalancer 経由で VM2 から到達できること** | **[要 SV-05]**。P-0-2 の公開ポートに含まれるかを併せて確認 |

### 10.2 サブスクリプション設計

**`NotificationSubscription` の主要フィールドと本構成での値** [確]:

| フィールド | 値域 | **[判] 本構成での値** |
|---|---|---|
| `context` **(必須)** | `mo` / `tenant` | **`mo`**（`tenant` はテナント全体で拠点分離ができない） |
| `source` | managed object のグローバル ID | **拠点グループの MO ID**（`context: mo` のとき必須） |
| `subscription` **(必須)** | トピック名。**`pattern: '^[a-zA-Z0-9]+$'` / `minLength: 1` / `maxLength` の定義なし** | **拠点コード**（§1.4・§2.5） |
| `subscriptionFilter.apis` | `alarms` / **`alarmsWithChildren`** / `events` / **`eventsWithChildren`** / `managedobjects` / `measurements` / `operations` / `*` | **`alarmsWithChildren`**（案件アプリ向け）／必要に応じて `eventsWithChildren` |
| `subscriptionFilter.typeFilter` | 型の完全一致、または `or` の OData 式 | **§6.4 の型で絞る**（例: `'x_Alarm_Intrusion' or 'c8y_UnavailabilityAlarm'`） |
| `fragmentsToCopy` | 指定した独自フラグメント**のみ**を含める | ⚠️ **生体情報・個人情報に関わるフラグメントを案件アプリへ渡さないために使う** |
| `nonPersistent` | boolean | **`false`（既定・永続）**。⚠️ `subscription` 名が同じでも `nonPersistent` が違えば**別トピック**になる |

**`alarmsWithChildren` は `source.id` の managed object と、その配下の全 descendant managed object のアラームを購読します** [確]:

> *"The `alarmsWithChildren` and `eventsWithChildren` APIs subscribe to alarms and events respectively from the managed object identified by the `source.id` field, and **all of its descendant managed objects**."*

> ⚠️⚠️ **「descendant」がどの関係を辿るのかは、公式のどこにも書かれていません** — 本書の前提 2 が [要] である理由です。
>
> 公式の記述は一貫して関係を特定しない語だけで、**`childAssets` / `childDevices` / `childAdditions` のどれを辿るかを述べた一次情報は存在しません**。OpenAPI（latest / 2026 / 2025）・Edge OpenAPI・Tech Community 全文検索・WebSearch・domain-model の 7 経路で確認済みです。
>
> **決定的な対比材料**: 同じ Cumulocity の Alarm/Event **REST** API は明示的に区別しています —
>
> ```yaml
> withSourceAssets:    'alarms for related source assets will also be included'
> withSourceDevices:   'alarms for related source devices will also be included'
> withSourceAdditions: ...
> ```
>
> **区別する語彙が製品内に存在するのに、Notification 2.0 の説明ではあえて使われていません。** 「全部辿る」「片方だけ」のどちらの可能性も残ります。**実機確認以外に確定手段はありません**（§14 CT-4）。§1.1 の記述もこれに合わせて [推] に格下げしてあります。

#### **[判] 本構成のサブスクリプション定義**

| # | サブスクリプション | `context` | `source` | `apis` | `typeFilter` | コンシューマ |
|---|---|---|---|---|---|---|
| **NS-1** | 拠点ごとのアラーム | `mo` | **各拠点グループ** | `alarmsWithChildren` | 案件が必要とする型 | 案件アプリ（拠点担当） |
| **NS-2** | D14 の自動クリップ保存 | `mo` | 拠点 root グループ | `alarmsWithChildren` | `x_Alarm_<種別>` のみ | 基盤のクリップ保存サービス |
| **NS-3** | 新規デバイス登録の検知 | **`tenant`** | — | `managedobjects` | — | **基盤の運用サービスのみ**（案件アプリには渡さない） |

> ⚠️ **`mo` 文脈では「新規 managed object の作成」を受け取れません**（作成時点で ID が無いため）[確]。**新拠点・新装置の追加を検知したい場合は、別途 `tenant` 文脈のサブスクリプション（NS-3）が必要**です。

> ⚠️ **`tenant` 文脈では measurements を購読できません** [確]。オフロードに計測を含める場合は REST ページング（§7.5 案 B）が必要です。

**context ごとの API 対応表** [確]:

| Context | MO Create | MO Update & Delete | Alarms | Events | Measurements | Operations |
|---|---|---|---|---|---|---|
| `mo` | **✗** | ✓ | ✓ | ✓ | ✓ | ✓ |
| `tenant` | ✓ | ✗ | ✓ | ✓ | **✗** | ✓ |

> ⚠️ **前提 2（`alarmsWithChildren` が `childAssets` を辿るか）が NG の場合、NS-1 は §1.6 の代替案 (i)（main device を `source` にする）に切り替えます。** 拠点あたり 2 サブスクリプションになりますが、案 β なら成立します。

### 10.3 ⚠️ RBAC バイパス — 構成の中心問題

**Core OpenAPI 仕様に逐語で** [確]:

> *"**⚠️ Caution:** If you assign Notification 2.0 roles or permissions to users, they can create Notification 2.0 subscriptions and receive notifications for any device, including those to which assigned inventory roles do not grant access, **bypassing the inventory role RBAC**."*

| 事実 | 帰結 |
|---|---|
| `POST /notification2/subscriptions` の必要ロールは**テナントレベルの `ROLE_NOTIFICATION_2_ADMIN` 単独**。`source` によるスコープ指定がない | 付与された主体は**任意のデバイスのサブスクリプションを作れる** |
| `POST /notification2/token` も**同じロールを要求する** | トークン取得だけを許すこともできない |
| Alarm API は *"...if the user has access to alarms through inventory roles, only those alarms are returned"* | **Notification 2.0 は意図的かつ文書化された例外** |
| 2025 / 2026 / Latest の 3 版でバイト一致 | **将来版で直る見込みは薄い** |

**[判] 設計方針**:

| # | 決定 | 内容 |
|---|---|---|
| **D-a** | `ROLE_NOTIFICATION_2_ADMIN` は**トークン発行プロキシのサービスユーザーにのみ付与**する | §11.4 の R-09 |
| **D-b** | 「拠点担当者に自拠点だけの通知購読権を **Cumulocity のロールで** 与える」ことは**原理的に不能**。ただし**サブスクリプション自体は拠点グループ単位に絞れる**（§10.2）。**基盤がサブスクリプションを定義し、トークン発行プロキシで配る**ことで実質的な拠点分離が成立する | §10.4 |
| **D-c** | 顧客間の分離が必要になった時点（2 顧客目）で単一テナント集約は成立しない。**顧客ごとに Edge を分ける**（拠点ごとではない） | — |
| **D-d** | **運用知識基盤も拠点分離の外側に立つ。ただし「常時付与」ではなく承認時の一時付与に絞る** | §4.10 G-2 |
| **D-e** | **案件アプリ・基盤標準アプリの REST 呼び出しは原則エンドユーザーの OIDC トークンで行い、サービスユーザー代行を禁止する** | サービスユーザー代行は Notification 2.0 と同型の RBAC バイパスになる |

### 10.4 トークン発行プロキシ

**[判] 基盤標準部品として「トークン発行プロキシ」を置きます。**

```
案件アプリ ──① OIDC トークンで認証 ──► トークン発行プロキシ（基盤標準・VM1）
                                          │ ② 呼び出し元が購読してよい拠点を判定
                                          │    （Keycloak のグループクレーム or Inventory ロール）
                                          │ ③ ROLE_NOTIFICATION_2_ADMIN を持つ
                                          │    基盤サービスユーザー（R-09）として
                                          ▼
                                      POST /notification2/token
                                      { subscriber, subscription: "<拠点コード>", expiresInMinutes }
案件アプリ ◄── ④ 当該トピックにスコープされたトークンのみ返す
```

| # | 設計要件 | 根拠 |
|---|---|---|
| 1 | **発行されるトークンはトピックにスコープされる** [確]（JWT の `topic` クレーム） | プロキシが渡すサブスクリプション名を絞れば、案件アプリは他拠点のトピックを読めない |
| 2 | **`ROLE_NOTIFICATION_2_ADMIN` を持つのはプロキシのサービスユーザーだけ** | §11.4 R-09 |
| 3 | **`expiresInMinutes` の既定は 1440 分（24 時間）** [確]。⚠️ **上限値の定義は無い**（OpenAPI に `maximum` / `minimum` の指定なし） | **案件アプリ側にトークン再取得のロジックが必要**（**[要 SV-36]** 案件側との合意事項）。**[判] プロキシ側で既定より短い値を明示指定してください**（下記） |
| 4 | `shared: true` は**冗長化ではなく負荷分散**（下記） | 誤用すると通知が取りこぼされる |
| 5 | ⚠️ **実装主体が未確定**（TB-3） | **決めないと通知経路全体が止まる** |
| 6 | **WebSocket の接続仕様** [確]: パス `/notification2/consumer/`、クエリは `token`（必須）と `consumer`（任意）のみ。ポートは 443 / 80 / `cumulocity:8111` | §12.5 の疎通対象ポートの確定（SV-05）に必要 |
| 7 | ⚠️ **アイドル 5 分でサーバ側から切断される。未 ack が 1000 件に達すると配信が止まる** [確] | **案件アプリに再接続と ack の実装が必須**（SV-36 の合意事項に含めること） |

> ⚠️ **トークン失効は完全な歯止めにはなりません** [確]:
>
> > *"Tokens only allow a consumer to connect to the Messaging Service. **If a token expires while its consumer is connected, the consumer is not automatically logged out or disconnected.**"*
>
> **接続中のコンシューマは失効後も受信し続けます。** RBAC バイパス対策としてプロキシを置いた以上、`expiresInMinutes` の短縮だけに頼らず、**不正なコンシューマを止める手段（`POST /notification2/unsubscribe`）を runbook に用意してください**（§10.5）。

> ⚠️⚠️ **`shared: true` は「冗長化」ではありません** [確]:
>
> > *"Shared consumer tokens allow **parallelization of the consumer client workload**"* / *"each consumer client receives a **non-overlapping subset (share)** of the notifications"* / *"Messages received by a shared consumer client **must** be acknowledged on the same WebSocket connection."*
>
> **冗長化のつもりで 2 台に `shared: true` のトークンを配ると、通知が 2 台に分配されます**（両方が全件受け取るわけではありません）。**片方が落ちればその分は処理されません。** 冗長化が目的なら、Active/Standby にして**同時に接続しない**構成にしてください。

> ⚠️ **このプロキシは §10.3 D-e と同じ思想です。Notification 2.0 だけロールを直接渡す例外を作らないでください。**

### 10.5 コンシューマのライフサイクル

| 事実 [確] | 帰結 |
|---|---|
| サブスクライバは**最初の WebSocket 接続時に生成**される | 事前作成は不要 |
| **一度作られたサブスクライバは、WebSocket が切断されても削除されない** | ⚠️ 停止した案件アプリの分のメッセージが**溜まり続ける** |
| Messaging Service は、消費されるか TTL に達するか**明示的に unsubscribe されるまで**メッセージを永続化する | ⚠️⚠️ **ディスク逼迫ではなく「データ投入の停止」を招く**（下記） |
| **Hard quota**: メッセージバックログ **25 MiB** / TTL **36 時間** | 「無限に溜まる」わけではないが、**25 MiB に達した時点で書き込み側が壊れる** |
| 解除は `POST /notification2/unsubscribe?token=<token>` | ⚠️ **DELETE ではなく POST**。当該トピック・サブスクライバのトークンを作ってから呼ぶ |

> ⚠️⚠️ **バックログ満杯の帰結は「ディスクを食う」ではなく「アラーム・イベントが登録できなくなる」です** [確]:
>
> > *"If the backlog for a Notifications 2.0 topic has reached its quota limit, **any API request to the Cumulocity platform that would be published onto that topic will receive HTTP response code 500.** … Note that for requests using the PERSISTENT processing mode, the Cumulocity operational store will still be updated. This can lead to duplicated entries…"*
> > — Core OpenAPI §Notification 2.0 Service Quotas
>
> **拠点 A の案件アプリが停止して ack を止めると、25 MiB に達した時点でその拠点のアラーム・イベントの POST が 500 で失敗するようになります。** 「通知が届かない」だけでなく **「データが記録されなくなる」** という、まったく重さの違う障害です。rev.1 はこれを容量問題として扱っていました。
>
> **これは §10.3 の RBAC バイパスと並ぶ、Notification 2.0 由来の第 2 の構造的リスクです。** 案件アプリの停止が基盤のデータ収集を壊すため、**案件アプリの稼働状況を基盤側が監視する必要があります**（下表 2）。

**[判] 運用設計に必ず入れること**:

| # | 対応 |
|---|---|
| 1 | **廃止した案件アプリ・試験用サブスクライバの unsubscribe 手順**を runbook 化する（[担当範囲] RB-10） |
| 2 | ⭐ **サブスクライバ数とバックログ量を監視する。公式 UI に専用画面があります** [確] |
| 3 | **拠点撤収時に該当サブスクリプションを解除する**（§3.9） |
| 4 | **§7.6 の容量サイジングにバックログ分（最大 25 MiB × トピック数）を含める** |

> ⭐ **[確] SV-33 は解消しました。監視手段は公式に用意されています。**
> `Administration > Monitoring > Messaging Service` に次の列があり、**公式が危険域まで示しています**:
>
> | 指標 | 危険域（公式） |
> |---|---|
> | Subscribers | > 5 |
> | Message backlog | > 20 MB |
> | Used backlog | > 80% |
> | Unacknowledged messages | > 1000 |
> | Last acknowledged | >= 1 day |
>
> 同画面から **UI 経由の unsubscribe も可能**です。出典は §16 に既載の [Monitoring](https://cumulocity.com/docs/standard-tenant/monitoring/) です。
>
> **rev.1 の「取れない場合は棚卸し運用が唯一の歯止め」という前提は誤りでした。** この画面の値を §12.7 のメタ監視に取り込み、**Used backlog 80% でアラートを上げてください**（25 MiB 到達＝データ投入停止の直前です）。

### 10.6 人への通知経路

**変-2 によりメール通知を採用しないため、Cumulocity から人へ直接届く通知経路はありません。**

| 経路 | 状態 |
|---|---|
| スマートルール「On alarm send email / SMS / escalate」 | **不採用**（変-2・§8.1 RL-c） |
| **Notification 2.0 → 案件アプリ → 人（アプリの通知機能）** | **これが唯一の経路** |
| Cumulocity Web UI / 紐づけ確認アプリでの目視 | 能動的な通知ではない |

> ⚠️⚠️ **基盤自身の異常が誰にも届きません。** `c8y_Application_Down` / `c8y_Application_Unhealthy` や Edge 自体の障害は、メールが無い以上 **メタ監視（§12.7）が唯一の検知経路**です。**Prometheus エンドポイントを保守拠点側の監視に接続し、アラート通知先を Cumulocity の外側に持つことを設計に入れてください**（SV-34 / TB-5）。

### 10.7 通知設計の禁則

| # | 禁則 | 理由 |
|---|---|---|
| **NX-1** | **`ROLE_NOTIFICATION_2_ADMIN` を案件アプリ・業務ユーザーに付与しない** | §10.3。付与した瞬間に拠点分離が無効化される |
| **NX-2** | **`context: tenant` のサブスクリプションを案件アプリに渡さない** | 拠点分離が成立しない |
| **NX-3** | **`fragmentsToCopy` を指定せずに個人情報を含むフラグメントを流さない** | 生体情報・個人情報の持ち出し |
| **NX-4** | **`nonPersistent` を用途によって使い分けない** | 同名でも別トピックになるため、設定ミスが「なぜか届かない」として現れる |
| **NX-5** | **unsubscribe 手順のないサブスクライバを作らない** | §10.5 |

---

## 11. ロール・アクセス制御定義

### 11.1 アクセス主体の棚卸し

**「業務アプリに与える権限」を定義する前に、Cumulocity にアクセスする主体を全て洗い出します。抜けがあると、後から場当たり的にロールが増えます。**

| # | 主体 | 種別 | 認証 | 配置 | 何をするか |
|---|---|---|---|---|---|
| **S-1** | **基盤運用者** | 人 | SSO（Keycloak） | 業務端末 / 保守端末 | テナント設定・デバイス管理・オペレーション発行 |
| **S-2** | **拠点オペレーター** | 人 | SSO | 業務端末 | 自拠点のアラーム対応・デバイス状態確認 |
| **S-3** | **業務閲覧者** | 人 | SSO | 業務端末 | Cockpit で自拠点のアラーム・イベントを閲覧 |
| **S-4** | **保守要員**（AI モデル） | 人 | SSO | 保守端末（保守VPN） | ソフトウェアリポジトリ登録・更新オペレーション発行 |
| **S-5** | **紐づけ確認アプリ** | アプリ（ブラウザ） | **エンドユーザーの OIDC トークン**（D-e） | VM2 | イベント参照 → VMS 映像と突合 |
| **S-6** | **生体認証SA / 他アセット** | アプリ（ブラウザ / サーバ） | エンドユーザー OIDC / サービスユーザー | VM2 | アラーム参照・通知購読 |
| **S-7** | **トークン発行プロキシ** | サービス | サービスユーザー | VM1 | `POST /notification2/token`（§10.4） |
| **S-8** | **運用知識基盤（読取）** | サービス | サービスユーザー | VM1 | 拠点横断でイベント・アラーム・計測を読む |
| **S-9** | **運用知識基盤（書込）** | サービス | サービスユーザー | VM1 | 設定配布オペレーションの発行（§4.10 のガードレール下） |
| **S-10** | **オフロードバッチ** | サービス | サービスユーザー | VM1 | REST ページングで読み出し（§7.5） |
| **S-11** | **プロビジョニングツール / CI** | サービス | **ローカルユーザー（SSO 対象外）** | CI 環境 | デバイス登録・証明書アップロード・設定投入 |
| **S-12** | **break-glass** | 人 | **ローカル（SSO 対象外）** | — | SSO 障害時の緊急アクセス |
| **S-13** | **デバイス**（thin-edge） | デバイス | **X.509 証明書**（SSO 対象外） | 拠点 / VM1 | データ送出・オペレーション実行 |
| **S-14** | **クリップ保存サービス** | サービス | サービスユーザー | VM1 | NS-2 を購読 → `exportClip`（§10.2） |
| **S-15** 〈rev.3〉 | ⚠️⚠️ **Edge ホスト OS / k3s 管理者** | 人 | **OS ログイン（Cumulocity の認証系の外）** | Edge ホスト | インストール・アップグレード・バックアップ・障害調査（§12）。⚠️ **Cumulocity の RBAC も監査ログも一切効きません** |
| **S-16** 〈rev.3・CU-18 決定後に確定〉 | **画像配信 / 監査書込マイクロサービス** | サービス | サービスユーザー（**R-14**） | Edge（Microservice Hosting） | 画像の認可判定 → 配信 → `POST /audit/auditRecords`。**CU-18 で (A)/(A′)/(D)/(B) を採る場合に必要**（§15.3） |
| **S-17** 〈rev.4〉 | **監査担当** | 人 | SSO | 業務端末 | **監査ログの月次レビュー**（§12.10.8）と break-glass 使用時の事後確認。**R-17**（§11.2）で `ROLE_AUDIT_READ` を付与する（AS-13(c) の「監査担当」の実体）。⚠️ **rev.3 まで AS-13 が「監査担当」を参照しながら、主体一覧にもロール定義にも存在しませんでした** — 誰がレビューし、どのロールで読むかの両方が未定義だった漏れを解消 |

> ⚠️⚠️ **S-15 は本書の統制がまったく効かない主体です** 〈rev.3 で追加〉。
> **Edge は root アクセスが可能で k3s 上で動きます**（§15.3）。root を持てば **MongoDB を直接読める**（全画像・全監査レコード）、**`kubectl get secret` でサービスユーザーの資格情報を取得できる**、**`/var/lib/rancher/k3s` を丸ごと持ち出せる**。§11.9 AP-c「Operational Store への直接アクセスは行わない」は**規約であって強制力がありません**。
>
> **CU-18 でどの案を採ってもこの経路は塞がりません。** 対応は技術ではなく運用側になります — **保持者の限定・分掌・踏み台経由でのセッション記録**のいずれかを §12 で定義するか、**「Edge ホスト管理者の画像アクセスは技術的に監査不能」を法務に明示**してください。**[要決定] CU-29（§15.3）** として起票しました。

> ⚠️ **S-13（デバイス）と S-7/S-11/S-12 は SSO 対象外です** [確]。SSO を有効にした後も、これらは証明書またはローカル認証で動きます。**SSO 切替時に壊れないことを §14 CT-26 で確認してください。**

### 11.2 グローバルロール定義

**[判] 5 つのグローバルロールを定義します**〈rev.4 訂正 — R-17（監査担当）を追加し 4→5 に〉。

| ID | ロール名 | 対象主体 | 含める権限 |
|---|---|---|---|
| **R-01** | **基盤運用者** | S-1 | Tenant Manager 相当 + Application management ADMIN + Retention rules ADMIN + CEP management ADMIN + Device control ADMIN + **Option management ADMIN**（`ROLE_OPTION_MANAGEMENT_ADMIN`・§11.8）+ **Own user management READ** |
| **R-02** 〈rev.2 で改訂〉 | **拠点オペレーター** | S-2 | **Own user management READ + Device Management アプリアクセスのみ。**<br>**データ系（Alarms / Events / Inventory / Measurements）はグローバルに付与しない** → R-04a（Inventory ロール）で拠点別に付与 |
| **R-03** 〈rev.2 で改訂〉 | **業務閲覧者** | S-3 | **Own user management READ + Cockpit アプリアクセスのみ。**<br>**データ系はグローバルに付与しない** → R-04b で拠点別に付与 |
| **R-10** 〈本書で追加〉 | **保守（AI モデル）** | S-4 | **ソフトウェアリポジトリ書込**（`ROLE_INVENTORY_ADMIN`）+ Device control ADMIN + Device Management アプリアクセス + Own user management READ。⚠️ **「files 権限」という権限カテゴリは存在しません**（公式の権限カテゴリ 21 種に無し）。**[要 SV-46]** ソフトウェアバイナリの書込に追加ロールが要るかを実機確認すること |
| **R-17** 〈rev.4 で追加〉 | **監査担当** | S-17 | **`ROLE_AUDIT_READ` + Own user management READ + Device Management/Cockpit アプリアクセス（閲覧のみ）。`ROLE_AUDIT_ADMIN` は含めない**（AS-13 と抵触する）| 〈rev.4 訂正〉S-17（監査担当）と、直後の「付与先は R-01 基盤運用者と監査担当のみ」という規約に、この R-17 まで**具体的なロール定義が存在しなかった**（主体は定義されたが権限の実体が無い状態だった）。§13.3.4 の投入対象・AS-13(c) の CI 検査対象に R-17 を追加すること |

> ⚠️⚠️⚠️ **rev.1 の R-02 / R-03 は、拠点分離を丸ごと無効化していました。**
>
> §11.3 は「Inventory ロールが拠点分離の実装点です」と宣言していますが、rev.1 は R-02 に `Alarms ADMIN` / `Events READ` / `Inventory READ`、R-03 に `Alarms READ` / `Events READ` / `Inventory READ` を **グローバルロール**として与えていました。**グローバルにデータ系ロールを持つと、Inventory ロールによる絞り込みが効かなくなります** [確]:
>
> > *"The role ROLE_ALARM_READ is not required, but if a user has this role, **all the alarms on the tenant are returned**. If a user has access to alarms through inventory roles, only those alarms are returned."*
> > — Core OpenAPI `getAlarmCollectionResource`
>
> > *"In a Role Based Access Control (RBAC) approach **you must use the inventory roles in order to have the correct level of separation. Apart from some global permissions (like "own user management") customer users will not be assigned any roles.**"*
> > — Core OpenAPI §Access rights and permissions
>
> **§10.3 で Notification 2.0 の RBAC バイパスを厳密に潰しているのに、REST 側にそれより広い穴が最初から開いていました。** §11.10 の否定テスト CT-5（拠点 A の Manager が拠点 B のアラームを取得できないこと）は、rev.1 のロール定義では**必ず失敗します**。
>
> **[判] 原則: 顧客ユーザーに付与してよいグローバル権限は「Own user management READ」と「アプリケーションアクセス」だけです。** データに触る権限はすべて Inventory ロール経由にしてください。

> ⚠️⚠️ **「Own user management」の READ が無いとログインできません。** SSO のアクセスマッピングで割り当てる**全ロールに必ず含めてください**。

> ⚠️ **API ロール名に `ROLE_INVENTORY_UPDATE` は存在しません** [確]。OAS に出現するのは `ROLE_INVENTORY_ADMIN` / `_CREATE` / `_READ` のみです。**UI の権限レベル（READ/CREATE/UPDATE/ADMIN）と API ロール名は 1:1 対応しない**ため、投入スクリプトでハマります。**実際の `ROLE_*` 名は投入前に `GET /user/roles` で確認してください。**

> ⚠️ **⭐ Cloud Remote Access のロール（`ROLE_REMOTE_ACCESS_ADMIN`）は、どのグローバルロールにも含めません**（§4.7 手段 4）。**既定ではどのグループ・ユーザーにも割り当てられていません**ので、「付けない」を維持するだけで足ります。§13.7 AS-4 でこのロール名を検査してください。

> ⚠️ **⭐ `ROLE_AUDIT_ADMIN` も、どのグローバルロールにも含めません** 〈rev.3 で追加〉。
>
> `POST /audit/auditRecords` の Required roles は *"ROLE_AUDIT_ADMIN **OR** ROLE_SYSTEM **OR** AUDIT_ADMIN permission on the resource"* です [確]。**このロールを人間ユーザーに付けると、監査レコードを自分で捏造できます。** 監査ログは *"cannot be externally modified after being written"* [確]（＝**書いた後は改竄できない**）という性質で証跡たり得ているので、**「書く権限」を配ると証跡としての価値がゼロになります。**
>
> **付与先は「監査レコードを書くサービスユーザー」のみ**とし、§13.7 **AS-13** で検査してください。**これは CU-18 の結論がどれになっても変わらないガードレールです。**
>
> ⚠️ **監査ログの閲覧権限（`ROLE_AUDIT_READ`）も絞ってください。** 個人情報アクセスの記録を残す設計を採る場合、**監査ログ自体が「誰がいつ誰の顔を見たか」の集合体**になります。付与先は **R-01 基盤運用者と監査担当のみ**です。
>
> ⚠️⚠️ **グローバルロールだけ絞っても足りません** 〈rev.3 で追加〉。**`inventoryRolePermission.scope` の enum に `AUDIT` が実在します** [確]（`ALARM` / **`AUDIT`** / `EVENT` / `MANAGED_OBJECT` / `MEASUREMENT` / `OPERATION` / `*`）。監査レコードの `source` は**対象カメラ**（＝拠点グループ配下）と決めているため、**誰かが Inventory ロールに Audits READ を足した瞬間、拠点ユーザーが「自拠点で誰が誰の顔を見たか」を読めるようになります。** → **AS-13 で「全 Inventory ロールの `permissions` に `scope: AUDIT` が含まれないこと」を検査してください。これは CU-18 でどの案を採っても必要です。**

> ⚠️⚠️ **R-01「Tenant Manager 相当」を `ROLE_*` に展開してください** 〈rev.3 で追加〉。
>
> 本節は R-01 だけを標準ロールの流用のまま残しています。しかし**本書自身が「UI の権限レベルと API ロール名は 1:1 対応しない。投入前に `GET /user/roles` で確認せよ」と警告している**とおり、中身が不明です。**Tenant Manager に `ROLE_AUDIT_ADMIN` / `ROLE_INVENTORY_READ` / `ROLE_BINARY_READ` が含まれていれば**:
>
> - **AS-13 は最初から不合格**になります
> - **CU-18 でどの案を採っても、R-01 は全拠点の画像を無監査で取得できます**（(D) でも `ROLE_INVENTORY_READ` が 4 択の 1 つ目を満たすため）
>
> **CU-19 と同時に実機で `GET /user/roles` を引き、R-01 の実体を確定させてください**（**CU-27**）。外せないなら「**基盤運用者の画像アクセスは技術的に監査不能**」を法務に明示する必要があります。

### 11.3 Inventory ロール定義と割当

**Inventory ロールが拠点分離の実装点です。**

**rev.2 で R-02 / R-03 からデータ系のグローバル権限を外したため、ここが唯一のデータアクセス経路になります**（§11.2）。

| ID | ロール名 | 権限 | 割当先 |
|---|---|---|---|
| **R-04a** | **拠点Manager** | Alarms ALL / Events READ / **Inventory ALL** / Measurements READ | 拠点オペレーター（S-2）× 担当拠点グループ |
| **R-04b** | **拠点Reader** | Alarms READ / Events READ / Inventory READ / Measurements READ | 業務閲覧者（S-3）× 担当拠点グループ |

> ⚠️⚠️ **`Events READ` は「スナップショット画像のダウンロード権限」を含みます** [確]〈rev.3 で追加〉。
>
> `GET /event/events/{id}/binaries` の Required roles は *"ROLE_EVENT_READ **OR** EVENT_READ permission on the source"* です（Core OpenAPI `getEventBinaryResource`）。**イベント本体の参照と添付画像の取得を、Cumulocity 標準の権限モデルで分離することはできません。**
>
> **したがって R-04a / R-04b を割り当てた利用者は、UI を経由せず `curl` で直接スナップショットを取得できます。** スナップショットが個人情報（顔が判別できる画像）を含む場合、**「UI にログを仕込む」だけの対策は監査証跡になりません**（迂回できるため）。→ **§6.8 制約 5・6 / CU-18 / CT-5b**
>
> ⚠️ **S-5（紐づけ確認アプリ）も同じです。** エンドユーザーの OIDC トークンで動く（D-e）ため、**同じトークンで REST を直叩きできます。**

> ⚠️⚠️ **`Inventory ALL` は削除権限を含みます** 〈rev.3 で追加〉。
> `DELETE /inventory/managedObjects/{id}` は *"… **OR** `MANAGE_OBJECT_ADMIN` permission on the source"* で通り、*"**Deleting a parent group will also delete its subgroups.** The `cascade` parameter controls whether the devices assigned to the groups are also deleted."* [確]。**つまり拠点 Manager は自拠点のカメラを、さらには拠点グループごと（cascade で配下デバイスも）削除できます。**
>
> rev.2 で `CHANGE` → `ALL` に上げた理由は「一覧が開けないから」だけで、**削除を含むことの評価をしていませんでした。** 削除自体は監査ログに残りますが（`type=Inventory`）、**誰も見ていなければ意味がありません** → §12.7 ⑭ で監視対象に入れました。**「拠点 Manager に削除させるか」を CU-3（物理削除 / 論理削除）の入力に追加してください。**

> ⚠️ **`CHANGE` は `READ` を含みません** [確]: *"CHANGE - to modify objects (**does not include READ permission**)"* / *"ALL - to read AND modify objects"*。
> rev.1 は R-04a を `Inventory CHANGE` としていましたが、**グローバルの `Inventory READ` を外した rev.2 では、これだと拠点 Manager がデバイス一覧を開けなくなります。** `Inventory ALL` に改めました。

| # | 規約 | 理由 |
|---|---|---|
| **IR-a** | **拠点グループに割り当てる。デバイス個別に割り当てない** | *"Inventory roles are inherited from groups to all their direct and indirect subgroups, **and to the devices in these groups**."* [確] |
| **IR-b** | ⚠️ **手動割当は SSO の `inventoryMappings` に次回ログインで上書きされる** [確] | **割当は必ず `inventoryMappings` で行う**（§11.6） |
| **IR-c** | **投入は `c8y api` で `/user/inventoryroles` を直接叩く** | go-c8y-cli にトップレベルの `inventoryroles` サブコマンドが存在しない |
| **IR-d** | **冪等化は 409 黙殺**（[投入ガイド] パターン E） | `POST /user/inventoryroles` は **409 "Duplicate" を返す** [確] ので、このパターンが有効。⚠️ **グループ作成（G-01）とは異なる**ので混同しないこと（§13.3.3） |

> ⚠️⚠️ **[要 SV-47] 継承が「デバイス配下の子デバイス」まで届くかは公式に書かれていません。**
> 公式が保証するのは **「グループ → サブグループ → それらのグループに属するデバイス」** までです。**カメラ・BOX は `childAssets` ではなく `childDevices` 経由で main device にぶら下がる**ため（§1.3）、**Inventory ロールが到達するかは未確認**です。
>
> **これは §10.2 の `alarmsWithChildren` と構造的に同一の疑問です。** 通知側だけ確認して安心すると、REST 側で拠点分離が破れます。**§14 CT-4 の実機確認で、通知と Inventory ロールを同じ階層構成で同時に検証してください。**
>
> **もし届かない場合の影響**: 拠点 Manager がカメラ個別のアラーム・インベントリを見られなくなり、§11.3 の設計が成立しません。代替は「カメラも `childAssets` で拠点グループに直接ぶら下げる」（§1.3 の階層設計の変更）になります。

> ⚠️ **Inventory ロールは Notification 2.0 には効きません**（§10.3）。**「Inventory ロールで拠点分離できている」と考えると、通知経路で穴が空きます。**

### 11.4 サービスユーザー

**[判] サービスユーザーはすべて SSO 対象外のローカルユーザーとし、資格情報は CI シークレットストアで管理します。**

| ID | 用途 | 対象主体 | 最小ロール | 備考 |
|---|---|---|---|---|
| **R-05** | VM2 アセット用 | S-6 | 最小限の READ | ⚠️ **原則としてエンドユーザー OIDC トークンを使う（D-e）。サービスユーザーはサーバサイド常駐が必要な場合のみ** |
| **R-06a** | **運用知識基盤（読取）** | S-8 | 拠点横断 READ（Alarms / Events / Measurements / Inventory） | **分析処理は常時こちらで稼働させる** |
| **R-06b** | **運用知識基盤（書込）** | S-9 | `ROLE_DEVICE_CONTROL_ADMIN` + `ROLE_INVENTORY_ADMIN` | ⚠️ **承認を伴う経路からのみ実行。§4.10 G-2 で拠点権限は一時付与** |
| **R-08** | **証明書アップロード / CA 操作** | S-11 | `ROLE_TENANT_ADMIN` または `ROLE_TENANT_MANAGEMENT_ADMIN`（**かつ現テナントであること**） | ⚠️ **`tedge cert upload` が Basic 認証しか使わないため**ローカルユーザーが必要（§3.4 P-b）。**Cumulocity API 側の制約ではありません** — OpenAPI は両エンドポイントとも `security` に `SSO` を列挙しています |
| **R-09** | **トークン発行プロキシ** | S-7 | ⚠️ **`ROLE_NOTIFICATION_2_ADMIN` を持つ唯一の主体** | §10.4。**他の誰にも付与しない** |
| **R-11** 〈本書で追加〉 | **オフロードバッチ** | S-10 | 拠点横断 READ | §7.5 |
| **R-12** 〈本書で追加〉 | **クリップ保存サービス** | S-14 | Alarms READ + 通知購読用トークンの受領 | §10.2 NS-2 |
| **R-14** 〈rev.3・**CU-18 決定後に確定**〉 | **画像配信 / 監査書込マイクロサービス** | S-16 | **`ROLE_AUDIT_ADMIN`**（監査レコード書込）+ 画像読取（(D) なら **`ROLE_BINARY_READ` のみ**。`ROLE_INVENTORY_READ` は付けない） | §15.3。⚠️ **`ROLE_AUDIT_ADMIN` を持つ唯一の主体**（AS-13）。**認可判定の要件は §11.11** |

> **[判] R-06 を 2 つに分ける理由**: 単一アカウントだと**分析処理のバグが本番デバイスの設定を壊せる状態**になります。実装コストはユーザーを 1 つ増やすだけです。

> ⚠️⚠️ **「拠点横断 READ」はスナップショット画像の取得権限を含みます** 〈rev.3 で追加〉。
>
> §6.8 制約 6 により **`Events READ` ＝ 添付画像の DL 権限**です。したがって **R-06a（運用知識基盤・読取）と R-11（オフロードバッチ）は、全拠点の顔画像を証跡なしで取得できます。** §11.3 の警告は R-04a / R-04b と S-5 しか名指ししていませんでしたが、**実際にはこちらのほうが範囲が広い**（拠点横断のため）。
>
> ⚠️ **CU-18 で案 (D) を採る場合はさらに注意が要ります。** (D) の 403 は「グローバルの `ROLE_INVENTORY_READ` / `ROLE_BINARY_READ` を持たない」ことが前提ですが、**R-06a は定義上 Inventory の拠点横断 READ を持ちます**。実装がグローバルロールに落ちると **(D) の隔離はこの 1 アカウントで破れます**。
>
> **[判] 拠点横断 READ を持つサービスユーザー（R-05 / R-06a / R-11）には、次を規約とします**:
>
> 1. **`ROLE_BINARY_READ` / `ROLE_BINARY_ADMIN` を付与しない**
> 2. **Inventory 権限は root 拠点グループへの Inventory ロールで与え、グローバル `ROLE_INVENTORY_READ` を使わない**（⚠️ **SV-47 の結果次第では Inventory ロールが `childDevices` に届かず、グローバルへ逃げる圧力がかかります。CU-18 の前提条件として SV-47 の結論を先に得てください**）
> 3. **R-05「最小限の READ」は具体的な permission 一覧に展開する**（現状は未定義）
>
> → **CK-8(b) / AS-14(b) の検査対象**（§2.8 / §13.7）

> ⚠️ **R-10（保守・AI モデル）は `ROLE_INVENTORY_ADMIN` を持ちます** 〈rev.3 で追加〉。CU-18 で案 (D) を採ると**画像は AI モデルと同じファイルリポジトリに入る**ため、**R-10 は画像を読み・差し替え・削除できます**。しかも R-10 は業務上まさにそのリポジトリを触る主体です。**`type` による分離は権限境界ではありません。** SV-46（ソフトウェアバイナリ書込に本当に `ROLE_INVENTORY_ADMIN` が要るか）の実機確認と併せて、**`ROLE_INVENTORY_CREATE` 等への降格**を検討してください。R-06b も同じ理由で検査対象です。

> ⚠️ **サービスアカウントの復旧手段は「削除 → 同名で再作成」です**（§12.8）。**ロール・Inventory ロール・アプリケーションアクセスの再割当が必要**になります（ユーザーを削除すると割当も消えるため）。**投入スクリプトが冪等なら、再実行がそのまま復旧手段になります。**
>
> **[確] ユーザー ID は変わりません。** Cumulocity のユーザー ID は**ユーザー名そのもの**です（OpenAPI の example: `self: 'https://<TENANT_DOMAIN>/user/{tenantId}/users/jdoe'` / `id: jdoe` / `userName: jdoe`）。rev.1 は「ID が変わるため再割当が必要」としていましたが、理由が誤りでした。
>
> **したがって CI 側で同期が必要なのは ID ではなくパスワードです。** 再作成時に新しいパスワードが発行されるため、**CI シークレットストアの更新を復旧手順に必ず含めてください**（[担当範囲] RB-9〈rev.4 訂正 — rev.3 までは RB-11（バッファ溢れ対応）を誤って参照していました〉）。ID を参照している設定は書き換え不要です。

### 11.5 break-glass と管理者アカウント

| ID | 対象 | 内容 |
|---|---|---|
| **R-07** | **break-glass（両テナント）** | SSO 対象外の緊急用ローカル管理者。**TFA + 資格情報の封緘保管 + 使用時アラート** |
| — | `management` テナントの `admin` | **エスクロー保管が必須** |
| — | `edge` テナントの `admin` | 同上 |

**[判] 必須の統制**（§12.8 の制約から導かれます）:

| # | 統制 |
|---|---|
| 1 | **両テナントの admin と break-glass の資格情報を封緘してエスクロー保管**（保管場所・開封手順・開封記録） |
| 2 | **四半期ごとにログイン可能性を確認する**（必要になる前に有効性を確かめる） |
| 3 | **管理者アカウントのパスワード有効期限は無期限にする**。期限切れは自分で変更できるので致命的ではないが、**期限切れに気づかないまま緊急時を迎える**事故を避ける |
| 4 | **サービスアカウントについてはエスクロー不要**（削除 → 再作成で復旧できるため）。CI シークレットストアでの管理を継続 |

> ⚠️⚠️ **残る唯一の復旧不能ケース**: **両テナントの admin と break-glass の資格情報を全て失った場合**です。ほかに管理者が居なければ、削除・再作成を実行する主体そのものが居ません。**単一 Edge に全拠点が集約されている構成では影響が全拠点に及びます。**

### 11.6 SSO（Keycloak）連携とロールマッピング

`POST /tenant/loginOptions`（メディアタイプ **`application/vnd.com.nsn.cumulocity.authconfig+json`**、小文字）で完全に API 投入できます [確]。必要ロールは `ROLE_TENANT_ADMIN` **OR** `ROLE_TENANT_MANAGEMENT_ADMIN`。

**主要フィールド**:

| フィールド | 本構成での役割 |
|---|---|
| `type: OAUTH2` / `grantType: AUTHORIZATION_CODE` | Keycloak は **OAuth2 authorization code grant のみ**（**SAML 非対応**）[確] |
| `issuer` / `clientId` / `audience` | Keycloak 担当から受領する入力値 |
| `signatureVerificationConfig.jwks.jwksUrl` | **署名鍵は RSA のみ** [確]。閉域網では **Edge → JWKS 到達性**が必要 |
| `accessTokenToUserDataMapping.*` / `userIdConfig.jwtField` | ユーザー識別 |
| `onNewUser.dynamicMapping.mappings[]` | **グローバルロール（R-01〜R-03, R-10）とアプリアクセスの割当** |
| ⭐ `onNewUser.dynamicMapping.inventoryMappings[]` | **拠点 Inventory ロールの割当 ＝ 拠点分離の実装点** |
| `sessionConfiguration` | セッション |

> ⚠️⚠️ **`id` の扱いは POST と PUT で正反対です** [確]。rev.1 は「必ず除去」としていましたが、PUT では逆に必須です。
>
> | 操作 | スキーマ | `id` |
> |---|---|---|
> | `POST /tenant/loginOptions` | `allOf: [authConfig, {properties: {id: {readOnly: true}}}]` | **除去する** |
> | `PUT /tenant/loginOptions/{typeOrId}` | `allOf: [authConfig, {required: [id]}]` | **必須**（example にも `id: 924997e5-…`） |
>
> **[判] 冪等化手順を次に改めます**（[投入ガイド] パターン B の「PUT 先行」とは順序が逆になります）:
>
> 1. `GET /tenant/loginOptions` で既存の `id` を取得
> 2. 見つかれば **`id` を載せて PUT**
> 3. 見つからなければ **`id` を除いて POST**

#### ⚠️ 必須ガード

| # | ガード | 内容 |
|---|---|---|
| **1** | ログインモードで SSO を選ぶと**ログイン画面から Basic Auth / OAI-Secure の選択肢が消える**（*"Single sign-on redirect - … If selected, will remove Basic Auth and OAI-Secure login options."*）。⚠️ **`management` テナントの保護は無条件ではありません**: 公式が述べるのは *"**If external communication to the Management tenant has been blocked**, then … the authentication method is automatically set to 'Basic authentication'"* だけです。**この条件が成立するかを §12.3 で確認してください** | [確] |
| **1b** | **切替のたびに強制ログアウトされる**（*"Each time you change the login mode you will be forced to log out."*）。**切替手順に「別ブラウザ／別セッションで復旧経路を開いたまま作業する」ことを明記** | [確] |
| **2** | **`edge` テナントにも break-glass（R-07）を残す。** ガード 1 は `management` しか守らず、しかも条件付き。⚠️ **他人のローカルユーザーのパスワードは管理者でも変更できない**（§12.8）ため、**「作っておく」だけでなく有効性を定期確認する** | [確] |
| **2b** | ⚠️ **break-glass の TFA は SSO redirect 下では使えません。** *"Note that the **TOTP method is only available with the login mode 'OAI-Secure'**."* — SSO redirect にすると OAI-Secure が消えるため、**R-07 の TFA 要件が成立しません。** 代替の第 2 要素（物理的なシークレット保管・複数名承認）を §11.5 で決めること | **[要 SV-48]** |
| **3** | 再割当ポリシーの既定は「毎ログインで全ロール再計算（ルール外はクリア）」。**手動編集は次回ログインで上書きされる** | [確] |
| **4** | どのルールにもマッチしないと `access denied`（デフォルト拒否）。**クレーム名の 1 文字違いで全ユーザーがログイン不可**になる | [確] |
| **5** | **デバイス認証とサービスユーザーは SSO 対象外** | [確] |
| **6** | **順序**: 先に Edge 側でロールを定義 → 対応するグループを Keycloak で作る | [判] |
| **7** | ⚠️⚠️ **[要 SV-04] SSO からローカル認証へ戻す手順を、実機で検証して runbook 化すること**。**API 経路の一部は確定しました**が、決定的な穴が残っています（下記） | **最優先** |
| **8** | ⭐ **切替前ゲート**: SSO ユーザー 1 名で実際にログインでき、期待するロールと Inventory ロールが付与されることを、**ログインモードを切り替えずに**確認する | [判] |

> ⚠️⚠️ **[要 SV-04] ガード 7 の核心 — 「ログインモード」の実体が API 仕様に見当たりません。**
>
> - `PUT /tenant/loginOptions/{typeOrId}` は実在し、`security` に `Basic: []` を含むので **Basic 認証で叩けます** [確]
> - **しかし `authConfig` スキーマに「preferred login mode」に相当するフィールドがありません。** `visibleOnLoginPage` / `onlyManagementTenantAccess` はありますが該当せず、生 spec を `loginMode` / `preferred.*login` で検索してもヒットしません
>
> **つまり「ログインモード」は別のテナントオプションである可能性が高く、`/tenant/loginOptions` の PUT だけでは戻せないおそれがあります。** これが SV-04 の本当のリスクです。
>
> **検証手順に次を追加してください**:
> 1. SSO を有効化する**前**に `GET /tenant/options` で全カテゴリ・全キーをダンプして保存する
> 2. SSO 有効化**後**に同じダンプを取り、**差分からログインモードに相当するキーを特定する**
> 3. そのキーを Basic 認証で書き戻せることを実機で確認する
>
> **[判] 上記 1 のダンプ取得を、SSO 有効化の作業手順に必須ステップとして組み込んでください。** 事後には差分が取れません。

> ⚠️ **併せて `BasicAuthenticationRestrictions` に注意してください** [確]。`forbiddenClients` / `forbiddenUserAgents` がスキーマ化されており、これらが設定されていると **Basic 認証での復旧そのものが塞がれます**。上記 1 のダンプで現在値を確認しておくこと。

### 11.7 ⚠️ 権限設計の禁則

| # | 禁則 | 理由 |
|---|---|---|
| **AX-1** | **`ROLE_NOTIFICATION_2_ADMIN` を R-09 以外に付与しない** | §10.3。**CI で機械検査する**（§2.8 CK-6） |
| **AX-2** | **案件アプリ・基盤標準アプリの REST 呼び出しでサービスユーザー代行を使わない**（D-e） | Notification 2.0 と同型の RBAC バイパスになる |
| **AX-3** | **Inventory ロールをデバイス個別に手動割当しない** | 次回ログインで `inventoryMappings` に上書きされる |
| **AX-4** | **Cloud Remote Access のロールを誰にも割り当てない** | §4.7 |
| **AX-5** | **`ROLE_INVENTORY_UPDATE` を使わない**（存在しない） | 投入スクリプトが失敗する |
| **AX-6** | **`credentials.*` をテナントオプションに置かない** | §4.10 G-14 |
| **AX-7** | **拠点横断の書込権限を常時保持するアカウントを作らない** | §4.10 G-2 |

### 11.8 外部アプリ登録と CORS

**VM2 のアプリケーションが Cumulocity の REST を叩くには、ロールだけでは足りません。**

| ID | 設定 | 内容 |
|---|---|---|
| **P-01** | **紐づけ確認アプリの登録** | ⚠️ **VM2 ホストのため `createHostedApplication` は不適**。**`type: EXTERNAL` のアプリケーション登録**（アプリスイッチャーとアクセス制御に載せるため）。**[要 SV-24]** Cumulocity にバンドルを載せる案との二択 |
| **P-02** | **テナントのアプリ購読** | `POST /tenant/tenants/{t}/applications` |
| **T-02** | ⭐ **CORS 許可オリジン** | テナントオプション `access.control` / `allow.origin`。**既定 `*` からの絞り込み**。⚠️ **必要ロールは `ROLE_OPTION_MANAGEMENT_ADMIN`**（R-01 に追加済み・§11.2） |
| **P-03** | 標準ダッシュボード | Cockpit の JSON エクスポート/インポート。**Edge の Cockpit に "Managing exports — Export data to either CSV or Excel files." は含まれる** [確]。**[要 SV-11]** ダッシュボード定義そのもののエクスポート可否は別途確認 |

> **[確] `allow.origin` の仕様** — Core OpenAPI `POST /tenant/options` "Default option categories":
>
> > **access.control** / `allow.origin` / 既定 `*` / *"**Comma separated list of domains allowed for execution of CORS. Wildcards are allowed (for example, `*.cumulocity.com`)**"*
>
> ⚠️ **rev.1 の「スキーム + ホスト + ポートで完全一致」は公式の記述と食い違います。** 公式は「**ドメインのカンマ区切り**」「**ワイルドカード可**」としか述べていません。**実際に受け付ける表記（スキームやポートを含められるか）は実機で確認してください**（**[要 CU-8b]**）。
>
> **[判] ワイルドカードが使えるため、VM2 のアプリを共通ドメイン配下に置けば `*.example.internal` の 1 エントリで済みます。** オリジンを 1 つずつ列挙する運用（アプリ追加のたびに設定変更）を避けられるので、**VM2 側のドメイン設計と合わせて決めてください。**

> ⚠️ **CORS のオリジン一覧が未確定だと、アプリ側が「認証は通るが API が呼べない」状態になります。** VM2 の各アプリ担当からオリジンのリストを受領してください。

> ⚠️ **運用知識基盤（S-8/S-9）はサーバサイド常駐プロセスなので CORS は不要です。** ブラウザから叩くアプリだけが対象です。

### 11.9 API 利用規約

**[判] Cumulocity の REST を叩く全主体が守る規約です。IF 仕様書として配布します。**

| # | 規約 | 理由 |
|---|---|---|
| **AP-a** | ⭐ **ページングは `prev` / `next` リンクで辿る。ページ番号を自前で加算しない** | `acl.algorithm-version` の `OPTIMIZED` は一致件数が閾値（既定 2000）未満のときだけ適用され、超えると `LEGACY` に落ちる [確K]。*"navigation links via 'prev' and 'next' will work properly and this should be the only way of iterating through multiple pages"* |
| **AP-b** | **エンドユーザーの OIDC トークンを使う**（D-e） | §11.7 AX-2 |
| **AP-c** | **Operational Store への直接アクセスは行わない**（REST API 経由のみ） | 構成図の配置の要点 |
| **AP-d** | **通知トークンは既定 1440 分（24 時間）で失効する。再取得ロジックを実装する** | §10.4 要件 3。⚠️ **既定値であって上限ではない**（プロキシ側で短縮する）。⚠️ **失効しても接続中のコンシューマは切断されない** |
| **AP-g** | ⭐ **WebSocket はアイドル 5 分で切断される。未 ack 1000 件で配信が止まる。再接続と ack を必ず実装する** | §10.4 要件 7。**ack を怠るとバックログ 25 MiB に達し、その拠点のデータ投入が 500 で失敗します**（§10.5） |
| **AP-e** | **c8y-proxy 経由の呼び出しは散発的に 401 を返す。リトライを実装する** | [確]（§6.8 制約 4） |
| **AP-f** | **`type` フィルタ・`fragmentType` フィルタで絞ってから取得する** | 全件走査は Edge の負荷になる（§7.7） |

### 11.10 否定テストによる検証

**[判] 権限設計は「できること」ではなく「できないこと」を検証してください。**

| # | 否定テスト | 合格条件 | §14 |
|---|---|---|---|
| 1 | 拠点 A のユーザー資格情報で拠点 B のデータを **API 直叩き** | 403 / 404、または**空コレクション** | CT-5 |
| 1b | ⭐ **拠点 A のユーザーで `GET /alarm/alarms`（絞り込みなし）を叩く** | **自拠点のアラームだけが返る**（全テナント分が返らない） | CT-5 |
| 1c | ⭐ **拠点 A のユーザーで `GET /inventory/managedObjects`（絞り込みなし）を叩く** | **自拠点配下の MO だけが返る** | CT-5 |
| 1d | ⭐ **拠点 A の Manager が、拠点 A の main device 配下の「カメラ」のアラームを取得できる** | 取得できる（**届かなければ SV-47 が NG** → §11.3） | CT-4 |
| 2 | 拠点 A の案件アプリの認証情報でトークン発行プロキシに**拠点 B のサブスクリプション**を要求 | 拒否される | CT-7 |
| 3 | 拠点 A 用トークンで拠点 B のトピックに WebSocket 接続 | 接続できない | CT-7 |
| 4 | `ROLE_NOTIFICATION_2_ADMIN` の付与先を機械検査 | R-09 以外に付いていない | CT-6 |
| 5 | 手動付与した Inventory ロールが次回ログインで消えること | 消える（＝ `inventoryMappings` が正） | CT-26 |
| 6 | Cumulocity UI に Remote access のボタンが出ないこと | 出ない | CT-17 |
| 7 | 拠点 B の案件アプリに拠点 A のアラームが**届かない**こと | 届かない | CT-8 |
| 8 | `fragmentsToCopy` で除外したフラグメントが通知に**含まれない**こと | 含まれない | CT-8 |

> ⚠️⚠️ **テスト 1b / 1c は rev.1 のロール定義では必ず失敗しました。** グローバルの `Alarms READ` / `Inventory READ` があると **テナント全体が返る**ためです（§11.2）。rev.2 でグローバル権限を削減したことで初めて意味のあるテストになります。**ロール改訂と否定テストはセットで実施してください。**

### 11.11 ⚠️⚠️ 画像配信マイクロサービス（R-14）の認可仕様 〈rev.3 で追加〉

**⚠️ CU-18 で (A) / (A′) / (D) / (B) のいずれかを採る場合、本節は必須要件です。決定前でも先に読んでください — この要件を落とすと、案 (D) の権限分離は「直叩きは 403、正規経路は全開」になります。**

> ⚠️⚠️ **[判] マイクロサービスは confused deputy になります。**
>
> §15.3 案 (D) の「標準の権限モデルで拠点分離できる」は、**直叩き経路に限った話**です。**正規の閲覧経路は `ROLE_BINARY_READ` を持つ R-14 で、そこには Cumulocity の拠点分離が一切効きません。** R-14 が「この利用者はこのバイナリを見てよいか」を**自前で判定しなければ、拠点 A のユーザーが拠点 B の `binaryId` を投げれば 200 が返ります。** 監査レコードは「拠点 A の人が見た」と正しく残るだけで、**アクセスは成功します。**
>
> **これは (A) / (A′) / (B) でも同じ構造です**（配信主体が特権を持ち、拠点分離を自前で再現する必要がある）。

**[判] R-14 が満たすべき要件**:

| # | 要件 | 理由 |
|---|---|---|
| **MS-a** | **エンドユーザーの OIDC トークンを検証し、その利用者の Inventory ロール割当を Cumulocity に問い合わせて認可判定する** | R-14 のサービスユーザー権限で判定してはいけない（それが confused deputy） |
| **MS-b** | **binary → 拠点の判定方法を定義する。** ⚠️ CK-8(a) によりバイナリ MO はどのグループにも属さないので、**グループからは引けません** | 逆引き経路を決めないと実装できない |
| **MS-c** | ⚠️⚠️ **binary → event → camera → group の逆引きに依存しないこと。** `EVENT` リテンション 90 日でイベントが先に消えると、**バイナリは「どの拠点のものか判定できない孤児」になります**（(D) はバイナリにリテンションが効かないため必ず発生する） | **[判] バイナリ MO 自身に `x_Site.siteId` を持たせてください。**グループには入れないので CK-8(a) と両立します |
| **MS-d** | **判定不能なときは fail-close（403）** | 孤児バイナリを黙って配信しない |
| **MS-e** | ⚠️ **監査レコードの `user` は、検証済み OIDC トークンの `userIdConfig.jwtField` からのみ導出する。リクエストボディ／ヘッダ由来の値を使わない** | `auditRecord.user` は `readOnly` ではない [確] ため、**呼出側の値をそのまま入れる実装だと任意の名前になりすませます**。AS-13 で守ったはずの改竄不能性が実装 1 行で無効化されます |
| **MS-f** | ⚠️ **アプリケーションログに userId / binaryId / 画像バイト列を出さない** | 出すと **`ROLE_AUDIT_READ` の保護外・`AUDIT` リテンションの外**に、平文で第 2 のアクセス記録ができます。**§15.3 で ingress アクセスログ案を却下した理由③とまったく同じ問題**が自作マイクロサービスで再発します |
| **MS-g** | **監査レコードの時刻は `creationTime`（プラットフォーム付与）を正とする。`time` は事象時刻の参考値** | `time` は writable、`creationTime` は `readOnly` [確]。**書き手が決める値は証跡になりません** |

> ⚠️ **否定テスト CT-5d（§14.2）が本節の合格判定です。** 「拠点 A のユーザーが拠点 B の `binaryId` を R-14 に要求 → 403」を確認できるまで、(D) の拠点分離は成立していません。

---

## 12. Cumulocity Edge セットアップ

> **本章は「設置から設定投入を始められる状態まで」（§12.1〜§12.9）と、稼働後の定常運用・計画作業（§12.6 バックアップ・§12.7 監視・§12.10 運用設計 〈rev.4〉）を扱います。設定項目そのものは §13 です。実施手順の詳細は [設定書] §5 P0〜P1 が正です。**

### 12.1 セットアップの全体像

```
①  ネットワーク前提の確定  ── install 後に直せない（§12.2）
②  ドメイン 2 つ + TLS 証明書 + ライセンス + registry credentials の受領
③  容量サイジング（§7.6）── storageClassName は install 後変更不可
④  検証環境 Edge の構築  ── 前提 2〜8（§0.7）をここで潰す 〈rev.3 で範囲拡大〉
                              ＋ CT-5b / CT-5c（前提 9 の (D)/(B) 評価）
⑤  本番 Edge のインストール  ── 冪等化不可・1 回きり
⑥  閉域網固有の設定（CA / no_proxy）
⑦  マイクロサービス稼働確認（Apama-ctrl / Smartrule / Messaging Service）
⑧  admin パスワード変更 + エスクロー保管
⑨  スナップショット取得 + リストア試験  ← ここを通るまで §13 に進まない
──────────────────────────────────────────
    §13 設定投入へ
```

### 12.2 ⚠️ install 前に確定すべき項目

**⑤に進む前に必ず潰してください。**

> ⚠️⚠️ **rev.1 は本節を「install 後に変更できない項目」としていましたが、公式に照らすと変更**できる**ものが多く含まれていました。** Edge の全ページを `immutable` / `cannot be changed` で横断検索した結果、**真に変更不可と明記されているのは `spec.storageClassName` だけ**です。
>
> | 項目 | 実際 | 根拠 |
> |---|---|---|
> | `storageClassName` | ❌ **変更不可** | *"This value is used only during the Edge installation and **can't be changed for existing installations**."* |
> | MongoDB 容量 | ⚠️ **増やせる。減らせない** | *"Once Edge is installed, **you can only increase this value, but cannot reduce**."* |
> | **ドメイン** | ✅ **変更可能** | *"**You can later update the domain and license** to match your environment by following the steps outlined in Modifying Edge."* |
> | **TLS 証明書** | ✅ **変更可能** | `c8yedge config --set-file tlsSecret.tls.key=<path> --set-file tlsSecret.tls.crt=<path>` / CR の `spec.tlsSecretName` |
> | **ライセンス** | ✅ **変更可能** | `c8yedge config --set-file licenseKey=<path>` / CR の `spec.licenseKey` |
> | CPU / RAM | ✅ **増設可能** | *"**You can add more CPU cores or RAM to the host at any time**, and Edge will use the additional resources automatically without further configuration."* |
>
> **この誤りの実害**: **TLS 証明書の期限更新という不可避の定期作業が「禁止事項」に分類されていました。** 手順は §12.5 に追記してあります。

| # | 項目 | 内容 | 確度 |
|---|---|---|---|
| **P-0-1** | **CIDR 衝突** | K3s の Service / Pod CIDR と社内網アドレス空間の重複確認。**オンプレ閉域網は 10.0.0.0/8 が一般的で衝突しやすい**。衝突すると「Web UI は動くが SSO だけ落ちる」のような散発的症状が出る | [推] |
| **P-0-2** | **公開ポート** | `cumulocity-ontoplb`（LoadBalancer）: **443, 8443, 1883, 8883**。Edge operator メトリクス: **3443**。*"Edge requires that your Kubernetes cluster does not have an Ingress provider ... enabled on common ports"* | [確] |
| **P-0-3** | **時刻同期（NTP）** | 社内 NTP を Edge ホスト・Keycloak・画像解析装置・外部Gateway に設定（§6.9 TM-c） | [推] |
| **P-0-4** | **到達性マトリクス** | Edge → 社内 DNS / Keycloak(JWKS) / オブジェクトストレージ / Genetec、拠点網 → Edge:8883,443、**案件アプリ(VM2) → Edge の WebSocket**（§10.1）、保守VPN → Edge | [推] |
| **P-0-5** | **ハードウェア最小要件** | CPU **8 コア** / RAM **16 GB** / Disk **150 GB**。**MongoDB は AVX 命令 + x86-64-v3 以降が必須**（`lscpu` で確認）。⚠️ **CPU/RAM は後から増設可能**（AVX / x86-64-v3 は CPU 交換になるので事前確認必須） | [確] |
| **P-0-6** | ⭐⭐ **`storageClassName` の確定** — **本節で唯一の真に不可逆な項目** | *"Once the `storageClassName` field is configured in the Edge custom resource (CR), it cannot be changed."* | [確] |
| **P-0-7** | **MongoDB 容量の初期値** | 既定 75GB。**増やせるが減らせない**ため、**やや小さめから始めて増設する**運用も選べる（§7.6） | [確] |
| **P-0-8** 〈rev.3〉 | ⭐ **仮想化基盤とスナップショット機能** | **VM1 の仮想化基盤（またはストレージ）が「ディスク全体の原子スナップショット」を提供するか。** ⚠️ **提供しないなら、バックアップは「Edge を停止してコールドコピー」しか選べず、取得のたびに全拠点が停止します**（§12.6.2 / CU-33）。⚠️ **`/var/lib/rancher/k3s` が複数ボリュームにまたがる場合、それらを同時点でスナップショットできること** | **[要確認]** |
| **P-2** | **ドメイン 2 つと TLS 証明書の SAN** | §12.3。**変更は可能だが波及が大きい**（下記） | [確] |
| **P-3** | **ライセンス**（ドメインに紐づく） | *"the license key must always be valid for the domain name, so any change of domain name should be made simultaneously with a change of license key."* **ドメインと同時に差し替えること** | [確] |

> ⚠️⚠️ **ドメイン変更は「不可能」ではなく「波及が極めて大きい」です。** `c8yedge config --set domain=` で変更できますが、**TLS 証明書の再発行、ライセンスの同時差し替え、Keycloak の redirect URI / backchannel logout URL、全 thin-edge の `c8y.url` と `c8y.mqtt`、全ブラウザの信頼設定**を巻き込みます。**デバイス接続開始後に行うと拠点に人を出す作業になります。** 事実上は事前確定すべき項目として扱ってください。

> ⚠️ **[要 SV-49] `admin` パスワードを「インストール後に独立して変更可能」とする根拠が見つかりません**（§12.3）。逆に Edge CR は `spec.email` について *"Changes made to the administrator email using the user interface or Cumulocity API **will be overwritten by the Edge operator**"* と述べており、**operator が reconcile するモデル**です。パスワードにも同様の上書きが働く可能性があるため、**実機で確認してください**（SV-37 の復旧手順と併せて）。

> ⚠️ **[要 SV-50] ライセンスの有効期限・更新サイクルは公式に記述が見つかりません。** 証明書と同種の「日付起点リスク」になりうるため、**ベンダーに書面で確認し、期限があるなら §12.7 のメタ監視に加えてください。**

### 12.3 2 テナント構成

**Edge は必ず `management` と `edge` の 2 テナントを持ちます** [確]。

| テナント | URL | 用途 | 本構成での使い方 |
|---|---|---|---|
| **edge** | `https://<domain>` | 業務データ・デバイス・ユーザーの本体 | **§13 の設定の大半はこちら** |
| **management** | `https://management-<domain>` | Edge プラットフォーム設定・**ブランディング** | ブランディングのみ（メールサーバーは変-2 により設定しない） |

- 両テナントともユーザー名は **`admin`**。初期パスワードは `c8yedge --cumulocity-password` または Edge CR の `spec.cumulocityPasswordSecretName`（Secret のキー名は **`C8Y_ADMIN_PASSWORD`**、8 文字以上）。**インストール後に独立して変更可能**
- **閉域網 DNS に両ホスト名の登録が必要**

> ⚠️⚠️ **TLS 証明書は両ホスト名をカバーすること。ワイルドカードは 1 ラベル階層しかカバーしません** [確]。
>
> - `*.iot.com` は `myown.iot.com` と `management-myown.iot.com` の**両方をカバー**
> - `*.myown.iot.com` を買うと `management-myown.iot.com` は**カバーされない**
> - **[判] 推奨**: **SAN に 2 つのホスト名を明示列挙**した証明書を社内 CA で発行。PEM 形式・完全なチェーンを正しい順序で

> **「Edge はシングルテナント」の正確な意味**: 比較表の *"Multi-tenancy | No; single tenant"* は**サブテナントを作れない**という意味です [確]。同じドキュメントが *"Edge has two tenants, management and edge"* とも述べています。**拠点ごとのサブテナント分離は不可能で、D17（グループ + Inventory ロール）が唯一の代替です。**

### 12.4 インストールと初期確認

**[判] デプロイ形態は `c8yedge` CLI（K3s 同梱）を採用します。**

| 選択肢 | 内容 | 評価 |
|---|---|---|
| **(a) `c8yedge` CLI（K3s 同梱）** | `sudo c8yedge install`。オフラインは `c8yedge package` → `sudo c8yedge install -s "<OFFLINE-PACKAGE>"` | **✅ 採用**。閉域網でエアギャップ導入が可能 |
| (b) 持ち込み K8s + Helm | `helm upgrade --install c8yedge-operator oci://registry.c8y.io/edge/helm-charts/...` → Edge CR 作成 | 非推奨。**Kubernetes 1.34.x のみ**、**単一ノードクラスタのみサポート** |

**インストール後の必須確認**:

| # | 確認 | 完了条件 |
|---|---|---|
| 1 | 両テナント URL でログイン可能 | `https://<domain>` と `https://management-<domain>` |
| 2 | Edge CR のエクスポートと Git コミット | `kubectl get --namespace=c8yedge edge/c8yedge -o yaml`。**`status`, `metadata.{uid,resourceVersion,creationTimestamp}` を除去** |
| 3 | admin パスワードの変更（両テナント独立） | 変更後に両テナントでログイン可能 |
| 4 | ⭐ **両テナント admin と break-glass のエスクロー保管** | 封緘・保管場所・開封手順が文書化され、実際に保管されている |
| 5 | **マイクロサービス配備の確認と是正** | **Apama-ctrl / Smartrule** が稼働（§8.7 の前提）。`kubectl get pods -n c8yedge` |
| 6 | ⭐ **Messaging Service の稼働確認** | **Notification 2.0 の前提**（§10.1）。2026 Edge には `Included` だが**実際に動いていることを確認** |
| 7 | ⭐ **スナップショット取得 + リストア試験** | **試験成功まで §13 の設定投入に進まない** |

> **2026 Edge に `Included` と明記されているもの** [確]: Messaging Service / MQTT Service / Microservice-based data broker / Microservice Hosting / **Streaming Analytics** / OPC UA。**Data Hub のみ "On request via Professional Services"**（§7.5）。

### 12.5 閉域網固有の設定

| ID | 設定 | 内容 | 確度 |
|---|---|---|---|
| **E-11** | ⭐ **`c8yedge-operator-config` ConfigMap** | **閉域網で必須**。`ca.crt`（信頼する追加 CA の PEM バンドル）と `no_proxy`（**両テナントドメイン + Pod CIDR + Service CIDR を必ず含める**）。**適用後は operator の再起動が必要** | [確] |
| — | **社内 CA ルート証明書の配布** | 全ブラウザ・**全 thin-edge デバイス**・**Edge operator** が Edge を信頼する | [確] |
| — | **デバイス側の信頼ストア** | thin-edge の `c8y.root_cert_path`（MQTT）と **`c8y.proxy.ca_path`（HTTP）は別キー** [確]。⚠️ **既定はどちらもシステム既定（`/etc/ssl/certs`）** なので、「両方を明示設定しないと失敗する」わけではありません。**実際の失敗モードは「片方だけカスタムパスを指定したとき」**です | [確] |

#### ⭐ Edge サーバー証明書の更新手順 〈rev.2 で追加〉

**`c8yedge` の場合** [確]:

```bash
sudo c8yedge config --set-file tlsSecret.tls.key=<path-to-key> \
                    --set-file tlsSecret.tls.crt=<path-to-cert>
```

**自前 K8s の場合** [確]: `kubectl create secret tls edge-tls-secret --cert=<crt> --key=<key>` → Edge CR の `spec.tlsSecretName` を更新して `kubectl apply`

> ⚠️⚠️ **rev.1 は §12.2 で「TLS は install 後に変更できない」と誤記していたため、この定期作業の手順が本書から欠落していました。** 証明書の有効期限は必ず来ます。**§12.7 のメタ監視に「Edge サーバー証明書（`<domain>` / `management-<domain>`）の残存期間」を必ず加え、更新を runbook 化してください**（⚠️ 〈rev.4 訂正〉rev.3 までは [担当範囲] RB-13 を参照していましたが、**RB-13 は R-08 アカウントのライフサイクル管理であり別物**です。**TLS 証明書更新の runbook は [担当範囲] §10.3 の一覧に存在しません** — §12.10.6 の追補一覧から [担当範囲] へ起票してください）。失効すると**全拠点・全案件アプリの HTTPS / WebSocket が同時に全断**します。

> ⚠️ **証明書・社内 CA のローテーション手順が必要です**（SV-22）。社内 CA ルートを更新すると **thin-edge 側の信頼ストア更新が必要**で、数百台 × N 拠点が同時に接続不能になり得ます。しかも**配布経路（Cumulocity 経由のソフトウェア更新）も同時に使えません**。**新旧 CA を並行して信頼させる移行期間**を設計に入れてください。

### 12.6 バックアップ／リストア 〈rev.3 で全面設計・rev.4 で完成（推奨値と runbook）〉

**D17 により全拠点が 1 台の Edge に集約されています。バックアップは本構成で唯一の全損対策です。**

#### 12.6.1 ⚠️⚠️ 公式が提供しているのは「復元手順」だけです

**[確] 公式ドキュメント（Edge operations の Backup and recovery）の構成**:

| ステップ | 公式の記述 | 評価 |
|---|---|---|
| **Step 1 — 保全対象** | *"The following directories must be preserved for disaster recovery. **Implement appropriate data protection measures (for example, backups or redundant storage)** to safeguard these directories:"* → **`/var/lib/rancher/k3s`（1 ディレクトリのみ）** | ⚠️⚠️ **取得方法は「組織の責務」として丸投げされています。これが本節を設計しなければならない理由です** |
| **Step 2 — 復元先の準備** | 同一（または互換）OS を用意／**`K3s` と Edge のデータが存在しないクリーンなディスク**であること／バックアップを対象機から利用可能にする（*"For **snapshot-based or storage-level backups**, ensure the snapshot is accessible"* とあり、スナップショット方式は想定されている） | [確] |
| **Step 3 — データ復元** | *"Restore the backup so that the directories land in their original locations, **preserving paths, ownership, and permissions**. **Omitting directories or restoring to incorrect locations can corrupt the cluster.**"* → `ls -ld /var/lib/rancher/k3s` で確認 | [確] |
| **Step 4 — 同一版で再インストール** | `sudo c8yedge install --version <previous_version>`（閉域は `-s c8yedge.tar`）。*"The installer detects that Cumulocity Edge is already installed and **waits for the recovery to complete** before exiting."* 成功時に `Edge recovered successfully.` が出る | [確] |
| — | ⚠️ *"Installing a different Edge version on top of a restored data set is **unsupported** and may fail the upgrade guard rails."* | [確]（P-0 / E-07） |
| **自前 K8s の場合** | *"Ensure that **Persistent Volume Claims (PVCs) and the Edge custom resource configuration** are included in your backup strategy."* | [確]。本構成は `c8yedge`（K3s）なので上の行が適用されます |

> ⚠️⚠️ **[判] 公式の runbook は「create a consistent backup」と述べていますが、Step 1 に一貫性を担保する手順がありません。**
>
> **`c8yedge` CLI に backup / stop / start / quiesce に相当するコマンドは存在しません** [確]。存在するのは **`install` / `package` / `config` / `upgrade` / `uninstall` の 5 つだけ**です〈rev.4 訂正 — rev.3 は `package`（閉域用オフラインパッケージ生成・§12.4）を数え落として「4 つ」と書いていました。backup / stop / quiesce が無いという結論は変わりません〉。
>
> **否定の探索経路**: ① Edge operations ページ（Backup and recovery 節）の全文を取得して確認 → 取得コマンドの記載なし ② Edge manage ページの `c8yedge` サブコマンド一覧を確認 → `config` / `upgrade` / `uninstall` のみ（サービス停止・保守モードの記載なし。サービス操作は `kubectl rollout restart` 等で行う旨のみ）③ Edge installing ページを確認 → 取得手順の記載なし。**この 3 経路をもって「取得側は製品機能として提供されていない」と判断しています。**
>
> **したがって「どうやって整合の取れたコピーを作るか」は本書で決める必要があります** → **12.6.2 / [要決定] CU-33**。

#### 12.6.2 ⚠️⚠️ 取得方式 — [要決定] CU-33

**ここが本節の中心です。`/var/lib/rancher/k3s` には次が同居しています** 〈rev.4 で K3s 公式に基づき配下パスを特定〉:

| 中身 | 場所（`/var/lib/rancher/k3s` 配下） | 一貫性の観点 |
|---|---|---|
| **MongoDB のデータ**（local-path の Persistent Volume。既定 75GB・P-0-7） | `storage/`（local-path PV の既定パス [確]） | ⚠️ 書き込み中のコピーは破損しうる |
| **K3s の datastore**（クラスタ状態そのもの。**シングルノードの既定は SQLite** [確]） | `server/db/` | ⚠️ 同上。壊れると *"can corrupt the cluster"* |
| ⭐ **K3s server token** 〈rev.4〉 | `server/token` | **datastore 内の機密データはこの token で暗号化されています。** K3s 公式: *"If you do not use the same token value when restoring, the snapshot will be unusable, as the token is used to encrypt confidential data within the datastore itself."* [確] → **`server/db` だけ取って token を欠くバックアップは復元不能**。公式保全対象 `/var/lib/rancher/k3s` を丸ごと取れば両方含まれます |
| containerd のイメージ・その他 PV | `agent/` ほか | 容量の大半 |

> ⭐ **「1 ディレクトリで足りる」ことの裏付け** 〈rev.4〉: K3s 公式は SQLite datastore について *"No special commands are required to back up or restore the SQLite datastore. To back up the SQLite datastore, take a copy of `/var/lib/rancher/k3s/server/db/`."* と述べており [確]、datastore・token・local-path PV のすべてが `/var/lib/rancher/k3s` 配下にあります。Edge 公式の保全対象指定と整合します。⚠️ ただし `/etc/rancher/k3s/`（K3s 設定）は保全対象に**含まれていません**（Edge 公式ページに `etc/rancher` の言及 0 件・rev.4 で全文確認）— 復元手順が「クリーン OS + 再インストール」であり、**インストーラが再生成する前提** [推] です。

**[確] MongoDB 公式の要件**（WiredTiger）:

> *"The database must be valid when the snapshot takes place. This means that all writes accepted by the database need to be fully written to disk: either to the journal or to data files."*
> *"For the WiredTiger storage engine, the data files reflect a consistent state **as of the last checkpoint**. Checkpoints occur with every 2 GB of data or every minute."*
> *"MongoDB supports volume-level back up ... **when the MongoDB instance's data files and journal files reside on separate volumes**. However, to create a coherent backup, **the database must be locked** and all writes ... suspended during the backup process."*

> ⚠️ 〈rev.4〉**「2 GB ごとまたは 1 分ごと」という一次情報は MongoDB 公式内で記述が割れています。** 上記引用元（backup-with-filesystem-snapshots チュートリアル）には現在もこの一文がありますが、MongoDB 公式の WiredTiger リファレンスページ（無バージョン最新版・v7.0/v8.0/v8.2 いずれも）は *"MongoDB configures WiredTiger to create checkpoints...at intervals of 60 seconds."* とのみ記載し、2GB 条件への言及がありません（rev.4 で複数バージョンを横断確認）。**本書の結論（クラッシュコンシステントなスナップショットは概ね直近 1 分以内のチェックポイント状態に整合する）はどちらの記述でも成立するため実害はありませんが、[確] として引用する一次情報が内部で不整合を起こしている点は認識しておいてください。**

**[判] 上記から、取得方式は次の 3 択になります。**

| 案 | 方式 | 一貫性 | ダウンタイム | 評価 |
|---|---|---|---|---|
| ❌ **(i) 稼働中に `tar` / `rsync` でファイルコピー** | ディレクトリを走査してコピー | ⚠️⚠️ **無い。** コピーに時間がかかる間もファイルは更新され続けるため、**MongoDB も K3s datastore も「途中の状態が混ざったコピー」になります** | なし | ⚠️ **採ってはいけません。** 「バックアップを取っている」つもりで**復元できないものを溜める**のが最悪の結果です |
| ⭐ **(ii) VM / ストレージレベルの原子スナップショット** | VM1 のディスクを**一瞬で**スナップショット | **[推] 有効**。データとジャーナルが同一ボリュームにあるため、スナップショット時点で「journal または data files に書き終わったもの」が揃い、**起動時のジャーナル再生で整合します**。K3s datastore も同時点 | **ほぼ無し** | **本命。** ⚠️ **成立条件が 2 つあります**（下記） |
| **(iii) Edge を停止してコールドコピー** | K3s を停止 → ファイルコピー → 起動 | **確実** | ⚠️ **コピー時間ぶん全拠点が停止**（150GB 級） | **(ii) の成立条件が満たせない場合の代替。** ⚠️ **停止コマンドが公式に無い**ため、K3s の systemd サービス停止で代用することになり、**手順自体が非公式**になります |

> ⚠️⚠️ **案 (ii) の成立条件**:
>
> | # | 条件 | 未確認なら |
> |---|---|---|
> | **1** | **VM1 の仮想化基盤が「ディスク全体の原子スナップショット」を提供すること**。⚠️ **`/var/lib/rancher/k3s` が複数ボリュームに分かれている場合、それらを同時点でスナップショットできること**（またぐと整合が壊れます） | 案 (iii) へ |
> | **2** | **Edge の MongoDB が data files と journal を同一ボリュームに置いていること** [推]〈rev.4 で根拠を補強〉。**Edge CR の MongoDB 用 PVC は `mongod-data-edge-db-rs0-0` の 1 個のみ** [確] で、**CRD に journal を別ボリュームへ分ける設定は存在しません**（CRD ページ全文に `journal` の出現 0 件・rev.4 で確認）。WiredTiger の既定配置は dbPath 配下の `journal/` サブディレクトリなので**同一 PV に同居する**と判断できます。ただし公式に journal の配置を明言した記述は無いため [確] には上げません。⚠️ **別ボリュームなら MongoDB 公式は `db.fsyncLock()` を要求しており、案 (ii) 単独では不十分になります** | **`db.fsyncLock()` を挟む手順が要る。ただし §7.1 DS-c（Operational Store への直接アクセス不可）と抵触するため、代替として案 (iii)** |
>
> **[要決定] CU-33（§15.3）** — **①取得方式（(ii)/(iii)）②取得頻度 ③保持世代数 ④保管先**。**②③④は RPO と保管容量、そして CU-30（個人情報の破棄）に直結します。**

> ⚠️ **VM1 の仮想化基盤が本書に記載されていません。** → **P-0-8「仮想化基盤とスナップショット機能の確認」として §12.2 に追加済み**〈rev.4 で本文の記載を追い付かせた — rev.3 は「追加してください」のままでした〉。**P-0-8 の確認結果が CU-33 の入力です。**

#### 12.6.2b [判] CU-33 への回答 — 推奨設計 〈rev.4 で追加〉

**rev.3 は CU-33 を「決めてください」で止めていました。本節で ①〜④ の推奨値を設計として提示します。** P-0-8（仮想化基盤の確認）と CT-36（負荷下の一貫性実証）が通るまでは**仮決定**ですが、**値が無いと保管先の調達も監視のしきい値も決まらない**ため、先に値を置きます。

| 決定事項 | **[判] 推奨** | 根拠 |
|---|---|---|
| **①取得方式** | ⭐ **案 (ii) VM 全ディスクの原子スナップショット → 保管先へエクスポート → スナップショット即削除**。P-0-8 不成立時のみ案 (iii)（コールドコピー・下記手順） | 公式が *"For snapshot-based or storage-level backups, ensure the snapshot is accessible and ready for restore"* とスナップショット方式を復元手順の前提として明示的に許容 [確]。**installing-edge も VM イメージのエクスポートに好意的**（*"the advantage of most VM technologies is that a running image can be exported to be run in another environment without further configuration"* [確]） |
| **①-2 静止化** | **crash-consistent（無停止・ゲスト静止化なし）を基本とする**。ハイパーバイザのゲスト静止化（quiesce / fs-freeze）は**あれば使うが、成立要件にはしない** | MongoDB は journal 再生で整合 [確]（12.6.2）、K3s の SQLite も同様のクラッシュ回復を持つ [推]。**CT-36 が crash-consistent の実証試験そのもの** |
| **②取得頻度** | **日次 1 回（データ流量が最少の時間帯）＋ イベント駆動 4 点**〈rev.4 訂正 — UP-c（§12.10.2）と重複していた「アップグレード完了後」を正式に4点目として編入〉: (a) `c8yedge upgrade` 直前（BK-c）(b) `c8yedge upgrade` 完了直後・新版で1世代（BK-c／UP-c と同一事象） (c) §13 の大規模設定投入の直前 (d) テナント CA 作成直後（BK-l・下記） | 名目 RPO = 24h。⭐ **実効損失はデータ種別で非対称**（下記） |
| **③保持世代** | **日次 7 世代 ＋ 週次 4 世代 ＋ 直近アップグレード前 1 世代**（≒ 最大 12 世代） | ⚠️ **遡及日数はローテーション方式に依存し、本節だけでは確定しない**（12.6.3 BK-m に生成規則と実際の遡及日数を定義） |
| **④保管先** | **Edge ホスト（VM1）と別障害ドメインの保管先**（別筐体・別ラック電源。可能なら別建屋）。**容量は実ディスクサイズ×12世代を上限に確保し、P-0-5 の最小要件 150GB では計画しない**（⚠️ 〈rev.4 訂正〉150GB は P-0-5 のハードウェア**最小**要件であり、実際の調達サイズや P-0-7 による MongoDB 増設後のサイズとは別物。実サイズ確定後に本節の容量を再計算すること。増設時の再計算トリガは §12.10.1「容量・スケールの再算定」に統合済み）。暗号化・アクセス制御は 12.6.6 | 同一筐体・同一ストレージへの保管は「全損対策」にならない（D17 — これが唯一の全損対策）。転送帯域は RTO ④（実ディスクサイズ分の転送）に直結するため、**保管先選定時に復元方向の転送時間も見積もること** |

> ⭐ **RPO 24h の実効的な意味 — 失われるものの非対称** 〈rev.4〉:
>
> | データ | T-Δ〜障害発生の間の変化 |
> |---|---|
> | 計測・イベント（デバイス発） | ⭐ **thin-edge のストア&フォワードが再送するため、バッファ保持範囲内なら埋まる**（RS-2 / SV-44） |
> | 設定（リテンション・ロール・ルール等） | ⭐ **Git から再投入できる**（BK-g） |
> | ⚠️ **Cumulocity 内でしか起きない状態変化** — アラームの CLEARED、オペレーションの進行状態、監査レコード、ユーザー操作、Notification 2.0 のコンシューマ位置 | ⚠️ **丸ごと失われます。どの経路でも再現できません。** 24h ぶんのこれらを失えないなら頻度を上げる（ただし取得中の I/O 負荷と保管容量が線形に増える） |

**取得手順（案 (ii)）**〈rev.4 訂正 — 旧称「runbook RB-BK1」は削除。[担当範囲]の RB-nn 体系と字面が衝突し、§12.10.6 が自ら禁じた「本書側での runbook 採番」を先取りしてしまっていたため。正式な runbook 番号は [担当範囲] 側での採番（§12.10.6）を待つ〉:

```
0. 前提: P-0-8 確認済み / 保管先に到達可能 / 空き容量 ≥ 1 世代ぶん
1. VM1 の全ディスクを原子スナップショット（crash-consistent。複数ディスク構成なら同時点で）
2. スナップショットから保管先へフルイメージをエクスポート
   （ファイル単位で取りたい場合もイメージをマウントして /var/lib/rancher/k3s を
     paths / ownership / permissions を保持して取り出す — 復元 Step 3 の要件 [確]）
3. ⚠️ スナップショットを即時削除（残置禁止 — BK-k）
4. manifest を世代に併置（BK-b の実体）:
   取得日時 / Edge バージョン / 方式 / サイズ / チェックサム /
   Edge CR エクスポート（§12.4 確認 2 と同じ整形）/ 対応するインストーラ一式の所在（BK-j）
5. チェックサム検証（エクスポート直後に 1 回）
6. 監視へ報告: 最新世代の取得時刻・世代数・保管先空き容量（§12.7 ⑮ の入力）
```

> [要実機確認] **Edge バージョンの機械的な取得方法**（manifest 用）。インストーラ出力 *"running version 2025.0.X"* [確] が一次候補だが、稼働中に取るコマンドは W1 で確定すること（Edge CR / `c8yedge` の版表示の実体確認）。

**取得手順（案 (iii)・P-0-8 不成立時の代替）** 〈rev.4 で K3s 公式手順に基づき具体化 — rev.3 の「手順自体が非公式」を更新。**Edge としての停止手順は依然として公式提供が無い**が、K3s の停止は K3s 公式手順 [確] に乗る〉:

```
1. sudo systemctl stop k3s          # ⚠️ スーパーバイザが止まるだけ。コンテナ（MongoDB 含む）は動き続ける [確]
2. sudo /usr/local/bin/k3s-killall.sh
   # 全コンテナ停止 + containerd リセット。"The cluster data will not be deleted." [確]
3. tar --numeric-owner --acls --xattrs -cpf <保管先>/k3s-<日付>.tar /var/lib/rancher/k3s
4. sudo systemctl start k3s
5. 起動確認: 両テナント URL でログイン / kubectl get pods -n c8yedge が全て Running
6. manifest・チェックサム・監視報告（案 (ii) の 4〜6 と同じ）
```

> ⚠️ **案 (iii) の停止時間 = 手順 1〜4 の間、全拠点が Edge に書けません。** 150GB 級のコピーは転送先性能次第で**数十分〜時間単位**。mosquitto 既定 1000 件のバッファ（SV-44）を超えた拠点はデータが欠落します。**案 (iii) を常用する場合は、取得頻度を週次に落とすか、バッファ増量（[担当範囲]）とセットで決めること。**

> ⭐ **復元経路は 2 本になります**（12.6.4b で詳述）: フルイメージを取っているので、**「VM 復元」— VM イメージごと戻す**（仮想化基盤が健在な場合。OS 準備・再インストールが不要になり **RTO の ③④⑤ が大幅に縮む**）と、**「クリーン復元」— 公式手順どおりクリーン OS ＋ データ復元 ＋ `c8yedge install --version`**（基盤ごと失った場合・別筐体への復元）の両方が同じ成果物から実行できます。⚠️ 〈rev.4 訂正〉旧称「(R-A)/(R-B)」は削除（既存のグローバルロール ID R-01〜R-16 と同一プレフィックスで紛らわしいため）。**CT-35 は少なくとも「クリーン復元」で実施してください** — こちらだけ試験していないと、基盤全損時に初見の手順をぶっつけ本番でやることになります。**「VM 復元」も別途 CT-35b として年 1 回試験すること**（12.6.4b）。

#### 12.6.3 [判] バックアップ規約

| # | 規約 | 理由 |
|---|---|---|
| **BK-a** | ⚠️⚠️ **稼働中のファイル単位コピー（`tar` / `rsync`）を「バックアップ」と呼ばない** | 一貫性が無く、**復元できないものを溜めることになります**（12.6.2 案 (i)） |
| **BK-b** | **取得した世代ごとに Edge のバージョンを記録する** | **復元は同一版必須** [確]。版が分からないバックアップは**復元できません** |
| **BK-c** | ⚠️ **`c8yedge upgrade` の直前に必ず取得する** | アップグレード失敗時の唯一の戻り道。⚠️ **アップグレード後のバックアップでしか新版には戻れず、旧版のバックアップは旧版でしか復元できません**（BK-b） |
| **BK-d** | **バックアップは暗号化して保管する** | **中身は MongoDB 全体 ＝ 顔画像・監査レコード・特徴量に紐づく external ID**（12.6.6 / CU-30） |
| **BK-e** | ⚠️ **保持世代数と破棄手順を決め、破棄を実行する** | **リテンション（§7.2）と削除バッチ（X-04）はバックアップに及びません。** 破棄しない限り個人情報は残り続けます |
| **BK-f** | **復元手順を runbook 化し、定期的に実施する**（[担当範囲] RB-7） | ⚠️ **取得できていることと復元できることは別です。** 本構成の復元は「クリーン OS + 再インストール」なので**手順が長く、ぶっつけ本番では失敗します** |
| **BK-g** | ⚠️ **設定は Git を正とし、バックアップに依存しない**（§13.1 / AS-1） | **設定（リテンション・ロール・スマートルール等）は Git 定義から再投入できます。** バックアップが必要なのは**再現できないもの＝実データ**です。この二重化は本構成の強みなので、**「バックアップが古い」場合でも設定だけは最新に戻せる**ことを runbook に明記してください |
| **BK-h** | ⚠️ **バックアップの取得成功・世代数・最新取得時刻を監視する**（§12.7 ⑮） | **静かに止まるのがバックアップの典型的な壊れ方**です。⚠️ **判定は「ジョブの成否」ではなく「最新バックアップの経過時間 > RPO」**にしてください（削除バッチ ⑪ と同じ考え方） |
| **BK-i** | ⚠️ **部分復元はできない前提で設計する** | 保全対象はクラスタ全体です。**「1 拠点だけ」「1 デバイスだけ」戻すことはできません**。個別復旧が要る事象（誤削除等）は、**バックアップではなく §13 の Git 定義と再投入で対処する**設計にしてください |
| **BK-j** 〈rev.4〉 | ⭐ **世代ごとに、同一バージョンのインストーラ一式（`c8yedge` バイナリ ＋ 閉域用オフラインパッケージ `c8yedge.tar`）を保管先に併置する** | 復元は **`sudo c8yedge install --version <previous_version>`（閉域は `-s c8yedge.tar`）** [確] ＝ **当該版のインストーラが無いと復元できません**。⚠️ しかも **アップグレードは連続メジャー間のみ** [確]（*"Edge supports upgrades only between consecutive major releases"*）で、**ロールバック・ダウングレード手順は存在しません**（rev.4 で manage-edge 全文を `rollback|downgrade` で確認・0 件）。**閉域網では障害発生後にダウンロードで取り寄せることもできません** |
| **BK-k** 〈rev.4〉 | ⚠️ **ハイパーバイザ上のスナップショット残置を「バックアップ」と呼ばない。エクスポート完了後は即時削除する** | スナップショットは**同一ストレージ上の差分**であり、①ストレージ全損に対して無力 ②チェーンが伸びるほど I/O 性能が劣化し統合失敗リスクが増える。**保管先へエクスポートされて初めて 1 世代**です（BK-a と同じ「つもりバックアップ」の亜種） |
| **BK-l** 〈rev.4〉 | ⭐ **テナント CA 作成（§13.3.5 D-10）の直後・最初のデバイス登録の前に、必ず 1 世代取得する** | **CA の実体（証明書＋秘密鍵）はテナントデータ内** [確]（*"The private key is stored in an encrypted tenant option."*）。**CA 作成より古い世代にしか戻れない場合、CA が消えて全デバイスの証明書が検証不能**になり、全拠点の再エンロールになります（12.6.4 RS-1）。この 1 世代が「デバイス登録を始めてよい」ゲートです |
| **BK-m** 〈rev.4〉 | ⭐ **③の「日次7世代＋週次4世代」は GFS 方式（週次は日次バックアップのうち週内の1本を長期保持ラベルへ昇格・複製ではない）とする。遡及日数は最大約28日（週次4世代分）で、日次7世代はこの範囲に包含される** | 〈rev.4 訂正 — レビューで「遡及約35日」の内訳計算が本文に無いと指摘され検算。日次7日と週次28日を単純加算した35日は、両者が重複しない設計を前提とした場合にのみ成立するが、その前提は明記されていなかった。**標準的なGFS方式（週次が日次の範囲を包含する）を正とし、28日に訂正する。** 12.6.2b③・12.6.6 の日数もこれに統一 |

#### 12.6.4 ⚠️⚠️ リストア後に起きること — 「戻せば元通り」ではありません

**[判] リストアは時刻を T-Δ に巻き戻します。デバイス・案件アプリ・外部システムは巻き戻りません。** この非対称から次が発生します。**runbook に復旧後手順として組み込んでください。**

| # | 事象 | 何が起きるか | 対応 |
|---|---|---|---|
| **RS-1** | ⚠️⚠️ **T-Δ 以降に発行されたデバイス証明書がテナントに存在しない** | §3 は certificate-authority（EST）でデバイスを登録します。**巻き戻すと、その間に登録・更新された拠点のデバイスが接続できなくなる可能性があります**（前提 3 / SV-08 に依存）。⭐ 〈rev.4 で見通し改善〉**再接続できる見込みが立ちました [推]**: ①CA の実体はテナントデータ内 ＝ **バックアップに含まれ、リストアで同一 CA が戻る** [確] ②接続時の照合は **trusted CA へのチェーン検証** [確] なので、CA が同一なら T-Δ 以降に発行された証明書も検証は通る ③CA 機能は **auto-registration が既定で ON** [確]（*"The certificate is stored in the trusted certificate repository with auto-registration turned on by default."*）なので、**消えたデバイスユーザーは初回 MQTT コールで自動再作成される** [確]。⚠️ 前提が 2 つ: **CA 作成が T-Δ より前**（→ BK-l）、**auto-registration を OFF にしていない**（→ **[判] OFF にしないこと**。OFF だと T-Δ 以降のデバイスは登録レコードが無く接続不能） | **[要検証] CU-34** — 上記 [推] を **CT-35(c) で実測**して確定。⚠️ **T-Δ 以降に作られた MO・child device 構成・ツイン**は失われたままなので、**thin-edge 側の再登録挙動（エンティティ再作成）も同時に観測すること**。再エンロールが要る結論なら拠点に人を出すことになり RTO に直結します |
| **RS-2** | **thin-edge が網断中と同じ挙動で再送する** | ストア&フォワードにより、**T-Δ 以降のイベント・計測が再送されます**。⭐ **`time` が事象時刻で固定されていれば（§6.9 TM-a）データは正しく埋まります** | **TM-a が守られていることが復旧品質を決めます。** ⚠️ **mosquitto 既定 1000 件を超えた分は欠落します**（SV-44）→ Δ の上限を決める根拠 |
| **RS-3** | ⚠️ **アラーム状態が巻き戻る** | CLEARED にしたアラームが ACTIVE に戻ります。**未対応アラームが再出現し、運用者は「対応済みのものを再対応」します** | 復旧後に**アラーム棚卸し**を runbook 手順に入れる |
| **RS-4** | ⚠️ **PENDING オペレーションが再実行される** | T-Δ 時点で PENDING だったオペレーション（モデル配布・設定配布）が**もう一度デバイスへ流れます** | ⚠️ **復旧直後は §4.10 G-1 のキルスイッチを先に引き、棚卸ししてから解除する**手順にしてください |
| **RS-5** | ⚠️ **Notification 2.0 のサブスクリプションとコンシューマ位置が巻き戻る** | 案件アプリへ**重複配信**が起きます。逆に、購読を解除したはずのコンシューマが復活します | 復旧後に §10.5 の棚卸し（N-04 / CT-30）を実施。**案件アプリ側の冪等性が前提になる**ことを [連携仕様] に明記 |
| **RS-6** | ⚠️⚠️ **リテンションで消えたはずのデータが復活する** | 90 日を過ぎて削除されたイベント・画像・監査レコードが**戻ります**。⚠️ **個人情報の保存期間の観点では違反になりえます** | **復旧後にリテンションが再走して消えることを確認する**（§7.2 の削除は定期実行）。⚠️ **「戻した直後は保存期間超過のデータが存在する」ことを法務と合意しておく**（CU-30） |
| **RS-7** | ⚠️ **監査レコードが巻き戻る** | 「Audit API に `PUT`/`DELETE` が無いので改竄不能」（§15.3）という性質は**リストア経路では成立しません** | **リストアの実施自体を、Cumulocity の外側（§12.7 の監視系）に記録する** |
| **RS-8** | **ライセンス・ドメイン・TLS はバックアップに含まれる** | 復元先で**同じドメインを名乗る**必要があります（ライセンスはドメインに紐づく・P-3） | 復元先の DNS / 証明書を runbook に含める |
| **RS-9** | ⚠️ **画像バイナリとイベントの整合**（CU-18 で案 (D) を採る場合） | バイナリ MO とイベントは同じ MongoDB にあるため**同時に巻き戻ります**（整合は保たれる）。⚠️ **ただしオブジェクトストレージ（案 (A)/(A′)）は巻き戻りません** → **Cumulocity 側の参照 ID とストレージ実体がずれます** | 案 (A)/(A′) を採る場合、**復旧後に参照 ID の突合を行う**手順が要る |

#### 12.6.4b 復元手順の骨子 〈rev.4 で追加 — BK-f の「runbook 化」の実体。旧称「runbook RB-BK2」は削除（理由は §12.6.2b 訂正注記と同じ）〉

**発動判断** [判]: リストアは**最後の手段**です。次の順で当てはまるものを先に使ってください。

| 事象 | 対処 |
|---|---|
| 設定の誤投入・誤変更 | **Git 定義から再投入**（BK-g / §13）。リストアしない |
| 個別データの誤削除 | **復旧不能と扱う**（BK-i — 部分復元はできない。全体を巻き戻すコストと RS-1〜9 に見合わない） |
| Edge が起動しない・データ破損・ホスト/ディスク全損 | **リストア発動** |

**復元経路の選択**〈rev.4 訂正 — 旧称「(R-A)/(R-B)」を「VM 復元」/「クリーン復元」に改称。R-01〜R-16 と紛らわしいラベルの解消に加え、「VM 復元」に手順が無くCT-35でも試験されないという指摘を受け、手順を追記した〉:

```
仮想化基盤は健在か?
 ├─ はい → 【VM 復元】
 │          ⭐ OS 準備・再インストール不要。RTO ③④⑤ が縮む
 │          ⚠️ 戻した直後の VM は T-Δ 時点の自分として起動する
 │          1. 保管先の最新世代（BK-k によりハイパーバイザ上のスナップショットは
 │             既に削除済みのため、使う実体は「エクスポート済みフルイメージ」）を
 │             仮想化基盤へインポート
 │          2. 新規 VM として登録。ネットワーク設定・ホスト名・IP を元の VM1 と一致させる
 │             （manifest の Edge CR エクスポート・§12.4 確認 2 と突き合わせる）
 │          3. 起動 → K3s / Edge サービスが自動起動することを確認（CU-40 と同じ確認観点）
 │          4. 両テナント URL でログイン確認
 └─ いいえ（基盤ごと全損・別筐体へ復元）
          → 【クリーン復元】（公式手順）:
            1. 同一（または互換 [確]）OS をクリーンインストール
            2. K3s / Edge のデータが残っていないことを確認 [確]
            3. バックアップから /var/lib/rancher/k3s を復元
               — paths / ownership / permissions を保持 [確]。ls -ld /var/lib/rancher/k3s で確認
            4. sudo c8yedge install --version <manifest記載の版>   # 閉域: -s c8yedge.tar（BK-j で併置済み）
            5. "Edge recovered successfully." を確認 [確]
```

> [要実機確認] 【VM 復元】の手順 1〜2（イメージインポート・VM 定義の再現手順）は仮想化基盤の製品によって具体コマンドが変わるため、P-0-8 の確認（仮想化基盤の特定）と同時に手順書へ落とし込むこと。

**復旧後チェックリスト**（RS-1〜RS-9 の消化順。**順序に意味があります**）:

| 順 | 作業 | 対応 RS |
|---|---|---|
| 1 | ⚠️ **§4.10 G-1 のキルスイッチを先に引く**（PENDING オペレーションの再実行を止める） | RS-4 |
| 2 | **リストア実施を Cumulocity の外側（§12.7 の監視系・運用台帳）に記録する**（リストア自体は Edge 内の監査に残らない） | RS-7 |
| 3 | 両テナント URL でログイン / `kubectl get pods -n c8yedge` 全 Running / DNS・TLS が復元先を向いている | RS-8 |
| 4 | **デバイス再接続の確認** — 特に T-Δ 以降にエンロールした拠点（CU-34 の実証値がここで効く） | RS-1 / RS-2 |
| 5 | **オペレーション棚卸し** → 問題ないものだけ再開し、キルスイッチを解除 | RS-4 |
| 6 | **アラーム棚卸し**（CLEARED が ACTIVE に戻っている） | RS-3 |
| 7 | **Notification 2.0 の購読棚卸し**（§10.5 N-04 / CT-30）＋ **案件アプリへ「重複配信が起きる」旨を連絡** | RS-5 |
| 8 | **リテンションの再走を確認**（保存期間超過データが消えること。消えるまでの一時超過は CU-30 で法務合意済みであること） | RS-6 |
| 9 | （案 (A)/(A′) 採用時）**オブジェクトストレージとの参照 ID 突合** | RS-9 |
| 10 | ⭐ **バックアップジョブを再開し、復旧完了時点で新規 1 世代を取得する**（復旧直後が最も無防備） | — |

> [判] **リストア試験（CT-35）は年 1 回以上 ＋ Edge アップグレードの都度、R-B 経路で実施**してください。試験のたびに本 runbook の所要時間を計測し、12.6.5 の RTO 実測値を更新すること（[担当範囲] RB-7 / W2-8）。

#### 12.6.5 RTO / RPO の算定（SV-29 / TE-10 の入力）

**[判] 本構成の RTO は「復旧作業時間」であって「サービス再開時間」ではありません。**

```
RPO ＝ バックアップ取得間隔（＋ 案 (ii) なら最後のチェックポイントまでの誤差）
       ⚠️ ただし thin-edge のストア&フォワードが効く範囲では、実効的なデータ損失は
          RPO ではなく「バッファ保持時間を超えた分」に縮む（RS-2）
          → ⭐ バッファ容量（SV-44）が実効 RPO を決める。ここが本構成の特徴

RTO ＝ ①障害検知      （§12.7 の監視で決まる）
     ＋ ②判断・招集
     ＋ ③OS 準備      （クリーンな状態にする。⚠️ 既存データがあってはいけない）
     ＋ ④データ復元    （150GB 級の転送時間）
     ＋ ⑤c8yedge install（公式ログ例では update 完了まで 3 分程度だが、要実測）
     ＋ ⑥復旧後手順    （RS-1〜RS-9。⚠️ RS-1 が「拠点に人を出す」になると桁が変わる）
```

> ⚠️⚠️ **③④が効くため、RTO は分単位にはなりません。** 「Edge 停止許容時間」を BOX / thin-edge のバッファ保持時間の実測から決める（SV-29 / TE-10）とき、**RTO がそれを超えるならバッファを増やすか、冗長構成を検討する**ことになります。⚠️ **Edge は単一サーバ構成で HA がありません** [確] ので、**冗長化は「もう 1 台用意して手動で切り替える」以上のことはできません。**

> ⚠️ **⑥の RS-1（証明書）が RTO の支配項になりうる**点に注意してください。CU-34 の結論次第で RTO が数時間から数日に変わります。

#### 12.6.6 ⚠️⚠️ バックアップは個人情報を含みます

保全対象は `/var/lib/rancher/k3s` 丸ごと ＝ **MongoDB 全体**です。したがって **スナップショット画像も監査レコードも、特徴量に紐づく external ID も入ります。**

| # | 帰結 |
|---|---|
| **1** | ⚠️ **リテンションや削除バッチ（X-04）で画像を消しても、バックアップには残ります。** 「保存期間を守っている」と言えるのは**バックアップの保持世代と破棄手順まで設計したとき**だけです（§7.4 / BK-e） |
| **2** | ⚠️ **リストアすると監査レコードが巻き戻ります**（RS-7）。「改竄不能」はリストア経路では成立しません |
| **3** | ⚠️ **媒体の持ち出しは Cumulocity の監査ログに一切残りません。** S-15（Edge ホスト管理者）の統制と一体で考える必要があります（§11.1 / CU-29） |
| **4** | ⚠️ **復旧直後は保存期間を超過したデータが一時的に存在します**（RS-6） |

**[要決定] CU-30（§15.3）** — **暗号化・保管場所・アクセス制御・保持世代・破棄手順**を決め、**CU-5（保存義務期間）の適用範囲にバックアップを含めるかを法務に確認してください。**

**[判] CU-30 への推奨案** 〈rev.4 で追加 — 法務確認の叩き台。確認が済むまで仮決定〉:

| 論点 | 推奨 | 理由・注意 |
|---|---|---|
| **暗号化** | **保管先ストレージで at-rest 暗号化**（LUKS などのボリューム暗号化、またはバックアップ製品の暗号化機能）。転送経路も暗号化（閉域内でも平文で流さない） | バックアップ処理自体に暗号化を組み込むより、**保管先で一括して掛けるほうが手順が壊れにくい**。⚠️ 世代ファイル単位の暗号化を選ぶ場合、**鍵を失った世代は復元不能** — 復元試験（CT-35）は**必ず暗号化込みで**実施すること |
| **鍵管理** | **§12.4 確認 4 の admin エスクローと同じ体系で、別封筒として保管**。鍵の交換（ローテーション）時は**旧鍵で読める世代が残っている間、旧鍵を破棄しない** | 鍵とデータを同じ場所に置かない。**鍵の喪失 ＝ 全世代の喪失**であり、バックアップの単一障害点は媒体ではなく鍵になる |
| **アクセス制御** | **保管先へアクセスできる者を名簿化し、S-15（Edge ホスト管理者）の統制（CU-29）と同一の枠組みに載せる**。取り出し（リストア・試験・監査）は **Cumulocity の外側の台帳に記録** | **媒体の持ち出しは Cumulocity の監査ログに一切残りません**（帰結 3）。バックアップは「MongoDB 全体のオフライン写し」なので、**アクセス統制の水準は本番 DB と同格**が原則 |
| **破棄** | **世代ローテーション（12.6.2b ③）による自動削除を一次手段**とし、**媒体の交換・廃棄時は暗号消去（鍵の破棄）または物理破壊**。破棄の実施も台帳に記録 | BK-e の実体。**「ローテーションから漏れた手動コピー」が最大の抜け穴** — アップグレード前の臨時世代（BK-c）や試験用コピーも台帳・破棄の対象に含めること |
| **法務確認の論点**（確認リスト） | ① CU-5（保存義務期間）の適用範囲に**バックアップを含めるか** ② **最悪残存期間 ＝「元データの保存期間 ＋ 世代保持日数（最大約 28 日・12.6.2b ③ / BK-m）」**を許容するか ③ リストア直後に**保存期間超過データが一時的に復活する**（RS-6）ことの扱い ④ 特定個人からの**削除請求にバックアップ内は応じられない**（BK-i と同型 — 部分削除も不可能）ことの扱い | ⚠️〈rev.4 訂正〉②③④は**「12.6.2b の推奨設計（世代保持12・個人情報を含む方式）を採る限り」回避不能**という条件付きの主張であり、無条件の技術的不可能ではない。**回避策は存在する**（世代保持を短くする、または CU-18 (A) 系を採ってバックアップから個人情報を除く）が、**いずれも RPO の悪化・画像アクセス監査の要件放棄という別のコストを伴うため、本節では採らずに提示する**。**「現行推奨設計を前提にすると、このコストは技術的に避けられない」ということを法務へ提示するのが本節の役割** |

#### 12.6.7 検証

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-35** 〈rev.3〉 | ⭐⭐ **リストア試験**（§12.4 確認 7 / [担当範囲] W2-8） | ⚠️ **「復元コマンドが完走した」を合格条件にしないでください。**<br>**(a)** 別筐体（クリーン OS）へ復元し、`Edge recovered successfully.` を確認<br>**(b)** **両テナント URL でログインでき、デバイス・グループ・リテンションルール・スマートルールが復元前と一致する**<br>**(c)** ⭐ **実デバイス（thin-edge）1 台が復元後の Edge に再接続でき、データが流れる**（RS-1 の実証）<br>**(d)** **復元に要した時間を計測し、RTO の実測値とする**（12.6.5 ③④⑤）<br>**(e)** **RS-3 / RS-4 / RS-5 が実際に起きることを観測し、復旧後手順に反映する** |
| **CT-36** 〈rev.3〉 | ⭐ **バックアップ一貫性の検証**（CU-33 の方式決定後） | **負荷をかけた状態**（デバイスからデータを流しながら）で取得したバックアップから復元し、**MongoDB と K3s の両方が正常起動すること。**<br>⚠️ **静止状態で取ったバックアップだけで試験すると、案 (i) の欠陥を見逃します** |
| **CT-35b** 〈rev.4〉 | **【VM 復元】経路の試験**（12.6.4b。年 1 回、CT-35 とは別日程で） | 仮想化基盤上でイメージインポート → VM 定義再現 → 起動 → 両テナントログインまで完走すること。**手順 1〜2 の具体コマンドを runbook に固定化できていること**（[要実機確認]の解消）。⚠️ **CT-35 が「クリーン復元」のみを試験対象とするため、より高速な「VM 復元」を無試験のまま本番に温存しない** |

> ⚠️ **CT-35 の成功を §13 に進むゲートにしてください**（§12.4 確認 7）。**[要 SV-29] RTO/RPO を数値で決め**、BOX と thin-edge のバッファ保持時間を実測して「Edge 停止許容時間」として明記してください。

### 12.7 Edge 自身の監視

> *"The Edge operator exposes a **Prometheus-compatible metrics endpoint, `https://<domain>:3443/metrics`**"* [確]

| # | 設計 | 内容 |
|---|---|---|
| **1** | **Otel Collector の prometheus receiver で取り込む** | OTLP ネイティブ出力は未記載。これが正攻法 |
| **2** | ⭐ **アラート通知先を Cumulocity の外側に持つ** | **変-2 でメールが無いため、これが基盤異常の唯一の検知経路**（SV-34 / TB-5） |
| **3** | **監視対象に含めるもの** | ①Edge 自体の死活 ②`c8y_Application_Down` / `_Unhealthy` ③**デバイス証明書の有効期限**（§3.6）④**Notification 2.0 のサブスクライバ数・バックログ量**（**公式 UI に危険域つきの画面あり** — §10.5。**Used backlog 80% でアラート**）⑤ディスク使用率 ⑥PENDING オペレーション件数（§4.11）⑦⭐ **Edge サーバー証明書（`<domain>` / `management-<domain>`）の残存期間**（§12.5）⑧⭐ **オフロードジョブの成功／失敗とラグ**（§7.5。**サイレント失敗が 90 日続くとデータが恒久的に失われる**）⑨ **`te/errors` に出る拒否メッセージ**（§6.7）⑩ **[要 SV-50]** ライセンス期限（存在する場合）<br>〈rev.3 で追加〉⑪ ⭐ **画像削除バッチ（X-04）** — ⚠️ **判定は「バッチの成否」ではなく「`GET /inventory/binaries?type=<画像用type>` の最古 `creationTime` が保存期間を超えていないこと」**。バッチが「成功して 0 件削除」でも異常は起こりえます（§7.4）<br>⑫ **break-glass のログイン** — `GET /audit/auditRecords?type=UserAuthentication&user=<break-glass>`。§11.5 R-07 は「使用時アラート」を要求しているが、変-2 でメールが無いため**実現手段が未定**（SV-34 / TB-5 に紐づけること）<br>⑬ **マイクロサービスのデプロイ** — `type=Application` の監査レコード。**R-14 の差し替えは実質 `ROLE_AUDIT_ADMIN` の取得と同じ**（AS-15）<br>⑭ **MO の削除** — `type=Inventory`。⚠️ **R-04a の `Inventory ALL` は cascade 削除を含みます**（§11.3）<br>⑮ ⭐ **バックアップ**（§12.6 BK-h）— **判定は「ジョブの成否」ではなく「最新バックアップの経過時間 > RPO」**。併せて**世代数**と**保管先の空き容量**。⚠️ **静かに止まるのがバックアップの典型的な壊れ方**で、気づくのは復旧が必要になった瞬間です<br>〈rev.4 で追加〉⑯ **時刻ずれ** — Edge ホスト・Keycloak・デバイスの NTP 同期状態（P-0-3 は「設定せよ」だけで**ずれの検知が監視対象に入っていませんでした**。ずれると TLS 判定・JWT `exp`/`iat`・イベント突合が同時に壊れ、エラーから原因に辿り着けません — §6.9 TM-c） |
| **4** | ⚠️ **メタ監視の向き** | 構成図は `保守端末 → VM1`（pull）。**D9「自社側から拠点への着信接続は設けない」とは逆向き**。VPN 内なら D9 の例外に収まるが、**FW 審査と攻撃面の観点で設計判断として再確認が必要**（SV-23） |

> ⚠️ **[要 TB-5] 誰が刈り取り、誰に通報するかが未確定です。** これが決まらないと、**基盤自身の異常が誰にも届きません。**

### 12.8 メールサーバー不採用の帰結

**メール通知を採用しないため、メールサーバーは設定しません**（変-2）。公式はメールサーバーが 2 つの機能を担うと述べています [確]:

> *"Configuring an email server enables you to receive email notifications about events, alarms, and also to reset your password."*

**パスワードリセットへの影響はケースによって異なります**:

| ケース | メール要否 | 根拠 |
|---|---|---|
| **SSO ユーザー（Keycloak 管理）** | **不要** | Cumulocity 側にパスワードが存在しない。*"password reset in Cumulocity is disabled for users created through an external authentication server."* **リセットは Keycloak の機構で行う** |
| ローカルユーザーの**新規作成時**にパスワードを設定 | **不要** | `POST /user/{t}/users` の `password` フィールド（writeOnly・6〜32 文字）で直接指定できる |
| **自分自身**のパスワード変更 | **不要** | `PUT /user/currentUser/password` |
| ⚠️ **他人のローカルユーザーのパスワードをリセット** | ⚠️ **必要** | 下記 |

> ⚠️⚠️ **「運用側（管理者）が他ユーザーのパスワードをリセットすればよい」は、現行の Cumulocity では API 上できません** [確]。
>
> - OpenAPI（`PUT /user/{tenantId}/users/{userId}`）: *"Note that you cannot update the password or email of another user, they can only be updated for the current user."*
> - go-c8y-cli: *"In more recent Cumulocity versions, you can't set a fixed password for another user."*
>
> **つまり他人のローカルユーザーに対する唯一の正規リセット手段がメールです。**

**回避策**:

| # | 手段 | 適用対象 | 留意点 |
|---|---|---|---|
| 1 | **削除 → 同名で再作成** | **サービスアカウント**（R-05, R-06, R-08, R-09, R-11, R-12） | ユーザー削除で割当も消えるため**ロール・Inventory ロール・アプリアクセスの再割当が必要**。**ユーザー ID（＝ユーザー名）は変わらない**が、**パスワードが変わるので CI シークレットの更新が必須**。**投入スクリプトが冪等なら再実行で復旧できる** |
| 2 | **admin が複数居る状態を保つ** | 両テナントの管理者 | ただし同じ制約を受けるため、**admin 自身の資格情報は削除・再作成では救えない** |
| 3 | **資格情報のエスクロー保管** | **両テナントの admin と break-glass のみ** | §11.5 |

> ⚠️ **[要 SV-37] Edge のホスト側から admin パスワードを復旧する手段があるか**を検証環境で試す価値が高いです。初期パスワードは K8s Secret `cumulocityPasswordSecretName` 由来ですが、**インストール後に Secret を変更してパスワードが変わるかは未確認**です。**成立すればエスクローの必要性が下がります。**

### 12.9 セットアップ完了判定

**次がすべて ✅ になるまで §13 の設定投入に進まないでください。**

- [ ] §12.2 の **P-0-1〜P-0-7** / P-2 / P-3 が確定し、記録されている 〈rev.3 で修正 — P-0-7（MongoDB 容量。増やせるが減らせない）がこぼれていた〉
- [ ] §7.6 の容量サイジングが完了し、`storageClassName` と MongoDB 容量に反映されている
- [ ] 両テナント URL でログイン可能
- [ ] Edge CR がエクスポートされ Git にコミットされている
- [ ] 両テナントの admin パスワードが変更され、**エスクロー保管されている**
- [ ] 社内 CA ルートが全ブラウザ・Edge operator に配布されている
- [ ] `c8yedge-operator-config`（`ca.crt` / `no_proxy`）が適用され operator が再起動されている
- [ ] Apama-ctrl / Smartrule が稼働している
- [ ] ⭐ **Messaging Service が実際に動作している**
- [ ] ⭐⭐ **CT-35 リストア試験が成功している**（§12.6.7）⚠️ **「復元コマンドが完走した」では不十分** — 別筐体への復元・両テナントログイン・**実デバイスの再接続**・**所要時間の実測**まで<br>  ⚠️ **カメラの静止画スナップショット（§6.8）とは別物です**〈rev.3 で改称〉
- [ ] **CU-33（バックアップ取得方式）が決まり、CT-36（負荷をかけた状態での一貫性検証）が成功している** 〈rev.3〉
- [ ] **P-0-8（仮想化基盤の原子スナップショット可否）が確認されている** 〈rev.3〉
- [ ] **RS-1〜RS-9（リストア後手順）が runbook 化されている**（[担当範囲] RB-7）〈rev.3〉
- [ ] 検証環境で §0.7 の**前提 2〜8** に結論が出ている 〈rev.3 で範囲拡大〉
- [ ] **CT-5b / CT-5c を実施済み**（前提 9 / CU-18 で (D)/(B) を残す場合。§14.2）〈rev.3〉
- [ ] **前提 9（CU-18）が合意されている**、または「§13.3.6 の `AUDIT` 行を実行しない」ことが明示的に決まっている 〈rev.3〉
- [ ] **CU-36（ホスト OS のディストリビューションと版）が確定・記録されている**（§12.10.4 — 復元先の「同一または互換 OS」の判定基準になる）〈rev.4〉

### 12.10 運用設計 — 定常運用と計画作業 〈rev.4 で新設〉

**なぜこの節が要るか**: rev.4 で 3 文書（本書 / [担当範囲] / [設定書]）を横断調査した結果、運用作業が **① [担当範囲] §10.3 の runbook 一覧 ② §12.7 の監視項目 ③ 各所に散在する「定期」「runbook 化せよ」の記述** の 3 箇所に分かれ、**相互に参照されていない**ことが分かりました。その結果、**Edge 本体のアップグレード・計画停止・ホスト OS のパッチ・TLS 証明書の更新周期・ライセンス更新・アカウント棚卸し・監査ログレビューが「どこにも属さない作業」として欠落**していました。本節が Cumulocity Edge 側の運用設計の受け皿です。**担当・時期の確定は [担当範囲] が正**（§0.2）— 本節の [判] は [担当範囲] への起票内容を含みます。

#### 12.10.1 運用作業カタログ

**[判] 周期はすべて初期値の提案です。**「手順の所在」列が太字（**rev.4 で新設** / **runbook 未**）の行が、rev.4 まで設計から欠落していた作業です。

| 作業 | 周期（[判] 初期値） | 手順の所在 | 監視との対応 |
|---|---|---|---|
| バックアップ取得 | **日次** ＋ イベント駆動 4 点 | §12.6.2b（[担当範囲] RB-7 の実体・正式番号は §12.10.6 で起票） | §12.7 ⑮ |
| リストア試験（DR 訓練） | **年 1 回 ＋ アップグレードの都度**（クリーン復元経路）＋ **年 1 回**（VM 復元経路・CT-35b） | §12.6.4b（[担当範囲] RB-7） | — |
| Edge 本体アップグレード | **年 1 回以上**（連続メジャー追従） | **§12.10.2（rev.4 で新設）** | §12.7 ② |
| Edge TLS 証明書更新 | 発行有効期間に従う（**残 30 日で着手**） | §12.5（コマンド）＋ **runbook 未**（12.10.6 で起票） | §12.7 ⑦ |
| テナント CA 更新日対応（10/2） | 年 1 回 | [担当範囲] RB-4 | §12.7 ③ |
| ホスト OS パッチ | **月次確認・四半期適用**（再起動を伴うものは計画停止枠） | **§12.10.4（rev.4 で新設）** | — |
| アカウント・アクセス権棚卸し | **四半期** | **§12.10.7（rev.4 で新設）** | §12.7 ⑫ |
| 監査ログレビュー | **月次** | **§12.10.8（rev.4 で新設）** | §12.7 ⑫⑬⑭ |
| Notification コンシューマ棚卸し（N-04） | 四半期（購読の増減があれば随時） | §10.5 / [担当範囲] RB-10 | §12.7 ④ |
| インベントリ整合検査 | CI 毎回 ＋ 検査 5 は日次 | §3.10 / §13.7 AS-9 | — |
| 容量・スケールの再算定 | **半期 ＋ 拠点追加のたび ＋ MongoDB 容量増設のたび**〈rev.4 訂正〉 | §7.6 の算出手順を再実行（トリガ: §12.7 ⑤ が警告域に入ったとき）。**このとき §12.6.2b④ のバックアップ保管容量（実ディスクサイズ×12世代）も同時に再計算すること** — MongoDB は増やせるが減らせず（P-0-7）、ディスク実サイズが変わるたびにバックアップ容量計画が追随しないと保管先が慢性的に不足する | §12.7 ⑤ |
| ライセンス期限確認 | SV-50 の回答待ち（期限があるなら**残 90 日で発注**） | §12.2 P-3 ＋ **runbook 未**（12.10.6） | §12.7 ⑩ |
| シークレットローテーション | R-08 は [担当範囲] RB-13。**サービスユーザーは年 1 回** [判] | §11.4（再作成手順）＋ RB-9（CI 同期） | — |

> ⚠️ **運用体制そのもの（要員数・当番・受付窓口・SLA・エスカレーション先）は 3 文書のどこにもありません。** 本書のスコープ（Cumulocity 側設計）でもないため、ここでは設計せず **[担当範囲] への起票事項**として明示します（12.10.6）。**SV-34 / TB-5（基盤異常の通報先）は「通報を受けた人が何をするか」まで含めて決めないと閉じません。**

#### 12.10.2 Edge 本体のアップグレード運用 〈rev.4 で新設〉

**[確] 前提となる製品仕様**（rev.4 で一次確認）:

- *"Edge supports upgrades only between consecutive major releases. If your deployment is several major releases behind, you must upgrade to and validate each intermediate major release before proceeding to the target release."* — **メジャーはスキップできません**
- **ロールバック・ダウングレード手順は存在しません**（manage-edge 全文を `rollback | roll back | downgrade` で走査して 0 件）。**戻し手段は「直前バックアップへのリストア」だけ**です（BK-c）
- *"Upgrades with no downtime — No"* — **アップグレードは停止を伴います**
- コマンド: `c8yedge upgrade` / `c8yedge upgrade --version <version>` / 閉域は `c8yedge upgrade -s "<OFFLINE-PACKAGE>"`

**[判] 運用規約**:

| # | 規約 | 理由 |
|---|---|---|
| **UP-a** | **年 1 回以上、リリースに追従する**（メジャーを溜めない） | スキップ不可のため、**放置すると多段アップグレード**になり、停止時間と検証工数が段数ぶん積算します。⚠️ 追従周期の確定には**ベンダーのリリースサイクルとサポート期限の書面確認が必要** → **[要確認] CU-35**（SV-50 と同じ照会経路で） |
| **UP-b** | **検証環境で同一版・同一手順を先行し、thin-edge 互換（TE-2 のマトリクス）を確認してから本番** | 本番が初回実行にならないように。§14 の smoke（CT-1 相当）を検証環境で通す |
| **UP-c** | **直前に BK-c のバックアップ、完了後すみやかに新版で 1 世代取得**（＝ §12.6.2b②のイベント駆動 4 点の (a)(b)そのもの。ここでの UP-c は同じ規約への参照であり、別途の取得を追加するものではない）〈rev.4 訂正 — レビューで「§12.6.2bは3点と数えているのにUP-cが4点目を課しており矛盾」と指摘され、§12.6.2b②を4点に修正して整合させた〉 | **旧版バックアップは旧版でしか復元できず**（BK-b）、**アップグレード完了の瞬間から「新版の世代」が 1 つも無い状態**になるため |
| **UP-d** | **「アップグレード失敗の戻し」は障害リストアと同じ重さとして計画する** | 戻し ＝ リストア（12.6.4b）＝ **RS-1〜RS-9 が全部付いてきます**。**停止許容時間内に収まらない前提**で、失敗時は「戻す」より「前に進めて直す」が第一選択になるよう、UP-b の事前検証に重心を置きます |
| **UP-e** | **閉域では新版オフラインパッケージ（`c8yedge package` の成果物）を事前入手し、検証環境で使った同一物（チェックサム一致）を本番に使う。入手物は BK-j の保管体系に載せる** | Edge 側のバイナリ持ち込み運用はこれまで**未設計**でした（thin-edge 側の TE-17 に相当する Edge 側の欄） |
| **UP-f** | **アップグレードは計画停止（12.10.3）として扱う** | *"Upgrades with no downtime — No"* [確] |

**手順骨子**（runbook 化して [担当範囲] §10.3 へ — 12.10.6）: 事前（CU-35 確認 → パッケージ入手・検証 → 検証環境通過 → BK-c → 告知）→ 実施（計画停止枠で `c8yedge upgrade`）→ 確認（§12.4 確認 1〜7 の再実施）→ 事後（新版バックアップ・manifest 更新・§12.7 の監視が全項目復帰していること）。

#### 12.10.3 計画停止・起動 〈rev.4 で新設〉

**[確] Edge に停止・起動コマンドはありません**（§12.6.1）。K3s の停止は K3s 公式手順（`systemctl stop k3s` → `k3s-killall.sh`・§12.6.2b）に乗ります。

**停止中に起きること**: デバイス → mosquitto バッファに蓄積（**上限超過で欠落** — SV-44）／案件アプリ → 通知断・API 断／UI 断。**復旧時**: 溜まった再送とアラームが集中します（§8.4 リプレイ抑止・§5 の死活アラーム）。

**[判] 規約**:

| # | 規約 | 内容 |
|---|---|---|
| **ST-a** | **停止ウィンドウは「Edge 停止許容時間」以内** | 許容時間は BOX / thin-edge のバッファ保持時間の実測から決める（SV-29 / TE-10 — 12.6.5 と同じ入力）。**実測が済むまで計画停止を予定しない** |
| **ST-b** | **停止・起動の標準手順**: ①事前告知（案件アプリ担当含む）②§4.10 G-1 キルスイッチ（進行中オペレーションの抑止）③（作業内容が要求するなら）バックアップ取得 ④`systemctl stop k3s` → `k3s-killall.sh` ⑤ホスト作業 ⑥起動 → §12.4 確認 1〜7 ⑦キルスイッチ解除 ⑧再送・アラーム消化を観測（§8.4 のルールが効いていること） | リストア復旧後チェックリスト（12.6.4b）の縮退形。**②を飛ばすと、復旧直後に PENDING オペレーションがデバイスへ流れます** |
| **ST-c** | **ホスト再起動（OS パッチ等）は ST-b の縮退形として扱う** | K3s は systemd サービスとして自動起動する [推] — ⚠️ **無操作で Edge が完全復帰するかは実機確認が要ります** → **[要確認] CU-40**（W1。⑥の確認項目で合否判定） |

#### 12.10.4 ホスト OS の運用 〈rev.4 で新設 — 従来 0 件〉

rev.4 の横断検索で、**Edge ホストの OS パッチ・カーネル更新・OS アップグレード・再起動手順は 3 文書に 1 行も無い**ことを確認しました（検索語: OS パッチ / カーネル / kernel / reboot / RHEL / Ubuntu ほか — ヒットはすべて thin-edge・operator 再起動の文脈）。

| # | [判] 規約 | 内容 |
|---|---|---|
| **OS-a** | **ディストリビューションと版を確定し、記録する** → **[要決定] CU-36** | 公式は *"consult the operating system configurations required by K3s"* [確] ＝ **K3s のサポート OS から選ぶ**。⚠️ **復元先の「同一（または互換）OS」[確] の判定基準になる**ため、**版まで台帳に記録**（バックアップ manifest・12.6.2b と同じ台帳）。検証環境と本番で揃えること |
| **OS-b** | **セキュリティパッチは月次で確認し、再起動を伴うものは四半期の計画停止枠（ST-c）で適用** | 閉域網のため自動更新は無い。パッチ持ち込みは thin-edge 側 TE-17 と同じ社内リポジトリ経路に相乗りする |
| **OS-c** | **OS のメジャーアップグレードは in-place ではなく「別筐体に新 OS を立て、R-B 経路（12.6.4b）でリストア」を正とする** | ①in-place 失敗時の戻しが無い ②**リストア手順の実地訓練を兼ねる**（CT-35 の実施機会）③「クリーンな状態に復元」という公式復元手順と同型で、サポート面の説明がつく |
| **OS-d** | **K3s・containerd を OS のパッケージ管理で入れない・更新しない** | K3s は Edge インストーラが同梱・管理します。OS 側で `containerd` / `runc` を別途入れると競合します [推] |
| **OS-e** | **Edge / K3s / OS の脆弱性情報の受け口と対応期限を決める** → **[要決定] CU-37** | 現状、脆弱性対応のリードタイムが設計されているのは**画像解析装置（thin-edge）側だけ**（[担当範囲] TE-17）。Edge 本体・K3s・ホスト OS の CVE は**誰も受信していません**。ベンダー通知の登録先・深刻度別の適用期限（例: Critical は次回を待たず臨時の計画停止枠）を決めること |

#### 12.10.5 監視しきい値の初期値カタログ 〈rev.4 で新設〉

**§12.7 は「何を見るか」を定義しましたが、数値のしきい値が書かれているのは Used backlog 80% だけでした。** 以下はすべて **[判] 初期値** — 運用開始 1 か月と CT-28（負荷試験）の実測で見直してください。

| §12.7 の項目 | [判] 初期しきい値 | 根拠 |
|---|---|---|
| ① Edge 死活 | scrape 連続 3 回失敗で危険 | 誤検知（単発のネットワーク断）と全断の分離 |
| ③ デバイス証明書期限 | 残 **30 日**警告 / **14 日**危険 | 拠点に人を出す判断のリードタイム（§3.6） |
| ④ Notification バックログ | **Used backlog 80%** で警告 [確]（公式 UI の危険域） / Last ack ≥ 1 day | §10.5 の公式画面と同値 |
| ⑤ ディスク使用率 | **70%** 警告 / **85%** 危険 | 警告域で §7.6 の再算定（12.10.1）を発動し、**MongoDB 増設（増やせるが減らせない）の判断時間**を確保する。⚠️ 増設の実コマンドと再起動要否は **[要確認] CU-41** |
| ⑥ PENDING オペレーション | 通常ピークの 2 倍（実測前は **100 件**） | §4.11 の滞留検知 |
| ⑦ Edge TLS 証明書 | 残 **30 日**警告 / **14 日**危険 | 失効 ＝ 全拠点・全アプリ同時全断（§12.5） |
| ⑧ オフロードラグ | **7 日**超で警告 | 90 日リテンションに対する余裕（§7.5 — 失敗が 90 日続くと恒久喪失） |
| ⑨ `te/errors` | 1 件でも記録・日次レビュー（実測後に閾値化） | 拒否は仕様不整合のシグナル（§6.7） |
| ⑩ ライセンス期限 | 残 **90 日**警告（SV-50 で期限が確認されたら） | 発注・書面手続きのリードタイム |
| ⑪ 画像削除バッチ | 最古 `creationTime` > 保存期間 **+ 1 日** | §7.4 の判定式そのまま |
| ⑫ break-glass ログイン | **1 件で即時通報** | §11.5 R-07（使用時アラート） |
| ⑮ バックアップ | 最新世代の経過時間 > **28 時間**（RPO 24h + ジョブ実行時間・開始時刻ばらつきに対するマージン 4h。⚠️〈rev.4 訂正〉当初 26h としていたが、150GB 級のフルイメージエクスポートは「数十分〜時間単位」の変動があり、2h マージンでは取得成功時にも誤検知しうるため広げた）/ 世代数 < 計画値 / 保管先空き < 1 世代ぶん | BK-h・12.6.2b。⚠️ **実装経路が未定義** → **[要確認] CU-42**〈rev.4〉: バックアップジョブは Edge operator の `:3443/metrics`（§12.7 冒頭）の管轄外（ホスト OS 上で動く自前ジョブのため）。**[判] 推奨**: ジョブ完了時に取得時刻・世代数・保管先空き容量を **node_exporter の textfile collector 形式でホスト上に出力**し、Otel Collector の該当 receiver で拾う（Edge 本体のメトリクスパイプラインとは別系統として明示的に構築する） |
| ⑯ 時刻ずれ | **1 秒**警告 / **30 秒**危険 | TLS・JWT の時刻判定余裕（P-0-3）。デバイス側は `time` の品質（TM-a）にも直結 |

#### 12.10.6 期限台帳と runbook 追補 — [担当範囲] への起票 〈rev.4 で新設〉

**期限・鍵の台帳**（[判] Git 管理。`secrets/` と同じく**値は置かずメタデータのみ**）: Edge TLS 証明書（2 ホスト名）／社内 CA ルート／テナント CA（毎年 10/2）／デバイス証明書の失効分布（§3.6）／ライセンス（SV-50 待ち）／R-08・サービスユーザーのローテーション日／**バックアップ暗号鍵**（12.6.6 — 世代との対応）。**§12.7 の期限系監視（③⑦⑩）はこの台帳を正とする。**

**[担当範囲] §10.3 への runbook 追補起票**（⚠️ **採番は [担当範囲] 側で行うこと** — 本書が RB-14〜を勝手に振ると CU-17 と同じ ID 衝突を再生産します）:

| 追補する runbook | 契機 | 手順の種 |
|---|---|---|
| **Edge TLS 証明書更新** | 定期（残 30 日） | §12.5 のコマンド + 事前告知 + 更新後の全断確認 |
| **Edge 本体アップグレード** | 年 1 回以上 | §12.10.2 の手順骨子 |
| **計画停止・起動**（OS パッチ含む） | 四半期 | §12.10.3 ST-b |
| **ライセンス更新** | SV-50 の結果次第 | §12.2 P-3（`c8yedge config --set-file licenseKey=`） |
| **ディスク増設**（MongoDB 容量拡張） | §12.7 ⑤ 警告時 | [要確認] CU-41 の結果 |
| **監査ログ定期レビュー** | 月次 | §12.10.8 AU-a |
| **Notification バックログ満杯（500 応答）からの復旧** | 障害 | unsubscribe による解放 → 再購読 → 欠落期間の特定（RB-11 と同型）。⚠️ **unsubscribe でバックログが即時解放されるかは実機確認**（CT-30 に追加） |
| **データ誤削除時の影響特定** | 障害 | BK-i の帰結（個別復元不可）を前提に、影響範囲特定 → Git 再投入（設定）／オフロード先からの読み戻し可否（実データ・§7.5）→ 案件アプリへの通知 |

> あわせて rev.4 で **[担当範囲] の参照誤り 2 件を本書側で訂正済み**（§11.5 の RB-11 → RB-9、§12.5 の RB-13 誤参照）。[担当範囲] 側の改訂時に突き合わせてください（§15.5 DOC-5）。

#### 12.10.7 アカウント・アクセス権の定期棚卸し 〈rev.4 で新設 — 従来 0 件〉

**入退社・異動の処理は 3 文書のどこにもありませんでした**（検索語: 入退社 / 棚卸 / アクセス権レビュー / joiner / leaver — ヒットは設計時の主体棚卸しとコンシューマ棚卸しのみ）。

**構造上の前提** [確]: SSO ユーザーのロールは **Keycloak のグループクレームで毎ログイン時に解決**され、**手動付与は次回ログインで消えます**（§11.6）。したがって**人の入退社の一次処理は Keycloak 側**にあり、Cumulocity 側に残る棚卸し対象は**ローカルユーザー（admin ×2・break-glass・R-08・サービスユーザー）とロール割当の実機状態**です。

| # | [判] 規約 | 内容 |
|---|---|---|
| **AC-a** | **四半期棚卸し**: ①ローカルユーザーの実機一覧 vs `users.json`（Git）の突合（AS-1 系で機械化可能）②グローバルロール割当が §11.2 の設計と一致（AS-13(c) は CI 済み — 残りを目視）③Inventory ロール割当 vs 拠点一覧 ④**休眠検出** — 最終ログインが 90 日超のアカウント（`GET /audit/auditRecords?type=UserAuthentication` で機械化） | **SSO 側が正しくても、ローカルユーザーは Cumulocity 側でしか検出できません** |
| **AC-b** | **Keycloak 側と「退場 SLA」を合意する** → **[要決定] CU-38** | 退社から Keycloak アカウント無効化までの猶予中は **Cumulocity にログインできます**。SLA（例: 当日中）と、緊急時に Cumulocity 側で先に塞ぐ手段（該当ユーザーの無効化）を合意しておく |

#### 12.10.8 監査ログの定期レビュー 〈rev.4 で新設〉

§12.7 ⑫⑬⑭ は**自動監視**への落とし込みでした。**「人が読む」レビューはどこにも設計されておらず、しかも §11.1 のアクセス主体一覧に「監査担当」が存在しません**（S-1〜S-16 に無い — 誰がレビューするのか主体が未定義）。

| # | [判] 規約 | 内容 |
|---|---|---|
| **AU-a** | **月次レビュー観点**: ①break-glass ログイン（0 件が正常）②`type=Application`（マイクロサービスのデプロイ — AS-15 の CI 結果と突合）③`type=Inventory` の削除 ④リテンションルールの変更（AS-11b と突合）⑤ロール・ユーザー変更 ⑥（CU-18 決定後）画像アクセスレコードのサンプル検査 | レビューは「§12.7 のアラートが鳴らなかったことの確認」ではなく、**閾値化できない異常（正規権限の悪用パターン）を人が見る**工程です |
| **AU-b** | **レビュー結果は Cumulocity の外側の台帳に記録する** | 監査レコードの改竄不能性は**リストア経路では成立しません**（RS-7）。レビュー記録まで Cumulocity 内に置くと共倒れします |
| **AU-c** | **レビュー主体は §11.1 の S-17（監査担当）。§11.2 の R-17 で `ROLE_AUDIT_READ` を付与する（rev.4 で追加済み）。AS-13(c) の CI 検査対象に R-17 を含めること** | 主体の棚卸しに存在しない役割は、権限設計からも漏れ続けます（rev.3 まで AS-13 が「監査担当」を参照しながら主体一覧にもロール定義にも不在でした） |

#### 12.10.9 Keycloak・監視基盤への要求仕様 〈rev.4 で新設〉

**Keycloak 本体は担当外**（[担当範囲] §1.2）ですが、**「Keycloak が止まると何が起きるか」がどの文書にも書かれていませんでした**。

**停止影響** [推]: SSO ユーザー全員がログイン不可（既存トークンの失効まで漸進的に拡大）。**デバイス接続・データ収集・通知配信は影響を受けません**（証明書 / Basic 認証で Keycloak を経由しないため）。**唯一の管理経路は break-glass（§11.5）**になります。

| # | [判] | 内容 |
|---|---|---|
| **KC-a** | **Keycloak 担当への要求仕様を文書で提示する** → **[要決定] CU-38 に含める** | ①可用性目標（Edge の RTO と整合させる — SSO 断が Edge 復旧のボトルネックにならないこと）②**Keycloak 自身のバックアップ**（realm 設定・ユーザー・グループ。**Edge のバックアップには一切含まれません**）③NTP（P-0-3）④Keycloak バージョンアップ時の互換確認項目（redirect URI・グループクレーム名・JWKS — 1 文字違いで全ユーザーがログイン不可・CU-9）⑤停止・メンテナンスの相互告知経路 |
| **KC-b** | **「Keycloak 長時間障害」を break-glass の発動契機として §11.5 に紐づける** | RB-8（SSO からローカル認証への切り戻し）は**設定を戻す**手順であり、**障害中の一時運用**（break-glass で最小限の運用を続ける）とは別物です。混同しないこと |
| **KC-c** | **監視基盤（Otel Collector / Prometheus）自身の死活を、監視基盤の外で検知する手段を 1 つ持つ**（例: heartbeat の不達検知を通報先側で） | 変-2 でメールが無い本構成では **§12.7 が基盤異常の唯一の検知経路**です。その §12.7 自身が沈黙したことを誰も検知できない構成は、BK-h で禁じた「静かに止まる」の再生産です。⚠️ **通報先・当番（SV-34 / TB-5）が決まるまで、本番デバイスの接続開始（W5 以降）に進まないことを推奨します** [判] |

#### 12.10.10 Edge プラットフォームログの扱い 〈rev.4 で新設 — 従来 0 件〉

**Edge 自身のログ（Pod ログ・operator・MongoDB・ingress）とホスト syslog の収集・保存はどの文書にもありませんでした**（§12.7 は Prometheus メトリクスのみ）。なお Edge CR の `storageClassName` 説明は PV の用途に *"application data, microservice images, and logs"* を挙げており [確]、**ログは PV 内 ＝ バックアップに含まれます**。

| # | [判] 規約 | 内容 |
|---|---|---|
| **LG-a** | **平時は Pod ログを外部収集しない。障害調査時に取得する手順（`kubectl logs` / `journalctl -u k3s` / 対象 Pod の特定表）を runbook に持つ** | ①ログへの個人情報混入リスク（MS-f の禁則は自作サービスにしか効かない — プラットフォームのログに何が出るかは制御外）②収集すると `AUDIT` リテンションと `ROLE_AUDIT_READ` の保護の外に個人情報アクセスの痕跡を二重化しかねない（§15.3 の ingress ログ案を却下した理由③と同根） |
| **LG-b** | **恒常収集するなら対象を絞って Otel Collector 経由**（operator と ingress のエラーレベルのみ等）→ **[要決定] CU-39** | 全量収集は保管・保護・保存期間の設計（それ自体が §7 級の作業）を伴います。**「収集しない」も設計判断として明記するのが本規約の趣旨** |
| **LG-c** | **ホスト syslog / journald はホスト内ローテーション（OS 既定）とし、保存期間を OS-a の台帳に記録** | S-15 の操作痕跡（CU-29 の踏み台セッション記録と併せて）の最低限の保全 |

#### 12.10.11 変更管理と検証環境の維持 〈rev.4 で新設〉

**CF-a〜CF-f / AS-1〜AS-15（§13）で「実機と Git の乖離」は技術的に強く守られていますが、Git リポジトリ自体の運用ルールと、検証環境を「維持する」設計がありませんでした。**

| # | [判] 規約 | 内容 |
|---|---|---|
| **CM-a** | **main ブランチ保護 + PR 必須 + レビュー 1 名以上**。実機への適用は CI からのみ（人は PR を出すところまで） | CF-a（UI 手作業禁止・D10）の Git 側の対。**リテンション短縮（不可逆・§7.2）と ロール変更は承認者を分ける**（M-9 の穴の受け皿） |
| **CM-b** | **リリース単位はタグ**。CF-b のフェーズ先頭エクスポートと対にする | 切り戻しの起点（BK-g — 設定は Git が正） |
| **CM-c** | **検証環境は W1 で使い捨てず、常設で維持する** | 用途: アップグレード先行（UP-b）・リストア試験（CT-35 の定期実施）・ルール/設定の事前検証（§8.7）・障害再現。⚠️ **ライセンスがもう 1 本要るかを SV-01 の商流確認に追加**すること |
| **CM-d** | **本番適用は「検証環境で同一コミットを適用済み」をゲートとする**（CI で強制） | 「検証環境の構成が本番とずれていて意味がない」の防止は **CF-f（環境別 values は L1-C のみ）+ AS-1 の双方向差分**で機械的に担保する |

---

## 13. Cumulocity 設定設計

> **本章は §1〜§11 の定義を、Cumulocity Edge の具体的な設定項目・設定手段・設定値に落とします。** 適用手順のフェーズ割付は [設定書] §5（P0〜P7）が正で、本章は「何をどんな値で設定するか」を担います。

### 13.1 設定の分類と管理方式

| 分類 | 内容 | 変わるタイミング | 投入手段 | リポジトリ |
|---|---|---|---|---|
| **L0 / 基盤インフラ層** | Edge そのもの（ドメイン・TLS・ライセンス・K8s リソース） | 環境ごとに必ず変わる | `c8yedge` CLI / `kubectl` + Edge CR | `config/edge/` |
| **L1-B / 基盤標準初期値** | 全案件で同一の「製品としての既定値」 | 基盤のバージョンアップ時のみ | go-c8y-cli（環境非依存） | `config/platform/` |
| **L1-C / 本構成固有設定** | この構成でのみ必要な設定 | 案件・環境ごとに変わる | go-c8y-cli（環境別 values） | `config/site/` |
| **L1-M / management テナント** | ブランディング等 | 稀 | go-c8y-cli（`--session management`） | `config/mgmt/` |

**[判] 管理方式の規約**:

| # | 規約 | 理由 |
|---|---|---|
| **CF-a** | **全設定を構成コードで管理する。UI からの手作業を禁止**（D10） | 再現性。手作業は次回の投入で上書き／重複する |
| **CF-b** | **各フェーズの先頭で「対象リソースの GET エクスポート + タグ付きコミット」を実施する** | §13.7 の往復差分の基準になり、切り戻しの起点にもなる |
| **CF-c** | ⚠️⚠️ **`C8Y_MODE=ci` を明示する。未指定は危険** | **未指定時の既定は `prod`**（`SessionModeProduction`）で、**Create/Update/Delete が全てエラーになり非ゼロ終了します** [確]（*"If set to false then all POST related commands will **return an error**."*）。`--sessionMode ci` フラグでも指定可。⚠️ rev.1 は「エラーも出さずに完走する」としていましたが誤りで、**実際はエラーで落ちます**（結論は同じ） |
| **CF-d** | **全コマンドに `--session <name>` を明示する** | management / edge の 2 セッションを取り違えると、投入先を間違える |
| **CF-e** | **`--dry --dryFormat json` の出力に `Authorization: Basic` ヘッダが含まれ得る** | CI ログに残さない |
| **CF-f** | **L0 / L1-B / L1-C の分離を崩さない** | **この分離ができているかが、2 案件目を `site/` だけの作業で立ち上げられるかを決める** |

### 13.2 ⭐ 定義 → 設定項目 トレース表

**本書の各定義章が、どの設定項目になるかの対応です。この表が §13 の索引です。**

| 定義（章） | 設定項目 ID | 設定対象 | 分類 | 詳細 |
|---|---|---|---|---|
| §1 デバイス管理グループ | **G-01** 拠点 device group 階層 | edge | L1-C | §13.3.3 |
| ~~§1.4 グループの external ID~~ | ~~**G-02**~~ **rev.2 で廃止**（G-04 に統合） | — | — | §13.3.3 |
| §1.5 グループのフラグメント | **G-04**〈新〉 グループの `x_Site` | edge | L1-C | §13.3.3 |
| §2 命名規約 | — | **設定ではなく規約文書 + バリデータ** | L1-B | §13.3.12 |
| §2.8 機械検査 | **CK-1〜CK-8** | CI | — | §13.7 |
| §3.3 オンボーディング方式 | **D-01** 登録方式（テナント CA） | edge | L1-C | §13.3.5 |
| §3.3 テナント CA | **D-10**〈新〉 `certificate-authority` フィーチャー有効化 + CA 作成 | edge | L1-C | §13.3.5 |
| §3.4 登録主体 | **R-08** 証明書アップロード用ローカルユーザー | edge | L1-B | §13.3.4 |
| §3.5 child 代理登録 | **D-07** child device 登録 | edge | L1-C | プロビジョニングツール |
| §3.8 ツイン管理 | **D-11**〈新〉 フラグメントの情報源固定 | — | 規約 | §3.8 |
| §4.2 supported operations | **D-05** supported operations | **thin-edge 側** | — | [担当範囲] §5.8 |
| §4.4 モデル配布 | **M-01** ソフトウェアリポジトリ | edge | L1-B | §13.3.9 |
| §4.5 設定配布 | **K-03 / K-04** 設定リポジトリ・デバイスプロファイル | edge | L1-C | §13.3.9 |
| §4.7 リモートアクセス封じ | **R-13**〈新〉 Cloud Remote Access ロールを付与しない | edge | L1-B | §13.3.4 |
| §4.10 ガードレール | **K-01〜K-08** 運用知識基盤の統制 | edge + 運用 | L1-C | §13.3.9 |
| §5.5 死活の値 | **D-04** `c8y_RequiredAvailability` | **thin-edge 側**（main）/ プロビジョニングツール（child） | — | §5.5 |
| §6.4 型カタログ | **S-04** イベント/アラーム型・ペイロード規約 | **規約文書 + バリデータ** | L1-B | §13.3.12 |
| §7.2 リテンション | **X-01** リテンションルール | edge | L1-B | §13.3.6 |
| §7.5 オフロード | **X-03** オフロードバッチ | 外部実装 | L1-C | §13.3.6 |
| §8.2 ルール | **S-01** スマートルール / **S-02** EPL apps | edge | L1-B | §13.3.7 |
| §8.3 自動クリア | **S-03** アラーム自動クリア | edge | L1-B | §13.3.7 |
| §9.2 アラームマッピング | ⭐ **T-03** `alarm.type.mapping` | edge | L1-B | §13.3.1 |
| §10.2 サブスクリプション | **N-01** Notification 2.0 サブスクリプション | edge | L1-C | §13.3.8 |
| §10.4 トークン発行プロキシ | **N-02** プロキシ配備 / **R-09** サービスユーザー | edge + VM1 | L1-B | §13.3.8 |
| §10.1 前提 | **N-03** Messaging Service 稼働確認 | edge | L0 | §12.4 |
| §10.5 ライフサイクル | **N-04** コンシューマ棚卸し | 運用 | — | §13.3.8 |
| §11.2 グローバルロール | **R-01〜R-03 / R-10** | edge | L1-B | §13.3.4 |
| §11.3 Inventory ロール | **R-04 / G-03** | edge | L1-B / L1-C | §13.3.4 |
| §11.4 サービスユーザー | **R-05 / R-06 / R-09 / R-11 / R-12** | edge | L1-B / L1-C | §13.3.4 |
| §11.5 break-glass | **R-07** | edge + management | L1-B | §13.3.4 |
| §11.6 SSO | **A-01〜A-05** | edge | L1-C | §13.3.11 |
| §11.8 CORS / アプリ登録 | ⭐ **T-02** CORS / **P-01〜P-03** | edge | L1-C / L1-B | §13.3.1 / §13.3.10 |
| §12.7 Edge 監視 | **O-01** Prometheus エンドポイント接続 | 外部 | — | §12.7 |

### 13.3 設定項目定義（詳細）

> **凡例**: 設定手段の **CLI** は go-c8y-cli（`c8y`）。**API** は `c8y api` で直接エンドポイントを叩くもの。**冪等**は [投入ガイド] のパターン A〜E。

#### 13.3.1 テナントオプション（T 系）

| ID | カテゴリ / キー | **[判] 設定値** | 設定手段 | 冪等 | 由来 |
|---|---|---|---|---|---|
| **T-01** | `configuration` / `acl.algorithm-version` | **`OPTIMIZED`**（10.16+ の既定のまま） | `c8y tenantoptions updateBulk` | 上書き | §11.9 AP-a |
| ⭐ **T-02** | `access.control` / `allow.origin` | **VM2 の全アプリのオリジンをスキーム+ホスト+ポートで列挙**（既定 `*` からの絞り込み）。例: `https://app1.example.local:8443,https://app2.example.local` | 同上 | 上書き | §11.8 |
| ⭐ **T-03** | `alarm.type.mapping` / `<ALARM_TYPE>` | **§9.2 の表のとおり**（下記） | 同上 | 上書き | §9.2 |
| **T-04** | `configuration` / `acl.measurement.only-accessible-fragments` | **`true`**（検討） | 同上 | 上書き | [定義書] |

**T-03 の投入内容**（`config/platform/tenantoptions.alarm-type-mapping.json`）:

```json
{
  "c8y_UnavailabilityAlarm": "MAJOR|ゲートウェイ応答なし",
  "x_CameraDown":            "MAJOR|カメラ応答なし",
  "x_BoxSilent":             "MAJOR|BOX無通信",
  "x_BoxParseError":         "WARNING|BOXペイロード変換不能",
  "x_ModelUpdateFailed":     "CRITICAL|モデル適用失敗"
}
```

> ⚠️ **`x_Alarm_<種別>` は案件側が種別を確定してから追加します。** 種別が増えるたびにこのファイルへの追記が必要です（§9.2）。

> ⚠️ **[要 SV-30] `PUT /tenant/options/{category}` のマージ／置換セマンティクスが未確認です。** **置換なら、カテゴリ内の一部だけを投入すると他のキーが消えます。** 検証環境で確認してから `updateBulk` を使ってください。

> ⚠️ **`credentials.*` で始まるキーは取得時に除外してください**（[投入ガイド] §5）。Git に載せてはいけません。

#### 13.3.2 フィーチャートグル

| ID | フィーチャー | 設定値 | 設定手段 | 由来 |
|---|---|---|---|---|
| **P-04a** | `certificate-authority` | **有効化** | `c8y features enable --key certificate-authority` | §3.3 / §13.3.5 |
| **P-04b** | その他 | **[要]** 実環境で `c8y features list` を取得して差分管理 | `c8y features enable/disable` | — |

#### 13.3.3 拠点グループ（G 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **G-01** | 拠点 device group 階層 | **root（`sites`）→ 拠点グループ（`site001`…）の 2 段**（§1.2 / §1.7 G-e）。**`type` は root=`c8y_DeviceGroup` / サブ=`c8y_DeviceSubGroup`**（§1.5） | `c8y inventory create` + `childAssets` 追加、または一括登録 CSV の `PATH`（§3.3） | ⚠️⚠️ **C: 存在チェック → 分岐**（下記） |
| ~~**G-02**~~ | ~~グループの external ID~~ | **rev.2 で廃止。** グループの同定は **`x_Site.siteId`（G-04）** で行う（§1.4） | — | — |
| **G-03** | Inventory ロールの割当 | **SSO の `inventoryMappings` で管理**（§11.3 IR-b） | §13.3.11 の `authConfig` | 上書き |
| **G-04**〈新〉 | ⭐ グループの `x_Site` フラグメント | `{ "siteId": "site001", "siteName": "…" }`。**rev.2 で G-02 を統合し、これが拠点グループの唯一の同定キーになりました** | **作成時に `c8y inventory create` の本文へ含める**（後付け UPDATE にしない） | **C: 存在チェック** |

> ⚠️⚠️ **G-01 に「409 黙殺」は使えません** [確]。`POST /inventory/managedObjects` の応答定義は **`201` / `401` / `403` / `422` のみで 409 がありません**。一意制約を持つ API（`POST /user/{t}/users`、`POST /notification2/subscriptions`）は 409 を返すので、**これは仕様上の意図**です。
>
> **つまり同名・同 `siteId` のグループを再作成すると、エラーにならず黙って 2 個目が生えます。** rev.1 の冪等化パターンでは**再実行のたびに拠点グループが増殖**しました。
>
> **[判] 正しい手順**:
>
> ```bash
> # ① x_Site.siteId で既存を引く（グループ名ではなく siteId で引くこと）
> EXISTING=$(c8y inventory find --session edge -o json \n>   --query "has(c8y_IsDeviceGroup) and x_Site.siteId eq '$SITE_ID'" --select id)
>
> # ② 無ければ作成（x_Site を作成時の本文に含める）。あれば更新のみ
> ```
>
> ⚠️ **`--force` は upsert ではありません**（*"Do not prompt for confirmation"* の意味）。go-c8y-cli に upsert / sync 系のコマンドはありません。
>
> ⚠️ **`x_Site` を「作成後に UPDATE で付ける」のもやめてください。** 作成と同定キー付与が 2 ステップに分かれると、UPDATE が失敗したときに**同定キーを持たない孤児グループ**が残り、次回実行でさらにもう 1 つ作られます。**`c8y inventory create` の本文に `x_Site` を含めて 1 ステップにしてください**（§13.5）。

**拠点定義ファイルの例**（`config/site/sites/site001.yaml`）:

```yaml
siteId:    site001                  # 英数字のみ（§2.5）
siteName:  "〇〇工場 東棟"           # 表示名（日本語可）
devices:
  imageAnalyzer:
    serial:  SN00123                # → extID: site001-ANLZ-SN00123（main はコロン不可・§2.2 X-f）
    cameras:
      - serial: SN12345             # → extID: site001:CAM-SN12345
        vmsCameraId: gsc-cam-8842   # ★必須（未設定はツールがエラー）
        location: "東門"
  boxGateway:
    boxes:
      - serial: SN98765             # → extID: site001:BOX-SN98765
        model:    "BX-200"
        channels: 4
```

> **[判] 2 拠点目が「このファイルの追加だけ」で立つことを設計品質の判定基準にしてください**（§3.9）。

#### 13.3.4 ロール・ユーザー（R 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **R-01** | グローバルロール「基盤運用者」 | §11.2 | `c8y usergroups create` → `c8y userroles addRoleToGroup` | **C: 存在チェック** |
| **R-02** | 「拠点オペレーター」 | §11.2 | 同上 | 同上 |
| **R-03** | 「業務閲覧者」 | §11.2（**Cockpit アプリアクセスを含む**） | 同上 | 同上 |
| **R-10**〈新〉 | 「保守（AI モデル）」 | §11.2 | 同上 | 同上 |
| **R-04** | Inventory ロール「拠点Manager」「拠点Reader」 | §11.3 | ⚠️ **`c8y api POST /user/inventoryroles`**（専用サブコマンドが無い） | **E: 409 黙殺** |
| **R-05** | サービスユーザー（VM2 アセット用） | ローカルユーザー + 最小ロール。**SSO 対象外** | `c8y users create` → `c8y userroles addRoleToUser` | **C: 存在チェック** |
| **R-06a/b** | 運用知識基盤（読取 / 書込） | §11.4 | 同上 | 同上 |
| **R-07** | **break-glass（両テナント）** | SSO 対象外の緊急用ローカル管理者。**TFA + エスクロー** | 同上（`--session management` と `--session edge` の両方） | 同上 |
| **R-08** | 証明書アップロード用ローカルユーザー | `ROLE_TENANT_ADMIN` or `ROLE_TENANT_MANAGEMENT_ADMIN` | 同上 | 同上 |
| **R-09** | トークン発行プロキシ用 | ⚠️ **`ROLE_NOTIFICATION_2_ADMIN` はここだけ** | 同上 | 同上 |
| **R-11**〈新〉 | オフロードバッチ用 | 拠点横断 READ | 同上 | 同上 |
| **R-12**〈新〉 | クリップ保存サービス用 | Alarms READ | 同上 | 同上 |
| **R-13**〈新〉 | ⭐ **Cloud Remote Access ロール** | **どのロール・どのユーザーにも割り当てない** | — （**投入しないことが設定**） | **CI で検査**（§13.7） |
| **R-15** 〈rev.3〉 | ⭐ **`ROLE_AUDIT_ADMIN`** | **どのグローバルロール・どの人間ユーザーにも割り当てない。**付与先は R-14（画像配信 / 監査書込マイクロサービス）のみ | — （**投入しないことが設定**） | **CI で検査**（§13.7 AS-13） |
| **R-16** 〈rev.3〉 | ⭐ **Inventory ロールの `scope: AUDIT`** | **どの Inventory ロールにも含めない。** `inventoryRolePermission.scope` の enum に `AUDIT` が実在するため（§11.2） | — （**投入しないことが設定**） | **CI で検査**（§13.7 AS-13） |
| **R-17** 〈rev.4〉 | グローバルロール「監査担当」 | §11.2（`ROLE_AUDIT_READ` のみ。`ROLE_AUDIT_ADMIN` は含めない） | `c8y usergroups create` → `c8y userroles addRoleToGroup` | **C: 存在チェック** |
| **A-05a** | パスワードポリシー | ⚠️ **`Password validity limit` は**テナント全体**の設定で、アカウント単位に指定できません**（*"use '0' for unlimited validity of passwords (default value)"*）。§11.5 統制 3 の「管理者アカウントの有効期限は無期限」は**実装不能**なので、テナント方針として決め直すこと（**[要 CU-14]**） | `c8y tenantoptions updateBulk`（`password` カテゴリ） | 上書き |
| **A-05b** | TFA | ⚠️ **`password` テナントオプションでは設定できません。** `PUT /tenant/tenants/{tenantId}/tfa` ／ **`c8y tenants tfa update --strategy TOTP`** を使うこと。⚠️ **TOTP は OAI-Secure ログインモード限定**（§11.6 ガード 2b） | `c8y tenants tfa update` | 上書き |

> ⚠️ **投入前に `GET /user/roles` で実際の `ROLE_*` 名を確認してください。** UI の権限レベルと API ロール名は 1:1 対応しません（§11.2）。

> ⚠️ **ロール割当はワイルドカードが使えます**（[投入ガイド] パターン A）。`c8y userroles addRoleToGroup --group "基盤運用者" --role "ROLE_INVENTORY_*"` のようにグループ名・ロール名で指定でき、**ID 解決が不要**です。

#### 13.3.5 デバイス登録（D 系）

| ID | 設定項目 | **[判] 設定値 / 手順** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **D-10**〈新〉 | **テナント CA の作成** | ①`c8y features enable --key certificate-authority` ②`c8y devicemanagement certificate-authority create` | CLI | **C: 存在チェック** |
| **D-01** | 登録方式 | **テナント CA + EST**（§3.3） | — | — |
| **D-02** | 登録主体 | プロビジョニングツール（R-08 の資格情報で実行） | — | — |
| **D-03** | device type 規約 | `x_ImageAnalyzer` / `x_Camera` / `x_BoxGateway` / `x_SensingBox`（§2.3） | プロビジョニングツール | — |
| **D-06** | external ID | `{siteId}:{機器種別}-{シリアル}`（§2.2） | プロビジョニングツール（thin-edge の `@id`） | — |
| **D-07** | child device 登録 | §3.5 手順 4〜7 | プロビジョニングツール（`POST /te/v1/entities`） | **C: 存在チェック** |
| **D-08** | main device の拠点グループ編成 | **拠点あたり 2 回**（案 β） | `c8y api POST /inventory/managedObjects/{groupId}/childAssets` | **E: 409 黙殺** |
| **D-09** | アラームストーム抑止 | child に `c8y_RequiredAvailability = {"responseInterval": 32767}`。⚠️ **`0` は全アラーム抑止になるため不可**（§5.2）。**thin-edge の初回接続より前に投入すること**（§5.5） | プロビジョニングツール（twin） | **C: 存在チェック** |
| **D-12**〈新〉 | ⚠️ **`autoRegistrationEnabled` の無効化** | **方式 (2) を使った場合のみ**。一括登録完了後に必ず無効化 | `c8y api PUT /tenant/tenants/{t}/trusted-certificates/{id}` | 上書き |

> ⚠️ **D-10 は前提 3（SV-08 / TE-8）に依存します。** NG の場合は方式 (2)（自己署名 CA + trusted certificate）に切り替え、**D-12 の無効化を必ずセットで実施**してください。

#### 13.3.6 データ保持（X 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **X-01** | リテンションルール | `EVENT` / `MEASUREMENT` / `ALARM` / `OPERATION` = **90 日**、**`AUDIT` = [要決定]** | `c8y retentionrules create` | ⚠️ **D: 宣言的な集合適用（新規作成 → 旧削除の順）** |
| **X-02** | イベント添付バイナリ | **及ぶ [確K]**（§7.4）。⚠️ **CT-22③ で実証すること**〈rev.3 で更新 — 「未確認」表記が §7.4 の「解消」と分裂していた〉 | — | — |
| **X-03** | オフロード | **REST ページング方式（案 B）**。`prev`/`next` で辿る | 外部バッチ（R-11 の資格情報） | — |
| **X-04** 〈rev.3・**CU-18 決定後に確定**〉 | **画像削除バッチ**（案 (D) 採用時のみ） | **[要決定]**。ファイルリポジトリにはリテンションが効かないため（§7.4）、**保存期間を守る唯一の手段**。実装主体は §15.4 で決定 | 外部バッチ | — |

**投入手順**（§7.2 の反転手順）:

```bash
# ① 現行ルールを取得して Git コミット
c8y retentionrules list --session edge -o json > config/platform/retentionrules.current.json

# ② 新ルールを先に作成（--dry で事前確認）
c8y retentionrules create --session edge --dataType EVENT       --maximumAge 90 --dry
c8y retentionrules create --session edge --dataType MEASUREMENT --maximumAge 90
c8y retentionrules create --session edge --dataType ALARM       --maximumAge 90
c8y retentionrules create --session edge --dataType OPERATION   --maximumAge 90
c8y retentionrules create --session edge --dataType AUDIT       --maximumAge <要決定>

# ③ 旧ルールを個別に削除（存在する場合のみ。⚠️ [要 SV-43]）
```

> ⚠️⚠️ **順序を守ってください。** 削除が先だと、作成に失敗した場合に**リテンションルールがゼロの状態で放置**されます（§7.2）。

> ⚠️⚠️ **[要 SV-43] 手順③は空振りする可能性があります。** 「既定 60 日」は公式に明記されていますが、**それが `c8y retentionrules list` で列挙・削除できるオブジェクトとして存在するという記述は、docs / OpenAPI / Tech Community のいずれにもありません**（3 経路で探索済み）。
>
> **手順①の出力（`retentionrules.current.json`）を見て分岐してください**:
>
> | ①の結果 | 意味 | 対応 |
> |---|---|---|
> | 60 日のルールが列挙される | 削除可能なオブジェクトとして存在する | ③をそのまま実行 |
> | **空、または 60 日のルールが無い** | **60 日はシステム設定側の値** | ⚠️ **③は不要。ただし `GET /tenant/system/options` で保持日数キーを探し、90 日ルールより先にシステム設定の 60 日が効かないことを検証環境で必ず確認すること** |
>
> **後者だった場合、90 日のルールを入れても 60 日でデータが消える可能性が残ります。§7.2 のデータ保持設計そのものが崩れるため、W1 の最優先項目です。**

> ⚠️ **`--dry` が正しいフラグ名です** [確]（`--dry-run` は存在しません）。`--dryFormat <markdown|dump|json|curl>` で出力形式を選べます。⚠️ **`--dryFormat json` / `curl` は Authorization ヘッダを出力に含めます** — CI ログに残さないでください（§13.1 CF-e）。

> ⚠️ **`--maximumAge` を省略すると 365 日になります**（go-c8y-cli の bodyTemplate 既定）。**全ルールで明示してください。**

> ⚠️ **`AUDIT` の日数が未決のままこの手順を実行しないでください。**

> ⚠️⚠️ **`AUDIT` は CU-18 → CU-5 → 本手順②の順に依存します** 〈rev.3 で追加〉。
>
> **CU-18 が「監査レコードを書く」結論（(A)/(A′)/(D)/(B)）なら、個人情報アクセス記録の保存義務期間が `AUDIT` の日数を支配します**（§7.2）。**CU-18 が未決のまま暫定値を入れないでください** — §7.2 のとおり**保持日数の短縮方向は不可逆**で、短く入れて後から延ばしても**消えたデータは戻りません**。
>
> **[判] CU-18 未決なら、手順②の `AUDIT` 行だけをスキップし、他の 4 種を先に投入してください。** 既定ルールの削除（手順③）も `AUDIT` 分は保留します。

> ⭐ **`AUDIT` は 2 本立てにできます** 〈rev.3〉。`retentionRule.type` は *"Used by the data types ALARM, **AUDIT**, EVENT and MEASUREMENT"* [確] なので、**画像アクセス記録だけ長期保持**が可能です（§7.2）:
>
> ```bash
> # 画像アクセス記録（監査レコードの type を専用値にしておくこと・§15.3）
> c8y retentionrules create --session edge --dataType AUDIT --type <画像アクセス用type> --maximumAge <法定値>
> # その他の監査記録
> c8y retentionrules create --session edge --dataType AUDIT --maximumAge <障害調査要件>
> ```

> ⚠️⚠️ **`AUDIT` のルールは `management` テナントから `editable: false` で作成することを検討してください** 〈rev.3〉。
> `retentionRule.editable` は *"It can be updated only by the **Management tenant**."* [確]。**`edge` テナントの R-01 は Retention rules ADMIN を持つため、そのままだと `maximumAge=1` のルールを足して証跡を狙い撃ちで消せます**（§7.2）。`editable: false` にすればこの経路が塞がります。**投入テナントが変わるので、§13.1 の管理方式（どのセッションで投入するか）に反映してください。**

#### 13.3.7 ルール（S 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **S-01** | スマートルール | §8.2 RU-1 / RU-2 | ⚠️ **[要 SV-32]** managed object の `type` / `fragmentType` の実値が未確定。**GUI で 1 つ作成 → `c8y inventory find` で観測してから投入スクリプトを書く** | **C: 存在チェック** |
| **S-02** | EPL apps | §8.2 RU-3 / RU-4 / RU-5 | `c8y api POST /service/cep/eplfiles`。**ソースごと Git 管理** | **B: PUT 先行 → 404 なら POST** |
| **S-03** | アラーム自動クリア | §8.3 | ⚠️ **EPL**（時限クリアに使える組み込みスマートルールは存在しない・§8.3） | 同上 |
| **S-04** | **イベント/アラーム型・ペイロード規約** | §6.4 / §6.5 | ⚠️ **設定ではなく規約文書 + バリデータ**（§13.3.12） | — |

> ⚠️⚠️ **前提**: **Apama-ctrl / Smartrule マイクロサービスへの購読**が必要です（§12.4 確認 5）。**加えて、Edge が同梱する Apama-ctrl のバリアントによっては EPL Apps 自体が使えません**（§8.1 [要 SV-45]）。**S-02 / S-03 の投入は SV-45 の確認後に着手してください。**

> ⚠️ **投入は型名（§6.4）の確定が前提です。** 型が決まる前にルールは投入できません。

> ⚠️ **雛形には必ずリプレイ抑止ガードを組み込んでください**（§8.4）。**案件側が触る条件式とは別ファイルに分離**することを推奨します（§8.6）。

#### 13.3.8 通知（N 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **N-01** | サブスクリプション定義 | **§10.2 の NS-1 / NS-2 / NS-3** | `c8y api POST /notification2/subscriptions`（**R-09 の資格情報**） | **C: 存在チェック** |
| **N-02** | トークン発行プロキシ | VM1 に配備。§10.4 | 外部実装（**[要 TB-3]** 担当未確定） | — |
| **N-03** | Messaging Service 稼働確認 | §12.4 確認 6 | `kubectl get pods -n c8yedge` | — |
| **N-04** | コンシューマ棚卸し | unsubscribe runbook + バックログ監視 | 運用手順 | — |

**NS-1 の定義例**（拠点ごとに 1 つ生成）:

```json
{
  "context": "mo",
  "source":  { "id": "<拠点グループの MO ID>" },
  "subscription": "site001",
  "subscriptionFilter": {
    "apis": ["alarmsWithChildren"],
    "typeFilter": "'c8y_UnavailabilityAlarm' or 'x_CameraDown' or 'x_BoxSilent' or 'x_Alarm_Intrusion'"
  },
  "fragmentsToCopy": ["x_Site", "x_Camera"],
  "nonPersistent": false
}
```

> ⚠️ **`fragmentsToCopy` を必ず指定してください**（§10.7 NX-3）。指定しないと**全フラグメントが案件アプリへ流れます**。生体情報・個人情報に関わるフラグメントを渡さないための唯一の手段です。

> ⚠️ **`subscription` 名は英数字のみ**（§2.5）。拠点コードをそのまま使います。

> ⚠️ **`source` にはグループの MO ID（内部 ID）が必要**なので、**G-01 / G-04 の投入後**にしか実行できません（§13.4）。⚠️ **external ID ではなく内部 ID です** — `x_Site.siteId` で引いた MO の `id` を使ってください（§1.4）。

#### 13.3.9 リポジトリ系（M / K 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **M-01** | ソフトウェアリポジトリ | `softwareType = aimodel`（§4.4） | `c8y software create` / `c8y software versions create` | **C: 存在チェック** |
| **M-02** | モデル配布オペレーション | `c8y bulkoperations create`（**`--creationRampSec` 必須**・§4.8）。⚠️ **フラグ名は `--creationRampSec`**。`creationRamp` は API ボディのフィールド名 | CLI | — |
| **M-03** | モデルの保持世代 | **[要 TE-20]** 保持世代数と削除運用 | 運用手順 | — |
| **K-03** | 設定リポジトリ | 配布対象が確定してから（**[要 SV-17]**） | `c8y configuration create` / `send` | **C: 存在チェック** |
| **K-04** | デバイスプロファイル | 型ごとの「あるべき設定の束」。**[要 SV-17]** | `c8y deviceprofiles create` | 同上 |
| **K-01〜K-08** | 運用知識基盤の統制 | **§4.10 のガードレール G-1〜G-14** | 運用手順 + R-06 の権限設計 | — |

| **M-04** 〈rev.3〉 | ⚠️ **ファイルリポジトリの `type` 名前空間規約** | **[要決定]**（CU-18 で案 (D) を採る場合は必須）。ファイルリポジトリには **AI モデル・設定スナップショット・（(D) なら）画像**が混在する。**識別子が無いと CK-8(a) と X-04 削除バッチが実装できない**。例: AI モデル `x_AiModel` / 画像原本 `x_Snapshot` / サムネイル `x_SnapshotThumb` | 規約 + バリデータ | — |

> ⚠️ **`files` リポジトリにはリテンションが適用されません** [確]。ソフトウェアリポジトリは MongoDB 格納のため、**保持世代数と削除運用を決めないと肥大化します**（M-03）。

> ⚠️⚠️ **CU-18 で案 (D) を採ると、個人情報を含む画像が AI モデルと同じリポジトリに同居します** 〈rev.3 で追加〉。
> **`type` による分離は権限境界ではありません** — `ROLE_INVENTORY_ADMIN` を持つ **R-10（保守）と R-06b** は、画像も読み・差し替え・削除できます（§11.4）。**M-04 の規約は「実装のための識別子」であって、統制ではないこと**を取り違えないでください。統制は CK-8(b) / AS-14(b) のロール検査側です。

#### 13.3.10 アプリケーション（P 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **P-01** | 紐づけ確認アプリ | ⚠️ **VM2 ホストのため `type: EXTERNAL` で登録**（`createHostedApplication` は不適）。**[要 SV-24]** | `c8y api POST /application/applications` | **C: 存在チェック** |
| **P-02** | テナントのアプリ購読 | `POST /tenant/tenants/{t}/applications` | `c8y api` | **E: 409 黙殺** |
| **P-03** | 標準ダッシュボード | Cockpit の JSON エクスポート/インポート。**[要 SV-11]** Edge での可否 | — | — |

#### 13.3.11 SSO（A 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **A-01** | `loginOptions` / `authConfig` | §11.6。**Keycloak 担当から受領した issuer / clientId / JWKS / クレーム名** | `c8y api POST /tenant/loginOptions`（メディアタイプ **`application/vnd.com.nsn.cumulocity.authconfig+json`**） | **B: PUT 先行 → 404 なら POST** |
| **A-02** | `onNewUser.dynamicMapping.mappings[]` | R-01〜R-03 / R-10 とアプリアクセスの割当 | 同上（A-01 に含まれる） | 同上 |
| **A-03** | ⭐ `onNewUser.dynamicMapping.inventoryMappings[]` | **拠点 Inventory ロールの割当 ＝ 拠点分離の実装点** | 同上 | 同上 |
| **A-04** | ログインモード切替 | ⚠️ **§11.6 ガード 8（切替前ゲート）合格が必須** | UI / API | — |
| **A-05a** | パスワードポリシー | §13.3.4 | `c8y tenantoptions updateBulk` | 上書き |
| **A-05b** | TFA | §13.3.4 | **`c8y tenants tfa update`** | 上書き |

> ⚠️ **エクスポート JSON から `id` を必ず除去してください**（リクエストボディでは `readOnly`）[確]。

> ⚠️ **クレーム名の 1 文字違いで全ユーザーがログイン不可になります**（§11.6 ガード 4）。**投入前に Keycloak 担当と値を突き合わせてください。**

#### 13.3.12 規約文書とバリデータ（設定ではない成果物）

**[判] 次の 5 つは Cumulocity の設定ではありませんが、設定と同じリポジトリで版管理してください。設定より先に確定が必要です。**

| # | 成果物 | 対応する定義 | 置き場所 |
|---|---|---|---|
| **C-1** | **イベント/アラーム型・ペイロード規約**（16KB 制約・`boundingBoxes` 上限を含む） | §6.4 / §6.5 / §6.7 | `config/platform/event-payload-spec.md` |
| **C-2** | **external ID 採番規約**（仮名化要否を含む） | §2.2 | `config/platform/external-id-spec.md` |
| **C-3** | **拠点コード命名規約** | §2.5 | 同上 |
| **C-4** | **ローカル MQTT 受け口の IF 契約** | §6.3 / §6.5 / §6.6 | `config/platform/local-mqtt-if-spec.md` |
| **C-5** | **エンティティ登録規約**（`@id` 統制） | §2.6 | 同上 |
| **V-1** | **ペイロードバリデータ**（C-1 の機械検査） | §6.5 | `scripts/validate-payload.*` |
| **V-2** | **インベントリ検査**（CK-1〜CK-8） | §2.8 | `scripts/assert-inventory.*` |

### 13.4 投入順序と依存関係

```
[前提] §12.9 のセットアップ完了判定が全て ✅
   │
   ├─ 規約の確定（C-1〜C-5）── ⚠️ これが遅れると下流が全部止まる
   │
   ▼
① CLI セッション準備（management / edge。C8Y_MODE=ci）
   ▼
② テナントオプション（T-01, T-04）／ CORS（T-02）／ フィーチャートグル（P-04）
   ▼
③ ロール定義（R-01〜R-03, R-10）→ ロールへの権限割当
   ├─ Inventory ロール定義（R-04）
   ├─ サービスユーザー（R-05, R-06, R-08, R-09, R-11, R-12）
   ├─ break-glass（R-07・両テナント）
   ├─ パスワードポリシー / TFA（A-05）
   └─ サービスアカウント復旧手順の確認（削除 → 再作成 → ロール再割当）
   ▼
④ 拠点グループ階層（G-01・x_Site を本文に含めて 1 ステップ / G-02 は rev.2 で廃止）
   ▼
⑤ テナント CA（D-10）── 前提 3 の結論が必要
   ▼
⑥ alarm.type.mapping（T-03）── C-1 の確定が前提
   ▼
⑦ リテンション（X-01）── AUDIT 日数の確定が前提。新規作成 → 旧削除の順
   ▼
⑧ SSO（A-01〜A-03）→ ⚠️ 切替前ゲート（A-04 の前）→ ログインモード切替（A-04）
   │                    → フォールバック確認（management ローカル admin + edge break-glass）
   ▼
⑨ ルール（S-01, S-02, S-03）── C-1 と SV-32 の結論が前提
   ▼
⑩ ソフトウェアリポジトリ（M-01）／ 設定リポジトリ・デバイスプロファイル（K-03, K-04）
   ▼
⑪ Notification 2.0 サブスクリプション（N-01）── ④（グループ MO ID）と C-1（型）が前提
   └→ トークン発行プロキシ配備（N-02）── ③（R-09）と ⑧（OIDC 検証）が前提
   ▼
⑫ アプリケーション登録（P-01, P-02）／ ダッシュボード（P-03）
   ▼
⑬ オフロードバッチ（X-03）── ⚠️ 稼働開始から 90 日以内が期限
   ▼
⑭ デバイスのプロビジョニング（D-07, D-08, D-09）── 段階ロールアウト
   ▼
⑮ §13.7 の投入後アサート → §14 の検証
```

**⚠️ クリティカルな依存**:

| 依存 | 内容 |
|---|---|
| **C-1（型規約）→ ⑥⑦⑨⑪** | 型名は `alarm.type.mapping` のキー・リテンションの type フィルタ・ルール条件・購読の `typeFilter` の**全ての結節点** |
| **④（グループ）→ ⑪** | サブスクリプションの `source` にグループの MO ID が要る |
| **前提 3 → ⑤** | NG なら登録方式そのものが変わる |
| **⑧ の切替前ゲート → A-04** | ゲート未合格で切り替えると全ユーザーがログイン不可になり得る |
| **AUDIT 日数 → ⑦** | 既定ルールを消すと再作成すべき値が必要 |
| ⭐ **CU-18 →（法務）→ CU-5 → ⑦** 〈rev.3〉 | **CU-18 が「監査レコードを書く」結論なら、個人情報アクセス記録の保存義務期間が `AUDIT` の日数を支配します**（§7.2）。**保持日数の短縮方向は不可逆**なので、CU-18 未決のまま暫定値を投入すると事故になります。→ **CU-18 未決なら ⑦ の `AUDIT` 行だけをスキップ**（§13.3.6） |
| **CT-5c → CU-23 → CU-18** 〈rev.3〉 | ⚠️ **CU-18 は「実機不要」ではありません。** (A)/(A′)/(C) は設計判断だけで選べますが、**(D)/(B) の評価には CT-5c（Edge 導入後の検証環境）が要ります**。**(D)/(B) を残すなら CT-5b / CT-5c を §12.4 直後に前倒ししてください**（§14.12 の 1d / 2b） |
| **CU-18 → R-14 / S-16 / M-04 / X-04 → ③⑬** 〈rev.3〉 | 決定するまで**画像配信マイクロサービス（R-14）・ファイルリポジトリの `type` 規約（M-04）・画像削除バッチ（X-04）**が起票できません。→ **§15.5 の「CU-18 決定後にやること」** |

### 13.5 冪等化パターンの割り当て

#### ⭐ 前提: どの API が重複を弾くか 〈rev.2 で追加〉

**冪等化パターンの選択は、この表がすべての根拠です。** API ごとに挙動が正反対なので、思い込みで選ばないでください。

| API | 重複チェック | 根拠（Release 2026 OpenAPI） | 確度 |
|---|---|---|---|
| **`POST /inventory/managedObjects`** | ❌ **無い** | 応答定義は **`201` / `401` / `403` / `422` のみ**。**同じ `name` / `type` / フラグメントの MO をいくつでも作れます** | [確] |
| **`POST /identity/globalIds/{id}/externalIds`** | ✅ **ある（409）** | *"**409**: Duplicate – Identity already bound to a different Global ID."* — **(`type`, `externalId`) がユニークキー** | [確] |
| **`POST .../{id}/childAssets`** | ✅ **ある（409）** | 応答定義には無いが、go-c8y-cli 公式例が *"silence 409 (duplicate) errors"* と明示 | [確] |
| **`POST /user/inventoryroles`** | ✅ **ある（409）** | 409 "Duplicate" | [確] |
| **`POST /devicecontrol/newDeviceRequests`** | ❌ **無い** | `201` / `401` / `403` / `422` のみ | [確] |
| **SmartREST `100`（MQTT のデバイス作成）** | ✅ **ある（find-or-create）** | *"Create a new device for the serial number in the inventory **if not yet existing**. An externalId for the device with type `c8y_Serial` and the device identifier of the MQTT clientId as value will be created."* | [確] |

> ⚠️⚠️ **原子的な「external ID 付き MO 作成」は Edge 2026 では使えません** [確]。
>
> **これは製品の違い（Edge か否か）ではなく、バージョンの差です。** Cumulocity の API 仕様は版ごとに公開されており、**`Latest version`（継続更新チャネル）の spec には `managedObjectCreate.externalIds` があります**。次のように**作成と external ID の紐づけが原子的**になります:
>
> > *"External identifiers can be assigned to managed objects upon creation by specifying a list in `externalIds` field. Given external identifiers must be valid and **not be already assigned to another managed object** or the request to create a single or multiple managed objects will **fail with a client error and not create anything**."*
>
> **しかしこのフィールドは Release 2026 / 2025 の OpenAPI には存在しません**（`externalIds` / `not be already assigned to another managed object` のいずれも 0 件。`Latest version` のみ 1 件。3 版を実際に取得して比較）。
>
> **Edge 2026 が使うのは Release 2026 です** [確]: *"Cumulocity Edge release 2026 uses the **Cumulocity platform release 2026**."* / *"Edge uses the **same software** as Cumulocity platform … the base software is the same, there are differences regarding the **activated optional features and pre-installed agents**."*
>
> **つまり「Edge だから機能が削られている」のではなく、単に Release 2026 の時点でこのフィールドがまだ無いだけです。** Edge が将来 `externalIds` を含む版へ上がれば使えるようになります（そのとき §13.5 の 2 ステップ問題は消えます）。**現時点では使えない前提で設計してください**（CU-15）。
>
> ⚠️ **[要] spec に無いことは実装に無いことの強い証拠ですが、同一ではありません。** 検証環境の Edge で `externalIds` 付きの `POST /inventory/managedObjects` を 1 度試し、無視されるか 422 になるかを確認しておくと確実です。

#### ⚠️⚠️ 帰結 — REST 経由の MO 作成は「重複」ではなく「孤児」を生みます

Edge では external ID 付きの MO 作成が **2 ステップ**になります。ここに落とし穴があります。

```
① POST /inventory/managedObjects            → 201（重複チェックが無いので必ず成功し、MO ができる）
② POST /identity/globalIds/{id}/externalIds → 409（その external ID は既に別の MO に bound）
```

**①が成功して②が失敗するため、external ID を持たない MO が残ります。** 再実行のたびに孤児が増え、しかも **②の 409 を「冪等だから」と黙殺する実装では気づけません。**

| 経路 | 再実行したときに起きること |
|---|---|
| **デバイス（thin-edge / MQTT）** | ✅ **安全**。SmartREST 100 が external ID をキーに find-or-create する。再接続しても重複しない |
| **プロビジョニングツール（REST・external ID を使う）** | ⚠️ **孤児 MO が増える**（上記） |
| **プロビジョニングツール（REST・`x_Site.siteId` を使う）** | ⚠️ **重複 MO が増える**。ただし作成が 1 ステップなので**孤児にはならず**、CK-1c で検出できる |

> **[判] REST で MO を作る箇所は、例外なく「作成前に存在確認」してください**（パターン C）。**409 黙殺（パターン E）を MO の作成に使ってはいけません。** 409 黙殺が有効なのは、上表で ✅ が付いている **childAssets 追加・Inventory ロール・アプリ購読**だけです。
>
> **[判] `x_Site.siteId` を同定キーにする**（§1.4）と、作成が 1 ステップに収まるため孤児が発生しません。**これがグループの external ID を廃止したもう 1 つの理由です。**

> ⚠️ **存在確認 → 作成には TOCTOU レースが残ります。** プロビジョニングツールを**同一拠点に対して並列実行しない**こと（拠点間の並列は問題ありません）。CI では拠点ごとに排他制御してください。

| パターン | 内容 | 適用先 |
|---|---|---|
| **A: 名前ベース参照** | `--group` / `--role` がワイルドカードを受ける | ロール割当（R-01〜R-03, R-10） |
| **B: PUT 先行 → 404 なら POST** | 更新優先 | `loginOptions`（A-01）、EPL apps（S-02） |
| **C: 存在チェック → 分岐** | GET してから作成 | ユーザー（R-05〜R-12）、グローバルロール定義、ソフトウェア（M-01）、サブスクリプション（N-01）、child device（D-07） |
| **D: 宣言的な集合適用** | ⚠️ **新規作成 → 旧削除の順** | リテンション（X-01） |
| **E: 重複エラー黙殺（409）** | 作成を試みて 409 を無視 | Inventory ロール定義（R-04）、**`childAssets` 追加（D-08）**、テナントのアプリ購読（P-02）<br>⚠️ **rev.2 でグループ（G-01 / G-02）を除外しました** — `POST /inventory/managedObjects` は 409 を返さないため、黙殺しても重複が防げません（§13.3.3） |

### 13.6 リポジトリ構成

> **本節は [設定書] §6.2 を引き継ぎ、本書の章立てに合わせて拡張したものです。**

```
config/
  edge/                              # L0（環境ごと）
    edge.yaml                        # kubectl get edge -o yaml（status/uid 除去）
    operator-config.yaml             # ca.crt / no_proxy（E-11）
  platform/                          # L1-B: 基盤標準初期値（環境非依存・製品）
    tenantoptions.alarm-type-mapping.json   # T-03（§13.3.1）
    tenantoptions.configuration.json        # T-01 / T-04
    usergroups.json                         # R-01〜R-03, R-10
    inventoryroles.json                     # R-04
    features.json                           # P-04
    retentionrules.json                     # X-01
    password-policy.json                    # A-05
    event-payload-spec.md                   # ★C-1（規約文書）
    external-id-spec.md                     # ★C-2 / C-3
    local-mqtt-if-spec.md                   # ★C-4 / C-5
    epl/*.mon                               # S-02（リプレイ抑止ガードは別ファイル）
    smartrules/*.json                       # S-01
    dashboards/*.json                       # P-03
  site/                              # L1-C: 構成固有（環境ごと）
    tenantoptions.access.control.json       # T-02（CORS: VM2 の全オリジン）
    loginoptions.oauth2.json                # A-01〜A-03（id 除去済み）
    sites/                                  # ★拠点定義（§13.3.3）
      site001.yaml  site002.yaml …
    notification-subscriptions.json         # N-01（拠点ごとに生成）
    users.json                              # R-05, R-06, R-08, R-09, R-11, R-12（＋CU-18 決定後に R-14）
    deviceprofiles/*.json                   # K-04
  mgmt/                              # management テナント管轄
    branding/
binaries/                            # ZIP は Git LFS または成果物リポジトリ
secrets/                             # 値は入れない。キー名の一覧のみ
scripts/
  export.sh   import.sh
  assert-config.sh                   # §13.7 往復差分
  assert-inventory.sh                # ★CK-1〜CK-8（§2.8）
  validate-payload.sh                # ★C-1 のバリデータ
```

> ⚠️ **`secrets/` に値を入れないでください。** R-05〜R-12 の資格情報は CI シークレットストアで管理します（§4.10 G-14）。

### 13.7 投入後アサート

**[判] 「投入した」ではなく「投入結果が期待どおり」を完了条件にします。**

| # | アサート | 合格条件 | 実装 |
|---|---|---|---|
| **AS-1** | **設定の往復差分** | **Git 定義 vs 実機の差分がゼロ** | `scripts/assert-config.sh`（GET → 正規化 → diff） |
| **AS-2** | システムオプションのアサート | 期待どおり | `c8y systemoptions list` |
| **AS-3** | ⭐ **`ROLE_NOTIFICATION_2_ADMIN` の付与先検査** | **R-09 以外に付いていない** | CK-6。**CI に載せる** |
| **AS-4** | ⭐ **Cloud Remote Access ロールの検査** | **`ROLE_REMOTE_ACCESS_ADMIN` がどのロール・どのユーザーにも付いていない**（既定で未割当なので、維持されていることの確認） | R-13。CI |
| **AS-5** | ⭐ **external ID の規約準拠** | 規約外の MO が 0 件 | CK-1。CI |
| **AS-6** | device type の規約準拠 | 4 種以外が 0 件 | CK-2。CI |
| **AS-7** | `vmsCameraId` の設定 | 未設定が 0 件 | CK-3。CI |
| **AS-8** | main device の拠点グループ所属 | 未所属が 0 件 | CK-4。CI |
| **AS-9** | external ID プレフィックスと実所属拠点の一致 | 不一致が 0 件 | CK-5。**日次** |
| **AS-10** | child device のフラグメント検査 | `c8y_SupportedOperations` と `com_cumulocity_model_Agent` が付いていない | CI |
| **AS-11** | リテンションルールの存在 | **6 種**（`ALARM` / `AUDIT` / `BULK_OPERATION` / `EVENT` / `MEASUREMENT` / `OPERATION`）のうち設計した種別が存在すること〈rev.3 で修正 — `dataType` の enum は 6 種 + `*` で、「5 種」は `BULK_OPERATION` が欠落していた〉。⚠️ **「既定 60 日ルールが残っていない」の検査は [要 SV-43] の結果が出るまで保留**（§7.2） | CI |
| **AS-11b** 〈rev.3〉 | ⭐ **`AUDIT` リテンションの値検査** | **`dataType=AUDIT` のルールが設計どおりの本数だけ存在し、`maximumAge` が CU-5 で決めた値以上であること。想定外の `type` / `source` 絞り込みルールが増えていないこと。**（可能なら `editable: false` であること） | CI。⚠️ **証跡を狙い撃ちで消す経路への唯一の歯止め**（§7.2） |
| **AS-12** 〈rev.3 で強化〉 | サブスクリプション定義 | 拠点数ぶん存在し、`fragmentsToCopy` が指定されている。⭐ **かつ、その中身に `c8y_IsBinary` / `x_SnapshotRef` / 個人情報を含むフラグメントが含まれないこと**（**否定リスト検査**） | CI |
| **AS-13** 〈rev.3・レビューで拡張〉 | ⭐ **監査ロールの付与先検査** | **(a)** `ROLE_AUDIT_ADMIN` が**人間ユーザーおよび人間向けグローバルロール（R-01〜R-03 / R-10）のいずれにも付いていない**。付与先は R-14 のみ（R-15）<br>**(b)** ⭐ **全 Inventory ロール（`GET /user/inventoryroles`）の `permissions` に `scope: AUDIT` が含まれないこと**（R-16。§11.2）<br>**(c)** `ROLE_AUDIT_READ` の付与先が R-01 と監査担当のみであること | CI |
| **AS-14** 〈rev.3・レビューで対象拡大〉 ⚠️ **CU-18 で案 (D) を採る場合のみ** | ⭐ **スナップショットのバイナリ MO の隔離検査** | **(a)** 画像用 `type`（M-04）のバイナリ MO が**どの拠点グループの `childAssets` にも入っていない**<br>**(b)** ⚠️ **グローバルの `ROLE_BINARY_READ` / `ROLE_BINARY_ADMIN` / `ROLE_INVENTORY_READ` / `ROLE_INVENTORY_ADMIN` を持つ主体が R-14 と（実体確認後の）R-01 だけであること。** 検査対象は **R-01 / R-02 / R-03 / R-05 / R-06a / R-06b / R-08 / R-10 / R-11 / R-12**（§15.3 案 (D)） | CK-8。CI |
| **AS-15** 〈rev.3〉 ⚠️ **CU-18 決定後** | **画像配信マイクロサービス（R-14）の同一性検査** | **デプロイされているアプリケーションのバイナリ ID / バージョンが Git 定義と一致すること。** ⚠️ **R-01 は Application management ADMIN を持つため、マイクロサービスを差し替えられる主体は実質 `ROLE_AUDIT_ADMIN` を持つのと同じ**です。AS-13 だけではこの経路を塞げません | CI |

> ⚠️ **AS-3 / AS-4 / AS-5 / AS-11b / AS-13 / AS-14 / AS-15 は「壊れても長期間気づけない」種類の破綻です。CI に載せることが唯一の歯止めです。**

---

## 14. 検証観点

### 14.1 検証の考え方

> **[判] 3 つの原則**:
>
> 1. **「設定した」を完了条件にしない。** スマートルールが「一覧に見える」ではなく「実際にアラームが生成された」を確認する
> 2. ⭐ **否定テストを主にする。** 拠点分離・supported operations・Remote access は「できないこと」が要件であり、肯定テストでは担保できない
> 3. **機械検査できるものは CI に載せる。** 人が目視する検証は、拠点が増えたときに必ず抜ける

**ID 体系**: 本書は **`CT-nn`**（Cumulocity Test）を使います。[担当範囲] §9 の `VT-nn`（受け入れ試験）・[設定書] §5 P7 の工程とは別体系です。⭐ は**合格しないと本番運用に入れない項目**です。

### 14.2 デバイスモデルと拠点分離

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-1** | 実デバイス 1 台の疎通 | **正しい拠点グループの配下**に現れる |
| **CT-2** | ⭐ **external ID の規約準拠** | 全デバイス・全 child が `{siteId}:{種別}-{シリアル}`。**規約外の MO が 1 つも無い**（CK-1・**CI に載せる**） |
| **CT-3** | `vmsCameraId` の設定 | **全カメラに設定されている**（未設定 0 件・CK-3） |
| **CT-4** | ⭐⭐ **「配下をどこまで辿るか」の一括検証**（前提 2 / SV-35 ＋ 前提 7 / SV-47） | **1 つの階層構成で通知と Inventory ロールを同時に検証する。**<br>**構成**: 拠点グループ G の `childAssets` にサブグループ G1 → G1 の `childAssets` にデバイス D → **D の `childDevices` にカメラ C**<br>**(a) 通知**: `{context:"mo", source:{id:G}, apis:["alarmsWithChildren"]}` で購読し、**D のアラームと C のアラームのどちらが届くか**<br>**(b) Inventory ロール**: G に Inventory ロールを割り当てた拠点 Manager が、**D と C のアラーム・インベントリを REST で取得できるか**<br>**判定**: D のみ → `childAssets` 限定。C も → `childDevices` も辿る<br>**NG の場合**: (a) は §1.6 の代替案 (i) へ。(b) は §1.3 の階層設計変更（カメラも `childAssets` で拠点グループに直付け）へ |
| **CT-5** | ⭐ **Inventory ロールの否定テスト** | 拠点 A のユーザー資格情報で拠点 B のデータを **API 直叩き**して 403/404 |
| **CT-5b** 〈rev.3〉 | ⭐ **添付画像の直叩き確認**（**肯定テスト**・§6.8 制約 6） | **拠点 Reader（R-04b・Events READ のみ）の資格情報で `GET /event/events/{id}/binaries` を直叩きし、スナップショット画像が 200 で取得できてしまうことを確認する。**<br>⚠️ **「取得できないこと」を期待する試験ではありません。** OpenAPI からは「取得できる」と読めますが、**これが実機で確認できると CU-18 の判断根拠が確定します**（＝ UI 側の対策だけでは監査証跡にならないことの実証）。<br>**併せて、UI から画像を表示・ダウンロードした直後に `GET /audit/auditRecords` を引き、対応する監査レコードが 1 件も増えていないことを確認する**（制約 5 の実証） |
| **CT-5c** 〈rev.3〉 | ⭐ **`/inventory/binaries` の否定テスト**（案 (D) の成立条件・CU-23 / CU-24） | **サービスユーザーで `POST /inventory/binaries` に画像を上げ、拠点グループ配下に置かない。**<br>**(a)** 拠点 Reader（R-04b）の資格情報で **`GET /inventory/binaries/{id}` が 403 になること**<br>**(b)** 同じ資格情報で **`GET /inventory/binaries`（メタデータ検索）に当該バイナリが列挙されないこと**<br>**(c)** `ROLE_BINARY_READ` を持つサービスユーザーでは **200 で取得できること**<br>⚠️ **(a) が 200 を返すなら案 (D) は成立しません。** CU-18 の選択肢が 1 つ減るので、**CT-5b と同時に W1 で実施してください**<br>**(d)** 〈rev.3 レビューで追加〉**拠点 Reader の資格情報で `GET /audit/auditRecords`（コレクション）が 403 になること。** ⚠️ **この操作だけ OpenAPI に Required roles の記載がありません**（`GET /audit` と `/audit/auditRecords/{id}` には明記あり）。**監査ログ全件＝「誰がいつ誰の顔を見たか」の集合体**なので、ここが開いていると (D) の効果より重い問題になります（CU-26） |
| **CT-5d** 〈rev.3〉 | ⭐⭐ **画像配信マイクロサービス（R-14）の認可の否定テスト**（§11.11） | **拠点 A のユーザーの OIDC トークンで、拠点 B のカメラに紐づく `binaryId` を R-14 に要求 → 403 になること。**<br>⚠️ **これが通ってしまうと、(D) は「直叩きは 403、正規経路は全開」になります。** CT-5c が合格しても本テストが不合格なら**拠点分離は成立していません**。<br>**併せて (b)** イベントが消えた（＝ 90 日経過した）バイナリに対して **fail-close（403）になること**（§11.11 MS-c / MS-d の孤児バイナリ対策）<br>**(c)** リクエストで `user` を偽装しても、**監査レコードの `user` が OIDC トークン由来の値になること**（MS-e） |
| **CT-6** | ⭐ **`ROLE_NOTIFICATION_2_ADMIN` の機械検査** | **R-09 以外のどのユーザー・ロールにも付いていない**（CK-6・**CI に載せる**） |

### 14.3 通知

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-7** | ⭐ **トークン発行プロキシの否定テスト** | ①拠点 A の案件アプリの認証情報で**拠点 B のサブスクリプション**を要求 → 拒否される ②拠点 A 用トークンで**拠点 B のトピックに WebSocket 接続できない** |
| **CT-8** | ⭐ **通知の到達試験** | 拠点 A でアラームを発生させ、①**拠点 A の案件アプリに届く** ②**拠点 B の案件アプリには届かない** ③**`fragmentsToCopy` で除外したフラグメントが含まれない** |

### 14.4 死活監視

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-9** | ⭐ **child の可用性アラーム抑止**（§5.3 の落とし穴） | **接続から 1 時間以上経過しても、カメラ・BOX に `c8y_UnavailabilityAlarm` が 1 件も出ない**（`responseInterval: 32767` が効いている） |
| **CT-9b** 〈rev.2〉 | ⭐⭐ **メンテナンスモードの副作用の否定テスト** | **`responseInterval: 0` にした child へ `x_CameraDown` を POST し、実際に登録されないことを確認する**（レスポンスの `self` が `/alarm/alarms/null` になる）。**そのうえで `32767` では正常に登録されることを確認する**。⚠️ **この 2 つを比較しないと、§5.4 の訂正が効いているか判定できません** |
| **CT-9c** 〈rev.2〉 | ⭐ **117 が既存値を上書きしないことの確認**（§5.5） | ①Cumulocity 側に先に `responseInterval` を入れてから thin-edge を初回接続 → **①の値が残る**<br>②thin-edge を先に接続してから `tedge config set c8y.availability.interval` を変更 → **Cumulocity 側の値は変わらない**<br>**②が確認できたら、既存拠点向けの一括置換手順（TE-5b）を確定する** |
| **CT-10** | **死活監視サービスの死** | サービスを kill → **Cumulocity 上でサービスが `down` になる**（LWT）→ watchdog が再起動 → **起動時にアラーム状態が再評価される** |
| **CT-11** | ゲートウェイ断 | 通信停止 → `responseInterval` 後に `c8y_UnavailabilityAlarm` → 復旧で**自動クリア** |
| **CT-12** | カメラ断 | 無応答 N 回で `x_CameraDown` → 復旧でクリア。**ヒステリシスが効きフラッピングしない** |

### 14.5 データとルール

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-13** | **severity 変更時の挙動**（§9.4） | **公式の予測どおり「severity 変更が無視される」ことを 1 ケースで確認する**（*"Any other changes are ignored"*）。⚠️ rev.1 は「観測して規約に反映する」としていたが、**公式に答えがあるため型規約の確定を待つ必要はない**（§9.4） |
| **CT-13b** 〈rev.2〉 | ⭐ **アラームマッピングの前方一致の確認**（§9.2） | `x_Alarm_` をキーに 1 件だけマッピングを置き、**`x_Alarm_Intrusion` と `x_Alarm_Loitering` の両方に効く**ことを確認する。**併せて、意図しない型（例 `x_Alarm` で始まる別型）が巻き添えにならないか**を AM-a〜AM-c の観点で確認 |
| **CT-14** | **ペイロード規約の検証** | ①16KB 超のイベントが HTTP に切り替わることを確認 ②16KB 超の**計測が拒否される**ことを確認 ③バリデータが規約違反を検出する |
| **CT-15** | ⭐ **リプレイ抑止**と**欠落**の両方 | ①網断復旧の一括再送で、**通知が復旧時刻にまとめて発火しない**（リプレイ抑止ガードが効いている）②**イベント自体は全件 Operational Store に残っている**<br>⚠️ **③〈rev.2 で追加〉30 分の網断中に送信したメッセージ数と、復旧後に Cumulocity に到達した数が一致すること。** thin-edge のブリッジ設定に `max_queued_messages` が無く、mosquitto 既定の 1000 件を超えると**欠落**しうる（SV-44・§8.4）。**欠落するならキュー容量の調整が先** |

### 14.6 オペレーション

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-16** | ⭐ **supported operations の否定テスト** | **カメラ・BOX の詳細画面にオペレーションの実行 UI が出ない** |
| **CT-17** | ⭐ **Remote access の無効化** | Cumulocity UI に Remote access のボタンが出ない。**`114` の一覧に `c8y_RemoteAccessConnect` が無い**。Cloud Remote Access ロールがどのユーザーにも付いていない |
| **CT-18** | モデル配布の成功 | 適用 → `x_ModelApplied` イベント → 以後の検知イベントの `modelVersion` が新しい |
| **CT-19** | ⭐ **モデル配布の失敗とロールバック** | 壊れたモデルを配布 → **旧モデルに戻って稼働継続** → Operation が FAILED → `x_ModelUpdateFailed` アラーム（CRITICAL） |
| **CT-20** | 設定配布 | `c8y_DownloadConfigFile` で設定が変わる。**`c8y_UploadConfigFile` で現在値が吸い上がる** |
| **CT-21** | ログ取得 | `c8y_LogfileRequest` でログが取得でき、**個人情報がマスクされている** |

### 14.7 データ保持・オフロード

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-22** | **リテンションの実効確認**（**検証環境で実施**） | ①短期リテンションを設定して観測 ②**`CLEARED` でないアラームが削除されないこと**を確認 ③**添付バイナリが対象に含まれるか**を確認（SV-06） |
| **CT-23** | **オフロード検証** | オフロード先の件数が Cumulocity 側と一致し、**external ID が含まれ GSC サイドカー JSON と突合できる**。**`prev`/`next` で 2000 件超を正しく辿れる** |

> ⚠️ **CT-22 は必ず検証環境で実施してください。本番でリテンション検証をすると本番データが消えます。**

### 14.8 運用・ガードレール

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-24** | ⭐ **キルスイッチ**（§4.10 G-1） | 一括オペレーションを実際に発行 → **3 手順で実際に止まる**（①書込アカウント無効化 ②bulk キャンセル ③PENDING 一括削除） |
| **CT-25** | **除外タグ**（§4.9 R-d） | `x_NoAutoConfig` を付けたデバイスが配信対象から外れる |

### 14.9 認証・SSO

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-26** | ⭐ **SSO 切替の否定テスト** | ①**手動付与した Inventory ロールが次回ログインで消える**（`inventoryMappings` が正） ②**SSO 対象外の主体（デバイス・サービスユーザー・R-08・break-glass）が SSO 切替の影響を受けない** ③各ロールで**期待通りの拠点だけが見える** |
| **CT-27** | ⭐ **SSO からローカル認証への切り戻し**（SV-04） | **実機で手順を実行し、runbook 化されている**。UI から選択肢が消えた状態で `PUT /tenant/loginOptions/{typeOrId}` が Basic 認証の API で叩けるか |

### 14.10 規模

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-28** | ⭐ **合算規模の負荷試験**（SV-14） | §7.7 の Wide シナリオ 1,200 クライアントに対する実測。**接続数（拠点数 × 2）とレート（child 台数比例）の両面**で測定し、**tps・E2E 遅延・CPU・MongoDB IO で数値化** |
| **CT-29** | **死活計測の送信量** | [連携仕様] §3.5 の試算（50 拠点 × 300 台 ≒ 250 msg/s）に対する実測と余裕率。**不足する場合は §5.9 の削減方針を適用** |

### 14.11 その他

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-30** | **コンシューマ棚卸し試験**（N-04） | 案件アプリを停止 → **バックログが増えることを観測** → `POST /notification2/unsubscribe` で解除できる |
| **CT-31** | 証明書の自動更新 | `tedge-cert-renewer@c8y.timer` が動作し、`.new` → 検証 → 差し替えが完了する |
| **CT-32** | ⭐ **証明書更新の輻輳シミュレート**（SV-25 改訂） | ⚠️ **rev.1 の「10/2 の CA 更新日をシミュレート」は不要になりました**（CA 更新でデバイス証明書は失効しない・§3.7）。**代わりに「同一日に発行された多数のデバイス証明書が、一斉に更新期（期限 30 日前）を迎える」状況を再現し、`simplereenroll` が輻輳せず完了することを確認してください。** `RandomizedDelaySec=5m` のジッターで足りるかを判定 |
| **CT-32b** 〈rev.2〉 | ⭐ **初回 `simpleenroll` の成立**（前提 3 / SV-25b） | ⚠️ **CT-32 とは別のテストケースとして独立させること。** rev.1 は初回エンロールの基礎検証を CA 更新シミュレーションに暗黙に含めており、**前提 3 本来の検証が抜けていました。** 閉域網で `tedge cert download c8y` が成功し、証明書が取得できることを単独で確認する |
| **CT-33** | **CORS 検証** | VM2 の各アプリから実際にブラウザ経由で Edge の REST を叩き 200 が返る |
| **CT-34** | **設定の往復差分** | **Git 定義 vs 実機の差分ゼロ**（AS-1） |

### 14.12 検証と未確定事項の対応

**⭐ 項目のうち、「NG だと設計そのものが変わる」ものを先に実施してください。**

| 優先 | 試験 | NG だと何が変わるか |
|---|---|---|
| **1** | **CT-4**（配下の辿り方） | **§1 の拠点分離方式・§10 の購読設計・§11.3 の Inventory ロールが同時に破綻**。§1.6 の代替案、または §1.3 の階層設計変更へ |
| **2** | **CT-32b / 証明書の EST 可否**（前提 3） | §3.3 の登録方式が変わり、§3.6 の自動更新を別途設計 |
| **1b** 〈rev.2〉 | **SV-43**（リテンション既定 60 日の正体） | **90 日ルールを入れても 60 日で消える可能性。§7.2 のデータ保持設計が崩れる** |
| **1c** 〈rev.2〉 | **SV-45**（Apama-ctrl バリアント） | **§8 の RL-b・RU-2〜RU-5 が全て EPL 前提。§8 の過半が成立しない** |
| **1d** 〈rev.3〉 | **前提 9 / CU-18**（画像の置き場所と監査方式） | **§6.8 の初期版設計・§11.3 のロール割当・§13 の投入対象・構成図が同時に変わる。** ⚠️ **CU-5（`AUDIT` 日数）が決まらず §13.4 ⑦ が止まる**。**保持日数の短縮は不可逆**なので暫定値で進めてはいけない |
| **2b** 〈rev.3〉 | **CT-5b + CT-5c + CT-5d**（§14.2） | **CT-5c(a) が 200 なら案 (D) が消え、CU-18 は (A)/(A′)/(B)/(C) に絞られる。** **CT-5d が 403 にならないなら (D)/(B) の「拠点分離できる」が成立しない**（正規経路が全開）。⚠️ **CU-18 は「実機不要」ではありません** — (D)/(B) を残すなら §12.4 直後に前倒しすること |
| **3** | **CT-8**（通知の到達） | §10 が成立しない。**変-2 と併せて通知経路がゼロ**になる |
| **4** | **CT-13**（severity の 2 層規則） | §6.4 の型カタログと §9 のマッピングが変わる |
| **5** | **CT-9 / CT-9b**（child の可用性とアラーム抑止） | 対処方針 2 が効かない場合、§5.4 の対処 3（`c8y.availability.enable false`）または 4（`@health`）へ。**4 を採るとアラーム型規約が変わる** |
| **6** | **CT-28**（合算規模） | 拠点数の上限・死活間隔・Edge のリソース設計が変わる |

---

## 15. 未確定事項・前提条件

> **ID 体系**: 本書で新たに提起するものは **`CU-nn`**（Cumulocity Undetermined）。既存の `SV-nn`（[設定書]）・`TE-nn` / `TB-nn`（[担当範囲]）・`V-nn`（vendor-questions）との対応を併記します。

### 15.0 rev.2 での増減 〈ファクトチェック反映〉

**一次情報照合の結果、[要] 項目が入れ替わりました。** 詳細は [ファクトチェック結果](Cumulocity利用設計書_ファクトチェック結果.md) を参照してください。

#### ⭐ 解消・縮小できたもの（W1 の検証工数から外せる）

| 項目 | 結論 | 該当 |
|---|---|---|
| **SV-33** Notification 2.0 の監視可否 | **解消。** `Administration > Monitoring > Messaging Service` に公式の監視画面と危険域がある | §10.5 |
| **SV-25 / TE-7** CA 更新時の再エンロール | **解消。** CA 更新で既存デバイス証明書は失効しない。CT-32 は「初回エンロール日の集中」に付け替え | §3.7 |
| **CU-13 / CT-13** severity 変更時の挙動 | **解消。** *"Any other changes are ignored"* ＝ severity 変更は無視される。CT-13 は 1 ケースの確認に縮小 | §9.4 |
| **SV-06** 添付バイナリのリテンション | **解消。** 及ぶ（[確K]）。ただし `files` リポジトリには及ばない | §6.8 / §7.4 |
| **CU-1** 拠点コードの形式 | **決定材料が揃った。** `subscription` の `pattern: '^[a-zA-Z0-9]+$'` は [確]。案 (a) を前倒しで確定できる | §1.4 |
| **前提 4 / SV-05** Edge の Notification 2.0 | **「使えるか」は [確]**（`Messaging Service: Included`）。残るのは有効化手順と WS 到達性のみ | §10.1 |
| **CT-22②** CLEARED 以外は消えない | **解消**（公式に逐語） | §7.3 |
| **M-1 / M-2 / m-9** CLI コマンド・設定キーの実在性 | **すべて実在。** go-c8y-cli の既定ブランチが `v2` であることが誤判定の原因だった | §3.6 / §2.6 |
| **TE-6** external ID の文字種・長さ | **解消。** Identity API 側は無制約（スキーマに `pattern`/`minLength`/`maxLength` の定義が無い）。制約が効くのは main device の証明書 CN 経路だけ | §2.7 |
| **SV-07** `ENROLLMENT_OTP` × `PATH` 併用 | **解消。** 併用可（CSV ヘッダ表に独立した任意列として並記）。排他は `CREDENTIALS` × `AUTH_TYPE` のみ | §3.3 |
| **SV-38** `childAssets` 追加の 409 | **解消。** 409 を返す（go-c8y-cli 公式例が `silentStatusCodes` で黙殺している）。⚠️ **グループ作成（G-01）は 409 を返さない**ので混同しないこと | §3.5 / §13.3.3 |

#### ⚠️ 新たに [要] となったもの

| ID | 項目 | 該当 | 重要度 |
|---|---|---|---|
| **SV-43** | **リテンションの「既定 60 日」が削除可能なオブジェクトとして存在するか** | §7.2 / §13.3.6 | **最優先**（§7.2 のデータ保持設計が崩れうる） |
| **SV-45** | **Edge 同梱の Apama-ctrl バリアント**（EPL Apps が使えるか） | §8.1 | **最優先**（§8 の過半が EPL 前提） |
| **SV-47** | **Inventory ロールが `childDevices` まで継承されるか** | §11.3 | **最優先**（前提 2 と同型） |
| **SV-04**（内容変更） | **「ログインモード」の実体が OpenAPI に無い。** SSO 切り戻しが `loginOptions` の PUT だけで戻らないおそれ | §11.6 | 高 |
| **SV-44** | **thin-edge のオフラインバッファ容量**（mosquitto 既定 1000 件を超えると欠落しうる） | §8.4 | 高 |
| **SV-38** | `childAssets` POST の 409 は仕様に定義が無い | §3.5 | 中 |
| **SV-39** | certificate-authority の Preview → GA 移行（全デバイス再登録が要る告知あり） | §3.6 | 中 |
| **SV-40** | sm-plugin の 5 分タイムアウトに AI モデルの install が収まるか | §4.4 | 中 |
| **SV-41** | Edge で Cloud Remote Access マイクロサービスが使えるか（使えないなら §4.7 の懸念が消える） | §4.7 | 中 |
| **SV-42** | イベント本体と添付を分離して後付けする方式を採るか | §6.8 | 中 |
| **SV-46** | ソフトウェアバイナリ書込に追加ロールが要るか（「files 権限」は存在しない） | §11.2 | 中 |
| **SV-48** | SSO redirect 下での break-glass の第 2 要素（TOTP が使えない） | §11.6 | 中 |
| **SV-49** | Edge の `admin` パスワードが operator に上書きされないか | §12.2 / §12.3 | 中 |
| **SV-50** | Edge ライセンスの有効期限・更新サイクル | §12.2 | 中 |
| **TE-23** | MO の `type` が登録後に変更できるか（`@type` の話と混線している疑い） | §3.8 / §2.6 | 中 |
| **CU-8b** | `allow.origin` がスキーム・ポートを含む表記を受け付けるか | §11.8 | 低 |
| **CU-14** | パスワード有効期限はテナント単位。「管理者だけ無期限」は実装不能 | §11.5 / §13.3.4 | 低 |
| **SV-14b** | child device 台数の別軸評価（MO 総数・課金・クエリ性能） | §7.7 | 中 |
| ~~**CU-15**~~ 〈rev.2 でほぼ解決〉 | **Edge 2026 では使えません** [確]。`managedObjectCreate.externalIds` と `/identity/search` は **`Latest version` の spec にのみ存在**し、**Release 2026 / 2025 の spec には 0 件**。**Edge 2026 = platform Release 2026** なので、**原子的な冪等化は使えない前提**で設計する。⚠️ **製品差ではなく版差**なので、Edge の版が上がれば使えるようになる（§13.5） | §13.5 | 低（実機で 1 度確認） |
| **CU-16** 〈rev.2〉 | 実テナントで**同名グループを 2 件作れる**ことの実測（OpenAPI 仕様からの推論） | §13.3.3 | 中 |
| **TE-5b** | 既存拠点の `c8y_RequiredAvailability` 一括置換手順 | §5.5 | 中 |

### 15.0b rev.3 での追加 〈個人情報アクセスの監査〉

**「スナップショット画像に人の顔が判別できる形で写る場合、Cumulocity Edge 上での表示・ダウンロードを監査ログとして残せるか」という問いから、事実確認と論点の起票を行いました。** 事実は該当章に反映済みです。**案の選定（CU-18）は未決のまま残しています。**

| 種別 | ID | 内容 | 反映先 |
|---|---|---|---|
| **[確]**（事実） | — | **参照・DL は Cumulocity の監査ログに残らない。** 記録対象は変更操作とログイン / ログアウトのみで、`auditRecord.type` は 20 値の固定 enum。**Edge 固有の制限ではない** | **§6.8 制約 5** |
| **[確]**（事実） | — | **添付の取得権限をイベント閲覧権限から分離できない。** `GET /event/events/{id}/binaries` は `ROLE_EVENT_READ`。**R-04a / R-04b の保有者は REST 直叩きで画像を取得できる** | **§6.8 制約 6 / §11.3** |
| **[確]**（事実） | — | **`POST /audit/auditRecords` で自前の監査レコードを書ける**（`ROLE_AUDIT_ADMIN`）。書いた後は改竄不能。**Edge は `Microservice Hosting` が `Included`** | **§6.8 / §11.2** |
| **[判]**（ガードレール） | **AS-13** | **`ROLE_AUDIT_ADMIN` を人間ユーザーに付けない**（付けると証跡を捏造できる）。`ROLE_AUDIT_READ` も R-01 と監査担当のみ | **§11.2 / §13.7** |
| **[判]**（ガードレール） | **CK-8 / AS-14** | ⚠️ **CU-18 で案 (D) を採る場合のみ有効。** **(a)** バイナリ MO をどの拠点グループにも入れない **(b)** `ROLE_BINARY_READ` / `ROLE_INVENTORY_READ` を拠点ロールに付けない。**どちらか一方でも崩れると 403 が静かに破れる** | **§2.8 / §13.7 / §15.3 案 (D)** |
| **[確]**（事実）〈追記〉 | — | ⭐ **制約 6 はイベント添付に固有。** `GET /inventory/binaries/{id}` は `ROLE_INVENTORY_READ` / `ROLE_BINARY_READ` / **owner** / **`MANAGE_OBJECT_READ` on the resource** の 4 択で、**判定対象がバイナリ MO 自身に変わる**ため、**格納先を移すだけで権限分離できる**（＝案 (D)）。**「1 イベント 1 バイナリ」も外れる** | **§6.8 / §15.3 案 (D)** |
| **[確]**（事実）〈追記〉 | — | ⚠️ **`POST /audit/auditRecords` は「任意のログ API」ではない。** `type` は固定 enum、`source.id` 必須、`severity` は `readOnly`、書き込みに `ROLE_AUDIT_ADMIN`。⭐ **ただし `PUT` / `DELETE` が存在せず、個別レコードの改竄・削除は API から不可能** | **§15.3** |
| **[要決定]** | **CU-18** | **画像の置き場所と監査方式**（A / A′ / **D** / B / C の 5 案）。**§6.8 の「将来: オブジェクトストレージへ移行」を初期版から必須にするか、あるいは `/inventory/binaries` に移すか** | **§15.3 / 前提 9** |
| **[要決定]**（入力追加） | **CU-5** | `AUDIT` の日数に **③個人情報アクセス記録の保存義務期間**を入力として追加。決定主体に**法務**を追加 | **§7.2 / §15.2** |
| **[要確認]** | **CU-19〜CU-22** | `ROLE_AUDIT_ADMIN` の付与可否 / `user` フィールドの設定可否 / UI のプレビュー有無 / `AUDIT` の `fragmentType` 絞り込み | **§15.3** |
| **[要確認]** | **CU-23〜CU-25** | ⭐ **`/inventory/binaries/{id}` で拠点ユーザーが 403 になるか**（案 (D) の成立条件）/ メタデータ検索で何が見えるか / `source.id` の実在検証 | **§15.3** |
| **[要確認]** | **CU-26〜CU-32** 〈レビューで追加〉 | `GET /audit/auditRecords` の必要ロール / ⭐ **R-01「Tenant Manager 相当」の実体** / 監査レコードの `type` 値 / **S-15（Edge root）の統制** / **バックアップの暗号化・破棄** / `AUDIT` リテンションの時刻基準 / **映像クリップのアクセス監査** | **§15.3** |
| **[判]**（必須要件） | **§11.11**（新設） | ⭐ **画像配信マイクロサービス（R-14）の認可仕様。** ここを落とすと (A)/(A′)/(D)/(B) いずれも「直叩きは塞いだが正規経路は全開」になる | **§11.11 / CT-5d** |
| **[検証]** | **CT-5b / CT-5c / CT-5d** | 添付の直叩き確認（肯定）／`/inventory/binaries` の否定テスト／⭐ **R-14 の認可の否定テスト** | **§14.2 / §14.12 の 2b** |

> ⚠️ **CU-18 が決まるまで起票を保留しているもの**: §11.4 の **R-14**（監査レコードを書くサービスユーザー）、§13 の投入対象、監査レコードのフィールド規約。**先に CU-18 を決めてください。**

### 15.1 本書の前提が崩れるもの（最優先）

| # | 項目 | 確認方法 | 影響 | 既存 ID |
|---|---|---|---|---|
| **前提 2** | **`alarmsWithChildren` が `childAssets` を辿るか** ⚠️ **公式には「descendant managed objects」としか書かれておらず、関係の種別を特定した記述は存在しない**（7 経路で確認済み・§10.2）。**実機確認以外に確定手段なし** | 検証環境（CT-4）**＋ プロダクトサポートへの書面照会を推奨** | **§1 の拠点分離方式と §10 の購読設計が同時に破綻** | **SV-35 / TE-12** |
| **前提 3** | **certificate-authority（EST）が Edge で使えるか** | 検証環境 + ベンダー照会 | §3.3 の登録方式と §3.6 の証明書更新自動化が変わる | **SV-08 / TE-8 / V8** |
| **前提 4** | ~~Edge 上で Notification 2.0 が使えるか~~ → **[確] 使える**（`Messaging Service: Included`）。残るのは **2026 での有効化手順と WebSocket の LoadBalancer 到達性** | 検証環境（CT-8） | 到達できなければ §10 の経路が成立しない。⚠️ **Messaging Service の追加リソース（+2 CPU / +4GB RAM / PV 3 本）を §7.6・§12 に反映すること** | **SV-05** |
| **前提 6** 〈rev.2 で追加〉 | **Edge 同梱の Apama-ctrl バリアントで EPL Apps が使えるか** | §12.4 の導入直後 | **§8 の RL-b・RU-2〜RU-5 が全て EPL 前提。使えないと §8 の過半が成立しない** | **SV-45** |
| **前提 7** 〈rev.2 で追加〉 | **Inventory ロールが `childDevices`（カメラ・BOX）まで継承されるか** | 検証環境（CT-4・前提 2 と同時） | **拠点 Manager がカメラ個別のアラーム・インベントリを見られず §11.3 が崩れる** | **SV-47** |
| **前提 8** 〈rev.2 で追加〉 | **リテンションの「既定 60 日」が削除可能なオブジェクトか、システム設定側の値か** | 検証環境（W1 最優先） | **システム設定側なら、90 日ルールを入れても 60 日で消える。§7.2 が崩れる** | **SV-43** |
| **前提 9** 〈rev.3 で追加〉 | ⭐⭐ **「スナップショットは Cumulocity イベント添付のみ」（構成図の初期版設計）を維持できるか** — 画像が個人情報を含む場合、**参照は監査ログに残らず（§6.8 制約 5）、権限分離もできない（制約 6）**。**監査証跡を要件とするなら、画像の置き場所を決め直す必要がある** | **設計判断**（実機不要）。案件・法務との合意 | **§6.8 の初期版設計・§11.3 のロール割当・§13 の投入対象が変わる。** 構成図側の修正も要る | **CU-18** |
| **前提 1** | **外部Gateway の配置方式（案 α / β）** | 設計判断 | §1.2 の階層図・§6.2 のデータ経路・§10.2 の購読設計 → **付録 A** | [担当範囲] W0-10 |
| **前提 5** | **型規約・external ID 規約の関係チームとの合意** | 合意プロセス | §7.2・§8・§9・§10.2 が全部やり直し | [担当範囲] C-1 / C-2 |

### 15.2 値が未確定のもの（着手前に埋めるべき具体値）

| # | 項目 | 該当章 | なぜ最初か | 決定主体 | 既存 ID |
|---|---|---|---|---|---|
| **CU-1** | **拠点コードの形式**（案 (a) 英数字統一 / 案 (b) 2 系統） | §1.4 / §2.5 | `x_Site.siteId`・サブスクリプション名・topic id・device external ID の全てに効く。**後から変えると全拠点の再登録**。⭐ **`subscription` の `^[a-zA-Z0-9]+$` は [確] なので、案 (a) の根拠は確定済み。前倒しで決められます** | 基盤 × 案件 | — |
| **CU-2** | **external ID の仮名化要否** | §2.2 | **クラウドへ出る特徴量に紐づく個人情報上の論点**。仮名化するなら採番形式そのものが変わる | **法務 / 顧客合意** | [担当範囲] C-2 |
| **CU-3** | **デバイス撤去時の物理削除 / 論理削除** | §3.9 | 物理削除すると過去イベントの `source` が欠損し、オフロード済みデータと突合できなくなる | 基盤 | — |
| **CU-4** | **`x_BoxParseError` と `x_Alarm_<種別>` のクリア条件** | §9.6 | **クリア条件のないアラームは永久に消えない**（§7.3） | 基盤 × 案件 | — |
| **CU-5** 〈rev.3 で入力追加〉 | **リテンション `AUDIT` の日数** | §7.2 / §13.3.6 / **§6.8** | **既定ルールを消すと再作成すべき値が必要**。決まらないとリテンション投入自体ができない。⭐ **rev.3 で入力が 1 つ増えました**: 設定変更の追跡・障害調査要件に加え、**個人情報アクセス記録の保存義務期間**から逆算する（CU-18 が「監査レコードを書く」結論なら、その記録も `AUDIT` に入る）。**3 つ目が支配的になる可能性が高い** | 顧客契約 / 監査要件 / **法務**〈rev.3 で追加〉 | [設定書] K-c / G-13 |
| **CU-6** | **`x_Detection_<種別>` / `x_Alarm_<種別>` の種別列挙** | §6.3.3 / §9.2 | 型が決まらないと `alarm.type.mapping`・ルール・購読フィルタが投入できない | **案件側** | [担当範囲] C-1 |
| **CU-7** | **`responseInterval` の実値** | §5.5 | 過大＝検知漏れ、過小＝アラーム氾濫 | 基盤（実測） × 案件（許容誤検知率） | **SV-26** |
| **CU-8** | **CORS 許可オリジンの完全なリスト** | §11.8 / §13.3.1 | **紐づけ確認アプリを含む**。不足するとアプリが「認証は通るが API が呼べない」 | VM2 の各アプリ担当 | [設定書] T-02 |
| **CU-9** | **Keycloak の realm / clientId / issuer / JWKS URL / グループクレーム名** | §11.6 / §13.3.11 | **1 文字違いで全ユーザーがログイン不可** | Keycloak 担当 | [設定書] A-01/A-02 |
| **CU-10** | **容量サイジングの入力値**（拠点数・台数・検知レート・スナップショットサイズ） | §7.6 | **`storageClassName` は install 後変更不可** | 案件 × 基盤 | **TE-22** |
| **CU-11** | **死活計測の送信量削減方針**（間引き / 集約 / transient） | §5.9 | Edge のスループット上限に直結 | 基盤（実測後） | **TE-15 / V14** |
| **CU-12** | **モデルの保持世代数と旧版削除運用** | §4.4 | ソフトウェアリポジトリ（MongoDB）が肥大化する | 基盤 | **TE-20** |

### 15.3 実装方式が変わるもの

| # | 項目 | 確認方法 | 優先度 | 既存 ID |
|---|---|---|---|---|
| **CU-18** 〈rev.3 で追加〉 | ⭐⭐ **スナップショット画像の置き場所と、個人情報アクセスの監査方式**（下の 5 案） | ⚠️ **「実機不要」ではありません**〈rev.3 レビューで修正〉。**(A)/(A′)/(C) は設計判断のみ。(D)/(B) は CT-5c / CT-5d（Edge 導入後）の結果が要る** → §14.12 の 2b | **最高**。決まらないと §11.4 の R-14・§11.1 の S-16・§13.3.9 の M-04・§13.3.6 の X-04・CU-5 の日数が確定しない（§13.4） | — |
| **CU-19** 〈rev.3〉 | マイクロサービスの `requiredRoles` に **`ROLE_AUDIT_ADMIN`** を指定してサービスユーザーに付与できるか | manifest に記載してデプロイ → `GET /user/currentUser` の effectiveRoles を確認 | **高**（CU-18 でどの案を採っても前提になる） | — |
| **CU-20** 〈rev.3〉 | `POST /audit/auditRecords` で **`user` フィールドを任意の値に設定できるか**（サービスユーザー名で上書きされないか） | サービスユーザーで実操作者名を入れて POST → GET で確認。⚠️ OpenAPI 上 `user` は `readOnly` ではない（`application` は `readOnly`）ので**設定できるはず** [推] | **高**。**できないと「誰が見たか」が残せず、証跡の意味がなくなる** | — |
| **CU-21** 〈rev.3〉 | 標準 Device Management UI が**イベント添付画像をインラインでプレビューするか、ダウンロードリンクのみか** | Edge 導入後に実機で確認（CT-5b と同時） | 中。**「見た」の定義（表示 / DL）と、UI 側の実装点が変わる** | — |
| ~~**CU-22**~~ 〈rev.3 で起票 → **同 rev.3 のレビューで解消**〉 | ~~`AUDIT` リテンションの絞り込み~~ | **実機確認不要でした。** `retentionRule` スキーマに明記: `fragmentType` は *"Used by EVENT, MEASUREMENT, OPERATION and BULK_OPERATION"* ＝ **`AUDIT` 不可**／**`type` は *"Used by ALARM, AUDIT, EVENT and MEASUREMENT"* ＝ `AUDIT` 可**／`source` は全 dataType 可 | ⭐ **「画像アクセス記録だけ長期保持」は `type` で実現できます**（§7.2 / §13.3.6）。**代わりに `type` の選び方が設計判断になりました**（下の CU-28） | — |
| **CU-23** 〈rev.3〉 | ⭐ **`/inventory/binaries/{id}` で拠点ユーザーが 403 になるか**（案 (D) の成立条件） | 検証環境（**CT-5c**）。R-04b の資格情報で `GET /inventory/binaries/{id}` を直叩き | **最高**（CU-18 で (D) を検討する場合）。**403 にならないなら (D) は成立せず、(A) / (A′) / (B) に絞られる** | — |
| **CU-24** 〈rev.3・レビューで修正〉 | **`GET /inventory/binaries`（メタデータ検索）で何が見えるか** — **OpenAPI に Required roles の記載が無い操作**。⚠️ **rev.3 初出時は「この操作だけ」と書いたが誤り。252 operation 中 18 件が同様に欠落しており、`GET /inventory/managedObjects` も含まれる** | 検証環境（CT-5c(b) と同時）。R-04b の資格情報で一覧を引き、**401/403 が返るか、少なくとも当該バイナリが列挙されないこと**を確認 | **高**。⚠️ **本体が 403 でも「いつ・どのカメラで何件撮られたか」がメタデータから漏れる可能性**がある。**列挙できるなら (D) は「本体は 403 だがメタデータは全開」になる**ので CU-23 と同格の成立条件 | — |
| **CU-26** 〈rev.3 レビュー〉 | ⭐ **`GET /audit/auditRecords`（コレクション）の必要ロール** — 同じく **OpenAPI に Required roles の記載が無い**（`GET /audit` と `/audit/auditRecords/{id}` には明記あり） | 検証環境（**CT-5c(d)**）。拠点 Reader の資格情報で 403 になることを確認 | **高**。**監査ログ全件 ＝「誰がいつ誰の顔を見たか」の集合体**。ここが開いていると (D) の効果より重い | — |
| **CU-27** 〈rev.3 レビュー〉 | ⭐ **R-01「Tenant Manager 相当」の実体**（`ROLE_AUDIT_ADMIN` / `ROLE_INVENTORY_READ` / `ROLE_BINARY_READ` を含むか） | `GET /user/roles` で展開（CU-19 と同時） | **最高**。⚠️ **含まれていれば AS-13 は最初から不合格**で、**CU-18 でどの案を採っても R-01 は全拠点の画像を無監査で取得できます**（§11.2） | — |
| **CU-28** 〈rev.3 レビュー〉 | **監査レコードに使う `auditRecord.type` の値** | 設計判断 | **高**。⚠️ **`Event` を選ぶと、CU-22 で判明した `type` によるリテンション分割ができず、Audit logs 画面でも既存記録に埋もれます。** 20 値の enum のうち**本構成で発生しない値**を選ぶこと | — |
| **CU-29** 〈rev.3 レビュー〉 | ⚠️⚠️ **S-15（Edge ホスト OS / k3s 管理者）の統制** | 設計判断 + 運用設計 | **高**。**CU-18 でどの案を採っても塞がらない経路**。保持者の限定・分掌・踏み台経由のセッション記録を決めるか、「技術的に監査不能」を法務に明示するか（§11.1） | — |
| **CU-30** 〈rev.3 レビュー → **rev.4 で推奨案を記載**〉 | ⚠️ **Edge バックアップの暗号化・保管・保持世代・破棄** — **§12.6.6 に [判] 推奨案あり**（保管先での at-rest 暗号化・鍵は admin エスクローと同体系・世代ローテーション破棄・台帳） | **残るのは法務確認のみ**（§12.6.6 の確認リスト ①〜④ — 最悪残存期間 35 日・リストア時の一時復活・削除請求に応じられないこと） | **高**。**バックアップには画像も監査レコードも入ります**（§12.6）。削除バッチで消してもバックアップに残り、リストアで監査レコードが巻き戻ります | — |
| **CU-31** 〈rev.3 レビュー〉 | **`AUDIT` リテンションの削除判定が `time` と `creationTime` のどちらか** | 検証環境（CT-22 と同時）。過去日付の `time` を持つ監査レコードを投入して観測 | 中。⚠️ **`time` 基準なら、過去日付を書くだけで次回リテンションで消せます**（`time` は writable）＝「`PUT`/`DELETE` が無いから改竄不能」の唯一の抜け穴。`time` 基準なら §11.11 MS-g に「`time` は `creationTime` ± N 分以内」の検証を足す | — |
| **CU-33** 〈rev.3 → **rev.4 で推奨設計を記載**〉 | ⭐⭐ **バックアップの取得方式・頻度・保持世代・保管先** — **§12.6.2b に [判] 推奨値あり**（①VM 原子スナップショット→エクスポート→即削除・②日次＋イベント駆動 3 点・③日次 7＋週次 4＋upgrade 前 1・④別障害ドメインに 2TB） | **残るのは ①P-0-8（仮想化基盤）の確認 ②CT-36 の実証 ③保管先の現物調達と取得ジョブの実装主体の決定**（§15.4） | **最高**。⚠️ **公式は取得手順を提供していません**（*"implement appropriate data protection measures"* のみ）。**稼働中のファイルコピーは一貫性が無く復元できません** | **SV-29 / TE-10** |
| **CU-34** 〈rev.3 → **rev.4 で見通し改善**〉 | ⭐ **リストア後にデバイスの再エンロールが要るか**（§12.6.4 RS-1）。**「再エンロール不要で再接続できる」見込みが立ちました [推]** — CA の実体はバックアップに含まれ（テナントオプション + trusted certificate [確]）、照合はチェーン検証、**auto-registration が既定 ON** なのでデバイスユーザーは初回 MQTT コールで再作成される [確]。前提: CA 作成が T-Δ より前（→ BK-l）・auto-registration を OFF にしていない | 検証環境（**CT-35(c)**）。T-Δ 以降に登録したデバイスが復元後に接続できるかを実測（**thin-edge のエンティティ再作成挙動も同時に観測**） | **高**。⚠️ **[推] が覆って再エンロールが要るなら「拠点に人を出す」ことになり、RTO が数時間から数日に変わります**（12.6.5 ⑥）。前提 3（SV-08 / EST）の結論に依存 | — |
| **CU-32** 〈rev.3 レビュー〉 | ⚠️ **映像クリップ（D14 / RU-6 / NS-2）のアクセス監査** | 設計判断 | **高**。**アラーム昇格で同じ人物の「動画」が自動的にオブジェクトストレージへ出ています**（§7.5）。CU-18 は静止画に閉じており、**(A)/(A′)/(D) はいずれも「Cumulocity の外へ監査要件を移す」だけ**。移送先の監査を CU-18 と同格で決めること | — |
| **CU-25** 〈rev.3〉 | `POST /audit/auditRecords` の **`source.id` に実在しない MO ID を渡せるか**（検証されるか） | 検証環境で存在しない ID を POST → 4xx が返るか | 中。**返る場合、削除済みカメラに対する監査レコードが書けなくなる**（§3.9 の物理削除・CU-3 と関係する） | — |
| **CU-13** | **severity 変更時の 2 層の一意性規則の相互作用** | 検証環境（CT-13） | **高**。型規約の確定前に必要 | — |
| — | **運用知識基盤の「設定の更新」対象に画像解析装置・カメラを含めるか** | 設計判断（実機不要） | **最高**。含めないなら K-03〜K-05 / R-06 書込側が丸ごと不要になりリスクが激減 | **SV-17** |
| — | **カメラ（child device）への構成オペレーションが成立するか** | 検証環境 | 中。⚠️ **§4.3（child に何も申告しない）と両立しない**（§4.5） | **SV-28** |
| — | イベント添付バイナリがリテンションの対象か → **及ぶ [確K]。残るのは CT-22③ での実証のみ**〈rev.3 で表記統一 — §7.4 の「解消」と本欄の「未確認」が分裂していた〉 | 検証環境（CT-22③） | 中 | **SV-06** |
| — | 一括登録 CSV で `ENROLLMENT_OTP` と `PATH` を併用できるか | 検証環境 | 高 | **SV-07** |
| — | `PUT /tenant/options/{category}` のマージ／置換セマンティクス | 検証環境 | **高**。置換なら再実行で設定が壊れる | **SV-30** |
| — | スマートルールの managed object の `type` / `fragmentType` | 検証環境で GUI 作成 → 観測 | **高**。実値が分からないと投入スクリプトが書けない | **SV-32 / U-01** |
| — | Notification 2.0 のサブスクライバ数・バックログ量を監視できるか | Prometheus エンドポイント | 高。**取れない場合は棚卸し運用が唯一の歯止め** | **SV-33** |
| — | 紐づけ確認アプリを EXTERNAL 登録にするか Cumulocity にホストするか | 設計判断 | 高 | **SV-24** |
| — | Cockpit の Import/export が Edge で使えるか | 検証環境 | 中 | **SV-11** |
| — | `files/max.size` の実値 | `GET /tenant/system/options` | 中 | **SV-13 / TE-16** |
| — | BOXアダプタのマッピング定義を Cumulocity 経由で配るか Ansible か | 設計判断 | 中 | **SV-27 / TB-2** |
| — | `bulkoperation.creationramp` の意味（既定値か下限か） | ベンダー照会 | 低（G-4 が主統制） | **SV-19** |

#### ⭐ CU-18 の詳細 — 画像の置き場所と監査方式 〈rev.3 で追加〉

**前提となる事実は §6.8 制約 5・6 に記載済みです。** 要点だけ再掲すると、**① 参照・DL は Cumulocity の監査ログに残らない**、**② 添付の取得権限をイベント閲覧権限から分離できない**、の 2 つが同時に効きます。その結果、**UI 側だけの対策は REST 直叩きで迂回され、監査証跡になりません。**

**[要決定] 次の 5 案のどれを採るかを決めてください。本書では判断していません。**

> ⚠️ **(D) の記述量が多いのは、rev.3 で新たに見つかった選択肢で説明が要るからであって、推奨だからではありません** 〈rev.3 レビューで明記〉。**分量を推奨度と読まないでください。** 実際、レビューの結果 **(D) は「リテンションが効かない」「削除バッチが AX-7 に抵触」「AI モデルと同居」「孤児バイナリ」といった固有コストを多く抱えており、(B) には (D) に無い利点（リテンションが効く）があります。**

> ⚠️⚠️ **比較の前提 — 下表の「迂回可能性」は `Cumulocity 内での` 迂回に限った評価です** 〈rev.3 レビューで明記〉。**どの案を採っても塞がらない経路が 3 つあります**:
>
> - **S-15（Edge ホスト root / k3s 管理者）** — MongoDB 直読・Secret 取得・データ持ち出し（§11.1 / CU-29）
> - **Edge バックアップ** — 画像も監査レコードも含む。削除しても残り、リストアで巻き戻る（§12.6 / CU-30）
> - **映像クリップ** — アラーム昇格で同じ人物の動画が自動的に Cumulocity の外へ出ている（D14 / RU-6 / CU-32）
>
> **また、R-01（基盤運用者）については CU-27 の実体確認が済むまで、どの案でも「無監査で全画像取得可能」の可能性が残ります。**

| 案 | 内容 | Cumulocity 内での迂回可能性 | 実装コスト | 主なトレードオフ |
|---|---|---|---|---|
| **(A) Cumulocity に置かない** | 原本・サムネイルともオブジェクトストレージへ。Cumulocity のイベントには**参照キーのみ**（例 `x_SnapshotRef`）。閲覧は自作アプリ経由で、そこでアクセス記録を残す | **なし**（Cumulocity 側に画像が存在しない） | 中 | **§6.8 の「将来」が初期版に前倒しになる。構成図側の修正が要る。**<br>⚠️ **監査要件が移送先に丸ごと移りますが、本書に移送先の監査設計はありません**（CU-32）。⚠️ **移送先は §7.5 のオフロード先・クリップ保存先と同じバケットになる公算が高く、R-11 の資格情報で画像も読める設計になりやすい。**<br>⚠️ **「個人情報が存在しない」は言い過ぎです** — カメラ名・設置場所・検知時刻・検知種別は Cumulocity に残ります |
| **(A′) マスク済みサムネイルのみ Cumulocity** | (A) に加え、**顔をマスクした**サムネイルだけをイベント添付として残す | **なし**（マスクが正しく効いている限り） | 中 | **Cumulocity 上で一次確認できる運用性を残せる。**<br>⚠️ **マスク処理の責務が [担当範囲] 外＝品質保証の主体が居ません。顔検出漏れ＝素の個人情報がそのまま入ります。**<br>⚠️ **「個人情報でなくなる」は法務判断であって設計判断ではありません**（服装・体型・場所・時刻 + カメラ ID で特定可能な場合がある）。<br>⚠️ 制約 2 が残るので**原本ハッシュを併置できません** |
| ⭐ **(D) `/inventory/binaries` に格納**〈rev.3 で追加〉 | イベント添付をやめ、**ファイルリポジトリ**（`POST /inventory/binaries`）に格納。イベントには参照 ID のみ（`x_SnapshotRef`）。閲覧は `ROLE_BINARY_READ` を持つ R-14 経由で、そこで `POST /audit/auditRecords` | **直叩きは 403 にできる**（下の「(D) の根拠」）。⚠️⚠️ **ただし正規経路（R-14）に拠点分離を自前で実装しないと全開になります**（§11.11 / CT-5d） | 中 | ⚠️⚠️ **リテンションが効きません** [確]。**削除バッチ（X-04）が保存期間を守る唯一の手段**になり、その監視まで要る（§7.4 / §12.7 ⑪）。⚠️ **削除バッチは「拠点横断の書込権限を常時保持するアカウント」になり §11.7 AX-7 に抵触**（回避策: アップロード主体と同じサービスユーザーで削除する＝ `owner` 条項。ただし**その主体は常に全画像を読める**）<br>⚠️ **バイナリ 1 件＝ MO 1 件**で §7.7 SV-14b に直撃。**MO 作成/削除の監査レコードも増える**（CU-10 の入力）<br>⚠️ **AI モデルと同じリポジトリに同居**し、`ROLE_INVENTORY_ADMIN` を持つ R-10 / R-06b が読み・書き換え・削除できる（§11.4）<br>⚠️ **イベントが 90 日で消えるとバイナリが孤児になる**（§11.11 MS-c） |
| **(B) 暗号化して添付＋復号マイクロサービスで監査** | 画像を暗号化して `POST /event/events/{id}/binaries`。**鍵は Edge 上のマイクロサービスのみが保持**し、復号時に `POST /audit/auditRecords` | ⚠️ **暗号文の取得自体は無監査で誰でもできます**（`Events READ`）。平文化は防げる | **大** | ⭐ **利点**〈rev.3 レビューで追記〉: **イベント添付なのでリテンションが効きます**（§7.4）。**(D) の最大の欠点（削除バッチが唯一の保存期間統制）が (B) にはありません**<br>⚠️ **標準 UI では画像が表示できない**（専用ウィジェット必須）。⚠️ **harvest-now-decrypt-later** — 鍵が将来漏れると過去分が全滅。⚠️ 鍵は k8s Secret ＝ **Edge root で読める**。⚠️ 鍵ローテーションで旧画像が復号不能。⚠️ 制約 2 が残るのでサムネイル不可。⚠️ **復号サービスも (D) と同じ confused deputy**（§11.11） |
| **(C) 監査を諦め、アクセスできる人を極小化** | §11.3 の `Events READ` 保有者を最小人数に絞り、「画像を見られる人 ＝ 事前審査済みの少人数」という**別の統制**に置き換える | — | 小 | **監査要件を満たさない可能性がある。** 採るなら**法務の明示的な承認**が要る。<br>⚠️ **「少人数」の分母を数え違えないこと** — `Events READ` を減らしても **R-01 / R-06a / R-06b / R-08 / R-10 / R-11 / break-glass / S-15（Edge root）は全員残ります。**<br>⚠️ 制約 6 により**画像だけを外すことはできません**。R-04b から `Events READ` を外すと、**業務閲覧者は Cockpit で検知イベントを何も見られなくなります**（「運用が変わる」より重い） |

##### ⭐ (D) の根拠 — なぜ `/inventory/binaries` だと権限分離できるのか 〈rev.3 で追加〉

**`/event/events/{id}/binaries` と `/inventory/binaries/{id}` では、必要ロールの構造が違います** [確]（Core OpenAPI）:

| API | Required roles | 拠点オペレーター / 閲覧者（R-02 / R-03 + R-04a / R-04b）はどうなるか |
|---|---|---|
| `GET /event/events/{id}/binaries` | `ROLE_EVENT_READ` **OR** `EVENT_READ` permission **on the source** | ⚠️ **取得できる。** source ＝ 対象カメラは担当拠点グループ配下にあり、Inventory ロールで `EVENT_READ` が届くため（§6.8 制約 6） |
| `GET /inventory/binaries/{id}` | `ROLE_INVENTORY_READ` **OR** `ROLE_BINARY_READ` **OR** **owner of the resource** **OR** `MANAGE_OBJECT_READ` permission **on the resource** | ⭐ **4 つのどれも満たしません**（下記）→ **403 が期待できる** |

- **`ROLE_INVENTORY_READ` / `ROLE_BINARY_READ`（グローバル）**: §11.2 rev.2 の原則により、**顧客ユーザーには付与しません**（付与してよいのは Own user management READ とアプリアクセスのみ）
- **owner of the resource**: アップロードするのは画像解析パイプライン側のサービスユーザー／デバイスであり、**閲覧者は owner になりません**
- **`MANAGE_OBJECT_READ` on the resource**: 判定対象が**イベントの source（カメラ）ではなくバイナリ MO 自身**に変わります。**バイナリ MO を拠点グループ配下に置かなければ、Inventory ロールは到達しません**

> ⭐ **これが (D) の核心です。** 制約 6（権限分離不能）は **`/event/events/{id}/binaries` に固有の性質**であって、Cumulocity 全体の性質ではありません。**格納先を変えるだけで、独自暗号なしに「イベントは見えるが画像は 403」を作れます。**
>
> ⚠️ **ただし「403 になること」は仕様からの推論です。実機で確認してください** → **CT-5c** / **CU-23**。

> ⚠️⚠️ **(D) の 403 は「置き方」に依存するので、CI で守り続ける必要があります** 〈rev.3 で追加〉。
>
> 上の 4 つの選択肢は、**運用の途中で 1 つでも満たされると静かに穴が開きます**:
>
> - 誰かがバイナリ MO を **`childAssets` で拠点グループにぶら下げた** → `MANAGE_OBJECT_READ` が届いて 200 になる
> - 誰かが R-02 / R-03 に **グローバルの `ROLE_INVENTORY_READ` / `ROLE_BINARY_READ` を足した** → 全バイナリが見える
>
> **これは §11.2 で潰したはずの「グローバルにデータ系ロールを持つと Inventory ロールの絞り込みが効かなくなる」問題と同型です。** 初回の設定が正しくても、**次に権限を足した人が壊します。**
>
> **→ §2.8 CK-8 / §13.7 AS-14 として機械検査を起票しました**（(D) 採用時のみ有効）。**AS-3 / AS-4 / AS-5 / AS-13 と同じく「CI が唯一の歯止め」の種類です。**
>
> ⚠️ **検査対象は拠点側ロールではありません**〈rev.3 レビューで修正〉。`POST .../childAssets` は **source と child の両方**に権限を要求するため（§2.8 の注記）、**拠点 Manager は自力でバイナリをぶら下げられません。** 実際に危ないのは **グローバルにデータ系ロールを持つ主体** — **R-01 / R-05 / R-06a / R-06b / R-08 / R-10 / R-11** です（§11.4）。

> ⚠️⚠️ **(D) の 403 は「直叩き」にしか効きません** 〈rev.3 レビューで追加〉。
> 正規の閲覧経路である **R-14（画像配信マイクロサービス）には Cumulocity の拠点分離が一切効きません。** R-14 が自前で認可判定をしなければ、**拠点 A のユーザーが拠点 B の `binaryId` を投げれば 200** です。**§11.11 が (D)/(B) 採用時の必須要件**、**CT-5d がその合格判定**です。

> ⚠️ **`owner` 条項の非対称に注意してください** 〈rev.3 レビューで追加〉。
> `GET` / `PUT` / `DELETE /inventory/binaries/{id}` はいずれも *"… OR **owner of the resource** OR …"* を含みます [確]。**つまり画像をアップロードした主体（画像解析パイプライン側のサービスユーザー）は、全画像を常に読める・書き換えられる・消せます。**
>
> - **利点**: 削除バッチ（X-04）を**この主体で動かせば、グローバル書込ロール無しで削除できます** → §11.7 **AX-7**（拠点横断の書込権限を常時保持するアカウントを作らない）への抵触を回避できます
> - **代償**: **その主体は常に全画像を読めます** → **AS-14(b) の検査対象に含めてください**

> ⚠️⚠️ **(D) を採っても制約 5（参照・DL は監査ログに残らない）は解消しません。** **格納先を変えても、監査レコードを自前で書く必要性は消えません。** (D) が解消するのは制約 6 だけです。

**(D) の副次的な効果**:

| 効果 | 内容 |
|---|---|
| ⭐ **制約 2 が外れる** | **「1 イベント 1 バイナリ」（2 件目は 409）はイベント添付の制約**です。ファイルリポジトリなら**原本とサムネイルの 2 本を同じイベントに紐づけられます**（イベント側に `x_SnapshotRef: { fullId, thumbId }` を持たせる） |
| ⭐ **網断耐性が 1 段上がる** | イベント ID に依存しないため、**「先にバイナリを上げて ID を得る → イベントを MQTT で送る」**という順序が取れます（§6.8 の SV-42 の設計が単純化されます） |
| **最小権限にできる** | 画像配信マイクロサービスには **`ROLE_BINARY_READ` だけ**を与えられます（`ROLE_INVENTORY_READ` は不要。専用ロール族が存在する） |

**(D) の追加コスト**:

| コスト | 内容 |
|---|---|
| ⚠️⚠️ **削除運用が必須** | リテンションが効かないため、**放置すると無期限に残ります**。⚠️ **これは §4.4 論点 2（AI モデルの世代管理・TE-20）とまったく同じ問題ですが、件数が桁違いです**（検知件数 × 2）。**個人情報の保存期間を守るには削除バッチが唯一の手段**であり、**バッチが止まったことを検知する監視まで含めて設計が要ります** |
| ⚠️ **MO 総数の増加** | バイナリ 1 件 ＝ managed object 1 件。§7.7 のスケール制約（**SV-14b**: MO 総数・課金・クエリ性能）に直撃します。**CU-10 の容量サイジングに「検知レート × 2」の MO 増を入力として追加してください** |
| **ファイルリポジトリが AI モデルと混在** | Administration の Files repository に AI モデルと画像が並びます。**`type` で分離する規約**が要ります（`ROLE_INVENTORY_READ` を持つ R-01 基盤運用者には一覧で見えます） |
| **サイズ上限が別系統** | イベント添付の既定 50MiB とは別に、`files/max.size` が効きます（実値は **SV-13 / TE-16** で未確認） |

> ⚠️ **(A) / (A′) / (D) / (B) のいずれも、`POST /audit/auditRecords` を書くサービスユーザー（**R-14** / **S-16**）が必要です。** 枠は §11.1 / §11.4 に予約済みで、**中身は CU-18 決定後に確定**します（§15.5）。
>
> ⚠️⚠️ **そしていずれの案でも、その配信主体は特権を持ち、拠点分離を自前で再現する必要があります** → **§11.11 が必須要件です。** ここを落とすと、どの案も「直叩きは塞いだが正規経路は全開」になります。

##### `POST /audit/auditRecords` で何が書けるか — 「任意のログ API」ではありません 〈rev.3 で追加〉

**⚠️ 汎用ログ API と誤解しないでください。「半分だけ任意」です** [確]（Core OpenAPI `auditRecord` スキーマ）。

| | フィールド | 可否 |
|---|---|---|
| **自由に書ける** | `activity`（実施した操作の要約）・`text`（詳細）・`user`（実施者）・`time` | ✅ いずれも自由文字列 |
| **自由に書ける** | **任意の追加フラグメント** | ✅ *"It is possible to add an arbitrary number of additional properties as a list of key-value pairs"*。**独自フラグメント（例 `x_ImageAccess`）で構造化データを載せられます** |
| ⚠️ **書けない** | **`type`** | ❌ **20 値の固定 enum**。**独自 type は作れません** → `Event` 等を流用するしかない |
| ⚠️ **必須** | `activity` / `source` / `text` / `time` / `type` の **5 つが required** | ⚠️ **`source.id` が必須**。*"The platform component ID to which the audit is associated"* ＝ **紐づけ先の ID が無いログは書けません**（→ **CU-25**） |
| ⚠️ **書けない** | `application` / `severity` / `changes` / `creationTime` / `id` / `self` | ❌ すべて `readOnly`。**`severity` を自分で指定することはできません** |

⭐ **証跡として強い性質** [確]: **Audit API には `PUT` も `DELETE` も存在しません。** OpenAPI に定義されているのは `GET /audit` / **`POST` + `GET` /audit/auditRecords** / `GET /audit/auditRecords/{id}` の 4 つだけです（spec 全 149 パスを走査して確認）。**個別レコードの改竄も削除も REST API からはできません**（docs の *"persistently stored and cannot be externally modified after being written"* と整合）。

> ⚠️⚠️ **ただし「改竄不能」と言い切れるのは REST API 経由に限ります** 〈rev.3 レビューで訂正〉。**rev.3 初出時は「消える経路はリテンションルールだけ」と書き、それを安全性の根拠として扱っていましたが、逆です。** 次の 3 経路が残ります:
>
> | # | 経路 | 内容 |
> |---|---|---|
> | **1** | ⚠️⚠️ **リテンションルールによる狙い撃ち消去** | **R-01 は Retention rules ADMIN を持ちます。** `dataType=AUDIT` + `type=<画像アクセス用>` + `maximumAge=1` を 1 本入れれば証跡だけ消せます（`source` で特定カメラ分だけも可能）。→ **対策: `management` テナントから `editable: false` で作成 + AS-11b**（§7.2 / §13.3.6） |
> | **2** | ⚠️ **`time` を過去日付で書く** | `time` は writable。**リテンションが `time` 基準なら、書いた瞬間から「古いレコード」になり次回削除で消えます**（判定基準は未確認・**CU-31**）。→ **対策: §11.11 MS-g** |
> | **3** | ⚠️ **バックアップからのリストア** | 監査レコードが巻き戻ります（§12.6 / CU-30） |
>
> **また、S-15（Edge root）は MongoDB を直接操作できます**（§11.1 / CU-29）。

> ⚠️ **「任意のアプリケーションログの置き場」として使わないでください。** 理由:
>
> - **`AUDIT` リテンションが全 type に一律にかかります**（種別ごとに変えられるかは **CU-22** で未確認）。ログを混ぜると、**監査証跡の保持期間がログの都合に引きずられます**
> - **Administration の Audit logs 画面のフィルタは type / 日時 / user の 3 つだけ**です。独自 type が作れない以上、**自作レコードは既存の監査記録に埋もれます**
> - **書き込みに `ROLE_AUDIT_ADMIN` が要ります**（§11.2）。アプリが自由に叩ける API ではなく、**この 1 本のためにサービスユーザーを立てる**設計になります
> - Cumulocity 自身の位置づけは *"Whenever a **security-relevant** event occurs, it must be logged for potential auditing"* です。**「セキュリティ上意味のある事象」に限って使ってください。**

> ⚠️ **監査レコードの中身にも規約が要ります**（CU-18 決定後に §6.8 か §11 に規約として起こすこと）。現時点で確定している制約:
>
> - **`auditRecord.type` は固定 enum なので独自 type は作れない** [確]。⚠️ **ただし `Event` を流用してはいけません**〈rev.3 レビューで訂正〉。**`type` は `AUDIT` リテンションの絞り込みキーであり**（§7.2）、**Cumulocity 自身が使う値を選ぶと ①画像アクセス記録だけ長期保持する分割ができない ②Audit logs 画面の type フィルタで既存記録に埋もれる ③機械抽出できない**。**20 値のうち本構成で発生しない値を選んでください** → **CU-28**
> - `source` は **対象カメラ**（画像解析装置ではない）。§6.2 の `source` 規約に合わせる。⚠️ **(D) を採る場合も `source` はカメラのままにしてください**（バイナリ MO にすると、削除バッチでバイナリを消したときに `source` が迷子になります）。⚠️ **ただし `source` にカメラを使うと `AUDIT_READ` permission on the source が効きます** — Inventory ロールに `scope: AUDIT` を入れないこと（§11.2 / AS-13(b)）
> - ⭐ **`x_ImageAccess` に画像の SHA-256・`binaryId`・`length` を入れてください**〈rev.3 レビューで追加〉。`PUT /inventory/binaries/{id}` / `PUT /event/events/{id}/binaries` が存在するため、**「誰がいつ見たか」を残しても「その時点で何が写っていたか」は残りません。** ハッシュは個人情報ではないので下の禁則に抵触しません
> - ⭐ **証跡の時刻は `creationTime`（プラットフォーム付与・`readOnly`）を正とし、`time` は事象時刻の参考値**とする（§11.11 MS-g）
> - ⚠️ **監査レコード自体に画像・特徴量・氏名を書かない。** 書くのは ID と操作種別のみ。**監査ログが二次的な個人情報の器になるのを防ぐ**（Cumulocity 自身も *"avoid logging details such as username, IP address or the contents of messages, unless needed for security auditing"* と述べている）。⚠️ **同じ禁則を R-14 のアプリケーションログにも適用してください**（§11.11 MS-f） — 適用しないと、**ingress アクセスログ案を却下した理由③がそのまま自作マイクロサービスで再発します**

> ⚠️ **検討したが推奨しない案: Edge の ingress / Web サーバのアクセスログ収集。** Edge は root アクセスが可能で k3s 上で動くため技術的には可能ですが、**① SSO（Cookie / Bearer）だと操作者名がアクセスログに出ない**、**② Edge 内部の ingress 設定の書き換えはサポート外でアップグレードで失われる**、**③ Cumulocity の監査ログではないため `AUDIT` リテンションと `ROLE_AUDIT_READ` の保護の外に出る**（個人情報アクセス記録を平文テキストで長期保管することになる）。**この 3 点により、本書では選択肢に含めていません。**

### 15.4 運用に効くもの

| # | 項目 | 確認方法 | 優先度 | 既存 ID |
|---|---|---|---|---|
| — | **テナント CA 自動更新（毎年 10/2）時の再エンロール動作** | 検証環境でシミュレート（CT-32） | **最高** | **SV-25 / TE-7** |
| — | **SSO からローカル認証へ戻す手順** | 実機検証 → runbook 化（CT-27） | **最高** | **SV-04** |
| — | **全拠点合算のスケール** | 負荷試験（CT-28） | **最高** | **SV-14** |
| — | **RTO/RPO、BOX・thin-edge のバッファ保持時間** | 実測 + 設計判断（**§12.6.5 に算定式を追加**〈rev.3〉）。⚠️ **RTO の支配項は ③OS 準備 + ④150GB 級の転送 + ⑥RS-1（証明書の再エンロール）。分単位にはなりません** | **最高** | **SV-29 / TE-10 / CU-33 / CU-34** |
| — | ⭐ **バックアップ運用の担当**〈rev.3〉 | 担当決定 | **最高**。⚠️ **公式に取得ツールが無いため、取得ジョブ・保管・暗号化・世代破棄・監視（§12.7 ⑮）・定期リストア試験（BK-f / RB-7）はすべて自前です。** D17 により**これが全損に対する唯一の備え**です | **CU-33 / CU-30** |
| — | **基盤自身の異常のアラート通知先** | 設計 + 担当決定 | **高**。⚠️ **メール不採用のため唯一の検知経路** | **SV-34 / TB-5** |
| — | **トークン発行プロキシ（N-02）の実装主体** | 担当決定 | **高**。決めないと通知経路全体が止まる | **TB-3** |
| — | ⭐ **画像削除バッチ（X-04）の実装主体**〈rev.3〉 | 担当決定 | **高**（CU-18 で案 (D) を採る場合）。⚠️ **ファイルリポジトリにはリテンションが効かないため、これが個人情報の保存期間を守る唯一の手段になります**（§7.4）。**バッチの停止を検知する監視（§12.7 ⑪）まで含めて設計すること。** ⚠️ **§11.7 AX-7（拠点横断の書込権限を常時保持しない）との整合を先に取ること** — `owner` 条項を使う回避策あり（§15.3） | **[要決定]**（TB-4 と同じ書式で起票すること） |
| — | **オフロードバッチ（X-03）の実装主体** | 担当決定 | **高**。⚠️ **稼働開始から 90 日以内が期限** | **TB-4** |
| — | Edge のホスト側から admin パスワードを復旧する手段 | 検証環境 | 高。成立すればエスクローの必要性が下がる | **SV-37** |
| — | 証明書・社内 CA ローテーション手順（新旧並行の移行期間） | 設計 + 検証環境 | 高 | **SV-22** |
| — | メタ監視の向き（D9 との整合） | 設計判断 | 中 | **SV-23** |
| — | 案件アプリのトークン再取得ロジック | 案件側との合意 | 中 | **SV-36** |
| — | 運用知識基盤が必要とする分析データの期間 | 設計判断 | 高。**90 日超ならオフロードが唯一のデータ源** | **SV-20** |
| **CU-35** 〈rev.4〉 | **Edge のリリースサイクル・各版のサポート期限** | ベンダーへ書面照会（SV-50 と同じ経路で） | **高**。UP-a（年 1 回以上の追従）の周期の根拠。**メジャーはスキップ不可** [確] なので、放置コストが段数で積算する（§12.10.2） | — |
| **CU-36** 〈rev.4〉 | **ホスト OS のディストリビューションと版の確定・記録** | 設計判断（K3s のサポート OS から選定）→ 台帳記録 | **高**。**復元先の「同一（または互換）OS」[確] の判定基準**（§12.6.1 Step 2 / §12.10.4 OS-a）。検証環境と本番で揃える | — |
| **CU-37** 〈rev.4〉 | **Edge / K3s / ホスト OS の脆弱性情報の受け口と対応期限** | 設計判断 + ベンダー通知の登録 | **高**。**現状、CVE を誰も受信していません**（thin-edge 側 TE-17 のみ設計済み）。閉域網は自動更新が無く、受け口が無ければ永遠に当たらない（§12.10.4 OS-e） | **TE-17（thin-edge 側）** |
| **CU-38** 〈rev.4〉 | **Keycloak 担当との合意事項一式** — 退場 SLA（AC-b）・可用性目標・Keycloak 自身のバックアップ・バージョンアップ時の互換確認・相互告知（KC-a） | Keycloak 担当との合意（文書） | **高**。**Keycloak は Edge のバックアップに一切含まれず**、停止すると SSO 全断（break-glass のみ残る）。この影響評価自体が rev.4 まで欠落 | **CU-9（受領値）** |
| **CU-39** 〈rev.4〉 | **Edge プラットフォームログの恒常収集の範囲**（収集しない / operator・ingress のエラーのみ 等） | 設計判断 | 中。**「収集しない」も明示的な設計判断として記録する**（§12.10.10 LG-a/LG-b — 個人情報混入と保護外の二重化リスクがあるため既定は「収集しない」） | — |
| **CU-40** 〈rev.4〉 | **ホスト再起動後に Edge が無操作で完全復帰するか**（K3s の自動起動 → 全 Pod Running → 両テナント応答） | 検証環境（W1。ST-c の合否判定） | **高**。復帰しないなら、**停電・OS パッチのたびに手作業**が要り、計画停止 runbook（§12.10.3）の書き方が変わる | — |
| **CU-41** 〈rev.4〉 | **MongoDB 容量増設の実手順**（`c8yedge config` での増加コマンド・再起動要否・所要時間） | 検証環境で実測 | 中。§12.7 ⑤ の警告（70%）から増設完了までのリードタイムが決まらないと、しきい値の妥当性も決まらない（§12.10.5） | — |
| **CU-42** 〈rev.4〉 | **バックアップ経過時間（§12.10.5 ⑮）を Otel Collector パイプラインへ取り込む実装方式** | 設計判断 + 検証環境で実装 | 中。バックアップジョブはホスト OS 上の自前ジョブで Edge operator の `:3443/metrics` の管轄外。[判] 推奨は node_exporter textfile collector（§12.10.5） | — |

### 15.5 文書管理上の未処理事項

| # | 項目 | 内容 |
|---|---|---|
| **DOC-6** 〈rev.4 で CU-14 から改称〉 | ⚠️ **[設定書] §4 の移管作業** | 本書 §0.3 の移管表に従い、[設定書] §4 を削除して本書への参照に置き換える。**それまでは記述が二重に存在する**（齟齬があった場合は本書が優先） |
| **DOC-7** 〈rev.4 で CU-15 から改称〉 | **[連携仕様] への反映** | 本書で新規定義した `x_Site` / `x_SensingBox` を [連携仕様] §4.3 に反映する |
| **DOC-8** 〈rev.4 で CU-16 から改称〉 | **[担当範囲] との相互参照の整理** | [担当範囲] §4（Cumulocity Edge 担当作業）が本書を参照する形に更新する |
| **CU-17** | **項目 ID 体系の統一** | 本書で追加した ID（R-10〜R-16 / G-04 / D-10〜D-12 / M-03〜M-04 の一部）を [設定書] §3 の体系に正式に吸収する。**放置すると [設定書] 側が別の内容で同じ ID を採番した場合に衝突する**。✅ 〈rev.4 で解消〉**本書内の `CU-14/15/16` 二重採番**（§15.0 の技術的な CU-14/15/16 と、本節にあった文書管理系の CU-14/15/16 が別内容で衝突していた問題）**は、本節側を `DOC-6/7/8` へ改称して解消しました。** ⚠️ **`CK-7` は参照だけで表に行がありません**（§2.8。未解消のまま残っています）。**次回改版で解消してください** |
| **DOC-1** 〈rev.3〉 | ⭐ **CU-18 決定後にやること**（現在は保留中） | 決定したら**以下をまとめて起票・確定**してください。**散文で書くと迷子になるので、この行を消化リストとして使ってください**:<br>□ **§11.1 S-16**（画像配信 / 監査書込マイクロサービス）の内容確定<br>□ **§11.4 R-14** の最小ロール確定（(D) なら `ROLE_BINARY_READ` + `ROLE_AUDIT_ADMIN`）<br>□ **§11.11** の認可仕様を実装要件として [担当範囲] に渡す<br>□ **§13.3.9 M-04**（ファイルリポジトリの `type` 名前空間規約）<br>□ **§13.3.6 X-04**（画像削除バッチ）と **§15.4 の実装主体**（TB-4 と同じ書式で）<br>□ **§13.2 トレース表**に「§6.8 スナップショット格納」の行を追加<br>□ **§13.3.4** に R-14 の行、**§13.6 `users.json`** に R-14<br>□ **§13.4** に R-14 デプロイ工程を追加<br>□ **CU-28**（監査レコードの `type` 値）の確定<br>□ **CU-5** の `AUDIT` 日数を法務と確定 → §13.3.6 ②<br>□ ⭐ **構成図 `cumulocity-iot-architecture.drawio` の注記修正**（DOC-2） |
| **DOC-2** 〈rev.3〉 | ⭐ **構成図との齟齬** | 構成図の注記が **「スナップショット: 初期版は Cumulocity イベント添付のみ…将来はオブジェクトストレージ本体+サムネイル添付へ移行(設計文書§4.6)」** のままです。**前提 9 / CU-18 は (A)/(A′)/(D) のいずれでもこの注記と正面衝突します。** 本文は 2 箇所で「構成図側の修正も要る」と書いていますが**作業として起票されていませんでした**。⚠️ 参照先「設計文書§4.6」も §0.3 の移管でリンク切れになります |
| **DOC-3** 〈rev.3〉 | **[担当範囲] との齟齬** | ① **§6.6 / TE-14 / W0-11**（スナップショット添付ワーカー）が `tedge upload c8y` → **イベント添付**を前提に担当案を組んでいる。**CU-18 で (A)/(A′)/(D) を採るとこの議論の土台ごと消えます**（(D) では `tedge upload c8y` のショートカットが使えない）。⚠️ **W0-11 は CU-18 より前に来るので、順序を入れ替える必要があります**<br>② リテンション日数（`AUDIT`）の決定主体・時期が rev.3 の CU-5 更新（③個人情報アクセス記録の保存義務期間 / 法務）を反映していない |
| **DOC-4** 〈rev.3・rev.4 で④追加〉 | **[設定書] との齟齬** | ① `AUDIT` を「**K-c（設定変更の追跡要件）から逆算**」としており、rev.3 の 3 つ目の入力と食い違う<br>② X-02 が「リテンションが添付に及ぶかは公式に記載なし（SV-06）」のまま（本書 §7.4 は「及ぶ [確K]」）<br>③ 構成図トレースがイベント添付固定<br>④ ⚠️ 〈rev.4〉**[設定書] §4.10 がサービスアカウント削除→再作成の理由を「ユーザー ID が変わるため」としたまま**で、本書 §11.4「[確] ユーザー ID は変わりません…rev.1 は理由が誤りでした」という訂正と正面から矛盾している。**齟齬があった場合は本書が優先**（§0.3）だが、[設定書] 側の対象者リスト（R-05, R-06, R-08, R-09）も R-11/R-12 を含まず古いままなので、移管作業（DOC-6）で一括して解消すること |
| **DOC-5** 〈rev.4〉 | ⭐ **[担当範囲] への運用設計の反映** | rev.4 の §12.10 新設に伴う [担当範囲] 側の作業:<br>□ **§10.3 の runbook 追補 8 件**（§12.10.6 の表。**採番は [担当範囲] 側で** — 本書は意図的に RB-14 以降を振っていません）<br>□ **本書が rev.4 で訂正した参照誤り 2 件の突合**（§11.5 RB-11→RB-9・§12.5 の RB-13 誤参照）<br>□ **運用体制（要員・当番・受付窓口・SLA・エスカレーション）の設計** — 3 文書のどこにも無いことを rev.4 で確認済み（§12.10.1）。SV-34 / TB-5（通報先）とセットで<br>□ **周期作業カタログ（§12.10.1）の担当列を埋める**<br>□ TE-17（脆弱性対応リードタイム）に **Edge / K3s / OS 側（CU-37）**を追加 |

---

## 付録 A. 案 α（単一 thin-edge インスタンス）を採用する場合の差分

> **本文は案 β（拠点ごとに thin-edge インスタンスを分ける）を前提に書かれています。** [担当範囲] W0-10 で案 α（現案・単一 thin-edge が全拠点の BOX を収容）を選ぶ場合、本付録の差分を本文に上書きしてください。

### A.1 構造的な非対称

**案 α では「1 thin-edge インスタンス = 全拠点」となり、D17 の拠点分離（1 拠点 = 1 グループ）と噛み合いません。**

```
資産階層（グループ）              接続階層（thin-edge B が作る）
拠点A (group)                     BOXゲートウェイ (main device・拠点に属せない)
 ├── 画像解析装置 ─┐               ├── BOX_A1  ← 拠点Aの機器
 │    └── カメラ    │              ├── BOX_B1  ← 拠点Bの機器
 └── BOX_A1  ←─────┴── 同一MOを     └── BOX_C1  ← 拠点Cの機器
                     両方から参照
```

### A.2 §1（グループ）への差分

| # | 差分 | 内容 |
|---|---|---|
| **α-1** | ⚠️ **BOX 全台を個別に `childAssets` へ追加する工程が必要** | 案 β は拠点あたり 2 回で済むが、**案 α は BOX の台数分すべて**。`POST /inventory/managedObjects/{siteGroupId}/childAssets`。冪等化は 409 黙殺<br>⭐ **代替**: 一括登録 CSV の **`PATH` 列**でグループ階層を自動生成できる（`ENROLLMENT_OTP` と併用可・§3.3）。**グループ作成・所属付けが 1 回の CSV 投入で済み、再実行も冪等**になるため、案 α ではこちらを検討する価値があります |
| **α-2** | **BOXゲートウェイ本体の配置** | GW はどの拠点にも属せないため、**共通グループ（`common`）に置く**（§1.2） |
| **α-3** | §1.3 の規約「拠点グループへの編成は main device に対してのみ」が**成立しない** | BOX（child）を個別に編成する必要があるため、**§3.10 の検査 4（CK-4）を BOX にも拡張**する |

### A.3 §5・§9（死活・アラーム）への差分

| # | 差分 | 内容 |
|---|---|---|
| **α-4** | ⚠️ **GW 停止時に全拠点の BOX が同時に影響を受ける** | §5.4 の `responseInterval: 32767` は前提として、**`x_BoxSilent` を EPL で拠点単位に集約する**ことを検討（§8.2 RU-5 が**必須になる**） |
| **α-5** | ⚠️ **GW 本体のアラームがどの拠点からも見えない** | ①**Inventory ロール**: 拠点オペレーターは GW 本体を見られない ②**Notification 2.0**: 拠点グループ購読に GW 本体のアラームが乗らない → **案件アプリは GW 障害を知れない**。拠点側からは「その拠点の BOX が全部黙った」ようにしか見えない |
| **α-6** | α-5 への手当（**追加設計が必要**） | **GW 障害を拠点オペレーター・案件アプリに伝える別経路**（EPL によるロールアップアラーム、または共通グループを購読する別サブスクリプション）を設計する |

### A.4 §10（通知）への差分

| # | 差分 | 内容 |
|---|---|---|
| **α-7** | **§1.6 の代替案 (i) が使えない** | 前提 2（`alarmsWithChildren` が `childAssets` を辿る）が NG の場合、案 β なら main device を `source` にする代替が効くが、**案 α では main device が全拠点を代表するため代替にならない**。⚠️ **前提 2 が NG かつ案 α の場合、拠点分離の実現方式そのものを再設計する必要がある** |
| **α-8** | **共通グループ用のサブスクリプションを追加** | α-6 の手当として NS-4（`source` = 共通グループ）を定義する |

### A.5 §2（命名）・§7（規模）への差分

| # | 差分 | 内容 |
|---|---|---|
| **α-9** | **child ID 名前空間の統制** | 全拠点分の BOX 名が **1 インスタンス内で一意**である必要がある。§2.2 の `{siteId}:` プレフィックスで担保されるが、**命名の自由度は失われる** |
| **α-10** | ⚠️ **単一インスタンスのスループット実測が必要** | 全拠点分のイベントが 1 プロセスに集中する。**[要 TE-13]** child device 数の上限はドキュメントに記載なし |
| **α-11** | **ストア&フォワードのサイジング** | Edge 停止時に**全拠点分が 1 つのバッファに溜まる**（案 β なら 1 拠点分に縮む）。[担当範囲] §5.7 のサイジング式の入力が変わる |
| **α-12** | ⚠️ **障害影響範囲** | **全レガシー拠点が同時停止**する SPOF になる。§12.6 の RTO/RPO 設計に反映 |

### A.6 案 α を採る場合の追加検証項目

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-A1** | BOX 全台の拠点グループ編成 | **全 BOX が正しい拠点グループの `childAssets` に入っている**（CK-4 の拡張） |
| **CT-A2** | GW 障害の可視性 | GW を停止 → **拠点オペレーターと案件アプリの両方が「GW 障害である」と分かる** |
| **CT-A3** | アラームストームの抑止 | GW 停止 → **全拠点分の BOX アラームが個別に上がらず、拠点単位に集約される** |
| **CT-A4** | 単一インスタンスのスループット | 全拠点分の想定レートで 1 インスタンスが処理できる |

---

## 16. 出典

### 社内文書

- [IoT連携チーム_担当範囲設計書.md](IoT連携チーム_担当範囲設計書.md) — **本書のベース。担当範囲・thin-edge 側実装・WBS・Ready for Service の正**
- [Cumulocity構成準拠セットアップ設定書.md](Cumulocity構成準拠セットアップ設定書.md) — **適用手順（P0〜P7）の正**。⚠️ §4 は本書へ移管（§0.3 / DOC-6）
- [Cumulocity設定定義書.md](Cumulocity設定定義書.md) — 設定項目の網羅カタログ。⚠️ [設定書] §0.2 の訂正が優先
- [Cumulocity設定エクスポート投入ガイド.md](Cumulocity設定エクスポート投入ガイド.md) — go-c8y-cli の実装技法・冪等化パターン A〜E
- [../../IoTPlatform_cc/data-integration-spec_new.md](../../IoTPlatform_cc/data-integration-spec_new.md) — IF-nn の正
- [../../IoTPlatform_cc/architecture-camera-monitoring.md](../../IoTPlatform_cc/architecture-camera-monitoring.md) — デバイスモデル・ペイロード規約の原典
- [../../IoTPlatform_cc/design-decisions.md](../../IoTPlatform_cc/design-decisions.md) — D1〜D17
- [../../IoTPlatform_cc/vendor-questions-cumulocity.md](../../IoTPlatform_cc/vendor-questions-cumulocity.md) — V1〜V20
- [cumulocity-iot-architecture.drawio](cumulocity-iot-architecture.drawio) — **「全体構成(配置構成・レビュー反映)」タブのみ**
- [Cumulocity利用設計書_ファクトチェック結果.md](Cumulocity利用設計書_ファクトチェック結果.md) — **rev.2 の改訂根拠**。8 名の独立レビュアーによる一次情報照合
- [factcheck/](factcheck/) — 各レビュアーの全文レポート（逐語引用・探索経路つき）。**`B1_externalid.md` は external ID の要否調査**

### Cumulocity Edge

| 内容 | URL |
|---|---|
| インストール・2 テナント・TLS 要件・ライセンス・registry credentials・前提要件 | `https://cumulocity.com/docs/2026/edge/installing-edge/` |
| **機能比較表（`Included` の一覧）・100 tps/CPU コア** | `https://cumulocity.com/docs/2026/edge/edge-introduction/` |
| メールサーバー・`c8yedge config`（**ドメイン / TLS / ライセンスの更新手順**）| `https://cumulocity.com/docs/2026/edge/manage-edge/` |
| **バックアップとリカバリ**（⚠️ **提供されているのは復元手順のみ。取得手順は *"implement appropriate data protection measures"* の 1 行**・§12.6.1）**・Prometheus メトリクスエンドポイント** | `https://cumulocity.com/docs/2026/edge/edge-operations/` |
| **MongoDB — Back Up with Filesystem Snapshots** 〈rev.3〉 — *"the data files reflect a consistent state as of the last checkpoint. Checkpoints occur with every 2 GB of data or every minute."* / データとジャーナルが**別ボリューム**なら `db.fsyncLock()` が必要（§12.6.2） | `https://www.mongodb.com/docs/manual/tutorial/backup-with-filesystem-snapshots/` |
| **K3s — Cluster Datastore** 〈rev.4〉 — *"SQLite is the default datastore"*（シングルノードの既定・§12.6.2） | `https://docs.k3s.io/datastore` |
| **K3s — Backup and Restore（SQLite）** 〈rev.4〉 — *"take a copy of `/var/lib/rancher/k3s/server/db/`"* / **server token（`server/token`）を欠くと復元不能**（*"the snapshot will be unusable"*・§12.6.2） | `https://docs.k3s.io/datastore/backup-restore` |
| **K3s — Stopping K3s** 〈rev.4〉 — `systemctl stop k3s` はスーパーバイザのみ停止・**コンテナ停止には `k3s-killall.sh`**（*"The cluster data will not be deleted."*・§12.6.2b 案 (iii)） | `https://docs.k3s.io/upgrades/killall` |
| **K3s — Local Path Provisioner** 〈rev.4〉 — PV の既定パスは **`/var/lib/rancher/k3s/storage`**（§12.6.2） | `https://docs.k3s.io/add-ons/storage` |
| **実測ベンチマーク（Narrow / Wide シナリオ）** | `https://cumulocity.com/docs/2026/edge/benchmarks/` |
| ブランディング・Web SDK | `https://cumulocity.com/docs/2026/edge/using-edge/` |
| **Edge CR の spec 13 項目**（`storageClassName` は変更不可 / `mongodb…storage` は増加のみ可）| `https://cumulocity.com/docs/2026/edge/edge-custom-resource-definition/` |

### Cumulocity Core API・デバイス管理

| 内容 | URL |
|---|---|
| **Core OpenAPI**（Notification 2.0 の RBAC バイパス・`NotificationSubscription` スキーマ・`bulkNewDeviceRequests`・`creationRamp` / `groupId` / `failedParentId`・ロール名・パスワード制約・**アラーム de-duplication と suppression**・**Notification 2.0 の service quotas**・**⭐ `alarm.type.mapping`（docs サイトには記載が無く、ここだけが根拠）**・〈rev.3〉**`auditRecord.type` の固定 enum 20 値**（§6.8 制約 5）・〈rev.3〉**`getEventBinaryResource` の Required roles ＝ `ROLE_EVENT_READ`**（§6.8 制約 6 / §11.3）・〈rev.3〉**`postAuditRecordCollectionResource` の Required roles ＝ `ROLE_AUDIT_ADMIN`**（§11.2 / AS-13）・〈rev.3〉**`getBinaryResource` の Required roles ＝ `ROLE_INVENTORY_READ` / `ROLE_BINARY_READ` / owner / `MANAGE_OBJECT_READ` on the resource**（§15.3 案 (D) の根拠）・〈rev.3〉**Audit API に `PUT` / `DELETE` が存在しないこと**） | `https://cumulocity.com/api/core/dist/c8y-oas.yml` |
| **Fragment library**（`c8y_SupportedConfigurations` / `c8y_DownloadConfigFile` / `c8y_UploadConfigFile` / `c8y_RequiredAvailability`） | `https://cumulocity.com/docs/device-integration/fragment-library/` |
| 設定管理の 3 系統 | `https://cumulocity.com/docs/device-management-application/managing-device-data/` |
| bulk operation ウィザード | `https://cumulocity.com/docs/device-management-application/monitoring-and-controlling-devices/` |
| **テナント CA・デバイス証明書**（毎年 10/2 の自動更新） | `https://cumulocity.com/docs/device-certificate-authentication/certificate-authority/` , `/device-certificate-authentication/device-certificates/` |
| **Inventory roles performance improvements（KB）** — OPTIMIZED の 2000 件閾値・`prev`/`next` | `https://community.cumulocity.com/t/inventory-roles-performance-improvements/513` |
| Single sign-on / Basic settings | `https://cumulocity.com/docs/authentication/sso/` , `/authentication/basic-settings/` |
| **Managing users**（外部認証ユーザーのパスワードリセット不可） | `https://cumulocity.com/docs/standard-tenant/managing-users/` |
| **Monitoring**（Notification 2.0 のトピック名・サブスクライバのライフサイクル・unsubscribe） | `https://cumulocity.com/docs/standard-tenant/monitoring/` |
| Managing data / Ecosystem | `https://cumulocity.com/docs/standard-tenant/managing-data/` , `/standard-tenant/ecosystem/` |
| Smart rules / **Alarm mapping（前方一致の挙動）** | `https://cumulocity.com/docs/cockpit/smart-rules/` , `/standard-tenant/alarm-mapping/` |
| **スマートルールのプリセット全 11 種**（時限自動クリアが無いことの根拠） | `https://cumulocity.com/docs/cockpit/smart-rules-collection/` |
| **SmartREST static templates**（117 は既存値を上書きしない・305/306） | `https://cumulocity.com/docs/smartrest/mqtt-static-templates/` |
| **MQTT device integration**（ClientId 形式・**コロンは deviceIdentifier に使用不可**） | `https://cumulocity.com/docs/device-integration/mqtt/` |
| **Streaming Analytics introduction**（Apama-ctrl バリアントごとの機能差） | `https://cumulocity.com/docs/streaming-analytics/introduction-analytics/` |
| **Event file binary attachment retention rules（KB）** — 添付にもリテンションが及ぶ | `https://community.cumulocity.com/t/event-file-binary-attachment-retention-rules/14387` |
| **Audit logs** 〈rev.3〉 — **記録対象は create / update / delete とログイン・ログアウトのみ**。参照・DL は記録されない（§6.8 制約 5） | `https://cumulocity.com/docs/2025/standard-tenant/audit-logs/` , `/2024/standard-tenant/audit-logs/` |
| **Security aspects** 〈rev.3〉 — *"enables applications and agents to write audit logs, which are persistently stored and **cannot be externally modified after being written**"* / *"writes own audit records related to **login and device control operations**"* | `https://cumulocity.com/docs/2026/concepts/security/` |
| **Handling personal data "at rest" in log files（Apama）** 〈rev.3〉 — ログに個人情報を書かない原則（§15.3 CU-18 の監査レコード規約） | `https://cumulocity.com/apama/docs/latest/protect-personal-data/handling-personal-data-at-rest-in-log-files/` |

> ⚠️⚠️ **`cumulocity.com/api/...` の Redoc ページは、ブラウザ以外から取得すると 4KB 程度の SPA シェルしか返りません。** 「記述が無い」と誤判定する原因になります。**必ず上記の生 spec（`c8y-oas.yml`、約 1.5MB）をダウンロードして検索してください。**
>
> ⚠️ **本書の [確] を疑うときの作法**: 一次情報を最低 2 経路（docs の別ページ / OpenAPI 生 spec / GitHub ソース / Web 検索）で探し、**探索経路を書けるときだけ「記述が無い」と結論してください。** rev.1 に対するレビューでは、この手順を踏まなかったことによる誤判定（実際には公式に記述があるのに「無い」と判断する）が複数回発生しました。
>
> ⚠️ **rev.3 でも同種の誤判定が 1 件発生しました**: 「監査ログの enum に read 系の値は存在しない」と書きましたが、**Audit logs ページの type 表に `Report` → *"…statistics accessed"* があり反例でした**（§6.8 制約 5）。**スキーマの enum だけを見て「無い」と判断せず、docs 側の一覧表も必ず突き合わせてください。**

### thin-edge.io（バージョン 2.0.1）

| 内容 | URL |
|---|---|
| **エンティティ管理** ⚠️ 親ページはカテゴリ索引のみ。実体は子ページ（**アンダースコア表記に注意**） | `…/operate/entity-management/auto-registration/`（自動登録の無効化）, `…/operate/entity-management/mqtt_api/`（`@id`）, `…/operate/entity-management/rest_api/` |
| **MQTT API リファレンス**（`te/` トピック体系・retain・ペイロード規約） | `https://thin-edge.github.io/thin-edge.io/references/mqtt-api/` |
| ⚠️ **QoS 要件とアラームの一意性（型 + severity）の逐語根拠は別ページ**（`references/mqtt-api/` に QoS の記述は無い） | `https://thin-edge.github.io/thin-edge.io/start/raise-alarm/` |
| **Thin Edge JSON**（`time` はローカル受信時に thin-edge が付与） | `https://thin-edge.github.io/thin-edge.io/understand/thin-edge-json/` |
| **Remote access**（`PASSTHROUGH` で任意 TCP・表示 3 条件） | `https://thin-edge.github.io/thin-edge.io/operate/c8y/remote-access/` |
| **Cumulocity マッパー**（supported operations の完全リスト） | `https://thin-edge.github.io/thin-edge.io/references/mappers/c8y-mapper/` |
| **Builtin mapping rules**（`max_payload_size` の逐語根拠） | `https://thin-edge.github.io/thin-edge.io/references/mappers/builtin-flows/` |
| Supported Operations（child は動的削除不可） | `https://thin-edge.github.io/thin-edge.io/operate/c8y/supported-operations/` |
| Cloud Profiles（`c8y.enable.*`） | `https://thin-edge.github.io/thin-edge.io/operate/c8y/cloud-profiles/` |
| **Availability Monitoring**（既定 1 時間・child への適用） | `https://thin-edge.github.io/thin-edge.io/operate/c8y/device-availability/` |
| Health Monitoring / systemd watchdog | `https://thin-edge.github.io/thin-edge.io/operate/c8y/health-monitoring/` , `/operate/monitoring/systemd-watchdog/` |
| Cumulocity Proxy（認証なし・401 の既知事象） | `https://thin-edge.github.io/thin-edge.io/references/cumulocity-proxy/` |
| ファイルアップロード（`tedge upload c8y`） | `https://thin-edge.github.io/thin-edge.io/operate/c8y/upload-files/` |
| **証明書管理**（EST / 自動更新 / `minimum_duration`） | `https://thin-edge.github.io/thin-edge.io/references/certificate-management/` |
| クラウド認証・CA 配置（**Edge は自己署名証明書**） | `https://thin-edge.github.io/thin-edge.io/operate/security/cloud-authentication/` |
| sm-plugin API（**ロールバックはプラグイン責務**・**5 分タイムアウト**） | `https://thin-edge.github.io/thin-edge.io/references/software-management-plugin-api/` |
| 設定管理 ⚠️ **このページには「rollback (on error)」が**ある**（2.0 の新機能）** | `https://thin-edge.github.io/thin-edge.io/references/agent/tedge-configuration-management/` |
| ⚠️ **「ロールバックなし」の逐語根拠はこちら**（file プラグイン経由の場合） | `https://thin-edge.github.io/thin-edge.io/extend/config-management/` |

> ⚠️ **thin-edge.io の設定値を確認するときは、実機で `tedge config list --doc` を正としてください。** 公式ドキュメントには既知の不備があります（[担当範囲] §12 参照）。

> ⚠️⚠️ **上記の thin-edge URL はすべてバージョン無しパスです。** 参照時点（2026-08-20）のヘッダ表示は `Version: 2.0.1` ですが、`/2.0.1/` の permalink は 404 で、アーカイブは `1.7.1` と `next` しかありません。**2.1 がリリースされると、同じ URL の内容が変わります。** 版差が効く記述（設定キー・既定値・ロールバックの有無など）を引用するときは、**参照日を併記**してください。

### ツール

- [go-c8y-cli](https://goc8ycli.netlify.app/docs/) — 全設定投入の主力。⚠️ **GitHub の既定ブランチは `master` ではなく [`v2`](https://github.com/reubenmiller/go-c8y-cli/tree/v2)**。`master` を見てコマンドの実在を判定すると誤ります（コマンド定義は `api/spec/json/*.json`、ただし manual 実装のコマンドはそこに無い）
- [go-c8y-cli settings](https://goc8ycli.netlify.app/docs/configuration/settings/) — `session.mode`（**既定は `prod`＝読取専用**）
- [c8y users resetUserPassword](https://goc8ycli.netlify.app/docs/cli/c8y/users/c8y_users_resetuserpassword/) — *"you can't set a fixed password for another user"*

---

**改版履歴**

| 版 | 日付 | 内容 |
|---|---|---|
| rev.1 | 2026-08-20 | 初版。[担当範囲] をベースに、Cumulocity 側設計を概念軸で再構成。[設定書] §4「主要設計の詳細」を本書へ移管（§0.3）。案 β 前提・案 α は付録 A |
| **rev.2** | **2026-08-21** | **一次情報照合（ファクトチェック）の反映**。8 名の独立レビュアーが約 350 件の事実主張と出典 URL 39 件を Cumulocity 公式 docs / Core OpenAPI 生 spec / thin-edge.io ソース / go-c8y-cli ソースと逐語照合。**設計変更を伴う修正 7 件**（下表）ほか、確度記号の格上げ 23 件・出典の修正 6 件。詳細は [ファクトチェック結果](Cumulocity利用設計書_ファクトチェック結果.md) |
| **rev.3** | **2026-08-23** | **個人情報を含むスナップショット画像のアクセス監査**（前提 9 / CU-18）の追加と、**5 観点レビュー**（一次情報照合 / 整合性・ID / 敵対的レビュー / 個人情報横断 / 実行可能性）の反映。**新規 ID**: 前提 9 / 制約 5・6（§6.8）/ CU-18〜CU-32 / CT-5b〜CT-5d / AS-11b・AS-13〜AS-15 / CK-8 / R-14〜R-16 / S-15・S-16 / M-04 / X-04 / DOC-1〜DOC-4 / **§11.11（新設）**。**CU-18 は未決のまま残しています** |
| **rev.4** | **2026-08-24** | ⭐⭐ **バックアップ設計の完成と運用設計の新設。4 観点の並列サブエージェントレビュー＋独立検証を実施し、確認された 13 件をすべて反映**（confirmed 13 / rejected 0）。①rev.3 が [要決定] で止めていた **CU-33（取得方式・頻度・世代・保管先）と CU-30（暗号化・破棄）に [判] 推奨設計を記載**（§12.6.2b / §12.6.6）②**取得・復元手順の実体化**（§12.6.2b / §12.6.4b。**VM 復元経路の手順と CT-35b を追加**）③K3s（datastore・token・killall・local-path）と certificate-authority の一次情報で §12.6 を補強、**CU-34 に「再エンロール不要」の見通し [推]** ④3 文書横断のカバレッジ調査（28 トピック）で判明した欠落を **§12.10（運用設計）として新設** — Edge アップグレード（UP-a〜f）・計画停止（ST-a〜c）・ホスト OS（OS-a〜e）・監視しきい値・期限台帳・アカウント棚卸し（AC）・監査レビュー（AU / **S-17・R-17 新設**）・Keycloak 依存（KC）・プラットフォームログ（LG）・変更管理（CM）。**新規 ID**: BK-j〜BK-m / UP / ST / OS / AC / AU / KC / LG / CM 各系列 / S-17 / R-17 / CT-35b / CU-35〜CU-42 / DOC-5 |

#### rev.4 のレビューで訂正した自らの誤り

| # | 章 | rev.4 初出時の記述 | 訂正 |
|---|---|---|---|
| 1 | §12.6.6 | CU-30 法務確認リストで「②③④は技術的に回避不能」と無条件に断定 | **自己矛盾。** 同じセル内で世代保持短縮・CU-18 (A) 系採用という回避策を提示していた。「現行の推奨設計を前提とする限り」という条件付きに訂正 |
| 2 | §12.6.2b | 「CU-33への回答」表がヘッダ4列・データ行3列で崩れていた | 3列に統一。実害（列ずれ表示）を解消 |
| 3 | §11.1 / §11.2 | S-17（監査担当）に `ROLE_AUDIT_READ` を付与する具体的ロールが存在しなかった | **R-17 を §11.2 に新設**し、§13.3.4 に投入行を追加 |
| 4 | §12.6.2b / §12.6.4b | 取得・復元手順に「runbook RB-BK1/RB-BK2」と本書側で仮の runbook 番号を振っていた | §12.10.6 で「採番は [担当範囲] 側」と自ら定めた原則と矛盾。**ラベルを削除**（正式番号は [担当範囲] 側の起票を待つ） |
| 5 | §12.6.2b / §12.6.4b | 復元経路を「(R-A)/(R-B)」と表記 | 既存のグローバルロール ID（R-01〜R-16）と紛らわしい。**「VM 復元」/「クリーン復元」に改称** |
| 6 | §12.6.4b | 「VM 復元」経路に具体的な操作手順が無く、CT-35 の試験対象からも除外されていた | **手順を追記し、CT-35b（年1回・VM 復元経路専用の試験）を新設** |
| 7 | §12.6.2b③ | 「日次7世代＋週次4世代＋upgrade前1世代 ≒ 遡及約35日」の内訳計算が本文に無い | 検算すると素直な GFS 方式では約28日。**BK-m で生成規則を明記し、28日に訂正** |
| 8 | §12.10.5 | バックアップ監視しきい値「経過時間>26時間」の実装経路が未定義 | 28時間に緩和し、**node_exporter textfile collector 経由の実装方式を明記**（CU-42） |
| 9 | §12.6.2b④ | 保管容量「150GB×12世代≒2TB」の150GBが P-0-5 の最小要件（下限）のまま | **実ディスクサイズ基準に訂正**し、§12.10.1 の容量再算定トリガに統合 |
| 10 | §12.10.2 UP-c | §12.6.2b②「イベント駆動3点」と数が食い違う4点目を独自に追加 | §12.6.2b②を4点に統合し、UP-cは同一事象への参照に修正 |
| 11 | §15.5 | `CU-14/15/16` が §15.0 と本節とで別内容に二重採番されたまま（文書自身が CU-17 で認識していた既知の未解消事項） | 本節側を **DOC-6/7/8 に改称**して解消 |
| 12 | §12.6.2 | MongoDB チェックポイント間隔の引用元が公式内で記述が割れている点を未注記 | 脚注を追加（結論への実害は無いことも明記） |
| 13 | [設定書] との関係 | サービスアカウント復旧理由について [設定書] §4.10 が「ID が変わるため」のまま残り、本書の訂正と矛盾 | **DOC-4 に④として追加**（[設定書] は別途改訂が必要・§0.3 の方針どおり本書側の記述を正とする） |

#### rev.4 で追加・訂正した主な内容

| # | 章 | 内容 |
|---|---|---|
| 1 | **§12.6.2b（新設）** | ⭐⭐ **CU-33 への回答**: ①VM 原子スナップショット → エクスポート → 即削除（crash-consistent。公式が snapshot-based backup を復元前提として許容 [確]）②日次＋イベント駆動 3 点 ③日次 7＋週次 4＋upgrade 前 1（≒35 日 — 個人情報の最悪残存期間として法務確認の入力に）④別障害ドメインに 2TB。**案 (iii) コールドコピーを K3s 公式手順（`systemctl stop k3s` → `k3s-killall.sh`）で具体化** |
| 2 | **§12.6.2 / §12.6.3** | K3s の一次情報で保全対象を精密化: **`server/token` を欠くと復元不能** [確]・SQLite datastore・local-path の各パス。**新規約 BK-j（世代ごとに同一版インストーラ一式を併置 — アップグレードは連続メジャーのみ・ロールバック手順なし [確] のため）/ BK-k（スナップショット残置禁止）/ BK-l（テナント CA 作成直後に 1 世代）** |
| 3 | **§12.6.4 RS-1 / CU-34** | ⭐ **リストア後のデバイス再接続に「再エンロール不要」の見通し** [推]: CA はバックアップに含まれ、照合はチェーン検証、**auto-registration 既定 ON でデバイスユーザーは初回 MQTT コールで再作成** [確]。CT-35(c) で実証する |
| 4 | **§12.6.4b（新設）** | **復元 runbook**: 発動判断（Git 再投入との使い分け）→ 経路選択（R-A: VM イメージ復元 / R-B: 公式のクリーン OS 経路）→ **復旧後チェックリスト 10 手順（キルスイッチが先頭）**。リストア試験は年 1 回＋アップグレード時・R-B 経路で |
| 5 | **§12.6.6** | **CU-30 への推奨案**: 保管先 at-rest 暗号化・鍵は admin エスクロー体系で別封筒・S-15 統制と同枠のアクセス台帳・世代ローテーション破棄。**法務確認リスト 4 点**（最悪残存 35 日・リストア時の一時復活・削除請求に応じられない 等） |
| 6 | **§12.10（新設）** | ⭐⭐ **運用設計**: 周期作業カタログ／**Edge 本体アップグレード運用**（連続メジャーのみ・戻し＝リストアの重さ・閉域パッケージ運用）／**計画停止・起動**／**ホスト OS 運用**（rev.3 まで 0 件）／**監視しきい値の初期値**（§12.7 の 16 項目に数値）／**期限台帳と [担当範囲] への runbook 追補 8 件**／**アカウント棚卸し**（0 件だった入退社対応）／**監査ログ月次レビュー**（S-17 新設）／**Keycloak への要求仕様**（停止影響の評価が 0 件だった）／**プラットフォームログの方針**／**変更管理・検証環境の常設化** |
| 7 | **訂正** | ①`c8yedge` サブコマンドは 4 つではなく **5 つ**（`package` の数え落とし・§12.6.1）②[担当範囲] への参照誤り 2 件（§11.5 の RB-11 → **RB-9**、§12.5 の「RB-13」は R-08 ライフサイクルの ID であり **TLS 証明書更新の runbook は存在しない** → §12.10.6 で起票）③§12.6.2 の「P-0-8 を追加してください」が追加済みなのに残っていた記述の解消 |

#### rev.3 で追加した主な内容

| # | 章 | 内容 |
|---|---|---|
| 1 | **§6.8** | **制約 5**（画像の参照・DL は監査ログに残らない）・**制約 6**（`GET /event/events/{id}/binaries` は `ROLE_EVENT_READ` で権限分離できない）。⭐ **制約 6 はイベント添付に固有**で、`/inventory/binaries` なら標準の権限モデルで分離できる |
| 2 | **§15.3 CU-18** | 画像の置き場所と監査方式の **5 案**（A / A′ / D / B / C）。**本書では判断していません** |
| 3 | **§11.11（新設）** | ⭐ **画像配信マイクロサービス（R-14）の認可仕様。** (A)/(A′)/(D)/(B) いずれでも配信主体は特権を持つため、**拠点分離を自前で再現しないと「直叩きは 403、正規経路は全開」になります** |
| 4 | **§7.2 / §13.3.6** | ⭐ **`AUDIT` リテンションは `type` / `source` で絞れる**（`fragmentType` は不可）→ 画像アクセス記録だけ長期保持できる。⚠️ **同時に、証跡を狙い撃ちで消す経路にもなる**（対策: `management` テナントから `editable: false`） |
| 5 | **§12.6（全面設計）** | ⭐⭐ **バックアップ／リストア。** 公式は**復元手順しか提供しておらず、取得手順は組織の責務**（`c8yedge` に backup / stop コマンドは無い）。**取得方式 3 案の比較（CU-33）・規約 BK-a〜BK-i・リストア後の二次問題 RS-1〜RS-9・RTO/RPO 算定式・CT-35/CT-36**。⚠️ **稼働中の `tar`/`rsync` は一貫性が無く復元できません** |
| 6 | **§11.1 / §11.4 / §2.8 / §13.7** | ガードレール群 — `ROLE_AUDIT_ADMIN` を人間に付けない（R-15 / AS-13）・Inventory ロールに `scope: AUDIT` を入れない（R-16）・バイナリ MO の隔離（CK-8 / AS-14）・`AUDIT` リテンションの値検査（AS-11b）・R-14 の同一性検査（AS-15） |

#### rev.3 のレビューで訂正した自らの誤り

| # | 章 | rev.3 初出時の記述 | 訂正 |
|---|---|---|---|
| 1 | §6.8 制約 5 | 「`auditRecord.type` の enum に **read/view/download に相当する値は存在しない**」 | **誤り。** `Report` → *"…statistics accessed"* が反例。**Cumulocity は一部の参照を監査している。** 正しい理由は「**画像取得 API が監査対象操作の一覧に含まれていない**」 |
| 2 | §15.3 CU-22 | 「`AUDIT` に `fragmentType` 絞り込みが効くか **[要確認]**」 | **実機確認不要。** spec に明記。`fragmentType` は不可だが **`type` / `source` は可** |
| 3 | §15.3 | 「独自 type が作れないので **`Event` を流用するしかない**」 | **誤り。** `type` はリテンションの絞り込みキーなので、**本構成で発生しない enum 値を選ぶべき**（CU-28） |
| 4 | §15.3 | 「**消える経路はリテンションルールだけ**」を安全性の根拠とした | **逆。** リテンションは**証跡を狙い撃ちで消す攻撃経路**。他に `time` 改竄・リストア・Edge root |
| 5 | §15.3 CU-18 | 確認方法「**設計判断（実機不要）**」 | **(D)/(B) は CT-5c / CT-5d が要る**（§14.12 の 2b） |
| 6 | §2.8 / §13.7 | CK-8(b) / AS-14(b) の検査対象を **R-02 / R-03 / R-04a / R-04b** とした | **R-04a/b は Inventory ロールでグローバル `ROLE_*` の器ではなく検査不能。** 実際に危ないのは **R-01 / R-05 / R-06a / R-06b / R-08 / R-10 / R-11** |
| 7 | §15.3 | (B) の評価「(D) が成立するなら採る理由はほぼない」 | **不公平。** (B) は**イベント添付なのでリテンションが効く** ＝ (D) の最大欠点（削除バッチ）が無い |
| 8 | §7.4 | 「§5 のメタ監視」 | **§12.7** |
| 9 | §15.3 CU-24 | 「OpenAPI に**この操作だけ** Required roles の記載が無い」 | 252 operation 中 **18 件**が同様。`GET /inventory/managedObjects` も含む |
| 10 | §16 | 出典表の途中に引用ブロックを挿入 | **表が分断され 18 行が崩れていた**。引用を表の後ろへ移動 |

#### rev.2 の主な設計変更

| # | 章 | rev.1 の記述 | rev.2 での訂正 |
|---|---|---|---|
| 1 | §11.2 | 拠点オペレーター／業務閲覧者に `Alarms` / `Events` / `Inventory` を**グローバルロール**で付与 | **グローバルのデータ系権限は拠点分離を無効化する**（*"all the alarms on the tenant are returned"*）。データ系は Inventory ロール経由に一本化 |
| 2 | §5.4 | child device は**メンテナンスモード**（`responseInterval: 0`）へ落とす | **メンテナンスモードはそのデバイスの全アラームを抑止する**。`x_CameraDown` も出なくなるため、**十分長い正値（32767）**に変更 |
| 3 | §5.4 / §3.8 | `c8y_RequiredAvailability` は「thin-edge が正。Cumulocity 側から書くと上書きされる」 | **逆。SmartREST 117 は既存値を上書きしない**。**Cumulocity 側を正**に変更し、初回接続前に投入する手順を追加 |
| 4 | §9.2 | `x_Alarm_<種別>` は種別ごとに個別キーが必要 | **アラーム型は前方一致**。1 キーで全種別に効く一方、**短いキーは意図しない型まで抑止する**（AM-a〜AM-c を追加） |
| 5 | §3.7 | 10/2 の CA 更新で**全拠点のデバイス証明書が同時失効**する | **失効しない**（*"existing device certificates remain valid until their expiration"*）。リスクを「**初回エンロール日の集中**」に付け替え |
| 6 | §12.2 | ドメイン・TLS・ライセンスは install 後に**変更できない** | **変更できる**。真に不可逆なのは `storageClassName` のみ。**Edge TLS 証明書の更新手順を §12.5 に追加** |
| 7 | §10.5 | 通知バックログは**ディスク逼迫**の経路 | **25 MiB の Hard quota に達すると、その拠点のアラーム・イベント POST が 500 で失敗**する。監視手段は公式 UI にあり（SV-33 解消） |
| 8 | §1.4 / §2.2 / §13.3.3 | 拠点グループにも external ID を振る。main device の external ID は `site001:ANLZ-…` | **グループの external ID は廃止**（`x_Site.siteId` を同定キーに）。**main device はコロンが使えない**（証明書 CN 制約）ため `site001-ANLZ-…` に変更。**グループ作成の「409 黙殺」は誤り**（`POST /inventory/managedObjects` は 409 を返さず、再実行で重複グループが生える） |

**併せて訂正した誤り**: §8.2 RU-2（時限自動クリアのスマートルールは存在しない）、§6.9 TM-d（時刻を付けるのは Cumulocity ではなく thin-edge・ローカル受信時）、§6.8 制約 1（既存イベントへの後付け添付は可能）、§2.8 CK-1（`GET /identity/externalIds` は存在しない）、§13.1 CF-c（`prod` はエラーで落ちる。既定が `prod`）、§11.4（ユーザー ID は変わらない。変わるのはパスワード）、§5.9 / §7.7（Wide シナリオは本構成に直撃しない）。

**rev.1 が正しかったことを確認した主な項目**: §1.4 の `subscription` 名パターン `^[a-zA-Z0-9]+$`、§3.6 の go-c8y-cli コマンド 3 種、§2.6 の `entity_store.auto_register` 両キー、§7.7 の「約 100 tps/CPU コア」、§4.8 の `creationRamp`、§5.2 の `responseInterval` 値域、§10.3 の RBAC バイパス。
