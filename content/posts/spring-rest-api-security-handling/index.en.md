---
title: "Implementing Spring Security exception handling in REST API"
date: 2020-06-17
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - spring security
  - rest api
---
Last time, we explained the login method using REST API and authentication and authorization using JWT. However, from the perspective of overall application security, just being able to log in does not mean everything is over. For example, there is error handling.

By creating a Spring Serucity Configuration class (an inherited class of `WebSecurityConfigurerAdapter`) and restricting the URLs that can be accessed by role, it will issue an error for unauthorized requests, but you may also want to handle your own errors with a custom class.

So, in addition to logging in, this time I would like to introduce the Spring Security settings required for authentication and authorization with the REST API. What we will introduce here is as follows.

1. Handling when a user who is not logged in accesses a specific URL
2. Handling when accessing a specific URL while logged in
3. Handling when login fails
4. Handling when logged out

Now, let's take a look at how to create a custom class one by one.

## AuthenticationEntryPoint

To handle cases where an unauthenticated user accesses a protected URL, you need an implementation of `AuthenticationEntryPoint`. Here is an example that returns `UNAUTHORIZED` in the response.

```java
public class JWTAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse respose, AuthenticationException exception) throws IOException, ServletException {
        // Configure the response (401 UNAUTHORIZED)
        response.sendError(HttpStatus.UNAUTHORIZED.value(), HttpStatus.UNAUTHORIZED.getReasonPhrase());
    }
}
```

## AccessDeniedHandler

Handling cases where a logged-in user accesses a URL they are not allowed to use requires an implementation of `AccessDeniedHandler`. Here is an example that returns `FORBIDDEN`.

```java
public class JWTAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException exception) throws IOException, ServletException {
        // Configure the response (403 FORBIDDEN)
        response.sendError(HttpStatus.FORBIDDEN.value(), HttpStatus.FORBIDDEN.getReasonPhrase());
    }
}
```

## AuthenticationFailureHandler

To handle login failures, you need an implementation of `AuthenticationFailureHandler`. Here is an example that returns `FORBIDDEN`.

```java
public class JWTAuthenticationFailureHandler implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,  HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        // Configure the response (403 FORBIDDEN)
        response.sendError(HttpStatus.FORBIDDEN.value(), HttpStatus.FORBIDDEN.getReasonPhrase());
    }
}
```

## LogoutSuccessHandler

To handle logout, you need an implementation of `LogoutSuccessHandler`. In REST APIs, including JWT-based login, the client holds authentication information such as a token rather than relying on a server-side session. Because of that, logout usually means deleting the authentication information on the client side and simply returning `OK` from the server.

```java
public class JWTLogoutSuccessHandler implements LogoutSuccessHandler {

    @Override
    public void onLogoutSuccess(final HttpServletRequest request, final HttpServletResponse response,
            final Authentication authentication) throws IOException, ServletException {
        // Configure the response (200 OK)
        response.setStatus(HttpStatus.OK.value());
    }
}
```

## Configuration class settings

Finally, register the custom Handler classes created so far in the Configuration class according to their respective purposes. The registration itself is not difficult, just remember the method. Here we are modifying the class created in the previous post.

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
                // Configure exception handling
                .exceptionHandling()
                // Configure the handler used when the user is not logged in
                .accessDeniedHandler(new JWTAccessDeniedHandler())
                // Configure the handler used when the user lacks permission
                .authenticationEntryPoint(new JWTAuthenticationEntryPoint())
                .and()
                // Configure logout
                .logout()
                // Configure the URL used for logout
                .logoutUrl("/logout")
                // Specify the LogoutSuccessHandler
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
            // Configure the handler used when authentication fails
            jsonFilter.setAuthenticationFailureHandler(new JWTAuthenticationFailureHandler());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        return jsonFilter;
    }
}
```

Here we are creating new instances of classes such as Filter and Handler, but you can also register them as beans with `@Component`. I think it would be better to register a bean, but since the code would be long, I omitted it here.

Also, it is generally said that it is better to attach DI to the constructor rather than the field, but if the field is declared final, DI will be automatically applied even if you use Lombok's `@RequiredArgsConstructor`, so this time I tried writing it that way.

## Finally

In the case of handling, it's not difficult as all you have to do is implement an interface with only one method and register it in the Configuration class. Each Handler interface takes a request, response, and authentication information as arguments, so you can perform various processes such as creating and returning a custom exception view model or creating a controller for error handling and redirecting.

From a security perspective, the occurrence of a stylized Spring Security error is a notification that the server side is created with Spring, so it may lead to app vulnerability issues. Therefore, when creating a web application, I think it is better to thoroughly manage the information leaked to the outside.

See you soon!
