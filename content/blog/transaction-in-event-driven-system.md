---
title: "ハイブリッドなトランザクショナル・アウトボックス方式のイベント配信パターン"
description: '本記事では、3つの専用テーブルを用いて異なるイベント配信要件に対応するハイブリッドなイベント配信パターンを紹介します。レイテンシに敏感なイベントにはフォールバックのスキャンを伴う即時発行、時刻ベースのイベントには定期ポーリング、大量データにはバッチ集約を用います。'
tags:
  - 分散システム
  - プログラミング
pubDate: 'Oct 5 2025'
heroImageId: '14627615-0daa-4380-1161-88c270934400'
heroImageSource: 'Pixiv'
heroImageSourceUrl: 'https://www.pixiv.net/artworks/140516731'
heroImageAuthor: 'ぺんたごん'
heroImageAuthorUrl: 'https://www.pixiv.net/users/10950860'
pinned: true
---

> この記事は、<https://blog.plr.moe/blog/transaction-in-event-driven-system>から翻訳したものである。

## 問題: 信頼できるイベント発行は難しい

分散システムでは、イベントを確実に発行することは見た目以上に難しい課題です。次の2つをアトミックに行う必要があります。
1. データベースを更新する
2. メッセージキューにイベントを発行する

しかし、これらは2つの別々のシステムです。データベースのトランザクションは成功したのにメッセージキューがダウンしていたら、どうなるでしょうか？イベントを失います。逆にメッセージは発行されたのにデータベースがロールバックしたら？幻のイベント (phantom event) が生まれます。

これが **二重書き込み問題 (dual-write problem)** であり、イベント駆動アーキテクチャの悩みの種です。

## トランザクショナル・アウトボックス

[トランザクショナル・アウトボックスパターン](https://microservices.io/patterns/data/transactional-outbox.html) は、次の方法でこれを解決します。
1. 同一のデータベーストランザクション内で、イベントをアウトボックステーブルに書き込む
2. 別プロセスがアウトボックスを読み取り、メッセージキューに発行する
3. 配信に成功したイベントを発行済みとしてマークする

これは機能しますが、トレードオフがあります。
- **ポーリングはレイテンシを増やす**: 数秒ごとにデータベースを確認するため、イベント配信が遅延します
- **Change Data Capture は複雑**: Debezium のようなリアルタイム CDC ソリューションは強力ですが、運用上のオーバーヘッドが増えます
- **MQ を流れる大きなペイロード**: イベントデータ全体がメッセージキューを通過します

## 実例: マルチプロバイダ対応の AI チャットプラットフォーム

具体例で考えてみましょう。複数の AI プロバイダ (OpenAI、Anthropic、Google など) に対応するチャットプラットフォームを構築するとします。このシステムは次を扱う必要があります。

- **特定の時刻に失効する**ユーザーのサブスクリプション
- 数百万回の API 呼び出しにわたる**トークン使用量の追跡**
- サービス有効化を引き起こす**決済処理**
- Redis にキャッシュされ、最終的に**永続化**が必要な AI 応答

これらはそれぞれ異なる一貫性とレイテンシの要件を持っており、私たちのハイブリッドパターンにとって格好のケーススタディとなります。

## より良いアプローチ: Claim Check とバッファを備えたハイブリッドアウトボックス

私は、異なる配信メカニズムを専用テーブルに分離するパターンを実装しました。核心となる洞察は、イベントの種類ごとに要件が異なり、それぞれ最適化されたストレージを使うべきだ、ということです。

```
┌─────────────────────────────────────────────────────────────┐
│                     AIチャットサービス                      │
└───┬─────────────────┬─────────────────┬─────────────────────┘
    │                 │                 │
    │ 1. 即時イベント │ 2. 将来の       │ 3. 大量データ
    │                 │ イベントを予約  │ を蓄積
    ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────┐
│                      3つの専用テーブル                      │
├─────────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────────┐  │
│  │ outbox_events: ハイブリッド (AI応答, 決済)            │  │
│  │   → 即時発行 + 30秒ごとのスキャナfallback             │  │
│  │   → インデックス: (status, created_at) で高速スキャン │  │
│  │   → クリーンアップ: 30日後にアーカイブ                │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ scheduled_events: 将来のトリガー (サブスク)           │  │
│  │   → scheduled_at <= NOW() をポーリング                │  │
│  │   → インデックス: (scheduled_at, status) で時刻検索   │  │
│  │   → クリーンアップ: 実行後に削除                      │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ accumulation_buffer: バッチ処理 (トークン使用量)      │  │
│  │   → MQ不要、5分ごとに直接集約                         │  │
│  │   → インデックス: (user_id, created_at) でグループ化  │  │
│  │   → クリーンアップ: 集約後に即削除                    │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ { event_id: UUID または series }
                          │
                          ▼
                    ┌──────────┐
                    │    MQ    │
                    └─────┬────┘
                          │
                          ▼
                    ┌──────────┐
                    │ Consumer │
                    └──────────┘
```

最悪のケース (MQ 障害、プロセスクラッシュ、ネットワーク分断) でも、すべてのイベントは最終的に処理されます。データベースのトランザクションが、イベントが決して失われないことを保証します。

コンシューマは ID でイベントを取得しステータスを確認するため、重複配信は自然に処理されます。これは、実際にはすでに送信済みのイベントをスキャナが再発行してしまう場合に重要です。

また、複雑な CDC インフラが不要なため、メンテナンスは非常に容易です。

核心となる洞察は、**イベントの種類ごとに異なるストレージ戦略が必要である**という点です。

**データベーススキーマの設計判断:**
- `outbox_events`: `scheduled_at` カラムは不要 (必要ない)
- `scheduled_events`: 将来のトリガーのために `scheduled_at` が必要
- `accumulation_buffer`: 最小限のスキーマ。高速な挿入のために `BIGSERIAL` を使い、ステータスフィールドは不要。処理後に削除するだけでよい。

### ハイブリッドイベント (ホットパス): AI 応答の永続化

ユーザーがメッセージを送信すると、AI の応答はまず即時取得のために Redis にキャッシュされます。これを次の目的で PostgreSQL に永続化する必要があります。
- 長期保存と検索
- 分析および学習データ
- 監査証跡

これは**レイテンシに敏感**ですが、致命的ではありません。即時発行が失敗しても、30秒の遅延は許容できます。

```rust
async fn save_ai_response(
    pool: &PgPool,
    mq: &MessageQueue,
    response: AiResponse,
) -> Result<()> {
    let mut tx = pool.begin().await?;
    // ここで何か処理をしてからイベントを発行する
    let event = sqlx::query_as!(
        OutboxEvent,
        r#"
        INSERT INTO outbox_events (event_type, payload, status)
        VALUES ($1, $2, $3)
        RETURNING id, event_type, payload, status, created_at, sent_at, processed_at
        "#,
        "ai_response_ready",
        sqlx::types::Json(EventPayload::AiResponseReady { /* ... */ }) as _,
        EventStatus::Pending as EventStatus,
    )
    .fetch_one(&mut *tx)
    .await?;
    tx.commit().await?;
    
    // コミット後に即時発行 (ノンブロッキング)
    let event_id = event.id;
    let mq = mq.clone();
    tokio::spawn(async move {
        if let Err(e) = eager_publish(&mq, event_id).await {
            tracing::warn!("AI応答の即時発行に失敗: {:?}", e);
            // スキャナが30秒以内に拾う
        }
    });
    
    Ok(())
}
```

**なぜここでハイブリッドなのか？** ほとんどの AI 応答は、リアルタイムの使用状況を表示する分析ダッシュボードのために迅速な永続化を必要とします。即時発行は 99% のケースで1秒未満のレイテンシを実現し、スキャナが結果整合性を保証します。

### 予約イベント (コールドパス): サブスクリプションの失効

サブスクリプションは既知の将来時刻に失効します。**即時発行は不要**で、期限が来たイベントをポーリングするだけで十分です。

```rust
async fn create_subscription(
    pool: &PgPool,
    user_id: Uuid,
    plan: SubscriptionPlan,
    duration_days: i32,
) -> Result<Subscription> {
    let mut tx = pool.begin().await?;
    
    let expires_at = Utc::now() + Duration::days(duration_days as i64);
    
    // サブスクリプションを作成
    let subscription = sqlx::query_as!(
        Subscription,
        "INSERT INTO subscriptions (user_id, plan, expires_at, status)
         VALUES ($1, $2, $3, $4)
         RETURNING *",
        user_id,
        plan as SubscriptionPlan,
        expires_at,
        SubscriptionStatus::Active as SubscriptionStatus,
    )
    .fetch_one(&mut *tx)
    .await?;
    
    // 専用テーブルに失効イベントを予約
    sqlx::query!(
        r#"
        INSERT INTO scheduled_events (event_type, payload, scheduled_at, status)
        VALUES ($1, $2, $3, $4)
        "#,
        "subscription_expiry",
        ScheduledEventPayload::SubscriptionExpiry{ /* ... */ },
        expires_at,
        EventStatus::Pending as EventStatus,
    )
    .execute(&mut *tx)
    .await?;
    
    tx.commit().await?;
    
    Ok(subscription)
}

// スキャナは毎分実行され、期限切れのサブスクをチェックする
async fn scan_scheduled_events(pool: &PgPool, mq: &MessageQueue) -> Result<()> {
    let now = Utc::now();
    
    let due_events = sqlx::query_as!(
        ScheduledEvent,
        r#"
        SELECT id, event_type, payload, scheduled_at,
               status as "status: EventStatus", sent_at, processed_at
        FROM scheduled_events
        WHERE status = $1 AND scheduled_at <= $2
        LIMIT 100
        "#,
        EventStatus::Pending as EventStatus,
        now,
    )
    .fetch_all(pool)
    .await?;
    
    for event in due_events {
        publish_scheduled_event(pool, mq, event.id)?;
    }
    
    Ok(())
}

// コンシューマ: サブスク失効を処理
async fn handle_subscription_expiry(
    pool: &PgPool,
    event: OutboxEvent,
) -> Result<()> {
    let payload: EventPayload = serde_json::from_value(event.payload)?;
    
    if let EventPayload::SubscriptionExpiry { subscription_id, user_id } = payload {
        // サブスクリプションのステータスを更新
        sqlx::query!(
            "UPDATE subscriptions SET status = $1 WHERE id = $2",
            SubscriptionStatus::Expired as SubscriptionStatus,
            subscription_id,
        )
        .execute(pool)
        .await?;
        
        // API アクセスを取り消す
        revoke_api_keys(user_id).await?;
        
        // 通知メールを送信
        send_expiry_notification(user_id).await?;
    }
    
    Ok(())
}
```

**なぜ予約のみなのか？** サブスクリプションの失効は**決して緊急ではありません**。ちょうど深夜0時に起きようが60秒後になろうが問題になりません。毎分のポーリングで十分であり、即時発行の複雑さを完全に回避できます。

### ポーリングのみのイベント (MQ なし): トークン使用量の集約

AI API を呼び出すたびにトークン使用量が発生します。これをリアルタイムで追跡すると、1日あたり数百万件のイベントでシステムが圧迫されてしまいます。代わりに、**使用量をローカルに蓄積し、定期的にバッチ更新**します。

```rust
// バックグラウンドジョブは5分ごとに実行される
async fn accumulate_token_usage(pool: &PgPool) -> Result<()> {
    // accumulation buffer から未処理のトークン使用量をすべて取得
    let usage_records = sqlx::query!(
        r#"
        SELECT id, user_id, tokens, model, created_at
        FROM accumulation_buffer
        LIMIT 10000
        "#,
    )
    .fetch_all(pool)
    .await?;
    
    if usage_records.is_empty() {
        return Ok(());
    }
    
    // user_id でグループ化しトークンを合計
    let mut usage_by_user: HashMap<Uuid, i32> = HashMap::new();
    for record in &usage_records {
        *usage_by_user.entry(record.user_id).or_insert(0) += record.tokens;
    }
    
    // ユーザーのクォータを一括更新
    let mut tx = pool.begin().await?;
    // ここで何か処理をする
    tx.commit().await?;
    
    tracing::info!(
        "{} 件のトークン使用量レコードを {} 人のユーザー分集約しました",
        usage_records.len(),
        usage_by_user.len()
    );
    
    Ok(())
}

// アキュムレータを定期的に実行
async fn run_token_accumulator(pool: PgPool) {
    let mut interval = tokio::time::interval(Duration::from_secs(300)); // 5分
    
    loop {
        interval.tick().await;
        
        if let Err(e) = accumulate_token_usage(&pool).await {
            tracing::error!("トークンアキュムレータのエラー: {:?}", e);
        }
    }
}
```

**なぜポーリングのみなのか？** トークン使用量の追跡には次の特徴があります。
- **レイテンシに敏感でない**: ユーザーはダッシュボードでクォータを確認するのであって、リアルタイムではない
- **大量**: 1日あたり数百万件の小さなイベント
- **自然にバッチ化できる**: 5分ごとの蓄積でまったく問題ない

ここで MQ を使うのは無駄です。`accumulation_buffer` テーブルは**シンプルなバッチ処理の仕組み**として機能し、定期ポーリングによって効率的に集約されます。

### 適切なツールを選ぶ

```txt
このイベントは outbox_events、scheduled_events、accumulation_buffer のどれを使うべきか？

├─ 実行時刻が既知の将来イベントか？
│  └─ YES → scheduled_events
│  
├─ 大量 (>1000/秒) かつ集約可能か？
│  └─ YES → accumulation_buffer
│  
└─ 低レイテンシ配信が必要か？
   ├─ YES → outbox_events (ハイブリッド)
   └─ NO → classic_outbox (ポーリングのみ)
```

| ユースケース | テーブル | パターン | レイテンシ | 理由 |
|----------|-------|---------|---------|-----|
| AI 応答の同期 | `outbox_events` | ハイブリッド (即時 + スキャナ) | <1秒 (99%)、<30秒 (99.99%) | ユーザー向け分析には速度が必要 |
| 決済処理 | `outbox_events` | ハイブリッド (即時 + スキャナ) | <1秒 (99%)、<30秒 (99.99%) | サービス有効化は迅速であるべき |
| サブスクリプションの失効 | `scheduled_events` | 予約 (ポーリングのみ) | 約60秒 | 正確なタイミングは重要でない |
| トークン使用量 | `accumulation_buffer` | ポーリングのみ (MQ なし) | 約5分 | 大量で、時間にシビアでない |

## 採用する理由・しない理由

### なぜ1つのテーブルにしないのか？

3つの専用テーブルを使うことで、単一の統合テーブルでは得られない大きなアーキテクチャ上の利点が得られます。

**1. 最適化されたインデックス**
- `outbox_events`: 直近の失敗を高速にスキャンするための `(status, created_at)` インデックス
- `scheduled_events`: 効率的な時刻ベースのクエリのための `(scheduled_at, status)` インデックス
- `accumulation_buffer`: 集約時の高速なグループ化のための `(user_id, created_at)` インデックス

単一のテーブルでは、異なるアクセスパターンをカバーする複数のインデックスが必要となり、インデックスの肥大化と書き込みの低速化を招きます。

**2. 独立したクリーンアップ戦略**
- `outbox_events`: 30日後にアーカイブ (監査証跡)
- `scheduled_events`: 実行後すぐに削除 (履歴的価値なし)
- `accumulation_buffer`: 集約後に削除 (すでに `users.tokens_used` に反映済み)

これにより、テーブルが無限に肥大化するのを防ぎ、クエリ性能を一定に保てます。

**3. 分離されたパフォーマンス特性**
- 大量のトークン使用量の書き込みが、レイテンシに敏感な AI 応答イベントをブロックしない
- 予約イベントのスキャンが、アウトボックススキャナの性能に干渉しない
- 各テーブルを個別にチューニングできる (vacuum 設定、autovacuum の閾値)

**4. 明確な運用上の境界**
異なるチームやサービスが異なるテーブルを所有できます。
- 決済チーム: `outbox_events` (クリティカルパス)
- サブスクリプションチーム: `scheduled_events` (バックグラウンドジョブ)
- 分析チーム: `accumulation_buffer` (データパイプライン)

### なぜイベント ID だけを渡すのか？

ペイロード全体ではなくイベント ID だけをメッセージキューに発行すること (別名 [claim check](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check)) で、いくつかの利点が得られます。

**1. メッセージキューの負荷軽減**
- 巨大になりうるイベントペイロードに対し、(UUID だけの) 極小メッセージ
- MQ とコンシューマ間のネットワーク帯域使用量が減る
- 小さなメッセージにより、MQ は大幅に高いスループットを処理できる

**2. メッセージサイズ制限の回避**
- ほとんどのメッセージキューにはサイズ制限がある (例: RabbitMQ はデフォルト 128MB、SQS は 256KB)
- 埋め込みや大きなコンテキストを含む AI 応答は、これらの制限を超えることがある
- PostgreSQL ではイベントペイロードのサイズは無制限

**3. 信頼できる唯一の情報源 (Single Source of Truth)**
- イベントデータはデータベースにのみ存在し、MQ に複製されない
- イベント処理ロジックを更新しても、常に最新のデータを問い合わせられる
- コンシューマが遅い場合でも、古いペイロードの問題が起きない

**4. より良いリソース活用**
- データベースはインデックス付きの構造化データの保存に最適化されている
- メッセージキューは保存ではなく高速な配信に最適化されている
- 各システムが最も得意なことを行う

**5. デバッグの簡素化**
- イベントの詳細を確認するにはデータベースを直接クエリすればよい
- 調査のために MQ からメッセージをキャプチャする必要がない
- イベント履歴が MQ の保持期間とは独立して保存される

トレードオフは、コンシューマ側でイベントごとにデータベースクエリが1回増えることですが、毎秒 (数百万ではなく) 数千件というこのユースケースでは、利点に比べれば無視できる程度です。

## トレードオフと考慮事項

### 重複配信の可能性
即時発行は成功したがステータスの更新に失敗した場合、スキャナが再発行します。コンシューマは**冪等でなければなりません**。AI チャットプラットフォームの場合:
- AI 応答の同期: 更新前に `content` がすでに設定されているか確認する
- 決済処理: 決済ゲートウェイの冪等キーを使う
- トークンの集約: 本質的に冪等 (ID ごとにすでに集約されている)

### データベース負荷の増加
ハイブリッドなコンシューマはすべて、イベントの詳細を取得するためにデータベースをクエリする必要があります。高スループットの場合:
- コンシューマのクエリにはリードレプリカを使う
- コネクションプーリングを追加する (例: pgBouncer)
- 頻繁にアクセスされるイベントを Redis にキャッシュする

**ここで3テーブル構成が役立つ点:**
- 各テーブルが小さい = キャッシュヒット率が向上する
- スキャナ同士が同じインデックスを奪い合わない
- 書き込みの多い `accumulation_buffer` が `outbox_events` の読み取りをブロックしない

### イベントのクリーンアップ戦略
各テーブルは、その目的に応じた独自のクリーンアップ戦略を持ちます。

- outbox_events のクリーンアップ: 30日後にアーカイブ (監査証跡)
- scheduled_events のクリーンアップ: 実行後すぐに削除
- accumulation_buffer のクリーンアップ: 集約後に削除 (前述)。これは集約処理の中でインラインに行われます。

**テーブルごとのクリーンアップの利点:**
- 「万能な」保持ポリシーによる妥協が不要
- テーブルが小さい = クエリが高速で vacuum の性能も向上
- ユースケースごとに明確なデータライフサイクル管理

### スキャナ間隔のチューニング
30秒のバックログ閾値と5分のトークン集約間隔は設計上の選択です。要件に応じてチューニングしてください。

| テーブル | スキャナの種類 | 間隔 | 根拠 |
|-------|-------------|----------|-----------|
| `outbox_events` | ハイブリッドのバックログスキャナ | 30〜60秒 | レイテンシとデータベース負荷のバランス |
| `scheduled_events` | 予約イベントスキャナ | 1分 | サブスクの失効に1分未満の精度は不要 |
| `accumulation_buffer` | トークンアキュムレータ | 5〜10分 | 大量で、ユーザーがクォータを確認する頻度は低い |

**ヒント**: 最初は長めの間隔から始め、実際のユーザーのニーズに応じて短くしていきましょう。早すぎる最適化はリソースの無駄です。

## Rust がこのパターンをより良くする

上記のコード例に加えて、Rust はこのアーキテクチャに独自の利点をもたらします。

### 型安全なスキーマとイベント状態

`serde` を使ってイベントペイロードに強い型付けを施すことで、あらゆる種類のシリアライズ・デシリアライズの脆弱性を排除でき、同時に ADT (代数的データ型) とパターンマッチングによって列挙的なデータをエレガントに扱えるようになります。

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, sqlx::Type)]
#[sqlx(type_name = "event_status", rename_all = "lowercase")]
pub enum EventStatus {
    Pending,    // 配信待ち
    Processed,  // コンシューマ完了
    Failed,     // 恒久的な失敗
}
```

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "event_name")]
pub enum EventPayload {
    AiResponseReady {
        conversation_id: Uuid,
        message_id: Uuid,
        redis_key: String,
        provider: AiProvider,
        model: String,
        tokens_used: i32,
    },
    PaymentCompleted {
        user_id: Uuid,
        payment_id: Uuid,
        plan: SubscriptionPlan,
        amount: Decimal,
    },
    // ... その他のハイブリッドイベント
}
```

```rust
#[derive(Debug, sqlx::FromRow)]
pub struct OutboxEvent {
    pub id: Uuid,
    pub event_type: String,
    pub payload: sqlx::types::Json<EventPayload>,
    pub status: EventStatus,
    pub created_at: DateTime<Utc>,
    pub sent_at: Option<DateTime<Utc>>,
    pub processed_at: Option<DateTime<Utc>>,
}

#[derive(Debug, sqlx::FromRow)]
pub struct ScheduledEvent {
    // ...
}

#[derive(Debug, sqlx::FromRow)]
pub struct AccumulationRecord {
    // ...
}
```

### コンパイル時のデータベーススキーマ検証

これはキラー機能です。`cargo build` を実行すると、sqlx は次を行います。
1. 開発用データベースに接続する
2. 実際のスキーマに対してすべてのクエリを検証する
3. 型安全な Rust の構造体を生成する
4. デプロイ前に不整合を検出する

```
$ cargo sqlx prepare
Connecting to database...
Building query metadata for 94 queries...
Successfully saved query metadata to .sqlx/

$ cargo build
   Compiling outbox-service v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 1.14s
```

データベーススキーマを変更すると、クエリはコンパイル時に壊れます。
```
$ cargo build
error: error returned from database: column "scheduled_for" does not exist
  --> src/scanner.rs:23:5
```

これにより、本番環境のバグの一群を丸ごと排除できます。

## このパターンを使うべきとき

**次のような場合に最適です:**
- イベントの種類ごとにレイテンシ要件が異なる
- MQ のオーバーヘッドを必要としない大量イベントがある
- スケジュールが既知の時刻ベースのイベントがある
- 監査可能性とイベントの再生 (replay) が必要
- 強い型付けを要求する Rust/TypeScript スタック

**次のような場合には不向きです:**
- すべてのイベントが同一の要件を持つ (よりシンプルな CDC や純粋なポーリングを使う)
- データセンター間レプリケーションが必要 (イベントストリーミング基盤を検討する)
- イベントが本当に一時的で永続化の必要がない
- チームにスキャナを運用する余力がない
