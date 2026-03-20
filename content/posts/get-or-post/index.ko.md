---
title: "GET인가, POST인가"
date: 2020-09-21
categories:
  - recent
image: "../../images/magic.webp"
tags:
  - blog
  - http
translationKey: "posts/get-or-post"
---

[MDN 설명](https://developer.mozilla.org/ko/docs/Web/HTTP/Methods)에 따르면 HTTP 메서드는 기본적으로 "리소스에 대한" 동작이라고 정의됩니다. 그래서 GET은 리소스의 표현, POST는 리소스 생성, PUT은 리소스 치환, DELETE는 리소스 삭제처럼 설명하곤 합니다. 이런 내용을 바탕으로 CRUD를 설명하는 자료도 많아서, 자연스럽게 API 설계도 생성, 조회, 수정, 삭제를 기준으로 생각하게 됩니다. 그리고 요청 처리의 [멱등성](https://ko.wikipedia.org/wiki/%EB%A9%B1%EB%93%B1%EC%84%B1)도 중요하게 다뤄집니다.

## 이론과 현실

이론적으로는 이렇게 이해해도, 실제 업무에서 애플리케이션으로 풀어보면 고민이 생깁니다. 이번에 그런 경우가 있었습니다. "겉으로 보기에는 단순히 리소스를 돌려주는 것 같지만, 내부적으로는 생성이나 수정이 함께 일어나는 처리는 어떻게 해야 할까?"라는 의문이 있었습니다. 이유는 다음과 같은 요구사항 때문이었습니다.

- 클라이언트에 어떤 파일의 다운로드를 제공하는 API가 있다.
- 서버에서 파일을 가져오는 경우는 두 가지다.
  - 파일이 이미 생성되어 있고, 서버는 그 파일을 그대로 반환하기만 한다.
  - 요청에 따라 서버가 DB 레코드를 파일로 만들어 클라이언트에 반환한다.
- 클라이언트가 파일을 다운로드하면 서버는 DB를 업데이트한다.
  - 서버는 DB에 "파일 출력 완료" 플래그와 "갱신 사용자" 정보를 기록한다.

"이미 파일이 만들어져 있고 서버는 그걸 그대로 돌려줄 뿐"이라면 GET으로 충분합니다. 클라이언트 입장에서도 단순히 파일을 받는 것이고, 서버도 이미 존재하는 리소스를 반환할 뿐입니다. 일반적으로 GET에 기대하는 "리소스의 표현"과도 잘 맞습니다.

그렇다면 "요청에 따라 서버가 DB 레코드를 파일로 만들어 반환하는" 경우는 어떨까요? 파일을 새로 만드는 것은 "리소스 생성"에 해당할까요? 아니면 이미 존재하는 레코드를 다른 형태로 가공하는 것일 뿐이라서 "리소스의 표현"에 해당할까요? 게다가 파일 요청뿐 아니라 클라이언트가 전달한 정보를 바탕으로 DB도 업데이트하므로, 생성에 가까워 보이기도 합니다.

## 누구의 관점인가

이 문제는 백엔드 엔지니어이자 애플리케이션 관점에서의 고민입니다. 그렇다면 클라이언트, 즉 사용자 관점에서는 어떨까요? 다음처럼 생각할 수 있습니다.

- 클라이언트는 파일이 이미 만들어져 있는지, 새로 만들어지는지에는 관심이 없다.
  - 요청은 "파일을 달라"는 것이지, "파일을 만들어 달라"는 것이 아니다.
- 파일을 만들지 여부는 서버의 내부 사정이다.
  - 클라이언트는 파일을 요청할 뿐, 내부 처리 방식까지 알 필요는 없다.

즉 클라이언트 입장에서는 파일을 다운로드하는 행위가 결국 "리소스의 표현"을 요청하는 것에 가깝습니다. 서버 쪽 사정과는 다르죠. 이런 경우에는 어느 쪽 기준을 우선할지가 중요합니다. 저는 우선순위는 클라이언트라고 봅니다. 애플리케이션은 애초에 클라이언트의 요구를 만족시키기 위해 존재하기 때문입니다. 그렇다면 서버 내부의 사정 때문에 요청을 POST로 바꿀 이유는 없습니다.

## 스펙에서는

다만 클라이언트 관점만으로 "이 경우는 GET이 맞다"라고 단정하려면 또 다른 근거가 필요합니다. 예를 들어 HTTP 표준이 있습니다. HTTP 스펙에는 [다음과 같은 절](https://tools.ietf.org/html/rfc7231#section-4.2.1)이 있습니다.

> 4.2.1. Safe Methods
>> Request methods are considered "safe" if their defined semantics are essentially read-only; i.e., the client does not request, and does not expect, any state change on the origin server as a result of applying a safe method to a target resource. Likewise, reasonable use of a safe method is not expected to cause any harm, loss of property, or unusual burden on the origin server.

> 4.2.1 안전한 메서드
>> 클라이언트가 서버의 상태 변화를 요구하지 않고, 변화도 기대하지 않는 읽기 전용 요청은 "안전"하다고 본다. 안전한 메서드를 올바르게 사용하면 서버에 해나 손해가 되는 상황을 일으키지 않는다는 뜻이다.

이 설명만 보면 클라이언트 요청 때문에 서버 상태가 바뀌는 경우는 GET이 아니어야 할 것처럼 보입니다. 하지만 중요한 건 그 다음 문장입니다.

>> This definition of safe methods does not prevent an implementation from including behavior that is potentially harmful, that is not entirely read-only, or that causes side effects while invoking a safe method. What is important, however, is that the client did not request that additional behavior and cannot be held accountable for it. For example, most servers append request information to access log files at the completion of every response, regardless of the method, and that is considered safe even though the log storage might become full and crash the server.

>> 안전한 메서드라고 해서, 완전히 읽기 전용이 아니거나 부작용을 일으킬 수 있는 구현까지 막는 것은 아니다. 중요한 것은 그 동작이 클라이언트의 요청이 아니라는 점이며, 그 책임을 클라이언트에게 돌릴 수 없다는 것이다. 예를 들어 대부분의 서버는 모든 메서드에 대해 액세스 로그를 남기지만, 로그 때문에 저장 공간이 가득 차 서버가 장애를 일으킬 수 있어도 요청 자체는 안전하다고 본다.

결국 중요한 것은 구현상의 사정은 서버의 책임이라는 점입니다. 안전한 요청인지 판단할 때는 클라이언트가 어떤 요청을 했는지를 봐야 합니다. HTTP 메서드는 그런 행위의 본질을 나타내는 것이지, 서버 내부 사정을 표현하는 수단은 아닙니다. 따라서 클라이언트가 "리소스의 표현"을 요청하는 상황이라면 서버는 GET으로 응답하는 것이 맞다고 봅니다.

## 마지막으로

실제 웹사이트를 보면, 순수한 의미의 "리소스 표현"으로서의 GET은 오히려 많지 않은 것 같습니다. 예를 들어 방문만 해도 방문자 수가 늘어나는 사이트도 있습니다. 이 경우 클라이언트는 리소스 생성이나 갱신을 요청한 것이 아니고, 서버가 내부적으로 상태를 바꾸는 것입니다. 그럼에도 GET으로 동작해도 크게 이상하지 않습니다. 만약 이런 변경까지 모두 POST로 처리한다면, GET을 쓸 장면이 거의 사라질지도 모릅니다.

다만 그렇다고 해서 리소스 생성이나 갱신이 필요한 요청을 전부 GET으로 처리하는 것도 위험합니다. 원칙으로 돌아가 보면 "POST로 생성한 뒤 GET으로 조회한다"는 방식도 있을 수 있습니다. 이 경우 트랜잭션은 두 번 발생하니 성능상 불리할 수 있지만, 상황에 따라서는 더 안전한 설계가 될 수도 있습니다.

결국 중요한 건 "모든 경우에 무조건 GET"도 아니고 "무조건 POST"도 아니라는 점입니다. 애플리케이션 전체의 설계와 향후 방향을 함께 보고, 그 요청을 클라이언트가 어떻게 인식하는지가 무엇보다 중요합니다. 원론적인 이야기처럼 들리지만, 이런 고민이 API 설계에서는 가장 오래 남습니다.
