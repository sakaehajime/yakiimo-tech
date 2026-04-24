---
title: "RTK で Claude Code のトークンを9割削減した話"
displayTitle: "RTK で Claude Code の<br>トークンを9割削減した話"
date: 2026-04-24T12:00:00+09:00
draft: false
tags: ["Claude Code", "RTK", "自動化", "トークン削減", "効率化"]
categories: ["自動化"]
description: "Claude Code のトークン消費を 60〜90% 削減できる Rust 製 CLI ツール RTK（Rust Token Killer）。Windows での導入手順と実測値、CLAUDE.md で Claude に自発的に使わせる設定まで紹介します。"
---

前回の [NotebookLM × Claude Code](/posts/notebooklm-claude-code-research/) で「調べ物のトークンを逃がす」話をしました。今回はそれと組み合わせたい **もう一つの秘密兵器、RTK（Rust Token Killer）** の話です。

## コマンド出力こそがトークンの敵

Claude Code を使っていると、こんな場面よくありますよね。

- `git log` を叩いたら 500 行のコミット履歴が Claude の目の前に流れる
- `pnpm test` の出力が何百行と表示されて、そのほとんどが「成功ログ」
- `gh pr view` で PR の全情報が延々と返ってくる

これ全部、**Claude のコンテキストに食い込んでトークンを消費しています**。本当に必要な情報はごく一部なのに、付随情報まで全部読ませているわけです。

そこで登場するのが RTK。コマンドと Claude の間に挟まって、**出力をフィルタリング・圧縮してから渡す** Rust 製の CLI プロキシです。

## インストール（Windows）

公式のインストールスクリプトは Unix 系向けなので、Windows はバイナリを直接取ってくる方式で入れます。

1. [RTK の GitHub Releases](https://github.com/anthropics/rtk) から `rtk-x86_64-pc-windows-msvc.zip` をダウンロード
2. 解凍して `rtk.exe` を `%USERPROFILE%\.local\bin\` に配置
3. `~/.local/bin` が PATH に入っていることを確認

動作確認:

```bash
rtk --version
# rtk 0.35.0
```

## 使い方：`rtk` を付けるだけ

難しい設定はいりません。**普段のコマンドの前に `rtk` を付けるだけ**。

```bash
# 今まで
git status
pnpm test

# RTK 版
rtk git status
rtk pnpm test
```

RTK 側で「このコマンドなら失敗したテストだけ抽出」「このコマンドならファイル単位に集約」といったフィルタが自動で効きます。

## 実測：カテゴリ別の削減率

Notion の検証レポートから抜粋した、代表的な削減率:

| カテゴリ | 代表コマンド | 削減率 |
|---|---|---|
| テスト | `rtk vitest` / `rtk playwright` | **90〜99%** |
| ビルド | `rtk tsc` / `rtk lint` | 70〜87% |
| Git | `rtk git status` / `rtk git diff` | 59〜80% |
| GitHub | `rtk gh pr view` | 26〜87% |
| パッケージ管理 | `rtk pnpm install` | 70〜90% |

テスト系の削減率は頭抜けていて、`rtk vitest` は**失敗したテストだけ**を抽出してくれるので、グリーンなテストが 500 個通過してもトークンはほぼゼロ。これはヤバい。

## CLAUDE.md に書いて自動化する

ただ、毎回自分で `rtk` を付けるのは忘れがち。そこで `~/.claude/CLAUDE.md` に「全コマンドに `rtk` を付けろ」ルールを書いておきます。

`rtk init -g` というコマンドを叩くと、グローバルの CLAUDE.md に以下のような指示が自動で追記されます。

```markdown
## RTK 利用ルール

すべての Bash コマンドには `rtk` プレフィックスを付ける。

# ❌ Wrong
git add . && git commit -m "msg" && git push

# ✅ Correct
rtk git add . && rtk git commit -m "msg" && rtk git push
```

これで Claude が勝手に `rtk` 付きで実行してくれるようになります。放っておいても節約される、最高。

## Windows での注意：Hook モードは使えない

RTK には「Hook モード」という更に踏み込んだ仕組みもあるんですが、これは Unix 専用。Windows では使えません。

代わりに Windows は [CLAUDE.md](/posts/how-to-write-claude-md/) モードでの運用になります。つまり「Claude に自発的に `rtk` を使わせる」やり方ですね。ちょっと遠回りに聞こえますが、実用上は十分効きます。

## 設定を変えたら Claude Code を再起動

これは [settings.json 記事](/posts/claude-code-settings-per-project/) でも触れましたが、`CLAUDE.md` の変更は再起動で反映されます。`rtk init -g` の直後は一度 Claude Code を落として立ち上げ直してください。

## どれだけ削減できてるか確認する

累積の削減量は `rtk gain` で見られます。

```bash
rtk gain
# Total commands:    142
# Tokens saved:      58,340 (72.4%)
```

使い始めて1週間で数万トークンが浮きます。課金プランで使っている人なら、月数百円〜数千円レベルで差が出ます。

また `rtk discover` を叩くと、**あなたの Claude Code セッションを解析して「RTK をまだ使っていないコマンド」を教えてくれる**機能もあります。これは定期的に回すとよいです。

## NotebookLM と組み合わせると最強

前回紹介した [NotebookLM 連携](/posts/notebooklm-claude-code-research/) が「読解の外注」だとすれば、RTK は「出力の圧縮」。役割が違うので両方入れるのが正解です。

- **重い読み物** → NotebookLM に逃がす
- **コマンド出力** → RTK で圧縮

この2つで、Claude Code のトークン消費は体感半分以下になります。

## まとめ

RTK は **「`rtk` を付けるだけ」で 60〜90% のトークン削減**ができる、非常にコスパのいいツールでした。

- Windows でもバイナリ配置で動く
- `rtk init -g` で Claude が自発的に使ってくれる
- NotebookLM 連携と併用すると破壊力が倍

トークンを気にせず Claude Code を振り回せる環境を作る。長く付き合うなら、絶対入れておいて損はないです。

次回はシリーズの締めとして、Claude Code を2週間使って気づいた「マネジメントが9割」という話を書きます。
