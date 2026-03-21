---
title: "SwiftUI를 써 본 첫인상 2"
date: 2022-08-20
categories: 
  - swift
image: "../../images/swift.webp"
tags:
  - swift
  - swiftui
  - gui
  - kotlin
  - compose
translationKey: "posts/swift-ui-first-impression-2"
---

지난 글에 이어 이번에는 SwiftUI를 만지면서 느낀 점을 정리해 보겠습니다. 저처럼 지금까지 백엔드 경험만 있던 엔지니어가 GUI를 만들면, 화면 레이아웃이나 색감, 화면 사이의 연결 방식처럼 익숙하지 않은 개념 때문에 꽤 헷갈릴 수 있습니다. 그중에서도 특히 다루기 까다로운 기능이 따로 있지 않을까 하는 생각도 들었습니다.

저는 프로덕션 코드를 써 본 적은 없지만, 예전부터 [React Native](https://reactnative.dev/), [Flutter](https://flutter.dev/), [Jetpack Compose](https://developer.android.com/jetpack/compose)를 조금씩 만져 본 적이 있어서 SwiftUI의 "화면을 구성하는 방식"도 어느 정도 비슷하겠다고 생각했습니다. 그런데 막상 해 보니, 백엔드 쪽에서는 잘 만나지 못했던 개념이 실제로 있더군요.

이번 글에서는 그 SwiftUI 중에서도 특히 눈에 들어왔던 기능과 개념을 중심으로 이야기해 보겠습니다.
## SwiftUI

먼저 SwiftUI 자체를 간단히 보면, 이 프레임워크는 선언형(Declarative) UI에 속합니다. [React](https://ko.reactjs.org/)나 [Vue](https://vuejs.org/) 같은 프런트엔드 프레임워크와 비슷한 계열이라고 보면 됩니다. 화면을 이루는 요소들, 즉 Widget이나 Component, Material처럼 프레임워크마다 다르게 부르는 단위들을 객체로 선언하고, 그 조합으로 화면을 완성하는 방식입니다. 이런 선언형 UI는 SwiftUI만의 특징이 아니라 React Native, Flutter, Jetpack Compose 같은 모바일 프레임워크에도 널리 쓰이고 있습니다.

구현 방식도 프레임워크마다 조금씩 다르지만, SwiftUI에서는 개별 요소를 View라고 부르고, [View](https://developer.apple.com/documentation/swiftui/view) protocol을 `struct`로 구현해 나갑니다. 그래서 목록 화면이라면 한 행을 보여 주는 View, 그 행들을 리스트로 묶는 View, 목록 위나 아래에 메뉴를 붙이는 View가 각각 하나의 `struct`로 나뉘어 정의되는 식입니다.

이런 방식은 프레임워크가 정한 패러다임을 따라가면 되기 때문에, 저처럼 배경이 전혀 다른 엔지니어라도 구현 자체는 크게 어렵지 않았습니다.

다만 실제 앱을 만들려면 화면만 그려서는 끝이 아닙니다. 화면 요소를 만들었다면, 그걸 앱의 로직과 연결해야 하고, 그 과정에서 백엔드 쪽에서는 잘 보지 못한 개념이 등장합니다. 바로 "상태"입니다.
## 상태

백엔드 애플리케이션은 보통 요청이 오고, 그에 대한 응답이 돌아가는 흐름이 분명합니다. 이런 처리에는 "도중에 상태가 바뀐다"는 개념이 거의 없습니다. 데이터도 대개 저장소에 영속화되거나, 요청이 끝날 때까지만 잠깐 존재하는 경우가 많습니다.

하지만 화면의 세계는 다릅니다. 모바일 앱은 슬라이더, 버튼, 텍스트 박스처럼 상태가 계속 바뀌는 요소로 구성되는 경우가 많습니다. 예를 들어 파일 다운로드 진행 상황을 보여 주는 프로그레스 바가 있다면, 단순히 진행률만 표시하면 되니 한 스레드에서 처리하는 정도로 충분할 수 있습니다.

그런데 다운로드에 "일시 정지" 기능이 붙는 순간 이야기가 달라집니다. 진행 중에 버튼을 누르면 멈췄다가, 다시 누르면 이어서 진행되는 식이죠. 이처럼 "멈춘 상태"와 "재개한 상태"를 어딘가에 기억해 둘 필요가 있습니다. 즉, 사용자의 입력을 바로 처리하지 않더라도 나중에 다시 이어서 쓸 수 있게 저장해 두는 장치가 필요하다는 뜻입니다.

SwiftUI에도 물론 상태를 다루는 방법이 있습니다. 다만 상태가 View 하나에만 필요한지, 여러 View가 함께 써야 하는지, 아니면 앱 전체에서 공유해야 하는지에 따라 선택이 달라집니다. View 하나에 필요한 값을 앱 전체에서 관리할 이유는 없으니, 범위에 맞게 나누는 편이 훨씬 깔끔합니다. 여기서는 그 스코프별로 어떤 도구를 쓰는지 정리해 보겠습니다.
### 개별 View 상태

가장 작은 단위인 View부터 보겠습니다. 앞에서 말한 목록 화면이 좋은 예입니다. Apple 튜토리얼에서는 다음과 같은 화면을 만듭니다.
![List View](view_list.webp)
이 화면에서는 오른쪽 위 버튼을 눌러 목록을 필터링할 수 있습니다. 일부 행에 붙어 있는 하트는 "즐겨찾기"를 뜻하고, 즐겨찾기만 볼지 혹은 아이템 카테고리로 걸러 볼지를 선택할 수 있습니다.

이때 "필터할 카테고리"나 "즐겨찾기만 표시할지 여부"는 이 View 안에서만 의미가 있는 값입니다. 다른 화면에서는 알 필요가 없는 데이터죠. 이런 경우에 쓰는 것이 [@State](https://developer.apple.com/documentation/swiftui/state)입니다. 코드는 다음과 같습니다. (일부 생략)
```swift
// 목록 화면
struct LandmarkList: View {
    // 즐겨찾기만 표시할지 여부
    @State private var showFavoritesOnly = false
    // 필터할 카테고리
    @State private var filter = FilterCategory.all
    
    // 카테고리 종류
    enum FilterCategory: String, CaseIterable, Identifiable {
        case all = "ALL"
        case lakes = "Lakes"
        case rivers = "Rivers"
        case mountains = "Mountains"
        
        var id: FilterCategory { self }
    }
    
    // 아이템에 필터를 적용
    var filteredLandmarks: [Landmark] {
        modelData.landmarks.filter { landmark in
            (!showFavoritesOnly || landmark.isFavorite)
                && (filter == .all || filter.rawValue == landmark.category.rawValue )
        }
    }

    var body: some View {
        NavigationView {
            // 아이템 표시 영역
            List(selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    NavigationLink {
                        LandmarkDetail(landmark: landmark)
                    } label: {
                        LandmarkRow(landmark: landmark)
                    }
                }
            }
            .toolbar {
                ToolbarItem {
                    // 툴바에 필터 버튼 추가
                    Menu {
                        // 선택 가능한 카테고리
                        Picker("Category", selection: $filter) {
                            ForEach(FilterCategory.allCases) { category in
                                Text(category.rawValue).tag(category)
                            }
                        }
                        .pickerStyle(.inline)
                        
                        // 즐겨찾기만 표시할지 토글
                        Toggle(isOn: $showFavoritesOnly) {
                            Label("Favorites only", systemImage: "heart.fill")
                        }
                    } label: {
                        Label("Filter", systemImage: "slider.horizontal.3")
                    }
                }
            }
        }
    }
}
```

즉, 하나의 View 안에서만 쓰는 상태는 `@State`로 관리합니다.
### 부모 - 자식 관계의 View가 공유하는 상태

이제 한 View 안의 상태는 다룰 수 있게 됐지만, 다음으로 궁금한 건 여러 View, 특히 부모-자식 관계에 있는 View끼리 상태를 어떻게 공유하느냐입니다. 예를 들어 아까의 목록 화면에서는 각 행도 하나의 View입니다.

사실 부모 View에서 자식 View로 상태를 넘기는 예는 이미 앞 코드에 들어 있습니다. "즐겨찾기만 표시할지" 토글이 그 예인데, 여기서 `isOn`에 부모 상태를 넘기고 있죠. 다만 `@State`로 선언한 `Boolean`과 연결하려면 `isOn`에 값을 넘길 때 `$`를 붙여야 합니다. `$`를 붙이면 `Boolean`이 아니라 [`Binding<Boolean>`](https://developer.apple.com/documentation/swiftui/binding) 형태로 전달됩니다. 이렇게 래퍼를 넘겨 주면 [Toggle](https://developer.apple.com/documentation/swiftui/toggle/) 안에서도 부모 상태를 바꿀 수 있습니다. `Toggle`은 목록 화면과는 별도의 View지만, 누를 때마다 부모의 상태인 `showFavoritesOnly`가 바뀌는 구조입니다.

나중에 따로 글을 쓰고 싶지만, Jetpack Compose에도 비슷한 방식으로 상태를 나눠 다룰 수 있는 방법이 있습니다. 예를 들어 `@State`처럼 간단한 상태를 다루려면 다음처럼 쓸 수 있습니다.
```kotlin
// true/false 상태
var toggle: Boolean by remember { mutableStateOf(false) }

if toggle {
    println("On!")
    toggle = false
} else {
    println("Off!")
    toggle = true
}
```

이 코드는 [Delegation](https://kotlinlang.org/docs/delegation.html)을 이용한 것입니다. `mutableStateOf<T>`가 돌려주는 값은 [`MutableState<T>`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState)이지만, `by`를 사용하면 마치 실제 `Boolean`을 다루는 것처럼 보이게 됩니다.

또 `MutableState<T>`를 분해해서 상태 값과 변경 함수를 따로 받을 수도 있습니다. 이 경우도 앞의 `Binding`과 비슷한 역할을 한다고 볼 수 있습니다.
```kotlin
// 텍스트 상태와 값 변경
val (content: String, onValueChange: (String) -> Unit) = remember { mutableStateOf("") }

// 텍스트 필드에 상태를 표시하고, 변경되면 상태도 갱신한다
TextField(value = content, onValueChange = onValueChange)
```

이처럼 다른 요소가 상태를 바꾸게 하려면, 결국 그 상태를 감싼 객체를 함께 다루게 됩니다. SwiftUI든 Jetpack Compose든 이 점은 비슷합니다. 다만 Swift는 `$`로 래퍼에 접근하고, Kotlin은 상태 객체를 어떤 형태로 선언할지부터 생각해야 한다는 점이 서로 다른 재미있는 포인트입니다.
### 앱 전체에서 공유하는 상태

이제는 더 큰 단위의 상태를 보겠습니다. 앱 화면은 여러 종류가 있고, 그중에는 부모-자식 관계가 아닌 것도 많습니다. Apple 튜토리얼 앱을 예로 들면, 아래처럼 탭이 나뉘어 있는 구조가 대표적입니다.
![Menu View](view_menu.webp)
아래의 "Featured"나 "List"를 누르면 표시 화면이 바뀌지만, 이 둘은 부모-자식 관계라고 보기는 어렵습니다. 코드는 다음과 같습니다.
```swift
struct ContentView: View {
    @State private var selection: Tab = .featured
    
    enum Tab {
        case featured
        case list
    }
    
    var body: some View {
        // 탭 메뉴
        TabView(selection: $selection) {
            // Featured 화면
            CategoryHome()
                .tabItem {
                    Label("Featured", systemImage: "star")
                }
                .tag(Tab.featured)
            
            // LandmarkList 화면
            LandmarkList()
                .tabItem {
                    Label("List", systemImage: "list.bullet")
                }
                .tag(Tab.list)
        }
    }
}
```

상황에 따라서는 이렇게 서로 대등한 화면끼리 상태를 공유해야 할 때도 있습니다. 예를 들어 쇼핑 앱이라면, 다른 화면으로 넘어가도 장바구니 내용은 계속 유지되어야 합니다. 이처럼 지금 보고 있는 화면과 무관하게 앱 전체에서 공유해야 하는 상태가 있습니다.

그렇다면 어떻게 해야 할까요? 시작점이 되는 View에 `@State`를 두는 방법도 있습니다. 하지만 관리할 항목이 많아질수록 코드가 금방 복잡해집니다. 시작 View에는 여러 개의 `@State`가 필요하고, 그 값을 각 화면에 계속 전달해야 하니까요. 결국 상태는 하나의 객체에 모아 두고 재사용하는 편이 훨씬 낫습니다.

Apple 튜토리얼도 같은 방향을 제시합니다. 화면 사이에서 공유할 데이터로 Landmark 정보와 사용자 프로필을 한데 묶어 제공하는 방식이 있습니다. 먼저 상태 객체는 다음과 같습니다. (일부 생략)
```swift
final class ModelData: ObservableObject {
    @Published var landmarks: [Landmark] = load("landmarkData.json")
    @Published var profile = Profile.default
}
```

여기서 Landmark 정보는 JSON에서 읽어오고, 프로필은 enum으로 관리합니다. 그리고 [ObservableObject](https://developer.apple.com/documentation/combine/observableobject) protocol을 쓰고 있는 것도 보입니다. `ObservableObject`를 사용하면 `ModelData`는 변경 가능한 상태를 보관하고, 값이 바뀌었을 때 그 상태를 참조하는 화면에 알려 주는 `Publisher`처럼 동작합니다. 각 상태 프로퍼티에는 [@Published](https://developer.apple.com/documentation/combine/published)를 붙여 상태임을 표시합니다.

이제 이 상태 객체를 실제 앱에서 어떻게 쓰는지 보겠습니다. 튜토리얼에서는 앱의 메인 `struct`에 다음처럼 연결합니다. (일부 생략)
```swift
@main
struct LandmarksApp: App {
    // ModelData를 상태 객체로 선언
    @StateObject private var modelData = ModelData()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(modelData) // 메인 화면에 상태 객체를 넘긴다
        }
    }
}
```

보면 알 수 있듯이, 앱에서 만든 상태 객체는 [@StateObject](https://developer.apple.com/documentation/swiftui/stateobject)로 선언해 메인 화면에 넘깁니다. 또 `View`로 만든 화면이라면 어디든 `environmentObject()`로 상태 객체를 전달할 수 있어서, 일부 화면만 프리뷰로 띄울 때도 같은 방식으로 동작을 확인할 수 있습니다. 예를 들어 Landmark 목록 화면의 프리뷰는 다음처럼 작성합니다.
```swift
struct LandmarkList_Previews: PreviewProvider {
    static var previews: some View {
        LandmarkList()
            .environmentObject(ModelData()) // 상태 객체를 넘긴다
    }
}
```

상태 객체를 화면에 넘겨 주면, 그 안에서는 그냥 꺼내 쓰기만 하면 됩니다. [@EnvironmentObject](https://developer.apple.com/documentation/swiftui/environmentobject)를 붙여 상태 객체를 선언하면 자동으로 DI되어 해당 화면에서 사용할 수 있습니다. 다시 목록 화면 예시를 보면, `landmarks` 데이터를 이용해 목록을 만드는 걸 볼 수 있습니다. (일부 생략)
```swift
struct LandmarkList: View {
    // 상태 객체
    @EnvironmentObject var modelData: ModelData

    // 상태 객체에서 데이터를 꺼내 필터를 적용
    var filteredLandmarks: [Landmark] {
        modelData.landmarks.filter { landmark in
            (!showFavoritesOnly || landmark.isFavorite)
                && (filter == .all || filter.rawValue == landmark.category.rawValue )
        }
    }
```

참고로 Jetpack Compose는 이런 방식과는 조금 다르게, 객체 자체를 `remember`해서 쓰는 편입니다. 예를 들면 다음 같은 형태가 됩니다. 애초에 요소를 만드는 접근 방식이 다르기 때문일 수도 있습니다.
```kotlin
// 앱 전체에서 공유할 상태
class ApplicationState(
    val environment: MutableState<Environment>,
    val hash: MutableState<String>
)

// 상태 초기화
@Composable
fun rememberApplicationState(
    environment: MutableState<Environment> = mutableStateOf(Environment.PRODUCTION),
    hash: MutableState<String> = mutableStateOf("")
): ApplicationState {
    return remember(environment, hash) {
        ApplicationState(environment, hash)
    }
}

// 상태 정의
val appState = rememberApplicationState()
```

### 지속 가능한 상태

지금까지 본 상태는 앱이 실행 중일 때만 유효했습니다. 그것만으로 충분한 경우도 있지만, 어떤 경우에는 상태를 저장해 두고 싶을 때도 있습니다. 예를 들어 학습용 앱이라면 어디까지 진행했는지 기록해 두고 싶겠죠. 이런 데이터는 앱을 다시 열어도 그대로 남아 있기를 기대합니다.

물론 이런 경우에는 DB를 쓰거나 서버에 저장하는 방법도 있습니다. 하지만 그런 방식은 더 이상 "화면 상태"라고 부르기엔 애매합니다. 화면을 업데이트하면서 동시에 보존하고 싶은 값이라면, 매번 DB를 갱신하고 읽어 오는 방식은 너무 무거울 수 있습니다. 이럴 때 쓰는 것이 [@AppStorage](https://developer.apple.com/documentation/swiftui/appstorage)입니다. 화면 갱신과 영속화를 함께 처리할 수 있습니다.

튜토리얼에 따르면 기본값으로 저장된 일부 데이터도 이 어노테이션으로 읽어올 수 있습니다. 목록 화면의 "즐겨찾기" 버튼이 그 예인데, 버튼 아이콘을 `@AppStorage`에서 참조하고 있습니다. 코드는 다음과 같습니다. (일부 생략)
```swift
struct FavoriteButton: View {
    // 버튼은 하트 아이콘으로 표시한다
    @AppStorage("Favorite.iconType")
    private var iconType: IconType = .heart
}
```

## 마지막으로

지금까지 접해 온 분야와는 완전히 다른 개념과 접근법이 많아서 정말 흥미로웠습니다. 동시에 "이렇게 이해해도 맞나?" 하는 의문도 자꾸 생겼습니다. 아마 익숙하지 않은 분야를 처음 만질 때는 늘 비슷한 느낌일 것입니다.

그래도 이번 경험으로 앱을 만들 때 데이터를 어떻게 다뤄야 하는지에 대한 큰 그림은 조금 잡혔습니다. 아직 화면 구성이나 UX처럼 더 어려운 부분은 남아 있지만, 적어도 "움직이는" 무언가를 만들 수 있겠다는 감각은 얻었습니다. 저처럼 백엔드 위주로 일해 온 사람에게도 비슷한 출발점이 될 수 있으면 좋겠습니다.
