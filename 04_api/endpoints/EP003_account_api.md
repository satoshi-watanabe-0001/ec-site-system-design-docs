---
title: "ahamo ECサイト アカウント管理API契約書"
version: "1.0.0"
template_version: "1.0.0"
created_date: "2026-02-12"
phase: "Phase 2A (Pre-Implementation Design)"
status: "draft"
owner: "watanabe"
reviewers: []
approved_by: ""
approved_date: ""
---

# API契約書: アカウント管理API (EC-278)

> **目的**: Phase 2Aで作成するAPI契約書  
> **スコープ**: マイページ機能に必要なエンドポイント一覧と基本的な入出力定義  
> **作成タイミング**: 実装開始前  
> **関連チケット**: EC-278 (PBI-AC-002: アカウント管理)

---

## メタデータ

| 項目 | 内容 |
|-----|------|
| **プロジェクト名** | ahamo ECサイト アカウント管理 |
| **プロジェクトコード** | EC-278 |
| **作成日** | 2026-02-12 |
| **作成者** | Devin AI |
| **ステータス** | `draft` |
| **関連ドキュメント** | EP001_auth_api.md（認証API） |

---

## API概要

### 目的

ahamo契約者向けマイページ機能のAPIを提供し、以下を実現する：

- ダッシュボード情報の一括取得（契約・データ使用量・請求・端末・通知）
- 契約情報の詳細取得
- データ使用量の取得（日別・月別推移、チャージ履歴含む）
- 請求・支払い情報の取得（請求履歴、支払い方法含む）
- アカウント設定の取得・更新（連絡先・パスワード・通知設定）
- プラン変更・オプション管理

### スコープ

**対象範囲**:
- ダッシュボード情報取得
- 契約情報取得
- データ使用量取得
- 請求・支払い情報取得
- アカウント設定取得・更新
- プラン変更
- オプション追加・解約

**対象外範囲**:
- ユーザー登録（認証APIで管理）
- パスワードリセット（認証APIで管理）
- 管理者機能

### ベースURL

| 環境 | URL | 用途 |
|------|-----|------|
| **開発環境（MSW）** | `http://localhost:3000` | MSWモック開発 |
| **開発環境** | `http://localhost:3001` | ローカルバックエンド |
| **ステージング** | `https://staging-api.ahamo-ec.example.com` | 受入テスト |
| **本番環境** | `https://api.ahamo-ec.example.com` | 本番サービス |

---

## エンドポイント一覧

### 命名規則

```yaml
naming_convention:
  - RESTful な命名（リソース名は複数形）
  - ケバブケース使用: /api/v1/account/data-usage
  - 動詞は HTTP メソッドで表現（URL には含めない）
  - 全エンドポイントで認証必須（Bearer Token）
```

---

### 1. アカウント管理

#### 1.1 ダッシュボード情報取得

```yaml
endpoint: GET /api/v1/account/dashboard
summary: "マイページダッシュボードに表示する全情報を一括取得"
description: |
  契約情報サマリー、データ使用量、請求予定額、端末情報、
  通知一覧、支払い方法を一括で取得する。

request:
  headers:
    - "Authorization: Bearer {access_token}"
    - "Accept: application/json"

response:
  success:
    status_code: 200
    description: "ダッシュボード情報取得成功"
    body_schema:
      - contract: object
        description: "契約情報"
        fields:
          - contractId: string
            description: "契約ID"
            example: "CT-2024-001"
          - contractor: object
            description: "契約者情報"
            fields:
              - lastName: string
                example: "山田"
              - firstName: string
                example: "太郎"
              - lastNameKana: string
                example: "ヤマダ"
              - firstNameKana: string
                example: "タロウ"
              - dateOfBirth: string (date)
                example: "1990-05-15"
              - postalCode: string
                example: "150-0001"
              - address: string
                example: "東京都渋谷区神宮前1-2-3"
              - phoneNumber: string
                example: "090-1234-5678"
              - email: string
                example: "test@docomo.ne.jp"
          - details: object
            description: "契約内容"
            fields:
              - phoneNumber: string
              - contractDate: string (date)
              - planName: string
                example: "ahamo"
              - dataCapacity: number
                description: "データ容量（GB）"
                example: 20
              - monthlyBasicFee: number
                description: "月額基本料金（税込）"
                example: 2970
              - options: array[OptionService]
                description: "契約中オプション一覧"

      - dataUsage: object
        description: "データ使用量"
        fields:
          - usedData: number
            description: "使用済みデータ量（GB）"
            example: 12.5
          - totalData: number
            description: "合計データ容量（GB）"
            example: 100
          - remainingData: number
            description: "残りデータ量（GB）"
            example: 87.5
          - usagePercentage: number
            description: "使用率（%）"
            example: 12.5
          - updatedAt: string (datetime)
            description: "更新日時（ISO 8601）"
          - billingPeriodStart: string (date)
          - billingPeriodEnd: string (date)

      - billing: object
        description: "請求情報"
        fields:
          - currentMonthTotal: number
            description: "今月の請求予定額（税込）"
            example: 6050
          - basicFee: number
          - callCharge: number
          - optionFee: number
          - otherCharges: number
          - discount: number
          - previousMonthTotal: number
          - monthOverMonthDiff: number
            description: "前月比（円）"
          - details: array[BillingDetail]
          - billingPeriodStart: string (date)
          - billingPeriodEnd: string (date)

      - device: object
        description: "契約中端末情報"
        fields:
          - name: string
            example: "iPhone 16 Pro Max"
          - imageUrl: string
          - purchaseDate: string (date)
          - paymentStatus: string
            example: "分割払い中"
          - remainingBalance: number (nullable)
          - remainingInstallments: number (nullable)

      - notifications: object
        description: "通知情報"
        fields:
          - unreadCount: number
          - items: array[Notification]

      - paymentMethod: object
        description: "支払い方法"
        fields:
          - id: string
          - type: string
            enum: ["credit_card", "bank_transfer"]
          - lastFourDigits: string (nullable)
          - cardBrand: string (nullable)
          - expiryDate: string (nullable)
          - bankName: string (nullable)
          - accountType: string (nullable)
          - accountLastFourDigits: string (nullable)
          - isDefault: boolean

  error:
    - status_code: 401
      error_code: "UNAUTHORIZED"
      description: "認証トークンが無効または期限切れ"
    - status_code: 500
      error_code: "INTERNAL_ERROR"
      description: "サーバーエラー"

authentication: "必須（Bearer Token）"
rate_limit: "60リクエスト/分/ユーザー"
```

---

#### 1.2 契約情報取得

```yaml
endpoint: GET /api/v1/account/contract
summary: "契約者情報と契約内容の詳細を取得"
description: |
  契約者の個人情報、契約プラン詳細、契約中オプション一覧を取得する。

request:
  headers:
    - "Authorization: Bearer {access_token}"

response:
  success:
    status_code: 200
    description: "契約情報取得成功"
    body_schema:
      - contractId: string
      - contractor: object (ContractorInfo)
      - details: object (ContractDetails)

  error:
    - status_code: 401
      error_code: "UNAUTHORIZED"
    - status_code: 404
      error_code: "CONTRACT_NOT_FOUND"
      description: "契約情報が見つからない"

authentication: "必須（Bearer Token）"
rate_limit: "60リクエスト/分/ユーザー"
```

---

#### 1.3 データ使用量取得

```yaml
endpoint: GET /api/v1/account/data-usage
summary: "今月のデータ使用量、使用量推移、チャージ履歴を取得"
description: |
  現在のデータ使用状況、日別・月別の使用量推移、
  データチャージ履歴を取得する。

request:
  headers:
    - "Authorization: Bearer {access_token}"

response:
  success:
    status_code: 200
    description: "データ使用量取得成功"
    body_schema:
      - current: object
        description: "今月の使用状況"
        fields:
          - usedData: number
          - totalData: number
          - remainingData: number
          - usagePercentage: number
          - updatedAt: string (datetime)
          - billingPeriodStart: string (date)
          - billingPeriodEnd: string (date)

      - history: object
        description: "使用量推移"
        fields:
          - daily: array[DailyUsage]
            description: "日別使用量（過去30日）"
            item_fields:
              - date: string (date)
              - usage: number (GB)
          - monthly: array[MonthlyUsage]
            description: "月別使用量（過去6ヶ月）"
            item_fields:
              - month: string (YYYY-MM)
              - usage: number (GB)
              - limit: number (GB)

      - charges: array[DataCharge]
        description: "データチャージ履歴"
        item_fields:
          - id: string
          - chargedAt: string (datetime)
          - amount: number (GB)
          - fee: number (円)
          - expiresAt: string (datetime)

  error:
    - status_code: 401
      error_code: "UNAUTHORIZED"
    - status_code: 500
      error_code: "INTERNAL_ERROR"

authentication: "必須（Bearer Token）"
rate_limit: "60リクエスト/分/ユーザー"
```

---

#### 1.4 請求・支払い情報取得

```yaml
endpoint: GET /api/v1/account/billing
summary: "請求予定額、請求履歴、支払い方法を取得"
description: |
  今月の請求予定額の内訳、過去の請求履歴、
  登録済み支払い方法を取得する。

request:
  headers:
    - "Authorization: Bearer {access_token}"

response:
  success:
    status_code: 200
    description: "請求情報取得成功"
    body_schema:
      - current: object (BillingInfo)
        description: "今月の請求情報"

      - history: array[BillingHistory]
        description: "過去の請求履歴"
        item_fields:
          - id: string
          - billingMonth: string (YYYY年MM月)
          - amount: number
          - status: string
            enum: ["paid", "pending", "overdue"]
          - paidAt: string (datetime, nullable)
          - downloadUrl: string (nullable)

      - paymentMethod: object (PaymentMethod)
        description: "登録済み支払い方法"

  error:
    - status_code: 401
      error_code: "UNAUTHORIZED"
    - status_code: 500
      error_code: "INTERNAL_ERROR"

authentication: "必須（Bearer Token）"
rate_limit: "60リクエスト/分/ユーザー"
```

---

#### 1.5 アカウント設定取得

```yaml
endpoint: GET /api/v1/account/settings
summary: "アカウント設定情報を取得"
description: |
  連絡先情報（メール、電話番号）と通知設定を取得する。

request:
  headers:
    - "Authorization: Bearer {access_token}"

response:
  success:
    status_code: 200
    description: "設定情報取得成功"
    body_schema:
      - email: string
        example: "test@docomo.ne.jp"
      - phoneNumber: string
        example: "090-1234-5678"
      - notificationPreferences: object
        fields:
          - email: boolean
          - sms: boolean
          - push: boolean

  error:
    - status_code: 401
      error_code: "UNAUTHORIZED"

authentication: "必須（Bearer Token）"
rate_limit: "60リクエスト/分/ユーザー"
```

---

#### 1.6 アカウント設定更新

```yaml
endpoint: PATCH /api/v1/account/settings
summary: "アカウント設定を部分更新"
description: |
  連絡先情報、パスワード、通知設定を部分的に更新する。
  リクエストボディに含まれるフィールドのみ更新。

request:
  content_type: "application/json"
  headers:
    - "Authorization: Bearer {access_token}"
    - "Content-Type: application/json"

  body_schema:
    optional_fields:
      - email: string
        description: "新しいメールアドレス"
        format: "email"
      - phoneNumber: string
        description: "新しい電話番号"
      - password: object
        description: "パスワード変更"
        fields:
          - current: string
            description: "現在のパスワード"
          - newPassword: string
            description: "新しいパスワード"
            min_length: 8
      - notificationPreferences: object
        description: "通知設定"
        fields:
          - email: boolean
          - sms: boolean
          - push: boolean

response:
  success:
    status_code: 200
    description: "設定更新成功"
    body_schema:
      - email: string
      - phoneNumber: string
      - notificationPreferences: object

  error:
    - status_code: 400
      error_code: "VALIDATION_ERROR"
      description: "バリデーションエラー"
    - status_code: 401
      error_code: "UNAUTHORIZED"
    - status_code: 422
      error_code: "INVALID_PASSWORD"
      description: "現在のパスワードが不正"

authentication: "必須（Bearer Token）"
rate_limit: "10リクエスト/分/ユーザー"
```

---

### 2. プラン・オプション管理

#### 2.1 利用可能プラン一覧取得

```yaml
endpoint: GET /api/v1/account/plans
summary: "変更可能なプラン一覧を取得"

request:
  headers:
    - "Authorization: Bearer {access_token}"

response:
  success:
    status_code: 200
    body_schema:
      - array[PlanInfo]
        item_fields:
          - id: string
          - name: string
            example: "ahamo"
          - dataCapacity: number (GB)
          - monthlyFee: number (円)
          - description: string
          - features: array[string]

authentication: "必須（Bearer Token）"
rate_limit: "30リクエスト/分/ユーザー"
```

---

#### 2.2 プラン変更

```yaml
endpoint: POST /api/v1/account/plans/change
summary: "プランを変更する"

request:
  content_type: "application/json"
  headers:
    - "Authorization: Bearer {access_token}"
  body_schema:
    required_fields:
      - planId: string
        description: "変更先プランID"
      - effectiveDate: string
        enum: ["next_billing", "immediately"]
        description: "適用時期"

response:
  success:
    status_code: 200
    body_schema:
      - success: boolean
      - message: string
        example: "プラン変更を受け付けました。次回請求日から適用されます。"
      - effectiveDate: string (date)

  error:
    - status_code: 400
      error_code: "INVALID_PLAN"
    - status_code: 401
      error_code: "UNAUTHORIZED"
    - status_code: 409
      error_code: "PLAN_CHANGE_IN_PROGRESS"
      description: "既にプラン変更申請中"

authentication: "必須（Bearer Token）"
rate_limit: "5リクエスト/分/ユーザー"
```

---

#### 2.3 利用可能オプション一覧取得

```yaml
endpoint: GET /api/v1/account/options/available
summary: "追加可能なオプション一覧を取得"

request:
  headers:
    - "Authorization: Bearer {access_token}"

response:
  success:
    status_code: 200
    body_schema:
      - array[OptionInfo]
        item_fields:
          - id: string
          - name: string
          - monthlyFee: number
          - description: string

authentication: "必須（Bearer Token）"
rate_limit: "30リクエスト/分/ユーザー"
```

---

#### 2.4 オプション追加

```yaml
endpoint: POST /api/v1/account/options/{option_id}
summary: "オプションサービスを追加する"

path_parameters:
  - option_id: string
    description: "追加するオプションID"

request:
  headers:
    - "Authorization: Bearer {access_token}"

response:
  success:
    status_code: 200
    body_schema:
      - success: boolean
      - message: string
        example: "オプションを追加しました"

  error:
    - status_code: 400
      error_code: "ALREADY_SUBSCRIBED"
    - status_code: 401
      error_code: "UNAUTHORIZED"
    - status_code: 404
      error_code: "OPTION_NOT_FOUND"

authentication: "必須（Bearer Token）"
rate_limit: "10リクエスト/分/ユーザー"
```

---

#### 2.5 オプション解約

```yaml
endpoint: DELETE /api/v1/account/options/{option_id}
summary: "オプションサービスを解約する"

path_parameters:
  - option_id: string
    description: "解約するオプションID"

request:
  headers:
    - "Authorization: Bearer {access_token}"

response:
  success:
    status_code: 200
    body_schema:
      - success: boolean
      - message: string
        example: "オプションを解約しました"

  error:
    - status_code: 401
      error_code: "UNAUTHORIZED"
    - status_code: 404
      error_code: "OPTION_NOT_FOUND"
    - status_code: 409
      error_code: "CANCELLATION_IN_PROGRESS"

authentication: "必須（Bearer Token）"
rate_limit: "10リクエスト/分/ユーザー"
```

---

### 3. エンドポイント概要一覧

| メソッド | エンドポイント | 概要 | 認証 |
|----------|--------------|------|------|
| GET | /api/v1/account/dashboard | ダッシュボード情報一括取得 | 必須 |
| GET | /api/v1/account/contract | 契約情報詳細取得 | 必須 |
| GET | /api/v1/account/data-usage | データ使用量取得 | 必須 |
| GET | /api/v1/account/billing | 請求・支払い情報取得 | 必須 |
| GET | /api/v1/account/settings | アカウント設定取得 | 必須 |
| PATCH | /api/v1/account/settings | アカウント設定更新 | 必須 |
| GET | /api/v1/account/plans | 利用可能プラン一覧取得 | 必須 |
| POST | /api/v1/account/plans/change | プラン変更 | 必須 |
| GET | /api/v1/account/options/available | 利用可能オプション一覧取得 | 必須 |
| POST | /api/v1/account/options/{option_id} | オプション追加 | 必須 |
| DELETE | /api/v1/account/options/{option_id} | オプション解約 | 必須 |

**詳細**: Phase 5で完全版API仕様書（OpenAPI 3.0）に記載

---

## 認証・認可

### 認証方式

```yaml
mechanism:
  type: "JWT (JSON Web Token)"
  format: "Bearer Token"

token_usage:
  header: "Authorization: Bearer {access_token}"
  
token_management:
  - 認証APIで取得したアクセストークンを使用
  - トークン期限切れ時はリフレッシュトークンで更新
  - 全エンドポイントで認証必須
```

### 認可ルール

```yaml
authorization:
  - 自分自身のアカウント情報のみアクセス可能
  - トークンに含まれるユーザーIDで対象を特定
  - 他ユーザーの情報へのアクセスは403 Forbidden
```

---

## 共通型定義

### OptionService

```yaml
fields:
  - id: string
  - name: string
  - monthlyFee: number
  - description: string
  - startDate: string (date)
  - status: string
    enum: ["active", "pending"]
```

### Notification

```yaml
fields:
  - id: string
  - title: string
  - body: string
  - type: string
    enum: ["info", "important", "campaign", "warning"]
  - isRead: boolean
  - createdAt: string (datetime)
  - linkUrl: string (nullable)
```

### BillingDetail

```yaml
fields:
  - label: string
  - amount: number
```

---

## エラーレスポンス形式

```yaml
error_format:
  standard:
    status: "error"
    message: string
    
  validation:
    status: "error"
    message: string
    details: array[ValidationError]
    
  validation_error:
    field: string
    message: string

http_status_codes:
  - 200: 成功
  - 400: バリデーションエラー
  - 401: 認証エラー
  - 403: 認可エラー
  - 404: リソース未検出
  - 409: 競合（重複操作等）
  - 422: 処理不可能
  - 429: レート制限超過
  - 500: サーバーエラー
```

---

## 非機能要件

### パフォーマンス

```yaml
performance:
  response_time:
    target: "200ms以内（p95）"
    max: "1000ms"
  throughput:
    target: "1000リクエスト/秒"
```

### キャッシュ戦略

```yaml
caching:
  dashboard:
    cache_control: "private, max-age=300"
    description: "5分間キャッシュ（フロントエンドのstaleTimeと同期）"
  contract:
    cache_control: "private, max-age=3600"
    description: "契約情報は変更頻度が低いため1時間キャッシュ"
  data_usage:
    cache_control: "private, max-age=300"
    description: "5分間キャッシュ"
  billing:
    cache_control: "private, max-age=300"
    description: "5分間キャッシュ"
```

---

## 変更履歴

| 日付 | バージョン | 変更内容 | 変更者 |
|------|-----------|---------|--------|
| 2026-02-12 | 1.0.0 | 初版作成 | Devin AI |
