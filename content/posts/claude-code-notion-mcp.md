---
title: "Claude Code × Notion MCP で運用レポートを自動投稿する仕組み"
displayTitle: "Claude Code × Notion MCP で<br>運用レポートを自動投稿する仕組み"
date: 2026-05-03T00:30:00+09:00
draft: false
tags: ["Claude Code", "MCP", "Notion", "AI", "自動化"]
categories: ["AI & Claude Code"]
description: "Claude Code から MCP（Model Context Protocol）経由で Notion に直接ページを作成・検索する実装ガイド。MCPの仕組み・Notion MCPのセットアップ・実際の運用レポート自動投稿のコード例まで、実体験ベースでまとめます。"
---

Claude Code を使い始めて2週間ほど経った頃、「**会話の中で書いた運用レポートを Notion に直接保存できたらラクなのに**」と思うようになりました。

今までは、Claude が出してくれたレポートをコピーして、Notion を開いて、ページを作って、ペーストする。地味に手数が多い。これを **MCP（Model Context Protocol）** で自動化したので、その実装記録を残します。

## MCP とは

**MCP（Model Context Protocol）** は、Claude のような AI と外部ツールを接続するための共通プロトコルです。Anthropic が公開していて、多くの実装が出ています。

イメージとしては「**AIとサービスの間に立つ通訳**」。

```
[Claude Code]  ←→  [MCPサーバー]  ←→  [Notion API]
                  ↑
              ここが通訳
```

Claude は「Notion にページ作って」と言うだけ。MCP サーバーが Notion API を叩いて結果を返してくれる。Claude 側は API キーや HTTP の細かい話を知らなくていい、という分担です。

## なぜ Notion API を直接叩かないか

Claude API から直接 Notion API を叩く方法もあります。なぜ MCP を経由するか。

- **会話のフローで使える**: 「先週のレポート探して」「これ追記して」と日本語で書ける
- **認証情報の管理を外注できる**: API トークンは MCP サーバー側だけが持つ
- **複数ツールに横展開しやすい**: Notion 用に作った設定の感覚で Slack も Gmail も繋がる

要するに「Claude 自身を操作する手間が減る」のがメインの利益です。

## Notion MCP のセットアップ

### ① Notion 側のインテグレーション作成

1. Notion の「設定とメンバー」→「**インテグレーション**」
2. 「**新しいインテグレーションを作成**」
3. 名前（例: `Claude Code`）を入力、ワークスペースを選ぶ
4. 表示される **Internal Integration Token**（`ntn_...` で始まる）をコピー
5. アクセスしたい Notion ページ（例: 運用レポート用の親ページ）を開いて、右上「**...**」→「**接続を追加**」→ 作ったインテグレーションを選択

これで Notion 側は完了。

### ② Claude 側に MCP サーバーを登録

Claude Desktop or Claude Code の設定ファイル（`~/.claude/mcp_settings.json` など、環境による）に以下を追記。

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "NOTION_API_TOKEN": "ntn_xxxxxxxxxxxxxx"
      }
    }
  }
}
```

`NOTION_API_TOKEN` には①で取得したトークンを設定。**この値は絶対にコミットしない**。設定ファイル自体を `.gitignore` に入れるか、環境変数経由にしておくのが安全です。

### ③ Claude Code を再起動

設定変更後は再起動が必要。立ち上げ直すと、MCP サーバーが認識されて Notion 系のツールが Claude の手元に届きます。

## 使ってみる: 運用レポートを Notion に投稿

実際に私が使ったフロー。Claude にこう頼みました。

> 直近4日間のブログ運用をレポートにして、Notion の「やきいもテック ブログ開設レポート」ページの子ページとして追加してください。

Claude が裏でやってくれること:

1. `notion-search` で「やきいもテック ブログ開設レポート」を検索
2. ヒットしたページの ID を取得
3. `notion-create-pages` を呼んで、その ID を親に指定して新規ページを作成
4. 公開記事一覧やトラブル対処ログを Notion 形式の Markdown で書き込み

ターミナル上では Claude が「検索しました」「作成しました」と一行ずつ報告してくれます。終わったら Notion を開くと、新しい子ページがきれいに並んでいる。

**手動でやれば10分以上かかる作業**が、3行のお願いで終わります。

## 実装上のポイント

### 1. 親ページの ID は最初に検索で取る

「ID を直接指定してください」だと毎回貼り付けが必要ですが、「**○○というページの子として作って**」と頼めば Claude が検索→ID取得→作成、と勝手にやってくれます。会話で済むのが MCP のうまみ。

### 2. データソース vs データベース vs ページの違いに注意

Notion API の概念で、**データベースは複数のデータソースを持てる**仕組みになっています。データベースの直下にページを作ろうとしてエラーが出る場合、内側のデータソースを取ってから作る、というステップが必要なことがあります。

具体的には: `notion-fetch` でデータベースを取得 → 中の `<data-source url="collection://...">` を見つけて → そっちを親に指定する、という流れです。

### 3. Markdown 仕様の確認

Notion-MCP は独自の「Notion-flavored Markdown」を使います。普通の Markdown と微妙に違うところがあるので、初回は仕様を確認しておくと事故が減ります。MCP 経由で `notion://docs/enhanced-markdown-spec` を fetch すると公式仕様が読めます。

## ハマりどころ

### トークンを設定したのに動かない

Claude を再起動していない、または設定ファイルのパスが違うケースがほとんどです。`mcp_settings.json` の場所は環境によって違うので、ドキュメントを見て自分の環境に合うパスに置く。

### 「ページが見つかりません」と言われる

Notion 側のインテグレーション接続を該当ページにしていないとアクセスできません。**ページごとに「接続」が必要**な仕様なので、対象ページの「...」メニューからインテグレーションを追加します。

### URL とページ ID の混同

Notion のページ URL は `https://www.notion.so/Page-Title-1234567890abcdef` のような形式で、後ろの hex 文字列が ID です。MCP 経由で渡すときは ID だけ抜き出すか URL 全体を渡すかを揃えると安定します。

## 拡張アイデア

Notion MCP が動き出したら、横展開が楽になります。

- **Slack MCP**: 重要レポートが完成したら Slack にも通知
- **Gmail MCP**: 関連メールを検索してレポートに付ける
- **Google Calendar MCP**: 会議メモを Notion に同期

「会話で完結する自動化」が一段増えるたびに、生活がじわじわ楽になります。

## まとめ

- **MCP は AI と外部ツールを繋ぐ共通プロトコル**
- Notion MCP のセットアップは Notion 側のインテグレーション作成 + Claude 側の設定ファイル追記の2ステップ
- 「○○ページの子として作って」と日本語でお願いするだけで、検索 → ID 取得 → ページ作成が自動で走る
- トークンは絶対にコミットしない（環境変数 or 設定ファイル除外で管理）

Claude Code でブログ運用や日々の記録をしている人には、Notion MCP は導入コスト以上のリターンがあります。週次レポートや月次振り返りを Notion に蓄積する習慣がある方は、特に。

次回は、Notion MCP と組み合わせて Slack にも自動投稿する話を書く予定です。
