---
title: "アカウント管理API仕様書"
version: "1.0.0"
last_updated: "2026-01-30"
status: "Draft"
template_type: "Technical Specification"
---

# アカウント管理API仕様書

> マイページ機能を提供するREST APIの仕様書。アカウント情報、契約情報、請求情報、データ使用量、プラン・オプション管理を含む。

**API Version**: v1.0.0  
**Document Version**: 1.0.0  
**Last Updated**: 2026-01-30  
**Status**: Draft

---

## 目次

1. [概要](#概要)
2. [認証](#認証)
3. [エンドポイント](#エンドポイント)
4. [データモデル](#データモデル)
5. [エラーコード](#エラーコード)
6. [レート制限](#レート制限)
7. [変更履歴](#変更履歴)

---

## 概要

### APIの目的

アカウント管理APIは、ECサイトのマイページ機能を提供します。認証済みユーザーに対して、アカウント情報の参照・更新、契約情報の確認、請求・支払い管理、データ使用量の確認、プラン・オプションの管理機能を提供します。

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

すべてのエンドポイントは認証が必要です。リクエストヘッダーに有効なJWTアクセストークンを含める必要があります。

```
Authorization: Bearer <access_token>
```

---

## エンドポイント

### アカウント管理

#### プロフィール取得

**エンドポイント**: `GET /api/v1/account/profile`

**説明**: 認証ユーザーのアカウント情報を取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "id": "ACC001",
  "email": "user@docomo.ne.jp",
  "name": "山田 太郎",
  "phoneNumber": "090-1234-5678",
  "address": {
    "postalCode": "100-0001",
    "prefecture": "東京都",
    "city": "千代田区",
    "street": "丸の内1-1-1",
    "building": "東京ビル101"
  },
  "dateOfBirth": "1990-01-15",
  "createdAt": "2024-01-15T10:00:00Z",
  "updatedAt": "2025-06-20T14:30:00Z"
}
```

#### プロフィール更新

**エンドポイント**: `PUT /api/v1/account/profile`

**説明**: 認証ユーザーのアカウント情報を更新します。

**認証**: 必須

**リクエストボディ**

```json
{
  "name": "山田 太郎",
  "phoneNumber": "090-1234-5678",
  "address": {
    "postalCode": "100-0001",
    "prefecture": "東京都",
    "city": "千代田区",
    "street": "丸の内1-1-1",
    "building": "東京ビル101"
  }
}
```

**レスポンス: `200 OK`**

```json
{
  "success": true,
  "message": "プロフィールを更新しました"
}
```

#### パスワード変更

**エンドポイント**: `PUT /api/v1/account/password`

**説明**: 認証ユーザーのパスワードを変更します。

**認証**: 必須

**リクエストボディ**

```json
{
  "currentPassword": "current_password",
  "newPassword": "new_password",
  "confirmPassword": "new_password"
}
```

**レスポンス: `200 OK`**

```json
{
  "success": true,
  "message": "パスワードを変更しました"
}
```

#### 通知設定取得

**エンドポイント**: `GET /api/v1/account/notification-settings`

**説明**: 認証ユーザーの通知設定を取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "email": {
    "billing": true,
    "campaign": true,
    "serviceUpdate": true,
    "maintenance": false
  },
  "push": {
    "billing": true,
    "campaign": false,
    "dataUsage": true
  },
  "sms": {
    "security": true,
    "billing": false
  }
}
```

#### 通知設定更新

**エンドポイント**: `PUT /api/v1/account/notification-settings`

**説明**: 認証ユーザーの通知設定を更新します。

**認証**: 必須

**リクエストボディ**

```json
{
  "email": {
    "billing": true,
    "campaign": true,
    "serviceUpdate": true,
    "maintenance": false
  },
  "push": {
    "billing": true,
    "campaign": false,
    "dataUsage": true
  },
  "sms": {
    "security": true,
    "billing": false
  }
}
```

**レスポンス: `200 OK`**

```json
{
  "success": true,
  "message": "通知設定を更新しました"
}
```

---

### 契約情報

#### 契約サマリー取得

**エンドポイント**: `GET /api/v1/contract/summary`

**説明**: 契約情報のサマリーを取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "contractId": "CNT-2024-001234",
  "phoneNumber": "090-1234-5678",
  "planName": "ahamo",
  "dataCapacity": 20,
  "status": "active",
  "contractDate": "2024-01-15"
}
```

#### 契約詳細取得

**エンドポイント**: `GET /api/v1/contract/details`

**説明**: 契約の詳細情報を取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "contractId": "CNT-2024-001234",
  "phoneNumber": "090-1234-5678",
  "status": "active",
  "contractDate": "2024-01-15",
  "contractor": {
    "name": "山田 太郎",
    "nameKana": "ヤマダ タロウ",
    "dateOfBirth": "1990-01-15",
    "address": {
      "postalCode": "100-0001",
      "prefecture": "東京都",
      "city": "千代田区",
      "street": "丸の内1-1-1",
      "building": "東京ビル101"
    }
  },
  "plan": {
    "planId": "PLAN001",
    "planName": "ahamo",
    "monthlyFee": 2970,
    "dataCapacity": 20,
    "voiceCallIncluded": true,
    "internationalRoaming": true
  },
  "sim": {
    "simType": "eSIM",
    "iccid": "8981100000001234567",
    "activationDate": "2024-01-15"
  },
  "options": [
    {
      "optionId": "OPT001",
      "optionName": "大盛りオプション",
      "monthlyFee": 1980,
      "startDate": "2024-03-01"
    }
  ]
}
```

#### デバイス情報取得

**エンドポイント**: `GET /api/v1/contract/device`

**説明**: 契約に紐づくデバイス情報を取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "deviceId": "DEV001",
  "deviceName": "iPhone 15 Pro",
  "manufacturer": "Apple",
  "model": "A3101",
  "imei": "123456789012345",
  "color": "ナチュラルチタニウム",
  "storage": "256GB",
  "purchaseDate": "2024-01-15",
  "warrantyEndDate": "2025-01-14",
  "installmentRemaining": 12,
  "installmentMonthlyAmount": 4150
}
```

---

### 請求・支払い

#### 今月の請求取得

**エンドポイント**: `GET /api/v1/billing/current`

**説明**: 今月の請求情報を取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "billingId": "BILL-2026-01",
  "billingMonth": "2026-01",
  "totalAmount": 4950,
  "dueDate": "2026-02-28",
  "status": "pending",
  "breakdown": [
    {
      "item": "基本料金（ahamo）",
      "amount": 2970
    },
    {
      "item": "大盛りオプション",
      "amount": 1980
    }
  ],
  "previousBalance": 0,
  "adjustments": 0
}
```

#### 請求履歴取得

**エンドポイント**: `GET /api/v1/billing/history`

**説明**: 過去の請求履歴を取得します。

**認証**: 必須

**クエリパラメータ**

| パラメータ | 型 | 必須 | 説明 |
|----------|-----|------|------|
| limit | number | No | 取得件数（デフォルト: 12） |
| offset | number | No | オフセット（デフォルト: 0） |

**レスポンス: `200 OK`**

```json
{
  "history": [
    {
      "billingId": "BILL-2025-12",
      "billingMonth": "2025-12",
      "totalAmount": 4950,
      "paidDate": "2025-12-25",
      "status": "paid"
    },
    {
      "billingId": "BILL-2025-11",
      "billingMonth": "2025-11",
      "totalAmount": 4950,
      "paidDate": "2025-11-25",
      "status": "paid"
    }
  ],
  "totalCount": 12
}
```

#### 支払い方法取得

**エンドポイント**: `GET /api/v1/billing/payment-method`

**説明**: 登録されている支払い方法を取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "paymentMethodId": "PM001",
  "type": "credit_card",
  "cardBrand": "VISA",
  "lastFourDigits": "1234",
  "expiryMonth": 12,
  "expiryYear": 2027,
  "holderName": "TARO YAMADA",
  "isDefault": true
}
```

#### 支払い方法更新

**エンドポイント**: `PUT /api/v1/billing/payment-method`

**説明**: 支払い方法を更新します。

**認証**: 必須

**リクエストボディ**

```json
{
  "type": "credit_card",
  "cardNumber": "4111111111111111",
  "expiryMonth": 12,
  "expiryYear": 2027,
  "holderName": "TARO YAMADA",
  "securityCode": "123"
}
```

**レスポンス: `200 OK`**

```json
{
  "success": true,
  "message": "支払い方法を更新しました"
}
```

---

### データ使用量

#### 今月のデータ使用量取得

**エンドポイント**: `GET /api/v1/data-usage/current`

**説明**: 今月のデータ使用量を取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "period": "2026-01",
  "usedData": 15.5,
  "totalData": 20,
  "remainingData": 4.5,
  "usagePercentage": 77.5,
  "resetDate": "2026-02-01",
  "dailyUsage": [
    {
      "date": "2026-01-29",
      "usage": 0.8
    },
    {
      "date": "2026-01-28",
      "usage": 1.2
    }
  ]
}
```

#### データ使用量履歴取得

**エンドポイント**: `GET /api/v1/data-usage/history`

**説明**: 過去のデータ使用量履歴を取得します。

**認証**: 必須

**クエリパラメータ**

| パラメータ | 型 | 必須 | 説明 |
|----------|-----|------|------|
| months | number | No | 取得月数（デフォルト: 6） |

**レスポンス: `200 OK`**

```json
{
  "history": [
    {
      "period": "2025-12",
      "usedData": 18.2,
      "totalData": 20
    },
    {
      "period": "2025-11",
      "usedData": 16.8,
      "totalData": 20
    }
  ]
}
```

#### データ追加購入

**エンドポイント**: `POST /api/v1/data-usage/add-on`

**説明**: データ容量を追加購入します。

**認証**: 必須

**リクエストボディ**

```json
{
  "addOnId": "ADDON001",
  "dataAmount": 1
}
```

**レスポンス: `200 OK`**

```json
{
  "success": true,
  "message": "1GBを追加しました",
  "newTotalData": 21,
  "chargedAmount": 550
}
```

---

### プラン管理

#### 利用可能プラン一覧取得

**エンドポイント**: `GET /api/v1/plans`

**説明**: 変更可能なプラン一覧を取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "currentPlanId": "PLAN001",
  "plans": [
    {
      "planId": "PLAN001",
      "planName": "ahamo",
      "monthlyFee": 2970,
      "dataCapacity": 20,
      "description": "20GB + 5分かけ放題",
      "features": [
        "20GBのデータ容量",
        "5分以内の国内通話無料",
        "海外82の国・地域でそのまま使える"
      ],
      "isCurrent": true
    },
    {
      "planId": "PLAN002",
      "planName": "ahamo大盛り",
      "monthlyFee": 4950,
      "dataCapacity": 100,
      "description": "100GB + 5分かけ放題",
      "features": [
        "100GBのデータ容量",
        "5分以内の国内通話無料",
        "海外82の国・地域でそのまま使える"
      ],
      "isCurrent": false
    }
  ]
}
```

#### プラン変更シミュレーション

**エンドポイント**: `POST /api/v1/plans/simulate`

**説明**: プラン変更時の料金シミュレーションを行います。

**認証**: 必須

**リクエストボディ**

```json
{
  "newPlanId": "PLAN002"
}
```

**レスポンス: `200 OK`**

```json
{
  "currentPlan": {
    "planId": "PLAN001",
    "planName": "ahamo",
    "monthlyFee": 2970
  },
  "newPlan": {
    "planId": "PLAN002",
    "planName": "ahamo大盛り",
    "monthlyFee": 4950
  },
  "priceDifference": 1980,
  "effectiveDate": "2026-02-01",
  "prorationAmount": 0
}
```

#### プラン変更申請

**エンドポイント**: `POST /api/v1/plans/change`

**説明**: プラン変更を申請します。

**認証**: 必須

**リクエストボディ**

```json
{
  "newPlanId": "PLAN002",
  "effectiveDate": "2026-02-01"
}
```

**レスポンス: `200 OK`**

```json
{
  "success": true,
  "message": "プラン変更を受け付けました",
  "changeRequestId": "CHG-2026-001",
  "effectiveDate": "2026-02-01"
}
```

---

### オプション管理

#### オプション一覧取得

**エンドポイント**: `GET /api/v1/options`

**説明**: 利用可能なオプションと契約中のオプションを取得します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "subscribedOptions": [
    {
      "optionId": "OPT001",
      "optionName": "大盛りオプション",
      "monthlyFee": 1980,
      "description": "+80GBのデータ容量追加",
      "startDate": "2024-03-01",
      "canCancel": true
    }
  ],
  "availableOptions": [
    {
      "optionId": "OPT002",
      "optionName": "かけ放題オプション",
      "monthlyFee": 1100,
      "description": "国内通話が24時間かけ放題",
      "features": [
        "国内通話24時間無料",
        "SMS送信料は別途"
      ]
    },
    {
      "optionId": "OPT003",
      "optionName": "端末補償サービス",
      "monthlyFee": 825,
      "description": "故障・紛失時の端末補償",
      "features": [
        "故障時の修理費用補償",
        "紛失時の端末交換"
      ]
    }
  ]
}
```

#### オプション申込

**エンドポイント**: `POST /api/v1/options/:optionId/subscribe`

**説明**: オプションを申し込みます。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "success": true,
  "message": "オプションを申し込みました",
  "startDate": "2026-02-01"
}
```

#### オプション解約

**エンドポイント**: `DELETE /api/v1/options/:optionId/subscribe`

**説明**: オプションを解約します。

**認証**: 必須

**レスポンス: `200 OK`**

```json
{
  "success": true,
  "message": "オプションを解約しました",
  "endDate": "2026-01-31"
}
```

---

## データモデル

### AccountProfile

アカウントプロフィール情報。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| id | string | Yes | アカウントID |
| email | string | Yes | メールアドレス |
| name | string | Yes | 氏名 |
| phoneNumber | string | Yes | 電話番号 |
| address | Address | Yes | 住所情報 |
| dateOfBirth | string | Yes | 生年月日（YYYY-MM-DD） |
| createdAt | string | Yes | 作成日時（ISO 8601） |
| updatedAt | string | Yes | 更新日時（ISO 8601） |

### ContractSummary

契約サマリー情報。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| contractId | string | Yes | 契約ID |
| phoneNumber | string | Yes | 電話番号 |
| planName | string | Yes | プラン名 |
| dataCapacity | number | Yes | データ容量（GB） |
| status | string | Yes | 契約ステータス |
| contractDate | string | Yes | 契約日（YYYY-MM-DD） |

### BillingInfo

請求情報。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| billingId | string | Yes | 請求ID |
| billingMonth | string | Yes | 請求月（YYYY-MM） |
| totalAmount | number | Yes | 合計金額 |
| dueDate | string | Yes | 支払期限（YYYY-MM-DD） |
| status | string | Yes | 請求ステータス |
| breakdown | BillingItem[] | Yes | 内訳 |

### DataUsage

データ使用量情報。

| フィールド | 型 | 必須 | 説明 |
|----------|-----|------|------|
| period | string | Yes | 対象期間（YYYY-MM） |
| usedData | number | Yes | 使用済みデータ量（GB） |
| totalData | number | Yes | 総データ容量（GB） |
| remainingData | number | Yes | 残りデータ量（GB） |
| usagePercentage | number | Yes | 使用率（%） |
| resetDate | string | Yes | リセット日（YYYY-MM-DD） |

---

## エラーコード

### HTTPステータスコード

| コード | 説明 | 使用例 |
|-------|------|--------|
| 200 | OK | 正常完了 |
| 400 | Bad Request | バリデーションエラー |
| 401 | Unauthorized | 認証エラー |
| 403 | Forbidden | 権限エラー |
| 404 | Not Found | リソースが見つからない |
| 500 | Internal Server Error | サーバー内部エラー |

---

## レート制限

### 制限値

| エンドポイント | 制限 | 期間 |
|--------------|------|------|
| GET系エンドポイント | 100リクエスト | 1分間 |
| POST/PUT/DELETE系エンドポイント | 30リクエスト | 1分間 |

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|-----------|------|--------|---------|
| 1.0.0 | 2026-01-30 | Devin AI | 初版作成（EC-278対応） |
