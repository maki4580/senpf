**Cumulocity Edge/Core と Azure IoT Central は対応関係が1対1ではありません**。

Cumulocityでは、

```text
現場：Cumulocity Edge
本社：Cumulocity Core / Central Cumulocity
```

のように、現場側にもCumulocityそのものを置けます。

一方で Azure IoT Central は基本的に **クラウド上のSaaS型IoTアプリ** です。Microsoftの説明でも、IoT Centralは「ready-made UX and API surface」で、裏側のPaaS群を事前に組み合わせたIoT向けアプリ基盤として説明されています。([Microsoft Azure][1])
そのため、現場側には **Azure IoT Edge** または **Azure IoT Operations** を置いて、クラウド側に **Azure IoT Central** を置く構成になります。

---

# まず製品の置き換えイメージ

```text
Cumulocity構成                         Azure構成
────────────────────────────────────────────────────────
Cumulocity Edge       ≒               Azure IoT Edge
                                       または Azure IoT Operations

Cumulocity Core       ≒               Azure IoT Central
                                       ただしSaaS型

Cumulocity DataHub    ≒               IoT Central Data Export
                                       + Event Hubs
                                       + Data Lake Storage
                                       + Azure Data Explorer等

Cumulocity Event      ≒               IoT Central Telemetry/Event
                                       または IoT Hub Message

Cumulocity Alarm      ≒               IoT Central Rules
                                       + Action Groups
                                       + Logic Apps / Functions

VMS                   =               VMS / NVR
                                       そのまま別管理
```

注意点として、Azure IoT Edgeは「Cumulocity Edgeの完全な代替」ではありません。
Azure IoT Edgeは、エッジデバイス上でコンテナ化アプリをデプロイ・実行・監視するランタイムです。AI解析、プロトコル変換、ローカルバッファ、現場内ルーティングに向いています。([Microsoft Learn][2])

---

# 推奨構成

今回の要件をAzure IoT Centralで組むなら、こうです。

```text
企業A 現場1
=================================================================

 [Camera-01..N] ───────────────┬──────────────→ [VMS / NVR]
      │                        │                 │
      │ RTSP / ONVIF           │                 ├─ 映像録画
      │                        │                 ├─ クリップ生成
      │                        │                 ├─ サムネイル生成
      │                        │                 └─ VMS API
      │                        │
      ▼                        │
 ┌─────────────────────┐       │
 │ EdgePC1              │       │
 │ AI映像解析            │       │
 │ - 人物検知            │       │
 │ - 侵入検知            │       │
 │ - 人数カウント        │       │
 │ - GPU推論             │       │
 │ - Local Queue         │       │
 └─────────┬───────────┘       │
           │                   │
           │ AIイベントJSON     │ VMS clipId / playbackUrl
           ▼                   │
 ┌─────────────────────┐       │
 │ Event Adapter        │◀──────┘
 │ - Azure形式変換       │
 │ - cameraId変換        │
 │ - correlationId付与   │
 │ - vmsClipId付与       │
 └─────────┬───────────┘
           │
           │ MQTT / AMQP / HTTPS
           ▼
 ┌─────────────────────┐
 │ EdgePC2              │
 │ Azure IoT Edge        │
 │ - IoT Edge Runtime    │
 │ - Edge Hub            │
 │ - Edge Agent          │
 │ - Sensor Adapter      │
 │ - Store & Forward     │
 │ - Local Dashboard任意  │
 └─────────┬───────────┘
           │
           │ IoT Hub経由
           ▼

Azure / 企業Aクラウド側
=================================================================

 ┌─────────────────────┐
 │ Azure IoT Central    │
 │ - Device Template    │
 │ - Device Management  │
 │ - Telemetry View     │
 │ - Rules              │
 │ - Dashboard          │
 │ - Jobs / Commands    │
 └─────────┬───────────┘
           │
           │ Data Export
           ▼
 ┌─────────────────────┐
 │ Azure Event Hubs      │
 │ - Warm Path           │
 │ - ストリーム連携       │
 └─────────┬───────────┘
           │
           ├──────────────→ [Azure Functions / Logic Apps]
           │                   ├─ 通知
           │                   ├─ VMS API連携
           │                   └─ データ整形
           │
           ▼
 ┌─────────────────────┐
 │ Azure Data Lake      │
 │ Storage Gen2         │
 │ - IoTイベント履歴      │
 │ - センサー履歴        │
 │ - VMSメタデータ       │
 │ - AIモデルログ        │
 └─────────┬───────────┘
           │
           ▼
 ┌─────────────────────┐
 │ 分析 / BI             │
 │ - Azure Data Explorer │
 │ - Microsoft Fabric    │
 │ - Synapse             │
 │ - Power BI            │
 └─────────────────────┘
```

---

# Azure構成でのデータフロー

## 1. AIイベント・センサーデータの流れ

```text
企業A 現場1
────────────────────────────────────────────

[Camera]
   │
   │ RTSP / ONVIF
   ▼
[EdgePC1: AI映像解析]
   │
   │ 生成データ:
   │ - personDetected = true
   │ - personCount = 2
   │ - confidence = 0.94
   │ - cameraId = CAM-001
   │ - timestamp = 2026-06-08T10:15:23+09:00
   │ - vmsClipId = clip-xxxx
   │ - correlationId = site001-CAM001-xxxx
   │
   ▼
[Event Adapter]
   │
   │ MQTT / HTTPS
   ▼
[EdgePC2: Azure IoT Edge]
   │
   │ Edge Hubで一時保持・再送
   ▼
[Azure IoT Hub]
   │
   ▼
[Azure IoT Central]
   │
   │ Telemetry / Event / Rules
   ▼
[Data Export]
   │
   ▼
[Event Hubs / Blob / ADLS / Azure Data Explorer]
   │
   ▼
[分析 / BI / ML]
```

IoT CentralのData Exportは、IoT Centralアプリからフィルタ・拡張したIoTデータを継続的にEvent Hubsへ送る機能として説明されています。([Microsoft Learn][3])
また、IoT CentralのData ExportはBlob Storage、Event Hubs、Service Bus、Azure Data Explorerなどの宛先へストリームでき、VNetやプライベートエンドポイントで宛先を保護する構成も取れます。([Microsoft Learn][4])

---

## 2. 映像データの流れ

```text
企業A 現場1
────────────────────────────────────────────

[Camera]
   │
   │ RTSP / ONVIF / VMS独自方式
   ▼
[VMS / NVR]
   │
   ├─ 常時録画
   ├─ イベント録画
   ├─ クリップ生成
   ├─ サムネイル生成
   └─ 映像検索API
```

ここはCumulocity案と同じで、**映像本体はIoT Centralへ入れない** 方がいいです。
IoT Centralには、映像に紐づくイベントメタデータだけを送ります。

---

## 3. 映像とイベントを紐づける分析フロー

```text
              ┌──────────────────────┐
              │ Azure IoT Central     │
              │ Telemetry / Rules     │
              │ Device Metadata       │
              └──────────┬───────────┘
                         │
                         │ Data Export
                         ▼
              ┌──────────────────────┐
              │ Azure Event Hubs      │
              └──────────┬───────────┘
                         │ Capture / Function / Stream処理
                         ▼

┌────────────────────────────────────────────────────┐
│ Azure Data Lake Storage Gen2                        │
│                                                    │
│  iot_events                                        │
│  ┌────────────┬──────────┬───────────────┐         │
│  │ event_id   │ camera_id│ event_time    │         │
│  └────────────┴──────────┴───────────────┘         │
│                                                    │
│  vms_clips                                         │
│  ┌────────────┬──────────┬───────────────┐         │
│  │ clip_id    │ camera_id│ start/end     │         │
│  └────────────┴──────────┴───────────────┘         │
│                                                    │
│  camera_master                                     │
│  ┌────────────┬──────────┬───────────────┐         │
│  │ camera_id  │ site_id  │ vms_camera_id │         │
│  └────────────┴──────────┴───────────────┘         │
│                                                    │
│  join key:                                         │
│  - vmsClipId                                      │
│  - correlationId                                  │
│  - cameraId + timestamp range                     │
└────────────────────┬───────────────────────────────┘
                     │
                     ▼
          [Azure Data Explorer / Fabric / Power BI]
                     │
                     ▼
       検知イベント、設備状態、該当映像URLを横断分析
```

Event Hubs Captureを使うと、Event Hubsに流れたストリーミングデータをBlob StorageまたはData Lake Storageへ自動保存できます。Microsoft Learnでは、Data Lake Storageへの保存やParquet形式でのキャプチャにも触れられています。([Microsoft Learn][5])

---

# 機能の置き場所

## 現場側

| 機能       | Azure構成での置き場所                         | 補足                               |
| -------- | ------------------------------------- | -------------------------------- |
| 映像保存     | VMS / NVR                             | Azure IoT Centralには保存しない         |
| AI映像解析   | EdgePC1                               | GPU搭載、Docker推奨                   |
| AI結果変換   | Event Adapter                         | Azure IoT Central向けペイロードに変換      |
| センサー収集   | EdgePC2 / IoT Edge Module             | OPC-UA、Modbus、MQTTなど             |
| ローカルバッファ | Azure IoT Edge Edge Hub / Local Queue | 通信断対策                            |
| 現場内簡易画面  | 任意のWeb UI / Grafana                   | IoT CentralはクラウドUIなので現場閉域UIは別途用意 |

## クラウド側

| 機能             | Azureサービス                                         |
| -------------- | ------------------------------------------------- |
| デバイス管理・ダッシュボード | Azure IoT Central                                 |
| ルール・通知         | IoT Central Rules + Logic Apps / Functions        |
| ストリーム受信        | Event Hubs                                        |
| データ保存          | Azure Data Lake Storage Gen2                      |
| 時系列分析          | Azure Data Explorer                               |
| BI             | Power BI / Microsoft Fabric                       |
| ETL            | Azure Functions / Stream Analytics / Data Factory |
| 認証・権限          | Microsoft Entra ID                                |
| 監視             | Azure Monitor / Log Analytics                     |

---

# Cumulocity構成との差分

ここが設計上の注意ポイントです。

```text
Cumulocity案
────────────────────────────────
現場に Cumulocity Edge がある
現場だけでIoT管理画面・イベント管理が成立しやすい
本社Coreと同期する親子構成が作りやすい

Azure IoT Central案
────────────────────────────────
IoT Central自体はクラウドSaaS
現場には IoT Edge / IoT Operations を置く
現場内で完結する管理画面は別途必要になりやすい
Data Lake連携はAzureネイティブで強い
Power BI / Fabric との相性が良い
```

つまり、Azureで同じ世界観を作るなら、

```text
Cumulocity Edge
  ↓
Azure IoT Edge + 独自ローカルUI + ローカルDB
```

に分解して考えるのが自然です。

---

# Azure IoT Centralで注意すべき点

## 1. オンプレ完結は苦手

Azure IoT CentralはクラウドSaaSです。
Cumulocity Edgeのように「現場内にIoT管理基盤を丸ごと置く」発想とは違います。

そのため、WAN断時にも現場運用を止めたくないなら、現場側に以下を置く必要があります。

```text
EdgePC2
├─ Azure IoT Edge
├─ Local Queue
├─ Local DB
├─ Local Dashboard
└─ Local Alerting
```

Azure IoT Edge自体は、クラウド接続が切れてもエッジ側で判断を続ける用途に向いています。Microsoftの説明でも、IoT Edgeは分析をデバイス近くに置き、より速い洞察とオフライン意思決定を可能にするものと説明されています。([Microsoft Learn][2])

---

## 2. 「イベント」「アラーム」の表現がCumulocityと違う

Cumulocityでは、

```text
Event
Alarm
Measurement
```

が明確です。

Azure IoT Centralでは、基本は

```text
Telemetry
Property
Command
Rules
```

で考えます。

対応はこんな感じです。

```text
Cumulocity Measurement
  → IoT Central Telemetry

Cumulocity Event
  → IoT Central Telemetry/Event的データ
  → messageTypeやeventTypeで表現

Cumulocity Alarm
  → IoT Central Rule
  → Action Group / Logic Apps / Teams通知 / メール

Cumulocity Inventory
  → IoT Central Device Template / Device Properties
```

AIイベントのJSON例はこうです。

```json
{
  "eventType": "PersonDetected",
  "siteId": "site-001",
  "cameraId": "CAM-001",
  "vmsCameraId": "VMS-CAM-001",
  "eventTime": "2026-06-08T10:15:23.456+09:00",
  "eventStartTime": "2026-06-08T10:15:20.000+09:00",
  "eventEndTime": "2026-06-08T10:15:35.000+09:00",
  "confidence": 0.94,
  "personCount": 2,
  "zone": "north_gate",
  "vmsClipId": "clip-20260608-101523-CAM001",
  "vmsClipUrl": "https://vms.example.local/clips/xxxxx",
  "aiModelVersion": "person-detector-v1.3.2",
  "correlationId": "site001-CAM001-20260608T101523456"
}
```

---

## 3. IoT Centralの長期分析はData Export前提

IoT Centralの画面は運用監視には便利ですが、長期分析やVMSメタデータとの結合分析は、Data Lake側へ出した方がいいです。

おすすめはこれです。

```text
IoT Central
   ↓ Data Export
Event Hubs
   ↓ Capture
Azure Data Lake Storage Gen2
   ↓
Azure Data Explorer / Fabric / Power BI
```

IoT CentralはデータをEvent Hubsへほぼリアルタイムにエクスポートでき、Event Hubs側でCaptureを有効にすればData Lakeへ蓄積できます。([Microsoft Learn][3])

---

# ストレージ構成

```text
┌────────────────────┬────────────────────────────┬────────────────────────┐
│ データ種別           │ 保存先                       │ 理由                     │
├────────────────────┼────────────────────────────┼────────────────────────┤
│ 生映像               │ VMS / NVR                    │ 大容量・録画検索向き       │
│ 映像クリップ          │ VMS / 映像ストレージ          │ 証跡確認・再生向き          │
│ VMSメタデータ         │ ADLS Gen2                    │ イベントと結合分析するため   │
│ AI検知イベント        │ IoT Central → Event Hubs → ADLS│ 監視・分析の両方に使う      │
│ アラーム相当          │ IoT Central Rules / Functions │ 通知・対応フロー向き         │
│ センサー時系列        │ IoT Central → ADLS / ADX      │ 可視化・時系列分析向き       │
│ AIモデルログ          │ ADLS / Log Analytics          │ 精度検証・再学習向き         │
│ デバイス台帳          │ IoT Central Device Properties │ 管理画面で扱いやすい         │
│ 分析用テーブル        │ ADX / Fabric Lakehouse        │ BI・横断分析向き             │
└────────────────────┴────────────────────────────┴────────────────────────┘
```

---

# 最終おすすめ構成

Azure IoT Centralで行くなら、私はこの構成にします。

```text
企業A 現場1
────────────────────────────────────────────

[Camera/Sensor]
   ├──────────────→ [VMS]
   │                  ├─ 映像保存
   │                  ├─ Clip ID生成
   │                  └─ VMS API
   │
   ├──────────────→ [EdgePC1: AI映像解析]
   │                    ├─ 人物検知
   │                    ├─ 人数カウント
   │                    ├─ 侵入検知
   │                    └─ AIイベント生成
   │
   └──────────────→ [EdgePC2: Azure IoT Edge]
                        ├─ Sensor Adapter
                        ├─ Event Adapter
                        ├─ Edge Hub
                        ├─ Store & Forward
                        └─ Local Dashboard任意

Azure Cloud
────────────────────────────────────────────

[Azure IoT Central]
   ├─ Device Template
   ├─ Device Management
   ├─ Telemetry
   ├─ Rules
   ├─ Dashboard
   └─ Commands

        ↓ Data Export

[Azure Event Hubs]
   ├─ Warm Path
   └─ Capture

        ↓

[Azure Data Lake Storage Gen2]
   ├─ iot_events
   ├─ iot_telemetry
   ├─ vms_clips
   ├─ camera_master
   └─ ai_model_logs

        ↓

[Azure Data Explorer / Fabric / Power BI]
   ├─ イベント傾向分析
   ├─ 現場別比較
   ├─ 映像リンク参照
   ├─ 誤検知分析
   └─ AI再学習データ抽出
```

---

# 設計ルール

```text
1. 映像本体はAzure IoT Centralに送らない。
2. 映像本体はVMS/NVRで保存する。
3. IoT CentralにはAI判定結果とセンサー値をTelemetryとして送る。
4. アラーム相当はIoT Central Rulesで判定する。
5. 長期分析はData ExportでEvent Hubs / ADLS / ADXへ逃がす。
6. VMS映像との紐づけには cameraId, eventTime, vmsClipId, correlationId を必ず持たせる。
7. WAN断対策としてEdgePC2にIoT EdgeのStore & Forward、EdgePC1にLocal Queueを置く。
8. 現場でクラウド断時も画面が必要なら、IoT Centralとは別にローカルダッシュボードを置く。
9. VMS URLは直接保存より、VMS連携API経由の短期URL発行が安全。
10. 全機器をNTP/PTPで時刻同期する。
```

---

# 重要な補足

Azure IoT Centralを新規採用する場合は、**ライフサイクルと契約可否を必ずMicrosoftまたは販売パートナーに確認**してください。2024年にIoT Centralの廃止告知をめぐる混乱があり、MicrosoftはAzure IoTへのコミットメントについて説明を出していますが、古い情報や非公式記事も混在しています。([TECHCOMMUNITY.MICROSOFT.COM][6])
長期運用・複数現場・オンプレ/エッジ重視なら、Azure IoT Centralだけでなく **Azure IoT Hub + Azure IoT Edge / Azure IoT Operations + ADLS/Fabric** の構成も比較対象に入れた方が安全です。

[1]: https://azure.microsoft.com/en-us/products/iot-central?utm_source=chatgpt.com "Azure IoT Central - IoT Solution Development"
[2]: https://learn.microsoft.com/en-us/azure/iot-edge/about-iot-edge?utm_source=chatgpt.com "What is Azure IoT Edge"
[3]: https://learn.microsoft.com/en-us/azure/iot-central/core/howto-export-to-event-hubs?utm_source=chatgpt.com "Export data to Event Hubs - Azure IoT Central"
[4]: https://learn.microsoft.com/en-us/azure/iot-central/core/howto-connect-secure-vnet?utm_source=chatgpt.com "Export IoT Central data to a secure virtual network destination"
[5]: https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-capture-overview?utm_source=chatgpt.com "Capture Streaming Events - Azure Event Hubs"
[6]: https://techcommunity.microsoft.com/blog/iotblog/microsofts-commitment-to-azure-iot/4059725?utm_source=chatgpt.com "Microsoft's commitment to Azure IoT"
