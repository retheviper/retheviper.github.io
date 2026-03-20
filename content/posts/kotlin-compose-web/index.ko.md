---
title: "Kotlin만으로 파일 서버 만들기"
date: 2022-10-10
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - compose
  - gui
translationKey: "posts/kotlin-compose-web"
---

프로그래밍 언어마다 잘하는 일이 다르고, 실제 프로젝트에서는 채용 난이도나 운영 비용 같은 현실적인 조건까지 함께 고려하게 됩니다. 그래서 회사 일에서는 기술 선택의 기준이 비교적 분명한 편입니다. 반면 개인 프로젝트는 조금 다릅니다. 팀 사정에 맞출 필요가 없으니, 익숙한 언어나 써 보고 싶은 도구를 기준으로 골라도 큰 문제가 없지요.

저도 개인적으로 만드는 작은 앱이나 자동화 스크립트는 가능한 한 Kotlin이나 Python으로 작성합니다. 그중에서도 Kotlin은 업무에서도 계속 쓰고 있어 가장 손에 익은 언어입니다. 게다가 서버 사이드나 JVM에만 갇히지 않고, 여러 프레임워크와 플랫폼으로 확장해 볼 수 있다는 점도 꽤 매력적입니다.

이번에 만든 것도 그런 흐름의 연장선에 있습니다. 제목 그대로, Kotlin만으로 파일 서버용 웹 애플리케이션을 만들어 본 이야기입니다.
## 배경

먼저 왜 이런 앱을 만들게 되었는지부터 설명하겠습니다. 본가에는 예전에 조립해 둔 Windows PC가 한 대 있는데, 요즘은 제가 자주 내려가지 않아서 평소에는 거의 쓰지 않습니다. 그래도 지금 쓰는 Mac과 파일을 주고받아야 할 때가 가끔 있습니다.
그동안은 Microsoft의 [OneDrive](https://www.microsoft.com/ko-kr/microsoft-365/onedrive/online-cloud-storage)를 이용해 파일을 옮겼습니다. 한쪽에서 파일을 폴더에 넣어 두면 클라우드로 업로드되고, 반대편 PC에서 자동으로 동기화되는 방식입니다. 안정적으로 잘 동작하긴 했지만, 가만히 생각해 보면 클라우드를 한 번 거치는 과정이 조금 번거롭게 느껴졌습니다. 동기화 전후로 파일을 복사하거나 정리하는 작업도 은근히 손이 갔고요.
그래서 인터넷을 통해 직접 파일을 주고받는 간단한 앱을 하나 만들어 보면 어떨까 싶었습니다. 비슷한 기능을 제공하는 서비스는 이미 있을 수 있지만, 요구사항 자체가 복잡하지 않았고 며칠 정도면 시도해 볼 수 있겠다는 판단이 들었습니다. SFTP 같은 선택지도 있었지만, 이번에는 GUI로 좀 더 편하게 쓰고 싶었습니다.
## 요구사항

무언가 만들기 시작하면 우선 요구사항부터 간단히 정리해 두는 편입니다. 이번에는 기능적으로 아래 정도만 되면 충분하다고 봤습니다.
- 서버 앱을 시작하면 클라이언트가 서버 스토리지에 액세스할 수 있습니다.
- 서버 경로를 지정하면 해당 내용(파일 및 폴더)을 볼 수 있습니다.
- 폴더를 클릭하면 표시중인 경로가 변경됩니다.
- 파일을 클릭하면 다운로드 가능
- 경로에 파일을 업로드할 수 있습니다.

기능이 정해졌으니 이제는 기술 선택입니다. 이번에는 처음부터 "가능한 한 전부 Kotlin으로 해결해 보자"라는 방향을 잡았습니다.
프론트엔드는 [Compose for Web](https://compose-web.ui.pages.jetbrains.team/)을 써 보기로 했습니다. 예전에 [Compose for Desktop](https://www.jetbrains.com/lp/compose-desktop/)으로 간단한 앱을 만들어 본 적이 있었는데, 같은 계열의 도구로 웹까지 다뤄 보면 재미있겠다는 생각이 들었습니다. 언어를 통일하고 싶다는 점도 컸고, 프론트엔드 경험이 많지 않다 보니 완전히 새로운 도구보다 조금이라도 손에 익은 기술을 고르고 싶었습니다.
백엔드는 [Ktor](https://ktor.io/)로 정했습니다. 평소에는 주로 Spring을 쓰지만, Ktor도 예전에 잠깐 만져 봤을 때 기동이 빠르고 구현이 가볍다는 인상이 있었습니다. 여기도 결국 Kotlin 친화적이라는 점이 가장 큰 이유였습니다.
큰 축은 이렇게 프론트엔드의 Compose for Web과 백엔드의 Ktor 두 가지입니다. 그 외 라이브러리는 구현하면서 필요한 것을 골랐고, 가능하면 Kotlin 기반이거나 JetBrains에서 제공하는 도구, 또는 공식 문서에서 직접 안내하는 선택지를 우선했습니다.
## Frontend

프론트엔드는 앞에서 말한 Compose for Web으로 구현했습니다. 처음 써 보는 기술이기도 했고, 제 쪽 프론트엔드 경험도 많지 않아서 결과적으로 가장 시간을 많이 쓴 부분이었습니다. 여기서는 실제로 써 보면서 느낀 점을 `좋았던 점`, `생각과 달랐던 점`, `문제가 되었던 점` 세 가지로 나눠 보겠습니다.
### 좋은 점

가장 좋았던 점은 역시 Compose로 데스크톱 앱을 만들어 본 경험을 그대로 살릴 수 있었다는 점입니다. Compose에서는 [`remember`와 `MutableState<T>`](https://developer.android.com/jetpack/compose/state#state-in-composables)을 조합해 상태를 관리하고, [`@Composable`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/Composable) 함수 단위로 화면 요소를 나눠 구현합니다. 이 감각은 웹에서도 거의 그대로 이어졌습니다.
예를 들어 "지정한 경로를 탐색한다" 같은 기능을 만들 때도 현재 경로를 상태로 들고 있고, 서버에서 받아 온 파일이나 폴더 목록을 표시하는 UI를 별도의 `@Composable` 함수로 분리해 다룰 수 있었습니다. 데스크톱 Compose에서 익숙했던 방식이 크게 흔들리지 않았다는 점이 특히 좋았습니다.
그 밖에도 Kotlin이기 때문에 Coroutine을 자연스럽게 쓸 수 있고, 서버 측과 같은 리포지토리 안에서 소스 코드를 함께 관리할 수 있다는 점도 장점이었습니다. 특히 Gradle에서 Kotlin 플러그인을 `multiplatform`으로 두면 프런트엔드는 JavaScript로, 서버 측은 JVM 바이트코드로 컴파일되기 때문에 프로젝트를 한 덩어리로 다루기 편했습니다.
### 생각한 것과 다른 점

물론 데스크톱 Compose와는 꽤 다른 부분도 있었습니다. 가장 크게 느낀 점은, 언어로는 Kotlin을 쓰더라도 HTML과 CSS를 완전히 피할 수는 없다는 점입니다. 여기서도 `div`나 `form` 같은 태그를 다뤄야 하고, 마우스 오버 시 커서를 바꾸는 식의 동작도 결국 태그의 `attr`을 조절해야 했습니다. 예를 들어 아래 파일 업로드 컴포넌트는 Kotlin으로 작성하지만, 감각적으로는 HTML을 그대로 쓰는 것에 가깝습니다.
```kotlin
@Composable
private fun FileUploadForm(currentPath: String) {
    Div {
        Form(
            action = "$API_URL$ENDPOINT_UPLOAD",
            attrs = {
                method(FormMethod.Post)
                encType(FormEncType.MultipartFormData)
                target(FormTarget.Blank)
            }
        ) {
            HiddenInput {
                name("target")
                value(currentPath)
            }
            Input(InputType.File) { name("file") }
            Input(InputType.Submit)
        }
    }
}
```

이 부분은 데스크톱에서 Compose를 쓰는 경험과는 꽤 달랐습니다. Kotlin으로 작성하긴 하지만, 실제로는 HTML과 CSS를 Kotlin DSL로 감싼 형태에 가깝다는 인상이 강했습니다. 결국 어느 정도의 프런트엔드 지식은 여전히 필요합니다. 이미 [React](https://reactjs.org/)나 [Vue.js](https://vuejs.org/) 같은 도구에 익숙하다면, 굳이 Compose for Web을 선택해야 할 이유는 크지 않을 수도 있습니다.
또 하나는 Kotlin/JS와 Kotlin/JVM이 한 프로젝트 안에 공존하다 보니, IntelliJ의 자동완성이나 Gradle 변경 반영 방식도 평소와는 조금 다르게 느껴졌습니다. 예를 들어 의존성을 바꿔도 바로 반영되지 않는 경우가 있었습니다.
### 문제였던 점

의외로 가장 애를 먹인 부분은 빌드였습니다. Compose for Web은 `index.html`과 Webpack으로 번들된 `js` 파일을 조합해 동작하므로, 겉보기에는 Gradle 명령 하나로 끝나는 단순한 구조입니다. 하지만 내부적으로 [yarn](https://yarnpkg.com/) 같은 도구가 함께 움직이다 보니 생각보다 변수가 많았습니다.
비슷한 오류 사례를 찾아보니, [Kotlin 버전을 `v1.6.20` 이상으로 올리면 해결된다](https://github.com/Kotlin/kotlinx-datetime/issues/193)는 이야기가 있었습니다. 문제는 당시 최신 Compose 버전이던 [v1.1.1](https://github.com/JetBrains/compose-jb/releases/tag/v1.1.1)이 `v1.6.10`까지만 지원한다는 점이었습니다. 결국 제 경우에는 Compose를 `v1.2.0` 베타로 올리고 Kotlin을 `v1.7.10`으로 맞추는 방식으로 해결했습니다. 새 기술을 쓸 때 자주 맞닥뜨리는 종류의 문제이긴 합니다.
또 HTTP 클라이언트로 [Ktor Client](https://ktor.io/docs/getting-started-ktor-client.html)를 쓰고 있었는데, 대용량 파일 업로드를 고려해 `form` 태그 대신 클라이언트 코드로 Multipart 전송을 처리하려고 하니 기대만큼 잘 되지 않았습니다. Ktor Client는 멀티플랫폼 대응이라 [엔진을 선택할 수](https://ktor.io/docs/http-client-engines.html) 있지만, Kotlin/JS에서 사용할 수 있는 엔진으로는 [공식 예제](https://ktor.io/docs/request.html#binary) 방식대로 구현해도 `File` 객체를 직접 다루는 데 제약이 있었습니다. 이 부분은 앞으로 개선을 기다리거나, WebSocket 같은 다른 접근을 검토해야 할 듯합니다.
## 백엔드

다음은 백엔드입니다. 이쪽은 비교적 익숙한 영역이고, Ktor 자체에 대해서도 다른 글에서 다룬 적이 있어서, 여기서는 기술 소개보다는 로직 구현에서 겪은 시행착오를 중심으로 정리하겠습니다.
### 파일 트리 찾아보기

이 앱은 먼저 파일을 탐색할 수 있어야 하므로, 클라이언트가 지정한 경로를 읽고 그 안의 파일과 폴더를 반환해야 합니다. 문제는 이 결과를 어떤 JSON 구조로 돌려줄지였습니다. 처음에는 한 가지 방식을 먼저 구현해 보고, 그다음 실제 사용성을 보며 판단하기로 했습니다.
#### 모두 얻기

처음에는 다음과 같은 형태로 구현을 시도했습니다. 경로를 지정하면 그 아래에 있는 모든 폴더를 따라 가서 부모와 자식 관계를 중첩으로 표현하는 형태입니다.
```json
{
  {
    "name": "Documents",
    "type": "directory",
    "children": [
      {
        "name": "SomeDocument.pdf",
        "type": "file",
        "size": "1391482",
        "mimeType": "application/pdf"
      }
    ]
  },
  {
    "name": "Images",
    "type": "directory",
    "children": []
  }
}
```

이런 파일 트리를 반환하기 위해 서버 쪽에서는 아래와 같은 코드를 사용했습니다.
```kotlin
// 루트 경로를 지정하면 자식 요소(파일과 폴더)를 모두 가져온다
val files =  Files.list(root)
        .filter { !it.isHidden() }
        .map { it.toFileTree() }
        .toList()

// Path를 JSON 객체로 변환한다
fun Path.toFileTree(): FileTree {
    return FileTree(
        name = this.fileName.toString(),
        size = if (this.isDirectory()) null else this.fileSize(),
        type = if (this.isDirectory()) FileType.DIRECTORY else FileType.FILE,
        children = if (this.isDirectory()) {
            Files.list(this)
                .filter { !it.isHidden() }
                .map { it.toFileTree() }
                .toList()
        } else {
            null
        }
    )
}
```

[Files.walk()](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#walk-java.nio.file.Path-java.nio.file.FileVisitOption...-)를 쓰면 지정한 경로 아래의 파일 트리를 전부 `Stream<Path>`로 가져올 수 있습니다. 다만 그 결과를 위와 같은 JSON 구조로 다시 묶는 일은 생각보다 간단하지 않습니다. 한 번 펼쳐진 결과를 기준으로 부모-자식 관계를 다시 추적해 중첩 JSON으로 합쳐야 하기 때문입니다.

그래서 여기서는 [Files.list()](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#list-java.nio.file.Path-)를 사용해, 지정한 경로 바로 아래 요소만 가져오고 디렉터리일 경우에만 다시 자식 요소를 읽는 방식으로 구현했습니다. 재귀는 들어가지만 구조는 훨씬 단순합니다.

다만 이렇게 하면 원하는 파일 트리를 JSON으로 돌려줄 수는 있어도 다른 문제가 생깁니다. 경로가 루트에 가까워질수록 탐색 시간이 길어지고, 응답 JSON도 빠르게 커집니다. 게다가 프런트엔드에서 이 중첩 구조를 그리는 쪽도 만만치 않아 보였습니다. 그래서 이 방법은 일단 접고 두 번째 방식을 택했습니다.
#### 중첩하지 않음

다음으로 시도한 방법은, 처리를 지정한 경로 바로 아래로만 제한하는 것이었습니다. 중첩 JSON을 없애고, 해당 경로에 어떤 파일과 폴더가 들어 있는지만 목록으로 돌려주는 방식입니다. 형태는 아래와 같습니다.
```json
[
  {
    "name": "Documents",
    "type": "directory"
  },
  {
    "name": "Images",
    "type": "directory"
  }
]
```

이렇게 하면 내부 폴더를 한 번에 전부 추적할 필요가 없어지므로 응답도 더 빠르고 가벼워집니다. 처음부터 이 방향으로 갔어야 했다는 생각도 들었지만, 결과적으로는 이 방식이 프런트엔드 구현에도 더 잘 맞았습니다. 하위 폴더로 더 들어가고 싶다면, 그 경로만 다시 백엔드에 보내서 목록을 받아 오면 됩니다. 코드도 재귀를 없애면서 훨씬 단순해졌습니다.
```kotlin
val files =  Files.list(root)
        .filter { !it.isHidden() }
        .map { it.toFileTree() }
        .toList()

// Path를 JSON 객체로 변환한다
fun Path.toFileTree(): FileTree {
    return FileTree(
        name = this.fileName.toString(),
        size = if (this.isDirectory()) null else this.fileSize(),
        type = if (this.isDirectory()) FileType.DIRECTORY else FileType.FILE
    )
}
```

### 파일 업로드

파일 업로드는 Multipart 데이터를 어떻게 처리하느냐의 문제인데, 이 부분은 Ktor답게 비교적 단순하게 구현할 수 있었습니다. 아래는 실제 코드입니다.
```kotlin
// router
post(ENDPOINT_UPLOAD) {
    // Multipart 데이터를 수신
    val multipart = call.receiveMultipart()
    // 파일 저장 경로
    var path = Path.of(ROOT_DIRECTORY)
    multipart.forEachPart { part ->
        when (part) {
            // 루트가 아닌 경로를 지정하면 저장 경로를 갱신
            is PartData.FormItem -> {
                if (part.name == "target") {
                    path = path.resolve(part.value)
                }
            }
            // 파일 데이터를 저장
            is PartData.FileItem -> {
                withContext(Dispatchers.IO) {
                    val file = Files.createFile(path.resolve(part.originalFileName!!))
                    part.streamProvider().use { input ->
                        Files.newOutputStream(file).use { output ->
                            input.copyTo(output)
                        }
                    }
                }
            }
            // 둘 다 아니면 일단 출력
            else -> {
                println("Unknown part: $part")
            }
        }
        // 처리가 끝난 데이터는 dispose
        part.dispose()
    }
}
```

다만 개인적으로는 파일 I/O가 들어가는 처리에서 NIO를 더 선호해서, 처음에는 [Files.copy()](https://docs.oracle.com/javase/ko/8/docs/api/java/nio/file/Files.html#copy-java.io.InputStream-java.nio.file.Path-java.nio.file.CopyOption...-)를 써 보려고 했습니다. 그런데 아래처럼 쓰면 파일 저장이 기대한 대로 되지 않았습니다. Coroutine과의 조합에서 주의할 점이 있는 것 같아서, 이 부분은 조금 더 확인이 필요했습니다.
```kotlin
val file = Files.createFile(path.resolve(part.originalFileName!!))
Files.copy(part.streamProvider(), file) // 파일이 저장되지 않는다
```

### 파일 다운로드

파일 다운로드 쪽도 로직 자체는 복잡하지 않아서, 거의 Ktor 문법에 맞게 구현한 수준입니다. 저는 개인적으로 `Path`를 선호해서 그렇게 작성했을 뿐이라 여기서는 코드만 간단히 소개하겠습니다. 한 가지 주의할 점은 업로드 때와 마찬가지로, 파일명을 응답에 넣을 때 URL 경로 규칙에 맞게 인코딩해야 한다는 점입니다.
```kotlin
get(ENDPOINT_DOWNLOAD) {try {
    val filepath = call.request.queryParameters["filepath"] ?: ""
    val path = FileService.getFullPath(filepath) // 루트 디렉터리부터의 전체 경로를 가져온다
    if (Files.notExists(path)) {
        call.respond(HttpStatusCode.BadRequest, "File not found")
    } else {
        call.response.header(
            name = HttpHeaders.ContentDisposition,
            value = ContentDisposition.Attachment.withParameter(
                key = ContentDisposition.Parameters.FileName,
                value = path.fileName.toString().encodeURLPath()
            ).toString()
        )
        call.respondFile(path.toFile())
    }
}
```

### 주의해야 할 곳

하나의 프로젝트에서 Kotlin/JS와 Kotlin/JVM을 함께 다룰 때는 `build.gradle.kts`에서 의존성을 다음처럼 나눠 지정할 수 있습니다.
```kotlin
kotlin {
    sourceSets {
        // Kotlin/JS 의존성
        val jsMain by getting {
            dependencies {
                implementation(compose.web.core)
                implementation(compose.runtime)
                // ...생략
            }
        }
        // Kotlin/JVM 의존성
        val jvmMain by getting {
            dependencies {
                implementation("io.ktor:ktor-server-core-jvm:$ktor_version")
                implementation("io.ktor:ktor-server-auth-jvm:$ktor_version")
                // ...생략
            }
        }
    }
}
```

다만 Compose는 `plugin`으로 적용해야 해서 프로젝트 전체에 영향을 줍니다. 이번 구성은 프론트엔드용 Compose를 먼저 빌드한 뒤, 서버가 그 결과물을 정적 파일로 제공하는 방식이었는데, 이 때문에 백엔드를 실행할 때도 Compose 런타임이 필요했습니다. 이 의존성을 추가하지 않으면 Ktor가 제대로 뜨지 않았습니다. Kotlin/JS 쪽에만 더 깔끔하게 한정하는 방법이 있을 수도 있겠지만, 우선은 JVM 의존성에 런타임을 명시적으로 추가해 해결했습니다.
```kotlin
val jvmMain by getting {
    dependencies {
        implementation("io.ktor:ktor-server-core-jvm:$ktor_version")
        implementation("io.ktor:ktor-server-auth-jvm:$ktor_version")
        // ...생략
        implementation(compose.runtime) // Compose 런타임
    }
}
```

## 기타

Kotlin/JS와 Kotlin/JVM을 한 프로젝트로 묶었을 때 가장 좋았던 점은 `common` 소스셋을 통해 코드를 공유할 수 있다는 점이었습니다. 예를 들어 JSON 응답용 `data class`를 `common`에 두면 프론트엔드와 백엔드에서 같은 타입을 그대로 사용할 수 있습니다. `Enum`이나 상수도 함께 공유할 수 있어서 구현이 꽤 편했습니다.
이번에는 쓰지 않았지만, Ktor Server에는 [Type-safe Routing](https://ktor.io/docs/type-safe-routing.html#resource_classes)도 있습니다. Ktor Client 쪽에도 [Type-safe Request](https://ktor.io/docs/type-safe-request.html#define_route)가 있어 프론트엔드와 백엔드 양쪽에서 같은 방식으로 활용할 수 있습니다. 다음에 Ktor로 비슷한 구조를 만들 일이 있으면 꼭 써 보고 싶은 기능입니다.
## 마지막으로

파일 업로드 쪽이 아직 기대한 만큼 정리되지 않아 앱이 완전히 마무리된 상태는 아니지만, 전체적으로는 꽤 재미있는 경험이었습니다. Kotlin으로 할 수 있는 일의 범위가 생각보다 넓다는 점을 다시 확인했고, 다음에도 비슷한 개인 프로젝트가 있으면 또 도전해 볼 생각입니다. 다만 아직은 성숙도가 아주 높다고 보긴 어려워서, 예상하지 못한 지점에서 문제가 생기거나 참고할 자료가 적다는 점은 분명한 한계였습니다. 당장 프로덕션에서 쓰기에는 조금 조심스러운 기술이라는 인상도 남았습니다.

앱 전체 코드는 GitHub에 공개해 두었으니 [여기](https://github.com/retheviper/FileTransporter)에서 확인할 수 있습니다.
