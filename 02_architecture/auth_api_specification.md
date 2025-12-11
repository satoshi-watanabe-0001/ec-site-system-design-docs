# 認証API仕様書

## 概要

本ドキュメントは、EC-274で実装されたログインAPIの仕様を定義する。

## エンドポイント

### POST /api/v1/auth/login

ユーザー認証を行い、JWTトークンを発行する。

#### リクエスト

**Content-Type**: `application/json`

**リクエストボディ**:

| フィールド | 型 | 必須 | 説明 | バリデーション |
|-----------|-----|------|------|---------------|
| email | string | はい | メールアドレス | 有効なメール形式 |
| password | string | はい | パスワード | 6〜100文字 |

**リクエスト例**:

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

#### レスポンス

**成功時 (200 OK)**:

| フィールド | 型 | 説明 |
|-----------|-----|------|
| access_token | string | JWTアクセストークン |
| refresh_token | string | リフレッシュトークン |
| token_type | string | トークンタイプ（"Bearer"） |
| expires_in | number | トークン有効期限（秒） |
| user | object | ユーザー情報 |
| user.id | string | ユーザーID |
| user.name | string | ユーザー名 |
| user.email | string | メールアドレス |

**レスポンス例**:

```json
{
  "access_token": "eyJhbGciOiJIUzUxMiJ9...",
  "refresh_token": "eyJhbGciOiJIUzUxMiJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "id": "1",
    "name": "山田太郎",
    "email": "user@example.com"
  }
}
```

#### エラーレスポンス

**バリデーションエラー (400 Bad Request)**:

入力値が不正な場合に返却される。

```json
{
  "success": false,
  "error_code": "VALIDATION_ERROR",
  "message": "入力値が不正です",
  "field_errors": {
    "email": "有効なメールアドレス形式で入力してください",
    "password": "パスワードは6文字以上100文字以下で入力してください"
  },
  "timestamp": "2025-12-10T07:00:00.000Z",
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**認証エラー (401 Unauthorized)**:

認証情報が無効な場合に返却される。

```json
{
  "success": false,
  "error_code": "AUTHENTICATION_FAILED",
  "message": "認証情報が無効です",
  "timestamp": "2025-12-10T07:00:00.000Z",
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**サーバーエラー (500 Internal Server Error)**:

予期しないエラーが発生した場合に返却される。

```json
{
  "success": false,
  "error_code": "INTERNAL_SERVER_ERROR",
  "message": "サーバー内部エラーが発生しました",
  "timestamp": "2025-12-10T07:00:00.000Z",
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

## セキュリティ

### 認証方式

- JWTトークンベースの認証を使用
- アクセストークンの有効期限: 1時間
- リフレッシュトークンの有効期限: 24時間

### パスワード

- BCryptによるハッシュ化
- 最小6文字、最大100文字

### トークン使用方法

認証が必要なエンドポイントにアクセスする際は、HTTPヘッダーにトークンを含める。

```
Authorization: Bearer <access_token>
```

## フロントエンド連携

本APIは、フロントエンドの`useAuthStore`と連携するように設計されている。

レスポンスの`user`オブジェクトは、フロントエンドの`User`型と互換性がある。

```typescript
interface User {
  id: string
  name: string
  email: string
}
```

## 関連チケット

- JIRA: EC-274
