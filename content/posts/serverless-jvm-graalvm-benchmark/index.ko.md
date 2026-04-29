---
title: "서버리스 환경에서 JVM과 GraalVM Native Image를 비교해 봤다"
date: 2026-04-29
translationKey: "posts/serverless-jvm-graalvm-benchmark"
categories:
  - java
image: "../../images/java.webp"
tags:
  - java
  - jvm
  - graalvm
  - serverless
  - spring
  - ktor
  - quarkus
---

서버리스 환경에서는 첫 요청에 얼마나 빨리 응답할 수 있는지가 꽤 중요합니다. 항상 프로세스가 데워져 있는 서버와 달리, 콜드 스타트가 느리면 그 지연이 그대로 사용자 경험에 드러납니다. 메모리 사용량도 비용과 직접 연결되기 쉽습니다.

예전에 [JVM은 아직 진화한다](../jvm-evolution/)라는 글에서 Leyden이나 Loom 등에 대해 썼습니다. JVM은 단지 오래된 런타임이 아니라, 시작 시간, warm-up, 메모리 효율을 개선하는 방향으로 지금도 진화하고 있습니다. 특히 Java 25에서는 Leyden과 관련된 개선도 조금씩 들어오기 시작했기 때문에, 이번에는 "GraalVM Native Image를 쓰면 실제로 얼마나 달라질까", "Java 25의 JVM도 어디까지 버틸 수 있을까"를 서버리스에 가까운 조건에서 비교해 봤습니다.

비교에 사용한 리포지토리는 [Serverless](https://github.com/retheviper/Serverless)입니다. 측정 결과 HTML 요약은 [serverless-summary.html](https://github.com/retheviper/Serverless/blob/main/docs/reports/serverless-summary.html)에 있습니다.

## 무엇을 비교했는가

이번에는 같은 HTTP API와 같은 PostgreSQL 접근을 가진 애플리케이션을 아래 구성으로 비교했습니다.

| 대상 | 런타임 |
|---|---|
| Quarkus | JVM Java 25 |
| Quarkus | GraalVM Native Image Java 25 |
| Spring Boot | JVM Java 25 |
| Spring Boot | GraalVM Native Image Java 25 |
| Ktor | JVM Java 25 |
| Ktor | GraalVM Native Image Java 25 |

Ktor Kotlin/Native도 별도 항목으로 넣었지만, DB 포함 비교 대상에는 넣지 않았습니다. 이유는 단순합니다. JDBC는 JVM API이기 때문에 JVM / GraalVM Native Image와 같은 PostgreSQL 경로를 공유할 수 없습니다. 현재로서는 실험적인 서버 시작 측정으로 보는 편이 맞다고 생각합니다.

API 형태는 다음처럼 맞췄습니다.

| Endpoint | 처리 |
|---|---|
| `GET /health` | JSON health response |
| `GET /api/items/{id}` | PostgreSQL point read와 JSON object serialization |
| `GET /api/items?limit=50` | PostgreSQL ordered list read와 JSON array serialization |
| `GET /api/items/report` | PostgreSQL aggregate query와 JSON array serialization |

단순히 "프로세스가 시작됐다"만 보고 싶었던 것이 아니라, DB 연결, JSON 직렬화, DI/component graph 초기화, HTTP server startup까지 포함하고 싶었습니다. 그래서 주요 지표는 `apiReadyMillis`로 잡았습니다. 이는 `/api/items/1`이 실제로 응답할 때까지 걸린 시간입니다.

## 측정 결과

HTML 요약의 중앙값을 보면 결과는 꽤 분명했습니다.

| Target | Median API ready ms | Peak RSS MiB | Median steady RSS MiB |
|---|---:|---:|---:|
| Quarkus GraalVM Native | 81 | 41.2 | 27.3 |
| Spring Boot GraalVM Native | 118 | 98.9 | 60.4 |
| Ktor GraalVM Native | 243 | 34.9 | 28.9 |
| Ktor JVM | 641 | 160.4 | 134.1 |
| Quarkus JVM | 710 | 151.1 | 129.1 |
| Spring Boot JVM | 1716 | 332.3 | 270.7 |

가장 눈에 띄는 것은 역시 GraalVM Native Image의 시작 속도입니다. Quarkus Native는 중앙값 81ms, Spring Native는 118ms, Ktor Native는 243ms였습니다. JVM 버전도 Java 25를 사용했기 때문에 예전 JVM과 비교하면 상당히 개선되어 있을 테지만, 그래도 콜드 스타트에서는 Native Image의 차이가 크게 드러납니다.

메모리도 같은 경향입니다. Ktor Native와 Quarkus Native는 steady RSS가 30MiB 전후에 머물렀습니다. 반면 JVM 버전은 Ktor와 Quarkus가 130MiB 전후, Spring Boot는 270MiB 전후였습니다. 서버리스처럼 메모리 할당이 과금이나 동시 실행 수에 영향을 주는 환경에서는 이 차이를 무시하기 어렵다고 봅니다.

## 프레임워크별 차이

Quarkus는 이런 용도에 꽤 강하다는 인상을 받았습니다. Native Image에서는 시작 시간도 메모리도 작고, JVM 버전에서도 Spring Boot보다 훨씬 가볍게 나왔습니다. Quarkus는 원래 빌드 시점 처리를 강하게 활용하는 설계라서 GraalVM Native Image와 궁합이 좋을 것이라고 예상했지만, 실제 DB 포함 API ready에서도 꽤 좋은 숫자가 나왔습니다.

Spring Boot는 JVM 버전에서 가장 무겁게 나왔습니다. 이것은 Spring이 나쁘다기보다, DI, auto configuration, application context 초기화의 두께가 그대로 콜드 스타트에 반영된 것으로 보는 편이 자연스럽습니다. 다만 Native Image로 만들면 118ms까지 줄어들었고, Spring AOT와 GraalVM의 효과는 꽤 컸습니다.

Ktor는 JVM 버전에서는 Quarkus JVM과 가까운 위치였지만, Native Image에서는 Quarkus나 Spring보다 느리게 나왔습니다. 이번 Ktor Native에서는 CIO engine을 사용했고, Ktor의 runtime serializer lookup에 기대지 않고 `kotlinx.serialization` serializer를 명시적으로 호출하도록 했습니다. 또한 Hikari의 native-image metadata 문제를 피하기 위해 DB 포함 타깃에서는 리포지토리 안에 작은 eager JDBC pool을 구현했습니다. 즉 Ktor 자체뿐 아니라 Native Image 대응을 위한 구현상의 제약도 숫자에 포함되어 있습니다.

## 구현상의 제약

GraalVM Native Image는 단순히 JVM 애플리케이션을 그대로 컴파일하면 끝나는 것이 아니었습니다.

이번에도 몇 가지 제약이 있었습니다.

- Quarkus native build에서는 일반 JAR package와 native executable을 같은 빌드에서 동시에 만들 수 없다
- Spring native build에서는 Spring AOT와 GraalVM Native Build Tools plugin을 사용해야 한다
- Kotlin Spring application에서는 Spring AOT가 Kotlin metadata를 읽기 위해 `kotlin-reflect`가 필요하다
- Ktor GraalVM native는 CIO engine을 사용해야 한다
- Ktor native에서는 serializer lookup이나 Hikari 주변을 그대로 쓰기 어려운 부분이 있다
- Kotlin/Native는 JDBC를 쓸 수 없으므로 DB 포함 JVM/GraalVM 비교와는 별도 항목이 된다

이 부분은 벤치마크 숫자를 볼 때 중요하다고 생각합니다. Native Image는 분명 빠르고 가볍지만, 그 대신 build time, reflection, dynamic proxy, serializer, JDBC driver, framework integration 등을 의식해야 합니다.

물론 Quarkus나 Spring Boot는 이 부분을 꽤 많이 흡수해 줍니다. 그래서 프레임워크의 Native Image 대응도가 실제 사용성 차이로 이어지기 쉽습니다. Ktor처럼 얇은 프레임워크는 직접 신경 써야 할 부분이 늘어나는 대신, 어디서 무엇을 하는지는 더 잘 보이는 면도 있습니다.

## 예전 5개 스택 비교와의 차이

예전에 [Spring과 Ktor, 5개 구성을 비교해 봤다](../spring-ktor-five-stack-benchmark/)에서는 같은 PostgreSQL을 사용해 Spring MVC JDBC, Spring MVC virtual thread, Spring WebFlux R2DBC, Ktor JDBC, Ktor R2DBC를 비교했습니다.

그때는 throughput을 중심으로 봤기 때문에, "요청을 계속 받는 상태에서 무엇이 강한가"가 주요 관심사였습니다. 결과적으로는 Spring MVC + JDBC 같은 전통적인 구성이 꽤 강했고, 비동기나 R2DBC가 항상 이기는 것은 아니라는 인상이었습니다.

이번에는 거기서 관점을 바꿔 서버리스적인 콜드 스타트와 메모리를 봤습니다. 즉 정상 상태의 req/s가 아니라, 프로세스를 시작해서 첫 DB-backed JSON response를 반환할 때까지의 시간입니다.

이 둘은 같은 성능 비교라도 보고 있는 대상이 다릅니다. 상주 서버라면 throughput이나 latency distribution이 중요하지만, 서버리스나 scale-to-zero에 가까운 환경에서는 cold start와 RSS가 꽤 큰 의미를 갖습니다. 그래서 이전 결과와 이번 결과는 모순된다기보다, 서로 다른 축을 보고 있다고 생각하는 편이 좋겠습니다.

## 환경에 따른 차이는 있을 수 있다

이번 측정 환경은 arm macOS입니다. Native binary는 host-specific이므로 x64 Linux에서 같은 차이가 난다고 단정할 수는 없습니다.

실제 서버리스 환경은 대부분 Linux이고, CPU 아키텍처, container runtime, filesystem, network, PostgreSQL까지의 거리, 메모리 제한, 프로세스 시작 방식에 따라 결과는 달라질 수 있습니다. 특히 Native Image는 빌드된 OS / architecture의 영향을 받기 때문에, 최종적으로는 배포 대상에 가까운 환경에서 측정해야 합니다.

또한 이번에는 `SERVERLESS_DEPENDENCY_COUNT`의 기본값을 180으로 두고, DI가 많은 애플리케이션을 어느 정도 흉내 냈습니다. 다만 실제 Spring 애플리케이션은 더 많은 bean, auto configuration, configuration properties, external clients, security filter, metrics 등을 가질 수 있습니다. DI 대상이 늘어날수록 Spring Boot JVM의 시작 시간과 메모리에 더 큰 영향을 줄 수 있고, Native Image 쪽에서도 reachability metadata나 AOT 처리의 복잡성이 늘어날 수 있습니다.

즉 이번 숫자는 "이 조건에서는 이랬다"는 실측값이지, 모든 Spring / Ktor / Quarkus 애플리케이션에 그대로 일반화할 수 있는 것은 아닙니다.

## 마지막으로

이번 결과만 보면, 서버리스 환경에서 즉각적인 응답성과 메모리 효율을 중시한다면 GraalVM Native Image는 꽤 유력합니다. 특히 Quarkus Native와 Spring Native는 DB 연결과 JSON 응답까지 포함해도 매우 짧은 시간 안에 ready 상태가 되었습니다.

반면 JVM도 아직 끝난 선택지는 아니라고 생각합니다. Java 25를 사용한 JVM 버전에서도 Ktor와 Quarkus는 1초 미만에 API ready가 되었고, Leyden 같은 개선이 계속 진행되면 앞으로 차이가 더 줄어들 가능성도 있습니다. JVM의 유연성, 디버깅의 쉬움, 기존 라이브러리와의 궁합을 생각하면 단순히 Native Image만 고르면 된다는 이야기도 아닙니다.

다만 콜드 스타트와 메모리 숫자만 보면, 지금 시점에서는 Native Image의 강점이 꽤 분명했습니다. 상주 서버라면 이전 5개 스택 비교처럼 throughput을 봐야 하지만, 서버리스에서는 또 다른 판단 기준이 필요합니다.

결국 어떤 환경에서, 얼마나 자주 cold start가 발생하고, 메모리 비용을 어느 정도 신경 쓰는지에 따라 달라집니다. 이번 [Serverless](https://github.com/retheviper/Serverless)는 그런 판단을 위한 자료 중 하나로, 앞으로도 조금 더 키워 보고 싶습니다.

그럼 또!
