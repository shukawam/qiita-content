---
title: OCHaCafe LLM のエコシステムでの QA について
tags:
  - oci
  - langchain
  - LLM
  - langfuse
private: false
updated_at: '2024-07-17T22:39:53+09:00'
id: f41c2a6000b0da069643
organization_url_name: oracle
slide: false
ignorePublish: false
---

# はじめに

本記事は、2024/07/10 に行われた [Oracle Cloud Hangout Cafe Season8 #6 - LLM のエコシステム](https://ochacafe.connpass.com/event/320593/)でいただいた質問内容に対する回答記事です。
嬉しいことに当日は多くの質問をいただきましたが、時間の都合上答えることができなかったものが多くあるので、このような形で回答とさせていただければと思います。

尚、当日の資料と動画は以下で公開されています。

https://speakerdeck.com/oracle4engineer/ecosystem-of-llm

https://www.youtube.com/watch?v=dDenmzq7uBM

# いただいた質問とその回答

## 録画や資料は共有されますでしょうか

[はじめに](#はじめに)でも記載しましたが、今回に限らず当日の資料や動画は全て公開されます。
また、過去のアーカイブの一覧は、こちらから参照可能です。

https://developer.oracle.com/ja/community/japan-events/cloud-hangout-cafe.html

## LangChain で行う LLM 開発は Dify でも実行可能でしょうか？

[Dify](https://dify.ai/jp) は、LLM アプリケーションを開発するためのローコード開発プラットフォームです。
LangChain を活用して作成するようなアプリケーションをお手軽に作成することができるため、質問に対する回答は「Yes」となるでしょう。

Dify の公式ドキュメントに LangChain との違いが述べられているので、さらに気になる方はこちらも併せてご参照ください。

https://dify.ai/blog/dify-vs-langchain

## LLM の学習においてベクトルデータベースは必須でしょうか？その理由はどのようなことでしょうか？milvus 以外にも weaviate、cozodb、volox などありますが、どのように選択すればよいでしょうか？

（当日の文脈的に RAG のナレッジベースとしてベクトルデータベースが必須なのか？という質問かと思われますのでその文脈で回答します）

答えは、「No」です。RAG のナレッジベースとしてベクトルデータベースがよく用いられるのは、プロンプト拡張に用いるコンテキスト・データを渡す際に

- LLM に入力可能なトークン/文字数に限りがある
- LLM の推論 API を実行するのに、トークン/文字数に比例してコストが発生する
- 余計な情報を渡すとテキスト生成のノイズになり得る

という背景があり、ユーザーの入力と関連性が高いものをピンポイントに必要十分な数を渡すことが望ましいためです。この要件を満たすために、ベクトルデータベースがよく用いられます。
つまり、ユーザーの入力に関連するものを検索して、プロンプトに埋め込むという要件だけを考えてみると、対象の検索システムは、Elasticsearch/OpenSearch のような全文検索エンジンでもいいですし、任意の Web Search API を実行して得られた結果を使うでも良いわけです。

また、数多くあるベクトルデータベースの選定ですが当日、@minorun365 さんも言及されていた通り、コストに関する観点（セルフホスト可能？商用版のみ？等）は重要な選定理由かと思いますし、それ以外にも

- 格納するコンテキスト・データの性質にあったインデックス・メソッドが用意されているか？
- パフォーマンス（1 秒あたりに実行可能なクエリ数、クエリの平均レイテンシー、データの取り込み時間、等）はどうなのか？
- 検索精度を向上させるための仕組み（ハイブリッド検索、等）には対応しているのか？
- 増加するデータ量やユーザーの要求にはどの程度追従できるのか？
- ベクトルデータベース自体の可用性はどのように担保されている、担保するのか？
- 格納するデータ自体の暗号化、アクセス制御、認証などのセキュリティ要件が満たせるか？
- クエリのパフォーマンスを最適化するために必要な情報が簡単に取得でき、それがわかりやすい形で可視化できるか？
- ドキュメンテーションはきちんと整備されているか？
- エコシステムとの統合性はどうなのか？

あたりが主要な観点になるかと思われます。

## 高次の非構造データから RAG で文章生成する場合、ハルシネーションを抑制するのに有効な RAG の Trick は何でしょうか？RAG の結果をナレッジグラフ化したデータをもとに文章生成する方法が提案されていますが、これ以上に有効なアーキテクチャはありますでしょうか？

2 つ以上の検索方法（e.g. ベクトル検索 + キーワード検索）を組み合わせて、その検索結果と入力プロンプトとの関連性を再評価（リランキング）することで回答精度を向上させるハイブリッド検索が有名な手法として知られています。
また、HyDE(Hypothetical Document Embeddings) と呼ばれる入力プロンプトを LLM へ与えて仮のテキストを生成させ、その生成されたテキストをもとにベクトル検索を行うことで、ベクトルデータベースに格納されているコンテキスト・データとの関連性をより正確に評価できる可能性がある手法も知られています。これ以外の手法については、[LlamaIndex - A cheat sheet and some recipes for building rag](https://www.llamaindex.ai/blog/a-cheat-sheet-and-some-recipes-for-building-advanced-rag-803a9d94c41b) が参考になるかと思われます。いわゆる GraphRAG に関しては、次の勉強会で取り扱う予定のためご都合があいましたら参加いただけると幸いです。

## MySQL 9.01 でも vector type 使えるようになりましたが、HeatWave のほうがいいケースはありますか

:::note warn

以下の情報の一部(HeatWave で `vector_distance()` が使用可能)は、ドキュメントからは判断できず、個人によって書かれているブログ記事内で参照されている GitHub のコードから判断した情報となります。

https://lefred.be/content/use-oci-genai-and-mysql-heatwave-to-interact-with-your-wordpress-content/

https://github.com/lefred/oci-genai-hw/blob/837ccffe5b77bd551cd3b11b6f7bd9bde8f0860c/wp_genai.py#L119-L122

:::

正確には、MySQL 9.0.0 で vector data type が導入されました。[MySQL のドキュメント](https://dev.mysql.com/doc/refman/9.0/en/vector-functions.html)を参照すると、現状ベクトル関連の関数は `STRING_TO_VECTOR()`, `VECTOR_DIM()`, `VECTOR_TO_STRING()` のみとなっており、ベクトルデータの保存は可能ですが、ベクトル間の類似性を評価することができません。HeatWave MySQL を利用することで、ベクトル間の類似性評価を行うことができます。また、HeatWave MySQL では、HeatWave ノードの台数を追加することで、ベクトル間の類似性評価のための関数（`vector_distance()`）の処理性能を向上させることができます。

## LLM、SLM、MLLM では Langchain によるエコシステムの設計方法はどのような点が異なるでしょうか？

同様だとお考えください。

## RAG と Function Calling を使って LLM のアーキテクチャをうまく設計する Tips を教えていただけますでしょうか？

まず、Function Calling は、LLM の出力を外部関数を呼び出すための引数として利用する機能のことで、RAG の中で知識を補うための手段の一つです。
このとき、大事なことは自分の用意した関数 or Tool として提供されている関数を意図したものを呼び出すために、きちんとメタデータを設定することや、意図した通りに呼び出される/呼び出されなかったことを確認可能な仕組みを作っておくことかと考えています。例えば、単純にログ出力しておくことやトレース情報を取得する等の対応が考えられます。

## 実用可能な LLM の性能を担保する品質保証の仕方を教えて頂けますか？LLM プロダクトを商用リリースするため、ハルシネーションを防止する方法はどのような方法でしょうか？

LLM そのものの評価には、日本語であれば [JGLUE](https://github.com/yahoojapan/JGLUE) や [Japanese MT-Bench](https://github.com/Stability-AI/FastChat/tree/jp-stable/fastchat/llm_judge/data/japanese_mt_bench)、英語であれば [GLUE](https://gluebenchmark.com/) など有名なデータセットが存在するので、これを用いてベンチマークを実行し、ある一定水準であれば OK などと評価する方法があるでしょう。
一方、RAG のパイプラインを評価するためのフレームワークとして、[RAGAS](https://github.com/explodinggradients/ragas) は非常に有名かと思いますし、セッション中で取り上げた Langfuse でも似たようなことを実施することができます。

また、ハルシネーションの抑制案としては [高次の非構造データから...](#高次の非構造データから-rag-で文章生成する場合ハルシネーションを抑制するのに有効な-rag-の-trick-は何でしょうかrag-の結果をナレッジグラフ化したデータをもとに文章生成する方法が提案されていますがこれ以上に有効なアーキテクチャはありますでしょうか) で述べた通りです。

## Langfuse は LangChain や LlamaIndex のラッパーのようなイメージで合っておりますでしょうか？

LangChain や LlamaIndex との統合は用意されていますが、ラッパーではなく独立した LLMOps のためのツールだとお考えください。

## Llama ベースで日本語に特化したファインチューニングを行う上で LangChain で活用すべき機能（QLoRA など）は何でしょうか？

LangChain は、LLM を活用したアプリケーションを構築するためのフレームワークであり、LLM 自体を作成したりファインチューニングするようなフレームワークではないため、存在しません。

## LangChain では、Model Merge や Syntactic Attention Structure などの新しい LLM 設計手法を実装することは可能でしょうか？

LangChain は、LLM を活用したアプリケーションを構築するためのフレームワークであり、LLM 自体を作成するようなフレームワークではないため、不可です。

## LLM を構成する Transformer のアーキテクチャやハイパーパラメータを自動設計する NAS のような機能は実装されていますでしょうか？学習データのキュレーション・クレンジングなどの前処理を支援する機能はどのようにサポートされているでしょうか？

LangChain は、LLM を活用したアプリケーションを構築するためのフレームワークであり、LLM 自体を作成したりファインチューニングするようなフレームワークではないため、実装されていません。
前処理の支援機能に関しても同様です。

## LangChain で ChatGPT、Llama、Gemini、Claude、Command、Mistral をアンサンブルして多数決法で精度を上げるような LLM を設計することは可能でしょうか？

LangChain を使うことで実装コストを多少減らす＆実装自体は可能だと思いますが、自分で実装する部分はかなり多いと思われます。
大きく分けて、LLM の推論前にアンサンブルする手法と推論後にアンサンブルする手法が存在すると思いますが、推論前にアンサンブルする手法は最適な LLM を選択するルーター部分の作り込みが必要となりますし、この部分は私が知る限り、LangChain の機能としてまだ存在しません。一方、推論後にアンサンブルする方法は、複数 LLM が生成した結果をスコアリングし、結果の高いものを回答をして選択するようなアプローチが考えられますが、こちらも一連の流れを抽象化するような統合機能はまだ存在しない認識です。

:::note info

厳密には、推論中にアンサンブルする方法もあるようです。複数 LLM 間の協調アプローチに関しては、論文を発見しましたので、興味があれば併せて参照ください。

https://arxiv.org/html/2407.06089v1

:::

## LlamaIndex のメモリ会話履歴が弱いですけど、チャットアプリだとどちらがいいのかな

おっしゃる通り、2024/07 現在 LlamaIndex でチャットの会話履歴を管理するものは、SimpleChatStore, RedisChatStore, AzureChatStore のみとなっています。LangChain では、これよりも豊富にインテグレーションが提供されていますので、チャット履歴の保存に使用したいコンポーネントが見つかる可能性は高いと思います。

## LangChain や LlamaIndex で国内製の LLM を実装することは可能でしょうか？

LangChain では、任意の LLM を LangChain 対応させるための基底クラスが提供されています。

https://github.com/langchain-ai/langchain/blob/master/libs/core/langchain_core/language_models/llms.py

https://github.com/langchain-ai/langchain/blob/master/libs/core/langchain_core/language_models/chat_models.py

この中で定義されている I/F(`invoke`, `ainvoke`, `stream`, `astream`, ...)を実装することで任意の LLM を LangChain 対応させることが可能です。

# 終わりに

今回は、セッション中にいただいた質問への回答記事となりました。本記事中のセッションや資料が LLM アプリケーションの開発や運用をしている方にとって少しでも役に立てば幸いです。
