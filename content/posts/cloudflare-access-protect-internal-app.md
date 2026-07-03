---
title: "Cloudflare Access で社内アプリを5分で保護する手順"
displayTitle: "Cloudflare Access で社内アプリを<br>5分で保護する手順"
date: 2026-07-05T09:00:00+09:00
draft: false
tags: ["Cloudflare", "認証", "Cloudflare Access", "Webアプリのセキュリティ"]
categories: ["インフラ"]
description: "自作の社内アプリに独自のパスワード認証を実装するのはやめたほうがいい。Cloudflare Access を使えば、メールアドレスベースの認証を設定ファイルなし・コードなしで追加できる。Zero Trust ダッシュボードの操作手順とハマりどころをまとめました。"
---

私が作る社内アプリには、最初から Cloudflare Access を前提にしています。パスワード認証を自分で書かない、というルールを設けているからです。

理由はシンプルで、自前で認証を実装すると「セキュリティの穴を自分で管理しなければならない」からです。パスワードのハッシュ化方式、セッション管理、ブルートフォース対策、ログアウト処理——どれか1つ抜けるだけで脆弱になります。Cloudflare Access を使えばそこをまるごと委任できます。 😌

今回は、Cloudflare Workers + Next.js で作った社内アプリに Access を設定した手順をそのまま紹介します。

## 前提の確認

作業の前に確認しておくべきことが2つあります。

### ① アプリのドメインが Cloudflare で管理されていること

Cloudflare Access はドメインの DNS が Cloudflare 側にある前提で動きます。Cloudflare Pages や Workers にデプロイしているなら、デフォルトでそうなっています。独自ドメインを使う場合は、レジストラの NS を Cloudflare に向けてください。

### ② Cloudflare アカウントが無料プランでもOK

Zero Trust (Access を含む機能群) は、月50ユーザーまで無料です（2026年5月時点）。個人開発や少人数の社内ツールなら課金せずに使えます。

## 手順

### ① Zero Trust ダッシュボードへ

[Cloudflare ダッシュボード](https://dash.cloudflare.com) にログインして、左メニューの「Zero Trust」を開きます。初めての場合は「Get started」でチームドメイン名（`yourteam.cloudflareaccess.com`）を設定する画面が出ます。これはサインイン画面のURLになるので、適当な名前でOKです。なお、Cloudflare のUIは変更される場合があります。メニュー名が違う場合は「Access」や「Zero Trust」で検索してみてください。

### ② Application を作成

Zero Trust ダッシュボード → Access → Applications → 「Add an application」。

種類を選ぶ画面が出ます:
- Self-hosted: 自前サーバー・Cloudflare Workers など、自分で管理しているアプリに使う
- SaaS: Salesforce・GitHub など外部SaaSに使う

社内アプリなら「Self-hosted」を選びます。

設定項目:
- Application name: アプリ名（後から変更できる）
- Session Duration: セッションの有効期間。`24 hours` で十分
- Application domain: 保護したいドメインまたはパス。`app.example.com` や `workers.dev のドメイン/admin` のように絞れる

### ③ ポリシーの設定

Application を作った後、Access ポリシーを設定します。「誰にアクセスを許可するか」のルールです。

「Add a policy」から以下を設定:

- Policy name: `allow-team` など任意の名前
- Action: `Allow`（許可する）
- Configure rules: 「Include」→ `Emails` を選んで、許可するメールアドレスを直接入力

メールアドレスを直接入れる方法が最もシンプルです。`@example.com` のドメイン全体を許可する「Email domain」も使えます。

保存して戻ると、Application に紐づいたポリシーが有効になります。

### ④ 動作確認

設定完了後、対象のURLをブラウザで開くとメールアドレスの入力画面が表示されます。

1. メールアドレスを入力
2. Cloudflare から認証コードが届く（ワンタイムパスワード）
3. コードを入力するとアプリにアクセスできる

許可していないアドレスは「Access denied」になります。 😊

## ハマりどころ

### `preview_urls` のバイパスに注意

Cloudflare Workers や Pages のプレビューURLは、デフォルトで Access の外にいます。

つまり本番の `/` には Access がかかっていても、プレビューURL（`https://xxx.pages.dev` など）はそのまま誰でも開けてしまいます。

`wrangler.jsonc` に以下を必ず入れてください。

{{< notice type="warning" >}}
`preview_urls` を `false` にしないと、Cloudflare Access で本番を保護していても、プレビュー環境が認証なしで公開されたままになります。
{{< /notice >}}

```jsonc
{
  "preview_urls": false
}
```

### Application Domain の入力ミス

`https://` を含めてはいけません。ドメインのみを入れます。

- 間違い: `https://app.example.com`
- 正しい: `app.example.com`

あとスラッシュも不要。パスだけ絞りたい場合は「Path」フィールドに入れます。

### Pages のプレビューブランチ

Cloudflare Pages は `main` 以外のブランチを push すると自動でプレビューデプロイが走ります。このプレビューのURLも Access の管理外です。

Access の Application domain に `*.pages.dev` を追加するか、Pages の設定でプレビューデプロイを無効にするのが対策です。Pages → Settings → Builds & deployments → 「Branch deployment controls」から調整できます。

## これで何が変わるか

Cloudflare Access を入れると、アプリ側のコードに認証ロジックを一切書かなくて済みます。

アプリのロジックは「このメールアドレスを持つ人が来たら使わせる」という前提で動く。その判定を Cloudflare がやってくれる。セッション管理もトークンの有効期限も全部 Access 側です。

私はこれを「認証を Cloudflare に外注する」と表現しています。自前で実装するとバグの責任も自分持ちですが、外注すれば「Access のセキュリティ品質を信頼する」という判断だけでいい。 😎

ついでに言うと、「独自パスワードのハッシュ値をJSコードに書かない」という原則も自然に守れます。パスワード認証を実装する場合、ブラウザに送られるJSにハッシュが埋め込まれるケースがあります。ハッシュが漏れると解析される可能性があります。Access はそもそもパスワードを使わず、メール認証（OTP）で動くので、そもそも起きない構造です。

## まとめ

- Cloudflare Access は Zero Trust ダッシュボードから GUI で設定できる
- Self-hosted アプリに付けるなら Application → Policy → Email 許可の3ステップ
- `preview_urls: false` は必須。プレビューURLが認証をバイパスする
- 認証ロジックをアプリに書かなくていい、というのが一番の価値

独自パスワード認証は実装コストのわりにリスクが高い。個人開発や少人数の社内ツールほど、最初から Cloudflare Access に乗っかるほうが楽です。

![最後まで読んでくださりありがとうございます。よろしかったら、いいね / レビュー / ブックマークをお願いします！](https://yakiimo-tech.com/n/common/article-cta.jpg)
