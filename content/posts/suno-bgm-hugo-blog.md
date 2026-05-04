---
title: "Suno でオリジナル BGM を作って個人ブログに流し続ける話"
displayTitle: "Suno でオリジナル BGM を作って<br>個人ブログに流し続ける話"
date: 2026-05-04T05:00:00+09:00
draft: false
tags: ["Suno", "BGM", "Hugo", "個人ブログ", "AI音楽生成"]
categories: ["自動化"]
description: "Suno で生成した AI 音楽を Hugo + Cloudflare Pages の個人ブログに常時BGMとして流す実装記録。ffmpeg での opus/mp3 同時配信、全ページ通しでの連続再生、ハマりどころまで実体験ベースでまとめます。"
---

やきいもテックを開設した頃、トップページに**雰囲気作りのBGM**を置いていました。最初はフリー素材だったのですが、しばらくして「**自分のサイトの世界観に合った曲が欲しい**」と思うように。

そこで使ってみたのが **Suno**。生成した曲を Hugo サイトに組み込んで、全ページで途切れずに流せるようにしました。今回はその実装記録です。

## なぜ「サイトに合うBGM」が欲しいか

ブログの読者体験は、文字情報だけで作られているようで、**周辺の小さな要素**にも影響を受けます。

- フォント
- 配色
- スクロールの感触
- 余白の取り方
- そして、BGM

特に「読み物」として長く滞在してほしいサイトでは、空間の質感を作るBGMの効果が地味に効きます。とはいえ既存のフリー素材は、どこかで聞いた曲だったり、サイトのテーマに微妙に合わなかったり。

そこで「**Suno で自分のサイト専用に作る**」というアプローチに切り替えました。

## Suno でBGMを生成する

Suno は AI 音楽生成サービスで、テキストプロンプトから曲を作れます。月額のサブスク制で、1曲あたりの生成は実質的に無制限。

私が指定した内容はざっくりこんな感じ:

```
Style: ambient guitar, lo-fi, soft, instrumental, pomodoro focus
Tempo: slow, around 60-70 BPM
Mood: calm, contemplative, like working in a quiet cafe
Length: 3-4 minutes, with a subtle ending
```

サイトの読み物体験に合わせて、**集中を妨げない・歌詞なし・繰り返しでも飽きない**を条件にしました。何度か生成してから、気に入った1曲を採用。今は「アンビエントギター」系の3分23秒の曲が流れています。

## Hugo に組み込む

### ① 音声ファイルの最適化

Suno からダウンロードした mp3 をそのまま使ってもいいのですが、サイトに置くなら**サイズを軽く**しておきたい。ffmpeg で opus 形式にも変換します。

```bash
ffmpeg -y -i ambient.mp3 -c:a libopus -b:a 96k ambient.opus
```

| 形式 | サイズ | 対応ブラウザ |
|------|--------|--------------|
| mp3 | 4.9 MB | 全ブラウザ |
| opus | 2.8 MB | モダンブラウザ（Chrome/Firefox/Edge等） |

opus を優先で配信して、対応していないブラウザには mp3 を返す、という二段構えにします。

### ② Hugo の partial を作る

`layouts/partials/yt-bgm.html` に、`<audio>` タグを置く partial を作成。

```html
<audio id="yt-bgm-audio" loop preload="auto">
  <source src="{{ "audio/ambient.opus" | relURL }}" type="audio/ogg; codecs=opus">
  <source src="{{ "audio/ambient.mp3" | relURL }}" type="audio/mpeg">
</audio>
<button id="yt-bgm-toggle">♪ BGM</button>
```

`loop` 属性で同じ曲を繰り返します。トグルボタンで再生・停止を切り替え可能に。

### ③ 全ページに流し続ける仕組み

普通に `<audio>` を置くと、**ページ遷移のたびに再生位置がリセット**されてしまいます。記事を読んでスクロールしてリンクをクリック → 別ページに飛ぶたびに頭から流れ直すと、雰囲気どころか気が散ります。

そこで `sessionStorage` を使って再生位置を記憶。

```javascript
const audio = document.getElementById("yt-bgm-audio");
const KEY = "yt-bgm-state";

// ページ読み込み時、前回の状態を復元
window.addEventListener("DOMContentLoaded", () => {
  const saved = JSON.parse(sessionStorage.getItem(KEY) || "{}");
  if (saved.currentTime) audio.currentTime = saved.currentTime;
  if (saved.playing) audio.play();
});

// ページ離脱時に状態を保存
window.addEventListener("beforeunload", () => {
  sessionStorage.setItem(KEY, JSON.stringify({
    currentTime: audio.currentTime,
    playing: !audio.paused,
  }));
});
```

これで「曲がブラウザのタブを開いている間、続いて流れている」感覚を作れます。

## ハマりどころ

### 1. ブラウザの自動再生規制

最近のブラウザは、**ユーザー操作なしの自動再生をブロック**します。BGM を流すには「再生ボタンを最初に押してもらう」UI が必要。私のサイトでは小さなトグルボタンを画面右下に常時表示しています。

### 2. ホームページだけにBGMを置くとPVカウント問題

最初、Hugo の `IsHome` 判定でホームページだけに `<audio>` を出していましたが、別ページに飛ぶと音が止まる。**全ページに同じ partial を埋め込み**＋**`sessionStorage`で位置記憶**にしてから、体感が一気に良くなりました。

### 3. opus が再生されない一部環境

Safari でテストしたら opus が読まれないケースがありました。`<source>` を opus → mp3 の順で書いてフォールバックさせれば全ブラウザで動きます。順序が逆だと mp3 が優先されて opus のサイズメリットを取り逃すので注意。

### 4. ループの繋ぎ目

`loop` 属性で繋ぐと、曲によっては**繋ぎ目で僅かに無音**が入ります。Suno で生成する時に「subtle ending」と指示しておくと、終端が滑らかになって気になりにくいです。

## まとめ

- **AI で自分のサイト用のオリジナルBGMが作れる時代**
- Hugo なら partial と JavaScript 50 行で全ページ常時再生が実装できる
- ffmpeg で opus 変換すると配信サイズが半分弱に
- `sessionStorage` でページ遷移を跨いだ再生位置記憶を実装

「自分のサイトの世界観」を音まで含めて作る、というのは個人ブログならではの楽しみです。AI のおかげで、こういう細部のクラフト感を出すコストが大幅に下がりました。

ブログを始めて1ヶ月くらい経って「もう少し自分の色を出したい」と思った頃に、試してみるとちょうどいい温度感かもしれません。

次回は、生成した BGM を Cloudflare R2 や Workers Cache でさらに高速配信する話を書く予定です。
