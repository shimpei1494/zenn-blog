---
title: "【Spring Security】SecurityFilterChainとカスタムUserDetailsService"
emoji: "🗿"
type: "tech"
topics:
  - "java"
  - "springboot"
  - "springsecurity"
published: true
published_at: "2023-08-27 00:06"
---

## 初めに
前回以下の記事でSpring Securityを用いた認証の流れを確認しました。
https://zenn.dev/peishim/articles/6946f72e15affa
今回はSecurityFilterChainを作成して色々機能を追加していきましょう。加えて、前回の記事では認証する際にUserDetailsManagerを使用しましたが、今回はUserDetailsServiceを実装した自作クラスを使用してきます。原理までしっかりわかってない部分もありますが、使い方をしっかりまとめていこうと思います。

## 参考URL
この記事は主に以下のyoutubeを参考に自分で実装し、自分なりの解説を加えたような内容となっています。
[Spring Fest 2023 第二枠目（13:00 ~ 14:50）](https://www.youtube.com/watch?v=lg1ycXsDbEI&t=3941s)

## この記事でやること
ざっくりとですが、アクセス制限など設定したログイン認証ができるようにしていきます。

## 自作UserDetailsService
UserDetailsServiceを実装したUserDetailsServiceImplクラスを作成していきます。

:::details UserDetailsServiceImplクラスの全文
```java:UserDetailsServiceImpl.java
package com.example.bulletin.board.service;

import com.example.bulletin.board.dao.AccountDao;
import com.example.bulletin.board.entity.gen.Account;
import org.springframework.context.event.EventListener;
import org.springframework.security.authentication.event.AuthenticationFailureBadCredentialsEvent;
import org.springframework.security.authentication.event.AuthenticationSuccessEvent;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Date;

@Service
@Transactional
public class UserDetailsServiceImpl implements UserDetailsService {
    private final AccountDao accountDao;

    //何回以上失敗したらロックするか
    int lockingBoundaries = 3;

    public UserDetailsServiceImpl(AccountDao accountDao) {
        this.accountDao = accountDao;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Account account = accountDao.findByUserName(username)
                .orElseThrow(() -> new UsernameNotFoundException(username));
        return User.withUsername(account.getName())
                .password(account.getPassword())
                // 役割が設定（rolesも引数変わるが権限に使える、権限はStringをカンマ区切りで複数指定できる）
                .authorities(account.getAuthority())
                // 無効なアカウントはログインさせない(del_flgが1の場合、論理削除）
                .disabled("1".equals(account.getDelFlg()))
                // アカウントが有効期限が今日より過去の場合はtrueにして期限切れにする
                .accountExpired(account.getExpiration().before(new Date()))
                // パスワードの有効期限切れ
                .credentialsExpired(account.getPasswordExpiration().before(new Date()))
                // ログイン失敗回数3回以上でロック
                .accountLocked(account.getLoginFailureCount() >= lockingBoundaries)
                .build();
    }

    // ログイン失敗時のハンドラ
    @EventListener
    public void loginFailureHandle(AuthenticationFailureBadCredentialsEvent event) {
        String username = event.getAuthentication().getName();
        accountDao.incrementLoginFailureCount(username);
    }

    // ログイン成功時のハンドラ
    @EventListener
    public void loginSuccessHandle(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();
        // ログイン失敗回数を0にする
        accountDao.resetLoginFailureCount(username);
    }

}

```
:::

### loadUserByUsernameメソッド
このクラスではloadUserByUsernameメソッドをオーバーライドする必要があります。ログインの操作が画面上で行われるとこのメソッドが使われ、このメソッドで返すUserDetailsを基に入力されたユーザーがログインしてもいいかどうか等を判別します。
```java
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
// DBからユーザー名に一致するデータを取得し、accountに詰める
Account account = accountDao.findByUserName(username)
	.orElseThrow(() -> new UsernameNotFoundException(username));
return User.withUsername(account.getName())
	// パスワード
	.password(account.getPassword())
	// 役割を設定（rolesも引数の指定方法が変わるが、権限に使える、権限はStringをカンマ区切りで複数指定できる）
	.authorities(account.getAuthority())
	// 無効なアカウントはログインさせない(del_flgが1の場合、論理削除）
	.disabled("1".equals(account.getDelFlg()))
	// アカウントが有効期限が今日より過去の場合はtrueにして期限切れにする
	.accountExpired(account.getExpiration().before(new Date()))
	// パスワードの有効期限が今日より過去の場合はtrueにして期限切れにする
	.credentialsExpired(account.getPasswordExpiration().before(new Date()))
	// ログイン失敗回数lockingBoundaries回以上でロックする
	.accountLocked(account.getLoginFailureCount() >= lockingBoundaries)
	// 上で設定した情報をもったUserDetailsを作成
	.build();
}
```
:::message
ここで要は何をやっているかというと、データベースから対象ユーザーの情報を取得（account）→その情報から認証に必要なパスワードや有効期限などの情報をもったUserDetailsを作成し、作成したUserDetails（ユーザー名に合致したDBの情報）と入力されたパスワードを用いて認証が実行されるということです。
:::
今回はUserDetailsは自作せずに既存のUserクラスを用いて作成しています。どのような設定をしているかはコメントアウトに記載しているので見ていただければと思います。最低限パスワードを詰めるところだけあればログイン認証はできるので、あとは必要に応じて削除・追加してください。

一応自分が作ったアカウントを管理するaccountテーブルがどのような構造になっているかを以下の折りたたみ内に記載しておきます。
:::details accountテーブルの構造（phpmyadminで表示）
![](https://storage.googleapis.com/zenn-user-upload/8fc519ce56b9-20230826.png)
:::

### ログイン成功・失敗時に処理を行わせる
以下のコードの部分です。
```java
// ログイン失敗時のハンドラ
@EventListener
public void loginFailureHandle(AuthenticationFailureBadCredentialsEvent event) {
	String username = event.getAuthentication().getName();
	accountDao.incrementLoginFailureCount(username);
}

// ログイン成功時のハンドラ
@EventListener
public void loginSuccessHandle(AuthenticationSuccessEvent event) {
	String username = event.getAuthentication().getName();
	// ログイン失敗回数を0にする
	accountDao.resetLoginFailureCount(username);
}
```
今回はログイン失敗時に失敗回数を1プラスした値をDBに保存するという処理、ログイン成功時に失敗回数を0にするという処理を追加したかったため上記コードを作成しました。これはログイン失敗を何回繰り返したらアカウントをロックするという機能をつけるためですね。大事なことはDIコンテナ内で@EventListenerをメソッドに付与すること、メソッドの引数にハンドリングしたい認証イベントクラスを指定することの2つです。今回はUserDetailsServiceImplクラス内に記載しましたが、他のクラスでも＠Componentなどがついていれば大丈夫です。今回は成功・失敗時のイベントクラスしか使用しませんでしたが、他にもまだ種類は色々ありそうです。

## SecurityFilterChain
SecurityFilterChainはフィルタ処理を行うものです。通常Securityのconfigに記載してBean登録する形になるかと思います。今回はユーザー認証に関する内容を紹介するため記載しませんが、csrf対策に関する設定などもいじることができます。

:::details SecurityConfig.java全文
```java:SecurityConfig.java
package com.example.bulletin.board.config;

import org.springframework.boot.autoconfigure.security.servlet.PathRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig{

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((requests) -> requests
                    .requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
                    .requestMatchers("/login", "/").permitAll()
		    .requestMatchers("/admin/**").hasAuthority("ROLE_ADMIN")
                    .anyRequest().authenticated()
            )
            .formLogin((form) -> form
                    // ログインページへのパスを指定→コントローラーにもGET、/loginでの処理を記載する必要がある
                    .loginPage("/login")
                    // ログイン成功時に表示される画面へのパス
                    .defaultSuccessUrl("/")
                    .permitAll()
            )
            .logout((logout) -> logout.permitAll());

        return http.build();
    }
}
```
:::
前回の記事でも解説した部分は省略します。Spring Security6.0以前とは書き方が色々変わっている部分もあるので、ネットで記事を探す時は注意してください（今回はバージョン6.1.2です）。

### リクエストの制御
このページはログインしてないと見れないとかADMINの権限がないと見れないとかそういった設定を行います。
以下のコードの部分です。
```java
.authorizeHttpRequests((requests) -> requests
    .requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
    .requestMatchers("/login", "/").permitAll()
    .requestMatchers("/admin/**").hasAuthority("ROLE_ADMIN")
    .anyRequest().authenticated()
)
```

以下は/loginと/というパスで到達するページは認証なしでも見れます。permitALL→誰でも見れるよ！ってことですね。以下のようにカンマ区切りで複数記載することもできますし、requestMatchersを複数作成して"/login"と"/"をそれぞれ指定する形でも大丈夫です。
```java
.requestMatchers("/login", "/").permitAll()
```

順番前後しちゃいましたが、以下は静的リソースは誰でも見れますよということです。初めは.requestMatchers("/css/**").permitAll()みたいなコードを複数書いていたのですが、この1行でいけるみたいです。これがないと、誰でも見れるページだろうが、cssやjsなどはユーザー認証が必要でページに適用されないことになります。

```java
.requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
```
以下は/admin/** →つまり/admin/配下のページは、認証OKかつADMINの権限をもった人しか見れませんという制限です。権限がない状態でに遷移しようとすると403エラーになります。
```java
.requestMatchers("/admin/**").hasAuthority("ROLE_ADMIN")
```
上記のコードは以下のコードでも同じ意味になります。
```java
.requestMatchers("/admin/**").hasRole("ADMIN")
```
データベースにはROLE_ADMINといったデータが格納されていて、UserDetailsServiceImplの以下のコードでUserDetailsに格納されています。この情報を基にユーザーがどのような権限をもっているかが確認されるわけですね。
```java
.authorities(account.getAuthority())
```

元のコードに戻り、以下のコードではpermitAllなど上記で指定したページ以外は全て認証を必要とするということです。
```java
.anyRequest().authenticated()
```

リクエストのパスの指定方法などは[こちらの記事](https://b1san-blog.com/post/spring/spring-sec-auth/#%E3%83%91%E3%82%B9%E3%81%AE%E8%A8%AD%E5%AE%9A)がわかりやすかったです。

### メソッドセキュリティを用いたリクエストの制御（余談）
SecurityFilterChainを用いてアクセス制限をできるようになりましたが、アプリが複雑になりページ数が多くなるとアクセス制限をこのファイルだけで管理するのが難しくなる場合があります。その場合にはメソッドセキュリティというものを用いてコントローラでリクエストハンドラを作成するごとに認証の有無や権限について設定することができます。
以下の折りたたみに簡単に使い方を記載しておきます。
:::details メソッドセキュリティの使い方
セキュリティ構成クラスに@EnableMethodSecurityをつけます。以下のようなイメージです。
```java:SecurityConfig.java
@Configuration
@EnableMethodSecurity
public SecurityConfig {
```
Config内のSecurityFilterChainでは全て認証を必要とするような設定にしておくくらいで、アクセス制限に関する細かい設定はしなくても大丈夫です。

あとはコントローラのメソッドに@PreAuthorize(値)メソッドをつけて、値の部分にpermitAllなど指定するだけです。以下のようなイメージになります。
```java:コントローラ
@GetMapping("/")
@PreAuthoriz("permitAll")
public ModelAndView index(ModelAndView mav) {
```

:::

### ログインに関する設定
以下のようにformLoginを記載することで、フォーム認証が有効になるようです。
```java
.formLogin((form) -> form
    // ログインページへのパスを指定→コントローラーにもGET、/loginでの処理を記載する必要がある
    .loginPage("/login")
    // ログイン成功時に表示される画面へのパス
    .defaultSuccessUrl("/")
    .permitAll()
)
```
.loginPage("/login")のように記載することで、自作のログインページを使用することができます。これを使用する場合はもちろん自分で/loginにリクエストを処理できるコントローラとビューを作成する必要があります。本筋ではありませんが、私が作成したコントローラとビューの一部を参考に折りたたみに記載します。
:::details コントローラとビューの参考
```java:AuthController.java
package com.example.bulletin.board.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.ModelAndView;

@Controller
public class AuthController {

    @GetMapping("/login")
    public ModelAndView login(ModelAndView mav,
                              @RequestParam(value = "error", required = false) String error) {
        mav.setViewName("login");
        if (error != null) {
            mav.addObject("msg", "ログインできませんでした");
        } else {
            mav.addObject("msg", "ユーザー名とパスワードを入力");
        }
        return mav;
    }
}
```

```java:login.htmlの一部
<form method="post" th:action="@{/login}">
<p th:text="${msg}"></p>
<p th:if="${param.error != null and session.containsKey('SPRING_SECURITY_LAST_EXCEPTION')}"
th:text="${session.SPRING_SECURITY_LAST_EXCEPTION.message}"></p>
<label for="username" class="visually-hidden">ユーザー名</label>
<input type="text" id="username" name="username" class="form-control" placeholder="ユーザー名">
<label for="password" class="visually-hidden">パスワード</label>
<input type="password" id="password" name="password" class="form-control" placeholder="パスワード">
<div class="d-flex justify-content-center">
  <button type="submit" class="btn btn-primary mt-3">ログイン</button>
</div>
</form>
```
大事なことはhtmlでinputタグでusernameとpasswordというname属性を指定して、ログインのリクエストを送ることですね。
formのリクエストはth:action="@{/login}"のようにth:actionを設定することも忘れないようにしてください。th:actionを記載することでPOSTリクエストを送る際にcsrfを自動で埋め込んでくれるようになるのですが、actionで記載した場合には自動で埋め込んてくれずcsrfのエラーが発生します。

あとはログインに失敗した場合はセッションのSPRING_SECURITY_LAST_EXCEPTIONにエラーメッセージだったりが格納されるらしく、それを表示するようにしています。メッセージも日本語で表示されていましたが、エラーメッセージのカスタマイズはプロパティファイルで以下のように変更できそうです。
```
AbstractUserDetailsAuthenticationProvider.locked=アカウントはロックされています。
AbstractUserDetailsAuthenticationProvider.disabled=アカウントは使用できません。
AbstractUserDetailsAuthenticationProvider.expired=アカウントの有効期限が切れています。
AbstractUserDetailsAuthenticationProvider.credentialsExpired=パスワードの有効期限が切れています。
AbstractUserDetailsAuthenticationProvider.badCredentials=ログインIDまはたパスワードが間違っています。
```
:::

### ログアウトの制御
以下でログアウト機能を有効にして、permitAllで匿名ユーザー含む全てのユーザーに対してログアウトとログアウト成功時に遷移するパスへのアクセス権が付与されるようです。
```java
.logout((logout) -> logout.permitAll());
```

## （余談）タイムアウトについて
ここまでログイン認証でカスタマイズできることがだいぶ増えたと思いますが、ログインしたらどれくらい経ったらログアウト状態になるかを調べました。

Spring Bootではセッションのタイムアウトの時間はデフォルトで30分になっています。これは画面遷移せずに30分が経過したらセッションがタイムアウトになり、つまりログアウト状態になるということです。画面遷移を行った場合はこのタイムはリセットされます。

このタイムアウトの時間を変更するのに以下記事を参考にさせていただきました。
https://qiita.com/_Hammer0724/items/f3afd40754f6ba587768

私はやりやすい方法として以下のようにapplication.ymlに記載してタイムアウトの時間を60秒に変更できました。
```yml:application.yml
server:
  servlet:
    session:
      timeout: 60
```

時間指定の注意点は以下のとおりです。
- 数字だけなら秒数になる
- 1mなら1分、60sなら60秒という指定もできる
- 1分未満の値だと1分になる
- ただし、0やマイナスの値だとタイムアウト時間は無限になる

Spring BootのセッションとCookieについては以下の記事が参考になります。
色々設定を変えるためのプロパティについてもまとまっています。
https://b1san-blog.com/post/spring/spring-session/

## 最後に
SpringSecurityはまだまだいろんなことができると思いますが、とりあえずログインページ作って、基本的なログイン認証したり、アクセス制限したりができるようになりました。仕組み的なところへの理解を深めていこうと思うとなかなかに奥が深いですが、初学者の方に参考になれば幸いです。