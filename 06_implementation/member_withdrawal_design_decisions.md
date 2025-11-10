# 会員退会機能 設計決定事項ドキュメント

**チケット**: EC-18  
**作成日**: 2025-11-10  
**ステータス**: Phase 1 - 設計決定待ち  
**優先度**: 高（法的コンプライアンス要件あり）

---

## 🚨 重要な問題点

### 1. タイトル/内容の不一致

**問題**: Jiraチケットのタイトルと実際の内容が一致していません。

- **チケットタイトル**: 「会員情報編集画面」
- **実際のユーザーストーリー**: 「会員の退会手続き（個人情報削除対応）」

**確認が必要な事項**:
- これは単独の退会機能なのか？
- それとも会員情報編集画面の一部として退会機能を含めるのか？
- ルーティングは `/profile/withdraw/` で正しいか？（編集画面とは別導線にすべき）

**推奨**: 誤操作防止のため、退会機能は会員情報編集とは別の独立した画面とすることを推奨します。

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
- Apache Kafka / RabbitMQ を使用
- サービス間の非同期通信
- イベントソーシング

**フロントエンド技術スタック** (実際のバージョン):
- Next.js 16.0.1 (チケットには14と記載されているが実際は16)
- React 19.2.0 (チケットには18と記載されているが実際は19)
- Tailwind CSS 4.x (チケットには3と記載されているが実際は4)
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

## 🎯 解決すべき設計決定事項

### 1. データ削除ポリシー

#### 選択肢と推奨案

**A. 完全削除（物理削除）**
- メリット: データが完全に消える
- デメリット: 
  - バックアップからの復元が困難
  - 監査ログの維持が難しい
  - 関連データの整合性確保が複雑
- 推奨度: ❌ 非推奨

**B. ソフトデリート（論理削除）**
- メリット: データ復元が可能、監査ログ維持
- デメリット: 
  - 個人情報が残るためGDPR/個人情報保護法に抵触する可能性
  - 「忘れられる権利」を満たさない
- 推奨度: ❌ 非推奨

**C. データ匿名化 + アカウント無効化（推奨）** ✅
- メリット:
  - 個人を特定できない形でデータ保持
  - 法的要件（会計・税務）を満たしつつコンプライアンス対応
  - 監査ログ・分析データの維持が可能
- デメリット: 実装が複雑
- 推奨度: ✅ **強く推奨**

#### 推奨実装方針

**即時処理**:
1. アカウントステータスを `WITHDRAW_PENDING` → `WITHDRAWN` に変更
2. Auth Serviceでトークン失効・ログイン不可化
3. 個人識別情報（PII）を即時匿名化:
   - 氏名 → "退会済みユーザー_{ランダムID}"
   - メールアドレス → 削除または不可逆ハッシュ化
   - 電話番号 → 削除
   - 住所 → 削除
   - その他のPII → 削除または匿名化

**保持するデータ**:
- 注文履歴（法定保管義務のため）
  - customer_idは不可逆サロゲートキー（ランダムUUID）に差し替え
  - 注文金額、商品情報は保持（個人特定不可の形で）
- 会計・請求データ（税法・会社法の保管義務）
  - 個人情報は匿名化、金額情報のみ保持

**スケジュール削除**:
- バックアップ/ログ/検索インデックスの二次データ: 30〜90日後にパージ
- ユーザー通知文面に明記: 「完全なデータ削除には最大90日かかります」

#### 確認が必要な事項

🔴 **法務/コンプライアンスチームへの確認事項**:
1. 注文・会計データの法定保管期間（会社法・税法）
2. 保持すべきフィールドの具体的な範囲
3. バックアップデータの削除タイムライン
4. 監査ログの保持期間と匿名化方法
5. 第三者提供データ（マーケティングベンダー等）の削除手順

---

### 2. 分散トランザクション戦略

#### 推奨アプローチ: サガパターン（オーケストレーション）

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
- 冪等性キー: `user_id + timestamp`
- タイムアウト: 5秒

**Step 2: Auth Service - 認証無効化**
- エンドポイント: `POST /api/v1/auth/users/{userId}/deactivate`
- 処理内容:
  - Refresh tokenを失効
  - Access tokenをブラックリスト化
  - セッション無効化
  - ログイン不可フラグ設定
- 冪等性: user_id + deactivation_timestamp
- タイムアウト: 5秒
- 失敗時: 再試行（最大3回、エクスポネンシャルバックオフ）

**Step 3: Customer Service - PII匿名化**
- 処理内容:
  - 氏名、メールアドレス、電話番号、住所を匿名化
  - customer_idは保持（注文履歴との紐付け用）
  - 匿名化は不可逆（復元不可）
- 冪等性: 既に匿名化済みの場合はスキップ
- タイムアウト: 10秒

**Step 4: Kafkaイベント発行**
- トピック: `user.withdrawal.completed`
- イベント:
```json
{
  "event_id": "uuid",
  "event_type": "UserWithdrawalCompleted",
  "timestamp": "2025-11-10T07:45:00Z",
  "user_id": "uuid",
  "anonymized_user_id": "surrogate_uuid",
  "idempotency_key": "user_id_timestamp"
}
```

**Step 5: 各サービスでの非同期処理**
- Point/Coupon Service: ポイント失効、クーポン無効化
- Marketing Service: メール配信停止、セグメント除外
- Analytics Service: 識別子の無効化
- Notification Service: 退会完了メール送信
- その他のサービス: 必要に応じて二次データ処理

#### エラーハンドリング戦略

**基本方針**: 完全なロールバックは行わない（リスクが高い）

**失敗時の対応**:
1. `WITHDRAW_PENDING` 状態で留める
2. 再試行（エクスポネンシャルバックオフ、最大3回）
3. 再試行失敗時はDLQ（Dead Letter Queue）に送信
4. 運用チームへアラート
5. 手動介入による完了

**補償トランザクション**:
- Auth Service無効化成功後にCustomer Service匿名化が失敗した場合
  - Auth Serviceの再有効化は**行わない**（セキュリティリスク）
  - 代わりに再試行と運用介入で対応

#### 技術実装

**アウトボックスパターン**:
- Customer Serviceのトランザクション内で:
  1. 顧客データ更新
  2. outbox_eventsテーブルにイベント記録
- 別プロセスでoutboxをポーリングしてKafkaに発行

**冪等性保証**:
- 全APIリクエストに `Idempotency-Key` ヘッダー必須
- 形式: `{user_id}_{timestamp}_{operation}`
- サーバー側で重複リクエストを検知・スキップ

#### 確認が必要な事項

🔴 **バックエンドチームへの確認事項**:
1. Customer ServiceにサガオーケストレーションロジックをAPI追加可能か？
2. Auth Serviceに `/deactivate` エンドポイント追加可能か？
3. Kafkaトピック名の命名規則
4. アウトボックスパターンの実装状況
5. 冪等性キーの管理方法
6. DLQ（Dead Letter Queue）の運用体制

---

### 3. 再登録ポリシー

#### 選択肢と推奨案

**A. 即時再登録を許可（推奨）** ✅
- メリット:
  - ユーザー体験が良い
  - 実装がシンプル
  - メールアドレスの唯一制約を解除するだけ
- デメリット: 
  - 悪用の可能性（ポイント不正取得等）
- 推奨度: ✅ **推奨**

**B. クーリングオフ期間（30日間）を設ける**
- メリット: 悪用防止
- デメリット:
  - メールアドレスのハッシュを保持する必要がある
  - ハッシュ保持自体が個人情報保護法/GDPRの観点で問題になる可能性
  - 法務の合意が必要
- 推奨度: ⚠️ 法務確認が必要

#### 推奨実装方針

**即時再登録許可（オプションA）**:
1. 退会時にメールアドレスを完全削除
2. 再登録時は完全に新規ユーザーとして扱う
3. 過去のポイント・クーポン・注文履歴は引き継がない
4. UI上で明示: 「再登録した場合、過去のデータは引き継がれません」

#### 確認が必要な事項

🔴 **プロダクトオーナー/法務への確認事項**:
1. 再登録ポリシーの方針（即時許可 or クーリングオフ）
2. クーリングオフを採用する場合の法的妥当性
3. 悪用防止策の必要性（ポイント不正取得等）

---

### 4. コンプライアンス要件

#### GDPR（EU一般データ保護規則）

**「忘れられる権利」（Right to be Forgotten）**:
- 個人データの削除または匿名化が必要
- 例外: 法的義務（会計・税務）がある場合は保持可能
- 対応: データ匿名化 + 法的根拠の文書化

#### 個人情報保護法（日本）

**利用目的達成後の削除義務**:
- 利用目的を達成した個人情報は削除または匿名化が原則
- 例外: 法令に基づく保管義務（会社法・税法）
- 対応: 保持期間と範囲を明確化

#### 会社法・税法（日本）

**会計書類の保管義務**:
- 会社法: 10年間
- 税法: 7年間（法人税法）
- 対象: 注文書、請求書、領収書、会計帳簿
- 対応: 注文・決済データは匿名化して保持

#### 確認が必要な事項

🔴 **法務/コンプライアンスチームへの確認事項**:
1. 注文・会計データの具体的な保管期間
2. 保持すべきフィールドの詳細リスト
3. 匿名化の具体的な方法（ハッシュ化、マスキング、削除）
4. バックアップデータの削除タイムライン
5. 第三者提供データ（マーケティングベンダー）の削除手順
6. 監査ログの保持期間と匿名化方法
7. ユーザー通知文面の法的妥当性

---

## 📊 データ削除/匿名化マトリクス（草案）

| サービス | フィールド | 処理方法 | 保持期間 | 備考 |
|---------|----------|---------|---------|------|
| **Customer Service** | | | | |
| | name | 匿名化 | 即時 | "退会済みユーザー_{UUID}" |
| | email | 削除 | 即時 | 完全削除 |
| | phone_number | 削除 | 即時 | 完全削除 |
| | addresses | 削除 | 即時 | 完全削除 |
| | status | 更新 | 永続 | "WITHDRAWN" |
| | customer_id | 保持 | 永続 | 注文履歴との紐付け用 |
| **Auth Service** | | | | |
| | access_token | 失効 | 即時 | ブラックリスト化 |
| | refresh_token | 失効 | 即時 | 削除 |
| | session | 削除 | 即時 | 全セッション無効化 |
| | login_enabled | 更新 | 永続 | false |
| **Order Service** | | | | |
| | customer_id | サロゲート化 | 7-10年 | 不可逆UUID |
| | order_amount | 保持 | 7-10年 | 金額情報のみ |
| | shipping_address | 匿名化 | 7-10年 | 都道府県のみ保持 |
| **Payment Service** | | | | |
| | customer_id | サロゲート化 | 7-10年 | 不可逆UUID |
| | payment_amount | 保持 | 7-10年 | 金額情報のみ |
| | card_info | 削除 | 即時 | 完全削除 |
| **Point/Coupon Service** | | | | |
| | points | 失効 | 即時 | ポイント失効 |
| | coupons | 無効化 | 即時 | クーポン無効化 |
| **Marketing Service** | | | | |
| | email_subscription | 停止 | 即時 | 配信停止 |
| | segments | 除外 | 即時 | セグメント除外 |
| **Analytics Service** | | | | |
| | user_identifier | 無効化 | 即時 | 識別子無効化 |
| | aggregated_data | 保持 | 永続 | 集計データは保持 |
| **Notification Service** | | | | |
| | email_address | 削除 | 即時 | 完全削除 |
| | notification_history | 匿名化 | 90日 | 送信履歴は匿名化 |

**注**: この表は草案です。法務/コンプライアンスチームの確認が必要です。

---

## 🔧 API契約草案

### エンドポイント

```
POST /api/v1/customers/me/withdraw
```

### リクエスト

```typescript
interface WithdrawRequest {
  confirmationText: string;  // "退会します" (完全一致)
  reason?: string;           // 任意（最大500文字）
  idempotencyKey: string;    // 冪等性キー
}
```

**バリデーション**:
- `confirmationText`: 必須、"退会します" と完全一致
- `reason`: 任意、最大500文字（PII扱い、保存する場合は法務確認必要）
- `idempotencyKey`: 必須、形式 `{user_id}_{timestamp}`

### レスポンス

**成功時（200 OK）**:
```typescript
interface WithdrawResponse {
  status: "success";
  message: "退会処理が完了しました";
  withdrawnAt: string;  // ISO 8601形式
  dataRetentionNotice: string;  // "バックアップデータの完全削除には最大90日かかります"
}
```

**エラー時**:

| ステータスコード | エラーコード | メッセージ | 原因 |
|---------------|------------|----------|------|
| 400 | INVALID_CONFIRMATION | 確認テキストが正しくありません | confirmationTextが不一致 |
| 400 | INVALID_REASON_LENGTH | 理由は500文字以内で入力してください | reasonが長すぎる |
| 401 | UNAUTHORIZED | 認証が必要です | トークンなし/無効 |
| 409 | ALREADY_WITHDRAWN | このアカウントは既に退会済みです | 既に退会済み |
| 409 | WITHDRAWAL_IN_PROGRESS | 退会処理が進行中です | WITHDRAW_PENDING状態 |
| 500 | WITHDRAWAL_FAILED | 退会処理に失敗しました。しばらくしてから再度お試しください | サーバーエラー |
| 503 | SERVICE_UNAVAILABLE | サービスが一時的に利用できません | サービス停止中 |

### タイムアウト・リトライ

- **クライアント側タイムアウト**: 30秒
- **サーバー側タイムアウト**: 25秒
- **リトライ**: クライアント側では自動リトライしない（ユーザーに再実行を促す）
- **冪等性**: 同じ `idempotencyKey` での重複リクエストは同じレスポンスを返す

### セキュリティ

- **認証**: Bearer Token必須
- **CSRF対策**: SameSite Cookie + CSRFトークン
- **Rate Limiting**: 同一ユーザーから1時間に3回まで

---

## 📨 Kafkaイベントスキーマ草案

### トピック名

```
user.withdrawal.initiated
user.withdrawal.pii-anonymized
user.withdrawal.completed
user.withdrawal.failed
```

### イベントスキーマ: UserWithdrawalCompleted

```typescript
interface UserWithdrawalCompletedEvent {
  event_id: string;              // UUID
  event_type: "UserWithdrawalCompleted";
  event_version: "1.0";
  timestamp: string;             // ISO 8601
  
  // ペイロード
  payload: {
    user_id: string;             // 元のユーザーID
    anonymized_user_id: string;  // 匿名化後のサロゲートID
    withdrawn_at: string;        // ISO 8601
    reason_provided: boolean;    // 理由が提供されたか（内容は含めない）
  };
  
  // メタデータ
  metadata: {
    idempotency_key: string;     // 冪等性キー
    source_service: "customer-service";
    correlation_id: string;      // トレーシング用
  };
}
```

### キー設計

- **パーティションキー**: `user_id`
- **順序保証**: 同一ユーザーのイベントは同じパーティションに送信

### 冪等性

- `event_id` をユニークキーとして使用
- 消費側で重複検知・スキップ

---

## 🎨 フロントエンド実装方針

### ディレクトリ構造

```
apps/customer-app/
├── app/
│   └── profile/
│       └── withdraw/
│           └── page.tsx          # 退会ページ
├── components/
│   └── profile/
│       └── withdraw/
│           ├── WithdrawForm.tsx          # 退会フォーム
│           ├── WithdrawConfirmation.tsx  # 確認ダイアログ
│           └── WithdrawComplete.tsx      # 完了画面
├── lib/
│   ├── api/
│   │   └── customer.ts           # Customer API関数
│   ├── schemas/
│   │   └── withdraw.schema.ts    # Zodバリデーションスキーマ
│   ├── hooks/
│   │   └── useWithdraw.ts        # 退会カスタムフック
│   └── stores/
│       └── auth-store.ts         # 既存（ログアウト連携）
```

### Zodバリデーションスキーマ

```typescript
// lib/schemas/withdraw.schema.ts
import { z } from 'zod';

export const withdrawSchema = z.object({
  confirmationText: z.literal("退会します", {
    errorMap: () => ({ message: "「退会します」と正確に入力してください" })
  }),
  reason: z.string()
    .max(500, "理由は500文字以内で入力してください")
    .optional()
});

export type WithdrawFormData = z.infer<typeof withdrawSchema>;
```

### API関数

```typescript
// lib/api/customer.ts
import { apiClient } from './client';
import type { WithdrawFormData } from '../schemas/withdraw.schema';

export interface WithdrawResponse {
  status: 'success';
  message: string;
  withdrawnAt: string;
  dataRetentionNotice: string;
}

export async function withdrawAccount(data: WithdrawFormData): Promise<WithdrawResponse> {
  const idempotencyKey = `${Date.now()}_${Math.random()}`;
  
  const response = await apiClient.post('/api/v1/customers/me/withdraw', {
    ...data,
    idempotencyKey
  });
  
  return response.data;
}
```

### React Queryフック

```typescript
// lib/hooks/useWithdraw.ts
import { useMutation } from '@tanstack/react-query';
import { withdrawAccount } from '../api/customer';
import { useAuthStore } from '../stores/auth-store';
import { useRouter } from 'next/navigation';

export function useWithdraw() {
  const router = useRouter();
  const clearUser = useAuthStore(state => state.clearUser);
  
  return useMutation({
    mutationFn: withdrawAccount,
    onSuccess: (data) => {
      // ローカルトークンをクリア
      clearUser();
      
      // 完了画面へ遷移
      router.push('/profile/withdraw/complete');
    },
    onError: (error) => {
      console.error('Withdrawal failed:', error);
    }
  });
}
```

### UI要件

**確認フロー**:
1. 退会理由入力（任意）
2. 確認テキスト入力（"退会します"）
3. 確認ダイアログ表示
4. 退会実行
5. 完了画面表示 + 自動ログアウト

**エラーハンドリング**:
- フィールドレベルエラー（Zod）
- APIエラー（extractErrorMessage）
- ネットワークエラー
- タイムアウトエラー

**アクセシビリティ**:
- WCAG 2.1 AA準拠
- キーボード操作対応
- スクリーンリーダー対応
- 適切なARIA属性
- フォーカス管理

**ローディング状態**:
- ボタンdisabled
- ローディングスピナー
- 二重送信防止

---

## ⚠️ リスクと注意事項

### 高リスク項目

1. **分散トランザクションの整合性**
   - Auth Service無効化後にCustomer Service匿名化が失敗する可能性
   - 対策: 再試行 + 運用監視 + 手動介入体制

2. **バックアップデータの削除**
   - 即時削除は技術的に困難
   - 対策: ユーザー通知文面に明記（最大90日）

3. **イベント順序性**
   - Kafkaイベントの順序保証なし
   - 対策: 各消費側で冪等性を確保

4. **トークン失効タイミング**
   - 退会直後のAPI呼び出しが401になる可能性
   - 対策: フロント側で成功時に即座にローカルトークンクリア

5. **ログへのPII出力**
   - 監査ログに個人情報が残る可能性
   - 対策: ログ出力時に匿名化、アクセス制御

### 中リスク項目

1. **二重送信対策**
   - ボタン多重クリック、ネットワーク再送
   - 対策: UI側でloading/disabled、冪等性キー

2. **理由フィールドのPII**
   - 自由記述でPIIが含まれる可能性
   - 対策: 保存する場合は法務確認、最短保持期間

3. **再登録時の悪用**
   - ポイント不正取得等
   - 対策: プロダクトオーナーと対策検討

---

## 📝 次のステップ

### 即座に実施すべきこと

1. **プロダクトオーナーへの確認依頼**
   - タイトル/内容不一致の確認
   - 単独機能か会員情報編集の一部か
   - 再登録ポリシーの方針

2. **法務/コンプライアンスチームへの確認依頼**
   - データ保持期間（会社法・税法）
   - 保持すべきフィールドの詳細
   - 匿名化方法の妥当性
   - 再登録ポリシーの合法性
   - ユーザー通知文面の確認

3. **バックエンドチームへの確認依頼**
   - 退会APIの実装可能性
   - Kafkaトピック名の命名規則
   - アウトボックスパターンの実装状況
   - DLQ運用体制

### 確認完了後の実装フロー

1. API契約の確定（ec-site-shared-contracts）
2. バックエンドAPI実装
3. Kafkaイベントスキーマ確定
4. フロントエンド実装
5. 統合テスト
6. セキュリティレビュー
7. 法務レビュー
8. デプロイ

---

## 📚 参考資料

- [GDPR - Right to Erasure](https://gdpr-info.eu/art-17-gdpr/)
- [個人情報保護法](https://www.ppc.go.jp/)
- [会社法 第432条（会計帳簿の保存）](https://elaws.e-gov.go.jp/document?lawid=417AC0000000086)
- [法人税法 第150条の2（帳簿書類の保存）](https://elaws.e-gov.go.jp/document?lawid=340AC0000000034)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)

---

## 変更履歴

| 日付 | バージョン | 変更内容 | 作成者 |
|-----|----------|---------|-------|
| 2025-11-10 | 1.0 | 初版作成 | Devin |
