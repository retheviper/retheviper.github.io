---
title: "Send files from Rest API to Rest API"
date: 2020-02-10
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - rest api
  - java
  - spring
---
When developing a web application, there are times when it is not necessary for all functions to be self-contained with just one `Rest API`. For example, there may be a server (API) that already has the functionality you want to incorporate. In such cases, you can easily achieve your goal by simply calling the API. In fact, I looked into it because I needed to communicate with other Rest API servers at work, but Spring already had a class that could handle such cases. RestTemplate is the main character of this post.

Using RestTemplate, you can easily call Http method APIs such as get, post, and delete. You can also customize requests and responses to suit your situation. For example, you can create custom headers or make requests to transfer large files. So this time, I will use RestTemplate to show you how to create a component that calls that API when there is already an API that processes and returns a file when you upload it.

## Server side example

This is when a server that processes files already exists, so first you need to understand the request and response formats used by the API you want to call. Let's say that when a file is uploaded, there is a method that reads the file information from the header, and reads the body where the file data is written. If there is no problem with the header information, write the file to local storage and process it. The processed file is then returned to the server as a response.

```java
@PostMapping("/upload")
public ResponseEntity<StreamingResponseBody> fileupload(HttpServletRequest request) {
    // Get the file size from the request header
    long fileLength = request.getContentLengthLong();
    // File size 0 causes an IOException
    if (fileLength == 0) {
       throw new IOException("data size is 0");
    }

    // Save the file as a temporary file
    String fileName = request.getHeader("Content-File-Name");
    Path tempFile = Files.createTempFile("process_target_", fileName);
    try (InputStream is = request.getInputStream()) {
        Files.copy(is, tempFile);
    } catch (IOException e) {
        e.printStackTrace();
    }

    // ...perform some processing on the file

    // Create headers for the response
    HttpHeaders headers = new Headers();
    // Create a body that writes the processed file
    StreamingResponseBody body = new StreamingResponseBody() {
        @Override
        public void writeTo(OutputStream outputStream) throws IOException {
            byte[] bytes = Files.readAllBytes(path);
            outputStream.write(bytes);
        }
    };
    // Return a response containing the headers, body, and HttpStatus
    return new ResponseEntity<StreamingResponseBody>(body, headers, HttpStatus.OK);
}
```

If the server is configured like this, the person calling the API needs to write file information in the Request header and write file data in the body before transferring it. Since the processing result Response also contains file data, we need to process it to receive it. To do this, let's create custom requests and responses and put them on the RestTemplate.

## Use RestTemplate

This time we will use the execute() method of RestTemplate, and the arguments for this method are as follows.

1. API URL (String or URI)
2. Http method (ENUM)
3. Request (RequestCallback implementation class)
4. Response (ResponseExtractor implementation class)

The reason why we specify both the request and response, unlike when using methods such as get(), post(), and delete(), is that as mentioned above, file transfer is required for both requests and responses. You can also specify a URI variable as an argument with execute(), but currently the URI is fixed, so we don't use it. Now, let's take a look at how to implement the request and response interface.

## Request

Create an implementation class for RequestCallback to be used in requests. Let's create a header and body by passing a file as a constructor argument to this interface. Implementing RequestCallback will override a method called doWithRequest(). By setting the header and body to ClientHttpRequest, which is an argument of this method, you can upload files at the time of request. See the code below.

```java
public class FileTransferRequestCallback implements RequestCallback {

    // File to upload
    private Path path;
    
    // Constructor used to put file information into the header
    public PdfConvertRequestCallback(File file) {
        this.path = file.toPath();
    }

    @Override
    public void doWithRequest(ClientHttpRequest request) throws IOException {
        // Create headers from the file
        request.getHeaders().set("Content-Length", Files.size(path));
        request.getHeaders().set("Content-File-Name", path.getFileName().toString());
        // Write the file into the body
        try (InputStream is = Files.newInputStream(file); OutputStream os = request.getBody()) {
            is.transferTo(os);
        }
    }
}
```

The header contains the file size and file name requested by the server. Then, get the file to be uploaded as an InputStream and write it to the body, which is an OutputStream. This completes the file upload settings for requests. Next is the response.

## Response

In the response, implements ResponseExtractor. In this case, the method called extractData() will be overridden. You can get the Http status code, header, and body from ClientHttpResponse, which is an argument of this method, just like when making a request. If you create an instance of ResponseEntity from this response result and return it with the response result, RestTemplate will return ResponseEntity as the communication result.

To return a ResponseEntity, you need to specify the body type. Let's specify the type of InputStream to specify that the body of the response is a file. Also, if you put the body of ClientHttpResponse as is in ResponseEntity, InputSteam will be closed, so copy the body. I decided to change it to byte[] and then generate a ByteArrayInputStream.

```java
public class FileTransferResponseExtractor implements ResponseExtractor<ResponseEntity<InputStream>> {

    @Override
    public ResponseEntity<InputStream> extractData(ClientHttpResponse response) throws IOException {
        // Copy the response body
        byte[] bytes = response.getBody().readAllBytes();
        // Return ResponseEntity with the status code, headers, and body data
        return ResponseEntity.status(response.getStatusCode()).headers(response.getHeaders()).body(new ByteArrayInputStream(bytes));
    }
}
```

Now you can get the response file. Next, just call the API on RestTemplate.

## Calling the Rest APIAs mentioned earlier, executing RestTemplate's methods is easy. First, create a URL and an instance of the file you want to upload. Then, create instances of the RequestCallback and ResponseExtractor that you created earlier (ResponseExtractor has no state, so you can register it as a bean).

If you specify the URL, Http method type, RequestCallback, and ResponseExtractor in the execute() argument, you can obtain the result as a ResponseEntity, from which you can further obtain the status code, header, and body. Now you can process the uploaded file and get the processed file immediately.

```java
// Prepare the arguments to pass to RestTemplate
String url = "http://api/v1/file/upload";
File uploadFile = new File("path/to/upload_file.txt");
FileTransferRequestCallback requestCallback = new FileTransferRequestCallback(uploadFile);
FileTransferResponseExtractor responseExtractor = new FileTransferResponseExtractor();

// Call the API with RestTemplate and get the result
ResponseEntity<InputStream> responseEntity = new RestTemplate().execute(url, HttpMethod.POST, requestCallback, responseExtractor);

// Get the HTTP status from ResponseEntity
if (responseEntity.getStatusCode() != HttpStatus.OK) {
    throw new IOException();
}
// Get the headers from ResponseEntity
HttpHeaders headers = responseEntity.getHeaders();

// Get the body from ResponseEntity
try (InputStream is = responseEntity.getBody()) {
    File downloadedFile = new File("path/to/downloaded_file.txt");
    Files.copy(is, downloadedFile);
}
```

Surprisingly easy! Now you can exchange files with other Rest APIs.

## Finally

In addition to separating classes and methods by function, the Rest API may also be separated depending on function. Such was the case in this case. Of course, since it goes through the internet, this method may be less stable than putting all the functions together in one Rest API, but I think it's a good method in terms of ensuring reusability. If it is within the same server, the probability of communication failure will be lower, and there seems to be room for various uses.

I haven't had much content to post on my blog lately (although I plan on continuing to study), so I was worried about what to write next, but I'm glad I was able to create some interesting parts. The world of Spring is also really wide and deep. See you soon!
