---
title: "AIの脱獄危険度を測る新基準「CJS」を90秒で解説"
date: 2026-07-07T21:00:00+09:00
draft: false
tags: ["AIニュース", "AIセキュリティ", "Claude", "Anthropic", "生成AI"]
categories: ["サキのAIニュース"]
cover:
  image: "https://i.ytimg.com/vi/BdbNs-ORSRs/hqdefault.jpg"
  alt: "AIの脱獄危険度を測る新基準「CJS」を90秒で解説"
---

こんにちは、サキです！

今日はAIの「脱獄」の危険度を測る新基準「CJS」を解説します。

ちょっと物騒な言葉ですが、AIを安心して使うための大事な話です 😊

## 動画版（90秒）

{{< youtube BdbNs-ORSRs >}}

![脱獄の危険度を測る新基準CJS](https://lh3.googleusercontent.com/d/16b2Bougs1ondoUG1aQaJeP89UIkVmdxV)

## ジェイルブレイク（脱獄）とは

脱獄（ジェイルブレイク）とは、AIの安全ガードを特殊な指示で突破してしまうこと。

突破されると、本来出ないはずの危険な回答が出てしまいます。

![安全ガードを突破する攻撃のこと](https://lh3.googleusercontent.com/d/1zuyg4Z67NFZ7u_H8jXu8TE-g3te2qBHs)

## CJSは5段階の「ものさし」

Anthropicが提案したCJSは、脱獄の深刻度を0から4の5段階で採点する共通のものさしです。

![深刻度をCJS-0〜4の5段階で採点](https://lh3.googleusercontent.com/d/1uYSqXJui75JhSiY4aj2xU3MXTdbTb6Vb)

レベル0は実害ほぼなし。

数字が上がるほど深刻になって、レベル4は致命的です。

![0=実害なし ↔ 4=致命的](https://lh3.googleusercontent.com/d/1jiFSYfltEjTLXumpQhDy0O3QTEXjgaO9)

Webセキュリティの世界には「CVSS」という脆弱性の深刻度を採点する共通基準がありますが、そのAI版と考えると分かりやすいですよ 🤔

![WebでいうCVSSのAI版](https://lh3.googleusercontent.com/d/1Emge_38hH6DRnRDXrkcG-3rHCT5LTWIk)

## なぜ共通基準が必要？

共通のものさしがあれば、研究者も企業も同じ言葉で危険度を議論できて、修正も速くなります。

「これはCJS-3だから最優先で直そう」という会話ができるわけです。

![同じ言葉で危険度を語れる→修正が速い](https://lh3.googleusercontent.com/d/1oXi5OaelS5glt43boitaqEBZUW9pTo-C)

## Fable 5ではもう動いている

最新モデルのFable 5では、この考え方とセットで、依頼を4段階に自動分類する安全対策が動いています。

無害な依頼がたまに弾かれるのも、この仕様のためなんです。

![無害な依頼が弾かれるのも仕様のうち](https://lh3.googleusercontent.com/d/17pcwooNw-eD7eao7vCmObfmqkOm9hvmF)

## まとめ

AIの安全ニュースを見かけたら、「これはCJSでいくつ？」の視点で見てみてください。

ニュースの深刻度がぐっと読み取りやすくなります。

![「CJSいくつ？」の視点で見よう](https://lh3.googleusercontent.com/d/1pD5-FW2zbaPzLl-hhqnWLxcZ4Xn47ln8)

## 出典（一次ソース）

- Anthropic公式ニュース: [https://www.anthropic.com/news/fable-safeguards-jailbreak-framework](https://www.anthropic.com/news/fable-safeguards-jailbreak-framework)

※記事のイラストはGoogle NotebookLM、動画のナレーションは音声合成（Irodori-TTS）で生成しています。

![最後まで読んでくださりありがとうございます。よろしかったら、いいね・レビュー・ブックマークをお願いします！](https://yakiimo-tech.com/n/common/article-cta.jpg)
