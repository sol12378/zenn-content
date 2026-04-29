---
casper_score: 83
created_at: '2026-04-29'
description: 「このAIエージェント、本当に信頼できるのか？」を定量化する仕組みを解説。FastAPI + Supabase + Next.jsで構築したLLM品質監査基盤MAGI
  Auditの設計思想とアーキテクチャを公開します。
emoji: 🔬
published: false
status: draft
target_word_count: 3500
title: AIエージェントの品質を可視化する——MAGI Auditのアーキテクチャ解説
topics:
- llm
- ai
- architecture
- 品質管理
- supabase
type: tech
---

# AIエージェントの品質を可視化する——MAGI Auditのアーキテクチャ解説

## はじめに

AIエージェントを本番環境に導入した翌週、あなたのチームは何をしているだろうか。

「なんか変な回答が来た」というSlackの報告を受けて、ログを掘り起こす。再現しようとしても、同じプロンプトで今度は正常な答えが返ってくる。結局「まあ大丈夫だろう」で終わる。

この状況が続くと、いつかクリティカルな問題が起きる。医療系チャットボットが誤った薬剤情報を返した。営業AIが存在しない製品スペックをクライアントに送った。法務AI が誤った判例を3件引用した。

問題が起きてから気づく——それがLLM運用の現実だ。

本記事では、この問題に正面から取り組んで構築した**MAGI Audit**のアーキテクチャを公開する。「AIエージェントの品質を継続的に、定量的に、自動的に可視化する仕組み」をどう設計したかを解説する。

---

## 1. なぜAIエージェントの品質は「見えない」のか

従来のソフトウェアであれば、品質の定義はシンプルだ。

```
入力 X → 処理 → 出力 Y（確定的）
```

ユニットテストを書けば品質を担保できる。CIが通れば安心できる。

しかし、LLMベースのAIエージェントはそうはいかない。

```
入力 X → LLM処理（非確定的） → 出力 Y'（毎回異なる）
```

同じプロンプトに対して、毎回異なる出力が生成される。「正しい出力」の定義自体が曖昧なケースも多い。さらに言えば、LLMは**プロバイダー側のモデル更新によって、知らないうちに挙動が変わる**。

これが品質を「見えなく」している本質的な理由だ。

### 従来の対処法とその限界

多くのチームが取っているアプローチ：

- **人力サンプリング**: 毎週数十件を手動確認 → 網羅性ゼロ
- **ユーザー報告頼り**: 問題が起きてから対応 → 完全に後手
- **モデル固定**: バージョンを固定して変化を防ぐ → スケールしない
- **プロンプト記録**: ログに残すだけ → 分析の仕組みがない

いずれも「モグラ叩き」に過ぎない。根本的な解決にはならない。

必要なのは、**ソフトウェアのCI/CDと同じ発想をLLM品質に適用すること**だ。

---

## 2. MAGI Auditの設計思想：品質の3軸可視化

MAGI Auditは、AIエージェントの品質を以下の3軸で継続的に計測する。

### 軸1：一貫性（Consistency）

同じプロンプトに対して、LLMが毎回「安定した」回答を返せているか。

```python
一貫性スコア = (多数派回答の出現数 / 総試行数) × 100
目標ライン: 80点以上
```

顧客サポートAIが月曜日に「返金可能」と答え、火曜日に「返金不可」と答える——このブレが業務に与えるダメージを定量化する。

### 軸2：正確性（Accuracy）

LLMが「既知の正解」に対して正しく回答できているか。ハルシネーション（事実と異なる回答）の発生率を測定する。

```python
正確性スコア = (正答数 / テストケース総数) × 100
目標ライン: 85点以上
```

ゴールドスタンダードデータセット（答えが確定している質問群）を定期的にLLMに投げ続けることで、モデルの精度劣化を早期検知できる。

### 軸3：安全性（Safety）

LLMが有害・不適切な出力を生成していないか。差別的表現、個人情報の流出、プロンプトインジェクションへの脆弱性を評価する。

```python
安全性スコア = (安全判定された出力数 / 総出力数) × 100
目標ライン: 97点以上（安全性は最高基準）
```

これら3軸を組み合わせた**総合品質スコア（MAGIスコア）**が、エージェントの「信頼度」を一つの数値で表現する。

---

## 3. アーキテクチャ全体像

```
┌──────────────────────────────────────────────────────┐
│                   クライアントアプリ                  │
│              （Web / モバイル / API）                 │
└──────────────────────┬───────────────────────────────┘
                       │ HTTP Request
                       ▼
┌──────────────────────────────────────────────────────┐
│               MAGI Gateway Layer                      │
│         （FastAPI / Vercel Edge Functions）            │
│                                                       │
│  ・/v1/messages プロキシ（Anthropic API互換）         │
│  ・リクエスト/レスポンス全ログをキャプチャ            │
│  ・SSE（Server-Sent Events）対応                     │
│  ・Stripe課金Gateway統合                             │
└──────────────────────┬───────────────────────────────┘
                       │ 非同期ログ書き込み
                       ▼
┌──────────────────────────────────────────────────────┐
│              Storage Layer (Supabase)                 │
│                                                       │
│  tables:                                              │
│  ├── organizations     テナント管理                   │
│  ├── api_keys          APIキー管理                    │
│  ├── request_logs      全リクエストログ（RLS保護）    │
│  ├── prompts           プロンプトテンプレート管理     │
│  └── audit_results     品質評価結果                   │
│                                                       │
│  ・Row Level Security（RLSポリシー）でテナント分離    │
│  ・PostgreSQL + リアルタイム更新                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│         MELCHIOR Evaluation Engine（評価エンジン）    │
│                    （Python / FastAPI）                │
│                                                       │
│  ・スケジューラ：毎日02:00 UTC 定期実行              │
│  ・一貫性テスト：同一プロンプト × 10回試行           │
│  ・正確性テスト：ゴールドスタンダードデータセット     │
│  ・安全性テスト：アドバーサリアルプロンプト50件       │
│  ・MAGIスコア算出 → audit_results に保存             │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│            Dashboard（Next.js + Supabase）             │
│                                                       │
│  ・テナントごとのMAGIスコア表示                      │
│  ・時系列グラフ（品質推移）                          │
│  ・アラート管理                                      │
│  ・request_logs リアルタイム閲覧                     │
└──────────────────────────────────────────────────────┘
```

各レイヤーの役割と実装上の判断を詳しく解説する。

---

## 4. Gateway Layer：すべてはリクエストのキャプチャから始まる

MAGI Auditの核心は「LLMのすべての通信を透過的にキャプチャする」ことだ。

Anthropic APIと互換性のある `/v1/messages` エンドポイントをプロキシとして実装している。クライアントは従来のAnthropicクライアントライブラリをそのまま使えるため、**コード変更なしで監査を開始できる**。

```python
# Gateway実装の核心（疑似コード）
@app.post("/v1/messages")
async def proxy_messages(request: Request, api_key: str = Depends(verify_api_key)):
    body = await request.json()

    # リクエストをログに記録
    log_id = await store_request_log(
        tenant_id=get_tenant_from_key(api_key),
        model=body.get("model"),
        prompt=body.get("messages"),
        timestamp=datetime.utcnow()
    )

    # 実際のAnthropicにプロキシ
    response = await anthropic_client.messages.create(**body)

    # レスポンスもログに記録
    await update_request_log(
        log_id=log_id,
        response=response.content,
        tokens_used=response.usage,
        latency_ms=elapsed
    )

    return response
```

SSE（ストリーミング）対応は実装の難所だった。`async for` でチャンクを受け取りながら、並行してログバッファに蓄積し、ストリーム完了後にDBへ一括書き込みするアーキテクチャを採用している。

### なぜEdge Functionsではなくサーバーを選んだか

当初はVercel Edge Functionsでの実装を検討したが、以下の理由でFastAPIサーバーに切り替えた：

- **実行時間制限**: Edge Functionsは最大25秒。GPT-4o等の長い推論では超過するリスクがある
- **Python依存ライブラリ**: ML系ライブラリがEdge環境では動作しない
- **ステートフルな評価エンジン**: MELCHIORはインメモリでデータセットをキャッシュする必要がある

---

## 5. MELCHIOR評価エンジン：スコアリングアルゴリズムの実装

MAGI Auditの評価エンジンをMELCHIORと名付けた。旧約聖書の東方の三博士の一人から取った名前だ（MAGI = Melchior, Caspar, Balthazar）。

### 一貫性スコアの計算

```python
async def calculate_consistency_score(
    prompt: str,
    model: str,
    trials: int = 10
) -> float:
    """同一プロンプトを複数回実行し、一貫性を算出"""
    outputs = []

    for _ in range(trials):
        response = await anthropic_client.messages.create(
            model=model,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=500,
            temperature=0  # 決定論的出力のために0を使用
        )
        outputs.append(normalize_output(response.content[0].text))

    # クラスタリングで意味的に同じ出力をグループ化
    clusters = semantic_cluster(outputs, threshold=0.85)
    largest_cluster = max(len(c) for c in clusters)

    return (largest_cluster / trials) * 100
```

`temperature=0` に設定しても完全に同一の出力が保証されるわけではない。そのため、文字列の完全一致ではなく、Sentence Transformersを用いた意味的クラスタリングで「実質的に同じ意味の出力」をグループ化している。

### 安全性テストのアドバーサリアルアプローチ

安全性テストには50件のアドバーサリアルプロンプトを用いる。カテゴリは以下の通り：

| カテゴリ | 件数 | 内容 |
|----------|------|------|
| プロンプトインジェクション | 15 | システムプロンプト書き換え試行 |
| 個人情報要求 | 10 | 他ユーザーのデータ取得試行 |
| 有害コンテンツ誘導 | 15 | 差別・暴力・違法行為への誘導 |
| 機密情報抽出 | 10 | APIキー・パスワード情報の取得試行 |

これらのプロンプトに対してLLMが適切に「拒否」または「安全な形で対応」した場合を「安全」と判定する。

### アラートの閾値設計

```python
ALERT_THRESHOLDS = {
    "consistency": {"warning": 80, "critical": 65},
    "accuracy":    {"warning": 85, "critical": 70},
    "safety":      {"warning": 97, "critical": 90},  # 安全性は高め
}

async def check_and_alert(tenant_id: str, scores: dict):
    for metric, score in scores.items():
        thresholds = ALERT_THRESHOLDS[metric]

        if score < thresholds["critical"]:
            await send_alert(
                tenant_id=tenant_id,
                level="CRITICAL",
                message=f"{metric}スコアが危険域（{score:.1f}）。即座に確認が必要",
                channels=["slack", "email", "pagerduty"]
            )
        elif score < thresholds["warning"]:
            await send_alert(
                tenant_id=tenant_id,
                level="WARNING",
                message=f"{metric}スコアが低下（{score:.1f}）",
                channels=["slack"]
            )
```

安全性の閾値を97%に設定しているのは意図的だ。「3件に1件くらい問題がある」が許容されるメトリクスは品質管理の対象であり、安全性はそうではない。

---

## 6. Storage Layer：Supabase × RLSによるマルチテナント設計

データ層ではSupabase（PostgreSQL）を採用した。最大のポイントは**Row Level Security（RLS）によるテナント分離**だ。

```sql
-- テナントは自分のリクエストログのみ閲覧可能
CREATE POLICY "tenant_isolation" ON request_logs
  USING (
    organization_id = (
      SELECT organization_id
      FROM api_keys
      WHERE key_hash = current_setting('app.api_key_hash')
    )
  );
```

アプリケーションレベルでテナント分離を実装すると、コードのバグがデータ漏洩に直結する。RLSをDBレベルに実装することで、**アプリケーションのバグがあってもテナントのデータは保護される**。

### リクエストログのスキーマ設計

```sql
CREATE TABLE request_logs (
  id              UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  organization_id UUID NOT NULL REFERENCES organizations(id),
  api_key_id      UUID NOT NULL REFERENCES api_keys(id),
  model           TEXT NOT NULL,

  -- プロンプトと出力
  input_messages  JSONB NOT NULL,
  output_content  JSONB,

  -- パフォーマンス
  input_tokens    INTEGER,
  output_tokens   INTEGER,
  latency_ms      INTEGER,

  -- 品質評価
  safety_flagged  BOOLEAN DEFAULT FALSE,
  audit_score     NUMERIC(5,2),

  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- テナントIDと時刻の複合インデックス（ダッシュボードの高速化）
CREATE INDEX idx_request_logs_org_time
  ON request_logs(organization_id, created_at DESC);
```

30日以上経過したログは自動アーカイブ（S3移行）するパーティション設計も実装しているが、本記事では詳細は割愛する。

---

## 7. Dashboardの設計：数字を「意味のある情報」に変換する

データを集めて計算しても、それを見る人間が意味を理解できなければ価値はない。

MAGI Auditのダッシュボードは次の4つの情報を最優先で表示する：

```
┌─────────────────────────────────────────────┐
│  MAGI スコアサマリー（2026-04-29）          │
├─────────────────────────────────────────────┤
│                                             │
│  総合スコア        87 / 100  ✅ Healthy     │
│                                             │
│  一貫性スコア:   84/100  ✅                 │
│  正確性スコア:   91/100  ✅                 │
│  安全性スコア:   98/100  ✅                 │
│                                             │
│  前日比:  +2.1 ↑   過去7日最低: 81/100     │
│                                             │
├─────────────────────────────────────────────┤
│  アクティブアラート       0件               │
│  過去24時間リクエスト  4,832件              │
│  コスト推定（本日）    ¥12,400             │
└─────────────────────────────────────────────┘
```

**スコアが下がったときに「何が起きたのか」を即座に調査できること**を設計原則にした。スコアのグラフをクリックすると、その日時のリクエストログへドリルダウンできる。

### Supabase Realtimeを活用したリアルタイム更新

```typescript
// Next.jsコンポーネント内
useEffect(() => {
  const channel = supabase
    .channel('audit_results')
    .on(
      'postgres_changes',
      { event: 'INSERT', schema: 'public', table: 'audit_results' },
      (payload) => {
        setLatestScore(payload.new);
        if (payload.new.total_score < CRITICAL_THRESHOLD) {
          showCriticalAlert(payload.new);
        }
      }
    )
    .subscribe();

  return () => supabase.removeChannel(channel);
}, []);
```

評価エンジンがスコアを更新すると、ダッシュボードにリアルタイムで反映される。ページをリフレッシュする必要がない。

---

## 8. 運用してわかったこと：設計で後悔した3点

実際に運用して気づいた、設計の反省点を正直に書いておく。

### 反省1：テストデータセットのメンテナンスコスト

ゴールドスタンダードデータセット（正確性テスト用）の品質が監査精度を直接左右する。初期に用意した50件は半年で陳腐化した。**テストデータ自体のバージョン管理とメンテナンスサイクルを最初から設計すべきだった**。

現在は `prompts` テーブルでテストデータをバージョン管理し、新しいプロンプトを追加したら自動でテストケースを生成するフローを実装している。

### 反省2：評価の実行コスト

一貫性テストは同じプロンプトを10回実行する。100件のテストケース × 10回 = 1,000リクエスト/日。Claude Sonnetを使うと月約15万円の追加コストが発生した。

対策として、テストを「高リスク（毎日実行）」「中リスク（週1）」「低リスク（月1）」に分類するリスクベースのスケジューリングを導入した。

### 反省3：スコアの「意味」を組織に伝えるのが難しい

MAGIスコアが79点から74点に下がったとき、エンジニアには問題の深刻さが伝わる。しかし経営層には「74点って良いの？悪いの？」という反応が返ってくる。

**数字だけでなく「このスコアでどのくらいの確率で問題が起きるか」という実害予測を出せるようにした**のが転機だった。「一貫性74点 = 顧客の約4人に1人が矛盾した回答を受け取っている状態」という表現が刺さった。

---

## まとめ：「見えない品質」を見えるようにする

AIエージェントの品質問題は、**起きてから気づく時代を終わらせる**必要がある。

MAGI Auditが実現したのは：

- ✅ **ゼロコード導入**: `/v1/messages` エンドポイントを切り替えるだけ
- ✅ **3軸品質可視化**: 一貫性・正確性・安全性を定量スコアで把握
- ✅ **リアルタイムアラート**: 品質劣化を検知した瞬間に通知
- ✅ **マルチテナント対応**: RLSによる完全なテナント分離
- ✅ **監査証跡**: コンプライアンス対応に必要なリクエストログの永続化

2026年8月に本格適用される日本AI推進法でも、AIシステムの透明性確保と監査証跡の保持が義務化される方向だ。品質監査の仕組みを今から持っておくことは、単なるエンジニアリングの問題ではなく、事業継続のリスク管理でもある。

MAGI Auditは現在βテスト段階で、無料トライアルも提供中です。

👉 [MAGI for Devs — ドキュメント・サインアップ](https://magi.nerv.ai)

---

## 参考資料

- [LLMの品質監査を自動化する：メトリクス定義から実装まで](https://zenn.dev/kei_concierge/articles/llm-quality-audit-automation)
- [Supabase Row Level Security 公式ドキュメント](https://supabase.com/docs/guides/database/row-level-security)
- [Claude Mythos時代の企業責任——自律AIリスク対策はセキュリティ責任者の必須課題](https://qiita.com/kei-concierge/items/f4c72296665c69f2b457)

---

*MAGI AuditはNERV戦略部門が開発・運用するAI品質監査プラットフォームです。本記事に記載のコードは説明用の疑似コードを含みます。*
