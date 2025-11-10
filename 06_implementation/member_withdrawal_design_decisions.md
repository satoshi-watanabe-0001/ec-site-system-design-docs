# 会員退会機能 設計決定事項ドキュメント（確定版）

**チケット**: EC-18  
**作成日**: 2025-11-10  
**更新日**: 2025-11-10  
**ステータス**: Phase 1 - 設計決定確定（一部追加確認待ち）  
**優先度**: 高（法的コンプライアンス要件あり）

---

## 📋 ステークホルダー確認結果

### ✅ 確定した方針

#### プロダクトオーナーからの回答
1. **機能の位置づけ**: 会員情報編集画面の一部
2. **再登録ポリシー**: クーリングオフ期間を設ける
3. **悪用防止策**: 防止策を実装（具体案は本ドキュメントで提案）

#### 法務/コンプライアンスチームからの回答
1. **データ保持期間**: 7年（注文・会計データ、バックアップ、監査ログ）
2. **保持すべきフィールド**: 本ドキュメントで提案
3. **匿名化方法**: ソフトデリート
4. **バックアップデータの削除タイムライン**: 7年
5. **第三者提供データの削除手順**: 本ドキュメントで提案
6. **監査ログの保持期間と匿名化方法**: 7年、ソフトデリート
7. **再登録ポリシーの合法性**: メールアドレスのハッシュ保持は合法
8. **ユーザー通知文面**: 本ドキュメントで仮置き作成

#### バックエンドチームからの回答
1. **API実装可能性**: 別途実装予定（仮置きで設計）
2. **Kafka設定**: 具体的な質問事項を整理して再確認
3. **アウトボックスパターンの既存実装状況**: 未実装
4. **冪等性キーの管理方法**: 標準的なもの
5. **DLQ運用体制**: 本ドキュメントで仮置き
6. **ユーザー状態の標準定義**: 本ドキュメントで提案

### ⏳ 追加確認が必要な事項

1. **クーリングオフ期間の具体的な日数**: 30日 or 60日？
2. **ソフトデリートの解釈**: 論理削除フラグ + PII即時削除 or PIIも含めて7年保持？
3. **UI構成**: プロフィール編集画面から専用ページ遷移 or モーダル表示？

---

## 🚨 重要な問題点（解決済み）

### 1. タイトル/内容の不一致 ✅ 解決

**問題**: Jiraチケットのタイトルと実際の内容が一致していない。

- **チケットタイトル**: 「会員情報編集画面」
- **実際のユーザーストーリー**: 「会員の退会手続き（個人情報削除対応）」

**確定した方針**: 会員情報編集画面の一部として実装
- プロフィール編集画面内に「退会」セクションを設置
- そこから専用の退会ページ（`/profile/withdraw`）に遷移（推奨）
- ルーティング: `/profile/withdraw`

### 2. リポジトリパスの誤り ✅ 解決

**問題**: チケット記載のパスが誤り

- **チケット記載**: `apps/customer-app/src/app/profile/withdraw/`
- **正しいパス**: `apps/customer-app/app/profile/withdraw/`（`src`ディレクトリは存在しない）

### 3. 技術スタックのバージョン不一致 ✅ 解決

**チケット記載**: Next.js 14, React 18, Tailwind CSS 3

**実際のバージョン**: 
- Next.js 16.0.1
- React 19.2.0
- Tailwind CSS 4.x

---

## 📋 現状調査結果

### システムアーキテクチャ

**マイクロサービス構成** (13サービス):
- Auth Service (Port 8081) - 認証・認可、ユーザー管理
- Customer Service (Port 8089) - 顧客情報管理
- Order Service (Port 8085) - 注文管理
- Payment Service (Port 8086) - 決済管理
- Point/Coupon Service (Port 8087) - ポイント・クーポン管理
- Marketing Service (Port 8090) - マーケティング
- Analytics Service (Port 8091) - 分析
- Notification Service (Port 8093) - 通知
- その他5サービス

**イベント駆動アーキテクチャ**:
- Apache Kafka を使用
- サービス間の非同期通信
- イベントソーシング

**フロントエンド技術スタック**:
- Next.js 16.0.1
- React 19.2.0
- Tailwind CSS 4.x
- TypeScript 5.x
- Zustand (グローバル状態管理)
- React Query (サーバー状態管理)
- React Hook Form (フォーム状態管理)

### 既存実装の確認結果

**Customer Service エンティティ**:
```java
@Entity
@Table(name = "customers")
public class Customer {
  @Id
  @GeneratedValue(strategy = GenerationType.UUID)
  private UUID id;
  
  @Column(name = "status", nullable = false, length = 20)
  @Builder.Default
  private String status = "active";
  
  // 関連エンティティ
  @OneToOne(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
  private CustomerProfile profile;
  
  @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<CustomerAddress> addresses;
  
  @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<CustomerPhoneNumber> phoneNumbers;
}
```

**現在の状態**:
- `status` フィールドが存在（現在は "active" のみ）
- 退会関連のステータス（WITHDRAW_PENDING, WITHDRAWN等）は未定義
- 退会APIエンドポイントは未実装
- API契約（ec-site-shared-contracts）に退会関連の定義なし

---

## 🎯 確定した設計方針

### 1. データ削除ポリシー

#### ✅ 確定方針: ソフトデリート + PII即時削除（ハイブリッド）

**重要**: 「ソフトデリート」を単なる論理削除フラグだけにせず、GDPR「忘れられる権利」との整合性を確保するため、以下のハイブリッド方式を採用します。

#### 実装方針

**即時処理**:
1. **アカウントステータスを `WITHDRAW_PENDING` → `WITHDRAWN` に変更**
2. **Auth Serviceでトークン失効・ログイン不可化**
3. **個人識別情報（PII）を即時削除または空文字化**:
   - 氏名 → NULL または空文字
   - メールアドレス → 削除（クーリングオフ用にHMACのみ別レジストリに保持）
   - 電話番号 → NULL
   - 住所（都道府県、市区町村、番地、建物名） → NULL
   - 生年月日 → NULL
   - その他のPII → NULL

**保持するデータ（7年間）**:
- **注文履歴**（法定保管義務: 会社法・税法）
  - 注文ID、注文番号、注文日時
  - 合計金額、税額、配送料、決済金額
  - 商品SKU、商品名、単価、数量、小計
  - 決済方法種別（カード番号等の機密情報は保持しない）
  - 配送先は都道府県レベルまで（詳細住所は削除）
  - 顧客参照: `customer_surrogate_id`（ランダムUUID、元のuser_idとは切断）
- **会計・請求データ**（税法・会社法の保管義務）
  - 金額情報、税務情報（個人特定不可の形で）
- **監査ログ**（7年間保持、ソフトデリート）
  - 操作履歴（who/when/what）
  - PIIはマスク化またはトークン化
  - アクセス制限と用途限定を明記

**削除するデータ**:
- アクティブ系テーブルからのPII（即時）
- バックアップ/ログ/検索インデックスの二次データ（7年後）
- 第三者提供データ（即時〜30日以内）

#### 法的根拠とGDPR整合性

**法的根拠**:
- 会計・税務の保全義務（会社法第432条、法人税法第150条の2）に基づく7年間保持
- PIIは最小化・不可逆化し、目的外利用を禁止
- 監査ログは法令順守と不正防止のために必要最小限を保持

**GDPR整合性**:
- 「忘れられる権利」（GDPR第17条）は、法的義務に基づく保持を例外として認める
- PIIを即時削除し、法定保持が必要な非PIIのみ保持することで整合性を確保
- バックアップは「運用から事実上不可視」（暗号化鍵分離、復元時は再度削除処理を必須化）

#### データ保持マトリクス

| データ種別 | フィールド例 | 処理方法 | 保持期間 | 根拠 |
|-----------|------------|---------|---------|------|
| **PII（個人識別情報）** | 氏名、メールアドレス、電話番号、詳細住所、生年月日 | 即時削除（NULL化） | 0日 | GDPR第17条 |
| **注文ヘッダ** | 注文ID、注文日時、合計金額、税額 | 保持（customer_idはサロゲートキーに差し替え） | 7年 | 会社法第432条 |
| **注文明細** | 商品SKU、商品名、単価、数量 | 保持 | 7年 | 会社法第432条 |
| **配送先情報** | 都道府県 | 集約して保持 | 7年 | 会社法第432条 |
| **配送先情報** | 市区町村、番地、建物名 | 削除 | 0日 | GDPR第17条 |
| **決済情報** | 決済方法種別 | 保持 | 7年 | 会社法第432条 |
| **決済情報** | カード番号、CVV | 削除（元々保持していない） | 0日 | PCI DSS |
| **ポイント/クーポン** | 保有ポイント、クーポンID | 失効処理 | 0日 | - |
| **監査ログ** | 操作履歴（マスク化済み） | 保持 | 7年 | 内部統制 |
| **バックアップ** | 全データ | 暗号化保管（不可視化） | 7年 | BCP |

---

### 2. 分散トランザクション戦略

#### ✅ 確定方針: サガパターン（オーケストレーション）

**オーケストレーター**: Customer Service (Port 8089) を起点とする

**状態遷移**:
```
ACTIVE → WITHDRAW_PENDING → WITHDRAWN
         ↓ (失敗時)
         WITHDRAW_FAILED (再試行可能)
```

#### サガステップ（順序実行）

**Step 1: アカウント無効化準備**
- Customer Serviceで状態を `WITHDRAW_PENDING` に変更
- 冪等性キー: `{user_id}_{timestamp}_{operation}`
- タイムアウト: 5秒

**Step 2: Auth Service - 認証無効化**
- エンドポイント: `POST /api/v1/auth/users/{userId}/deactivate`（仮置き）
- 処理内容:
  - Refresh tokenを失効
  - Access tokenをブラックリスト化
  - セッション無効化
  - ログイン不可フラグ設定
- 冪等性: user_id + deactivation_timestamp
- タイムアウト: 5秒
- 失敗時: 再試行（最大3回、エクスポネンシャルバックオフ）

**Step 3: Customer Service - PII削除**
- 処理内容:
  - 氏名、メールアドレス、電話番号、住所をNULL化
  - メールアドレスのHMACを別レジストリ（cooling_off_registry）に保存
  - customer_idは保持（注文履歴との紐付け用）
  - 削除は不可逆（復元不可）
- 冪等性: 既に削除済みの場合はスキップ
- タイムアウト: 10秒

**Step 4: Kafkaイベント発行**
- トピック: `user.withdrawal.completed`（仮置き、バックエンドチームと調整）
- イベントスキーマ:
```json
{
  "schema_version": "1.0",
  "event_id": "uuid-v7",
  "event_type": "UserWithdrawalCompleted",
  "occurred_at": "2025-11-10T08:00:00Z",
  "user_id": "original-uuid",
  "customer_surrogate_id": "surrogate-uuid",
  "idempotency_key": "user_id_timestamp_withdraw",
  "trace_id": "distributed-trace-id",
  "reason": "user_requested"
}
```

**Step 5: 各サービスでの非同期処理**
- Point/Coupon Service: ポイント失効、クーポン無効化
- Marketing Service: メール配信停止、セグメント除外、第三者ベンダーへの削除リクエスト
- Analytics Service: 識別子の無効化
- Notification Service: 退会完了メール送信
- その他のサービス: 必要に応じて二次データ処理

#### エラーハンドリング戦略

**基本方針**: 完全なロールバックは行わない（セキュリティリスクが高い）

**失敗時の対応**:
1. `WITHDRAW_PENDING` 状態で留める
2. 再試行（エクスポネンシャルバックオフ、最大3回）
3. 再試行失敗時はDLQ（Dead Letter Queue）に送信
4. 運用チームへアラート（Slack/PagerDuty）
5. 手動介入による完了（Runbook参照）

**補償トランザクション**:
- Auth Service無効化成功後にCustomer Service削除が失敗した場合
  - Auth Serviceの再有効化は**行わない**（セキュリティリスク）
  - 代わりに再試行と運用介入で対応

#### 技術実装

**アウトボックスパターン**（未実装のため導入計画が必要）:
- Customer Serviceのトランザクション内で:
  1. 顧客データ更新
  2. outbox_eventsテーブルにイベント記録
- 別プロセス（Outbox Publisher）でoutboxをポーリングしてKafkaに発行
- テーブル構造（例）:
```sql
CREATE TABLE outbox_events (
  id UUID PRIMARY KEY,
  aggregate_type VARCHAR(50) NOT NULL,
  aggregate_id UUID NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
  created_at TIMESTAMP NOT NULL,
  published_at TIMESTAMP,
  retry_count INT DEFAULT 0
);
```

**冪等性保証**:
- 全APIリクエストに `Idempotency-Key` ヘッダー必須
- 形式: `{user_id}_{timestamp}_{operation}`（例: `123e4567-e89b-12d3-a456-426614174000_1699603200_withdraw`）
- サーバー側で重複リクエストを検知・スキップ
- 冪等性キーの保持期間: 48〜72時間（TTL）

---

### 3. 再登録ポリシー

#### ✅ 確定方針: クーリングオフ期間を設ける

**クーリングオフ期間**: 30日間（仮置き、最終確認待ち）

#### 実装方針

**クーリングオフレジストリ**:
- 専用テーブル `cooling_off_registry` を作成
- メールアドレスの生データは保持せず、HMAC-SHA256(email, secret)のみ保持
- TTL（Time To Live）で自動削除

**テーブル構造**:
```sql
CREATE TABLE cooling_off_registry (
  id UUID PRIMARY KEY,
  hmac_email VARCHAR(64) NOT NULL UNIQUE,
  reason VARCHAR(20) NOT NULL DEFAULT 'withdrawal',
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL,
  INDEX idx_hmac_expires (hmac_email, expires_at)
);
```

**フロー**:
1. **退会時**: 
   - メールアドレスのHMAC-SHA256を計算
   - cooling_off_registryに登録（expires_at = 現在時刻 + 30日）
   - ユーザーテーブルからメールアドレスを削除
2. **新規登録時**:
   - 入力されたメールアドレスのHMAC-SHA256を計算
   - cooling_off_registryを検索
   - 未期限切れのレコードが存在する場合は登録拒否
   - エラーメッセージ: 「このメールアドレスは現在使用できません。退会後XX日間は再登録できません。」
3. **期限切れ後**:
   - バッチ処理で期限切れレコードを自動削除
   - 再登録が可能になる

**法的根拠**:
- メールアドレスのハッシュ保持は合法（法務確認済み）
- HMACは不可逆で復元不能、照合のみ可能
- 目的限定（再登録防止）、期間限定（30日）

**再登録時の扱い**:
- 完全に新規ユーザーとして扱う
- 過去のポイント・クーポン・注文履歴は引き継がない
- UI上で明示: 「再登録した場合、過去のデータは引き継がれません」

---

### 4. 悪用防止策

#### ✅ 提案内容

**1. 退会直前の再認証**
- パスワード再入力必須
- 2FA（Two-Factor Authentication）が有効な場合は2FAも必須
- セッションタイムアウト: 再認証後10分以内に退会完了が必要

**2. 確認テキスト入力**
- ユーザーに「退会します」の完全一致入力を要求
- 大文字小文字を区別
- コピー&ペースト可能（アクセシビリティ考慮）

**3. 未解決トランザクションのチェック**
- 以下の状態がある場合は退会不可:
  - 返金処理中の注文
  - 未発送の注文
  - 配送中の注文
  - 未解決の問い合わせ/クレーム
  - チャージバック処理中の決済
- エラーメッセージ: 「未解決の取引があるため、現在退会できません。詳細はカスタマーサポートにお問い合わせください。」

**4. ポイント/クーポンの即時失効**
- 退会時に保有ポイントを即時失効
- 保有クーポンを即時無効化
- UI上で明示: 「退会すると、保有ポイント（XX pt）とクーポン（XX枚）が失効します」

**5. 新規登録のレートリミット**
- 同一IP/デバイスからの頻繁な登録を制限
- 1時間あたり3回まで
- 1日あたり10回まで
- 超過時はCAPTCHA表示またはアカウント作成一時停止

**6. リスクベース判定**
- 以下の場合は退会をブロックまたは運用審査:
  - 高額注文直後（例: 10万円以上の注文から7日以内）
  - 返品処理中
  - ポイント大量取得直後（例: 1万pt以上取得から7日以内）
  - 不正利用の疑いがある場合

**7. 監査ログの記録**
- 退会操作の詳細をセキュアロギング
- 記録内容: user_id, timestamp, IP address, user agent, reason（任意）, re-auth method
- アクセス制限: 運用チームとセキュリティチームのみ閲覧可能
- 保持期間: 7年

---

### 5. ユーザー状態の標準定義（サービス横断）

#### ✅ 提案内容

**状態定義**:

| 状態 | 値 | 意味 | ログイン可否 | データ状態 |
|-----|---|------|------------|----------|
| **ACTIVE** | `ACTIVE` | 通常利用可能 | ✅ 可能 | 通常 |
| **WITHDRAW_PENDING** | `WITHDRAW_PENDING` | 退会処理進行中 | ❌ 不可 | PII削除準備中 |
| **WITHDRAWN** | `WITHDRAWN` | 退会完了 | ❌ 不可 | PII削除済み |
| **WITHDRAW_FAILED** | `WITHDRAW_FAILED` | 退会処理失敗 | ❌ 不可 | 要運用介入 |

**状態遷移**:
```
ACTIVE → WITHDRAW_PENDING → WITHDRAWN
         ↓ (失敗時)
         WITHDRAW_FAILED → (運用介入) → WITHDRAWN
```

**不変条件**:
- `WITHDRAWN` 状態ではログイン不可
- `WITHDRAWN` 状態ではPIIが空/削除済み
- `WITHDRAW_PENDING` から `ACTIVE` への遷移は運用介入のみ（通常は発生しない）
- イベントは `idempotency_key` で一意

**実装ガイドライン**:
- 全サービスで状態名を厳密に統一（微妙な違い（DELETED/INACTIVE等）を避ける）
- 状態チェックは Auth Service または API Gateway 層で一元化
- フロントエンドだけでブロックしない（バックエンドでも必ずチェック）

---

### 6. 第三者提供データの削除手順

#### ✅ 提案内容

**データマップ作成**:

| ベンダー種別 | 具体例 | 識別子 | 削除API | SLA | 証跡取得方法 |
|------------|-------|-------|---------|-----|------------|
| メール配信 | SendGrid, Mailchimp | email, vendor_user_id | DELETE /api/v3/marketing/contacts | 24時間 | API response + webhook |
| CDP | Segment, mParticle | user_id, anonymous_id | DELETE /v1/users/{id} | 48時間 | API response |
| MA | Marketo, HubSpot | email, contact_id | DELETE /rest/v1/leads/{id} | 72時間 | API response |
| 広告 | Google Ads, Facebook Ads | hashed_email | 配信停止リスト追加 | 即時 | API response |
| 分析 | Google Analytics, Mixpanel | user_id | User Deletion API | 30日 | API response |
| 決済 | Stripe, PayPal | customer_id | DELETE /v1/customers/{id} | 即時 | API response |

**連携フロー**:
1. `user.withdrawal.completed` イベントを「vendor-erasure-processor」が購読
2. 各ベンダーAPIで削除/停止リクエストを送信
3. リクエストID、時刻、結果、再試行回数を記録
4. 成功: 証跡を保存
5. 失敗: DLQに送信 + 運用チームへアラート
6. SLA超過: エスカレーション

**照合方法**:
- 原則: `vendor_user_id` でのマッチング（事前に保持している場合）
- メールアドレスが必要な場合: 退会前に削除リクエストを送信（退会処理の一部として）
- ハッシュ化メールアドレスが必要な場合: ベンダー仕様に合わせて対応

**証跡管理**:
- テーブル: `vendor_erasure_log`
```sql
CREATE TABLE vendor_erasure_log (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  vendor_name VARCHAR(50) NOT NULL,
  vendor_user_id VARCHAR(255),
  request_id VARCHAR(255),
  request_at TIMESTAMP NOT NULL,
  response_status INT,
  response_body TEXT,
  retry_count INT DEFAULT 0,
  completed_at TIMESTAMP,
  INDEX idx_user_vendor (user_id, vendor_name)
);
```

**運用**:
- ベンダーごとにRunbook作成
- 月次監査レポート（削除完了率、SLA遵守率）
- 年次ベンダーリスト見直し

---

### 7. DLQ運用体制

#### ✅ 提案内容（仮置き）

**トピック命名規則**:
- パターン: `{original_topic}.dlq`
- 例: `user.withdrawal.completed.dlq`

**保持期間**:
- 14日間（Kafka retention設定）
- 14日経過後は自動削除

**監視とアラート**:
- メトリクス:
  - DLQ件数（累積）
  - DLQ滞留時間（最古メッセージの経過時間）
  - 再試行失敗数（24時間あたり）
- アラート閾値:
  - DLQ件数 > 10件: Slack通知
  - DLQ件数 > 50件: PagerDuty通知
  - 滞留時間 > 24時間: Slack通知
  - 滞留時間 > 72時間: PagerDuty通知
- 通知先: #ops-alerts（Slack）、on-call engineer（PagerDuty）

**再処理手順**:
1. **原因調査**:
   - DLQメッセージを確認
   - エラーログを確認
   - 失敗原因を分類（恒久的 or 一時的）
2. **恒久的エラー**（データ不整合、スキーマ違反等）:
   - データ補正を実施
   - 補正後に再送
3. **一時的エラー**（ネットワーク障害、サービス停止等）:
   - 障害復旧を確認
   - 再送（指数バックオフで自動再試行済みのため、手動re-drive）
4. **再処理ツール**:
   - コマンド: `kafka-dlq-redriver`
   - オプション:
     - `--topic`: DLQトピック名
     - `--range`: 再処理範囲（offset範囲 or 時刻範囲）
     - `--dry-run`: 実行前確認
     - `--throttle`: スロットリング（例: 10 msg/sec）
5. **冪等性確認**:
   - 再処理前に `idempotency_key` の重複チェック
   - 既に処理済みの場合はスキップ

**権限管理**:
- 再処理権限: 運用チームのみ
- 監査ログ記録: 誰が、いつ、何を再処理したか

**Runbook**:
- タイトル: 「DLQ再処理手順書」
- 内容:
  - 失敗原因の分類フローチャート
  - 原因別の対応手順
  - 再処理コマンド例
  - エスカレーション基準
  - 連絡先リスト

---

### 8. ユーザー通知文面

#### ✅ 提案内容（仮置き）

**退会確認画面の文言**:

```
会員退会に関する重要なお知らせ

退会手続きを行うと、以下の影響があります。ご確認の上、お進みください。

【アカウントについて】
・アカウントは利用できなくなり、ログインができなくなります
・保有ポイント（XX pt）とクーポン（XX枚）は失効します
・退会後XX日間は、同一メールアドレスでの再登録ができません

【個人情報について】
・個人情報（氏名・住所・電話番号・メールアドレス等）は当社の法令順守方針に基づき削除されます
・注文・会計に関する情報は関係法令（会社法・税法）に基づき最大7年間保管されますが、個人が特定されない形で管理されます
・バックアップに残存する情報は運用からアクセスできない形で保管され、所定の期間経過後に削除されます

【注文について】
・未処理の注文、返金処理中の取引がある場合、退会手続きが完了できない場合があります
・過去の注文履歴は閲覧できなくなります

【再登録について】
・退会後XX日間のクーリングオフ期間経過後、再登録が可能です
・再登録した場合、過去のデータ（ポイント・クーポン・注文履歴）は引き継がれません

上記内容を理解し、退会手続きを進めます
□ 同意する

確認のため、以下に「退会します」と入力してください
[                    ]

[キャンセル]  [退会手続きを進める]
```

**退会完了画面の文言**:

```
退会手続きが完了しました

ご利用いただき、ありがとうございました。

【今後について】
・アカウントは無効化され、ログインできなくなりました
・個人情報は削除されました
・注文・会計情報は法令に基づき保管されますが、個人が特定されない形で管理されます
・XX日後から、同一メールアドレスでの再登録が可能になります

【お問い合わせ】
退会に関するご質問は、カスタマーサポートまでお問い合わせください。
※ただし、退会後は本人確認が困難なため、対応できない場合があります

[トップページへ戻る]
```

**退会完了メール**:

```
件名: 【ECサイト】退会手続き完了のお知らせ

○○ 様

この度は、ECサイトをご利用いただき、誠にありがとうございました。
退会手続きが完了いたしましたので、お知らせいたします。

■ 退会完了日時
YYYY年MM月DD日 HH:MM

■ 今後について
・アカウントは無効化され、ログインできなくなりました
・個人情報は削除されました
・注文・会計情報は関係法令に基づき最大7年間保管されますが、個人が特定されない形で管理されます
・XX日後から、同一メールアドレスでの再登録が可能になります

■ お問い合わせ
退会に関するご質問は、以下までお問い合わせください。
カスタマーサポート: support@example.com
※ただし、退会後は本人確認が困難なため、対応できない場合があります

今後とも、ECサイトをよろしくお願いいたします。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ECサイト運営事務局
Email: support@example.com
URL: https://example.com
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

※このメールは送信専用です。返信いただいても対応できません。
```

---

## 🔧 バックエンドチームへの具体的な質問事項（Kafka設定）

以下の点について確認が必要です：

### 1. トピック命名規則
- **質問**: 既存のトピック命名規約は何ですか？
- **提案**: `user.withdrawal.requested` / `user.withdrawal.completed` / `user.withdrawal.failed`
- **確認事項**: ドメイン名（user）、イベント名（withdrawal）、状態（completed）の順で良いか？

### 2. パーティションキーとパーティション数
- **質問**: パーティションキーは `user_id` で良いですか？（同一ユーザーの順序保証のため）
- **質問**: パーティション数の推奨は？（例: 12〜24、将来スケールを想定）
- **確認事項**: 現在の他のトピックのパーティション数は？

### 3. スキーマ管理
- **質問**: スキーマレジストリ（Avro/JSON Schema）は使用していますか？
- **質問**: スキーマのバージョニングポリシーは？（後方互換性の要件）
- **確認事項**: 既存のイベントスキーマの標準フォーマットは？

### 4. 配信保証
- **質問**: at-least-once前提で、コンシューマ側で冪等実装する方針で良いですか？
- **質問**: exactly-onceが必要なケースはありますか？
- **確認事項**: 既存のイベント処理の配信保証レベルは？

### 5. イベントヘッダ
- **質問**: `trace_id`、`idempotency_key` をヘッダに含める標準はありますか？
- **質問**: 他に必須のヘッダはありますか？（例: `correlation_id`、`causation_id`）
- **確認事項**: 既存のイベントヘッダの標準フォーマットは？

### 6. 監視とSLA
- **質問**: Kafka Lagや処理時間のSLAは設定されていますか？
- **質問**: 監視ツールは何を使用していますか？（Prometheus, Grafana, Datadog等）
- **確認事項**: アラート閾値の標準は？

### 7. 保持期間と圧縮
- **質問**: イベントの保持期間（retention time）は？
- **質問**: ログ圧縮（log compaction）は使用していますか？
- **確認事項**: PIIを含めない原則は徹底されていますか？

### 8. Outboxパターンの導入計画
- **質問**: Outboxパターンは未実装とのことですが、導入計画はありますか？
- **質問**: 導入までの暫定対応は？（直接Kafka送信 + 手動リトライ？）
- **確認事項**: 導入スケジュールとリソース確保は？

---

## 📊 API契約草案

### POST /api/v1/customers/me/withdraw

**概要**: 会員退会リクエスト

**リクエスト**:
```json
{
  "confirmation_text": "退会します",
  "reason": "サービスを利用しなくなったため",
  "re_auth_token": "password-or-2fa-token"
}
```

**レスポンス（成功）**:
```json
{
  "status": "success",
  "message": "退会手続きが完了しました",
  "data": {
    "user_id": "uuid",
    "status": "WITHDRAWN",
    "withdrawn_at": "2025-11-10T08:00:00Z"
  }
}
```

**レスポンス（エラー）**:
```json
{
  "status": "error",
  "message": "未解決の取引があるため、退会できません",
  "errors": [
    {
      "field": "pending_transactions",
      "message": "未発送の注文があります（注文番号: ORD-12345）"
    }
  ],
  "timestamp": "2025-11-10T08:00:00Z"
}
```

**エラーコード**:
- `400 Bad Request`: 確認テキスト不一致、再認証失敗
- `409 Conflict`: 未解決トランザクションあり、クーリングオフ期間中
- `429 Too Many Requests`: レートリミット超過
- `500 Internal Server Error`: サーバーエラー

**冪等性**:
- ヘッダ: `Idempotency-Key: {user_id}_{timestamp}_withdraw`
- 同一キーでの再リクエストは200を返し、既存の結果を返す

---

## 🎨 フロントエンド実装方針

### UI構成

**入口**: プロフィール編集画面（`/profile/edit`）
- 「退会」セクションを設置
- 「退会手続きへ」リンクで専用ページに遷移

**退会ページ**: `/profile/withdraw`（専用ページ、モーダルではない）

**理由**: 
- 誤操作防止
- 詳細な説明が必要
- アクセシビリティ（スクリーンリーダー、キーボード操作）
- 多段階フロー

### 画面フロー

**Step 1: 影響説明**
- 退会の影響を詳細に説明
- チェックボックス: 「上記内容を理解しました」
- ボタン: 「次へ」

**Step 2: 再認証**
- パスワード再入力フォーム
- 2FA有効な場合は2FAコード入力
- ボタン: 「認証する」

**Step 3: 確認テキスト入力**
- 確認テキスト入力: 「退会します」
- 理由入力（任意、最大500文字）
- ボタン: 「退会手続きを進める」

**Step 4: 最終確認ダイアログ**
- モーダルダイアログ表示
- 「本当に退会しますか？この操作は取り消せません。」
- ボタン: 「キャンセル」「退会する」

**Step 5: 処理中**
- ローディングスピナー表示
- 「退会手続きを処理しています...」

**Step 6: 完了**
- 自動ログアウト
- 退会完了ページ表示
- 「退会手続きが完了しました」
- ボタン: 「トップページへ戻る」

### コンポーネント構成

```
apps/customer-app/app/profile/withdraw/
├── page.tsx                    # 退会ページ（メインコンテナ）
├── layout.tsx                  # 退会専用レイアウト（必要に応じて）
└── components/
    ├── WithdrawStep1.tsx       # Step 1: 影響説明
    ├── WithdrawStep2.tsx       # Step 2: 再認証
    ├── WithdrawStep3.tsx       # Step 3: 確認テキスト入力
    ├── WithdrawConfirmDialog.tsx  # Step 4: 最終確認ダイアログ
    ├── WithdrawProcessing.tsx  # Step 5: 処理中
    └── WithdrawComplete.tsx    # Step 6: 完了

apps/customer-app/lib/
├── api/
│   └── customer.ts             # 退会API関数
├── schemas/
│   └── withdraw.schema.ts      # Zod検証スキーマ
├── hooks/
│   └── useWithdraw.ts          # 退会カスタムフック
└── stores/
    └── withdraw-store.ts       # 退会フロー状態管理（Zustand）
```

### 状態管理

**Zustand Store** (`withdraw-store.ts`):
```typescript
interface WithdrawState {
  currentStep: number;
  isProcessing: boolean;
  error: string | null;
  agreedToTerms: boolean;
  reAuthToken: string | null;
  confirmationText: string;
  reason: string;
  
  setCurrentStep: (step: number) => void;
  setAgreedToTerms: (agreed: boolean) => void;
  setReAuthToken: (token: string) => void;
  setConfirmationText: (text: string) => void;
  setReason: (reason: string) => void;
  reset: () => void;
}
```

**React Query Mutation** (`useWithdraw.ts`):
```typescript
const withdrawMutation = useMutation({
  mutationFn: withdrawAccount,
  onSuccess: (data) => {
    // ログアウト処理
    clearUser();
    clearTokens();
    // 完了ページへ遷移
    router.push('/profile/withdraw/complete');
  },
  onError: (error) => {
    // エラーハンドリング
    setError(extractErrorMessage(error));
  }
});
```

### バリデーション

**Zod Schema** (`withdraw.schema.ts`):
```typescript
export const withdrawSchema = z.object({
  confirmationText: z.literal("退会します", {
    errorMap: () => ({ message: "「退会します」と正確に入力してください" })
  }),
  reason: z.string()
    .max(500, "理由は500文字以内で入力してください")
    .optional(),
  reAuthToken: z.string()
    .min(1, "パスワードを入力してください")
});

export type WithdrawFormData = z.infer<typeof withdrawSchema>;
```

### アクセシビリティ

**WCAG 2.1 AA準拠**:
- フォーカス管理: 各ステップ遷移時に適切な要素にフォーカス
- ARIA属性:
  - `role="alert"` でエラーメッセージ
  - `aria-live="polite"` でステップ変更通知
  - `aria-describedby` で説明文との関連付け
- キーボード操作:
  - Tab/Shift+Tabでフォーカス移動
  - Enter/Spaceでボタン操作
  - Escapeでダイアログ閉じる
- スクリーンリーダー:
  - 各ステップのタイトルを `<h1>` で明示
  - フォームラベルを `<label>` で関連付け
  - エラーメッセージを読み上げ

### エラーハンドリング

**ネットワークエラー**:
- タイムアウト: 30秒
- リトライ: 自動リトライなし（ユーザーに再試行を促す）
- エラーメッセージ: 「通信エラーが発生しました。しばらくしてから再度お試しください。」

**バリデーションエラー**:
- フィールドレベルエラー表示
- エラーフィールドにフォーカス
- エラーメッセージを赤字で表示

**ビジネスロジックエラー**:
- 未解決トランザクション: 「未発送の注文があります（注文番号: ORD-12345）。注文完了後に再度お試しください。」
- クーリングオフ期間中: 「このメールアドレスは現在使用できません。退会後XX日間は再登録できません。」
- レートリミット: 「リクエストが多すぎます。しばらくしてから再度お試しください。」

---

## ⚠️ リスクと注意事項

### 高リスク項目

**1. GDPR整合性**
- リスク: ソフトデリートだけではGDPR「忘れられる権利」を満たさない可能性
- 対策: PIIを即時削除し、法定保持が必要な非PIIのみ保持
- 確認: 法務レビュー必須

**2. 分散トランザクションの整合性**
- リスク: 部分的な失敗により、データ不整合が発生する可能性
- 対策: サガパターン、冪等性保証、DLQ運用、監視アラート
- 確認: バックエンドチームとの綿密な連携

**3. バックアップデータの扱い**
- リスク: 7年間のバックアップ保持がGDPR的に長すぎる可能性
- 対策: 「運用から事実上不可視」（暗号化鍵分離、復元時は再度削除処理を必須化）
- 確認: 法務に再相談、可能なら短縮

**4. 第三者ベンダーへの削除漏れ**
- リスク: ベンダーへの削除リクエストが失敗し、PIIが残存する可能性
- 対策: DLQ運用、SLA監視、月次監査レポート
- 確認: ベンダーリストの完全性確認

### 中リスク項目

**5. イベント順序の保証**
- リスク: Kafkaイベントの順序が保証されない場合、不整合が発生
- 対策: パーティションキーを `user_id` にして同一ユーザーの順序保証
- 確認: バックエンドチームとの設計レビュー

**6. 再認証の省略**
- リスク: 再認証を省略すると、セッションハイジャック時に不正退会される可能性
- 対策: 再認証必須、2FA有効時は2FAも必須
- 確認: セキュリティレビュー

**7. UIの誤操作**
- リスク: ユーザーが誤って退会してしまう可能性
- 対策: 多段階フロー、確認テキスト入力、最終確認ダイアログ
- 確認: UXレビュー、ユーザーテスト

### 低リスク項目

**8. パフォーマンス**
- リスク: 7年間のソフトデリートデータがDBを圧迫
- 対策: パーシャルインデックス、90日後アーカイブ、コールドストレージ活用
- 確認: DBチームとの容量計画

**9. ログのPII混入**
- リスク: ログやイベントにPIIが混入する可能性
- 対策: ログ出力時にPIIをマスク、イベントペイロードにPIIを含めない
- 確認: コードレビュー、ログ監査

---

## 📝 次のステップ

### Phase 1完了のために必要なこと

1. ✅ **ステークホルダーからの回答取得** - 完了（一部追加確認待ち）
2. ⏳ **追加確認事項の回答取得**:
   - クーリングオフ期間の具体的な日数（30日 or 60日）
   - ソフトデリートの解釈確認（論理削除フラグ + PII即時削除で良いか）
   - UI構成の確認（専用ページ遷移 or モーダル表示）
3. ⏳ **バックエンドチームとのKafka設定詳細確認**
4. ⏳ **設計ドキュメントの最終化**
5. ⏳ **Phase 1完了報告**

### Phase 2: 実装構造（Phase 1完了後）

1. API契約の確定（ec-site-shared-contracts）
2. バックエンドAPI実装（Customer Service, Auth Service）
3. Kafkaイベントスキーマ確定
4. Outboxパターン導入（または暫定対応）
5. フロントエンド実装
6. 統合テスト
7. セキュリティレビュー
8. 法務レビュー
9. デプロイ

---

## 📚 参考資料

- [GDPR - Right to Erasure (Article 17)](https://gdpr-info.eu/art-17-gdpr/)
- [個人情報保護法](https://www.ppc.go.jp/)
- [会社法 第432条（会計帳簿の保存）](https://elaws.e-gov.go.jp/document?lawid=417AC0000000086)
- [法人税法 第150条の2（帳簿書類の保存）](https://elaws.e-gov.go.jp/document?lawid=340AC0000000034)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Idempotency in Distributed Systems](https://stripe.com/blog/idempotency)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)

---

## 📞 連絡先

**プロダクトオーナー**: [PO名]  
**法務/コンプライアンス**: [担当者名]  
**バックエンドチーム**: [チームリーダー名]  
**フロントエンドチーム**: [チームリーダー名]  
**セキュリティチーム**: [担当者名]

---

**ドキュメント履歴**:
- 2025-11-10: 初版作成（Phase 1 設計決定待ち）
- 2025-11-10: 確定版作成（ステークホルダー回答反映、一部追加確認待ち）
