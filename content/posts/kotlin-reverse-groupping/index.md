---
title: "Kotlinでデータの逆転グルーピング"
date: 2022-01-29
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
---

DBを設計する時と、最終的にアプリで活用するデータの形は大きく変わるケースがあります。特に後から機能を付け加えるとそうなりますね。もちろん正規化などを通じてより効率的にデータを保存する方法を考える必要のあるDBと、データをいかに加工して使うかを工夫するアプリの違いによるものもありますが、アプリの改修が続くと同じデータでも活用する箇所や表現の仕方が変わってくるからでもあるかなと思います。

そういうわけで、今回はそのようなケースで一つ活用できる方法をご紹介したいと思います。アルゴリズムというわけでもありませんし、より効率的な方法はあるかなと思いますが、応用すれば結構色々な場所で使えそうな方法ではないかと思います。

## シナリオ

例えば以下のようなシナリオがあるとします。

1. 社員はA、Bという二つの部署に配属される
2. 社員が部署に配属される日付はそれぞれ

この場合、データの作りには色々な観点があるかと思いますが、まず部署の配属日を基準にデータを作るとしたら、部署の種類、配属日とその日付に配属となった社員のリストを持つような形になるかと思います。Kotlinのコードとして表現するとしたら以下のような形ですね。

```kotlin
enum class DepartmentType { A, B }

data class Department(
    val departmentType: DepartmentType,
    val date: LocalDate,
    val employers: List<Employer>
) {
    data class Employer(
        val id: Int
    )
}
```

ここで社員の3人がいて、それぞれ部署Aと部署Bに配属された日付が違うケースがあるとしましょう。データとしては、以下のようなものです。

| 社員番号 | 部署A配属 | 部署B配属 |
|---|---|---|
| 1 | 1月1日 | 1月1日 |
| 2 | 1月1日 | 2月1日 |
| 3 | 2月1日 | 2月1日 |

上記のデータを持って、先ほどの部署のデータを実際のリストとして作るとしたら以下のようになるかなと思います。

```kotlin
val departments = listOf(
    Department(
        departmentType = DepartmentType.A,
        date = LocalDate.of(2022, 1, 1),
        employers = listOf(
            Department.Employer(id = 1),
            Department.Employer(id = 2)
        ),
    ),
    Department(
        departmentType = DepartmentType.A,
        date = LocalDate.of(2022, 2, 1),
        employers = listOf(
            Department.Employer(id = 3)
        ),
    ),
    Department(
        departmentType = DepartmentType.B,
        date = LocalDate.of(2022, 1, 1),
        employers = listOf(
            Department.Employer(id = 1)
        ),
    ),
    Department(
        departmentType = DepartmentType.B,
        date = LocalDate.of(2022, 2, 1),
        employers = listOf(
            Department.Employer(id = 2),
            Department.Employer(id = 3),
        )
    )
)
```

ただ、これを社員を基準に、それぞれの部署に配属された日付をデータとして加工するにはどうしたら良いでしょうか。社員番号と部署に配属となった日付の二つを持つような形です。例えば、コードで表現すると以下のようなものです。

```kotlin
data class JoinedDates(
    val employerId: Int,
    val departmentA: LocalDate,
    val departmentB: LocalDate
)
```

つまり、やりたいことは先ほどの`departments`を、最終的に以下のようなデータにしたいということですね。

```kotlin
[
  JoinedDates(employerId=EmployerId(value=1), departmentA=2022-01-01, departmentB=2022-01-01), 
  JoinedDates(employerId=EmployerId(value=2), departmentA=2022-01-01, departmentB=2022-02-01), 
  JoinedDates(employerId=EmployerId(value=3), departmentA=2022-02-01, departmentB=2022-02-01)
]
```

データの整列の基準がひっくり返されるので、どうしたら良いかと悩ましくなる場面です。今回は、これを解決した自分の方法を紹介したいと思います。

## ロジック

Departmentを基準に考えるとEmployerのデータが複数になりますが、これを逆転させて、Employerを基準に複数のDepartmentを持つという形に加工したいというのが今回の要件です。だとすると、考えられるロジックは以下がポイントかなと思います。

1. EmployerのID単位でまとめる
1. EmployerごとにDepartmentをType別に分けた配列を持たせる

まずはネスとしているEmployerのリストに入り、そのIDを抽出する必要がありますね。このIDは重複させたくないので、MapのKeyにしておくと良さげです。

あとは、そのMapにEmployerのIDがKeyとして存在するかどうかで以下の処理をすると良いでしょう。

1. Keyが存在しない場合は、新しくDepartmentのタイプとその日付をinsert
2. Keyが存在する場合は、そのvalueを取り出してDepartmentのタイプと日付を追加

なので、一回DepartmentのListをMapに変換して、さらにJoinedDatesのListに変換することとなります。ちょうど上記の分岐については、[compute()](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html#compute-K-java.util.function.BiFunction-)を活用するとできるので、中間データとしてのMapがどんな形になるかを考えると良いかなと思います。

私の場合はMapの方がデータを取りやすいので、最終的には以下のような処理となりました。

```kotlin
fun List<Department>.toJoinedDates(): List<JoinedDates> {
    // 中間データ
    val tempMap = mutableMapOf<Int, Map<DepartmentType, LocalDate>>()

    this.forEach { department ->
        // Departmentのタイプとその日付のPair
        val departmentJoined = department.departmentType to department.date

        department.employers.forEach { employer ->

            // EmployerのIDがKeyとして存在したら足して、なかったらMapを追加
            tempMap.compute(employer.id) { _, value ->
                value?.let { value + departmentJoined } ?: mapOf(departmentJoined)
            }
        }
    }

    // 中間データをJoinedDatesのListに変えて返却
    return tempMap.map { (id, department) ->
        JoinedDates(
            employerId = id,
            departmentA = department.getValue(DepartmentType.A),
            departmentB = department.getValue(DepartmentType.B)
        )
    }
}
```

## 共通ロジック化

ジェネリックを使ったclassとして上記のロジックを一部分離すれば、似たようなケースで色々使い回せるのではないかと思いましたので、以下のようなコードも書いてみました。

```kotlin
class Aggregator<T, K, V, R> {
    private val tempMap = mutableMapOf<T, Map<K, V>>()

    // データの追加
    fun add(key: T, value: Pair<K, V>) {
        tempMap.compute(key) { _, existingValue ->
            existingValue?.let { existingValue + value } ?: mapOf(value)
        }
    }

    // 指定したListとして取得
    fun getList(transfer: (T, Map<K, V>) -> R): List<R> {
        return tempMap.map { transfer(it.key, it.value) }
    }
}
```

これの場合は以下のような使い方ができます。

```kotlin
val aggregator = Aggregator<Int, DepartmentType, LocalDate, JoinedDates>()

// データの追加
departments.forEach { a ->
    a.employers.forEach { b ->
        aggregator.add(
            key = b.id,
            value = a.departmentType to a.date
        )
    }
}

// Listの結果を取得
val joinedDates = aggregator.getList { id, joinedDate ->
    JoinedDates(
        employerId = id,
        departmentA = joinedDate.getValue(DepartmentType.A),
        departmentB = joinedDate.getValue(DepartmentType.B)
    )
}
```

汎用性はあるものの、呼び出し元のコードが増えたり、指定している型の意味や意図がよくわからないので適切なKDocやコメントがないと少しわかりにくいところがデメリットかもしれませんね。ただ大事なのは中間データの型と`compute()`による分岐処理なので、そこだけをうまく取り出して他でも応用できるかなと思います。

## 最後に

サーバサイドKoltinだと、多くの場合にデータを`List`として扱うのが普通かなと思いますが、場合によっては`Map`を使うのもロジックを書いていく中では良い選択になるかと思います。特に、今回紹介した`compute()`以外でも、[getOrPut()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/get-or-put.html)、[getOrDefault()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-map/get-or-default.html)などの機能が便利なので色々と活用できる場面が多いかなと思います。この処理は[前回のポスト](../exposed-mapping-record-to-object)でも似たようなものを紹介したことがありますので、興味のある方はそちらも参考にしてください。

プログラミング言語が提供するスタンダードライブラリは色々と見逃しやすいところがあるかなと思いますが、よくドキュメントや自動補完で一覧に登場する関数に注目すると、こういう風に必要なものがいきなり現れることもあるかと思います。まだ私もKotlinを触って1年ほどしか経ってないひよこなものなので、これからもどんどん新しい発見があると嬉しいなと思いますね。

では、また！
