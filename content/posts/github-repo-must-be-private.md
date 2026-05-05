---
title: "GitHub リポジトリを必ず Private にすべき理由と、誤って Public にしてしまった時の対処"
displayTitle: "GitHub リポジトリを必ず Private にすべき理由と、<br>誤って Public にしてしまった時の対処"
date: 2026-05-06T03:00:00+09:00
draft: false
tags: ["GitHub", "セキュリティ", "リポジトリ管理", "Cloudflare Pages", ".env"]
categories: ["Webアプリのセキュリティ"]
description: "個人開発のリポジトリを GitHub に置くなら Private 必須。なぜ Public ではいけないのか、具体的に何が漏れるのか、誤って公開してしまった時の対処、Private のまま Cloudflare Pages にデプロイする方法までまとめます。"
---

「個人で作っているだけだから、リポジトリを Public にしてもいい」

開発を始めたばかりの頃、私もそう思っていました。実際、初めて作ったリポジトリは何の気なしに Public にしていた時期があります。

その後セキュリティを学んで、**個人開発こそ Private 必須**だと考えを改めました。今回はその理由と、誤って Public にしてしまった時の対処をまとめます。

## なぜ Public ではいけないのか

「動くコードを公開して何が悪いのか」と思うかもしれません。問題は**コード以外も一緒に公開される**点にあります。

### 1. API キーやパスワードが履歴に残る

「コミットしてから慌てて削除した」では消えません。Git は履歴を保持するので、**過去のどこかのコミットに `.env` を追加した瞬間がある**だけで、そのキーは永久に世界に晒され続けます。

GitHub には自動スキャナーがあり、Public リポジトリへのコミットは数分以内にスキャンされて、見つかった API キーは正規プロバイダ（AWS, Stripe など）に通報される仕組みになっています。逆に言うと、**通報前に悪意のあるBotが拾う可能性も同等にある**ということです。

### 2. 内部 URL や構成情報が漏れる

ソースコードには案外こういう情報が残ります。

- 開発環境のホスト名（`http://localhost:3000` だけでなく `https://staging.example.com` のような内部URLも）
- 使っているサービスの社内パス
- データベースのテーブル構造（マイグレーションファイル）
- 認証フロー（誰がどこにアクセスできる権限を持つか）

攻撃者が**本番環境に侵入する下準備**として、これらの情報は非常に価値があります。

### 3. 依存ライブラリのバージョンから脆弱性を逆算される

`package.json` や `requirements.txt` を見れば、攻撃者は「このサイトは○○の vX.Y.Z を使っている → そのバージョンには既知の脆弱性がある → これで攻撃が通るかも」と推測できます。

Public リポジトリは、攻撃者にとって**事前リコンの宝の山**です。

### 4. プライベートな試行錯誤がインターネットに残る

コミットメッセージや README に「あ、これ試してみる」とメモ書きしていたものが、永久にインターネットアーカイブに残ります。GitHub から削除しても Wayback Machine やミラーサイトに痕跡が残るケースは普通にあります。

## Private で運用する利益

逆に Private にしておけば、

- API キーをうっかりコミットしても、漏洩経路が限定される（共同開発者のみ）
- 本番URL、構成、依存関係、すべて自分の中だけ
- 試行錯誤も安心して残せる
- 後から会社や受託で使う可能性が出てきても、移管しやすい

GitHub の **Free プラン**でも Private リポジトリは無制限に作れます。Private にしないメリットは、ほぼゼロです。

## 誤って Public にしてしまった時の対処

うっかり Public で作ってしまった場合、**Settings → Visibility → Private に戻すだけでは不十分**です。Public だった期間に、誰かがクローンしている可能性があるからです。

### 即時対応の優先順位

1. **API キー・トークンを全部 Rotate**: 漏れた鍵は使えなくする。これが最優先
2. **リポジトリを Private に変更**: それ以上の流出を止める
3. **過去の機密ファイルを履歴から削除**: 後述

### 履歴から機密ファイルを完全削除する

Git は履歴を保持するため、ファイルを削除してコミットしても**過去のコミットには残っています**。完全削除には専用ツールが必要。

```bash
# git-filter-repo を使う（推奨）
pip install git-filter-repo
git filter-repo --path .env --invert-paths
git push --force --all
git push --force --tags
```

`git-filter-repo` は Git 公式が推奨する履歴書き換えツール。`.env` を全コミットから消去します。`--force` push は破壊的なので、共同開発者がいる場合は事前周知必須。

### それでも漏れた可能性は残る

繰り返しますが、**Public だった瞬間にクローンされた可能性は消せません**。鍵の Rotate を最優先にする理由はそこにあります。

## Private のまま Cloudflare Pages にデプロイする

「Private にしたら、Cloudflare Pages が読めなくなるのでは？」と心配になるかもしれませんが、問題ありません。

### 手順

1. Cloudflare Dashboard → Workers & Pages → 「ページを作成」
2. 「**Git に接続**」を選ぶ
3. GitHub アカウントを連携する画面で「**Cloudflare Pages インストール**」を承認
4. 表示されるリポジトリ一覧（Private 含む）から選ぶ
5. ビルド設定を入れて「保存してデプロイ」

Cloudflare Pages は GitHub Apps として連携するので、Private リポジトリでもアクセス権が付与されます。**コードはコードのまま Private、デプロイは自動**という運用が成立します。

### `wrangler.jsonc` がある場合の注意

Cloudflare Workers と組み合わせる構成の場合、`wrangler.jsonc` の `preview_urls` を **無効化（`false`）必須**です。

```jsonc
{
  "preview_urls": false
}
```

これを有効のままにすると、プレビュー環境がインターネット公開されてしまい、Cloudflare Access で本番を保護していても**プレビューは認証なしで覗かれます**。Private 化のメリットを台無しにする落とし穴です。

## 事前防止: コミット時点で .env を弾く

万が一を防ぐために、**コミットする前に .env を検出して止める**仕組みを入れておくと安心です。

### `.gitignore` に必ず書く

```
.env
.env.*
.envrc
!.env.example
!.env.sample
```

### pre-commit hook で念押し

`.git/hooks/pre-commit` に簡単なスクリプトを置きます。

```bash
#!/bin/bash
if git diff --cached --name-only | grep -E "\.env(rc)?(\..*)?$" | grep -v "\.example\|\.sample"; then
  echo "ERROR: .env file detected. Use .env.example for templates."
  exit 1
fi
```

権限を与えれば有効化:

```bash
chmod +x .git/hooks/pre-commit
```

これで、たとえば `git add .env` してから `git commit` しようとしても、コミット前に止まります。

## まとめ

- **個人開発こそ Private 必須**: API キー・内部URL・依存関係がすべて防衛資産
- **Public にしてしまったら、まず鍵を Rotate**。履歴削除は二番目
- **Private + Cloudflare Pages は普通に共存できる**（GitHub Apps 連携）
- **コミット前の防衛線**として `.gitignore` + pre-commit hook を入れる

セキュリティは「失敗してから学ぶ」と高くつくジャンルです。Private 設定は、デフォルトで全プロジェクトに適用しておくのが、結局一番楽な防御策でした。

次回は、`.env` 周りの実装をもう一段堅くする「PreToolUse hook と環境変数管理を組み合わせた多層防御」を書く予定です。
