---
title: "REST API에서 로그인 구현하기"
date: 2020-05-30
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - rest api
translationKey: "posts/spring-rest-api-how-to-login"
---

새로운 프로젝트를 맡으면서 Angular와 Spring Boot로 SPA(Single Page Application)를 만들게 됐습니다. 아직 요구사항 정의 단계라 구현까지는 시간이 좀 남았고, 저도 상류 공정은 익숙하지 않아서 매일 배우는 중입니다. 일이 끝난 뒤에는 연습을 겸해 간단한 자작 애플리케이션도 조금씩 만들고 있습니다.

Spring Boot로 REST API를 만드는 데는 어느 정도 익숙하지만, 인증과 인가에 대한 지식은 아직 부족합니다. 그래서 이번 자작 앱에서는 일부러 Spring Security를 넣어 보기로 했습니다. Spring Security 관련 글을 읽으면서 역할에 따라 접근 가능한 URL을 제한하는 구조는 이해했지만, 막상 직접 REST API에 적용하려고 하니 문제가 생겼습니다. 참고하던 예제 대부분이 옛날식 Spring MVC 패턴을 기준으로 작성돼 있었기 때문입니다.

그래서 이번 글에서는 Spring MVC 패턴과 REST API에서 로그인 구현이 어떻게 달라지는지 간단히 정리해 보겠습니다. 실제 로그인 구현 방법은 다음 글에서 다룰 예정입니다.

## 컨트롤러 구현

먼저 MVC 패턴과 REST API에서 컨트롤러를 어떻게 구현하는지 보겠습니다.

### MVC 패턴의 경우

지금도 많은 교육 과정에서 Spring을 가르칠 때는 전통적인 MVC 패턴과 JSP를 함께 다루는 경우가 많습니다. 예를 들어 `@Controller`가 붙은 컨트롤러 클래스에 `@RequestMapping` 메서드를 만들고, 그 안에서 `Model`이나 `ModelAndView`로 데이터와 뷰 경로를 넘기는 방식입니다. 엄밀히 말하면 `JSP를 쓴다 = MVC`는 아니지만, 많은 Spring MVC 프로젝트가 JSP를 사용하니 여기서는 그런 전통적인 방식으로 이해해도 됩니다.

글로 설명하는 것보다 코드로 보는 편이 더 쉽습니다. 예를 들어 `/home`에 접속하면 서버 시간을 보여 주는 간단한 예시는 다음과 같습니다. `ModelAndView`를 쓰더라도 결국 데이터와 뷰를 함께 넘긴다는 점은 비슷합니다.

```java
@Controller
public class HomeController {
    
    // home이라는 JSP 파일과 서버 시간을 보여 준다
    @RequestMapping(value = "/home", method = RequestMethod.GET)
    public String home(Model model) {
        Date date = new Date();
        model.addAttribute("serverTime", date);
        return "home";
    }
}
```

### REST API의 경우

저 역시 처음에는 Spring MVC와 JSP를 배우며 이런 방식에 익숙해졌습니다. 하지만 회사에 들어간 뒤에는 Spring Boot와 REST API를 쓰게 되면서 코드 스타일이 조금 달라졌습니다. 요즘은 JSP보다 Angular, React, Vue 같은 JavaScript 프레임워크를 더 많이 쓰기 때문에, 서버는 JSON 형태로 데이터만 돌려주면 되는 경우가 많습니다. 이럴 때는 보통 DTO를 뷰 모델처럼 사용합니다.

```java
@Data
public class HomeViewModel implements Serializble {
    
    // 서버 시간을 담는다
    public Date serverTime;
}
```

이후에는 `@RestController`가 붙은 컨트롤러를 만들고, 만든 DTO를 반환하는 메서드를 작성하면 됩니다. 어노테이션 이름과 쓰임새는 조금 달라졌지만, 실제로는 뷰 대신 JSP 경로를 반환하지 않게 된 것뿐이고, 돌려주는 데이터 자체는 같습니다.

```java
@RestController
@RequestMapping("/api/v1")
public class HomeApiController {

    // 커스텀 뷰 모델에 데이터를 담아 반환한다
    @GetMapping("/home")
    public HomeViewModel getHome() {
        HomeViewModel model = new HomeViewModel();
        model.setServerTime(new Date());
        return model;
    }
}
```

프런트엔드의 JavaScript 프레임워크는 이 응답 본문을 받아 화면에 표시합니다. 이런 구조에서는 서버가 데이터 모델과 비즈니스 로직에만 집중하면 되므로, 역할 분담이 더 분명해집니다.

## 로그인하려면

이제 MVC 패턴과 REST API의 차이를 간단히 봤으니 본론으로 돌아가겠습니다. 앞에서 Spring Security를 언급했지만, 실제 로그인 자체는 Spring Security 없이도 비슷한 문제를 다루게 됩니다. REST API는 하나의 아키텍처일 뿐이고, 로그인은 그보다 더 기본적인 문제입니다. 그래서 이번에는 Spring Security 이야기는 잠시 뒤로 미루고, 두 아키텍처에서 로그인을 어떻게 처리하는지에 집중해 보겠습니다.

### MVC 패턴

MVC 패턴에서는 보통 Session을 이용해 로그인을 구현합니다. 로그인 메서드에서 `HttpServletRequest`를 받아 세션을 꺼내고, 로그인 정보는 폼 데이터로 받는 방식이 익숙합니다.

사용자 입장에서는 화면에서 ID와 비밀번호를 입력합니다. JSP는 이 데이터를 `form`으로 받아 POST로 컨트롤러에 전달합니다. 컨트롤러는 ID와 비밀번호를 Service에 넘겨 검증하고, 문제가 없으면 로그인 정보를 Session에 저장합니다. 이런 흐름은 대략 아래처럼 생깁니다.

```java
// URL은 /login, 메서드는 POST
@RequestMapping(value = "/login", method = RequestMethod.POST)
public String login(User user, HttpServletRequest request) {
    // Service에서 사용자 정보를 가져온다
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // 요청에서 세션을 꺼낸다
    HttpSession session = request.getSession();
    // 세션에 사용자 ID를 저장한다
    session.setAttribute("userId", loginedUser.getId());
    return "/";
}
```

로그인이 되면 다른 메서드에서 Session을 확인해 로그인 상태를 판단하고, 로그아웃할 때는 Session을 삭제하면 됩니다.

```java
// URL은 /logout, 메서드는 GET
@RequestMapping(value = "/logout", method = RequestMethod.GET)
public String logout(HttpServletRequest request) {
    // 요청에서 세션을 꺼낸다
    HttpSession session = request.getSession();
    // 세션을 무효화한다
    session.invalidate();
    return "/";
}
```

Spring Security를 쓰면 코드가 조금 달라지긴 하지만, Session을 사용하는 기본 흐름은 같습니다. 이런 방식으로 MVC 패턴에서는 인증과 인가를 처리할 수 있습니다.

## 여기서 문제가 생긴다

하지만 Session 방식은 REST API에서는 그대로 쓰기 어렵습니다. REST API의 핵심 특징 중 하나가 `Stateless`, 즉 상태를 가지지 않는다는 점이기 때문입니다. 여기서 상태란 서버가 클라이언트의 상태를 유지하는 것을 말합니다. 기존 방식에서는 클라이언트가 서버에 접속하는 순간부터 서버가 그 상태를 관리해야 했습니다. Session은 바로 그런 상태를 관리하기 위한 장치이므로, REST API에는 잘 맞지 않습니다. 물론 억지로 적용할 수는 있지만, REST의 철학과는 어긋납니다.

Stateless이기 때문에 REST API는 클라이언트 요청마다 필요한 모든 정보를 함께 보내야 합니다. 따라서 로그인 정보도 Session에 올려 두는 대신, 요청마다 자신이 로그인 상태라는 것을 증명하는 데이터를 어떤 형태로든 보내야 합니다.

### 그럼 어떻게 처리할까

클라이언트가 요청마다 본질적인 데이터뿐 아니라 인증 정보도 보내는 방법은 이미 HTTP에 있습니다. 바로 `Header`입니다. 요청 본문에는 작업에 필요한 데이터를 두고, 인증 정보는 Header에 넣으면 됩니다.

클라이언트가 Header에 인증 정보를 넣으려면 먼저 서버 쪽에서 인증을 해 줘야 합니다. 이 인증의 구체적인 내용은 다음 글에서 다루기로 하고, 여기서는 Header에 실을 데이터를 서버가 어떻게 돌려줄지에 집중하겠습니다.

Spring Boot에서 응답 Header에 데이터를 넣는 방법은 두 가지가 있습니다. 앞에서 본 것처럼 응답을 다루는 `HttpServletResponse`를 직접 쓰는 방법과 `ResponseEntity`로 응답을 감싸는 방법입니다.

## `HttpServletResponse`로 Header 추가

먼저 `HttpServletResponse`를 쓰는 방법입니다. 단순히 응답에 데이터를 실어 보내고 싶을 때 쓰기 좋습니다. MVC 패턴에서 로그인할 때와 마찬가지로, 로그인 메서드의 인자로 `HttpServletResponse`를 받으면 응답 Header에 데이터를 넣을 수 있습니다.

```java
@PostMapping("/login")
public User login(User user, HttpServletResponse response) {
    // Service에서 사용자 정보를 가져온다
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // 응답 Header에 사용자 ID를 넣는다
    response.addHeader("userId", loginedUser.getId());
    // Body를 반환한다(Header는 자동 포함된다)
    return loginedMember;
}
```

## `ResponseEntity`로 Header 추가

다른 방법인 `ResponseEntity`를 쓰면, 인자를 사용자 정보(ID와 비밀번호)만 받으면 됩니다. 이름에서도 알 수 있듯이 `ResponseEntity`는 Body와 Header, HTTP Status(예: 200 OK)까지 함께 다룰 수 있는 클래스입니다. 사용법은 간단합니다. 로그인 메서드의 반환값인 Body를 `ResponseEntity`로 감싸서 Header 정보와 함께 돌려주면 됩니다.

```java
@PostMapping("/login")
public ResponseEntity<Member> login(User user) {
    User loginedUser = service.getUser(user.getId(), user.getPassword());
    // 응답의 Header와 Body를 함께 설정해 반환한다
    return ResponseEntity.ok().header("userId", loginedUser.getId()).body(loginedUser);
}
```

## 그다음 단계

응답 Header에 넣은 정보는 프런트엔드 쪽에서 요청할 때마다 다시 Header에 실어 보내게 됩니다. 그런데 요청마다 보내려면 어딘가에 그 정보를 저장해야 합니다. 쿠키를 쓰는 방법도 있지만 보안상 위험이 있어서, 많은 경우 브라우저의 로컬 스토리지에 저장하는 방식이 쓰입니다. 이 부분은 프런트엔드 영역이므로 여기서는 깊게 다루지 않겠습니다.

로그아웃은 어떻게 하느냐는 문제도 있습니다. 프런트엔드에서는 Header에 담을 인증 정보를 삭제하거나 잘못된 정보를 보내는 식으로 처리할 수 있습니다. 서버 쪽에서는 일정 시간이 지나면 인증 정보를 더 이상 쓸 수 없도록 만드는 방식도 있습니다.

## 마지막으로

REST API 로그인에는 아직 고민해야 할 점이 더 있습니다. 예를 들어 HTTP Header에 어떤 인증 정보를 넣어야 하는지, 그 데이터를 어떻게 만들지, 요청마다 Header로 인증 정보를 주고받는 방식이 보안과 리소스 측면에서 괜찮은지 같은 문제입니다.

그래도 이번 글로 REST API에서 로그인 흐름을 어떻게 바라봐야 하는지는 어느 정도 정리할 수 있었습니다. 다음 글에서는 이 Header와 Spring Security를 이용해 실제 JWT 로그인 구현으로 이어가 보겠습니다.
