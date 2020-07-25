---
title: "JWTによるREST APIのログインを実現する"
date: 2020-06-10
categories: 
  - spring
photos:
  - /assets/images/sideimage/spring_logo.jpg
tags:
  - spring
  - spring security
  - rest api
  - jwt
---

最近は、本格的にSpring SecurityによるREST APIでのログインの実装を勉強しています。実際の業務で使うことになるかどうかは分かりませんが、とりあえずREST APIで一般的な認証認可はどのように実装すべきで、Spring Securityはどう動くものかをまず理解しておかないとという気がして(そして最近はあまり実装していないので、感覚を失いたくないという願望もあり)、とりあえず自分で調べたことと、実装したコードの記録を兼ねて書きたいと思います。

他にもいろいろな方法があるのかも知れませんが、今回自分が選んだのはJWTとSpring Securityを使ったREST APIでの認証認可のところです。初めはSpring SecurityもJWTもREST APIでの認証認可も全く知識がなかったのでかなり苦労しましたが、とりあえず成功したので、わかったことをまとめてみました。

## JWTとは

JWT(JSON Web Token)は、署名や暗号化などを含む情報をJSONオブジェクトとして伝達することができる、オープン標準[RFC 7519](https://tools.ietf.org/html/rfc7519)です。JWTのデータは署名されているためこの伝達されるデータの送信元の特定や途中でデータが入れ替えされなかったか(改ざんされてないか)の検証などができるので、多くの場合にユーザの認証で使われます。

このようにJWTの中には署名や暗号化などに必要なデータは全て含まれているので、サーバサイドではJWTを受け取ることでデータのバリデーションチェックができるなど自己完結性があって、既存のSessionを使った方法とは別にStatelessな方法としてユーザの認証と認可を可能にします。なので今回紹介する、REST APIでのログインのために適している認証方式といえますね。

## JWTでのログインシナリオ

まずJWTを使ったログインの場合、クライアントからログインに必要なクレデンシャル(IDやパスワードなど)をサーバに送ると、サーバからはユーザの情報に基づいてJWTを発行して返します。クライアントはこの情報をリクエスト毎に載せてサーバに伝達し、サーバではJWTのバリデーションを行なってからレスポンスを返すようになります。

ここでJWTとして作られたJWTをクライアント側に送る時やクライアントがリクエストを送る時、レスポンスやリクエストのHTTP Headerに載せることになります。なぜSessionではなくHeaderに載せるかについては[このポスト](../../../05/30/spring-rest-api-how-to-login)を参考にしてください。

ログインに成功したあとは、Sessionを使う場合と同じく認証させているかをチェックして各URLへのアクセスを認可するようになりますね。ここはSpring Securityが担当するようになります。

## JWTとSpring Securityによるログイン

では、以上のシナリオをどう実現できるかを考えてみます。まず、ログイン要請を受けるコントローラとメソッドを用意する必要がありますね。そして、そのコントローラからはサービスクラスから入力されたクレデンシャルを検証してもらいます。検証できたら、その情報を元にJWTのJWTを作る必要がありますね。

### Spring Securityの設定(JWTを使う前)

まずはREST APIでの認証認可を簡単にSpring Securityで実装します。Spring Securityそのものだけでもかなり膨大な量を勉強しなければならないのですが、ここではまず、DBに登録されているユーザの情報を取得し、そのユーザのロールによってアクセスできるURLを制限するという機能だけを実現します。(他にも、ログインとはあまり関係のないクラスのコードは省略しています)

#### Entityクラス

まずはDBからユーザの情報を取得するためのクラスを作る必要がありますね。UserDetailsをimplementしたクラスを簡単に作っておきます。これは既存にユーザのエンティティとして作っても良いですが、認証のための専用のクラスとなるので、別クラスとして作成しても構わないです。ただ、その場合はちゃんとテーブルで既存のユーザ情報と紐づくように管理する必要がありますね。

以下のコードは、UserDetailsクラスをSpring Data JPA基準のエンティティとして作成した例です。ここからユーザ名(username)とロール(roles)をJWTに載せて認証と認可に使うことにします。

```java
@Data
@Entity
public class User implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    // ユーザ名(一般的にID)
    private String username;
    
    // ユーザアカウントが満了されているかの設定    
    private boolean accountNonExpired;

    // ユーザアカウントがロックされているかの設定
    private boolean accountNonLocked;

    // ユーザアカウントのクレデンシャルが満了されているかの設定
    private boolean credentialsNonExpired;

    // ユーザアカウントが活性化されているかの設定
    private boolean enabled;

    // 認可のためのユーザのロール
    @ElementCollection(fetch = FetchType.EAGER)
    private List<String> roles = new ArrayList<>();

    // ユーザの認可情報を取得する
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.roles.stream()
                    .map(SimpleGrantedAuthority::new)
                    .collect(Collectors.toUnmodifiableList());
    }
}
```

#### Serviceクラス

UserDetailsを取得するためのServiceクラスを作っておきます。このクラスでログイン後のユーザ情報を取得するようになります。UserDetailsServiceのメソッドはユーザ名からUserDetailsを取得するための`loadUserByUsername`しかないので、これを適切にRepositoryから取得できるように実装します。

```java
@Service
public class UserServiceImpl implements UserDetailsService {

    // UserDetailsを取得できるRepository
    private final UserRepository repository;

    @Autowired
    public UserServiceImpl(UserRepository repository) {
        this.repository = repository;
    }
    
    // 認証後にユーザ情報を取得するためのメソッド
    @Override
    public UserDetails loadUserByUsername(final String username) throws UsernameNotFoundException {
        return this.repository.findByUsername(username);
    }
}
```

#### Configurationクラス

認可のためのConfigurationクラスです。ここでは`USER`というロールが設定されてない場合、どのURLにもアクセスできないようにしておきました。ログイン後、ユーザがURLにアクセスためのリクエストを送ると、この設定によりユーザのロールを確認してアクセスを認可するようになります。このロールはユーザを作成するとき、createUserなどのメソッドでrolesを`ROLE_USER`として保存するようにしておくと良いです。

また、今回作るのはREST APIであり、JWTによる認証と認可を行うことになるので、いくつかのデフォルト設定を変えておきます。例えば[Basic Auth](https://ja.wikipedia.org/wiki/Basic%E8%AA%8D%E8%A8%BC)と[CSRF](https://ja.wikipedia.org/wiki/%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%B5%E3%82%A4%E3%83%88%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88%E3%83%95%E3%82%A9%E3%83%BC%E3%82%B8%E3%82%A7%E3%83%AA)の設定や、セッション設定などがあります。

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(final HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                // Basic認証を使わない
                .httpBasic().disable()
                // CSRF設定を使わない
                .csrf().disable()
                // セッションはStatelessなので使わない
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                // USERではないとどのURLでもアクセスできない
                .authorizeRequests()
                .anyRequest().hasRole("USER");
    }
}
```

これでSpring Securityを使うための必要最低限の準備は終わりました。他にユーザのCRUDのためのServiceやRepositoryは適宜作成してあるという前提として、次に進めましょう。

### JWTの依存関係の追加

次に、本格的にJWTを使うための設定を行います。Spring BootでJWTを使うためには依存関係の追加が必要です。JWT自体は仕様が決まっていて、規格に合わせて適切なJSONとして作成した後にBase64としてエンコードしても実現はできますが、こういうものを扱う場合はなるべくライブラリを使った方が安全ですね。

Spring(Java)で使えるJWTのライブラリはいくつかありますが、JWT自体が標準なのでどちらを選んでも基本は同じです。ただ、ライブラリ毎にJWTの仕様にどこまで対応しているのかは違う場合があるので、公式サイトから[対応しているライブラリのリスト](https://jwt.io/#libraries-io)を確認してどれを使うかを選びましょう。このポストでは`JSON Web Token Support For The JVM`を使った場合での実装方法を紹介します。

Mavenの場合は以下のように依存関係を追加します。

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

Gradleの場合の場合は以下となります。

```groovy
dependencies {
    implementation 'io.jsonwebtoken:jjwt:0.9.1'
}
```

### JWTを提供するクラスを作る

依存関係を追加したら、ログイン成功時のResponseとクライアントからのリクエスト時のHeaderに載せるためのJWTを実際に作ってくれるクラスの作成が必要となります。色々な方法があると思いますが、ここではJWTを作成して検証もしてくれるようなクラスを作ります。その後はリクエスト毎に、HeaderのJWTを検証するためのクラスを作って行きます。

#### JWTの仕様

JWTを作る前に、まず簡単にJWTがどんな構造を持っているかを見ていきましょう。JWTは`Header`、`Payload`、`Signature`という三つの要素で構成されています。それぞれの要素はBase64の文字列としてエンコードして、つなげることで一つのJWTが構成されます。

##### Headerの構成

Headerではこのトークンがどのようなもので、どのアルゴリズムでエンコードされているかを表す情報を載せるようになります。

| id | 意味 | 詳細 |
|---|---|---|
| typ | Token Type | JWTのタイプ(=JWT) |
| alg | Hashing Algoritym | エンコードする時のアルゴリズム |

##### Payloadの構成

Payloadはnameとvalueのペアで構成された複数の情報単位で構成されている、いわばBodyのようなもので、ここに載せる個々の情報単位たちをClaimと呼びます。今回のログインではこのClaimにユーザのIDとロール、そしてトークンが発行された期間を載せることにします。

| id | 意味 | 詳細 |
|---|---|---|
| jti | JWT ID | JWTの識別子 |
| sub | subject | JWTのユニークなキー |
| iss | issuer | JWTを発行者 |
| aud | audience | JWTの利用者 |
| iat | issued at | JWTが発行された時間 |
| nbf | not before | JWTの開始時間 |
| exp | expiration time | JWTの満了時間 |

##### Sigunature

SignatureではHeaderとPayloadをエンコードしたあと、更に任意のシークレットキーを持ってエンコードします。これによって、サーバ側ではクライアントが送ってきたJWTをSignatureをデコードしてHeaderとPayloadを取得できるようになります。

Header、Payload、Signature順で正しく作成したJWTは、[JWTの公式サイト](https://jwt.io)から検証できます。以下のようにJWTの構造と格納しているデータを確認してデバッグができるので、興味のある方はぜひ試してみてください。

![](/assets/images/postimage/jwt_structure.png)

### Token Providerクラスの実装

では、JWTがどんな情報により構成されているかがわかったので、必要な情報を載せて実際のJWTを作成するクラスを作りましょう。ここでHeader、Payload、Signatureの全部を埋める必要はなく、最低限の情報だけ使うことにします。

まずトークンを作成するメソッドを作ります。ここで作成されるJWTのPayloadに載せるClaimは、ユーザ名とロールに限定します。また、JWTが発行された時間と満了時間を設定して、期間外では使えないようにします。そして最後にカスタムシークレットキーを設定してSignatureを作成することにします。

他には、トークンを読み込んでDBのユーザ情報を取得するメソッド、リクエストのHeaderからトークンを取得するメソッド、トークンの有効期間を検証するメソッド、トークンにのせたユーザ名(ID)を取得するメソッドなども作っておきます。(こちらは別途クラスで切っても良さそうな気はします)

```java
@Component
public class JWTProvider {

    // Signatureのエンコードに使うシークレットキー
    private static final String TOKEN_SECRET_KEY = "This is secrect!";

    // トークンの有効期間(1時間)
    private static final long TOKEN_VAILD_DURATION = 1000L * 60L * 60L;

    // ユーザ情報を取得するためのサービスクラス
    private final UserDetailsService service;

    @Autowired
    public JWTProvider(UserDetailService service) {
        this.service = service;
    }

    // UserオブジェクトからJWTを作成する
    public String createToken(User user) {
        // Claimとしてユーザ名とロールを載せる
        Claims claims = Jwts.claims().setSubject(user.getId());
        claims.put("roles", user.getRoles());
        // トークンの開始時間と満了時間を決める
        Date iat = new Date();
        Date exp = new Date(start.getTime() + TOKEN_VAILD_DURATION);
        // JWTの作成
        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(iat)
                .setExpiration(exp)
                .signWith(SignatureAlgorithm.HS256, TOKEN_SECRET_KEY)
                .compact();
    }

    // トークンからユーザ情報を取得する
    public Authentication getAuthentication(final String token) {
        final UserDetails userDetails = this.service.loadUserByUsername(this.getSubject(token));
        return new UsernamePasswordAuthenticationToken(userDetails, "", userDetails.getAuthorities());
    }

    // リクエストのHeaderからトークンを取得する
    public String resolveToken(final HttpServletRequest request) {
        return request.getHeader("X-AUTH-TOKEN");
    }

    // トークンの有効期間を検証する
    public boolean validateToken(final String token) {
        try {
            final Jws<Claims> claims = Jwts.parser().setSigningKey(this.secretKey).parseClaimsJws(token);
            return !claims.getBody().getExpiration().before(new Date());
        } catch (Exception e) {
            return false;
        }
    }

    // トークンからユーザ名を取得する
    pubic String getSubject(final String token) {
        return Jwts.parser().setSigningKey(this.secretKey).parseClaimsJws(token).getBody().getSubject();
    }
}
```

### カスタムFilterクラスの実装

次に、ログインするときにリクエストを検証するためのFilterクラスを作成します。最初ユーザがログインすると、このクラスでは先に作成したJWTProviderを使って、Headerからトークンを取得、有効期間の検証を行った後、問題なければDBからユーザ情報を取得してSpring Securityの認証情報(Authentication)としてセットするようになります。

```java
@Component
public class JWTAuthenticationFilter extends GenericFilterBean {

    // トークンを検証するためのProvider
    private final JWTProvider provider;

    @Autowired
    public JWTAuthenticationFilter(JWTProvider provider) {
        this.provider = provider;
    }

    // ログインに対するフィルタリングを行う
    @Override
    public void doFilter(final ServletRequest request, final ServletResponse response, final FilterChain filterChain)
            throws IOException, ServletException {
        final String token = this.provider.resolveToken((HttpServletRequest) request);
        if (token != null && this.provider.validateToken(token)) {
            final Authentication auth = this.provider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        filterChain.doFilter(request, response);
    }
}
```

### Spring SecurityにJWTでのログイン設定を追加する

では、JWTを作成して検証するための準備が終わったので、これからはSpring Securityでこのクラスたちを使って認証と認可を行うための設定をする版ですね。どれが定番だとかという訳ではないですが、どちらも好みに合わせて(もしくは要件に合わせて)使えるのではないかと思い、三つの方法を用意しました。

#### FormLoginを使う場合

Spring SecurityのFormLoginを利用してログインのためのURLと、ログイン後のJWT作成を全て設定しておきたい場合の実装です。クライアント側ではログインのためのクレデンシャルをPOSTのFormデータとして送るようになります。

ここではログインを処理するためのURLとクレデンシャルのためのパラメータなどを設定して、ログインに成功したらトークンを作成して返すための`AuthenticationSuccessHandler`を設定することになります。

##### AuthenticationSuccessHandlerの作成

まずログインに成功した場合にトークンを返すためのSuccessHandlerを作成します。ログインに成功した場合、Authenticationクラスにユーザ情報が保存されるので、それをProviderに渡してトークンを作成してもらった後にレスポンスのHeaderに載せて返すことをやっています。

```java
@Component
public class JWTAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    // トークンを作成するためのProvider
    final private JWTProvider provider;

    @Autowired
    public JWTAuthenticationSuccessHandler(JWTProvider provider) {
        this.provider = provider;
    }

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                        HttpServletResponse response,
                                        Authentication auth) throws IOException, ServletException {
        // すでにレスポンスで情報を返した場合は何もしない
        if (response.isCommitted()) {
            return;
        }
        // ログインに成功したユーザ情報を取得する
        User user = (User) auth.getPrincipal();
        // Headerにトークンを作成して載せる
        response.setHeader("X-AUTH-TOKEN", this.provider.createToken(user));
        // HTTP Statusは200 OK
        response.setStatus(HttpStatus.OK.value());
    }
}
```

##### Spring Securityの設定

では、ログインに成功したときのHandlerクラスを作成したので、Spring Securityの設定を変えていきます。必要なのは`loginProcessingUrl()`(ログインの処理をするためのPOST通信が行われるURLとHanlderの設定で、ユーザ名のパラメータ名やパスワードのパラメタ名はデフォルトと違う場合にだけ必要となります。そしてFilterはデフォルト設定だと`UsernamePasswordAuthenticationFilter`が先に実行されるので、先ほど作成した`JWTAuthenticationFilter`を使うための設定をします。

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // ログインが成功した場合の処理のためのHandler
    private final JWTAuthenticationSuccessHandler successHandler;

    // ログイン以降の認証認可のためのFilter
    private final JWTAuthenticationFilter filter;

    @Autowired
    public SecurityConfig(JWTAuthenticationSuccessHandler successHandler, JWTAuthenticationFilter filter) {
        this.successHandler = successHandler;
        this.filter = filter;
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(final HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                .httpBasic().disable()
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                // formLoginを使う
                .formLogin()
                // POSTでクレデンシャルをもらい、ログイン処理を行うURL(UserDetailsServiceを使うことになる)
                .loginProcessingUrl("/api/v1/web/login")
                // ログイン処理ようのURLには認証認可なしでアクセスできる
                .permitAll()
                // ユーザ名のパラメータ(デフォルトはusername)
                .usernameParameter("id")
                // ユーザパスワードのパラメータ(デフォルトはpassword)
                .passwordParameter("pass")
                // ログインに成功したら実行されるsuccessHandlerの指定
                .successHandler(this.successHandler)
                .and()
                .authorizeRequests()
                .anyRequest().hasRole("ROLE_USER")
                .and()
                // デフォルトのFilter設定を変える
                .addFilterBefore(this.filter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

#### JSONを使えるFilterを使う場合

Spring SecurityのFormLoginを使う場合、ログインのためのクレデンシャルはFormデータである必要があります。しかし、REST APIならばデータのやりとりはJSONが基本ですね。なのでクレデンシャルもJSONで送るようにしたいのですが、Spring Securityではそのようなメソッドを提供していません。

こういう場合は、カスタムUsernamePasswordAuthenticationFilterクラスを作ってJSONをパースする必要があります。FormLoginを使う場合に比べて少し複雑になりますが、REST APIらしくやってみましょう。FromLoginでの実装ができると、こちらはFilterをもう一つ追加するだけの感覚で実装ができます。

##### JsonUsernamePasswordAuthenticationFilterの作成

クレデンシャルを確認するSpringのデフォルトのFilterクラスはUsernamePasswordAuthenticationFilterというもので、これを継承したカスタムクラスを作り、そこからJSONをパースして使うようにします。以下の実装は、FormデータとJSONの両方に対応している例です。

```java
public class JsonUsernamePasswordAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    // Headerからコンテントタイプを取得するための定数
    private static final String CONTENT_TYPE = "Content-Type";
    
    // JSONデータを保存するためのMap
    private Map<String, String> jsonRequest;

    // ユーザ名を取得する
    @Override
    protected String obtainUsername(HttpServletRequest request) {
        return getParameter(request, getUsernameParameter());
    }

    // パスワードを取得する
    @Override
    protected String obtainPassword(HttpServletRequest request) {
        return getParameter(request, getPasswordParameter());
    }
    
    // JSONもしくはFormデータのクレデンシャルを取得し、Authenticationとして載せる
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        if (headerContentTypeIsJson(request)) {
            ObjectMapper mapper = new ObjectMapper();
            try {
                this.jsonRequest = mapper.readValue(request.getReader().lines().collect(Collectors.joining()),
                        new TypeReference<Map<String, String>>() {
                        });
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
        String username = obtainUsername(request) != null ? obtainUsername(request) : "";
        String password = obtainPassword(request) != null ? obtainPassword(request) : "";
        UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);
        setDetails(request, authRequest);
        return this.getAuthenticationManager().authenticate(authRequest);
    }

    // リクエストからパラメータ(ユーザ名とパスワード)を取得する
    private String getParameter(HttpServletRequest request, String parameter) {
        if (headerContentTypeIsJson(request)) {
            return jsonRequest.get(parameter);
        } else {
            return request.getParameter(parameter);
        }
    }

    // HeaderからコンテントタイプがJSONかどうかを判定する
    private boolean headerContentTypeIsJson(HttpServletRequest request) {
        return request.getHeader(CONTENT_TYPE).equals(MediaType.APPLICATION_JSON_VALUE);
    }
}
```

##### AuthenticationSuccessHandlerの作成

ここはFormLoginの場合と同じです。詳細については[こちらを](#AuthenticationSuccessHandlerの作成)参考にしてください。

##### Spring Securityの設定

ここでは、作成したカスタムUsernamePasswordAuthenticationFilterにFilterProcessUrlと、AuthenticationManager、AuthenticationSuccessHandlerを設定して、デフォルトのフィルタ設定を変えるようになります。この設定ではFormLoginが要らなくなリます。

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // ログインが成功した場合の処理のためのHandler
    private final JWTAuthenticationSuccessHandler successHandler;

    // ログイン以降の認証認可のためのFilter
    private final JWTAuthenticationFilter filter;

    @Autowired
    public SecurityConfig(JWTAuthenticationSuccessHandler successHandler, JWTAuthenticationFilter filter) {
        this.successHandler = successHandler;
        this.filter = filter;
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(final HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                .httpBasic().disable()
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                // FormLoginは使わない
                .formLogin().disable()
                .authorizeRequests()
                .anyRequest().hasRole("ROLE_USER")
                .and()
                // 認証前にJWTのFilterを設定
                .addFilterBefore(this.filter, UsernamePasswordAuthenticationFilter.class)
                // UsernamePasswordAuthenticationFilterはカスタムクラスに代替
                .addFilterAt(getJsonUsernamePasswordAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
    }

    // カスタムUsernamePasswordAuthenticationFilterの設定
    private JsonUsernamePasswordAuthenticationFilter getJsonUsernamePasswordAuthenticationFilter() {
        JsonUsernamePasswordAuthenticationFilter jsonFilter = new JsonUsernamePasswordAuthenticationFilter();
        try {
            // ログインを処理するURLの設定
            jsonFilter.setFilterProcessesUrl("/api/v1/web/login");
            // AuthenticationManagerの設定
            jsonFilter.setAuthenticationManager(this.authenticationManagerBean());
            // AuthenticationSuccessHandlerの設定
            jsonFilter.setAuthenticationSuccessHandler(this.successHandler);
            jsonFilter.setUsernameParameter("id");
            jsonFilter.setPasswordParameter("pass");
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        return jsonFilter;
    }
}
```

#### ログイン用のControllerを使う場合

他のURLをControllerで制御しているのと同じく、ログイン専用のContollerを作る場合の例です。一般的なControllerとあまり使い方は変わらないので、こちらの方がやりやすい感もしますね。

また、レスポンスとしてResponseEntityやカスタムクラスも使えるのでHeaderにトークンを載せるだけでなく、Bodyに何かデータを埋めて共に送る必要のある場合はこちらの方が良い選択なのかも知れません。

##### Controllerの作成

前述した通り、一般的なREST API用のControllerとあまり変わりないものを作ります。ログインようのURLと、それに紐づくメソッドを作り、ログイン時の認証を担当することになります。

```java
@RestController
@RequestMapping("api/v1/web")
public class SignApiController {

    // クレデンシャルを検証するためのサービスクラス
    private final UserService service;
    
    // トークンを作成するためのProvider
    private final JWTProvider provider;

    @Autowired
    public SignApiController(MemberService service, JWTProvider provider) {
        this.service = service;
        this.provider = provider;
    }

    // Formデータでクレデンシャルをもらい、認証を行う
    @PostMapping("/login")
    public void login(@Validated @RequestBody LoginMemberForm form, HttpServletResponse response) {
        // クレデンシャルからユーザ情報を取得
        User user = this.service.getUser(form.getId(), form.getPassword());
        // 取得した情報でトークンを作成
        String token = this.provider.createToken(user);
        // Headerにトークンを作成して載せる
        response.setHeader("X-AUTH-TOKEN", this.provider.createToken(user));
        // HTTP Statusは200 OK
        response.setStatus(HttpStatus.OK.value());
    }
}
```

##### Spring Securityの設定

Controllerを使った場合は、認可認可なしでもログイン用のURLにアクセスできる設定と、Filterを使うための設定を追加します。

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // ログイン以降の認証認可のためのFilter
    private final JWTAuthenticationFilter filter;

    @Autowired
    public SecurityConfig(JWTAuthenticationFilter filter) {
        this.provider = provider;
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(final HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                .httpBasic().disable()
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                // ログイン処理ようのURLには認証認可なしでアクセスできる
                .antMatchers("/api/v1/web/login/").permitAll()
                .anyRequest().hasRole("ROLE_USER")
                .and()
                // デフォルトのFilter設定を変える
                .addFilterBefore(this.filter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

### テスト

今まで実装したログインのテストは、CURLもしくは[Postman](https://www.postman.com)などのツールで簡単にできます。

例えば、FormLoginを使った場合のログインはこちらになります。

```bash
curl -i -X POST "http://localhost:8080/api/v1/web/login" -d "id=user" -d "pass=1234"
```

Postmanを使ったJSONでのログインテストはこちらになります。(X-AUTH-TOKENでJWTが帰ってきたのを確認できます!)

![](/assets/images/postimage/jwt_postman_login.png)

## 最後に

思ったよりSpring Security周りの設定がいろいろと必要となり、自分の欲しがっていたレクチャはあまりなかったのでかなり苦労しましたが、これでなんとかREST APIでのJWTを使ったログインは実装できました。これだけを別途ライブラリとして作っても良いかと思いますね…

でも、まだこれで完全な設定ができた訳ではありません。ここではSuccessHandlerのみを作成しましたが、場合によってはログインに失敗した場合の`AuthenticationFailureHandler`が必要になる可能性もあります。また、これはあくまでログインに関するポストなので扱ってはなかったのですが、認証認可できてないURLへのアクセスに対するException Handlingも必要です。また、JWTを使った認証の場合、クライアントがトークンを持ってしまうのでサーバ側からログアウトを制御できないという点があり、クライアント側の実装ではそこに対しての対策も考えなければなりません。

が、今回はとりあえず最小限の目標は達成できたということで、いったんここまでとなります。では、また！