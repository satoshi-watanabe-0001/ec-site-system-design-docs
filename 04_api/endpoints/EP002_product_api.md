---
title: "商品カテゴリAPI仕様書"
version: "1.0.0"
last_updated: "2025-12-14"
status: "Approved"
template_type: "Technical Specification"
---

# 商品カテゴリAPI仕様書

> 商品カテゴリの一覧取得、詳細取得、おすすめ商品取得機能を提供するREST APIの仕様書。

**API Version**: v1.0.0  
**Document Version**: 1.0.0  
**Last Updated**: 2025-12-14  
**Status**: Approved

---

## 目次

1. [概要](#概要)
2. [認証](#認証)
3. [エンドポイント](#エンドポイント)
4. [データモデル](#データモデル)
5. [エラーコード](#エラーコード)
6. [レート制限](#レート制限)
7. [バージョニング](#バージョニング)
8. [変更履歴](#変更履歴)

---

## 概要

### APIの目的

商品カテゴリAPIは、ECサイトの商品カテゴリ情報を提供します。カテゴリ一覧の取得、カテゴリ詳細（商品リスト含む）の取得、おすすめ商品の取得機能を提供します。

### ベースURL

- **Production**: `https://api.ec-site.example.com/api/v1`
- **Staging**: `https://staging-api.ec-site.example.com/api/v1`
- **Development**: `http://localhost:8080/api/v1`

### プロトコル

- **Protocol**: HTTPS (TLS 1.2+)
- **Content-Type**: `application/json`
- **Character Encoding**: UTF-8

---

## 認証

### 認証方式

商品カテゴリAPIは**公開エンドポイント**として提供されます。認証なしでアクセス可能です。

---

## エンドポイント

### 1. カテゴリ一覧取得

**エンドポイント**: `GET /api/v1/products/categories`

**説明**: アクティブな商品カテゴリの一覧を取得します。表示順序でソートされた結果を返却します。

**認証**: 不要（公開エンドポイント）

#### リクエスト例

```bash
curl -X GET "http://localhost:8080/api/v1/products/categories" \
  -H "Content-Type: application/json"
```

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "カテゴリ一覧を取得しました",
  "data": [
    {
      "categoryCode": "iphone",
      "displayName": "iPhone",
      "heroImageUrl": "/images/categories/iphone.jpg",
      "leadText": "最新のiPhoneシリーズをご覧ください",
      "productCount": 15
    },
    {
      "categoryCode": "android",
      "displayName": "Android",
      "heroImageUrl": "/images/categories/android.jpg",
      "leadText": "多彩なAndroidスマートフォン",
      "productCount": 20
    },
    {
      "categoryCode": "docomo-certified",
      "displayName": "ドコモ認定リユース品",
      "heroImageUrl": "/images/categories/refurbished.jpg",
      "leadText": "30日以内無料交換可能",
      "productCount": 10
    },
    {
      "categoryCode": "accessories",
      "displayName": "アクセサリ",
      "heroImageUrl": "/images/categories/accessories.jpg",
      "leadText": "スマートフォン関連アクセサリ",
      "productCount": 50
    }
  ],
  "timestamp": "2025-12-14T10:00:00.000Z",
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

### 2. カテゴリ詳細取得

**エンドポイント**: `GET /api/v1/products/categories/{categoryCode}`

**説明**: 指定されたカテゴリの詳細情報と、そのカテゴリに属する商品リストを取得します。ページネーション、キーワード検索、ソート機能をサポートします。

**認証**: 不要（公開エンドポイント）

#### パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|----------|-----|------|------|
| categoryCode | string | Yes | カテゴリコード（例: iphone, android） |

#### クエリパラメータ

| パラメータ | 型 | 必須 | デフォルト | 説明 |
|----------|-----|------|-----------|------|
| keyword | string | No | - | 商品名検索キーワード（最大100文字） |
| page | integer | No | 0 | ページ番号（0始まり） |
| size | integer | No | 20 | 1ページあたりの件数（1〜100） |
| sort | string | No | name | ソートフィールド（name, price, createdAt） |
| order | string | No | asc | ソート順序（asc, desc） |

#### リクエスト例

```bash
curl -X GET "http://localhost:8080/api/v1/products/categories/iphone?page=0&size=10&sort=price&order=desc" \
  -H "Content-Type: application/json"
```

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "カテゴリ詳細を取得しました",
  "data": {
    "category": {
      "categoryCode": "iphone",
      "displayName": "iPhone",
      "heroImageUrl": "/images/categories/iphone.jpg",
      "leadText": "最新のiPhoneシリーズをご覧ください"
    },
    "products": [
      {
        "productId": 1,
        "productName": "iPhone 15 Pro Max",
        "description": "最新のiPhone 15 Pro Max。A17 Proチップ搭載。",
        "price": 189800,
        "manufacturer": "Apple",
        "modelName": "iPhone 15 Pro Max",
        "storageCapacity": "256GB",
        "colorCode": "#1C1C1E",
        "colorName": "ブラックチタニウム",
        "imageUrls": [
          "/images/products/iphone15promax_black_1.jpg",
          "/images/products/iphone15promax_black_2.jpg"
        ],
        "campaigns": [
          {
            "campaignCode": "NEW_YEAR_2025",
            "badgeText": "新春セール"
          }
        ]
      }
    ],
    "meta": {
      "pagination": {
        "page": 0,
        "perPage": 10,
        "total": 15,
        "pages": 2
      }
    }
  },
  "timestamp": "2025-12-14T10:00:00.000Z",
  "requestId": "550e8400-e29b-41d4-a716-446655440001"
}
```

#### エラーレスポンス

**404 Not Found** - カテゴリが存在しない

```json
{
  "timestamp": "2025-12-14T10:00:00.000+00:00",
  "status": 404,
  "error": "Not Found",
  "message": "カテゴリが見つかりません: invalid-category",
  "path": "/api/v1/products/categories/invalid-category"
}
```

---

### 3. おすすめ商品取得

**エンドポイント**: `GET /api/v1/products/categories/{categoryCode}/recommendations`

**説明**: 指定されたカテゴリのおすすめ商品を取得します。

> **注意**: 現在の実装では、ec-site-recommendation-serviceとの連携が未実装のため、空のリストを返却します。

**認証**: 不要（公開エンドポイント）

#### パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|----------|-----|------|------|
| categoryCode | string | Yes | カテゴリコード（例: iphone, android） |

#### リクエスト例

```bash
curl -X GET "http://localhost:8080/api/v1/products/categories/iphone/recommendations" \
  -H "Content-Type: application/json"
```

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "おすすめ商品を取得しました（現在おすすめ商品はありません）",
  "data": {
    "recommendations": []
  },
  "timestamp": "2025-12-14T10:00:00.000Z",
  "requestId": "550e8400-e29b-41d4-a716-446655440002"
}
```

#### 将来の実装時のレスポンス例

```json
{
  "success": true,
  "message": "おすすめ商品を取得しました",
  "data": {
    "recommendations": [
      {
        "productId": 1,
        "productName": "iPhone 15 Pro Max",
        "description": "最新のiPhone 15 Pro Max",
        "price": 189800,
        "manufacturer": "Apple",
        "modelName": "iPhone 15 Pro Max",
        "imageUrls": ["/images/products/iphone15promax.jpg"],
        "recommendationReason": "人気商品"
      }
    ]
  },
  "timestamp": "2025-12-14T10:00:00.000Z",
  "requestId": "550e8400-e29b-41d4-a716-446655440003"
}
```

#### エラーレスポンス

**404 Not Found** - カテゴリが存在しない

```json
{
  "timestamp": "2025-12-14T10:00:00.000+00:00",
  "status": 404,
  "error": "Not Found",
  "message": "カテゴリが見つかりません: invalid-category",
  "path": "/api/v1/products/categories/invalid-category/recommendations"
}
```

---

## データモデル

### CategoryListResponse

カテゴリ一覧レスポンスのデータモデル。

```json
{
  "success": "boolean",
  "message": "string",
  "data": "CategorySummary[]",
  "timestamp": "string (ISO 8601)",
  "requestId": "string (UUID)"
}
```

### CategorySummary

カテゴリサマリーのデータモデル。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| categoryCode | string | Yes | カテゴリコード |
| displayName | string | Yes | 表示名 |
| heroImageUrl | string | No | ヒーロー画像URL |
| leadText | string | No | リードテキスト |
| productCount | number | Yes | 商品数 |

### CategoryDetailResponse

カテゴリ詳細レスポンスのデータモデル。

```json
{
  "success": "boolean",
  "message": "string",
  "data": {
    "category": "CategoryInfo",
    "products": "ProductItem[]",
    "meta": "Meta"
  },
  "timestamp": "string (ISO 8601)",
  "requestId": "string (UUID)"
}
```

### CategoryInfo

カテゴリ情報のデータモデル。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| categoryCode | string | Yes | カテゴリコード |
| displayName | string | Yes | 表示名 |
| heroImageUrl | string | No | ヒーロー画像URL |
| leadText | string | No | リードテキスト |

### ProductItem

商品アイテムのデータモデル。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| productId | number | Yes | 商品ID |
| productName | string | Yes | 商品名 |
| description | string | No | 商品説明 |
| price | number | Yes | 価格（税込） |
| manufacturer | string | No | メーカー名 |
| modelName | string | No | モデル名 |
| storageCapacity | string | No | ストレージ容量 |
| colorCode | string | No | カラーコード（HEX） |
| colorName | string | No | カラー名 |
| imageUrls | string[] | No | 商品画像URLリスト |
| campaigns | CampaignBadge[] | No | 適用キャンペーンリスト |

### CampaignBadge

キャンペーンバッジのデータモデル。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| campaignCode | string | Yes | キャンペーンコード |
| badgeText | string | Yes | バッジ表示テキスト |

### Pagination

ページネーション情報のデータモデル。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| page | number | Yes | 現在のページ番号（0始まり） |
| perPage | number | Yes | 1ページあたりの件数 |
| total | number | Yes | 総件数 |
| pages | number | Yes | 総ページ数 |

### CategoryRecommendationResponse

おすすめ商品レスポンスのデータモデル。

```json
{
  "success": "boolean",
  "message": "string",
  "data": {
    "recommendations": "RecommendedProduct[]"
  },
  "timestamp": "string (ISO 8601)",
  "requestId": "string (UUID)"
}
```

### RecommendedProduct

おすすめ商品のデータモデル。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| productId | number | Yes | 商品ID |
| productName | string | Yes | 商品名 |
| description | string | No | 商品説明 |
| price | number | Yes | 価格（税込） |
| manufacturer | string | No | メーカー名 |
| modelName | string | No | モデル名 |
| imageUrls | string[] | No | 商品画像URLリスト |
| recommendationReason | string | No | おすすめ理由 |

---

## エラーコード

### HTTPステータスコード

| コード | 説明 | 使用例 |
|-------|------|--------|
| 200 | OK | 正常取得 |
| 400 | Bad Request | バリデーションエラー（パラメータ不正） |
| 404 | Not Found | カテゴリが存在しない |
| 500 | Internal Server Error | サーバー内部エラー |

### バリデーションエラーメッセージ

| パラメータ | エラー条件 | メッセージ |
|----------|-----------|----------|
| keyword | 100文字超過 | キーワードは100文字以内で指定してください |
| page | 0未満 | ページ番号は0以上である必要があります |
| size | 1未満 | ページサイズは1以上である必要があります |
| size | 100超過 | ページサイズは100以下である必要があります |

---

## レート制限

### 制限値

| エンドポイント | 制限 | 期間 |
|--------------|------|------|
| GET /api/v1/products/categories | 100リクエスト | 1分間 |
| GET /api/v1/products/categories/{categoryCode} | 100リクエスト | 1分間 |
| GET /api/v1/products/categories/{categoryCode}/recommendations | 50リクエスト | 1分間 |

### レート制限超過時のレスポンス

**429 Too Many Requests**

```json
{
  "timestamp": "2025-12-14T10:00:00.000+00:00",
  "status": 429,
  "error": "Too Many Requests",
  "message": "リクエスト制限を超過しました。しばらく待ってから再度お試しください。",
  "path": "/api/v1/products/categories"
}
```

---

## バージョニング

### バージョン管理方針

- APIバージョンはURLパスに含める（例: `/api/v1/products/categories`）
- メジャーバージョン変更時は新しいパスを追加
- 旧バージョンは最低6ヶ月間サポートを継続

### 現在のバージョン

- **v1**: 現在のアクティブバージョン

---

## フロントエンド連携

### 静的データとの関係

フロントエンド（ec-site-demo-frontend）では、現在以下の静的カテゴリデータを使用しています。

```typescript
const CATEGORIES = [
  { id: 1, name: 'iPhone', slug: 'iphone', displayOrder: 1 },
  { id: 2, name: 'Android', slug: 'android', displayOrder: 2 },
  { id: 3, name: 'ドコモ認定リユース品', slug: 'docomo-certified', displayOrder: 3 },
  { id: 4, name: 'アクセサリ', slug: 'accessories', displayOrder: 4 }
]
```

バックエンドAPIとの連携時は、`slug`フィールドを`categoryCode`として使用します。

---

## 関連ドキュメント

- [OpenAPI仕様書（Swagger形式）](../openapi/openapi.yaml) - 機械可読なAPI仕様
- [データモデル文書](../../02_architecture/database/DM001_data_model.md) - データベース設計
- [商品カテゴリモジュール設計書](../../06_implementation/module_specifications/MS002_product_module.md) - 実装詳細

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|-----------|------|--------|---------|
| 1.0.0 | 2025-12-14 | Devin AI | 初版作成 |
| 1.0.1 | 2025-12-14 | Devin AI | OpenAPI仕様書へのリンクを追加 |
