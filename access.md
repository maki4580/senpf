<<<<<<< HEAD
自前機能がCumulocityのInventory / Measurements / Events / Alarms / Operationsなどへアクセスするには、基本的に**必ず認証と権限付与が必要**だよ。

ざっくり言うと、認証方式は「自前機能をどこに置くか」で変わるよ。

**1. Cumulocity Microserviceとして動かす場合**
Cumulocity Edge上のMicroservice Hostingに載せる場合は、CumulocityがMicroservice用の実行環境と認証情報を渡してくれる。

Microserviceには実行時に、例えば以下のような環境変数が渡される。

```text
C8Y_BASEURL
C8Y_BOOTSTRAP_TENANT
C8Y_BOOTSTRAP_USER
C8Y_BOOTSTRAP_PASSWORD
C8Y_TENANT
C8Y_USER
C8Y_PASSWORD
```

使い方のイメージはこう。

```text
自前Microservice
  -> C8Y_BASEURL を使って Cumulocity REST API にアクセス
  -> service user の認証情報でAPI実行
  -> requiredRoles に定義された権限の範囲だけ操作可能
```

この方式では、`cumulocity.json`の`requiredRoles`に必要な権限を定義する。ここでいう`requiredRoles`は名前が少し紛らわしいけど、実体は`ROLE_INVENTORY_READ`や`ROLE_ALARM_ADMIN`のような**permission文字列**だよ。

例として、自前機能がInventoryを読み、EventsとAlarmsを書き込むなら、概念的にはこういう権限が必要になる。

```json
{
  "requiredRoles": [
    "ROLE_INVENTORY_READ",
    "ROLE_EVENT_ADMIN",
    "ROLE_ALARM_ADMIN"
  ]
}
```

実際にどのpermissionが必要かはAPIごとに確認する必要があるけど、設計思想は「必要最小限」だよ。

**2. Edge端末上の独立Dockerからアクセスする場合**
Cumulocity Microserviceではなく、Edge端末上の独立Dockerとして動かす場合は、Cumulocityから自動でservice userが渡されるわけではない。

この場合は主に次のどれかを使う。

| 方式 | 使いどころ |
| --- | --- |
| 専用APIユーザー + Basic認証 | シンプルなサーバ間連携 |
| 外部IdPのBearer token | 企業IdP、OAuth2/OIDCと統合したい場合 |
| デバイス認証 / MQTT / SmartREST | デバイスやゲートウェイとしてデータ送信する場合 |
| thin-edge.io | Linux Edge端末をCumulocity管理対象にしたい場合 |

一番分かりやすいのは、Cumulocityに**自前機能専用のAPIユーザー**を作る方式。

```text
自前Docker
  -> Cumulocity Edge REST API
  -> 専用APIユーザーで認証
  -> そのユーザーに付与された権限だけ実行可能
```

例えば、やってはいけないのは、管理者ユーザーをそのまま自前機能に持たせること。漏えいした時の被害が大きすぎるからね。

設計としてはこうするのが安全。

```text
ユーザー名: svc-custom-function
用途: 自前機能からCumulocityへEvents/Alarms/Measurements登録
権限:
  - Inventory READ
  - Event ADMIN
  - Alarm ADMIN
  - Measurement ADMIN
禁止:
  - User管理
  - Tenant管理
  - 不要なOperation発行
```

**3. 外部Bearer tokenを使う場合**
Cumulocityは、外部の認可サーバーが発行したOAuth2 access tokenをBearer tokenとして受け付ける構成もある。

ただし、これは「Bearer tokenを投げれば何でもOK」という意味ではないよ。Cumulocity側で事前設定が必要。

必要になるものはだいたいこれ。

```text
- 外部Bearer token認証を有効化
- JWTのissuer / audience / expirationを検証
- 署名検証用のJWKSや証明書を設定
- claimからCumulocityユーザーまたはアプリケーションIDへマッピング
- claimから権限やアプリケーションアクセスをマッピング
- 必要に応じてintrospectionまたはuserinfo検証を設定
```

この方式は、Azure ADやKeycloakなどと統合して、サービス間認証を企業のIdPに寄せたい場合に向いてる。

```text
自前機能
  -> IdPへclient credentials grantでtoken取得
  -> Authorization: Bearer <access_token>
  -> Cumulocity REST APIへアクセス
  -> Cumulocityがtoken検証・claim mapping
```

**4. SSOとサービス間認証は分けて考える**
ここ、かなり大事。

SSOは基本的に「人間ユーザーがCumulocity UIへログインするため」の仕組み。自前機能の常駐処理やバッチ処理は、人間ユーザーのSSOとは分けるのが普通だよ。

整理するとこう。

| 用途 | 推奨 |
| --- | --- |
| 人がCumulocity UIを見る | SSO |
| UI操作をきっかけにMicroserviceが処理する | 可能ならユーザーコンテキストを引き継ぐ |
| 常駐バッチが定期処理する | service user または client credentials |
| 外部システムが連携する | 専用APIユーザーまたは外部Bearer token |
| デバイスがデータ送信する | デバイス認証、MQTT、SmartREST、thin-edge.io |

**5. APIアクセス時に見るべき権限**
自前機能が何をするかで必要権限が変わる。

| やりたいこと | 必要になる権限の考え方 |
| --- | --- |
| カメラやサービス情報を見る | Inventory READ |
| サービス状態を更新する | Inventory ADMIN相当 |
| 測定値を書き込む | Measurement ADMIN相当 |
| イベントを書き込む | Event ADMIN相当 |
| アラームを作成/更新する | Alarm ADMIN相当 |
| デバイス/サービスへ操作を出す | Operation / Device Control系の権限 |
| Tenant Optionsを読む/書く | Tenant Option系の権限 |

正確なpermission名は採用APIとCumulocityバージョンで確認する必要があるよ。

**おすすめ設計**
今回の構成なら、私はこう分けるのが良いと思う。

```text
A. Cumulocity Microservice方式
  - Cumulocityデータを深く参照する業務ロジック向け
  - requiredRolesで最小権限を定義
  - 設定はTenant Options
  - 状態はInventory / Events / Alarms / Measurementsへ保存

B. 独立Docker方式
  - GPU、特殊デバイス、ローカル永続化が必要な機能向け
  - 専用APIユーザーまたは外部Bearer tokenでREST API利用
  - 認証情報はSecret管理し、定期ローテーション

C. デバイス/ゲートウェイ方式
  - データ送信中心ならMQTT / SmartREST / thin-edge.io
  - Cumulocity上ではManaged Object / Device Serviceとして管理
```

**設計書に入れるなら**
この一文を入れると分かりやすいと思う。

```text
自前実装機能からCumulocity APIへアクセスする場合は、配置方式に応じてMicroservice service user、専用APIユーザー、外部IdPのBearer token、またはデバイス認証を利用する。いずれの場合も、Inventory、Measurements、Events、Alarms、Operations等の利用APIごとに最小権限を付与し、人間ユーザーのSSO認証と常駐処理・外部連携のサービス間認証は分離して設計する。
```

参照元はCumulocity公式の [Microservices general aspects](https://cumulocity.com/docs/microservice-sdk/general-aspects/)、[Using the REST interface](https://cumulocity.com/docs/microservice-sdk/rest/)、[Single sign-on](https://cumulocity.com/docs/authentication/sso/) あたり。

=======
自前機能がCumulocityのInventory / Measurements / Events / Alarms / Operationsなどへアクセスするには、基本的に**必ず認証と権限付与が必要**だよ。

ざっくり言うと、認証方式は「自前機能をどこに置くか」で変わるよ。

**1. Cumulocity Microserviceとして動かす場合**
Cumulocity Edge上のMicroservice Hostingに載せる場合は、CumulocityがMicroservice用の実行環境と認証情報を渡してくれる。

Microserviceには実行時に、例えば以下のような環境変数が渡される。

```text
C8Y_BASEURL
C8Y_BOOTSTRAP_TENANT
C8Y_BOOTSTRAP_USER
C8Y_BOOTSTRAP_PASSWORD
C8Y_TENANT
C8Y_USER
C8Y_PASSWORD
```

使い方のイメージはこう。

```text
自前Microservice
  -> C8Y_BASEURL を使って Cumulocity REST API にアクセス
  -> service user の認証情報でAPI実行
  -> requiredRoles に定義された権限の範囲だけ操作可能
```

この方式では、`cumulocity.json`の`requiredRoles`に必要な権限を定義する。ここでいう`requiredRoles`は名前が少し紛らわしいけど、実体は`ROLE_INVENTORY_READ`や`ROLE_ALARM_ADMIN`のような**permission文字列**だよ。

例として、自前機能がInventoryを読み、EventsとAlarmsを書き込むなら、概念的にはこういう権限が必要になる。

```json
{
  "requiredRoles": [
    "ROLE_INVENTORY_READ",
    "ROLE_EVENT_ADMIN",
    "ROLE_ALARM_ADMIN"
  ]
}
```

実際にどのpermissionが必要かはAPIごとに確認する必要があるけど、設計思想は「必要最小限」だよ。

**2. Edge端末上の独立Dockerからアクセスする場合**
Cumulocity Microserviceではなく、Edge端末上の独立Dockerとして動かす場合は、Cumulocityから自動でservice userが渡されるわけではない。

この場合は主に次のどれかを使う。

| 方式 | 使いどころ |
| --- | --- |
| 専用APIユーザー + Basic認証 | シンプルなサーバ間連携 |
| 外部IdPのBearer token | 企業IdP、OAuth2/OIDCと統合したい場合 |
| デバイス認証 / MQTT / SmartREST | デバイスやゲートウェイとしてデータ送信する場合 |
| thin-edge.io | Linux Edge端末をCumulocity管理対象にしたい場合 |

一番分かりやすいのは、Cumulocityに**自前機能専用のAPIユーザー**を作る方式。

```text
自前Docker
  -> Cumulocity Edge REST API
  -> 専用APIユーザーで認証
  -> そのユーザーに付与された権限だけ実行可能
```

例えば、やってはいけないのは、管理者ユーザーをそのまま自前機能に持たせること。漏えいした時の被害が大きすぎるからね。

設計としてはこうするのが安全。

```text
ユーザー名: svc-custom-function
用途: 自前機能からCumulocityへEvents/Alarms/Measurements登録
権限:
  - Inventory READ
  - Event ADMIN
  - Alarm ADMIN
  - Measurement ADMIN
禁止:
  - User管理
  - Tenant管理
  - 不要なOperation発行
```

**3. 外部Bearer tokenを使う場合**
Cumulocityは、外部の認可サーバーが発行したOAuth2 access tokenをBearer tokenとして受け付ける構成もある。

ただし、これは「Bearer tokenを投げれば何でもOK」という意味ではないよ。Cumulocity側で事前設定が必要。

必要になるものはだいたいこれ。

```text
- 外部Bearer token認証を有効化
- JWTのissuer / audience / expirationを検証
- 署名検証用のJWKSや証明書を設定
- claimからCumulocityユーザーまたはアプリケーションIDへマッピング
- claimから権限やアプリケーションアクセスをマッピング
- 必要に応じてintrospectionまたはuserinfo検証を設定
```

この方式は、Azure ADやKeycloakなどと統合して、サービス間認証を企業のIdPに寄せたい場合に向いてる。

```text
自前機能
  -> IdPへclient credentials grantでtoken取得
  -> Authorization: Bearer <access_token>
  -> Cumulocity REST APIへアクセス
  -> Cumulocityがtoken検証・claim mapping
```

**4. SSOとサービス間認証は分けて考える**
ここ、かなり大事。

SSOは基本的に「人間ユーザーがCumulocity UIへログインするため」の仕組み。自前機能の常駐処理やバッチ処理は、人間ユーザーのSSOとは分けるのが普通だよ。

整理するとこう。

| 用途 | 推奨 |
| --- | --- |
| 人がCumulocity UIを見る | SSO |
| UI操作をきっかけにMicroserviceが処理する | 可能ならユーザーコンテキストを引き継ぐ |
| 常駐バッチが定期処理する | service user または client credentials |
| 外部システムが連携する | 専用APIユーザーまたは外部Bearer token |
| デバイスがデータ送信する | デバイス認証、MQTT、SmartREST、thin-edge.io |

**5. APIアクセス時に見るべき権限**
自前機能が何をするかで必要権限が変わる。

| やりたいこと | 必要になる権限の考え方 |
| --- | --- |
| カメラやサービス情報を見る | Inventory READ |
| サービス状態を更新する | Inventory ADMIN相当 |
| 測定値を書き込む | Measurement ADMIN相当 |
| イベントを書き込む | Event ADMIN相当 |
| アラームを作成/更新する | Alarm ADMIN相当 |
| デバイス/サービスへ操作を出す | Operation / Device Control系の権限 |
| Tenant Optionsを読む/書く | Tenant Option系の権限 |

正確なpermission名は採用APIとCumulocityバージョンで確認する必要があるよ。

**おすすめ設計**
今回の構成なら、私はこう分けるのが良いと思う。

```text
A. Cumulocity Microservice方式
  - Cumulocityデータを深く参照する業務ロジック向け
  - requiredRolesで最小権限を定義
  - 設定はTenant Options
  - 状態はInventory / Events / Alarms / Measurementsへ保存

B. 独立Docker方式
  - GPU、特殊デバイス、ローカル永続化が必要な機能向け
  - 専用APIユーザーまたは外部Bearer tokenでREST API利用
  - 認証情報はSecret管理し、定期ローテーション

C. デバイス/ゲートウェイ方式
  - データ送信中心ならMQTT / SmartREST / thin-edge.io
  - Cumulocity上ではManaged Object / Device Serviceとして管理
```

**設計書に入れるなら**
この一文を入れると分かりやすいと思う。

```text
自前実装機能からCumulocity APIへアクセスする場合は、配置方式に応じてMicroservice service user、専用APIユーザー、外部IdPのBearer token、またはデバイス認証を利用する。いずれの場合も、Inventory、Measurements、Events、Alarms、Operations等の利用APIごとに最小権限を付与し、人間ユーザーのSSO認証と常駐処理・外部連携のサービス間認証は分離して設計する。
```

参照元はCumulocity公式の [Microservices general aspects](https://cumulocity.com/docs/microservice-sdk/general-aspects/)、[Using the REST interface](https://cumulocity.com/docs/microservice-sdk/rest/)、[Single sign-on](https://cumulocity.com/docs/authentication/sso/) あたり。

>>>>>>> 26542e9ec6bf3085f088f18b2d127e58b6bf21ab
自信度は90%。Cumulocityの具体的なpermission名やEdge環境でのMicroservice isolationの扱いは、採用バージョン・契約・実機設定で確認した方がいいよ。