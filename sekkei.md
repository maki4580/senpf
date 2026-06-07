うん、この構成は **大枠ではかなり良い** です。
ただし、いくつか「そのままだと後で刺さりそうなトゲ」があります。特に注意するのは **CumulocityとVMSの責務分離**、**映像とイベントの紐づけキー**、**Data Lakeへの流し方**、**EdgePC1とEdgePC2の役割境界** です。

---

# まず結論

提示された構成は成立します。

```text
企業A 現場1
カメラ/センサー
   ↓
EdgePC1：映像解析
   ↓
EdgePC2：Cumulocity Edge
   ↓
企業A 本社：Cumulocity Core
   ↓
企業A 本社：Data Lake
```

映像系は別系統で、

```text
企業A 現場1
カメラ/センサー
   ↓
VMS
```

これも正しいです。

ただし、より実運用向けにするなら、こう整理した方が安全です。

```text
映像そのもの        → VMS / 映像ストレージ
AI判定結果          → Cumulocity Edge
イベント/アラーム    → Cumulocity Core
長期分析データ       → Data Lake
映像とイベントの紐づけ → eventId / cameraId / timestamp / clipUrl
```

Cumulocity Edgeは工場などの現場に置くオンサイト向けのCumulocityで、産業PCやローカルサーバ上で動作する位置づけです。公式ドキュメントでも、Edgeはオンサイトに置く「single-server variant of the Cumulocity platform」と説明されています。([Cumulocity][1])

---

# 指摘したい点

## 1. 「カメラ/センサー → EdgePC1」は少し混ざっている

カメラ映像はEdgePC1に流してよいです。
でも、温度センサー、振動センサー、PLCなどの通常IoTデータまで全部EdgePC1に通す必要はありません。

おすすめはこうです。

```text
カメラ映像       → EdgePC1：映像解析
センサーデータ   → EdgePC2：Cumulocity Edge
```

つまり、EdgePC1は **映像AI専用**、EdgePC2は **IoT管理・イベント集約専用** に分ける。

```text
[Camera] ───────→ [EdgePC1: AI映像解析]
                      │
                      │ AIイベント
                      ▼
[Sensor/PLC] ───→ [EdgePC2: Cumulocity Edge]
```

この方が、GPU負荷でCumulocity Edge側が巻き添えを食らいません。
AI処理はCPU/GPUを激しく使うので、Cumulocity Edgeとは分離した方が保守しやすいです。

---

## 2. Cumulocityには映像を入れない方がいい

ここは今の考え方でOKです。

Cumulocityには、映像ファイルではなく、

```text
人を検知した
人数は3人
カメラIDはCAM-001
場所は現場1 北ゲート
該当映像はVMSのclipId=xxxx
```

のような **メタデータ** を入れるべきです。

Cumulocityはデバイス接続、管理、監視、イベント、アラーム、分析・可視化のIoT基盤として使うのが自然です。DataHubもCumulocity上のIoTデータをData Lakeへオフロードし、SQL分析できるようにするための機能として説明されています。([Cumulocity][2])

映像本体までCumulocityに入れようとすると、ストレージ・帯域・検索・保持期間が一気に重くなります。映像はVMS、イベントはCumulocity、この分離がきれいです。

---

## 3. VMS映像とCumulocityイベントを紐づける設計が最重要

今回の肝はここです。

単にCumulocityへイベントを入れるだけだと、後で分析するときに、

```text
このイベントの映像はどれ？
この検知の10秒前から30秒後の映像を見たい
同じ人物検知がVMS上ではどの録画に対応する？
```

が分からなくなります。

なので、Cumulocityのイベントには必ず以下を持たせるべきです。

```json
{
  "type": "camera_PersonDetected",
  "text": "Person detected at 現場1 北ゲート",
  "time": "2026-06-08T10:15:23.456+09:00",
  "source": {
    "id": "Cumulocity上のCamera ManagedObject ID"
  },
  "siteId": "site-001",
  "cameraId": "CAM-001",
  "vmsCameraId": "VMS-CAM-001",
  "vmsClipId": "clip-20260608-101523-CAM001",
  "vmsClipUrl": "https://vms.example.local/clips/clip-20260608-101523-CAM001",
  "eventStartTime": "2026-06-08T10:15:20.000+09:00",
  "eventEndTime": "2026-06-08T10:15:35.000+09:00",
  "confidence": 0.94,
  "personCount": 2,
  "zone": "north_gate",
  "aiModelVersion": "person-detector-v1.3.2",
  "correlationId": "site001-CAM001-20260608T101523456"
}
```

特に重要なのはこの5つです。

| 項目                                         | 用途                            |
| ------------------------------------------ | ----------------------------- |
| `cameraId`                                 | Cumulocity側のカメラ識別             |
| `vmsCameraId`                              | VMS側のカメラ識別                    |
| `time` / `eventStartTime` / `eventEndTime` | 映像検索の時間軸                      |
| `vmsClipId` / `vmsClipUrl`                 | VMS映像への直接参照                   |
| `correlationId`                            | Data Lake上でイベントと映像メタ情報を結合するキー |

ここがないと、後で分析沼に足を取られます。ぬかるみデータ湖になっちゃう 🫠

---

# 推奨する製品・機能・ストレージ構成

## 全体の製品構成

```text
┌──────────────────────────────────────────────┐
│ 企業A 現場1                                  │
│                                              │
│  ┌─────────────┐        ┌─────────────────┐ │
│  │ 監視カメラ   │───────→│ VMS / NVR        │ │
│  └──────┬──────┘ 映像   │ 映像保存          │ │
│         │               │ クリップ管理      │ │
│         │ RTSP/ONVIF    └─────────────────┘ │
│         │                                    │
│         ▼                                    │
│  ┌─────────────────┐                        │
│  │ EdgePC1          │                        │
│  │ AI映像解析        │                        │
│  │ - 人物検知        │                        │
│  │ - 人数カウント    │                        │
│  │ - 侵入検知        │                        │
│  │ - イベント生成    │                        │
│  └──────┬──────────┘                        │
│         │ AIイベントJSON                     │
│         ▼                                    │
│  ┌─────────────────┐                        │
│  │ EdgePC2          │                        │
│  │ Cumulocity Edge  │                        │
│  │ - Device Mgmt    │                        │
│  │ - Inventory      │                        │
│  │ - Events         │                        │
│  │ - Alarms         │                        │
│  │ - Measurements   │                        │
│  └──────┬──────────┘                        │
│         │ 同期/転送                           │
└─────────┼────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────┐
│ 企業A 本社                                    │
│                                              │
│  ┌─────────────────┐                        │
│  │ Cumulocity Core  │                        │
│  │ - 全現場統合管理  │                        │
│  │ - Dashboard      │                        │
│  │ - Alarm管理      │                        │
│  │ - Streaming分析  │                        │
│  └──────┬──────────┘                        │
│         │ DataHub / API / ETL                │
│         ▼                                    │
│  ┌─────────────────┐                        │
│  │ Data Lake        │                        │
│  │ - イベント履歴    │                        │
│  │ - センサー履歴    │                        │
│  │ - VMSメタデータ   │                        │
│  │ - 分析用テーブル  │                        │
│  └─────────────────┘                        │
└──────────────────────────────────────────────┘
```

---

# 機能の置き場所

## 現場側

| コンポーネント       | 役割                                   |
| ------------- | ------------------------------------ |
| カメラ           | 映像取得                                 |
| センサー/PLC      | 温度、振動、設備状態などの取得                      |
| VMS/NVR       | 映像保存、録画検索、クリップ生成                     |
| EdgePC1       | AI映像解析、人物検知、侵入検知、人数カウント              |
| Event Adapter | AI解析結果をCumulocity形式に変換               |
| EdgePC2       | Cumulocity Edge。現場内のデバイス・イベント・アラーム管理 |

## 本社側

| コンポーネント            | 役割                                   |
| ------------------ | ------------------------------------ |
| Cumulocity Core    | 複数現場の統合管理、ダッシュボード、アラーム、分析            |
| Cumulocity DataHub | Cumulocityの運用DBからData Lakeへデータをオフロード |
| Data Lake          | 長期保存、横断分析、BI/ML分析                    |
| BIツール              | Power BI、Tableau、Grafanaなど           |
| 分析基盤               | Spark、Dremio、Trino、Python、Notebookなど |

Cumulocity DataHubは、Cumulocityプラットフォームの運用DBからData Lakeへデータをオフロードし、SQLで分析できるようにする機能として説明されています。ドキュメントでは、Cumulocity内のデータはドキュメント形式で管理され、DataHubが分析しやすいリレーショナル/カラム指向形式へ変換する、とされています。([Cumulocity][3])

---

# ストレージ構成

```text
┌────────────────────┬──────────────────────┬────────────────────────┐
│ データ種別           │ 保存先                 │ 理由                     │
├────────────────────┼──────────────────────┼────────────────────────┤
│ 生映像               │ VMS/NVR                │ 大容量・録画検索向き       │
│ 映像クリップ          │ VMSまたは映像ストレージ  │ 証跡確認向き               │
│ AI検知イベント        │ Cumulocity Edge/Core    │ 監視・通知・運用向き        │
│ アラーム              │ Cumulocity Edge/Core    │ 対応管理向き               │
│ センサー時系列        │ Cumulocity Edge/Core    │ 設備監視・可視化向き        │
│ 長期IoT履歴           │ Data Lake               │ 分析・機械学習向き          │
│ VMS映像メタデータ      │ Data Lake               │ イベントとの結合分析向き     │
│ AIモデル実行ログ       │ Data Lake / ログ基盤     │ 精度検証・再学習向き        │
└────────────────────┴──────────────────────┴────────────────────────┘
```

Cumulocityは「現在の運用状態を見る場所」、Data Lakeは「長期分析する場所」、VMSは「映像を見る場所」と分けるのがよいです。

---

# データフロー図

## 1. IoTイベント・AIイベントのデータフロー

```text
企業A 現場1
────────────────────────────────────────────

[カメラ]
   │
   │ RTSP / ONVIF
   ▼
[EdgePC1: AI映像解析]
   │
   │ 例:
   │ - person detected
   │ - personCount = 2
   │ - confidence = 0.94
   │ - cameraId = CAM-001
   │ - timestamp = 2026-06-08T10:15:23+09:00
   │ - vmsClipId = xxxx
   │
   ▼
[Event Adapter]
   │
   │ REST / MQTT
   ▼
[EdgePC2: Cumulocity Edge]
   │
   │ Events / Alarms / Measurements
   │
   ▼
[企業A 本社: Cumulocity Core]
   │
   │ DataHub / API / ETL
   ▼
[企業A 本社: Data Lake]
   │
   ▼
[BI / 分析 / ML / レポート]
```

---

## 2. 映像データフロー

```text
企業A 現場1
────────────────────────────────────────────

[カメラ]
   │
   │ 映像ストリーム
   │ RTSP / ONVIF / 独自プロトコル
   ▼
[VMS / NVR]
   │
   ├─ 常時録画
   ├─ イベント録画
   ├─ クリップ生成
   ├─ サムネイル生成
   └─ 映像検索API
```

---

## 3. イベントと映像を紐づける分析フロー

ここが今回の追加要件ですね。

```text
                 ┌──────────────────────┐
                 │ Cumulocity Core       │
                 │ Events / Alarms       │
                 │ Measurements          │
                 └──────────┬───────────┘
                            │
                            │ DataHub / ETL
                            ▼
┌────────────────────────────────────────────────┐
│ Data Lake                                      │
│                                                │
│  c8y_events                                    │
│  ┌────────────┬──────────┬───────────────┐     │
│  │ event_id   │ camera_id│ timestamp     │     │
│  └────────────┴──────────┴───────────────┘     │
│                                                │
│  vms_clips                                     │
│  ┌────────────┬──────────┬───────────────┐     │
│  │ clip_id    │ camera_id│ start/end     │     │
│  └────────────┴──────────┴───────────────┘     │
│                                                │
│  join key: camera_id + timestamp range         │
│  または correlationId / vmsClipId              │
└────────────────────┬───────────────────────────┘
                     │
                     ▼
              [分析処理 / BI / ML]
                     │
                     ▼
          「検知イベント発生時の映像を参照」
```

---

# 分析用のデータモデル例

Data Lakeでは、最低限この3テーブルを作ると使いやすいです。

## `iot_events`

Cumulocity由来のイベント。

```text
event_id
tenant_id
site_id
source_managed_object_id
camera_id
event_type
event_time
severity
text
confidence
person_count
zone
vms_clip_id
correlation_id
```

## `vms_clips`

VMS由来の映像クリップ情報。

```text
vms_clip_id
site_id
camera_id
start_time
end_time
duration_sec
storage_path
playback_url
thumbnail_url
retention_until
```

## `camera_master`

カメラ台帳。

```text
camera_id
vms_camera_id
cumulocity_managed_object_id
site_id
area
zone
ip_address
model
enabled
```

結合イメージはこうです。

```sql
SELECT
  e.event_id,
  e.event_time,
  e.event_type,
  e.person_count,
  c.playback_url
FROM iot_events e
JOIN vms_clips c
  ON e.camera_id = c.camera_id
 AND e.event_time BETWEEN c.start_time AND c.end_time;
```

より堅くするなら、AIイベント生成時に `vms_clip_id` まで確定させて、時間範囲JOINに頼らない設計がベターです。

---

# Cumulocity内のデータ種別の使い分け

| 判定結果       | Cumulocity上の型       | 例                       |
| ---------- | ------------------- | ----------------------- |
| 人物を検知した    | Event               | PersonDetected          |
| 立入禁止エリアに侵入 | Alarm               | RestrictedAreaIntrusion |
| 人数カウント     | Measurement         | personCount = 3         |
| AIモデル異常    | Alarm               | AIModelError            |
| VMS接続断     | Alarm               | VMSConnectionLost       |
| カメラ死活      | Measurement / Alarm | online=false            |
| GPU使用率     | Measurement         | gpuUtilization = 82%    |

CumulocityのStreaming AnalyticsはApamaベースで、リアルタイムのフィルタ、集約、パターン検知、アクション生成に使えると説明されています。したがって「同じカメラで5分以内に人物検知が10回以上ならアラーム化」みたいな処理はCumulocity側に寄せてもよいです。([Cumulocity][4])

---

# おかしな点・修正した方がいい点

## 指摘1：EdgePC1とEdgePC2間の通信断対策が必要

今の構成だと、

```text
EdgePC1 → EdgePC2
```

が切れたときに、AIイベントが消える可能性があります。

対策として、EdgePC1側に **ローカルキュー** を置くべきです。

```text
EdgePC1
├─ AI解析
├─ Event Adapter
└─ Local Queue
   ├─ SQLite
   ├─ Redis
   └─ MQTT Broker
```

推奨は以下です。

```text
AI解析結果
   ↓
Local Queue
   ↓
Cumulocity Edgeへ送信
   ↓
失敗したら再送
```

---

## 指摘2：時刻同期が絶対に必要

映像とイベントを紐づけるなら、全機器の時刻ズレが命取りです。

```text
Camera
VMS
EdgePC1
EdgePC2
Cumulocity Core
Data Lake
```

全部をNTP同期してください。
高精度にやるならPTPも検討です。

特にこの結合をするなら、

```text
cameraId + timestamp
```

が命綱になります。

時刻が30秒ズレると、別の人が映っている映像を引っ張る可能性があります。これはけっこう怖いです。

---

## 指摘3：VMSの映像URLをCumulocityに入れる場合は権限設計が必要

`vmsClipUrl` をイベントに持たせるのは便利です。

でも、CumulocityのユーザーがそのURLを踏んだときに、VMS側の認証・認可をどうするか決める必要があります。

選択肢はこのあたり。

```text
方式A：VMSのログイン画面に飛ばす
方式B：短時間だけ有効な署名付きURLを発行する
方式C：CumulocityからVMS連携マイクロサービス経由で参照する
方式D：Data Lake側にサムネイルだけ置く
```

おすすめは **C：VMS連携マイクロサービス経由** です。

```text
Cumulocity Dashboard
   ↓
VMS連携Microservice
   ↓
VMS API
   ↓
映像再生URL/サムネイル取得
```

直接URLをイベントにベタ書きすると、URL変更・認証変更・セキュリティ事故に弱くなります。

---

## 指摘4：Data LakeにはVMSの「映像本体」ではなく「映像メタデータ」を置く方がよい

Data Lakeに映像本体まで置くかは要検討です。

多くの場合は、

```text
VMS              → 映像本体
Data Lake        → 映像メタデータ
Cumulocity       → イベント/アラーム
```

で十分です。

Data Lakeに置くべきVMSメタデータは、

```text
clip_id
camera_id
start_time
end_time
file_path
playback_url
thumbnail_url
event_id
```

です。

映像本体までData Lakeに持つのは、AI再学習や長期証跡が必要な場合だけでOKです。

---

## 指摘5：「Cumulocity Core」という呼び名は要確認

構成上は「本社側の中央Cumulocity」を **Core** と呼ぶのは分かりやすいです。
ただ、製品契約や正式名称として「Cumulocity Core」と呼ぶかは、販売元・契約形態で確認した方がよいです。

公式ドキュメント上は、Cumulocity Edgeがオンサイト版で、中央側は通常のCumulocity Platformとして説明されることが多いです。公式のドキュメントページでも、Edgeはオンサイトソリューション、Cumulocity全体はデバイス接続・管理・制御・アプリケーション開発・分析のドキュメント群として整理されています。([Cumulocity][5])

設計書では、誤解を避けるためにこう書くのがよさそうです。

```text
企業A本社：Cumulocity Platform / Central Cumulocity
通称：Cumulocity Core
```

---

# 改善後の推奨構成図

```text
企業A 現場1
=================================================================

 [Camera-01..N] ───────────────┬──────────────→ [VMS / NVR]
      │                        │                 │
      │ RTSP/ONVIF             │                 ├─ 映像録画
      │                        │                 ├─ クリップ生成
      │                        │                 ├─ サムネイル
      │                        │                 └─ VMS API
      │                        │
      ▼                        │
 ┌─────────────────────┐       │
 │ EdgePC1              │       │
 │ AI映像解析            │       │
 │ - 人物検知            │       │
 │ - 侵入検知            │       │
 │ - 人数カウント        │       │
 │ - モデル管理          │       │
 │ - Local Queue         │       │
 └─────────┬───────────┘       │
           │                   │
           │ AIイベントJSON     │ VMS clipId / playbackUrl
           ▼                   │
 ┌─────────────────────┐       │
 │ Event Adapter        │◀──────┘
 │ - Cumulocity形式変換  │
 │ - cameraId変換        │
 │ - correlationId付与   │
 │ - vmsClipId付与       │
 └─────────┬───────────┘
           │ REST / MQTT
           ▼
 ┌─────────────────────┐
 │ EdgePC2              │
 │ Cumulocity Edge       │
 │ - Inventory           │
 │ - Device Management   │
 │ - Events              │
 │ - Alarms              │
 │ - Measurements        │
 │ - Local Dashboard     │
 └─────────┬───────────┘
           │
           │ 同期/転送
           ▼

企業A 本社
=================================================================

 ┌─────────────────────┐
 │ Cumulocity Platform  │
 │ 通称: Core           │
 │ - 全現場統合監視      │
 │ - Dashboard          │
 │ - Alarm Management   │
 │ - Streaming Analytics│
 │ - Microservices      │
 └─────────┬───────────┘
           │
           │ DataHub / API / ETL
           ▼
 ┌─────────────────────┐
 │ Data Lake            │
 │ - c8y_events          │
 │ - c8y_measurements    │
 │ - c8y_alarms          │
 │ - vms_clips           │
 │ - camera_master       │
 │ - ai_model_logs       │
 └─────────┬───────────┘
           │
           ▼
 ┌─────────────────────┐
 │ BI / 分析 / ML        │
 │ - 検知傾向分析         │
 │ - 現場別リスク分析     │
 │ - 映像参照分析         │
 │ - AI精度評価           │
 └─────────────────────┘
```

---

# データフロー詳細

## A. 映像解析イベント登録

```text
1. Camera が映像を出力
        │
        ▼
2. EdgePC1 が映像を取得してAI解析
        │
        ▼
3. 人物検知/侵入検知/人数カウントを生成
        │
        ▼
4. Event Adapter がCumulocity形式へ変換
        │
        ├─ cameraId
        ├─ timestamp
        ├─ confidence
        ├─ personCount
        ├─ zone
        ├─ vmsClipId
        └─ correlationId
        │
        ▼
5. Cumulocity Edgeへ送信
        │
        ▼
6. Cumulocity Coreへ同期/転送
        │
        ▼
7. DataHub/API/ETLでData Lakeへ保存
```

---

## B. 映像保存

```text
1. Camera が映像を出力
        │
        ▼
2. VMS/NVR が録画
        │
        ├─ 常時録画
        ├─ イベント録画
        ├─ クリップ生成
        └─ サムネイル生成
        │
        ▼
3. VMSメタデータをData Lakeへ連携
        │
        ├─ cameraId
        ├─ clipId
        ├─ startTime
        ├─ endTime
        ├─ playbackUrl
        └─ thumbnailUrl
```

---

## C. 分析処理

```text
Data Lake
  │
  ├─ Cumulocity由来イベント
  │    event_id, camera_id, event_time, person_count, alarm_status
  │
  ├─ VMS由来映像メタデータ
  │    clip_id, camera_id, start_time, end_time, playback_url
  │
  └─ カメラ台帳
       camera_id, site_id, area, zone

          │
          ▼

結合処理
  camera_id + event_time between start_time and end_time
  または
  vms_clip_id / correlationId

          │
          ▼

分析結果
  - 侵入検知件数
  - 時間帯別の人物検知傾向
  - カメラ別の誤検知率
  - アラーム対応時間
  - 該当映像へのリンク
```

---

# ツールの組み合わせ案

## 最小構成

```text
Cumulocity Edge
Cumulocity Platform/Core
VMS
AI解析アプリ
Event Adapter
Data Lake
BIツール
```

## 現実的な実装構成

```text
現場側
├─ VMS / NVR
├─ EdgePC1
│  ├─ Docker
│  ├─ NVIDIA GPU
│  ├─ Python / FastAPI
│  ├─ YOLO / OpenVINO / DeepStream
│  └─ Local Queue
│
├─ EdgePC2
│  └─ Cumulocity Edge
│
└─ NTP Server

本社側
├─ Cumulocity Platform/Core
├─ Cumulocity DataHub
├─ Data Lake
│  ├─ S3互換ストレージ / HDFS / Azure Data Lake等
│  └─ Parquet / Iceberg / Delta Lake等
├─ SQL Query Engine
│  ├─ Dremio
│  ├─ Trino
│  └─ Spark
└─ BI
   ├─ Power BI
   ├─ Tableau
   └─ Grafana
```

DataHubはData LakeとBI/分析ツールをつなぐ用途に向いており、Cumulocityの製品説明でもData Lakeへのオフロード、BIツール連携、オープン形式でのクエリが説明されています。([Cumulocity][6])

---

# 最終的なおすすめ構成

一番バランスがいいのはこれです。

```text
企業A 現場1
────────────────────────────────────────────

[Camera/Sensor]
   ├──────────────→ [VMS]
   │                  └─ 映像保存
   │
   ├──────────────→ [EdgePC1: AI映像解析]
   │                    └─ AIイベント生成
   │
   └──────────────→ [EdgePC2: Cumulocity Edge]
                        ├─ センサーデータ収集
                        ├─ AIイベント受信
                        ├─ アラーム生成
                        └─ 現場ダッシュボード

企業A 本社
────────────────────────────────────────────

[Cumulocity Platform/Core]
   ├─ 全現場統合監視
   ├─ アラーム管理
   ├─ ダッシュボード
   └─ Streaming Analytics

        ↓

[Data Lake]
   ├─ Cumulocityイベント
   ├─ センサー時系列
   ├─ VMSメタデータ
   ├─ カメラ台帳
   └─ AIモデルログ

        ↓

[分析/BI/ML]
   ├─ イベント傾向分析
   ├─ 現場別比較
   ├─ 映像参照分析
   ├─ 誤検知分析
   └─ 再学習データ抽出
```

---

# この構成での設計ルール

最後に、設計書にそのまま書ける粒度でまとめます。

```text
1. 映像本体はCumulocityに保存しない。
2. 映像本体はVMS/NVRで保存する。
3. CumulocityにはAI判定結果、イベント、アラーム、人数などの数値を保存する。
4. Cumulocityイベントには cameraId, timestamp, vmsClipId, correlationId を必ず付与する。
5. Data LakeにはCumulocityデータとVMSメタデータを保存する。
6. 分析処理では cameraId + timestamp、または vmsClipId / correlationId で結合する。
7. EdgePC1はAI処理専用、EdgePC2はCumulocity Edge専用に分離する。
8. EdgePC1にはローカルキューを持たせ、通信断時もイベントを欠落させない。
9. 全機器をNTP/PTPで時刻同期する。
10. VMS映像参照は直接URLではなく、可能ならVMS連携マイクロサービス経由にする。
```

全体として、あなたの構成は「CumulocityをIoTイベントの司令塔、VMSを映像の金庫、Data Lakeを分析の大鍋にする」形で、とても自然です。
修正すべき最大ポイントは **“映像とイベントを後から確実に結合できるID設計”** です。これさえ最初に握っておけば、あとから分析・BI・AI再学習まできれいに伸ばせます。

[1]: https://cumulocity.com/docs/2026/edge/edge-introduction/?utm_source=chatgpt.com "Edge - Cumulocity Release 2026 documentation"
[2]: https://www.cumulocity.com/?utm_source=chatgpt.com "Cumulocity | Cumulocity"
[3]: https://cumulocity.com/docs/datahub/working-with-datahub/?utm_source=chatgpt.com "Working with Cumulocity DataHub"
[4]: https://cumulocity.com/docs/streaming-analytics/introduction-analytics/?utm_source=chatgpt.com "Introduction - Cumulocity documentation"
[5]: https://cumulocity.com/docs/?utm_source=chatgpt.com "Cumulocity documentation"
[6]: https://www.cumulocity.com/product/datahub/?utm_source=chatgpt.com "DataHub"
