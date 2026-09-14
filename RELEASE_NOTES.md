# sample-ec-service リリースノート

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
