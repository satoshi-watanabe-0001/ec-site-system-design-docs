---
title: "ADR-001: ECサイト技術スタック選定"
status: "Accepted"
date: "2025-12-14"
decision_makers: ["アーキテクトチーム", "開発リード"]
template_type: "Architecture Decision Record"
---

# ADR-001: ECサイト技術スタック選定

## ステータス

**Accepted** - 2025-12-14

## コンテキスト

ECサイトのバックエンドおよびフロントエンドの技術スタックを選定する必要があります。以下の要件を満たす技術選定が求められています。

### ビジネス要件

1. **スケーラビリティ**: 将来的なトラフィック増加に対応できること
2. **保守性**: 長期的なメンテナンスが容易であること
3. **開発効率**: 迅速な機能開発が可能であること
4. **セキュリティ**: ECサイトとして必要なセキュリティ要件を満たすこと

### 技術要件

1. **マイクロサービスアーキテクチャ**: 独立したサービスとしてデプロイ可能
2. **RESTful API**: 標準的なAPI設計
3. **JWT認証**: ステートレスな認証機構
4. **レスポンシブUI**: モバイルファーストのUI設計

### チーム状況

- バックエンド開発者: Java/Spring Boot経験者が多数
- フロントエンド開発者: React/TypeScript経験者が多数
- DevOps: Docker/Kubernetes運用経験あり

---

## 決定

### バックエンド技術スタック

| 技術 | バージョン | 選定理由 |
|------|-----------|---------|
| **Java** | 17 (LTS) | 長期サポート、パフォーマンス向上、チーム習熟度 |
| **Spring Boot** | 3.2.0 | エンタープライズ標準、豊富なエコシステム |
| **Spring Security** | 6.x | 堅牢なセキュリティフレームワーク |
| **Spring Data JPA** | 3.x | データアクセスの抽象化 |
| **PostgreSQL** | 15 | 信頼性、拡張性、JSON対応 |
| **jjwt** | 0.11.5 | JWT処理の標準ライブラリ |
| **Flyway** | 9.x | データベースマイグレーション管理 |
| **Lombok** | 1.18.x | ボイラープレートコード削減 |

### フロントエンド技術スタック

| 技術 | バージョン | 選定理由 |
|------|-----------|---------|
| **Next.js** | 15 | App Router、サーバーコンポーネント、SEO最適化 |
| **React** | 19 | 最新機能、パフォーマンス向上 |
| **TypeScript** | 5.x | 型安全性、開発効率向上 |
| **Tailwind CSS** | v4 | ユーティリティファースト、高速開発 |
| **Zustand** | 4.x | 軽量な状態管理 |
| **TanStack Query** | 5.x | サーバー状態管理、キャッシング |
| **React Hook Form** | 7.x | フォーム管理 |
| **Zod** | 3.x | スキーマバリデーション |

### 認証方式

| 項目 | 選定内容 | 理由 |
|------|---------|------|
| **認証方式** | JWT (JSON Web Token) | ステートレス、スケーラブル |
| **署名アルゴリズム** | HMAC-SHA256 | セキュリティと性能のバランス |
| **パスワードハッシュ** | BCrypt | 業界標準、適応的ハッシュ |

---

## 検討した代替案

### バックエンド代替案

#### 代替案1: Node.js + Express

**メリット**:
- フロントエンドとの言語統一
- 軽量で高速な開発

**デメリット**:
- チームのJava習熟度を活かせない
- エンタープライズ向け機能が限定的
- 型安全性の確保に追加作業が必要

**却下理由**: チームのJava経験を活かし、Spring Bootの豊富なエコシステムを活用する方が効率的と判断。

#### 代替案2: Go + Gin

**メリット**:
- 高パフォーマンス
- シンプルな言語仕様
- 軽量なバイナリ

**デメリット**:
- チームの学習コストが高い
- エンタープライズ向けフレームワークが未成熟
- ORMの選択肢が限定的

**却下理由**: 学習コストと開発速度を考慮し、チームが習熟しているJava/Spring Bootを選定。

#### 代替案3: Python + FastAPI

**メリット**:
- 高速な開発
- 自動ドキュメント生成
- 非同期処理のサポート

**デメリット**:
- チームのPython経験が限定的
- 大規模アプリケーションでの実績が少ない
- 型チェックが実行時のみ

**却下理由**: チームの習熟度とエンタープライズ実績を重視し、Spring Bootを選定。

### フロントエンド代替案

#### 代替案1: Vue.js + Nuxt.js

**メリット**:
- 学習曲線が緩やか
- 公式ツールの充実

**デメリット**:
- チームのReact経験を活かせない
- エコシステムがReactより小さい

**却下理由**: チームのReact経験を活かし、より大きなエコシステムを持つReact/Next.jsを選定。

#### 代替案2: Angular

**メリット**:
- フルスタックフレームワーク
- 強力な型システム
- 大規模アプリケーション向け

**デメリット**:
- 学習曲線が急
- バンドルサイズが大きい
- チームの経験が限定的

**却下理由**: 学習コストとチームの習熟度を考慮し、React/Next.jsを選定。

### 認証方式代替案

#### 代替案1: セッションベース認証

**メリット**:
- シンプルな実装
- サーバー側でセッション管理

**デメリット**:
- ステートフルでスケーラビリティに課題
- マイクロサービス間での共有が複雑

**却下理由**: マイクロサービスアーキテクチャとの親和性を重視し、JWTを選定。

#### 代替案2: OAuth 2.0 + OpenID Connect

**メリット**:
- 標準化されたプロトコル
- ソーシャルログイン対応

**デメリット**:
- 実装が複雑
- デモ段階では過剰な機能

**却下理由**: 現段階ではシンプルなJWT認証で十分と判断。将来的な拡張として検討。

---

## 結果

### 期待される効果

1. **開発効率の向上**: チームの習熟した技術を活用することで、迅速な開発が可能
2. **保守性の確保**: 広く使われている技術により、長期的なメンテナンスが容易
3. **スケーラビリティ**: マイクロサービスアーキテクチャとJWT認証により、水平スケーリングが可能
4. **セキュリティ**: Spring SecurityとBCryptにより、堅牢なセキュリティを実現

### リスクと対策

| リスク | 影響度 | 対策 |
|-------|--------|------|
| Spring Boot 3.xの新機能への習熟 | 中 | 公式ドキュメントの学習、チーム内勉強会 |
| Next.js 15の新機能への習熟 | 中 | 公式ドキュメントの学習、段階的な導入 |
| Tailwind CSS v4への移行 | 低 | 公式マイグレーションガイドの活用 |

### 技術的負債

1. **React 19の新機能**: 段階的に新機能を導入し、既存コードとの互換性を維持
2. **Next.js App Router**: 従来のPages Routerからの移行を計画的に実施

---

## 実装ガイドライン

### バックエンド

```java
// Spring Boot 3.2 + Java 17 の推奨パターン

// Record型の活用（Java 17）
public record LoginRequest(
    @NotBlank @Email String email,
    @NotBlank @Size(min = 6, max = 100) String password,
    boolean rememberMe
) {}

// Spring Security 6.x の設定
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .anyRequest().authenticated())
            .build();
    }
}
```

### フロントエンド

```typescript
// Next.js 15 + React 19 の推奨パターン

// Server Component（デフォルト）
export default async function ProductsPage() {
  const categories = await getCategories()
  return <CategoryList categories={categories} />
}

// Client Component（インタラクティブな機能が必要な場合）
'use client'
export function LoginForm() {
  const form = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
  })
  // ...
}

// Zustand ストア
export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      isAuthenticated: false,
      login: (user) => set({ user, isAuthenticated: true }),
      logout: () => set({ user: null, isAuthenticated: false }),
    }),
    { name: 'auth-storage' }
  )
)
```

---

## 参照

### 公式ドキュメント

- [Spring Boot 3.2 Reference](https://docs.spring.io/spring-boot/docs/3.2.0/reference/html/)
- [Spring Security 6.x Reference](https://docs.spring.io/spring-security/reference/)
- [Next.js 15 Documentation](https://nextjs.org/docs)
- [React 19 Documentation](https://react.dev/)
- [Tailwind CSS v4 Documentation](https://tailwindcss.com/docs)

### 関連ADR

- なし（初回決定）

### 関連設計書

- EP001_auth_api.md - 認証API仕様書
- EP002_product_api.md - 商品カテゴリAPI仕様書
- MS001_auth_module.md - 認証モジュール設計書
- MS002_product_module.md - 商品カテゴリモジュール設計書

---

## 変更履歴

| 日付 | 変更者 | 変更内容 |
|------|--------|---------|
| 2025-12-14 | Devin AI | 初版作成 |
