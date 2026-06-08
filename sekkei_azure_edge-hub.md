**Azure IoT Hub + Azure IoT Edge** で組む場合は、IoT Centralよりも「部品を自分で組み上げる」構成になります。

ざっくり言うと、

```text
IoT Central構成
  → SaaS画面付きの完成品寄り

IoT Hub + IoT Edge構成
  → 拡張性重視のPaaS構成
  → 画面、ルール、分析、データ保存を自分で設計する
```

です。

今回のように、**IoTデバイスのメトリクス + VMS映像 + AI映像解析 + Data Lake分析** をしっかり作るなら、むしろ **IoT Hub + IoT Edgeの方が本格構成に向いています** 🛠️

---

# 結論

Azure IoT Hub + Azure IoT Edgeで構成するなら、こうなります。

```text
企業A 現場1
────────────────────────────────────────────

カメラ/センサー
   ├─→ VMS / NVR
   │     └─ 映像保存
   │
   ├─→ EdgePC1: AI映像解析
   │     └─ 人物検知 / 侵入検知 / 人数カウント
   │
   └─→ EdgePC2: Azure IoT Edge
         ├─ センサーデータ収集
         ├─ AIイベント受信
         ├─ Store & Forward
         └─ IoT Hub送信

Azure Cloud
────────────────────────────────────────────

Azure IoT Hub
   ├─ Device Identity
   ├─ Device Twin / Module Twin
   ├─ Message Routing
   ├─ Direct Method
   └─ Jobs

        ↓

Event Hubs / Service Bus / Blob / ADLS
        ↓
Stream Analytics / Functions
        ↓
ADLS Gen2 / Azure Data Explorer / Fabric
        ↓
Power BI / 独自Webダッシュボード
```

Azure IoT EdgeはIoT Hubを拡張し、データをローカルで分析してクラウド送信量を減らし、イベントへ素早く反応し、オフラインでも動作できる仕組みです。([Microsoft Learn][1])
また、IoT HubはDevice-to-Cloud通信として、D2Cメッセージ、Device Twinのreported properties、ファイルアップロードを使い分ける設計になります。([Microsoft Learn][2])

---

# Cumulocity / IoT Central との対応関係

```text
┌──────────────────────┬────────────────────────┬────────────────────────────┐
│ Cumulocity             │ Azure IoT Central       │ Azure IoT Hub + IoT Edge     │
├──────────────────────┼────────────────────────┼────────────────────────────┤
│ Cumulocity Edge        │ Azure IoT Edge           │ Azure IoT Edge               │
│ Cumulocity Core        │ Azure IoT Central        │ Azure IoT Hub + 自前アプリ群  │
│ Inventory              │ Device Template/Property │ Device Twin / DB             │
│ Measurement            │ Telemetry                │ D2C Message / ADX            │
│ Event                  │ Telemetry/Event風データ  │ D2C Message / Event Hub      │
│ Alarm                  │ Rules                    │ Functions / Stream Analytics │
│ DataHub                │ Data Export              │ Message Routing / Event Hubs │
│ Dashboard              │ IoT Central Dashboard    │ Power BI / Grafana / Web App │
│ Device Command         │ Command                  │ Direct Method / C2D Message  │
└──────────────────────┴────────────────────────┴────────────────────────────┘
```

IoT HubにはIoT Centralのような完成済みダッシュボードや業務画面はありません。
その代わり、**ルーティング・ストリーム処理・DB・BI・Webアプリを自由に組める** のが強みです。

---

# 推奨構成図

```text
企業A 現場1
=================================================================

                    ┌─────────────────────┐
                    │ Local NTP / PTP      │
                    │ Time Server          │
                    └─────────┬───────────┘
                              │ 時刻同期
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
 [Camera-01..N]          [VMS / NVR]          [Sensor / PLC]
        │                     │                     │
        │ 映像保存             │                     │ メトリクス
        ├────────────────────→│                     │
        │                     │                     │
        │ RTSP / ONVIF        │                     │ OPC-UA / Modbus /
        ▼                     │                     │ MQTT / REST
 ┌─────────────────────┐      │                     │
 │ EdgePC1              │      │                     │
 │ AI映像解析            │      │                     │
 │ - 人物検知            │      │                     │
 │ - 侵入検知            │      │                     │
 │ - 人数カウント        │      │                     │
 │ - GPU推論             │      │                     │
 │ - Local Queue         │      │                     │
 └─────────┬───────────┘      │                     │
           │                  │                     │
           │ AIイベントJSON    │ VMS clipId/API       │
           ▼                  │                     │
 ┌─────────────────────┐      │                     │
 │ Event Adapter Module │◀─────┘                     │
 │ - cameraId変換       │                            │
 │ - vmsClipId付与      │                            │
 │ - correlationId付与  │                            │
 │ - UTC時刻付与         │                            │
 └─────────┬───────────┘                            │
           │                                        │
           ▼                                        ▼
 ┌────────────────────────────────────────────────────────┐
 │ EdgePC2: Azure IoT Edge                                │
 │                                                        │
 │  ┌─────────────────┐  ┌─────────────────────────────┐ │
 │  │ edgeAgent        │  │ edgeHub                     │ │
 │  │ Module管理       │  │ ルーティング/Store&Forward   │ │
 │  └─────────────────┘  └─────────────────────────────┘ │
 │                                                        │
 │  ┌─────────────────┐  ┌─────────────────────────────┐ │
 │  │ Sensor Adapter   │  │ AI Event Receiver           │ │
 │  │ OPC-UA/Modbus    │  │ AIイベント受信               │ │
 │  └─────────────────┘  └─────────────────────────────┘ │
 │                                                        │
 │  ┌─────────────────┐  ┌─────────────────────────────┐ │
 │  │ Local DB/Queue   │  │ Local Dashboard 任意         │ │
 │  │ SQLite/Redis等   │  │ 現場閉域運用                 │ │
 │  └─────────────────┘  └─────────────────────────────┘ │
 └─────────┬──────────────────────────────────────────────┘
           │
           │ MQTT / AMQP / HTTPS
           ▼

Azure Cloud
=================================================================

 ┌─────────────────────┐
 │ Azure IoT Hub        │
 │ - Device Identity    │
 │ - Device Twin        │
 │ - Module Twin        │
 │ - Message Routing    │
 │ - Direct Method      │
 │ - Jobs               │
 └─────────┬───────────┘
           │
           ├──────────────→ [Blob / ADLS Gen2]
           │                  生ログ・長期保存
           │
           ├──────────────→ [Event Hubs]
           │                  ストリーム分析
           │
           ├──────────────→ [Service Bus]
           │                  業務連携・通知
           │
           ▼
 ┌─────────────────────┐
 │ Stream Processing    │
 │ - Azure Functions    │
 │ - Stream Analytics   │
 │ - Azure Data Explorer│
 └─────────┬───────────┘
           │
           ▼
 ┌─────────────────────┐
 │ Data Platform        │
 │ - ADLS Gen2          │
 │ - Azure Data Explorer│
 │ - Fabric Lakehouse   │
 │ - SQL Database       │
 └─────────┬───────────┘
           │
           ▼
 ┌─────────────────────┐
 │ Visualization / App  │
 │ - Power BI           │
 │ - Grafana            │
 │ - Web App            │
 │ - Teams/メール通知    │
 └─────────────────────┘
```

---

# データフロー

## 1. IoTメトリクス系データ

```text
企業A 現場1
────────────────────────────────────────────

[Sensor / PLC]
   │
   │ OPC-UA / Modbus / MQTT
   │
   ▼
[EdgePC2: Sensor Adapter Module]
   │
   │ メトリクス整形
   │ {
   │   deviceId,
   │   metricName,
   │   metricValue,
   │   metricTime,
   │   sequenceNo
   │ }
   │
   ▼
[edgeHub]
   │
   │ Store & Forward
   ▼
[Azure IoT Hub]
   │
   │ Message Routing
   ├────────────→ [ADLS Gen2: raw telemetry]
   ├────────────→ [Event Hubs: stream]
   └────────────→ [Functions/Stream Analytics]
                    │
                    ▼
              [ADX / Fabric / SQL]
```

IoT Hubのメッセージルーティングでは、デバイスから届いたテレメトリをBlob Storage、Service Bus、Event Hubsなどへ条件付きで送れます。各IoT HubにはEvent Hubs互換の組み込みエンドポイントがあり、カスタムエンドポイントも作成できます。([Microsoft Learn][3])

---

## 2. 映像・AIイベント系データ

```text
企業A 現場1
────────────────────────────────────────────

[Camera]
   ├────────────────────────────→ [VMS / NVR]
   │                               ├─ 常時録画
   │                               ├─ イベント録画
   │                               ├─ クリップ生成
   │                               └─ VMS API
   │
   └────────→ [EdgePC1: AI映像解析]
                  │
                  │ 人物検知 / 侵入検知 / 人数カウント
                  ▼
              [AI Event Adapter]
                  │
                  │ {
                  │   eventType,
                  │   cameraId,
                  │   eventTime,
                  │   confidence,
                  │   personCount,
                  │   vmsClipId,
                  │   correlationId
                  │ }
                  │
                  ▼
              [EdgePC2: Azure IoT Edge / edgeHub]
                  │
                  ▼
              [Azure IoT Hub]
                  │
                  ├────────→ [Event Hubs]
                  ├────────→ [ADLS Gen2]
                  └────────→ [Functions]
                                │
                                ├─ アラーム生成
                                ├─ VMSメタ情報補完
                                └─ 業務通知
```

映像本体はIoT Hubに送らない方がいいです。
IoT Hubには **AIイベントと映像メタデータ** だけを送ります。

---

## 3. VMS映像データフロー

```text
企業A 現場1
────────────────────────────────────────────

[Camera]
   │
   │ RTSP / ONVIF / VMS独自方式
   ▼
[VMS / NVR]
   │
   ├─ 生映像保存
   ├─ クリップ生成
   ├─ サムネイル生成
   ├─ 録画保持期間管理
   └─ VMS API

        │
        │ VMS metadata export
        ▼

Azure Cloud / Data Platform
────────────────────────────────────────────

[Functions / Data Factory / Custom ETL]
   │
   ▼
[ADLS Gen2 / ADX / Fabric]
   │
   ├─ vms_clips
   ├─ camera_master
   └─ recording_status
```

---

## 4. 分析時の結合フロー

```text
┌─────────────────────────────────────────────────────┐
│ ADLS Gen2 / ADX / Fabric                             │
│                                                     │
│  iot_metrics                                        │
│  ┌──────────┬────────────┬────────────────────┐     │
│  │ sensorId │ metricName │ metricTime         │     │
│  └──────────┴────────────┴────────────────────┘     │
│                                                     │
│  ai_events                                          │
│  ┌──────────┬────────────┬────────────────────┐     │
│  │ cameraId │ eventType  │ eventTime          │     │
│  └──────────┴────────────┴────────────────────┘     │
│                                                     │
│  vms_clips                                          │
│  ┌──────────┬────────────┬────────────────────┐     │
│  │ cameraId │ clipId     │ startTime/endTime  │     │
│  └──────────┴────────────┴────────────────────┘     │
│                                                     │
│  camera_master                                      │
│  ┌──────────┬────────────┬────────────────────┐     │
│  │ cameraId │ siteId     │ zoneId             │     │
│  └──────────┴────────────┴────────────────────┘     │
│                                                     │
│ 結合キー:                                            │
│ - correlationId                                      │
│ - vmsClipId                                          │
│ - cameraId + eventTime between start/end             │
│ - siteId + 時間範囲                                   │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
              [Power BI / ADX Dashboard / Web App]
                           │
                           ▼
        「設備異常前後の映像」「人物検知時のセンサー値」
        「カメラ別検知傾向」「誤検知分析」を確認
```

---

# NTP / PTPは必要？

これは **必要** です。
Azure IoT Hub + IoT Edge構成でも、CumulocityやIoT Central構成と同じく、いやむしろ本格分析ではさらに重要です。

理由は、IoT Edgeがオフラインで動き、再接続後にデータを送れるからです。IoT Edgeは一度IoT Hubへ接続した後、Edgeデバイスや下位デバイスが断続的またはインターネットなしでも動作できるように設計されています。([Microsoft Learn][4])
そのため、Azureに届いた時刻は「発生時刻」ではありません。

```text
現場発生時刻: eventTime / metricTime
Azure到着時刻: enqueuedTime / ingestedAt
```

映像との紐づけには必ず **現場発生時刻** を使います。

```text
映像検索         → eventTime / metricTime
遅延監視         → ingestedAt - eventTime
再送・重複排除   → messageId / sequenceNo / correlationId
```

おすすめはこれです。

```text
企業A 現場1
────────────────────────────

Local NTP / PTP Time Server
   ├─ Camera
   ├─ VMS / NVR
   ├─ EdgePC1: AI解析
   ├─ EdgePC2: Azure IoT Edge
   ├─ Sensor Gateway
   └─ PLC / IPC
```

高速な製造ラインやミリ秒単位の解析が必要ならPTP、人物検知や設備異常前後の映像確認ならNTPで足りることが多いです。

---

# IoT Hub + IoT Edgeでの機能配置

```text
┌────────────────────┬─────────────────────────────┬────────────────────────────┐
│ 機能                 │ 配置先                        │ 補足                         │
├────────────────────┼─────────────────────────────┼────────────────────────────┤
│ 映像保存             │ VMS / NVR                     │ 生映像はAzure IoTへ送らない    │
│ AI映像解析           │ EdgePC1                       │ GPU搭載、Docker化推奨          │
│ AIイベント整形        │ EdgePC1 or EdgePC2 Module      │ IoT Hub向けJSONへ変換          │
│ センサー収集          │ EdgePC2 IoT Edge Module        │ OPC-UA/Modbus/MQTT変換         │
│ エッジルーティング     │ IoT Edge edgeHub               │ Store & Forward                │
│ モジュール管理         │ IoT Edge edgeAgent             │ Edge module lifecycle          │
│ デバイス台帳          │ IoT Hub Device Twin + DB       │ IoT Centralより自作要素が多い   │
│ 状態・設定管理         │ Device Twin / Module Twin      │ 設定同期・状態報告              │
│ 一括操作             │ IoT Hub Jobs                   │ Twin更新やDirect Method実行     │
│ コマンド実行           │ Direct Method / C2D Message    │ 再起動、設定反映、診断など       │
│ メッセージ振り分け     │ IoT Hub Message Routing        │ ADLS/Event Hubs/Service Busへ   │
│ ルール・アラーム       │ Functions / Stream Analytics   │ IoT Central Rules相当を自作     │
│ 長期保存             │ ADLS Gen2                      │ Raw/Bronze保存                  │
│ 時系列分析            │ Azure Data Explorer            │ 高速検索・時系列分析             │
│ BI                  │ Power BI / Grafana / Web App   │ 画面は別途設計                  │
└────────────────────┴─────────────────────────────┴────────────────────────────┘
```

Device Twinは、IoT Hubとデバイス間で状態や設定情報を同期するJSONドキュメントで、タグ、desired properties、reported propertiesなどを使ってメタデータ・設定・状態を扱います。([Microsoft Learn][5])
また、IoT Hub Jobsを使うと、複数デバイスに対してプロパティ更新やDirect Method呼び出しをスケジュールできます。([Microsoft Learn][6])

---

# データ種別ごとの保存先

```text
┌────────────────────┬────────────────────────────┬────────────────────────────┐
│ データ種別           │ 保存先                       │ 理由                         │
├────────────────────┼────────────────────────────┼────────────────────────────┤
│ 生映像               │ VMS / NVR                    │ 大容量・録画検索向き           │
│ 映像クリップ          │ VMS / 映像ストレージ          │ 証跡確認・再生向き              │
│ VMSメタデータ         │ ADLS Gen2 / ADX / SQL         │ IoTデータと結合分析             │
│ AI検知イベント        │ IoT Hub → Event Hubs / ADLS   │ ストリーム処理・長期保存         │
│ センサーメトリクス     │ IoT Hub → ADLS / ADX          │ 時系列分析                      │
│ アラーム相当          │ Functions / Stream Analytics → DB │ 状態管理・通知              │
│ デバイス状態          │ Device Twin / Module Twin     │ 最新状態・設定管理              │
│ デバイス台帳          │ Device Twin Tags + SQL/Cosmos DB │ 検索・業務属性管理           │
│ Edgeログ             │ Log Analytics / ADLS          │ 障害解析                        │
│ 時刻同期状態          │ ADX / Log Analytics           │ 映像紐づけ品質の監視             │
│ 分析用テーブル         │ ADX / Fabric / Lakehouse      │ BI・ML                          │
└────────────────────┴────────────────────────────┴────────────────────────────┘
```

---

# 推奨データモデル

## `iot_metrics`

```text
deviceId
sensorId
siteId
metricName
metricValue
metricTime
ingestedAt
sequenceNo
messageId
correlationId
```

## `ai_events`

```text
eventId
eventType
siteId
cameraId
vmsCameraId
eventTime
ingestedAt
personCount
confidence
zoneId
vmsClipId
correlationId
aiModelVersion
messageId
sequenceNo
```

## `vms_clips`

```text
vmsClipId
siteId
cameraId
vmsCameraId
clipStartTime
clipEndTime
playbackUrl
thumbnailUrl
retentionUntil
```

## `device_master`

```text
deviceId
siteId
deviceType
area
zoneId
iotHubDeviceId
edgeDeviceId
status
```

## `time_sync_status`

```text
deviceId
deviceType
siteId
offsetMs
lastSyncTime
ntpServer
syncStatus
reportedAt
```

---

# IoT Hubに送るJSON例

## AIイベント

```json
{
  "messageType": "aiEvent",
  "eventType": "PersonDetected",
  "siteId": "site-001",
  "cameraId": "CAM-001",
  "vmsCameraId": "VMS-CAM-001",
  "eventTime": "2026-06-08T01:00:05.123Z",
  "localTime": "2026-06-08T10:00:05.123+09:00",
  "personCount": 2,
  "confidence": 0.94,
  "zoneId": "north_gate",
  "vmsClipId": "clip-20260608-100005-CAM001",
  "correlationId": "site001-CAM001-20260608T010005123Z",
  "messageId": "msg-000001",
  "sequenceNo": 123456,
  "aiModelVersion": "person-detector-v1.3.2"
}
```

## センサーメトリクス

```json
{
  "messageType": "metric",
  "siteId": "site-001",
  "deviceId": "SENSOR-001",
  "metricName": "temperature",
  "metricValue": 35.2,
  "metricTime": "2026-06-08T01:00:04.900Z",
  "localTime": "2026-06-08T10:00:04.900+09:00",
  "sequenceNo": 98765,
  "messageId": "sensor-msg-000123"
}
```

IoT HubからStorageへJSONとして書き出す場合、base64化を避けるにはメッセージの `contentType` を `application/json`、`contentEncoding` を `UTF-8` に設定する必要があります。([Microsoft Learn][7])

---

# Message Routing設計例

```text
Azure IoT Hub
────────────────────────────────────────────

Route 1: ai-events
条件:
  messageType = 'aiEvent'
宛先:
  Event Hubs
用途:
  リアルタイム検知、通知、アラーム

Route 2: metrics
条件:
  messageType = 'metric'
宛先:
  ADLS Gen2 / Blob
用途:
  長期保存、バッチ分析

Route 3: device-status
条件:
  messageType = 'deviceStatus'
宛先:
  Service Bus
用途:
  保守システム連携

Route 4: all-raw
条件:
  true
宛先:
  ADLS Gen2
用途:
  生ログ保管、再処理
```

IoT HubはDevice-to-Cloudメッセージのほか、デバイスライフサイクルイベント、Device Twin変更イベント、接続状態イベントなどもルーティング対象にできます。([Microsoft Learn][8])

---

# アラーム・通知の作り方

IoT CentralではRulesがありましたが、IoT Hubには業務ルール画面はありません。
なので、以下で作ります。

```text
IoT Hub
   ↓ Message Routing
Event Hubs
   ↓
Azure Functions / Stream Analytics
   ↓
Alarm DB / Service Bus / Teams / Mail / Web App
```

例：

```text
人物検知 confidence >= 0.8
かつ zoneId = restricted_area
かつ 時間帯 = 夜間
   ↓
Critical Alarm生成
   ↓
Service Busへ送信
   ↓
Teams通知 / 保守システム連携 / Web画面表示
```

---

# 現場側EdgePCの構成案

## EdgePC1: AI映像解析

```text
EdgePC1
├─ Ubuntu / Windows
├─ NVIDIA GPU
├─ Docker
├─ AI推論コンテナ
│  ├─ YOLO / DeepStream / OpenVINO
│  ├─ RTSP入力
│  └─ Person Detection
├─ VMS Client / VMS API Client
├─ Local Queue
│  ├─ Redis / SQLite / MQTT Broker
│  └─ 再送制御
└─ AI Event Adapter
```

## EdgePC2: Azure IoT Edge

```text
EdgePC2
├─ Azure IoT Edge Runtime
├─ edgeAgent
├─ edgeHub
├─ Sensor Adapter Module
├─ AI Event Receiver Module
├─ Time Sync Monitor Module
├─ Local DB / Queue
└─ Optional Local Dashboard
```

EdgePC1とEdgePC2を分ける考え方は引き続きおすすめです。
AI推論のGPU負荷がIoT Edgeの通信・バッファリングに影響しにくくなります。

小規模なら1台にまとめることも可能です。

```text
小規模:
  EdgePC1 = AI解析 + IoT Edge

中規模以上:
  EdgePC1 = AI解析専用
  EdgePC2 = IoT Edge専用
```

---

# 最終おすすめ構成

```text
企業A 現場1
=================================================================

[Camera/Sensor]
   ├──────────────→ [VMS / NVR]
   │                  ├─ 映像保存
   │                  ├─ Clip ID生成
   │                  └─ VMS API
   │
   ├──────────────→ [EdgePC1: AI映像解析]
   │                    ├─ 人物検知
   │                    ├─ 侵入検知
   │                    ├─ 人数カウント
   │                    ├─ vmsClipId取得
   │                    └─ AIイベント生成
   │
   └──────────────→ [EdgePC2: Azure IoT Edge]
                        ├─ Sensor Adapter
                        ├─ AI Event Receiver
                        ├─ edgeHub Store & Forward
                        ├─ Time Sync Monitor
                        └─ Local Dashboard任意

Azure Cloud
=================================================================

[Azure IoT Hub]
   ├─ Device Identity
   ├─ Device Twin / Module Twin
   ├─ Message Routing
   ├─ Direct Method
   └─ Jobs

        ├────────→ [ADLS Gen2: raw / bronze]
        ├────────→ [Event Hubs: streaming]
        ├────────→ [Service Bus: business events]
        └────────→ [Event Grid: device lifecycle events]

[Processing]
   ├─ Azure Functions
   ├─ Stream Analytics
   └─ Data Factory

[Data Platform]
   ├─ ADLS Gen2
   ├─ Azure Data Explorer
   ├─ Fabric Lakehouse
   └─ SQL / Cosmos DB

[Visualization]
   ├─ Power BI
   ├─ Grafana
   └─ Custom Web App
```

---

# この構成での設計ルール

```text
1. 映像本体はIoT Hubへ送らない。
2. 映像本体はVMS/NVRで保存する。
3. IoT HubへはAIイベント、センサーメトリクス、デバイス状態を送る。
4. Message Routingでデータ種別ごとにEvent Hubs / ADLS / Service Busへ振り分ける。
5. Device Twinはデバイス状態・設定・台帳属性に使う。
6. Module TwinはEdgeモジュールの設定管理に使う。
7. Direct Methodは再起動・診断・即時操作に使う。
8. Jobsは複数デバイスへの一括設定変更に使う。
9. VMS連携にはvmsClipId, playbackUrl, thumbnailUrlを持たせる。
10. 分析結合はcorrelationId / vmsClipId / cameraId + time rangeで行う。
11. 現場機器はLocal NTP/PTPで同期する。
12. Azure到着時刻ではなく、現場発生時刻を分析キーにする。
13. sequenceNo, messageIdで再送時の重複排除を行う。
14. IoT Central相当の画面・ルールはPower BI/Web App/Functionsで実装する。
```

---

# Cumulocity / IoT Central / IoT Hub + IoT Edge の選び分け

```text
┌────────────────────┬──────────────────────────────┐
│ 構成                 │ 向いているケース                 │
├────────────────────┼──────────────────────────────┤
│ Cumulocity           │ 現場Edgeと中央管理を一体で使いたい │
│ Azure IoT Central    │ 早く画面付きIoTアプリを作りたい     │
│ IoT Hub + IoT Edge   │ 大規模・高度分析・自由設計したい    │
└────────────────────┴──────────────────────────────┘
```

今回のように、**VMS映像メタデータ、AI検知イベント、IoTメトリクス、Data Lake分析、独自ダッシュボード** まで考えるなら、Azure案では **IoT Hub + IoT Edge が一番きれいに拡張できます**。

その代わり、IoT Centralより実装量は増えます。
でも、構成の自由度は高くて、後から「拠点追加」「AIモデル追加」「分析基盤追加」「既存業務システム連携」をやりやすいです。

[1]: https://learn.microsoft.com/en-us/azure/iot-edge/?utm_source=chatgpt.com "Azure IoT Edge documentation"
[2]: https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-d2c-guidance?utm_source=chatgpt.com "Azure IoT Hub device-to-cloud options"
[3]: https://learn.microsoft.com/en-us/azure/iot-hub/tutorial-routing?utm_source=chatgpt.com "Tutorial - Configure message routing - Azure IoT Hub"
[4]: https://learn.microsoft.com/en-us/azure/iot-edge/offline-capabilities?utm_source=chatgpt.com "Operate Azure IoT Edge devices offline"
[5]: https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-device-twins?utm_source=chatgpt.com "Understand and use device twins in IoT Hub - Azure"
[6]: https://learn.microsoft.com/ja-jp/azure/iot-hub/iot-hub-devguide-jobs?utm_source=chatgpt.com "複数のデバイスで Azure IoT Hub ジョブをスケジュールする"
[7]: https://learn.microsoft.com/ja-jp/azure/iot-hub/iot-hub-devguide-endpoints?utm_source=chatgpt.com "Azure IoT Hub のエンドポイントについて"
[8]: https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-messages-d2c?utm_source=chatgpt.com "Understand Azure IoT Hub Message routing"
