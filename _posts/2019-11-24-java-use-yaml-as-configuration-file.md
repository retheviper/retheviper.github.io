---
title: "外部設定ファイルを扱うクラスを作る"
date: 2019-11-24
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - yaml
  - java
---

アプリケーションを作る場合、考慮しなければならないことの一つは「設定ファイルを作る」ことです。設定値のデータがアプリケーションの内部にあると、ディコンパイルしない限りそれを変えられる方法がなくからですね。なので一般的にはアプリケーションの柔軟性のためにも動的に変える必要のある設定値はアプリケーションの外に別途のファイルとして置く場合が多いです。ゲームでいえばセーブファイルみたいなものですね。

外部のファイルを読み込むことに関してはすでに様々な方法があるとは思いますが、今回は自分で使っている方法を共有します。簡単に、外部設定ファイルは[YAML](https://ja.wikipedia.org/wiki/YAML)で記載してプログラムの起動時に読み込んでシングルトンクラスのフィールドとして持す方法です。I/Oを一回だけにできて、どこからも参照できるというメリットがあります。設定ファイルのフォーマットとしてYAMLを選定したのは、書きやすく読みやすいというメリットもあって、自分が主に扱っているSpringで使っているためでもあります。JSONでも問題はないですが、項目が増えるほどJSONは読みづらくなるので…

とにかく、準備するものはYAMLを読み込むためのライブラリーです。ここでは[SnakeYaml](https://bitbucket.org/asomov/snakeyaml/src/default)を使います。ファイルを読み込んでオブジェクト化するだけなので他のライブラリーを使っても構いません。

では、まず以下のようなYAMLファイルがあるとしましょう。

```yaml
Development:
  id: "develop"
  version: 8
  use_local_storage: true
Debugging:
  id: "debug"
  version: 11
  use_local_storage: false
```

DevelopmentとDebuggingという二つのケースでそれぞれ違う値を使いたい、というシナリオで簡単に書いてみました。これを利用するコードは以下です。

```java
// 共用の設定情報クラス
public final class Settings {

    // シングルトンクラスなので自分のインスタンスを持っている
    private static final Settings UNIQUE_INSTANCE = new Settings();

    // 設定ファイルから読み込んだ値を一次的に入れておくためのフィールド
    private static final Map<String, Map<String, Object>> SETTINGS_FROM_FILE = new HashMap<>();

    // 設定ファイル名
    private static final String SETTINGS_FILENAME = "application-settings.yml";

    // Developmentの設定情報が入るMap
    @Getter(lazy = true)
    private final Map<String, Object> developmentSettings = setDevelopmentSettings();

    // Debuggingの設定が入るMap
    @Getter(lazy = true)
    private final Map<String, Object> debuggingSettings = setDebuggingSettings();

    // イニシャライザーブロックで最初からファイルを読み込む処理
    static {
        // ファイルを指定して読み込む
        ClassLoader classloader = UNIQUE_INSTANCE.getClassLoader();
        URL resource = classloader.getResource(SETTINGS_FILENAME);
        try (InputStreamReader reader = new InputStramReader(resource.openStream())) {
            // 読み込んだYAMLファイルをパースしてMapに値を取り込む
            Yaml yaml = new Yaml();
            Map<String, Map<String, Object>> importedMap = autoCast(yaml.load(reader));
            // 読み込んだ値をフィールドのMapに移す
            for (Map.Entry<String, Map<String, Object>> entry : importedMap.entrySet()) {
                SETTINGS_FROM_FILE.put(entry.getKey(), entry.getValue());
            }
        } catch (IOException e) {
            // 例外処理
        }
    }

    // コンストラクターは外部からアクセスできない
    private Settings() {
    }

    // Lazy Getterで要請が入った時点でインスタンスを作るためのメソッド
    private Map<String, Object> setDevelopmentSettings() {
        return Collections.unmodifiableMap(SETTINGS_FROM_FILE.get("Development"));
    }

    // Lazy Getterで要請が入った時点でインスタンスを作るためのメソッド
    private Map<String, Object> setDebuggingSettings() {
        return Collections.unmodifiableMap(SETTINGS_FROM_FILE.get("Debugging"));
    }

    // 外部からインスタンスを取得するためのGetter
    public static Settings getInstance() {
        return UNIQUE_INSTANCE;
    }

    // オブジェクトのキャストをより簡単にするためのメソッド
    @SuppressWarnings("unchecked")
    private static <T> T autoCast(final Object object) {
        return (T)object;
    }
}
```

ここではLazy Getterを使ってDevelopmentとDebuggingの設定のフィールドが、get要請が入った時点でインスタンスが作られるようにしています。こうしている理由は、イニシャライザーブロックでファイルを読み込んだ後から個別フィールドに値を入れたい + フィールドはprivate finalにしたい + Settingsクラスはシングルトンとしてプログラムの起動時にインスタンスが生成されるようにしたいからです。Lazy Getterを設定しておくとprivate finalを維持しつつ、フィールドの初期化は後に担保できて、一度インスタンスが生成されるとキャッシュとして残りますので便利です。もしこうでなく、コンストラクターやフィールドで初期化しようとするとその時点がファイルを読み込む前となってしまうので注意しましょう。

## 最後に

今回紹介したコードを使うと、設定クラスはシングルトンクラスとしてアプリケーションの起動時にインスタンスが生成され、その時ファイルを読み込んでMapとしてメモリー上に載せ、どこからでも固定された値をGetterで取得できます。もしYAMLの設定がより深くネストしたり、項目が増えたり、ファイル名が変わったりする場合はMapとフィールドを調整するだけで簡単に変更ができますね。

単純に動くだけでなく、維持補修が簡単で無駄のないコードを書くことこそが重要なので、何か頻繁に変わるデータを扱うためにはこう言った仕組みを(自分が紹介したものと同じではなくても)考える必要があるのでは、と思います。これからもどんなものが良いコードなのかを常に意識しないと、ですね。