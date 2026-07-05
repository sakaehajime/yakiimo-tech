---
title: "MarkdownをやめてHTMLに？Anthropic社員が提唱する「読まれる報告書」の新常識"
date: 2026-07-05T11:00:00+09:00
draft: true
tags: ["Anthropic", "Claude Code", "HTML", "AI活用"]
categories: ["AI活用"]
---

AIに報告書や計画書を書かせると、長いMarkdownのテキストが返ってきて、正直読みきれない。そんな経験はありませんか。今回は、AnthropicのClaude CodeチームのThariqさん（@trq212）がXとClaude Blogで発表した仕事術「The Unreasonable Effectiveness of HTML（HTMLの不合理なほどの有効性）」を紹介します。 😊

![Markdown、読みきれない問題](/n/claude-code-html-reports-slides/slide_01.jpg)

## 提案はシンプル「AIの出力をHTMLにする」

Claude Codeに計画書・レポート・レビュー結果を書かせる時、普通はMarkdownのテキストで受け取ります。Thariqさんの提案は、これを「インタラクティブなHTMLページ」として出力させるというものです。

![MarkdownをやめてHTMLに](/n/claude-code-html-reports-slides/slide_02.jpg)

なぜHTMLなのか。理由は3つあります。

- Claude Codeはファイル・Git履歴・Slack や Linear（MCP経由）・ブラウザなど大量の情報を読めるので、それらを統合したリッチなレポートを作る材料が揃っている
- HTMLならグラフ・フローチャート・注釈付きの差分・スライダー付きの試作品など、文章では伝わりにくいものを視覚的に表現できる
- ブラウザでそのまま見られるので、リンクを送るだけで共有できる

![視覚的で共有もラク](/n/claude-code-html-reports-slides/slide_03.jpg)

## 具体的な使い方4パターン

### 1. 仕様策定・計画

複数のアプローチをグリッド状に並べて比較検討します。文章で「案Aは〜、案Bは〜」と読むより、並べて見比べる方が圧倒的に判断が速いです。

![使い方1: 計画づくり](/n/claude-code-html-reports-slides/slide_04.jpg)

### 2. コードレビュー

PRの変更点を注釈付きの差分やフローチャートで可視化します。Thariqさんは「GitHubの標準の差分表示より分かりやすい」とまで言っています。

![使い方2: コードレビュー](/n/claude-code-html-reports-slides/slide_05.jpg)

### 3. デザイン・プロトタイプ

スライダーでアニメーションの動きを調整できる試作品をその場で作らせます。「もう少し速く」「もう少し弾む感じ」を口頭で往復するより速いです。

![使い方3: プロトタイプ](/n/claude-code-html-reports-slides/slide_06.jpg)

### 4. 使い捨てエディタ

個人的にはこれが一番面白い発想でした。チケットの並び替えや設定変更など、その作業専用の一時的な編集画面をAIに作らせて、画面上で操作した結果をAIに戻すというワークフローです。ツールを「開発」するのではなく、その場で「使い捨てる」わけです。 😏

![使い方4: 使い捨てエディタ](/n/claude-code-html-reports-slides/slide_07.jpg)

## トレードオフも正直に語られている

いいことずくめではなく、弱点も明確です。

![注意: 生成は2〜4倍遅い](/n/claude-code-html-reports-slides/slide_08.jpg)

- HTMLはMarkdownより生成に2〜4倍の時間がかかる
- HTMLの差分はノイズが多く、Gitでのバージョン管理には向かない

![注意: Gitと相性が悪い](/n/claude-code-html-reports-slides/slide_09.jpg)

ただし生成コストについては、100万トークン級のコンテキストが当たり前になった今、トークン増は実用上の問題にならない、というのが筆者の立場です。

## 本当の狙いは「ループ内に留まる」

この手法の核心は見た目ではありません。長いMarkdownの計画書は誰も読まなくなり、結果として「読まずに承認」、つまりAIへの丸投げが起きます。視覚的に分かりやすいHTMLなら人がちゃんと読むので、開発者が意思決定に関与し続けられる。Thariqさんはこれを「Stay in the Loop（ループ内に留まる）」と呼んでいます。

![本当の狙いはループ内に留まる](/n/claude-code-html-reports-slides/slide_10.jpg)

AIの出力が増えるほど「読まれない文書」も増えます。毎日の自動レポートをHTMLで受け取る形に変えるだけで、読む率が上がって丸投げ事故を減らせる。地味ですが効く考え方だと思います。 😌

この内容はショート動画でも配信しています。チャンネル「AI Times -サキ-」もよろしくお願いします。

出典: Thariqさん（@trq212、Anthropic Claude Codeチーム）のXポストおよびClaude Blog
※この記事のスライドはNotebookLMで生成しています。

![最後まで読んでくださりありがとうございます。よろしかったら、いいね / レビュー / ブックマークをお願いします！](https://yakiimo-tech.com/n/common/article-cta.jpg)
