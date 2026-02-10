---
title: "Zenn CLI + CodexでAI駆動執筆はじめました"
emoji: "✨"
type: "idea"
topics: ["zenn", "cli", "codex"]
published: true
---
「Zenn CLI + CodexでAI駆動執筆はじめました」というノリで、no reviewでAIが書いたものをそのまま上げてみるテスト投稿です。とはいえ内容は真面目に、Zenn CLI の導入と最初の一歩をざっくりまとめます。

## Zenn CLIとは
Zenn と GitHub リポジトリを連携すると、ローカルの好きなエディターで記事や本を管理できます。そのときの裏方を担うのがオープンソースの Zenn CLI で、Markdown ファイルの生成やプレビュー、デプロイまでワークフローを整えてくれる相棒です。

## 導入手順
### 1. 事前準備
- Zenn と GitHub を連携して、同期先のリポジトリを用意しておくとスムーズ。
- Node.js は 14 以上が必須。古い環境(v12 など)では動かないので、まずは Node.js を更新しましょう。

### 2. CLI をインストールする
プロジェクトディレクトリで npm を初期化し、Zenn CLI を依存関係に追加します。

```bash
npm init --yes
npm install zenn-cli
```

### 3. Zenn 用セットアップ
続けて初期設定を実行すると、`articles/` や `books/` フォルダ、README などが生成されます。

```bash
npx zenn init
```

ここまででひとまず導入完了。Node v15 系でも動作報告が出ていますし、`npx zenn init` が動かない場合は `npm install zenn-cli` を事前に実行したかを確認しましょう。グローバルインストール(`npm install -g zenn-cli`)でも問題ありません。

## CLI をアップデートする
CLI の表示が zenn.dev と食い違う、あるいはアップデート通知が出た場合は次のコマンドで最新版を取得。

```bash
npm install zenn-cli@latest
```

依存パッケージの警告や `npm audit` のメッセージは、現時点では深刻な問題ではないと公式が案内しています。

## コンテンツを作成・編集する
- 記事: `npx zenn new:article --slug your-slug --title "タイトル"`
- 本: `npx zenn new:book`
- プレビュー: `npx zenn preview` (必要なら `--port 3000` や `--no-watch`)

プレビューはコマンドを実行したターミナルを占有するので、終了させるか別ウィンドウで作業しましょう。リモート環境でプレビューする場合は、ポートフォワードや code-server などの構成に応じて `localhost` へ接続できるように設定が必要です。

## よくあるハマりどころ
- Windows で `npm install zenn-cli` が Git を探せず失敗するケースは、Git for Windows を入れるか、CLI の最新版で解消されていることを確認。
- `npx zenn preview` で `.next` ディレクトリを見つけられないエラーは、`npm install zenn-cli@latest` で修正済み。
- port 表示が `http://localhost:80` のままになる VS Code バグ報告もありましたが、実際は指定ポートで表示されるので気にしなくて OK。

## まとめ
AI が骨子を作り、人間が内容を整える時代になったので、ローカルに Zenn CLI を入れておけばすぐに記事を書いてプレビューして公開できます。デプロイは GitHub に push するだけ。もしスケジュール公開したいなら Front Matter に `published_at` を追加するのもお忘れなく。
