---
emoji: "💸"
title: "LLM APIコスト削減の落とし穴——開発現場で繰り返される7つのアンチパターンと対処法"
topics: ["LLM", "Python", "API", "コスト最適化", "AI"]
type: "tech"
published: true
---

# LLM APIコスト削減の落とし穴——開発現場で繰り返される7つのアンチパターンと対処法

> **このシリーズについて**
> - パート1：Batch API・キャッシュ・モデル選定で月次コストを半減する実装ガイド
> - **パート2（本記事）**：開発現場で繰り返される7つのアンチパターン
> - パート3：企業規模別の実装ケーススタディ

---

LLM APIを導入した直後は「コントロールできている」という感覚がある。

ところが数か月後、請求書を開くと想定の2〜3倍の金額が並んでいる。
そのとき原因を追いかけると、決まって同じパターンが出てくる。

この記事では、実際の開発現場で何度も観測してきた**7つのアンチパターン**を整理する。
それぞれ「なぜ起きるか」「コストへの影響」「すぐ直せる実装」をセットで示す。

---

## アンチパターン1：全リクエストを最高品質モデルで処理する

### なぜ起きるか

「品質が大事だから」という正論がすべてのリクエストに適用されてしまう。
分類・要約・フォーマット変換など、実際には軽量モデルで十分なタスクまで、Claude OpusやGPT-4に投げ続ける。

### コストへの影響

軽量モデル（Gemini Flash）と比較すると、**単価差は最大200倍**に達することがある。
月1,000リクエストの分類タスクをGemini Flashで処理すれば$38。
同じタスクをClaude Opusで処理すると$112,500。

### 対処法

タスクの複雑度に応じてモデルを振り分けるルーターを実装する。

```python
def select_model(task_type: str, complexity: int) -> str:
    """
    task_type: "classify" | "summarize" | "generate" | "analyze"
    complexity: 1-10（1=単純、10=高度な推論が必要）
    """
    if task_type in ("classify", "format") or complexity <= 3:
        return "gemini-1.5-flash-latest"   # 最安・軽量
    elif complexity <= 6:
        return "claude-3-5-sonnet-20241022"  # バランス
    else:
        return "claude-3-opus-20240229"      # 高精度が必要な場合のみ

# 使用例
model = select_model("classify", complexity=2)
# → gemini-1.5-flash-latest（コスト95%削減）
```

---

## アンチパターン2：`max_tokens` を設定しない

### なぜ起きるか

「長い回答が必要かもしれない」という懸念から、トークン上限を設定しないままにする。
結果として、モデルが不要な説明・補足・例示を延々と生成し続ける。

### コストへの影響

出力トークンは入力の**4〜5倍のコスト**がかかるプロバイダーが多い（Claude Sonnetの場合、入力$3/1Mに対し出力$15/1M）。
不要な出力が1,000トークン増えるだけで、月1,000リクエストなら$15の追加コストになる。

### 対処法

プロンプトで出力形式を明示し、`max_tokens`を設定する。

```python
from anthropic import Anthropic

client = Anthropic()

def classify_text(text: str) -> str:
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=50,  # 分類タスクなら50で十分
        messages=[{
            "role": "user",
            "content": f"""以下のテキストを分類してください。
結果のみをJSON形式で返してください。説明は不要です。

テキスト: {text}

出力形式:
{{"category": "positive|negative|neutral", "confidence": 0.0-1.0}}"""
        }]
    )
    return response.content[0].text

# max_tokens=50 vs デフォルト（4096）で約98%のトークン削減
```

---

## アンチパターン3：同一の長いシステムプロンプトを毎回送信する

### なぜ起きるか

RAGやエージェント実装でよくある。
大量のコンテキスト（ドキュメント・ルール定義・ペルソナ設定）を毎リクエスト送り続けている。

### コストへの影響

50,000トークンのシステムプロンプトを1,000回送信すれば、入力トークンだけで5,000万トークン。
Claude Sonnetなら**$150/月**が入力トークンだけで消える。

### 対処法

Claude APIのPrompt Cachingを有効にする。

```python
from anthropic import Anthropic

client = Anthropic()

SYSTEM_CONTEXT = """
[50,000トークン相当の会社規定・製品仕様書・RAGドキュメント]
"""

def ask_with_cache(question: str) -> str:
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=500,
        system=[
            {
                "type": "text",
                "text": SYSTEM_CONTEXT,
                "cache_control": {"type": "ephemeral"}  # キャッシュ有効化
            }
        ],
        messages=[{
            "role": "user",
            "content": question
        }]
    )
    return response.content[0].text

# 初回: 50,000トークン分のコスト
# 2回目以降: キャッシュ命中でコスト約90%削減
# 1,000リクエストで$150 → 約$20（約87%削減）
```

---

## アンチパターン4：エラー時に無制限リトライする

### なぜ起きるか

「確実に処理したい」という意図でexponential backoffなしのリトライを実装する。
レート制限エラー（429）が発生したとき、同じリクエストを何度も送り続ける。

### コストへの影響

一部のプロバイダーでは、レート制限に引っかかる手前のリクエストもトークン課金されることがある。
無限リトライが起動すると、**同じ処理に数百倍のコスト**がかかる事例も報告されている。

### 対処法

リトライ回数に上限を設け、backoffを正しく実装する。

```python
import time
import anthropic
from anthropic import RateLimitError, APIError

def call_with_retry(
    client: anthropic.Anthropic,
    messages: list,
    max_retries: int = 3,
    base_delay: float = 1.0
) -> str:
    for attempt in range(max_retries):
        try:
            response = client.messages.create(
                model="claude-3-5-sonnet-20241022",
                max_tokens=500,
                messages=messages
            )
            return response.content[0].text

        except RateLimitError:
            if attempt == max_retries - 1:
                raise
            wait = base_delay * (2 ** attempt)  # exponential backoff
            print(f"Rate limited. Waiting {wait}s...")
            time.sleep(wait)

        except APIError as e:
            # 4xx系エラーはリトライしない（コスト無駄）
            if 400 <= e.status_code < 500:
                raise
            if attempt == max_retries - 1:
                raise
            time.sleep(base_delay)

    return ""
```

---

## アンチパターン5：開発環境と本番環境で同じモデルを使う

### なぜ起きるか

「本番と同じ挙動を確認したい」という理由で、開発・テスト時もClaude OpusやGPT-4を使う。

### コストへの影響

ローカルでのデバッグ・チューニング・テストで発生するリクエスト数は、本番の数倍になることが多い。
それをすべて最高品質モデルで処理すると、開発フェーズだけで**月額の30〜50%**を消費する。

### 対処法

環境変数でモデルを切り替える。

```python
import os

def get_model() -> str:
    env = os.getenv("APP_ENV", "development")
    models = {
        "development": "gemini-1.5-flash-latest",    # 最安
        "staging":     "claude-3-5-haiku-20241022",  # バランス
        "production":  "claude-3-5-sonnet-20241022", # 本番品質
    }
    return models.get(env, "gemini-1.5-flash-latest")
```

**削減効果**: 開発コストを最大95%削減。本番品質の確認はstaging環境のみに限定。

---

## アンチパターン6：コストをモニタリングしていない

### なぜ起きるか

機能開発に集中するあまり、コスト計測の仕組みを後回しにする。

### コストへの影響

モニタリングなしでは、コスト増加に気づくのが月次請求後になる。
小さなコードの変更が月額を数倍にしても、しばらく気づけない。

### 対処法

最小構成でも即日使えるコスト計測を実装する。

```python
import json
import datetime
from pathlib import Path

class CostTracker:
    def __init__(self, log_path: str = "cost_log.jsonl"):
        self.log_path = Path(log_path)
        self.pricing = {
            "claude-3-5-sonnet-20241022": {"input": 3.0, "output": 15.0},
            "claude-3-opus-20240229":     {"input": 15.0, "output": 75.0},
            "gemini-1.5-flash-latest":    {"input": 0.075, "output": 0.3},
        }

    def log(self, endpoint: str, model: str, input_tokens: int, output_tokens: int):
        price = self.pricing.get(model, {"input": 0, "output": 0})
        cost_usd = (input_tokens * price["input"] + output_tokens * price["output"]) / 1_000_000

        record = {
            "timestamp": datetime.datetime.now().isoformat(),
            "endpoint": endpoint,
            "model": model,
            "cost_usd": round(cost_usd, 6),
        }

        with self.log_path.open("a") as f:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")

tracker = CostTracker()
```

---

## アンチパターン7：バッチ処理できるのにリアルタイムAPIを使い続ける

### なぜ起きるか

「非同期処理の実装が面倒」という理由で、夜間バッチや一括処理もリアルタイムAPIで実行し続ける。

### コストへの影響

Anthropic Batch APIはリアルタイムAPIの**50%のコスト**で同一処理が実行できる。
即時性が不要な処理（日次レポート・データ分類・ドキュメント解析など）を毎月倍のコストで処理し続けることになる。

### 対処法

即時性が不要なタスクはBatch APIに移行する。

```python
import anthropic

client = anthropic.Anthropic()

def batch_analyze(texts: list[str]) -> str:
    requests = [
        {
            "custom_id": f"text-{i}",
            "params": {
                "model": "claude-3-5-sonnet-20241022",
                "max_tokens": 200,
                "messages": [{
                    "role": "user",
                    "content": f"要約してください（3行以内）: {text}"
                }]
            }
        }
        for i, text in enumerate(texts)
    ]

    batch = client.batches.create(requests=requests)
    return batch.id

# コスト比較:
# リアルタイムAPI 1,000件: $7.50
# Batch API 1,000件: $3.75（50%削減）
```

---

## まとめ：7つのアンチパターン一覧

| # | アンチパターン | コスト影響 | 対処難度 |
|---|--------------|-----------|--------|
| 1 | 全リクエストを最高品質モデルに送る | 最大200倍 | ★☆☆ |
| 2 | `max_tokens` を設定しない | 最大98%の無駄 | ★☆☆ |
| 3 | 長いシステムプロンプトをキャッシュしない | 最大87%の無駄 | ★★☆ |
| 4 | 無制限リトライ | 数百倍のリスク | ★☆☆ |
| 5 | 開発・テストに本番モデルを使う | 開発コスト30〜50% | ★☆☆ |
| 6 | コストモニタリングをしない | 異常を検知できない | ★★☆ |
| 7 | 夜間バッチにリアルタイムAPIを使う | 常に50%の無駄 | ★★☆ |

アンチパターンを修正するだけで、多くの場合**月額コストの30〜60%削減**が実現できる。
まず手をつけやすいものから着手し、モニタリングを入れて効果を測定しながら進めることをすすめる。

---

*AI API管理ツール「MAGI for Devs」β版公開中：https://magi-platform.com*
