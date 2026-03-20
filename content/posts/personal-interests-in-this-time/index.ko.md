---
title: "최근 눈여겨보고 있는 것들"
date: 2020-06-29
categories: 
  - recent
image: "../../images/magic.webp"
tags:
  - apple
  - mac
  - deno
  - javascript
  - c#
  - dotnet
  - flutter
  - rust
translationKey: "posts/personal-interests-in-this-time"
---

이 업계에서는 늘 공부를 계속해야 한다고들 하지만, 변화 속도가 워낙 빠르다 보니 무엇을 따라가야 하고 무엇을 믿어야 할지 헷갈릴 때가 있습니다. 새로운 언어가 계속 등장하는 와중에, 예전에는 성능 문제 때문에 주목받지 못했던 언어가 갑자기 인기를 얻기도 하고, 당연하다고 여겼던 패러다임이 뒤집히기도 합니다. 지금은 JavaScript의 시대라고 해도 과언이 아닌 것 같지만, 앞으로는 또 어떻게 바뀔지 모르겠습니다. 그래서 이번에는 제가 개인적으로 흥미롭게 보고 있는 IT 업계 흐름을 몇 가지 정리해 보려고 합니다.

## Apple Silicon Mac

얼마 전 Apple의 개발자 행사인 [WWDC2020](https://developer.apple.com/videos/play/wwdc2020/101)이 열렸습니다. 매년 꼭 라이브로 보는 것은 아니지만, 이번에는 재택근무 덕분에 통근 시간이 사라진 덕분에 처음부터 끝까지 실시간으로 볼 수 있었습니다.

iOS, iPadOS, macOS의 변화도 물론 흥미로웠지만, 이번 행사에서 가장 인상적이었던 건 역시 Mac의 메인 프로세서가 [Apple 자체 칩으로 전환된다는 발표](https://www.apple.com/newsroom/2020/06/apple-announces-mac-transition-to-apple-silicon)였습니다. 저는 PowerPC 시절의 Mac은 직접 써 보지 못했지만, PPC에서 Intel로 넘어갈 때도 전환은 꽤 성공적이었다고 알고 있어서 이번에도 결국은 잘 자리잡지 않을까 생각합니다. 개인적으로 iPad도 쓰고 있는데, Intel CPU보다 성능이 아쉽다는 인상을 받은 적은 거의 없었습니다.

물론 Boot Camp를 쓸 수 없게 될 가능성이 있고, x86 기반의 서드파티 앱이 문제를 일으킬 수도 있기 때문에 새 프로세서 Mac이 출시되자마자 바로 기존 Mac을 교체할 생각은 없습니다. 가격도 만만치 않고요. 그래도 Apple이 [LLVM](https://llvm.org), [Catalyst](https://developer.apple.com/mac-catalyst) 같은 기술에 오랫동안 투자해 온 것을 보면, 생각보다 빠르고 매끄럽게 전환이 진행될 수도 있겠다는 느낌이 듭니다. 특히 저는 주로 웹 애플리케이션을 다루기 때문에 low-level 기술에 직접 닿을 일이 많지 않고, 결국 사용하는 언어의 컴파일러만 새 프로세서를 잘 지원해 주면 되는 경우가 많습니다. WWDC에서도 Xcode 프로젝트는 다시 빌드만 하면 된다는 설명이 있었죠.

다만 궁금한 점은 제가 쓰고 있는 Intel Mac에 대한 macOS 지원이 언제까지 이어질지 정도입니다. 최근 Windows도 꽤 좋아지고 있지만, 제 작업 환경에서는 굳이 Windows로 옮겨야 할 이유는 없기 때문에 Apple 쪽 전환이 잘 된다면 크게 문제는 없을 것 같습니다. 그리고 이번 변화는 단순히 CPU 교체가 아니라 SoC 전환이기 때문에, iPad나 iPhone에서 이미 쓰이던 센서나 뉴럴 엔진 같은 요소를 Mac에서도 더 적극적으로 활용할 수 있다는 점도 큰 장점으로 보입니다.

물론 이런 감상은 어디까지나 개발자인 제 입장에서 본 이야기입니다. 엔터프라이즈 환경에서는 아무리 [Rosetta 2](https://www.apple.com/newsroom/2020/06/apple-announces-mac-transition-to-apple-silicon)와 [Universal Binary 2](https://developer.apple.com/documentation/xcode/building_a_universal_macos_binary)가 잘 만들어졌다고 해도, 어디선가 호환성이나 성능 문제가 생길 수 있다고 생각합니다. 그 부분은 당연히 조심해서 봐야겠죠.

또 $500에 A12Z 기반 개발 키트를 빌릴 수 있다는 이야기도 있었기 때문에, 새 프로세서 Mac의 성능과 호환성 문제는 생각보다 빨리 드러날지도 모르겠습니다. 우선은 그 부분을 가장 주의 깊게 보고 싶습니다. 성능뿐 아니라 발열과 전력 소모가 얼마나 좋아졌는지도 꽤 궁금합니다.

## Deno

저는 JavaScript를 아주 깊게 다루는 편은 아니라 Node.js에도 엄청 익숙한 편은 아닙니다. 그래도 요즘 웹 애플리케이션을 이야기할 때 Node.js를 빼놓기 어려운 시대라는 건 분명합니다. 제 경우에는 JavaScript보다 TypeScript 쪽에 더 좋은 인상을 받아 왔기 때문에, Node 생태계에서도 TypeScript를 더 자연스럽게 쓸 수 있으면 좋겠다고 생각하고 있었습니다. 그런 흐름 속에서 Node 창시자가 만든 [Deno](https://deno.land)라는 새 런타임이 등장한 건 꽤 흥미로운 일이었습니다.

기본적으로는 Node의 반성점들을 바탕으로 다시 설계한 런타임처럼 보입니다. 그리고 무엇보다 TypeScript 컴파일러를 내장하고 있어서, 별도로 JavaScript로 트랜스파일하지 않고도 TypeScript를 바로 다룰 수 있다는 점이 개인적으로는 가장 큰 장점처럼 느껴졌습니다.

다만 Deno에도 당연히 약점은 있습니다. 기존 Node.js 모듈을 그대로 쓸 수 없다는 점, 그리고 TypeScript 컴파일이 아직 느리다는 점이 대표적입니다. Rust로 자체 TypeScript 컴파일러를 만들 계획이 있다는 이야기도 있었지만, 언제 완성될지는 알 수 없기 때문에 당분간은 Deno 기반 프로젝트가 아주 빠르게 늘어나기는 어렵지 않을까 싶습니다.

## Blazor

5월에는 [Microsoft Build 2020](https://news.microsoft.com/build2020)에서 [Blazor WebAssembly 정식 릴리스](https://devblogs.microsoft.com/aspnet/blazor-webassembly-3-2-0-now-available)가 발표됐습니다. 이제 .NET과 C#으로 브라우저에서 실행되는 웹 애플리케이션을 만들 수 있게 된 셈입니다.

Node.js 기반 웹 애플리케이션의 장점으로 자주 언급되는 것이 "서버와 클라이언트를 하나의 언어로 개발할 수 있다"는 점인데, Blazor를 보면 꼭 JavaScript를 써야만 그런 구성이 가능한 것은 아니라는 생각이 듭니다. 물론 JavaScript도 좋은 언어지만, 언어 자체의 한계가 분명한 부분도 있기 때문에 C# 같은 컴파일 언어를 브라우저에서 활용할 수 있다는 점은 꽤 매력적입니다.

게다가 Blazor 계열로 PWA를 구현할 수 있는 Blazor PWA, Electron과 WebView 기반 데스크톱 앱을 만들 수 있는 Blazor Hybrid, HTML 요소 없이 네이티브 앱을 지향하는 Blazor Native 같은 흐름도 이어질 예정이라고 하니, 다른 컴파일 언어들도 브라우저 실행 환경 쪽으로 더 확장되지 않을까 기대하게 됩니다.

WSL이나 GitHub까지 포함해서 보면, 여러 의미에서 최근 Microsoft의 변화와 투자는 정말 인상적입니다.

## Flutter for web

[Flutter](https://flutter.dev)는 iOS와 Android를 동시에 개발할 수 있고, React Native와 비교해도 네이티브 앱에 더 가까운 성능을 낼 수 있다는 점이 강점으로 자주 언급됩니다. 그런데 최근에는 Flutter를 이용한 웹 애플리케이션도 조금씩 늘어나고 있는 것 같습니다. Flutter의 언어인 [Dart](https://dart.dev)가 애초에 차세대 JavaScript를 목표로 출발했다는 점을 생각하면 이상한 흐름은 아닙니다.

이렇게 하나의 언어로 모바일과 웹을 모두 다룰 수 있다는 점은 Blazor와 비슷한 맥락에서 확실히 매력적입니다. 다만 개인적으로는 Google이 만들고 있다는 점 때문에, "차라리 Dart 대신 Kotlin이었으면 어땠을까?" 하는 생각도 듭니다. 그리고 Google은 자사 서비스를 꽤 빠르게 접는 경우도 있어서 Flutter가 얼마나 오래 생태계를 유지할 수 있을지에 대한 불안도 조금 남아 있습니다.

Microsoft의 Xamarin이 아주 강한 인상을 남기지 못한 상황이라, 모바일 쪽에서는 Flutter를 선택하는 팀이 더 많아질 수도 있겠다는 생각은 듭니다. 하지만 웹 애플리케이션까지 포함하면 저는 여전히 C#을 활용할 수 있는 Blazor 쪽이 더 매력적으로 보입니다.

## Rust

[Rust](https://www.rust-lang.org)는 흔히 post-C, post-C++ 언어로 언급되는데, 최근 인기를 보면 정말 무서울 정도입니다. 아직 엔터프라이즈 레벨에서는 기존 시스템 자산이나 숙련된 인력 부족 등의 이유로 Rust가 널리 쓰이고 있지는 않은 것 같지만, C/C++에 가까운 성능을 내면서도 더 안정적이라는 점이 가장 큰 장점으로 자주 이야기됩니다.

저는 평소 웹 애플리케이션 위주로 작업하기 때문에 시스템과 직접 맞닿는 개발을 자주 하지는 않습니다. 그래도 PWA처럼 웹에서도 데스크톱 수준 성능을 요구하는 경우가 생기고, Java에서는 직접 다루기 애매한 바이너리 파일 처리 같은 한계를 느낄 때가 있어서, Rust 같은 언어를 다룰 수 있으면 더 좋은 애플리케이션을 만들 수 있지 않을까 하는 생각이 듭니다.

특히 [Discord](https://discord.com)가 원래 쓰던 [Golang](https://golang.org)을 일부 영역에서 Rust로 교체했다는 이야기를 보고 더 흥미가 생겼습니다. Go 역시 C/C++의 대안으로 자주 언급되는 언어인데, 그럼에도 Rust로 옮길 만큼의 장점이 무엇인지 궁금해졌기 때문입니다.

Kotlin도 결국 Java와의 높은 호환성이 있었기 때문에 빠르게 자리잡을 수 있었다고 생각합니다. 그런데 Rust는 그런 호환성조차 거의 없는 언어인데도 이 정도 주목을 받는다는 점에서, 그 장점이 무엇인지 한 번 제대로 이해해 보고 싶다는 생각이 듭니다.

## 마지막으로

새로운 기술과 변화가 끊임없이 나오는 걸 보고 있으면 기대감과 압박감이 동시에 듭니다. 그래도 전반적으로는 모두 긍정적인 방향의 변화라고 생각합니다. 특히 각 언어가 발전해 가는 모습을 보면, Java의 모토였던 `Write once, run everywhere`가 이제는 더 넓고 발전된 형태로 여러 언어에서 구현되고 있는 것처럼 느껴집니다.

결국 점점 더 "무슨 언어를 쓰느냐"보다 "그 언어로 무엇을 할 수 있느냐"가 중요해지는 시대가 오고 있는 것 같습니다. 그래서 지금은 일단 이런 변화들을 따라갈 수 있도록 제 기술을 계속 다듬는 것이 더 중요하다고 생각합니다. 요즘은 구현할 시간이 많지 않지만, 개인 프로젝트라도 조금씩 진행하면서 이런 흐름을 직접 경험해 보고 싶습니다.
