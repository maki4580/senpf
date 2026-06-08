**Azure IoT Central構成でもNTPは必須級**です。
むしろ、**IoT CentralはクラウドSaaSなので、現場側の時刻同期の重要度はCumulocity Edge構成より上がる**と考えた方がいいです。

理由はシンプルで、映像とIoTメトリクスを紐づけるときに使う基準は結局これだからです。

```text
cameraId + eventTime
sensorId + metricTime
vmsClipStartTime / vmsClipEndTime
```

クラウドに届いた時刻ではなく、**現場で実際に発生した時刻** が命綱になります 🕰️

---

# 結論

Azure IoT Central構成でも、以下は必須です。

```text
Camera
VMS / NVR
EdgePC1: 映像解析
EdgePC2: Azure IoT Edge
センサー / PLC
Azure側の受信基盤
```

このうち、**Azure側はMicrosoft管理のクラウド時刻**で動きますが、現場側は企業A側でNTP/PTP同期を設計する必要があります。

```text
現場側の時刻同期  → 企業Aが設計・運用する
Azure側の時刻     → Microsoft管理
```

Azure IoT Edgeはオフライン時にもメッセージを保存し、再接続後に転送できますが、その場合クラウド到着時刻は実発生時刻から遅れます。Microsoft Learnでも、IoT Edge hubがオフライン時のdevice-to-cloudメッセージを保存し、再接続時に転送すること、TTLやディスク容量設定が重要であることが説明されています。([Microsoft Learn][1])

つまり、分析で使うべき時刻は、

```text
Azure到着時刻ではなく、デバイス/エッジが付与した発生時刻
```

です。

---

# なぜAzure IoT CentralでもNTPが必要か

## 1. IoT Centralの受信時刻だけでは映像と合わない

例えば、人物検知が現場で10:00:05に起きたとします。

```text
10:00:05  現場で人物検知
10:00:06  EdgePC2へ送信
10:05:30  WAN復旧
10:05:31  Azure IoT Centralに到着
```

このとき、クラウド側の到着時刻だけを見ると、

```text
10:05:31に人物検知が起きた
```

ように見えてしまいます。

でも、VMSで見るべき映像は、

```text
10:00:05前後の映像
```

です。

だから、AIイベントには必ず現場時刻を入れます。

```json
{
  "eventType": "PersonDetected",
  "eventTime": "2026-06-08T10:00:05.123+09:00",
  "cameraId": "CAM-001",
  "vmsClipId": "clip-001",
  "correlationId": "site001-CAM001-20260608T100005123"
}
```

---

## 2. Azure IoT EdgeのStore & Forwardでは、発生時刻と到着時刻がズレる

Azure IoT Edgeは現場断・WAN断に強いです。
Edge Hubがメッセージを保持して、再接続後にクラウドへ送れます。これは強い味方です。([Microsoft Learn][1])

ただし、これにより必ず以下の2つの時刻が生まれます。

```text
発生時刻: eventTime / observedAt
到着時刻: enqueuedTime / cloudReceivedAt
```

分析・映像紐づけでは **発生時刻** を使います。
運用監視・通信遅延分析では **到着時刻** を使います。

```text
映像検索         → eventTime
遅延監視         → cloudReceivedAt - eventTime
データ欠損確認   → sequenceNo
再送/重複排除    → messageId / correlationId
```

---

# Azure IoT Central構成での推奨アーキテクチャ

```text
企業A 現場1
=================================================================

                     ┌─────────────────────┐
                     │ Local NTP / PTP      │
                     │ Time Server          │
                     └─────────┬───────────┘
                               │ 時刻同期
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
 [Camera/Sensor]          [VMS / NVR]          [EdgePC1: AI解析]
      │                        │                      │
      │ 映像                   │ 映像保存              │ AIイベント生成
      │                        │                      │
      ├──────────────→ [VMS録画ストレージ]             │
      │                                                │
      │ RTSP/ONVIF                                     │
      ▼                                                │
 [EdgePC1: AI映像解析]                                  │
      │                                                │
      │ eventTime, cameraId, vmsClipId                 │
      ▼                                                │
 [Event Adapter]                                       │
      │                                                │
      │ MQTT / HTTPS                                   │
      ▼                                                │
 [EdgePC2: Azure IoT Edge]
      │
      │ Edge Hub Store & Forward
      ▼
 Azure IoT Central / IoT Hub
      │
      │ Data Export
      ▼
 Event Hubs / ADLS / ADX / Fabric
      │
      ▼
 映像・メトリクス紐づけ分析
```

---

# データフロー図

## 1. IoTメトリクス系データ

```text
企業A 現場1
────────────────────────────────────────────

[Sensor / PLC]
   │
   │ 例:
   │ - temperature = 35.2
   │ - vibration = 0.31
   │ - metricTime = 2026-06-08T10:00:04.900+09:00
   │ - sequenceNo = 123456
   ▼
[EdgePC2: Azure IoT Edge]
   │
   │ Store & Forward
   │ messageId / correlationId付与
   ▼
[Azure IoT Central]
   │
   │ Telemetry / Rules / Dashboard
   ▼
[Data Export]
   ▼
[Event Hubs / ADLS / Azure Data Explorer]
```

---

## 2. 映像・AIイベント系データ

```text
企業A 現場1
────────────────────────────────────────────

[Camera]
   ├────────────────────────────→ [VMS / NVR]
   │                               ├─ 常時録画
   │                               ├─ クリップ生成
   │                               └─ clipId / startTime / endTime
   │
   └────────→ [EdgePC1: AI映像解析]
                  │
                  │ 人物検知 / 侵入検知 / 人数カウント
                  │ eventTimeは現場時刻で付与
                  ▼
              [Event Adapter]
                  │
                  │ vmsClipId / correlationId付与
                  ▼
              [EdgePC2: Azure IoT Edge]
                  │
                  ▼
              [Azure IoT Central]
                  │
                  ▼
              [Data Export]
                  │
                  ▼
              [ADLS / ADX / Fabric]
```

---

## 3. VMS映像メタデータ連携

```text
[VMS / NVR]
   │
   │ VMS API / ETL / Batch / Event Hook
   ▼
[ADLS / ADX / Fabric]
   │
   ├─ vms_clips
   │   - clipId
   │   - cameraId
   │   - startTime
   │   - endTime
   │   - playbackUrl
   │   - thumbnailUrl
   │
   └─ vms_recording_status
       - cameraId
       - recordingStatus
       - storageStatus
       - retentionUntil
```

---

# 分析時の結合キー

Azure構成でも、結合キーはCumulocity案とほぼ同じです。

```text
最優先:
  correlationId
  vmsClipId

次点:
  cameraId + eventTime between clipStartTime and clipEndTime

補助:
  siteId
  zoneId
  aiModelVersion
  sequenceNo
```

分析テーブルはこうします。

```text
iot_metrics
────────────────────────────
deviceId
sensorId
siteId
metricName
metricValue
metricTime
ingestedAt
sequenceNo
correlationId

ai_events
────────────────────────────
eventId
eventType
siteId
cameraId
eventTime
ingestedAt
personCount
confidence
zoneId
vmsClipId
correlationId

vms_clips
────────────────────────────
vmsClipId
siteId
cameraId
clipStartTime
clipEndTime
playbackUrl
thumbnailUrl
retentionUntil

camera_master
────────────────────────────
cameraId
vmsCameraId
iotCentralDeviceId
siteId
area
zoneId
```

結合はこうです。

```sql
SELECT
  e.eventId,
  e.eventTime,
  e.eventType,
  e.personCount,
  m.metricName,
  m.metricValue,
  c.playbackUrl
FROM ai_events e
LEFT JOIN iot_metrics m
  ON e.siteId = m.siteId
 AND m.metricTime BETWEEN e.eventTime - INTERVAL '30 seconds'
                      AND e.eventTime + INTERVAL '30 seconds'
LEFT JOIN vms_clips c
  ON e.cameraId = c.cameraId
 AND e.eventTime BETWEEN c.clipStartTime AND c.clipEndTime;
```

ただし、できれば時間範囲JOINに頼りきらず、AIイベント生成時に `vmsClipId` か `correlationId` を付けるのが安全です。

---

# 時刻設計のルール

ここは設計書に強めに書いていいです。

```text
1. 現場内にLocal NTP Serverを設置する。
2. Camera、VMS、EdgePC1、EdgePC2、Sensor Gatewayを同じNTP Serverに同期する。
3. 高精度が必要な場合はPTPを検討する。
4. IoT Central到着時刻ではなく、現場発生時刻を分析キーに使う。
5. eventTime / metricTime / clipStartTime / clipEndTime はUTCまたはタイムゾーン付きISO 8601で保存する。
6. ingestedAt / enqueuedTime は遅延監視用として別項目に保存する。
7. sequenceNo と messageId を持たせて、再送時の重複を排除する。
8. EdgePC1/EdgePC2の時計ズレを監視メトリクス化する。
```

おすすめは、保存時刻を **UTC** に統一することです。

```json
{
  "eventTime": "2026-06-08T01:00:05.123Z",
  "localTime": "2026-06-08T10:00:05.123+09:00",
  "timezone": "Asia/Tokyo"
}
```

画面表示だけJSTに変換します。

---

# Cumulocity構成との違い

```text
Cumulocity Edge構成
────────────────────────────────
現場にCumulocity Edgeがある
現場内でイベント/アラーム管理を完結しやすい
Core同期前に現場側で管理しやすい
それでもNTPは必須

Azure IoT Central構成
────────────────────────────────
IoT CentralはクラウドSaaS
現場はAzure IoT EdgeがStore & Forwardする
クラウド到着時刻と現場発生時刻がズレやすい
そのためNTPは必須級
現場完結画面が必要なら別途Local Dashboardが必要
```

つまり、Azure構成ではこう考えます。

```text
Cumulocity Edgeの代わりに
Azure IoT Edge + Local Queue + Local DB + Local Dashboard
を現場側に置く
```

---

# 推奨するAzure構成

```text
企業A 現場1
=================================================================

                     ┌─────────────────────┐
                     │ Local NTP / PTP      │
                     │ Time Server          │
                     └─────────┬───────────┘
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                 │
             ▼                 ▼                 ▼
       [Camera/Sensor]     [VMS / NVR]     [EdgePC1: AI解析]
             │                 │                 │
             │                 │                 ├─ 人物検知
             │                 │                 ├─ 侵入検知
             │                 │                 ├─ eventTime付与
             │                 │                 ├─ vmsClipId付与
             │                 │                 └─ Local Queue
             │                 │
             │                 ├─ 映像保存
             │                 ├─ clipId生成
             │                 ├─ start/end管理
             │                 └─ VMS API
             │
             └──────────────┐
                            ▼
                  [EdgePC2: Azure IoT Edge]
                    ├─ Sensor Adapter
                    ├─ Event Adapter
                    ├─ Edge Hub
                    ├─ Store & Forward
                    ├─ Local DB
                    └─ Local Dashboard任意
                            │
                            ▼
                    [Azure IoT Central]
                    ├─ Device Template
                    ├─ Telemetry
                    ├─ Rules
                    ├─ Dashboard
                    └─ Data Export
                            │
                            ▼
                    [Event Hubs]
                            │
                            ▼
                    [ADLS Gen2 / ADX / Fabric]
                    ├─ iot_metrics
                    ├─ ai_events
                    ├─ vms_clips
                    ├─ camera_master
                    └─ time_sync_status
                            │
                            ▼
                    [Power BI / 分析 / ML]
```

---

# 追加した方がいいテーブル

Azure構成では、時刻ズレを監視するテーブルを作るのがおすすめです。

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

例えば、

```text
Camera-01 offsetMs = +120
EdgePC1   offsetMs = +8
VMS       offsetMs = -15
SensorGW  offsetMs = +2500
```

みたいに見えるようにします。

しきい値例：

```text
±500ms以内  正常
±1秒超      警告
±5秒超      異常
```

映像紐づけが目的なら、最低でも秒単位、できればサブ秒単位で同期しておきたいです。人物検知・侵入検知の映像確認ならNTPで十分なことが多いですが、製造ラインの高速イベントやPLCとの厳密な同期が必要ならPTPを検討します。

---

# Azure IoT Centralでのデータ項目例

## AIイベント

```json
{
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

## VMSクリップメタデータ

```json
{
  "vmsClipId": "clip-20260608-100005-CAM001",
  "siteId": "site-001",
  "cameraId": "CAM-001",
  "vmsCameraId": "VMS-CAM-001",
  "clipStartTime": "2026-06-08T01:00:00.000Z",
  "clipEndTime": "2026-06-08T01:00:30.000Z",
  "playbackUrl": "https://vms.example.local/play/clip-20260608-100005-CAM001",
  "thumbnailUrl": "https://vms.example.local/thumb/clip-20260608-100005-CAM001",
  "retentionUntil": "2026-07-08T00:00:00.000Z"
}
```

---

# 機能・ストレージ・ツール構成

```text
┌────────────────────┬────────────────────────────┬────────────────────────────┐
│ 領域                 │ 推奨ツール                   │ 役割                         │
├────────────────────┼────────────────────────────┼────────────────────────────┤
│ 現場時刻同期          │ Local NTP / PTP Grandmaster  │ 映像・IoTデータの時刻基準       │
│ 映像保存              │ VMS / NVR                    │ 映像本体・クリップ管理           │
│ AI解析                │ EdgePC1 + GPU + Docker       │ 人物検知・侵入検知・人数カウント │
│ IoTエッジ中継          │ Azure IoT Edge               │ Store & Forward、モジュール管理 │
│ クラウドIoT管理        │ Azure IoT Central             │ デバイス管理・可視化・ルール     │
│ ストリーム連携          │ Event Hubs                   │ Data Exportの受け口              │
│ 長期保存              │ ADLS Gen2                    │ イベント・メトリクス・VMSメタ保存│
│ 時系列/探索分析         │ Azure Data Explorer          │ 高速なログ・時系列分析            │
│ BI                   │ Power BI / Fabric             │ 可視化・レポート                 │
│ ETL                  │ Functions / Data Factory      │ VMSメタデータ連携・整形           │
│ 監視                  │ Azure Monitor / Log Analytics │ 遅延・時刻ズレ・Edge状態監視      │
└────────────────────┴────────────────────────────┴────────────────────────────┘
```

---

# 最終判断

Azure IoT Central構成でも、NTP/PTPは必要です。
特に以下の要件があるなら、**必須要件として設計書に入れるべき**です。

```text
IoTメトリクスと映像を同じタイムラインで見たい
人物検知イベントから該当映像を開きたい
設備異常の前後映像を確認したい
Data Lake上でメトリクス・AIイベント・VMSメタを結合分析したい
WAN断後の再送データも正しい時刻で分析したい
```

Azure IoT Centralはクラウド側の受信・可視化・ルール処理は便利ですが、**現場でいつ起きたか** は現場機器の時計に依存します。
なので設計の芯はこれです。

```text
Azureの到着時刻を信じるのではなく、
現場で同期された時刻を信じる。
```

この一本の背骨があると、VMS映像とIoTメトリクスの分析がぐっと安定します。

[1]: https://learn.microsoft.com/en-us/azure/iot-edge/offline-capabilities?utm_source=chatgpt.com "Operate Azure IoT Edge devices offline"
