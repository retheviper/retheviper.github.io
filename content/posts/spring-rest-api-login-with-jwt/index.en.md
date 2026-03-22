---
title: "Implementing REST API Login with JWT"
date: 2020-06-10
translationKey: "posts/spring-rest-api-login-with-jwt"
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - spring security
  - rest api
  - jwt
---

Recently, I've been seriously studying the implementation of login using REST API using Spring Security. I don't know if I'll ever use it in actual work, but I feel like I need to first understand how general authentication and authorization should be implemented in the REST API and how Spring Security works (I haven't implemented it much lately, so I don't want to lose my sense of it), so I'd like to write about what I researched and record the code I implemented.

There may be other methods, but the one I chose this time is REST API authentication and authorization using JWT and Spring Security. At first, I had a hard time because I had no knowledge of Spring Security, JWT, or REST API authentication and authorization, but I was successful anyway, so I summarized what I learned.

## What is JWT?

JWT (JSON Web Token) is an open standard [RFC 7519](https://tools.ietf.org/html/rfc7519) that can convey information, including signatures and encryption, as JSON objects. Since JWT data is signed, it is possible to identify the source of the transmitted data and verify that the data has not been replaced (tampered with) during the process, so it is often used for user authentication.

In this way, the JWT contains all the data necessary for signatures, encryption, etc., so the server side is self-contained and can perform data validation checks by receiving the JWT, making it possible to authenticate and authorize users as a stateless method in addition to the existing method using Session. Therefore, it can be said that this authentication method is suitable for logging in with the REST API, which we will introduce here.

## Login scenario with JWT

First, when logging in using JWT, when the client sends the credentials required for login (ID, password, etc.) to the server, the server issues and returns a JWT based on the user information. The client sends this information to the server with each request, and the server validates the JWT before returning the response.

When the JWT created here is sent back to the client, and when the client later sends requests, it is placed in the HTTP header of the response or request. Please refer to [this post](../spring-rest-api-how-to-login/) for why it should go in the header rather than the session.

After successfully logging in, it will check whether you are authenticated and authorize access to each URL, just like when using Session. Spring Security will be in charge here.

## Login with JWT and Spring Security

Now, let's think about how the above scenario can be realized. First, we need to prepare a controller and method to receive login requests. The controller then validates the credentials input from the service class. Once you can verify it, you need to create a JWT based on that information.

## Spring Security settings (before using JWT)

First, we will easily implement authentication and authorization using REST API using Spring Security. Spring Security itself requires a considerable amount of study, but here we will first implement the functionality of obtaining user information registered in the DB and restricting the URLs that can be accessed depending on the user's role. (In addition, code for classes that have little to do with login is omitted.)

### Entity class

First, we need to create a class to retrieve user information from the DB. Let's easily create a class that implements UserDetails. You can create this as an existing user entity, but since it is a dedicated class for authentication, you can create it as a separate class. However, in that case, you need to properly manage the table so that it is linked to existing user information.

The code below is an example of creating the UserDetails class as a Spring Data JPA standard entity. From here, we will put the username and roles in the JWT and use them for authentication and authorization.

```java
@Data
@Entity
public class User implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    // User name (generally used as the ID)
    private String username;
    
    // Whether the user account is expired
    private boolean accountNonExpired;

    // Whether the user account is locked
    private boolean accountNonLocked;

    // Whether the user's credentials are expired
    private boolean credentialsNonExpired;

    // Whether the user account is enabled
    private boolean enabled;

    // User roles for authorization
    @ElementCollection(fetch = FetchType.EAGER)
    private List<String> roles = new ArrayList<>();

    // Get the user's authorities
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.roles.stream()
                    .map(SimpleGrantedAuthority::new)
                    .collect(Collectors.toUnmodifiableList());
    }
}
```

### Service class

Create a Service class to retrieve UserDetails. This class will retrieve user information after login. The only method of UserDetailsService is `loadUserByUsername` to retrieve UserDetails from the user name, so implement this so that it can be retrieved appropriately from the Repository.

```java
@Service
public class UserServiceImpl implements UserDetailsService {

    // Repository for retrieving UserDetails
    private final UserRepository repository;

    @Autowired
    public UserServiceImpl(UserRepository repository) {
        this.repository = repository;
    }
    
    // Method for retrieving user information after authentication
    @Override
    public UserDetails loadUserByUsername(final String username) throws UsernameNotFoundException {
        return this.repository.findByUsername(username);
    }
}
```

### Configuration class

Configuration class for authorization. Here, we have made it impossible to access any URL unless the role `USER` is set. After logging in, when a user sends a request to access a URL, this setting will check the user's role and authorize access. When creating this role, it is a good idea to save the roles as `ROLE_USER` using a method such as createUser.

Also, since we are building a REST API and handling authentication and authorization with JWT, we will change some default settings. For example, this includes settings related to [Basic authentication](https://en.wikipedia.org/wiki/Basic_access_authentication), [CSRF](https://en.wikipedia.org/wiki/Cross-site_request_forgery), and sessions.

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
                // Do not use Basic authentication
                .httpBasic().disable()
                // Do not use CSRF protection here
                .csrf().disable()
                // Do not use sessions because the API is stateless
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                // Require the USER role for all URLs
                .authorizeRequests()
                .anyRequest().hasRole("USER");
    }
}
```

This completes the minimum necessary preparations for using Spring Security. Let's proceed to the next step assuming that you have created other services and repositories for the user's CRUD as appropriate.

## Adding JWT dependencies

Next, we will configure settings to use JWT in earnest. To use JWT with Spring Boot, you need to add dependencies. JWT itself has fixed specifications, and it can be achieved by creating JSON appropriate to the standard and then encoding it as Base64, but when dealing with something like this, it is safer to use a library if possible.

There are several JWT libraries available for Spring and Java, but because JWT itself is standardized, the basics are the same whichever library you choose. That said, the level of support for the JWT specification can vary, so check the [list of supported libraries](https://jwt.io/#libraries-io) on the official site before deciding. In this post, I use `JSON Web Token Support For The JVM`.

For Maven, add dependencies as follows.

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

In case of Gradle, it is as follows.

```groovy
dependencies {
    implementation 'io.jsonwebtoken:jjwt:0.9.1'
}
```

## Create a class that provides JWT

After adding the dependencies, you will need to create a class that will actually create the JWT to be placed in the Response when a login is successful and the Header when a request is made from the client. I think there are various ways to do this, but here we will create a class that will create and verify the JWT. After that, create a class to validate the Header JWT for each request.

### JWT specifications

Before creating a JWT, let's first take a quick look at what structure a JWT has. JWT consists of three elements: `Header`, `Payload`, and `Signature`. Each element is encoded as a Base64 string and concatenated to form a single JWT.

#### Header configuration

The Header will now contain information about what this token is and what algorithm it is encoded with.

| id | Meaning | Details |
|---|---|---|
| typ | Token Type | JWT type (=JWT) |
| alg | Hashing Algorithm | Encoding algorithm |

#### Payload configuration

Payload is like a body, consisting of multiple information units made up of name and value pairs, and each information unit posted here is called a claim. For this login, we will post the user ID, role, and period for which the token was issued in this claim.

| id | Meaning | Details |
|---|---|---|
| jti | JWT ID | JWT identifier |
| sub | subject | JWT unique key |
| iss | issuer | JWT issuer |
| aud | audience | JWT users |
| iat | issued at | time the JWT was issued |
| nbf | not before | JWT start time |
| exp | expiration time | JWT expiration time |

#### Signature

Signature encodes the Header and Payload, and then further encodes it using an arbitrary secret key. This allows the server side to decode the signature of the JWT sent by the client and obtain the Header and Payload.

A JWT built correctly from the Header, Payload, and Signature can be checked on the [official JWT site](https://jwt.io). You can inspect the JWT structure and the stored data there, so it is useful for debugging.

![Structure of JWT](jwt_structure.webp)

## Implementation of Token Provider class

Now that we know what information a JWT is made up of, let's create a class that contains the necessary information and creates the actual JWT. There is no need to fill in all of Header, Payload, and Signature here; we will use only the minimum information.

First, create a method to create a token. The claims to be placed in the JWT payload created here are limited to the user name and role. Also, set the time when the JWT was issued and the expiration time so that it cannot be used outside of the period. Finally, we will set a custom secret key and create a signature.

In addition, we will also create a method to read the token and retrieve the DB user information, a method to retrieve the token from the request header, a method to verify the validity period of the token, and a method to retrieve the user name (ID) on the token. (I think it would be a good idea to separate this into a separate class.)

```java
@Component
public class JWTProvider {

    // Secret key used to encode the signature
    private static final String TOKEN_SECRET_KEY = "This is secret!";

    // Token validity period (1 hour)
    private static final long TOKEN_VALID_DURATION = 1000L * 60L * 60L;

    // Service class for retrieving user information
    private final UserDetailsService service;

    @Autowired
    public JWTProvider(UserDetailsService service) {
        this.service = service;
    }

    // Create a JWT from a User object
    public String createToken(User user) {
        // Put the user name and roles into the claims
        Claims claims = Jwts.claims().setSubject(user.getId());
        claims.put("roles", user.getRoles());
        // Set the issued time and expiration time
        Date iat = new Date();
        Date exp = new Date(iat.getTime() + TOKEN_VALID_DURATION);
        // Create the JWT
        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(iat)
                .setExpiration(exp)
                .signWith(SignatureAlgorithm.HS256, TOKEN_SECRET_KEY)
                .compact();
    }

    // Get user information from the token
    public Authentication getAuthentication(final String token) {
        final UserDetails userDetails = this.service.loadUserByUsername(this.getSubject(token));
        return new UsernamePasswordAuthenticationToken(userDetails, "", userDetails.getAuthorities());
    }

    // Get the token from the request header
    public String resolveToken(final HttpServletRequest request) {
        return request.getHeader("X-AUTH-TOKEN");
    }

    // Validate the token expiration
    public boolean validateToken(final String token) {
        try {
            final Jws<Claims> claims = Jwts.parser().setSigningKey(TOKEN_SECRET_KEY).parseClaimsJws(token);
            return !claims.getBody().getExpiration().before(new Date());
        } catch (Exception e) {
            return false;
        }
    }

    // Get the user name from the token
    public String getSubject(final String token) {
        return Jwts.parser().setSigningKey(TOKEN_SECRET_KEY).parseClaimsJws(token).getBody().getSubject();
    }
}
```

## Implementing a custom Filter class

Next, create a Filter class to validate requests when logging in. When a user logs in for the first time, this class uses the JWTProvider created earlier to obtain the token from the Header, verify the validity period, and if there is no problem, obtain the user information from the DB and set it as Spring Security authentication information.

```java
@Component
public class JWTAuthenticationFilter extends GenericFilterBean {

    // Provider used to validate the token
    private final JWTProvider provider;

    @Autowired
    public JWTAuthenticationFilter(JWTProvider provider) {
        this.provider = provider;
    }

    // Filter login-related requests
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

## Add JWT login settings to Spring Security

Now that we have completed the preparations for creating and validating the JWT, let's move on to setting up Spring Security to perform authentication and authorization using these classes. I'm not saying which one is the standard, but I think you can use either one depending on your preference (or requirements), so I've prepared three methods.

### When using FormLogin

This is an implementation when you want to use Spring Security's FormLogin to set the login URL and JWT creation after login. On the client side, login credentials will be sent as POST form data.

Here you will set the URL to process the login, parameters for credentials, etc., and set `AuthenticationSuccessHandler` to create and return a token if the login is successful.

#### Creating AuthenticationSuccessHandler

First, create a SuccessHandler to return a token if login is successful. If the login is successful, the user information is saved in the Authentication class, so we pass it to the Provider, have it create a token, and then return it by putting it in the header of the response.

```java
@Component
public class JWTAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    // Provider used to create the token
    final private JWTProvider provider;

    @Autowired
    public JWTAuthenticationSuccessHandler(JWTProvider provider) {
        this.provider = provider;
    }

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                        HttpServletResponse response,
                                        Authentication auth) throws IOException, ServletException {
        // Do nothing if the response has already been committed
        if (response.isCommitted()) {
            return;
        }
        // Get the user information for the successful login
        User user = (User) auth.getPrincipal();
        // Create the token and place it in the header
        response.setHeader("X-AUTH-TOKEN", this.provider.createToken(user));
        // HTTP status is 200 OK
        response.setStatus(HttpStatus.OK.value());
    }
}
```

#### Spring Security settings

Now that we have created a Handler class for when login is successful, let's change the Spring Security settings. What you need is `loginProcessingUrl()` (URL where POST communication is performed for login processing) and Handler settings. The user name parameter name and password parameter name are only required if they are different from the default. And for Filter, `UsernamePasswordAuthenticationFilter` will be executed first with the default settings, so configure it to use `JWTAuthenticationFilter` that you created earlier.

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // Handler for successful login processing
    private final JWTAuthenticationSuccessHandler successHandler;

    // Filter for authentication and authorization after login
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
                // Use formLogin
                .formLogin()
                // URL that receives credentials via POST and performs login
                .loginProcessingUrl("/api/v1/web/login")
                // Allow access to the login URL without authentication
                .permitAll()
                // User name parameter (default is username)
                .usernameParameter("id")
                // User password parameter (default is password)
                .passwordParameter("pass")
                // SuccessHandler to run after login succeeds
                .successHandler(this.successHandler)
                .and()
                .authorizeRequests()
                .anyRequest().hasRole("ROLE_USER")
                .and()
                // Change the default filter settings
                .addFilterBefore(this.filter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

### When using a Filter that can use JSON

When using Spring Security's FormLogin, the login credentials must be form data. However, with REST API, data exchange is basically JSON. Therefore, I would like to send the credentials in JSON as well, but Spring Security does not provide such a method.

In this case, you need to create a custom UsernamePasswordAuthenticationFilter class to parse the JSON. It's a little more complicated than using FormLogin, but let's try doing it like a REST API. Once you can implement FromLogin, you can implement it as if you just added another Filter.

#### Creating JsonUsernamePasswordAuthenticationFilter

Spring's default Filter class that checks credentials is called UsernamePasswordAuthenticationFilter. Create a custom class that inherits from this and use it to parse JSON. The implementation below is an example that supports both Form data and JSON.

```java
public class JsonUsernamePasswordAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    // Constant used to read the content type from the header
    private static final String CONTENT_TYPE = "Content-Type";
    
    // Map used to store JSON data
    private Map<String, String> jsonRequest;

    // Get the user name
    @Override
    protected String obtainUsername(HttpServletRequest request) {
        return getParameter(request, getUsernameParameter());
    }

    // Get the password
    @Override
    protected String obtainPassword(HttpServletRequest request) {
        return getParameter(request, getPasswordParameter());
    }
    
    // Read credentials from JSON or form data and wrap them as Authentication
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

    // Get parameters (user name and password) from the request
    private String getParameter(HttpServletRequest request, String parameter) {
        if (headerContentTypeIsJson(request)) {
            return jsonRequest.get(parameter);
        } else {
            return request.getParameter(parameter);
        }
    }

    // Check whether the content type in the header is JSON
    private boolean headerContentTypeIsJson(HttpServletRequest request) {
        return request.getHeader(CONTENT_TYPE).equals(MediaType.APPLICATION_JSON_VALUE);
    }
}
```

#### Creating AuthenticationSuccessHandler

This is the same as with FormLogin. Please refer to [the section above](#creating-authenticationsuccesshandler) for details.

#### Spring Security settings

Here, we will set FilterProcessUrl, AuthenticationManager, and AuthenticationSuccessHandler for the custom UsernamePasswordAuthenticationFilter we created to change the default filter settings. This setting eliminates the need for FormLogin.

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // Handler for successful login processing
    private final JWTAuthenticationSuccessHandler successHandler;

    // Filter for authentication and authorization after login
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
                // Do not use FormLogin
                .formLogin().disable()
                .authorizeRequests()
                .anyRequest().hasRole("ROLE_USER")
                .and()
                // Register the JWT filter before authentication
                .addFilterBefore(this.filter, UsernamePasswordAuthenticationFilter.class)
                // Replace UsernamePasswordAuthenticationFilter with a custom class
                .addFilterAt(getJsonUsernamePasswordAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
    }

    // Configuration for the custom UsernamePasswordAuthenticationFilter
    private JsonUsernamePasswordAuthenticationFilter getJsonUsernamePasswordAuthenticationFilter() {
        JsonUsernamePasswordAuthenticationFilter jsonFilter = new JsonUsernamePasswordAuthenticationFilter();
        try {
            // Configure the login processing URL
            jsonFilter.setFilterProcessesUrl("/api/v1/web/login");
            // Configure the AuthenticationManager
            jsonFilter.setAuthenticationManager(this.authenticationManagerBean());
            // Configure the AuthenticationSuccessHandler
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

### When using a controller for login

This is an example of creating a login-only Controller, just like controlling other URLs with a Controller. The usage is not much different from a general Controller, so I feel that this one is easier to use.

Also, you can use ResponseEntity or a custom class as a response, so this may be a better choice if you need to not only put a token in the Header but also fill in some data in the Body and send it along with it.

#### Creating a controller

As mentioned above, we will create a Controller that is not much different from a general REST API Controller. You will create a login URL and a method associated with it, and will be responsible for authentication at login.

```java
@RestController
@RequestMapping("api/v1/web")
public class SignApiController {

    // Service class used to validate credentials
    private final UserService service;
    
    // Provider used to create the token
    private final JWTProvider provider;

    @Autowired
    public SignApiController(MemberService service, JWTProvider provider) {
        this.service = service;
        this.provider = provider;
    }

    // Receive form-data credentials and authenticate
    @PostMapping("/login")
    public void login(@Validated @RequestBody LoginMemberForm form, HttpServletResponse response) {
        // Get user information from the credentials
        User user = this.service.getUser(form.getId(), form.getPassword());
        // Create a token from the retrieved information
        String token = this.provider.createToken(user);
        // Create the token and place it in the header
        response.setHeader("X-AUTH-TOKEN", this.provider.createToken(user));
        // HTTP status is 200 OK
        response.setStatus(HttpStatus.OK.value());
    }
}
```

#### Spring Security settings

If you use Controller, add settings to allow access to the login URL without authorization and settings to use Filter.

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // Filter for authentication and authorization after login
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
                // Allow access to the login URL without authentication
                .antMatchers("/api/v1/web/login/").permitAll()
                .anyRequest().hasRole("ROLE_USER")
                .and()
                // Change the default filter settings
                .addFilterBefore(this.filter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

## test

You can easily test the login you have implemented so far using tools such as CURL or [Postman](https://www.postman.com).

For example, if you use FormLogin, the login is here.

```bash
curl -i -X POST "http://localhost:8080/api/v1/web/login" -d "id=user" -d "pass=1234"
```

Click here for a JSON login test using Postman. (You can see the JWT is back with X-AUTH-TOKEN!)

![JWT Postman Login](jwt_postman_login.webp)

## lastly

I had a lot of trouble setting up Spring Security than I expected, and there weren't many lectures I wanted, so I managed to implement login using JWT in the REST API. I think it would be a good idea to create a separate library for just this...

However, this does not mean that the settings are complete. I only created a success handler here, but in some cases you may also need an `AuthenticationFailureHandler` for failed logins. Also, since this post is focused on login, I did not cover it, but exception handling is also required for URLs that fail authentication or authorization. Additionally, when using JWT authentication, the client owns the token, so logout cannot be controlled entirely from the server side. The client implementation therefore also needs to account for that.

However, this time I was able to achieve the minimum goal, so I'll leave it here for now. See you soon!
