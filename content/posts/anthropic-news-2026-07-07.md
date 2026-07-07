---
title: "Anthropicニュースまとめ 2026/7/7 — Claude Code開発秘話と政府の4.66億行監査事例"
date: 2026-07-07T23:50:00+09:00
draft: true
tags: ["Anthropic", "Claude", "Claude Code", "AIニュース"]
categories: ["AIニュース"]
---

Anthropic公式の発信を毎日チェックしてお届けする「サキのAIニュース」、ブログ詳細版です。今日のポイントは3つ。①Claude Codeの開発秘話を描いた特集が公開 ②カナダの州政府がClaudeで4.66億行のコードを20時間で監査した事例 ③Claude Code v2.1.202リリース、です。

## 特集「The Making of Claude Code」

Claude Codeが社内の実験ツールからAnthropicを代表するコーディングエージェントに育つまでの舞台裏を、開発に関わった研究者・エンジニア・初期ユーザーの証言で描いた読み物特集です。ページに「ターミナル風表示で読む」「普通の記事として読む」の2つのモードが用意されている遊び心のある作りで、Claude Codeユーザーなら開発の裏話としてかなり楽しめます。 😊

出典: [The Making of Claude Code](https://www.anthropic.com/features/making-of-claude-code)

## アルバータ州政府がClaudeで政府システムを一斉監査

個人的に今日いちばん驚いた事例です。カナダ・アルバータ州の技術革新省が、27省庁にまたがる政府システムのセキュリティ点検にClaudeを使いました。

- Claude Code上で約50体の自律エージェントを並列で動かし、4億6,600万行のコードを約20時間で分析（従来手法なら約6.5年相当）
- 見つかった脆弱性には、修正コードの生成・テスト作成・古いシステムの現代的な言語での作り直しまで実施
- 95のセキュリティ基準でアプリを継続監視する専用エージェントも構築済み

先週のClaude Codeアップデートで「サブエージェントのバックグラウンド実行が標準化」されましたが、その並列化の方向性が政府規模で実運用されている実例です。個人の自動化でも「1体のAIに長くやらせる」より「複数のAIに分担させる」設計が主流になっていくのを感じます。 😎

出典: [アルバータ州政府の事例](https://www.anthropic.com/news/alberta-government-claude-cybersecurity)

## Claude Code v2.1.202

- 新機能: /config に「Dynamic workflow size」設定が追加。Claudeが自動生成するワークフロー（複数エージェントでの作業計画）の規模を、ユーザー側で調整できるようになりました
- 挙動変更: /review は高速なシングルパスレビューに戻り、複数エージェントでの深いレビューは /code-review に分離。「軽く見てほしい時」と「徹底的に見てほしい時」でコマンドを使い分ける形です
- 修正多数: 履歴検索のクラッシュ、モバイル/Web版リモートからのコマンド失敗、キャプションなし画像が無言で捨てられる問題など

出典: [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

## 稼働状況

7/6〜7/7に短時間の障害が3件（複数モデル・Claude.ai・Sonnet 5のエラー増加。Claude.aiの件ではClaude Codeのログイン失敗も発生）。いずれも1時間半以内に解決済みで、現在は全システム正常です。

## 今日のアクション

/review と /code-review の使い分けは今日から意識する価値があります。私は「日常のPRは/review、リリース前の重要な変更だけ/code-review」という運用にしました。 😌

この内容は60〜90秒のショート動画でも毎日配信しています。チャンネル「AI Times -サキ-」もよろしくお願いします。

![最後まで読んでくださりありがとうございます。よろしかったら、いいね / レビュー / ブックマークをお願いします！](https://yakiimo-tech.com/n/common/article-cta.jpg)
