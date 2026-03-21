---
title: "For login with REST API"
date: 2020-05-30
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - rest api
---

A new project has started and we have decided to create a SPA (Single Page Application) using Angular and Spring boot, but it is still at the requirements definition stage and there is still some time left until implementation. I don't have much experience in so-called upstream processes, so I'm struggling every day, but I'm making my own applications little by little after work to practice.

Although I am somewhat familiar with creating REST APIs with Spring Boot, I still don't have much knowledge about authentication/authorization, so I purposely decided to include Spring Security in my app. However, after reading some articles on Spring Security and understanding that URLs that can be accessed are restricted based on roles, I tried to implement it in my own application, but a problem arose. As usual, I implemented it using the REST API, but the Spring Security code I was referring to was mostly for the classic Spring MVC pattern.

So, in this post, I will briefly write about the Spring MVC pattern and what is required to log in to the REST API. (The actual login method is planned for a future post)``

## Controller implementation

First, let's take a look at how to implement the MVC pattern and REST API controller.

## For MVC pattern

I think the Spring training still used in many courses is mostly based on the traditional MVC pattern with JSP. For example, you can write a method annotated with `@RequestMapping` in a controller class annotated with `@Controller`, then use `Model` or `ModelAndView` to pass both data and a view path, usually a JSP. Using JSP is not the same thing as the MVC pattern itself, but many Spring MVC projects rely on JSP, so I will refer to that style here as the legacy MVC pattern.

I think it's easier to understand if you write it in code rather than just expressing it in text. For example: This is a simple example that returns the server time when you connect to the URL `/home`. Even if you use ModelAndView, I don't think there's much of a difference in what you're doing, just putting the data and view together in the Model.

```java
@Controller
public class HomeController {
    
    // Display the JSP file named home and the server time
    @RequestMapping(value = "/home", method = RequestMethod.GET)
    public String home(Model model) {
        Date date = new Date();
        model.addAttribute("serverTime", date);
        return "home";
    }
}
```

## For REST API

In my case, I first learned the Spring MVC pattern and JSP, so I was used to writing like this, but after joining the company, I encountered Spring Boot and REST API, and the way I wrote code changed a little. The recent trend is that it is often created using JavaScript frameworks such as Angular/React/Vue rather than JSP, so it is better to return only the data in JSON format. To put data as JSON in the response body, create a DTO class as a view model.

```java
@Data
public class HomeViewModel implements Serializble {
    
    // Display the server time
    public Date serverTime;
}
```

All that is left to do is create a controller class with the `@RestController` annotation and create a method that returns the created DTO class. The type and usage of annotations have changed slightly, but the data returned is the same except that the JSP file in charge of the view has been no longer specified.

```java
@RestController
@RequestMapping("/api/v1")
public class HomeApiController {

    // Fill a custom view model with data and return it
    @GetMapping("/home")
    public HomeViewModel getHome() {
        HomeViewModel model = new HomeViewModel();
        model.setServerTime(new Date());
        return model;
    }
}
```

The front-end JavaScript framework then retrieves it from the response body and displays it on the screen. In this way, with REST API, you only need to think about the data model and business logic on the server side, and the roles of each are better divided.

## To log in

Now that we have briefly compared the Spring MVC pattern and REST API, let's get back to the main topic. I talked about Spring Security earlier, but I think it would be the same story even without introducing Spring Security. Since the REST API is just one architecture, the question of whether or not a framework can be implemented is already a different story. Also, login is an issue that predates Spring Security. So this time, I would like to put Spring Security aside and talk about how to log in in two architectures.

## MVC pattern

In the MVC pattern, login is often achieved using Session (I think). For example, if you create a method that handles login and specify `HttpServletRequest` as an argument, you can obtain the session from there. After that, I received the form data (ID and password) as another argument, which is also the method I first learned.

Let's start from the user's perspective. In a normal web application, you will be prompted to enter your ID and password to log in on the screen. JSP receives that data as `form` and sends it to the controller as POST. Then, the controller sends the ID and password to the Service class for verification, and if there are no problems, the login information is posted in Session. In such a scenario, if you create a login method in your controller, it will probably look like the code below. (Actually, there are no cases where only the user ID is listed, so just for reference)

```java
// URL is /login and the method is POST
@RequestMapping(value = "/login", method = RequestMethod.POST)
public String login(User user, HttpServletRequest request) {
    // Get the user from the service
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // Get the session from the request
    HttpSession session = request.getSession();
    // Store the user ID in the session
    session.setAttribute("userId", loginedUser.getId());
    return "/";
}
```

Once you are logged in, you will need to use another method to verify the Session and determine whether you are logged in. If you want to log out, just destroy the Session. For example, let's look like this:

```java
// URL is /logout and the method is GET
@RequestMapping(value = "/logout", method = RequestMethod.GET)
public String logout(HttpServletRequest request) {
    // Get the session from the request
    HttpSession session = request.getSession();
    // Invalidate the session
    session.invalidate();
    return "/";
}
```

The code changes considerably when using Spring Security, but the basic flow remains the same when using Session. Authentication/authorization using the MVC pattern can now be achieved.

## Here's the problem

However, a session-based approach does not fit a REST API. One of the biggest characteristics of REST is that it is `stateless`. The "state" here means client state from the server's point of view. Traditionally, the server kept track of that state after the client connected. Sessions are used for exactly that purpose, so as explained earlier, they should not be applied to a REST API unless you force the design in that direction.

Since it is stateless, the REST API sends all necessary information from the client to the server for each client request. What we can infer from this is that rather than storing login information in Session, the client needs to send some form of data to prove that it is logged in for each request.

## So what should I do?

There was already a way in HTTP for clients to send authentication data, as well as essential data, with each request. That is, Header. The client's request contains the data for the operation as the body, and it would be nice to put the authentication information there as the header.

In order for the client to post authentication information in the Header, it must first authenticate from the server side. I will talk about the details of this authentication in a future post, but first I will assume that the data to be placed in the Header is complete, and I will explain how to return this in the server response.

There are two ways to put data in the response header with Spring Boot. As with the Spring MVC request mentioned earlier, the two methods are to use `HttpServletResponse`, a class for handling responses, and to wrap the response data in `ResponseEntity`. Any method can be used, but

## Add Header in HttpServletResponse

First is HttpServletResponse. This is a simple method that can be used when you want to include data in the response. As with login in the MVC pattern, data can be placed in the header of the response by specifying it as an argument of the login method.

```java
@PostMapping("/login")
public User login(User user, HttpServletResponse response) {
    // Get the user from the service
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // Put the user ID into the response header
    response.addHeader("userId", loginedUser.getId());
    // Return the body (the header is included automatically)
    return loginedMember;
}
```

## Add Header with ResponseEntity

In the case of the other method, ResponseEntity, the arguments only need to be user information (ID and password). As you can probably tell from the name, ResponseEntity is a class that can handle all the information necessary for a response, including the Body and Header, as well as HTTP Status (200 OK, etc.). It is easy to use; wrap the Body, which is the return value of the login method, in ResponseEntity, and return information such as Header.

```java
@PostMapping("/login")
public ResponseEntity<Member> login(User user) {
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // Return the header and body together as the response
    return ResponseEntity.ok().header("userId", loginedUser.getId()).body(loginedUser);
}
```

## After that?

The information placed in the header of the response will be sent in the header for each request on the front end side. However, we need to store this information somewhere in order to send it with every request. There is also a way to use cookies, but due to security risks, in most cases it seems that cookies are saved in the browser's local storage. This is front-end territory, so I won't go into depth here.

There is also the issue of what happens when you log out, and this seems to be achieved in a variety of ways. For example, methods that can be done from the front end include deleting the authentication information listed in the Header or sending incorrect information. On the server side, there are measures such as making authentication information unusable after a certain period of time.

## lastly

Actually, there are still other things to think about when logging in with the REST API. For example, even if you understand that information necessary for authentication should be placed in the HTTP Header, you may wonder what kind of data should actually be placed, how to create that data, and whether handling authentication information in the Header for each request is okay in terms of security and resources.

But anyway, I now understand one of the mechanisms for logging in using the REST API. Next time, I would like to discuss how to implement login using this Header and Spring Security, and also address the above issues. See you soon!
