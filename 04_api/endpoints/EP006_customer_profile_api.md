# EP006: 顧客プロファイルAPI設計書

## チケット情報
- **チケットID**: EC-17
- **タイトル**: 会員情報編集API実装
- **作成日**: 2025-11-07
- **ステータス**: 設計中

## 概要

顧客（会員）の基本情報を更新するためのREST APIエンドポイントを提供します。
本APIは、会員自身が自分の情報を編集する場合と、管理者が会員情報を編集する場合の両方をサポートします。

## 設計上の決定事項

### 1. リポジトリ選択
- **決定**: `ec-site-customer-service`リポジトリを拡張
- **理由**: チケットで言及されている`ec-site-user-service`は存在せず、システムアーキテクチャによるとCustomer Service（ポート8089）が顧客管理を担当するため

### 2. ポート設定
- **決定**: ポート8089を使用
- **理由**: システムアーキテクチャでCustomer Serviceはポート8089で稼働すると定義されている

### 3. URL設計
- **決定**: 日本語URLと英語URLの両方を提供
  - 日本語: `PUT /api/v1/会員情報編集/{id}`（チケット仕様通り）
  - 英語: `PUT /api/v1/customers/{id}/profile`（RESTful標準）
- **理由**: チケット仕様との互換性を保ちつつ、RESTful APIの標準的な命名規則にも準拠

### 4. リクエストフィールド
- **決定**: 包括的なProfileUpdateRequestを実装
  - 基本フィールド: name, status
  - 拡張フィールド: firstName, lastName, addresses, phoneNumbers
- **理由**: ユーザーストーリーで「氏名、住所、電話番号」が言及されているため、将来の拡張性も考慮

## エンドポイント仕様

### 1. 会員情報更新API

#### エンドポイント
```
PUT /api/v1/会員情報編集/{id}
PUT /api/v1/customers/{id}/profile
```

#### 認証・認可
- **認証**: JWT Bearer Token必須
- **認可**: 
  - 一般ユーザー: 自分自身のプロファイルのみ編集可能
  - 管理者（ADMIN）: 全ての会員のプロファイルを編集可能
- **実装**: `@PreAuthorize("hasRole('USER') and #id == principal.id or hasRole('ADMIN')")`

#### リクエスト

**パスパラメータ**
| パラメータ名 | 型 | 必須 | 説明 |
|------------|-----|------|------|
| id | String | ○ | 顧客ID（UUID形式） |

**ヘッダー**
| ヘッダー名 | 値 | 必須 | 説明 |
|-----------|-----|------|------|
| Authorization | Bearer {token} | ○ | JWT認証トークン |
| Content-Type | application/json | ○ | リクエストボディの形式 |

**リクエストボディ**
```json
{
  "name": "山田太郎",
  "status": "active",
  "firstName": "太郎",
  "lastName": "山田",
  "addresses": [
    {
      "type": "home",
      "postalCode": "100-0001",
      "prefecture": "東京都",
      "city": "千代田区",
      "addressLine1": "千代田1-1-1",
      "addressLine2": "マンション101号室",
      "isDefault": true
    }
  ],
  "phoneNumbers": [
    {
      "type": "mobile",
      "number": "090-1234-5678",
      "isDefault": true
    }
  ]
}
```

**フィールド定義**
| フィールド名 | 型 | 必須 | バリデーション | 説明 |
|------------|-----|------|--------------|------|
| name | String | ○ | @NotBlank, @Size(max=100) | 表示名 |
| status | String | × | @Pattern(regexp="^(active\|inactive)$") | ステータス（active/inactive） |
| firstName | String | × | @Size(max=50) | 名 |
| lastName | String | × | @Size(max=50) | 姓 |
| addresses | Array | × | @Valid | 住所リスト |
| phoneNumbers | Array | × | @Valid | 電話番号リスト |

**Address オブジェクト**
| フィールド名 | 型 | 必須 | バリデーション | 説明 |
|------------|-----|------|--------------|------|
| type | String | ○ | @NotBlank, @Pattern(regexp="^(home\|work\|other)$") | 住所タイプ |
| postalCode | String | ○ | @NotBlank, @Pattern(regexp="^\\d{3}-\\d{4}$") | 郵便番号 |
| prefecture | String | ○ | @NotBlank, @Size(max=10) | 都道府県 |
| city | String | ○ | @NotBlank, @Size(max=50) | 市区町村 |
| addressLine1 | String | ○ | @NotBlank, @Size(max=100) | 住所1 |
| addressLine2 | String | × | @Size(max=100) | 住所2（建物名等） |
| isDefault | Boolean | × | - | デフォルト住所フラグ |

**PhoneNumber オブジェクト**
| フィールド名 | 型 | 必須 | バリデーション | 説明 |
|------------|-----|------|--------------|------|
| type | String | ○ | @NotBlank, @Pattern(regexp="^(mobile\|home\|work)$") | 電話番号タイプ |
| number | String | ○ | @NotBlank, @Pattern(regexp="^\\d{2,4}-\\d{2,4}-\\d{4}$") | 電話番号 |
| isDefault | Boolean | × | - | デフォルト電話番号フラグ |

#### レスポンス

**成功時（200 OK）**
```json
{
  "status": "success",
  "message": "会員情報編集が正常に更新されました",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "山田太郎",
    "updatedAt": "2025-11-07T11:00:00Z"
  }
}
```

**バリデーションエラー（400 Bad Request）**
```json
{
  "status": "error",
  "message": "入力値が不正です",
  "errors": [
    {
      "field": "name",
      "message": "名前は必須です"
    },
    {
      "field": "addresses[0].postalCode",
      "message": "郵便番号の形式が正しくありません"
    }
  ]
}
```

**認証エラー（401 Unauthorized）**
```json
{
  "status": "error",
  "message": "認証が必要です",
  "code": "AUTHENTICATION_REQUIRED"
}
```

**認可エラー（403 Forbidden）**
```json
{
  "status": "error",
  "message": "この操作を実行する権限がありません",
  "code": "INSUFFICIENT_PERMISSIONS"
}
```

**顧客が見つからない（404 Not Found）**
```json
{
  "status": "error",
  "message": "指定された顧客が見つかりません",
  "code": "CUSTOMER_NOT_FOUND"
}
```

**サーバーエラー（500 Internal Server Error）**
```json
{
  "status": "error",
  "message": "サーバー内部エラーが発生しました",
  "code": "INTERNAL_SERVER_ERROR"
}
```

## データモデル設計

### データベーススキーマ

#### customers テーブル
```sql
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100),
    updated_by VARCHAR(100),
    version INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_customers_status ON customers(status);
CREATE INDEX idx_customers_updated_at ON customers(updated_at);
```

#### customer_profiles テーブル
```sql
CREATE TABLE customer_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(customer_id)
);

CREATE INDEX idx_customer_profiles_customer_id ON customer_profiles(customer_id);
```

#### customer_addresses テーブル
```sql
CREATE TABLE customer_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    type VARCHAR(20) NOT NULL,
    postal_code VARCHAR(10) NOT NULL,
    prefecture VARCHAR(10) NOT NULL,
    city VARCHAR(50) NOT NULL,
    address_line1 VARCHAR(100) NOT NULL,
    address_line2 VARCHAR(100),
    is_default BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_customer_addresses_customer_id ON customer_addresses(customer_id);
CREATE INDEX idx_customer_addresses_is_default ON customer_addresses(customer_id, is_default);
```

#### customer_phone_numbers テーブル
```sql
CREATE TABLE customer_phone_numbers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    type VARCHAR(20) NOT NULL,
    number VARCHAR(20) NOT NULL,
    is_default BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_customer_phone_numbers_customer_id ON customer_phone_numbers(customer_id);
CREATE INDEX idx_customer_phone_numbers_is_default ON customer_phone_numbers(customer_id, is_default);
```

## アーキテクチャ設計

### レイヤー構成

```
┌─────────────────────────────────────────┐
│         Controller Layer                │
│  CustomerProfileController              │
│  - リクエスト受付                        │
│  - バリデーション                        │
│  - レスポンス生成                        │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│          Service Layer                  │
│  CustomerProfileService                 │
│  - ビジネスロジック                      │
│  - トランザクション管理                  │
│  - 認可チェック                          │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│        Repository Layer                 │
│  CustomerRepository                     │
│  CustomerProfileRepository              │
│  CustomerAddressRepository              │
│  CustomerPhoneNumberRepository          │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         Database Layer                  │
│  PostgreSQL (customer_schema)           │
└─────────────────────────────────────────┘
```

### クラス設計

#### Controller
- `CustomerProfileController`: プロファイル更新エンドポイントを提供

#### Service
- `CustomerProfileService`: プロファイル更新のビジネスロジックを実装

#### Repository
- `CustomerRepository`: 顧客エンティティのCRUD操作
- `CustomerProfileRepository`: プロファイルエンティティのCRUD操作
- `CustomerAddressRepository`: 住所エンティティのCRUD操作
- `CustomerPhoneNumberRepository`: 電話番号エンティティのCRUD操作

#### DTO
- `ProfileUpdateRequest`: リクエストDTO
- `ProfileUpdateResponse`: レスポンスDTO
- `AddressUpdateRequest`: 住所更新DTO
- `PhoneNumberUpdateRequest`: 電話番号更新DTO

#### Entity
- `Customer`: 顧客エンティティ
- `CustomerProfile`: プロファイルエンティティ
- `CustomerAddress`: 住所エンティティ
- `CustomerPhoneNumber`: 電話番号エンティティ

#### Exception
- `CustomerNotFoundException`: 顧客が見つからない例外
- `UnauthorizedCustomerAccessException`: 認可エラー例外

## セキュリティ設計

### 認証
- JWT Bearer Tokenによる認証
- Auth Serviceで発行されたトークンを検証

### 認可
- Spring Securityの`@PreAuthorize`アノテーションを使用
- ユーザーは自分自身のプロファイルのみ編集可能
- 管理者（ADMIN）は全てのプロファイルを編集可能

### データ保護
- パスワードは含まない（Auth Serviceで管理）
- 個人情報は暗号化して保存（必要に応じて）
- 監査ログの記録（created_by, updated_by, updated_at）

## ロギング設計

### ログレベル
- **INFO**: 正常な処理の開始・完了
- **WARN**: 顧客が見つからない等の業務エラー
- **ERROR**: システムエラー、予期しない例外

### ログ出力例
```java
log.info("Customer profile update started - customerId: {}, userId: {}", customerId, currentUserId);
log.info("Customer profile updated successfully - customerId: {}, processingTime: {}ms", customerId, processingTime);
log.warn("Customer not found - customerId: {}", customerId);
log.error("Failed to update customer profile - customerId: {}", customerId, exception);
```

### 構造化ログ
```json
{
  "timestamp": "2025-11-07T11:00:00Z",
  "level": "INFO",
  "service": "customer-service",
  "traceId": "550e8400-e29b-41d4-a716-446655440000",
  "message": "Customer profile updated successfully",
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "660e8400-e29b-41d4-a716-446655440001",
  "processingTime": 150
}
```

## パフォーマンス要件

### レスポンスタイム
- **目標**: 500ms以内
- **最大**: 1000ms

### スループット
- **目標**: 100 req/sec
- **最大**: 500 req/sec

### 最適化戦略
- データベースインデックスの適切な設定
- N+1問題の回避（JPA Fetch戦略の最適化）
- 楽観的ロックによる同時更新制御

## エラーハンドリング

### エラーコード一覧
| コード | HTTPステータス | 説明 |
|-------|--------------|------|
| AUTHENTICATION_REQUIRED | 401 | 認証が必要 |
| INSUFFICIENT_PERMISSIONS | 403 | 権限不足 |
| CUSTOMER_NOT_FOUND | 404 | 顧客が見つからない |
| VALIDATION_ERROR | 400 | バリデーションエラー |
| CONCURRENT_UPDATE_ERROR | 409 | 同時更新エラー |
| INTERNAL_SERVER_ERROR | 500 | サーバー内部エラー |

### グローバル例外ハンドラー
- `@RestControllerAdvice`を使用
- 全ての例外を統一フォーマットで返却
- スタックトレースは本番環境では非表示

## テスト戦略

### 単体テスト
- Controller層: MockMvcを使用したAPIテスト
- Service層: Mockitoを使用したビジネスロジックテスト
- Repository層: @DataJpaTestを使用したデータアクセステスト

### 統合テスト
- @SpringBootTestを使用したエンドツーエンドテスト
- TestContainersを使用した実データベーステスト

### テストカバレッジ
- **目標**: 80%以上
- **最小**: 70%以上

## API Gateway連携

### ルーティング設定
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: customer-profile-update
          uri: lb://customer-service
          predicates:
            - Path=/api/v1/会員情報編集/**,/api/v1/customers/*/profile
            - Method=PUT
          filters:
            - StripPrefix=0
```

## 運用設計

### モニタリング
- Prometheusメトリクスの公開
- レスポンスタイムの監視
- エラー率の監視

### アラート設定
- エラー率が5%を超えた場合
- レスポンスタイムが1000msを超えた場合
- データベース接続エラーが発生した場合

## 今後の拡張予定

### Phase 2
- プロファイル画像のアップロード機能
- メールアドレスの変更機能（確認メール送信）

### Phase 3
- プロファイル変更履歴の記録
- 変更通知機能

## 参考資料

- [システムアーキテクチャ設計書](../02_architecture/system_architecture.md)
- [organization-standards](https://github.com/satoshi-watanabe-0001/organization-standards)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/)
