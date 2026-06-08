**基本的に Event Adapter は各製品に最初から含まれる標準機能ではなく、自前で作る連携部品** と考えるのが正しいです。

ただし、完全にゼロから作るというより、各製品が持っている **SDK/API/モジュール実行基盤/ルーティング機能** を使って作る、という位置づけです。

---

# 結論

```text
Event Adapter
= AI解析結果やVMSメタデータを、
  Cumulocity / Azure IoT Central / Azure IoT Hub が扱える形式へ変換して送信する自作コンポーネント
```

役割はこんな感じです。

```text
AI解析結果
{
  camera: "cam01",
  label: "person",
  confidence: 0.94
}

        ↓ Event Adapter

IoT基盤向けデータ
{
  eventType: "PersonDetected",
  cameraId: "CAM-001",
  eventTime: "2026-06-08T01:00:05.123Z",
  personCount: 1,
  confidence: 0.94,
  vmsClipId: "clip-xxxx",
  correlationId: "site001-CAM001-..."
}
```

つまり、Event Adapterは **通訳兼荷造り係** です。
AIの荒削りな検知結果を、IoT基盤が飲み込みやすい形に詰め直す小さな職人さん、という感じです 🧰

---

# なぜ標準機能だけでは足りないのか

各製品には、データを受ける仕組みはあります。

| 製品                       | 標準で持っているもの                                                      | Event Adapterは標準？               |
| ------------------------ | --------------------------------------------------------------- | ------------------------------- |
| Cumulocity               | REST API、MQTT、SmartREST、デバイス管理、イベント/アラーム/メジャメント                 | いいえ。連携ロジックは自作                   |
| Azure IoT Central        | Device Template、Telemetry、Properties、Commands、Rules             | いいえ。送信側の整形ロジックは自作               |
| Azure IoT Hub + IoT Edge | IoT Hub D2C Message、Device Twin、Message Routing、IoT Edge Module | いいえ。ただしIoT Edge Moduleとして実装しやすい |

CumulocityはMQTTによるデバイス統合やREST APIを提供していて、デバイス認証・登録・管理などを扱えますが、AI解析結果をどういうイベント型にするか、VMSのclipIdをどう付与するか、といった業務固有の変換は自分で実装します。([cumulocity.com][1])

Azure IoT CentralではDevice Templateがテレメトリ・プロパティ・コマンドの形を定義しますが、そのテンプレートに合わせたJSONを現場側で作って送る処理は別途必要です。([Microsoft Learn][2])

Azure IoT Hub + IoT Edgeでは、IoT Edge Moduleとしてコンテナ化した処理を現場PCに配置し、edgeHubでルーティングできます。なので3構成の中では、Event Adapterを一番自然に「製品の上に載せる部品」として実装しやすいです。([Microsoft Learn][3])

---

# Event Adapterがやること

Event Adapterはだいたい以下を担当します。

```text
1. AI解析結果を受け取る
2. カメラIDをIoT基盤側のIDへ変換する
3. VMSのclipId / playbackUrl / thumbnailUrlを付与する
4. eventTimeをUTC ISO 8601形式にそろえる
5. correlationIdを生成する
6. personCount / confidence / zoneIdなどを正規化する
7. Cumulocity / Azure向けのJSONへ変換する
8. MQTT / HTTPS / RESTで送信する
9. 送信失敗時にローカルキューへ保存する
10. 再送時の重複排除用にmessageId / sequenceNoを付ける
```

図にするとこうです。

```text
[EdgePC1: AI映像解析]
   │
   │ AI解析結果
   ▼
┌────────────────────────────┐
│ Event Adapter               │
│                              │
│  入力変換                    │
│   - AI結果JSON受信            │
│   - カメラID変換              │
│                              │
│  付加情報                    │
│   - vmsClipId取得             │
│   - correlationId生成         │
│   - eventTime正規化           │
│                              │
│  出力変換                    │
│   - Cumulocity形式            │
│   - IoT Central形式           │
│   - IoT Hub形式               │
│                              │
│  信頼性                      │
│   - Local Queue               │
│   - Retry                     │
│   - Deduplication             │
└──────────────┬─────────────┘
               │
               ▼
[EdgePC2: IoT基盤側エッジ/ゲートウェイ]
```

---

# 構成ごとの実装位置

## 1. Cumulocity構成

```text
[EdgePC1: AI映像解析]
   │
   ▼
[Event Adapter: 自作]
   │ REST / MQTT / SmartREST
   ▼
[EdgePC2: Cumulocity Edge]
   │
   ▼
[Cumulocity Platform / Core]
```

Cumulocityの場合、Event Adapterは以下のどちらかで作るのが自然です。

```text
方式A: EdgePC1上の自作プロセス
方式B: EdgePC2側に置くCumulocity Microservice / 外部サービス
```

おすすめは **方式AまたはEdgePC2外部の軽量サービス** です。
理由は、VMS APIやAI推論結果に近い場所で処理した方が、映像側との紐づけがしやすいからです。

CumulocityにはOPC UA Gatewayのような標準ゲートウェイもありますが、それはOPC UA機器との連携向けです。VMS連携やAI検知結果の業務変換は、案件固有の実装になります。([cumulocity.com][4])

---

## 2. Azure IoT Central構成

```text
[EdgePC1: AI映像解析]
   │
   ▼
[Event Adapter: 自作]
   │ MQTT / HTTPS
   ▼
[EdgePC2: Azure IoT Edge]
   │
   ▼
[Azure IoT Central]
```

IoT Centralの場合も、Event Adapterは自作です。

IoT Central側でできるのは、

```text
Device Templateでデータ構造を定義する
Telemetryとして受ける
Rulesで条件判定する
Dashboardで表示する
Data Exportで外部へ出す
```

という部分です。

でも、以下は標準機能だけでは決まりません。

```text
AI結果をどのTelemetry名にするか
cameraIdをどう変換するか
vmsClipIdをどこから取るか
correlationIdをどう作るか
confidenceの閾値をどこで判定するか
```

なので、現場側にEvent Adapterが必要です。

---

## 3. Azure IoT Hub + Azure IoT Edge構成

```text
[EdgePC1: AI映像解析]
   │
   ▼
[Event Adapter: 自作IoT Edge Module]
   │
   ▼
[edgeHub]
   │
   ▼
[Azure IoT Hub]
```

この構成では、Event Adapterは **Azure IoT Edge Moduleとして自作する** のが一番きれいです。

IoT Edgeではモジュールをコンテナ化してデバイス上で動かし、deployment manifestでモジュールとルートを定義できます。([Microsoft Learn][5])
edgeHubはIoT Hub互換のメッセージングやルーティングをサポートするため、Event Adapter ModuleからedgeHubへ送れば、その後はIoT Hubへ流せます。([Microsoft Learn][3])

この場合はこうなります。

```text
EdgePC2: Azure IoT Edge
────────────────────────────
edgeAgent
edgeHub

自作Module:
  - AI Event Adapter
  - Sensor Adapter
  - VMS Metadata Adapter
  - Time Sync Monitor
```

IoT Hub側では、D2Cメッセージは時系列テレメトリやアラート向け、Device Twinのreported propertiesは状態報告向け、File Uploadは動画など大きいファイル向け、という使い分けが説明されています。([Microsoft Learn][6])
今回のAIイベントやメトリクスは基本的に **D2C Message** で送るのが自然です。

---

# どこまでが製品標準で、どこからが自作か

```text
┌────────────────────┬──────────────────────────┬────────────────────────────┐
│ 項目                 │ 製品標準でできること       │ 自作が必要なこと              │
├────────────────────┼──────────────────────────┼────────────────────────────┤
│ デバイス接続          │ MQTT/HTTPS/AMQP/REST等     │ VMS/AI結果の業務変換           │
│ データ受信            │ Telemetry/Event受信        │ JSON項目設計・正規化            │
│ デバイス管理          │ Device/Twin/Inventory      │ カメラ台帳とのマッピング         │
│ ルーティング          │ EdgeHub/IoT Hub Routing等  │ messageTypeや条件設計           │
│ アラーム              │ Rules/Functions等          │ どの条件を異常とするか           │
│ 映像保存              │ 対象外                     │ VMS連携、clipId取得             │
│ 映像リンク            │ 一部URL表示は可能           │ 認証付きURL発行、権限制御         │
│ データ分析            │ Data Export/ADX等          │ 分析テーブル設計、結合処理        │
│ 再送・重複排除         │ EdgeHub等に一部機能あり     │ messageId/sequenceNo設計        │
└────────────────────┴──────────────────────────┴────────────────────────────┘
```

---

# Event Adapterを作らない構成は可能？

可能ではあります。
ただし、その場合は **AI解析アプリ自身がEvent Adapterの役割を内包する** ことになります。

```text
案1: Event Adapterを独立させる
[AI解析] → [Event Adapter] → [IoT基盤]

案2: AI解析アプリに内包する
[AI解析 + Event Adapter機能] → [IoT基盤]
```

小規模なら案2でもOKです。

ただ、中規模以上では案1をおすすめします。

理由はこれです。

```text
AIモデルを変えてもIoT連携部を変えなくていい
Cumulocity/Azureの切り替えがしやすい
VMS連携を共通化できる
再送・重複排除を共通化できる
テストしやすい
```

Event Adapterを独立させると、AIとIoT基盤のあいだに「防波堤」ができます。波が高い日にありがたい部品です 🌊

---

# 実装パターン

## パターンA：EdgePC1上に置く

```text
[EdgePC1]
  ├─ AI推論
  ├─ Event Adapter
  └─ Local Queue
        │
        ▼
[EdgePC2]
```

向いているケース：

```text
VMS APIをEdgePC1から呼ぶ
AI解析結果との結合をすぐ行いたい
EdgePC2をなるべくシンプルにしたい
```

## パターンB：EdgePC2上に置く

```text
[EdgePC1]
  └─ AI推論
        │
        ▼
[EdgePC2]
  ├─ Event Adapter
  ├─ IoT Edge / Cumulocity Edge
  └─ Local Queue
```

向いているケース：

```text
IoT送信処理をEdgePC2に集約したい
現場内の複数AIサーバからイベントを受けたい
IoT Hub / Cumulocityへの認証情報をEdgePC2に閉じ込めたい
```

## パターンC：AIアプリに内包

```text
[EdgePC1: AI解析 + IoT送信]
   │
   ▼
[IoT基盤]
```

向いているケース：

```text
PoC
カメラ台数が少ない
短期検証
連携先が1つだけ
```

本番ではAかBが扱いやすいです。

---

# 私のおすすめ

今回のように、3製品構成を比較しながら設計するなら、**Event Adapterは自作の独立コンポーネント** として設計するのがおすすめです。

```text
[AI解析]
   │
   ▼
[共通Event Adapter]
   ├─ Cumulocity Connector
   ├─ Azure IoT Central Connector
   └─ Azure IoT Hub Connector
```

内部構造はこうです。

```text
Event Adapter
────────────────────────────
Core Logic
  - 入力正規化
  - ID変換
  - 時刻変換
  - correlationId生成
  - VMSメタデータ付与
  - バリデーション
  - Retry / Queue

Connector
  - Cumulocity REST/MQTT
  - IoT Central MQTT/HTTPS
  - IoT Hub Device SDK
```

こうしておくと、CumulocityからAzureへ移行する、または現場ごとに採用基盤が違う、という場合にも対応しやすいです。

---

# まとめ

**Event Adapterは、製品に含まれる標準機能ではなく、自前で作る連携部品** です。

ただし、それぞれの製品には実装を支える仕組みがあります。

```text
Cumulocity
  → REST / MQTT / SmartREST / Microservice SDKを使って実装

Azure IoT Central
  → Device Templateに合わせたTelemetry送信処理として実装

Azure IoT Hub + IoT Edge
  → IoT Edge Moduleとして実装するのが最も自然
```

設計書ではこう書くと誤解が少ないです。

```text
Event Adapterは、AI解析結果およびVMSメタデータをIoT基盤向けイベント形式へ変換・送信するカスタムアプリケーションである。
Cumulocity、Azure IoT Central、Azure IoT Hubの標準機能ではなく、各製品のAPI/SDK/メッセージング機能を利用して実装する。
```

この表現なら、製品標準機能と自作範囲の境界がかなりクリアになります。

[1]: https://cumulocity.com/docs/2024/device-integration/mqtt/?utm_source=chatgpt.com "Device integration using MQTT"
[2]: https://learn.microsoft.com/en-us/azure/iot-central/core/concepts-device-templates?utm_source=chatgpt.com "What are device templates in Azure IoT Central"
[3]: https://learn.microsoft.com/en-us/azure/iot-edge/iot-edge-runtime?utm_source=chatgpt.com "Azure IoT Edge runtime and architecture explained"
[4]: https://cumulocity.com/docs/device-integration/opcua/?utm_source=chatgpt.com "OPC UA - Cumulocity documentation"
[5]: https://learn.microsoft.com/en-us/azure/iot-edge/module-composition?utm_source=chatgpt.com "Deploy modules and establish routes in Azure IoT Edge"
[6]: https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-d2c-guidance?utm_source=chatgpt.com "Azure IoT Hub device-to-cloud options"
