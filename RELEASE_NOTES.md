# sample-ec-service リリースノート

## 2026-09-08 — PR #20: カタログ属性を products に分離し、storefront を商品詳細対応

https://github.com/enterprise-oss-lab/sample-ec-service/pull/20

在庫数 (`inventories.count`) は毎秒変わるが、カタログ属性（名前・説明・価格・画像）は日〜月単位でしか変わらない。両者が同じテーブル・同じレスポンスに混ざっていると可変な1列のせいでレスポンス全体をキャッシュできなくなるため、カタログ属性を `products` テーブルに分離した。

- `products` テーブルを新設し、`inventories` と 1:1 対応する seed を追加
- inventory サービスに `GET /products`, `GET /products/:id` を追加
- `inventories.name` を削除し、`GET /inventories` から `name` を除去（破壊的変更）
- storefront に商品詳細ページ (`/products/:id`) を追加し、カタログ/在庫を別クエリ・別 staleTime（カタログ1時間 / 在庫0）に分離

構成図: [architecture/2026-09-08-pr20.html](architecture/2026-09-08-pr20.html)（最新版は [index.html](index.html)）
