---
article_id: ARTICLE-010
author: kei
created_at: 2026-04-13
emoji: 🔍
platform: zenn
published: true
published_at: 2026-04-18 09:00:00+09:00
status: ready-to-publish
target_publish: 2026-04-18
title: AIガバナンスツールを自作した話 — LLM監査の実装記録
topics:
- LLM
- FastAPI
- Supabase
- AI
- 個人開発
type: tech
---

# AIガバナンスツールを自作した話 — LLM監査の実装記録

## なぜ作ろうと思ったか

LLMをプロダクションで使い始めると、必ずある問題に直面する。

「なんでこのレスポンス、昨日より精度が落ちてるんだ？」

ログを遡ろうとするが、API呼び出しの記録はAnthropicのコンソールには入力・出力の生データが残っていない。どのリクエストで問題が起きたのか、追跡する手段がない。

私が最初にこれを感じたのは、Claude Codeを使ったコード補完の精度が日によってバラつくことに気づいたときだ。同じプロンプトのはずなのに、ある日はすっきりとしたコードを出力し、ある日は冗長な回答を返す。

その原因を特定しようとしたとき、「監査できる状態を作らないといけない」と強く思った。

---

## 「AIガバナンス」という言葉について

「ガバナンス」というと、大企業のリスク管理や規制対応の話に聞こえるかもしれない。

でも個人開発者にとっての AIガバナンスは、もっとシンプルだ。

**「何が起きているかを把握し、改善できる状態にすること」**

それだけだ。

把握できていなければ改善できない。改善できなければコストと品質の両方がブラックボックスになる。

---

## 設計の出発点：何を記録したいか

最初に「何を記録したいか」を整理した。

### 必須データ

1. **リクエスト/レスポンスのペア** — 何を聞いて、何が返ってきたか
2. **トークン数** — 入力・出力それぞれ
3. **使用モデル** — どのモデルを呼んだか
4. **レイテンシ** — 何秒かかったか
5. **タイムスタンプ** — いつ呼んだか
6. **コスト換算** — いくらかかったか（モデル価格 × トークン数で計算）

### あると良いデータ

7. **プロジェクト/タグ** — どのコンテキストでの呼び出しか
8. **品質スコア** — レスポンスの品質を何らかの形で定量化
9. **エラー情報** — 失敗したリクエストの詳細

---

## 技術スタック選定

### バックエンド：FastAPI

Pythonのエコシステムとの相性が良く、非同期処理に強い。LLM関連の処理は待機時間が長いので、asyncioベースのFastAPIは自然な選択だった。

```python
from fastapi import FastAPI
from pydantic import BaseModel
import time

app = FastAPI()

class LLMRequest(BaseModel):
    model: str
    messages: list
    project: str = "default"

@app.post("/proxy/messages")
async def proxy_anthropic_messages(request: LLMRequest):
    start_time = time.time()

    # Anthropic APIへの実際のリクエスト
    response = await anthropic_client.messages.create(
        model=request.model,
        messages=request.messages,
        max_tokens=4096
    )

    latency = time.time() - start_time

    # ログ記録
    await log_request(
        model=request.model,
        input_tokens=response.usage.input_tokens,
        output_tokens=response.usage.output_tokens,
        latency=latency,
        project=request.project
    )

    return response
```

### データベース：Supabase

PostgreSQLベースのBaaSで、リアルタイム購読とRow Level Securityが使いやすい。個人プロジェクトの規模では、フルマネージドなSupabaseが運用コストを最小化できる。

ログのスキーマはシンプルに始めた：

```sql
CREATE TABLE llm_requests (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    model TEXT NOT NULL,
    input_tokens INT NOT NULL,
    output_tokens INT NOT NULL,
    cost_usd DECIMAL(10, 6),
    latency_ms INT,
    project TEXT DEFAULT 'default',
    quality_score DECIMAL(3, 2),
    request_hash TEXT  -- 重複検出用
);
```

### フロントエンド：Next.js + Vercel

ダッシュボードはNext.jsで構築し、Vercelにデプロイした。Chart.jsでコスト推移グラフを、テーブルコンポーネントでリクエスト一覧を表示する。

---

## 品質スコアリングの実装

一番難しかったのが「品質スコア」の定義だ。

LLMのレスポンス品質を自動的に数値化する方法はいくつかあるが、複雑なモデルを使わずに実用的なスコアを出したかった。

最終的に採用したアプローチは**ルールベース + メタ評価LLM**の組み合わせだ。

### ルールベーススコア（0-50点）

```python
def calculate_rule_score(response_text: str, request_context: dict) -> float:
    score = 50.0

    # 長さペナルティ（無駄に長い回答はマイナス）
    if len(response_text) > request_context.get("expected_length", 1000) * 2:
        score -= 10

    # コードブロックの一貫性（要求があるのに含まれていない場合）
    if request_context.get("expects_code") and "```" not in response_text:
        score -= 15

    # エラーパターンの検出
    error_patterns = ["I cannot", "I'm unable to", "I don't have access"]
    for pattern in error_patterns:
        if pattern in response_text:
            score -= 5

    return max(0, score)
```

### メタ評価LLM（0-50点）

高コストなリクエスト（$0.05以上）については、別のLLM（Claude Haiku）を使って品質を自動評価する。

```python
async def llm_quality_eval(request: str, response: str) -> float:
    eval_prompt = f"""
以下のAIレスポンスの品質を0-50点で評価してください。
評価基準：正確性、完全性、簡潔さ、実用性

リクエスト: {request[:500]}
レスポンス: {response[:1000]}

評価点数のみ返してください（数字のみ）:
"""
    result = await anthropic_client.messages.create(
        model="claude-3-haiku-20240307",  # 評価用は安いモデルを使う
        messages=[{"role": "user", "content": eval_prompt}],
        max_tokens=10
    )

    try:
        return float(result.content[0].text.strip())
    except ValueError:
        return 25.0  # パース失敗時はデフォルト
```

---

## 実装して見えてきたこと

3週間ほど自分のClaude Code使用データを収集してみて、気づいたことがある。

### 発見1：コストの偏りが予想以上に大きい

全APIコールの**上位5%が全体コストの40%を占めていた**。

その大半は「大きなコードファイルをまるごとコンテキストに入れたリクエスト」だった。コンテキストの持ち方を変えることで、コストを30%削減できた。

### 発見2：夜間のバッチ処理が想定外に高い

自動化したバッチ処理（テスト実行時のLLM評価）が深夜に動いており、1日あたり$2ほど消費していた。モデルをHaikuに変えることで$0.3まで下げられた。

### 発見3：品質スコアが低い時間帯がある

深夜0時〜2時のリクエストで品質スコアが平均より15%低かった。同じプロンプトなのになぜ…と調べると、コンテキストが積み上がって長くなりすぎていることが原因だった。コンテキストリセットのタイミングを変えることで改善した。

---

## 現在の状態と今後

現在はベータ版として自分の使用データで動かしている。4/25からパブリックβとして公開予定だ。

**MAGI for Devs** → https://magi-console.vercel.app

今後実装したい機能：

- Codex CLIへの対応（現在はAnthropic APIのみ）
- OpenAI API対応
- チーム機能（複数人での使用量管理）
- Slack/Discord通知連携

---

## まとめ

「AIガバナンス」は難しい話ではない。

**記録して、見えるようにして、改善する。**

そのサイクルを回せる仕組みを作ること。それが個人開発者にとってのAIガバナンスだ。

自分で作ったことで、APIコストが30%減り、品質の問題を早期に検出できるようになった。何より「ブラックボックス感」がなくなったことが一番大きかった。

同じ課題を感じている方がいれば、ぜひ試してみてほしい。フィードバックも歓迎します。

---

*この記事の続き（具体的なセットアップ手順）はMAGI for Devsのドキュメントに書く予定です → https://magi-console.vercel.app*
