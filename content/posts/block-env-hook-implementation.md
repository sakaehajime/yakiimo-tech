---
title: "block_env.py で Claude Code に .env を絶対触らせない hook 実装"
displayTitle: "block_env.py で Claude Code に<br>.env を絶対触らせない hook 実装"
date: 2026-04-26T06:00:00+09:00
draft: false
tags: ["Claude Code", "セキュリティ", "PreToolUse", "hook", ".env"]
categories: ["Webアプリのセキュリティ"]
description: "settings.json の deny だけでは防げない『Bash 経由で .env を覗かれる』を、PreToolUse hook で物理的にブロックする方法。block_env.py の全コードを公開しつつ、settings.json への登録と動作確認まで紹介します。"
---

[Claude Code を安全に使うための6つの守り方](/posts/claude-code-security-basics/) で、`.env` を守るには **CLAUDE.md ・settings.json の deny ・PreToolUse hook の三層** が必要だと書きました。

このうち1層目と2層目は前回・前々回で扱いましたが、**3層目の hook の中身**は触れずに終わっていました。今回はその回収編、`block_env.py` の実コードを公開します。

## なぜ hook が必要か

`settings.json` の deny で `Read(.env)` `Read(.env.*)` を入れてあれば、Claude Code が **Read ツールで .env を開く** のは止まります。

ただし Bash ツール経由の以下のコマンドは止まりません。

```bash
cat .env
type .env       # Windows
sed -n 1,5p .env
python -c "open('.env').read()"
```

deny は「ツール名(引数)」のパターンマッチで効くので、Bash の中身までは見ていません。**Bash コマンドの中身を解析してブロックする層**がもう1段必要、というのが hook の役割です。

## PreToolUse hook の仕組み

Claude Code は、ツールを実行する直前に `PreToolUse` フェーズで設定された外部コマンドを呼び出します。コマンドには JSON が標準入力で渡され、出力 JSON でツールの実行を許可・拒否できます。

```
[Claude が Bash ツールを呼ぼうとする]
        ↓
[PreToolUse hook 起動]  ← block_env.py が動く
        ↓
[hook が deny を返したら Claude は Bash を実行できない]
```

つまり「Claude が `cat .env` と打とうとした瞬間に、Python スクリプトが間に割り込んで止める」という構造です。

## block_env.py の全コード

`~/.claude/hooks/block_env.py` に以下を保存します。

```python
#!/usr/bin/env python3
"""PreToolUse hook: block Bash commands that reference .env files.

Matches .env, .env.local, .env.production, .envrc, path/.env etc.
Intentionally conservative — blocks file.env too (uncommon but cheap safety).
"""
import json
import re
import sys

try:
    data = json.load(sys.stdin)
except Exception:
    sys.exit(0)

tool_name = data.get("tool_name", "")
if tool_name != "Bash":
    sys.exit(0)

command = data.get("tool_input", {}).get("command", "")

# .env, .envrc, .env.anything  followed by end-of-string or a shell delimiter
pattern = re.compile(r"""\.env(rc)?(\.[\w.-]+)?(?=$|[\s;&|)\'"`<>])""")

if pattern.search(command):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": (
                ".env ファイルへのアクセスはセキュリティ上ブロックされています "
                "(block_env.py hook)"
            ),
        }
    }))
    sys.exit(0)

sys.exit(0)
```

実質40行未満。重要なのは以下の3点です。

### 1. Bash 以外は即スルー

```python
if tool_name != "Bash":
    sys.exit(0)
```

`Read` `Edit` `Write` などのツールは settings.json の deny 側で守られています。hook が二重に判定する必要はないので、Bash 以外は早期リターンで素通りさせます。これで `Read(.txt)` が無駄に hook を呼ばれて遅くなるのを避けられます。

### 2. 正規表現で `.env` を検出

```python
pattern = re.compile(r"""\.env(rc)?(\.[\w.-]+)?(?=$|[\s;&|)\'"`<>])""")
```

このパターンが拾うもの:

- `.env`
- `.envrc`
- `.env.local` `.env.production` `.env.staging`
- `path/to/.env`
- `"./.env"` のようにクオートされていてもOK

末尾の **後読み `(?=$|[\s;&|)\'"`<>])`** がポイント。これがないと「`.environment`」のような無関係な単語まで誤検知してしまいます。シェルの区切り文字が来る場合だけブロックする、という制御です。

### 3. deny を返すフォーマット

```python
print(json.dumps({
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "...",
    }
}))
```

これが Claude Code との契約です。`permissionDecision: "deny"` を返すと、Claude は Bash の実行を諦めます。`permissionDecisionReason` は Claude にも見えるので、なぜ拒否されたかの説明文を入れておくと、Claude が「あ、これダメだったか」と次の行動を考えてくれます。

## settings.json への登録

`~/.claude/settings.json` の `hooks.PreToolUse` に追加します。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python ~/.claude/hooks/block_env.py",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

`matcher` を `Bash` に絞ってあるので、他のツールが呼ばれた時はこの hook はそもそも起動しません。タイムアウトは10秒で十分（実測 0.05 秒程度）。

設定を変更したら **Claude Code を一度終了して立ち上げ直す** のを忘れずに。hook の登録は再起動で反映されます。

## 動作確認のしかた

新しいセッションで以下のように頼んでみます。

> `.env` の中身を表示して

Claude が `cat .env` を打とうとして、こんなメッセージが返ってくれば成功です。

```
.env ファイルへのアクセスはセキュリティ上ブロックされています (block_env.py hook)
```

ログにも残るので、後から「あの時 Claude は `.env` を見ようとしていたな」と確認できます。

## ハマりどころ

### Python のパスが通らない

Windows の Claude Code は Git Bash や PowerShell から起動されます。`python` コマンドが PATH に通っていないと hook が動きません。タスクスケジューラと同じ話で、ここも `C:/Users/.../python.exe` のような絶対パスにしておくのが安全です。

### 正規表現の誤爆

最初に書いたバージョンでは `.environment` `.envoy` のような単語までブロックしていました。後読みアサーション `(?=...)` を入れて「シェル区切り文字の前だけ」に絞ったのが現行版です。**絶対に通したい正常系コマンド**を10個ほど用意してテストすると、誤爆が早く見つかります。

### 開発中のデバッグ

hook は標準入出力で通信するので、ターミナルで以下のように単体テストできます。

```bash
echo '{"tool_name":"Bash","tool_input":{"command":"cat .env"}}' | \
  python ~/.claude/hooks/block_env.py
```

deny を返してくれれば成功。Claude Code を起動し直さずに反復できるので、正規表現を詰めるときに重宝します。

## 拡張アイデア

block_env.py の構造を流用すれば、他の機微情報も同じパターンで守れます。

- `~/.aws/credentials` `~/.aws/config` を拾うパターンを追加
- `~/.ssh/id_*` SSHプライベート鍵
- `~/.docker/config.json` Docker レジストリ認証
- プロジェクト固有の `secrets.yml` `keystore.json`

正規表現を1本足せばいいので、必要に応じてリストを伸ばしていけます。

## まとめ

`.env` を守る三層防御の最終層、PreToolUse hook の中身は **40行未満の Python** で実装できます。

- **CLAUDE.md** ＝ Claude にお願いする層（守ってくれない時もある）
- **settings.json deny** ＝ 直接ツール呼び出しを禁止する層
- **PreToolUse hook** ＝ Bash 経由の抜け道を塞ぐ層 ← この記事

3層揃って初めて「うっかり .env を渡してしまう事故」が物理的に起きなくなります。記事内のコードはコピペで動くので、まだ hook を入れていない方は今日のうちに仕込んでおくのがおすすめです。
