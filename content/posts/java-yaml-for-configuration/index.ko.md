---
title: "외부 설정 파일 읽는 클래스 만들기"
date: 2019-11-24
categories:
  - java
image: "../../images/java.webp"
tags:
  - yaml
  - java
translationKey: "posts/java-yaml-for-configuration"
---

애플리케이션을 만들 때 꼭 고민해야 하는 것 중 하나가 설정 파일입니다. 설정값이 코드 내부에만 있으면 디컴파일하지 않는 이상 바꿀 방법이 없습니다. 그래서 보통은 자주 바뀌는 설정을 애플리케이션 밖의 별도 파일로 분리합니다. 게임으로 치면 세이브 파일 같은 개념입니다.

외부 파일을 읽는 방법은 여러 가지가 있지만, 이번 글에서는 제가 실제로 쓰는 방식을 소개합니다. 설정 파일은 [YAML](https://ko.wikipedia.org/wiki/YAML)로 작성하고, 프로그램 시작 시 읽어서 싱글톤 클래스의 필드로 들고 있는 방식입니다. 한 번만 I/O를 하면 되고 어디서든 참조할 수 있다는 점이 장점입니다. 포맷으로 YAML을 고른 이유는 쓰기 쉽고 읽기 쉽기 때문이고, 제가 주로 쓰는 Spring에서도 잘 맞기 때문입니다. JSON도 가능하지만 항목이 많아질수록 읽기 어려워집니다.

준비할 것은 YAML을 읽는 라이브러리입니다. 여기서는 [SnakeYAML](https://bitbucket.org/asomov/snakeyaml/src/default)을 사용합니다. 파일을 읽어 객체로 바꾸기만 하면 되니 다른 라이브러리를 써도 상관없습니다.

예를 들어 아래와 같은 YAML이 있다고 하겠습니다.

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

Development와 Debugging 두 경우에 서로 다른 값을 쓰는 상황을 가정한 예입니다. 이를 다루는 코드는 다음과 같습니다.

```java
// 공용 설정 정보 클래스
public final class Settings {

    // 싱글톤 인스턴스
    private static final Settings UNIQUE_INSTANCE = new Settings();

    // 설정 파일에서 읽은 값을 임시로 담는 필드
    private static final Map<String, Map<String, Object>> SETTINGS_FROM_FILE = new HashMap<>();

    // 설정 파일 이름
    private static final String SETTINGS_FILENAME = "application-settings.yml";

    // Development 설정
    @Getter(lazy = true)
    private final Map<String, Object> developmentSettings = setDevelopmentSettings();

    // Debugging 설정
    @Getter(lazy = true)
    private final Map<String, Object> debuggingSettings = setDebuggingSettings();

    // 초기화 블록에서 파일을 먼저 읽는다
    static {
        ClassLoader classloader = UNIQUE_INSTANCE.getClass().getClassLoader();
        URL resource = classloader.getResource(SETTINGS_FILENAME);
        try (InputStreamReader reader = new InputStreamReader(resource.openStream())) {
            Yaml yaml = new Yaml();
            Map<String, Map<String, Object>> importedMap = autoCast(yaml.load(reader));
            for (Map.Entry<String, Map<String, Object>> entry : importedMap.entrySet()) {
                SETTINGS_FROM_FILE.put(entry.getKey(), entry.getValue());
            }
        } catch (IOException e) {
            // 예외 처리
        }
    }

    private Settings() {
    }

    // Lazy getter에서 필요할 때 값을 꺼낸다
    private Map<String, Object> setDevelopmentSettings() {
        return Collections.unmodifiableMap(SETTINGS_FROM_FILE.get("Development"));
    }

    // Lazy getter에서 필요할 때 값을 꺼낸다
    private Map<String, Object> setDebuggingSettings() {
        return Collections.unmodifiableMap(SETTINGS_FROM_FILE.get("Debugging"));
    }

    public static Settings getInstance() {
        return UNIQUE_INSTANCE;
    }

    @SuppressWarnings("unchecked")
    private static <T> T autoCast(final Object object) {
        return (T) object;
    }
}
```

여기서는 Lazy Getter를 사용해 `developmentSettings`와 `debuggingSettings`가 필요할 때 만들어지도록 했습니다. 이유는 초기화 블록에서 파일을 먼저 읽고, 그 뒤에 개별 필드에 값을 넣고 싶었기 때문입니다. 동시에 필드는 `private final`로 유지하고 싶었습니다. Lazy Getter를 쓰면 `private final`을 유지하면서도 값은 나중에 채울 수 있고, 한 번 만들어진 뒤에는 캐시처럼 유지되니 편리합니다. 반대로 생성자나 필드 초기화에서 바로 처리하려고 하면, 파일을 읽기 전에 초기화가 끝나 버릴 수 있으니 주의해야 합니다.

## 마지막으로

이 방식을 쓰면 설정 클래스는 싱글톤으로서 애플리케이션 시작 시 생성되고, 그때 파일을 읽어서 Map으로 메모리에 올린 뒤 어디서든 같은 값을 참조할 수 있습니다. YAML 구조가 더 깊어지거나 항목이 늘어나더라도, Map과 필드만 조정하면 비교적 쉽게 따라갈 수 있습니다.

단순히 동작하는 것만이 아니라, 설정을 어디서 읽고 어떻게 공유할지 구조를 미리 정해 두는 것도 중요합니다. 자주 바뀌는 데이터를 다룰 때는 이런 방식이 꽤 실용적입니다.
