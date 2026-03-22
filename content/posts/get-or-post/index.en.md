---
title: "Between GET and POST"
date: 2020-09-21
translationKey: "posts/get-or-post"
categories: 
  - recent
image: "../../images/magic.webp"
tags:
  - blog
  - http
---
According to the [MDN explanation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods), HTTP methods are mainly defined in terms of resources. Therefore, when explaining the methods to write, GET is used to represent a resource, POST is used to create a resource, PUT is used to replace a resource, and DELETE is used to delete a resource. Based on this, many materials explain the basics of CRUD, and learners naturally think about API design in terms of creating, reading, replacing, and deleting resources. The other important thing is whether the request handling is [idempotent](https://en.wikipedia.org/wiki/Idempotence).

## Theory != Reality

However, even if you understand this in theory, it can be troublesome when you try to express your actual business as an application. That's the case this time. The question was, ``What should I do when the process looks like it simply returns a resource, but actually involves creating or modifying a resource internally?'' Because there were such requirements.

- There is an API that provides downloading of a certain file to the client.
- There are two cases when retrieving files from the server:
  - The file has already been created and the server just returns it to the client.
  - In response to a request, the server writes (creates) a DB record to a file and returns it to the client.
- When the client downloads the file, the server updates the DB.
  - The server registers the flag "File output completed" and the information of the logged-in user as the "update user" in the DB.

If the file has already been created and the server just returns it to the client, GET will do the trick. From the client's perspective, the server just returns the resources it already has. This matches the "representation of resources" that is generally expected from GET, and there is no problem.

But what about "In response to a request, the server writes records from the DB to a file and returns them to the client"? Does creating a new file fall under "creating a resource"? Or, since it already exists as a record, does it just need to be processed into a different form, so does it fall under the category of "resource expression"? Or, since the DB is updated with information provided by the client in addition to requesting files, does this apply to "creating a resource"?

## From whose perspective should it be viewed?The above issues are from my own perspective, as a backend engineer and an application. If so, what do you think from the client's (user's) point of view? You can think of the following:

- The client is not interested in whether the file has already been created or if it will be created for you.
  - A request is "give me a file", not "create a file".
- It is up to the server (application) whether or not to create a file according to the client's request.
  - The client only requests the file and does not know what is going on inside.

In other words, from the client's perspective, the act of downloading a file is nothing more than a request for "representation of a resource." The circumstances are different from the server side. In this case, it's obvious which option suits your needs. The client should be the priority. Because applications exist in the first place because of client requirements. Therefore, there is no reason to make the request POST for reasons on the server side.

## In Spec

However, I think there is a different basis for deciding that GET is correct in this case from the client's perspective. For example, the HTTP standard. Just like the text about HTTP specifications, there is a [chapter like the following](https://tools.ietf.org/html/rfc7231#section-4.2.1).

> 4.2.1. Safe Methods
>> Request methods are considered "safe" if their defined semantics are essentially read-only; i.e., the client does not request, and does not expect, any state change on the origin server as a result of applying a safe method to a target resource. Likewise, reasonable use of a safe method is not expected to cause any harm, loss of property, or unusual burden on the origin server.

> 4.2.1 Safe methods
>> Read-only requests where the client does not change or expect the server's state to change are considered "safe." Proper use of safe methods prevents situations that may cause harm or damage to the server.

Looking at this, it seems that if the client's request causes a change in the server's resources, it is not a GET. But what's important is this:>> This definition of safe methods does not prevent an implementation from including behavior that is potentially harmful, that is not entirely read-only, or that causes side effects while invoking a safe method. What is important, however, is that the client did not request that additional behavior and cannot be held accountable for it. For example, most servers append request information to access log files at the completion of every response, regardless of the method, and that is considered safe even though the log storage might become full and crash the server.

>> Safe methods do not prevent implementations that are not fully read-only, have side effects, or otherwise cause harm. The important thing is that the action is not the client's request, nor is the client responsible. For example, many servers record access logs for every method, and the logs can cause the server to run out of storage and cause the server to fail, but requests are still safe in this case.

In other words, the important thing is that implementation considerations are the responsibility of the server side, and are the actions of the client. The conclusion is that for secure requests, you should consider what the client's request is. The HTTP method represents the essence of such an action, and is not an expression of the server's convenience. So, from the client's perspective, if the request is for "representation of a resource," I think the server should respond with a GET.

## Finally

When I look at actual websites, I feel that GET is rarely used as a pure "representation of resources". For example, there are some sites that update their ``visitor counter'' just by visiting them. Even in this case, the client does not create or request the resource, and the server updates it on its own, but it is still treated as GET, and there is no discomfort. In fact, if all such resource changes were to be treated as POST, there might be no need to use GET at all.

However, you must be careful that the idea of ​​making all requests that involve creating or updating resources GET is also dangerous. If we go back to the principle of how to handle resources, there may be a way to create them using POST and then retrieve them using GET. In this case, two transactions occur, so performance may be inferior, but depending on the case, it may be a safer design.

Therefore, I think the conclusion is that it is better to select an appropriate HTTP method based on the overall design of the application and its future direction, rather than choosing it "for all cases" (though this is just a basic theory). The important thing is not to say, ``In these cases, you should definitely use GET'' or ``In these cases, you should definitely use POST.''
