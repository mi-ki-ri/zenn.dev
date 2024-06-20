---
title: "transformers.jsを使ってローカルでLLMを動かすという野望" # 記事のタイトル
emoji: "🤖" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["transformers", "AI", "transformersjs", "nodejs"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

## あらまし

ネットサーフィンしていたら [transformers.js](https://huggingface.co/docs/transformers.js/index)というライブラリを見つけました。

これは Python の transformers と同じような機能を JS で提供しているもので、ブラウザでも動くらしいです。すごい。

生成 AI を使ったアプリを開発しようと思ったらコストは問題になります。

あるいはユーザーに OPENAI などの API_KEY を取得してもらう手もありますが、初心者には難しいです。

しかしブラウザで生成 AI を動かせればコストはユーザー持ちになり、キーの取得のような手続きも必要ありません。

いい事ずくめなので試してみました。

目標はテキストからハッシュタグを生成することです。

あと今回はまだブラウザではなくローカルの nodejs 環境です。

```txt
今日は大きい犬を見ました。
```

上のようなテキストを入力して、

```js
[{ tag: "犬" }, { tag: "大きい" }, { tag: "アニマル" }];
```

こういう配列をゲットしたい。

[JavaScript で LLM を弄ってみる【transformers.js】](https://zenn.dev/saldra/articles/44bc401e773a62)を参考にさせてもらいました。

## モデルのコンバート

この transformers.js は ONNX という形式のモデルを読み込むようになっています。

日本語テキスト生成モデルの ELYZA を使いたかったので、ひとまずコンバートをかけてみます。

コンバート用のスクリプトは[公式リポジトリ](https://github.com/xenova/transformers.js)の中に scripts として入っています。

```bash
python -m venv venv
. ./venv/bin/activate
pip install -r scripts/requirements.txt
python -m scripts.convert --quantize --model_id <model_name_or_path>
```

こんな感じでコンバートスクリプトを走らせることができます。

### デフォルトのプロンプトが無いエラー

ELYZA の場合、デフォルトのプロンプトが無いとかいうエラーが出ます。

軽微なエラーだと思ったので convert.py を **直にいじって** 解決しました。

`scripts/convert.py`の 336 行目あたり、

```python
# setattr(tokenizer, 'chat_template', tokenizer.default_chat_template) # これだとエラー
setattr(tokenizer, "chat_template", "") # しかたなく空白を入れる
```

このようなことをしてエラーを強引に鎮めました。

### 自分の環境では失敗

が、結局自分の環境(Macbook)ではうまくコンバートできませんでした。

普通のコンバートはできるのですが、Quantize されたモデルは出てこないのです。

GPU の問題か、CUDA の問題のようです……。

仕方なくありものを使ってみます。

## テキスト生成

### rinna の場合

上記 saldra さんの記事の"saldra/rinna-japanese-gpt2-xsmall-onnx"を利用してみました。

```js
import { env, pipeline } from "@xenova/transformers";
env.allowLocalModels = false;
env.localModelPath = "models/";

let generator = await pipeline(
  "text-generation",
  "saldra/rinna-japanese-gpt2-xsmall-onnx"
);
let result = await generator(
  "以下のテキストに適切なハッシュタグをつけてください。\n\n<text>今日は大きい犬を見ました。</text>"
);
console.log(result);
```

上記のような JS を書いて実行してみました。

環境はローカルの node なのでブラウザでは有りませんが、原理的には同じはず。

```bash
[
  {
    generated_text: '以下のテキストに適切なハッシュタグをつけてください。 <text>今日は大きい犬を見ました。</text>'
  }
]
```

返答は返ってきたけど、ハッシュタグは生成されていません。オウム返し的な……。

### TinyLlama-1.1B-Chat の場合

別のモデルを探してみました。

[https://huggingface.co/models](https://huggingface.co/models)の左ペインにある絞り込みを利用して text-generation で ONNX のモデルをいくつか探してみます。

日本語モデルではないので、英語での入力になります。

TinyLlama 1.1B-Chat の ONNX 版があったので、使わせてもらいました。

```js
import { pipeline, env } from "@xenova/transformers";
env.allowLocalModels = true;
env.localModelPath = "models/";

const modelName = "Xenova/TinyLlama-1.1B-Chat-v1.0";
const generator = await pipeline("text-generation", modelName);

// Define the list of messages
const messages = [
  { role: "system", content: "you're a instagram tagging ai." },
  { role: "system", content: "please tagging following text for instagram." },
  {
    role: "system",
    content:
      "answer is JSON format, like[{tag: 'dog'},{tag: 'cat'},{tag: 'animal'}]",
  },
  {
    role: "user",
    content: "this is my dog.",
  },
];

// Construct the prompt
const prompt = generator.tokenizer.apply_chat_template(messages, {
  tokenize: false,
  add_generation_prompt: true,
});

// Generate a response
const result = await generator(prompt, {
  temperature: 2,
});
console.log(result);
```

```bash
[
  {
    generated_text: '<|system|>\n' +
      "you're a instagram tagging ai.\n" +
      '<|system|>\n' +
      'please tagging following text for instagram.\n' +
      '<|system|>\n' +
      "answer is JSON format, like[{tag: 'dog'},{tag: 'cat'},{tag: 'animal'}]\n" +
      '<|user|>\n' +
      'this is my dog.\n' +
      '<|assistant|>\n' +
      'please tagging following text for instagram.'
  }
]
```

しかしオウム返し癖は直らず……。

## 結論

何かが足りないのだと思うので、もう少し技術の発展を待つことにします。
