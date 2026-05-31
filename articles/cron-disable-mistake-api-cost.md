---
emoji: 🔧
published: true
title: AIエージェントを disable するときの落とし穴：cron エントリ残存でAPI無駄消費した話
topics:
- AI
- Claude
- Cron
- インフラ
- マルチエージェント
type: tech
---

# AIエージェントを disable するときの落とし穴：cron エントリ残存でAPI無駄消費した話

## はじめに

マルチエージェント組織を運用していると、時々メンバーを削除・無効化する場面が出てくる。

「このエージェント、もう必要ないから disable しておこう」

その判断は正しい。でも、ちょっと待ってほしい——あなたは **cron エントリの削除も忘れずに済ませた** だろうか？

実際に起きた話だ。複数のエージェントを disable したのに、彼らの定期実行タスク（cron）は生き続けていた。結果、毎日毎時間、存在しないエージェントが API を叩き続け、コストが無駄に消費され続けた。

この記事は、その教訓とチェック方法をまとめたものだ。

---

## 問題：disable 時に cron.md が放置される

### 実例：複数エージェント の無効化漏れ

マルチエージェント組織では、定期実行タスクを `cron.md` ファイルで一元管理することが一般的だ。

```yaml
# cron.md の例
schedule: 0 9 * * *
type: llm
name: agent_daily_planning
prompt: |
  今日の業務計画を立ててください
---

schedule: 0 10 * * *
type: command
command: python3 /scripts/analytics.py --date today
```

ここで問題が発生する。**エージェントが disable された場合、cron.md のエントリは自動削除されない** のだ。

### API コスト無駄消費の仕組み

disable されたエージェントの cron タスクが実行されると：

1. **LLM型タスク**なら、Claude API（またはその他のLLM API）が毎回呼び出される
2. **Command型タスク**なら、スクリプトが毎回実行される

複数のエージェント × 平均3個のcronエントリ = 複数のタスク。

もし各タスクが 1 日に複数回実行されれば（例：朝・昼・夜）、1日の無駄なAPI呼び出し数は急速に増加する。

**月間では、無駄消費の総量は数百〜数千回に達することもある。**

単価が $0.01/実行なら、月額 $16.20 の無駄。大したことないように見えるかもしれないが、同時に 10名のエージェントを disable すれば、月額 $162 の無駄になる。

---

## 根本原因：cron 生存期間管理の曖昧性

cron エントリは **エージェントのライフサイクルと独立している**。

| フェーズ | エージェント状態 | cron 状態 |
|---------|------------|---------|
| **稼働中** | 🟢 enabled | 🟢 実行中 |
| **一時停止** | 🟡 disabled | 🔴 **放置（削除されない）** |
| **完全削除** | ❌ removed | 🔴 **放置（削除されない）** |

### なぜこんなことが起こるのか

**理由1**：エージェント管理とcron管理が異なるシステム・チームで行われている

- エージェント管理：AI組織管理層（`anima disable` コマンド等）
- cron管理：運用チーム（`cron.md` の手動編集）

意思疎通が取れないと、双方が「相手が削除するだろう」と思い込む。

**理由2**：disable 時に自動削除ロジックがない

多くのマルチエージェント基盤では、disable イベントに対して「対応する cron を自動削除する」トリガーが実装されていない。

**理由3**：事後的な確認メカニズムがない

disable 後、「本当に cron が全て削除されたか」を検査するスクリプト・プロセスがない。

---

## 解決策：disable 時の 3 ステップチェック

実務で確立した対処パターンは以下の通り。

### Step 1：disable 前に cron エントリをリスト化

```bash
# disable する前に、該当エージェントのcron一覧を取得
cat cron.md | grep -E "name:|schedule:" | grep -B1 "agent-name"

# 出力例：
# schedule: 0 9 * * *
# name: agent_daily_planning
```

### Step 2：エージェントを disable

```bash
# 例：agent-1 を disable
anima disable agent-1

# または、複数一括：
anima disable agent-1 agent-2 agent-3
```

### Step 3：cron.md から該当エントリを削除

```bash
# cron.md から agent-1 関連のエントリを削除
# （手動編集推奨 — 誤削除を防ぐため必ずレビュー）

git diff cron.md

# 変更を確認して commit
git add cron.md
git commit -m "Remove cron entries for disabled agent: agent-1"
```

**[IMPORTANT]** disable 直後に cron 削除を完了すること。1時間の遅延でも数十回の無駄なAPI呼び出しが発生する。

---

## 検査スクリプト：disable漏れを自動検出

disable 後に「本当にcronが削除されたか」を確認するスクリプト。

```python
#!/usr/bin/env python3
"""
Cron 無効化エージェント検査スクリプト
disable されたエージェントのcronエントリが残っていないか確認
"""

import subprocess
import re
from pathlib import Path

def get_disabled_agents():
    """disable されたエージェント一覧を取得"""
    result = subprocess.run(['anima', 'list', '--disabled'], capture_output=True, text=True)
    disabled = []
    for line in result.stdout.split('\n'):
        if line.strip() and not line.startswith('Name'):
            parts = line.split()
            if parts:
                disabled.append(parts[0])
    return disabled

def check_cron_entries(cron_file='cron.md'):
    """cron.md内の全エントリをパース"""
    if not Path(cron_file).exists():
        print(f"❌ {cron_file} が見つかりません")
        return {}

    with open(cron_file, 'r') as f:
        content = f.read()

    # エントリをセクション分割（`---` で区切られていると仮定）
    entries = content.split('---')
    cron_agents = {}

    for entry in entries:
        # エージェント名を抽出（`name:` フィールドから）
        name_match = re.search(r'name:\s*(\w+)', entry)
        schedule_match = re.search(r'schedule:\s*(\S+)', entry)

        if name_match:
            agent_name = name_match.group(1)
            schedule = schedule_match.group(1) if schedule_match else '(no schedule)'

            if agent_name not in cron_agents:
                cron_agents[agent_name] = []
            cron_agents[agent_name].append(schedule)

    return cron_agents

def main():
    disabled_agents = get_disabled_agents()
    cron_agents = check_cron_entries()

    print("=" * 60)
    print("Cron Disable 検査結果")
    print("=" * 60)

    print(f"\n✅ Disable 済みエージェント: {len(disabled_agents)} 件")
    for agent in disabled_agents:
        print(f"  - {agent}")

    print(f"\n📋 Cron に登録されているエージェント: {len(cron_agents)} 件")

    # 危険な状態を検出
    dangerous = []
    for agent in disabled_agents:
        if agent in cron_agents:
            dangerous.append(agent)

    if dangerous:
        print(f"\n🔴 **DANGER**: 以下の disable 済みエージェントのcronが残っています：")
        for agent in dangerous:
            print(f"  - {agent}: {cron_agents[agent]}")
        print(f"\n→ cron.md から上記エントリを削除してください")
        return 1
    else:
        print(f"\n✅ 問題なし：disable 済みエージェントのcron削除は完了しています")
        return 0

if __name__ == '__main__':
    exit(main())
```

**実行例**：

```bash
$ python3 check-disabled-cron.py

============================================================
Cron Disable 検査結果
============================================================

✅ Disable 済みエージェント: 3 件
  - old-agent-v1
  - expired-data-collector
  - legacy-analyzer

📋 Cron に登録されているエージェント: 42 件

🔴 **DANGER**: 以下の disable 済みエージェントのcronが残っています：
  - old-agent-v1: ['0 9 * * *', '0 12 * * *']
  - expired-data-collector: ['0 */4 * * *']

→ cron.md から上記エントリを削除してください
```

---

## disable チェックリスト

disable を実施するたびに、以下のチェックリストを確認すること。

```markdown
## エージェント Disable チェックリスト

### 削除対象
- [ ] エージェント名: ___________________
- [ ] 理由: ___________________
- [ ] disable 実行日時: ___________________

### Step 1: 事前確認
- [ ] cron.md内の該当エントリを確認
  `grep "{agent_name}" cron.md`
- [ ] エントリ数をカウント（___件）
- [ ] 関連スクリプト・トリガーを確認

### Step 2: Disable 実行
- [ ] `anima disable {agent_name}` を実行
- [ ] 状態確認：`anima list --disabled | grep {agent_name}`

### Step 3: Cron 削除
- [ ] cron.md から削除対象エントリをマーク
- [ ] 別途ファイルにバックアップ（誤削除防止）
- [ ] `git diff cron.md` で確認
- [ ] commit & push

### Step 4: 検査
- [ ] `python3 check-disabled-cron.py` で削除漏れ検出
- [ ] スクリプト出力が `✅ 問題なし` を返す
- [ ] **検査完了まで cron.md の編集を確定させない**

### 完了
- [ ] 全ステップ完了
- [ ] チェックリスト記録をアーカイブ
```

---

## まとめ：disable は **cron 削除までセット**

### 忘れてはいけないこと

1. **disable ≠ 完全削除**
   - エージェントを disable しても、cron は動き続ける

2. **cron.md は手動管理**
   - disable イベントで自動削除されない
   - 必ず手動で削除すること

3. **事後検査が重要**
   - disable後、削除漏れを必ず検査する
   - スクリプト化して自動化を推奨

4. **金銭的インパクト**
   - 無駄なAPI呼び出しは、月額 $100+ のコストになり得る
   - 複数エージェントの disable 時は特に注意

### disable 手順の公式化

```
1. cron エントリ確認
2. anima disable {agent}
3. cron.md から該当エントリを削除
4. 検査スクリプトで確認
5. git commit & push
6. チェックリスト記録
```

この 6 ステップを **毎回確実に実行** すること。「さっき disable したのに cron が残ってた」という事態を防ぐには、プロセスの自動化と事後検査が不可欠だ。

---

## 関連テーマ

AIエージェント組織の運用では、このような「見えない落とし穴」が他にもある。詳しくは以下を参照：

- **Cron失敗パターンとデバッグ**：マルチエージェント環境での定期実行タスク管理の教訓
- **API コスト管理**：複数のLLM プロバイダー間でのコスト最適化戦略
- **エージェント ライフサイクル管理**：組織スケールでのメンバー管理ベストプラクティス
