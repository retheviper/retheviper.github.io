---
title: "REST APIでのログインのためには"
date: 2020-05-30
categories: 
  - spring
photos:
- /assets/images/sideimage/spring_logo.jpg
tags:
  - spring
  - rest api
---

新しい案件が始まって、AngularとSpring bootによるSPA(Single Page Application)を作ることとなりましたが、まだ要件定義の段階で実装まではしばらく時間が残っています。いわゆる上流工程にはあまり経験がないので日々奮闘中ではありますが、仕事が終わった後の時間には少しづつ、練習をかねて自作アプリケーションを作っています。

Spring BootでREST APIを作ることにはある程度慣れてはいるものの、まだ認証/認可に対してあまり知識がないので、自作アプリではわざとSpring Securityを入れることにしました。しかし、Spring Securityの記事をいくつか読んでロールによってアクセスできるURLに制限をかけたりしているのだなと理解した後に、いざ自作アプリケーションに導入してみようとしたら、問題ができました。いつも通りREST APIで実装していたのですが、自分が参考にしていたSpring Securityのコードはほぼ昔ながらのSpring MVCパターンのためのものだったのです。

なので、今回のポストはSpring MVCパターンとREST APIのログインするため必要となるのが何かについて簡単に書いてみようと思います。(実際のログインの方法は今後のポストを予定しています)``

## コントローラの実装

まずMVCパターンとREST APIのコントローラをどのように実装するかからみていきましょう。

### MVCパターンの場合

今も多くの研修で行われているSpringの研修は、普通のMVCパターンとJSPによるものが多いのではないか、と思います。例えば`@Controller`アノテーションをつけたコントローラクラスに、`@RequestMapping`アノテーションをつけたメソッドを書いて、そのメソッドでは`Model`もしくは`ModelAndView`などのクラスを使ってデータやビュー(主にJSPファイルのパス)を書いていくような形ですね。実際は`JSPを使う=MVCパターン`というわけでもないですが、多くのSpring MVCパターンのプロジェクトがJSPを使っているのでまずはこのようなレガシーなものをMVCパターンと呼ばせていただきます。

ただ文章で表現するよりは、コードを持って書いた方がわかりやすいと思います。例えば以下のようなものです。`/home`というURLに接続すると、サーバの時間を返す簡単な例題ですね。ModelAndViewを使うとしても、Modelにデータとビューを共につめてるだけでやっていることとしてはあまり変わらないと思います。

```java
@Controller
public class HomeController {
    
    // homeというJSPファイルとサーバの時間を表示する
    @RequestMapping(value = "/home", method = RequestMethod.GET)
    public String home(Model model) {
        Date date = new Date();
        model.addAttribute("serverTime", date);
        return "home";
    }
}
```

### REST APIの場合

自分の場合も、やはり最初はSpring MVCパターンとJSPから学んだのでこのような書き方に慣れていましたが、入社後にはSpring BootとREST APIというものに出会い、コードの書き方が少し変わりました。最近のトレンドだとやはりJSPよりもAngular/React/VueなどのJavaScriptフレームワークを使って作成する場合が多いので、JSONの形としてデータだけを返したらよくなります。JSONとしてデータをレスポンスボディに載せるには、普通にDTOクラスをビューモデルとして作ります。

```java
@Data
public class HomeViewModel implements Serializble {
    
    // サーバの時間を表示する
    public Date serverTime;
}
```

あとは、`@RestController`アノテーションをつけたコントローラクラスを作り、作成したDTOクラスを返すメソッドを作成するだけです。少しアノテーションの種類や使い方も少し変わりましたが、内容としてはビューを担当しているJSPファイルの指定が消えただけで返しているデータは同じです。

```java
@RestController
@RequestMapping("/api/v1")
public class HomeApiController {

    // カスタムビューモデルにデータをつめて渡す
    @GetMapping("/home")
    public HomeViewModel getHome() {
        HomeViewModel model = new HomeViewModel();
        model.setServerTime(new Date());
        return model;
    }
}
```

そしてフロントエンドのJavaScriptフレームワークでは、レスポンスのボディから取得して画面に表示するようになりますね。このようにREST APIだと、サーバサイドではあくまでデータモデルとビジネスロジックだけを考えれば良いようになっていて、よりそれぞれの役割の分担がよくなっていますね。

## ログインをするには

では、とりあえず簡単にSpring MVCパターンとREST APIの比較をしてみたので本題に戻りましょう。先にSpring Securityの話をしましたが、実際はSpring Securityを導入しなくても同じ話になると思います。REST APIはアーキテクチャの一つにすぎないので、フレームワークを導入できるかどうかの問題はすでに別の話になってしまいます。また、ログインというのはSpring Security以前の問題です。なので今回はSpring Securityの話はさておき、二つのアーキテクチャでどのようにログインをするかについて話したいと思います。

### MVCパターンの場合

MVCパターンでは、Sessionでログインを実現する場合が多い(と思い)ます。例えばログインを担当するメソッドを作成して、引数として`HttpServletRequest`を指定すると、そこからセッションを取得することができますね。あとは、またの引数としてFormのデータ(IDとパスワード)を受けて、これは自分が最初に学んだ方式でもあります。

まずユーザの観点から話してみましょう。普通のWebアプリケーションでは、画面のほうでログインのためにIDとパスワードを入力するようになります。JSPではそのデータを`form`として受け取り、POSTとしてコントローラに送りますね。そうしたらコントローラではIDとパスワードをServiceクラスに送って検証してもらい、問題なかったらログインした情報をSessionに載せます。こういうったシナリオで、コントローラにログインのためのメソッドを作るとおそらく以下のコードのような形のなるはずです。(実際はユーザのIDだけ載せる場合はないので参考までに)

```java
// URLは/login、メソッドはPOST
@RequestMapping(value = "/login", method = RequestMethod.POST)
public String login(User user, HttpServletRequest request) {
    // Serviceからユーザを取得する
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // リクエストからセッションを取得する
    HttpSession session = request.getSession();
    // SessionにユーザIDを載せる
    session.setAttribute("userId", loginedUser.getId());
    return "/";
}
```

これでログインができたら、あとは他のメソッドでSessionの検証をして、ログインされているかどうかを判断することになります。ログアウトする場合は、Sessionを破棄すればいいです。例えば以下のようになりましょう。

```java
// URLは/logout、メソッドはGet
@RequestMapping(value = "/logout", method = RequestMethod.GET)
public String logout(HttpServletRequest request) {
    // リクエストからセッションを取得する
    HttpSession session = request.getSession();
    // Sessionを破棄する
    session.invalidate();
    return "/";
}
```

Spring Securityを使った場合はかなりコードが変わってきますが、Sessionを使う場合にこう言った基本的なフローは変わりませんね。これでMVCパターンでの認証/認可は実現できます。

## ここで問題が

しかし、Sessionを使った方法はREST APIだと使えなくなります。なぜかというと、REST APIの最大の特徴の一つは`Stateless`、つまり`状態を持たない`というのであるからです。ここでいう状態というのはサーバに取ってのクライアントのステータスのことです。今まで通りだとクライアントはサーバに接続した瞬間からサーバに自分の状態を管理させる必要がありました。こういう状態を管理するためのものがSessionであり、先に説明した通りだとREST APIには適用すべきではないですね。(無理やり適用させるとしたら不可能でもないですが…)

そしてStatelessである故に、REST APIではクライアントのリクエスト毎にクライアントから必要な全ての情報をサーバに送るようにしています。ここから推論できるのはSessionでログインの情報を載せておくのではなく、クライアントからリクエスト毎に自分はログインしているということを証明するデータを何かの形で送る必要があるということになります。

### ではどうしたら？

クライアントがリクエスト毎に、本質的なデータのみでなく、認証のためのデータを送るための手段は、すでにHTTPの中にありました。つまり、Headerです。クライアントのリクエストには操作のためのデータがBodyとして入っていて、そこに認証のための情報をHeaderとして載せればいいですね。

クライアントがHeaderに認証のための情報を載せるには、まずサーバサイドから認証をしてもらわねばなりません。この認証の詳細については今後のポストで話すことにして、まずはHeaderに載せるデータが完成されたと仮定し、これをサーバのレスポンスでどう返すかについて述べます。

Spring BootでレスポンスのHeaderにデータを載せるためには二つの方法があります。先に述べたSpring MVCのリクエスト同様、レスポンスを扱うためのクラスである`HttpServletResponse`を使う方法と、`ResponseEntity`でレスポンスデータをラッピングすることです。どれも使える方法ですが、

## HttpServletResponseでHeaderを追加する

まずはHttpServletResponseです。シンプルに、「レスポンスにデータを載せたい」という時に使える方法ですね。MVCパターンでのログインと同じく、ログインするメソッドの引数に指定すると、レスポンスのHeaderにデータを載せることができます。

```java
@PostMapping("/login")
public User login(User user, HttpServletResponse response) {
    // Serviceからユーザを取得する
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // レスポンスのHeaderにユーザのIDを載せる
    response.addHeader("userId", loginedUser.getId());
    // Bodyを返す(Headerは自動で含まれる)
    return loginedMember;
}
```

## ResponseEntityでHeaderを追加する

もう一つの方法であるResponseEntityの場合は、引数はユーザ情報(IDとパスワード)のみで良くなります。名前からもわかると思いますが、ResponseEntityはBodyとHeaderを含め、HTTP Status(200 OKなど)を含めレスポンスに必要な情報は全て扱えるクラスです。使い方は簡単で、ログインメソッドの戻り値となるBodyをResponseEntityで包み、Headerなどの情報も一緒に詰めて返します。

```java
@PostMapping("/login")
public ResponseEntity<Member> login(User user) {
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // レスポンスとしてHeaderとBodyを一緒にセットして返す
    return ResponseEntity.ok().header("userId", loginedUser.getId()).body(loginedUser);
}
```

## そのあとは？

レスポンスのHeaderに載せた情報は、フロントエンド側でリクエスト毎にHeaderに載せて送ることになります。しかし、リクエスト毎に送るためにはどこかにこの情報を保存する必要がありますね。Cookieを使う方法もありますが、セキュリティ情のリスクがあるため多くの場合ではブラウザのローカルストレージに保存するようになっているようです。これはフロントエンドの領域なので、ここでは深掘りしません。

また、ログアウトの場合はどうなるか、という問題もありますが、これは様々な方法で実現しているようです。例えばフロントエンドからできる方法としては、Headerに載せる認証情報を削除したり、間違った情報を送るようにするような方法があります。そしてサーバサイドでは、一定時間がすぎると認証情報を使えなくするなどの対策があります。

## 最後に

本当は、REST APIでのログインに関してはまだ考えなければならないことは他にもあります。例えば、HTTP Headerに認証のため必要となる情報を載せるということは理解できても、実際はどのようなデータを載せればいいか、そのデータはどう作るか、リクエスト毎にHeaderで認証情報を扱うということはセキュリティやリソースの面で大丈夫かなど。

でも、とにかくこれでREST APIでログインするための仕組みの一つはわかりました。次回は、このHeaderとSpring Securityを使ってのログインを実装する方法を述べながら、以上の問題についても扱いたいなと思っています。では、また！