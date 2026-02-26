---
title: "Qwen-VLモデル徹底解剖：LoRAチューニングによる性能向上の道筋を探る"
emoji: "🔍"
type: "tech"
topics: ["qwen", "vit", "lora", "vlm", "machinelearning"]
published: true
---

## 0. はじめに：モチベーションとお断り

このドキュメントは、Alibabaが開発した高性能なマルチモーダルモデル「Qwen-VL」について、その仕組みを深く理解し、最終的にはLoRA（Low-Rank Adaptation）を用いたファインチューニングで性能をさらに引き上げることを目指す、個人的な学習の記録です。

:::message alert
**本ドキュメントを読む上での注意点**

- **情報の鮮度について：** 本記事は約半年前（2025年夏頃）に調査・作成したメモをベースにしています。マルチモーダルAIの分野は進歩が非常に速いため、最新のモデルやベンチマーク結果と異なる部分がある可能性があります。ご了承ください。
- **最終目標：** 私が個人的にやりたいのは、**LoRAでQwenの画像認識性能を上げること**です。特に、LoRAの教師データをQwenモデルの特性に合わせて最適化することで、より効率的かつ効果的な精度向上を実現することが目的です。
- **発表形式の試み：** こういった形式で知識をまとめて発表するのは初めての試みです。そのため、情報の粒度や説明の分かりやすさなど、至らない点があるかもしれません。今回の内容は入門編と位置づけ、皆さんからのフィードバックを元に次回以降の内容を改善していきたいと考えています。
- **網羅性について：** 参照している各論文について、全てを網羅的に解説するわけではありません。あくまで私が「面白い」「重要だ」と感じた部分を重点的に紹介しています。より深い理解のためには、ぜひ元論文をご参照ください。
- **理論と実装のバランス：** 論文を読んで理論を学ぶことは非常に楽しいですが、それだけでは不十分です。実際にモデルを動かし、学習データを自分の目で確かめ、試行錯誤する「実装面」の重要性を、皆さんと共有したいと考えています。**理論の勉強も楽しいけど、むしろデータを眺めるなど実装面をみなさん忘れがちじゃないか？** という警鐘を鳴らしたいです。
- **数式は少なめに：** このドキュメントでは、複雑な数式は極力避け、Qwenがどのような技術要素で構成されているのか、その「コンセプト」と「アーキテクチャ」の理解に集中します。
- **期待値調整：** 初めての勉強会資料作成ということもあり、まずは内容を薄めに全体をさらっています。この内容でどれくらいのボリュームになるか、どこに疑問が生まれるかは未知数です。ぜひ気軽に質問を投げかけてください。それが次回のテーマになります。
:::

## 1. ウォームアップ：基礎技術の復習

Qwen-VLのような最新モデルを理解するためには、その基盤となっている技術についての知識が不可欠です。ここでは、特に重要な画像認識モデルの進化について簡単に振り返ります。（Transformerの核心技術であるMulti-Head AttentionやRoPEについては、次回以降で詳しく掘り下げる予定です。）

### 画像認識モデルの進化：Faster R-CNNとYOLO

かつての画像認識、特に物体検出の分野では、「どこに」「何があるか」を特定するために複雑なパイプラインが必要でした。その中で代表的な2つのアプローチがFaster R-CNNとYOLOです。

#### Faster R-CNN: 精度重視の二段階検出

Faster R-CNNは、「領域提案」と「分類」を2つのステップに分けて行うことで、高い検出精度を実現しました。

1. **Region Proposal Network (RPN):** まず、画像の中から「ここに物体がいそうだ」という候補領域（バウンディングボックス）を多数提案します。
2. **Classification & Refinement:** 次に、提案された各領域に対して、それが何の物体であるかを分類し、同時にバウンディングボックスの正確な位置を調整します。

この丁寧な二段階処理により精度は高くなりますが、処理速度が遅いという課題がありました。

#### YOLO (You Only Look Once): 速度重視の一段階検出

YOLOは、その名の通り「一度見るだけ」で物体の位置と種類を同時に予測する、画期的なアプローチを提唱しました。

- 画像をグリッドに分割し、各グリッドセルがその場所に含まれる物体のバウンディングボックスとクラス確率を直接予測します。
- これにより、処理が非常に高速になり、リアルタイムでの物体検出が可能になりました。初期のモデルは精度面で課題がありましたが、バージョンアップを重ねるごとに精度も大幅に向上しています。

ViTやQwen-VLは、これらの物体検出モデルとは異なるアプローチ、つまり画像を言語のように扱うことで、より柔軟で強力な画像理解能力を獲得しています。

## 2. Vision Transformer（ViT）の世界：画像認識の革命

自然言語処理の世界で絶大な成功を収めたTransformer。そのアーキテクチャを画像認識に応用したのが「Vision Transformer (ViT)」です。この登場は、画像認識のパラダイムを大きく変えました。

### 用語の整理：ViT, Image Encoder, VLM

まず、似たような用語がいくつかあるので、ここで整理しておきましょう。

- **ViT (Vision Transformer):** 厳密には、Dosovitskiyらが2020年の論文 "An Image Is Worth 16x16 Words" で提案した特定のモデルアーキテクチャを指します。その構成は「**16×16ピクセルのパッチ分割** ＋ **深いTransformerエンコーダ層** ＋ **画像分類用のMLPヘッド**」からなり、元々は画像分類タスクのために設計されました。
- **Image Encoder:** より一般的な用語で、画像を入力として受け取り、その特徴を数値のベクトル（特徴量）に変換するコンポーネント全般を指します。ViTはこのImage Encoderの一種と考えることができます。
- **VLM (Vision-Language Model):** 画像と言語の両方を扱うことができるモデルの総称です。多くの場合、内部にImage Encoder（ViTベースのものが多い）と、言語を処理するLLM（Large Language Model）を含んでいます。Qwen-VLもVLMの一つです。

### ViTは本来、画像識別のモデル

ViTの最も革新的なアイデアは、画像を小さな四角形の「パッチ」に分割し、それぞれを言語モデルにおける「単語（トークン）」のように扱う点にあります。

![ViTの入力画像分割のイメージ](/images/qwen-vl-vit-lora-study/vit-input.jpg)
*左: 元の画像。右: 画像を16x16ピクセルのパッチに分割し、シーケンス（単語の列）のように並べる様子。*

この処理の流れは以下のようになります。

1. **パッチ化 (Patching):** 入力画像を、例えば16x16ピクセルの重なりのないパッチに分割します。
2. **線形埋め込み (Linear Embedding):** 各パッチを平坦化（1次元のベクトルに変換）し、線形変換を適用して、Transformerが扱える次元のベクトル（トークン）に変換します。
3. **位置エンコーディングの追加:** 各パッチが画像のどこにあったかの位置情報を付与します。
4. **Transformer Encoderへ入力:** こうして生成された「画像トークンの列」を、標準的なTransformerのエンコーダに入力します。
5. **タスク固有のヘッド:** Transformer Encoderの出力を、分類タスクであればMLPヘッド、他のタスクであれば対応するヘッドに入力し、最終的な出力を得ます。

![ViTのモデルアーキテクチャ](/images/qwen-vl-vit-lora-study/vit.png)
*ViTの全体アーキテクチャ図（Dosovitskiy et al., 2020より）。Input Layer, Vision Encoder (Transformer Encoder), Head (MLP Head) の3部構成。*

### Vision-Languageタスクへの応用

ViTの成功以降、この「画像をトークン列として扱う」アイデアは、画像と言語を結びつける様々なタスクに応用されていきました。ここではViTの思想を受け継いだ2つの代表的な研究を紹介します。

#### ViLBERT (2019): 物体認識ベースのアプローチ

ViLBERTは、ViTが登場する少し前に提案されたモデルですが、画像と言語をTransformerで融合する先駆的な試みでした。タスクはVQA（Visual Question Answering）です。

ViLBERTの面白い点は、画像を直接パッチ化するのではなく、まず**Faster R-CNNのような物体認識モデルを使って画像内の物体を検出し、その物体の特徴量をトークンとして利用する**点です。言語側のトークンと画像側の「物体トークン」を、それぞれ別のTransformerに入力し、途中で相互に情報を交換（Co-Transformer）させながら理解を深めていきます。

![ViLBERTのアーキテクチャ](/images/qwen-vl-vit-lora-study/vilbert.png)
*ViLBERTの構造。画像から物体を抽出し、言語と協調的に処理する。（Lu et al., 2019）*

#### CPTR (2021): 物体認識に依存しないEnd-to-Endアプローチ

一方、CPTRはImage Captioning（画像の情景を説明する文章を生成するタスク）において、ViTのアイデアをより直接的に活用しました。

CPTRは、ViLBERTのように外部の物体認識モデルに頼るのではなく、**ViTと同様に画像を直接パッチ化**します。これにより、事前学習済みの物体検出器が認識できないような物体や文脈も捉えられる可能性が生まれ、End-to-End（入力から出力まで一気通貫）での学習が可能になりました。

![CPTRのアーキテクチャ](/images/qwen-vl-vit-lora-study/cptr.png)
*CPTRの構造。Encoder-Decoder型のTransformerで、画像パッチを直接扱う。（Liu et al., 2021）*

このように、ViTの登場以降、VLMは「物体認識」という中間ステップから解放され、より柔軟で表現力の高いモデルへと進化していきました。Qwen-VLもこの流れの最先端に位置するモデルです。

## 3. 主役の登場：Qwen-VLのアーキテクチャと学習戦略

いよいよ本題であるQwen-VLについて見ていきましょう。Qwen-VLは、Alibaba Cloudによって開発された、非常に強力なVision-Language Model（VLM）です。特に、高解像度の画像や文書画像の解析において優れた性能を発揮します。

### Qwen-VLの全体アーキテクチャ

Qwen-VLは、大きく3つのコンポーネントから構成されています。

![Qwen-VLのアーキテクチャ](/images/qwen-vl-vit-lora-study/qwen.png)
*Qwen 2.5 VLの全体像 (Alibaba, 2025)。Vision Encoder, LLM, そして両者を繋ぐ部分から構成される。*

1. **Vision Encoder:** ViTベースの画像エンコーダです。入力された画像を処理し、特徴量のシーケンス（画像トークン）に変換します。
2. **LLM (Large Language Model):** テキストの処理を担当する大規模言語モデルです。Qwen-VLでは、強力な`Qwen2.5-LLM`がベースとして使われています。
3. **Merger (or Projector):** Vision Encoderが出力した画像トークンを、LLMが解釈できる形式に変換するための小さなネットワークです。論文によってはProjectorとも呼ばれます。

#### データの流れ：画像はどのようにLLMに渡されるのか？

Qwen-VLの画像処理には特徴的な工夫があります。特に、高解像度画像を効率的に扱うための「Dynamic Resolution」が重要です。

> データは28の倍数にリサイズされ、VisionEncoderに渡されます。VisionEncoderはストライド14でパッチにし、パッチを4グループにしてから1次元に並べてLLMに渡します（可変長）。

このプロセスにより、入力画像の解像度に応じて、LLMに入力される画像トークンの数を柔軟に変えることができ、計算効率と性能を両立させています。

### Qwen-VLの学習戦略

Qwen-VLの高性能を支えているのは、巧妙に設計された学習プロセスです。計算効率化の工夫から、多段階にわたる学習フェーズまで、様々な技術が盛り込まれています。

#### 計算効率化とデータ拡張の工夫

- **Windowed Attention:** 通常のAttentionは計算量がシーケンス長の2乗（O(n²)）で増えますが、隣接するトークンとのみAttentionを計算するWindowed Attentionを採用することで、計算量を線形（O(n)）に抑えています。
- **MRoPE (Multimodal Rotary Position Embedding):** 近年のLLMで主流となっている位置エンコーディングRoPEをマルチモーダル向けに拡張したMRoPEを採用し、テキストと画像の位置関係をより効果的に学習します。
- **データ拡張 (Data Augmentation):** Qwen-VLの学習では、特に画像の解像度に関する動的な拡張が用いられます。「Qwen 2.5 VL Technical Report」によると、学習中、各画像は `{336, 448, 672, 896, 1008}` といった複数の解像度の中からランダムに選ばれたサイズにリサイズされます。これにより、モデルは様々なサイズの画像に対応できる頑健性（ロバストネス）を獲得します。これは「Dynamic Resolution」と呼ばれる特徴の中核をなす技術です。

#### 2つの際立った特徴

- **Document Parsing:** 請求書や論文など、複雑なレイアウトを持つ文書画像の解析（OCRや情報抽出）が非常に得意です。これは後述する独自の学習データ形式が大きく貢献しています。
- **Absolute Coordinates & Dynamic Resolution:** 画像内の絶対座標を理解し、様々な解像度の画像（Dynamic Resolution）を柔軟に扱える能力を持っています。これにより、画像の特定領域に関する細かい指示にも対応できます。

#### 巧妙な多段階学習

Qwen-VLは、一度にすべてのパラメータを学習するのではなく、段階的に学習を進めていきます。これにより、各コンポーネントがそれぞれの役割を効率的に学習できます。

1. **LLMの初期化:** まず、強力な学習済みLLMである`Qwen2.5-LLM`を初期値としてロードします。これにより、高い言語能力を最初から備えた状態になります。
2. **Pre-training (事前学習) - Phase 1:**
   - **対象:** Vision Encoderのみを学習させます。
   - **データ:** OCR（文字認識）、Image Captioning（画像説明文生成）、物体検出などの基本的な視覚タスクのデータセットを使用します。ここで画像の基本的な特徴を捉える能力を養います。
3. **Pre-training (事前学習) - Phase 2:**
   - **対象:** Vision EncoderとLLMの両方を学習させます。
   - **データ:** より高度で複雑なデータセットを使用します。ここで画像と言語を統合する高度な能力を獲得します。
4. **Pre-training (事前学習) - Phase 3:**
   - **対象:** Vision EncoderとLLMの両方。
   - **データ:** さらに長いシーケンス長のデータで学習を継続し、長文や高解像度画像への対応能力を強化します。
5. **SFT (Supervised Fine-Tuning) & DPO (Direct Preference Optimization):**
   - **対象:** LLM部分のみを学習させます。
   - **解釈:** この段階ではVision Encoderはすでに「完成」しており、凍結（パラメータを更新しない）されます。対話能力や指示追従能力といった、より人間らしい応答性能はLLM側が担うという思想です。SFTで対話形式のデータを学習し、DPOでより好ましい応答を生成するように調整します。

:::message
**Phase 2で使われる学習データの種類と例**

Qwen-VLは、多様なタスクをこなすために、非常に幅広い種類のデータセットで学習しています。以下にその一部を紹介します。

| データカテゴリ | 概要 | 代表的なデータセット例 |
| --- | --- | --- |
| OCR & Document Parsing | 画像内の文字を認識し、文書の構造を理解するタスク。 | DocVQA、ChartQA |
| Image Captioning | 画像の内容を説明する文章を生成するタスク。 | COCO Captions、TextCaps |
| Visual Question Answering (VQA) | 画像についての質問に答えるタスク。 | VQAv2、GQA |
| Interleaved Data | Webページのように、画像とテキストが混在・連続するデータ。 | MMC4、LAION-COCO |
| Multimodal Mathematics | 数式や図を含む数学の問題を解くタスク。 | MathVista |
| その他 | Webエージェントタスク、動画の理解、純粋なテキストデータセットなども組み合わせて学習。 | — |
:::

### Qwen-VLの学習データを覗いてみる

モデルの性能はデータによって決まると言っても過言ではありません。Qwen-VLは、学習データの形式にも大きな特徴があります。

#### SFTフェーズ：標準的なChatML形式

SFT（対話ファインチューニング）フェーズでは、OpenAIが提唱したChatML（Chat Markup Language）形式が採用されています。これは、システムプロンプト、ユーザーの発話、アシスタントの応答を明確に区別するためのフォーマットです。

```
<|im_start|>system
You are a helpful assistant.
<|im_end|>
<|im_start|>user
Who were the founders of Microsoft?
<|im_end|>
<|im_start|>assistant
```

#### Visionタスク：独自規格「QwenVL HTML Format」

Qwen-VLの真骨頂は、画像認識タスク、特に文書解析で使われるこの独自フォーマットにあります。これは、HTMLに似た構文で、テキストの位置情報を`data-bbox`（バウンディングボックス）属性として埋め込む形式です。

![QwenVL HTML Formatの出力例](/images/qwen-vl-vit-lora-study/qwenhtml.png)
*Qwen-VLが請求書の画像を解析し、その構造とテキストをHTMLライクな形式で出力した例。各要素に位置情報(bbox)が付与されている。*

#### 実践例1：請求書の認識（得意なタスク）

実際に請求書の画像をQwen-VLに入力すると、驚くほど正確にその構造をHTML形式で書き出してくれます。これにより、単なるOCR（文字を読むだけ）ではなく、文書のレイアウトや意味的なまとまり（どこが請求先住所で、どこが明細テーブルかなど）まで理解していることがわかります。

**プロンプト例:**

```
# System Prompt
You are an AI specialized in recognizing and extracting text from images.
Your mission is to analyze the image document and generate the result in
QwenVL Document Parser HTML format using specified tags while maintaining
user privacy and data integrity.

# User Prompt
QwenVL HTML
```

**出力（全文）:**

````json
{
  "response": "```html\n<html><body>\n<h2 data-bbox=\"103 94 235 128\">Invoice</h2>...",
  "messages": [
    {"role": "user", "content": "QwenVL HTML"},
    {"role": "assistant", "content": "```html\n<html><body>..."}
  ],
  "images": [
    {
      "path": "https://quickbooks.intuit.com/oidam/intuit/sbseg/en_us/Blog/Illustration/example-invoice-image-us-en.png"
    }
  ]
}
````

出力されたHTMLには、`<h2 data-bbox="103 94 235 128">Invoice</h2>` のように各要素に位置情報が付与されており、請求書の構造（宛先、明細テーブル、合計金額など）が正確に再現されています。

#### 実践例2：楽譜の認識（苦手なタスク）

一方で、面白いけれども、まだ完璧ではないタスクもあります。例えば、ブラームスの楽譜を読み取らせてみました。

![入力したブラームスの楽譜](/images/qwen-vl-vit-lora-study/brahms.png)
*入力画像: 楽譜の一部*

結果を見ると、`Allegro non troppo, ma con brio`のようなテキスト部分は正しく読み取れています。しかし、音符や記号そのものは、ABC記譜法というテキストベースの音楽フォーマットに変換しようとしていますが、その内容は正確ではありません。これは、楽譜という非常に専門的で構造化された記号体系の解釈が、まだ発展途上であることを示唆しており、興味深い点です。

#### リーダーボードでの位置

ちなみに、様々なVLMの性能を比較する `Open VLM Leaderboard` (2024年5月時点)では、Qwen-VL Plusが48位にランクインしています。これは非常に競争の激しい分野であり、モデルは日々進化しています。

## 4. まとめと今後の展望

今回は、Qwen-VLをLoRAでチューニングするという最終目標に向けた第一歩として、その背景にあるViTからQwen-VL自体のアーキテクチャ、そして特徴的な学習データまでを概観しました。

今回の調査で、Qwen-VLの強みが、特に文書解析タスクにおいて、`data-bbox`を含む独自のHTML形式のデータで学習している点にあることが見えてきました。これは、LoRAでファインチューニングを行う際、**どのような形式の教師データを用意すればモデルのポテンシャルを最大限に引き出せるか**という重要なヒントを与えてくれます。

### 次回以降で探求したいこと

今回の調査で、さらに深掘りしたいテーマがいくつか出てきました。

- **Qwenのデータ拡張戦略：** Qwenが事前学習で具体的にどのようなデータ拡張（Augmentation）を行っているのかを詳細に調査したい。（今回は解像度の動的変更について触れました）
- **オープンデータセットの探求：** 公開されているマルチモーダル系のデータセット（例えばLAIONなど）を実際に眺めてみて、どのようなデータがモデルの能力を向上させるのかを肌で感じたい。
- **InferenceのScaling Lawの調査：** モデルの性能と、推論時の解像度やパラメータ数の関係性（Scaling Law）についての論文（Chu et al., 2025など）を読み解きたい。
- **Transformerの基礎復習：** 今回は軽く流したMulti-Head Attentionや、Qwenでも採用されているRoPEの仕組みについて、図解を交えながらじっくり勉強したい。
- **Context Engineeringの実践：** プロンプトや入力データを工夫することでAIエージェントの性能を最大化する「コンテキストエンジニアリング」についての知見（Manus社のブログなど）を学び、Qwenの性能評価に応用したい。

## 引用文献

- Qwen 2.5 VL Tech Report (Alibaba 2025) - https://arxiv.org/abs/2502.13923
- Alibaba Cloud - https://www.alibabacloud.com/help/en/model-studio/vision
- ViT入門 (書籍) - [Amazon.co.jp](https://www.amazon.co.jp/Vision-Transformer%E5%85%A5%E9%96%80-Computer-Library/dp/4297130580)
- Microsoft - Chat Markup Language - [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/chat-markup-language)
- An image is worth 16x16 words (Dosovitskiy 2020) - https://arxiv.org/pdf/2010.11929
- ViLBERT (Lu 19) - https://arxiv.org/abs/1908.02265
- CPTR (Liu 21) - https://arxiv.org/pdf/2101.10804
- You Only Look Once (Redmon 2016) - https://arxiv.org/abs/1506.02640
