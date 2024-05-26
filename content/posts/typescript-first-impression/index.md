---
title: "JavaプログラマーがみたTypeScript"
date: 2020-05-16
categories: 
  - typescript
image: "../../images/typescript.jpg"
tags:
  - node
  - javascript
  - typescript
---

このたびは、新しい案件で[Angular](https://angular.io)とSpring BootによるWebアプリケーションの開発を担当することになりました。Spring Bootは今までずっとやってきているのであまり問題はないと思いますが、問題はAngularです。Angularはバージョン2からTypeScriptを使うことになっているらしく、JavaScriptとjQueryを少し触ったくらいの経験しかないのでそもそもTypeScriptがどんなものかすら知らなかったです。ただわかっているのは、JavaやC#みたいに静的言語みたくコードを書いたら、それをいい感じにコンパイルしてJavaScriptに変えてくれる、ということだけでした。

なのでまずTypeScriptの勉強を始めましたが、言語そのものの特徴に含め、TypeScriptで何かを作る時に必要となる知識は意外と大きいことに気付きました。今回のポストでは正直な感想と、どんなものが必要なのかをJavaしか知らない初心者である自分の観点から述べていきたいと思います。

## 驚くほどJavaに似ている

元々C/C++、C#、JavaといったいわゆるC-family言語と似ているとの話は聞いていましたが、実際TypeScriptを触ってみると本当にそうでした。最初、なんの情報もなかった時(型を指定できて、コンパイルしたらJavaScriptに変わるという話だけ聞いている時)、ただ普通の変数の宣言に型指定ができるようになっているだけかと思っていたのですが、それだけでなく意外とオブジェクト指向に合わせて発展、もしくは回帰している印象を受けました。

### 型指定はいい感じ

型の指定は、元々動的言語であるJavaScriptを昔ながらの静的言語化するものですね。でも、書き方自体は昔のままでなく、KotlinやSwiftのようなモダンな言語に似ています。型をまず書くのではなく、変数として宣言したあとに型をつける[^1]のですね。例えば以下のようになります。

Javaでの型指定

```java
int a = 1000;
```

TypeScriptの型指定

```typescript
var a: number = 1000;
```

また、こういう型指定を、TypeScriptでは「型注釈」とも呼ぶらしいです。TypeScriptはコンパイルされたらランタイムで型が決まるJavaScriptになるので、コンパイラーのために注釈をつけてあげる、という意味に近いようです。

### 型以外の類似点

TypeScriptには型指定だけでなく、C#やJavaプログラマーならすでに慣れているAccess修飾子、Interface、Class、Generics、Decorator(Annotation)なども用意されています。これらの一部は最近JavaScriptでも対応している機能ではありますが、実際のアプリを開発する業務ではブラウザの制約から最新のJavaScriptを使えない場合もあり不便ですね。しかし、TypeScriptを使うとコンパイラのオプションを指定することでどのバージョンのJavaScriptとしてソースコードを出力するか選択できるので、ブラウザをあまり気にせず同じコードを書けます。これは本当にありがたいですね。

他に、依存関係をnpmといったパッケージマネージャで管理できるというのも良いところ。これはJavaScriptでもできる(というか、そちらが先ですが)ことですが、Importと組み合わせたらほぼC-familyと似たような感覚で運用できるという面が良いですね。JavaScriptのモジュールというのも、またES6から対応しているのでそれ以前のバージョンを使う場合だとかなり面倒くなるかも知れない、と思いました。知識不足なだけかもしれませんが…

## 気になったところ

TypeScriptの良いところは、やはり昔ながらの言語に慣れている人にはかなり快適にコードをかけるような環境を提供していて、コンパイルタイムでエラーを探せるという、その何ふさわしいものとなっている点だと思います。ただ、少し気になったところもあったのでそれについても書きます。

### 結局はJavaScript

TypeScriptはJavaScriptのSuper setなので、JavaScriptの機能をそのまま利用できるというのが特徴だとも言われています。最初はそれをメリットとして捉えていましたが、勉強をしながらやはりデメリットもあるんだなと感じました。JavaScriptをそのままかけるということは、結局言い方を変えると、ソースコードの中にJavaScript(Vanilla)とTypeScriptが混在していても問題なくなるということですね。こうなった場合にTypeScriptがSuper setという特徴を諦めないかぎりコードが動かなくなることはないと思いますが、人の観点からするとかなり混乱するコードが生まれる可能性もあるのではないかと思いますね。

#### JavaScriptでも書けるということの意味

そもそもJavaScriptの歴史の方が長く、ユーザも絶対的にJavaScriptの方が多いです。そして、私みたいにC-family言語ではなくJavaScriptからプログラミングに入門した人もいるので、そのような人にとってTypeScriptはあえて使うメリットのない、ただの不便なものにしか思われない可能性もあると思います。また、JavaScriptの方に慣れている人がTypeScriptに移ったとしても、結局はJavaScriptと変わらない書き方になる可能性もなくはないはずですね。

TypeScriptがSuper setとして企画された理由は、既存のJavaScriptプログラマーを吸収するための政策だったと思いますが、プログラミング言語は自由が多いほど混乱しやすいと思うので、これはやはりメリットでもデメリットでもあると思いました。

#### prototype

クラス基盤のオブジェクト指向言語から入門しているからか、自分にはJavaScriptの`prototype`という概念があまりわかってないです。ともかくJavaScriptにはこのprototypeというのがあって、オブジェクトを変数として使うこともできるという特徴を持っていますね[^2]。Javaプログラマーとしてはオブジェクトはクラス、クラスはファイル一つ、という風に考えることが一般的なので、こういう自由度に触れるとどうしたらいいか混乱しそうな感覚です。そしてもちろん、これはTypeScriptよりもJavaScriptの特徴ではありますが、TypeScriptがSuper setである以上避けられない問題なのですね。

#### Module

モジュールがJavaScriptに導入されて、TypeScriptでももちろんそのまま使えます。個別のファイルにクラスを一つづつ作成して、インポートして使っているとまるでJavaと大差ないようにも思われますね。でもこのモジュールというものも実際はややこしいところがあるようで、書き方をどうするか考えなくてはならない場合もあるようです。

例えばPythonのように`as`を入れることでインポートしたモジュールに別名をつけることができるけど、そうしたら問題が怒るケースもあるとか、CORS設定で塞がる場合もあるとかというのが最初はかなり複雑だなと感じました。まだ深堀りしてないので実際は使ってみないとわからないと思いますが、そもそも使ってみないとわからないという部分はやはり気になりますね。

```typescript
// 二つの書き方ができるが、一緒ではない
import express from "express";
import * as express from "express";
```

#### コンパイル

TypeScriptをインストールすると、ターミナルから`tsc`というコマンドを使えるようになります。そしてTypeScriptで作成したファイルは`.ts`というファイルに書いて、tscコマンドで`.js`に変わりますね。`tsconfig.json`ファイルを定義することでどうコンパイルするか様々なオプションを指定することもできます。例えばJavaScriptにコンパイルする時、どんなバージョンでコンパイルするか、どのファイルをコンパイルするか、`.js`ファイルはどこに吐き出すかなど。

ただ、コンパイルにかなり時間がかかるのもあり、コンパイルされたファイルは結局JavaScriptになります[^3]。そしてこれは、TypeScriptで作成しても「ランタイムでは弱タイプになる」ということを意味しますね。ちゃんとタイプを明示して(`any`を使わず)コードを作成しただけでは不十分なケースが十分あり得るので、ランタイムでも型による問題が起こらないようにする必要性があるかも知れないという印象を受けました。

最近はnpmの代替ランタイムとしてTypeScriptをそのまま使える[Deno](https://deno.land)のようなものも登場していますが、やはり内部的にはtscを使っているらしく、このコンパイル速度が遅いのが最大の問題として挙げられていました。Rustで新しいコンパイラを作成しているとのことですが、それがいつ完成されるかもわからないですね。

そしてtscのみでなく、[webpack](https://webpack.js.org)を使う場合はts-loaderを使ったりするとかなり初期設定が複雑な印象を受けました。他に[Babel](https://babeljs.io)などを使う場合もあるらしいですが、コンパイルをするためという理由でこのようなベンドラやコンパイラをまた勉強しなくてはならないというのは少し不便な部分ですね。JavaScriptと紐付けられている言語としての運命みたいなものかもしれませんが…

## すっきりしたところ

### JavaScriptライブラリを使える

「TypeScriptではJavaScriptのライブラリも使える」という話も聞きましたが、実際それがどうやって可能になったのかが疑問でした。例えばTypeScriptにも型推論があってある程度、型を宣言しなくても良い場合はあるものの、基本的には型注釈で明示的に型を指定してあげないとうまくコンパイルできない部分もありますね。しかし、すでに存在しているJavaScriptのライブラリ全てがTypeScriptを考慮して作られているとは考えられないので、これをTypeScriptではどのような形で対応しているかが一番の疑問でした。

答えは意外と簡単で、`.d.ts`という形で「型定義ファイル」を作成すると良い、ということでした。すでにTypeScriptに対応しているライブラリの場合もこの型定義ファイルによってTypeScriptが型を参照できるようにしていて、自作することも可能。そしてメジャーなものの場合、`node_modules`にインストールできる形で型定義ファイルを提供している場合もありました。

すでに作成されている型定義ファイルのインストールも簡単で、例えばnodeのTypeScript型定義が必要な場合は`npm install --save-dev @types/node`のようにコマンド一つでインストールができて、GitHubの[リポジトリ](https://github.com/DefinitelyTyped/DefinitelyTyped)を参照するとどんなライブラリの型定義ファイルが提供されているか確認することもできました。

## 最後に

まだ色々気になるところはありますが、多くはTypeScriptそのものというよりもJavaScriptの特性から起因しているものが多いため、現時点でフロントエンドエンジニアがどの言語を選択すべきかというと、やはりTypeScriptの方が良いのではないかと思いました。もちろん、これは自分のスタートがJavaであることもあるとは思いますが、やはりコンパイルタイムで多くの問題を事前にキャッチできるというメリットは捨てがたいものだと思っています。

現在はJavaScriptとの互換性のために色々初期設定など煩雑だったりコンパイルに時間がかかるなどの問題もいくつかありますが、これらもAngularみたいに最初からTypeScriptを前提に作られるフレームワークやライブラリが増えたり、ブラウザからTypeScriptに対応したり(これは可能性が低いかな…)したら自然に解決される問題ではないかとも思いますね。

最初は単純にAngularを使うために勉強したものの、意外としっかりしていて、これからの未来もかなり明るい感じだったのでエンジニアの皆さんにもぜひお勧めしたいと思っているところです。

ああ、Pythonも型指定できるようになるといいな…[^4]

[^1]: もちろん、最近はC#もJavaも`var`で変数を宣言することもできますが、習慣もあり、元々静的言語では型から書いた方が良いのではないかと個人的には思っています。
[^2]: しかも、ES6で登場したClassは結局プロトタイプ基盤の`function`の書き方をちょっと変えただけなのですね。
[^3]: Javaのようにバイトコードに変わったり、Cのように機械語に変わるようなものとは少し違うのでTypeScriptのコンパイルは`トランスパイル`とも呼ぶらしいです。
[^4]: 知識不足でした！Pythonも3.6から型の宣言ができるようになっています。