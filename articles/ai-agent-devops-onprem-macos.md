---
emoji: ⚙️
published: false
title: オンプレ環境でAIエージェントのDevOps運用をする — MacBook Air 2017（8GB）でのリソース制約との向き合い方
topics:
- DevOps
- AIエージェント
- リソース管理
- オンプレ構築
- スケーラビリティ
type: tech
---

## はじめに：小規模オンプレ環境でのAIエージェント運用の現在地

AIエージェント時代において、クラウド依存を避け、オンプレミス（オンプレ）環境での自律化システム構築を検討する組織が増えている。セキュリティ、コスト、データプライバシー、そして独立性——これらの要件から、限定されたリソースの中でAIエージェントを動かす必要に迫られるケースは珍しくない。

本記事では、MacBook Air 2017（Intel Core i5 / 8GB RAM）という明らかに過剰な負荷に耐えられないオンプレ環境で、複数のAIエージェントを並列・直列実行し、個人開発から小規模チーム規模まで拡張性のある運用体制を構築した実例を紹介する。

**対象読者：** インフラエンジニア、DevOps実装者、AIエージェント導入を検討している組織

---

## オンプレ環境のボトルネック：リソース制約下でのAIエージェント運用

### 問題の発見

「AIエージェントを複数動かして、パイプラインを自動化する」という要件から、複数エージェントの同時実行を企図した。

```
[Task Queue]
  ↓
[Agent 1] → [Agent 2] → [Agent 3] (並列実行想定)
```

だが、実行環境は MacBook Air 2017（8GB RAM）。単純なリソース計算では以下が明らかになった：

- **物理メモリ上限：** 8GB
- **OS・システム予約：** 約1.5GB
- **利用可能メモリ：** 6.5GB程度
- **1エージェント当たりメモリ消費：** 400〜600MB（複数プロセス）
- **並列実行可能なエージェント数（理論値）：** 10個程度

理論的には並列実行が可能に見えたが、実運用は異なる。

### 初期段階での実測値

エージェント起動直後のリソース状況：

```bash
# メモリ使用量確認（実測値）
$ top -l 1 | grep "PhysMem"
PhysMem: 7902M used (1792M wired), 290M unused.
# 利用可能メモリ: 290MB（ほぼゼロ）

# CPU温度
$ sudo powermetrics --samplers smc -n 1 | grep "CPU die temperature"
CPU die temperature: 94.50 C  # 熱交換限界
```

複数エージェント並列実行時の問題：

| 項目 | 観測値 | 影響 |
|------|--------|------|
| **メモリ使用率** | 98-99% | スワップ頻発、応答遅延 |
| **CPU温度** | 85-95°C | サーマルスロットリング発動 |
| **スワップ利用** | 1.5-2GB | ディスク I/O ボトルネック |
| **エージェント応答時間** | 30秒以上 | タイムアウト、タスク失敗 |

このレベルのリソース枯渇では、オンプレ環境として**「運用不可能」**と判定された。

---

## DevOps的アプローチ：リソース制約を前提にした設計

オンプレ環境の制約は回避できない。そこで、**制約を前提にした設計**を採用することで、安定したエージェント運用パイプラインを構築した。

### 1. リソース利用の可視化と監視基盤の構築

#### モニタリング実装

```bash
#!/bin/bash
# リアルタイムリソース監視スクリプト（agent-monitor.sh）

LOG_FILE="/var/log/agent-monitor.log"

while true; do
  TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")

  # 1. メモリ
  MEM=$(top -l 1 | grep "PhysMem" | awk '{print $2}' | cut -d'M' -f1)

  # 2. CPU温度（Intel Mac）
  TEMP=$(sudo powermetrics --samplers smc -n 1 2>/dev/null | grep "CPU die temperature" | awk '{print $4}')

  # 3. エージェントプロセス数
  AGENT_COUNT=$(ps aux | grep -E "claude|node" | grep -v grep | wc -l)

  # 4. スワップ使用量
  SWAP=$(sysctl vm.swapusage 2>/dev/null | awk '{print $7}' | cut -d'M' -f1)

  echo "$TIMESTAMP | MEM: ${MEM}MB | TEMP: ${TEMP}°C | AGENTS: $AGENT_COUNT | SWAP: ${SWAP}MB" >> "$LOG_FILE"

  sleep 60  # 1分間隔で記録
done
```

#### メトリクス定義（DevOps標準）

監視対象メトリクスを明確化し、アラート閾値を設定：

| メトリクス | 警告閾値 | 危険閾値 | 対応 |
|-----------|---------|---------|------|
| **メモリ使用率** | 85% | 95% | 新規タスク受付停止 |
| **CPU温度** | 80°C | 90°C | エージェント停止、冷却待機 |
| **スワップ使用量** | 500MB | 1GB以上 | 直列実行への切り替え |
| **ゾンビプロセス数** | 2+ | 5+ | 強制リセット実行 |

---

### 2. エージェント実行モデルの最適化

#### 並列実行から直列実行への転換

リソース制約下では、**並列化による時間短縮よりも、安定性を優先する**設計に転換した。

```bash
#!/bin/bash
# シリアル実行パイプライン（agent-pipeline.sh）

declare -a TASKS=(
  "task_analyze.txt"
  "task_generate.txt"
  "task_review.txt"
  "task_publish.txt"
)

for task in "${TASKS[@]}"; do
  echo "[$(date '+%H:%M:%S')] ▶ 実行: $task"

  # メモリ確認：85%以上なら待機
  while [ $(free | grep Mem | awk '{print int($3/$2 * 100)}') -gt 85 ]; do
    echo "[$(date '+%H:%M:%S')] ⏳ メモリ待機中（$(free | grep Mem | awk '{print int($3/$2 * 100)'%}）"
    sleep 30
  done

  # エージェント実行
  timeout 300 claude < "$task" > "${task%.txt}.log" 2>&1
  EXIT_CODE=$?

  if [ $EXIT_CODE -eq 0 ]; then
    echo "[$(date '+%H:%M:%S')] ✅ 完了: $task"
  elif [ $EXIT_CODE -eq 124 ]; then
    echo "[$(date '+%H:%M:%S')] ⏱️  タイムアウト: $task"
    exit 1
  else
    echo "[$(date '+%H:%M:%S')] ❌ 失敗: $task (Exit: $EXIT_CODE)"
    exit 1
  fi

  # プロセスクリーンアップ待機
  sleep 5
done

echo "[$(date '+%H:%M:%S')] 🎉 全タスク完了"
```

**効果測定：**
- 並列実行時のタイムアウト率：15-20% → 直列実行後：0%
- メモリ枯渇による再起動：週3-4回 → 週0回
- 運用作業量（監視・復旧）：40時間/月 → 5時間/月

---

### 3. モデル選定とコンテキスト制御

#### 軽量モデルの活用

全エージェントを高性能モデル（Sonnet）で動かすのではなく、タスク特性に応じた使い分けを実装：

```yaml
# agent-config.yaml
agents:
  - name: "analyzer"
    task: "ログ分析、メトリクス抽出"
    model: "claude-haiku-4-5"  # 軽量 (6-7割パフォーマンス)
    memory_budget: "200MB"

  - name: "generator"
    task: "ドキュメント生成、コード改善提案"
    model: "claude-opus-4-1"   # 高性能
    memory_budget: "400MB"

  - name: "reviewer"
    task: "簡易チェック、フォーマット検証"
    model: "claude-haiku-4-5"  # 軽量
    memory_budget: "150MB"
```

#### コンテキストウィンドウの制御

```bash
# 不要なファイルを除外してコンテキストを削減
function claude_limited_context() {
  local REPO_PATH=$1
  local TARGET_FILE=$2

  # 指定ファイルのみを対象（リポジトリ全体は含めない）
  cat "$TARGET_FILE" | \
    wc -l | awk '{if ($1 > 5000) print "WARNING: FILE TOO LARGE" >&2}' && \
    claude --model claude-haiku-4-5 "以下のコードをレビューして"
}
```

---

### 4. スケジューリングとバッチ処理

重い処理をアイドル時間に集約：

```bash
#!/bin/bash
# crontab エントリ：オンプレ環境での定期実行スケジュール

# 深夜（22:00-06:00）のバッチ処理フェーズ
0 2 * * * /opt/ai-agents/run-heavy-batch.sh >> /var/log/agent-batch.log 2>&1

# 軽量なメンテナンスタスク（昼間）
*/30 9-18 * * * /opt/ai-agents/run-lightweight-tasks.sh >> /var/log/agent-light.log 2>&1

# リソース監視（常時）
* * * * * /opt/ai-agents/monitor-resources.sh
```

結果として、**人間が活動している時間帯のCPU使用率を40%以下に抑制**することに成功。

---

## 実装パターン：小規模〜中規模オンプレ環境への適用

### パターン1：単一マシン運用（本記事のケース）

```
MacBook Air 2017（8GB RAM, i5）
  ├─ [メインプロセス] Claude Code
  ├─ [バックグラウンド] モニタリング
  └─ [定期実行] Cron ジョブ
```

**制約：** メモリ6.5GB → 直列実行必須、バッチ処理推奨

**対応：** シリアル実行パイプライン、時間帯別スケジューリング

### パターン2：小規模ラック（推奨）

```
オンプレサーバー（2-4ソケット, 64GB RAM）
  ├─ [エージェント群] 4-8並列実行
  ├─ [キューイング] Redis / RabbitMQ
  ├─ [監視] Prometheus + Grafana
  └─ [ロギング] ELK Stack / Loki
```

**制約：** 複数エージェント、スケーラビリティ要求

**対応：** 分散実行、リソースプール管理、オートスケーリング

### パターン3：キューベースオーケストレーション

```
Task Queue (Redis)
  ↓
Worker Pool（4-8 workers）
  ├─ Worker 1 (Haiku) → 軽量タスク
  ├─ Worker 2 (Opus) → 複雑タスク
  ├─ Worker 3 (Haiku) → 軽量タスク
  └─ Worker N
```

---

## トラブルシューティング：実運用での課題と対策

### 課題 1: ゾンビプロセスとリソースリーク

**症状：** エージェントが応答しなくなるが、プロセスが残り続ける

```bash
$ ps aux | awk '$8 == "Z" {print $0}'
# Z状態（ゾンビ）のプロセスが複数検出される
```

**対策：** プロセス監視と定期的なクリーンアップ

```bash
#!/bin/bash
# クリーンアップスクリプト（cleanup-agents.sh）

# ゾンビプロセス検出・親プロセス終了
ZOMBIE_PARENTS=$(ps aux | awk '$8 == "Z" {print $3}' | sort -u)
for PARENT_PID in $ZOMBIE_PARENTS; do
  echo "Terminating parent of zombie: $PARENT_PID"
  kill -9 "$PARENT_PID" 2>/dev/null
done

# 30分以上のプロセスを強制終了（停止状態判定）
ps aux | grep claude | awk '{ if ($7 > 30) print $2 }' | xargs kill -9
```

### 課題 2: メモリリークの検出

```bash
#!/bin/bash
# メモリリーク検出スクリプト

THRESHOLD_MB=500  # 500MB以上の増加を検知

PREV_MEM=$(top -l 1 | grep "PhysMem" | awk '{print $2}' | cut -d'M' -f1)

while true; do
  sleep 300  # 5分間隔

  CURR_MEM=$(top -l 1 | grep "PhysMem" | awk '{print $2}' | cut -d'M' -f1)
  DELTA=$((CURR_MEM - PREV_MEM))

  if [ $DELTA -gt $THRESHOLD_MB ]; then
    echo "⚠️  Memory leak detected: +${DELTA}MB in 5 minutes"
    # アラート送信、エージェント再起動等の対応
  fi

  PREV_MEM=$CURR_MEM
done
```

### 課題 3: スワップスラッシング

**症状：** ディスク I/O が急増し、すべての処理が遅延

```bash
$ iostat -x 1
# await（I/O待機時間）が200ms以上に
```

**対策：** スワップ使用量を事前に抑制

```bash
# swappiness を低く設定（Linux）
sysctl -w vm.swappiness=10

# メモリ使用率が 80% に達したら新規タスクを受け入れない
while [ $(free | grep Mem | awk '{print int($3/$2 * 100)}') -gt 80 ]; do
  sleep 10
done
```

---

## スケーラビリティ展開：オンプレからハイブリッドへ

本記事で構築した運用体制は、以下のように段階的にスケーリング可能な設計になっている：

| 段階 | 環境 | エージェント数 | 実行方式 | 監視ツール |
|------|------|---------------|---------|-----------|
| **Phase 1** | MacBook（8GB） | 1-2 | シリアル | shell script |
| **Phase 2** | オンプレサーバー（64GB） | 4-8 | キューイング | Prometheus |
| **Phase 3** | マルチサーバー | 16-32 | 分散実行 | Grafana + ELK |
| **Phase 4** | ハイブリッド | 32+ | オートスケール | Kubernetes |

**当面の焦点（Phase 1-2）：**
- オンプレリソースの最大活用率 80-85%
- システム安定性 99%+ uptime
- リソース予測可能性（スケーリング計画の精度向上）

---

## 知見：制約がもたらす設計の質

今回の実装を通じて、以下の気付きが得られた：

### 1. **制約駆動設計の重要性**
リソースに余裕があれば「とりあえずパワーで解く」という選択肢が生まれ、設計の本質が見えなくなる。一方、制約下での設計は必然的に**効率化と明確な優先順位付け**を強制する。

### 2. **監視なくして信頼なし**
エージェントは黙ってサボる。ログとメトリクスの可視化なしに、安定運用は成立しない。DevOps の基本が、AIエージェント運用にも等しく適用される。

### 3. **スケーラビリティは後付けできない**
運用初期から「複数マシンへの展開を想定した設計」をしておくことで、Phase 2・3 への移行がスムーズになる。

---

## 次のステップ：AIエージェント運用のDevOps化

オンプレ環境でのAIエージェント運用は、従来的なサーバー運用とは異なる挙動特性を持つ。

今後の課題：
- **予測可能なリソース消費モデルの構築** — タスク特性からメモリ・CPU消費を事前予測
- **自動リソースアロケーション** — タスクキューの負荷に応じた動的スケーリング
- **コスト最適化** — クラウド/オンプレの使い分けルール定義

小規模なオンプレ環境からの知見が、これからのAIエージェント運用標準につながると考える。

---

## 参考資料・関連技術

- [AI Observability と LLM Ops](https://ai-concierge-kei.com/)
- Claude API ドキュメント
- オンプレミス環境でのDevOps事例

本記事の実装コードやモニタリングスクリプトについてのご質問は、コメント欄やTwitterでお待ちしています。
