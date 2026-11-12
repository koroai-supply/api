# 頃合 Public API

商品を探し、見積もりを確認し、注文するための公開APIです。
このリポジトリで仕様書と利用ガイドを管理します。契約の正本は[openapi.yaml](openapi.yaml)です。

**現在は開発中で、まだ提供開始していません。** 仕様はv0.5.0の草案で、以下のURLと実行例は提供開始後の利用を想定しています。
予定の接続先は`https://api.koroai.supply`です。`kr_test_`はテスト注文、`kr_live_`は実注文になります。

## できること

| 操作 | エンドポイント |
| --- | --- |
| 商品を名前で検索 | `GET /v1/products/search?q=...` |
| JANで商品を照会 | `GET /v1/products/{jan}` |
| 見積もりを作成・照会 | `POST /v1/quotes`、`GET /v1/quotes/{id}` |
| 注文を作成 | `POST /v1/orders` |
| クイック注文の商品を照会・注文 | `GET /v1/quick-order`、`POST /v1/quick-order` |
| 注文の一覧・状況確認 | `GET /v1/orders`、`GET /v1/orders/{id}` |
| 取消可能な注文を取消 | `DELETE /v1/orders/{id}` |
| 未取扱商品の調査を依頼・確認 | `POST /v1/product-requests`、`GET /v1/product-requests`、`GET /v1/product-requests/{id}` |

住所・カード・購入上限の変更、入会、管理操作はWebで行います。
ESP32・Linux・Arduinoから使う場合は、[ハードウェア向けSDK](https://github.com/koroai-supply/hw)を参照してください。

## はじめる

### CLIの端末ログイン（追加予定）

独立したOSSの [`ks` CLI](https://github.com/koroai-supply/cli) は、[OAuth 2.0 Device Authorization Grant（RFC 8628）](https://www.rfc-editor.org/rfc/rfc8628.html)でログインする予定です。公開APIキーをCLIへコピーする必要はありません。この節と `openapi.yaml` のOAuthエンドポイントは**契約草案で、現時点では利用できません**。

1. `ks login` が本番用の端末コードを取得し、Webの承認URLとコードを表示します。CLIにテスト用の操作はありません。
2. 利用者はWebにログインし、端末名、ホーム、本番用であること、許可する操作を確かめて承認します。URLを開いただけでは承認しません。
3. CLIは指定された間隔以上でトークンを照会します。端末・ホームに結び付いた短命のaccess tokenを公開APIへ送り、回転するrefresh tokenをOSの資格情報ストアに保存します。`ks logout`でその端末を失効できます。

CLI向けOAuthと従来のAPIキーは併存します。OAuthでもホームごとのAPI利用設定、月間購入予算、冪等性を同じ正本で判定します。認可サーバーは `https://koroai.supply`、公開APIは `https://api.koroai.supply` を予定しています。実装と配備が揃うまでは、以下のAPIキー例を使ってください。

Webでホームの月間購入上限を設定し、API利用を有効にすると、契約者がAPIキーを確認・更新できます。
ホーム共通の本番用とテスト用を各1本、自動で用意します。キーの追加・削除や、キーごとの権限・購入上限の設定はありません。

最初は`kr_test_`から始まるテスト用キーで確認します。

```sh
export KOROAI_API_KEY='自分のテスト用キー'
```

すべてのリクエストで`Authorization: Bearer <key>`を送ります。
モードはキーで決まり、本文やqueryでは上書きできません。キーの末尾は43文字のbase64url文字列です。

| キー | 動作 |
| --- | --- |
| `kr_test_` | 検証用の注文記録を作成。実注文・決済・在庫の引当は行わない |
| `kr_live_` | ホームの月間購入予算と超過時の動作に従って即時に注文 |

キーは暗号化して保存し、購入上限の設定後にWebで再表示できます。更新すると古い値はすぐ無効になります。
ホームの購入上限を解除すると、キーの表示・更新とAPIの利用を停止します。
キーは秘密として扱い、ソースコード・ログ・会話へ貼り付けないでください。

## 商品を確認する

JANが分かる場合は完全一致で照会します。

```sh
curl 'https://api.koroai.supply/v1/products/4901234567894' \
  -H "Authorization: Bearer $KOROAI_API_KEY"
```

商品名で探す場合は検索語を指定します。検索語なしの商品一覧は提供しません。

```sh
curl --get 'https://api.koroai.supply/v1/products/search' \
  -H "Authorization: Bearer $KOROAI_API_KEY" \
  --data-urlencode 'q=お茶'
```

検索は商品名の部分一致で、1ページ最大50件です。続きがある場合は応答の`next_cursor`を`cursor`に指定します。
例のJANは説明用で、取扱商品を保証しません。

JANは8桁または13桁の文字列です。先頭ゼロを保持し、チェックディジットも検証します。
数量の`qty`は1〜10パッケージ、金額は税込の日本円です。リクエスト単位の`max_price`はありません。

## 注文する

1回の新しい注文に1つの`Idempotency-Key`を割り当てます。次の例は、あるボタンの注文連番をキーにしたものです。

```sh
export ORDER_KEY='kitchen-button-000001'

curl 'https://api.koroai.supply/v1/orders' \
  -H "Authorization: Bearer $KOROAI_API_KEY" \
  -H 'Content-Type: application/json' \
  -H "Idempotency-Key: $ORDER_KEY" \
  -d '{"jan":"4901234567894","qty":1}'
```

正常な作成はHTTP 201で、`data`に注文を返します。受付は決済・配送の完了を意味しません。
注文前に金額を確認したい場合は、同じJANと数量を`POST /v1/quotes`へ送って見積もりを作成できます。
未取扱商品は注文できません。`POST /v1/product-requests`へJANを送り、調査・取り扱いを依頼します。調査や運営の承認は購入の承認ではありません。

### クイック注文

ホームに1つ設定したクイック注文の商品を、商品を指定せずに1パッケージ注文できます。ボタンやCLIのように、押す・呼ぶだけで決まった商品を注文したい場合に使います。
商品の設定・変更・解除はアプリとMCPで行います。APIからは変更できません。

```sh
curl 'https://api.koroai.supply/v1/quick-order' \
  -H "Authorization: Bearer $KOROAI_API_KEY"
```

設定中の商品を`{"data":{"product":{...}}}`で返します。未設定の場合は`product`が`null`です。

```sh
export ORDER_KEY='kitchen-button-000002'

curl -X POST 'https://api.koroai.supply/v1/quick-order' \
  -H "Authorization: Bearer $KOROAI_API_KEY" \
  -H "Idempotency-Key: $ORDER_KEY"
```

本文は省けます（ボタンなどから送りやすいように）。送る場合は`Content-Type: application/json`で空のオブジェクト`{}`だけを受け付け、商品・数量は指定できません。数量は常に1です。
正常な作成はHTTP 201で、`POST /v1/orders`と同じ注文を返します。キーの種類、購入予算、冪等キーの扱いも同じです。
商品が設定されていない場合はHTTP 422の`quick_order_not_set`で、注文しません。

## 冪等性と再送

`Idempotency-Key`はAPI上は任意ですが、注文するクライアントでは指定してください。
使える文字は英数字と`.`、`_`、`:`、`-`、長さは1〜128文字です。同じキー・操作の同じ内容は同じ結果を返し、異なる内容はHTTP 409になります。

新しい注文と、同じ注文の再送を分けて扱います。

1. 新しい注文を受け付けたときだけ、端末IDと注文連番などからキーを決めます。
2. キー・JAN・数量を保存してからリクエストを送ります。
3. タイムアウトや応答喪失では、新しいキーを作らず、同じキー・同じ本文で同じPOSTを再送します。
4. 結果が未確定の間は、その注文を新しい注文として作り直しません。必要に応じて注文一覧でも結果を確認します。

上の実行例を再送する場合も、`ORDER_KEY`と本文を変えません。次の独立した注文だけが新しい連番になります。再送のたびに時刻や乱数から別のキーを作らないでください。
キーの更新後も、同じホーム・モードの注文履歴と再送の照合は引き継ぎます。

同じAPIキー・JANの短時間の入力も1件へまとめます。その窓内に数量の異なる入力を重ねた場合は409とし、意図せず数量を増やしません。この補助的な照合だけに頼らず、クライアントでも注文ごとのキーを保持してください。

429では`Retry-After`に指定された秒数以上待ちます。5xxや通信障害で再送する場合も、同じ注文キーを保ちます。

## 状況を確認する

注文一覧は次のように取得します。

```sh
curl 'https://api.koroai.supply/v1/orders' \
  -H "Authorization: Bearer $KOROAI_API_KEY"
```

成功は`{"data": ...}`、一覧は`{"data": [...], "next_cursor": null}`です。
1ページ最大50件で、`next_cursor`がある場合は次のリクエストの`cursor`に指定します。
特定の注文は、返された注文IDを`GET /v1/orders/{id}`へ指定して照会します。

注文と取り扱いリクエストは、認可されたホームかつキーと同じモードの記録を照会できます。
見積もりの照会と注文の取消は、そのキーで作成した記録に限ります。
注文は請求・手配・配送の状態を分けて返し、住所・カード・連絡先・内部のエラー本文は返しません。取り扱いリクエストは調査中から確認できます。

## 購入上限

Web・アプリと同じホームの月間購入予算を使います。月の境界は日本時間で、並行した注文もホーム単位で照合します。
テスト用キーでも予算を確認しますが、実注文の使用額には加算しません。
APIから購入上限を変更することはできません。

## エラーと入力の制約

エラーは`{"error":"固定コード"}`で返します。

| コード | 対応 |
| --- | --- |
| `not_found` | キーと対象を確認。無効なキーと参照できない対象で共通の応答 |
| `invalid_request`、`unsupported_media_type`、`request_too_large` | 入力・Content-Type・本文サイズを確認 |
| `forbidden`、`purchase_limit_exceeded` | Webで利用条件やホームの購入上限を確認 |
| `product_not_available` | 商品を確認し、必要なら取り扱いリクエストへ進む |
| `quick_order_not_set` | アプリかMCPでホームのクイック注文の商品を設定する |
| `idempotency_conflict` | 保存したキーと元の本文を確認。別のキーで自動的に注文し直さない |
| `quote_expired`、`order_not_cancelable` | 見積もり・注文の現在の状態を確認 |
| `rate_limited` | `Retry-After`秒以上待機 |
| `service_unavailable`、`upstream_unavailable`、`upstream_timeout` | 結果未確定の注文を保持し、同じキーで再送 |

リクエスト本文は16KiBまでです。未定義項目・重複query・`q`と`jan`の同時指定は拒否します。
すべての応答に`Cache-Control: no-store`を指定します。

## 開発とバージョン

仕様書はOpenAPI 3.1で、`openapi.yaml`をJSON表記のYAMLとして管理します。
GitHub ActionsでOpenAPIの妥当性と差分の空白エラーを確認し、`required-ci`で判定します。仕様検証の成功は、実サービスへの疎通・注文の確認を意味しません。

開発中の0.xは草案で、提供開始前に変更できます。提供開始後の破壊的変更は新しいメジャーパスで行います。
公開後の廃止は少なくとも180日前にこのリポジトリで予告します。

## ライセンス

このリポジトリの仕様書・利用ガイド・サンプルは **[MIT License](LICENSE)** です。
APIサービス自体の利用条件や、実注文の支払い・配送条件を定めるものではありません。
