---
title: "Kotlinのこれからを語る"
date: 2022-05-22
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - go
  - rust
  - python
  - javascript
  - dart
---

1年ほどサーバサイドKotlinを扱いながら、ふと「今のKotlinはどこまできていて、これからはどうなるんだろう」と思うようになりました。色々な観点があると思いますが、とりあえず市場においてどれほどの需要があり、展望（これからも積極的に採用され続けそう、苦戦しそうなどの）はどうかなど、いわゆるkotlinという言語の「ステータス」について自分が感じていることについて考えてみたくなったというわけです。

最近のトレンドを見ると、一つの言語において専門家になるというよりはさまざまな言語を使いこなせる、いわゆる`Polyglot`なプログラマが求められていて、常識みたいになっているとも言われているようです。確かに私自信もその経験があるかどうかは関係なく、案件によりさまざまな言語に触れるケースを多くみています。そして今は充実したドキュメントや記事をインターネットに溢れていて、UdemyやCourseraなど良質の講義を提供するサイトも色々とあるので経験がない言語だとしても入門が難しくてできないとは言えない時代になっている感覚でもあります。なので、自分が現在使っている言語がメインストリームに属してあるかどうかの問題は以前よりは重要でなくなった、といえるかもしれません。

ただ、立場や観点によっては一つの言語に集中したい場合もあるかと思います。例えば学生や、未経験からエンジニアに転職しようとする人にいきなり二つ以上の言語を扱えるように注文するのは難しいことでしょう。エンジニアの追求する技術においてもそうです。フロントエンドエンジニアがいきなり今すぐ使う予定でもないGoやJavaのようなバックエンドで使われる言語を勉強する必要はないはずです。そして会社としては、複数の言語を扱えるエンジニアを求めるということは採用において非常に厳しい条件となるはずです。なので、依然として市場において一つの言語のステータスというのは無視できないものなのではないかと私は思っています。

というわけで、今回は多少主観的な観点からの話になりますが、他の言語や分野で、Kotlinという言語の展望について考えてみたことを述べたいと思います。それでは、どうぞ。

## vs Java

### Better javaという捉え方

Kotlin(JVM)をJavaと比べると、コンパイル結果がバイトコードを生成するため、「Javaと互換性が完璧であり、性能もまた変わらない」というのが世間一般でいうKotlinの評価ではないかと思います。その上拡張関数やCoroutine、スコープ関数、Null安全性などさまざまな機能が揃っているので、表面上は`better java`と読んでも良いのではないかと思わせる面もあります。それに、JavaのバージョンアップでJVMの改良が行われると、結局それもKotlinの改善につながることとなりますね。Javaも1.8以降は半年に1回のリリース政策によりバージョンアップが早くなっていまずが、まだアプリケーションエンジニアの立場からするとKotlinと比べ惜しいところもなくはないかなと思います。

ここまでの話だと、Kotlinは完璧にJavaを代替できる言語であるかのように聞こえます。つまり、これからは全くJavaを使う理由はなくて、何もかもKotlinに移行するという選択肢しかないかのようにですね。しかし、業界の事情はどうなのでしょうか。

まずJavaの歴史から考えてみましょう。Javaは長い間、「世界で最もよく使われる言語」であって、他の言語が人気を得た今でもTop 5に入るほどの人気な言語となっています[^1]。そしてこれが示唆するのは、単純に今の人気、つまり、「これからも使われる可能性」だけの話ではなく、「今まで使われた回数」が圧倒的に高いということも意味するという点です。今まで作られた多くのシステムやアプリケーションがJava基盤になっているので、余程のことがない限りは維持保守や機能の拡張においてJavaのエンジニアを求めることになるでしょう。

また、こういう側面もあります。JVM言語としてJavaのメリットを活かしつつ、より発展したコードを書けるというコンセプトで登場した言語はKotlinだけではないということです。今までClojure・Scala・Groovyなどさまざまな言語が登場し、それぞれの言語がそれなりの需要や分野を確保・拡張できてはいるものの、そのうちどれも「Javaを超えた」という評価をもらってはいないのが現状かと思います。同じくKotlinの場合も、その立場が他のJVM言語と大きく変わっているとは言えないものではないでしょうか。なので、「JVM言語だ」「Javaよりモダンだ」という特徴は、少なくともKotlinが今後Javaを超えられるという根拠にはならないかと思っています。

モバイルではAndroidの言語としてJavaよりKotlinを採用する例が多くなっているかと思いますが、これはOracleとGoogleの訴訟絡みでJavaを1.8しか使えなかったことも理由の一つかと思います。現在Javaがよく使われているWebの場合、OpenJDKのバージョンに特に法理的な問題もなく、Java 17からはOracleJDKも無償で利用できるようになったので、モバイルとはまた状況は違うのではないかと個人的には思っています。

もちろん、上記の問題はJetbrainsでもその点は最初から認識していたため、最初からKotlinがJavaと相互運用できる言語として設計した部分はあります。なので、あくまで既存のJavaアプリケーションをKotlinでリプレイスする、というよりは、部分的な移行から新規開発で占有率を徐々に上げていくことを目標としているのではないかと思います。その戦略は十分に納得できるもので、あとは企業の方でJavaとKotlinという二つの言語を同時に運用することに抵抗がなければ、Javaを使っていた場合でも問題なくKotlinを受け入れられると思います。実際、自分の場合でもJavaからKotlinの移行は全く問題ありませんでした。

### Kotlinも強くなる

最近のフレームワークやライブラリの方をみると、まだKotlinがモバイル以外の分野での認知度は劣るものの、少しづつJavaがメインストリームであった分野で採用されているケースが増えてきているような気もします。例えば、自分が仕事で使っている[Spring boot](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.kotlin)、[Jackson](https://github.com/FasterXML/jackson-module-kotlin)、[AWS SDK](https://github.com/awslabs/aws-sdk-kotlin)などウェブアプリケーションで有名なものがKotlinに対応していて、他にも[jOOQ](https://www.jooq.org/doc/latest/manual/getting-started/jooq-and-kotlin/)、[jooby](https://jooby.io/v1/doc/lang-kotlin/)、[Javalin](https://javalin.io/)のようにJavaとKotlinの両方に対応しているものも増えています。

もしくは、Javaで存在していたライブラリをKotlin向けに調整したものもあります。例えば[TornadoFX](https://tornadofx.io/)、[gRPC](https://github.com/grpc/grpc-kotlin)、[RxKotlin](https://github.com/ReactiveX/RxKotlin)のようなものがそうです。そして、最初からKotlin専用として設計されたものも少なくないです。[Kotlin Serialization](https://github.com/Kotlin/kotlinx.serialization)、[Klaxon](https://github.com/cbeust/klaxon)、[DGS](https://github.com/Netflix/dgs-framework)、[Ktorm](https://www.ktorm.org/)、[Kotest](https://github.com/kotest/kotest)、[MockK](https://github.com/mockk/mockk)、[Exposed](https://github.com/JetBrains/Exposed)、[KMongo](https://litote.org/kmongo/)、[Xodus](https://github.com/JetBrains/xodus)、[Koin](https://insert-koin.io/)、[Kodein-DI](https://github.com/Kodein-Framework/Kodein-DI)などがそうですね。なので、Javaの世界に寄生していた数年前とは違って、Kotlinだけでもウェブアプリケーションを十分構築できるレベルまできているのではないか、というのが自分の考えです。

結論として、まだ二つの言語を比べると、Javaの方が圧倒的に規模はでかく、知名度でも上にあるのですが、Kotlinも競争できる力を身につけてきたので、これからは十分状況が変わる可能性がある、と思っています。

## vs Go

### 「早い」の美徳

仕事でGoを使っている立場からすると、Kotlinに比べGoの優越な側面はやはり「とにかく早い」ということではないかと思います。基本的にネイティブにコンパイルされる言語なのでランタイム性能も優秀なはずですが、コンパイルもビルドもとにかく早いのは確かに良いなと思いました。特に、コードの修正後にユニットテストで検証してみることが多いのですが、Kotlinのプロジェクトと比べるととにかく早いのでストレスがないですね。(Kotlinの場合は使っているウェブフレームワークがSpringで、テストケースがより多い、ビルド時にはシングルスレッドでやっているということもありますが)

そのほかにもGitHubのパッケージをそのまま使えたり、別途ライブラリを使わなくてもstructをすぐにJSONとして扱える(`omitempty`とかも便利な場面がある)なところは印象的で、かなりウェブ開発に特化されているなという印象までありました。ネイティブなのでビルドして生成されるバイナリのサイズが小さいのも良いですね。これらの特徴からして、最近トレンドとなっているサーバレスやマイクロサービスなどおいてはKotlinよりGoを採用した方が有利な面が多いかなと思います。

まだサーバがクラウド上のVMに移行したばかりの頃は、JVMを使う言語の問題はだいたいマシンスペックの向上により無視できました。しかし、サーバレスとマイクロサービスアーキテクチャが流行りながらJVMの特徴が再び問題となってきていますね。まずサーバレスだと、JVMが完璧にロードされるまで時間がかかるる上に、さらにコールドスタートにも時間がかかります[^2]。また、マイクロサービスにおいては、JVMが占めるヒープメモリとストレージが増えることでインスタンスごとのコストが増えるという問題が挙げられています。

### Kotlin != 遅い

このような問題に対して、サーバレスだと[Kotless](https://site.kotless.io/)のようなフレームワークが開発されていたり、[GraalVM](https://www.graalvm.org/)を利用してネイティブビルドができる[Quarkus](https://ja.quarkus.io/)や[Spring Native](https://github.com/spring-projects-experimental/spring-native)が開発されるなど、JVM言語でも最近のトレンドに合わせて改善が行われていはいます。

ランタイム性能という面では、JITによる最適化でJVM言語でもGoには劣らないという面もありますね。ベンチマークを見ると[Kotlin/JVMとの比較](https://programming-language-benchmarks.vercel.app/go-vs-kotlin)や[Kotlin/Native](https://www.lambrospetrou.com/articles/kotlin-http4k-graalvm-native-and-golang/)でわかるように、Goに対してKotlinが性能で劣る部分もあれば、優位にある部分もあるのがわかります。

また、[Go 1.18でジェエリックが導入](https://go.dev/blog/intro-generics)されていますが、[ジェネリックにより遅くなる可能性がある](https://planetscale.com/blog/generics-can-make-your-go-code-slower)という話もあり、これからもしGoに新しい機能が追加されるとしたら、それがコンパイル速度やランタイム性能に影響を及ぼす可能性もあるかなと思います。

なので、KotlinとGoという二つの言語で考えると、少なくともパフォーマンスという観点だけではGoにこだわる必要はないかなと思います。しかし、アプリケーションの開発において言語を選ぶ基準はパフォーマンスだけでなく、生産性やクラウドで対応している言語、エンジニアが確保できるかなど色々な側面があるので、Goの代わりにKotlinを選んだほうが効率的だとは言えないのも事実です。自分が転職を決めた時も、サーバサイドではGoのエンジニアを募集している企業の方がKotlinより多かったのですが、単純にパフォーマンスが基準だとしたらこのようなことにはならなかったでしょう。Googleが推している言語であるとか、パフォーマンスだけでなく生産性の面でも優れているなどさまざまな理由が複合的に作用した結果だと言えるものかと思います。

### それでも有利なのは

あとは、そもそもの知名度の問題ですね。Kotlinにおいて、ネイティブイメージのビルドができ、性能が劣らないとしても、多くの場合はKotlinをモバイル(Android限定)用の言語だと認識しているのが一般的かなと思います。なので、このような認識がエンジニアと企業で変わらない限り、これからもGoの方がサーバサイドでは市場において優位に立つという状況がしばらくは続くのではないかと思っています。

他にも、Goはその書きやすさからや入門のしやすさの人気もあると思いますが、それを踏まえると比較的書き方が複雑なKotlinの方が劣るのではないかという推測もできそうですね。自分にとってはKotlinの書き方が簡潔で良い感覚ですが、Goの書き方を簡潔だと思っている方もいるようです。確かに、キーワードが少なく、それらを覚えるのに労力が比較的少ないなら、よりロジックに集中した書き方ができるはずですね。Goで作られたアプリやCLIツールなどが増えているのも、そのような特徴からのものなのではないでしょうか。自分の場合は簡単なツールを作るときはPythonで書くのを好みますが、同じく気軽にコードが書ける稼働かの観点でいうと、KotlinよりGoが優れているとも言える気がします。なので、個人の趣味やサイドプロジェクトなどでよく使われ、それがまた人気につながるだろうと思うと、Goを好むエンジニアが増えるのもおかしくはないですね。

## vs Rust

### 最強の性能？

GCがないので同じネイティブでありながらもGoより性能が優秀だというRustですが、これもまたKotlinと同じく、知名度の問題で苦戦しているところがあるかなと思います。そもそもC/C++を代替するのが開発の目的でもあったため仕方ないのかもしれませんが、どちらかというとエンベデッドで使われるイメージがどうしてもあるような気がしますね。意外と[Figma](https://www.figma.com/)、[1Password](https://1password.com/jp/)、[Discord](https://discord.com/)、[Dropbox](https://www.dropbox.com/)、[Mozilla](https://www.mozilla.org/ja/)、[Line](https://line.me/ja/)、[npm](https://www.npmjs.com/)、[Cloudflare](https://www.cloudflare.com/ja-jp/)などさまざまな組織で採用されていて、[exa](https://github.com/ogham/exa)、[bat](https://github.com/sharkdp/bat)、[difftastic](https://github.com/Wilfred/difftastic)、[bottom](https://github.com/ClementTsang/bottom)などのCLIツールから[yew](https://yew.rs/)、[seed](https://seed-rs.org/)、[Dioxus](https://dioxuslabs.com/)、[Rocket](https://rocket.rs/)、[tide](https://github.com/http-rs/tide)、[poem](https://github.com/poem-web/poem)のようなGUIやウェブフレームワークなどがたくさん開発されていますが、これもまた特に調査してみないとわからないくらいです。

さまざまなベンチマークでその性能が検証されていて、使ってみたエンジニアからも評判の高いものとなっているRustですが、やはり知名度が低いので、企業からも採用するのはかなり難しい判断になるでしょう。実際Jetbrainsの去年の設問では[Rustは趣味もしくは個人用途、サイドプロジェクトで使う](https://www.jetbrains.com/ja-jp/lp/devecosystem-2021/rust/#Rust_how-do-you-use-rust)と答えた割合のエンジニアが多かったのを見ると、やはり企業の需要はあまりようです。ただ、逆にいうと、このようにRustに好意的なエンジニアが増え、さまざまなプロジェクトで使われ始めるといつか市場の状況も変わっていく可能性もあるということです。先ほど述べたGoのケースのように、比較的に歴史の短い若い言語でも十分その価値を立証できるのであれば市場でもメインストリームに合流できます。なので、Rustの未来はむしろ明るく、これからが期待される言語だと個人的には思っています。ただ、人気を得た後も、ウェブアプリケーションを開発するよりは今まで通りエンベデッド・システムプログラミングに特化していきそうな気がしますね。

### Kotlinをネイティブにしたら

RustをKotlinとの比較をするとしたら、Kotlin/Nativeがあるので、言語自体でできることはそう変わらないものの、Rustがエンベデッドやシステムプログラミングという分野でC/C++を代替していく傾向があるのに対して、これといった成果があまり見当たらないというのが問題かなと思います。特にKotlin/NativeはLLVM基盤なので、GraalVMによるネイティブコンバイるができるウェブフレームワークが登場している今はますますそのポジションが曖昧なものになっている気もします。Object-CやC/C++とのinteropができると言われていますが、そのようなユースケースだとそもそもObject-CやC/C++といった言語を使った方が色々と有利なのではないでしょうか。もちろん、Rustには所有権のような概念があり、他の言語と比べプログラミングが難しいとされているので、Kotlin/Nativeを採用した方がコーディングは楽になるかもしれません。でも、Nativeを追求するならやはりパフォーマンスが重視される場面が多いので、そこではGCのあるKotlinが不利な気がしますね。このような面からすると、やはりKotlin/Nativeのポジショニングが難しそうな気がします。

結論としては、Kotlin(JVM)とRustはそれぞれ特化した分野が違っていて、大きな変化がない限り互いの領域を蝕むことなく発展していきそうです。どちらかというとKotlin/Nativeが直接的なライバルになる可能性はありますが、そもそものポジショニングが曖昧なところがあるので、Nativeがどうしても必要な場面ではRustが使われる可能性が高いのではないか、という気がしています。

## vs Python

### 万能ツール

ここ数年で最も人気を得ている言語の一つ、Pythonの場合は、Kotlinと比べて見るとどうでしょうか。まず自分の場合だと、日常での自動化や簡単なツールを作る場面ではPythonの方をよく使っていて、本格的なウェブアプリケーションを開発するとしたらKotlinを選ぶことが多いです。もちろん、なんでもできる言語なので大規模のアプリケーションを作るのにPythonがNGというわけではないです。実際Uber、Google、PayPal、Netflixなど有種の企業がPythonを使っていて、あの有名なInstagramのサーバサイドもPythonで書かれていると言われていますね。

ただやはり、PythonはデータサイエンスやAIといった分野でよく使われているイメージがあり、使いやすく、そこまで性能が求められていない場面でなら良いものの、個人的にはその限界が明確であることが問題かなという気もします。本格的な業務用のアプリを開発した経験がないのであくまで印象と推測の話となりますが、Pythonをサーバサイドに取り入れている企業は大概がスタートアップであって、サービスが古くなるとインタープリター言語特有のメンテが難しくなるという問題が出てくる可能性が高いではないかと思いますね。JavaScriptの例もありますが、Pythonのタイプヒントはあくまでヒントであって、TypeScriptのようにコンパイルタイムで検出できるエラーを確実にわかるわけでもないです。あとは性能ですが、[GIL](https://wiki.python.org/moin/GlobalInterpreterLock)のような問題もあります。このような問題を認識しているため、検証用のアプリ(プロトタイプ)をPythonで書いてから他の言語に移行するという例もあるのかなと思ったりしています。

### Pythonだけの領域でもないが

逆にKotlinでいうと、Jupyterを使えるなど[Kotlinでもデータサイエンスに使える](https://kotlinlang.org/docs/data-science-overview.html#kotlin-libraries)のですが、すでにPythonが市場支配的な言語になっているところでどこまで伸びるかが問題な気がしますね。JetBrainsが主張するように、Pythonと比べ「静的型付け、Null安全性、パフォーマンス」というのは確かにKotlinが持つメリットではあるのですが、そもそものユーザ数が増える何かがないと占有率を上げるのはかなり難しくないのではないかと思います。Pythonは入門が簡単なので講座も多く、実際エンジニアではない人も使うケースが多いのですが、Kotlinはまだそのような面では弱い印象ですね。

以上のことからして、Pythonは依然としてデータサイエンスなど元々強かった分野に対してはこれからも需要が大きく変わることはなさそうです。ウェブという分野では競合になる可能性はありますが、どちらかというとKotlinを採用した方がより安定した開発ができるので大規模なアプリの開発ではKotlin、小規模ではPythonという形になるのではないかと思います。もちろん、大規模のアプリを開発するにあたってはまたの選択肢があるのでKotlinではない他の言語が採用される可能性の方が高そうですが、あくまで二つの言語を比べた場合の話となります。

## vs JavaScript

### 多芸多才

一つの言語でなんでもできちゃう言語が何かというと、過去はJava、少し前はPython、そして今はなんといってもJavaScriptではないかと思います。フロントエンド、バックエンド、モバイル、データサイエンスなどさまざまな分野で活躍している言語ですね。ランタイムの性能が問題となっている部分に対しても[Deno](https://deno.land/)のような新しいランタイムが登場したり、V8エンジンの持続的な改善によりだんだん補完されていって、静的型付けに関してもTypeScriptの台頭によって解決されています。まさに無敵の言語のようにも見えます。

フロントエンドにおいてはJavaScript以外は考えられない[^3]というのもあり、[WebAssembly](https://webassembly.org/)のような技術も発達していますが、これはまたウェブの画面描画だけでなく違う方向に向かっているような感じなので、これから何かあって(あるとは思いませんが)色々な分野で使われなくなるとしてもJavaScriptそのものが使われなくなることはないでしょう。そして同じ意味で、Kotlinがそのような分野に進出するのもかなりハードルが高いと思います。

### Kotlinでフロントエンド？

Kotlinで言えば、Kotlin/JVMとKotlin/Native以外に3つの軸として存在しているのが[Kotlin/JS](https://kotlinlang.org/docs/js-overview.html)であり、JetBrainsの発信を見るとそこそこ力を入れている感覚ではあります。他にも、[Compose Multiplatform](https://www.jetbrains.com/ja-jp/lp/compose-mpp/)を通じて、モバイルだけでなくウェブやデスクトップアプリにおいてもKotlinでGUIを作成できるようになったので、なるべく自分のサイドプロジェクトなどではKotlinで完結したいと思っている私の場合はこちらも応援したいと思っています。ただ、まだモバイル以外ではそこまでメジャーではなく、新しい技術の問題(ライブラリの不足、バージョンアップによる変化が激しいなど)が考えられるのでしばらくは様子見な感じですね。あと自分のような特殊な目的がない場合は、個人でも企業側としても無理して採用すべきメリットが薄いという問題もあるかなと思います。

バックエンドだとKotlinが競合になる可能性はまだ十分ではないかと思います。特に、今までJavaが採用されていた分野だと主にJVMの安定性や数値計算の精度など検証された安全性というものがあるので、これから言語を変えるとしたらKotlinを採用する確率が高いのではないかと思っているところですが、そのような分野だと、最初からJavaScriptによるバックエンドの採用は考えない可能性が高そうです。スタートアップのようにエンジニアの求人が難しく、使われる技術の数を減らしてなるべく工数の削減しようとするか、Pythonのようにプロトタイプのアプリを作るかなどの特殊な状況ではない限り積極的にバックエンドの言語としてJavaScriptを採用する例はあまりなさそうな気がしていて、これからもおそらくそれは大きく変化していく気はしません。ただ、自分のようにKotlinで何もかも解決したい、という方がJavaScript側にもいらっしゃるとしたら、そこはまた話が変わってくるかもしれませんね。フロントエンド、バックエンド、モバイル、デスクトップまで対応したい場合にはJavaScript以上のものがないので、その会社や個人の目的次第でJavaScriptが採用される可能性は高く、そのような状況こそKotlinは採用されない可能性が高いかなと思います。

## vs Dart

### GUIの最強者？

Dartの場合は、言語そのものというよりは[Flutter](https://flutter.dev/)が最近熱いですね。最初はモバイルでクロスプラットフォーム開発ができるということで注目されたものですが、Dartに対してはFlutterの最大の競合は[React Native](https://reactnative.dev/)だと言えるかなと思いますが、それも最近のトレンドを見ると少しづつ逆転してきているような気がします。もちろんこれはあくまでも「クロスプラットフォーム用のフレームワーク」という基準での比較であり、実際は色々と複雑な事情があるでしょう。例えば、フロントエンドエンジニアがモバイルの開発も担当していて、フロントエンドのライブラリとしてはReactを使っているなどの状況を考えると、ここでいきなりFlutterを採用する可能性は低くなるはずですので。

自分が思うにDartの最大の問題は、その最初の意図(JavaScriptを代替するという)はともかく、言語そのものの印象が薄いということです。少しだけ触ってみた感覚では、いわゆるC-Family言語としての馴染みはあっても、特段ここが魅力的だというところはあまり感じ取れなかったです。それが今はFlutterのおかがで使用率は上がってきていますが、それ以外の分野ではどうかなという疑問がまだあります。

### 可能性は他にもあるかもだけど

ただ、以前からGoogleの次世代OSである[Fuchsia](https://fuchsia.dev/)ではメインの開発環境となるという噂もあり、FuchsiaそのものがどんなOSになるかはまだ不明な状態ですが、もし噂通りAndroidの次世代のOSになるとしたら、ネイティブの開発そのものがDartによるものにもなり得る可能性はありますね。もしそうなると、ChromeOSを含めレガシーの環境を捨てることになるので、公式の開発の言語にKotlinを指定した時とは比べ物にならないインパクトがあることを想定すると、なかなか想像できない事態です。

もちろんDartもプログラミング言語なので、これからのフレームワークやライブラリの開発次第でいくらでも状況は変わる可能性があります。[こちらのリポジトリ](https://github.com/yissachar/awesome-dart)を参照すると、サーバサイドのフレームワークもすでにいくつか存在しているので、自分の考えているKotlinで全てを解決する、という目標においてはむしろDartの方がやりやすい可能性がありますね。Kotlinの方だと[Kotlin/Multiiplatform Mobile](https://kotlinlang.org/lp/mobile/)がありますが、これはどちらかというとビジネスロジックの共通化を目標としているものなので、結局iOSのコードを書く必要があります。もちろん、一部の企業でやっているように「UIはFlutterで、ビジネスロジックはネイティブで」ということもできるかとは思いますが、あまりメジャーなやり方にならないかなと思います。実際、Swiftの場合も[Vapor](https://vapor.codes/)のようなフレームワークがあり、サーバサイドでも十分使えるということをアピールしていますが、採用しているエンジニアや企業が限りなく少ないというのを見ると、単純に「できる」だけでは十分ではなさそうですので。

### モバイルでも強くなっていく

特に今年開催された[Google I/O](https://io.google/2022/intl/ja/)で確認できるように、Flutter 3ではさらにパフォーマンスの向上やFlutter Desktopの正式リリースなど様々な面での発展を見せていて、これからもFlutterの未来は明るく見えます。Flutterを採用している企業も増えてきているので、このような発展の恩恵を受け入れるのは結局時間の問題に過ぎない気がします。もちろん、ネイティブアプリの開発においても需要はこれからもあり得ると思いますが、クロスプラットフォームアプリでも事足りる分野が増えてくるとしたら、どちらがメインストリームになるかは目に見えるようなものですね。

このような状況では、今の占有率においてKotlinのホームグラウンドとなっていると言っても過言ではないモバイルの分野で、Flutterの成長ぶりはある意味、Kotlinにおいては脅威のようなものではないかという気がします。なので、これからKotlinならではのメリットをより強化していく必要がありそうですね。先の述べたKotlin/Multiplatform Mobileのようなものが、その役割をしてくれるのではないかと期待しています。そのほかでも、Kotlinでできることは多いので、分野を問わない連携を強化していくと十分Kotlinを利用するメリットはこれからも出てくるでしょう。

## 最後に

今回は、いつもと違って自分の考えが中心になる記事なので、色々と偏った判断があるかもしれませんが、とりあえずKotlinエンジニアとしての感想をまとめてみました。もちろん、自分の知見が足りてなく、モバイルやフロントエンド、データサイエンティスト、DevOpsエンジニアなど色々な分野で活躍されている方からしたら色々と間違っているか、的確ではない情報や判断も目立つかなとも思います。

ただ、一人のエンジニアとして、ただの時流を淡々と見つめているよりは、目指す目標に対して使っている技術や興味のいくものに注目し、自分なりの判断をしてみるのもまた必要なものではないかという気がして、このような記事を作成することになりました。また、このような記事を作成することで、この後に色々な変化があって自分の展望がどれだけあっているか、実際と比べてみるのもまた有意義な振り返りとなりそうな気もします。

今回はあまり情報がなく、Twitterにでもつぶやいたら良いかも知れない雑談に近いものですが、少しでもここでKotlinのことを改めて認識できたという方がいらっしゃるなら幸いです。

では、また！

[^1]: [IEEE Spectrum](https://spectrum.ieee.org/top-programming-languages)、[TIOBE](https://www.tiobe.com/tiobe-index/)、[Stack Overflow](https://insights.stackoverflow.com/survey/2021#section-most-popular-technologies-programming-scripting-and-markup-languages)、[Jetbrains](https://www.jetbrains.com/ja-jp/lp/devecosystem-2021/#Main_programming-languages)の調査結果を参照しました。
[^2]: Warm(アイドルインスタンスを常に立ち上げておく)で対応できる部分ではありますが、スケールアウトするとコールドスタートが必要となる場合があり、インスタンスを立ち上げておくことでコストがかかる問題は避けられないですね。
[^3]: Dartのような言語でJavaScriptを代替しようとした歴史がありますが、今は失敗していて、JavaScriptがより高度化した今はTypeScriptのようなスーパーセットやJavaScriptにトランスパイルできる言語でないとフロントエンドの言語を代替するのは難しいかと思われます。