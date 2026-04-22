---
title: "Claude API で『朝刊サマリー』を自動生成する話"
displayTitle: "Claude API で<br>『朝刊サマリー』を自動生成する話"
date: 2026-04-23T14:00:00+09:00
draft: false
tags: ["Claude API", "Python", "自動化", "プロンプトキャッシング", "Anthropic SDK"]
categories: ["自動化"]
description: "前回作った毎朝のニュース収集パイプラインに Claude API を挟んで、朝起きた時点で3行サマリーが揃っている状態にした話。プロンプトキャッシングでコストを抑える実装も合わせて紹介します。"
---

[前回の記事](/posts/daily-pipeline-task-scheduler/) で、毎朝 Google Sheet にニュース10件を揃えるパイプラインを作りました。数日使ってみて気付いたことがあります。

**タイトルだけ見てても、結局あとで本文を読みに行く羽目になる。**

せっかく自動化したのに、結局リーディングの時間は減っていない。そこで今回は、このパイプラインに Claude API を挟んで「3行サマリーまで揃った状態」で朝を迎える構成にしました。コストを抑えるための **プロンプトキャッシング** も込みで紹介します。

## 構成の変更点

前回の構成から差分だけ。

```
[Python スクリプト]
  ├─ feedparser で RSS 10件取得
  ├─ タイトル・URL・本文を整形
  ├─ ★ Claude API で3行サマリー生成  ← 新規
  └─ requests.post() で Webhook へ
```

`collect()` と `push()` の間に `summarize()` を挟むだけ。既存コードへの変更は最小で済みます。

## Anthropic SDK のセットアップ

```bash
pip install anthropic
```

API キーは [Anthropic Console](https://console.anthropic.com/) で発行して、環境変数に入れておきます。

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxx
```

Windows の Task Scheduler からは `.env` を自動で読んでくれないので、Python 側で `python-dotenv` を使って明示的にロードします。これは前回記事のハマりどころと同じ話です。

```python
from dotenv import load_dotenv
load_dotenv()
```

## 要約関数の実装（プロンプトキャッシング付き）

まず素直な実装から書いて、キャッシング対応に進みます。

```python
import os
from anthropic import Anthropic

client = Anthropic()

SYSTEM_PROMPT = """あなたはセキュリティエンジニア向けのニュース編集者です。
与えられたニュース本文を、以下のルールで3行に要約してください。

# ルール
- 1行目: 誰が／何が起きたか（事実ベース）
- 2行目: なぜ重要か（影響範囲・対象ユーザー）
- 3行目: 読者が取るべきアクション（該当すれば）
- 文末は「〜した」「〜である」調で統一
- 絵文字は使わない
- 固有名詞は略さず正式名称で

# 出力フォーマット例
Apache Struts 2 に遠隔コード実行の脆弱性 CVE-2026-XXXXX が公表された。
CVSS 9.8 で未パッチの本番環境が影響を受ける可能性がある。
最新版 6.8.0 以降へのアップグレードを推奨する。
"""


def summarize(title: str, body: str) -> str:
    resp = client.messages.create(
        model="claude-sonnet-4-7",
        max_tokens=300,
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"},
            }
        ],
        messages=[
            {
                "role": "user",
                "content": f"タイトル: {title}\n\n本文:\n{body[:2000]}",
            }
        ],
    )
    return resp.content[0].text
```

ポイントは `system` を**リスト形式**にして `cache_control` を付けているところ。これで Anthropic 側がこの部分をキャッシュしてくれて、2件目以降の呼び出しは **入力トークンが9割引** になります。

## なぜキャッシング必須か

今回は1回のバッチで10件を連続で処理します。毎回同じシステムプロンプトを送るので、キャッシュしないのは素直に損です。

体感としては、

- **キャッシュなし**: 10件で $0.08 くらい
- **キャッシュあり**: 10件で $0.01 くらい

5分以内の連続呼び出しで Anthropic 側がキャッシュヒットを返してくれる仕様なので、バッチ処理ならほぼ必ずヒットします。毎日動かす自動化で効いてくる差額です。

## 既存パイプラインに組み込む

`collect()` の戻り値を `summarize()` で処理してから `push()` へ流します。

```python
def collect():
    items = []
    for url in FEEDS:
        parsed = feedparser.parse(url)
        for entry in parsed.entries[:5]:
            items.append({
                "title": entry.title,
                "url": entry.link,
                "body": entry.get("summary", ""),  # 本文に変更
            })
    return items[:10]


def enrich(items):
    """各件に summary フィールドを足す"""
    for item in items:
        try:
            item["summary"] = summarize(item["title"], item["body"])
        except Exception as e:
            print(f"summarize failed for {item['url']}: {e}")
            item["summary"] = "(要約の生成に失敗)"
    return items


if __name__ == "__main__":
    items = collect()
    items = enrich(items)
    push(items)
```

Apps Script Webhook 側は前回のまま。渡される JSON に `summary` フィールドが増えただけなので、Sheet の列を1つ足せば終わりです。

## プロンプトを詰めるときのコツ

何度か試して落ち着いたのが以下の4点。

### 1. 出力フォーマット例を必ず入れる

「3行で」とだけ言うと、モデルは箇条書きや長文で返してくることがあります。例を1つ入れるだけで出力が安定します。

### 2. 文末スタイルを明示する

「〜した」調なのか「〜です・ます」調なのか明示しておかないと、記事ごとにブレます。Google Sheet で並べたときに目で読みやすくなるので、統一は大事。

### 3. 絵文字禁止を明示

モデルの現行バージョンは放っておくと絵文字を足してきます。セキュリティニュースに🚨を付けられても困るので、明示的に禁止しておきます。

### 4. 本文は 2000 文字でトリム

RSS によっては本文が丸ごと入っていて数万文字になることがあります。そのままだと入力トークン代が膨らむので、`body[:2000]` でトリム。3行要約の材料としてはこれで十分です。

## ハマりどころ

### レート制限に当たることがある

Tier 1 アカウントだと分間リクエスト数に制限があります。10件程度なら問題ないですが、将来50件100件と増やすなら、`time.sleep(1)` を入れるか [Anthropic Batch API](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing) を使うのが安全です。

### JSON 出力を使うなら `response_format` より prompting

要約を構造化 JSON で受け取りたい場面が将来出てきます。Anthropic は OpenAI のような `response_format` がないので、プロンプトで「JSON で返して」と指示する方が素直。`jsonschema` で検証してリトライ、が鉄板パターンです。

### プロンプトキャッシングの最小長

Sonnet 系だと **1024 トークン以上** ないとキャッシュされません。システムプロンプトが短い場合は、スタイルガイドや few-shot 例を足して長くするほうがコスト的に得、という逆転現象が起きます。

## 次にやるとよいこと

今回の構成で、朝 Sheet を開くと本文を読まなくても要旨がつかめる状態になりました。次にやりたいのは:

- **Slack 通知**: CVSS 9.0 以上の件だけ Slack にプッシュ
- **差分レポート**: 昨日と今日で新規ワードを抽出
- **週次まとめ記事**: 週末にまとめて Markdown 形式で自動生成

自動化は「1本目が動き出すまで」が一番しんどくて、その後は積み増しが楽になります。前回の Task Scheduler × Apps Script の1本目に、Claude API を1段足しただけで、毎朝の情報処理のしんどさがまた一段軽くなりました。

次回は Slack 通知編を書く予定です。
