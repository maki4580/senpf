# Cumulocity 利用設計書

**— 本構成における Cumulocity Edge の使い方の定義と、その設定への落とし込み —**

**対象構成**: [cumulocity-iot-architecture.drawio](cumulocity-iot-architecture.drawio) —「全体構成(配置構成・レビュー反映)」タブ**のみ**
**版**: rev.1（2026-08-20）
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
| **前提 4** | Edge 上で **Notification 2.0 が実動作する**（WebSocket が LoadBalancer 経由で到達できる） | §10 が丸ごと成立しない。**変-2 と併せて、通知経路がゼロになる** | **SV-05（最優先）** |
| **前提 5** | イベント/アラーム型・external ID 採番規約（§2.2・§6.4）が**関係チームと合意される** | §9 のマッピング・§7.2 の type フィルタ・§8 のルール条件・§10.2 の `typeFilter` が全部やり直し | [担当範囲] C-1 / C-2 |

> ⚠️ **前提 2〜4 は「検証環境で最初に潰す」項目です。** 本書の設計を本番へ投入する前に、[担当範囲] W1-2 / W1-3 / W1-10 を完了させてください。

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
  §12 Cumulocity Edge セットアップ
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
| **2** | **Inventory ロールの割当単位** | ロールは親グループ → サブグループ → デバイスへ継承される [確] | §11.3 |
| **3** | **Notification 2.0 サブスクリプションの `source`** | `context: mo` + `source` = 拠点グループ MO + `alarmsWithChildren` で拠点単位の購読になる [確] | §10.2 |
| **4** | **一括オペレーションの対象指定** | bulk operation の `groupId` は *"Identifies the target group on which this operation should be performed."* [確]。**API 経由の一括オペレーションはグループ指定が事実上の前提** | §4.8 |
| **5** | **段階ロールアウトの単位** | AI モデル配布・設定配布を 1 拠点 → 数拠点 → 全拠点で進める単位 | §4.9 |

> ⚠️ **役割 3 が成立するかは未確認です（前提 2 / SV-35）。** 成立しない場合の代替は §1.6 に置きます。

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
| **グループ external ID** | **拠点コード**（例: `site-001`）。**英数字のみ** | 冪等投入の土台。名前だけで参照すると再実行で重複グループが生える |
| **拠点コード** | **`^[a-zA-Z0-9]+$` に適合すること** | Notification 2.0 の `subscription` 名がこのパターン [確]。**ハイフン・アンダースコア・日本語は不可** |

> ⚠️⚠️ **`site-001` はハイフンを含むため、Notification 2.0 の `subscription` 名にはそのまま使えません** [確]。次のいずれかを §2.5 で確定してください。
>
> - **(a) 拠点コードを最初から英数字のみにする**（例: `site001`）。external ID・グループ名・トピック名を 1 つの値で通せる → **[判] 推奨**
> - (b) 拠点コード（`site-001`）とトピック名用コード（`site001`）を別に持ち、対応表を維持する
>
> **(b) は対応表が second source of truth になり、拠点追加のたびに同期漏れのリスクを負います。(a) を推奨します。**

**[判] 本書は以降 `site001` 形式（英数字のみ）を採用した表記を併記しますが、実値の確定は §2.5 に委ねます。**

### 1.5 グループに持たせるフラグメント

| フラグメント | 対象 | 内容 | 用途 |
|---|---|---|---|
| `c8y_IsDeviceGroup` | 全グループ | Cumulocity 標準 | グループとして認識される |
| `x_Site` | 拠点グループ | `{ "siteId": "site001", "siteName": "〇〇工場 東棟" }` | 拠点コードの正引き。**デバイス側の `x_Site` と同じキーを使う**（§6.4） |

> **[判] グループにも `x_Site` を持たせてください。** デバイス側だけに持たせると、「グループ ID から拠点コードを引く」処理が名前解析になります。

### 1.6 ⚠️ 拠点分離の成立条件と、不成立時の代替

**拠点分離は「グループを作れば成立する」ものではありません。3 つの条件が同時に必要です。**

| # | 条件 | 状態 | 不成立時に壊れるもの |
|---|---|---|---|
| **1** | Inventory ロールがグループ配下へ継承される | **[確]** 成立 | — |
| **2** | `alarmsWithChildren` / `eventsWithChildren` が **`childAssets` を辿る** | **[要] 未確認（SV-35 / TE-12）** | **拠点グループ単位の通知購読が全滅** |
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
| **G-a** | **すべてのグループに external ID を振る** | 名前だけで参照すると再実行で重複グループが生える |
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

| 対象 | external ID | 例 |
|---|---|---|
| 拠点グループ | `{siteId}` | `site001` |
| 画像解析装置（main device） | `{siteId}:ANLZ-{シリアル}` | `site001:ANLZ-SN00123` |
| IP カメラ（child device） | `{siteId}:CAM-{シリアル}` | `site001:CAM-SN12345` |
| BOXゲートウェイ（main device） | `{siteId}:GW-BOX` | `site001:GW-BOX` |
| 画像センシングBOX（child device） | `{siteId}:BOX-{シリアル}` | `site001:BOX-SN98765` |

**規約**:

| # | 規約 | 理由 |
|---|---|---|
| **X-a** | **`{siteId}` プレフィックスを必ず付ける** | 全拠点が単一テナントに集約されるため（D17）、シリアルだけでは衝突し得る。また案 α では 1 thin-edge インスタンス内での一意性も必要 |
| **X-b** | **機器種別は `ANLZ` / `CAM` / `GW` / `BOX` の 4 語に固定** | 型からの逆引きと機械検査を単純にする |
| **X-c** | **区切りは `:`（拠点）と `-`（種別とシリアル）で固定** | パース規則を 1 つにする |
| **X-d** | **シリアルは機器の物理シリアルを使う。連番を振らない** | 交換時に同一性を追跡できる（§3.9） |
| **X-e** | ⚠️ **一度登録した external ID は変更できない** [確] | thin-edge の `@id` は登録後変更不可。**間違えたら削除して作り直すしかない**（§2.6 E-e） |

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

**[要] いずれも実機で確認してください（TE-6 / V18）。**

| 対象 | 制約 | 確度 |
|---|---|---|
| Notification 2.0 の `subscription` 名 | **`^[a-zA-Z0-9]+$`** | [確] |
| thin-edge の topic id | `/` `+` `#` 不可。`/` は external ID 生成時に `:` に変換される | [確] |
| オペレーション定義ファイル名 | **英数字と `_` のみ**（ハイフン不可。使うと定義が無視される） | [確] |
| `c8y_RequiredAvailability.responseInterval` | `-32768`〜`32767`（範囲外は境界に丸められる） | [確] |
| external ID の文字種・長さ | **未確認**。`site001:CAM-SN12345` 形式が通るかを実機確認 | **[要 TE-6]** |
| `device.id`（証明書 CN 由来） | 同上 | **[要 TE-6]** |

### 2.8 規約準拠の機械検査

**[判] 規約は「文書にする」だけでは守られません。CI で機械検査してください。**

| # | 検査 | 実装 | 不合格時 |
|---|---|---|---|
| **CK-1** | 全 MO の external ID が `^[a-z0-9]+:(ANLZ\|CAM\|GW\|BOX)-.+$` に適合 | `GET /identity/externalIds` を全走査 | CI 失敗 |
| **CK-2** | 全デバイスの `type` が 4 種のいずれか | `GET /inventory/managedObjects?fragmentType=c8y_IsDevice` | CI 失敗 |
| **CK-3** | 全 `x_Camera` に `vmsCameraId` が設定されている | 同上 | CI 失敗 |
| **CK-4** | 全 main device が**いずれかの拠点グループの `childAssets`** に入っている | グループを走査して差集合を取る | CI 失敗 |
| **CK-5** | external ID のプレフィックス `{siteId}` と、実際に所属する拠点グループが一致 | 同上 | ⚠️ **BOXアダプタの誤配送を検出できる唯一の手段**（§6.2） |
| **CK-6** | `ROLE_NOTIFICATION_2_ADMIN` が §11.4 の指定ユーザー以外に付いていない | `GET /user/{t}/groups` + `/users` | CI 失敗 |

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
| IP カメラ（child） | なし | **なし** | **`responseInterval: 0`**（§5.3） | `x_Camera{vmsCameraId, location}` ← **`vmsCameraId` 必須** |
| BOXゲートウェイ | **あり** | **縮退**（§4.2） | **あり** | `x_Site{siteId}` |
| 画像センシングBOX（child） | なし | **なし** | **`responseInterval: 0`** | `x_SensingBox{model, fwVersion, channels}` |

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

**制約** [確]: ①`PATH` 使用時は `TYPE` と `NAME` 列も必要 ②`com_cumulocity_model_Agent.active` ヘッダ（値 `true`）が必要 ③**テナント CA の存在が前提** ④`ENROLLMENT_OTP` と `PATH` の併用例は仕様に無い（禁止記述も無い）→ **[要 SV-07]** ⑤UI ウィザードは `ENROLLMENT_OTP` 非対応。

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
| 3 | **main device を拠点グループの `childAssets` へ追加** | 拠点グループ配下に表示される | ○（409 黙殺） |
| 4 | **child device の明示登録**（`@id` 指定・§2.6） | 全 child が規約どおりの external ID で登録済み | ○ |
| 5 | **`x_Camera.vmsCameraId` の設定** | **未設定はツールがエラー** | ○ |
| 6 | child の `c8y_RequiredAvailability = {"responseInterval": 0}` 設定 | メンテナンスモード（§5.3） | ○ |
| 7 | `x_Site` / `x_SensingBox` フラグメントの設定 | §2.4 のとおり | ○ |
| 8 | supported operations の確認 | **child には何も出ていない**（§4.3） | — |

> **冪等化パターン**: 手順 3 は **409 黙殺**（[投入ガイド] パターン E）、手順 4〜7 は **存在チェック → 分岐**（パターン C）→ §13.5

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

**自動更新** [確]: `tedge-mapper` パッケージが `tedge-cert-renewer@c8y.timer` / `.service` を提供し、**毎時**（`OnCalendar=hourly` + `RandomizedDelaySec=5m`）判定します。判定基準は `certificate.validity.minimum_duration`（**既定 30 日**）[確S]。更新は EST **`simplereenroll`** で行われます。

**Cumulocity 側から見た論点**:

| # | 論点 | 内容 | 対処 |
|---|---|---|---|
| **1** | **更新は c8y-proxy 経由。Cumulocity に繋がっていることが前提** | mapper が動いていて Edge に到達できなければ更新できない [確S] | 長期オフラインを許さない運用、または 2 |
| **2** | **失効したら現地作業になる** | *"If for any reason the certificate expires, and the device can no longer communicate with Cumulocity, then the device will need to be registered again."* [確] | **`certificate.validity.minimum_duration` を「想定最大オフライン期間 × 3 以上」に設定**（既定 30 日では 10 日以上の断で危険） |
| **3** | **Edge は自己署名証明書を使う** | *"Updating the local certificate store is notably required to connect Cumulocity Edge, as this distribution of Cumulocity uses self-signed certificates"* [確] | 社内 CA / Edge の CA をデバイス側の信頼ストアへ（§12.5） |
| **4** | **有効期限をメタ監視の対象にする** | 期限切れは静かに起きる | §12.7 |
| **5** | **失効手順（拠点撤収・機器盗難）を runbook 化する** | **監査ログ対象**（[連携仕様] §4.7） | §3.9 |

### 3.7 ⚠️⚠️ テナント CA の毎年 10/2 自動更新

**テナント CA は毎年 10 月 2 日 02:00 に自動更新されます** [確]。

> **これは日付が確定した障害リスクです。** 閉域網で `simplereenroll` が期待どおり動かないと、**深夜帯に全拠点のデバイス証明書が順次失効し、全拠点が同時に接続不能**になります。しかも**配布経路（Cumulocity 経由のソフトウェア更新）も同時に使えません。**

**[判] 必須対応**:

| # | 対応 | 実施時期 |
|---|---|---|
| 1 | **検証環境で時計を進めてシミュレートする**（SV-25 / TE-7） | W1 |
| 2 | **証明書有効期限をメタ監視の対象にする** | §12.7 |
| 3 | **運用カレンダーに載せ、当日は要員を確保する** | 運用開始前 |
| 4 | **新旧 CA を並行して信頼させる移行期間を設計に入れる**（SV-22） | 設計 |
| 5 | ⚠️ **復旧局面の輻輳対策**: 全拠点が**同時に**再エンロールを試みる。①拠点グループ単位でずらす ②リトライにジッター・バックオフを持たせる、を runbook に含め、**複数デバイス同時実行のケースも検証する** | W1 + runbook |

### 3.8 デバイスツイン（フラグメント）の管理方針

**[判] 各フラグメントの「唯一の情報源」を 1 つに固定します。二重管理すると「なぜか設定が戻る」という切り分けの難しい事象になります。**

| フラグメント | 唯一の情報源 | Cumulocity 側から書いてよいか |
|---|---|---|
| `c8y_RequiredAvailability`（main device） | **thin-edge の `c8y.availability.interval`** | ❌ **書かない**。次回接続時に thin-edge の値で上書きされる |
| `c8y_RequiredAvailability`（child device） | **プロビジョニングツール**（`responseInterval: 0`） | ✅ プロビジョニングツールのみ |
| `c8y_SupportedOperations` | **thin-edge の capability メッセージ** | ❌ **書かない**（§4.2） |
| `x_Site` / `x_Camera` / `x_SensingBox` | **プロビジョニングツール**（entity 登録時の twin） | ✅ プロビジョニングツールのみ |
| `x_NoAutoConfig` | **運用者の判断**（手動または構成コード） | ✅ |
| `name` | **プロビジョニングツール** | ✅ |
| `type` | **プロビジョニングツール**（entity 登録時） | ⚠️ **登録後変更不可**（§2.6 E-e） |

### 3.9 デバイスの追加・交換・撤去

| 事象 | 手順 | 論点 |
|---|---|---|
| **拠点追加** | 構成コードに `sites/site-00N.yaml` を追加 → プロビジョニング実行（§3.5） | **[判] 2 拠点目が「`sites/` の追加だけ」で立つことが設計品質の判定基準**（[担当範囲] §2.6） |
| **カメラ / BOX の増設** | child 登録（§3.5 手順 4〜7）を追加分だけ実行 | 冪等なので全件再実行でよい |
| **カメラ / BOX の撤去** | thin-edge の entity 削除（空 retained）→ Cumulocity の MO を削除するか論理削除するか | ⚠️ **[要]** 過去のイベント・アラームの source が消える。**[判] MO は残し、`x_Retired` フラグメントで論理削除**を推奨 |
| **装置の交換**（同一 siteId・同一役割） | ⚠️ **シリアルが変わるため external ID が変わる**（§2.2 X-d）。新 external ID で登録 → 旧 MO を論理削除 | **[判] external ID を交換前後で同じにしない。** 同じにすると「いつ交換したか」が追跡できなくなる |
| **拠点撤収** | ①デバイス証明書の失効 ②MO の論理削除 ③Notification 2.0 サブスクリプションの unsubscribe（§10.5）④Inventory ロール割当の解除 | ⚠️ **③を忘れるとバックログが溜まり続ける** |

> ⚠️ **撤去・交換の手順は runbook 化してください**（[担当範囲] RB-2 / RB-3 / RB-12）。**特に「MO を物理削除するか論理削除するか」は先に決めてください。** 物理削除すると過去のイベントの `source` が欠損し、オフロード済みデータとの突合ができなくなります。

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

> ⚠️ **Cumulocity 側で supported operations を減らすことはできません。** これは**デバイス側（thin-edge）が申告するもの**です。したがって本節の「決定」は thin-edge の構成コードに落ちる要件になります（実装は [担当範囲] §5.8）。

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
| 3 | ⚠️ **child device の supported operations は動的に削除できません** [確]（*"Dynamic removal of an operation from the supported operation list is not supported for child devices."*）。**一度申告すると消せない** |

> **[判] 検証項目として明示的に置いてください**（§14 CT-16）: 「カメラ・BOX の詳細画面にオペレーションの実行 UI が出ないこと」。**一度出てしまうと後戻りできないため、パイロット拠点で必ず確認してください。**

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
| **4** | **リテンション** | オペレーションは 90 日で削除（§7.2）。**失敗した Operation の記録も消える**ため、結果は `x_ModelApplied` / `x_ModelUpdateFailed` として残す |
| **5** | **ロールバックは Cumulocity 側の機能ではない** | *"software_update に自動ロールバックは存在しない"* [確]。**プラグイン実装者の責務**（[担当範囲] §6.3） |

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
| 5 | ⚠️ **配信は URL フェッチ型** | デバイスが Edge の HTTP エンドポイントに到達でき、**社内 CA を信頼している**必要がある（§12.5） |
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

> ⚠️⚠️ **§4.3（child に何も申告しない）と「カメラへの設定配布」は両立しません。** カメラへの設定配布が要件になる場合、**child に `c8y_DownloadConfigFile` だけを申告する**という選択になりますが、**一度申告すると child からは削除できません** [確]。**SV-17 / SV-28 の決着まで、child には何も申告しない方針を維持してください。**

#### ⚠️ 設定配布の落とし穴

| # | 落とし穴 | 対処 |
|---|---|---|
| 1 | **ファイルベース設定更新にロールバックは無い** [確] | 独自 config plugin を書くか、配信前の値検証で守る（§4.10 G-3） |
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

**`c8y_RemoteAccessConnect` は Cumulocity から装置への SSH / VNC を張る機能です。** D9（拠点への着信接続を設けない）の趣旨に照らし、**画像解析装置では無効化します**。

> ⚠️ **無効化のための thin-edge 設定フラグは存在しません** [確]。次の組み合わせで封じます。

| # | 手段 | 主体 | 確度 |
|---|---|---|---|
| 1 | **`c8y-remote-access-plugin` パッケージを導入しない**（`tedge-full` を使わない） | thin-edge 側 | [判] 最も確実 |
| 2 | `/etc/tedge/operations/c8y/c8y_RemoteAccessConnect` を削除 → 114 の再送で Cumulocity 側の一覧からも消える | thin-edge 側 | [確] |
| 3 | `c8y-remote-access-plugin.socket` を disable | thin-edge 側 | [推] |
| 4 | ⭐ **Cumulocity テナント側で Cloud Remote Access のロールを誰にも割り当てない** | **Cumulocity 側（本書の担当）** | [判] |

> ⚠️ **2 だけではパッケージの再インストール・更新で復活し得ます。** **1 と 4 を主、2・3 を従**としてください。**4 は本書 §11.2 のロール定義に反映します。**

> **検証**（§14 CT-17）: 「Cumulocity UI に Remote access のボタンが出ないこと」「`114` の一覧に `c8y_RemoteAccessConnect` が無いこと」。

### 4.8 一括オペレーションと流量制御

| 項目 | 内容 | 確度 |
|---|---|---|
| **対象指定** | bulk operation の `groupId`（拠点グループ） | [確] |
| ⭐ **流量制御** | リクエスト本文の **`creationRamp`**（*"Delay between every operation creation in seconds."*、例 `15`）。**発行側で指定できる主たる制御** | [確] |
| 失敗分の再実行 | `failedParentId` で失敗分だけ再スケジュールできる | [確] |
| システムオプション | `device-control/bulkoperation.creationramp` は実在するが**読み取り専用**で、意味（既定値か下限か）を述べた一次記述は無い | **[推 / 要 SV-19]** |

> **[判] `creationRamp` を CI 側で必須パラメータ化してください。** 指定しないと、拠点数 × 装置数のオペレーションが一斉に生成され、Edge のスループット上限（§7.7）に直撃します。

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
| **IP カメラ** | **thin-edge A の ONVIF 監視サービス** | `x_CameraDown` | **`responseInterval: 0`（無効化）** |
| **画像センシングBOX** | **BOXアダプタ**（無通信タイムアウト） | `x_BoxSilent` | **`responseInterval: 0`（無効化）** |
| **Cumulocity のマイクロサービス** | **Cumulocity**（自動生成） | `c8y_Application_Down` / `c8y_Application_Unhealthy` | — |
| **Edge 自体** | **メタ監視**（Prometheus・§12.7） | Cumulocity の外 | — |

> ⚠️ **`c8y_Application_*` の対象は Cumulocity のマイクロサービスです。** VM1/VM2 上の非 Cumulocity コンポーネント（Keycloak・Otel・BOXアダプタ等）は対象外で、そちらは Otel 側の責務です。

### 5.2 `c8y_RequiredAvailability` の仕組み

| 項目 | 内容 | 確度 |
|---|---|---|
| 機能 | `responseInterval` を設定すると、指定時間内に通信が無い場合にアラームが自動生成される | [確] |
| 生成されるアラーム | `c8y_UnavailabilityAlarm`（既定 MAJOR / *"No data received from the device within the required interval."*） | [確] |
| **値域** | **`-32768`〜`32767`（分）。範囲外は境界に丸められる** | [確] |
| **無効化** | **`responseInterval: 0`**（メンテナンスモード） | [確] |
| 自動クリア | 通信再開時に Cumulocity が自動でクリア | [確] |

### 5.3 ⚠️⚠️ 落とし穴 — thin-edge は既定で全 child device に 60 分を設定する

thin-edge.io の公式ドキュメントに逐語で [確]:

> *"%%te%% main and child devices set their required interval during their first connection. Availability monitoring is enabled by default with a default required interval of 1 hour."*

**つまり何もしないと、代理登録した全カメラ・全 BOX に `c8y_RequiredAvailability = 60 分` が付きます。** カメラ・BOX は自分でハートビートを送らないため（既定のヘルス判定元はそのデバイス上の `tedge-agent` サービスで、代理登録した child には存在しない）、**接続から 1 時間後に全 child が一斉にアラームを上げると推測されます** [推]。

**数百台 × N 拠点の規模では、初回接続の 1 時間後に数百〜数千件のアラームが同時発生することを意味します。しかもアラームは `CLEARED` にならない限り消えません**（§7.3）。

### 5.4 **[判] 対処方針**

| # | 対処 | 内容 |
|---|---|---|
| **1** | **ゲートウェイの死活は thin-edge のハートビート機構を使う** | `c8y.availability.interval` を設定。**1 分未満または 0 を設定すると無効化される** [確] |
| **2** | ⭐ **child device は登録時にメンテナンスモードへ落とす** | `c8y_RequiredAvailability = {"responseInterval": 0}` を twin で設定（§3.5 手順 6）**[要 TE-5: 実機で child への twin 反映を確認]** |
| 3 | 代替案: `@health` を使う | child 登録メッセージの `@health` に監視サービスの topic id を指定すると、そのサービスが `up` の間だけハートビートが送られる [確]。**公式機構だが、アラーム型が `c8y_UnavailabilityAlarm` になり `x_CameraDown` 規約と食い違う**ため、**本書は 2 を採る** |

> ⚠️ **設定箇所を Cumulocity 側と thin-edge 側で二重管理しないでください。** main device の `c8y_RequiredAvailability` は **thin-edge の `c8y.availability.interval` が唯一の情報源**とし、Cumulocity 側から手で書かないこと（§3.8）。書くと次回接続時に thin-edge の値で上書きされ、「なぜか設定が戻る」という切り分けの難しい事象になります。

### 5.5 `responseInterval` の設計値

| 対象 | **[判] 提案値** | 設定箇所 | 根拠 |
|---|---|---|---|
| **画像解析装置** | **15 分** | thin-edge `c8y.availability.interval = 15m` | 死活計測が 1〜5 分間隔。WAN の瞬断・再接続で誤報しない余裕を取る。**過小だと回線瞬断のたびに拠点数ぶんのアラームが上がる** |
| **BOXゲートウェイ** | **10 分** | thin-edge `c8y.availability.interval = 10m` | 同一サーバールーム内で WAN 断の影響を受けないため短くできる |
| **IP カメラ / BOX** | **0（無効）** | プロビジョニングツール（twin） | 死活は `x_CameraDown` / `x_BoxSilent` に一元化 |

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
| **初回接続の 1 時間後**（§5.3 の落とし穴） | 全 child（数百〜数千） | **§5.4 の対処 2**（登録時に `responseInterval: 0`） |
| **WAN 瞬断**（`responseInterval` が過小） | 拠点数ぶんの main device | **§5.5 の値の実測確定** |
| **Edge 停止 → 復旧** | 全 main device | Cumulocity 側の自動クリアで解消。ただし通知が復旧時刻に集中する → §8.4 |
| **GW 停止**（案 α のみ） | 全拠点の BOX | 案 β では発生しない → 付録 A |
| **網断復旧時の一括再送** | 断の長さに比例 | **リプレイ抑止ガード**（§8.4） |

> ⚠️ **アラームは `CLEARED` にならない限り削除されません** [確]。ストームが 1 回起きると、手動クリアするまで Operational Store に残り続けます。**「起きてから消す」ではなく「起こさない」設計が必要です。**

### 5.9 死活計測の送信量とスケールへの影響

**死活計測は Edge の受信量の支配項になります。**

```
[連携仕様] §3.5 の試算 : 50 拠点 × 300 台 ÷ 60 秒 ≒ 250 msg/s（死活のみ）
Edge の目安            : 約 100 tps / CPU コア  [要・伝聞値]
Wide ベンチマーク      : 8 CPU / 16GB で 1,200 接続クライアント  [確]
```

> ⚠️⚠️ **Wide シナリオが本構成に直撃します。** 「1 拠点あたり数百台 × N 拠点」の child device 設計に対し、8 CPU で 1,200 クライアントが上限です（§7.7）。

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
| 計測 | **禁止** | 0 or 1 | retain すると再接続のたびに同じ計測が再送される |
| イベント | **禁止** | 1 | 同上 |
| **アラーム** | **必須** | **2** | **retain しないとアラームが上がらない／クリアできない** |
| **アラームのクリア** | **必須**（空メッセージ） | **2** | **クリアされずアラームが残り続ける** |
| ツイン | 必須 | — | 反映されない |
| サービス死活 | 必須 | — | 反映されない |

> ⚠️ **アラームの retain 要件は、実装者が最も間違えやすい点です**（[担当範囲] §7.4）。**IF 仕様書に太字で書き、疎通試験の必須項目にしてください。**

### 6.7 ⚠️ サイズ制約 — 16KB の壁

| 経路 | 上限 | 超えたときの挙動 | 確度 |
|---|---|---|---|
| **ローカル MQTT（計測）** | `c8y.mapper.mqtt.max_payload_size` = **16,184 バイト** | **拒否される（破棄される）。気づけない** | [確S] |
| **ローカル MQTT（イベント）** | 同上 | **HTTP に自動で切り替わる** | [確] |
| **REST（添付バイナリ）** | 既定 **50MiB**（チャンク 5MiB） | エラー | [確] |

> ⚠️⚠️ **16KB 制約と 50MiB 制約は別レイヤの話です。** 16KB は **MQTT トピック投入経路にのみ適用**され、REST（`tedge upload c8y` / c8y-proxy 直叩き）経路には及びません。混同しないでください。

> ⚠️ **イベントが 16KB を超えると HTTP 経路に切り替わり、ストア&フォワードの対象外になります** [確]。**網断中はそのイベントが失われます。**

**[判] 規約**:

| # | 規約 |
|---|---|
| **S-a** | **1 イベントのペイロードは 16KB 未満とする** |
| **S-b** | **`boundingBoxes` の件数上限を定める**（例: 32 件）。混雑シーンで検知数が増えると容易に超える |
| **S-c** | **超えた場合の挙動を規約で決める**（切り詰めるか、別イベントに分けるか） |
| **S-d** | **発行側でサイズを検証してから publish する**（超過を検知できるようにする） |

### 6.8 添付（スナップショット）の扱い

**構成図の「スナップショット: 初期版は Cumulocity イベント添付のみ」に対応します。**

| 項目 | 内容 |
|---|---|
| **保存先** | Cumulocity のイベント添付（**Operational Store と同一 MongoDB・同一保持期間で消える**） |
| **経路** | c8y-proxy 経由の REST。**MQTT ではない** |
| **推奨手段** | **`tedge upload c8y`**（イベント作成 + 添付 + イベント ID 取得を 1 コマンドで行う）[確] |
| **制約 1** | **既存イベントへの後付け添付はできない**（新規イベント作成とセット） |
| **制約 2** | **1 イベント 1 バイナリ**（2 件目は HTTP 409） |
| ⚠️ **制約 3** | **網断耐性なし**。HTTP 経路は**ストア&フォワードの対象外**。**発行側のローカルスプールが必須** |
| ⚠️ **制約 4** | c8y-proxy は *"requests to the proxy API are occasionally spuriously rejected with a `401 Not Authorized` status code"* [確] → **リトライを実装規約に入れる** |
| **将来** | オブジェクトストレージ本体 + サムネイル添付へ移行（構成図の配置の要点） |

> ⚠️ **[要 SV-06] リテンションが添付バイナリに及ぶかは公式に記載がありません。** 検証環境で短期リテンションを設定して観測してください。**及ばない場合、添付だけが MongoDB に残り続けます。**

> **[判] 添付を出すのは画像解析パイプラインのみです。** BOX はスナップショットを出しません（**[要 TE-9] N2 で確認**）。

### 6.9 時刻の規約

| # | 規約 | 理由 |
|---|---|---|
| **TM-a** | **`time` は「事象の発生時刻」。受信時刻ではない** | 再送時も同一値であることが重複判定の前提 |
| **TM-b** | **ISO 8601（タイムゾーン付き）または unix 秒** | [確] |
| **TM-c** | ⚠️ **全ノードで NTP を必須にする**（Edge ホスト・画像解析装置・外部Gateway・Keycloak） | **時刻ずれは「TLS の有効期限判定」「JWT の `exp`/`iat` 検証」「イベントと映像クリップの突合」を同時に壊し、エラーから原因に辿り着けなくなる**（IF-P03 / P-0-3） |
| **TM-d** | **`time` が無い場合は Cumulocity が受信時刻を付ける** | 網断復旧後の一括再送で**全イベントが復旧時刻になる**。**発行側が必ず `time` を入れること** |

> ⚠️ **TM-d は網断時に効きます。** `time` を省略すると、3 時間の網断から復旧した瞬間に「3 時間分のイベントが全部いま起きた」ことになり、時系列分析も映像突合も壊れます。**IF 仕様書で `time` を必須項目にしてください。**

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
| **`AUDIT`** | **[要決定]**（イベントとは別基準） | **設定変更の追跡・障害調査要件から逆算する**（§4.10 G-13） |

**Cumulocity の既定と制約** [確]:

| 事実 | 帰結 |
|---|---|
| 新規 Edge には**既定ルール（全履歴データ 60 日）**が先に存在する | 「あるべき集合」を宣言的に適用する必要がある |
| 上限は 10 年 | — |
| **`files` リポジトリには適用されない** | ソフトウェアリポジトリは別途削除運用が要る（§4.4 論点 2） |

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
| **`x_BoxParseError`** | **[要決定]** | ⚠️ デッドレター処理後に手動クリアか、時限自動クリアか |
| **`x_Alarm_<種別>`** | **[要決定]** | ⚠️ 案件側が決定 |

### 7.4 添付バイナリ

| 項目 | 状態 |
|---|---|
| リテンションが添付に及ぶか | **[要 SV-06]** 公式に記載なし。**検証環境で短期リテンションを設定して観測** |
| 及ばない場合の帰結 | **添付だけが MongoDB に残り続ける** → 独自の削除運用が必要 |
| サイズ | 既定 50MiB（チャンク 5MiB） |

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

> ⚠️⚠️ **`storageClassName` と MongoDB 容量は install 後に変更できません** [確]。**§12 のセットアップより前に確定が必要です。**

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
| 一般的な目安 | **約 100 tps / CPU コア** | [確]（**ベンダー資料からの伝聞値として扱う**） |
| Narrow（10 クライアント）8 CPU / 16GB | 25,000 measurement/s | [確] |
| Narrow 16 CPU / 32GB | 47,500 measurement/s | [確] |
| **Wide（各 1 measurement/s）8 CPU / 16GB** | **1,200 接続クライアント** | [確] |
| **Wide 16 CPU / 32GB** | **2,200 接続クライアント** | [確] |
| ハードウェア最小要件 | CPU 8 コア / RAM 16GB / Disk 150GB。**MongoDB は AVX 命令 + x86-64-v3 以降が必須** | [確] |

> ⚠️⚠️ **Wide シナリオが本構成に直撃します。** 「1 拠点あたり数百台 × N 拠点」の child device 設計に対し、8 CPU で 1,200 クライアントが上限です。
>
> **ただし「接続クライアント」の定義に注意してください。** 本構成で MQTT 接続を張るのは **main device（thin-edge インスタンス）だけ**で、child device（カメラ・BOX）は接続を持ちません。**接続数は「拠点数 × 2」（案 β）になります。** 一方、**measurement のレートは child 台数に比例**するため、Narrow 側の指標も併せて見る必要があります。
>
> **[要 SV-14] 全拠点合算の負荷試験は必須です**（§14 CT-28）。**「接続数」と「レート」の両面で実測してください。**

> ⚠️ **案 α を採ると、外部Gateway の thin-edge が 1 プロセスで全拠点分のイベントを処理します**（**[要 TE-13]** child device 数の上限はドキュメントに記載なし）→ 付録 A。

---

## 8. 条件一致メッセージの処理

### 8.1 処理系の選択肢と使い分け

**Cumulocity Edge には 3 つの処理系があり、いずれも 2026 Edge に `Included` です** [確]。

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
| **RU-2** | **アラームの自動クリア** | スマートルール | 各アラーム型 | CLEARED | **基盤**（§8.3） |
| **RU-3** | ⚠️ **リプレイ抑止ガード** | EPL | 全イベント | 抑止 | **基盤**（§8.4・**雛形に必須で組み込む**） |
| **RU-4** | **BOX 再送の重複排除の補助** | EPL | `x_Detection_*` の `eventUuid` | 重複の除外／検知 | ⚠️ **一次責務は BOXアダプタ**（§8.5） |
| **RU-5** | **拠点単位のロールアップアラーム** | EPL | `x_CameraDown` / `x_BoxSilent` の多発 | 拠点集約アラーム | 基盤（**案 α では必須**・付録 A） |
| **RU-6** | **D14 の自動クリップ保存** | — | `x_Alarm_<種別>` | `exportClip` 呼び出し | ⚠️ **ルールではなく Notification 2.0 のサブスクライバが実装**（§10.2） |

> ⚠️ **RU-6 は変-1 により設計が変わりました。** 以前は「EPL から HTTP を出す」案でしたが、**Notification 2.0 のサブスクライバが受けて `exportClip` を叩く**方式になり、Apama の HTTP クライアントプラグイン（SV-10）の可否は論点から外れました。

### 8.3 アラームの自動クリア

**§7.3 のとおり、リテンションは `CLEARED` のアラームしか削除しません。自動クリアを設計しないとアラームが単調増加します。**

| アラーム型 | クリア主体 | 実装 |
|---|---|---|
| `c8y_UnavailabilityAlarm` | **Cumulocity（自動）** | 設定不要 |
| `x_CameraDown` | thin-edge の監視サービス | 空 retained メッセージ（§6.6） |
| `x_BoxSilent` | BOXアダプタ | 同上 |
| `x_ModelUpdateFailed` | thin-edge の sm-plugin | 同上 |
| **`x_BoxParseError`** | **[要決定]** | デッドレター処理後の手動クリアか、**スマートルールによる時限自動クリア**か |
| **`x_Alarm_<種別>`** | **[要決定]** | 案件側が決定。**時限自動クリア（例: 24 時間）を既定にすることを推奨** |

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

### 8.5 重複排除の責務

| 事実 | 帰結 |
|---|---|
| **Cumulocity 側に重複排除機能はありません** | 責務を一元化する必要がある |
| **thin-edge にも重複排除機能はありません** | 同上 |
| BOX は「バッファ + 再送機能あり想定」（構成図） | **再送による重複が必ず発生する** |

**[判] 一次責務は BOXアダプタです**（[連携仕様] §4.5）。

| # | 規約 | 内容 |
|---|---|---|
| **DD-a** | **重複排除は BOXアダプタが行う**（複合キー + 永続ストア・仮 72h 窓） | 責務の一元化 |
| **DD-b** | **重複判定キーは external ID + `eventUuid` + `time`** | **`time` は BOX 側のイベント発生時刻**。再送時も同一値であることが前提（§6.9 TM-a） |
| **DD-c** | **Cumulocity 側（EPL）は「検知」に留め、除外の責務は持たない** | 二重に排除すると、どちらが落としたか分からなくなる |
| **DD-d** | ⚠️ **[要 TE-9]** BOX の送信仕様（プロトコル・シーケンス ID の有無・再送間隔・バッファ容量）が未確認 | **重複判定キーと保持ウィンドウ（仮 72h）が確定できない** |

> ⚠️ **Cumulocity サーバー側の de-duplicate は「ACTIVE 中の同一 `source` + `type`」を対象とし、severity は条件に含まれません**（[連携仕様] §3.5・§4.5）。これは**アラームにのみ働く仕組み**で、イベントの重複排除には使えません。**§6.5 の thin-edge 側の「型 + severity」規則とは別レイヤの話です**（§9.4）。

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
| **値の形式** | **`<SEVERITY>\|<TEXT>`**（パイプ区切り） | [確] |
| **効果** | アラーム型ごとに **severity とテキストを上書き**する | [確] |
| **抑止** | severity に **`NONE`** を指定するとそのアラームを抑止できる | [確] |
| **適用タイミング** | アラーム生成時 | [確] |

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

> ⚠️ **`x_Alarm_<種別>` は `<種別>` ごとに個別のキーになります。** 種別が案件から追加されるたびにマッピングの追加が必要です。**種別の追加手順を運用手順に含めてください。**

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
| **Cumulocity サーバー側の de-duplicate** | **ACTIVE 中の同一 `source` + `type`** が対象。**severity は条件に含まれない** | [連携仕様] §3.5・§4.5 |

**帰結**:

- thin-edge 側で severity を変えて再 publish すると、**thin-edge から見れば別のアラーム**になります。**旧 severity のアラームを別途クリアしないと 2 つ残ります**
- 一方 Cumulocity 側では同一 `source` + `type` として扱われる可能性があります

> ⚠️ **[要] この 2 つの規則の相互作用（severity を変えて再送した場合に Cumulocity 側で何が起きるか）は未検証です。** 型規約（§6.4）を確定する前に、検証環境で実際に確認してください（§14 CT-13）。

**[判] 暫定規約**: **アラームの severity を運用中に変えない設計にします。** severity を変えたい場合は、`alarm.type.mapping` で型ごとに固定し、デバイス側は常に同じ severity を送ります。**「軽度 → 重度への昇格」が必要な場合は、別の型のアラームとして扱ってください。**

### 9.5 抑止（`NONE`）の使いどころ

| ケース | `NONE` を使うか | 代替 |
|---|---|---|
| 検証中に大量発生するアラームを一時的に止めたい | ⚠️ **使わない** | 原因を直す。`NONE` にすると「直したつもりで実は抑止していただけ」になる |
| 特定のデバイスだけ止めたい | ❌ **使えない**（型単位の設定なのでデバイス指定できない） | メンテナンスモード（`responseInterval: 0`・§5.4） |
| 製品が自動生成するアラームで本構成では不要なもの | ✅ **候補** | 現時点で該当なし |

> **[判] `NONE` の使用は「恒久的に不要と判断した型」に限定し、使う場合は理由をコメントとして構成コードに残してください。** 抑止は「アラームが出ない」ことを意味し、切り分け時に極めて追いにくい設定です。

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
| `subscription` **(必須)** | トピック名。**パターン `^[a-zA-Z0-9]+$`** | **拠点コード**（§1.4・§2.5） |
| `subscriptionFilter.apis` | `alarms` / **`alarmsWithChildren`** / `events` / **`eventsWithChildren`** / `managedobjects` / `measurements` / `operations` / `*` | **`alarmsWithChildren`**（案件アプリ向け）／必要に応じて `eventsWithChildren` |
| `subscriptionFilter.typeFilter` | 型の完全一致、または `or` の OData 式 | **§6.4 の型で絞る**（例: `'x_Alarm_Intrusion' or 'c8y_UnavailabilityAlarm'`） |
| `fragmentsToCopy` | 指定した独自フラグメント**のみ**を含める | ⚠️ **生体情報・個人情報に関わるフラグメントを案件アプリへ渡さないために使う** |
| `nonPersistent` | boolean | **`false`（既定・永続）**。⚠️ `subscription` 名が同じでも `nonPersistent` が違えば**別トピック**になる |

**`alarmsWithChildren` は `source.id` の managed object と、その配下の全 descendant managed object のアラームを購読します** [確]。

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
| 3 | **`expiresInMinutes` の既定は 1440 分（24 時間）** [確] | **案件アプリ側にトークン再取得のロジックが必要**（**[要 SV-36]** 案件側との合意事項） |
| 4 | `shared: true` で共有コンシューマを作れる [確] | 案件アプリを冗長化する場合に使う |
| 5 | ⚠️ **実装主体が未確定**（TB-3） | **決めないと通知経路全体が止まる** |

> ⚠️ **このプロキシは §10.3 D-e と同じ思想です。Notification 2.0 だけロールを直接渡す例外を作らないでください。**

### 10.5 コンシューマのライフサイクル

| 事実 [確] | 帰結 |
|---|---|
| サブスクライバは**最初の WebSocket 接続時に生成**される | 事前作成は不要 |
| **一度作られたサブスクライバは、WebSocket が切断されても削除されない** | ⚠️ 停止した案件アプリの分のメッセージが**溜まり続ける** |
| Messaging Service は、消費されるか TTL に達するか**明示的に unsubscribe されるまで**メッセージを永続化する | ⚠️ **ディスク逼迫の経路**。単一 Edge に全拠点が集約されている本構成では影響が全拠点に及ぶ |
| 解除は `POST /notification2/unsubscribe?token=<token>` | 当該トピック・サブスクライバのトークンを作ってから呼ぶ |

**[判] 運用設計に必ず入れること**:

| # | 対応 |
|---|---|
| 1 | **廃止した案件アプリ・試験用サブスクライバの unsubscribe 手順**を runbook 化する（[担当範囲] RB-10） |
| 2 | **サブスクライバ数とバックログ量をメタ監視の対象にする**（**[要 SV-33]** Prometheus エンドポイントで取れるか未確認。**取れない場合は棚卸し運用が唯一の歯止め**） |
| 3 | **拠点撤収時に該当サブスクリプションを解除する**（§3.9） |
| 4 | **§7.6 の容量サイジングにバックログ分を含める** |

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

> ⚠️ **S-13（デバイス）と S-7/S-11/S-12 は SSO 対象外です** [確]。SSO を有効にした後も、これらは証明書またはローカル認証で動きます。**SSO 切替時に壊れないことを §14 CT-26 で確認してください。**

### 11.2 グローバルロール定義

**[判] 4 つのグローバルロールを定義します。**

| ID | ロール名 | 対象主体 | 含める権限 |
|---|---|---|---|
| **R-01** | **基盤運用者** | S-1 | Tenant Manager 相当 + Application management ADMIN + Retention rules ADMIN + CEP management ADMIN + Device control ADMIN + **Own user management READ** |
| **R-02** | **拠点オペレーター** | S-2 | Alarms ADMIN / Events READ / Inventory READ / Device control READ + **Own user management READ** |
| **R-03** | **業務閲覧者** | S-3 | Alarms READ / Events READ / Inventory READ + **Own user management READ** + **Cockpit アプリアクセス** |
| **R-10** 〈本書で追加〉 | **保守（AI モデル）** | S-4 | **ソフトウェアリポジトリ書込**（`ROLE_INVENTORY_ADMIN` + files 権限）+ Device control ADMIN + Device Management アプリアクセス + Own user management READ |

> ⚠️⚠️ **「Own user management」の READ が無いとログインできません。** SSO のアクセスマッピングで割り当てる**全ロールに必ず含めてください**。

> ⚠️ **API ロール名に `ROLE_INVENTORY_UPDATE` は存在しません** [確]。OAS に出現するのは `ROLE_INVENTORY_ADMIN` / `_CREATE` / `_READ` のみです。**UI の権限レベル（READ/CREATE/UPDATE/ADMIN）と API ロール名は 1:1 対応しない**ため、投入スクリプトでハマります。**実際の `ROLE_*` 名は投入前に `GET /user/roles` で確認してください。**

> ⚠️ **⭐ Cloud Remote Access のロールは、どのグローバルロールにも含めません**（§4.7 手段 4）。

### 11.3 Inventory ロール定義と割当

**Inventory ロールが拠点分離の実装点です。**

| ID | ロール名 | 権限 | 割当先 |
|---|---|---|---|
| **R-04a** | **拠点Manager** | Alarms ALL / Events READ / Inventory CHANGE / Measurements READ | 拠点オペレーター（S-2）× 担当拠点グループ |
| **R-04b** | **拠点Reader** | Alarms READ / Events READ / Inventory READ / Measurements READ | 業務閲覧者（S-3）× 担当拠点グループ |

| # | 規約 | 理由 |
|---|---|---|
| **IR-a** | **拠点グループに割り当てる。デバイス個別に割り当てない** | 親グループ → サブグループ → デバイスへ継承される [確] |
| **IR-b** | ⚠️ **手動割当は SSO の `inventoryMappings` に次回ログインで上書きされる** [確] | **割当は必ず `inventoryMappings` で行う**（§11.6） |
| **IR-c** | **投入は `c8y api` で `/user/inventoryroles` を直接叩く** | go-c8y-cli にトップレベルの `inventoryroles` サブコマンドが存在しない |
| **IR-d** | **冪等化は 409 黙殺**（[投入ガイド] パターン E） | 再実行しても重複しない |

> ⚠️ **Inventory ロールは Notification 2.0 には効きません**（§10.3）。**「Inventory ロールで拠点分離できている」と考えると、通知経路で穴が空きます。**

### 11.4 サービスユーザー

**[判] サービスユーザーはすべて SSO 対象外のローカルユーザーとし、資格情報は CI シークレットストアで管理します。**

| ID | 用途 | 対象主体 | 最小ロール | 備考 |
|---|---|---|---|---|
| **R-05** | VM2 アセット用 | S-6 | 最小限の READ | ⚠️ **原則としてエンドユーザー OIDC トークンを使う（D-e）。サービスユーザーはサーバサイド常駐が必要な場合のみ** |
| **R-06a** | **運用知識基盤（読取）** | S-8 | 拠点横断 READ（Alarms / Events / Measurements / Inventory） | **分析処理は常時こちらで稼働させる** |
| **R-06b** | **運用知識基盤（書込）** | S-9 | `ROLE_DEVICE_CONTROL_ADMIN` + `ROLE_INVENTORY_ADMIN` | ⚠️ **承認を伴う経路からのみ実行。§4.10 G-2 で拠点権限は一時付与** |
| **R-08** | **証明書アップロード / CA 操作** | S-11 | `ROLE_TENANT_ADMIN` または `ROLE_TENANT_MANAGEMENT_ADMIN` | ⚠️ **SSO ユーザーでは実行不可** [確]。§3.4 P-b |
| **R-09** | **トークン発行プロキシ** | S-7 | ⚠️ **`ROLE_NOTIFICATION_2_ADMIN` を持つ唯一の主体** | §10.4。**他の誰にも付与しない** |
| **R-11** 〈本書で追加〉 | **オフロードバッチ** | S-10 | 拠点横断 READ | §7.5 |
| **R-12** 〈本書で追加〉 | **クリップ保存サービス** | S-14 | Alarms READ + 通知購読用トークンの受領 | §10.2 NS-2 |

> **[判] R-06 を 2 つに分ける理由**: 単一アカウントだと**分析処理のバグが本番デバイスの設定を壊せる状態**になります。実装コストはユーザーを 1 つ増やすだけです。

> ⚠️ **サービスアカウントの復旧手段は「削除 → 同名で再作成」です**（§12.8）。**ユーザー ID が変わるため、ロール・Inventory ロール・アプリケーションアクセスの再割当が必要**になります。**投入スクリプトが冪等なら、再実行がそのまま復旧手段になります。**

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

> ⚠️ **リクエストボディのスキーマでは `id` が `readOnly` にオーバーライドされます。** エクスポート JSON から `id` を必ず除去してください [確]。

#### ⚠️ 必須ガード

| # | ガード | 内容 |
|---|---|---|
| **1** | ログインモードで SSO を選ぶと**ログイン画面から Basic Auth / OAI-Secure の選択肢が消える**。`management` テナントはローカル admin を維持 | [確] |
| **2** | **`edge` テナントにも break-glass（R-07）を残す。** ガード 1 は `management` しか守らない。⚠️ **他人のローカルユーザーのパスワードは管理者でも変更できない**（§12.8）ため、**「作っておく」だけでなく有効性を定期確認する** | [確] |
| **3** | 再割当ポリシーの既定は「毎ログインで全ロール再計算（ルール外はクリア）」。**手動編集は次回ログインで上書きされる** | [確] |
| **4** | どのルールにもマッチしないと `access denied`（デフォルト拒否）。**クレーム名の 1 文字違いで全ユーザーがログイン不可**になる | [確] |
| **5** | **デバイス認証とサービスユーザーは SSO 対象外** | [確] |
| **6** | **順序**: 先に Edge 側でロールを定義 → 対応するグループを Keycloak で作る | [判] |
| **7** | ⚠️ **[要 SV-04] SSO からローカル認証へ戻す手順を、実機で検証して runbook 化すること**。UI から選択肢が消えても `PUT /tenant/loginOptions/{typeOrId}` が Basic 認証の API で叩けるかが鍵 | **最優先** |
| **8** | ⭐ **切替前ゲート**: SSO ユーザー 1 名で実際にログインでき、期待するロールと Inventory ロールが付与されることを、**ログインモードを切り替えずに**確認する | [判] |

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
| **T-02** | ⭐ **CORS 許可オリジン** | テナントオプション `access.control` / `allow.origin`。**既定 `*` からの絞り込み**。**VM2 の全アプリ（紐づけ確認アプリを含む）のオリジンをスキーム + ホスト + ポートで列挙** |
| **P-03** | 標準ダッシュボード | Cockpit の JSON エクスポート/インポート。**[要 SV-11]** Edge での可否は未検証 |

> ⚠️ **CORS のオリジン一覧が未確定だと、アプリ側が「認証は通るが API が呼べない」状態になります。** VM2 の各アプリ担当からオリジンの完全なリストを受領してください。**ポート番号まで含めた完全一致**です。

> ⚠️ **運用知識基盤（S-8/S-9）はサーバサイド常駐プロセスなので CORS は不要です。** ブラウザから叩くアプリだけが対象です。

### 11.9 API 利用規約

**[判] Cumulocity の REST を叩く全主体が守る規約です。IF 仕様書として配布します。**

| # | 規約 | 理由 |
|---|---|---|
| **AP-a** | ⭐ **ページングは `prev` / `next` リンクで辿る。ページ番号を自前で加算しない** | `acl.algorithm-version` の `OPTIMIZED` は一致件数が閾値（既定 2000）未満のときだけ適用され、超えると `LEGACY` に落ちる [確K]。*"navigation links via 'prev' and 'next' will work properly and this should be the only way of iterating through multiple pages"* |
| **AP-b** | **エンドユーザーの OIDC トークンを使う**（D-e） | §11.7 AX-2 |
| **AP-c** | **Operational Store への直接アクセスは行わない**（REST API 経由のみ） | 構成図の配置の要点 |
| **AP-d** | **通知トークンは 24 時間で失効する。再取得ロジックを実装する** | §10.4 要件 3 |
| **AP-e** | **c8y-proxy 経由の呼び出しは散発的に 401 を返す。リトライを実装する** | [確]（§6.8 制約 4） |
| **AP-f** | **`type` フィルタ・`fragmentType` フィルタで絞ってから取得する** | 全件走査は Edge の負荷になる（§7.7） |

### 11.10 否定テストによる検証

**[判] 権限設計は「できること」ではなく「できないこと」を検証してください。**

| # | 否定テスト | 合格条件 | §14 |
|---|---|---|---|
| 1 | 拠点 A のユーザー資格情報で拠点 B のデータを **API 直叩き** | 403 / 404 | CT-5 |
| 2 | 拠点 A の案件アプリの認証情報でトークン発行プロキシに**拠点 B のサブスクリプション**を要求 | 拒否される | CT-7 |
| 3 | 拠点 A 用トークンで拠点 B のトピックに WebSocket 接続 | 接続できない | CT-7 |
| 4 | `ROLE_NOTIFICATION_2_ADMIN` の付与先を機械検査 | R-09 以外に付いていない | CT-6 |
| 5 | 手動付与した Inventory ロールが次回ログインで消えること | 消える（＝ `inventoryMappings` が正） | CT-26 |
| 6 | Cumulocity UI に Remote access のボタンが出ないこと | 出ない | CT-17 |
| 7 | 拠点 B の案件アプリに拠点 A のアラームが**届かない**こと | 届かない | CT-8 |
| 8 | `fragmentsToCopy` で除外したフラグメントが通知に**含まれない**こと | 含まれない | CT-8 |

---

## 12. Cumulocity Edge セットアップ

> **本章は「設置から設定投入を始められる状態まで」を扱います。設定項目そのものは §13 です。実施手順の詳細は [設定書] §5 P0〜P1 が正です。**

### 12.1 セットアップの全体像

```
①  ネットワーク前提の確定  ── install 後に直せない（§12.2）
②  ドメイン 2 つ + TLS 証明書 + ライセンス + registry credentials の受領
③  容量サイジング（§7.6）── storageClassName は install 後変更不可
④  検証環境 Edge の構築  ── 前提 2〜4（§0.7）をここで潰す
⑤  本番 Edge のインストール  ── 冪等化不可・1 回きり
⑥  閉域網固有の設定（CA / no_proxy）
⑦  マイクロサービス稼働確認（Apama-ctrl / Smartrule / Messaging Service）
⑧  admin パスワード変更 + エスクロー保管
⑨  スナップショット取得 + リストア試験  ← ここを通るまで §13 に進まない
──────────────────────────────────────────
    §13 設定投入へ
```

### 12.2 ⚠️ install 後に変更できない項目

**これらは `c8yedge install` の後では変更できません。⑤に進む前に必ず潰してください。**

| # | 項目 | 内容 | 確度 |
|---|---|---|---|
| **P-0-1** | **CIDR 衝突** | K3s の Service / Pod CIDR と社内網アドレス空間の重複確認。**オンプレ閉域網は 10.0.0.0/8 が一般的で衝突しやすい**。衝突すると「Web UI は動くが SSO だけ落ちる」のような散発的症状が出る | [推] |
| **P-0-2** | **公開ポート** | `cumulocity-ontoplb`（LoadBalancer）: **443, 8443, 1883, 8883**。Edge operator メトリクス: **3443**。*"Edge requires that your Kubernetes cluster does not have an Ingress provider ... enabled on common ports"* | [確] |
| **P-0-3** | **時刻同期（NTP）** | 社内 NTP を Edge ホスト・Keycloak・画像解析装置・外部Gateway に設定（§6.9 TM-c） | [推] |
| **P-0-4** | **到達性マトリクス** | Edge → 社内 DNS / Keycloak(JWKS) / オブジェクトストレージ / Genetec、拠点網 → Edge:8883,443、**案件アプリ(VM2) → Edge の WebSocket**（§10.1）、保守VPN → Edge | [推] |
| **P-0-5** | **ハードウェア最小要件** | CPU **8 コア** / RAM **16 GB** / Disk **150 GB**。**MongoDB は AVX 命令 + x86-64-v3 以降が必須**（`lscpu` で確認） | [確] |
| **P-0-6** | ⭐ **`storageClassName` の確定** | *"Once the `storageClassName` field is configured in the Edge custom resource (CR), it cannot be changed."* **§7.6 のサイジングを先に済ませる** | [確] |
| **P-2** | **ドメイン 2 つと TLS 証明書の SAN** | §12.3 | [確] |
| **P-3** | **ライセンス**（ドメインに紐づく） | *"the license key must always be valid for the domain name, so any change of domain name should be made simultaneously with a change of license key."* | [確] |

> ⚠️⚠️ **ドメイン変更は install 後に行わないでください。** TLS 証明書の再発行、Keycloak の redirect URI / backchannel logout URL、**全 thin-edge の `c8y.url` と `c8y.mqtt`**、全ブラウザの信頼設定を巻き込みます。**デバイス接続開始後に気づくと拠点に人を出す作業になります。**

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
| — | **デバイス側の信頼ストア** | thin-edge の `c8y.root_cert_path`（MQTT）と **`c8y.proxy.ca_path`（HTTP）は別キー**。**両方を明示設定しないと「MQTT は繋がるが添付だけ失敗する」** | [確] |

> ⚠️ **証明書・社内 CA のローテーション手順が必要です**（SV-22）。社内 CA ルートを更新すると **thin-edge 側の信頼ストア更新が必要**で、数百台 × N 拠点が同時に接続不能になり得ます。しかも**配布経路（Cumulocity 経由のソフトウェア更新）も同時に使えません**。**新旧 CA を並行して信頼させる移行期間**を設計に入れてください。

### 12.6 バックアップ／リストア

| 形態 | 保全対象 | 復元手順 |
|---|---|---|
| **`c8yedge`（K3s）** | **`/var/lib/rancher/k3s`** | 同一（または互換）OS を用意 → データ復元（**パス・所有者・権限を保持**）→ `sudo c8yedge install --version <previous_version>`（オフラインは `-s c8yedge.tar`） |
| 自前 K8s | **PVC と Edge CR** | 同上の考え方 |

> ⚠️ *"Installing a different Edge version on top of a restored data set is unsupported and may fail the upgrade guard rails."* **復元時は同一バージョンが必須です** [確]。

> ⚠️⚠️ **D17 により全拠点が 1 台の Edge に集約されています。** ディスク障害・MongoDB 破損・誤配信のいずれでも**全拠点が同時に停止**します。
>
> - **リストア試験の成功を §13 に進むゲートにしてください**（§12.4 確認 7）
> - **[要 SV-29] RTO/RPO を数値で決め**、BOX と thin-edge のバッファ保持時間を実測して「Edge 停止許容時間」として明記してください

### 12.7 Edge 自身の監視

> *"The Edge operator exposes a **Prometheus-compatible metrics endpoint, `https://<domain>:3443/metrics`**"* [確]

| # | 設計 | 内容 |
|---|---|---|
| **1** | **Otel Collector の prometheus receiver で取り込む** | OTLP ネイティブ出力は未記載。これが正攻法 |
| **2** | ⭐ **アラート通知先を Cumulocity の外側に持つ** | **変-2 でメールが無いため、これが基盤異常の唯一の検知経路**（SV-34 / TB-5） |
| **3** | **監視対象に含めるもの** | ①Edge 自体の死活 ②`c8y_Application_Down` / `_Unhealthy` ③**デバイス証明書の有効期限**（§3.6）④**Notification 2.0 のサブスクライバ数・バックログ量**（**[要 SV-33]** 取れるか未確認）⑤ディスク使用率 ⑥PENDING オペレーション件数（§4.11） |
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
| 1 | **削除 → 同名で再作成** | **サービスアカウント**（R-05, R-06, R-08, R-09, R-11, R-12） | ユーザー ID が変わるため**ロール・Inventory ロール・アプリアクセスの再割当が必要**。**投入スクリプトが冪等なら再実行で復旧できる** |
| 2 | **admin が複数居る状態を保つ** | 両テナントの管理者 | ただし同じ制約を受けるため、**admin 自身の資格情報は削除・再作成では救えない** |
| 3 | **資格情報のエスクロー保管** | **両テナントの admin と break-glass のみ** | §11.5 |

> ⚠️ **[要 SV-37] Edge のホスト側から admin パスワードを復旧する手段があるか**を検証環境で試す価値が高いです。初期パスワードは K8s Secret `cumulocityPasswordSecretName` 由来ですが、**インストール後に Secret を変更してパスワードが変わるかは未確認**です。**成立すればエスクローの必要性が下がります。**

### 12.9 セットアップ完了判定

**次がすべて ✅ になるまで §13 の設定投入に進まないでください。**

- [ ] §12.2 の P-0-1〜P-0-6 / P-2 / P-3 が確定し、記録されている
- [ ] §7.6 の容量サイジングが完了し、`storageClassName` と MongoDB 容量に反映されている
- [ ] 両テナント URL でログイン可能
- [ ] Edge CR がエクスポートされ Git にコミットされている
- [ ] 両テナントの admin パスワードが変更され、**エスクロー保管されている**
- [ ] 社内 CA ルートが全ブラウザ・Edge operator に配布されている
- [ ] `c8yedge-operator-config`（`ca.crt` / `no_proxy`）が適用され operator が再起動されている
- [ ] Apama-ctrl / Smartrule が稼働している
- [ ] ⭐ **Messaging Service が実際に動作している**
- [ ] ⭐ **スナップショット取得とリストア試験が成功している**
- [ ] 検証環境で §0.7 の前提 2〜4 に結論が出ている

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
| **CF-c** | ⚠️ **`C8Y_MODE=ci` を使う。`prod` は使わない** | **`--mode prod` は Create/Update/Delete が全て無効**で、スクリプトが**エラーも出さずに何も投入せず完走する** [確] |
| **CF-d** | **全コマンドに `--session <name>` を明示する** | management / edge の 2 セッションを取り違えると、投入先を間違える |
| **CF-e** | **`--dry --dryFormat json` の出力に `Authorization: Basic` ヘッダが含まれ得る** | CI ログに残さない |
| **CF-f** | **L0 / L1-B / L1-C の分離を崩さない** | **この分離ができているかが、2 案件目を `site/` だけの作業で立ち上げられるかを決める** |

### 13.2 ⭐ 定義 → 設定項目 トレース表

**本書の各定義章が、どの設定項目になるかの対応です。この表が §13 の索引です。**

| 定義（章） | 設定項目 ID | 設定対象 | 分類 | 詳細 |
|---|---|---|---|---|
| §1 デバイス管理グループ | **G-01** 拠点 device group 階層 | edge | L1-C | §13.3.3 |
| §1.4 グループの external ID | **G-02** グループの external ID | edge | L1-C | §13.3.3 |
| §1.5 グループのフラグメント | **G-04**〈新〉 グループの `x_Site` | edge | L1-C | §13.3.3 |
| §2 命名規約 | — | **設定ではなく規約文書 + バリデータ** | L1-B | §13.3.12 |
| §2.8 機械検査 | **CK-1〜CK-6** | CI | — | §13.7 |
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
| **G-01** | 拠点 device group 階層 | **root（`sites`）→ 拠点グループ（`site001`…）の 2 段**（§1.2 / §1.7 G-e） | `c8y inventory create` + `childAssets` 追加、または一括登録 CSV の `PATH`（§3.3） | **E: 409 黙殺** |
| **G-02** | グループの external ID | **拠点コード**（英数字のみ・§1.4） | `c8y identity create` | **E: 409 黙殺** |
| **G-03** | Inventory ロールの割当 | **SSO の `inventoryMappings` で管理**（§11.3 IR-b） | §13.3.11 の `authConfig` | 上書き |
| **G-04**〈新〉 | グループの `x_Site` フラグメント | `{ "siteId": "site001", "siteName": "…" }` | `c8y inventory update` | **C: 存在チェック** |

**拠点定義ファイルの例**（`config/site/sites/site001.yaml`）:

```yaml
siteId:    site001                  # 英数字のみ（§2.5）
siteName:  "〇〇工場 東棟"           # 表示名（日本語可）
devices:
  imageAnalyzer:
    serial:  SN00123                # → extID: site001:ANLZ-SN00123
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
| **A-05** | パスワードポリシー / TFA | **管理者アカウントの有効期限は無期限**（§11.5 統制 3） | `c8y tenantoptions updateBulk`（`password` カテゴリ） | 上書き |

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
| **D-09** | アラームストーム抑止 | child に `c8y_RequiredAvailability = {"responseInterval": 0}` | プロビジョニングツール（twin） | **C: 存在チェック** |
| **D-12**〈新〉 | ⚠️ **`autoRegistrationEnabled` の無効化** | **方式 (2) を使った場合のみ**。一括登録完了後に必ず無効化 | `c8y api PUT /tenant/tenants/{t}/trusted-certificates/{id}` | 上書き |

> ⚠️ **D-10 は前提 3（SV-08 / TE-8）に依存します。** NG の場合は方式 (2)（自己署名 CA + trusted certificate）に切り替え、**D-12 の無効化を必ずセットで実施**してください。

#### 13.3.6 データ保持（X 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **X-01** | リテンションルール | `EVENT` / `MEASUREMENT` / `ALARM` / `OPERATION` = **90 日**、**`AUDIT` = [要決定]** | `c8y retentionrules create` | ⚠️ **D: 宣言的な集合適用（新規作成 → 旧削除の順）** |
| **X-02** | イベント添付バイナリ | **[要 SV-06]** リテンションが及ぶか未確認 | — | — |
| **X-03** | オフロード | **REST ページング方式（案 B）**。`prev`/`next` で辿る | 外部バッチ（R-11 の資格情報） | — |

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

# ③ 旧ルール（既定 60 日）を個別に削除
```

> ⚠️⚠️ **順序を守ってください。** 削除が先だと、作成に失敗した場合に**リテンションルールがゼロの状態で放置**されます（§7.2）。

> ⚠️ **`AUDIT` の日数が未決のままこの手順を実行しないでください。**

#### 13.3.7 ルール（S 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **S-01** | スマートルール | §8.2 RU-1 / RU-2 | ⚠️ **[要 SV-32]** managed object の `type` / `fragmentType` の実値が未確定。**GUI で 1 つ作成 → `c8y inventory find` で観測してから投入スクリプトを書く** | **C: 存在チェック** |
| **S-02** | EPL apps | §8.2 RU-3 / RU-4 / RU-5 | `c8y api POST /service/cep/eplfiles`。**ソースごと Git 管理** | **B: PUT 先行 → 404 なら POST** |
| **S-03** | アラーム自動クリア | §8.3 | スマートルールまたは EPL | 同上 |
| **S-04** | **イベント/アラーム型・ペイロード規約** | §6.4 / §6.5 | ⚠️ **設定ではなく規約文書 + バリデータ**（§13.3.12） | — |

> ⚠️ **前提**: **Apama-ctrl / Smartrule マイクロサービスへの購読**が必要です（§12.4 確認 5）。

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

> ⚠️ **`source` にはグループの MO ID（グローバル ID）が必要**なので、**G-01 / G-02 の投入後**にしか実行できません（§13.4）。

#### 13.3.9 リポジトリ系（M / K 系）

| ID | 設定項目 | **[判] 設定値** | 設定手段 | 冪等 |
|---|---|---|---|---|
| **M-01** | ソフトウェアリポジトリ | `softwareType = aimodel`（§4.4） | `c8y software create` / `c8y software versions create` | **C: 存在チェック** |
| **M-02** | モデル配布オペレーション | `c8y bulkoperations create`（**`creationRamp` 必須**・§4.8） | CLI | — |
| **M-03** | モデルの保持世代 | **[要 TE-20]** 保持世代数と削除運用 | 運用手順 | — |
| **K-03** | 設定リポジトリ | 配布対象が確定してから（**[要 SV-17]**） | `c8y configuration create` / `send` | **C: 存在チェック** |
| **K-04** | デバイスプロファイル | 型ごとの「あるべき設定の束」。**[要 SV-17]** | `c8y deviceprofiles create` | 同上 |
| **K-01〜K-08** | 運用知識基盤の統制 | **§4.10 のガードレール G-1〜G-14** | 運用手順 + R-06 の権限設計 | — |

> ⚠️ **`files` リポジトリにはリテンションが適用されません** [確]。ソフトウェアリポジトリは MongoDB 格納のため、**保持世代数と削除運用を決めないと肥大化します**（M-03）。

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
| **A-05** | パスワードポリシー / TFA | §13.3.4 | `c8y tenantoptions updateBulk` | 上書き |

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
| **V-2** | **インベントリ検査**（CK-1〜CK-6） | §2.8 | `scripts/assert-inventory.*` |

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
④ 拠点グループ階層（G-01）→ external ID（G-02）→ x_Site（G-04）
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

### 13.5 冪等化パターンの割り当て

| パターン | 内容 | 適用先 |
|---|---|---|
| **A: 名前ベース参照** | `--group` / `--role` がワイルドカードを受ける | ロール割当（R-01〜R-03, R-10） |
| **B: PUT 先行 → 404 なら POST** | 更新優先 | `loginOptions`（A-01）、EPL apps（S-02） |
| **C: 存在チェック → 分岐** | GET してから作成 | ユーザー（R-05〜R-12）、グローバルロール定義、ソフトウェア（M-01）、サブスクリプション（N-01）、child device（D-07） |
| **D: 宣言的な集合適用** | ⚠️ **新規作成 → 旧削除の順** | リテンション（X-01） |
| **E: 重複エラー黙殺（409）** | 作成を試みて 409 を無視 | Inventory ロール定義（R-04）、グループ（G-01/G-02）、`childAssets` 追加（D-08）、テナントのアプリ購読（P-02） |

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
    users.json                              # R-05, R-06, R-11, R-12
    deviceprofiles/*.json                   # K-04
  mgmt/                              # management テナント管轄
    branding/
binaries/                            # ZIP は Git LFS または成果物リポジトリ
secrets/                             # 値は入れない。キー名の一覧のみ
scripts/
  export.sh   import.sh
  assert-config.sh                   # §13.7 往復差分
  assert-inventory.sh                # ★CK-1〜CK-6（§2.8）
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
| **AS-4** | ⭐ **Cloud Remote Access ロールの検査** | **どのロール・どのユーザーにも付いていない** | R-13。CI |
| **AS-5** | ⭐ **external ID の規約準拠** | 規約外の MO が 0 件 | CK-1。CI |
| **AS-6** | device type の規約準拠 | 4 種以外が 0 件 | CK-2。CI |
| **AS-7** | `vmsCameraId` の設定 | 未設定が 0 件 | CK-3。CI |
| **AS-8** | main device の拠点グループ所属 | 未所属が 0 件 | CK-4。CI |
| **AS-9** | external ID プレフィックスと実所属拠点の一致 | 不一致が 0 件 | CK-5。**日次** |
| **AS-10** | child device のフラグメント検査 | `c8y_SupportedOperations` と `com_cumulocity_model_Agent` が付いていない | CI |
| **AS-11** | リテンションルールの存在 | 5 種すべてが存在し、既定 60 日ルールが残っていない | CI |
| **AS-12** | サブスクリプション定義 | 拠点数ぶん存在し、`fragmentsToCopy` が指定されている | CI |

> ⚠️ **AS-3 / AS-4 / AS-5 は「壊れても長期間気づけない」種類の破綻です。CI に載せることが唯一の歯止めです。**

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
| **CT-4** | ⭐ **拠点グループ購読**（前提 2 / SV-35 の実証） | 拠点グループを `source` にしたサブスクリプションで、**配下デバイス（child を含む）のアラームが届く**。**届かない場合は §1.6 の代替案 (i) に切り替える** |
| **CT-5** | ⭐ **Inventory ロールの否定テスト** | 拠点 A のユーザー資格情報で拠点 B のデータを **API 直叩き**して 403/404 |
| **CT-6** | ⭐ **`ROLE_NOTIFICATION_2_ADMIN` の機械検査** | **R-09 以外のどのユーザー・ロールにも付いていない**（CK-6・**CI に載せる**） |

### 14.3 通知

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-7** | ⭐ **トークン発行プロキシの否定テスト** | ①拠点 A の案件アプリの認証情報で**拠点 B のサブスクリプション**を要求 → 拒否される ②拠点 A 用トークンで**拠点 B のトピックに WebSocket 接続できない** |
| **CT-8** | ⭐ **通知の到達試験** | 拠点 A でアラームを発生させ、①**拠点 A の案件アプリに届く** ②**拠点 B の案件アプリには届かない** ③**`fragmentsToCopy` で除外したフラグメントが含まれない** |

### 14.4 死活監視

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-9** | ⭐ **child のアラーム抑止**（§5.3 の落とし穴） | **接続から 1 時間以上経過しても、カメラ・BOX に `c8y_UnavailabilityAlarm` が 1 件も出ない** |
| **CT-10** | **死活監視サービスの死** | サービスを kill → **Cumulocity 上でサービスが `down` になる**（LWT）→ watchdog が再起動 → **起動時にアラーム状態が再評価される** |
| **CT-11** | ゲートウェイ断 | 通信停止 → `responseInterval` 後に `c8y_UnavailabilityAlarm` → 復旧で**自動クリア** |
| **CT-12** | カメラ断 | 無応答 N 回で `x_CameraDown` → 復旧でクリア。**ヒステリシスが効きフラッピングしない** |

### 14.5 データとルール

| # | 試験 | 合格条件 |
|---|---|---|
| **CT-13** | **severity 変更時の挙動**（§9.4 の 2 層の一意性規則） | severity を変えて再 publish したときに Cumulocity 側で何が起きるかを観測し、**結果を規約に反映する**（型規約の確定前に実施） |
| **CT-14** | **ペイロード規約の検証** | ①16KB 超のイベントが HTTP に切り替わることを確認 ②16KB 超の**計測が拒否される**ことを確認 ③バリデータが規約違反を検出する |
| **CT-15** | ⭐ **リプレイ抑止** | 網断復旧の一括再送で、**通知が復旧時刻にまとめて発火しない**（ルール雛形のリプレイ抑止ガードが効いている）。**イベント自体は全件 Operational Store に残っている** |

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
| **CT-32** | ⭐ **テナント CA 自動更新（10/2）のシミュレート**（SV-25） | 時計を進めて再エンロールが成功する。**複数デバイス同時実行のケースも試す**（§3.7 対応 5） |
| **CT-33** | **CORS 検証** | VM2 の各アプリから実際にブラウザ経由で Edge の REST を叩き 200 が返る |
| **CT-34** | **設定の往復差分** | **Git 定義 vs 実機の差分ゼロ**（AS-1） |

### 14.12 検証と未確定事項の対応

**⭐ 項目のうち、「NG だと設計そのものが変わる」ものを先に実施してください。**

| 優先 | 試験 | NG だと何が変わるか |
|---|---|---|
| **1** | **CT-4**（拠点グループ購読） | **§1 の拠点分離方式と §10 の購読設計が同時に破綻**。§1.6 の代替案へ |
| **2** | **CT-32 / 証明書の EST 可否**（前提 3） | §3.3 の登録方式が変わり、§3.6 の自動更新を別途設計 |
| **3** | **CT-8**（通知の到達） | §10 が成立しない。**変-2 と併せて通知経路がゼロ**になる |
| **4** | **CT-13**（severity の 2 層規則） | §6.4 の型カタログと §9 のマッピングが変わる |
| **5** | **CT-9**（child のアラーム抑止） | 対処方針 2 が効かない場合、§5.4 の代替案 3（`@health`）へ。**アラーム型規約が変わる** |
| **6** | **CT-28**（合算規模） | 拠点数の上限・死活間隔・Edge のリソース設計が変わる |

---

## 15. 未確定事項・前提条件

> **ID 体系**: 本書で新たに提起するものは **`CU-nn`**（Cumulocity Undetermined）。既存の `SV-nn`（[設定書]）・`TE-nn` / `TB-nn`（[担当範囲]）・`V-nn`（vendor-questions）との対応を併記します。

### 15.1 本書の前提が崩れるもの（最優先）

| # | 項目 | 確認方法 | 影響 | 既存 ID |
|---|---|---|---|---|
| **前提 2** | **`alarmsWithChildren` が `childAssets` を辿るか** | 検証環境（CT-4） | **§1 の拠点分離方式と §10 の購読設計が同時に破綻** | **SV-35 / TE-12** |
| **前提 3** | **certificate-authority（EST）が Edge で使えるか** | 検証環境 + ベンダー照会 | §3.3 の登録方式と §3.6 の証明書更新自動化が変わる | **SV-08 / TE-8 / V8** |
| **前提 4** | **Edge 上で Notification 2.0 が実動作するか**（WebSocket が LoadBalancer 経由で到達できるか） | 検証環境（CT-8） | **§10 が丸ごと成立しない。通知経路がゼロになる** | **SV-05** |
| **前提 1** | **外部Gateway の配置方式（案 α / β）** | 設計判断 | §1.2 の階層図・§6.2 のデータ経路・§10.2 の購読設計 → **付録 A** | [担当範囲] W0-10 |
| **前提 5** | **型規約・external ID 規約の関係チームとの合意** | 合意プロセス | §7.2・§8・§9・§10.2 が全部やり直し | [担当範囲] C-1 / C-2 |

### 15.2 値が未確定のもの（着手前に埋めるべき具体値）

| # | 項目 | 該当章 | なぜ最初か | 決定主体 | 既存 ID |
|---|---|---|---|---|---|
| **CU-1** | **拠点コードの形式**（案 (a) 英数字統一 / 案 (b) 2 系統） | §1.4 / §2.5 | グループ external ID・サブスクリプション名・topic id の全てに効く。**後から変えると全拠点の再登録** | 基盤 × 案件 | — |
| **CU-2** | **external ID の仮名化要否** | §2.2 | **クラウドへ出る特徴量に紐づく個人情報上の論点**。仮名化するなら採番形式そのものが変わる | **法務 / 顧客合意** | [担当範囲] C-2 |
| **CU-3** | **デバイス撤去時の物理削除 / 論理削除** | §3.9 | 物理削除すると過去イベントの `source` が欠損し、オフロード済みデータと突合できなくなる | 基盤 | — |
| **CU-4** | **`x_BoxParseError` と `x_Alarm_<種別>` のクリア条件** | §9.6 | **クリア条件のないアラームは永久に消えない**（§7.3） | 基盤 × 案件 | — |
| **CU-5** | **リテンション `AUDIT` の日数** | §7.2 / §13.3.6 | **既定ルールを消すと再作成すべき値が必要**。決まらないとリテンション投入自体ができない | 顧客契約 / 監査要件 | [設定書] K-c / G-13 |
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
| **CU-13** | **severity 変更時の 2 層の一意性規則の相互作用** | 検証環境（CT-13） | **高**。型規約の確定前に必要 | — |
| — | **運用知識基盤の「設定の更新」対象に画像解析装置・カメラを含めるか** | 設計判断（実機不要） | **最高**。含めないなら K-03〜K-05 / R-06 書込側が丸ごと不要になりリスクが激減 | **SV-17** |
| — | **カメラ（child device）への構成オペレーションが成立するか** | 検証環境 | 中。⚠️ **§4.3（child に何も申告しない）と両立しない**（§4.5） | **SV-28** |
| — | イベント添付バイナリがリテンションの対象か | 検証環境（CT-22） | 高 | **SV-06** |
| — | 一括登録 CSV で `ENROLLMENT_OTP` と `PATH` を併用できるか | 検証環境 | 高 | **SV-07** |
| — | `PUT /tenant/options/{category}` のマージ／置換セマンティクス | 検証環境 | **高**。置換なら再実行で設定が壊れる | **SV-30** |
| — | スマートルールの managed object の `type` / `fragmentType` | 検証環境で GUI 作成 → 観測 | **高**。実値が分からないと投入スクリプトが書けない | **SV-32 / U-01** |
| — | Notification 2.0 のサブスクライバ数・バックログ量を監視できるか | Prometheus エンドポイント | 高。**取れない場合は棚卸し運用が唯一の歯止め** | **SV-33** |
| — | 紐づけ確認アプリを EXTERNAL 登録にするか Cumulocity にホストするか | 設計判断 | 高 | **SV-24** |
| — | Cockpit の Import/export が Edge で使えるか | 検証環境 | 中 | **SV-11** |
| — | `files/max.size` の実値 | `GET /tenant/system/options` | 中 | **SV-13 / TE-16** |
| — | BOXアダプタのマッピング定義を Cumulocity 経由で配るか Ansible か | 設計判断 | 中 | **SV-27 / TB-2** |
| — | `bulkoperation.creationramp` の意味（既定値か下限か） | ベンダー照会 | 低（G-4 が主統制） | **SV-19** |

### 15.4 運用に効くもの

| # | 項目 | 確認方法 | 優先度 | 既存 ID |
|---|---|---|---|---|
| — | **テナント CA 自動更新（毎年 10/2）時の再エンロール動作** | 検証環境でシミュレート（CT-32） | **最高** | **SV-25 / TE-7** |
| — | **SSO からローカル認証へ戻す手順** | 実機検証 → runbook 化（CT-27） | **最高** | **SV-04** |
| — | **全拠点合算のスケール** | 負荷試験（CT-28） | **最高** | **SV-14** |
| — | **RTO/RPO、BOX・thin-edge のバッファ保持時間** | 実測 + 設計判断 | **最高** | **SV-29 / TE-10** |
| — | **基盤自身の異常のアラート通知先** | 設計 + 担当決定 | **高**。⚠️ **メール不採用のため唯一の検知経路** | **SV-34 / TB-5** |
| — | **トークン発行プロキシ（N-02）の実装主体** | 担当決定 | **高**。決めないと通知経路全体が止まる | **TB-3** |
| — | **オフロードバッチ（X-03）の実装主体** | 担当決定 | **高**。⚠️ **稼働開始から 90 日以内が期限** | **TB-4** |
| — | Edge のホスト側から admin パスワードを復旧する手段 | 検証環境 | 高。成立すればエスクローの必要性が下がる | **SV-37** |
| — | 証明書・社内 CA ローテーション手順（新旧並行の移行期間） | 設計 + 検証環境 | 高 | **SV-22** |
| — | メタ監視の向き（D9 との整合） | 設計判断 | 中 | **SV-23** |
| — | 案件アプリのトークン再取得ロジック | 案件側との合意 | 中 | **SV-36** |
| — | 運用知識基盤が必要とする分析データの期間 | 設計判断 | 高。**90 日超ならオフロードが唯一のデータ源** | **SV-20** |

### 15.5 文書管理上の未処理事項

| # | 項目 | 内容 |
|---|---|---|
| **CU-14** | ⚠️ **[設定書] §4 の移管作業** | 本書 §0.3 の移管表に従い、[設定書] §4 を削除して本書への参照に置き換える。**それまでは記述が二重に存在する**（齟齬があった場合は本書が優先） |
| **CU-15** | **[連携仕様] への反映** | 本書で新規定義した `x_Site` / `x_SensingBox` を [連携仕様] §4.3 に反映する |
| **CU-16** | **[担当範囲] との相互参照の整理** | [担当範囲] §4（Cumulocity Edge 担当作業）が本書を参照する形に更新する |
| **CU-17** | **項目 ID 体系の統一** | 本書で追加した ID（R-10〜R-13 / G-04 / D-10〜D-12 / M-03 の一部）を [設定書] §3 の体系に正式に吸収する。**放置すると [設定書] 側が別の内容で同じ ID を採番した場合に衝突する** |

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
| **α-1** | ⚠️ **BOX 全台を個別に `childAssets` へ追加する工程が必要** | 案 β は拠点あたり 2 回で済むが、**案 α は BOX の台数分すべて**。`POST /inventory/managedObjects/{siteGroupId}/childAssets`。冪等化は 409 黙殺 |
| **α-2** | **BOXゲートウェイ本体の配置** | GW はどの拠点にも属せないため、**共通グループ（`common`）に置く**（§1.2） |
| **α-3** | §1.3 の規約「拠点グループへの編成は main device に対してのみ」が**成立しない** | BOX（child）を個別に編成する必要があるため、**§3.10 の検査 4（CK-4）を BOX にも拡張**する |

### A.3 §5・§9（死活・アラーム）への差分

| # | 差分 | 内容 |
|---|---|---|
| **α-4** | ⚠️ **GW 停止時に全拠点の BOX が同時に影響を受ける** | §5.4 の `responseInterval: 0` は前提として、**`x_BoxSilent` を EPL で拠点単位に集約する**ことを検討（§8.2 RU-5 が**必須になる**） |
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
- [Cumulocity構成準拠セットアップ設定書.md](Cumulocity構成準拠セットアップ設定書.md) — **適用手順（P0〜P7）の正**。⚠️ §4 は本書へ移管（§0.3 / CU-14）
- [Cumulocity設定定義書.md](Cumulocity設定定義書.md) — 設定項目の網羅カタログ。⚠️ [設定書] §0.2 の訂正が優先
- [Cumulocity設定エクスポート投入ガイド.md](Cumulocity設定エクスポート投入ガイド.md) — go-c8y-cli の実装技法・冪等化パターン A〜E
- [../../IoTPlatform_cc/data-integration-spec_new.md](../../IoTPlatform_cc/data-integration-spec_new.md) — IF-nn の正
- [../../IoTPlatform_cc/architecture-camera-monitoring.md](../../IoTPlatform_cc/architecture-camera-monitoring.md) — デバイスモデル・ペイロード規約の原典
- [../../IoTPlatform_cc/design-decisions.md](../../IoTPlatform_cc/design-decisions.md) — D1〜D17
- [../../IoTPlatform_cc/vendor-questions-cumulocity.md](../../IoTPlatform_cc/vendor-questions-cumulocity.md) — V1〜V20
- [cumulocity-iot-architecture.drawio](cumulocity-iot-architecture.drawio) — **「全体構成(配置構成・レビュー反映)」タブのみ**

### Cumulocity Edge

| 内容 | URL |
|---|---|
| インストール・2 テナント・TLS 要件・ライセンス・registry credentials・前提要件 | `https://cumulocity.com/docs/2026/edge/installing-edge/` |
| **機能比較表（`Included` の一覧）・100 tps/CPU コア** | `https://cumulocity.com/docs/2026/edge/edge-introduction/` |
| メールサーバー・外部 IP とポート一覧・`c8yedge config` | `https://cumulocity.com/docs/2026/edge/manage-edge/` |
| **バックアップとリカバリ・Prometheus メトリクスエンドポイント** | `https://cumulocity.com/docs/2026/edge/edge-operations/` |
| **実測ベンチマーク（Narrow / Wide シナリオ）** | `https://cumulocity.com/docs/2026/edge/benchmarks/` |
| ブランディング・Web SDK | `https://cumulocity.com/docs/2026/edge/using-edge/` |
| **Edge CR の spec 12 項目** | `https://cumulocity.com/docs/2026/edge/edge-custom-resource-definition/` |

### Cumulocity Core API・デバイス管理

| 内容 | URL |
|---|---|
| **Core OpenAPI**（Notification 2.0 の RBAC バイパス・`NotificationSubscription` スキーマ・`bulkNewDeviceRequests`・`creationRamp` / `groupId` / `failedParentId`・ロール名・パスワード制約） | `https://cumulocity.com/api/core/dist/c8y-oas.yml` |
| **Fragment library**（`c8y_SupportedConfigurations` / `c8y_DownloadConfigFile` / `c8y_UploadConfigFile` / `c8y_RequiredAvailability`） | `https://cumulocity.com/docs/device-integration/fragment-library/` |
| 設定管理の 3 系統 | `https://cumulocity.com/docs/device-management-application/managing-device-data/` |
| bulk operation ウィザード | `https://cumulocity.com/docs/device-management-application/monitoring-and-controlling-devices/` |
| **テナント CA・デバイス証明書**（毎年 10/2 の自動更新） | `https://cumulocity.com/docs/device-certificate-authentication/certificate-authority/` , `/device-certificate-authentication/device-certificates/` |
| **Inventory roles performance improvements（KB）** — OPTIMIZED の 2000 件閾値・`prev`/`next` | `https://community.cumulocity.com/t/inventory-roles-performance-improvements/513` |
| Single sign-on / Basic settings | `https://cumulocity.com/docs/authentication/sso/` , `/authentication/basic-settings/` |
| **Managing users**（外部認証ユーザーのパスワードリセット不可） | `https://cumulocity.com/docs/standard-tenant/managing-users/` |
| **Monitoring**（Notification 2.0 のトピック名・サブスクライバのライフサイクル・unsubscribe） | `https://cumulocity.com/docs/standard-tenant/monitoring/` |
| Managing data / Ecosystem | `https://cumulocity.com/docs/standard-tenant/managing-data/` , `/standard-tenant/ecosystem/` |
| Smart rules / **Alarm mapping** | `https://cumulocity.com/docs/cockpit/smart-rules/` , `/standard-tenant/alarm-mapping/` |

### thin-edge.io（バージョン 2.0.1）

| 内容 | URL |
|---|---|
| **エンティティ管理**（MQTT API / REST API / 自動登録の無効化 / `@id`） | `https://thin-edge.github.io/thin-edge.io/operate/entity-management/` |
| **MQTT API リファレンス**（`te/` トピック体系・retain / QoS 要件・ペイロード規約） | `https://thin-edge.github.io/thin-edge.io/references/mqtt-api/` |
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
| sm-plugin API（ロールバックなし・5 分タイムアウト） | `https://thin-edge.github.io/thin-edge.io/references/software-management-plugin-api/` |
| 設定管理（ロールバックなし） | `https://thin-edge.github.io/thin-edge.io/references/agent/tedge-configuration-management/` |

> ⚠️ **thin-edge.io の設定値を確認するときは、実機で `tedge config list --doc` を正としてください。** 公式ドキュメントには既知の不備があります（[担当範囲] §12 参照）。

### ツール

- [go-c8y-cli](https://goc8ycli.netlify.app/docs/) — 全設定投入の主力
- [c8y users resetUserPassword](https://goc8ycli.netlify.app/docs/cli/c8y/users/c8y_users_resetuserpassword/) — *"you can't set a fixed password for another user"*

---

**改版履歴**

| 版 | 日付 | 内容 |
|---|---|---|
| rev.1 | 2026-08-20 | 初版。[担当範囲] をベースに、Cumulocity 側設計を概念軸で再構成。[設定書] §4「主要設計の詳細」を本書へ移管（§0.3）。案 β 前提・案 α は付録 A |
