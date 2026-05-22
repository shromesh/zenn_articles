---
title: "Claude CodeでOpus 4.6を使う方法"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "claude", "ai", "anthropic"]
published: true
---

## `/model`コマンドではOpus 4.7しか選べない

Claude Codeで`/model`を実行すると、Opusとして選べるのはOpus 4.7だけです。

```
/model
```

表示される選択肢にOpus 4.6は含まれていません。

## 解決策: モデルIDを直接指定する

`/model`コマンドには、モデルIDを引数として直接渡せます。以下のコマンドを入力してください。

```
/model claude-opus-4-6[1M]
```

実行すると、次のように表示されます。

```
Set model to Opus 4.6 (1M context) for this session
```

これでセッション中はOpus 4.6が使われるようになります。

## 補足: よくある間違い

以下の指定方法ではモデルが見つからずエラーになります。

```
# NG: ハイフン付きフラグとして解釈される
/model --opus4.6

# NG: フラグ形式
/model --claude-opus-4-6

# NG: [1M]サフィックスがない(1MコンテキストバージョンでないOpus 4.6になってしまう)
/model claude-opus-4-6
```

正しいモデルIDは `claude-opus-4-6[1M]` です。
