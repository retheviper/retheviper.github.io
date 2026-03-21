---
title: "Kotlin으로 데스크톱 앱 만들기"
date: 2022-09-09
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - compose
  - gui
translationKey: "posts/kotlin-compose-desktop"
---

백엔드 개발을 하다 보면 자동화 테스트만으로는 충분하지 않은 경우가 있습니다. 이상적으로는 유닛 테스트부터 통합 테스트, 시나리오 테스트까지 모두 잘 갖춰져 있으면 좋겠지만, 현실에서는 늘 그렇게 되지 않습니다. 서비스가 커지면서 사양이 바뀌고, 기술 부채를 줄이거나 운영 대응을 얹는 과정에서 처음 가정했던 흐름이 계속 흔들리기 때문입니다.
그래서 지금 상태의 애플리케이션이 실제로 예상대로 동작하는지 사람 손으로 확인해야 하는 장면도 생깁니다. 특히 테스트용 데이터 패턴이 많거나, 기획 쪽에서 직접 여러 경우를 눌러 보며 확인해야 하는 경우에는 더 그렇습니다. 이번 글은 그런 수동 테스트를 돕기 위해 Compose for Desktop으로 간단한 테스트 도구를 만든 이야기입니다.
## 목표와 설계, 기술 선정

사실 리포지토리 안에는 예전부터 테스트 도구가 하나 있었습니다. 다만 확인해 보니 2년 넘게 방치돼 있었고, 하필 제가 거의 다뤄 본 적도 없는 Ruby on Rails로 만들어져 있었습니다. 물론 Ruby를 새로 공부해서 기존 도구를 고쳐 쓰는 방법도 있었겠지만, 아래 이유 때문에 처음부터 다시 만드는 편이 낫다고 판단했습니다.
1. Kotlin 엔지니어가 Ruby 앱을 유지하는 것은 좋지 않습니다.
2. 문서화가 진행되지 않고 사용이 불편함

방향을 그렇게 정한 뒤에는 실제로 테스트를 수행할 쪽, 즉 기획 측 요청을 받아 필요한 기능을 정리했습니다. 목표가 "테스트를 할 수 있는 도구"로 비교적 명확했기 때문에 요구사항 자체는 단순한 편이었습니다.
- 테스트 데이터 파일을 불러오기
- 백엔드 앱 API 호출
- API 실행 결과를 파일에 기록

테스트 툴로서는 상기에 열거하고 있는 요구 사항을 만족하고 있다면 테스트 툴로서는 합격이라고 하는 것입니다. 그러나 실제 테스트를 하고 싶은 쪽이 먼저 엔지니어가 아니고 앞으로도 엔지니어가 아닌 사람이 도구를 만질 가능성이 있습니다. 거기까지를 고려하여 다음과 같은 추가 목표를 세웠습니다.
- 별도 환경 구축 없이 바로 쓸 수 있어야 한다
- 절차서를 보지 않아도 대충 만져서 쓸 수 있을 만큼 단순해야 한다

여기까지 결정하면 다음에 요구되는 기능의 세부 사항을 파고 갑니다. 설계서를 쓸 정도는 아니지만, 토대가 되는 설계와 같습니다.
- 데이터 읽기 및 쓰기
  - 도구를 사용할 수 있는 사람은 SQL을 사용할 수 있다 = 테이블(표)을 읽을 수 있다
  - 테이블의 형태로 데이터의 입출력을 할 수 있는 것이 알기 쉽다
  - 테스트 데이터는 CSV로 로드
  - API 실행 결과도 CSV에 기록
- API 호출 가능
  - HTTP 클라이언트로 GET · POST하기
  - API 호출에는 토큰 필요
    - 토큰은 보안 문제로 소스 코드에 포함되는 NG
    - 그러나 매번 입력하는 것은 귀찮습니다.
    - 앱을 실행하고 처음에는 토큰을 입력하고 다음에 토큰을 사용하도록 합니다.
- 프로덕션 이외의 환경이 대상
  - 여러 환경이 있으므로 어느 것을 선택할 수 있습니다.
  - 이것도 매번 입력은 귀찮습니다.
  - 먼저 한 번만 선택할 수 있도록 하고 싶습니다.

이 요구사항을 다시 기술 관점에서 보면 방향은 꽤 명확했습니다. 환경 구축 없이 써야 한다면 실행 가능한 바이너리 형태가 좋고, 사용법이 단순해야 한다면 GUI가 더 잘 맞습니다.

예를 들어 파일을 읽고 쓰려면 경로 선택이 필요하고, 토큰과 환경 선택도 입력 UI가 필요합니다. 이런 걸 CLI로 풀면 엔지니어가 아닌 사람에게는 꽤 불편합니다. 반면 GUI라면 파일 경로는 다이얼로그로, 토큰은 텍스트 박스로, 환경 선택은 드롭다운 메뉴로 자연스럽게 대응할 수 있습니다. 그래서 최종적으로는 "실행 파일 하나로 시작할 수 있는 GUI 앱"을 만들기로 했습니다.
## Compose for Desktop

요구사항이 정리되면 다음은 기술 선택입니다. 우선 언어는 Kotlin으로 정했습니다. 앞으로도 같은 팀의 Kotlin 엔지니어가 유지보수할 가능성이 높았기 때문입니다. 게다가 이미 백엔드 앱에서 쓰던 HTTP 클라이언트가 있어서 일부 코드를 그대로 옮겨 올 수 있다는 점도 컸습니다.

GUI 프레임워크로는 [Compose for Desktop](https://www.jetbrains.com/ko-kr/lp/compose-desktop/)을 선택했습니다. Kotlin은 Java와 호환되니 [Swing](https://ko.wikipedia.org/wiki/Swing), [JavaFX](https://openjfx.io/) 같은 기존 툴킷도 당연히 후보였고, [TornadoFX](https://github.com/edvin/tornadofx) 같은 선택지도 있었습니다. 그래도 이번에 굳이 Compose를 택한 데에는 몇 가지 이유가 있었습니다.
개인적으로 모바일 쪽에도 관심이 있어서 Compose를 한 번 제대로 써 보고 싶었던 이유도 있었습니다. 그리고 앞으로도 Kotlin 엔지니어가 유지보수할 가능성이 높다면, 모바일 경험이 있거나 적어도 Compose에 흥미를 가진 사람이 팀 안에 있을 가능성도 높다고 봤습니다. Compose는 비교적 새로운 기술이지만, 선언형 UI라는 흐름을 생각하면 Android 쪽에서도 메인스트림이 될 가능성이 크다고 판단했습니다.

또 Compose는 원래 멀티플랫폼을 염두에 두고 발전해 온 덕분에 Windows, macOS, Linux를 가리지 않고 실행 파일을 만들 수 있다는 점도 매력적이었습니다. 테스터가 어떤 OS를 쓰더라도 비슷한 방식으로 도구를 사용할 수 있으니까요.

물론 처음 써 보는 기술인 만큼 시행착오도 있었습니다. 그래서 아래에서는 실제로 도구를 만들면서 기억해 둘 만하다고 느낀 점 몇 가지를 적어 보겠습니다.
### 상태 관리

[SwiftUI를 써 본 첫인상 2](../swift-ui-first-impression-2/)에서도 접한 상태 관리입니다만, Compose에서도 마찬가지로 GUI를 취급하게 되므로, 상태 관리가 중요합니다. 이번에는 앱으로서의 화면이 하나 밖에 없기 때문에, 복수의 화면에 걸쳐 상태를 관리할 필요는 없을까라고 생각했습니다만, 그래도 역시 처리를 실시하기 위해서는 앱 전체에서 공유하는 상태로서 관리가 필요한 것이 몇가지 있었습니다.
다만, 상기 포스트에서도 말했듯이, SwiftUI와 Compose와는 상태 관리의 방식이 조금 다릅니다. SwiftUI에서는 상태가 어디에서 사용되는지에 따라 명확하게 사용되는 어노테이션이나 클래스 등이 바뀌었다면, Compose에서는 대체로 [remember()(https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-sum mary?hl=ko#remember(kotlin.Function0))과 [`MutableState<T>`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState?hl=ko)의 조합으로 충분합니다. 화면의 구성 요소의 최소 단위를 Compose에서는 Widget에서도 사용법이 같다는 것은, SwiftUI와 비교하면 정의하는 것은 간단합니다만, 사용법에는 조금 주의가 필요하다고 하는 감각이었습니다.
첫째, Compose의 상태는 다음과 같은 세 가지 방법으로 정의할 수 있습니다.
```kotlin
// Delegate로 정의한다
var isOn: Boolean by remember { mutableStateOf(false) }
// 값을 직접 바꿀 수 있다
isOn = false

// 분해 선언으로 정의한다
val (isOff: Boolean, setIsOff: (Boolean) -> Unit) = remember { mutableStateOf(true) }
// 참조와 갱신이 분리된다
if (isOff) {
    setIsOff(!isOff) // toggle
}

// MutableState<T>로 다룬다
val isNotOff: MutableState<Boolean> = remember { mutableStateOf(false) }
// 래퍼이므로 값을 갱신하려면 value에 접근해야 한다
isNotOff.value = !isNotOff.value
```

여기서 Delegate에서 `var`로 정의하면 가장 사용하기 쉽지만 Intellij에서는 컴파일 오류가 발생하기 쉽습니다. 왜냐하면 Delegate를 사용하려면 [`androidx.compose.runtime.setValue`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#(androidx.compose.runtime.MutableState).setValue(kotlin.Any,kotlin.reflect.KProperty,kotlin.String)) 및 [`androidx.compose.runtime.getValue`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#(androidx.compose.runtime.State).getValue(kotlin.Any,kotlin.reflect.KProperty))를 import해야 하지만 이것이 자동으로 수행되지 않기 때문입니다. 처음 이 에러의 이유를 모르거나 바쁜 경우에 일일이 import문을 써 가는 것이 꽤 번거로울 수 있습니다. 다만 이것은 아직 Intellij의 Compose 대응이 완벽하지 않은 데서 오는 문제이므로, 언젠가는 해소될 가능성이 큽니다.
분해 선언에서 값의 참조와 갱신을 따로 사용하는 방식은 처음 보면 다소 낯설 수 있습니다. 하지만 Compose의 일부 Widget에 상태를 건네줄 때는 꽤 자연스럽습니다. 대표적인 예가 [`TextField`](https://developer.android.com/reference/kotlin/androidx/compose/material/package-summary#TextField(kotlin.String,kotlin.Function1,androidx.compose.ui.Modifier,kotlin.Boolean,kotlin.Boolean,androidx.compose.ui.text.TextStyle,kotlin.Function0,kotlin.Function0,kotlin.Function0,kotlin.Function0,kotlin.Boolean,androidx.compose.ui.text.input.VisualTransformation,androidx.compose.foundation.text.KeyboardOptions,androidx.compose.foundation.text.KeyboardActions,kotlin.Boolean,kotlin.Int,androidx.compose.foundation.interaction.MutableInteractionSource,androidx.compose.ui.graphics.Shape,androidx.compose.material.TextFieldColors))입니다. 실제 코드에서는 다음과 같이 사용됩니다.
```kotlin
val (text: String, updateText: (String) -> Unit) = remember { mutableStateOf("") }
TextField(
    onValueChange = setValue, // TextField에 문자를 입력하면 그 값으로 text를 갱신한다
    value = text // text 값을 TextField에 표시한다
)
```

마지막으로 `MutableState<T>` 자체를 들고 가는 방식입니다. 직접 값을 바꾸려면 `.value`에 접근해야 해서 가장 불편해 보이지만, 실제로는 꽤 자주 쓰게 됩니다. 앱 전체에서 상태를 공유하거나 여러 Widget 사이를 넘겨야 할 때는 아래처럼 클래스 필드로 두는 방식이 자연스럽기 때문입니다.
```kotlin
// 앱 전체에서 공유하기 위해 클래스에 상태를 정의한다
class AppState(
    val isOn: MutableState<Boolean>
)
```

물론 [getter/setter](https://kotlinlang.org/docs/properties.html#getters-and-setters)를 따로 두면 `.value`에 직접 접근하지 않고 일반 프로퍼티처럼 다룰 수도 있습니다. 다만 상태로 관리할 항목이 많아질수록 이 방식은 코드 양이 빠르게 늘어난다는 단점이 있습니다.
```kotlin
class AppState(
    private val _isOn: MutableState<Boolean>
) {
    var isOn: Boolean
        get() = _isOn.value
        set(value) { _isOn.value = value }
}
```

이와 같이, Compose에서의 상태에는 정의하는 방법이 여러 가지 있어, 각각의 특징이 있는 것이므로 어느 장면에서 사용하는가에 따라서 적절한 정의의 방법을 생각하는 것이 무엇보다 중요하다는 인상입니다.
### Swing/AWT

Compose for Desktop의 장점 중 하나는 Swing과 [AWT](https://en.wikipedia.org/wiki/Abstract_Window_Toolkit)와의 호환성입니다. 처음에는 기본적인 기능이 대부분 다 들어 있을 것 같았지만, 실제로는 일부 기능을 Swing이나 AWT에 기대야 하는 경우도 있었습니다. 제가 만든 테스트 도구에서도 그런 지점이 몇 군데 있었습니다.
예를 들어 파일 선택 기능이 그렇습니다. CSV를 불러오기 위해 파일 선택 다이얼로그를 띄우고 싶었는데, 당시에는 Compose 위젯만으로는 이 부분을 처리하기 어려워서 AWT의 [FileDialog](https://docs.oracle.com/javase/ko/8/docs/api/java/awt/FileDialog.html)를 사용해야 했습니다. 아래는 구현 예입니다.
```kotlin
 // 선택한 파일 이름을 상태로 보관한다
var fileName by remember { mutableStateOf("") }
// AWT 파일 선택 대화상자를 사용한다
FileDialog(ComposeWindow()).apply {
        // 선택 가능한 파일은 CSV로 제한한다
        setFilenameFilter { _, name -> name.endsWith(".csv", ignoreCase = true) }
        isVisible = true
        // 파일이 선택되면 상태를 갱신한다
        if (!file.isNullOrBlank()) {
            fileName = file
        }
    }
```

하지만 이것만으로 충분하지는 않았습니다. 폴더만 고를 수 있게 하고 싶을 때는 `FileDialog`가 그다지 좋은 선택이 아니었기 때문입니다. 이름 그대로 파일 선택을 전제로 한 컴포넌트라, 폴더 선택 전용으로 쓰기에는 제약이 있었습니다. 그래서 이 경우에는 Swing 쪽 도움도 받아야 했습니다. 코드는 아래와 같습니다.
```kotlin
// 선택한 폴더 경로를 상태로 보관한다
var selectedPath by remember { mutableStateOf("") }
// Swing 파일 선택 대화상자를 디렉터리만 선택하도록 설정한다
val fileChooser = JFileChooser().apply {
        dialogTitle = "Choose Directory"
        fileSelectionMode = JFileChooser.DIRECTORIES_ONLY
    }
// 대화상자를 표시한다
if (fileChooser.showOpenDialog(ComposeWindow()) == JFileChooser.APPROVE_OPTION) {
    // 대화상자에서 선택한 경로가 현재 상태와 다르면 선택한 디렉터리의 절대 경로로 갱신한다
    val path = fileChooser.selectedFile.absolutePath
    if (selectedPath != path) {
        selectedPath = path
    }
}
```

이번 도구에서는 이 두 가지 정도만 Swing/AWT를 함께 썼지만, 어떤 앱을 만드느냐에 따라 다른 API가 더 필요해질 수도 있다는 걸 보여 주는 사례라고 생각합니다. Compose가 아직 비교적 새로운 도구인 만큼, 앞으로 버전이 올라가면서 더 다양한 위젯이 들어오길 기대하게 됩니다.
### 빌드

Compose를 고른 이유 중 하나였던 "실행 가능한 바이너리를 만들 수 있다"는 점은 실제로도 만족도가 높았습니다. `gradle` 명령 하나로 실행 파일 패키지를 만들 수 있고, macOS에서 빌드하면 일반 앱처럼 패키지가 생성됩니다. 내부를 보면 실행에 필요한 JRE와 의존 Jar가 함께 들어 있어서, 완전한 네이티브 앱이라기보다 JVM 런타임을 포함한 패키지에 가깝습니다.

빌드 옵션도 꽤 다양합니다. OS별로 다른 아이콘을 쓰거나, 기본 포함 대상이 아닌 모듈을 따로 넣는 식의 설정이 가능합니다. 아래는 실제 예시입니다.
```kotlin
compose.desktop {
    application {
        mainClass = "com.testtool.MainKt" // 실행 시 메인 클래스를 지정
        nativeDistributions {
            packageName = "Test Tool"
            packageVersion = "1.0.0"
            modules("java.sql", "java.naming") // 기본값에 포함되지 않는 패키지를 추가
            macOS {
                iconFile.set(project.file("misc/appicon/icon-mac.icns"))
            }
            windows {
                iconFile.set(project.file("misc/appicon/icon-win.ico"))
            }
        }
    }
}
```

다만 빌드할 때는 주의가 필요합니다. Compose는 내부적으로 [jpackage](https://docs.oracle.com/javase/ko/14/docs/specs/man/jpackage.html)를 사용하므로 우선 Java 15 이상이 필요합니다. 또 CPU 아키텍처에 따라 설치하는 JDK가 달라지기 때문에, 빌드하는 머신과 다른 아키텍처를 대상으로 한 바이너리를 바로 만들 수는 없습니다.
즉 제 Mac에서 빌드하면 Apple Silicon용 바이너리가 나오고, Intel Mac에서 빌드하면 x64용 바이너리가 나옵니다. 실제로 Compose로 App Store에 앱을 제출한 사례도 보이지만, 당시에는 Rosetta 실행을 전제로 Intel Mac에서 빌드한 경우가 적지 않아 보였습니다. Universal Binary까지 고려한다면 결국 JDK 쪽 지원 상황도 함께 봐야 합니다.
## 마지막으로

이번에는 기능이 제한된 단순한 도구를 만들었기 때문에, 앞으로 Compose로 멀티 윈도우나 다크 모드, 내비게이션처럼 더 많은 기능을 얹어 보면 또 다른 발견이 있을 것 같습니다. 개인적으로는 꽤 좋은 경험이었고, 생각보다 구현 난이도도 높지 않았습니다. 비슷한 도구를 다시 만들 기회가 있다면 Compose를 또 써 볼 것 같습니다.
아직 역사가 아주 긴 프레임워크는 아니어서 부족한 기능이나 참고 자료도 있지만, Kotlin을 주력으로 쓰는 엔지니어라면 한 번쯤 시도해 볼 가치는 충분합니다. 특히 GUI 쪽에 관심이 있다면 더 그렇습니다.
