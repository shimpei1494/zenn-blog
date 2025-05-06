---
title: "DockerでNext.jsの環境構築"
emoji: "🐳"
type: "tech"
topics:
  - "docker"
  - "nextjs"
  - "nodejs"
published: true
published_at: "2023-03-18 09:51"
---

## 実施したこと

今回はDockerでNext.jsの環境構築を行いました。その内容をなるべくDocker初心者でもわかりやすいように解説していきたいと思います。そこまで難しい内容ではないと思うので、今回作成したコードは気軽に使ってもらえればと思います！

:::message
2023/5/12にcompose.ymlのコードを追記し、説明を最後の方につけました。
:::

## 今回の背景（興味がない人は飛ばしてください）
普段Next.jsを個人開発で使用する際は、Dockerは使用していませんでした。というのも、voltaというnode.jsのバージョン管理ツールを使用しているので、アプリごとでnodeのバージョンが異なっても、ピンを刺せば勝手にnodeのバージョンが切り替わるから、とりあえずDocker使わなくてもいいかなと思っていました。まだ同じバージョンのnodeしか使ったことがないので、この機能を実感はしていないのですが、このあたり興味がある方は以下リンクを参考にしてみてください。
[Voltaを解説！手間のかからないJavaScriptツールマネージャ](https://kobatech-blog.com/volta/)

今回に関しては、実務でNext.jsを用いたアプリ作成に挑んでいるのですが、他のプロジェクトでDockerを使用していないプロジェクトがありました。voltaも使用していないため、もし環境を共有したいとなった時には自分の作ったアプリの環境をDockerで共有できた方が圧倒的に使い勝手がいいだろうと思って、DockerでNext.jsの環境構築をすることになりました。バックエンドの環境もDockerで作っているので、それなら全部Dockerの方がいいですよね。

:::message
なお、今回はローカル環境でNext.jsのアプリケーションを作成した後に、Dockerのコンテナを作成し、共有しやすくしたといった感じなので、アプリケーション作成の部分には触れていない点はご了承ください。
:::

## ディレクトリ構成
アプリのディレクトリ構成は以下のような感じです。
- compose.yml
- Dockerfile
- package.json
- package-lock.json
- srcディレクトリ
- publicディレクトリ
- その他

package.jsonやpackage-lock.jsonにはどのようなライブラリがインストールされているかという情報が記載されており、これらは今回修正することは特にありません。今回Dockerで環境構築に使用するためにコードを書く必要のあるファイルはcompose.ymlとDockerfileです。ディレクトリ構成は色々アレンジ可能だと思いますが、この後のコードをそのままでコンテナを起動したい場合にはcompose.yml、Dockerfile、package.jsonの3つは同じディレクトリに配置させましょう。

## コード記述
compose.ymlとDockerfileを以下のように記述してください。compose.ymlのnetworksの部分は他のコンテナのアプリと通信を行うために書いたものなので、Next.jsだけのアプリであれば、丸々削除してください。

```yml:compose.yml
services:
  app:
    build:
      context: .
    ports:
      - "12000:3000"
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    command: sh -c "npm run dev"
    networks:
      - default
      - snowboard_default #snowboadコンテナのネットワーク名
volumes:
  node_modules:
networks:
  snowboard_default:
    external: true
```

```:Dockerfile
FROM node:18.12-alpine

WORKDIR /app/

COPY ./package.json ./
RUN npm install
```

あとはターミナルでcompose.ymlが存在するディレクトリに移動し、以下コマンドを実行することによって、コンテナを起動することができます。
```
$ docker compose build
$ docker compose up
```
buildの方のコマンドは、イメージ（今回はDockerfileの内容）を基にコンテナ作成しますよって感じで、upの方のコマンドはcompose.ymlの内容でコンテナ起動しますよって感じのイメージになります！
:::message alert
既に、Next.jsのアプリが作成されていないとコンテナは起動できないので、気をつけてください。
:::

## コードの解説
少しだけ解説をしていきます。
### compose.ymlの解説
compose.ymlではこういう設定でコンテナを起動しますよということが書かれています。

```yml
build:
  context: .
```
compose.ymlと同じディレクトリにあるDockerfileでコンテナをビルドするといった内容です。なので、この記述をそのまま使用する場合にはcompose.ymlとDockerfileを同じディレクトリに配置する必要があります。

```yml
ports:
  - "12000:3000"
```
ここの左の数字が12000だとコンテナ起動時に http://localhost:12000 で作成したアプリを確認することができます。今回に関しては他のコンテナで使用しているポート番号と被らないようになんとなく12000にしているだけです。ポート番号が被っているとコンテナを起動することができません。3000ポートはよく使われるので避けた方がいいかなと自分は思っています。

```yml
command: sh -c "npm run dev"
```

通常nextjsアプリをローカルで起動する時にnpm run devってやると思います。まさにそれが書いてあります。先ほどdocker compose upでコンテナを起動するという話をしたかと思いますが、今回の場合は「コンテナを起動する」＝「コンテナ内でnpm run dev」を実行するということと同義なのです。 コンテナの起動に失敗する場合には、npm run devがうまく動作しない原因が何かあると考えると良いと思います。先ほどNextjsアプリが作成していないとコンテナ起動しないという注意も記載しましたが、それもアプリ作成していないのにnpm run devしても動作しないといった話です。

```yml
volumes:
  - .:/app
```
この記載により、「現在のディレクトリ」と「コンテナ内の/app/ディレクトリ」が同期されます。イメージ的には、VScodeなどのエディタで自分のPC内のコードを修正したら、その内容がコンテナ内にリアルタイムでコピーされる→アプリ自体はコピーされたコンテナ内のコードを基に動作しているといった感じです。

### Dockerfileの解説
Dockerfileではコンテナのイメージをビルドします。
先ほどのコンテナ起動のコマンドだとdocker compose buildした時にDockerfileを基にコンテナイメージが作られます。
```
FROM node:18.12-alpine
```
DockerHubにあるnodeのイメージを持ってきます。今回はnodeのバージョン18.12にしたのですが、ここは状況に応じて、適宜好きなバージョンにしてください。

```
WORKDIR /app/
```
コンテナ内のディレクトリは/app/に移動します。

```
COPY ./package.json ./
```
現在のディレクトリのpackage.jsonをコンテナ内のディレクトリ（現在は/appディレクトリ）にコピーします。つまり、コピー後のコンテナ内は/app/package.jsonといった感じに配置されます。私はこの記述をするのを忘れて、初めエラーが出てしまっていました。というのもcompose.ymlで現在のディレクトリと同期されていると思っていましたが、build時はcompose.ymlの内容は関係ないので、コンテナ内にpackge.jsonは存在しないのです。compose.ymlの内容が反映されるのはdocker compose upする時だということに気をつけましょう。

```
RUN npm install
```
コンテナ内のpackage.jsonを基に必要なライブラリをインストールします。これでpackage.jsonにnextjsに必要なライブラリがしっかり記載されていれば、nextjsを起動するための環境を持ったコンテナが作成されるといった感じです。

### 2023/5/12追記
compose.ymlのコードに関して、以下に示すnode_modules関連の部分を追記しました。
```yml
    volumes:
      - .:/app
      - node_modules:/app/node_modules
```

```yml
volumes:
  node_modules:
```
理由としては私のコードをgit cloneした他の人がdocker compose upでNext.jsアプリを起動しようとしたらnode_modulesが存在せず、起動できなかったからです。

node_modulesはgit管理しないため、git cloneしてもローカルのコードには存在しません。ビルド時にnpm installしており、そのタイミングでコンテナ内にはnode_modulesは作られますが、docker compose upで起動するときにnode_modulesが存在しないローカルのファイルがコンテナ内にバインドマウントで同期することによって、コンテナ内のnode_modulesも消えるようです。

node_modulesを消えないようにするため、Dockerのボリュームを用いるように追記では変更しました。バインドマウントを使用していても、ボリュームを使用していればボリュームの方が優先されてnode_modulesが残るようです。

以下リンクを参考にしたので、詳しくは以下記事をご確認ください。
https://qiita.com/P-man_Brown/items/c2a69d943853cb381fbe

:::message
私は自分でNextアプリを作成したので、アプリ内にnode_modulesが作成されていたため、node_modulesをボリュームに保存しなくても不具合は生じていませんでした。
:::

## 最後に
なんとなくDockerでNext.jsアプリを動かす環境を作るイメージが湧いたでしょうか？
もしわかりづらいところや間違っているところがあればコメントにて教えていただけると幸いです。

## 参考リンク
https://qiita.com/higemegane1992/items/defd193f4c8752ca9996
https://zenn.dev/temple_c_tech/articles/setup-next-on-docker
https://qiita.com/suzuki0430/items/fcf57968365d15419601