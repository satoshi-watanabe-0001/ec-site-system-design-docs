---
title: "アカウントAPI仕様書"
version: "1.0.0"
last_updated: "2026-03-05"
status: "Draft"
template_type: "Technical Specification"
---

# アカウントAPI仕様書

> ahamoマイページ機能を提供するREST APIの仕様書。ダッシュボード、契約情報、データ使用量、請求情報、設定変更、プラン変更、オプション管理の各エンドポイントを定義。

**API Version**: v1.0.0  
**Document Version**: 1.0.0  
**Last Updated**: 2026-03-05  
**Status**: Draft  
**Ticket**: EC-278

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

アカウントAPIは、ahamoマイページの各機能を提供します。ユーザーのダッシュボード表示、契約情報の確認、データ使用量の確認、請求情報の表示、プロフィール更新、パスワード変更、通知設定、プラン変更、オプション管理の機能を含みます。

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

すべてのエンドポイントは **Bearer Token** 認証が必要です。認証APIで取得したJWTアクセストークンをAuthorizationヘッダーに含めてリクエストしてください。

```
Authorization: Bearer <access_token>
```

認証ヘッダーが欠落または無効な場合、`401 Unauthorized` が返却されます。

---

## エンドポイント

### 1. ダッシュボード情報取得

**エンドポイント**: `GET /api/v1/account/dashboard`

**説明**: マイページダッシュボードに表示するサマリー情報を取得します。

**認証**: 必須（Bearer Token）

#### レスポンス: `200 OK`

```json
{
  "userId": "user-001",
  "userName": "山田太郎",
  "currentPlan": "ahamo（20GB）",
  "monthlyCharge": 2970,
  "dataUsage": {
    "used": 12.5,
    "limit": 20,
    "unit": "GB",
    "percentage": 62.5,
    "billingCycleEnd": "2026-03-31"
  },
  "billingPreview": {
    "currentMonth": "2026年3月",
    "estimatedAmount": 2970,
    "paymentDate": "2026-04-25"
  },
  "device": {
    "name": "iPhone 15 Pro",
    "imei": "353012345678901"
  },
  "notifications": [
    {
      "id": "notif-001",
      "title": "データ使用量のお知らせ",
      "message": "今月のデータ使用量が60%を超えました",
      "date": "2026-03-01",
      "isRead": false
    }
  ]
}
```

---

### 2. 契約情報取得

**エンドポイント**: `GET /api/v1/account/contract`

**説明**: ユーザーの契約詳細情報を取得します。

**認証**: 必須（Bearer Token）

#### レスポンス: `200 OK`

```json
{
  "contractId": "CNT-2024-001234",
  "contractorName": "山田太郎",
  "phoneNumber": "090-1234-5678",
  "email": "test@docomo.ne.jp",
  "simType": "eSIM",
  "contractStartDate": "2024-01-15",
  "planId": "ahamo-20gb",
  "planName": "ahamo（20GB）",
  "monthlyCharge": 2970,
  "dataCapacity": 20,
  "freeCallMinutes": 5,
  "contractRenewalDate": "2026-01-15",
  "device": {
    "name": "iPhone 15 Pro",
    "manufacturer": "Apple",
    "imei": "353012345678901",
    "purchaseDate": "2024-01-15"
  },
  "options": [
    {
      "id": "opt-001",
      "name": "かけ放題オプション",
      "monthlyPrice": 1100,
      "startDate": "2024-01-15"
    }
  ]
}
```

---

### 3. データ使用量取得

**エンドポイント**: `GET /api/v1/account/data-usage`

**説明**: 当月のデータ使用量と使用履歴を取得します。

**認証**: 必須（Bearer Token）

#### レスポンス: `200 OK`

```json
{
  "currentUsage": {
    "used": 12.5,
    "limit": 20,
    "unit": "GB",
    "percentage": 62.5,
    "remainingDays": 26,
    "billingCycleStart": "2026-03-01",
    "billingCycleEnd": "2026-03-31"
  },
  "dailyUsage": [
    {
      "date": "2026-03-01",
      "usage": 0.8
    },
    {
      "date": "2026-03-02",
      "usage": 1.2
    }
  ],
  "monthlyUsage": [
    {
      "month": "2025-10",
      "usage": 15.2,
      "limit": 20
    },
    {
      "month": "2025-11",
      "usage": 18.7,
      "limit": 20
    }
  ]
}
```

---

### 4. 請求情報取得

**エンドポイント**: `GET /api/v1/account/billing`

**説明**: 請求履歴と支払い方法の情報を取得します。

**認証**: 必須（Bearer Token）

#### レスポンス: `200 OK`

```json
{
  "currentBilling": {
    "month": "2026年3月",
    "totalAmount": 4070,
    "paymentDate": "2026-04-25",
    "status": "unpaid"
  },
  "billingHistory": [
    {
      "month": "2026年2月",
      "totalAmount": 4070,
      "paymentDate": "2026-03-25",
      "status": "paid",
      "items": [
        {
          "name": "ahamo（20GB）基本料金",
          "amount": 2970
        },
        {
          "name": "かけ放題オプション",
          "amount": 1100
        }
      ]
    }
  ],
  "paymentMethod": {
    "type": "credit_card",
    "displayName": "VISA **** 1234",
    "expiryDate": "2028-12"
  }
}
```

---

### 5. プロフィール更新

**エンドポイント**: `PUT /api/v1/account/profile`

**説明**: ユーザーの氏名・メールアドレスを更新します。

**認証**: 必須（Bearer Token）

#### リクエストボディ

```json
{
  "name": "山田太郎",
  "email": "newemail@docomo.ne.jp"
}
```

#### バリデーション

| フィールド | 型 | 必須 | ルール |
|----------|-----|------|--------|
| name | string | Yes | 必須、1文字以上 |
| email | string | Yes | 必須、有効なメールアドレス形式 |

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "プロフィールを更新しました。"
}
```

---

### 6. パスワード変更

**エンドポイント**: `PUT /api/v1/account/password`

**説明**: ユーザーのパスワードを変更します。

**認証**: 必須（Bearer Token）

#### リクエストボディ

```json
{
  "currentPassword": "oldpassword123",
  "newPassword": "newpassword456"
}
```

#### バリデーション

| フィールド | 型 | 必須 | ルール |
|----------|-----|------|--------|
| currentPassword | string | Yes | 必須 |
| newPassword | string | Yes | 必須、8文字以上 |

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "パスワードを変更しました。"
}
```

#### エラーレスポンス

**400 Bad Request** - 現在のパスワードが不正

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "現在のパスワードが正しくありません。"
}
```

---

### 7. 通知設定更新

**エンドポイント**: `PUT /api/v1/account/notifications`

**説明**: 通知設定を更新します。

**認証**: 必須（Bearer Token）

#### リクエストボディ

```json
{
  "emailNotifications": true,
  "campaignNotifications": false,
  "billingNotifications": true,
  "dataUsageAlerts": true
}
```

#### バリデーション

| フィールド | 型 | 必須 | ルール |
|----------|-----|------|--------|
| emailNotifications | boolean | Yes | メール通知の有効/無効 |
| campaignNotifications | boolean | Yes | キャンペーン通知の有効/無効 |
| billingNotifications | boolean | Yes | 請求通知の有効/無効 |
| dataUsageAlerts | boolean | Yes | データ使用量アラートの有効/無効 |

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "通知設定を更新しました。"
}
```

---

### 8. プラン変更

**エンドポイント**: `POST /api/v1/account/plan-change`

**説明**: 契約プランの変更を申請します。変更は翌月1日より適用されます。

**認証**: 必須（Bearer Token）

#### リクエストボディ

```json
{
  "newPlanId": "ahamo-100gb"
}
```

#### バリデーション

| フィールド | 型 | 必須 | ルール |
|----------|-----|------|--------|
| newPlanId | string | Yes | 有効なプランID |

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "プランの変更を受け付けました。翌月1日より適用されます。",
  "effectiveDate": "2026-04-01"
}
```

---

### 9. オプション一覧取得

**エンドポイント**: `GET /api/v1/account/options`

**説明**: 契約中のオプションと利用可能なオプションの一覧を取得します。

**認証**: 必須（Bearer Token）

#### レスポンス: `200 OK`

```json
{
  "subscribedOptions": [
    {
      "id": "opt-001",
      "name": "かけ放題オプション",
      "description": "国内通話が24時間かけ放題",
      "monthlyPrice": 1100,
      "startDate": "2024-01-15"
    }
  ],
  "availableOptions": [
    {
      "id": "opt-002",
      "name": "大盛りオプション",
      "description": "データ容量を+80GB追加",
      "monthlyPrice": 1980,
      "category": "データ",
      "features": [
        "データ容量+80GB",
        "テザリングも対象"
      ]
    }
  ]
}
```

---

### 10. オプション追加

**エンドポイント**: `POST /api/v1/account/options`

**説明**: オプションサービスを追加します。

**認証**: 必須（Bearer Token）

#### リクエストボディ

```json
{
  "optionId": "opt-002"
}
```

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "オプションを追加しました。"
}
```

---

### 11. オプション解除

**エンドポイント**: `DELETE /api/v1/account/options/{optionId}`

**説明**: 契約中のオプションサービスを解除します。月末で適用終了となります。

**認証**: 必須（Bearer Token）

#### パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|----------|-----|------|------|
| optionId | string | Yes | 解除するオプションのID |

#### レスポンス: `200 OK`

```json
{
  "success": true,
  "message": "オプションを解除しました。月末で適用終了となります。"
}
```

---

## データモデル

### AccountDashboardResponse

ダッシュボード表示用のサマリーデータ。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| userId | string | Yes | ユーザーID |
| userName | string | Yes | ユーザー名 |
| currentPlan | string | Yes | 現在のプラン名 |
| monthlyCharge | number | Yes | 月額料金（税込） |
| dataUsage | DataUsageSummary | Yes | データ使用量サマリー |
| billingPreview | BillingPreview | Yes | 請求プレビュー |
| device | DeviceInfo | Yes | 利用端末情報 |
| notifications | Notification[] | Yes | 通知一覧 |

### ContractInfoResponse

契約詳細情報。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| contractId | string | Yes | 契約ID |
| contractorName | string | Yes | 契約者名 |
| phoneNumber | string | Yes | 電話番号 |
| email | string | Yes | メールアドレス |
| simType | string | Yes | SIMタイプ（eSIM/nanoSIM） |
| contractStartDate | string | Yes | 契約開始日（ISO 8601） |
| planId | string | Yes | プランID |
| planName | string | Yes | プラン名 |
| monthlyCharge | number | Yes | 月額料金（税込） |
| dataCapacity | number | Yes | データ容量（GB） |
| freeCallMinutes | number | Yes | 無料通話分数 |
| contractRenewalDate | string | Yes | 契約更新日（ISO 8601） |
| device | DeviceInfo | Yes | 利用端末情報 |
| options | ContractOption[] | Yes | 契約中のオプション一覧 |

### DataUsageResponse

データ使用量詳細。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| currentUsage | CurrentUsage | Yes | 当月のデータ使用状況 |
| dailyUsage | DailyUsage[] | Yes | 日別データ使用量 |
| monthlyUsage | MonthlyUsage[] | Yes | 月別データ使用量履歴 |

### BillingInfoResponse

請求情報。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| currentBilling | MonthlyBill | Yes | 当月の請求情報 |
| billingHistory | MonthlyBill[] | Yes | 請求履歴 |
| paymentMethod | PaymentMethodInfo | Yes | 支払い方法 |

### AccountOptionsResponse

オプションサービス情報。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| subscribedOptions | ContractOption[] | Yes | 契約中のオプション |
| availableOptions | AvailableOption[] | Yes | 利用可能なオプション |

---

## エラーコード

### HTTPステータスコード

| コード | 説明 | 使用例 |
|-------|------|--------|
| 200 | OK | 正常レスポンス |
| 400 | Bad Request | バリデーションエラー |
| 401 | Unauthorized | 認証ヘッダーの欠落・トークン無効 |
| 404 | Not Found | リソースが見つからない |
| 500 | Internal Server Error | サーバー内部エラー |

### 共通エラーレスポンス形式

```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "認証が必要です。ログインしてください。"
}
```

---

## レート制限

### 制限値

| エンドポイント | 制限 | 期間 |
|--------------|------|------|
| GET エンドポイント | 60リクエスト | 1分間 |
| PUT/POST/DELETE エンドポイント | 30リクエスト | 1分間 |

---

## バージョニング

### バージョン管理方針

- APIバージョンはURLパスに含める（例: `/api/v1/account/dashboard`）
- メジャーバージョン変更時は新しいパスを追加
- 旧バージョンは最低6ヶ月間サポートを継続

### 現在のバージョン

- **v1**: 現在のアクティブバージョン

---

## セキュリティ考慮事項

### 認証・認可

- すべてのエンドポイントでBearer Token認証が必要
- トークンの有効期限を確認し、期限切れの場合は401を返却
- ユーザーは自身のデータのみアクセス可能

### 通信セキュリティ

- 本番環境ではHTTPS必須
- TLS 1.2以上を使用

---

## 関連ドキュメント

- [認証API仕様書](EP001_auth_api.md) - 認証エンドポイント
- [商品API仕様書](EP002_product_api.md) - 商品エンドポイント

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|-----------|------|--------|---------|
| 1.0.0 | 2026-03-05 | Devin AI | 初版作成（EC-278） |
