---
title: "Zenn CLI + CodexでAI駆動執筆はじめました"
emoji: "✨"
type: "idea"
topics: ["zenn", "cli", "codex"]
published: true
---
「Zenn CLI + CodexでAI駆動執筆はじめました」というタイトルで、Zenn CLI + Codexを使って執筆するテスト投稿です。公式の記事ではcliのインストール、github連携、過去記事のローカルへの同期が別の記事になっていないので、この記事では1つの記事だけで完結することを意識して書きました。リポジトリ作成から公開までを 0〜6 の 7 ステップでまとめています。

## 0. リポジトリを作成して clone する
まずは GitHub 上に Zenn 用のリポジトリ（例: `zenn_articles`）を作成し、ローカルに clone します。Organization / 個人どちらでもOKです。

```bash
git clone git@github.com:your-account/zenn_articles.git
cd zenn_articles
```

このディレクトリ配下で以降の作業を進めます。

## 1. Zenn CLI をインストールする
リポジトリ直下で npm を初期化し Zenn CLI を導入します。

```bash
npm init --yes
npm install zenn-cli
npx zenn init
```

`init` で `articles/` `books/` などが揃い、準備が完了します。アップデートや不具合対応が入ったら `npm install zenn-cli@latest` で追従しましょう。

## 2. Zenn と GitHub アカウントを連携する
Zenn のダッシュボードで [GitHub リポジトリを紐付け](https://zenn.dev/dashboard/deploys)ます。

## 3. 既存記事をエクスポートしてローカルに置く
ダッシュボードから[既存記事をエクスポート](https://zenn.dev/settings/export)し、`articles/slug.md` として保存すれば、過去記事もローカル編集できます。Front Matter の `slug` がズレないよう注意しつつ、Zenn CLI で `npx zenn preview` を走らせれば差分確認もラクです。

## 4. AGENTS.md を用意する
今回のように Codex に執筆を任せるなら、プロジェクトのルールやワークフローを `AGENTS.md` にまとめておくのが吉。記事の配置ルールや公開方法、プレビュー手順などを共有しておけば、AI エージェントが迷わず作業できます。以下は今回貼り付けた実例です。

````AGENTS.md
````




## 5. 「新しい記事を書きたい・レビューしたい」など要望を伝える
記事のテーマや書き出し、レビューの有無、公開タイミングなどを具体的に指示すると、AI が文脈を掴んだ上でドラフトを生成してくれます。必要なら `published_at` を指定した予約公開もこの時点で決めておきましょう。

## 6. Codex が執筆し、commit/push まで自動化
指示が揃えば Codex が `npx zenn new:article` でファイルを作り、本文を執筆し、`git commit` と `git push` まで実施。Zenn 側の連携が済んでいれば、そのままデプロイが走り公開完了です。あとは `npx zenn preview` でローカル確認しながら、気になる点があれば追記・修正して再 push するだけ。

---
以上、AI 駆動で Zenn 記事を量産するためのシンプルな 7 ステップでした。これで「書きたい」と思った瞬間に Codex へ投げれば、レビューなしのラフな記事でもサクッと公開できます。
