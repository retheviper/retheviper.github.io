---
title: "JVM은 아직 진화한다"
date: 2025-07-08
categories:
  - java
image: "../../images/java.webp"
tags:
  - jvm
  - java
  - kotlin
  - spring
translationKey: "posts/jvm-evolution"
---

현대의 개발 환경에서는 Go나 Rust처럼 가벼운 런타임이 많이 받아들여지면서, JVM의 방향성에 의문을 제기하는 목소리도 늘었습니다. 저 역시 비슷한 생각을 한 적이 있습니다. 특히 클라우드 환경에서는 JVM의 기동 시간과 메모리 사용량이 문제가 되는 경우가 적지 않아서, 서버리스에서는 Python이나 Node.js를 고르는 일도 있었습니다.

그렇다고 JVM이 끝난 것은 아닙니다. 오히려 지금도 새 프로젝트를 통해 성능을 더 끌어올릴 여지가 계속 열리고 있습니다. 이 글에서는 Leyden과 Loom 같은 주요 프로젝트를 중심으로, JVM의 성능이 앞으로 어디까지 좋아질 수 있는지 살펴보겠습니다.

## JIT에서 AOT + PGO로

[JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation)은 바이트코드를 실행 중에 네이티브 코드로 컴파일하는 방식입니다. 런타임에서 프로파일을 보면서 최적화할 수 있다는 장점이 있지만, 워밍업 시간이 필요하고 실행 초반 성능이 들쭉날쭉하다는 단점도 있습니다.

이런 문제를 더 줄이기 위해 주목받는 것이 [Graal](https://www.graalvm.org/)입니다. Graal은 고성능 JIT 컴파일러로, inlining, escape analysis, vectorization 같은 최적화를 통해 기존 JIT보다 더 안정적인 성능을 노립니다.

실제로 [Twitter 사례](https://www.oracle.com/a/ocom/docs/graalvm-twitter-casestudy-constellation.pdf)에서는 Graal JIT 도입으로 CPU 사용률을 8~11% 줄였고, Kotlin 마이크로벤치마크에서는 [18% 성능 향상](https://martijndwars.nl/2020/02/24/graal-vs-c2.html)이 보고된 바 있습니다.

## Leyden: 빠른 기동과 안정성

[Project Leyden](https://openjdk.org/projects/leyden/)은 JVM 애플리케이션의 기동 시간과 워밍업 시간을 본질적으로 줄이는 것을 목표로 합니다.

기존 JIT 기반 최적화는 실행 중에 프로파일을 모으면서 점진적으로 성능을 끌어올리는 방식이었습니다. 반면 Leyden은 이를 사전에 준비된 정적 최적화 쪽으로 옮기려 합니다.

이를 위해 Leyden은 `condensers`라는 개념을 도입합니다. 이 아카이브에는 다음 정보가 들어갑니다.

* 사용 클래스 목록과 linking 정보
* 초기 힙 상태(AppCDS + heap snapshot)
* 프로파일 정보(PGO)
* 컴파일된 코드

이렇게 되면 애플리케이션은 시작하자마자 JIT 이후와 비슷한 성능을 기대할 수 있고, 실행 초반의 흔들림도 줄일 수 있습니다.

예를 들어 [Quarkus](https://quarkus.io/) 팀은 Leyden의 초기 구현을 바탕으로 다음과 같은 개선을 보고했습니다.

* 기동 시간 50% 이상 단축
* Native Image에 가까운 응답 속도 확보
* 메모리 사용량 최대 30% 감소

Leyden과 관련된 JEP으로는 [JEP 483](https://openjdk.org/jeps/483)과 [JEP 515](https://openjdk.org/jeps/515) 등이 있고, JDK 25 이후 순차적으로 반영될 예정입니다.

## ZGC, Shenandoah, 그리고 LXR

GC는 JVM 성능에 큰 영향을 줍니다. 최근의 [ZGC](https://wiki.openjdk.org/display/zgc/Main)와 [Shenandoah GC](https://wiki.openjdk.org/display/shenandoah/Main)는 짧은 pause를 목표로 설계됐고, ZGC는 큰 힙에서도 1ms 미만의 GC pause를 노립니다.

연구 단계의 [LXR GC](https://www.steveblackburn.org/pubs/papers/lxr-pldi-2022.pdf)는 ZGC보다 더 나은 성능을 보여 주며, tail latency를 크게 줄이고 throughput을 높였다는 보고도 있습니다.

[Cassandra 벤치마크](https://www.datastax.com/blog/apache-cassandra-benchmarking-40-brings-heat-new-garbage-collectors-zgc-and-shenandoah)에서는 Shenandoah가 p99 레이턴시를 77% 줄였다고 합니다. 이런 결과는 작은 서비스부터 큰 JVM 애플리케이션까지, 지터를 줄이면서 처리량을 유지하는 데 의미가 있습니다.

## Valhalla의 메모리 효율

[Project Valhalla](https://openjdk.org/projects/valhalla/)는 `value class`를 JVM에 도입해 힙 객체 의존을 줄이고 캐시 지역성을 높이려는 프로젝트입니다.

일반 객체는 힙에 흩어져 있고 헤더도 필요해서 GC와 캐시 미스에 부담이 됩니다. 반면 value class는 구조체처럼 인라인될 수 있어서 메모리 사용 효율을 높일 수 있습니다.

특히 Kotlin의 data class는 Valhalla의 영향을 크게 받을 수 있습니다. 배열이나 벡터 계산 같은 성능 민감한 영역에서는 [큰 향상](https://www.reddit.com/r/java/comments/1dnhgut/i_actually_tested_jdk_20_valhalla_here_are_my/)도 기대할 수 있습니다.

## Loom으로 스케일 가능한 I/O 동시성

[Project Loom](https://openjdk.org/projects/loom/)은 가상 스레드를 통해, 기존의 블로킹 I/O 코드를 거의 그대로 유지하면서도 많은 동시 요청을 처리할 수 있게 하려는 시도입니다.

여기서 중요한 건 기존의 병렬 처리 방식과의 차이입니다.

기존 Java, 그리고 Spring MVC는 요청마다 OS 스레드를 하나씩 사용하는 구조였습니다. 요청 수가 많아지면 스레드 고갈이나 컨텍스트 스위칭 비용이 쉽게 문제가 됩니다.

이 문제를 해결하기 위해 등장한 것이 [Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)입니다. WebFlux는 [Reactor](https://projectreactor.io/) 기반의 논블로킹 비동기 모델로, 적은 수의 스레드로 많은 요청을 처리할 수 있습니다.

하지만 그 대가로 개발자는 `Mono`와 `Flux`를 이해해야 하고, 비동기 파이프라인을 직접 구성해야 하며, 디버깅도 복잡해집니다. Kotlin은 `suspend` 함수 덕분에 비동기 처리를 비교적 간단하게 쓸 수 있지만, Java에서는 여전히 코드가 복잡했습니다.

Loom의 가상 스레드는 WebFlux 수준의 확장성을 가지면서도, 기존처럼 동기식 코드로 작성할 수 있다는 점에서 큰 변화입니다. 코드도 기존 스레드 기반 코드와 거의 동일하게 유지됩니다.

Java 21에서는 가상 스레드가 정식 도입됐고, [Spring MVC의 VirtualThreadTaskExecutor](https://docs.spring.io/spring-framework/reference/integration/virtual-threads.html) 같은 예도 나왔습니다. 기존 Servlet 기반 구조를 크게 바꾸지 않고도 가상 스레드를 도입할 수 있고, [WebFlux보다 낮은 지연 시간과 높은 처리량을 보인다는 비교](https://github.com/chrisgleissner/loom-webflux-benchmarks)도 있습니다.

Netty나 Tomcat 같은 서버도 Loom 대응을 진행하고 있으니, 앞으로 가상 스레드는 표준 선택지 중 하나가 될 가능성이 높아 보입니다.

## 정리

이 글에서 소개한 JVM 개선 포인트를 요약하면 다음과 같습니다.

* [Graal](https://www.graalvm.org/)과 [Leyden](https://openjdk.org/projects/leyden/)으로 JIT 불안정성과 워밍업 시간을 줄임
* [ZGC](https://wiki.openjdk.org/display/zgc/Main), [Shenandoah](https://wiki.openjdk.org/display/shenandoah/Main), [LXR GC](https://arxiv.org/abs/2210.17175)로 낮은 지연 시간의 GC를 구현
* [Valhalla](https://openjdk.org/projects/valhalla/)로 메모리 효율을 높임
* [Loom](https://openjdk.org/projects/loom/)으로 I/O 확장성을 개선

## 마지막으로

JVM은 오래된 플랫폼이지만, 최근에는 한계만 이야기되던 시기에서 벗어나 다시 진화하고 있습니다.

특히 Leyden과 Loom 같은 프로젝트는 Java를 단순히 하위 호환이 좋은 언어가 아니라, 지금의 요구에 맞는 성능 기반으로 다시 다듬으려는 시도라고 볼 수 있습니다. Kotlin이나 Scala 같은 다른 JVM 언어들도 이런 개선의 혜택을 그대로 받을 수 있으니, 앞으로의 변화도 충분히 기대할 만합니다.

애플리케이션 개발의 패러다임과 성능 기반은 계속 바뀌고 있고, 그 다음이 무엇이 될지는 아직 알 수 없습니다. 그래도 JVM이 이렇게 계속 방향을 바꿔 가며 살아남고 있다는 사실 자체는 여전히 흥미롭습니다.

[^1]: JIT가 최적화를 하기 위해 필요한 초기 단계입니다.
[^2]: 실행 시간의 변동폭을 뜻합니다.
