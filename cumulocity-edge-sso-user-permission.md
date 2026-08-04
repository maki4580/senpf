# Cumulocity Edge + 現場IdP（SSO）構成におけるユーザー・権限の管理と連携

**作成日**: 2026-08-05
**対象バージョン**: Cumulocity Release 2026（公式ドキュメント記載内容）
**前提シナリオ**: Cumulocity Edge を現場に設置し、同一現場に立てたIdP（Keycloak等）と Cumulocity Edge の SSO 設定で連携した状態

---

## 1. 全体像：どこに何を登録するか

| 管理対象 | 登録場所 | 備考 |
|---|---|---|
| **アカウント本体**（ID・パスワード・MFA・入退社） | **現場のIdP** | Cumulocity側に事前登録不要 |
| **グループ／組織属性** | **現場のIdP** | JWTのクレームとして払い出す |
| **権限の定義**（グローバルロール、インベントリロール、アプリケーションアクセス） | **Cumulocity Edge（edgeテナント）** | Administration アプリで定義 |
| **IdPのクレーム → Cumulocityロールの対応表** | **Cumulocity Edgeの SSO設定「アクセスマッピング」** | ここが唯一の連携点 |

要するに **「誰か」はIdP、「何ができるか」はEdge、その紐付けはEdgeのアクセスマッピング** という分担。

---

## 2. 前提：Edge には2つのテナントがある

- `edge` テナント … `https://<domain_name>`（例 `https://myown.iot.com`）— 業務ユーザーが使う本体
- `management` テナント … `https://management-<domain_name>` — Edge運用管理用

SSO設定は**テナントごと**の設定。通常は `edge` テナントに現場IdPを設定し、`management` 側はローカル admin を残す構成が現実的（後述のフォールバック問題のため）。

DNSに両ドメインの登録が必要で、IPアドレス直打ちではなく**ドメイン名でのアクセスが前提**（オンプレでは「ドメインベースのテナント解決」が正しく構成されていること、がSSOの前提条件として明記されている）。

---

## 3. ユーザーはどう登録される？ → 事前登録不要、初回ログインで自動生成

1. ユーザーがブラウザで `https://myown.iot.com` にアクセス
2. Edge がIdPへリダイレクト（OAuth2 authorization code grant）
3. IdPで認証 → JWT（アクセストークン／IDトークン）が Edge に返る
4. Edge がトークン署名を検証（JWKS URI 等）
5. **初回ログイン時に Edge 内へユーザーが自動作成される**（いわゆるシャドウユーザー）
   - ユーザー名 = SSO設定の「User ID」で指定したトークンのトップレベルクレーム
   - 氏名・メール・電話は「ユーザーデータマッピング」で任意のクレームから取り込み可
6. Cookieでセッション確立

Administration の「ユーザー」一覧には外部認証由来のユーザーとして並ぶが、**パスワードリセットはCumulocity側では不可**。

---

## 4. 権限はどう連携される？ → アクセスマッピング

SSO設定画面（Administration > 設定 > 認証 > シングルサインオン）で、JWTクレームの条件（`=`, `!=`, `contains`、ワイルドカード `*`、AND結合）に対して、以下3種を割り当てる。

- **グローバルロール** … システム全体の権限（Inventory / Alarms / Device control 等 × READ/CREATE/UPDATE/ADMIN）
- **アプリケーション（デフォルトアプリ）** … どのアプリを開けるか
- **インベントリロール** … 特定のデバイスグループに対する Manager / Reader

例：IdPが `groups: ["iot-operator-north"]` を出す → 該当ルールで「グローバルロール: business」「アプリ: cockpit + 自社アプリ」「インベントリロール: 北エリアグループのManager」を付与。

> **注意1**: どのルールにもマッチしないと `access denied` でログイン不可（デフォルト拒否）。
> **注意2**: 「Own user management」の READ 権限を含むロールを必ず付与しないと、そもそもログインできない。最頻出のハマりどころ。

### 再割当てポリシー（3択）

| 設定 | 挙動 |
|---|---|
| 作成時のみ適用 | 初回ログイン時だけロール付与。以降はEdge側で手動管理 |
| 毎回再割当て（ルール外は保持） | ルールに載るロールのみ毎ログインで上書き、他は温存 |
| **毎回再割当て（ルール外はクリア）★デフォルト** | 毎ログインで全ロールを再計算。**Edge側で手動編集しても次回ログインで上書きされる** |

デフォルトのままなら「権限の実体はIdPのグループが単一情報源」になる。手動運用を混ぜたい場合は上2つを選ぶ必要がある。

---

## 5. Edgeに登録したアプリのログインはどうなる？

**アプリごとの独立ログインは発生しない。** Edgeにホストされたアプリ（Cockpit、Device Management、および自社のカスタムWebアプリ）は、すべて**プラットフォームの同一セッション（Cookie）を共有**する。

```
ブラウザ → https://myown.iot.com/apps/<your-app>
   ↓ 未ログインなら
   → IdP（現場）でログイン
   ↓ JWT + Cookie
   → アプリ画面表示（以後は同一ドメイン内のどのアプリも再ログイン不要）
```

アプリ側の制御は2段階：

1. **アプリを開けるか** = アプリケーションアクセス
   - ユーザー個別付与／グローバルロールに含めて付与／SSOアクセスマッピングで付与、のいずれか
   - SSOユーザーではデフォルト設定だと手動付与が上書きされるため、**アクセスマッピング側で管理するのが正解**
   - 前提として、そのアプリが `edge` テナントにインストール／サブスクライブ済みであること
2. **アプリ内で何ができるか** = ロール（パーミッション）
   - カスタムマイクロサービスは manifest の `roles` で独自パーミッション（例 `ROLE_MYAPP_ADMIN`）を宣言でき、それがグローバルロール編集画面の権限一覧に現れる → そのロールをアクセスマッピングに含める
   - manifest の `requiredRoles` は**マイクロサービスのサービスユーザー用**であり、人間のSSOユーザーとは別物

---

## 6. 現場IdP側の要件・設定

- **OAuth2 authorization code grant** 必須。**SAMLは非対応**
- トークンはJWTで、`iss` / `aud` / `exp` + 一意のユーザー識別子を含むこと
- 署名鍵は**RSAのみ**（楕円曲線鍵は非対応）。検証方法は JWKS URI ／ Azure AD 証明書ディスカバリ ／ ADFSマニフェスト ／ 手動公開鍵登録
- **リダイレクトURI に Edge のドメイン**（`https://myown.iot.com/...`）をIdP側のクライアント設定へ登録
- Keycloakならバックチャネルログアウト対応：Backchannel Logout URL に `https://<domain>/user/logout/oidc` を設定すると、IdP側ログアウトでEdge側セッションも終了
- ブラウザのCookie有効化が必須（SSO機能がCookieベースのため）

閉域環境では、**Edge → IdPのJWKSエンドポイント到達性**と、**ブラウザ → IdP到達性**の両方が必要。

---

## 7. 運用上の注意（設計時に押さえるべき点）

- ログインモードで「Single sign-on」を選ぶと、**ログイン画面から Basic Auth / OAI-Secure の選択肢が消える**。IdP障害時に誰もEdgeに入れなくなるリスクがあるため、`management` テナントはローカル admin を維持する等のフォールバック設計を必ず入れる。
- ローカルユーザーを作成しようとすると「SSOでログインできないローカルユーザーを作ろうとしている」旨の警告が出る。SSOユーザーとローカルユーザーの併存自体は可能だが、UI上のログイン導線は上記のモード設定に従う。
- デバイス認証（デバイス資格情報／証明書）とマイクロサービスのサービスユーザーは**SSOの対象外**。SSOはあくまで人間のUIログイン用。
- IdPのグループ設計＝Cumulocityの権限設計になるため、**先にEdge側でグローバルロール／インベントリロールを定義 → それに対応するグループをIdPで作る**という順序で進めるのが手戻りが少ない。

---

## 8. 出典

- [Single sign-on - Cumulocity Release 2026 documentation](https://cumulocity.com/docs/2026/authentication/sso/)
- [Single sign-on - Cumulocity documentation (current)](https://cumulocity.com/docs/authentication/sso/)
- [Basic settings (authentication / login modes) - Release 2026](https://cumulocity.com/docs/2026/authentication/basic-settings/)
- [Managing users - Cumulocity documentation](https://cumulocity.com/docs/standard-tenant/managing-users/)
- [Managing permissions - Cumulocity documentation](https://cumulocity.com/docs/standard-tenant/managing-permissions/)
- [Edge Introduction - Release 2026](https://cumulocity.com/docs/2026/edge/edge-introduction/)
- [Installation (Edge) - Release 2026](https://cumulocity.com/docs/2026/edge/installing-edge/)
- [Managing Edge - Release 2026](https://cumulocity.com/docs/2026/edge/manage-edge/)
- [Microservice SDK - General aspects](https://cumulocity.com/docs/microservice-sdk/general-aspects/)

---

## 9. 未確定事項 / 実機確認すべき点

- **Edge専用のSSO解説ページは公式ドキュメントに存在しない。** 本書はプラットフォーム共通のSSO仕様と、Edgeのテナント／ドメイン構成を突き合わせた整理である。
- `edge` テナントの Administration > 設定 > 認証 に「シングルサインオン」タブが実際に表示されるか（テナントの権限設定次第で非表示になり得る）を実機で最初に確認すること。
- SSO設定を Management テナント専用に制限できる仕組みがあるため、Edge の `edge` テナントから設定変更可能かはインストール構成に依存する可能性がある。
