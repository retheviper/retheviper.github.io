---
title: "Refresh Token과 Sliding Session으로 JWT 보완하기"
date: 2020-08-02
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - spring security
  - rest api
  - jwt
translationKey: "posts/refresh-token-and-sliding-session"
---

지난번에는 JWT와 함께 Spring Security에서 이를 구현하는 방법에 대해 [글](../spring-rest-api-login-with-jwt)을 썼습니다. 그런데 Access Token만 발급하는 방식에는 몇 가지 취약점이 있다고 합니다. 그래서 이번에는 Access Token만 사용할 때의 특징과 문제점, 그리고 이를 보완하는 방법을 정리해 보려고 합니다.

JWT 기반 인증은 ISO 표준이라고는 해도, 정답이라고 할 만한 구현 방식이 정해져 있는 것은 아닙니다. 그래서 아래에서 소개하는 방식도 개념 위주로만 보시면 됩니다.

## Access Token

Access Token만 사용하는 경우, 최초 인증 이후에는 서버가 비밀 키로 요청에 담긴 토큰만 확인하면 되기 때문에 데이터베이스나 스토리지 I/O 없이 빠르게 처리할 수 있습니다. 하지만 토큰을 클라이언트가 가지고 있기 때문에, 서버 쪽에서 강제로 세션을 끝내는 것은 어렵습니다. 로그아웃을 하려면 결국 클라이언트에서 토큰을 지우는 수밖에 없습니다.

이 방식의 문제는, 토큰이 클라이언트에 있으니 누군가 토큰을 탈취해도 서버가 즉시 알아차릴 방법이 없다는 점입니다. 그래서 보통은 토큰에 만료 시간을 두어 일정 시간이 지나면 사용할 수 없게 만듭니다.

### 토큰 만료의 딜레마

토큰에 만료 시간을 두면, 그 시간을 얼마나 짧게 잡아야 하는지가 문제입니다. 짧게 잡으면 탈취되더라도 금방 무력화되니 안전하지만, 사용자는 자주 다시 로그인해야 합니다. 반대로 길게 잡으면 사용성은 좋아지지만 탈취된 토큰도 오래 악용될 수 있습니다.

## Refresh Token

Refresh Token은 만료된 Access Token을 갱신하기 위한 별도의 토큰입니다. 서버는 인증할 때 클라이언트에게 Access Token과 Refresh Token을 함께 발급합니다. 보통 Access Token은 30분 정도의 짧은 만료 시간을 두고, Refresh Token은 한 달 정도로 더 길게 둡니다.

클라이언트가 로그인한 뒤 30분이 지나 Access Token이 만료되면, Refresh Token으로 서버에 새 Access Token 발급을 요청합니다. 서버는 클라이언트 요청에 담긴 Refresh Token을 검증하고, 문제가 없으면 Access Token을 다시 발급합니다. 이렇게 하면 Access Token이 탈취되더라도 금방 만료되고, 사용자는 Refresh Token 덕분에 매번 로그인하지 않아도 됩니다. 서버 쪽에서 Refresh Token을 관리하므로, 문제가 생기면 해당 토큰을 무효화하는 것도 가능합니다.

다만 Refresh Token도 완벽한 해법은 아닙니다. 서버가 검증할 데이터를 따로 저장해야 하므로 추가 I/O가 생길 수 있습니다. 또 Access Token 갱신을 위해 서버와 클라이언트 양쪽에 기능을 더해야 합니다. 클라이언트가 Refresh Token을 저장해야 한다는 점도 여전히 고민거리입니다.

## Sliding Session

Sliding Session은 사용 중인 세션을 자동으로 연장해 주는 방식입니다. 요청마다 새 Access Token을 발급하는 방법도 있지만, 특정 요청에서만 갱신할 수도 있습니다. 예를 들어 사용 시간이 긴 페이지, 예를 들면 문의 작성 페이지나 장바구니에 상품을 추가하는 페이지에서 이런 방식을 쓸 수 있습니다. 또는 Access Token의 `iat`를 보고 새 토큰 발급을 요청하는 방식도 있습니다.

하지만 Sliding Session은 만료 시간이 너무 길면 사실상 세션이 무한히 연장될 수 있다는 문제가 있습니다. 반대로 사용자가 오래 작업하지 않는 서비스라면 이 기능 자체가 크게 필요 없을 수도 있습니다.

## Refresh Token + Sliding Session

마지막은 Refresh Token과 Sliding Session을 함께 쓰는 방식입니다. 일반적인 Sliding Session과 다른 점은 Access Token이 아니라 Refresh Token을 갱신한다는 것입니다. 이렇게 하면 Access Token을 자주 갱신할 필요가 줄어듭니다. 또 접근 빈도가 낮은 사용자도 비교적 자주 로그인할 필요가 없어집니다. 두 방식의 장점을 적절히 섞은 전략이라고 볼 수 있습니다.

하지만 이 방식도 완벽하지는 않습니다. 서버에서 Refresh Token을 만료시키지 않는 한 사용자는 계속 리소스에 접근할 수 있으므로, 기기 자체가 탈취되면 대응이 어렵습니다. 그래서 비밀번호를 주기적으로 변경하게 하거나, 민감한 리소스에 접근할 때는 다시 인증하게 하는 방식이 필요할 수 있습니다.

## 마지막으로

인증은 웹 애플리케이션에서 가장 중요하고도 기본적인 기능입니다. 문제는 보안을 강화할수록 사용자 경험이 나빠질 수 있다는 점입니다. 그래서 실제 서비스에서는 여러 방식이 공존하고, 각각의 장단점에 맞춰 다른 전략을 선택합니다. 정답이 하나라고 말하기는 어렵습니다.

또 JWT의 payload는 암호화되지 않기 때문에, 토큰이 탈취되면 내용이 그대로 노출될 수 있다는 한계도 있습니다. 이를 보완하기 위해 [PASETO](https://developer.okta.com/blog/2020/07/23/introducing-jpaseto) 같은 방식도 제안되고 있지만, 아직 JWT만큼 널리 쓰일지는 더 지켜봐야 합니다.

결국 보안은 암호화 방식만의 문제가 아니라, 사용자가 불편하지 않게 쓸 수 있는 인증 흐름과 운영 전략까지 함께 설계해야 하는 영역입니다. Refresh Token과 Sliding Session도 그런 고민 속에서 선택할 수 있는 현실적인 타협안이라고 볼 수 있습니다.
