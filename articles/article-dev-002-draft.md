---
emoji: ⚡
published: true
title: Claude Code のトークン消費を半分にした5つのテクニック
topics:
- claude-code
- apiコスト
- トークン最適化
- llm
- 開発効率化
type: tech
---

# Claude Code のトークン消費を半分にした5つのテクニック

月末にAnthropicの請求書を開いて「え、また？」となった経験はないだろうか。

Claude Codeは強力だ。コードを書いてくれる、レビューしてくれる、バグを直してくれる。しかしその代償として、気づかないうちにトークンが積み上がっていく。特に長いシステムプロンプトや、ファイル全体を毎回渡すような使い方をしていると、トークン消費は雪だるま式に増える。

この記事では、実際に月のトークン消費を約50%削減できた5つのテクニックを紹介する。コード例込みで説明するので、今日から即適用できる。

---

## なぜClaudeのトークンは爆発するのか

Claude APIのコストはシンプルだ。**送ったトークン数 × 単価 + 受け取ったトークン数 × 単価**。問題は「送ったトークン数」が想像以上に膨らみやすいことにある。

よくある消費パターンを整理すると：

- **ファイル全体を毎回渡している**: 1000行のファイルを10回渡せば1万行分のトークン
- **システムプロンプトが肥大化している**: 500トークンのプロンプトを100リクエスト送ると5万トークン
- **何でもOpusやSonnetに頼っている**: 軽いタスクに高価なモデルを使うのは損
- **同じコンテキストを繰り返し送信している**: キャッシュを使えば最大90%削減できる

これらを組み合わせると、月$50-100の請求が半分以下に収まるケースは珍しくない。

詳しいコストの把握方法については、[Claude Code CLIのAPIコスト、把握できていますか？](https://qiita.com/kei-concierge/items/e7ea7019e87f04178aa0) も参考にしてほしい。

---

## 5つのテクニック

### テクニック1: ファイル全体でなく差分だけ渡す

最も即効性が高いのがこれだ。「このファイルのバグを直して」と言いながら1000行のファイルを渡すのは、医者に「体の具合が悪い」と全裸で診察室に入るようなものだ。必要な部分だけ渡せばいい。

**Beforeの例（NG）:**

```
以下のファイルを修正してください:
[1000行のコード全体を貼り付け]
```

**Afterの例（OK）:**

```bash
# 変更箇所のみ抽出してClaudeに渡す
git diff HEAD~1 HEAD -- "*.py" | head -200

# 特定ファイルの変更だけ見たい場合
git diff HEAD~1 HEAD -- src/api/handler.py
```

```bash
# より実用的な使い方: 変更ファイルを特定してから内容を渡す
CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)
echo "変更されたファイル:"
echo "$CHANGED_FILES"

# 差分のみ抽出（コンテキスト行数を3行に絞る）
git diff -U3 HEAD~1 HEAD
```

**削減効果**: ファイル全体を渡す場合と比べて、差分が10-20行程度なら **60〜70%のトークン削減** が見込める。1000行のファイルで20行の変更なら、送信トークンは98%減だ。

ポイントは「変更の意図と変更箇所だけ」をClaudeに渡すこと。ファイル全体が必要なケース（大規模リファクタリング、初回のコードレビューなど）は例外だが、日常的な修正・デバッグ作業では差分で十分なことがほとんどだ。

---

### テクニック2: システムプロンプトを最小化する

「毎回500トークンのシステムプロンプトを送っている」という人は要注意だ。1日100リクエストなら、それだけで5万トークン/日、月150万トークンの浪費になる。

**よくあるNG例:**

```python
# 毎回長いシステムプロンプトを送っている
system_prompt = """
あなたはPythonの専門家です。コードはPEP8に準拠し、型ヒントを使用し、
docstringを書き、エラーハンドリングを適切に行い、テストを書き、
セキュリティに配慮し、パフォーマンスを最適化し、可読性を高め...
[500トークン以上続く]
"""
```

**改善策1: `.claude/CLAUDE.md` を活用する**

Claude Codeはプロジェクトルートの `.claude/CLAUDE.md` を自動的に読み込む。ここに開発規約やコンテキストを書いておけば、毎回プロンプトに含める必要がなくなる。

```bash
# プロジェクトのルートに作成
mkdir -p .claude
cat > .claude/CLAUDE.md << 'EOF'
# プロジェクト概要
FastAPI + PostgreSQL構成のWebAPI。Python 3.12使用。

# コーディング規約
- PEP8準拠、型ヒント必須
- 例外はカスタム例外クラスを使用
- テストはpytestで書く

# ディレクトリ構成
src/api/    - APIエンドポイント
src/models/ - データモデル
src/services/ - ビジネスロジック
EOF
```

**改善策2: システムプロンプトを本当に必要な最小限にする**

```python
# 最小化されたシステムプロンプト
system_prompt = "Python専門家。PEP8、型ヒント必須。簡潔に答える。"
# これだけで十分なことが多い
```

**削減効果**: システムプロンプトを500トークン → 50トークンに削減した場合、100リクエスト/日なら月約135万トークン削減。

---

### テクニック3: モデルを使い分ける

全てのタスクにSonnetやOpusを使う必要はない。タスクの複雑さに応じてモデルを使い分けることが、コスト削減の最大の鍵の一つだ。

**料金比較（2024年現在）:**

| モデル | Input (1Mトークン) | Output (1Mトークン) | 主な用途 |
|--------|-------------------|---------------------|----------|
| claude-haiku-4-5 | $0.80 | $4.00 | 簡単なタスク・大量処理 |
| claude-sonnet-4-5 | $3.00 | $15.00 | 標準的な開発タスク |
| claude-opus-4-5 | $15.00 | $75.00 | 複雑な推論・設計 |

Haikuはコスト面でSonnetの約1/4、Opusの約1/20だ。「変数名を直して」「コメントを英語に翻訳して」「このエラーメッセージを改善して」といったタスクにOpusを使うのは明らかに過剰だ。

**実装例: タスク複雑度に応じたモデルルーティング**

```python
import anthropic

client = anthropic.Anthropic()

# 簡単なタスク → haiku（低コスト）
def simple_task(prompt: str) -> str:
    """変数名変更、コメント追加、軽微なフォーマット修正など"""
    response = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=256,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text

# 標準的な開発タスク → sonnet
def standard_task(prompt: str) -> str:
    """機能実装、バグ修正、コードレビューなど"""
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text

# 複雑な推論が必要なタスク → opus
def complex_task(prompt: str) -> str:
    """アーキテクチャ設計、複雑なアルゴリズム、深い技術的分析など"""
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text

# 使い分けの目安
TASK_COMPLEXITY = {
    "rename_variable": simple_task,
    "fix_typo": simple_task,
    "implement_feature": standard_task,
    "code_review": standard_task,
    "system_design": complex_task,
    "algorithm_optimization": complex_task,
}
```

**削減効果**: 従来すべてSonnetを使っていた場合、タスクの50%をHaikuに移行するだけでコストを約35%削減できる計算になる。

---

### テクニック4: Prompt Cachingを活用する

Claude APIには **Prompt Caching** という機能がある。同じコンテキスト（システムプロンプト、長いドキュメントなど）を繰り返し送信する場合、最初の送信後はキャッシュから読み込まれ、**キャッシュヒット時はInput料金が約90%オフ**になる。

コードレビューで同じシステムプロンプト（コーディング規約など）を毎回送っているなら、これは使わない手はない。

```python
import anthropic

client = anthropic.Anthropic()

# コードレビュー用の長いシステムプロンプトをキャッシュする
def review_code(code: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        system=[
            {
                "type": "text",
                "text": """あなたはシニアエンジニアによるコードレビュアーです。
以下の観点でレビューを行ってください：

1. セキュリティ: SQLインジェクション、XSS、認証・認可の問題
2. パフォーマンス: N+1クエリ、不要なループ、メモリリーク
3. 可読性: 変数名、関数名、コメントの適切さ
4. テスタビリティ: 依存性注入、副作用の分離
5. エラーハンドリング: 例外処理の網羅性、ユーザーへのフィードバック

問題を発見した場合は重要度（CRITICAL/HIGH/MEDIUM/LOW）と改善案を示してください。
...[長いプロンプトの場合でも、このブロックはキャッシュされる]""",
                "cache_control": {"type": "ephemeral"}  # キャッシュ有効化
            }
        ],
        messages=[
            {
                "role": "user",
                "content": f"以下のコードをレビューしてください:\n\n```python\n{code}\n```"
            }
        ]
    )
    return response.content[0].text

# 1回目: キャッシュミス（通常料金）
result1 = review_code("def login(user, pwd): ...")

# 2回目以降: キャッシュヒット（Input料金が約90%オフ）
result2 = review_code("def register(email, pwd): ...")
result3 = review_code("def logout(token): ...")
```

**キャッシュが有効になる条件:**
- システムプロンプトが1,024トークン以上（Sonnet/Opusの場合）
- 同じキャッシュブロックを5分以内に再利用している
- `cache_control: {"type": "ephemeral"}` を明示的に指定している

**削減効果**: 100トークンのシステムプロンプトを100回送る場合（通常10,000トークン分の課金）→ キャッシュ有効時は1,000 + 9,000×0.1 = 1,900トークン分の課金。**約81%削減**。

---

### テクニック5: バッチリクエストでAPIコールを減らす

「ファイルA、B、Cをそれぞれレビューして」と3回リクエストを送るより、1回で3ファイルをレビューさせた方が効率的なケースがある。また、Anthropic APIにはMessage Batches APIという非同期バッチ処理機能があり、**通常より50%安い料金**で処理できる。

**同一リクエスト内で複数タスクをまとめる:**

```python
import anthropic

client = anthropic.Anthropic()

# NG: 3回の個別リクエスト
def review_files_bad(files: list[str]) -> list[str]:
    results = []
    for f in files:
        response = client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=512,
            messages=[{"role": "user", "content": f"レビュー:\n{f}"}]
        )
        results.append(response.content[0].text)
    return results

# OK: 1リクエストに複数ファイルをまとめる
def review_files_good(files: dict[str, str]) -> str:
    file_contents = "\n\n".join(
        f"### {name}\n```\n{content}\n```"
        for name, content in files.items()
    )
    response = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"以下の{len(files)}ファイルをまとめてレビューしてください:\n\n{file_contents}"
        }]
    )
    return response.content[0].text
```

**Message Batches APIを使った非同期バッチ処理（50%オフ）:**

```python
import anthropic
import time

client = anthropic.Anthropic()

def batch_process(tasks: list[dict]) -> list[str]:
    """複数タスクをバッチで処理（料金50%オフ）"""

    # バッチリクエスト作成
    batch = client.messages.batches.create(
        requests=[
            {
                "custom_id": f"task-{i}",
                "params": {
                    "model": "claude-haiku-4-5",
                    "max_tokens": 512,
                    "messages": [{"role": "user", "content": task["prompt"]}]
                }
            }
            for i, task in enumerate(tasks)
        ]
    )

    # バッチ完了まで待機（非同期処理）
    while True:
        batch = client.messages.batches.retrieve(batch.id)
        if batch.processing_status == "ended":
            break
        time.sleep(10)

    # 結果取得
    results = []
    for result in client.messages.batches.results(batch.id):
        if result.result.type == "succeeded":
            results.append(result.result.message.content[0].text)

    return results

# 使用例: 20ファイルのコメント自動生成を一括処理
files_to_comment = [
    {"prompt": f"このコードにdocstringを追加: def func_{i}(): pass"}
    for i in range(20)
]
results = batch_process(files_to_comment)
```

**削減効果**: バッチAPIの50%割引と、リクエストのオーバーヘッド削減を合わせると、大量処理タスクでは**40〜60%のコスト削減**が可能。ただしバッチ処理はリアルタイム性が不要な場合に限られる。

---

## 実際の削減効果（Before/After）

各テクニックの削減効果をまとめると：

| テクニック | 削減効果 | 適用難易度 |
|------------|----------|-----------|
| 差分だけ渡す | 60〜70% | ★☆☆（簡単） |
| システムプロンプト最小化 | 30〜50% | ★☆☆（簡単） |
| モデルルーティング | 30〜40% | ★★☆（中程度） |
| Prompt Caching | 50〜90% | ★★☆（中程度） |
| バッチリクエスト | 40〜60% | ★★★（要設計） |

全てを同時に適用する必要はない。まずは「差分だけ渡す」「モデルを使い分ける」の2つから始めるだけでも、体感できるレベルでコストが変わる。

**実際の変化イメージ（月$80使っているケース）:**

```
Before: $80/月
  - システムプロンプト肥大化: $20
  - ファイル全体の送信: $30
  - 全タスクにSonnet使用: $30

After: $38/月（約52%削減）
  - システムプロンプト最小化: $10（-$10）
  - 差分のみ送信: $9（-$21）
  - Haiku/Sonnet使い分け: $19（-$11）
```

---

## まとめ

Claude Codeは強力なツールだが、使い方を工夫しないとコストが際限なく膨らむ。今回紹介した5つのテクニックを改めて整理する：

1. **差分だけ渡す** — `git diff` を活用して不要なコンテキストをカット
2. **システムプロンプトを最小化** — `.claude/CLAUDE.md` を使って共通情報を分離
3. **モデルを使い分ける** — タスクの複雑さに応じてHaiku/Sonnet/Opusを選択
4. **Prompt Cachingを活用する** — 繰り返し送信するコンテキストをキャッシュ
5. **バッチリクエストを使う** — 大量タスクはバッチAPIで50%オフ

どれも今日から適用できる実践的なテクニックだ。「どのくらいトークンを使っているか分からない」という方は、まず現状把握から始めてほしい。コストの見える化ができれば、どのテクニックが最も効果的か自ずと見えてくる。

Claude Codeをより賢く、よりコスト効率良く使う方法については、[ai-concierge-kei.com](https://ai-concierge-kei.com/) でも継続的に発信していく。ぜひブックマークしておいてほしい。

---

*最終更新: 2026年4月*
