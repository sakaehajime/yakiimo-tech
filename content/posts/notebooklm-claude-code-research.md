---
title: "NotebookLM × Claude Code でゼロトークン・リサーチ"
displayTitle: "NotebookLM × Claude Code で<br>ゼロトークン・リサーチ"
date: 2026-04-24T10:00:00+09:00
draft: false
tags: ["Claude Code", "NotebookLM", "自動化", "トークン削減"]
categories: ["自動化"]
description: "Claude Code で調べ物をするとトークンを食います。リサーチ部分だけ無料の NotebookLM に外注して、Claude には要約だけ渡す。海外で話題の「ゼロトークン・リサーチ」手法を導入した話。"
---

Claude Code を使い込んでくると、じわじわ効いてくるのが**リサーチのトークン消費**です。

「このライブラリの使い方を調べて」「この記事の要点まとめて」——こういう調べ物を Claude に直接させると、入力も出力もトークンを食います。月末の請求を見てため息をつくタイプの人、いますよね（私です）。

そこで最近導入したのが **NotebookLM × Claude Code 連携**。海外で話題の「ゼロトークン・リサーチ」手法で、体感のトークン消費が劇的に減ります。今回はそのやり方を紹介します。

## 発想の転換：調べ物は NotebookLM に外注する

NotebookLM は Google の無料ツールで、**ソースを投入するとそれを読み込んで質問に答えてくれる** AI です。

発想はシンプルで、こうです。

1. 調べたい資料（URL・PDF・テキスト）を **NotebookLM に投入**
2. NotebookLM の中で質問を繰り返して、必要な情報を**要約させる**
3. Claude Code には**要約だけ**を渡す

ポイントは、NotebookLM の処理は**無料**ということ。つまり重い読解作業は NotebookLM に任せて、Claude Code には仕上げの実装だけをお願いする分担ができます。

従来 200 円かかっていた作業が 20 円で済む、くらいのインパクトです。

## セットアップ：`nlm` CLI を入れる

Claude Code からコマンド一発で NotebookLM を操作できる `nlm` という CLI があります。これを入れておきましょう。

```bash
npm install -g notebooklm-mcp-cli
nlm login
```

ブラウザが開いて Google ログインを求められるので、普段 NotebookLM で使っているアカウントでログイン。認証情報が `~/.notebooklm-mcp-cli/` に保存されます。

## スキルを入れると Claude が CLI を自発的に使う

`nlm` を入れただけだと Claude Code はこの CLI の存在を知りません。そこで「スキル」としてインストールします。

```bash
nlm skill install claude-code --level user
```

これで `~/.claude/skills/nlm-skill/` にスキルファイルが置かれて、Claude Code が自動で「あ、調べ物は `nlm` を使えばいいんだな」と理解してくれるようになります。

## 使ってみる

### ノートブックを作って URL を投入

```bash
nlm notebook create "Claude Code セキュリティ調査"
nlm source add <notebook-id> --url https://example.com/security-article
```

複数URLを投入すれば、それら全部を読み込んだ「そのテーマ専門家」が爆誕します。

### 質問する

```bash
nlm chat <notebook-id> "この記事で推奨されている対策をリスト化して"
```

NotebookLM が読んで、要約を返してくれます。**このやり取りは Claude のトークンを1つも消費しません**。

### Claude Code に仕上げだけお願いする

要約が手に入ったら、Claude Code に渡して実装を頼みます。

```
NotebookLMで整理した対策リストをもとに、settings.json に反映して
```

Claude は要約だけ読めばいいので、元記事を全文読み込むより遥かにトークンが少なくて済みます。

## 効く場面

この連携が特に効くのは、こういう作業。

- **新しいライブラリの公式ドキュメントを読み込ませる** — 複数ページ投入して「認証部分だけ要約して」
- **競合記事を大量に調査する** — 10本の記事を投入して「共通点と相違点を出して」
- **長尺の YouTube 動画を要約する** — URL 投入だけで OK
- **プロジェクトの決定事項を長期保存する** — auto memory とは別の「長期記憶」として使う

特に最後の「長期記憶」は地味に効きます。Claude Code のセッションはいずれ消えますが、NotebookLM に蓄積した知識は残り続けます。プロジェクト横断の知識ベースとして使えるんですね。

## 組み合わせで「2層の記憶」になる

Claude Code の記憶戦略を整理するとこうなります。

- **auto memory（`~/.claude/projects/` 配下）** — 短期〜中期。セッションごとの作業記憶
- **NotebookLM** — 長期。プロジェクト横断の知識ベース

どちらも役割が違うので、並行して使うのが正解です。

## 注意点

良いことずくめに聞こえますが、いくつか前提があります。

- **Chrome をリモートデバッグモードで起動**しておく必要がある場面もある（`nlm` の一部機能で）
- **Google アカウントでログイン**していること
- **NotebookLM は完全無料ではない**（ソース数やクエリ数に制限あり。ただし通常利用では十分すぎる量）

機密性の高いコードや社内資料を NotebookLM に投入するのは、当然だけどやめておきましょう。Google のサーバに載せることになるので、[セキュリティ記事](/posts/claude-code-security-basics/) の原則どおり「本番データと遊び場は混ぜない」を守ってください。

## まとめ

NotebookLM × Claude Code は、**重い読解を無料側に逃がして、実装だけ課金側に回す**という分担の話でした。

一度セットアップしてしまえば、あとは Claude が自発的に `nlm` を叩いてくれます。毎月の請求が気になる人、調べ物の多い人には本当におすすめ。

次回は、もう一段トークン消費を減らせる秘密兵器「RTK（Rust Token Killer）」を紹介します。
