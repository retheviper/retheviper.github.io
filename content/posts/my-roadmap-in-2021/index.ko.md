---
title: "개인적인 2021년 로드맵"
date: 2021-01-24
categories: 
  - recent
image: "../../images/map.webp"
tags:
  - blog
  - javascript
  - typescript
  - frontend
  - go
  - rust
  - flutter
  - mac
translationKey: "posts/my-roadmap-in-2021"
---

엔지니어로 일하다 보면 회사 방향이나 고객 요구, 지금까지 쌓아 온 경력 때문에 기술 스택이 어느 정도 정해지는 경우가 많습니다. 회사 입장에서는 자연스러운 일이지만, 개인의 장기 목표까지 그 기준에 맞춰야 하는 것은 아닙니다. 저는 엔지니어라면 업무와 별개로도 업계 흐름을 꾸준히 따라가고, 필요하면 스스로 로드맵을 세워 공부 범위를 넓혀 가야 한다고 생각합니다.

저 역시 주로 Java와 Spring 기반의 서버사이드 일이나 Jenkins, Shell, Linux를 다루는 인프라 일을 해 왔지만, 언젠가는 혼자서 앱이나 서비스를 하나쯤 만들어 보고 싶습니다. 그런 목표가 있으면 자연스럽게 “무엇이 필요할까”, “어떤 플랫폼과 언어, 프레임워크가 맞을까”를 따져 보게 됩니다. 결국 중요한 것은 자신에게 가장 합리적인 경로를 고르는 일입니다. 정답은 사람마다 다르지만, 회사가 개인의 장기 목표까지 대신 정해 주지는 않으니 목표 설정만큼은 스스로 해야 합니다.

그런 의미에서 올해는 "해 보고 싶은 것"과 "미리 익혀 두면 좋을 것"을 기준으로 몇 가지를 정리해 보기로 했습니다. 아직 구체적인 실행 계획이라기보다는 관심 목록에 가깝지만, 지금 시점에서 무엇을 눈여겨보고 있는지 남겨 둘 가치는 있다고 봤습니다.

## 언어

### TypeScript

언제가 될지는 모르겠지만, 적어도 앞으로 몇 년은 JavaScript의 영향력이 계속될 것 같습니다. 다만 지금의 JavaScript는 단순히 웹 표준 언어라는 이유만으로 설명하기 어려울 정도로 사용 범위가 넓어졌습니다. Node.js와 Electron 덕분에 브라우저 밖에서도 쓰이는 경우가 많고, 입문용 언어로 JavaScript를 배우는 사람도 많습니다. SPA 이후에는 서버보다 프런트엔드의 중요성이 더 커졌다는 느낌도 있습니다. 결국 애플리케이션은 사용자 경험을 위해 만들어지니, 화면과 더 가까운 언어가 더 큰 비중을 갖는 것은 자연스러운 흐름일지도 모릅니다.

백엔드만 보더라도 최근에는 서버 역할을 줄이거나 더 잘게 나누는 방향이 강합니다. [마이크로서비스](https://www.redhat.com/ko/topics/microservices), [BFF](https://www.atmarkit.co.jp/ait/articles/1803/12/news012.html), 서버리스 같은 키워드가 대표적입니다. 물론 JavaScript 자체의 발전도 영향을 줬겠지만, 그보다 애플리케이션 구조와 설계 방식이 바뀐 영향이 더 크다고 봅니다.

그래서 JavaScript는 최소한 기본기는 익혀 두어야겠다고 생각했습니다. 연수나 간단한 문법은 익힌 적이 있지만, 본격적인 앱을 만들어 본 경험은 많지 않았습니다. 적어도 [Express](https://expressjs.com)로 간단한 REST API 정도는 직접 만들어 봐야 도움이 될 것 같습니다. 프런트엔드도 조금은 다룰 수 있으면 더 좋겠고요.

이런 생각을 하다가 눈에 들어온 것이 TypeScript였습니다. 예전에 Udemy 강의로 한 번 접해 봤는데 인상이 좋았습니다. 최근에는 더 큰 인기를 얻고 있는 것 같아서, 실제로 어떤 흐름인지 Google Trend로 확인해 봤습니다.

{{< typescript_trend >}}

결과를 보면 TypeScript에 대한 관심이 실제로 꾸준히 늘고 있는 것처럼 보입니다. Angular, React, Vue 같은 주요 프레임워크와 라이브러리가 TypeScript를 공식 지원하기 시작한 것도 이유일 것이고, 정적 타입이 생산성에 도움이 된다는 점이 널리 받아들여진 결과이기도 하겠죠. 저는 Java를 먼저 접한 사람이라, 정적 타입을 사용할 수 있는 TypeScript 쪽이 더 잘 맞을 것 같다고 생각합니다.

### Go or Rust

개인적으로는 JVM 언어를 좋아하지만, 한편으로는 더 낮은 수준에 가까운 언어도 다뤄 보고 싶습니다. 당장 필요해서라기보다, 하드웨어 제어나 바이너리 데이터를 다루는 일처럼 저수준 언어만의 재미를 경험해 보고 싶은 호기심이 큽니다. 요즘은 IoT 덕분에 C 언어의 존재감도 여전하지만, 제가 하는 일처럼 일반적인 응용 소프트웨어 개발 관점에서는 C/C++보다 Go나 Rust가 더 현실적인 선택처럼 느껴집니다.

문제는 그 둘 중 무엇을 먼저 선택하느냐입니다. 순수 성능만 보면 Rust가 더 매력적일 수 있습니다. 실제로 [Discord](https://discord.com/)도 성능상의 이유로 [Go에서 Rust로 옮겼다고](https://blog.discord.com/why-discord-is-switching-from-go-to-rust-a190bbca2b1f) 했습니다. 반면 학습 난이도는 Rust가 더 높다고 알려져 있고, 실무 생산성은 보통 Go가 더 낫다고들 합니다.

고민 끝에 우선 Rust를 먼저 건드려 보는 쪽이 더 끌렸습니다. C나 C++에 가까운 감각을 익히는 데도 도움이 될 것 같고, [Stack Overflow survey](https://insights.stackoverflow.com/survey)에서 여러 차례 "가장 사랑받는 언어"로 꼽혔을 만큼 커뮤니티 성장세도 인상적이었기 때문입니다. 특히 [2020년 결과](https://insights.stackoverflow.com/survey/2020#technology-most-loved-dreaded-and-wanted-languages-loved)의 86.1%는 꽤 강하게 남았습니다.

다만 아직 실무에서는 Go가 더 널리 쓰이고 있고, 참고 자료나 개발자 관심도 면에서도 Go가 조금 더 앞서는 느낌입니다. Rust가 C/C++의 대체재를 목표로 설계된 언어라는 점에 비해 Go가 어디까지 같은 역할을 할 수 있는지는 아직 확신이 없습니다. 그래도 비슷한 문제를 풀 수 있다면 굳이 Rust에만 집착할 필요는 없겠죠. 언어의 성장성은 결국 커뮤니티 크기와도 연결되니, 우선 Google Trend로 두 언어의 관심도를 비교해 봤습니다.

Go는 일반 동사와 구분하려고 `golang`으로 검색하는 경우가 많고, Rust도 게임 이름과 겹치는 탓에 `rustlang`이 널리 쓰이진 않는 것 같습니다. 그래서 비교가 조금 더 중립적이라고 생각한 `go programming`과 `rust programming`을 기준으로 삼았습니다.

{{< go_rust_trend >}}

그래프만 보면 Rust가 더 인기 있어 보이지만, 아직은 Go도 충분히 강세입니다. 그래서 이건 급하게 결론 내리지 말고 조금 더 지켜보면서 정하려고 합니다.

### Kotlin

요즘은 Java의 `Write once, run everywhere`가 다른 언어들에서도 어느 정도는 비슷하게 구현되고 있습니다. 그럼에도 JVM 언어는 여전히 매력적입니다. JVM이 메모리 관리를 어느 정도 책임져 주기 때문에 안정적이고, 성능도 고수준 언어들 가운데서는 좋은 편입니다. 또 오랫동안 인기 언어였던 만큼 라이브러리와 프레임워크, 참고 자료가 풍부합니다. 바이트코드만 만들면 되니 Java 외에도 [Scala](https://www.scala-lang.org), [Clojure](https://clojure.org), [Groovy](https://groovy-lang.org), [Jython](https://www.jython.org), [JRuby](https://www.jruby.org), [Ceylon](https://ceylon-lang.org/), [Frege](https://github.com/Frege/frege), [Eta](https://eta-lang.org), [Haxe](https://haxe.org) 같은 다양한 언어가 JVM 위에서 돌아갑니다. 즉, JVM은 계속 살아남을 가능성이 크지만 Java라는 언어 자체는 다른 언어들로 일부 대체될 수 있습니다.

그 많은 선택지 중에서 저는 Kotlin을 택했습니다. 최근 Java도 빠르게 개선되고 있지만, 실제 엔터프라이즈 환경에서는 LTS 버전이 나오기 전까지는 체감하기 어렵습니다. 지금 당장 생산성을 높이면서 JVM도 그대로 쓰고 싶다면 Kotlin 같은 현대적인 언어로 옮겨 가기 좋은 시점이라고 봅니다. 특히 모바일 앱까지 생각한다면 더 그렇습니다.

Google이 밀고 있다는 점도 좋고, [Kotlin/Native](https://kotlinlang.org/docs/reference/native-overview.html)나 [Kotlin/JS](https://kotlinlang.org/docs/reference/js-overview.html)처럼 다른 플랫폼으로도 확장할 수 있다는 점도 매력적입니다. 실제로 Wantedly도 [Kotlin Multiplatform을 도입](https://www.wantedly.com/companies/wantedly/post_articles/282562)했다고 하니 가능성은 충분해 보입니다. 무엇보다 Kotlin은 JetBrains가 만들기 때문에 IntelliJ에서의 지원이 매우 좋습니다. 직접 써 본 인상도 상당히 완성도가 높았습니다. Swift보다도 더 만족스러웠던 기억이 있어서, Kotlin의 미래는 꽤 밝다고 생각합니다.

## 프레임워크 & 라이브러리

### Svelte

앞서 JavaScript 이야기를 조금 했지만, JavaScript 자체의 중요성은 이제 따로 설명할 필요가 없습니다. 다만 어떤 프레임워크와 라이브러리를 고를지는 여전히 정답이 명확하지 않은 문제입니다. 최근 몇 년 사이 수많은 도구가 생기고 사라졌고, 그 와중에 프런트엔드 3강이라 불리던 Angular, React, Vue 중에서는 React가 가장 우세한 흐름으로 보입니다. Google Trend도 그걸 보여 줍니다.

{{< angular_react_vue_trend >}}

하지만 프런트엔드 생태계는 여전히 복잡합니다. 도구가 너무 많아서, 하나를 고르기 위해 조사하는 일만으로도 적지 않은 시간과 에너지가 듭니다. 그래서 몇 년 전부터 [JavaScript Fatigue](https://www.google.com/search?newwindow=1&biw=1680&bih=836&sxsrf=ALeKk03Q7zTnfCMWJbsybKG4qkODOhqViA%3A1611463509708&ei=VfsMYOPbKpvahwO8zpiwCQ&q=javascript+fatigue&oq=javascript+fatigue&gs_lcp=CgZwc3ktYWIQAzIGCAAQBxAeMgIIADIECAAQHjIECAAQHjIECAAQHjIGCAAQBRAeOgQIIxAnOggIABAIEAcQHjoICAAQBRAKEB46CAgAEAgQChAeOgYIABAIEB46BAgAEBM6CAgAEAcQHhATOgoIABAHEAUQHhATUOeYAVjBwAFgnsQBaAFwAHgAgAGyA4gB8BOSAQkwLjcuMy4xLjGYAQCgAQGqAQdnd3Mtd2l6wAEB&sclient=psy-ab&ved=0ahUKEwij2sGw4bPuAhUb7WEKHTwnBpY4ChDh1QMIDQ&uact=5)라는 표현이 자연스럽게 쓰이게 된 것 같습니다. 그만큼 지금의 JavaScript 세계는 배우는 과정 자체가 피로할 때가 많습니다.

예를 들어 저처럼 JavaScript 경험이 거의 없는 사람이 프런트엔드 엔지니어가 되어 React를 하기로 마음먹는다면, 먼저 Node.js를 익혀야 하고, 패키지 매니저는 npm과 yarn 중 뭘 쓸지, 언어는 JavaScript로 할지 TypeScript로 할지부터 정해야 합니다. 이어서 [Webpack](https://webpack.js.org), [Babel](https://babeljs.io), [Redux](https://redux.js.org) 같은 도구도 알아야 합니다. 이름만 봐서는 역할이 바로 떠오르지 않는 프레임워크와 라이브러리도 많습니다. [Nuxt.js](https://nuxt.com)는 Vue 기반이고, [Nest.js](https://nestjs.com)는 Node.js용이며, [Next.js](https://nextjs.org)는 React 기반입니다. 이 중 무엇을 먼저 배울지 고민하다 보면 오히려 더 혼란스러워집니다. JavaScript 개발자들이 피로를 느끼는 것도 충분히 이해할 수 있습니다.

저는 이미 서버사이드 경험이 있으니, 프런트엔드까지 익혀서 혼자서 앱 하나를 끝까지 만들 수 있으면 좋겠다고 생각합니다. 회사에서 쓰는 프런트엔드 프레임워크가 있다면 그걸 먼저 배우는 게 가장 좋겠지만, 개인적으로는 어떤 도구를 선택할지 여전히 고민스럽습니다. React가 가장 인기라면 그것을 택하는 게 맞을까요? 나쁘지 않은 선택일 수는 있습니다. 하지만 앞으로 본격적으로 프런트엔드에 오래 투자할 생각이 없다면, 너무 많은 시간을 들이는 건 아깝게 느껴집니다. 그래서 대안으로 떠올린 것이 [Svelte](https://svelte.dev)였습니다.

Svelte의 장점은 여러 가지가 있지만, 제가 가장 끌린 부분은 단순함이었습니다. 코드가 짧고 구조가 직관적이라 익숙해지는 속도가 빠를 것 같았습니다. 물론 기본적인 기능 외에도 장점이 많겠지만, 가볍게 필요한 작업을 빠르게 끝내는 데는 꽤 적합해 보입니다. 물론 정식 프레임워크이기 때문에, 본격적인 애플리케이션을 만들 때도 충분히 쓸 만하다고 봅니다.

다만 단점도 있습니다. 아직은 Angular, React, Vue 같은 주류보다 덜 알려져 있고 사용 사례도 적습니다. 그나마 Google Trend를 보면 조금씩 관심이 올라가고 있어서 앞으로가 더 기대됩니다.

{{< svelte_trend >}}

### Flutter

지금은 웹 애플리케이션 위주로 개발하고 있지만, 모바일 쪽에도 관심이 있습니다. 어떤 언어와 프레임워크가 있는지 정도는 알아 두고 싶습니다. 최근 모바일 시장을 보면 네이티브보다 하이브리드나 크로스 플랫폼을 택하는 경우가 많아 보입니다. 정확한 통계를 본 것은 아니지만, 네이티브에 시간과 예산을 많이 쓰기 어려운 스타트업이나 벤처에서는 우선 크로스 플랫폼을 선호하는 분위기라고 느낍니다. 물론 복잡한 연산이나 OS 고유 기능이 필요한 경우에는 여전히 네이티브가 유리하겠지만, 기기 고유 기능을 깊게 쓰지 않는다면 크로스 플랫폼으로도 충분한 경우가 많습니다.

제 개인적인 경험으로는, iOS 14에서 도입된 위젯 기능을 활용한 간단한 앱을 만들고 싶어 조사해 봤을 때 OS 고유 기능이라서 어렵지 않을까 생각했지만, 실제로는 React Native나 Flutter로도 충분히 가능하다는 걸 알게 된 적이 있습니다.

예전에는 단순한 WebView로 만든 앱도 많았던 것 같습니다. 그렇다면 굳이 모바일 앱으로 만들 필요가 없겠죠. PWA라면 이야기가 다르지만요. 그와 비슷하게 웹 기술에서 출발한 하이브리드 모바일 프레임워크도 상당히 많습니다. 예를 들면 [Apache Cordova](https://cordova.apache.org), [Ionic](https://ionicframework.com), [NativeScript](https://nativescript.org), [React Native](https://reactnative.dev)가 있습니다. 이와는 다른 계열로, 전통적인 데스크톱 앱의 감각을 계승한 프레임워크로는 C# 기반의 [Xamarin](https://dotnet.microsoft.com/apps/xamarin)과 Dart 기반의 [Flutter](https://flutter.dev)가 있습니다.

이렇게 많은 하이브리드·크로스 플랫폼 프레임워크가 있지만, 그중에서도 도태되는 기술은 분명 있습니다. 그래서 다시 Google Trend를 봤습니다. 다만 비교 가능한 항목 수가 제한되어 있어서 Flutter는 제외했습니다.

{{< mobile_frameworks_trend >}}

적어도 NativeScript에는 관심이 많지 않고, Xamarin과 Cordova도 점점 관심이 줄고 있습니다. 남는 후보는 Ionic과 React Native입니다. 앞서 프런트엔드에서도 React가 우세해지는 흐름을 봤기 때문에, 웹 기술 기반의 크로스 플랫폼 모바일 프레임워크라면 React Native가 가장 자연스러운 선택처럼 보입니다.

문제는 Flutter입니다. Flutter는 React Native와 자주 비교됩니다. 그래서 둘도 다시 Google Trend로 비교해 봤습니다. 결과적으로는 Flutter 쪽이 더 우세해 보입니다.

{{< flutter_react_native_trend >}}

이유를 추측해 보면, 결국 iOS와 Android를 동시에 개발할 수 있다는 점에서 성능이 중요했기 때문이 아닐까 싶습니다. React Native는 JavaScript에서 네이티브 코드를 호출하는 구조라 병목이 생기기 쉽다는 이야기도 많습니다. 또 Dart라는 별도의 언어를 쓰면서도 Java나 C#처럼 문법이 익숙하고, HTML이나 XML과는 다른 선언형 UI를 쓸 수 있다는 점도 Flutter의 장점이라고 생각합니다.

제가 모바일 앱을 만든다면 아마 네이티브를 선택할 가능성이 큽니다. 크로스 플랫폼이 필요하다면 대개 웹 기반 앱으로도 충분하다고 느끼기 때문입니다. 그래도 상황에 따라서는 크로스 플랫폼이 좋은 선택이 될 수 있습니다. Flutter는 모바일을 넘어 더 많은 플랫폼으로 확장 중이니, 지금 새로 배운다면 Flutter가 더 나을지도 모릅니다. 이미 React에 익숙한 프런트엔드 개발자라면 React Native도 좋겠지만, 그렇지 않다면 Flutter가 더 적절해 보입니다. 그래서 당분간은 Flutter를 눈여겨보려고 합니다.

React Native에 대해서는 흥미로운 글도 몇 개 있어서 함께 적어 둡니다.

- [React Native at Airbnb](https://medium.com/airbnb-engineering/react-native-at-airbnb-f95aa460be1c)
- [React Native: A retrospective from the mobile-engineering team at Udacity](https://engineering.udacity.com/react-native-a-retrospective-from-the-mobile-engineering-team-at-udacity-89975d6a8102)
- [React Native를 채용해야 하는가 - Shopify를 통해 배우기](https://qiita.com/taneba/items/9903064aaaffdf041022)

## 하드웨어

### Apple silicon Mac

저는 원래 20년 가까이 Microsoft 제품만 써 왔습니다. 그러다 iPhone과 iPad를 계기로 처음 Apple 제품을 써 봤는데, 생각보다 저와 잘 맞았습니다. 업무에서 Linux를 쓰다 보니 터미널을 자연스럽게 사용할 수 있었던 점도 컸습니다. 그래서 앞으로도 Mac을 계속 쓰고 싶습니다. 제 환경에서는 Mac이 아니면 곤란한 일은 있어도, Windows가 아니면 곤란한 일은 거의 없습니다.

그래서 Apple Silicon Mac에도 자연스럽게 관심이 갔습니다. 다만 CPU 아키텍처가 갑자기 바뀌면 호환성 문제가 따라오기 마련이라, Apple이 이걸 어떻게 풀어낼지가 더 궁금했습니다. 발표 직후 여러 글을 읽고 나서 "성능, 발열, 전력 소모 면에서는 Intel보다 낫겠다"는 예상은 했지만, 그 차이가 어디서 오는지, 실제로 어느 정도 체감될지는 아직 확신이 없었습니다. 그래서 우선은 "2년 내 전환"이라는 Apple의 계획을 믿고 지켜보기로 했습니다.

지금은 M1 칩이 탑재된 Mac이 많이 나오고 있고 성능 검증도 충분히 진행되고 있습니다. 성능만 보면 기존 Intel Mac을 새로 살 이유는 거의 없어 보입니다. 아키텍처를 바꾸겠다고 선언할 정도면 Apple도 호환성 문제를 충분히 감수할 준비가 있었겠죠. 그래도 엔지니어 입장에서는 호환성과 안정성이 먼저 눈에 들어옵니다. 첫 메이저 버전은 늘 예상하지 못한 문제를 안고 있을 가능성이 높다는 것을 경험으로 알고 있기 때문입니다. 실제로 블루투스 문제, 초기 설정의 번거로움, 슬립에서 복귀하지 못하는 문제 등이 보고되었고, 외장 디스플레이도 공식적으로는 한 대만 지원합니다. 아마 Thunderbolt 3 대역폭 문제와 관련이 있을 겁니다. Universal Binary나 M1 Native로 컴파일된 앱도 빠르게 늘고 있지만, 아직은 아닌 앱도 많아서 불안한 부분이 있습니다.

그래도 언젠가는 모든 Mac이 Apple Silicon으로 바뀔 것이고, 지금 당장 M1 모델을 사지 않더라도 충분히 주목할 가치는 있습니다. 올해는 16인치 MacBook Pro의 전면 개편 이야기도 있어서, 만약 그게 사실이라면 저도 교체를 고민할 것 같습니다. 그런 모델이 나온다면 앞서 언급한 문제들, 적어도 외장 디스플레이나 [Bluetooth 문제](https://9to5mac.com/2021/01/21/macos-big-sur-11-2-rc-now-available)는 개선될 가능성이 큽니다. 현재 앱들의 M1 Native 대응 속도를 보면 연내에 꽤 많은 앱을 네이티브로 쓸 수 있을지도 모르겠습니다. 아직은 더 지켜봐야 하지만, JavaScript 중심으로 개발하는 분들에게도 M1 모델은 충분히 매력적일 수 있습니다. 다만 저는 AdoptOpenJDK가 아직 x64만 지원하는 시점이라 조금 미뤄두고 있습니다. 또 최근에는 [M1 Mac에서 Linux를 사용할 수 있게 되었으니](https://www.theverge.com/2021/1/21/22242107/linux-apple-m1-mac-port-corellium-ubuntu-details), 홈 서버 용도로 고려해 보는 것도 괜찮아 보입니다.

## 마지막으로

Java와 Spring조차 아직 완전히 능숙하다고 말하기 어려운 제가 지금 새 언어나 프레임워크를 배워 보겠다고 계획하는 건 무리일지도 모릅니다. 늘 고민되는 주제이기도 합니다. 하나의 언어를 깊게 파는 편이 좋을지, 아니면 트렌드를 따라가며 폭넓은 스킬을 갖추는 편이 좋을지. 깊이도 넓이도 모두 의미가 있지만, 앞으로 쌓아 갈 커리어에 무엇이 더 효율적인지는 쉽게 답이 나오지 않습니다.

제 나름의 결론은, 트렌드를 따라가는 일이 오히려 지금 가진 기술의 깊이도 함께 키워 준다는 것입니다. Java만 봐도 Lambda, `var`, 텍스트 블록 같은 변화는 결국 다른 언어 흐름과 무관하지 않았습니다. 다른 언어와 생태계를 함께 봐야 자신이 익숙한 기술도 더 잘 이해하게 된다는 뜻입니다. 그래서 이미 가진 스택만 붙들고 있기보다, 업계 흐름을 꾸준히 흡수하려는 태도가 중요하다고 생각합니다.

이번 글은 꽤 주관적인 이야기만 길게 적게 되었지만, 그만큼 당시 제가 무엇에 관심을 두고 있었는지를 정리해 둘 수 있어서 의미가 있었습니다. 앞으로 제 생각도 트렌드도 바뀔 수 있겠지만, 이런 기록은 나중에 다시 돌아볼 때 꽤 좋은 기준점이 됩니다.
