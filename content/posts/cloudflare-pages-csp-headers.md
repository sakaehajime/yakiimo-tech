---
title: "Cloudflare Pages の `_headers` で CSP を設定する。Hugo ブログでもできる多層防御"
displayTitle: "Cloudflare Pages の <code>_headers</code> で CSP を設定する。<br>Hugo ブログでもできる多層防御"
date: 2026-04-30T04:00:00+09:00
draft: false
tags: ["Cloudflare Pages", "CSP", "セキュリティ", "Hugo", "ヘッダー"]
categories: ["Webアプリのセキュリティ"]
description: "Hugo + Cloudflare Pages で運用している静的ブログに、CSP（Content Security Policy）などのセキュリティヘッダーを `_headers` ファイルだけで導入した記録。最小構成と Report-Only モード、ハマりどころまでまとめます。"
---

[前回記事](/posts/hugo-search-console-setup/) の最後で予告した CSP（Content Security Policy）編です。

「静的ブログにセキュリティヘッダーって必要？」と最初思いましたが、結論からいうと**入れたほうがいい**です。Cloudflare Pages なら `_headers` ファイル1つで完結するので、コストもほぼゼロ。今回は最小構成と運用上のコツをまとめます。

## CSP は何を守るか

CSP は HTTP レスポンスヘッダーの1つで、ブラウザに「このページではこういうリソースだけ読み込んでいいよ」と教える仕組みです。

主に守れるのは:

- **XSS（クロスサイトスクリプティング）**: 攻撃者がページに勝手な `<script>` を埋め込もうとしても、CSP で許可されていないなら実行されない
- **不正な外部リソース読み込み**: 知らないドメインの画像・CSS・JS が混入するのを防ぐ
- **iframe 埋め込み攻撃**: クリックジャッキング対策

静的ブログでも、コメント欄やアフィリエイトタグ経由で第三者スクリプトを呼ぶ場合があります。**CSP がないと、何かに侵入された瞬間に好き放題されかねない**。最小限のヘッダーを入れておくのが地味に効きます。

## Cloudflare Pages の `_headers` 仕組み

Cloudflare Pages は、サイトのルートに `_headers` という名前のテキストファイルを置いておくと、その内容を**全レスポンスに自動で付与**してくれます。

Hugo の場合、`static/_headers` に置けば、ビルド時に `public/_headers` にコピーされてデプロイされます。`hugo.toml` の変更も不要。

```
# static/_headers
/*
  X-Content-Type-Options: nosniff
  X-Frame-Options: DENY
  Referrer-Policy: strict-origin-when-cross-origin
```

`/*` は「すべてのパスに適用」の意味。インデントがスペース2つなのが地味なポイントで、タブだと反応しないことがあります。

## 最小構成のヘッダー4点

私が `_headers` に入れた4つを紹介します。

### 1. `X-Content-Type-Options: nosniff`

ブラウザの「コンテンツタイプ推測」を止めるヘッダー。攻撃者がテキストファイルを JavaScript として実行させる手口を防ぎます。書き得しかないので必須。

### 2. `X-Frame-Options: DENY`

自分のページが他サイトの iframe に埋め込まれるのを禁止。クリックジャッキング対策の基本。`SAMEORIGIN` でも良いですが、ブログなら `DENY` で十分。

### 3. `Referrer-Policy: strict-origin-when-cross-origin`

外部サイトに飛ぶときに送信される「どこから来たか」情報を制限。HTTPS → HTTPS のみオリジン情報を送り、それ以外では何も送りません。**プライバシーへの配慮**として標準的な設定です。

### 4. `Content-Security-Policy`

本命。許可するリソースの出所をホワイトリストで指定します。

```
Content-Security-Policy: default-src 'self'; img-src 'self' https://images.unsplash.com data:; style-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-inline'
```

### 各ディレクティブの意味

- `default-src 'self'`: 基本は同一オリジンのリソースのみ許可
- `img-src 'self' https://images.unsplash.com data:`: 画像は自サイト + Unsplash + Data URI を許可（ブログのヒーロー画像が Unsplash 由来のため）
- `style-src 'self' 'unsafe-inline'`: CSS は自サイト + インラインを許可（Hugo テーマがインラインスタイルを使う）
- `script-src 'self' 'unsafe-inline'`: JS も同様

`'unsafe-inline'` は本来 CSP の意味を弱めるディレクティブですが、Hugo + PaperMod は内部でインラインスクリプトを使う場面があり、外すとサイトが壊れます。**現実解として残す**のが私の選択でした。

## Report-Only モードで先にテスト

いきなり強い CSP を入れると**サイトが真っ白になる事故**が起きます。最初は `Content-Security-Policy-Report-Only` ヘッダーで試すのが安全。

```
/*
  Content-Security-Policy-Report-Only: default-src 'self'
```

この名前にすると、CSP 違反は**ブロックせず警告だけ**ブラウザのコンソールに出力されます。違反が出たリソースを `_headers` のホワイトリストに追記してから、`Content-Security-Policy` 本番ヘッダーに切り替える、という流れがおすすめです。

## 動作確認の方法

`_headers` を書いてプッシュ → Cloudflare Pages がビルド & デプロイ。1〜2分後に以下で確認します。

### ① ブラウザの Network タブ

DevTools → Network → ページ再読み込み → 自分のページを選択 → Response Headers を見る。

`Content-Security-Policy` などが付与されていれば成功。

### ② オンラインの検証ツール

[Mozilla Observatory](https://observatory.mozilla.org/) に自サイトのURLを入れると、セキュリティヘッダーの設定具合をスコアで評価してくれます。最初は D 〜 C くらい、`_headers` を整えると B+ あたりに上がるはず。

## ハマりどころ

### ① インデントはスペース、タブだと無視される

`_headers` のシンタックスはタブに弱め。私はこれで30分溶かしました。エディタの設定でスペース2つに揃えるのが鉄板。

### ② 行末スペースで設定が壊れることがある

ヘッダー行の末尾に余計なスペースが入ると、Cloudflare 側でパースエラーになり全体が無効化される場合があります。`.editorconfig` で末尾スペース除去を有効にしておくと安心。

### ③ `_headers` をコミットし忘れる

`static/_headers` は Hugo 側のソース、`public/_headers` がビルド成果物。Git に入れるべきは `static/_headers` です。`public/` を `.gitignore` してビルドのみで生成、というケースだと忘れがち。

### ④ Unsplash の画像が消える

ヒーロー画像に Unsplash を使っているなら、`img-src` に `https://images.unsplash.com` を必ず足してください。私は最初これを忘れて、トップの森の画像が消えました。

## 入れた後の安心感

たった1ファイルでこれだけ守れるのは、静的サイトのいいところ。動的なフレームワークだと middleware の話になって面倒ですが、`_headers` は **書いて置くだけ**。

セキュリティ系の設定は「やってもアクセスは増えない」ので後回しにしがちですが、コストが極小で守りが厚くなるなら、ブログ運営の最初の段階で済ませておくのが楽です。

[前々回の block_env.py 記事](/posts/block-env-hook-implementation/) が「Claude Code に .env を読ませない」自分の開発環境側の防御だとすれば、今回の `_headers` は「読者ブラウザに変なリソースを読ませない」公開サイト側の防御。両方そろって多層防御になります。

次回はこの設定をしたあとの Mozilla Observatory スコアの変化を、実測ベースで紹介する予定です。
