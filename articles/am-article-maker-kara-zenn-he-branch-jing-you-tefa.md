---
title: article_maker から Zenn へ branch 経由で反映するテスト
emoji: 🔍
type: tech
topics:
- zenn
- github
- branch
- article_maker
published: false
---

やあ！みんな！探求者のケイだよ！

この記事は、この `article_maker` から共有の `zenn-content` リポジトリへ記事を書き出し、`incoming/article-maker` branch を経由して Zenn に反映するための疎通確認だよ。まずは 1 本だけ安全に流して、生成物の置き場所と GitHub の PR 導線を確認するのが目的なんだ。

## 🔍 今回の確認ポイント

今回のテストでは、`output/zenn/articles/` にローカル成果物が出ることと、同じ内容が共有 repo の `articles/` にも同期されることを確認するよ。公開ブランチの `main` を直接触らず、作業用 branch だけを更新するので、Ubuntu 側の主系統ともぶつかりにくい構成なんだ。

## 🪜 実際の流れ

手順は単純で、`article_maker` で本文を作る、Zenn exporter で Markdown を出す、`incoming/article-maker` に commit する、最後に `main` への PR を作る、の 4 段階だよ。途中で slug に `am-` 接頭辞を付けているので、他プロジェクトの記事とも識別しやすいんだ。

```bash
git checkout incoming/article-maker
git add articles/*.md
git commit -m "Add Zenn test article from article_maker"
git push origin incoming/article-maker
```

## ✅ テストとして十分な理由

Zenn 反映の本質は、本文の品質そのものよりも、`articles/<slug>.md` が正しい frontmatter 付きで GitHub に載ることなんだ。だから最初の 1 本は短い検証記事でも十分価値があるよ。ここが通れば、次は自動生成の本文を同じレールに乗せればいいんだ。

それじゃあ、また次の探求で会おう！