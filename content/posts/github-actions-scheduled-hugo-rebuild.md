---
title: "GitHub Actions で Hugo の予約投稿を確実に拾わせる Scheduled Rebuild の作り方"
displayTitle: "GitHub Actions で Hugo の予約投稿を<br>確実に拾わせる Scheduled Rebuild の作り方"
date: 2026-05-11T00:00:00+09:00
draft: false
tags: ["GitHub Actions", "Hugo", "Cloudflare Pages", "予約投稿", "自動化"]
categories: ["自動化"]
description: "Hugo の buildFuture=false で除外される予約投稿を、GitHub Actions の cron で定時に Cloudflare Pages を再ビルドして拾わせる仕組み。実際に動いている YAML 全文と Deploy Hook の設定手順を実体験ベースで解説します。"
---

[Hugo + Cloudflare Pages を運用して詰まった5つのこと](/posts/hugo-cloudflare-pages-pitfalls/) で、自動デプロイが反応しない時のフォールバックとして「**Scheduled Hugo Rebuild という GitHub Actions ワークフロー**」に少しだけ触れました。

今回はその**ワークフローの中身**と、設定に必要な Cloudflare 側の準備をまとめます。Hugo + Cloudflare Pages 構成で予約投稿を運用する方には、入れておくと安心の仕組みです。

## なぜ定時リビルドが必要か

Hugo の設定 `buildFuture = false`（hugo.toml デフォルト）は、**記事の `date` が未来時刻だとビルド対象から除外**します。

「明後日の朝 10:00 に出てほしい記事」を今日プッシュしたとして、

```
記事の date:        2026-05-13T10:00:00+09:00
今日プッシュしたとき: 2026-05-11
```

プッシュした瞬間のビルドでは未来扱いで除外。指定時刻になっても**追加プッシュがなければ Cloudflare Pages は再ビルドしない**ので、永遠に出てきません。

これを解決するために、**「毎日定時に何もしないコミットなしで強制リビルド」**を仕込みます。

## 全体の流れ

```
[GitHub Actions cron]  毎日 JST 10:00
        ↓
[Deploy Hook URL を curl で叩く]
        ↓
[Cloudflare Pages がビルド開始]
        ↓
[未来日付だった記事の中で、現在 < date になったものが公開される]
```

Deploy Hook は Cloudflare Pages が用意してくれている「特定URLに POST すれば再ビルドする」エンドポイントです。

## Cloudflare 側の準備

### Deploy Hook を作る

1. Cloudflare ダッシュボード → Workers & Pages → 該当プロジェクト
2. 上部タブ「**設定**」→「**ビルドとデプロイ**」
3. 「**Deploy hooks**」セクション → 「フックを追加」
4. 名前（例: `scheduled-rebuild`）と対象ブランチ（`main`）を指定
5. 表示される URL（`https://api.cloudflare.com/client/v4/pages/webhooks/deploy_hooks/...`）をコピー

この URL は**シークレット扱い**です。誰でも叩けば誰でもリビルドできてしまうので、絶対に公開しないこと。

## GitHub 側の準備

### Secret に Deploy Hook URL を保存

1. GitHub のリポジトリページ → Settings → Secrets and variables → Actions
2. 「New repository secret」
3. Name: `CF_DEPLOY_HOOK`
4. Secret: 先ほどコピーした URL を貼る

Secret に入れておけば、ログにも表示されず、ワークフローからは `${{ secrets.CF_DEPLOY_HOOK }}` で参照できます。

### ワークフロー YAML を配置

`.github/workflows/scheduled-build.yml` に以下を保存。

```yaml
name: Scheduled Hugo Rebuild

on:
  schedule:
    # 毎日 JST 10:00（= UTC 01:00）に実行
    # cron書式: 分 時 日 月 曜日（★UTC基準★）
    - cron: '0 1 * * *'

  # 「Run workflow」ボタンで手動実行も可能
  workflow_dispatch:

jobs:
  trigger-deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Trigger Cloudflare Pages Rebuild
        run: |
          echo "Triggering Cloudflare Pages rebuild at $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
          curl -X POST "${{ secrets.CF_DEPLOY_HOOK }}" \
            -H "Content-Type: application/json" \
            --fail \
            --silent \
            --show-error
          echo ""
          echo "Rebuild triggered successfully."
```

これを main ブランチにプッシュすれば、翌 JST 10:00 から毎日自動実行されます。

## cron の落とし穴

**GitHub Actions の cron は UTC 基準**です。日本時間で書くと意図しない時刻に走ります。

| やりたい時刻（JST） | cron 書式（UTC） |
|----------------------|--------------------|
| 毎朝 10:00 | `0 1 * * *` |
| 毎晩 22:00 | `0 13 * * *` |
| 朝晩2回（10:00 / 22:00） | `0 1,13 * * *` |
| 3時間おき | `0 0,3,6,9,12,15,18,21 * * *` |

私のブログでは予約投稿が JST 午前10時前に集中するので、JST 10:00（UTC 01:00）に1回実行するだけで足りています。

## 手動実行も仕込んでおく

`workflow_dispatch:` を入れておくと、GitHub の Actions タブから**「Run workflow」ボタンで手動実行**できます。

```bash
# CLI からも叩ける
gh workflow run "Scheduled Hugo Rebuild"
```

公開直後の記事が反映されない時、これを叩けば即座にリビルドされます。日次の自動実行と組み合わせて、両方を持っておくと安心です。

## ハマりどころ

### 1. cron 開始までタイムラグがある

GitHub Actions の cron は、**設定後すぐには動きません**。コミットを main にプッシュしてから初回の発火まで、数十分〜数時間かかることがあります。最初は「動いてない？」と心配になりますが、翌日の指定時刻には確実に走るので待つ。

### 2. 高負荷時に遅延する

GitHub のインフラ状況によっては、cron が指定時刻ピッタリではなく**数分〜30分ほど遅れる**ことがあります。「JST 10:00 ぴったりに公開したい」という運用は避け、「**JST 09:00 までに記事の date を設定 → 10:00 過ぎに公開**」くらいの余裕を持たせるのが安全。

### 3. 失敗してもサイレントに終わる

`curl` が `--silent --show-error` 付きなので、Deploy Hook URL が間違っていても通知が来ない可能性があります。`--fail` を付けて HTTP エラー時に exit code 22 を返すようにしておくと、Actions タブで赤バツが出るので気付けます。

### 4. Deploy Hook を Public リポジトリに置かない

繰り返しますが Deploy Hook URL は**シークレット**。Public リポジトリの場合、`.github/workflows/` の YAML はそのまま見えるので、URL を直書きすると即漏洩します。**必ず GitHub Secret 経由で渡す**こと。

## 効果

導入してから、

- 予約投稿で時刻指定したら、忘れていても自動で出てくる
- ローカルでビルド失敗しても、定時に勝手に直る確率がある
- 手動デプロイトリガーも `gh workflow run` で叩ける

「**プッシュ忘れによる公開漏れ**」が事実上ゼロになりました。Hugo の予約投稿を運用する人には、入れない理由がない仕組みです。

## まとめ

- Hugo `buildFuture=false` で除外される未来日付記事を救うために、定時リビルドが必要
- Cloudflare Pages の **Deploy Hook URL** を GitHub Actions cron で叩く
- cron は **UTC 基準**（JST 10:00 = UTC 01:00）
- Deploy Hook は **GitHub Secret** に入れる（公開厳禁）
- `workflow_dispatch:` で手動実行ボタンも作っておく

15行ほどの YAML で、ブログ運用のストレスが一段消える仕組みです。

次回は、この仕組みを応用して「**プッシュごとに自動でデプロイログを Discord に通知する**」拡張を書く予定です。
