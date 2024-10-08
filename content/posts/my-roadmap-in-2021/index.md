---
title: "個人的な2021年のロードマップ"
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
---

エンジニアとして働いていると会社の方針・クライアントの要求・経歴のような、自分の意思以外のところから自分の技術スタックを決まってしまう場合が決して少なくないと思います。会社は利益集団なので、致し方ないのですが、個人としてはどうでしょうか。私は、エンジニアは常にトレンドとともに前に進むべき職種なので、業務としてはあまり機会がないとしても、やはり自分で何かロードマップを立てて、独学でもスキルアップを図るべきたと思っています。

例えば私は、どんな案件でも主にJavaとSpringのサーバサイドエンジニアか、Jenkins・Shell・Linuxを触るインフラエンジニアとして働いたことが多いのですが、何か一つは自分のアプリやサービスを作ってみたいと思っています。このような目標ないしは願望がある場合、それを成すためには何が必要か、と考えるようになり、そこから適合なプラットフォームは？言語は？フレームワークは？という風に考え始めて、そのうちでもっとも自分にとって合理的な道を選ぶようになります。何が合理的か、という基準は人それぞれですが(主観だけで決められるものでもないし)、会社が自分のエンジニアとしての目標を考えてくれる可能性は低いので、とにかくこういう目標設定は自分でなすべきでしょう。

そういう意味で、今年の自分のロードマップを、「やりたいこと」と「良さそうなこと」という基準からいくつか立ててみました。まだロードマップとしては何一つ計画を具体化してないので、ただの興味に近いものなのかもしれませんが…とにかく、今の時点で興味を持っているものや考えていることについて、Google Trendを持って語ります。軽く、「こいつは2021年にこういうものに注目するんだな」と思ってください。

## 言語

### TypeScript

いつまでになるかはわかりませんが、少なくともここ数年はJavaScriptの天下が続きそうですね。ただ、なぜそうかというと、Webの標準であるという強力な基盤がある上に、今はNode.jsやElectronのおかげでブラウザ以外でも使える場面が多いから、ということだけでは説明が難しくなりつつあるという側面もある印象です。今はもはやコーディングを学び始めるきっかけや入門の言語としてJavaScriptに触れるケースが多いし、SPAの登場以来からサーバサイドよりもフロントエンドの重要さが増してきたという感覚でもありますね。アプリケーションというのは、結局はユーザのためにデザインされるものであるということを考えると、より画面に密接な言語が持つ権限の方が大きくなるのは当然なのかもしれません。

そしてバックエンドだけをみるとしても、最近はなるべくサーバサイドの役割を減らしていくか、細かく分けていく感覚ですね。[マイクロサービス](https://www.redhat.com/ja/topics/microservices)、[BFF](https://www.atmarkit.co.jp/ait/articles/1803/12/news012.html)、サーバレスのようなキーワードが流行っているのがその証拠だと思います。もちろんJavaScriptという言語そのものの発達によるものもあるとは思いますが、アプリケーションのアーキテクチャやデザインの思想そのものが変わっているので、仕方ないことです。

そこで、少なくともJavaScriptは基礎だけでもできるようにしないと、と思いました。研修などで簡単な文法については学んだことがありますが、本格的なアプリを書いた経験はあまりなかったので、少なくとも[Express](https://expressjs.com)で簡単なREST APIを作ってみるとかの経験はかなり役立つかもしれません。また、フロントエンドも少しは触れるようになるとよりいいでしょう。

このように思ったときに、目に入ってきたのがTypeScriptでした。TypeScriptは以前、Udemyの講座で接したことがあり、気に入っていましたが、最近はかなり人気を得ているらしいですね。実際どうかは、まずGoogle Trendで確認してみました。

{{< typescript_trend >}}

結果をみると、確かにTypeScriptに対する興味は日々増えていっているような気がします。おそらくAngular・React・Vueのような有名フレームワークやライブラリからTypeScriptに公式対応し始めたのも理由と思いますし、やはり静的型付けの方が生産性が上がるというところがわかったきたからなのでしょう。私はJavaから触れた人間なので、静的型付けのできるTypeScriptの方を学んだ方が良いかなと思います。

### Go or Rust

個人的には、JVM言語が好きですが、やはり高水準言語&VMを挟む構造ということもあり、より低水準に近い言語も扱ってみたいと思っています。今すぐに必要なわけではないのですが、やはりハードウェア制御やバイナリデータを扱うなど、低水準言語ならではのことがやってみたいという純粋な好奇心が理由です。最近はIOTなどでC言語の人気も高くなっていたりしますが、組み込み系ならまだしも、いわゆる「応用ソフトウェア」を開発する身としては、CやC++、もしくはそれよりも古い言語よりは、GoやRustのような言語に触れてみた方が良さそうな気もします。

ただ、悩ましいのは、それでGoとRustのうち、どれを選ぶかということです。性能だけを考えたら、当然Rustなのかもしれません。多くの場合、Rustが性能ではGoより優れていると言われていますね。実際の例として、音声チャットツールで有名な[Discord](https://discord.com/)はGoからRustに移行しましたが、これが[性能のため](https://blog.discord.com/why-discord-is-switching-from-go-to-rust-a190bbca2b1f)だったと言っています。ただ、言語を学ぶこと自体の難易度は、やはりRustの方がGoより高いらしいですね。そして一般的に、生産性の方はGoが優れていると言われています。

なので、以上のことから、自分は何をやってみたらいいのかなと思ってみた結果、CやC++に近い低水準言語の感覚としてRustを触ってみたらどうかな、と思いました。どちらもマイナーな言語ではありますが、[Stack Overflow survey](https://insights.stackoverflow.com/survey)にて、数回も「もっともエンジニアから愛された言語」として選ばれたこともあるRustの方が、これからコミュニティの成長も期待できるのではないかなと思ったからでもあります。特に[2020年の結果](https://insights.stackoverflow.com/survey/2020#technology-most-loved-dreaded-and-wanted-languages-loved)は、86.1%というすごい結果になっているくらいですので。

しかし、まだ実務レベルでよく使われているのはやはりGoの方で、リファレンスの量やエンジニアの興味という面でも(仕事で使うとしたら仕方ないのですが)、Goの方がまだ少し優勢ではないのかという気もします。Rustが最初からCやC++の代替を目標としてデザインされた言語であることに比べ、Goという言語がどこまでそのような役割ができるのかというのはまだあまりわかってないのですが、もし同じようなことできるのであれば、あえてRustにこだわる必要はないのではないかと思ったりもしますね。特に、ある言語の成長性というのは、そのコミュニティの大きさにも関係するので…なので、とりあえずGoogle Trendの方で、二つの言語に対する興味度について調べてみました。

Goの場合は一般動詞(行く)と区別するため、多くの場合`golang`で検索するケースが多いらしいです。しかし、Rustもあまり状況は変わってなくて(しかも、ゲーム名としても使われているようですね)、`rustlang`という検索語はあまり使われてないと思うので、直接的な比較が難しいですね。なので、なるべく価値中立的なキーワードとして、`go programming`と`rust programming`を選んでみました。そしてその結果が、以下です。

{{< go_rust_trend >}}

結果だけをみると、やはりRustの方がすごい人気を得ているように見えますが、まだGoの方が優位にはありますね。なので、こちらの方が(急ぎでもないので)、もう少し観望しながら、ゆっくり決めようと思っています。

### Kotlin

今はJavaのいう`Write once, run everywhere`が、どの言語でも同じようなことができていて(逆にJavaでできない分野はありますが)、それでもJVM言語は依然として魅力的だと思っています。最初からJVMがヒープを設定するのでメモリ管理という面でも安定的で、パフォーマンスも今流行りの高水準言語に比べても優秀な方ですね。また、10年以上世界でもっとも人気な言語だったので、ライブラリ・フレームワーク・リファレンスも豊富ですね。また、バイトコードだけを生成すればいいので、コンパイルする前の言語はどれでも良いです。なのでJava以外でも[Scala](https://www.scala-lang.org)、[Clojure](https://clojure.org)、[Groovy](https://groovy-lang.org)、[Jython](https://www.jython.org)、[JRuby](https://www.jruby.org)、[Ceylon](https://ceylon-lang.org/)、[Frege](https://github.com/Frege/frege)、[Eta](https://eta-lang.org)、[Haxe](https://haxe.org)のような幾多の言語がJVMを利用できるようになっているわけですね。つまり、JVMこそ死なないが、Javaという言語そのものはこれらの言語のどれかに代替できるというわけです。

そしていろんな言語の候補があるわけですが、その中でも個人的にはKotlinを選びました。近年のJavaも急激なバージョンアップを重ねながら改善されてはいるものの、実際エンタープライズレベルでそういったバージョンアップの効果を期待できるのはLTSバージョンがでた時だけですね。なので、いますぐ生産性を上げながらもJVMをそのまま利用できるという面では、Kotlinのようなモダンな言語への転換を考えるにはちょうどいい時期なのではないかと思っています。もちろん、私みたいにモバイルアプリの開発を考えているとしたら、尚更ですね。

他にもGoogle推しの言語であることや、[Kotlin/Native](https://kotlinlang.org/docs/reference/native-overview.html)や[Kotlin/JS](https://kotlinlang.org/docs/reference/js-overview.html)など他の言語でコンパイルできるという点も良いですね(実際Wantedlyでは、[すでにKotlin Multiplatformを導入](https://www.wantedly.com/companies/wantedly/post_articles/282562)しているらしいです)。そして何より、Kotlinを開発しているのがJetBrainなので、Intellijでは完璧なサポートができるというところも無視できないメリットです。ほんと少しだけですが、使ってみた鑑賞としても、完成度がかなり高い感じの言語だったので(Swiftよりも)、そのようなところからKotlinの未来はかなり明るいと思っています。

## フレームワーク & ライブラリ

### Svelte

先ほど少しJavaScriptの話をしましたが、JavaScriptそのものの需要や重要性については語るまでもないとはいうものの、そのJavaScriptのフレームワーク・ライブラリはどれが良いかという課題だけは、少なくとも数年でこれが正解と言えるような状態ではないかと思います。ここ数年で幾多のフレームワークやライブラリが生まれ、消えていってますね。幸い、いわゆるフロントエンド3強のAngular・React・Vueの中ではReactが勝者になりつつある雰囲気ではあります。Google Trendの結果も、それを見せていますね。

{{< angular_react_vue_trend >}}

しかし、フロントエンド以外の世界はまた話が違います。まだ多くのフレームワークやライブラリが乱立していて、まるで戦国時代のような様子です。こんななかでは、一体どれを選ぶべきか悩ましいし、判断のための調査だけでもかなりの時間と努力が必要となります。このような状況なので、もう数年前から流行っている言葉なのですが、[JavaScript Fatigue](https://www.google.com/search?newwindow=1&biw=1680&bih=836&sxsrf=ALeKk03Q7zTnfCMWJbsybKG4qkODOhqViA%3A1611463509708&ei=VfsMYOPbKpvahwO8zpiwCQ&q=javascript+fatigue&oq=javascript+fatigue&gs_lcp=CgZwc3ktYWIQAzIGCAAQBxAeMgIIADIECAAQHjIECAAQHjIECAAQHjIGCAAQBRAeOgQIIxAnOggIABAIEAcQHjoICAAQBRAKEB46CAgAEAgQChAeOgYIABAIEB46BAgAEBM6CAgAEAcQHhATOgoIABAHEAUQHhATUOeYAVjBwAFgnsQBaAFwAHgAgAGyA4gB8BOSAQkwLjcuMy4xLjGYAQCgAQGqAQdnd3Mtd2l6wAEB&sclient=psy-ab&ved=0ahUKEwij2sGw4bPuAhUb7WEKHTwnBpY4ChDh1QMIDQ&uact=5)(JavaScript疲労)という言葉があるくらいです。それだけ現代のJavaScriptを学ぶということは大変なことでしょう。

例えば私みたいに、ほとんどJavaScriptの経験がない人がフロントエンドエンジニアとなって、Reactがもっとも人気があるからそれをやる、と決めたら、まずNode.jsから初めて、パッケージ管理としてはnpmを使うか、yarnを使うか、言語はJavaScriptそのままにするかそれともTypeScriptにするかを決め、次に必要なものとして[Webpack](https://webpack.js.org)や[Babel](https://babeljs.io)、[Redux](https://redux.js.org)を学ぶなどと、知っておくべきものと学ぶべきものが多いです。しかも、それぞれのフレームワークやライブラリがその名前だけでは何が何だかわからなくなります。[Nuxt.js](https://ja.nuxtjs.org)はVue基盤のフレームワークだけど、[Nest.js](https://nestjs.com)はNode.js用のフレームワークですね。そして[Next.js](https://nextjs.org)はまた、React基盤のフレームワークです。この中では、一体どれを学んだらいいか、どれが良いかというのは混乱するだけです。なのでJavaScriptを扱うエンジニアが、疲労を感じるのも当然のことでしょう。

自分の場合はすでにサーバサイドの実装がある程度はできるので、フロントエンドも触れるようになって、いわゆるフルスタックとして自分一人でアプリが書けたらいいなと思っています。ただ、会社で使われているフロントエンドのフレームワークがあればそれに触れたら良いのですが、個人レベルでは何が良いかはまだ悩ましいものですね。Reactがもっとも人気だから、やはりそれを選ぶべきか？それもいい選択なのかもしれませんが、これからも本格的にフロントエンドの開発に関わるつもりではない限り、本格的にフロントエンドに時間を投資するのはもったいない気もします。そこで考えた代案が、[Svelte](https://svelte.dev)でした。

Svleteの特徴(メリット)としては、色々とありますが、私がもっとも注目したところはかなりシンプルであるというところでした。コードが短いので、書き方に慣れるのが圧倒的に早そうな気がします。そのほかは付加的なメリットとしてよく、とにかく「必要な時にサクッとかける」ものとしては、かなり良さげなものではないかなと思ったりもします。もちろん、ちゃんとしたフレームワークなので、本格的なアプリケーションを作る時も良いでしょう。

ただデメリットとしては、やはりメジャーな3強に比べてそこまで知られても、使われてもないというところです。幸い、Google Trendで確認したところ、少しづつながら注目を得ているのでこれからな気はします。

{{< svelte_trend >}}

### Flutter

今はWebアプリケーションばかり書いている私ですが、モバイルの方にも興味があり、どのような言語とフレームワークがあるかだけは把握しておきたいと思っています。そして最近は、モバイルは多くの場合ネイティブよりもハイブリッド・クロースプラットフォームの方が多くなっているような気がします。正確なデータや統計をみたわけではないのであくまで推測に過ぎないのですが、多くの場合ネイティブアプリに投資する時間や予算の余裕のないスタートアップやベンチャー企業の場合は、とりあえずハイブリッド・クロースプラットフォームを好むような印象です。もちろん、複雑な演算やOS特有の機能を使うとしたらやはりネイティブと言われていますが、個人的な経験からだと、意外とハイブリッド・クロースプラットフォームでもできることは多いのでもうOSレベルでもなく、機器固有の機能を活用する必要がなければ大体ハイブリッド・クロースプラットフォームでも事足りるのでは、と思います。

(ここで個人的な経験というものは、iOS 14から導入されたウィージェット機能を活かした簡単なアプリを作ってみたいなと思い調べたところ、OS固有の機能なので難しいのではないかと思ったものが、意外とReact NativeやFlutterでも十分できるということがわかったことです)

そして、昔はただのWebViewでできていたアプリも多かったような気がしますが、それならあえてモバイルアプリとして作る必要がないですね(PWAならわかりますが)。でもそのような形のアプリがあったからか、Webの技術から影響され生まれたバイブリッドモバイルアプリのフレームワークもかなり多いような印象です。なのでJavaScriptでコードを書いたり、JavaScriptのフレームワークを基盤にしてアプリを書けるフレームワークがかなり多いですね。例えば[Apache Cordova](https://cordova.apache.org)、[Ionic](https://ionicframework.com)、[NativeScript](https://nativescript.org)、[React Native](https://reactnative.dev)がそのようなものです。もちろんJavaScript(Web)とは違う系統、つまり伝統的なデスクトップアプリを継承している印象のフレームワークとしてC#基盤の[Xamarin](https://dotnet.microsoft.com/apps/xamarin)とDart基盤の[Flutter](https://flutter.dev)がありますね。

これだけ多いハイブリッド・クロースプラットフォームモバイルアプリ用のフレームワークですが、中でもそろそろ淘汰されてそうな技術はあります。またここでGoogle Trendの結果をみてみましょう。5つの項目しか比較ができないので、Flutterは入れてないです。

{{< mobile_frameworks_trend >}}

少なくとも、NativeScriptにはあまり興味を持っている人がいなく、XamarinやCordovaの場合もだんだん興味が下がっているのを確認できます。そうすると、残りの結果としてはIonicとReact Nativeが残りますね。先ほどフロントエンドの話を少ししましたが、最近のフロントエンド3強の勝者がReactになりそうという現実からして、Web技術に基盤したハイブリッド・クロースプラットフォームモバイルアプリ用のフレームワークは、やはりReact Nativeが適切かなと思います。

しかし、問題となるのはFlutterです。FlutterはReact Nativeと比べられる場合が多いですね。なので、FlutterともGoogle Trendで比較してみることにします。結果としてはReact nativeと比べFlutterが優勢な気がしています。

{{< flutter_react_native_trend >}}

理由として上げられるのは、どちらもiOSとAndroidアプリを同時開発できるものであるという点を踏まえると、やはりパフォーマンス問題ではなかったのかという気がします。React Nativeでは、JavaScriptからネイティブコードを呼び出すという構造から必然的にボトルネックになるしかないと言われていますので。そして、あくまで推測なのですが、Dartという別の言語を採用していながらも、JavaやC#のような言語とかなり文法が似ていて、HTMLやXMLとは違う宣言型でのUIの実装ができるというところも、Flutterならではのメリットなのではないか、という気もします。

もし自分がモバイルアプリを作るとしたら、おそらくネイティブになる可能性が高いのではないかとは思いますが(ハイブリッド・クロースプラットフォームが必要であれば、大抵Web基盤のアプリで事足りそうなので)、場合によってはハイブリッド・クロースプラットフォームも良い選択肢になるでしょう。そしてFlutterはモバイルだけでなく、より多くのプラットフォームのためのフレームワークとして成長していく予定なので、もし今から学ぶとしたらFlutterの方が良いかもしれません。もちろん、Reactがすでにできるフロントエンドエンジニアだとしたら、React Nativeの方が良いとは思いますが、それ以外の場合はやはりFlutterの方が良さそうな気がします。なので、当面はFlutterを視野に入れておきたいものです。

そのほかに、React Nativeに関しては興味深い記事がいくつかあったので、いくつかの事例を以下に記載します。

- [React Native at Airbnb](https://medium.com/airbnb-engineering/react-native-at-airbnb-f95aa460be1c)
- [React Native: A retrospective from the mobile-engineering team at Udacity](https://engineering.udacity.com/react-native-a-retrospective-from-the-mobile-engineering-team-at-udacity-89975d6a8102)
- [React Nativeを採用すべきか〜Shopifyに学ぶ](https://qiita.com/taneba/items/9903064aaaffdf041022)

## ハードウェア

### Apple silicon Mac

私はもともと20年ほどOSはMicrosoftの製品ばかりを使ってきたものです。それがたまたま、iPhoneやiPadから初めてAppleの製品に触れてから思ったよりも自分との相性がよかったので(仕事でLinuxを使っていたので気軽にターミナルを使えるという点が大きいのですが)、今後も引き続きMacを使いたいと思っています。少なくとも、自分の環境ではMacではないと困ることはあっても、Windowsでないと困ることはあまりないですので。

なので、自然にApple Silicon Macにも興味を持ったわけなのですが、やはりいきなりCPUのアーキテクチャが変わるということは、やはり互換性を担保できない問題があるので、その問題に対してAppleはどのような形で解決策を出すのだろう、という疑問を持っていました。発表直後に[色々な記事](https://jp.techcrunch.com/2020/07/11/the-real-reason-why-apple-is-putting-apple-slicon-on-the-mac)を読んでみてから予想できたのは、少なくとも「性能(演算・発熱・電力消耗を含め)はIntelより優れている」というところでしたが、それはアーキテクチャがより進んだ工程で作られているからか、それともカスタマイズによるものか、またどれだけ優れているかというのはわからない状態でした。なので「2年で移行する」という話を信じ、まずは様子を見ようとしていました。

そして今はM1チップのMacが色々とでていて、その性能も検証されていますね。確かなのは性能だけを見ると既存のIntel Macを買う理由はもはやないかのように見えます。さすがに互換性という不安要素があるのに、CPUのアーキテクチャを変えるという宣言をするぐらいのものではあると思います。しかし、やはりエンジニアたる身としては、互換性と安定性にまず目が行くものです。なぜなら、私は最初のメジャーバージョンは必ず、何かわかってない問題を抱えている可能性が高いというのを経験で実感しているからです。実際、Bluetooth問題や初期化が難しい点、スリープモードから起き上がらない問題などが一部で報告されていて、外装ディスプレイも公式的には1台しか対応しないという問題もあります(おそらく、Thunderbolt 3の大域幅の問題なのではないかと思います)。また、続々とUniversalバイナリやM1 Nativeでコンパイルされたアプリも発表されていますが、やはりまだそうではないアプリもたくさんあるので、不安ではありますね。

しかし、それでもいつかはApple Siliconに全てのMacが転換されるだろうし、いますぐM1チップ搭載モデルを購入しないとしても、十分注目する価値はあるのではないかと思っています。いや、注目だけでなく、今年は16インチMacbook Proのフルチェンジの噂もあるので、もしそれが本当なら自分も乗り換えるのではないかと思っているくらいです。もしそれが出るなら、M1チップ搭載モデルの問題としてあげられたところを改善(少なくとも、外装ディスプレイの件や[Bluetooth問題](https://9to5mac.com/2021/01/21/macos-big-sur-11-2-rc-now-available)は改善されそうです)されるはずで、今のアプリケーションのM1 Native対応の速度を見ると年内には意外と多くのアプリをNativeに使えるのではないかと思われます。まだまだこれからが注目なのですが、JavaScript中心の開発を行う方にとっては今のM1搭載モデルも十分メリットがあるのではないかと思います(AdoptOpenjdkはまだx64のみなので私は見送りですが…)。また、最近[M1搭載モデルでLinuxを使える](https://www.theverge.com/2021/1/21/22242107/linux-apple-m1-mac-port-corellium-ubuntu-details)ようになったので、ホームサーバとしてこれらのMacを考慮してみるのも良いチョイスかと思います。

## 最後に

まだJavaとSpringも全ての機能を使いこなしているとも言えない自分が、今から新しい言語やフレームワークを学ぶという計画を立てるのは無理な話なのかしれません。これはいつも悩ましい主題です。一つの言語に関する知識やスキルを極めていった方が良いか、それとも常にトレンドを追いながら幅広い分野のスキルセットを持つべきか。深い知識も、広い知識も持っていて良いものではありますが、自分がこれから積み上げるキャリアを完成するにはどれがより効率的かという疑問は解消されないものです。

自分なりの答えを出すとしたら、トレンドを追った方が、より自分の持つスキルセットの深さを増して行く方にも作用するのではないかという気はします。JavaしかできないものなのでJavaのAPIを借りて例え話をすると、Java 1.8では他の言語の持つClosureから影響されてLambdaが導入されましたね。その他にもvarの型推論やテキストブロックなどの改善もまた違う言語から影響されたものです。このような変化は、そもそもJavaの開発者たちが他の言語に注目しなかったら起こらなかったことでしょう。なので、「他と比較することで自分をより深く理解することにもなる」のではないでしょうか。そういう意味からすると、自分がすでに持っているスキルセットのみでなく、業界の動向や流行りを早くキャッチして受け入れることこそ重要ではないかと思ったりもします。

この度はだいぶ主観的な意見だけ語る場となってしましましたが、どうでしょうか。またこれから自分の考えも、トレンドも変わっていくかもしれませんが、今は私の結論が紹介できただけでよかったかなと思います。そして、こうやって色々と自分の知らない分野について調べたり勉強したりするほど、自分には何もないなと実感でき、良い刺激になります。これからも色々と勉強していかないとですね。

では、また！
