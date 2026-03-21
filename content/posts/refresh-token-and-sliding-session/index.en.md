---
title: "Complementing JWT: Refresh Token and Sliding Session"
date: 2020-08-02
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring security
  - rest api
  - jwt
---
Last time, I wrote a [post](../spring-rest-api-login-with-jwt/) about how to implement this with Spring Security along with JWT, but it seems that there is a possible vulnerability if only an Access Token is issued using JWT. Therefore, this time, I would like to introduce the characteristics and problems of authentication that uses only Access Tokens, and technologies to supplement them.

In the first place, there seems to be no correct implementation method for authentication and authorization using JWT, although JWT is an ISO standard. Therefore, I will only introduce the concepts of the problems associated with issuing only Access Tokens and strategies to supplement them, which will be introduced below.

## Access Token

If you use only an Access Token, after the first authentication, the server uses the private key to confirm the token included in the request, which eliminates the need for I/O such as databases and storage, resulting in faster processing. However, since the token is held by the client sending the request, it is not possible to forcefully terminate the session. Therefore, the only way to process logout is to delete the token information from the client side.

Authentication methods that use Access Tokens have these characteristics; the client side owns the tokens, so if this is hijacked by a third party, there is no way to determine this from the server side. To prevent this problem, when issuing a token, we set a time limit in advance so that the token expires after a certain amount of time and cannot be reused.

## Token expiration dilemma

When setting the expiration date of the token in advance, there is an issue of how to set the expiration date. If it is short, security can be ensured because even if the token is hijacked, the token cannot be used immediately, but on the other hand, the user will have to log in frequently to be able to access the token. Conversely, if you make it longer, users will be able to use the service for a longer period of time without having to log in again, but the hijacked token will also be able to be used for a longer period of time.

## Refresh Token

Refresh Token is another Token for updating the expiration date of an expired Access Token. The server issues both an Access Token and a Refresh Token to the client during authentication. Here, set the Access Token expiration time to a short period of about 30 minutes, and set the Refresh Token to a long period of about 1 month.

If 30 minutes have passed after logging in on the client side and the Access Token has expired, use Refresh Token to request the server to issue a new Access Token. The server verifies the Refresh Token included in the client's request, and if there is no problem, reissues the Access Token. Now, even if the Access Token is hijacked, it will expire immediately, and the user will have a Refresh Token to automatically renew the expiration date, eliminating the need to log in every time. We also have the data to validate the Refresh Token on the server side, so we can expire the Refresh Token if there is a problem.

However, even when using Refresh Token, it is not perfect. The problem is that the data corresponding to the Refresh Token needs to be stored in a separate database or storage for the server to validate, which can result in additional I/O. Also, in order to update the Access Token using Refresh Token, it is necessary to add functionality to both the server and client side. Also, like the Access Token, the client side needs to save the Refresh Token, so you need to think about how to save it.

## Sliding Session

Sliding Session is a mechanism that automatically updates the Access Token if a user continues to use the session. There is a way to issue a new Access Token for each request, but there is also a way to process only specific requests. For example, it might be a page where you expect a user to spend a lot of time (creating a ticket), or a page where you expect them to take some action next (adding an item to a shopping cart). It seems that there are also cases where you check the IAT (token issuance time) of the Access Token and request the issuance of a new Access Token.

However, when updating the Access Token using Sliding Session, please be aware that if the expiration date is set to a long time in the first place, the session will be extended almost infinitely, and the user will be able to use the service without logging in at all. On the other hand, if the user does not work for a long time, there may be no need for Sliding Session itself.

## Refresh Token + Sliding SessionThe last method is to use Refresh Token and Sliding Session at the same time. The difference from applying a normal Sliding Session is that the Refresh Token is updated instead of the Access Token. By updating the Refresh Token, you no longer need to update the Access Token frequently. Also, since the Refresh Token is updated, even users who access the site infrequently will not need to log in frequently. It can be said that this is a strategy that appropriately combines the advantages of Refresh Token and Sliding Session.

However, this method cannot be said to be completely free of problems. Users can access resources until the Refresh Token expires from the server, so if a device such as a PC or smartphone is hijacked, we cannot respond. Therefore, you will need to periodically reset your account password or re-authenticate when accessing certain resources.

## Finally

Authentication is the most important and common feature of web applications, but there is a problem in that increasing security for the sake of security has a negative impact on user usability. Therefore, there are various methodologies, and various web applications use different methods to deal with vulnerabilities, but they all have advantages and disadvantages, so there may not be a correct answer.

Also, since the JWT payload is not encrypted (it's just a base64 string), there seems to be a problem that if the JWT is hijacked, its contents can be viewed. It seems that something called [PASETO](https://developer.okta.com/blog/2020/07/23/introducing-jpaseto) has been proposed to supplement this, but I don't know if it will be used as sub-supplied as JWT.

Also, regarding the issue of authentication methods in the first place, it is said that once quantum computers are available in general, it will be possible to analyze cryptography using conventional methods, so the very concept of encryption may change. If that happens, existing apps will have to be completely rebuilt.

Security is difficult. In addition to encryption, I feel that various efforts are needed to appropriately select a design and strategy that is convenient for users.