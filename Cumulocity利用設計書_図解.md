# Cumulocity利用設計書 図解(PlantUML)

`Cumulocity利用設計書.md` の「紐づき」に関する記述を PlantUML で図示したものです。
**正は設計書本文**であり、本ファイルは理解補助です。本文改訂時はこちらも追随させてください。

- 図1: デバイス管理の紐づき(資産階層と接続階層) — §1.2・§1.3・§2.2〜§2.4・§3.5
- 図2: データの帰属(source)と昇格 — §6.2・§6.4・§8.2
- 図3: イベントとスナップショット画像の紐づき — §6.8・§15.3 CU-18

描画は VS Code の PlantUML 拡張、または `java -jar plantuml.jar` で行えます。

---

## 図1 デバイス管理の紐づき — 資産階層と接続階層

設計書 §1.3 が「本書で最も間違えやすい箇所」と呼ぶ、**childAssets(資産階層・業務上の所属)と childDevices(接続階層・通信上の親子)が別物**である点を中心に描いています。実線白抜きひし形が childAssets、黒ひし形が childDevices です。

```plantuml
@startuml device-hierarchy
title デバイス管理の紐づき(案β・§1.2/§1.3)

object "拠点 root グループ" as root {
  type = c8y_DeviceGroup
  c8y_IsDeviceGroup
}

object "拠点グループ「拠点A」" as siteA {
  type = c8y_DeviceSubGroup
  c8y_IsDeviceGroup
  x_Site (siteId, siteName) ←同定キー
  external ID: 振らない(§1.4)
}

object "画像解析装置 (main device)" as anlz {
  type = x_ImageAnalyzer
  external ID: site001-ANLZ-SN00123 (c8y_Serial)
  x_Site (siteId)
}

object "IPカメラ ×N (child device)" as cam {
  type = x_Camera
  external ID: site001:CAM-SN12345
  x_Camera (vmsCameraId※必須, location)
  x_Site / c8y_RequiredAvailability=32767
}

object "BOXゲートウェイ (main device)" as gw {
  type = x_BoxGateway
  external ID: site001-GW-BOX (c8y_Serial)
  x_Site (siteId)
}

object "画像センシングBOX ×M (child device)" as box {
  type = x_SensingBox
  external ID: site001:BOX-SN98765
  x_SensingBox (model, fwVersion, channels)
  x_Site / c8y_RequiredAvailability=32767
}

root o-- siteA : childAssets
siteA o-- anlz : **childAssets(資産階層)**\nプロビジョニングツールが明示追加(§3.5 手順3)
siteA o-- gw : **childAssets(資産階層)**\nプロビジョニングツールが明示追加(§3.5 手順3)
anlz *-- cam : **childDevices(接続階層)**\nthin-edge A が自動生成
gw *-- box : **childDevices(接続階層)**\nthin-edge B が自動生成

note top of siteA
  拠点グループが単位になるもの(§1.1):
  ・Inventory ロールの割当
  ・Notification 2.0 の source
  ・bulk operation の groupId
end note

note bottom of root
  **MO と external ID の作られ方(§2.2・§3.5)**
  ・main device: thin-edge 初回接続時に SmartREST 100 が find-or-create。
  　external ID は証明書 CN と同一値(コロン使用不可・§2.2 X-f)
  ・child device: プロビジョニングツールが thin-edge Entity API で代理登録。
  　@id 指定値がそのまま external ID になる(§2.6 E-b)
  ・child は childAssets に入れない [判](§1.3 — 接続階層に任せる)

  ⚠ Inventory ロール継承・alarmsWithChildren 購読が
  　childDevices / childAssets の先まで届くかは未検証(前提2・SV-35 / SV-47)
end note
@enduml
```

読み取りポイント:

- **childAssets はプロビジョニングツールの明示操作、childDevices は thin-edge の自動生成**。拠点あたりの明示操作は main device 2 台ぶんの 2 回だけで済む(案βの効果・§1.2)。
- 拠点グループに external ID は無く、`x_Site.siteId` が唯一の同定キー(§1.4 rev.2)。
- main device の external ID は区切りがハイフン(証明書 CN 制約)、child はコロン(§2.2 X-c)。
- 図中の「⚠未検証」は、拠点分離設計全体が依存する最重要検証項目(前提2)。

---

## 図2 データの帰属(source)と昇格

§6.2 の警告「`source` は"そのデータが誰に帰属するか"。検知イベントの source は画像解析装置ではなく対象カメラ」を図にしたものです。矢印はすべて `source`(帰属先)を指します。

```plantuml
@startuml data-source
title データの帰属(source)と昇格(§6.2・§6.4・§8.2)
left to right direction

object "IPカメラ (child)" as cam
object "画像センシングBOX (child)" as box
object "画像解析装置 (main)" as anlz
object "BOXゲートウェイ (main)" as gw

object "x_CameraHealth" as mHealth <<Measurement>> {
  rtt / reachable
  生成元: ONVIF死活監視サービス
}

object "x_Detection_＜種別＞" as evDet <<Event>> {
  x_Detection (eventUuid, modelVersion,
  aiProduct, confidence, boundingBoxes, clipHint)
  生成元: 画像解析パイプライン / BOXアダプタ
}

object "x_ModelApplied" as evModel <<Event>> {
  modelVersion
  生成元: sm-plugin aimodel
}

object "x_CameraDown" as alCam <<Alarm MAJOR>> {
  生成元: ONVIF死活監視サービス
}

object "x_BoxSilent" as alSilent <<Alarm MAJOR>> {
  生成元: BOXアダプタ
}

object "x_BoxParseError" as alParse <<Alarm WARNING>> {
  生成元: BOXアダプタ
}

object "x_ModelUpdateFailed" as alModel <<Alarm CRITICAL>> {
  生成元: sm-plugin aimodel
}

object "c8y_UnavailabilityAlarm" as alUnav <<Alarm MAJOR>> {
  生成元: Cumulocity 自身
  (c8y_RequiredAvailability 判定)
}

object "x_Alarm_＜種別＞" as alBiz <<Alarm>> {
  severity: 案件側が決定
  生成元: Smart Rules / EPL
}

mHealth --> cam : source
evDet --> cam : source\n(パイプライン発)
evDet --> box : source\n(BOXアダプタ発)
evModel --> anlz : source
alCam --> cam : source
alSilent --> box : source
alParse --> gw : source
alModel --> anlz : source
alUnav --> anlz : source
alUnav --> gw : source
alBiz --> cam : source
alBiz --> box : source

evDet ..> alBiz : **昇格(RU-1・§8.2)**\nSmart Rules / EPL が\n条件一致分だけ Alarm 化

note bottom of evDet
  ⚠ source は「誰に帰属するか」(§6.2)。
  検知の source を装置にすると、カメラ単位の
  権限(§11.3)・購読(§10.2)ができなくなる。
end note

note bottom of alBiz
  検知イベントを直接 Alarm にしない(§6.1)。
  Alarm は同一 (source, type) で重複排除されるため
  検知件数が正しく残らない。
  Alarm は必ずクリア条件とセットで定義(§9.6)。
end note
@enduml
```

読み取りポイント:

- **Measurement は死活の定期記録 1 種のみ**。検知結果は Measurement で送らない(§6.1)。
- 業務アラーム `x_Alarm_＜種別＞` だけは点線(昇格)で生まれる。デバイスが直接発報するのではなく、Edge 内の Smart Rules / EPL が `x_Detection_＜種別＞` から条件一致分を昇格させる(§8.2 RU-1)。
- `c8y_UnavailabilityAlarm` だけは生成元が Cumulocity 自身であり、デバイス側実装では制御できない。

---

## 図3 イベントとスナップショット画像の紐づき

現行設計(イベント添付・§6.8)と、検討中の案(D)(`/inventory/binaries` 格納・§15.3 CU-18)の構造の違いです。**右側の案(D)は [要決定] であり、まだ採用されていません。**

```plantuml
@startuml binary-linkage
title スナップショット画像の紐づき — 現行 と 案(D)(§6.8・§15.3)

package "現行設計: イベント添付(§6.8)" {
  object "x_Detection_＜種別＞ (Event)" as ev1
  object "スナップショット (添付 Binary)" as bin1

  ev1 *-- "0..1" bin1 : 添付

  note bottom of bin1
    POST /event/events/{id}/binaries
    ・**1イベント1バイナリ**(2件目は 409・制約2)
    ・リテンション: イベントと同一保持期間で消える [確K]
    ・取得権限をイベント閲覧と分離できない(制約6)
    ・参照/DL は監査ログに残らない(制約5)
  end note
}

package "案(D): ファイルリポジトリ格納 [要決定 CU-18]" {
  object "x_Detection_＜種別＞ (Event)" as ev2 {
    x_SnapshotRef (fullId, thumbId)
  }
  object "バイナリ MO(原本)" as binFull
  object "バイナリ MO(サムネイル)" as binThumb

  ev2 ..> binFull : fullId で参照
  ev2 ..> binThumb : thumbId で参照

  note bottom of binFull
    POST /inventory/binaries(1件 = MO 1件)
    ・複数ファイル可(「添付スロット」という概念自体が無い)
    ・参照関係は x_SnapshotRef で**アプリ側が維持**
    　(プラットフォームの添付機能ではない)
    ・権限分離可: 直叩きは 403 が期待できる
    　(仕様からの推論。CT-5c / CU-23 で実機確認待ち)
    ・⚠ リテンションが効かない → 削除バッチ必須(§7.4)
    ・⚠ 拠点グループ配下にぶら下げないこと(CK-8)
    ・⚠ 参照/DL の無監査(制約5)は解消しない
  end note
}
@enduml
```

読み取りポイント:

- 現行の「1イベント1バイナリ」はイベント添付という仕組み固有の制約。案(D)はイベントから独立したバイナリ MO を参照キーで結ぶため、原本+サムネイルの複数持ちができる。
- 引き換えに、リテンション(自動削除)・MO 総数(SV-14b)・削除バッチ運用のコストを負う。
- どちらの案でも制約5(参照・DL が監査ログに残らない)は残るため、監査レコードの自前書き込みは別途必要(§15.3)。
