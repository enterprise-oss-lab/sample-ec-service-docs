# sample-ec-service リリースノート

## 2026-10-31 — PR #67: Debeziumで注文確定OutboxをKafkaへ配信

[PR #67](https://github.com/enterprise-oss-lab/sample-ec-service/pull/67) — mainへのマージコミット `7fa6cf15d7fbf8363bd3728ccdfb74d9e9a6c429`（2026-10-30 22:37:12 UTC / 2026-10-31 JST）。構成図はこのコミットの実コードに固定。

- Order PostgreSQLの `wal_level=logical` を有効化し、Debezium Connect 3.3.1.Finalと冪等なconnector登録サービスを追加。注文確定・Outbox INSERTの同一トランザクションは既存実装を利用する。
- `pgoutput` が `public.outbox_events` のcommit済みWALを読み、Outbox Event Routerで `order.confirmed` へ配信。初期snapshotあり、at-least-once。キーは注文IDではなく `event_id` のUUID文字列。値は展開したJSON payloadで、Connect envelopeを含めない。
- Kafkaイベントtopicとcompact設定のConnect内部topicを明示作成。設定・offset・状態はKafka volume、replication slot/publicationはDB volumeに永続化する。REST APIは `127.0.0.1:8083` に限定。
- 既存の在庫引当Saga（Order→requests→Inventory→results→Order）は継続。図では往復を集約して表示。OrderからInventoryの商品価格取得HTTP、ZITADEL認証・OTLP監視・任意analytics構成は省略。Flinkの重複排除・CDC後段処理は後続Issue #59。
- CIと専用Composeスモークを追加。commit済みpayload、rollback/未commit行の非配信、停止中INSERTの再起動後配信、初期化再実行を検証する構成。既存DBは必要に応じてOutbox migrationを適用し、connector長期停止時はWAL容量を監視する。

構成図: [PR #67時点](architecture/2026-10-31-pr67.html) / [最新版](index.html) / [JSON仕様](architecture/2026-10-31-pr67.json)。日本語本文、固定Viewer UIとHTML langは英語。

検証: architecture形式、archify validate → deliver、showcase 9/9・エラー0・警告0、コミット固定のソース参照10件を検証。1440×900・1600×1000・1920×1080・2048×1320でoverflowなし。最小ライト／最大ダークの画像を目視確認（visual_review: passed、correction_rounds: 2）。ドキュメントのみの更新のためアプリ／Docker smokeは未実行。

仕様SHA-256: `eac8f7977b1eca031bd56421ba5334907a6909e9c4978ad246b547f72b5e688d`（5718 bytes）。HTML SHA-256: `901be9ee8e2a905cdc8d3ef04be5330b7bbb4792dcd48ef223c01e8d77e85c28`（710981 bytes）。

## 2026-10-31 — PR #65: Iceberg REST Catalog・Object Storage・Trinoを構築

[PR #65](https://github.com/enterprise-oss-lab/sample-ec-service/pull/65) — mainへのマージコミット `93a1713088e905538da8859e7bd856fe58c019ae`（2026-10-30 22:34:55 UTC / 2026-10-31 JST）。構成図はこのコミットの実コードに固定。

`compose.analytics.yaml` にローカル分析基盤を追加。既存ECのRustFS 1.0.0-rc.4を共有し、商品画像用 `products` と分離した `analytics` bucketを使用する。

- Iceberg REST Catalog 1.8.1は `s3://analytics/` をwarehouseとし、SQLite（WAL）のcatalog DBをnamed volumeへ永続化。S3FileIOでRustFSへ接続する。
- Trino 483はIceberg REST connectorとnative S3でCatalog・RustFSへ接続し、Parquetを使用。Order／Inventory PostgreSQLへの接続は追加しない。
- bucket作成とcatalog volume権限設定をone-shotサービスで実施。healthcheckを待ってCatalog→Trinoを起動し、namespace-initで `bronze` / `gold` を冪等に作成する。
- ホスト公開はCatalogが `127.0.0.1:18181`、Trinoが `127.0.0.1:28080`（変更可能）。固定資格情報・認証なしのローカルPoCであり、本番用ではない。
- `scripts/analytics-smoke.sh` に書込・読取（3行・合計60）、再起動後の読取、停止、分析データの削除手順を追加。通常初期化ではテストテーブルを作らない。
- Flink connector・ジョブ・CDC連携は後続Issue #59の対象で、今回の実装には含まれない。

構成図: [PR #65時点](architecture/2026-10-31-pr65.html) / [最新版](index.html) / [JSON仕様](architecture/2026-10-31-pr65.json)。既存ECは集約表示し、認証・監視・在庫引当連携とInventoryの商品画像アクセスは補足カードに記載。本文は日本語、固定Viewer UIとHTMLのlang属性は英語。

検証: architecture形式、archify validate → deliver、showcase 9/9・エラー0・警告0、コミット固定のソース参照8件を検証。1440×900、1600×1000、1920×1080、2048×1320でoverflowなし。最小・最大サイズのライト／ダーク画像を目視確認（visual_review: passed、correction_rounds: 1）。ドキュメントのみの更新のため、Docker smoke／アプリケーションテストは未実行。

仕様SHA-256: `cf849cc467400d43e6d4334b4d0a355a5f7cbee67d02487f82f25c58b438ef38`（4842 bytes）。HTML SHA-256: `c695af093561c653a0d9013d7f0d1d615fb129601425472df899392700cb7f94`（704347 bytes）。

## 2026-10-17 — PR #66: pre-commitでLinter・Formatterを実行

[PR #66](https://github.com/enterprise-oss-lab/sample-ec-service/pull/66) — mainへのマージコミット `b022f5eb3fa627df4ee6ace7dd8cec02a6cb611d`（2026-10-17 08:00:04 UTC）。構成図はこのコミットの実コードを根拠としている。

コミット前に変更対象のコンポーネントで整形・lintを実行するlocal hookを追加。変更は `.pre-commit-config.yaml` とREADMEのみで、API・DB・サービス間接続に変更はない。

- Inventory: ステージ済みGoファイルをgofmtで整形し、Go変更時にサービス全体へ `golangci-lint run ./...` を実行。
- Order: 対象Pythonファイルへ `ruff format` / `ruff check` を実行。
- Admin / Storefront: 対象ファイルへPrettier、JS/TSファイルへESLintを実行。各frontendのディレクトリで既存設定を使用。
- detect-secretsからAdminの生成済みMSW workerを除外。固定チェックサムの誤検出を回避する。
- READMEにツール準備、`pre-commit install`、初回全件実行、整形後の再ステージ手順を追記。CIの品質チェックも継続する。

構成図: [PR #66時点](architecture/2026-10-17-pr66.html) / [最新版](index.html) / [JSON仕様](architecture/2026-10-17-pr66.json)。Composeのローカル構成を表示し、認証基盤は集約。Kafkaの結果経路・JWT/JWKS・OTLP送出は補足カードに記載。本文は日本語、固定Viewer UIとHTMLのlang属性は英語。

検証: architecture形式、archify validate → deliver、showcase 9/9・エラー0・警告0、コミット固定のソース参照10件を検証。4サイズ（1440×900、1600×1000、1920×1080、2048×1320）でoverflowなし。最小・最大サイズのライト／ダーク画像を目視確認（visual_review: passed、correction_rounds: 0）。ドキュメントのみの更新で、アプリケーションテストは未実行。

仕様SHA-256: `5b87767d190e2e94f544022289f9d221e7e05708f23db6a56e4ebaefb5919c6c`（6122 bytes）。HTML SHA-256: `ad7a95289b5269bee2c2a6eee0ec1fbdf4c06b8345bd4ea0d6775d1d6ceafa66`（709635 bytes）。

## 2026-10-17 — PR #55: Linter・FormatterとCI品質ゲートを導入

[PR #55](https://github.com/enterprise-oss-lab/sample-ec-service/pull/55) — mainへのマージコミット `07149685a08c66de63d7f775373efcf517e2ebad`（2026-10-17 07:30:21 UTC）。構成図はこのコミットの実コードを根拠としている。

4アプリのCIにlint・formatチェックを追加し、ユニットテスト前に実行する。サービス間接続・API・DBスキーマの構成変更はない。

- Storefront / Admin: ESLintに `eslint-config-prettier` を組み合わせ、Prettierの `format` / `format:check` を追加。生成済みMSW workerはPrettier対象外。CIのpnpmを10.32.1に固定。
- Inventory: golangci-lint v2.6.1（errcheck / govet / ineffassign / staticcheck / unused）とgofmtを追加。Close・Rollbackの戻り値を明示的に破棄する修正を含む。
- Order: Ruff 0.14.11でlint・import整列・formatを検査。Python 3.10を対象とする。
- ルートMakefileに `make lint` / `make format-check` / `make format` とアプリ別ターゲットを追加。既存コードの整形、README、lockfile、`.git-blame-ignore-revs` を更新。

構成図: [PR #55時点](architecture/2026-10-17-pr55.html) / [最新版](index.html) / [JSON仕様](architecture/2026-10-17-pr55.json)。Composeのローカル構成を表示し、認証基盤は集約。Kafkaの結果経路・JWT/JWKS・OTLP送出は補足カードに記載。本文は日本語、固定Viewer UIとHTMLのlang属性は英語。

検証: architecture形式、archify validate → deliver、showcase 9/9・エラー0・警告0、コミット固定のソース参照10件を検証。4サイズ（1440×900、1600×1000、1920×1080、2048×1320）でoverflowなし。最小・最大サイズのライト／ダーク画像を目視確認（visual_review: passed、correction_rounds: 0）。ドキュメントのみの更新で、アプリケーションテストは未実行。

仕様SHA-256: `496292a6101907d4721346be46332a99126fbe5a2441d6a9e7ef0da41daaec6b`（6114 bytes）。HTML SHA-256: `eaeb5a1233277d0537b913614a5bf874efe8e01e82d27d6937e32437ea75cb44`（709647 bytes）。

## 2026-10-13 — PR #53: 注文時点の単価を明細へ保存

[PR #53](https://github.com/enterprise-oss-lab/sample-ec-service/pull/53) — mainへのマージコミット `8f1bc203b3abd3da9fcc90bb2bc6de02b177e233`（2026-10-13 06:46:37 UTC）。構成図はこのコミットの実コードを根拠としている。

注文作成時にOrderがInventoryの `GET /products/{inventory_id}` をHTTPで呼び出し、送信された期待価格と照合する。一致した取得価格を `order_items.unit_price_at_order` に保存し、注文APIの明細レスポンスにも返す。照合は注文保存・Kafkaへの在庫引当要求発行より前に行われる。

- **API互換性変更**: `POST /orders` の各明細に `expected_unit_price` が必須。Storefront・モック・k6の送信形式も更新。
- 価格不一致は409、商品が存在しない場合は422、商品APIへの通信失敗・エラーレスポンス・不正価格は502。HTTPクライアントのタイムアウトは5秒。
- マイグレーション `003_add_unit_price_at_order.sql` は非負・NOT NULLの整数列を追加。既存明細は過去価格が不明なため0で埋める。0は実際の購入価格の復元ではない。
- Composeに `INVENTORY_BASE_URL=http://inventory:8080` とInventory起動依存を追加。既存のPostgreSQLボリュームには初期化SQLが再実行されないため、既存環境では003の適用が必要。

構成図: [PR #53時点](architecture/2026-10-13-pr53.html) / [最新版](index.html) / [JSON仕様](architecture/2026-10-13-pr53.json)。ローカルCompose構成を表示。認証基盤は集約し、Kafkaの結果経路・JWT/JWKS・OTLP送出は補足カードに記載。Storefrontの商品API参照とInventoryのS3操作はノード内に記載。本文は日本語、固定Viewer UIとHTMLのlang属性は英語。

検証: archify validate → deliver、showcase 9/9・エラー0・警告0、コード参照10件を検証。4サイズ（1440×900、1600×1000、1920×1080、2048×1320）でoverflowなし。ライト・ダーク画像を目視確認（visual_review: passed、correction_rounds: 2）。ドキュメントのみの更新で、アプリケーションテストは未実行。

仕様SHA-256: `056f667e5842fd28f5fa54e1a2175761f031fe78364175ce0fcc9242fbfcd211`（6155 bytes）。HTML SHA-256: `7bcc5d22050a1caeeb992040938d97015c312c8b55ee4808208eafea1a37d452`（709690 bytes）。

## 2026-10-11 — PR #52: 実ZITADELによる認証・認可E2Eと利用手順

[PR #52](https://github.com/enterprise-oss-lab/sample-ec-service/pull/52) — マージコミット: `be8f8674dd9172525ca72ea6f0dcad4982fd8624`（マージ日時: 2026-10-10 22:57:59 UTC / 2026-10-11 JST）。Webhook記載のマージ先は `auth/frontend-authentication`。図はこのコミットに固定し、ブランチの移動による混在を避けている。

既存の認証構成に対し、`scripts/auth-e2e.mjs` と `make test-auth-e2e` を追加。Compose上の実ZITADELとPlaywrightで購入者2名・管理者のログインを行い、注文作成と自身の履歴参照、他者注文への404、購入者の管理API利用への403、匿名アクセスへの401、管理者の商品作成・在庫調整・画像アップロード、購入者のAdmin画面拒否を検証する。

ルート・Storefront・Admin・Inventory・Order・k6のREADMEに、環境変数の準備、起動、認証付きAPIやE2Eの実行手順を追記・修正した。本PRの差分はテストと手順書であり、ZITADEL・JWT認可そのものの導入は先行変更による。

構成図: [PR #52時点](architecture/2026-10-11-pr52.html) / [最新版](index.html)。React SPA、ZITADEL（Traefik・Login・専用PostgreSQLを集約）、Inventory/Order、各DB、Redis、RustFS、Kafkaを表示。JWKS検証とOTLP送出は補足カードに記載。ローカル開発環境の構成であり、本番配備図ではない。本文は日本語、固定Viewer UIとHTMLのlang属性は英語。

検証: archify showcase 9/9、エラー・警告0、コミット固定のソース参照13件を検証。4画面サイズ（1440×900〜2048×1320）の収まりとライト・ダーク表示を確認。今回のドキュメント更新ではアプリケーションE2Eは実行していない。

## 2026-09-14 — PR #45: スタックPRの中間マージではMultica通知を抑制

https://github.com/enterprise-oss-lab/sample-ec-service/pull/45

`notify-multica-on-merge.yaml` の発火条件に、スタックPR（`pull_request.stack`）の位置チェックを追加した。スタックの途中PRがマージされた際に中間状態で構成図・リリースノート更新のautopilotが発火してしまうのを防ぎ、スタックの最後のPRがマージされたとき（または非スタックPRの通常マージ時）のみ通知するようにした。アプリケーションコード・アーキテクチャへの変更はなし。

- `.github/workflows/notify-multica-on-merge.yaml` の `if` 条件に `github.event.pull_request.stack == null || github.event.pull_request.stack.position == github.event.pull_request.stack.size` を追加

構成図: [architecture/2026-09-14-pr45.html](architecture/2026-09-14-pr45.html)（最新版は [index.html](index.html)）。本PR自体はCI設定のみの変更のため、現在の main (commit `e812bbb`) が持つ全体構成は前回更新時点 (PR #38: media_assets の HTTPハンドラ配線、`3d564b1`) から変わっておらず、この構成図はその現状を再確認する内容になっている。

## 2026-09-11 — PR #37: media_assets usecase層 (RegisterPending/CleanupExpired)

https://github.com/enterprise-oss-lab/sample-ec-service/pull/37

孤立アップロード（商品保存に紐付かないまま残る画像）を削除できるようにするため、media_assets の生存管理ユースケースを追加した。UploadImage 成功後に pending 登録し、商品保存トランザクション内で confirmed に遷移させ、確定しなかった pending は期限切れ後に削除する。

- `UploadImage` 成功後に `RegisterPending` を呼び、`media_assets` へ `status=pending, expires_at=now+TTL` で登録
- 商品作成/更新と同一 DB トランザクションで `Confirm`（pending→confirmed, product_id 紐付け）を実行し、画像保存と商品保存の2相コミット問題を回避
- `CleanupExpired` は期限切れ pending を1行ずつ `SELECT FOR UPDATE` でロックし、RustFS から削除→DB 行削除。ロック後に confirmed 等へ遷移済みの行は deleted/failed いずれにも計上せずスキップ

構成図: [architecture/2026-09-11-pr37.html](architecture/2026-09-11-pr37.html)（最新版は [index.html](index.html)）。このPR自体の差分に加え、本コミット時点の main（同日にマージ済みの PR #38: HTTPハンドラ配線・`POST /admin/media-assets/cleanup` を含む）を反映した最新の全体構成を示している。

## 2026-09-11 — PR #36: media_asset repository + product tx内confirm連携

https://github.com/enterprise-oss-lab/sample-ec-service/pull/36

PR #34 で追加した `media_assets` のドメイン層・マイグレーションに対し、Postgres リポジトリ実装を追加。あわせて、画像アップロードと商品保存が別トランザクションになることで生じる2相コミット問題を避けるため、商品の作成/更新トランザクション内で pending 画像を confirmed に遷移させる連携を実装した。

- `internal/repository/postgres/media_asset.go` に `mediaAssetRepository` / `txMediaAssetRepository` を追加。`RunInTx` は `SELECT FOR UPDATE` でロックを取得し並行更新から保護
- `txMediaAssetRepository.Confirm` を追加し、`storage_key` に紐づく pending 行を `confirmed` に遷移して `product_id` を紐付け（対象が見つからない場合は `ErrMediaAssetNotConfirmable`）
- `productRepository.Create` / `Update` から、商品保存と同一の DB トランザクション内で `Confirm` を呼び出すよう配線（`product.go`）
- usecase 層（`RegisterPending`/`CleanupExpired`）と HTTP 配線は後続の PR #37・PR #38 で完成している

構成図: [architecture/2026-09-11-pr36.html](architecture/2026-09-11-pr36.html)（最新版は [index.html](index.html)）。このPR自体の差分に加え、本コミット時点の main（同日にマージ済みの PR #37: usecase層、PR #38: HTTPハンドラ配線を含む）を反映した最新の全体構成を示している。

## 2026-09-11 — PR #34: media_assets のマイグレーション/config/domain層を追加

https://github.com/enterprise-oss-lab/sample-ec-service/pull/34

RustFS (S3互換ストレージ) にアップロードした商品画像の生存管理テーブル `media_assets` を新設。pending/confirmed の2状態と `expires_at` を持たせ、商品に紐付かないまま放置されたアップロードを孤立データとして扱えるようにする土台（マイグレーション・config・domain層）を追加した。リポジトリ実装・トランザクション内confirm連携・usecase層・HTTP配線は後続の PR #36〜#38 で完成している。

- `services/inventory/db/migrations/005_create_media_assets.sql` で `media_assets` テーブルを新設
- `config.go` に `MediaAssetPendingTTL` を追加
- `domain/media_asset.go` に `MediaAsset` 型と `MediaAssetRepository` インターフェースを追加
- `domain/image.go` / `adapter/storage/s3.go` に画像アップロード関連の型・実装を追加

構成図: [architecture/2026-09-11-pr34.html](architecture/2026-09-11-pr34.html)（最新版は [index.html](index.html)）。このPR自体の差分に加え、本コミット時点の main（同日にマージ済みの PR #36: リポジトリ実装+tx内confirm連携、PR #37: usecase層 RegisterPending/CleanupExpired、PR #38: HTTPハンドラ配線を含む）を反映した最新の全体構成を示している。

## 2026-09-11 — PR #38: media_assets の HTTPハンドラ・main.go配線 (最終Task)

https://github.com/enterprise-oss-lab/sample-ec-service/pull/38

`MediaAssetHandler` を新設して `POST /admin/media-assets/cleanup` を公開し、usecase/repository 層 (Task1〜3) で実装済みだった孤立アップロード削除運用をHTTP層まで配線して完成させた。あわせて `CreateProduct`/`UpdateProduct` に `ErrMediaAssetNotConfirmable` の 422 ハンドリングを、既存の `ErrInsufficientStock`/`ErrInvalidQuantity` と同じ分岐パターンで追加した。

- `services/inventory/internal/adapter/http/media_asset.go` を新規追加し `MediaAssetHandler.CleanupExpired` を実装
- `services/inventory/main.go` に `httphandler.NewMediaAssetHandler(mediaAssetUC).RegisterRoutes(r)` を配線
- `CreateProduct`/`UpdateProduct` (`inventory.go`) に `ErrMediaAssetNotConfirmable` → 422 の分岐を追加

構成図: [architecture/2026-09-11-pr38.html](architecture/2026-09-11-pr38.html)（最新版は [index.html](index.html)）。sample-ec-service の現在の main (commit `3d564b1`) を根拠に、storefront/admin (React) → inventory (Go+Gin) / order (FastAPI) → 各PostgreSQL、Redis cache-aside、RustFS (S3互換) によるメディアアセット管理、Kafka 経由の在庫引当saga、otel-lgtm への OTLP 送出までを含む全体構成を反映している。

## 2026-09-11 — PR #44: main マージ時のみ Multica に通知するよう CI を修正

https://github.com/enterprise-oss-lab/sample-ec-service/pull/44

Multica 通知ワークフローが `pull_request: closed` のあらゆるブランチで発火していたのを、`main` へのマージのみに限定した。アプリケーションコード・アーキテクチャへの変更はなし。

- `.github/workflows/notify-multica-on-merge.yaml` に `branches: [main]` を追加

構成図: [architecture/2026-09-11-pr44.html](architecture/2026-09-11-pr44.html)（最新版は [index.html](index.html)）。前回の構成図更新 (PR #20) 以降に main へ入った Inventory Service の Redis キャッシュ層（PR #40〜#42: 在庫は TTL 1秒、商品カタログは TTL 60秒の cache-aside）を含む、現在の main の構成を反映している。

## 2026-09-08 — PR #20: カタログ属性を products に分離し、storefront を商品詳細対応

https://github.com/enterprise-oss-lab/sample-ec-service/pull/20

在庫数 (`inventories.count`) は毎秒変わるが、カタログ属性（名前・説明・価格・画像）は日〜月単位でしか変わらない。両者が同じテーブル・同じレスポンスに混ざっていると可変な1列のせいでレスポンス全体をキャッシュできなくなるため、カタログ属性を `products` テーブルに分離した。

- `products` テーブルを新設し、`inventories` と 1:1 対応する seed を追加
- inventory サービスに `GET /products`, `GET /products/:id` を追加
- `inventories.name` を削除し、`GET /inventories` から `name` を除去（破壊的変更）
- storefront に商品詳細ページ (`/products/:id`) を追加し、カタログ/在庫を別クエリ・別 staleTime（カタログ1時間 / 在庫0）に分離

構成図: [architecture/2026-09-08-pr20.html](architecture/2026-09-08-pr20.html)（最新版は [index.html](index.html)）
