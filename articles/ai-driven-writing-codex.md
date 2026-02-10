---
title: "Zenn CLI + CodexでAI駆動執筆はじめました"
emoji: "✨"
type: "idea"
topics: ["zenn", "cli", "codex"]
published: true
---
「Zenn CLI + CodexでAI駆動執筆はじめました」というノリで、no reviewでAIが書いたものをそのまま上げてみるテスト投稿です。ここからは、AIと人が二人三脚で Zenn 記事を量産するための流れを 6 ステップでまとめます。

## 1. Zenn CLI をインストールする
Node.js 14+ を入れたら、リポジトリ直下で npm を初期化し Zenn CLI を導入します。

```bash
npm init --yes
npm install zenn-cli
npx zenn init
```

`init` で `articles/` `books/` などが揃い、準備が完了します。アップデートや不具合対応が入ったら `npm install zenn-cli@latest` で追従しましょう。

## 2. Zenn と GitHub アカウントを連携する
Zenn のダッシュボードで GitHub リポジトリを紐付けると、push したタイミングで自動配信されます。進捗やエラーは `https://zenn.dev/dashboard/deploys` から確認できるので、公開後のモニタリングにも便利です。

## 3. 既存記事をエクスポートしてローカルに置く
ダッシュボードから既存記事をエクスポートし、`articles/slug.md` として保存すれば、過去記事もローカル編集できます。Front Matter の `slug` がズレないよう注意しつつ、Zenn CLI で `npx zenn preview` を走らせれば差分確認もラクです。

## 4. AGENTS.md を用意する
今回のように Codex に執筆を任せるなら、プロジェクトのルールやワークフローを `AGENTS.md` にまとめておくのが吉。記事の配置ルールや公開方法、プレビュー手順などを共有しておけば、AI エージェントが迷わず作業できます。

## 5. 「新しい記事を書きたい・レビューしたい」など要望を伝える
記事のテーマや書き出し、レビューの有無、公開タイミングなどを具体的に指示すると、AI が文脈を掴んだ上でドラフトを生成してくれます。必要なら `published_at` を指定した予約公開もこの時点で決めておきましょう。

## 6. Codex が執筆し、commit/push まで自動化
指示が揃えば Codex が `npx zenn new:article` でファイルを作り、本文を執筆し、`git commit` と `git push` まで実施。Zenn 側の連携が済んでいれば、そのままデプロイが走り公開完了です。あとは `npx zenn preview` でローカル確認しながら、気になる点があれば追記・修正して再 push するだけ。

---
以上、AI 駆動で Zenn 記事を量産するためのシンプルな 6 ステップでした。これで「書きたい」と思った瞬間に Codex へ投げれば、レビューなしのラフな記事でもサクッと公開できます。
