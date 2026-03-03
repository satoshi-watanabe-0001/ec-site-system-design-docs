# マイページ アカウント管理 API設計書

## EC-278: ahamoマイページ アカウント管理機能

### 概要

ahamoマイページのアカウント管理機能で使用するREST APIの設計書です。
ユーザープロフィール、契約情報、データ使用量、請求情報、プラン管理、オプション管理、通知設定の各APIを定義します。

### 共通仕様

#### Base URL
```
{NEXT_PUBLIC_API_URL}/api/v1/account
```

#### 認証
- すべてのエンドポイントにBearer Token認証が必要
- ヘッダー: `Authorization: Bearer {access_token}`
- トークン取得: `/api/v1/auth/login` エンドポイントから取得

#### レスポンスフォーマット
- Content-Type: `application/json`
- 文字エンコーディング: UTF-8

#### 共通エラーレスポンス

| HTTPステータス | 説明 | レスポンス例 |
|---|---|---|
| 400 | リクエスト不正 | `{"error": "BAD_REQUEST", "message": "入力内容に誤りがあります"}` |
| 401 | 認証エラー | `{"error": "UNAUTHORIZED", "message": "認証が必要です"}` |
| 403 | 権限不足 | `{"error": "FORBIDDEN", "message": "アクセス権限がありません"}` |
| 404 | リソース未検出 | `{"error": "NOT_FOUND", "message": "指定されたリソースが見つかりません"}` |
| 429 | レート制限 | `{"error": "TOO_MANY_REQUESTS", "message": "リクエスト回数の上限に達しました"}` |
| 500 | サーバーエラー | `{"error": "INTERNAL_SERVER_ERROR", "message": "サーバーエラーが発生しました"}` |

#### レート制限
- 一般エンドポイント: 60リクエスト/分
- 更新系エンドポイント: 10リクエスト/分
- レスポンスヘッダー:
  - `X-RateLimit-Limit`: 制限値
  - `X-RateLimit-Remaining`: 残りリクエスト数
  - `X-RateLimit-Reset`: リセット時刻（UNIX timestamp）

---

## 1. プロフィール管理

### 1.1 GET /profile - プロフィール取得

ログインユーザーのプロフィール情報を取得します。

**リクエスト**: なし

**レスポンス (200 OK)**:
```json
{
  "userId": "user_001",
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "phoneNumber": "090-1234-5678",
  "dateOfBirth": "1990-01-15",
  "address": {
    "postalCode": "100-0001",
    "prefecture": "東京都",
    "city": "千代田区",
    "street": "丸の内1-1-1",
    "building": "東京ビル 5F"
  },
  "accountStatus": "active",
  "createdAt": "2023-04-01T00:00:00Z",
  "lastLoginAt": "2026-03-03T09:00:00Z"
}
```

### 1.2 PUT /profile - プロフィール更新

ユーザーの連絡先情報を更新します。

**リクエスト**:
```json
{
  "name": "田中太郎",
  "email": "tanaka-new@example.com",
  "phoneNumber": "090-1234-5678",
  "address": {
    "postalCode": "100-0001",
    "prefecture": "東京都",
    "city": "千代田区",
    "street": "丸の内1-1-1",
    "building": "東京ビル 5F"
  }
}
```

**バリデーション**:
| フィールド | 必須 | ルール |
|---|---|---|
| name | ○ | 1〜50文字 |
| email | ○ | メールアドレス形式 |
| phoneNumber | ○ | 日本の電話番号形式 |
| address.postalCode | ○ | `\d{3}-\d{4}` 形式 |
| address.prefecture | ○ | 都道府県名 |
| address.city | ○ | 1〜100文字 |
| address.street | ○ | 1〜200文字 |
| address.building | × | 0〜200文字 |

**レスポンス (200 OK)**:
```json
{
  "message": "プロフィールを更新しました"
}
```

### 1.3 PUT /password - パスワード変更

ログインパスワードを変更します。

**リクエスト**:
```json
{
  "currentPassword": "current_password",
  "newPassword": "new_password_123"
}
```

**バリデーション**:
| フィールド | 必須 | ルール |
|---|---|---|
| currentPassword | ○ | 現在のパスワード |
| newPassword | ○ | 8文字以上、英数字記号を含む |

**レスポンス (200 OK)**:
```json
{
  "message": "パスワードを変更しました"
}
```

**エラーレスポンス (400)**:
```json
{
  "error": "INVALID_PASSWORD",
  "message": "現在のパスワードが正しくありません"
}
```

---

## 2. 契約情報

### 2.1 GET /contract - 契約詳細取得

契約の詳細情報を取得します。

**リクエスト**: なし

**レスポンス (200 OK)**:
```json
{
  "contractNumber": "AHM-2024-001234",
  "status": "active",
  "startDate": "2024-01-15",
  "currentPlan": {
    "id": "plan_ahamo_large",
    "name": "ahamo大盛り",
    "monthlyPrice": 4950,
    "dataCapacity": 100,
    "description": "大容量100GBプラン",
    "features": [
      "データ容量100GB",
      "5分かけ放題",
      "海外82の国と地域で使える",
      "テザリング無制限"
    ]
  },
  "device": {
    "id": "device_001",
    "name": "iPhone 15 Pro",
    "manufacturer": "Apple",
    "color": "ナチュラルチタニウム",
    "storage": "256GB",
    "imei": "****1234",
    "purchaseDate": "2024-01-15",
    "installmentInfo": {
      "totalAmount": 159800,
      "monthlyAmount": 4439,
      "totalInstallments": 36,
      "remainingInstallments": 24,
      "remainingAmount": 106536
    }
  },
  "simInfo": {
    "simType": "esim",
    "phoneNumber": "090-1234-5678",
    "iccidLast4": "5678"
  },
  "options": [
    {
      "id": "opt_001",
      "name": "かけ放題オプション",
      "monthlyPrice": 1100,
      "description": "国内通話完全かけ放題",
      "status": "active"
    }
  ]
}
```

---

## 3. データ使用量

### 3.1 GET /data-usage - 当月データ使用量

当月のデータ使用量を取得します。

**リクエスト**: なし

**レスポンス (200 OK)**:
```json
{
  "usedData": 15.2,
  "totalCapacity": 20,
  "remainingData": 4.8,
  "usagePercentage": 76,
  "additionalData": 0,
  "isThrottled": false,
  "billingCycleStart": "2026-03-01",
  "billingCycleEnd": "2026-03-31",
  "lastUpdated": "2026-03-03T09:00:00Z",
  "dailyUsage": [
    { "date": "2026-03-01", "usage": 0.5 },
    { "date": "2026-03-02", "usage": 0.8 },
    { "date": "2026-03-03", "usage": 0.3 }
  ]
}
```

### 3.2 GET /data-usage/history - 使用量履歴

過去の月別データ使用量履歴を取得します。

**クエリパラメータ**:
| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| months | number | 6 | 取得月数（1〜12） |

**レスポンス (200 OK)**:
```json
{
  "history": [
    {
      "month": "2026-02",
      "usedData": 18.5,
      "totalCapacity": 20,
      "usagePercentage": 92.5
    },
    {
      "month": "2026-01",
      "usedData": 12.3,
      "totalCapacity": 20,
      "usagePercentage": 61.5
    }
  ]
}
```

### 3.3 GET /data-charge/history - チャージ履歴

データチャージ（追加購入）の履歴を取得します。

**レスポンス (200 OK)**:
```json
{
  "history": [
    {
      "id": "charge_001",
      "amount": 1,
      "price": 550,
      "type": "manual",
      "chargedAt": "2026-02-20T15:30:00Z"
    }
  ]
}
```

---

## 4. 請求・支払い

### 4.1 GET /billing/current - 当月請求

当月の請求情報を取得します。

**レスポンス (200 OK)**:
```json
{
  "billingMonth": 3,
  "billingYear": 2026,
  "totalAmount": 6050,
  "isConfirmed": false,
  "details": [
    { "name": "基本料金（ahamo大盛り）", "amount": 4950, "category": "plan" },
    { "name": "かけ放題オプション", "amount": 1100, "category": "option" },
    { "name": "ユニバーサルサービス料", "amount": 2, "category": "tax" },
    { "name": "電話リレーサービス料", "amount": 1, "category": "tax" },
    { "name": "消費税", "amount": -3, "category": "tax" }
  ],
  "dueDate": "2026-03-31",
  "paymentMethod": "credit_card"
}
```

### 4.2 GET /billing/history - 請求履歴

過去の請求履歴を取得します。

**クエリパラメータ**:
| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| months | number | 6 | 取得月数（1〜12） |

**レスポンス (200 OK)**:
```json
{
  "history": [
    {
      "billingMonth": "2026-02",
      "totalAmount": 6050,
      "paymentStatus": "paid",
      "paidAt": "2026-02-28"
    }
  ]
}
```

### 4.3 GET /payment-method - 支払い方法取得

登録済みの支払い方法を取得します。

**レスポンス (200 OK)**:
```json
{
  "type": "credit_card",
  "cardInfo": {
    "brand": "VISA",
    "last4": "1234",
    "expiryDate": "12/28",
    "holderName": "TARO TANAKA"
  },
  "bankAccountInfo": null
}
```

### 4.4 PUT /payment-method - 支払い方法更新

支払い方法を更新します。

**リクエスト**:
```json
{
  "type": "credit_card",
  "cardNumber": "4111111111111111",
  "expiryMonth": "12",
  "expiryYear": "2028",
  "holderName": "TARO TANAKA",
  "securityCode": "123"
}
```

**バリデーション**:
| フィールド | 必須 | ルール |
|---|---|---|
| type | ○ | "credit_card" / "bank_account" |
| cardNumber | △ | クレジットカード時必須、16桁 |
| expiryMonth | △ | クレジットカード時必須、01〜12 |
| expiryYear | △ | クレジットカード時必須、現在年以降 |
| holderName | △ | クレジットカード時必須 |
| securityCode | △ | クレジットカード時必須、3〜4桁 |

**レスポンス (200 OK)**:
```json
{
  "message": "支払い方法を更新しました"
}
```

---

## 5. プラン管理

### 5.1 GET /plan - プラン一覧・現在のプラン

利用可能なプランと現在のプランを取得します。

**レスポンス (200 OK)**:
```json
{
  "currentPlanId": "plan_ahamo_large",
  "plans": [
    {
      "id": "plan_ahamo",
      "name": "ahamo",
      "monthlyPrice": 2970,
      "dataCapacity": 20,
      "description": "シンプルワンプラン",
      "features": [
        "データ容量20GB",
        "5分かけ放題",
        "海外82の国と地域で使える"
      ]
    },
    {
      "id": "plan_ahamo_large",
      "name": "ahamo大盛り",
      "monthlyPrice": 4950,
      "dataCapacity": 100,
      "description": "大容量100GBプラン",
      "features": [
        "データ容量100GB",
        "5分かけ放題",
        "海外82の国と地域で使える",
        "テザリング無制限"
      ]
    }
  ]
}
```

### 5.2 PUT /plan - プラン変更

契約プランを変更します。

**リクエスト**:
```json
{
  "planId": "plan_ahamo"
}
```

**バリデーション**:
| フィールド | 必須 | ルール |
|---|---|---|
| planId | ○ | 有効なプランID |

**レスポンス (200 OK)**:
```json
{
  "message": "プランを変更しました。次の請求サイクルから適用されます。",
  "effectiveDate": "2026-04-01",
  "newPlanId": "plan_ahamo"
}
```

---

## 6. オプション管理

### 6.1 GET /options - オプション一覧

利用可能なオプションと契約状況を取得します。

**レスポンス (200 OK)**:
```json
{
  "options": [
    {
      "id": "opt_001",
      "name": "かけ放題オプション",
      "description": "国内通話が24時間かけ放題",
      "monthlyPrice": 1100,
      "category": "call",
      "isSubscribed": true,
      "features": ["国内通話無制限", "番号通知対応"]
    },
    {
      "id": "opt_002",
      "name": "ケータイ補償サービス",
      "description": "端末の故障・紛失時に交換対応",
      "monthlyPrice": 825,
      "category": "insurance",
      "isSubscribed": true,
      "features": ["故障時交換", "紛失時補償", "データ復旧"]
    },
    {
      "id": "opt_003",
      "name": "DAZN for docomo",
      "description": "スポーツ専門の動画配信サービス",
      "monthlyPrice": 3700,
      "category": "entertainment",
      "isSubscribed": false,
      "features": ["スポーツライブ配信", "見逃し配信", "マルチデバイス対応"]
    }
  ]
}
```

### 6.2 POST /options/{id} - オプション追加

指定したオプションを契約に追加します。

**パスパラメータ**:
| パラメータ | 型 | 説明 |
|---|---|---|
| id | string | オプションID |

**レスポンス (200 OK)**:
```json
{
  "message": "オプションを追加しました",
  "optionId": "opt_003",
  "effectiveDate": "2026-03-03"
}
```

### 6.3 DELETE /options/{id} - オプション解除

指定したオプションを契約から解除します。

**パスパラメータ**:
| パラメータ | 型 | 説明 |
|---|---|---|
| id | string | オプションID |

**レスポンス (200 OK)**:
```json
{
  "message": "オプションを解除しました",
  "optionId": "opt_001",
  "effectiveDate": "2026-04-01"
}
```

---

## 7. 通知

### 7.1 GET /notifications - 通知一覧

ユーザーの通知を取得します。

**クエリパラメータ**:
| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| limit | number | 20 | 取得件数（1〜100） |
| unreadOnly | boolean | false | 未読のみ |

**レスポンス (200 OK)**:
```json
{
  "notifications": [
    {
      "id": "notif_001",
      "title": "データ使用量が80%に達しました",
      "message": "今月のデータ使用量が80%を超えました。",
      "type": "data_usage",
      "isRead": false,
      "createdAt": "2026-03-02T10:00:00Z"
    }
  ],
  "unreadCount": 2,
  "totalCount": 15
}
```

### 7.2 PUT /notification-settings - 通知設定更新

通知の受信設定を更新します。

**リクエスト**:
```json
{
  "emailNotification": true,
  "smsNotification": false,
  "pushNotification": true,
  "campaignInfo": true,
  "billingNotification": true,
  "dataUsageAlert": true,
  "dataUsageAlertThreshold": 80
}
```

**バリデーション**:
| フィールド | 必須 | ルール |
|---|---|---|
| emailNotification | ○ | boolean |
| smsNotification | ○ | boolean |
| pushNotification | ○ | boolean |
| campaignInfo | ○ | boolean |
| billingNotification | ○ | boolean |
| dataUsageAlert | ○ | boolean |
| dataUsageAlertThreshold | ○ | 50〜100の整数 |

**レスポンス (200 OK)**:
```json
{
  "message": "通知設定を更新しました"
}
```

---

## 8. 端末情報

### 8.1 GET /devices - 端末一覧

契約に紐づく端末情報を取得します。

**レスポンス (200 OK)**:
```json
{
  "devices": [
    {
      "id": "device_001",
      "name": "iPhone 15 Pro",
      "manufacturer": "Apple",
      "color": "ナチュラルチタニウム",
      "storage": "256GB",
      "imei": "****1234",
      "status": "active",
      "purchaseDate": "2024-01-15"
    }
  ]
}
```

---

## データモデル定義

### UserProfile
| フィールド | 型 | 説明 |
|---|---|---|
| userId | string | ユーザーID |
| name | string | 氏名 |
| email | string | メールアドレス |
| phoneNumber | string | 電話番号 |
| dateOfBirth | string | 生年月日 |
| address | Address | 住所 |
| accountStatus | string | アカウント状態（active/suspended/closed） |
| createdAt | string | 作成日時 |
| lastLoginAt | string | 最終ログイン日時 |

### Address
| フィールド | 型 | 説明 |
|---|---|---|
| postalCode | string | 郵便番号 |
| prefecture | string | 都道府県 |
| city | string | 市区町村 |
| street | string | 番地 |
| building | string? | 建物名 |

### ContractDetail
| フィールド | 型 | 説明 |
|---|---|---|
| contractNumber | string | 契約番号 |
| status | string | 契約状態 |
| startDate | string | 契約開始日 |
| currentPlan | ContractPlan | 現在のプラン |
| device | ContractDevice | 端末情報 |
| simInfo | SimInfo | SIM情報 |
| options | ContractOption[] | 契約オプション |

### DataUsage
| フィールド | 型 | 説明 |
|---|---|---|
| usedData | number | 使用済みデータ量（GB） |
| totalCapacity | number | 総容量（GB） |
| remainingData | number | 残りデータ量（GB） |
| usagePercentage | number | 使用率（%） |
| additionalData | number | 追加データ量（GB） |
| isThrottled | boolean | 速度制限中か |
| dailyUsage | DailyUsage[] | 日別使用量 |

### CurrentBilling
| フィールド | 型 | 説明 |
|---|---|---|
| billingMonth | number | 請求月 |
| billingYear | number | 請求年 |
| totalAmount | number | 合計金額（税込） |
| isConfirmed | boolean | 確定済みか |
| details | BillingDetail[] | 請求明細 |
| dueDate | string | 支払い期限 |
| paymentMethod | string | 支払い方法 |

---

## 変更履歴

| 日付 | バージョン | 変更内容 |
|---|---|---|
| 2026-03-03 | 1.0.0 | 初版作成（EC-278） |
