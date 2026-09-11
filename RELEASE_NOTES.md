# sample-ec-service リリースノート

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
