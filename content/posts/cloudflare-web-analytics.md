---
title: "Cloudflare Web Analytics でプライバシーに配慮したアクセス解析を入れる"
displayTitle: "Cloudflare Web Analytics で<br>プライバシーに配慮したアクセス解析を入れる"
date: 2026-05-19T05:00:00+09:00
draft: false
tags: ["Cloudflare", "アクセス解析", "プライバシー", "Hugo", "Web Analytics"]
categories: ["自動化"]
description: "Cookie を使わずアクセス解析できる Cloudflare Web Analytics を Hugo ブログに導入する手順。GA4 との違い、プライバシーポリシーへの影響、設定の落とし穴まで実体験ベースでまとめます。"
---

[Search Console を導入した記事](/posts/hugo-search-console-setup/)で「GA4 を入れる人が多い」と書きましたが、私は GA4 を入れませんでした。代わりに **Cloudflare Web Analytics** を使っています。

理由は単純で、**Cookie を使わずに、最低限の数字だけ見たかった**から。今回はその導入記録です。

## なぜ GA4 を入れなかったか

Google Analytics 4 は高機能ですが、個人ブログにはオーバースペックだと感じました。

- **Cookie を使う** → プライバシーポリシーで同意取得の話が必要になる
- **設定項目が多い** → 最初の学習コストが高い
- **見たい数字は実はシンプル** → 「どの記事が読まれてるか」「何人来たか」くらい

個人の技術ブログで知りたいのは、ざっくりした傾向だけ。**そのために Cookie バナーを出して読者体験を削るのは本末転倒**だと考えました。

## Cloudflare Web Analytics の特徴

Cloudflare が無料で提供しているアクセス解析です。

- **Cookie を使わない**: 個人を追跡しない設計
- **クライアント側に重い JS を入れない**: 軽量ビーコン1つ
- **Cloudflare 利用者なら追加料金なし**
- **GDPR / 各国プライバシー法に配慮した作り**

計測できるのは、ページビュー・訪問者数・参照元・国・デバイス種別など。「誰が」ではなく「どれくらい」を見るツールです。

## 導入手順

### ① Cloudflare で Web Analytics を有効化

1. Cloudflare ダッシュボード → 左メニュー「**分析とログ**」→「**Web Analytics**」
2. 「**サイトを追加**」
3. ドメイン `yakiimo-tech.com` を入力
4. 表示される **JS スニペット**（`<script defer src="https://static.cloudflareinsights.com/beacon.min.js" ...>`）をコピー

### ② Hugo にスニペットを埋め込む

`layouts/partials/extend_head.html`（PaperMod なら head 拡張用 partial）に貼ります。

```html
{{ if eq (getenv "HUGO_ENV") "production" }}
<script defer
  src="https://static.cloudflareinsights.com/beacon.min.js"
  data-cf-beacon='{"token": "あなたのトークン"}'></script>
{{ end }}
```

`if eq ... "production"` で囲っておくと、**ローカル開発時はビーコンが飛ばない**ので、自分のテストアクセスで数字が汚れません。

### ③ デプロイして確認

`git push` → Cloudflare Pages デプロイ → 数時間後にダッシュボードに数字が出始めます。リアルタイムではなく、**反映に少しラグがある**点は把握しておくと安心。

## プライバシーポリシーへの影響

Cookie を使わないとはいえ、第三者（Cloudflare）がアクセス情報を処理することに変わりはないので、プライバシーポリシーには一言入れておくのが誠実です。

私のサイトではこう書いています。

> 当ブログでは、Cookie を使用しない Cloudflare Web Analytics を利用してアクセス状況を把握しています。これは個人を特定する情報を収集しません。

GA4 のときに必要だった「Cookie 同意バナー」は不要になります。読者体験を削らずに済むのが大きい。

## ハマりどころ

### 1. ローカル開発の数字が混ざる

`HUGO_ENV` で本番判定をしないと、`hugo server` でのローカル確認アクセスまで計測されます。Cloudflare Pages のビルド環境変数で `HUGO_ENV=production` を設定し、テンプレート側で分岐しておく。

### 2. 広告ブロッカーで一部計測されない

`cloudflareinsights.com` をブロックする拡張機能を入れている読者はカウントされません。**実数より少なめに出る**前提で見るのが正解。正確な売上計測には向かないですが、傾向把握には十分です。

### 3. CSP を設定しているとビーコンがブロックされる

[CSP を設定した](/posts/cloudflare-pages-csp-headers/)場合、`script-src` と `connect-src` に `static.cloudflareinsights.com` / `cloudflareinsights.com` を許可しないとビーコンが動きません。CSP 導入済みのサイトは、ここを忘れると無言で計測ゼロになります。

### 4. 反映が遅くて「動いてない？」と焦る

設定直後はダッシュボードに何も出ません。数時間〜半日待つ必要があります。最初の確認は焦らず、翌日見るくらいでちょうどいい。

## GA4 と使い分けるなら

「もっと詳しく分析したい」と思ったら、後から GA4 を**併用**することもできます。

- **Cloudflare Web Analytics**: 日常のざっくり把握（Cookie なし）
- **GA4**: コンバージョン計測や詳細なユーザー行動分析（Cookie あり）

ただし GA4 を足すと Cookie 同意の話が戻ってくるので、「**本当にその粒度の分析が必要か**」を考えてから判断するのがおすすめです。私はいまのところ Cloudflare Web Analytics だけで足りています。

## まとめ

- 個人ブログのアクセス解析は **Cookie なしの Cloudflare Web Analytics** が手軽
- Hugo なら `extend_head.html` にスニペット1つ + 本番判定で完了
- Cookie 同意バナーが不要 → 読者体験を削らない
- CSP 設定済みサイトは許可ドメインの追加を忘れずに

「数字を見たいけど、読者を追跡したいわけじゃない」という個人ブログには、ちょうどいい温度感の解析ツールでした。

次回は、Cloudflare Web Analytics の数字を毎週自動で要約して Notion に記録する仕組みを書く予定です。
