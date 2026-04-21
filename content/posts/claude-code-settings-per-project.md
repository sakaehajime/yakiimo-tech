---
title: "プロジェクトごとに settings.json を使い分ける技"
displayTitle: "プロジェクトごとに<br>settings.json を使い分ける技"
date: 2026-04-22T10:00:00+09:00
draft: true
tags: ["Claude Code", "セキュリティ", "settings.json", "CLAUDE.md"]
categories: ["Webアプリのセキュリティ"]
description: "Claude Code の settings.json には実は3つの階層があります。グローバル・プロジェクト・ローカルを使い分けて、チーム開発でも安全に Claude Code を運用する方法をまとめました。"
---

前回の [Claude Code を安全に使うための6つの守り方](/posts/claude-code-security-basics/) で、`.env` を読ませない設定を紹介しました。今回はその一歩先、**プロジェクトごとに権限設定を切り替える方法**です。

全プロジェクト共通のルールと、このプロジェクトだけのルール、自分だけの好み。この3つを別々のファイルで管理できると、運用がぐっと楽になります。

## settings.json には3階層ある

実は `settings.json` は一箇所じゃありません。

| 置き場所 | ファイル名 | 役割 |
|---|---|---|
| `~/.claude/` | `settings.json` | **グローバル** — 全プロジェクト共通 |
| プロジェクト直下 `.claude/` | `settings.json` | **プロジェクト** — リポジトリで共有 |
| プロジェクト直下 `.claude/` | `settings.local.json` | **ローカル** — 自分だけ（gitignore対象） |

下に行くほど優先度が高い、という仕組みです。グローバルで「こう」と決めておいたルールを、プロジェクトで上書きできる、というイメージですね。

## それぞれに何を書くか

### グローバル（`~/.claude/settings.json`）

**自分が触る全プロジェクトに効いてほしい安全ルール**をここに書きます。私はこんな感じ:

```json
{
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(.envrc)"
    ]
  }
}
```

`.env` 系を読めないように deny しています。どのプロジェクトで作業しても、この設定が最初に効きます。

### プロジェクト（`.claude/settings.json`）

**そのリポジトリ特有のルール**をここに。Git にコミットして、チーム全員で共有するファイルです。

```json
{
  "permissions": {
    "allow": [
      "Bash(pnpm test:*)",
      "Bash(pnpm lint)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "WebFetch(domain:internal.example.com)"
    ]
  }
}
```

「テストとLintは都度確認しなくていい」「本番ドメインへのアクセスはダメ」など、プロジェクトの事情を書きます。

### ローカル（`.claude/settings.local.json`）

**自分だけの好み**をここに。`.gitignore` に必ず追加してください。

```json
{
  "permissions": {
    "allow": [
      "Bash(open:*)",
      "Bash(code:*)"
    ]
  }
}
```

Mac だけで使う `open` コマンド、自分だけ使う VSCode 起動など、チームに共有しなくていいものを置く場所です。

## 危険コマンドは「事前承認制」にする

deny に入れておくと便利な代表例を挙げておきます。

- `Bash(rm -rf *)` — 言わずもがな
- `Bash(git push --force:*)` — 履歴を吹き飛ばすやつ
- `Bash(curl * | sh)` — ネット上のスクリプトを即実行するパターン
- `Read(**/credentials.json)` — 認証情報系

こう書いておけば、仮に Claude が「ちょっとこのコマンド実行していい？」と聞いてきても、deny されているので実行されません。ユーザーが承認ボタンを押し間違えたときの保険になります。

## チーム開発での使い分け戦略

Git 管理する `.claude/settings.json` と、しない `.claude/settings.local.json` をちゃんと分けるのが最大のポイント。

**共有したいもの** → `settings.json`
- 本番ドメインへのアクセス禁止
- プロジェクト共通で許可したい定型コマンド

**共有したくないもの** → `settings.local.json`
- 個人のエディタ起動コマンド
- 一時的に許可したい実験的な権限

`.gitignore` には `settings.local.json` を必ず書いておきましょう。うっかり自分の設定をチームに流してしまうと、他のメンバーで動かなくなる原因になります。

## 書き換えたら Claude Code を再起動

地味に忘れがちなのがこれ。`settings.json` は Claude Code の起動時に読まれるので、書き換えたら一度終了して立ち上げ直してください。

「設定したのに効いてない」というときは、だいたい再起動忘れです。

## まとめ

`settings.json` の3階層をまとめるとこう:

- **グローバル** — 自分の全プロジェクトに効かせたい安全ルール
- **プロジェクト** — チームで共有するルール（Gitにコミット）
- **ローカル** — 自分だけの好み（gitignore対象）

この仕組みを理解して使い分けられると、「便利さはそのままに、危険な操作だけ機械的に止める」運用ができます。セキュリティは「気をつける」より「仕組みで守る」が原則なので、ぜひ取り入れてみてください。

次回は、[スラッシュコマンド活用術](/posts/claude-code-slash-commands/) の記事で予告していた「自分専用スラッシュコマンドを作る方法」を紹介します。
