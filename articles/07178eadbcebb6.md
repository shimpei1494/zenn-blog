---
title: "MyBatis Generatorの導入方法"
emoji: "🗾"
type: "tech"
topics:
  - "java"
  - "mybatis"
published: true
published_at: "2023-08-14 16:39"
---

## 目的
実務でmybatis generatorを使用することになったが、自分で設定などやったことがなかったので、導入してみようといった記事になります。

:::message
MyBatisについて、知っている方は最初の部分は本筋ではないので、読み飛ばしてください。
:::

## MyBatisとは
そもそも、MyBatisとは何なのか？MyBatisはJavaのORマッパーです。要はJavaのアプリケーションとRDBとのやり取りがやりやすくなるように働いてくれるやつです。多くのORマッパーがデータベースのテーブルとオブジェクトを紐づけて管理しますが、MyBatisではテーブルではなくSQLの実行結果に対してORマッピングを行います。少しわかりづらいですが、SQLを割と普通に書いて、その結果をJavaのオブジェクトとして受け取れるようにするところでMyBatisには働いてもらうような感じです（厳密には結構違うかもです）。私が以前触っていたRailsのActiveRecordではユーザーテーブルの名前がpeishimのデータを取得したい場合、以下のように直感的に簡単にコードが書けましたが、MyBatisではちゃんとSQLのSELECT文を書いたりします。
```ruby
User.find_by(name: "peishim")
```
面倒に感じますが、大規模開発においてはSQLを書くことができることがメリットにもなるようです。その辺りあまり詳しくないので、詳細は[こちら](https://camp.trainocate.co.jp/magazine/about-mybatis-spring/)を参考にしてください。

## MyBatis Generatorとは
上で説明したように、全てのDB操作をする箇所でSQLを書くのは面倒です。そんな時に使えるのがMyBatis Generatorです。MyBatis Generatorを使用すれば、DBのテーブルの情報を基にSQLを実行できるファイルを自動生成してくれます。単一のテーブルに対してデータ取得・追加・更新・削除という基本的な処理は自動生成ファイルで達成することができるので、自作する必要がなくなります。2つ以上のテーブルをJOINする場合や受け取るエンティティのカラムを絞ってJavaのオブジェクトに渡したい場合などには自作のMapperが必要になります。実務では、可能であればMyBatis Generatorの自動生成ファイルを使用し、自動生成ではカバーできないSQLの場合は自作のMapperを作成しています。


:::message
ここからがMyBatis Generatorの導入に入っていきます。
導入に困ったら[こちら](https://mybatis.org/generator/index.html)の公式ページを見るのが良いでしょう。
:::
## 環境
- JDK：Amazon Correto17
- ビルドツール：maven
- RDBMS：mysql
- IDE：IntelliJ IDEA
- PC：mac


## MyBatisGeneratorのインストール
私はgradleではなくmavenを使用しているので、pom.xmlのdependenciesタグと同じ階層に以下を記載します。MyBatisGeneratorをインストールするのはこれだけです。

```xml:pom.xml
<plugin>
	<groupId>org.mybatis.generator</groupId>
	<artifactId>mybatis-generator-maven-plugin</artifactId>
	<version>1.4.2</version>
	<configuration>
		<configurationFile>${project.basedir}/src/main/resources/generatorConfig.xml </configurationFile>
		<overwrite>true</overwrite>
	</configuration>
</plugin>
```
configurationFileのタグについてはなくても問題なく動作したので、デフォルトでこの場所のこのファイルを見るようになっているのかもしれません。とりあえずgeneratorConfig.xmlというファイルにgeneratorの設定が書いてあるよということを記載しています。
気を付けたいポイントとしてはoverwriteをtrueにしないと、generatorを実行するたびに、新しくMapperのファイルが生成されてしまいます。新しいテーブルが追加されたりした時に、既に生成済みのファイルがもう一個できたら面倒ですよね？なので、なので同一テーブルについては上書きするような設定にするため、overwriteを記述しています。
:::message alert
当然pom.xmlにはMyBatisGenerator以外に、JDBCドライバとMyBatisを記載し、データベースと接続するための設定をapplication.yml等に記載する必要があります。これらができた上で、MyBatis Generatorを使用していくという前提で話をしています。
:::

## generatorConfig.xml（MyBatis Generatorの設定）
resources配下にmybatis generatorを実行するための設定ファイルであるgeneratorConfig.xmlを作成します。
完成したファイルの中身を以下に記載します。長いので折りたたみにしますが、部分部分で解説を行なっていきます。
:::details generatorConfig.xml
```xml:generatorConfig.xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE generatorConfiguration PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN" "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd" >
<generatorConfiguration>
    <classPathEntry
            location="/Users/peishim/.m2/repository/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar" />
    <context id="context1" targetRuntime="MyBatis3">

        <!--  MapperにMapperアノテーションを付与   -->
        <plugin type="org.mybatis.generator.plugins.MapperAnnotationPlugin"/>
        <!--  equalsおよびhashCodeを自動生成      -->
        <plugin type="org.mybatis.generator.plugins.EqualsHashCodePlugin"/>
        <!-- マッパxmlファイルを生成時に上書きするためのプラグイン -->
        <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>

        <!-- コメント生成の抑制 -->
        <commentGenerator>
            <property name="suppressDate" value="true" />
            <property name="addRemarkComments" value="false" />
        </commentGenerator>

        <!--     JDBCの設定 -->
        <jdbcConnection
                driverClass="com.mysql.cj.jdbc.Driver"
                connectionURL="jdbc:mysql://localhost:3306/db"
                userId="user"
                password="password"
        />

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!--  Entityの生成場所 -->
        <javaModelGenerator
                targetPackage="com.example.bulletin.board.entity.gen"
                targetProject="src/main/java/"
        />
        <!--  Mapper(xml)の生成場所 -->
        <sqlMapGenerator
            targetPackage="com.example.bulletin.board.mapper.gen"
            targetProject="src/main/resources/"
        />

        <!--  Mapper(java)の生成場所 -->
        <javaClientGenerator
                targetPackage="com.example.bulletin.board.mapper.gen"
                targetProject="src/main/java/"
                type="XMLMAPPER"
        />

        <!--  生成対象のテーブル -->
        <table tableName="post"  modelType="hierarchical" />
    </context>
</generatorConfiguration>
```
:::


```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE generatorConfiguration PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN" "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd" >
<generatorConfiguration>
</generatorConfiguration>
```
→初めのお決まり文なので、とりあえずコピペしとけば大丈夫です。

```xml
    <classPathEntry
            location="/Users/peishim/.m2/repository/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar" />
```
→この部分はどのjdbcドライバーをインストールしたかで変わると思います。mavenでインストールしたExternal Librariesディレクトリ内のmysqlのjdbcコネクタのファイルのpathをコピーして貼ったら正常に動作しました。

```xml
<context id="context1" targetRuntime="MyBatis3">
```
→MyBatis GeneratorはデフォルトでMyBatis3DynamicSqlというモードになっています（詳しくは[こちら](https://mybatis.org/generator/quickstart.html)）。詳しくはわかっていませんが、MyBatis3DynamicSqlモードではDynamicSqlSupportクラスが出力され、xmlのmapperファイルが生成されません。今回はxmlファイルのMapperを生成したかったので、その場合はtargetRuntimeにMyBatis3を指定する必要があります。

```xml
        <!--  MapperにMapperアノテーションを付与   -->
        <plugin type="org.mybatis.generator.plugins.MapperAnnotationPlugin"/>
        <!--  equalsおよびhashCodeを自動生成      -->
        <plugin type="org.mybatis.generator.plugins.EqualsHashCodePlugin"/>
```
→EqualsHashCodePluginについては必要かまで理解していませんが、モデルのequalsおよびhashCodeを自動生成するように一応追加しています。MapperAnnotationPluginに関しては、このプラグインを入れないと~Mapper.javaファイルにMapperファイルが記載されない状態で自動生成されます。DIする時にMapperがBean登録されていないと困るので、入れておきます。

```xml
        <!-- マッパxmlファイルを生成時に上書きするためのプラグイン -->
        <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>
```
→自分で検証は行なっていないが、[こちら](https://www.nextdoorwith.info/wp/se/system-dev-mybatis-generator/)の記事より、pom.xmlのoverwriteの記述だけだと、ファイル作成時にマージ（変更箇所が追加）されてしまうらしいので、追加しました。

```xml
        <!-- コメント生成の抑制 -->
        <commentGenerator>
            <property name="suppressDate" value="true" />
            <property name="addRemarkComments" value="false" />
        </commentGenerator>
```
→自動生成時のコメント内容に関するオプションです。suppressDateをtrueにしないと、自動生成ファイルのコメントに生成日時が記載され、generatorを実行するたびにテーブル構造に差分がなくても、必ずファイルに差分が発生してしまいます。addRemarkCommentsについてはfalse に設定すると、コメントにテーブルやカラムの説明情報は含まれないとのことでしたが、これは現状どちらでも良い気がしています。

```xml
        <!--     JDBCの設定 -->
        <jdbcConnection
                driverClass="com.mysql.cj.jdbc.Driver"
                connectionURL="jdbc:mysql://localhost:3306/db"
                userId="user"
                password="password"
        />
```
→自分で設定したデータベースへの接続情報を書きましょう。

```xml
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>
```
→javaTypeResolverというのはデータベースの情報からjavaの型に変換する定義の設定のようです。forceBigDecimalsをtrueにすることで、数値が必ずBigDecimal型に変換されるようです。実務で使用していたコードを参考にfalseに設定してみましたが、こうすると何がいいかまでは調べきれていません。とりあえずBicDecimalにしないと少数の値の計算が正しく行えない場合があるが、全てBigDecimal型に格納されるより、int型とかになってくれた方が扱いやすいみたいな感じだと推測しています。他にもjavaTypeResolverで日付データの受け渡しについての設定もでき、今回のように特に設定しなければ、デフォルトでDATE・TIME・TIMESTAMPのデータはjava.util.Date型に変換されるようです。
参照：https://mybatis.org/generator/configreference/javaTypeResolver.html

```xml
        <!--  Entityの生成場所 -->
        <javaModelGenerator
                targetPackage="com.example.bulletin.board.entity.gen"
                targetProject="src/main/java/"
        />
        <!--  Mapper(xml)の生成場所 -->
        <sqlMapGenerator
            targetPackage="com.example.bulletin.board.mapper.gen"
            targetProject="src/main/resources/"
        />

        <!--  Mapper(java)の生成場所 -->
        <javaClientGenerator
                targetPackage="com.example.bulletin.board.mapper.gen"
                targetProject="src/main/java/"
                type="XMLMAPPER"
        />
```
→コメント記載の通りそれぞれの自動生成ファイルが保存される場所を記載しています。Mapper(xml)とMapper(java)のtargetPackageは同じに設定しておくと自動で紐づくらしいので、この名前を変える場合は別途紐付けの設定を記載する必要が出てきそうですね。

```xml
        <!--  生成対象のテーブル -->
        <table tableName="post"  modelType="hierarchical" />
```
→コメント記載の通り、生成したいテーブルを記載します。この場合はpostテーブルに関するMapperに関するファイルを自動生成します。modelTypeについては実務を参考にhierarchicalを設定しています。これを設定しないとKeyファイルが作られないなどの違いが出てきますが、まだどのように使い分ければいいかはよくわかっていません。chatGPTに聞くとhierarchicalを設定することで、親子関係を持つテーブル間の関連性を反映するためにhasOneや hasManyの関連メソッドが生成されるみたいです。
今回の設定でpostテーブルに関する自動生成を行うと以下の5ファイルが生成されます。（ざっくりと用途も説明）
- Post.java →エンティティクラス
- Post.Example.java　→検索する場合などの条件設定のためのクラス
- PostKey.java →プライマリーキーを設定するクラス
- PostMapper.java →xmlのMappperに紐付き、このクラスを用いてデータ操作を実行
- PostMapper.xml →ここにSQLが記載される

## MyBatis Generatorの実行方法
自分の作成したプロジェクトのディレクトリで以下のコマンドを実行することで、自動生成ファイルを生成することができます。
```
mvn mybatis-generator:generate
```
毎回コマンド打つのは面倒だと思うので、私はIntelliJの画面右上の色々実行できるところで以下のように、設定しておけば簡単に実行することができます。
![](https://storage.googleapis.com/zenn-user-upload/9b94b4b1c79b-20230814.png =300x)
![](https://storage.googleapis.com/zenn-user-upload/2c8145b3f6ed-20230814.png =700x)

## 最後に
これでDB操作を自分でMapper作らずともできるようになった訳ですが、MyBatisGeneratorを設定するだけでも結構苦労した上、どういう設定にしたら便利なのか未だにわかってない部分が多いです。この記事が初学者の参考になれば幸いですし、私もわからない部分が多いので、いろいろアドバイスいただけるとありがたいです！