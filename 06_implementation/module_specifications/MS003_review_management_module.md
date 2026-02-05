# MS003 レビュー管理モジュール仕様書

## 概要

- **PBI番号**: PBI-034-FE
- **Jiraチケット**: EC-221
- **作成日**: 2026-02-05
- **ステータス**: 完了（フェーズ1）
- **PR**: https://github.com/satoshi-watanabe-0001/ec-site-frontend-workspace/pull/5

## 要件一覧

### 1. 機能要件

| 要件ID | 要件内容 | 優先度 | ステータス |
|--------|----------|--------|-----------|
| REQ-001 | ユーザーが投稿したレビューの一覧表示 | 高 | 完了 |
| REQ-002 | レビューのページネーション対応 | 中 | 完了 |
| REQ-003 | レビューの詳細表示 | 中 | 完了（一覧内表示） |
| REQ-004 | レビューの編集機能（将来拡張用） | 低 | プレースホルダー |
| REQ-005 | レビューの削除機能（将来拡張用） | 低 | 完了（モック） |

### 2. 技術要件

| 要件ID | 要件内容 | 優先度 | ステータス |
|--------|----------|--------|-----------|
| TECH-001 | 既存のprofile editページパターンに準拠 | 高 | 完了 |
| TECH-002 | React Query（@tanstack/react-query）によるデータ取得 | 高 | 完了 |
| TECH-003 | Zodによるバリデーションスキーマ定義 | 高 | 完了 |
| TECH-004 | TypeScript型定義の作成 | 高 | 完了 |
| TECH-005 | WCAG 2.1 Level AAアクセシビリティ対応 | 高 | 完了 |
| TECH-006 | レスポンシブデザイン（Tailwind CSS） | 高 | 完了 |

### 3. API要件（プレースホルダー）

バックエンドAPIコントラクトが未定義のため、以下のエンドポイントを想定：

| エンドポイント | メソッド | 説明 |
|---------------|---------|------|
| `/api/v1/reviews` | GET | ユーザーのレビュー一覧取得（ページネーション対応） |
| `/api/v1/reviews/{id}` | GET | 特定レビューの詳細取得 |
| `/api/v1/reviews` | POST | 新規レビュー作成 |
| `/api/v1/reviews/{id}` | PUT | レビュー更新 |
| `/api/v1/reviews/{id}` | DELETE | レビュー削除 |

## 実装計画

### フェーズ1: 基盤構築（今回実装）

1. **型定義とスキーマ**
   - `lib/types/review.types.ts` - レビュー関連の型定義
   - `lib/schemas/review.schema.ts` - Zodバリデーションスキーマ

2. **APIクライアント**
   - `lib/api/reviews.ts` - レビューAPI関数（モックデータ対応）

3. **カスタムフック**
   - `lib/hooks/useReviews.ts` - React Queryフック

4. **コンポーネント**
   - `components/reviews/ReviewList.tsx` - レビュー一覧コンテナ
   - `components/reviews/ReviewItem.tsx` - 個別レビュー表示

5. **ページ**
   - `app/mypage/reviews/page.tsx` - メインページ
   - `app/mypage/reviews/loading.tsx` - ローディング状態

### フェーズ2: 機能拡張（将来）

- 編集・削除機能の実装
- フィルタリング機能
- ソート機能

## 成果物一覧

| 成果物 | パス | ステータス |
|--------|------|-----------|
| 型定義 | `apps/customer-app/lib/types/review.types.ts` | 完了 |
| スキーマ | `apps/customer-app/lib/schemas/review.schema.ts` | 完了 |
| APIクライアント | `apps/customer-app/lib/api/reviews.ts` | 完了 |
| カスタムフック | `apps/customer-app/lib/hooks/useReviews.ts` | 完了 |
| ReviewListコンポーネント | `apps/customer-app/components/reviews/ReviewList.tsx` | 完了 |
| ReviewItemコンポーネント | `apps/customer-app/components/reviews/ReviewItem.tsx` | 完了 |
| レビュー管理ページ | `apps/customer-app/app/mypage/reviews/page.tsx` | 完了 |
| ローディングページ | `apps/customer-app/app/mypage/reviews/loading.tsx` | 完了 |

## 参考実装

- プロフィール編集ページ: `apps/customer-app/app/profile/edit/page.tsx`
- プロフィールAPI: `apps/customer-app/lib/api/profile.ts`
- プロフィールフック: `apps/customer-app/lib/hooks/useProfileEdit.ts`
- プロフィールスキーマ: `apps/customer-app/lib/schemas/profile.schema.ts`
