---
title: "Hugo の displayTitle で長いブログタイトルをきれいに2行表示する小技"
displayTitle: "Hugo の displayTitle で<br>長いブログタイトルをきれいに2行表示する小技"
date: 2026-05-07T03:00:00+09:00
draft: false
tags: ["Hugo", "PaperMod", "frontmatter", "ブログデザイン", "テンプレート"]
categories: ["自動化"]
description: "Hugo のテンプレートで title と displayTitle を分けて、本文ページではタイトルを意図した位置で改行する小技。一覧では1行、記事ページでは2行に分けて見せる実装手順を、実例ベースで紹介します。"
---

ブログを書いていると、**タイトルが横長になり過ぎて読みにくい**問題に当たります。

```
Hugo + Cloudflare Pages を運用して詰まった5つのこと（実体験ベース）
```

トップページの一覧では、自動的に折り返してくれて気になりませんが、**個別記事ページの大見出し**として表示すると、変なところで折り返されたり、デザインが崩れたりします。

そこで「**一覧では1行で、本文ページでは意図した位置で改行**」を両立させるのが、Hugo の **`displayTitle` パターン**。私のブログ全20記事すべてで使っている小技です。

## なぜ title に直接 `<br>` を書かないか

最初に思いつくのは、`title` に直接 `<br>` を入れる方法。

```yaml
title: "Hugo + Cloudflare Pages<br>を運用して詰まった5つのこと"
```

ですが、これは**やめたほうがいい**。

- ページの `<title>` タグや RSS フィードの `<title>` 要素にも `<br>` がそのまま入る
- Twitter / Facebook の OGP プレビューでも `<br>` が見えてしまう
- Google 検索結果のスニペットも崩れる

タイトルは「**マシン用のテキスト**」と「**ヒト用の表示**」で分けて持つのが正解です。

## 実装方法

### ① frontmatter に displayTitle を足す

```yaml
---
title: "Hugo + Cloudflare Pages を運用して詰まった5つのこと"
displayTitle: "Hugo + Cloudflare Pages を運用して<br>詰まった5つのこと"
---
```

`title` は機械が読む側、`displayTitle` は人が読む側、と役割を分けます。

### ② テンプレート側で displayTitle を優先する

`layouts/_default/single.html` の見出し部分を以下のように書きます。

```go
<h1 class="post-title">
  {{- with .Params.displayTitle -}}
    {{ . | safeHTML }}
  {{- else -}}
    {{ .Title }}
  {{- end }}
</h1>
```

ポイントは2つ:

- **`with` ... `else`**: displayTitle があればそっちを使い、なければ通常の Title にフォールバック
- **`safeHTML`**: `<br>` を文字列ではなく HTML として解釈させる

`safeHTML` を忘れると `&lt;br&gt;` がそのまま画面に出てしまうので注意。

### ③ トップページの一覧では title をそのまま使う

`layouts/index.html` でトップページの記事一覧を作る場合は、そっちでも displayTitle を使うか title を使うか選べます。私のブログでは、**トップページでも displayTitle を使う**ようにしています。

```go
<a href="{{ .RelPermalink }}">
  {{- with .Params.displayTitle -}}
    {{ . | safeHTML }}
  {{- else -}}
    {{ .Title }}
  {{- end -}}
</a>
```

トップでも本文ページと同じ位置で改行してくれて、一貫性が出ます。

## 実例

実際に使っている title / displayTitle のペアをいくつか:

```yaml
# 例1
title: "NotebookLM × Claude Code でゼロトークン・リサーチ"
displayTitle: "NotebookLM × Claude Code で<br>ゼロトークン・リサーチ"

# 例2
title: "Cloudflare Pages の `_headers` で CSP を設定する。Hugo ブログでもできる多層防御"
displayTitle: "Cloudflare Pages の <code>_headers</code> で CSP を設定する。<br>Hugo ブログでもできる多層防御"

# 例3（短いタイトルなら displayTitle 不要）
title: "やきいもテック、はじめました！"
# displayTitle なし → そのまま title が使われる
```

長いタイトルだけ displayTitle を足し、短いタイトルは省略する運用が楽です。

## ハマりどころ

### 1. `safeHTML` 忘れで `<br>` が文字として出る

最頻ミス。テンプレートの `{{ . | safeHTML }}` を `{{ . }}` と書くと、`<br>` が文字列扱いになって `&lt;br&gt;` として表示されます。

### 2. RSS フィードに `<br>` が混入

`<title>` タグに displayTitle ではなく title を使っているか確認。Hugo の標準 RSS テンプレートは title を使うので、明示的に displayTitle を書き換えていなければ問題ありません。

### 3. OGP メタタグも要確認

`<meta property="og:title" content="...">` も title であるべき。displayTitle だと SNS シェア時に `<br>` が見えてしまいます。PaperMod テーマ標準の挙動なら title が使われるので、独自テーマを書いた場合だけ注意。

### 4. 改行位置の選び方

意味の区切りで改行するのが基本。**「で」「は」「を」の後**で改行するのは避ける（読点的役割なので、その後ろは続いて欲しい）。**助詞の前か、句読点（、。）の後**がきれいに見えます。

```
× 「Hugo + Cloudflare Pages を運用<br>して詰まった5つのこと」
○ 「Hugo + Cloudflare Pages を運用して<br>詰まった5つのこと」
```

## まとめ

- **title** = マシン用（HTML title・RSS・OGP・SEO）
- **displayTitle** = ヒト用（h1見出しの表示）
- テンプレートでは `with .Params.displayTitle` + `safeHTML` で分岐
- 改行位置は意味の区切りで、助詞の前後が自然

この小技を入れるだけで、長いタイトルの記事でも見出しの収まりが一段良くなります。Hugo + PaperMod 構成で個人ブログを運用する方は、最初に仕込んでおくと後が楽。

次回は、displayTitle と組み合わせて使うと便利な「**サムネイル画像の自動生成（OGP用）**」の話を書く予定です。
