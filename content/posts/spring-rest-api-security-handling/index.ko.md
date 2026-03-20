---
title: "REST API에서 Spring Security 예외 처리하기"
date: 2020-06-17
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - spring security
  - rest api
translationKey: "posts/spring-rest-api-security-handling"
---

지난 글에서는 REST API에서 로그인하는 방식과 JWT를 이용한 인증·인가에 대해 이야기했습니다. 하지만 애플리케이션 전체 보안 관점에서 보면, 로그인만 됐다고 끝나는 것은 아닙니다. 에러 처리도 중요합니다.

Spring Security의 `WebSecurityConfigurerAdapter`를 상속한 Configuration 클래스를 만들어 두고, 역할에 따라 접근 가능한 URL을 제한해 두면 인증되지 않은 요청에 대해 에러를 낼 수 있습니다. 그런데 실제 서비스에서는 그 에러를 그대로 쓰기보다, 커스텀 클래스에서 원하는 방식으로 처리하고 싶을 때가 많습니다.

이번 글에서는 로그인 외에도 REST API의 인증·인가 과정에서 필요한 Spring Security 설정을 정리해 보겠습니다. 다룰 내용은 다음과 같습니다.

1. 로그인하지 않은 사용자가 특정 URL에 접근했을 때의 처리
1. 로그인했지만 권한이 없는 사용자가 접근했을 때의 처리
1. 로그인에 실패했을 때의 처리
1. 로그아웃했을 때의 처리

하나씩 커스텀 클래스로 어떻게 만드는지 보겠습니다.

## `AuthenticationEntryPoint`

`로그인하지 않은 사용자가 특정 URL에 접근했을 때`의 처리를 위해서는 `AuthenticationEntryPoint`를 구현해야 합니다. 여기서는 응답으로 `UNAUTHORIZED`를 보내는 예를 보겠습니다.

```java
public class JWTAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse respose, AuthenticationException exception) throws IOException, ServletException {
        // 응답 설정(401 UNAUTHORIZED)
        response.sendError(HttpStatus.UNAUTHORIZED.value(), HttpStatus.UNAUTHORIZED.getReasonPhrase());
    }
}
```

## `AccessDeniedHandler`

`로그인했지만 특정 URL에 접근할 권한이 없을 때`의 처리를 위해서는 `AccessDeniedHandler`를 구현해야 합니다. 여기서는 응답으로 `FORBIDDEN`을 보내는 예를 보겠습니다.

```java
public class JWTAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException exception) throws IOException, ServletException {
        // 응답 설정(403 FORBIDDEN)
        response.sendError(HttpStatus.FORBIDDEN.value(), HttpStatus.FORBIDDEN.getReasonPhrase());
    }
}
```

## `AuthenticationFailureHandler`

`로그인에 실패했을 때`의 처리를 위해서는 `AuthenticationFailureHandler`를 구현해야 합니다. 여기서는 응답으로 `FORBIDDEN`을 보내는 예를 보겠습니다.

```java
public class JWTAuthenticationFailureHandler implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,  HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        // 응답 설정(403 FORBIDDEN)
        response.sendError(HttpStatus.FORBIDDEN.value(), HttpStatus.FORBIDDEN.getReasonPhrase());
    }
}
```

## `LogoutSuccessHandler`

`로그아웃했을 때`의 처리를 위해서는 `LogoutSuccessHandler`를 구현해야 합니다. JWT를 쓰는 로그인 방식에서는 Session이 아니라 토큰을 클라이언트가 들고 있으므로, 서버에서 별도 처리를 하지 않는 경우가 많습니다. 로그아웃할 때는 클라이언트가 인증에 필요한 정보를 지우고, 서버는 단순히 `OK`를 반환하면 됩니다.

```java
public class JWTLogoutSuccessHandler implements LogoutSuccessHandler {

    @Override
    public void onLogoutSuccess(final HttpServletRequest request, final HttpServletResponse response,
            final Authentication authentication) throws IOException, ServletException {
        // 응답 설정(200 OK)
        response.setStatus(HttpStatus.OK.value());
    }
}
```

## Configuration 설정

마지막으로 Configuration 클래스에 앞에서 만든 커스텀 Handler들을 목적에 맞게 등록합니다. 메서드 이름과 위치만 정리하면 되므로 생각보다 어렵지 않습니다. 여기서는 이전 글에서 만든 클래스를 기반으로 수정했습니다.

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
                // 예외 처리 설정
                .exceptionHandling()
                // 로그인하지 않았을 때의 처리
                .accessDeniedHandler(new JWTAccessDeniedHandler())
                // 권한이 없을 때의 처리
                .authenticationEntryPoint(new JWTAuthenticationEntryPoint())
                .and()
                // 로그아웃 설정
                .logout()
                // 로그아웃에 사용할 URL
                .logoutUrl("/logout")
                // LogoutSuccessHandler 지정
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
            // 인증 실패 시 처리
            jsonFilter.setAuthenticationFailureHandler(new JWTAuthenticationFailureHandler());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        return jsonFilter;
    }
}
```

여기서는 Filter나 Handler 객체를 직접 새로 만들고 있지만, `@Component`로 Bean 등록해도 됩니다. 개인적으로는 Bean으로 등록하는 편이 더 낫다고 보지만, 여기서는 코드가 길어지는 것을 피하려고 생략했습니다.

또 일반적으로는 필드 주입보다 생성자 주입을 권장하지만, 필드를 `final`로 선언하면 Lombok의 `@RequiredArgsConstructor`로 자동 주입할 수 있으므로 이번에는 그 방식으로 작성했습니다.

## 마지막으로

Handler 구현은 결국 메서드 하나만 가진 인터페이스를 구현하고 Configuration에 등록하면 되므로 구조 자체는 어렵지 않습니다. 각 Handler 인터페이스는 요청, 응답, 인증 정보를 인자로 받으므로, 필요하면 커스텀 예외 응답 모델을 만들어 반환하거나 에러 전용 컨트롤러로 리다이렉트하는 식으로 다양하게 확장할 수 있습니다.

보안 관점에서 보면 정형화된 Spring Security 에러가 그대로 나가는 것은 서버가 Spring으로 작성됐다는 정보를 드러내는 셈이라, 취약점 분석에 힌트를 줄 수도 있습니다. 웹 애플리케이션을 만들 때는 인증 로직뿐 아니라 외부로 노출되는 오류 정보까지 함께 설계해야 합니다.
