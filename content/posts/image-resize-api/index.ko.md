---
title: "동적 이미지 리사이즈 API 만들기"
date: 2024-03-30
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - ktor
  - kotlin
  - scrimage
  - rust
translationKey: "posts/image-resize-api"
---

이번에는 동적 이미지 리사이즈 API를 만든 이야기를 정리해 보려 합니다. 여기서 말하는 동적 이미지 리사이즈 API는 원본 이미지의 URL과 원하는 크기를 넘기면, 그 크기로 조정된 이미지를 돌려주는 API입니다.

## 목적

왜 이런 API가 필요한지부터 먼저 설명해야 합니다. 지금까지는 이미지 전달을 할 때 사용자가 이미지를 업로드하면, 미리 썸네일도 함께 만들어 올리는 방식으로 처리했습니다.

하지만 이 방식으로는 모든 화면비와 해상도에 맞는 썸네일을 준비하기 어렵습니다. 그래서 프런트엔드에서 화면에 맞는 크기를 요청하고, API가 그에 맞게 리사이즈한 이미지를 돌려주는 구조로 바꾸고 싶었습니다.

## 설계·기술 선정

API는 마이크로서비스답게 최대한 단순하게 만들기로 했습니다. GET 엔드포인트 하나만 두고, 쿼리 파라미터로 원본 이미지 URL과 원하는 크기를 넘기면 리사이즈된 이미지를 반환하는 방식입니다. 프런트엔드에서는 바로 `img` 태그에 넣을 수 있어야 하므로, 응답은 이미지 바이트 그대로 보내고 헤더에 `Content-Type`을 지정합니다. 리사이즈뿐 아니라 이미지 포맷 변환도 지원하려고 했습니다.

배포는 [Cloud Run](https://cloud.google.com/run?hl=ko)에서 하기로 했습니다. 기존 API들도 같은 방식이어서 운영 모델을 맞추기 좋았고, 로컬에서는 `docker compose`로 편하게 개발할 수 있기 때문입니다. 콘텐츠 저장소로는 [Cloud Storage](https://cloud.google.com/storage?hl=ko)를 쓰고 있었으니 연동도 자연스러웠습니다. 최종적으로 처리된 이미지는 Cloud CDN에 저장해서, 두 번째 이후 요청부터는 캐시를 돌려주도록 구성했습니다.

회사에서 이미 웹 프레임워크로 Ktor를 쓰고 있었기 때문에 이 API도 Ktor로 맞췄습니다. 예전에 Node.js와 Rust로도 시도해 본 적이 있는데, 전자는 성능이 아쉬웠고 후자는 유지보수 부담이 커서 채택하지 않았습니다. 당시 만들던 Rust 버전에 가까운 샘플 코드는 GitHub에 공개해 두었고, [여기](https://github.com/retheviper/resize-api)에서 볼 수 있습니다.

이미지 변환과 리사이즈에는 [Scrimage](https://github.com/sksamuel/scrimage)를 선택했습니다. Java의 [ImageIO](https://docs.oracle.com/javase/ko/8/docs/api/javax/imageio/ImageIO.html)도 검토했지만, 리사이즈 대상과 결과 포맷으로 WebP를 다뤄야 했는데 이 부분을 지원하지 않는 경우가 많았습니다.

또 다른 방법으로는 [GraphicsMagick](http://www.graphicsmagick.org/)처럼 외부 도구를 쓰는 방식도 있습니다. 다만 이 경우 Java에서 받은 데이터를 한 번 파일로 저장한 뒤 다시 커맨드라인으로 실행해야 하므로 I/O 비용이 커집니다. 그래서 이번에는 Scrimage를 선택했습니다.

## 크기 조정

이제 실제 API를 구현해 보겠습니다. Ktor는 익숙한 편이라, 이번에는 Ktor 자체보다 Scrimage를 이용한 이미지 리사이즈 처리에 초점을 맞추겠습니다.

전체 흐름은 대략 아래와 같습니다.

1. 쿼리 파라미터에서 URL과 리사이즈 후 크기를 읽는다
2. 이미지를 가져온다
3. 이미지를 리사이즈한다
4. 이미지 포맷을 변환한다

Scrimage로 실제 처리하는 부분은 3, 4번입니다. 다만 리사이즈 전에 먼저 이미지 포맷과 크기를 확인해야 처리 효율을 높일 수 있습니다.

이번에는 PNG, JPEG, WEBP, GIF 네 가지 포맷을 지원하고, 리사이즈 후에는 WEBP로 변환하는 방식으로 처리합니다. 원본이 이미 WEBP라면 굳이 다시 바꿀 필요가 없고, 애초에 리사이즈가 필요 없는 경우도 있습니다. 그래서 먼저 포맷을 판별한 뒤, 필요할 때만 리사이즈를 수행하도록 했습니다.

### 이미지 형식 판별

URL이나 로컬 스토리지에서 이미지를 가져오면, 먼저 그 이미지의 포맷을 알아야 합니다. Scrimage는 이를 위해 [FormatDetector](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/format/FormatDetector.java)를 제공합니다.

사용법은 간단합니다. 읽어 온 이미지를 `ByteArray`로 넘기기만 하면 됩니다. API에서는 예상하지 못한 포맷이 들어오면 에러를 반환하고, 여기서 사용하는 [Format](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/format/Format.java)는 PNG, GIF, JPEG, WEBP입니다.

```kotlin
fun detectImageFormat(data: ByteArray): Format {
    return FormatDetector.detect(data).orElseThrow { IllegalArgumentException("Unsupported format") }
}
```

### PNG, JPEG 처리

가장 단순한 PNG와 JPEG부터 보겠습니다. 이 포맷들은 Scrimage의 [ImmutableImage](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/ImmutableImage.java)로 다룰 수 있습니다.

이미지 데이터를 `ImmutableImage`로 바꾸려면 [ImageReader](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/ImageReader.java) 인터페이스를 구현한 클래스를 사용합니다. 공식 문서에는 `ImmutableImage.loader()` 하나로 형식과 무관하게 읽을 수 있다고 되어 있지만, 실제 API를 빌드해 보면 AWT 관련 에러가 나므로 포맷에 맞는 리더를 직접 선택하는 편이 안전합니다.

```kotlin
fun asImmutableImage(rawData: ByteArray): ImmutableImage {
    return ImageIOReader().read(rawData)
}
```

`ImmutableImage`로 변환한 뒤에는 리사이즈를 수행합니다. Scrimage에는 리사이즈용 메서드가 마련되어 있으므로 그것을 그대로 쓰면 됩니다.

여기서 주의할 점은 `resize()`와 `resizeTo()`의 의미가 다르다는 것입니다. `resize()`는 비율 기준이고, `resizeTo()`는 지정한 크기로 맞춥니다. 둘 다 원본 종횡비를 보장하지 않으므로, 비율을 유지하고 싶다면 `scaleTo()`나 `scaleToWidth()`를 써야 합니다.

이번에는 width만 지정해서 종횡비를 유지하는 방식으로 만들었으므로 `scaleToWidth()`를 사용합니다.

```kotlin
fun resizeImmutableImage(image: ImmutableImage, width: Int): ImmutableImage {
    return image.scaleToWidth(width)
}
```

마지막으로 리사이즈 결과를 `ByteArray`로 내보내는 메서드를 둡니다. 어떤 포맷으로 변환할지에 따라 [ImageWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/ImageWriter.java) 구현체를 골라야 합니다. 이번에는 WEBP로 내보내려 하므로 [WebpWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/WebpWriter.java)를 사용합니다.

```kotlin
fun encodeImage(image: ImmutableImage): ByteArray {
    return image.bytes(WebpWriter.DEFAULT)
}
```

마지막으로 라우터에서는 `Content-Type`을 지정해 `ByteArray`를 응답합니다.

```kotlin
call.respondBytes(
    bytes = resizedImage,
    contentType = ContentType("image", "webp")
)
```

### WEBP의 경우

WEBP는 위에서 본 방식대로 읽을 때 `WebpImageReader`를 써야 합니다. 이후의 리사이즈 과정은 PNG, JPEG와 같습니다.

[scrimage-webp](https://github.com/sksamuel/scrimage/tree/master/scrimage-webp)가 제공하는 [WebpImageReader](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/WebpImageReader.java)를 사용해야 하므로, 포맷에 따라 리더를 바꿔 주어야 합니다. 그래서 `asImmutableImage`에 포맷 인자를 추가해, 포맷별로 적절한 리더를 선택하도록 했습니다.

```kotlin
fun asImmutableImage(rawData: ByteArray, format: Format): ImmutableImage {
    return when (format) {
        Format.WEBP -> WebpImageReader().read(rawData)
        Format.GIF, Format.PNG, Format.JPEG -> ImageIOReader().read(rawData)
    }
}
```

다른 처리 흐름은 PNG, JPEG와 동일합니다.

### GIF의 경우

GIF는 [AnimatedGif](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/AnimatedGif.java) 클래스로 다룹니다. 이 클래스에도 리사이즈 메서드가 있으므로 비슷하게 처리할 수 있지만, 사용하는 클래스가 다르기 때문에 별도의 메서드를 둡니다.

```kotlin
fun asAnimatedGif(rawData: ByteArray): AnimatedGif {
    return AnimatedGifReader.read(ImageSource.of(rawData))
}
```

`AnimatedGif`는 `frames` 속성으로 각 프레임을 보관하고 있으며, 이 프레임들은 `ImmutableImage`처럼 다룰 수 있습니다. 따라서 각 프레임을 따로 리사이즈한 뒤 다시 `AnimatedGif`로 조립하는 방식으로 처리합니다. 코드로 쓰면 조금 길지만, 구조는 다음과 같습니다.

```kotlin
suspend fun resizeAnimatedGif(gif: AnimatedGif, width: Int): AnimatedGif {
    val resizedData = ByteArrayOutputStream().use {
        StreamingGifWriter().prepareStream(it, BufferedImage.TYPE_INT_ARGB).use { stream ->
            gif.frames.mapIndexed { index, image ->
                stream.writeFrame(image.scaleToWidth(width), gif.getDelay(index))
            }
        }
        it.toByteArray()
    }
    return AnimatedGifReader.read(ImageSource.of(resizedData))
}
```

리사이즈 결과를 `ByteArray`로 내보내는 메서드도 둡니다. 기본 형태는 `ImmutableImage`와 비슷하지만, GIF를 WEBP로 바꾸는 경우에는 [Gif2WebpWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/Gif2WebpWriter.java)를 사용합니다. 이렇게 하면 WEBP로 바꾼 뒤에도 GIF 애니메이션을 유지할 수 있습니다.

```kotlin
fun encodeGif(gif: AnimatedGif): ByteArray {
    return gif.bytes(Gif2WebpWriter.DEFAULT)
}
```

## 처리 공통화

여기까지 PNG, JPEG, WEBP, GIF 네 가지 포맷에 대한 리사이즈 처리를 만들었습니다. 하지만 포맷마다 비슷한 코드를 반복하면 중복이 너무 많아지므로 공통화가 필요합니다. 특히 `ImmutableImage`와 `AnimatedGif`는 흐름이 꽤 비슷하므로 같이 묶을 수 있습니다.

### 공통 Interface 만들기

Scrimage에는 `ImmutableImage`와 `AnimatedGif`가 따로 있고, 공통 인터페이스가 없습니다. 그래서 먼저 공통 인터페이스를 하나 만들고, 이를 감싸는 Wrapper를 두었습니다. 각 클래스는 원본 객체를 보관하면서 필요한 속성을 인터페이스로 노출합니다.

```kotlin
sealed interface Image {
    val width: Int
}

class AnimatedGifWrapper(
    val animatedGif: AnimatedGif
) : Image {
    override val width: Int
        get() = animatedGif.frames.first().width
}

class ImmutableImageWrapper(
    val immutableImage: ImmutableImage
) : Image {
    override val width: Int
        get() = immutableImage.width
}
```

### 이미지 가져오기 공통화

외부에 공개하는 API로는 `asImage`를 만들고, 포맷에 따라 적절한 Wrapper를 돌려주도록 했습니다. 여기서 포맷 판별은 앞에서 만든 `detectImageFormat`을 사용합니다.

```kotlin
fun asImage(rawData: ByteArray): Image {
    return when (val format = detectImageFormat(rawData)) {
        Format.GIF -> asAnimatedGifWrapper(rawData)
        Format.WEBP, Format.PNG, Format.JPEG -> asImmutableImageWrapper(rawData, format)
    }
}

private fun asAnimatedGifWrapper(rawData: ByteArray): AnimatedGifWrapper {
    val gif = AnimatedGifReader.read(ImageSource.of(rawData))
    return AnimatedGifWrapper(gif)
}

private fun asImmutableImageWrapper(rawData: ByteArray, format: Format): ImmutableImageWrapper {
    val image = when (format) {
        Format.WEBP -> WebpImageReader().read(rawData)
        Format.GIF, Format.PNG, Format.JPEG -> ImageIOReader().read(rawData)
    }
    return ImmutableImageWrapper(image)
}
```

### 리사이즈 처리 공통화

리사이즈 처리도 같은 방식으로 공통화합니다. `resizeImage`라는 메서드를 두고, 포맷에 따라 적절한 리사이즈 함수를 호출합니다. 이때 `resizeImmutableImage`와 `resizeAnimatedGif`를 재사용하고, GIF 처리용으로는 `writeAnimatedGif`를 따로 분리했습니다.

`Image`를 `sealed interface`로 만들었기 때문에, `when` 식으로 분기하면 모든 경우를 안전하게 다룰 수 있습니다.

```kotlin
suspend fun resizeImage(image: Image, width: Int): Image {
    return when (image) {
        is AnimatedGifWrapper -> resizeAnimatedGif(image, width)
        is ImmutableImageWrapper -> resizeImmutableImage(image, width)
    }
}

private suspend fun resizeAnimatedGif(gifWrapper: AnimatedGifWrapper, width: Int): AnimatedGifWrapper {
    val gif = gifWrapper.animatedGif
    val resizedData = writeAnimatedGif { stream ->
        gif.frames.mapIndexed { index, image ->
            stream.writeFrame(image.scaleToWidth(width), gif.getDelay(index))
        }
    }
    val resizedGif = AnimatedGifReader.read(ImageSource.of(resizedData))
    return AnimatedGifWrapper(resizedGif)
}

private suspend fun writeAnimatedGif(writeFunction: (StreamingGifWriter.GifStream) -> Unit): ByteArray {
    return withContext(Dispatchers.IO) {
        ByteArrayOutputStream().use { out ->
            StreamingGifWriter().prepareStream(out, BufferedImage.TYPE_INT_ARGB).use { stream ->
                writeFunction(stream)
            }
            out.toByteArray()
        }
    }
}

private fun resizeImmutableImage(imageWrapper: ImmutableImageWrapper, width: Int): ImmutableImageWrapper {
    val resizedImage = imageWrapper.immutableImage.scaleToWidth(width)
    return ImmutableImageWrapper(resizedImage)
}
```

### 이미지 인코딩 처리 공통화

마지막으로 인코딩 처리도 공통화합니다. `encodeImage` 하나로 포맷별 인코딩을 처리하면, 호출하는 쪽은 세 메서드만 알면 됩니다.

```kotlin
suspend fun encodeImage(image: Image): ByteArray {
    return when (image) {
        is AnimatedGifWrapper -> image.animatedGif.bytes(Gif2WebpWriter.DEFAULT)
        is ImmutableImageWrapper -> image.immutableImage.bytes(WebpWriter.DEFAULT)
    }
}
```

이렇게 하면 호출하는 쪽에서는 `asImage`, `resizeImage`, `encodeImage` 세 메서드만 사용하면 됩니다. 크기도 `Image`의 프로퍼티로 갖고 있으니, 리사이즈가 필요한지도 쉽게 판단할 수 있습니다.

`ImageWriter`처럼 매번 새로 만들 필요가 없는 클래스는 싱글턴으로 두면 더 좋습니다.

## 마지막으로

여기까지 동적 이미지 리사이즈 API를 만드는 방법을 정리했습니다. Scrimage를 쓰면 PNG, JPEG, WEBP, GIF 네 가지 포맷을 다룰 수 있고, 공통화까지 해 두면 처리 중복도 줄일 수 있습니다.

이후에는 Cloud Run에서 동작하는 Dockerfile을 만들고 Cloud Storage를 연결하면 실제 이미지 서비스를 구성할 수 있습니다. 여기에 Cloud CDN을 붙이면 두 번째 요청부터는 캐시를 돌려주므로 부하도 줄일 수 있습니다. 구현 자체도 흥미로웠지만, 이미지 처리처럼 단순해 보이는 기능도 실제로는 고려할 부분이 많다는 점을 다시 확인한 프로젝트였습니다.
