# Cumulocity 利用設計書 ファクトチェック結果

**対象**: [Cumulocity利用設計書.md](Cumulocity利用設計書.md)（rev.1, 2026-08-20, 全 2984 行）
**実施日**: 2026-08-20
**方法**: 8 名の独立レビュアー（サブエージェント）が章を分担し、本書の事実主張を **一次情報に当たって逐語照合**
**一次情報**: Cumulocity 公式 docs（2026 版）／ Core OpenAPI 生 spec（`c8y-oas.yml`, 1.46 MB）／ Edge OpenAPI 生 spec ／ thin-edge.io v2.0.1 docs・GitHub ソース ／ go-c8y-cli（**既定ブランチ `v2`**）／ Cumulocity Tech Community
**検証規模**: 事実主張 **約 350 件** ＋ §16 出典 URL **39 件**

> 前回の [レビュー結果](Cumulocity利用設計書_レビュー結果.md) は「文書内整合性・リスク・運用 readiness」の観点でした。**本書は別軸で、記述が事実として正しいかだけを見ています。** 両者の指摘が食い違う箇所は本書を優先してください（前回レビューは一次情報照合をしていません）。
>
> 各レビュアーの全文レポート（逐語引用・探索経路つき、計 5,130 行）は [factcheck/](factcheck/) に格納しています。

| レポート | 担当範囲 | 件数 |
|---|---|---|
| [A1_sec0-2.md](factcheck/A1_sec0-2.md) | §0〜§2 命名・識別 | 35 |
| [A2_sec3.md](factcheck/A2_sec3.md) | §3 デバイス管理・証明書 | 40 |
| [A3_sec4-5.md](factcheck/A3_sec4-5.md) | §4〜§5 オペレーション・死活監視 | 55 |
| [A4_sec6-7.md](factcheck/A4_sec6-7.md) | §6〜§7 データモデル・保持 | 39 |
| [A5_sec8-9.md](factcheck/A5_sec8-9.md) | §8〜§9 ルール・アラームマッピング | 28 |
| [A6_sec10-11.md](factcheck/A6_sec10-11.md) | §10〜§11 通知・権限 | 65 |
| [A7_sec12-13.md](factcheck/A7_sec12-13.md) | §12〜§13 Edge・設定 | 67 |
| [A8_citations.md](factcheck/A8_citations.md) | §16 出典 ＋ §14〜§15・付録 A | 25 ＋ URL 39 |
| [B1_externalid.md](factcheck/B1_externalid.md) | **追加調査**: external ID の要否（§6b） | Q1〜Q6 |

---

## 0. 総評 — 本書の事実精度は高い

**判定分布（概算）**

| 判定 | 件数 | 意味 |
|---|---|---|
| ✅ 一致 | 約 280 | 一次情報の逐語記述と合致 |
| 🔶 部分的に不正確 | 約 45 | 方向は正しいが条件・数値・適用範囲の限定が欠落／過度な一般化 |
| ❌ 不一致 | **15** | 一次情報と矛盾、または存在しない API・コマンド |
| ⚠️ 検証不能 | 10 | 一次情報に記述が見当たらない（[確] なら格下げが必要） |
| 🔵 確度記号の過小 | 23 | 記述は正しいが [要]/[推] のまま。**検証工数を削減できる** |

**約 350 件のうち ❌ は 15 件（4%）** で、SmartREST テンプレート番号、`creationRamp` の意味、`responseInterval` の値域、thin-edge の `te/` トピック体系、`c8y.enable.*` 全キー、SSO の設定フィールド、Inventory ロール API、リテンションルールの `dataType` enum など、**細部にわたる [確] のほとんどが逐語で正確**でした。CLI コマンドの実在性も 15 項目中 14 項目が正しい。

**発端となった §1.4 の記述は正しいことが確定しました。** Core OpenAPI の `NotificationSubscription` スキーマに逐語で存在します。

```yaml
subscription:
  type: string
  pattern: '^[a-zA-Z0-9]+$'
  minLength: 1
```

`context` の enum は `[mo, tenant]`、`subscriptionFilter.apis` の enum は `[alarms, alarmsWithChildren, events, eventsWithChildren, managedobjects, measurements, operations, '*']`。**`maxLength` は未定義**なので §2.7 の制約表に「長さ制限なし」を追記してください。

### 0.1 本レビュー自身で 3 回起きた「探索不足による偽陰性」

今回の発端（他 AI が「そんな記述はない」と誤判定）と**同じ誤りが、本レビューの中でも 3 回起きました**。誤りの内容より、この再現性のほうが重要です。

| # | 誤判定 | 真因 |
|---|---|---|
| 1 | 「go-c8y-cli に `deviceregistration register-ca` は無い」（前回レビュー M-1 / m-9） | リポジトリの既定ブランチが `master` ではなく **`v2`**。`master` を見て「無い」と判定した |
| 2 | 「Edge docs に『100 tps/CPU コア』は存在しない」（A3） | 検索経路の不足。実際は edge-introduction の比較表に逐語で存在 |
| 3 | 統合時の確認で同じ箇所を grep して空振り（本レビュー実施者） | `.{120}tps.{120}` という前後 120 文字を要求するパターンを使い、短い `<td>` 行にマッチしなかった |

**教訓**: 「一次情報に記述がない」は、**探索経路を明示できたときだけ**主張してよい結論です。本書の [確] を疑う際は、最低 2 経路（docs 別ページ／OpenAPI 生 spec／GitHub ソース／WebSearch）を尽くしてください。なお `cumulocity.com/api/...` の Redoc ページは WebFetch すると 4KB の SPA シェルしか返らないため、**必ず生 spec（`https://cumulocity.com/api/core/dist/c8y-oas.yml`）を取得して grep** してください。

---

## 1. 前回レビューの訂正

前回レビューの指摘のうち、一次情報照合で**覆ったもの**です。

| 前回 | 指摘内容 | 今回の結論 |
|---|---|---|
| **M-1** | `c8y deviceregistration register-ca` の実在性が未確認 | **実在する。** go-c8y-cli `v2` ブランチに `docs/.../c8y_deviceregistration_register-ca.md` が存在。thin-edge 公式 `references/certificate-management.md` にも同コマンドが掲載。オプション名（`--id` / `--one-time-password` / `--key`）も一致 |
| **m-9** | `c8y devicemanagement certificate-authority create` の実在性が未確認 | **実在する。** `pkg/cmd/devicemanagement/certificate_authority/create/create.manual.go`。ただし docs 上は「(PREVIEW FEATURE)」表記 |
| **M-2** | `agent.entity_store.auto_register` の実在性が未確認 | **両キーとも v2.0.1 で実在し、両方とも実際に読まれている。** `tedge_config.rs` の `c8y.entity_store.auto_register`（L786-789, 既定 true）と `agent.auto_register`（L1240-1243, `deprecated_key = "c8y.entity_store.auto_register"`）。参照箇所は `c8y_mapper_ext/src/config.rs` L226 と `tedge_agent/src/agent.rs` L200 で**別々**。<br>**補正**: 再起動対象は `tedge-agent` だけでなく **`tedge-mapper-c8y` も必要** |
| **M-4** | 「約 100 tps/CPU コア」の確度表示が不統一。[判断記録] D17 は「版依存・伝聞値」と警告 | **公式 2026 docs に逐語で存在**するため §7.7 の [確] が正しく、§5.9 の [要・伝聞値] を [確] に格上げすべき。ただし**新たな問題**あり → F-8 |
| **M-3** | テナント CA「毎年 10/2 自動更新」の一般化の根拠が不明 | **公式に年次パターンとして明記**（*"Tenant CA is automatically renewed on 2 October at 02:00 AM every year."*）。ただし**そこから導かれるリスクのほうが誤っている** → F-5 |
| **C-1** | `alarmsWithChildren` の確度が §1.1 [確] と §1.6 [要] で正反対 | **[推]／[要] 側に揃えるのが正しい。** 7 経路の探索で「どの関係を辿るか」は公式のどこにも書かれていないことが確定 → §6 |
| **C-2** | Inventory ロールが child device まで到達する根拠が未検証 | **懸念は一次情報で裏付けられた。** 公式保証は「グループ→サブグループ→**それらのグループに属するデバイス**」までで、デバイス配下の `childDevices` は不記載 → §6 |
| **C-5** | Edge サーバー自身の TLS 証明書の失効監視が欠落 | **指摘は妥当。かつ更新手順も公式に存在する**（本書が「変更不可」と誤記していたため見落とされていた）→ F-6 |

---

## 2. Critical — 設計変更を要する誤り

### F-1. グローバルロールが拠点分離を丸ごと無効化している
**該当**: §11.2 R-02 / R-03、§11.10 CT-5 ｜ 出典: [A6-33 / A6-64](factcheck/A6_sec10-11.md)

§11.3 は「Inventory ロールが拠点分離の実装点です」と宣言していますが、§11.2 は R-02 に `Alarms ADMIN` / `Events READ` / `Inventory READ`、R-03 に `Alarms READ` / `Events READ` / `Inventory READ` を**グローバルロール**として与えています。

> The role ROLE_ALARM_READ is not required, but if a user has this role, **all the alarms on the tenant are returned**. If a user has access to alarms through inventory roles, only those alarms are returned.
> — `c8y-oas.yml` `getAlarmCollectionResource`

> In a Role Based Access Control (RBAC) approach **you must use the inventory roles in order to have the correct level of separation. Apart from some global permissions (like "own user management") customer users will not be assigned any roles.**
> — `c8y-oas.yml` §Access rights and permissions

**§10.3 で Notification 2.0 の RBAC バイパスを厳密に潰しているのに、REST 側にそれより広い穴が最初から開いています。** 否定テスト CT-5（拠点 A の Manager が拠点 B のアラームを取得できないこと）は、現行のロール定義では**必ず失敗します**。

**対処**: R-02 / R-03 のグローバル権限を「Own user management READ ＋ アプリケーションアクセス」のみに削減し、データ系（Alarms / Events / Inventory）は R-04a / R-04b の Inventory ロール側へ全面的に移す。あわせて **`Inventory CHANGE` は READ を含まない**（*"CHANGE - to modify objects (does not include READ permission)"*）ため、拠点 Manager には `Inventory ALL` を割り当てること（[A6-39](factcheck/A6_sec10-11.md)）。

### F-2. メンテナンスモードは「そのデバイスの全アラーム」を抑止する — §5 の死活監視が沈黙する
**該当**: §5.1・§5.4 対処 2・§5.5・§5.6 ｜ 出典: [A3-40](factcheck/A3_sec4-5.md)・[A5-26](factcheck/A5_sec8-9.md)（2 名が独立に発見）

§5.4 は「child device は登録時にメンテナンスモード（`responseInterval: 0`）へ落とす」と規定する一方、§5.6 は同じカメラに `x_CameraDown`（MAJOR）を上げる設計です。両立しません。

> **Alarm suppression** — If the source device is in maintenance mode, **the alarm is not created and not reported to the Cumulocity event processing engine.** … the self link of the alarm will be: `"self": "https://<TENANT_DOMAIN>/alarm/alarms/null"`
> — `c8y-oas.yml` `POST /alarm/alarms`

> **While a device is being maintained, no alarms for that device are raised.**
> — https://cumulocity.com/docs/device-management-application/monitoring-and-controlling-devices/

**「死活監視を設計したのに死活アラームが 1 件も上がらない」状態**になります。しかも POST は成功扱いで `self` が `/alarm/alarms/null` になるだけなので、発覚が極端に遅れます。EPL・スマートルールも発火しません。

**対処**: §5.4 の対処案を差し替える。副作用が最小なのは **child の `responseInterval` を十分長い正値にする**（最大 32767 分 ≒ 22.7 日）。代替として `c8y.availability.enable false`（main も止まる）、`@health` 方式（アラーム型が `c8y_UnavailabilityAlarm` になる）。§14 に「メンテナンス中の child へ `x_CameraDown` を POST して実際に登録されるか」を追加すること。

### F-3. SmartREST 117 は既存値を上書きしない — §5.5 の設計値が一台も反映されない
**該当**: §3.8・§5.4（行 926）・§5.5 ｜ 出典: [A2-31](factcheck/A2_sec3.md)・[A3-46 / A3-47](factcheck/A3_sec4-5.md)（2 名が独立に発見）

本書は「Cumulocity 側から手で書くと**次回接続時に thin-edge の値で上書きされ**、『なぜか設定が戻る』という切り分けの難しい事象になる」としていますが、**逆です**。

> Set required availability (117) … **This will only set the value if it does not exist. Values entered, for example, through the UI, are not overwritten.**
> — https://cumulocity.com/docs/smartrest/mqtt-static-templates/

> This template can be sent in a fire-and-forget approach during device startup because **it doesn't override already existing required availability configuration**
> — https://cumulocity.com/docs/device-integration/fragment-library/

thin-edge がこの 117 を使うことは `crates/extensions/c8y_mapper_ext/src/availability/actor.rs`（`C8ySmartRestSetInterval117`）で確認済み。

**帰結**: 初回接続で 60 分が書かれた後に `tedge config set c8y.availability.interval 15m` しても、**Cumulocity 側は 60 分のままです**。§5.5 の設計値（15 分 / 10 分）が一台も反映されません。

**対処**: 注記を反転する。①初回接続**前**に値を投入する手順、②既存装置は Cumulocity 側 MO を PUT（または SmartREST `107` でフラグメント削除後に 117 を再送）する移行 runbook、の両方を §5 に追加。あわせて §3.5 の手順順序（child 登録 → 可用性設定）も、thin-edge が child 新規登録時に 117 を送る（`availability/actor.rs` L137-141）ため要見直し。

### F-4. アラームマッピングのキーは「前方一致」— 意図しない型まで静かに抑止される
**該当**: §9.1（行 1640）・§9.2（行 1663）・§6.4 ｜ 出典: [A5-20](factcheck/A5_sec8-9.md)

本書は「⚠️ `x_Alarm_<種別>` は `<種別>` ごとに個別のキーになります。種別が案件から追加されるたびにマッピングの追加が必要です」としていますが、誤りです。

> The alarm type provided as an alarm mapping is **interpreted as alarm type prefix: `"<type-prefix>*"`**. If you create, for example, an alarm mapping to address alarms of type "crit-alarm", the mapping is effective for any type of alarm that starts with this value, for example, "crit-alarm-1", "crit-alarm-2", or "crit-alarm-xyz".
> — https://cumulocity.com/docs/standard-tenant/alarm-mapping/

**良い面**: `x_Alarm_` 1 キーで全種別に効くため、「種別追加のたびにマッピングを追加する」運用手順が丸ごと不要になります。
**危険な面**: 短いキーを置くと**意図しない型まで抑止・上書き**します。`NONE`（抑止）と組み合わせると「アラームが静かに消える」最悪ケースになります。

**対処**: §9.2 の運用注記を書き換え、**§6.4 の型名規約に「既存のマッピングキーのプレフィックスにならないこと」を追加**。§2.8 の CI 検査に同項目を入れるのが確実です。

### F-5. §3.7「10/2 に全拠点のデバイス証明書が同時失効」は公式記述と真逆
**該当**: §3.7・§14.11 CT-32・§15.4 SV-25 ｜ 出典: [A2-28 / A2-29 / A2-30](factcheck/A2_sec3.md)・[A8-03](factcheck/A8_citations.md)（2 名が独立に発見）

テナント CA が毎年 10/2 02:00 に自動更新されること自体は公式どおり（M-3 は解消）。しかし**そこから導かれる帰結が誤っています**。

> **The renewal process ensures that existing device certificates remain valid until their expiration.**
> **All CA metadata, private keys, and public keys remain unchanged, ensuring a seamless renewal process. Only NotAfter and NotBefore will be changed.**
> Device certificates issued by the CA continue to have 1 year validity from issuance date
> — https://cumulocity.com/docs/device-certificate-authentication/certificate-authority/

**同じ鍵ペア・同じ Subject での再署名**なので、既存デバイス証明書の署名検証は成立し続けます。「CA 更新日に全拠点が同時再エンロールして輻輳する」という前提は成立しません。

**存在しないリスクに W1 の検証工数と当日要員を割り当てる誤り**です。輻輳リスクの真因は「**初回エンロール日の集中**」（デバイス証明書は発行から 1 年、期限 30 日前に更新）であり、リスク分析自体は有効なので**対象を付け替えてください**。なお thin-edge は既に `RandomizedDelaySec=5m`（コメント: *"Add jitter to prevent a thundering herd of simultaneous certificate renewals"*）を持っています。CT-32 は大幅に縮小できます。

### F-6. §12.2「ドメイン・TLS・ライセンスは install 後に変更できない」は公式と真逆
**該当**: §12.2 P-0-5 / P-2 / P-3、§15.3 SV-22 ｜ 出典: [A7-02 / A7-04 / A7-05 / A7-27](factcheck/A7_sec12-13.md)

> **You can later update the domain and license to match your environment by following the steps outlined in Modifying Edge.**
> — https://cumulocity.com/docs/2026/edge/installing-edge/

```
c8yedge config --set domain=<domain-name> --set-file licenseKey=<path> \
  --set-file tlsSecret.tls.key=<path> --set-file tlsSecret.tls.crt=<path>
```

自前 K8s では CR の `spec.domain` / `spec.licenseKey` / `spec.tlsSecretName` を編集して `kubectl apply`（https://cumulocity.com/docs/2026/edge/manage-edge/ ）。

CPU / RAM も同様で、*"You can add more CPU cores or RAM to the host at any time, and Edge will use the additional resources automatically without further configuration."*。ストレージも *"Once Edge is installed, you can only increase this value, but cannot reduce."*（[A4-35](factcheck/A4_sec6-7.md)）。**Edge 全ページを `immutable|cannot be changed` で横断 grep した結果、真に変更不可と明記されているのは `spec.storageClassName` のみ**でした。

**最大の害**: **TLS 証明書の期限更新という不可避の定期作業が「禁止事項」に分類されている**ことです。前回レビュー C-5（Edge サーバー証明書の失効監視欠落）に対する手順が、本書自身の誤記によって塞がれていました。

**対処**: §12.2 の見出しを「install 後に変更できない項目」→「**変更コストが極めて高い項目**」に改め、`storageClassName` のみを「変更不可」に残す。§12.5 に TLS 証明書更新手順を [確] で追記し、§12.7 の監視対象に Edge サーバー証明書の期限を追加。

### F-7. Notification 2.0 のバックログ満杯は「ディスク逼迫」ではなく「データ投入の停止」
**該当**: §10.5・§12.7・§7.6 ｜ 出典: [A6-17](factcheck/A6_sec10-11.md)

> If the backlog for a Notifications 2.0 topic has reached its quota limit, **any API request to the Cumulocity platform that would be published onto that topic will receive HTTP response code 500.** … Note that for requests using the PERSISTENT processing mode, the Cumulocity operational store will still be updated. This can lead to duplicated entries…
> — `c8y-oas.yml` §Notification 2.0 Service Quotas

Hard quota は **メッセージバックログ 25 MiB**、**TTL 36 時間**（https://cumulocity.com/docs/standard-tenant/monitoring/ ）。

**拠点 A の案件アプリが ack を止めると、25 MiB 到達時点でその拠点のアラーム／イベント POST が 500 で失敗します。** 「溜まり続けてディスクを食う」ではなく「**データが入らなくなる**」という、まったく重さの違う障害です。TTL 36 時間の上限があるため「無限に溜まる」も不正確。

**対処**: §10.5 のコンシューマライフサイクル設計に quota を明記し、§12.7 の監視対象に追加（監視手段は §4 の 🔵 表 #1 を参照 — **公式 UI に危険域つきの監視画面があります**）。

---

## 3. Major

### F-8. §5.9 の Wide シナリオ誤適用と、公式ドキュメント自身の数値矛盾
**該当**: §5.9・§7.7 ｜ 出典: [A3-54 / A3-55](factcheck/A3_sec4-5.md)・[A4-36 / A4-37](factcheck/A4_sec6-7.md)・実施者による確認

**(a) Wide シナリオは本構成に直撃しません。** 本書は「8 CPU で 1,200 クライアントが上限」を child device 数に当てはめていますが、Wide の律速は**同時接続 MQTT クライアント数**です（*"These end-to-end test scenarios drive a number of **MQTT clients** …"*）。代理登録された child device は独自の MQTT 接続を持たない（拠点あたり thin-edge の 1 本に集約される）ため、接続数は**拠点数オーダー**です。律速は Narrow 側（8 スレッドで 25,000 measurement/s）に置き換え、child device 数は MO 数・デバイス課金・inventory クエリ性能として別軸で [要] 化してください。

**(b) 「約 100 tps/CPU コア」は公式に実在します。** レビュー中に「存在しない」との判定がありましたが誤りで、実施者が直接確認しました。

```
edge-intro.html:1177:<td style="text-align:left">Yes, limited to appr. 100 tps per CPU core</td>
```

— https://cumulocity.com/docs/2026/edge/edge-introduction/ 「Cumulocity Edge versus other Cumulocity deployments」比較表の `Vertical scalability` 行

**(c) ただし公式ドキュメント自身が矛盾しています。** 同じ Edge docs のベンチマークページは Narrow で 8 スレッド 25,000 measurement/s（≒ 3,100/スレッド）を示しており、比較表の 100 tps/core と **30 倍以上乖離**します。ベンチマークページには *"for illustrative purposes only"* の Caution があり、単位も「CPU **threads**」です。

**対処**: §5.9 の [要・伝聞値] を [確] に格上げしたうえで、「**公式に非整合な 2 つの数値がある。サイジングは PoC 実測を正とする**」と明記する。これは [判断記録] D17 の警告と整合し、SV-14（負荷試験必須）の根拠をむしろ強化します。

### F-9. §8.2 RU-2「スマートルールによる時限自動クリア」は存在しない
**該当**: §8.2 RU-2・§8.3 ｜ 出典: [A5-08](factcheck/A5_sec8-9.md)

スマートルールのプリセット全 11 種（https://cumulocity.com/docs/cockpit/smart-rules-collection/ ）に、時間経過でアラームをクリアするテンプレートはありません。クリアするのは閾値系・ジオフェンス系が「自分が作ったアラームを条件反転で戻す」だけです。RU-2 を EPL 側（RL-b）に移し、§8.2 と §8.7 の整合を取ってください。

### F-10. §6.9「`time` が無いと Cumulocity が受信時刻を付ける」は逆
**該当**: §6.9 TM-d ｜ 出典: [A4-24](factcheck/A4_sec6-7.md)

> *When thin-edge.io receives a measurement, it will add a timestamp to it before any further processing.* / *when not provided, thin-edge.io uses the current system time*
> — https://thin-edge.github.io/thin-edge.io/understand/thin-edge-json/

> *The default value for the time fragment will be the timestamp in utc time that is added by the tedge-mapper-c8y*
> — https://thin-edge.github.io/thin-edge.io/start/raise-alarm/

時刻を付けるのは thin-edge で、しかも**ローカル受信時**です（builtin-flows の既定フロー第 1 ステップが `{ builtin = "add-timestamp" }`）。したがって「網断復旧後の一括再送で全イベントが復旧時刻になる」という失敗モードは本構成では起きません。**`time` 必須という結論は維持し、理由だけ差し替えて**ください（理由が誤ったままだと、レビューで規約ごと否定されます）。

### F-11. §6.8「既存イベントへの後付け添付はできない」は誤り
**該当**: §6.8 制約 1 ｜ 出典: [A4-18](factcheck/A4_sec6-7.md)

> **Attach a file to a specific event** — Upload a file (binary) as an attachment of a specific event by a given ID.
> — `c8y-oas.yml` `/event/events/{id}/binaries`

これは `tedge upload c8y`（必ず新規イベントを作る）の制約であって、プラットフォームの制約ではありません。c8y-proxy 経由で後付け可能です。現状の記述は「**イベントは MQTT で確実に送り、添付だけスプールして後付けする**」という網断耐性上の最良解を誤って排除しています。

### F-12. §2.2 main device の external ID に含まれる `:` が MQTT ClientId 制約と衝突
**該当**: §2.2・§2.7・§2.8 CK-1 ｜ 出典: [A1-15](factcheck/A1_sec0-2.md)

> the following format should be used for the ClientId: `connectionType:deviceIdentifier:defaultTemplateIdentifier` … **Important** — The colon character has a special meaning in Cumulocity. Hence, it must not be used in the `deviceIdentifier`. … During an SSL connection with certificates, the `deviceIdentifier` must match the 'Common Name' of the used certificate
> — https://cumulocity.com/docs/device-integration/mqtt/

thin-edge は Cumulocity への MQTT ClientId に `device.id` をそのまま使います（`crates/core/tedge/src/cli/connect/command.rs` L993）。したがって main device の external ID を `site001:ANLZ-SN00123` にすると、証明書 CN = `device.id` = ClientId がコロン入りになり、`connectionType=site001` / `deviceIdentifier=ANLZ-SN00123` と誤解釈されうる。**child device 側は `:` で問題ありません**（thin-edge の既定形が `<main>:device:<child>` で、SmartREST 101 経由で作られるため）。

**対処**: main device のみ区切りを `-` にする（例 `site001-ANLZ-SN00123`）。§2.7 の `device.id` 行を [要 TE-6] → [確] に格上げし、§2.8 CK-1 の正規表現も連動改訂。

### F-13. §2.8 CK-1 が参照する `GET /identity/externalIds` は存在しない
**該当**: §2.8 CK-1 ｜ 出典: [A1-30](factcheck/A1_sec0-2.md)

Identity API のパスは 4 つのみ（`/identity`、`/identity/search`、`/identity/globalIds/{id}/externalIds`、`/identity/externalIds/{type}/{externalId}`）で、**external ID の全件コレクション取得 API はありません**。

**対処**: 「`GET /inventory/managedObjects?fragmentType=c8y_IsDevice` で MO を全件列挙 → 各 MO に `GET /identity/globalIds/{id}/externalIds`」に修正。あわせて CK-1 の正規表現 `^[a-z0-9]+:...` は §1.4 の `^[a-zA-Z0-9]+$`（大文字許容）と不整合で、かつ拠点グループの external ID（`site001`、コロンなし）が必ず不合格になります。

### F-14. §13.1「`--mode prod` はエラーも出さずに何も投入せず完走する」は誤り
**該当**: §13.1 CF-c ｜ 出典: [A7-28](factcheck/A7_sec12-13.md)

> **session.mode** … Enable POST commands. **If set to false then all POST related commands will return an error.** / `prod` Create/Update/Delete commands are disabled, so essentially it is a read-only mode
> — https://goc8ycli.netlify.app/docs/configuration/settings/

`pkg/mode/mode.go` の `ValidateCreateMode` / `ValidateUpdateMode` / `ValidateDeleteMode` が `fmt.Errorf(...)` を返し、**非ゼロ終了**します。しかも**未指定時の既定が `SessionModeProduction`（= prod）**。結論（`C8Y_MODE=ci` を明示せよ）は正しいので、理由の書き換えのみで済みます。`--sessionMode` フラグの併記も推奨。

### F-15. §11.4「削除→同名で再作成するとユーザー ID が変わる」は誤り
**該当**: §11.4 R-08・§12.8 ｜ 出典: [A6-46](factcheck/A6_sec10-11.md)

`c8y-oas.yml` の `user` スキーマ example は `self: 'https://<TENANT_DOMAIN>/user/{tenantId}/users/jdoe'` / `id: jdoe` / `userName: jdoe`。**Cumulocity のユーザー ID はユーザー名そのもの**で、同名再作成なら ID は変わりません。結論（ロール・Inventory ロール・アプリケーションアクセスの再割当が必要）は正しいので、理由を「ユーザー削除に伴い割当も消えるため」に修正してください。前回レビュー M-10（CI シークレット同期）の懸念は、ID 変更ではなくパスワード再発行の同期として整理し直すのが正確です。

### F-16. §4.5 の「設定管理にロールバックは無い」は出典が誤っており、条件付きでしか成立しない
**該当**: §4.5・§16 ｜ 出典: [A8-25](factcheck/A8_citations.md)・[A3-21](factcheck/A3_sec4-5.md)

§16 が出典としている `/references/agent/tedge-configuration-management/` には、**2.0.1 で逆の記述**があります。

> **rollback (on error)**: The agent calls the plugin's rollback command … **to restore the previous configuration.**

「ロールバックなし」の逐語根拠は**別ページ** `/extend/config-management/` の *"…powered by the file plugin, **no rollback to the old configuration is attempted** to maintain backward compatibility."* です（1.7.1 版には "rollback" が 0 件 ＝ **2.0 の新機能**）。

**対処**: §4.5 の [確] を「**file プラグイン経由では**ロールバックされない。typed file-based（プラグイン実装側）はロールバック可」に限定し、§16 の URL を差し替える。

### F-17. §11.8 CORS はワイルドカード可、かつ現行ロール定義では設定できない
**該当**: §11.8 T-02・§13.3.1・§11.2 R-01 ｜ 出典: [A6-60](factcheck/A6_sec10-11.md)・[A7-31](factcheck/A7_sec12-13.md)

> **access.control** / `allow.origin` | `*` | Comma separated list of domains allowed for execution of CORS. **Wildcards are allowed (for example, `*.cumulocity.com`)**
> — `c8y-oas.yml` `POST /tenant/options` "Default option categories"

「スキーム＋ホスト＋ポートの完全一致」という記述は公式にありません。さらに必要ロールは **`ROLE_OPTION_MANAGEMENT_ADMIN`** で、**R-01（基盤運用者）に "Option management" が含まれていないため現行定義では CORS を設定できません**。

### F-18. §13.3.6「既定 60 日ルールを個別に削除」— そのオブジェクトの存在が確認できない
**該当**: §13.3.6 手順③・§13.7 AS-11・§7.2 ｜ 出典: [A7-49](factcheck/A7_sec12-13.md)・[A4-30](factcheck/A4_sec6-7.md)（2 名が独立に指摘）

> By default, all historical data is deleted after 60 days (**configurable in the system settings by the platform administrator**)
> — https://cumulocity.com/docs/2026/standard-tenant/managing-data/

「60 日」は公式に存在しますが、それが**テナント内に列挙・削除できるリテンションルールのオブジェクトとして存在する**という記述は docs / OpenAPI / Community のいずれにもありません（3 経路探索済み）。手順③が空振りし AS-11 が無意味な CI 検査になるうえ、**システム設定側の 60 日が残っていれば 90 日ルールを入れても 60 日で消える**可能性が排除できず、§7.2 のデータ保持設計そのものが崩れます。W1 で最初に確認すべき項目です。

### その他の Major（要点のみ）

| # | 内容 | 該当 | 出典 |
|---|---|---|---|
| F-19 | **`Inventory CHANGE` は READ を含まない**（*"does not include READ permission"*）。F-1 の修正後、拠点 Manager がデバイス一覧を見られなくなる → `Inventory ALL` にすべき | §11.3 R-04a | [A6-39](factcheck/A6_sec10-11.md) |
| F-20 | **ログインモードの実体が OpenAPI に存在しない。** `PUT /tenant/loginOptions/{typeOrId}` は Basic で叩けるが、`authConfig` schema に「preferred login mode」相当のフィールドが無く、`loginMode` の grep も 0 件。**SSO 切り戻しが loginOptions の PUT だけでは戻らないおそれ** — SV-04 の本当のリスク | §11.6 ガード 7 | [A6-57](factcheck/A6_sec10-11.md) |
| F-21 | **`expiresInMinutes` に最大値の定義は無い**（既定 1440 は正）。さらに *"If a token expires while its consumer is connected, the consumer is not automatically logged out or disconnected."* — 失効は完全な歯止めにならない | §10.4 AP-d | [A6-24](factcheck/A6_sec10-11.md) |
| F-22 | **`shared` は「冗長化」ではなく負荷分散。** *"each consumer client receives a non-overlapping subset (share) of the notifications"* — 冗長化目的で使うと通知が分散して欠ける | §10.4 要件 4 | [A6-25](factcheck/A6_sec10-11.md) |
| F-23 | **16KB 上限はアラームにも適用され、判定は「変換後」。** `flows.rs`(2.0.1) の `alarms_flow()` にも `limit-payload-size` がある。計測は変換で数倍に膨張する（`{"temperature":25}` → `{"temperature":{"temperature":{"value":25.0}}}`）ため、ローカル長の検査では不十分 | §6.7 表・S-d | [A4-13 / A4-14](factcheck/A4_sec6-7.md) |
| F-24 | **アラームの重複排除は ACTIVE または ACKNOWLEDGED。** 運用者が「確認済み」にした後も新規アラームが立たない、という監視の穴を見落としている | §8.5・§9.4 | [A5-15](factcheck/A5_sec8-9.md)・[A3-52](factcheck/A3_sec4-5.md) |
| F-25 | **`--creationRamp` は CLI フラグとして存在しない。** 正しくは **`--creationRampSec`**（`creationRamp` は API ボディのフィールド名）。CLI 実在性検証 15 項目で唯一の誤り | §13.3.9 M-02 | [A7-58](factcheck/A7_sec12-13.md) |
| F-26 | **SSO の `id` は POST では readOnly、PUT では required。** パターン B（PUT 先行）と「必ず `id` を除去」が矛盾。「GET で `id` 取得 → 載せて PUT → 無ければ `id` 抜きで POST」に修正 | §13.3.11 | [A6-51](factcheck/A6_sec10-11.md)・[A7-63](factcheck/A7_sec12-13.md) |
| F-27 | **sm-plugin の 5 分タイムアウトが未記載。** *"If the command fails to return within 5 minutes, the sm-agent reports a timeout error: `4`: timeout."* — AI モデルの install が 5 分を超えると失敗扱いになる | §4.4 | [A3-12](factcheck/A3_sec4-5.md) |
| F-28 | **「一度申告した supported operation は消せない」は過度な一般化。** 公式警告は「**動的**削除は非対応」。①操作ファイル削除 → ②マッパー再起動 → ③`signal/sync`（*"republishes the supported operations for the main device and all its child devices"*）で減らせる。SmartREST 114 も *"send a new 114 request with the updated list"* と明記 | §4.3・§4.5 | [A3-09](factcheck/A3_sec4-5.md) |
| F-29 | **`c8y_RemoteAccessConnect` は SSH/VNC に加え Telnet、thin-edge の `PASSTHROUGH` で任意 TCP が通る。** 封じ込めの必要性はむしろ本書の想定より強い | §4.7 | [A3-23](factcheck/A3_sec4-5.md) |
| F-30 | **R-13/AS-4 にロール名がなく CI 検査を実装できない。** 正しくは **`ROLE_REMOTE_ACCESS_ADMIN`**（既定でどのグループ／ユーザーにも未割当） | §13.3.4 | [A7-43](factcheck/A7_sec12-13.md) |
| F-31 | **TFA は `password` テナントオプションでは設定できない。** `PUT /tenant/tenants/{tenantId}/tfa` ／ `c8y tenants tfa update --strategy TOTP`。さらに **TOTP は OAI-Secure 限定**で、SSO redirect 下では break-glass の TFA 要件が成立しない | §13.3.4 A-05・§11.5 R-07 | [A7-44](factcheck/A7_sec12-13.md)・[A6-48](factcheck/A6_sec10-11.md) |
| F-32 | **Edge 同梱の Apama-ctrl バリアントが不明。** `Apama-ctrl-starter` では EPL Apps ページが使えず、`Apama-ctrl-smartrules` では Analytics Builder も EPL Apps も使えない。RL-b・RU-3〜RU-5 が全て EPL 前提なので §12.4 で実機確認必須（Edge docs 全ページを `apama|epl|cep` で grep してもヒット無し） | §8.1・§8.7 | [A5-02](factcheck/A5_sec8-9.md) |
| F-33 | **thin-edge のオフラインバッファ（ストア&フォワード）の規定が見つからない。** thin-edge が書く mosquitto ブリッジ設定に `max_queued_messages` が無く（`crates/core/tedge/src/bridge/config.rs`）、mosquitto 既定 1000 件を超えると「一括再送」ではなく**欠落**する可能性。§8.4 のリプレイ抑止設計は前提から要検証。CT-15 に欠落検証を追加すべき | §8.4 | [A5-12](factcheck/A5_sec8-9.md) |

---

## 4. 確度記号の格上げと検証工数の削減（🔵）

**[要] のまま残っているが、公式に答えがある項目**です。W1 の検証項目から削除・縮小できます。

| # | 項目 | 公式回答 | 効果 |
|---|---|---|---|
| 1 | **SV-33** Notification 2.0 の subscriber 数・backlog 監視 | **できる。** `Administration > Monitoring > Messaging Service` に Subscribers / Message backlog / Used backlog / Unacknowledged messages / Last acknowledged の列と**公式の危険域**（>5 / >20MB / >80% / >1000 / >=1day）。UI から unsubscribe も可。出典は**§16 に既載** | **項目ごと削除可。** 前提「取れない場合は棚卸しが唯一の歯止め」は**誤りなので本文修正が必要** |
| 2 | **SV-25 / TE-7** CA 更新時の再エンロール | **発生しない**（F-5） | CT-32 を大幅縮小。輻輳検証は「デバイス証明書 1 年満了の一斉更新」へ付け替え |
| 3 | **CU-13 / CT-13** severity 変更時の挙動 | **公式に答えがある。** `POST /alarm/alarms` の de-duplication に *"**Any other changes are ignored**"* ＝ severity 変更は無視される | CT-13 を「観測」→「予測確認 1 ケース」に縮小。型規約の確定を待たず進行可 |
| 4 | **SV-07** `ENROLLMENT_OTP` × `PATH` 併用 | **併用可。** OpenAPI の CSV ヘッダ表に両方が独立した任意列として並記。排他は `CREDENTIALS` × `AUTH_TYPE` のみ | [要]→[推]。**副産物**: 付録 A の α-1（BOX 全台の childAssets 追加）を `PATH` 列で代替でき、グループ自動作成・再実行冪等になる |
| 5 | **前提 4 / CT-8** Edge 上の Notification 2.0 | **公式機能。** 2026 機能比較表で `Messaging Service: Included` | 「使えるか」は [確]（検証不要）。「2026 での有効化手順」「WS の LB 到達」のみ [要] 継続。⚠️ **+2 CPU / +4GB RAM / PV 3 本**を §7.6・§12 に反映必須 |
| 6 | **SV-04 / CT-27** SSO → ローカル切り戻し | **API 経路は確定。** `PUT /tenant/loginOptions/{typeOrId}` は `security: Basic: []` を持つ。落とし穴 `BasicAuthenticationRestrictions.forbiddenClients/forbiddenUserAgents` もスキーマ化 | 「叩けるか」は [推]。ただし **F-20 の懸念が残る**ので runbook 化＋実機確認は継続 |
| 7 | **CT-22②** CLEARED 以外は消えない | *"Alarms are only removed if they have a status of CLEARED."*（**§16 に既載**） | 試験②を削除。あわせて **ALARM に `fragmentType` が効かない**制約を §7.2 へ |
| 8 | **SV-11** Cockpit Import/export | 2026 Edge introduction の Cockpit 機能列挙に *"Managing exports — Export data to either CSV or Excel files."* | 対象を書き分ければ半分削除可 |
| 9 | **CU-1** 拠点コード形式 | `pattern: '^[a-zA-Z0-9]+$'` が [確]（本レビューの発端） | 案 (a)（英数字統一）の根拠が確定。**§2.5 の決定を前倒しできる** |
| 10 | **CU-8** CORS 許可オリジン | ワイルドカード可・既定 `*`（F-17） | リスク評価を下げられる。ただしロール不足の修正は必要 |
| 11 | **SV-30** tenant options のマージ／置換 | *"**Update one or more options** (by a specified category)"* ＋ go-c8y-cli `updateBulk`「Update multiple tenant options in provided category」＝マージを示唆 | [要]→[推]。単一キー版 PUT を既定手段にすれば結果非依存にできる |
| 12 | **SV-06** 添付バイナリのリテンション | Tech Community に回答あり（*"Yes, that is one of the main reasons why binaries should be attached to events…"*）。かつ §6.8 の保存先欄は「同一保持期間で消える」と断定しており**文書内で矛盾** | [要]→[確K]。文書内矛盾の解消が必要 |
| 13 | **§6.7 16KB** | Cumulocity 公式に *"For all Core MQTT connections to the platform, the maximum accepted payload size is **16184 bytes (16KiB)**, which includes both message header and body."* ＋ `tedge_config.rs` の `C8Y_MQTT_PAYLOAD_LIMIT = 16184` | [確S]→[確]。さらに「拒否されても気づけない」は誤りで、`te/errors` に出る |
| 14 | **§5.3 child への 117 送出** | `availability/actor.rs` の `EntityType::ChildDevice => … send_smartrest_set_required_availability_for_child_device` | [推]→[確S] |
| 15 | **§4.7 Remote access** | socket 必須、および表示 3 条件（マイクロサービス subscribe / Remote access ロール / `c8y_RemoteAccessConnect` 申告）は公式に逐語明記 | [推]/[判]→[確] |
| 16 | **§1.5 `c8y_IsDeviceGroup`** / **§2.7 external ID の文字種** | 前者は spec に明記。後者は `externalId` スキーマに `pattern`/`minLength`/`maxLength` が**一切定義されていない**ため「Identity API 側は無制約 [確]」と断定可 | TE-6 は F-12 の main device 経路だけに絞れる |
| 17 | **§3.9 デバイス削除挙動** | `DELETE /inventory/managedObjects/{id}` に `cascade`（既定 false、*"all the hierarchy will be deleted"*）・`forceCascade`・`withDeviceUser`（*"it deletes the associated device user (credentials)"*）。削除は非同期 | 撤収 runbook に `withDeviceUser` を明記 |
| 18 | **§10.1 WebSocket 仕様** | 固定パス `/notification2/consumer/`、クエリは `token`（必須）と `consumer`（任意）のみ、443/80/`cumulocity:8111`、**アイドル 5 分でサーバ切断**、**未 ack 1000 件で配信停止** | SV-05 の疎通対象ポート確定と、案件アプリの再接続要件（SV-36）に直結 |
| 19 | **§8.2 / §8.8 EPL の外部連携** | *"…these include the Time Format and **HTTP Client > JSON** with generic request/response event definitions bundles."* | SV-10 を「プラグインの有無」→「Edge でのネットワーク許可」に論点を絞れる |
| 20 | **§4.8 `creationRamp` 必須化** | Bulk operations タグ説明に *"**It is required to specify the delay** between the creation of subsequent operations."* ＋ *"Bulk operations are an asynchronous, **best-effort** mechanism … **not a precision scheduling tool**"* | [判]→[確]。後者は §4.9 R-c（変更ウィンドウ）の設計に反映すべき |
| 21 | **§12.5 Edge TLS 証明書更新** | `c8yedge config --set-file tlsSecret.tls.{key,crt}` ／ `kubectl create secret tls edge-tls-secret …` → `spec.tlsSecretName`（F-6） | 前回レビュー C-5 の解消手順が確定 |
| 22 | **§1.6 RBAC バイパス** | *"⚠️ **Caution:** If you assign Notification 2.0 roles or permissions to users, they can create Notification 2.0 subscriptions and receive notifications for any device, including those to which assigned inventory roles do not grant access, **bypassing the inventory role RBAC**."* | この逐語引用を本文に入れておくと、後任のレビューで揺り戻しが起きない |
| 23 | **§9.1 `alarm.type.mapping` の出典** | **docs サイトのどのページにも書かれておらず**、Core OpenAPI の `POST /tenant/options` "Default option categories" 表にのみ逐語記述がある | **§16 にこの URL を明記すること。** 記載がないと後任が「根拠なし」と誤判定する |

**逆に [要] のままが妥当（誤って削らないこと）**: 前提 2（`alarmsWithChildren`）／ SV-19 ／ SV-32 ／ α-10（TE-13, child 数上限）／ F-18（リテンション既定ルール）／ F-20（ログインモード）／ F-32（Apama バリアント）／ F-33（オフラインバッファ）。

---

## 5. §16 出典の検証結果

**URL の実在性は全 39 件が HTTP 200。リンク切れ・リダイレクトすり替えは 0 件。** 内容一致は ✅33 / 🔶6 / ❌0。`cumulocity.com/docs` が存在しないパスに本物の 404 を返すことを事前確認済みで、ソフト 404 による偽陽性ではありません。

Core OpenAPI の行に列挙された 6 主張（RBAC バイパス／`NotificationSubscription`／`bulkNewDeviceRequests`／`creationRamp`・`groupId`・`failedParentId`／ロール名／パスワード制約）は**すべて生 spec に実在**しました。

**修正が必要な 6 件**:

| # | 出典行 | 問題 | 対処 |
|---|---|---|---|
| 1 | Edge `manage-edge` の「外部 IP とポート一覧」 | **網羅的なポート一覧表は存在しない。** `kubectl get service` の出力例に `443:32443,8443:32442,1883:32083,8883:32084 …`（末尾省略）があるのみ。Edge 全 12 ページを取得して grep 済み | 「内容」列から「ポート一覧」を削除し、実機の `kubectl get service` を正とする旨を §12.5 に記載 |
| 2 | Edge CRD「spec の 12 項目」 | **実際は 13 項目（見出し 10）で不一致。** ⚠️ 2025 版にあった `messagingService` / `core` / `microservices` / `dataHub` が **2026 版で消失** | 項目数を修正。消失した 4 項目は §12 の設計に影響しうるため要確認 |
| 3 | thin-edge `operate/entity-management/` | **カテゴリ索引のみ。** 実体は `/auto-registration/`（アンダースコア注意）、`/mqtt_api/`、`/rest_api/` | 子ページ URL に差し替え |
| 4 | thin-edge `references/mqtt-api/`（QoS の根拠） | **当該ページに QoS の記述は 0 件**（`grep -ci qos` = 0）。根拠は `/start/raise-alarm/` の *"…with **QOS > 1** to ensure guaranteed processing"*。同ページには §6.5 が [確] 引用する *"Every alarm is uniquely identified by its type and severity."* もある | `/start/raise-alarm/` を出典に追加 |
| 5 | thin-edge `references/agent/tedge-configuration-management/`（ロールバックなし） | **当該ページと矛盾**（F-16） | `/extend/config-management/` に差し替え、主張を限定 |
| 6 | thin-edge 全 URL | 全ページのヘッダが `Version: 2.0.1` を表示するため現時点では正しいが、**全 URL がバージョン無しパス**で `/2.0.1/` permalink は 404（アーカイブは `1.7.1` と `next` のみ）。**2.1 リリースで内容が変わる** | 「参照時点 2026-08-20 / v2.0.1」の注記を追加 |

**追加を推奨する URL**: `alarm.type.mapping` の唯一の根拠である Core OpenAPI `POST /tenant/options`、Remote Access、Edge Messaging Service、bulk registration の CSV ヘッダ表。

**その他**: パスワード制約は OpenAPI が `minLength: 6, maxLength: 32` なのに対し **UI 既定は 8 文字**で食い違います（§12.8 に注記推奨）。また行 2097 と行 750 の引用符付き記述が逐語でない点は [A8_citations.md](factcheck/A8_citations.md) に記載しています。

---

## 6. 決着した最重要論点 — `alarmsWithChildren` の「子」

**結論: どの関係（`childDevices` / `childAssets` / `childAdditions`）を辿るのかは、公式一次情報のどこにも記述がありません。** 前回レビュー C-1 は「§1.1・§10.2 の [確] を [推] に格下げする」方向で解消するのが正解です。

見つかった記述は関係を特定しない語のみで、3 種類しかありません。

> The `alarmsWithChildren` and `eventsWithChildren` APIs subscribe to alarms and events respectively from the managed object identified by the `source.id` field, and **all of its descendant managed objects**.
> — `c8y-oas.yml` `subscriptionFilter.apis`（latest / 2026 / 2025 で同文）

> …allow subscriptions with managed object context to filter in, respectively, alarms or events for **all recursively descendant child managed objects of the source.id managed object**…
> — `c8y-oas.yml` §Overview > Subscription filters

> …allows alarm or event notifications from both the managed object that the subscription is created for, **as well as all it's children (transitively)** to be subscribed to.
> — Tech Community KB 3964

**「無い」ことの根拠（7 経路）**: ①Core OAS latest 全文 grep（Notification 2.0 の文脈に `childAssets`/`childDevices`/`childAdditions` は**一切出現せず**）②OAS 2026 / 2025（latest と同文、差分なし）③Edge OAS（`alarmsWithChildren` 0 件）④Community 全文検索 API（3 トピックのみ、全て `/raw/` で全文取得）⑤Community 追加検索（関係名に言及する投稿 0 件）⑥WebSearch 2 パターン ⑦docs `/concepts/domain-model/`。

**決定的な対比材料**: 同じ Cumulocity の Alarm/Event REST API は**明示的に区別しています**。

```yaml
withSourceAssets:    'alarms for related source assets will also be included'
withSourceDevices:   'alarms for related source devices will also be included'
withSourceAdditions: ...
```

2025 年 9 月追加の `withSourceChildren` の告知も *"alarms and events for related source **assets, devices and additions**"* と 3 種を明示列挙しています。**区別する語彙が製品内に存在するのに、Notification 2.0 の説明ではあえて使われていない** — この非対称は「全部辿る」「片方だけ」どちらの可能性も残します。

### 同型の未確認事項が Inventory ロール側にもある

前回レビュー C-2 の懸念が一次情報で裏付けられました。

> **Inventory roles are inherited from groups to all their direct and indirect subgroups, and to the devices in these groups.**

保証されているのは「グループ → サブグループ → **それらのグループに属するデバイス**」までで、**デバイス配下の子デバイス（カメラ・BOX）は不記載**です。通知側だけ確認して安心すると、REST 側で分離が破れます。

### 推奨する実機確認手順（両方を同時に検証すること）

1. 拠点グループ G の `childAssets` にサブグループ G1、G1 の `childAssets` にデバイス D、**D の `childDevices` にカメラ C** をぶら下げる
2. `{context:"mo", source:{id:G}, subscription:"testsite", subscriptionFilter:{apis:["alarmsWithChildren"]}}` を作成して WebSocket 接続
3. **(a) D にアラーム / (b) C にアラーム** を流し、どちらが届くかで判定（(a) のみ → `childAssets` のみ、(b) も → `childDevices` も辿る）
4. **同じ構成で、拠点 Manager ユーザーが C のアラーム・インベントリを REST で取得できるかも確認**（Inventory ロール側）
5. 文書化されていない挙動に設計の根幹を預けるため、**プロダクトサポートへの書面照会も推奨**（OAS が 3 版で不変なので短期の変更リスクは低い）

---

## 6b. 追加調査 — external ID は必要か

**設計者からの「そもそも external ID は不要ではないか」という問いに対する回答です。** 全文は [B1_externalid.md](factcheck/B1_externalid.md)。

### 結論: デバイスとグループで答えが正反対

| 対象 | 判定 | 理由 |
|---|---|---|
| **main device** | ⚠️ **回避不能。しかも値が拘束される** | `tedge cert create --device-id <X>` の `<X>` が **証明書 CN = MQTT ClientId = external ID = MO の `name`** すべてに固定される。EST の CSR も *"CSR must be a valid `PKCS#10` with **deviceID as Common Name (CN)**"* |
| **child device** | ⚠️ **回避不能**（自動生成される） | thin-edge が **`<main-device-id>:device:<topic-id>`** を自動生成（`/` → `:`）。`@id` を明示すればその値が使われる |
| **グループ** | ⭐ **不要。廃止できる** | `managedObject` スキーマに必須項目は 1 つも無い。**Cumulocity 自身が bulk 登録の `PATH` 列で名前ベースにグループを冪等生成しており、external ID を付けていない** |

**つまり本書が問うべきだったのは「external ID が必要か」ではなく「自動生成に任せるか、意味のある値を与えるか」でした。** 「採番しない」という選択肢はデバイスには存在しません。

### この調査で見つかった 3 件の追加不具合

**1. `site001:ANLZ-SN00123` は main device に使えません（❌ High）**

> *"**The colon character has a special meaning in Cumulocity. Hence, it must not be used in the `deviceIdentifier`.**"*
> *"Set CLIENT_ID to the common name of the device certificate. **The certificate common name must not contain `:`**."*
> — https://cumulocity.com/docs/device-integration/mqtt/

A1-15 の指摘（§3 の F-12）が、証明書 CN 側の逐語根拠でも裏付けられました。**child は `:` のままで問題ありません**（thin-edge の自動生成値自体がコロンを含む）。

**2. §13 G-01 の「409 黙殺」は誤り（❌ High）**

`POST /inventory/managedObjects` の応答定義は **`201` / `401` / `403` / `422` のみで 409 がありません**。一意制約を持つ API（`POST /user/{t}/users`、`POST /notification2/subscriptions`）は 409 を返すので、これは仕様上の意図です。**再実行のたびに同名グループが黙って増殖します。**

⚠️ **一方、`childAssets` への追加（D-08）の 409 黙殺は妥当**でした — go-c8y-cli 公式例に *"…to **silence 409 (duplicate) errors, as we don't care if the device is already assigned to the group**"* とあります（SV-38 解消）。**MO の作成は 409 を返さないが、childAssets への追加は返す**という非対称に注意が必要です。

**3. 「拠点コード = external ID = subscription 名」を 1 値で通す必然性は無い（🔶 Mid）**

- `subscription` は購読の名前であって MO の external ID ではない（*"Several subscriptions can share the same name"*）
- サブスクリプションが MO を指すのは **`source.id` = 内部 ID**（例 `id: '251982'`）
- 一意キーは (`source`, `context`, `subscription`) の三つ組
- **`^[a-zA-Z0-9]+$` は Core OpenAPI 全体で唯一の `pattern:` 宣言**で、他の識別子に文字種制約は無い

**§1.4 の「1 つの値で通せる」は [確] ではなく [判] です。** 実務上は推奨できますが、「そうしないと動かない」という書き方は誤りでした。

### 追補: MO 登録 API は重複チェックをしているか

**API ごとに挙動が正反対で、しかも API のバージョンによって差があります。**

| API | 重複チェック | 根拠（Release 2026 OpenAPI） |
|---|---|---|
| `POST /inventory/managedObjects` | ❌ **無い** | 応答定義は `201`/`401`/`403`/`422` のみ |
| `POST /identity/globalIds/{id}/externalIds` | ✅ **409** | *"Duplicate – Identity already bound to a different Global ID."* |
| `POST .../{id}/childAssets` | ✅ **409** | go-c8y-cli 公式例が `silentStatusCodes` で黙殺している |
| `POST /user/inventoryroles` | ✅ **409** | "Duplicate" |
| `POST /devicecontrol/newDeviceRequests` | ❌ **無い** | `201`/`401`/`403`/`422` のみ |
| SmartREST `100`（MQTT） | ✅ **find-or-create** | *"Create a new device for the serial number in the inventory **if not yet existing**."* |

**⚠️ 原子的な「external ID 付き MO 作成」は Edge 2026 では使えません。** **`Latest version`**（Release 2026 より新しい継続更新チャネル）の spec には `managedObjectCreate.externalIds` があり、

> *"Given external identifiers must be valid and **not be already assigned to another managed object** or the request … will **fail with a client error and not create anything**."*

**しかし Release 2026 / 2025 の spec には 0 件です**（`Latest version` のみ 1 件。3 版を実際に取得して grep で確認）。

**これは製品差ではなくバージョン差です** [確]: *"Cumulocity Edge release 2026 uses the **Cumulocity platform release 2026**."* / *"Edge uses the **same software** as Cumulocity platform … differences regarding the **activated optional features and pre-installed agents**."* つまり「Edge だから削られている」のではなく、Release 2026 の時点でまだ無いだけで、**Edge の版が上がれば使えるようになります**。CU-15 はこれで決着しました（spec ≠ 実装なので、検証環境で 1 度だけ実挙動を確認すると確実）。

**帰結 — REST 経由の MO 作成は「重複」ではなく「孤児」を生みます。**

```
① POST /inventory/managedObjects            → 201（重複チェックが無いので必ず成功）
② POST /identity/globalIds/{id}/externalIds → 409（既に別の MO に bound）
```

①が成功して②が失敗するため、**external ID を持たない MO が残ります**。②の 409 を「冪等だから」と黙殺する実装では気づけません。これは「グループが重複する」より追いにくい壊れ方です。

一方 **デバイス側（thin-edge / MQTT）は安全**です。SmartREST 100 が external ID をキーに find-or-create するため、再接続で重複しません。**壊れるのは REST を叩くプロビジョニングツールの経路だけ**です。

→ 文書には §13.5 に仕様表と帰結、§2.8 に **CK-1d（孤児 MO の検出）**、§3.5・§13.3.3 に「作成前に存在確認」を追記しました。

### 副次的に確定したこと

- **TE-6（external ID の文字種・長さ）は実機確認不要** — Identity API 側は `pattern` / `minLength` / `maxLength` の定義が一切なく**無制約**。制約が効くのは main device の証明書 CN 経路だけ
- **SV-07（`ENROLLMENT_OTP` × `PATH` 併用）は併用可** — CSV ヘッダ表に両方が独立した任意列として並記。排他は `CREDENTIALS` × `AUTH_TYPE` のみ。**副産物として付録 A の α-1（BOX 全台の childAssets 追加）を `PATH` 列で代替できます**
- **§1.5 にグループの `type` が欠けていた** — root = `c8y_DeviceGroup` / サブ = `c8y_DeviceSubGroup`（go-c8y-cli の `--excludeRootGroup` がこの値を前提にしている）
- **§2.2 X-d の理由が §3.9 と矛盾していた** — 「交換時に同一性を追跡できる」としつつ、§3.9 は「交換時は新 external ID を振る」としていた
- **`c8y devices get --id "c8y_Serial:xyz"` は動かない** — `IsID()` が数値のみを ID 判定するため名前検索になり 0 件。`c8y identity get` の 2 段構えが必要

---

## 7. W1 で最初に潰すべき項目（優先順）

| 順 | 項目 | 理由 |
|---|---|---|
| 1 | **F-1 グローバルロールの削減** | 文書修正のみで解消可能。拠点分離の成否を握る |
| 1b | **グループ作成の冪等化を「存在チェック」に変更**（§6b） | 文書修正のみで解消可能。**放置すると再実行のたびに拠点グループが増殖** |
| 1c | **main device の external ID からコロンを除去**（§6b） | プロビジョニング開始前でないと全拠点の再登録になる |
| 2 | **§6 の実機確認**（`alarmsWithChildren` ＋ Inventory ロール継承） | 前提 2。崩れると §1・§10・§11 が同時に破綻 |
| 3 | **F-2 / F-3 死活監視の設計やり直し** | 現行設計では死活アラームが出ない／設計値が反映されない |
| 4 | **F-18 リテンション既定ルールの存在確認** | §7.2 のデータ保持設計が崩れる可能性 |
| 5 | **F-4 アラームマッピング前方一致** | 型名規約（§6.4）の確定に必要。C-1 相当の結節点 |
| 6 | **F-32 Edge の Apama-ctrl バリアント** | RL-b・RU-3〜RU-5 が全て EPL 前提 |
| 7 | **F-6 の反映と Edge TLS 更新 runbook 化** | 定期作業が手順不在のまま放置されている |
| 8 | **F-20 ログインモードの実体確認** | SSO 切り戻しの可否。ロックアウトの実リスク |

**逆に、W1 から外してよい／縮小できるもの**: CT-32（CA 更新輻輳 → F-5）、SV-33（N2.0 監視 → 公式 UI あり）、CT-13（severity 変更 → 公式に答えあり）、CT-22②、SV-11、CU-1。詳細は §4 の表。

---

## 8. 総括

**本書の事実精度は高く、約 350 件の主張のうち一次情報と矛盾したのは 15 件（4%）です。** SmartREST テンプレート番号、OpenAPI のフィールド名と説明文、thin-edge の設定キーとソース上の既定値、CLI コマンドとフラグ名まで、[確] の大半が逐語で正確でした。確度記号の運用も規律が高く、むしろ **[要]/[推] のまま置かれている 23 件に公式の答えがあり、検証工数を削減できます**。

誤りには明確な偏りがあります。**「Cumulocity が X する」と書かれた箇所のうち、実際には thin-edge が X する（または誰も X しない）ものが集中しています** — F-3（117 の上書き主体）、F-10（時刻付与の主体）、F-11（添付の制約主体）、F-16（ロールバックの実装主体）。責務の境界を跨ぐ記述を書くときは、**どちらのレイヤの一次情報を見たかを出典に明記する**と再発を防げます。

もう一つの偏りは、**リスク分析の型（日付起点の一斉障害）を、事実確認より先に当てはめてしまったこと**です。F-5（CA 更新）はその典型で、分析の枠組み自体は優れているものの対象を取り違えており、存在しないリスクに検証工数が向けられていました。同じ枠組みは「デバイス証明書の初回エンロール日集中」に付け替えれば有効に機能します。

最後に、本レビューの過程で **「一次情報に記述がない」という誤判定が 3 回**（うち 1 回は実施者自身）起きました。今回の発端と同じ失敗です。本書の [確] を疑うときは、**探索経路を明示できないうちは否定しない**でください。§6 の `alarmsWithChildren` は、7 経路を尽くしたうえで「書かれていない」と結論できた例です。

---

**改版履歴**

| 版 | 日付 | 内容 |
|---|---|---|
| rev.1 | 2026-08-20 | 初版。8 名の独立レビュアーによる一次情報照合。事実主張 約 350 件・出典 URL 39 件を検証 |
