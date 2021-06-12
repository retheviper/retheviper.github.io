---
title: "REST APIでのSpring Securityの例外ハンドリングを実装する"
date: 2020-06-17
categories: 
  - spring
photos:
  - /assets/images/sideimage/spring_logo.jpg
tags:
  - spring
  - spring security
  - rest api
---

前回はREST APIでのログインの方式と、JWTを使った認証認可について説明しました。でも、アプリケーション全体のセキュリティという観点からするとログインができたということだけで全てが終わったわけではないです。例えばエラーハンドリングがありますね。

Spring SerucityのConfigurationクラス(`WebSecurityConfigurerAdapter`の継承クラス)を作成しておいて、ロールによってアクセスできるURLをの制限をかけておくことだけで認可されてないリクエストに対してエラーを出してくれますが、カスタムクラスで独自のエラーハンドリングをしたい場合もあリますね。

なので、今回はログインの以外に、REST APIでの認証認可周りで必要となるSpring Securityの設定を紹介したいと思います。ここで紹介するのは以下のようになります。

1. ログインしてない使用者が、特定のURLにアクセスする場合のハンドリング
1. ログインしているが、特定のURLにアクセスする場合のハンドリング
1. ログインに失敗した場合のハンドリング
1. ログアウトした場合のハンドリング

では、一つづつどうやってカスタムクラスを作るのかをみていきましょう。

## AuthenticationEntryPoint

`ログインしていない使用者が、特定のURLにアクセスする場合のハンドリング`のためには、AuthenticationEntryPointの実装が必要となります。ここではレスポンスとして`UNAUTHORIZED`を表示する例を紹介します。

```java
public class JWTAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse respose, AuthenticationException exception) throws IOException, ServletException {
        // レスポンスの設定(401 UNAUTHORIZED)
        response.sendError(HttpStatus.UNAUTHORIZED.value(), HttpStatus.UNAUTHORIZED.getReasonPhrase());
    }
}
```

## AccessDeniedHandler

`ログインしているが、特定のURLにアクセスする場合のハンドリング`のためには、AccessDeniedHandlerの実装が必要となります。ここではレスポンスとして`FORBIDDEN`を表示する例を紹介します。

```java
public class JWTAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException exception) throws IOException, ServletException {
        // レスポンスの設定(403 FORBIDDEN)
        response.sendError(HttpStatus.FORBIDDEN.value(), HttpStatus.FORBIDDEN.getReasonPhrase());
    }
}
```

## AuthenticationFailureHandler

`ログインに失敗した場合のハンドリング`のためには、AuthenticationFailureHandlerの実装が必要となります。ここではレスポンスとして`FORBIDDEN`を表示する例を紹介します。

```java
public class JWTAuthenticationFailureHandler implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,  HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        // レスポンスの設定(403 FORBIDDEN)
        response.sendError(HttpStatus.FORBIDDEN.value(), HttpStatus.FORBIDDEN.getReasonPhrase());
    }
}
```

## LogoutSuccessHandler

`ログアウトした場合のハンドリング`のためには、LogoutSuccessHandlerの実装が必要となります。JWTを使ったログインだけでなく、REST APIのログインはSessionではなくトークンのような情報をクライアントが持つので、サーバサイドで処理することはありません。ログアウトの場合はクライアント側で認証に必要な情報を削除するようにして、サーバサイドとしては単純にレスポンスとして`OK`を表示します。

```java
public class JWTLogoutSuccessHandler implements LogoutSuccessHandler {

    @Override
    public void onLogoutSuccess(final HttpServletRequest request, final HttpServletResponse response,
            final Authentication authentication) throws IOException, ServletException {
        // レスポンスの設定(200 OK)
        response.setStatus(HttpStatus.OK.value());
    }
}
```

## Configurationクラスの設定

最後に、Configurationクラスに今まで作成したカスタムHandlerクラスをそれぞれの目的に合わせて登録します。メソッドを覚えるだけで、登録の内容自体は難しくないです。ここでは以前のポストで作成したクラスを基盤に修正しています。

```java
@Configuration
@RequiredArgsConstructor
public class SecurityConfig extends WebSecurityConfigurerAdapter {

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
                .formLogin().disable()
                .and()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                .anyRequest().hasRole("USER")
                .and()
                // 例外ハンドリングの設定
                .exceptionHandling()
                // ログインしなかった場合のハンドラーの設定
                .accessDeniedHandler(new JWTAccessDeniedHandler())
                // 権限がない場合のハンドラーの設定
                .authenticationEntryPoint(new JWTAuthenticationEntryPoint())
                .and()
                // ログアウトの設定
                .logout()
                // ログアウトに使うURLを設定する
                .logoutUrl("/logout")
                // LogoutSuccessHandlerを指定する
                .logoutSuccessHandler(new JWTLogoutSuccessHandler())
                .and()
                .addFilterBefore(new JWTAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)
                .addFilterAt(getJsonUsernamePasswordAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
    }

    private JsonUsernamePasswordAuthenticationFilter getJsonUsernamePasswordAuthenticationFilter() {
        final JsonUsernamePasswordAuthenticationFilter jsonFilter = new JsonUsernamePasswordAuthenticationFilter();
        try {
            jsonFilter.setFilterProcessesUrl("/api/v1/web/login");
            jsonFilter.setAuthenticationManager(this.authenticationManagerBean());
            jsonFilter.setAuthenticationSuccessHandler(new JWTAuthenticationSuccessHandler());
            // 認証に失敗した場合のハンドラーの設定
            jsonFilter.setAuthenticationFailureHandler(new JWTAuthenticationFailureHandler());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        return jsonFilter;
    }
}
```

ここではFilterやHandlerなどのクラスのインスタンスを新しく作成していますが、`@Component`でBeanとして登録しておいても構いません。どっちかとするとBean登録した方が良いかなという気はしますが、ここではコードが長くなるので省略しておきました。

また、一般的にDIはフィールドよりもコンストラクタにつけた方が良いといいますが、フィールドをfinalとして宣言している場合ならLombokの`@RequiredArgsConstructor`を使っても自動でDIされますので今回はそのような書き方にしてみました。

## 最後に

ハンドリングの場合は、メソッドが一つしかないインタフェースを実装して、Configurationクラスに登録するだけなので難しくないですね。それぞれのHandlerインタフェースは、引数としてリクエストはレスポンス、認証情報をとっているので、場合によってはカスタム例外ビューモデルを作成して返したり、エラー処理用のControllerを作ってリダイレクトさせるなど様々な処理ができます。

セキュリティという面から考えると、定型化されたSpring Securityのエラーが出されるということはサーバサイドがSpringで作成されていることを知らせるようなものなので、アプリの脆弱性の問題に繋がる場合もあると思われますね。なのでWebアプリケーションを作る場合は、外部に漏出される情報に関しては徹底的に管理しておいた方が良いのではないかと思います。

では、また！
