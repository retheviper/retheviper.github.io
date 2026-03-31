---
title: "Spring과 Ktor, 5가지 구성을 비교해 봤다"
date: 2026-03-20
categories:
  - kotlin
image: ../../images/kotlin.webp
tags:
  - spring
  - ktor
  - benchmark
  - r2dbc
  - jdbc
  - virtual-thread
  - kotlin
translationKey: "posts/spring-ktor-five-stack-benchmark"
---

최근 [Exposed](https://www.jetbrains.com/exposed/)가 1.0부터 R2DBC를 정식 지원하게 되면서, Ktor에서도 DB 접근까지 포함한 전체 흐름을 논블로킹으로 구성하기가 한층 쉬워졌습니다. 원래 Ktor는 코루틴 기반으로 작성할 수 있으니, 여기에 R2DBC까지 더하면 "Kotlin다운 구성으로도 높은 동시 처리 성능을 꽤 자연스럽게 낼 수 있지 않을까?" 하는 기대가 생깁니다.

반면 Spring도 Spring Boot 3 계열부터 가상 스레드를 꽤 다루기 쉬워졌습니다. 그렇다면 전통적인 Spring MVC + JDBC 구성과 WebFlux + R2DBC 같은 리액티브 구성을 어떻게 비교해야 할지도 궁금해집니다. 여기에 Ktor + JDBC, Ktor + R2DBC까지 넣어 보면 프레임워크, 시리얼라이저, ORM, 드라이버 차이가 더 잘 드러날 것 같았습니다.

그래서 이번에는 같은 PostgreSQL, 같은 HTTP 시나리오를 기준으로 아래 5가지 구성을 동일한 조건에서 비교해 봤습니다.

## 비교 대상

이번에 비교한 구성은 다음 다섯 가지입니다.

| 구성 | 주요 스택 |
|---|---|
| Spring MVC JDBC (platform) | Tomcat, Jackson, jOOQ, JDBC, HikariCP, platform threads |
| Spring MVC JDBC (virtual) | Tomcat, Jackson, jOOQ, JDBC, HikariCP, virtual threads |
| Spring WebFlux R2DBC | Netty, Jackson, jOOQ SQL builder, R2DBC PostgreSQL, R2DBC pool |
| Ktor JDBC | Netty, kotlinx.serialization, Koin, Exposed JDBC, HikariCP |
| Ktor R2DBC | Netty, kotlinx.serialization, Koin, Exposed R2DBC, R2DBC PostgreSQL |

여기서 Spring MVC는 애플리케이션 자체는 동일하고, 가상 스레드만 켜고 끄는 방식입니다. Ktor는 JDBC 버전과 R2DBC 버전을 분리해 "Ktor 자체의 차이"인지, "R2DBC + Exposed R2DBC 경로의 차이"인지를 구분해서 보기 쉽게 했습니다.

개인적으로는 Ktor + `kotlinx.serialization` + Exposed처럼 JetBrains 계열 라이브러리로 통일된 구성을 꽤 좋아합니다. 코드 양도 비교적 적게 유지할 수 있고, Kotlin답게 작성하는 느낌도 강하니까요. 다만 취향과 성능은 별개이니, 이번에는 그 차이를 숫자로 확인해 보고 싶었습니다.

## 테스트 시나리오와 측정 방법

이번 비교는 단순히 "무엇이 더 빠른가"만 보려는 목적은 아니었습니다. 어느 레이어에서 차이가 생기는지 나눠 보기 쉽도록 시나리오를 몇 가지 유형으로 준비했습니다. DB를 쓰지 않는 가벼운 응답, 단순 조회, 큰 페이로드, 검색/집계에 가까운 읽기, 그리고 쓰기와 경합 시나리오입니다.

각 시나리오의 의도는 대략 다음과 같습니다.

| 분류 | 시나리오 | 확인하려는 점 |
|---|---|---|
| non-DB | `GET /health` | 가장 가벼운 HTTP 경로. 라우팅과 최소한의 응답 처리 차이가 드러나기 쉽다 |
| small payload | `GET /api/books/bench/small-json` | DB 없이 작은 JSON을 반환. 프레임워크와 시리얼라이저 자체의 가벼움을 보기 좋다 |
| large payload | `GET /api/books/bench/large-json?limit=200` | 큰 JSON 생성과 직렬화 비용을 본다 |
| point read | `GET /api/books/{id}` | PK로 1건 조회하는 전형적인 CRUD read |
| list read | `GET /api/books?limit=50` | 비교적 가벼운 목록 조회 |
| large list read | `GET /api/books?limit=500` | 건수가 많을 때의 materialize와 응답 생성 비용 |
| search read | `GET /api/books/search?q=Book&limit=20` | 조건이 붙는 조회 |
| aggregate-like read | `GET /api/books/bench/aggregate-report?limit=50` | 집계 성격의 읽기 경로 |
| write | `POST /api/checkouts` | 일반적인 재고 감소 + checkout 생성 |
| write contention | `POST /api/checkouts` (`bookId=1` 고정) | 같은 row를 반복 갱신할 때의 경합 |
| distributed write | `POST /api/checkouts` (랜덤 bookId) | 충돌을 어느 정도 분산한 쓰기 |

특히 `checkout` 계열은 해석에 주의가 필요합니다. 내부적으로는 "책 조회 -> 재고 확인 -> 재고 차감 -> checkout insert"라는 쓰기 트랜잭션을 수행하기 때문에, 단순 JSON 응답보다 커넥션 풀, 트랜잭션, 락 경합의 영향을 훨씬 크게 받습니다.

측정 조건은 아래와 같습니다.

- PostgreSQL 17을 매번 컨테이너로 띄우고 실행마다 초기화
- 같은 벤치마크 러너에서 동일한 HTTP 시나리오 실행
- 워밍업 1초, 측정 3초
- 동시 실행 수는 128과 256 두 패턴
- Spring MVC는 HikariCP 최대 32, Spring WebFlux와 Ktor R2DBC는 R2DBC pool 최대 32, Ktor JDBC는 HikariCP 최대 32

이번에는 프레임워크만 떼어내기보다, 실제로 많이 쓰는 조합 전체를 비교하고 싶었습니다. 그래서 프레임워크뿐 아니라 시리얼라이저, DI, ORM, 드라이버까지 포함해 그대로 측정했습니다. 즉, 이 결과는 "순수한 Ktor vs Spring"이라기보다 "이 조합으로 애플리케이션을 구성했을 때의 전체적인 차이"에 가깝습니다.

## 측정 결과

세부 수치는 마지막에 소개할 HTML 요약을 보는 편이 빠르지만, 먼저 인상적이었던 경향만 정리하면 다음과 같습니다.

### 1. 작은 JSON은 혼전이었고, 큰 JSON에서는 Spring MVC가 강했다

`smallJson`은 이제 한 스택이 일방적으로 앞선다고 보기 어려웠습니다. 동시 실행 128에서는 Ktor R2DBC 27,979 req/s, Ktor JDBC 27,790 req/s, Spring WebFlux R2DBC 27,737 req/s로 거의 붙어 있었고, 동시 실행 256에서는 Spring MVC platform이 26,531 req/s로 가장 높았지만 Spring WebFlux R2DBC 26,309 req/s, Ktor R2DBC 25,047 req/s도 아주 멀지는 않았습니다. 이 구간은 DB 접근이 없어서 프레임워크와 직렬화 경로의 순수한 비용이 드러나지만, 상위권끼리는 차이가 생각보다 크지 않았습니다.

반면 `largeJson`에서는 차이가 더 분명했습니다. 동시 실행 256 기준으로 Spring MVC platform이 3,273 req/s, Spring MVC virtual이 2,822 req/s였고, Ktor JDBC는 2,376 req/s, Spring WebFlux R2DBC는 2,174 req/s, Ktor R2DBC는 877 req/s였습니다. 큰 JSON 직렬화에서는 여전히 Jackson 쪽이 더 유리해 보이고, 특히 Ktor R2DBC는 여기서 격차가 크게 벌어졌습니다.

### 2. DB 읽기에서는 예상보다 Spring MVC가 강했다

핵심이라 생각했던 읽기 계열을 보면 `bookById`, `bookList`, `bookList500`, `bookSearch` 대부분에서 Spring MVC 두 구성이 상위권이었습니다. 예를 들어 동시 실행 256에서는 아래와 같은 결과가 나왔습니다.

| 시나리오 | 1위 | 2위 | Ktor JDBC | Ktor R2DBC |
|---|---|---|---|---|
| `bookById` | Spring MVC platform 12,315 | Spring MVC virtual 11,053 | 7,766 | 4,679 |
| `bookList` | Spring MVC virtual 7,821 | Spring MVC platform 7,458 | 5,076 | 2,446 |
| `bookList500` | Spring MVC virtual 2,108 | Spring MVC platform 2,035 | 1,388 | 466 |
| `bookSearch` | Spring MVC platform 8,098 | Spring MVC virtual 7,158 | 5,454 | 3,801 |

처음에는 코루틴 기반에 R2DBC까지 갖춘 Ktor R2DBC가, 적어도 읽기에서는 꽤 강하게 나올 수도 있겠다고 예상했습니다. 그런데 실제 결과는 오히려 반대였고, 상당히 정석적인 Spring MVC + JDBC가 기준점으로 아주 강했습니다.

가상 스레드도 여전히 흥미롭지만, 이번 결과에서는 platform과 virtual이 읽기 계열에서 번갈아 앞섰습니다. `bookById`와 `bookSearch`는 platform이, `bookList`와 `bookList500`은 virtual이 앞섰고, `aggregateReport`도 동시 실행 256에서 Spring MVC virtual 1,020 req/s, platform 948 req/s, Ktor JDBC 917 req/s로 거의 붙어 있었습니다. 가상 스레드는 여전히 유력한 옵션이지만, 켠다고 해서 모든 상황에서 자동으로 더 빨라진다고 보기는 어렵습니다.

### 3. Ktor JDBC는 나쁘지 않았지만, Ktor R2DBC는 꽤 고전했다

Ktor 내부 비교만 봐도 JDBC 버전과 R2DBC 버전의 차이는 이번에도 꽤 분명했습니다. 동시 실행 256 기준으로 `bookList500`은 Ktor JDBC가 1,388 req/s, Ktor R2DBC가 466 req/s였고, `largeJson`은 Ktor JDBC가 2,376 req/s, Ktor R2DBC가 877 req/s였습니다. `checkoutCreate`에서는 Ktor JDBC 2,959 req/s, Ktor R2DBC 2,941 req/s로 사실상 비슷했지만, 적어도 "R2DBC로 바꾸면 전체적으로 더 유리하다"는 결과는 아니었습니다.

이 차이를 보면 Ktor의 병목이 단순히 "Ktor 자체"에만 있다고 보기는 어렵고, Exposed R2DBC를 포함한 비동기 DB 접근 경로가 꽤 큰 영향을 주는 것처럼 보입니다. Spring WebFlux 역시 비DB 경로에서는 매우 강했지만, DB read에서는 JDBC 기반 Spring MVC를 뚜렷하게 앞서는 장면이 많지 않았습니다. 적어도 이번처럼 CRUD 중심 워크로드에서는 Reactive Streams 기반 DB 접근이 기대만큼 곧바로 성능 우위로 이어지지는 않았다고 보는 편이 자연스럽습니다.

### 4. hotspot 계열은 req/s만 보면 큰 의미가 없다

`checkoutHotspot`은 조금 특수합니다. 같은 책의 재고를 계속 깎다 보니 중간부터 실패가 대량으로 발생합니다. 그래서 이 구간은 순수 req/s보다 "실제로 성공한 요청이 얼마나 되는가"를 함께 봐야 합니다.

숫자만 보면 동시 실행 256에서 Ktor JDBC가 3,248 req/s로 가장 높지만, 실패 수도 9,816 있습니다. Spring WebFlux는 2,312 req/s에 실패 6,995, Spring MVC platform은 2,093 req/s에 실패 6,327, Ktor R2DBC는 1,375 req/s에 실패 4,177이었습니다. 즉, 이 시나리오는 여전히 "빠르다"는 수치 안에 "빠르게 실패한다"가 상당히 섞여 있으므로, 일반 시나리오와는 별도로 해석해야 합니다.

### 5. aggregateReport는 생각보다 큰 차이가 없었다

`aggregateReport`는 이번에는 특정 스택 하나가 크게 튄다기보다, 상위권이 꽤 촘촘하게 붙었습니다. 동시 실행 128에서는 Ktor JDBC가 984 req/s로 가장 높았고 Spring WebFlux R2DBC 973, Spring MVC virtual 971, Spring MVC platform 962가 거의 비슷했습니다. 동시 실행 256에서는 Spring MVC virtual 1,020 req/s, Spring MVC platform 948, Ktor JDBC 917, Spring WebFlux R2DBC 893 순이었습니다. 적어도 이번 결과만 놓고 보면, 이 시나리오는 "예외적으로 Ktor JDBC가 압도한다"기보다는 여러 구현이 비슷한 구간으로 보는 편이 맞겠습니다.

## 여기서 얻은 점

이번 결과에서 가장 인상적이었던 것은 "논블로킹이면 RPS가 더 높을 것이다"라는 기대가 생각만큼 단순하게 맞아떨어지지 않았다는 점이었습니다. 물론 논블로킹은 중요하고, 특정 워크로드에서는 아주 효과적일 수 있습니다. 다만 이번처럼 비교적 정석적인 CRUD 중심 벤치마크에서는 Spring MVC + JDBC 같은 전통적인 구성이 여전히 매우 강했습니다.

또 `kotlinx.serialization`은 작은 JSON에서는 꽤 선전하지만, 큰 JSON에서는 Jackson과의 차이가 더 드러나는 듯했습니다. 예전부터 이야기되던 경향이 이번 측정에서도 어느 정도 확인된 셈입니다.

R2DBC 역시 적어도 이번 조건에서는 "JDBC를 분명하게 대체할 만큼 빠르다"라고 말하기는 어려웠습니다. 설계 측면에서는 깔끔할 수 있어도, DB 접근이 포함된 실제 애플리케이션에서는 다른 종류의 오버헤드와 락 동작까지 같이 평가해야 한다는 인상을 받았습니다.

## 마지막으로

개인적인 취향만 놓고 보면, Ktor와 `kotlinx.serialization`, Exposed처럼 Kotlin 네이티브한 조합은 여전히 꽤 매력적입니다. 코드 스타일도 통일되고, 작성하는 감각도 좋습니다. 다만 AI 도움으로 코드를 작성하는 비용이 예전보다 낮아진 지금은, "얼마나 적은 코드로 쓸 수 있느냐" 못지않게 "최종적으로 얼마나 빠르게 동작하느냐"도 더 중요하게 봐야 하지 않을까 싶습니다.

그런 의미에서 성능을 우선한다면 지금도 Spring MVC 같은 전통적인 구성이 충분히 유력합니다. 반대로 코드 양, 가독성, Kotlin다운 작성 경험을 더 중시한다면 Ktor를 선택하는 것도 충분히 의미가 있습니다. 실제로 Ktor JDBC는 read 계열과 일부 write 계열에서 충분히 경쟁력이 있었고, 애플리케이션 특성에 따라서는 충분히 실용적일 수 있습니다.

세부 측정 결과는 Benchmark 리포지토리의 [README](https://github.com/retheviper/Benchmark/blob/main/README.md)와 [five-stack-summary.html](https://github.com/retheviper/Benchmark/blob/main/docs/reports/five-stack-summary.html)을 확인하는 편이 가장 빠릅니다. 개요, 목적, 비교 대상 스택, 측정 방법은 README에 정리돼 있으니, 시나리오별 수치를 더 자세히 보고 싶다면 그쪽을 참고해 주세요.
