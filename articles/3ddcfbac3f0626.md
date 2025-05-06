---
title: "【Next.js】アニメリスト検索アプリを作成してみた"
emoji: "🐮"
type: "tech"
topics:
  - "nextjs"
  - "api"
  - "正規表現"
  - "cors"
  - "chakraui"
published: true
published_at: "2023-02-10 21:54"
---

## アプリ概要
タイトルの通り、Next.jsを用いてアニメリスト検索アプリを作成しました。簡単に説明すると、クールごとのアニメの一覧をannictというサービスの人気順で表示するようなWebアプリケーションです。実際のアプリの挙動やソースコードは以下をご確認ください。
- [アニメリスト検索アプリ](https://anime-list-search-nine.vercel.app)
- [ソースコード](https://github.com/shimpei1494/anime-list-search)
![アプリ画像](https://storage.googleapis.com/zenn-user-upload/5429fffd132e-20230207.png =500x)
*完成アプリ画面（ダークモード）*

## 作成した理由
どの時期になんのアニメがやっているのかという情報を簡潔に知ることができるサイトはあまりないように感じたため、今回作成するに至りました。人気順で表示することで、あまりアニメに詳しくない人でも面白いアニメを見つけやすくなるということも心がけて作成しました。

## 環境
Next.jsとAPIを用いてフロントエンドのみで開発を行いました。
```
node -v -> v18.12.1
npm -v -> 8.19.2
```
```js:packge.json
"next": "13.1.1",
"react": "18.2.0"
```
## 使用したAPI
- [annict](https://developers.annict.com/)
- [しょぼいカレンダー](https://docs.cal.syoboi.jp/spec/feeds/)

:::message
ここから先のアプリの説明は、私が初めて使用した技術や発生したエラーなどを中心に紹介していきます。話としてはまとまりがないという点はご了承ください。
:::

## ChakraUIの使用
初めてChakraUIを使用してみました。ライブラリを入れて、すでに用意されたパーツ（コンポーネント）を使用することで、簡単にUIを構築することができました。色やサイズなどもCSSを記載せずともpropsというものを用いて、bootstrapのような感じで変更することができます。今回のアプリはパーツに関する多少のサイズ変更のみでレスポンシブになってくれたというのも非常に助かりました！
公式HP含め参考になったサイトを一部、以下に記載します。
[公式HP](https://chakra-ui.com/)
[Chakra UI の基本的な使い方](https://fwywd.com/tech/chakra-ui-howto)
### ライトモード、ダークモード
ChakraUIを用いることで、アイコンをクリックすることでライトモードとダークモードを簡単に切り替えることが可能になりました。以下リンクを見れば簡単に実装できるということがわかるかと思います！
[Chakra UI : ダークモードの切り替えをアイコンを使って実装する](https://qiita.com/hirochan/items/3ea354440ab53c71c265)
唯一の注意点としては背景や文字などについて、自分で色を設定した部分はライトモード、ダークモードで色が切り替わらないということですね（設定の仕方にもよりますが）。

### モーダル
こちらもChakraUIを用いて簡単に実装することができました。以下の公式リンクをコピペすればすぐに使用できますし、カスタマイズも簡単でした。
[公式：ChakraUIモーダル](https://chakra-ui.com/docs/components/modal)
モーダルのおかげで画面遷移なく、追加の情報を伝えられるようになったかと思います。

## annict api
今回、私の最初の目標は、APIを活用してアプリケーションを作成するということでした。アニメに関するAPIについて色々調べたところannictというサービスに辿り着きました。ドキュメントもしっかりしていて非常に使いやすいAPIでした。
以下コードのようにfetchを用いてannictのapiを叩いています。
```js:AnimeLists/index.jsx
  const handleSearch = async () => {
    const response = await fetch(`https://api.annict.com/v1/works?filter_season=${year}-${season}&per_page=50&sort_watchers_count=desc&access_token=${process.env.NEXT_PUBLIC_ACCESS_TOKEN}`);
    const res = await response.json();
    setLists(res.works)
  };
```
year、seasonの部分には選択した年とシーズンの情報が入るようになります。annictのapiを使用する場合はアクセストークンが必要なので、個人用のアクセストークンを発行し、環境変数として渡すようにしています。NEXT_PUBLIC_ACCESS_TOKENがアクセストークンの環境変数になります。annict apiからJSONで返ってきたレスポンスから必要なデータを取り出し使用しています。どういったレスポンスが返ってくるかなどは以下リンクを見てみてください。
[annict RESTAPI 作品の説明](https://developers.annict.com/docs/rest-api/v1/works)

## いつ放送のアニメなのかで検索
年とシーズンを選択することで、どのクールのアニメなのかを検索できるようにしています。デフォルトでは、現在のクールが選択されるようにしています。useStateを用いたyearの選択肢部分に関しては以下のようなコードで実装しています。

```js:AnimeLists/index.jsx
import { Select } from "@chakra-ui/select";
import { useState } from "react";
import {Years, nowSeasonNum, seasonArray} from "../ConstantArray"
//省略
const [year, setYear] = useState(Years.nowYear);
//省略
<Select w={100} value={year} onChange={e => handleYear(e)}>
  {Years.yearOption.map((theYear) => <option key={theYear}>{theYear}</option>)}
</Select>
```

## アニメの画像
### 画像のCORSエラー
アニメの画像はannict apiのレスポンスに含まれるurlから取得します。このurlは1つのサイトではなく、各アニメのドメインになるので、アニメリスト検索アプリのドメインとは異なります。この時next/imageを使用するとCORSのエラーが起こります。要は別ドメインだとセキュリティ的な制限により、エラーが起こるくらいの説明に留めておきましょう。この制限はhtmlのimgタグであればエラーが発生しません。今回はChakraUIのImageタグを用いて画像を表示しています。他の方法としては後述するnext.config.jsに設定を記載すれば、next/imageを使用してもエラーを回避できると思われます。
参考：[HTML5 における CORS について](https://qiita.com/nanocloudx/items/f49600b705be9d53a9b5)
### 画像の取得エラー時に「no image」を表示
CORSを解消しても、色々なサイトから画像を取得しようとするので、リンク先に画像がそもそも存在しないという可能性もあります。アニメリストを表示する際に画像が存在しない場合、つまりエラーが発生する場合には用意しておいた「no image」の画像を表示するようにしています。この時のコードを以下に示します。onErrorの部分がこのエラー時の処理となりますね。
```js:AnimeArticle/index.jsx
<Image src={theAnime.images.facebook.og_image_url} alt="アニメの画像" width={"70%"} height={"70%"} onError={(e) => e.target.src = 'images/no_image_yoko.jpg'}></Image>
```
ちなみにアニメ詳細のモーダルでは「no image」の画像は不要と考え、エラー時には画像を非表示にしています。


## しょぼいカレンダーapiからキャスト情報を取得
annictを使用すれば、アニメの情報を色々と取得することができましたが、アニメに対するキャストの情報はAPIでは取得できませんでした。個人的にキャスト情報は欲しいと思って色々模索し、しょぼいカレンダーに辿り着きました。annict apiのレスポンスには、しょぼいカレンダーのデータと紐づけるためのIDがあったため、しょぼいカレンダーからキャストの情報だけ取得するように実装しました。
やったこととしてはそれだけなのですが、実はここで色々と詰まることが多く、キャストを表示させるために約2週間かかってしまいました。詰まったところを3つに分けて以下に記載します。

### 再びCORSエラー
今度はしょぼいカレンダーのAPIをfetchを用いて叩く時にCORSのエラーが出ました（ちなみにannictの時には出ませんでした）。このエラーは結果としてnext.config.jsに設定の記述とapiを叩く部分のコードを修正することで解消することができました。
参考：[Rewriting to an external URL
](https://nextjs.org/docs/api-reference/next.config.js/rewrites#rewriting-to-an-external-url)

```js:next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://cal.syoboi.jp/:path*', 
      },
    ]
  },
}

module.exports = nextConfig
```
デプロイしても動くようにproductionモードかどうかでドメインが変わるように記述しています。
```js:AnimeArticle/index.jsx
const domain = process.env.NODE_ENV === "production" ? "https://anime-list-search-nine.vercel.app/api" : "http://localhost:3000/api";
const response = await fetch(`${domain}/db.php?Command=TitleLookup&TID=${tid}`);
```
深い理解はできていなくて、申し訳ないのですが、「cal.syoboi.jp」のドメインだとオリジンが異なり、CORSのエラーが出てしまうところnext.config.jsに設定を書くことで違うドメイン先でも同一ドメインのように使えるといった感じだと思います。私が設定したdevelopmentモードだと、本来の「https://cal.syoboi.jp/db.php~」 を 「http://localhost:3000/api/db.php~」 としてAPIを叩ける。productionモードだと、「https://cal.syoboi.jp/db.php~」 を 「https://anime-list-search-nine.vercel.app/api/db.php~」 としてAPIを叩けるといったイメージになるかと思います。

### レスポンスがXML
しょぼいカレンダーから返ってきたレスポンスはJSONではなくXMLでした。以下のリンク先のようなデータが返ってきます。
[返ってくるレスポンス例](https://cal.syoboi.jp/db.php?Command=TitleLookup&TID=6556)

このXMLをJSONに変換していくために私は「xml-js」というJSのライブラリを使用しました。詳細は別記事に書いたので、こちらを参考にしてください（[別記事](https://zenn.dev/peishim/articles/920aa4d2648548)）。XMLからJSONに変換し、キャスト情報が含まれる「Comment」のデータをテキストで取得するまでのコードを以下に記載します。
```js:AnimeArticle/index.jsx
// xmlで受け取ったテキストデータをjsonテキストに変換→コメントデータ部分のテキストを取り出す
const convert = require('xml-js');
const dataJsonText = convert.xml2json(dataText, {compact: true, spaces: 4});
const dataJson = JSON.parse(dataJsonText)
const dataComment = dataJson.TitleLookupResponse.TitleItems.TitleItem.Comment._text;
```

### 正規表現
Commentのテキストデータは以下のようになります。これは１アニメの情報になります。右にスクロールしていけばわかりますが、非常に長いです。なんのアニメかによってこの長さや内容も多少変わってきますが、ここからキャストの情報を取り出したいので、正規表現で抜き出すことにしました。
```
*リンク\r\n-[[公式 https://onimai.jp/]]\r\n-[[Twitter https://twitter.com/onimai_anime]]\r\n-[[YouTube(TOHO animation) https://www.youtube.com/@TOHOanimation]]\r\n\r\n*メモ\r\n**Twitter・YouTube\r\n-毎週月曜ABEMA配信版に沿ったオーディオコメンタリーを配信\r\n\r\n*スタッフ\r\n:原作:ねことうふ\r\n:掲載誌:月刊ComicRex(一迅社)\r\n:監督:藤井慎吾\r\n:シリーズ構成:横手美智子\r\n:キャラクターデザイン:今村亮\r\n:美術監督:小林雅代\r\n:色彩設計:土居真紀子\r\n:メインアニメーター:みとん、松隈勇樹、内山玄基、Kay Yu\r\n:撮影監督:伏原あかね\r\n:編集:岡祐司\r\n:音響監督:吉田光平\r\n:音響効果:長谷川卓也\r\n:音響制作:ビットグルーヴプロモーション\r\n:音楽:阿知波大輔、桶狭間ありさ\r\n:音楽制作:東宝ミュージック\r\n:プロデュース:EGG FIRM\r\n:制作:スタジオバインド\r\n:製作:「おにまい」製作委員会(東宝、博報堂DYミュージック＆ピクチャーズ、一迅社、AT-X、BS11、ぴあ、ポニーキャニオン、ムービック、TOKYO MX、ビットグルーヴプロモーション、EGG FIRM)\r\n\r\n*オープニングテーマ「アイデン貞貞メルトダウン」\r\n:作詞・作曲・編曲:やしきん\r\n:歌:えなこ feat.P丸様。\r\n\r\n*エンディングテーマ「ひめごと＊クライシスターズ」\r\n:作詞・作曲・編曲:おぐらあすか\r\n:歌:ONIMAI SISTERES(高野麻里佳、石原夏織、金元寿子、津田美波)\r\n\r\n*キャスト\r\n:緒山まひろ:高野麻里佳\r\n:緒山みはり:石原夏織\r\n:穂月かえで:金元寿子\r\n:穂月もみじ:津田美波\r\n:桜花あさひ:優木かな\r\n:室崎みよ:日岡なつみ\r\n\r\n*次回予告イラスト\r\n:#1:松尾祐輔\r\n:#2:米山舞\r\n:#3:Kay Yu\r\n:#4:中村豊\r\n:#5:石田可奈
```
キャスト情報を抜き出すあたりの処理は以下のコードで実行しています。詳細は別記事にて説明しています（[別記事](https://zenn.dev/peishim/articles/2f83d5d15bf628)）。行っていることとしては、キャスト情報がどのようなテキストで返ってくるかをパターンとして把握し、そこを抜き出し、不要な部分を取り除いているといった感じです。あまり正規表現を扱ったことがなかったので、この抽出は結構苦労しました。

```js:AnimeArticle/index.jsx
// キャスト情報の抜き出し（キャスト情報の後にテキストがある場合とない場合の２パターンがある）
let dataCast = []
if (dataComment.match(/\*キャスト[\s\S]*\*/)) {
  dataCast = dataComment.match(/\*キャスト[\s\S]*\*/);
} else {
  dataCast = dataComment.match(/\*キャスト[\s\S]*/);
} 
// 不要な部分を取り除く
const castText = dataCast[0].replace(/\*キャスト\r\n/,"").replace(/\r\n\*/,"")
// 改行を区切りとして配列に変換する
const castArray = castText.split(/\r\n/);
// 配列の空要素を削除
const castDisplayData = castArray.filter(function(s){return s !== "";});
// キャスト情報をuseStateで変更
setCastList(castDisplayData);
```

### 余談：初めはWebスクレイピングでキャスト情報を取得しようとしていた件
こちらはキャスト情報取得に当たって検討したが、結局実行しなかったWebスクレイピングの話なので完全に余談となります。初めはannictの各アニメの作品ページにキャスト情報の記載もあったので、そこからWebスクレイピングでデータを持ってこようと考えました。annictの規約を読んでもサーバーに負荷をかけなければ、特にスクレイピングがダメだという内容もありませんでした。JSでもPuppeteerというWebスクレイピングできる便利なライブラリがあるようなので、これを使ってみよう！と思いました。（参考：[Puppeteerを使って簡単にWebスクレイピングする](https://qiita.com/k1832/items/87a8cf609b4ccf2c6195)）

しかし、私がホスティングサービスとして利用しているVercelのポリシーではWebスクレイピングは良くない行為のようです（参考：[Vercel Fair Use Policy](https://vercel.com/docs/concepts/limits/fair-use-policy)）。ギリギリのところで気づくことができたので、Webスクレイピングを用いた方法は断念することにしました。
:::message alert
Vercelを用いてWebスクレイピングをやろうと考えていた人がいたら気をつけてください！
:::


## favicon
faviconを作成し、設定しました。作成に使用したツールとしては、VectornatorというMacやiPadで使用できるアプリです。VectornatorはAdobeのIllustratorのように使用できる無料アプリとなっており、私はデザイン初心者ですが、かなり色々なことができそうだなという印象でした。無料のアイコンもかなり充実しており、私のfaviconも検索のアイコンと（ナルトっぽい）アニメのアイコンを組み合わせて、色を付けただけです。今後は欲しいアイコンをVectornatorで探すのもありかと思いましたし、もう少し使い方を知っていきたいなと思いました。
参考：[Vectornator使い方](https://fuuno.net/SVG/vec00/vec00.html)

![](https://storage.googleapis.com/zenn-user-upload/f5af2e570d1b-20230210.jpeg =150x)
*作成したfavicon*


## まとめ
シンプルなアプリですが、約１ヶ月で、ある程度満足できるアプリを完成させることができました。コードとしてはリファクタリングできる部分も色々ありそうだなと思っています。また、ざっくりとした内容しか記載していないので、個々の技術を細かく説明した別の記事を書いてもいいかなとも思っています。
もし興味があればアプリやソースコードを使ってみてコメントくださると幸いです！