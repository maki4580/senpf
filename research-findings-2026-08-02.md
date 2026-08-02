# データ連携仕様書 技術調査結果 第2回(2026-08-02)

- 作成日: 2026-08-02(同日更新: 検証再実行の結果を§0に反映)
- 調査方法: Web検索5観点 → 一次情報26ソース取得 → 127主張抽出 → 上位25主張を**独立3エージェント反証投票で検証**(初回は利用上限で全滅、同日再実行で完了: **確定23 / 反証2 / エラー0**)
- 先行調査: [research-findings-2026-08-01.md](research-findings-2026-08-01.md)(検証済み19件はそちらを正とする)
- 反映先: [data-integration-spec.md](data-integration-spec.md)(2026-08-02反映済み)
- 凡例: ✅=検証確定(3-0) / ❌=反証 / 🔍=引用付き抽出だが検証キュー外(断定前に出典確認推奨)

## 0. 検証結果サマリー(2026-08-02 再実行)

**✅ 3-0で確定した11知見(統合後)**:

1. **Notification 2.0はEdge 2025(K8s版)で利用可能** — Edgeカスタムリソース `spec.messagingService.enabled=true` でMessaging Service(**Pulsarベース**)をインストール。オプション機能(既定無効)。追加リソース: **+2 CPUコア/+4GB RAM/+15GBディスク/Persistent Volume×3**(Pulsar Bookkeeper Ledgers/Journal, Zookeeper)。**2024リリースには存在せず2025新機能**
2. **Streaming Analytics(Analytics Builder含む)はEdge 2025の標準搭載(Included)** — 通知のフォールバック手段として追加コンポーネント不要。※「Included」はデプロイ上の位置づけで、商用課金条件は別途確認。DataHubはOptional(+10コア/+10GB)
3. **SSO(Release 2026)はOAuth2認可コードグラント+JWTのみ・SAML非対応**。**KeycloakはAzure ADと並ぶ明示サポートプロバイダ**。グローバルログアウト(Backchannel Logout)はKeycloak 12.0.0+。オンプレはdomain-based tenant resolutionの構成が前提。**SSOドキュメントにEdge固有の記載なし(Edge可否は残課題)**
4. **sm-pluginのCLI契約** — `/etc/tedge/sm-plugins` 配置、ファイル名=softwareType、必須コマンド list/prepare/install/remove/finalize(update-listは任意)、`install NAME [--module-version] [--file]`、**prepare/finalizeがトランザクション/ロールバックの括り**
5. **child deviceの明示登録** — `te/device/<id>//` へのretainedメッセージ `{"@type":"child-device"}`。**公式が明示登録を強く推奨**。**`@id` 指定でそれがそのままCumulocity External IDになる**(未指定時は `<main-id>:device:<child-id>` へ自動導出)。1.5.0でEntity Store HTTP API追加
6. **thin-edge 2.0.0(2026-04-10)でbuilt-in bridgeが既定化** — クラウド接続はtedge-mapper自身が処理し、mosquittoはローカルMQTT専用に。**mosquitto bridge固有のキュー問題は現行既定構成には該当しない**。オフラインバッファはローカルmosquittoのクライアントセッションキュー(`max_queued_messages` が効く側)に置き換わる
7. **tedge-agentはchild device上でも動作可能** — ゲートウェイ配下child deviceでもソフトウェア/設定/ログ/再起動オペレーションが利用可(`te/device/child001///cmd/software_update/<id>`)
8. **thin-edge 1.5.0がCumulocity Certificate Authority対応**(ワンタイムパスワードenrollment+自動更新サービス+PKCS#11/HSM)。**1.5.0時点でCA機能はPUBLIC PREVIEW**(2026年時点のGA状況は要確認)
9. **retention rules仕様確定** — dataType(6種+'*')×maximumAge(日数・最大10年)、絞り込みはfragmentType(EVENT/MEASUREMENT/OPERATION/BULK_OPERATIONのみ)・type(ALARM/AUDIT/EVENT/MEASUREMENTのみ)・source。editableはManagementテナントのみ変更可
10. **イベント添付は1イベント1バイナリ・2つ目は409・既定50MiB(チャンク5MiB)・設定変更可**(先行調査の再確認)
11. **Operationステータスは4値enum確定**(PENDING→EXECUTING→SUCCESSFUL/FAILED+failureReason。SmartREST 502対応)

**❌ 反証された2件(いずれもmosquitto issue #1729の詳細挙動)**:

- 「bridgeでmax_queued_messagesが無視され無制限にキューされる」(1-2で反証)
- 「メンテナが1.6.7で再現確認した」(1-2で反証)

→ **#1729の深刻度は本調査では確定していない**。ただし知見6(built-in bridge既定化)により、thin-edge 2.x採用ならこの問題自体が構成から消える。A7の実機確認は「採用バージョンのキュー挙動確認」として維持(仕様書§4.5反映済み)。

**検証キュー外(🔍のまま)の主要項目**: Notification 2.0のRBACバイパス、Inventoryロール約2000件フォールバック、bulk operationのcreationRamp/failedParentId、100tps/コア(2025版記載)、Alarm CLEARED限定削除、添付のイベント連動削除(コミュニティ回答)、SSOのRSA鍵限定・Own user management READ要件 — いずれも一次情報からの引用付き抽出であり信頼度は高いが、3票検証は未実施。

## 1. 【N8解決】Notification 2.0 は Edge 2025(K8s版)で利用可能

IF-H13a(通知コンシューマの第一候補)の成立条件が大きく前進した。

- Edge 2025(Kubernetes版)の機能比較表に「Messaging Service (for data broker and Notifications 2.0 capabilities): **Yes**」の記載
  — https://cumulocity.com/docs/2025/edge-kubernetes/k8-edge-introduction/
- ただし**既定では無効**。Edge custom resource definition の `messagingService` フィールド指定でオプションインストールされる
  > "Specifying this field installs the Cumulocity Messaging Service, which is required for using the microservice-based data broker and Notifications 2.0."
  — https://cumulocity.com/docs/2025/edge-kubernetes/edge-custom-resource-definition/
- **リソース要件: 追加 2 CPUコア + 4GB RAM + 15GBディスク**(ベース要件に上乗せ)
  — https://cumulocity.com/docs/2025/edge-kubernetes/installing-edge-on-k8/
- 配信保証: 「Consumers receive the topic messages reliably, with **at-least-once semantics, in order** and must **acknowledge each message** in turn.」— 本仕様書IF-H13aのat-least-once+ack設計と整合
  — https://cumulocity.com/api/core/

**⚠️ 重大なRBACバイパス(拠点分離の穴)**:
> "If you assign Notification 2.0 roles or permissions to users, they can create Notification 2.0 subscriptions and receive notifications for **any device**, including those to which assigned inventory roles do not grant access, **bypassing the inventory role RBAC**."
> — https://cumulocity.com/api/core/

→ Notification 2.0の購読権限は「拠点分離(D17)を貫通する全拠点データアクセス権」に等しい。**基盤の通知コンシューマ専用サービスアカウントのみに限定**し、一般ユーザー・案件側アカウントには絶対に付与しないこと(仕様書§3.21・§4.4に反映)。

**EPL/Streaming Analyticsについて**: Edge 2025のEdge operatorは「Apama, Smart Rules, OPCUA Management Server を含む既定マイクロサービス」を展開するとの記載(CRDページ)。先行調査の「Streaming Analytics同梱は反証(1-2)」は2026版ページ基準だった可能性。**技術的には載っているがライセンス形態は別問題**のため、N8のフォールバック判断は「技術的には利用可能な公算・商用条件はベンダー確認」に更新。

## 2. 【N8/IF-P04】SSO(OIDC)の製品制約 — Edge対応は依然未確認

- Release 2026 SSOドキュメント: **authorization code grant のみ対応**(JWT access/IDトークン)。**SAML非対応**
- **トークン署名はRSA鍵のみ対応。楕円曲線(EC)鍵は非対応** → Keycloakレルムの署名鍵設定に直接影響
- SSOユーザーは「Own user management」のREAD権限を持つロールが必須(なければログイン不可)
- SSOはCookieベース(ブラウザのCookie有効が前提)
- **SSOドキュメントにEdge対応可否の記載なし**。Edge CRDページにもSSO/OIDC/外部IdP設定フィールドは存在しない → IF-P04の△は維持。ベンダー確認事項として残る
— https://cumulocity.com/docs/2026/authentication/sso/, https://cumulocity.com/docs/2025/edge-kubernetes/edge-custom-resource-definition/

## 3. 【D17/N6】Edge 2025 プラットフォーム仕様

- **「Vertical scalability: Yes, limited to appr. 100 tps per CPU core」「Horizontal scalability: No」の記載が2025 K8s版ドキュメントに実在**
  — https://cumulocity.com/docs/2025/edge-kubernetes/k8-edge-introduction/
  → 先行調査で「反証」とされた100tps/コアは**版違い**(2026版introページには無い)。D17成立条件の試算根拠として「2025 K8s版ドキュメント記載値」と出典明記のうえ復活可。ただし採用リリース版確定後に再確認
- ベースハードウェア要件(2025 K8s版・オプション無し): **6 x86-64コア / 10GB RAM / 100GBディスク**。**CPUはAVX命令対応必須**(MongoDB要件)。オプション込みの合計例: +Messaging Service(2コア/4GB/15GB)+DataHub(10コア/16GB/100GB)
- サポートは**シングルノードKubernetesクラスタのみ・K8s 1.32.x のみ**(2025版)
- **ライセンスはドメイン名に紐づく**: Edge CRDに `licenseKey`(対象ドメインのライセンス)が必要。ライセンスファイルはプロダクトサポートへ申請。Edgeレジストリ資格情報も必要 = 商用ライセンス製品であり自由にインストール不可
  — https://cumulocity.com/docs/2025/edge-kubernetes/installing-edge-on-k8/, edge-custom-resource-definition/

## 4. 【N5解決】Retention rules(データ保持ルール)仕様

- 対象データ種別は6種+ワイルドカード: **ALARM, AUDIT, BULK_OPERATION, EVENT, MEASUREMENT, OPERATION, '*'**。`maximumAge` は日数指定(最大10年)
- **既定では全履歴データが60日で削除**(システム設定で変更可)
- 実行は**おおむね1日1回**。fragmentType・type・source(デバイスID)でスコープ可能
- **Alarmは status=CLEARED のもののみ削除される。ACTIVE/ACKNOWLEDGEDのAlarmはretention rulesでは消えない**
  → §3.10のAlarmエクスポート設計に直結: 「確認後はCLEARする」運用規定(2026-08-01調査)を守らないと、未クリアAlarmがOperational Storeに無期限に蓄積する
- **retention rulesはファイルリポジトリ(Inventoryバイナリ)には適用されない** → ソフトウェアリポジトリのモデルバイナリは自動削除されず、世代管理・削除はA10の運用設計が必須
- **イベント添付バイナリはイベント削除時に連動削除される**(孤立しない)。ただしこれは製品ドキュメント本文ではなく公式コミュニティでのエキスパート回答
  — https://cumulocity.com/docs/standard-tenant/managing-data/, https://cumulocity.com/api/core/, https://community.cumulocity.com/t/event-file-binary-attachment-retention-rules/14387
- `editable` フラグはManagementテナントのみ変更可 → Edge運用者が変更できる範囲の制約(要実機確認)

## 5. 【A10/IF-D01】Operation / Bulk Operation 仕様

- Operationステータスは**4値のみ: PENDING → EXECUTING → SUCCESSFUL / FAILED**。PENDINGで作成され、**遷移させるのはデバイス側の責務**
- `c8y_SoftwareUpdate` の指示内容: `action`(install/delete)、`name`、`version`、`url`(取得先)、`softwareType`。デバイス側はEXECUTING設定→適用→`c8y_SoftwareList`(インベントリ)更新→SUCCESSFUL/FAILED設定の順で処理
- **Bulk Operation**:
  - `creationRamp`: 個別Operation生成間の遅延秒数 = **段階投入の製品機能**(帯域圧迫の平準化に使える)
  - `groupId`(対象デバイスグループ)と `failedParentId`(失敗分再実行)は排他 — **失敗分のみの再実行が製品機能として存在**
  - `generalStatus`: SCHEDULED / EXECUTING / **EXECUTING_WITH_ERRORS** / SUCCESSFUL / FAILED / CANCELED + progress(pending/failed/executing/successful/all件数)
  - **オフラインデバイスのOperationはPENDINGのまま無期限滞留。自動タイムアウト・自動リトライなし** → 仕様書§3.18の「滞留管理(有効期限運用+進行判定から除外)を基盤側で自作」要件が製品仕様に裏付けられた
  - Bulkはサーバー側の束ね概念であり、デバイスからは個別Operationしか見えない
  — https://cumulocity.com/api/core/, https://cumulocity.com/docs/device-integration/fragment-library/, https://community.cumulocity.com/t/bulk-operation-api-best-practice/3124

## 6. 【N6/D17】Inventoryロールによる拠点分離の限界(3点)

1. **グローバルロールバイパス**: Inventory権限を含むグローバルロールを持つユーザーは、Inventoryロール設定に関係なく**全デバイスを閲覧・変更できる** → 拠点分離はInventoryロール「のみ」で権限設計した場合に限り有効。グローバルロールの付与ポリシー策定が必須
2. **約2000オブジェクトでの検索フォールバック**: Inventoryロール制限付きユーザーのアクセス可能オブジェクトが約2000件以上になると**旧検索アルゴリズムにフォールバックし、空ページが返り得る**(`statistics.next` URLでのページネーション必須)→ 50拠点×数百台では確実に超過。紐づけ確認アプリ(IF-H06)のサービスアカウントをInventoryロール制限にする場合、この挙動への対応が必要
3. **Notification 2.0バイパス**(§1参照)
- Inventoryロールは資産階層を継承(親グループ付与で配下全体に有効)。権限レベルはREAD/CHANGE/ALLの3段階
  — https://cumulocity.com/docs/standard-tenant/managing-permissions/, https://cumulocity.com/api/core/

## 7. 【IF-P01】X.509証明書ライフサイクル

- **Cumulocity certificate-authority機能**: プラットフォーム自身がCAとなり、CSR受理・署名済みX.509デバイス証明書発行を行う。**EST(RFC 7030)ベース**: `/.well-known/est/simpleenroll`(新規。テナント+デバイスシリアル+ワンタイムパスワード認証)、`/simplereenroll`(更新。デバイス資格情報/JWT)
- デバイス証明書の有効期間は**1年**。テナントCA証明書は**1095日(3年)で毎年自動更新**(鍵は不変、有効期間のみ更新)。**テナントごとにCAは1つのみ**
- **certificate-authority機能はPublic Preview(GA前)** — 採用判断時に成熟度確認が必要(出典: c8y-devicecertリポジトリの記載)
- **certificate-authorityドキュメントにEdge対応可否の記載なし** → Edgeでの利用可否はベンダー確認事項
- **thin-edge.io側の対応**: 1.5.0でCumulocity CA enrollment+**自動証明書更新サービス**+PKCS#11(HSM)対応。1.6.0でHSM経由CA更新・EC鍵対応。`tedge cert download c8y`(ワンタイムパスワード)でのCA型enrollmentが公式推奨。自己署名は開発/テスト用の位置づけ
- 従来型(trusted certificates): テナントにCA証明書(X.509 v3・PEM・CA:true)をアップロード。**デバイス個別証明書のアップロードは性能上非推奨**。autoRegistrationEnabled=trueで初回MQTT接続時の自動登録、無効時はCSV一括登録(AUTH_TYPE=CERTIFICATES。個別登録は非対応)
- **失効**: 侵害されたデバイス証明書はCumulocity管理のCRLに追加。CA侵害時はCA削除→新CA作成→全デバイス再enrollment
- 証明書チェーン長は最大10(クラウドの場合)。MQTT over WebSocketは証明書認証非対応。ポート8883
- mTLS→JWTセッショントークン交換(device access token)は**将来廃止予定と明記** → §3.5のdevice token記載は「現行仕様だが廃止方向」の注記が必要
  — https://cumulocity.com/docs/device-certificate-authentication/certificate-authority/, device-certificates/, managing-trusted-certificates/, https://thin-edge.github.io/thin-edge.io/start/connect-c8y/, https://github.com/thin-edge/thin-edge.io/releases, https://github.com/reubenmiller/c8y-devicecert
- 参考: EST更新タイミングのサーバー主導化はIETFドラフト(draft-ietf-lamps-est-renewal-info-00, 2026-02)段階でありRFC未成立。更新タイミングはクライアント側ポリシー(残存期間50%等)で設計するのが現行の正解

## 8. 【D16/A7】thin-edge.io の重要アップデート

- **thin-edge 2.0.0 は built-in bridge(tedge-mapperがマッピングとクラウド接続の両方を担う)が既定**となり、**mosquitto bridgeへの依存を置き換えた**。0.x→2.xは1.x経由の段階アップグレード必須、旧 `tedge/` トピック廃止(破壊的変更)
- **mosquitto issue #1729(max_queued_messagesがbridgeに効かない)は2020年に修正済み**(メンテナ発言+fixesブランチ)。現行mosquittoでは適用されるはず — ただし旧issueの再現条件(cleansession=false時の無制限キュー)は当時実在したため、**A7の実機確認(網断長時間継続時のディスク/メモリ実測)は維持**。thin-edge 2.x採用ならbuilt-in bridge側のキュー/スプール挙動を確認対象にする
- 1.5.0: **Entity Store HTTP API**(child device/サービスの明示登録機構)追加 → §4.1「自動登録無効化+明示登録」の実装手段。エンティティ登録は `@id` で明示ID指定可能(External ID命名制御のフック)。データ送信・コマンド受信する全エンティティは明示登録が必要(自動登録は設定で有効/無効切替可)
- child device数の数値上限はMQTT APIリファレンスに記載なし(数百台規模の実績確認はA7のまま)
  — https://github.com/thin-edge/thin-edge.io/releases, https://thin-edge.github.io/thin-edge.io/next/references/mqtt-api/

## 9. 【IF-S07】sm-plugin API 仕様

- プラグインは `/etc/tedge/sm-plugins` に配置した実行ファイル。起動時に `list` コマンド呼び出しで登録される。ファイル名がsoftwareTypeとのルーティングキー(例: `ai-model` プラグインが softwareType=`ai-model` を処理)。既定プラグインは `tedge config set software.plugin.default` で指定
- **終了コード契約: 0=成功 / 1=usage(未実行) / 2=恒久失敗(リトライ無意味) / 3=一時失敗(後でリトライ、網復旧時等) / 4=タイムアウト**
- **1コマンド呼び出しのタイムアウトは5分固定** → **IF-S07の設計制約**: モデルファイル差し替え+コンテナ再起動+起動ヘルスチェックが5分に収まらない場合、プラグインは即時応答し適用完了を別途イベント報告する非同期パターンが必要(A10のロールバック設計と同時に確定)
- `update-list`(バッチ)は任意実装。未実装なら exit 1 でパッケージ単位の install/remove にフォールバック
  — https://thin-edge.github.io/thin-edge.io/references/software-management-plugin-api/

## 10. 【§4.5根拠】再送・信頼性パターン(AWS一次情報)

- ジッタなしのcapped exponential backoffは**同期クラスタ化により競合をわずかしか減らさない**。Full Jitter(sleep = random(0, min(cap, base×2^attempt)))は100クライアント競合シミュレーションで**総呼び出し数を半減以下**。AWSはjittered backoffを「リモートクライアントの標準アプローチ」と推奨
- リトライはtransient障害(スロットリング=HTTP 429、一時的網断・サービス不可)に限定。**非transient障害はリトライせずcircuit breakerでfail fast**。**リトライを使う操作は冪等であること**(部分更新による状態破壊防止)
  — https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/, https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/retry-backoff.html

## 11. 【方法論】機械可読IF仕様(AsyncAPI)

- AsyncAPI 3.0はプロトコル非依存のメッセージ駆動API記述標準で**MQTTを明示サポート**。構造はServers(ブローカー接続)/Channels(トピック)/Operations(send/receive)/Messages(ペイロード+ヘッダ)で、IF仕様書の章立てにそのまま対応。QoS等のMQTT固有事項はMQTT bindingオブジェクトで表現。Apache 2.0ライセンス
- 本仕様書のMQTT系IF(IF-S04/W01/H01/H02)のAPI仕様書(物理仕様)をAsyncAPIで、REST系をOpenAPIで機械可読化する方針の裏付け
  — https://www.asyncapi.com/docs/reference/specification/v3.0.0

## 11.5 【転換2】競合製品: ThingsBoard PE ライセンス(🔍未検証・再実行時に取得)

検証再実行時にthingsboard.io(公式ライセンスドキュメント)の取得に成功。3年TCO比較(design-decisions.md 転換2)の判断材料:

- **perpetual fallback license**: 1年分のアップデート付きで**買い切り・永続利用可**(サブスクリプション失効後も固定バージョンで無期限稼働可能)。2年目以降のアップデートは割引価格で別途購入
- **ライセンスはインスタンス単位**(1ライセンス=1インスタンス)。**HA構成には最低2ライセンス必要**
- **閉域網(オフライン)でのライセンス認証方式・具体価格・対象バージョン範囲はドキュメントに記載なし** — 閉域採用の可否はCumulocity Edge同様ベンダー確認が必要

→ 中央集約(D17)構成なら必要ライセンス数は1〜2(HA時)であり、「拠点ごとN台」前提だった転換2のTCO試算は中央集約前提で引き直す価値がある。

## 12. 未達領域(今回も調査できなかったもの)

| # | 領域 | 状態 |
|---|---|---|
| 1 | Edge中央サーバー利用(1インスタンス多拠点収容)の**ライセンス上の可否・価格体系** | ドメイン紐づけライセンスであることまで判明。収容形態の可否は**ベンダー確認が唯一の手段**(D17成立条件(3)) |
| 2 | 競合製品比較(ThingsBoard/EMQX/Azure IoT Operations/AWS Greengrass)の実質評価 | 再実行でThingsBoard PEライセンスのみ取得(§11.5)。機能比較・EMQX/Azure/AWSは未着手 |
| 3 | certificate-authority機能・SSOの**Edgeでの**対応可否 | ドキュメント上は記載なし(不在の証明ではない)。ベンダー確認事項 |
| 4 | Edgeのサポート期限・リリースサイクル | 未着手 |
| 5 | ~~25主張の反証検証~~ | **✅完了(2026-08-02再実行)**。結果は§0参照(確定23/反証2/エラー0) |

## 13. 検証の再実行について

**実施済み(2026-08-02)**。`resumeFromRunId: "wf_7f8b9455-7e3"` で再実行し、検索・取得フェーズはキャッシュ再生、検証25主張×3票が完走した(108エージェント・エラー0)。結果は§0に統合済み。
