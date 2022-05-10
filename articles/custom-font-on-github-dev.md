---
title: "Github.devでも日本語等幅フォントを使いたい人に" # 記事のタイトル
emoji: "🖊️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["github", "vscode", "font", "chrome", "stylus"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---
githubのリポジトリ上で「 `.` 」を押すと（あるいはURLの _.com_ を _.dev_ に書き換えると）、 _github.dev_ が立ち上がります。

これは基本VSCodeで、リポジトリの内容を編集・コミットなど一連の動作ができます。

_Settings Sync_ も対応しているので、ローカルとほぼ同じ環境（ターミナルは使えません）で作業ができるのですが、

フォントが違う！

デフォルトで指定されているフォント（もう消してしまったので名前失念）は欧文フォントであるらしく、日本語は等幅になってくれません。

等幅フォントを入れたい――できればローカルと同じやつで。

そこで登場するのがChrome拡張機能です。

今回は [Stylus](https://chrome.google.com/webstore/detail/stylus/clngdbkpkpeebahjckkjfobafhncgmne?hl=ja)を導入しました。

これはカスタムスタイルシートを実現してくれる拡張機能です。

予めGoogle Fontsでmonospace系のフォントを選択しておきます。

![スクショ1](/images/custom-font-on-github-dev/screen_01.png)

![スクショ2](/images/custom-font-on-github-dev/screen_02.png)

![スクショ3](/images/custom-font-on-github-dev/screen_03.png)

必要なのは `@import url` から始まるURL文字列と、`font-family` に指定するためのフォントの名前です。

今回は _M PLUS 1 Code_ を選びました。

（monospaceフォントのはずなのにsans-serifになっています。指定ミス？）

_Stylus_ の管理画面でGithub.devに対するスタイルを記述します。

スタイルというか、フォントのインポートです。

![スクショ4](/images/custom-font-on-github-dev/screen_04.png)

これでGithub.devを訪れた際、M Plus Codeフォントが追加で読み込まれるはずです。

次にgithub.devの設定画面を開き、Font familyを指定します。

![スクショ5](/images/custom-font-on-github-dev/screen_05.png)

指定内容は先程コピペしたフォント名です。

これでgithub.devで好きな（CDNにある必要はある……）フォントを使えるようになりました。

Googleフォントはローカルにダウンロードもできるので、 _Settings Sync_ で常にM Plus Codeフォントという環境も可能です。

Webフォントで等幅の選択肢があまりないのが泣きどころですが、

github.devを使う人は試してみてはいかがでしょうか。