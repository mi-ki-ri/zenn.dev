---
title: "gemini-2.0-flash-exp に自作ボカロ曲を聞いてもらった"
emoji: "🎼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gemini", "llm", "audio", "music", "contest2024"]
published: true
---

## 概要

Google 製のマルチモーダル AI モデルである Gemini の、バージョン 2.0 が出ました。

せっかくマルチモーダルなので自分の作ったボカロ曲を聞かせて、歌詞起こしと分析を行ってみました。

Spotify の API から取ることのできる audio_features はけっこう精度があるのですが、当たり前ながら Spotify に曲を配信してもらわないと使うことができません。また、歌詞は人力です（プチリリと提携しているらしいので）。

Gemini2.0 でどの程度の精度が出るか、実験のつもりでやってみました。

## 歌詞起こし編

Python 向けライブラリ `google-genai` を使います。

```sh
pip install google-genai
```

で良いです。

少し迷ったのが、公式のドキュメントがまだ古いようで、ファイルアップロードの仕方が変わっていました。PyPI のドキュメントのほうが正しかったようです。

```python
client = genai.Client(api_key=YOUR_API_KEY)
myfile = client.files.upload(path=FILE_PATH)
```

上記の`client.files.upload`が現在の書き方のようです。

あとはそんなに難しいことはなく、

```python
uri = myfile.uri
mime = myfile.mime_type

response = client.models.generate_content(
    model="gemini-2.0-flash-exp",
    contents=[
        types.Part.from_uri(uri, mime),
        types.Part.from_text("この曲の歌詞を聞き取って書き出してみて"),
    ],
)
print(response.text)
```

このような感じで良いです。

### 結果

ちなみに、曲は[見切り発車 P - シチューソング feat. 猫村いろは](https://youtu.be/d_p8CNgTxHE)です。現時点での最新曲。

元歌詞は[BandCamp](https://mikirihasshap.bandcamp.com/track/--333)に上げています。

```txt
新しい季節始めました
新しいメニュー始めました
北風に負けずに一歩また
冷めし風を受けてもなお
君は微笑みを忘れない
僕は君のその笑顔が好きです

さぁおいでよ
ふたりの深皿が今
並べ暖かい湯気に満たされる
いただきましょう

野菜が、手間暇ぜいたく品
大事に大事に煮込みました
育てていたハーブも使いました
北のどちに響く室内
に君の笑顔が戻る
ビールを酌み交わすお疲れ様です

さぁおいでよ
ふたりの深皿が今
並べ暖かい湯気に満たされる
いただきましょう
```

悪くは無いけれど…　微妙にどこか違う。

### 結果 2 ボーカルステム

使っているのが VOCALOID Editor なのでボーカルだけの音声ファイルが存在します。

そちらなら精度が上がるかなと思って試してみたのがこちら。

```txt
新しい季節始めまして
新しいメニュー始めました
北風に負けぬシチューグラタン

生みし風を受けても尚
君は微笑みを忘れない
僕は君のその笑顔が好きです
さあ、踊れよ
二人の深皿が今

並べ暖かい湯気に満たされる
いただきましょう
野菜が手もはや贅沢品
大事に大事に煮込みました
育てていたハーブも使いました
ヒーターの音響く
室内に君の笑顔が戻る
ビールを酌み交わすお疲れ様です
さあ、踊れよ
二人の深皿が今

並べ暖かい湯気に満たされる
いただきましょう
```

多少精度が上がりました。

### 感想

やはり他の楽器の音に邪魔されて聞き取りづらい部分があったのでしょう。

本格運用するなら、STEM 分離してボーカルだけ取り出してから歌詞起こしするのが良いのかもしれません。

## 特徴分析編

今度は曲の感想というか特徴を挙げてもらうことにしました。

JSON でほしいなと思ったので、まずほしいスキーマの class 文を書きます。

```python
class SongInfo(BaseModel):
    tags: list[str]
    jenre: str
    instruments: list[str]
    liner_notes: str
    tempo: str
```

見ての通り、タグ、ジャンル、使用楽器、ライナーノーツ、大まかなテンポ感を得る予定です。

あとは先程のファイルとだいたい同じで、

```python
response = client.models.generate_content(
    model="gemini-2.0-flash-exp",
    contents=[
        types.Part.from_uri(uri, mime),
        types.Part.from_text(
            "この曲のジャンル、タグなどの情報をJSON形式で書き出してみてほしい。タグは5~8個程度ほしい。"
        ),
        types.Part.from_text(
            """例: {
              instruments: ['エレキギター','シンセベース'],
              jenre: "ネオソウル",
              tags: [ネオソウル, チルアウト, ローファイ, ギター, スローテンポ, シャッフルビート, 16フィール, インストゥルメンタル],
              tempo: "スロー",
              liner_notes: "この作品には素晴らしく情感のこもったギターと美しい音色があります。"
              }
            """
        ),
    ],
    config=types.GenerateContentConfig(
        temperature=0.5,
        response_mime_type="application/json",
        response_schema=SongInfo,
    ),
)
print(response.text)
```

こんな感じで書いてみました。

### 結果

```txt
src/見切り発車P - シチューソング feat. 猫村いろは.mp3
{
    "instruments": ["アコースティックギター", "エレキギター", "ベース", "ドラム"],
    "jenre": "J-POP",
    "liner_notes": "冬の始まりを告げるような、温かい雰囲気の曲。アコースティックギターの優しい音色が印象的。",
    "tags": ["J-POP", "アコースティック", "冬", "バラード", "温かい", "ギター", "スローテンポ"],
    "tempo": "スロー"
}
```

上記のような感じ。なかなか良い！

テンポがスローかっていうとそこまでではないけど、だいたい分かってくれてる！

### 結果 2 ジョアン・ジルベルト

調子に乗って CD からリッピングした音源も試してみました。

```txt
src/02 Ela É Carioca.mp3
{
  "instruments": ["acoustic guitar", "bass"],
  "jenre": "bossa nova",
  "liner_notes": "This song has a wonderful mellow guitar and beautiful tones.",
  "tags": ["bossa nova", "chill", "acoustic", "guitar", "slow tempo", "latin", "instrumental"],
  "tempo": "slow"
}
```

ジョアン・ジルベルトの Ela E Carioca という曲です。

ちゃんとボサノバを認識している！

### 結果 3 ジョン・レノン

```txt
src/15 Imagine [Live].mp3
{
  "instruments": ["acoustic guitar", "vocal"],
  "jenre": "folk",
  "liner_notes": "This is a cover of John Lennon's Imagine. It's a simple and beautiful rendition with acoustic guitar and vocals.",
  "tags": ["acoustic", "folk", "cover", "peace", "ballad", "simple", "emotional"],
  "tempo": "slow"
}
```

「John Lennon Acoustic」というアルバムに入っていた Imagine のライブ盤です。

ジョン本人だとは気づかなかったみたいだけど、曲を当てている！　すごい！

### 感想

歌詞起こしよりこちらのほうが有望そうでした。

例えば自分のライブラリ内でバッチ処理をして、タグをメタデータとしてもたせれば、音楽ファイルをムードで検索できるようになるかもしれません。期待大。

## 落とし穴

ただ落とし穴があって、実験中 wav ファイルでも構わずにアップロードしていたので、File Strage Quota をけっこう消費してしまいました。

たぶんしばらく待つと回復すると思うんですけど、バッチ処理などで連続して高速な処理をするには有料プランにアップグレードする必要があるかもしれません。
