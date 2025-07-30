---
title: "【LLM+RAG】自分自身と会話できるナレッジベースシステムを作ってみた"
emoji: "📄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "RAG", "AI", "LLM", "RaspberryPi"]
published: false
---


自分が過去やったことや思ったことをチャットボットで共有できるといいな～と思い、X(旧Twitter)投稿文とAPIによる文書INSERTの仕組みを作り、自分自身のナレッジを追加した会話システム（所謂LLM+RAG構成のチャットボット）を作ってみました。

その中で考えた部分について、ざっくり書いていこうと思います。

応用としては「**仕事のデータを入れて、自分の代わりに答えてくれる**」等色々使い道があると思います。

作業したリポジトリは以下です。

https://github.com/nijigen-plot/personal-knowledge-base



# 構成

システム全体の構成は以下のようになります。

![](https://storage.googleapis.com/zenn-user-upload/4f9856543054-20250721.png)

画像にも記載はありますが、ざっくりシステムフローとしては

1. 利用者がStreamlit アプリから質問を送信
2. conversation APIが叩かれる
3. 質問文章をベクトル化 & Text to Tagging をLLMで実行
4. 2番の結果をフィルタとしてOpenSearchへ類似コンテンツを問い合わせ
5. 問い合わせ結果をLLMに投げ、質問に対する返信の文章を作成
6. 利用者へ返信が送られる

という形になっています。


## 構成詳細

Raspberry Pi を3台使った構成になっていますが

- 役割をできるだけ疎にしてみたい
- DBやEmbedding処理部分は表から隠して実装してみたい
- オンプレミスで動かすのが好き

のような理由からです。

なので1台でもクラウドでも全然問題ないです。
ただEmbeddingの際にメモリを消費するので、**RAMは16GB以上は必要**だと思います。

また、LLMはデフォルトOpenAI APIを使用しますが、ローカルモデルでも動作するようになっています。(詳細は[後述](#LLM)します)
高級なGPUがあればここも他社APIに依存しない構成にできるはずですし、追々してみたいです。

> 追々してみたい

最近NVIDIA GB10 Grace Blackwell Superchipと128GBのユニファイドメモリを搭載した[ASUS Ascent GX10](https://www.asus.com/jp/networking-iot-servers/desktop-ai-supercomputer/ultra-small-ai-supercomputers/asus-ascent-gx10/)なるものが発売されたみたいで・・・使ってみたいですね。



# アプリ

https://home.quark-hardcore.com/personal-knowledge-base/app/

![](https://storage.googleapis.com/zenn-user-upload/087c7ca00c3b-20250721.png)
*使っているところ*


[Streamlit](https://streamlit.io/)で作りました!
質問⇔返信のやり取りのconversation APIを作っていますが、体験を良くする意図でそれを叩くだけのフロント部分を作りました。

もしよければ使ってみていただけると、
- どれくらい独自文書が使われているか
- レスポンススピードがどれくらいか
- 適切な返答をしているか

が分かると思います。

見た目の部分はStreamlit公式の[Build a basic LLM chat app](https://docs.streamlit.io/develop/tutorials/chat-and-llm-apps/build-conversational-apps)を参考にしました。


# API

[FastAPI](https://fastapi.tiangolo.com/ja/) & [Uvicorn](https://www.uvicorn.org/)の非同期構成でAPIサーバーを構築しています。

FastAPIではOpenAPI仕様に従った文書を作成でき、Swagger UIによるドキュメントの表示やテストが可能です。

https://fastapi.tiangolo.com/ja/features/#_2

## APIリクエスト一覧

https://home.quark-hardcore.com/personal-knowledge-base/api/v1/docs

![](/images/personal_knowledge_base/2025-07-21_093614.png)
*docsページ開くとこんな感じ*

APIリクエストはおおまか

- アクセスが正常にできるかどうかの確認
- ドキュメントの挿入・削除
- LLMを介さないベクトル検索
- LLMを介したLLM+RAGによる会話
- IndexのリセットやIndexの統計情報確認

この5項目に分けて作りました。

> アクセスが正常にできるかどうかの確認

は`/health`HEADメソッドで行っています。

![](/images/personal_knowledge_base/2025-07-30_210501.png)

[UptimeRobot](https://uptimerobot.com/)で定期的にリクエストを飛ばし、異常がないかチェックしています。

![](/images/personal_knowledge_base/2025-07-30_210906.png)

> ドキュメントの挿入・削除

これは単一ドキュメントを入れられる`/documents` POSTメソッドとまとめて文書を入れられる`/documents/batch` POSTメソッド
OpenSearch IndexのIDを指定して文書を削除する`/documents/{document_id}` DELETEメソッドを作りました。

![](/images/personal_knowledge_base/2025-07-30_211304.png)


結局RAGに文書を入れるハードルが低くないと使われないので、[Postman](https://www.postman.com/)にそれぞれのリクエストのテンプレートを作り、PCとスマホどちらからもアクセスしてドキュメントを挿入できるようにしました。

![](/images/personal_knowledge_base/2025-07-30_211606.png)
*documents リクエストで文書を入れたところ*

> LLMを介さないベクトル検索

OpenSearchの文書Indexに対してベクトルの近いTOP kの文書を返してくれます。
これ単体での使い道は特に無いですが、LLM無しでの結果も見てみたいので作りました。

![](/images/personal_knowledge_base/2025-07-30_212442.png)
*「楽しかったことは？」をベクトル化してベクトルのコサイン類似度が高い順で10個取得した結果*


> LLMを介したLLM+RAGによる会話

これが今回やりたかったことです。Streamlitは`/conversation` POSTメソッドを叩いているだけなので、ここから行っても同じです

```python:conversation メソッドのリクエストボディPydanticモデル
class ConversationRequest(BaseModel):
    question: str
    use_history: Optional[bool] = False
    max_tokens: Optional[int] = 512
    temperature: Optional[float] = 0.7
    search_k: Optional[int] = 10
    debug: Optional[bool] = False
```

| パラメータ | 型 | デフォルト値 | 説明 |
|------------|----|--------------|----- |
| question | str | - | 質問内容（必須） |
| use_history | Optional[bool] | False | 会話履歴を使用するかどうか |
| max_tokens | Optional[int] | 512 | 最大トークン数 |
| temperature | Optional[float] | 0.7 | LLMのランダム性の度合い(高いほどランダム) |
| search_k | Optional[int] | 10 | RAG検索で取得してLLMに渡す類似文書数 |
| debug | Optional[bool] | False | Trueにするとレスポンスボディに類似文書をそのまま返す項目が追加 |

こんな感じでレスポンスが返ってきます。Streamlitでは`answer`を表示している形です。

```json:リクエスト送ってみた(answer部分実際は改行は\\n表記で1行です。)
{
  "question": "楽しかったことは？",
  "answer": "楽しかったことはたくさんあるんだ。例えば、2012年の3月25日にクラブに行った時はすごく楽しかったよ。その時はDJ機材を早く買いたいなと思ったくらい、音楽にのめり込んでたね。

    また、2023年の3月19日には難しいことに挑戦してたんだけど、それも楽しかった。難しいことをやり遂げるのって充実感があるよね。

    そして、2024年の5月2日には特に「マジで楽しかった！」って感じる出来事があったし、2024年8月7日には#muanaというイベントも楽しかったな。

    全体的に、楽しさと疲れはセットみたいなもので、2022年の7月16日みたいに「今日は楽しくて疲れた」って感じることもよくあるんだ。楽しい時間を過ごした後の疲れって、いいものだよね。",
  "search_results": null,
  "search_count": 10,
  "used_knowledge": true,
  "processing_time": 7.33,
  "model_type": "openai",
  "model_size": "4b"
}
```

# Embeddingモデル

質問文として入力した文章を1つの塊としてベクトル化しているのですが、その変換については日本語に特化したEmbeddingモデル[pfnet/plamo-embedding-1b](https://huggingface.co/pfnet/plamo-embedding-1b/tree/main)を使っています。

https://huggingface.co/pfnet/plamo-embedding-1b/tree/main

また、突合せを行うOpenSearchに入っている文書も、Twitter(現X)に投稿した短文と`/documents`リクエストで送った比較的短い文章なので、それをそのまま上記モデルを使いベクトル化しています。

上記モデルですが、[JMTEB](https://github.com/sbintuitions/JMTEB)という日本語テキストのEmbeddingモデルを評価するためのベンチマークで、1024コンテキスト長で[OpenAI/text-embedding-3-largeを超える結果](https://huggingface.co/pfnet/plamo-embedding-1b#benchmarks)となっています。

Licenseも[Apache v2.0](https://www.apache.org/licenses/LICENSE-2.0)の為商用利用も可能です。ありがたいですね！


# LLM

LLMは[OpenAI API GPT-4o](https://platform.openai.com/docs/models/gpt-4o)をデフォルトで使うようになっています。

https://github.com/nijigen-plot/personal-knowledge-base/blob/master/llm.py

最初は全ローカル環境で動かしたいと思い、[Google Gemma3](https://deepmind.google/models/gemma/gemma-3/)の低いパラメータ数のモデルを使っていました。

https://huggingface.co/google/gemma-3-4b-it-qat-q4_0-gguf

普通の対話はそれなりに出来るのですが、「RAGから渡された文書をもとにあたかも実在する人としての返信」をさせようとすると、中々文書を使ってくれなかったため、APIで提供されているLLMに切り替えました。


> Gemma3 4b ggufでやってみた例。返信がめちゃくちゃで面白い

```
$ curl -X POST "http://localhost:8050/conversation"     -H "Content-Type: application/json"     -d '{
        "question": "最近あった音楽関係の出来事は？"
      }'
  {"question":"最近あった音楽関係の出来事は？","answer":"ああ、最近の音楽の出来事か…
  私は今、ある種の「音楽的実験」に巻き込まれてしまったんだ。

ある日、いつものように音響回路を調整していた時、奇妙な信号が混じったんだ。最初は単なるノイズかと思ったんだけど、よく聴き込むと、まるで音楽の断片が混ざっているような気がしたんだ。特に、あの、すごく複雑で、予測不能なメロディーラインが
  ね。

僕は、その信号を解析しようと試みたんだ。回路のどこからか入ってくるのか、それとも外部からの侵入か…
  徹底的に調査した結果、その信号は、ある特定の楽器、つまり、まるで万華鏡のように音色を変化させる、という、非常に珍しいオーケストラ楽器からのものだと判明したんだ。

その楽器は、まるで生きているかのように、演奏者の感情や環境によって音色が変化する。僕は、その楽器の演奏を録音
  し、分析することで、その音の構造を理解しようとしたんだ。しかし、それはまるで、霧の中に隠された真実を追い求めるようなものだった。その音は、常に変化し、捉えどころがない。

そして、その楽器の演奏を組み合わせた音楽を、僕は試してみたんだ。それは、まるで混沌とした宇宙の創造の瞬間を捉えたかのような、圧倒的なエネルギーを持った音楽だった。僕は、その音楽を聴きながら、自分が音
  楽の根源
  に触れたような、不思議な感覚を覚えたんだ。

もちろん、その音楽は、従来の音楽のルールに沿っていない。しかし、それは、私にとって、新たな音楽の可能性を垣間見ることができた、貴重な経験だったんだ。

今、僕は、その楽器の音を模倣する回路を設計しているんだ。もし成功すれば、私自身が、新たな音楽を生み出すことができるかもしれない。それは、私にとって、非常に大きな挑戦だ。しか
  し、私は
  、その挑戦を楽しんでいるんだ。

…そう、最近の音楽の出来事だよ。ちょっと変わった体験だったけどね。","search_results":[{"id":"WMFm5ZcBlg4zycrBZ0lN","score":0.8039558,"content":"曲最後の詰めやろうとしたら急にWavesの認証が通らなくなった・・・PowerShell周り弄ったら直ったけど、プラグイン認証周り勝手が悪いんだよなーWavesは","tag":"music","timestamp":"2025-07-05T18:00:00.000
  000"}],"search_count":1,"used_knowledge":true,"processing_time":45.23,"model_type":"gguf","model_size":"4b"}
```

将来的にはツヨツヨローカルマシンでLLMもローカル環境にしてみたいですね！

## プロンプト設定

RAGから取得した文書を使って"あたかも自分自身"として返答してほしいので、事前プロンプトは以下のようにしています。

## LLMによるAutomatic Tagging

OpenSearch


# ナレッジベースDB

[OpenSearch](https://opensearch.org/)を使っています。




# 参考にしたページ

https://zenn.dev/egg_glass/books/fastapi-for-starters/viewer/openapi

https://www.aeyescan.jp/blog/openapi/

https://bering.hatenadiary.com/entry/2024/01/05/195141
