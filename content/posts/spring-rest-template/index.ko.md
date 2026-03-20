---
title: "REST API에서 다른 REST API로 파일 보내기"
date: 2020-02-10
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - rest api
  - java
  - spring
translationKey: "posts/spring-rest-template"
---

웹 애플리케이션을 개발하다 보면 하나의 `REST API`만으로 모든 기능을 다 처리할 필요는 없습니다. 이미 필요한 기능이 구현된 다른 서버(API)가 있을 수도 있고, 이런 경우에는 그 API를 직접 호출하는 것만으로도 목적을 쉽게 달성할 수 있습니다. 실제로 다른 REST API 서버와 통신해야 하는 일을 조사하다 보니, Spring에는 이런 상황에 쓸 수 있는 클래스가 이미 준비돼 있었습니다. 바로 `RestTemplate`입니다.

`RestTemplate`을 쓰면 GET, POST, DELETE 같은 HTTP 메서드의 API를 쉽게 호출할 수 있습니다. 요청과 응답도 상황에 맞게 커스터마이징할 수 있고, 커스텀 헤더를 만들거나 큰 파일을 전송하는 요청도 만들 수 있습니다. 이번 글에서는 파일을 업로드하면 그 파일에 어떤 처리를 한 뒤 다시 돌려주는 API가 이미 존재한다고 가정하고, 그 API를 호출하는 부품을 만드는 방법을 소개하겠습니다.

## 서버 측 예시

이미 파일을 처리하는 서버가 있다고 가정해 봅시다. 먼저 호출하려는 API에서 어떤 요청과 응답 형태를 쓰는지 알아야 합니다. 파일이 업로드되면 헤더에서 파일 정보를 읽고, 본문에서 파일 데이터를 읽는 메서드가 있다고 해 보겠습니다. 헤더 정보에 문제가 없으면 임시 저장소에 파일을 쓰고, 그 뒤에 파일 처리를 합니다. 그리고 처리된 파일을 응답으로 돌려주는 서버입니다.

```java
@PostMapping("/upload")
public ResponseEntity<StreamingResponseBody> fileupload(HttpServletRequest request) {
    // 요청 헤더에서 파일 크기를 가져온다
    long fileLength = request.getContentLengthLong();
    // 파일 크기가 0이면 IOException
    if (fileLength == 0) {
       throw new IOException("data size is 0");
    }

    // 파일을 임시 파일로 저장한다
    String fileName = request.getHeader("Content-File-Name");
    Path tempFile = Files.createTempFile("process_target_", fileName);
    try (InputStream is = request.getInputStream()) {
        Files.copy(is, tempFile);
    } catch (IOException e) {
        e.printStackTrace();
    }

    // ...파일을 가지고 어떤 처리 수행

    // 응답용 헤더를 만든다
    HttpHeaders headers = new Headers();
    // 처리된 파일을 써 줄 본문을 만든다
    StreamingResponseBody body = new StreamingResponseBody() {
        @Override
        public void writeTo(OutputStream outputStream) throws IOException {
            byte[] bytes = Files.readAllBytes(path);
            outputStream.write(bytes);
        }
    };
    // 헤더, 본문, HttpStatus를 담아 응답 반환
    return new ResponseEntity<StreamingResponseBody>(body, headers, HttpStatus.OK);
}
```

이런 서버에 요청을 보내려면, 클라이언트는 요청 헤더에 파일 정보를 넣고 본문에는 파일 데이터를 담아 보내야 합니다. 그리고 처리 결과로 돌아오는 응답에도 파일 데이터가 있으니 이를 받아 처리하는 로직도 필요합니다. 그래서 요청과 응답을 각각 커스텀해서 `RestTemplate`에 넣어 보겠습니다.

## `RestTemplate` 사용하기

이번에는 `RestTemplate`의 `execute()` 메서드를 사용하겠습니다. 이 메서드의 인자는 다음과 같습니다.

1. API URL(String 또는 URI)
1. HTTP 메서드(ENUM)
1. 요청(RequestCallback 구현체)
1. 응답(ResponseExtractor 구현체)

기존의 get/post/delete 메서드와 달리 요청과 응답을 둘 다 직접 지정하는 이유는, 위에서 본 것처럼 파일 전송이 요청과 응답 양쪽에서 필요하기 때문입니다. `execute()`는 URI 변수를 받는 형태도 있지만, 여기서는 고정 URL만 쓰므로 생략하겠습니다. 이제 요청과 응답 인터페이스를 어떻게 구현하는지 보겠습니다.

## 요청

요청에 사용할 `RequestCallback` 구현체를 만듭니다. 생성자 인자로 파일을 넘기면 헤더와 본문을 만들도록 해 보겠습니다. `RequestCallback`을 구현하면 `doWithRequest()` 메서드를 오버라이드하게 됩니다. 이 메서드의 인자인 `ClientHttpRequest`에 헤더와 본문을 설정하면 파일 업로드 요청이 완성됩니다.

```java
public class FileTransferRequestCallback implements RequestCallback {

    // 업로드할 파일
    private Path path;
    
    // 헤더에 파일 정보를 넣기 위한 생성자
    public PdfConvertRequestCallback(File file) {
        this.path = file.toPath();
    }

    @Override
    public void doWithRequest(ClientHttpRequest request) throws IOException {
        // 파일에서 헤더를 만든다
        request.getHeaders().set("Content-Length", Files.size(path));
        request.getHeaders().set("Content-File-Name", path.getFileName().toString());
        // 본문에 파일을 쓴다
        try (InputStream is = Files.newInputStream(file); OutputStream os = request.getBody()) {
            is.transferTo(os);
        }
    }
}
```

헤더에는 서버가 요구하는 파일 크기와 파일명을 넣습니다. 그리고 업로드할 파일을 `InputStream`으로 읽어 `OutputStream`인 본문에 씁니다. 이렇게 하면 요청 쪽 파일 업로드 설정은 끝납니다.

## 응답

응답에서는 `ResponseExtractor`를 구현합니다. 이 경우 `extractData()` 메서드를 오버라이드하게 됩니다. 이 메서드의 인자인 `ClientHttpResponse`에서 HTTP 상태 코드, 헤더, 본문을 가져올 수 있습니다. 이 값을 바탕으로 `ResponseEntity`를 만들어 반환하면 `RestTemplate`의 결과로 `ResponseEntity`를 받을 수 있습니다.

`ResponseEntity`를 반환하려면 본문 타입을 지정해야 합니다. 여기서는 파일이 본문이므로 `InputStream`을 쓰겠습니다. 다만 `ClientHttpResponse`의 본문을 그대로 쓰면 스트림이 닫히므로, 먼저 복사해 두어야 합니다. 저는 `byte[]`로 바꾼 뒤 `ByteArrayInputStream`을 만드는 방식으로 처리했습니다.

```java
public class FileTransferResponseExtractor implements ResponseExtractor<ResponseEntity<InputStream>> {

    @Override
    public ResponseEntity<InputStream> extractData(ClientHttpResponse response) throws IOException {
        // 응답 본문을 복사한다
        byte[] bytes = response.getBody().readAllBytes();
        // 상태 코드, 헤더, 본문을 담아 ResponseEntity 반환
        return ResponseEntity.status(response.getStatusCode()).headers(response.getHeaders()).body(new ByteArrayInputStream(bytes));
    }
}
```

이제 응답 파일을 받을 준비가 끝났습니다.

## REST API 호출

`RestTemplate`으로 API를 호출하는 과정은 생각보다 단순합니다. 먼저 URL과 업로드할 파일 인스턴스를 준비하고, 앞에서 만든 `RequestCallback`과 `ResponseExtractor` 인스턴스도 만듭니다. `ResponseExtractor`는 상태를 가지지 않으니 Bean으로 등록해도 괜찮습니다.

`execute()`에 URL, HTTP 메서드, `RequestCallback`, `ResponseExtractor`를 넘기면 결과를 `ResponseEntity`로 받을 수 있습니다. 여기서 상태 코드, 헤더, 본문을 꺼내 쓰면 됩니다. 이렇게 하면 업로드한 파일을 처리한 뒤 결과 파일까지 바로 받아올 수 있습니다.

```java
// RestTemplate에 넘길 인자 준비
String url = "http://api/v1/file/upload";
File uploadFile = new File("path/to/upload_file.txt");
FileTransferRequestCallback requestCallback = new FileTransferRequestCallback(uploadFile);
FileTransferResponseExtractor responseExtractor = new FileTransferResponseExtractor();

// RestTemplate으로 API 호출하고 결과를 받는다
ResponseEntity<InputStream> responseEntity = new RestTemplate().execute(url, HttpMethod.POST, requestCallback, responseExtractor);

// ResponseEntity에서 상태 코드 확인
if (responseEntity.getStatusCode() != HttpStatus.OK) {
    throw new IOException();
}
// ResponseEntity에서 헤더를 얻는다
HttpHeaders headers = responseEntity.getHeaders();

// ResponseEntity에서 본문을 읽는다
try (InputStream is = responseEntity.getBody()) {
    File downloadedFile = new File("path/to/downloaded_file.txt");
    Files.copy(is, downloadedFile);
}
```

흐름 자체는 꽤 단순합니다. 이렇게 하면 다른 REST API와 파일을 주고받을 수 있습니다.

## 마지막으로

클래스와 메서드를 기능별로 나누듯이, REST API도 경우에 따라 기능을 분리해 둘 수 있습니다. 이번 예제가 바로 그런 경우였습니다. 물론 네트워크를 경유하므로 같은 서버 내부에서 처리하는 것보다 실패 가능성은 커질 수 있지만, 재사용성 측면에서는 꽤 좋은 방법입니다.

실제로는 같은 서버 안에서 처리하는 편이 더 단순한 경우도 많겠지만, 외부 API를 호출해야 하는 상황은 생각보다 자주 생깁니다. 그런 의미에서 `RestTemplate`로 요청과 응답을 커스터마이징해 두는 방식은 한 번쯤 익혀 둘 만한 패턴이라고 생각합니다.
