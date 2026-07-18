---
title: "Anthropic最新ニュース 7/15 — 先生向けClaude無料開放・Claude Code大型更新・カナダ研究支援"
date: 2026-07-16T07:00:00+09:00
draft: false
tags: ["AIニュース", "Claude", "Anthropic", "Claude Code", "生成AI"]
categories: ["サキのAIニュース"]
cover:
  image: "https://i.ytimg.com/vi/g7xUE9JQ5wk/hqdefault.jpg"
  alt: "Anthropic最新ニュース 7/15 — 先生向けClaude無料開放・Claude Code大型更新・カナダ研究支援"
---

こんにちは、サキです！YouTubeで毎日AIニュースを配信している「サキのAIニュース」の記事版をお届けしますね 😊 今回は7月14日〜15日の新着をまとめて扱います。

今日のポイントは3つです。

- 米国の小中高の先生向けに「Claude for Teachers」を発表。有料級の機能を1年間無料で提供
- Claude Codeが1日で3連続アップデート（2.1.208〜210）。動作が軽くなり、無人運用の安全性・安定性も強化
- カナダのAI研究へ1,000万カナダドルを拠出。法人向けには「Admin API」のユーザー管理機能がベータ提供に

## 動画版（90秒）

{{< youtube g7xUE9JQ5wk >}}

## Claude for Teachers：先生向けに有料級機能を無料開放

Anthropicが、アメリカの小中高（K-12）の先生向けプログラム「Claude for Teachers」を始めました。ふだんは有料で使えるような機能を、認定された先生には無料で開放するという内容です。

授業の準備、生徒のレベルに合わせた教え方の工夫、クラスの成績データの分析、事務作業の自動化などを助ける「教育向けの専用スキル集」が付いてきます。Canva Educationなど9つの教育サービスと連携し、全米50州の学習基準に沿った教材ともつながります。

生徒のデータがAIの学習に使われることはなく、教育プライバシー法（FERPA）にも準拠。料金は無料で、2027年6月30日までに登録した先生が1年間使えます。学校や地区向けの別プランも今後出す予定とのことです。

直接の対象は米国の先生ですが、Anthropicが「教育」という具体的な現場に踏み込んできた点が注目です。専用スキル集を配る形は、今後ほかの職種向けにも広がる可能性があります 🌱

## Claude Code：1日で3連続アップデート（2.1.208〜210）

7月14日にClaude Codeが3連続でアップデートされました（2.1.208 / 209 / 210、いずれも同日公開）。今回は大きめのメンテナンス回です。

とくに2.1.208はメモリ節約と速度改善が中心で、長時間のセッションが軽く動くようになりました（セッション記録のサイズ縮小やメモリリーク修正など）。目が不自由な方向けに、読み上げに適したプレーン表示の「スクリーンリーダー対応モード」も新設されました。2.1.210では、時間のかかるツールに「経過時間カウンター」が出るようになり、止まって見えて不安になる場面が減りました。

安全面も強化されています。サブエージェントが読み込んだ内容を通じた「間接的な指示のっとり（プロンプトインジェクション）」への対策が強くなり、「rm -rf ~」のような危険なコマンドは自動許可モードでも確認を出すよう直されました。無人・バックグラウンド運用まわりのバグも多数修正され、返信の取りこぼし防止やセッション再開時の誤表示修正、クラッシュ対策などが入っています。2.1.209はバックグラウンドセッションでモデル選択などのダイアログが開けない不具合の修正のみです。

夜間バッチや日次パイプラインを無人で回している人には、今回の「メモリ・速度改善」と「無人運用の安全性・安定性強化」はそのまま実利になります。とくに危険コマンドの確認強化と間接インジェクション対策は、自動実行を任せるうえで安心材料です。

## カナダのAI研究へ1,000万カナダドルを拠出

Anthropicが、AIの安全で良い使い方を探る研究のために、カナダの研究機関へ総額1,000万カナダドル分を拠出します。提供先はAmii（エドモントン）、Mila（モントリオール）、Vector Institute（トロント）など計8機関で、各機関にClaudeの利用クレジットを提供します（機関ごとの配分額は公表されていません）。研究の方向性や結論には口を出さない方針です。

これらの機関はスタートアップ支援枠にも加わり、関連する数百のカナダ系スタートアップにも、各5,000米ドル以上のAPIクレジットが提供されます。

## 法人向け「Admin API」ユーザー管理機能がベータ提供

会社向けプラン（Claude Enterprise）で、利用者の管理を管理用API経由で行えるようになりました。全Enterprise組織でベータ提供です。メンバー一覧の取得やメール検索、役割の変更、削除、招待の送信・取り消し、グループ管理などができます。

一般の利用者に直接の影響はなく、社内に多数のアカウントを抱える管理者が運用を効率化するための機能です。

## 稼働状況

対象期間に長時間の重大障害はありませんでしたが、短時間で復旧した障害が2件ありました。7月14日にclaude.aiのコンテナ作成で一部障害があり、Claude Codeなどに影響したものの約45分で復旧。7月13日にはClaude Haiku 4.5でエラーが増えましたが、約40分で復旧しています。いずれも解決済みです。

## 出典（一次ソース）

- Claude for Teachers: [https://www.anthropic.com/news/claude-for-teachers](https://www.anthropic.com/news/claude-for-teachers)
- カナダAI研究への拠出: [https://www.anthropic.com/news/canadian-ai-research](https://www.anthropic.com/news/canadian-ai-research)
- Claude Code CHANGELOG: [https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- Claude Platform リリースノート: [https://platform.claude.com/docs/en/release-notes/overview](https://platform.claude.com/docs/en/release-notes/overview)
- 稼働状況: [https://status.claude.com](https://status.claude.com)

※動画のナレーションは音声合成（Irodori-TTS）で生成しています。

![最後まで読んでくださりありがとうございます。よろしかったら、いいね・レビュー・ブックマークをお願いします！](https://yakiimo-tech.com/n/common/article-cta.jpg)
