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

```AGENTS.md
# 行動ルール
1. ユーザーが記事を作成したい、変更したいなどの指示をする場合、以下に従い、それを実行する。
2. 変更内容をユーザーにレビューさせる。「`npx zenn preview --port 8002`を用いてプレビューしてください。」と指示。
3. ユーザーが内容に関して承認したら、記事をデプロイするため、以下のルールに従いadd, commit, pushを行う。

# CLI で記事（article）を管理する
ファイルの配置ルール
1 つの記事の内容は、1 つのmarkdownファイル（◯◯.md）で管理します。ファイルはarticlesという名前のディレクトリ内に含める必要があります。

.
└─ articles
   ├── example-article1.md
   └── example-article2.md

# 記事の作成
以下のコマンドによりmarkdownファイルを簡単に作成できます。

```
$ npx zenn new:article
```

articles/ランダムなslug.mdというファイルが作成されます。slug（スラッグ）はその記事のユニークな ID のようなものです。

作成されたファイルの中身は次のようになっています。

---
title: "" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---
ここから本文を書く

👆 ファイルの上部には---に挟まれる形で記事の設定（Front Matter）が含まれています。ここに記事のタイトル（title）やトピックス（topics）などをyaml 形式で指定することになります。

コマンド実行時に記事の Front Matter をオプションで指定することもできます。

```
$ npx zenn new:article --slug 記事のスラッグ --title タイトル --type idea --emoji ✨
```

slug はa-z0-9、ハイフン-、アンダースコア_の 12〜50 字の組み合わせにする必要があります

本文に画像を挿入するには

# プレビューする
本文の執筆は、ブラウザでプレビューしながら確認できます。ブラウザでプレビューするためには次のコマンドを実行します。

```
$ npx zenn preview
```

このように各記事をプレビューをしながら執筆できます。

デフォルトではlocalhost:8000で立ち上がりますがnpx zenn preview --port 3000というようにポート番号の指定もできます。

npx zenn preview --no-watchのようにすることでファイルの監視と自動リロードが無効になります。

記事を公開する
記事を zenn.dev 上で公開するにはpublishedオプションがtrueになっていることを確認したうえで、ファイルをコミットし、Zenn と連携されている GitHub リポジトリにプッシュします。
Zenn と連携したリポジトリの登録ブランチにプッシュされると、同期（デプロイ）が開始されます。

デプロイ履歴はダッシュボードから見ることができます。デプロイ時にエラーが発生している場合もここから見る必要があります。

なおコミットメッセージに[ci skip]もしくは[skip ci]が含まれていると Zenn でのデプロイがスキップされます。

日時を指定して記事を公開する（公開予約する）
公開時間を指定して記事を公開するには、Front Matterにて published を true にした上で、 published_at を指定します。published_at のフォーマットは、 YYYY-MM-DD または YYYY-MM-DD hh:mm です。日付だけを指定した場合、時刻は 00:00 となります。

published: true # trueを指定する
published_at: 2050-06-12 09:03 # 未来の日時を指定する

この状態で、GitHubリポジトリへプッシュすると、zenn.dev上で記事が公開予約状態となり、公開予約時刻が過ぎると自動的に記事が公開されます。

published_at のタイムゾーンはJST（日本時間）です。

過去の公開日時で記事を公開する
他のブログサービスなどからzenn.devに記事を移行する際に公開日時を維持したい場合、Front Matterにて published_at に過去の日時を指定することで、zenn.dev上での公開日時を指定することができます。published_at のフォーマットは、 YYYY-MM-DD または YYYY-MM-DD hh:mm です。日付だけを指定した場合、時刻は 00:00 となります。

published: true # true/falseどちらでもOKです
published_at: 2010-01-01 08:00 # 過去の日時を指定する

公開日時の指定は一度しかできず、既に設定された値を変更することはできません。

記事の更新
記事の更新を行う場合も、markdownファイルを編集し、GitHub リポジトリへプッシュするだけで OK です。このとき slug が同一のものでないと別の記事として作成されてしまうので注意しましょう。

リポジトリの変更が zenn.dev に反映されるまでにしばらく時間がかかります。ダッシュボードからデプロイのステータスをご確認ください。

また、未ログイン状態ではしばらくキャッシュされた古い内容が表示される可能性があります。時間を置いた後にリロードしてご確認ください。

記事の削除
削除はダッシュボードから行います。安全のため、articlesディレクトリからmarkdownファイルを削除しても zenn.dev 上では削除はされません。
```




## 5. 「新しい記事を書きたい・レビューしたい」など要望を伝える
記事のテーマや書き出し、レビューの有無、公開タイミングなどを具体的に指示すると、AI が文脈を掴んだ上でドラフトを生成してくれます。必要なら `published_at` を指定した予約公開もこの時点で決めておきましょう。

## 6. Codex が執筆し、commit/push まで自動化
指示が揃えば Codex が `npx zenn new:article` でファイルを作り、本文を執筆し、`git commit` と `git push` まで実施。Zenn 側の連携が済んでいれば、そのままデプロイが走り公開完了です。あとは `npx zenn preview` でローカル確認しながら、気になる点があれば追記・修正して再 push するだけ。

---
以上、AI 駆動で Zenn 記事を量産するためのシンプルな 7 ステップでした。これで「書きたい」と思った瞬間に Codex へ投げれば、レビューなしのラフな記事でもサクッと公開できます。
