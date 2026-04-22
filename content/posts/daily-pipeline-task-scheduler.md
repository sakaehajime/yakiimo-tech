---
title: "タスクスケジューラ × Apps Script で『毎朝データが揃ってる』自動化を作った話"
displayTitle: "タスクスケジューラ × Apps Script で<br>『毎朝データが揃ってる』自動化を作った話"
date: 2026-04-20T14:00:00+09:00
draft: false
tags: ["自動化", "Python", "Apps Script", "Task Scheduler", "API連携"]
categories: ["自動化"]
description: "Windows タスクスケジューラ + Python + Apps Script Webhook の3点セットで、朝起きるとデータが揃っている自動化パイプラインを作った記録。最小構成のコードとハマりどころまで。"
---

毎朝 PC を開くと、今日チェックすべき Web セキュリティニュース10件が Google Sheet にすでに並んでいる。そんな状態をローカル PC だけで作った記録です。

構成はとてもシンプル。

- **Python スクリプト**: ニュースを拾って整形する人
- **Apps Script Webhook**: Google Sheet への追記口
- **Windows タスクスケジューラ**: 毎日 00:00 に発火する人

この3つを繋ぐだけで、朝起きるまでに「今日の情報収集」が終わっている仕組みができます。

## なぜ Apps Script 単体にしなかったか

最初は「Google Apps Script で全部書けばいいのでは？」と考えていました。ただ試してみるとしんどい場面が多く、方針を変えました。

- 外部サイトのスクレイピングは GAS だと遅い
- Python の `feedparser` / `BeautifulSoup` のほうが素直に書ける
- デバッグは手元のターミナルが楽

そこで役割分担することにしました。

- **重い処理（データ取得・整形）はローカル Python で**
- **永続化（Sheet 追記）だけ GAS に Webhook として任せる**

結果、疎結合で保守しやすいパイプラインになりました。片方が壊れてももう片方は動くのが、地味に効きます。

## 全体像

```
[Task Scheduler]  毎日 00:00
        │
        ▼
[Python スクリプト]
  ├─ feedparser で RSS 10件取得
  ├─ タイトル・URL・要約を整形
  └─ requests.post() で Webhook へ
        │
        ▼
[Apps Script Webhook]
  └─ Sheet の末尾に append
        │
        ▼
[Google Sheet]  朝見るだけでOK
```

## ステップ1: Apps Script Webhook を立てる

Google Sheet を新規作成して、拡張機能 → Apps Script を開きます。以下を貼ります。

```javascript
function doPost(e) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('news');
  const data = JSON.parse(e.postData.contents);

  data.items.forEach(item => {
    sheet.appendRow([
      new Date(),
      item.title,
      item.url,
      item.summary,
    ]);
  });

  return ContentService
    .createTextOutput(JSON.stringify({ ok: true, count: data.items.length }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

右上の「デプロイ」→「新しいデプロイ」→「ウェブアプリ」を選択。アクセス権を「全員」に設定すると、発行される URL が Webhook エンドポイントになります。

この URL は漏らさないこと。知っている人なら誰でも Sheet に書き込めてしまいます。

## ステップ2: Python で収集＆送信

```python
import feedparser
import requests

WEBHOOK_URL = "https://script.google.com/macros/s/XXX/exec"
FEEDS = [
    "https://example.com/security.rss",
    "https://another.com/feed",
]

def collect():
    items = []
    for url in FEEDS:
        parsed = feedparser.parse(url)
        for entry in parsed.entries[:5]:
            items.append({
                "title": entry.title,
                "url": entry.link,
                "summary": entry.get("summary", "")[:200],
            })
    return items[:10]

def push(items):
    r = requests.post(WEBHOOK_URL, json={"items": items}, timeout=30)
    r.raise_for_status()
    print(r.json())

if __name__ == "__main__":
    push(collect())
```

ターミナルで `python daily_news.py` を叩いて、Sheet に行が増えていれば成功です。

## ステップ3: Task Scheduler に登録

Windows で「タスクスケジューラ」を起動して、新規タスクを作ります。

- **名前**: `daily-news-pipeline`
- **トリガー**: 毎日 00:00
- **操作**: プログラムの開始
  - プログラム: `C:\Users\sakae\AppData\Local\Programs\Python\Python311\python.exe`
  - 引数: `C:\Users\sakae\scripts\daily_news.py`
  - 開始ディレクトリ: `C:\Users\sakae\scripts\`

「最上位の特権で実行」と「ユーザーがログオンしているかどうかにかかわらず実行」の2つをチェックしておくと、寝ている間でも動きます。

## ハマりどころ3つ

### 1. Webhook が 200 を返しても失敗していることがある

Apps Script はエラー時にも 200 で帰ってくる場合があるので、Python 側で `print(r.json())` を残して、翌朝に結果を確認できるようにしておくのが安全です。タスクスケジューラなら「操作」→「ログ取得を追加」で標準出力をファイルに流せます。

### 2. 相対パスは避ける

Task Scheduler が起動するプロセスは、作業ディレクトリが手元のターミナルと違います。スクリプト内のファイル参照は絶対パスにしておくと、夜中にコケて悲しむ確率が下がります。

### 3. 環境変数は明示的に読む

Task Scheduler のプロセスからはシェルの環境変数が見えません。`.env` ファイルは `python-dotenv` で読み込むようにして、API キー類は直書きせず `os.environ.get()` で取るのが安全です。

## 発展させるなら

この最小構成が動き出すと、乗せたくなる仕組みが増えてきます。

- **Claude API でサマリー生成**: ニュースタイトルに3行要約を付けて追記
- **Slack 通知**: 重大インシデントだけはプッシュする
- **週次レポート**: 蓄積したデータを週末に自動集計

「定時に動くパイプライン」が1本あるだけで、朝の情報収集のしんどさがかなり変わります。自動化の第一歩としておすすめです。

次回は、このパイプラインに Claude API を挟んで「朝刊サマリー」を自動生成する話を書きます。
