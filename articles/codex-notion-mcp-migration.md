---
title: "Codex に Notion MCP を追加して、スプシを Notion Table View へ自動移行した話"
emoji: "📋"
type: "tech"
topics: ["codex", "notion", "mcp", "ai"]
published: true
---

Codex（Claude Code）に Notion MCP を接続する手順と、ついでにスプシから Notion への移行を自動でやらせた話です。

## Codex に Notion MCP を追加する

### 1. MCP の設定を追加する

Codex の MCP 設定ファイル（`~/.claude/mcp_servers.json` など）に以下を追加します。

```json
{
  "mcpServers": {
    "notion": {
      "url": "https://mcp.notion.com/mcp"
    }
  }
}
```

追加したら、Codex に「開発タスク WBS を読みに行って」のように Notion を参照する指示を出します。認証が必要と言われたら、次のコマンドを実行してください。

```bash
codex mcp login notion
```

### 2. ブラウザでログインする

コマンドを実行するとブラウザが自動で開くので、Notion アカウントでログインします。

### 3. Codex のセッションを再起動する

認証が通ったら、Codex のセッションを再起動します。

これで Codex から Notion を直接操作できるようになります。

## おまけ：スプシから Notion Table View への自動移行

MCP を入れた直後に早速やらせたのが、スプシで管理していた顧客要望リストの Notion 移行でした。

Biz チームがスプシで顧客の要望をまとめていて、Dev チームはそれを見て各エンジニアにチケットとして振っていたのですが、「振ったかどうか」を毎回スプシとチケット管理を突き合わせて確認するのが面倒でした。Notion に移してしまえば AI が直接参照できるので、確認作業ごと自動化できます。

Codex に出した指示はこれだけです。

> スプシの内容を Notion の 建設業ポータル/Biz 配下に新規ページとして移行して。Table View でカンバンビューも使えるようにして。

Codex は以下の流れで勝手に作業を進めました。

1. **CSV の解析** — スプシからエクスポートした CSV を読み込み、列構造と 111行のデータを把握。ステータス列のユニーク値（長期改善タスク・近日対応予定・実装済みなど）も自動で抽出
2. **親ページの特定** — `notion-search` で「建設業ポータル」→「Bizタスク管理」を探し当てて、配置先を決定
3. **DB 作成** — `notion-create-database` でデータベースを新規作成。ステータス列は `select` 型にして、カンバンでのグループ化に対応
4. **レコード一括投入** — 111行を 25件ずつバッチ分割して `notion-create-pages` で全件投入
5. **ページ本文の差し替え** — DB のインライン表示に置き換え、Table View として閲覧・編集できる状態に

カンバンビューだけは Notion MCP の API に「ビューの新規作成」がまだないため、Notion UI 上で「+ Add a view」→「Board」→「Group by: ステータス」を手動で追加しました。30秒で終わります。

指示を出してから約3分で、111行のスプシが Notion データベースに変わりました。MCP の設定は最初の1回だけなので、以降は Codex に「Notion に〇〇して」と言うだけで済みます。
