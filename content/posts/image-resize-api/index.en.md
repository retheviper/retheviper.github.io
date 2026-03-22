---
title: "Building a Dynamic Image Resizing API"
date: 2024-03-30
translationKey: "posts/image-resize-api"
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - ktor
  - kotlin
  - scrimage
  - rust
---

This time, I created an API for dynamic image resizing, and I would like to introduce it. The dynamic image resizing API referred to here is an API that returns an image resized to that size when you specify the URL of the original image and the size you want to resize.

## Purpose

What is the reason for creating a dynamic image resizing API in the first place? Then I have to explain. Until now, when distributing images, when an end user uploaded an image, they had to create and upload a thumbnail image in advance.

However, the problem with that method is that you cannot create thumbnails that are compatible with all screen ratios and resolutions. Therefore, as an alternative, I would like to specify the screen-optimized size from the front end and retrieve the resized image from the API.

## Design/technology selection

We decided to make the API as a microservice as simple as possible. It has one Get endpoint, and if you specify the URL of the original image and the size you want to resize as query parameters, it will return the resized image. On the front end, we want to be able to use it as is in the img tag, so we return the image data as is (specify `Content-type` in the header). In addition to resizing, it also allows you to convert image formats.

Also, I decided to run the API on [Cloud Run](https://cloud.google.com/run?hl=ja). This is also the case with existing APIs, so this is to match the usage, and it is also because local development can be easily done using docekr compose. We also use [Cloud Storage](https://cloud.google.com/storage?hl=ja) to distribute content, so it is easy to integrate with it. The final processed image will then be stored in Cloud CDN, and subsequent calls will return the cache.

Our company already uses Ktor as a web framework, so we decided to use Ktor here as well for consistency. Before settling on Ktor, I had also tried Node.js and Rust, but I ruled out Node.js because of performance concerns and Rust because it would have been difficult to maintain internally. Sample code close to the Rust version I was building at the time is available on GitHub, so you can check it [here](https://github.com/retheviper/resize-api).

I decided to use [Scrimage](https://github.com/sksamuel/scrimage) for image conversion and resizing. We also considered Java's [ImageIO](https://docs.oracle.com/javase/8/docs/api/javax/imageio/ImageIO.html) as other candidates. However, it was necessary to process WebP as the format of the image to be resized and the format of the data to be returned, and many of them did not support it.

Another idea is to use a tool like [GraphicsMagick](http://www.graphicsmagick.org/) that converts and resizes images. In this case, the data obtained from Java is written to a file and then executed on the command line, which incurs I/O costs, so I decided to use Scrimage this time.

## Resize processing

Now let's write the actual API. Since I am familiar with Ktor, this time I will focus on resizing images using Scrimage rather than building an API with Ktor.

The overall processing flow of the API is roughly as follows.

1. Get URL and resized size from query parameters
2. Image acquisition
3. Image resizing
4. Image format conversion

Processing the image using Scrimage is part 3 and 4 here, but before performing the actual resizing, it is also necessary to first determine the format of the acquired image and check the size of the image. The reason is to improve processing efficiency.

This time, the API handles PNG, JPEG, WEBP, and GIF, resizes them when necessary, and then converts the result to WEBP. If the original image is already WEBP, there is no need to convert it again. Also, depending on the requested size, resizing itself may not be necessary. So the API first determines the image format and then performs resizing only when it is actually needed.

## Image format determination

When retrieving an image from a URL or local storage (this time we use Cloud Storage by mounting it on Cloud Run), we need to determine the format of the image. Scrimage provides a class called [FormatDetector](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/format/FormatDetector.java) to determine the image format.

It is easy to use, just pass the read image data as a Byte array as shown below. If a format that is not expected on the API is received, an error is returned, and the [Format](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/format/Format.java) returned here is PNG, GIF, JPEG, WEBP, and the one from Scrimage is used as is.

```kotlin
fun detectImageFormat(data: ByteArray): Format {
    return FormatDetector.detect(data).orElseThrow { IllegalArgumentException("Unsupported format") }
}
```

## PNG, JPEG processing

First, let's look at the simplest cases, PNG and JPEG. These formats will be treated as Scrimage's [ImmutableImage](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/ImmutableImage.java).

Here, to convert the image data to `ImmutableImage`, you use a class that implements the [ImageReader](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/ImageReader.java) interface. The [official docs](https://sksamuel.github.io/scrimage/) say that `ImmutableImage.loader()` can load images regardless of format, but when I actually built the API, it caused an AWT-related error. In practice, I had to switch the loader depending on the format.

```kotlin
fun asImmutableImage(rawData: ByteArray): ImmutableImage {
    return ImageIOReader().read(rawData)
}
```

After converting to ImmutableImage, perform resizing. Scrimage has a method for resizing, so use that to resize.

It should be noted here that there are methods such as `resize()` and `resizeTo()`, but the difference is that the former resizes by a percentage, and the latter resizes to a specified size. In these cases, the aspect ratio of the original image is not preserved, so you must use methods such as `scaleTo()` or `scaleToWidth()`.

This time, we will use `scaleToWidth()` to specify only the width and resize while preserving the aspect ratio.

```kotlin
fun resizeImmutableImage(image: ImmutableImage, width: Int): ImmutableImage {
    return image.scaleToWidth(width)
}
```

Finally, we will prepare a method to return the resized result as a ByteArray. You need to choose a class that implements [ImageWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/ImageWriter.java) depending on which format you want to convert. This time I want to use WEBP, so I will use [WebpWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/WebpWriter.java).

```kotlin
fun encodeImage(image: ImmutableImage): ByteArray {
    return image.bytes(WebpWriter.DEFAULT)
}
```

Finally, in the Router, specify the Content-Type and return a ByteArray.

```kotlin
call.respondBytes(
    bytes = resizedImage,
    contentType = ContentType("image", "webp")
)
```

## For WEBP

In the case of WEBP, it is necessary to use `WebpImageReader` when reading the above data. The subsequent resizing process is the same as for PNG and JPEG.

However, if the format is WEBP, you need to use [WebpImageReader](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/WebpImageReader.java) provided by [scrimage-webp](https://github.com/sksamuel/scrimage/tree/master/scrimage-webp). Therefore, it is necessary to change the class to be loaded depending on the format. Add the format to the `asImmutableImage` argument from earlier and change the class to be loaded depending on the format.

```kotlin
fun asImmutableImage(rawData: ByteArray, format: Format): ImmutableImage {
    return when (format) {
        Format.WEBP -> WebpImageReader().read(rawData)
        Format.GIF, Format.PNG, Format.JPEG -> ImageIOReader().read(rawData)
    }
}
```

Other processing is the same as for PNG and JPEG.

## For GIFs

For GIF, use the class [AnimatedGif](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/AnimatedGif.java) to resize it. Like ImmutableImage, there is a resizing method, so use that to resize it. Since the classes used for processing are different, a separate method is prepared for GIF.

```kotlin
fun asAnimatedGif(rawData: ByteArray): AnimatedGif {
    return AnimatedGifReader.read(ImageSource.of(rawData))
}
```

Also, in the case of AnimatedGif, data for each frame is held in a property called frames, and these can be treated as ImmutableImage. Therefore, resizing is performed for each frame and then converted back to AnimatedGif. It's a little complicated, but you can write it like this:

```kotlin
suspend fun resizeAnimatedGif(gif: AnimatedGif, width: Int): AnimatedGif {
    val resizedData = ByteArrayOutputStream().use {
        StreamingGifWriter().prepareStream(it, BufferedImage.TYPE_INT_ARGB).use { stream ->
            gif.frames.mapIndexed { index, image ->
                stream.writeFrame(image.scaleToWidth(width), gif.getDelay(index))
            }
            it.toByteArray()
        }
    return AnimatedGifReader.read(ImageSource.of(resizedData))
}
```

Finally, we will prepare a method to return the resized result as a ByteArray. Basically the same as ImmutableImage, but use [Gif2WebpWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/Gif2WebpWriter.java) when converting GIF to WEBP. Now, even after converting to WEBP, you can resize the GIF while preserving its animation.

```kotlin
fun encodeGif(gif: AnimatedGif): ByteArray {
    return gif.bytes(Gif2WebpWriter.DEFAULT)
}
```

## Unifying the Processing

Up to this point, we have been able to resize four formats: PNG, JPEG, WEBP, and GIF. However, if you write processing for each format, the processing will be duplicated, so it is necessary to standardize it. In particular, the processing of ImmutableImage and AnimatedGif is similar, so we will make them common.

## Create a common interface

In Scrimage, ImmutableImage and AnimatedGif are not only different classes, but they also do not have a common Interface, so we need to create one first. Here, we will create an Interface called Image and a class to implement it. Create each class as a Wrapper and have the properties of each class as properties of Interface.

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

## Unifying Image Loading

Next, create a method called `asImage` as an API to be exposed to the outside world, and return a Wrapper according to each format. Here, we will use `detectImageFormat` that we created earlier to determine the format.

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

## Unifying Resize Logic

Similarly, resizing processing will also be standardized. Here, we will create a method called `resizeImage` and perform the resizing process according to each format. Here, we will use `resizeImmutableImage` and `resizeAnimatedGif` that we created earlier for the resizing process. AnimatedGif resizing processing is also separated by creating a separate method called `writeAnimatedGif`.

Here, Image is created as a sealed interface, so branching can be covered using a when expression.

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
    return AnimatedGifWrapper(resizedGif, resizedData)
}

private suspend fun writeAnimatedGif(writeFunction: (StreamingGifWriter.GifStream) -> Unit): ByteArray {
    return withContext(Dispatchers.IO) {
        StreamingGifWriter().use {
            streamingGifWriter.prepareStream(it, BufferedImage.TYPE_INT_ARGB).use { stream -> writeFunction(stream) }
            it.toByteArray()
        }
    }
}

private fun resizeImmutableImage(imageWrapper: ImmutableImageWrapper, width: Int): ImmutableImageWrapper {
    val resizedImage = imageWrapper.immutableImage.scaleToWidth(width)
    return ImmutableImageWrapper(resizedImage, imageWrapper.format)
}
```

## Unifying Image Encoding

Finally, we will also standardize the image encoding process. Here, we will create a method called `encodeImage` and perform encoding processing according to each format.

```kotlin
suspend fun encodeImage(image: Image): ByteArray {
    return when (image) {
        is AnimatedGifWrapper -> image.animatedGif.bytes(Gif2WebpWriter.DEFAULT)
        is ImmutableImageWrapper -> image.immutableImage.bytes(WebpWriter.DEFAULT)
    }
}
```

This completes the commonality, and the caller can perform resizing simply by using three methods: `asImage`, `resizeImage`, and `encodeImage`. The size check is also a property of Image, so you can use it to determine whether resizing is necessary.

It is also a good idea to create classes such as ImageWriter that do not need to be instantiated each time as singletons.

## Finally

This was a long post, but that is the outline of how I built a dynamic image resizing API. By using Scrimage, I was able to handle PNG, JPEG, WEBP, and GIF, and by unifying the processing flow, I could avoid duplicating logic.

See you soon!
