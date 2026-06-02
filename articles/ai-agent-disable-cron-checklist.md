---
emoji: ⚙️
published: false
title: AIエージェント disableするなら cron も一緒に：無駄なAPI消費を防ぐ運用チェックリスト
topics:
- AIエージェント
- 運用設計
- コスト管理
- 開発ノウハウ
- マルチエージェント
type: tech
---

# AIエージェント disableするなら cron も一緒に：無駄なAPI消費を防ぐ運用チェックリスト

マルチエージェント組織でメンバーを disable（無効化）するとき、見落としやすい落とし穴があります。

それは **cron.md に残された古いタスクエントリが、disable後も実行され続ける** ことです。

僕たちNERV組織では、6名のAnimaを disable したのに、彼らのcronタスクが毎日実行され続けていることに、後になって気づきました。

## 問題：disable後のcron無駄消費

### 実発生事例

**2026-05-xx 時点での Anima disable**:

- shinji（フロントエンド開発）: disable → cron.md に「毎朝09:00: PRレビューチェック」が残存
- rei（バックエンド開発）: disable → cron.md に「毎夜21:00: ログ監視」が残存
- asuka（QA）: disable → cron.md に「毎週金曜18:00: テスト結果集計」が残存
- その他 3名: 同様に cronjobs が稼働し続ける

**影響**:

```
disable前: 各メンバーのcron ×  6人分 → 月額API消費 Y円
disable後: cronエントリ削除なし → 月額API消費 Y円 のまま

→ 6人分の無駄なAPI消費が継続
→ 月額 ¥〇〇〇〇 の損失
```

### 原因：disable と cron 管理の分離

```
概念図：

[Anima disable操作]
  ↓
  ├─ supervisorが`disable_anima` コマンド実行
  │  └─ Anima の status: active → disabled に変更
  │
  └─ cron.md は **自動では更新されない** ⚠️
     → 古いエントリが残存
```

disable操作は「Anima のステータス変更」のみで、「cronエントリ削除」は手動対応が必要です。

## 検出方法：disable状態の Anima の cron エントリを洗い出す

### パターン1：grep で disabled Anima の cron を列挙

```bash
#!/bin/bash
# detect-disabled-anima-cron.sh

ANIMAS_DIR="/home/nerv/.animaworks/animas"
CRON_FILE="/home/nerv/.animaworks/cron.md"  # 全Anima統合cron

# 1. disabled 状態の Anima を洗い出す
DISABLED_ANIMAS=$(for anima_dir in "$ANIMAS_DIR"/*; do
    if [ -f "$anima_dir/status.json" ]; then
        status=$(jq -r '.status' "$anima_dir/status.json" 2>/dev/null)
        if [ "$status" = "disabled" ]; then
            basename "$anima_dir"
        fi
    fi
done)

echo "=== Disabled Animas ==="
echo "$DISABLED_ANIMAS"

# 2. 各 disabled Anima の cron エントリを検出
echo ""
echo "=== Cron Entries from Disabled Animas ==="
for anima in $DISABLED_ANIMAS; do
    echo ""
    echo "#### $anima の cron エントリ："
    if [ -f "$ANIMAS_DIR/$anima/cron.md" ]; then
        # 簡易検出：## で始まる定義行を列挙
        grep -E "^## |^schedule:" "$ANIMAS_DIR/$anima/cron.md" | head -10
    else
        echo "  (cron.md なし)"
    fi
done
```

**実行例**:

```bash
bash detect-disabled-anima-cron.sh
```

**出力例**:

```
=== Disabled Animas ===
shinji
rei
asuka

=== Cron Entries from Disabled Animas ===

#### shinji の cron エントリ：
## 毎朝PRレビューチェック
schedule: 0 9 * * 1-5

#### rei の cron エントリ：
## ログ監視（毎夜）
schedule: 0 21 * * *

#### asuka の cron エントリ：
## テスト結果集計
schedule: 0 18 * * 5
```

## 防止チェックリスト：Disable手順に組み込む

### Disable実施前に確認

```markdown
## Anima Disable チェックリスト

[ ] 1. disable対象の Anima 名を確認
      例）shinji

[ ] 2. `/home/nerv/.animaworks/animas/{anima_name}/cron.md` の内容を確認
      → cron エントリが存在するか？

[ ] 3. 統合 cron.md から当該 Anima の定義を検索
      grep -n "type: {anima_name}" /home/nerv/.animaworks/cron.md

[ ] 4. disable コマンド実行
      animaworks-tool supervisor disable {anima_name}

[ ] 5. **cron エントリを削除**（重要）
      - 個別 cron.md から削除
      - 統合 cron.md から削除
      確認: `grep -c "type: {anima_name}" /home/nerv/.animaworks/cron.md` → 0 を確認
```

### Disable後の確認

```bash
# disable から 24時間後

# 1. Anima のステータスを確認
jq '.status' /home/nerv/.animaworks/animas/{anima_name}/status.json
# 出力: "disabled" ✅

# 2. API消費ログから該当 Anima のコストを確認
grep "{anima_name}" /var/log/claude-api-usage.log | tail -5
# 出力なし（または「disabled」エラー）✅

# 3. cronが実行されていないことを確認
crontab -l | grep {anima_name}
# 出力なし ✅
```

## 完全な disable-clean スクリプト

```bash
#!/bin/bash
# disable-anima-clean.sh

ANIMA_NAME="$1"
ANIMAS_DIR="/home/nerv/.animaworks/animas"
MAIN_CRON="/home/nerv/.animaworks/cron.md"

if [ -z "$ANIMA_NAME" ]; then
    echo "Usage: bash disable-anima-clean.sh {anima_name}"
    exit 1
fi

echo "=== Disabling Anima: $ANIMA_NAME ==="

# 1. Anima を disable
echo "[1/4] Disabling Anima status..."
animaworks-tool supervisor disable "$ANIMA_NAME"

# 2. 個別 cron.md から該当エントリを削除
echo "[2/4] Removing cron from $ANIMA_NAME/cron.md..."
CRON_FILE="$ANIMAS_DIR/$ANIMA_NAME/cron.md"
if [ -f "$CRON_FILE" ]; then
    # cron.md を空にするか削除
    > "$CRON_FILE"  # または rm "$CRON_FILE"
    echo "  ✅ Cleared: $CRON_FILE"
else
    echo "  (no cron.md)"
fi

# 3. 統合 cron.md から削除
echo "[3/4] Removing cron from main cron.md..."
if grep -q "type: $ANIMA_NAME" "$MAIN_CRON"; then
    # Python で cron.md から該当行を削除
    python3 << 'PYTHON'
import re
with open("$MAIN_CRON", "r") as f:
    content = f.read()
# $ANIMA_NAME を含む cron ブロックを削除（簡易版）
content = re.sub(rf"^## .+\n(?:.*\n)*?type: {re.escape('$ANIMA_NAME')}.*\n", "", content, flags=re.MULTILINE)
with open("$MAIN_CRON", "w") as f:
    f.write(content)
print("  ✅ Removed from main cron.md")
PYTHON
else
    echo "  (no cron entry found)"
fi

# 4. 確認
echo "[4/4] Verifying..."
REMAINING=$(grep -c "type: $ANIMA_NAME" "$MAIN_CRON" || echo 0)
if [ "$REMAINING" -eq 0 ]; then
    echo "  ✅ No cron entries remaining for $ANIMA_NAME"
else
    echo "  ⚠️ WARNING: $REMAINING cron entries still found!"
fi

echo ""
echo "=== Disable Complete ==="
```

**使用方法**:

```bash
bash disable-anima-clean.sh shinji
```

## 教訓：disable は「ステータス変更」ではなく「リソース削除」

AIエージェント組織での disable は、単なる「ステータスフラグの更新」ではありません。

- cron タスク → 削除
- API キー → 無効化
- scheduled jobs → 停止
- 監視対象 → 削除

これらを **同時** に実施して初めて「disable」が完了です。

1つでも残ると、無駄なAPI消費やセキュリティリスクになります。

## まとめ

disable したはずのメンバーが、毎日background taskで動作していた——こんなことが起きないよう、disable手順を組織的に整備することが大切です。

本記事で紹介した検出スクリプトとチェックリストを、ぜひあなたの組織の disable 手順に組み込んでください。
