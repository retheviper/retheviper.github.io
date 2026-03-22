---
title: "Implementing Role-based Authorization with Ktor"
date: 2021-08-09
translationKey: "posts/ktor-role-based-authorization"
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - kotlin
  - ktor
  - authorization
---

Last time, while introducing Ktor, I mentioned that Ktor does not yet support `Role-based Authorization`, so you will need to implement such functionality yourself. Ktor has a short history and is not a widely used framework like Spring, Django, or Rails, so it probably doesn't have enough functionality to create real applications compared to other frameworks. So, if you don't have the required functionality, you have no choice but to directly implement that functionality.

Fortunately, in Ktor, functions are called [Plugin](https://ktor.io/docs/plugins.html) and can be added in module units, so you can implement the required functions by creating a plugin. However, although using modules makes it easier to manage each function, you also need to know how the module functions and what manners are required.

This time, I tried implementing Ktor's role-based authorization as a `Plugin` while referring to an [article](https://medium.com/@shrikantjagtap99/role-based-authorization-feature-in-ktor-web-framework-in-kotlin-dda88262a86a) published on the internet. In this post, I want to explain how that custom `Plugin` works as a Ktor feature and how to implement it.

## What is Role-based Authorization?

First of all, what is `Role-based Authorization`? This is one of the "authorization" methods often used in web applications that controls API execution based on the user's `Role` (role). For example, in the case of an e-commerce site, any authenticated user should be able to inquire about a product, but functions such as "writing a notice" or "changing the number of products in stock" should be limited to users with such privileges (Admin). Therefore, the basic concept of `Role-based Authorization` is to set up `Role` such as "general user" and "administrator", and when a request to an API occurs, that `Role` is checked first, and only users with that authority can execute the API.

To achieve this, there are two main things that need to be introduced into existing apps: the concept of `Role` and a mechanism to filter requests based on it. In the former case, all you have to do is think about what roles exist and how to link them to users, so all you need to do is add tables and columns and link them to existing user information. However, for the latter, you first need to understand how the framework filters requests, starting with its structure. First of all, I would like to introduce how to handle requests with Ktor.

## Pipelines and Features

The most important feature of Ktor is the concept of [Pipeline](https://ktor.io/docs/pipelines.html). The official explanation for `Pipeline` is as follows.

> The pipeline is a structure containing a sequence of functions (blocks/lambdas) that are called one after another, distributed in phases topologically ordered, with the ability to mutate the sequence and to call the remaining functions in the pipeline and then return to current block.

It is difficult to understand from this explanation alone, but in short, it can be said that it refers to a unit of processing in Ktor. `Pipeline` allows you to define a series of processes from API calls to responses. Therefore, the typical function realized as `Pipeline` is `Router`, a function that defines handling for requests (corresponding to Spring's `Controller`).

Also, since `Pipeline` is extensible, it is also possible to realize a module (`Plugin` in official terms) by implementing a new `Pipeline` according to that format. A feature of Ktor is that by implementing these modules and installing them in your application, you can use the functionality of those modules. For example, to add [kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization), the official kotlin JSON mapper, to your application, write the code below.

```kotlin
fun Application.main() {
    install(ContentNegotiation) {
        json()
    }
}
```

Looking at the implementation of the `install` function called here, it looks like the following. You can see that the arguments are `feature` (module) and `configure`, which is the setting for that module.

```kotlin
public fun <P : Pipeline<*, ApplicationCall>, B : Any, F : Any> P.install(
    feature: ApplicationFeature<P, B, F>,
    configure: B.() -> Unit = {}
): F
```

I can guess that in the code used to install `kotlinx.serialization` earlier, `ContentNegotiation` is passed as `feature`, and the setting is to use `json`. In fact, the implementation of `ContentNegotiation` looks like this: Although some codes are omitted, you can see that some of the classes have a class called `Configuration` and a class called `companion object` that inherits from `ApplicationFeature`.

```kotlin
public class ContentNegotiation internal constructor(
    public val registrations: List<ConverterRegistration>,
    private val acceptContributors: List<AcceptHeaderContributor>,
    private val checkAcceptHeaderCompliance: Boolean = false
) {

    // ...

    /**
     * Configuration type for [ContentNegotiation] feature
     */
    public class Configuration {
        // ...
    }

    /**
     * Implementation of an [ApplicationFeature] for the [ContentNegotiation]
     */
    public companion object Feature : ApplicationFeature<ApplicationCallPipeline, Configuration, ContentNegotiation> {
        // ...
    }
```

As you can see from the implementation above, in order to function as `Pipeline`, you need a class called `Configuration` for module settings, and `companion object`, which inherits from `ApplicationFeature`, to function as a module. So, you can see that if you can define a class with this structure, you can implement your own module in your application.

## Plugin implementation

Now, let's say we actually create a function to determine authorization for requests as `Pipeline`. First, define the role. `enum` looks good. Here, we will simply create two types: administrator and general user.

```kotlin
enum class Role(val value: String) {
    ADMIN("ADMIN"),
    USER("USER")
}
```

These roles need to be managed in a table, etc., but they also need to be retrieved from the logged-in user information. For authorization, it is necessary to check whether the logged-in user has been granted a certain role. Therefore, we will also create a class to store the user information that inherits `io.ktor.auth.Principal`, and when the login is successful, we will store the user's role in this class (the method is not related to authorization, so we will omit it here). Below is a simple example for storing user information.

```kotlin
data class UserPrincipal(
    val username: String,
    val roles: Set<Role> = emptySet()
) : Principal
```

Next, create a function to restrict access by role. This is an image that specifies which role can access the `Router` endpoint. For example, I think it would be good to use it in the following format.

```kotlin
fun Application.main() {
    routing {
        // Accessible only to administrators
        withRole(Role.ADMIN) {
            get("/admin") {
                call.respond("This is admin page")
            }
        }

        // Accessible to regular users
        withRole(Role.USER) {
            get("/user") {
                call.respond("This is user page")
            }
        }
    }
}
```

As you can see in the usage of `Router`, code blocks (functions) in `Pipeline` can be nested, so it is also possible to sandwich one layer like this. It would be a good idea to check the role using the function `withRole` added here and determine whether you can access the API.

## Implementing AuthorizedRotueSelector

First, implement `RouteSelector`. This is to allow the authorization function that we will create to be nested inside `routing`. The simplest implementation looks like this:

```kotlin
class AuthorizedRouteSelector() : RouteSelector() {
    override fun evaluate(context: RoutingResolveContext, segmentIndex: Int) = RouteSelectorEvaluation.Constant
}
```

## Implementing child route

Using `AuthorizedRouteSelector` that we implemented earlier, we will create a function that actually functions as `child route`. Since this `child route` will be nested under `Router`, we will create an extension function for `Route`. As arguments, it would be a good idea to be able to set the role for judgment and the function of the endpoint nested under it. Implementation is as follows.

```kotlin
fun Route.withRole(val role: Role, build: Route.() -> Unit): Route {
    val authorizedRoute = createChild(AuthorizedRouteSelector())
    application.feature(RoleBaseAuthorizer).interceptPipeline(this, role)
    build()
    return authorizedRoute
}
```

The implementation here first creates `child route` with `AuthorizedRouteSelector`, then intercepts `Pipeline` and determines whether it corresponds to the role specified by the user. If there is no problem, run `build`, which will become nested `child route`. Finally, we return the instance of `child route` we created earlier so that we can nest endpoints.

`RoleBaseAuthorizer`, which is called when intercepting `Pipeline`, will be created as a separate class. This will be created as `Feature`.

## Implementation of authorization processing module

Now, let's implement the module (`Feature`) that is responsible for the full-fledged authorization process. As mentioned earlier, create a class that has `Configuration` and `Feature` inside. Here, `Configuration` is a class that allows you to configure how to obtain role information from the logged-in user. By doing this, you will be able to:

```kotlin
fun Application.main() {
    // Install RoleBaseAuthorizer as a feature
    install(RoleBaseAuthorizer) {
        // Configure how to obtain roles from the logged-in user
        getRoles { (it as UserPrincipal).roles }
    }
}
```

In `Configuration`, pass a function that retrieves `roles` from `UserPrincipal`, which is the logged-in user information. With this, `RoleBaseAuthorizer` will compare the role specified in the `withRole` function with the user's role.

Now that we have an image of how to set up the authorization module, let's implement `RoleBaseAuthorizer`. For example:

```kotlin
class RoleBaseAuthorizer(config: Configuration) {

    class Configuration {
        var userRoles: (Principal) -> Set<Role> = { emptySet() }

        // Set the role lookup function from the logged-in user information
        fun getRoles(roles: (Principal) -> Set<Role>) {
            userRoles = roles
        }
    }

    private val getRoles = config.userRoles

    fun interceptPipeline(
        pipeline: ApplicationCallPipeline,
        role: Role
    ) {
        // Pipeline placement
        pipeline.insertPhaseAfter(ApplicationCallPipeline.Features, Authentication.ChallengePhase)
        pipeline.insertPhaseAfter(Authentication.ChallengePhase, AuthorizationPhase)

        // Processing during interception
        pipeline.intercept(AuthorizationPhase) {
            // Retrieve the logged-in user's information
            val principal = call.authentication.principal<UserPrincipal>()
                ?: throw AuthorizationException("Missing principal")

            // Retrieve roles from the user information
            val roles = getRoles(principal)

            if (roles.none { role }) {
                // Processing when the user's roles do not include an allowed role
            }
        }
    }

    companion object Feature : ApplicationFeature<ApplicationCallPipeline, Configuration, RoleBaseAuthorizer> {

        override val key = AttributeKey<RoleBaseAuthorizer>("RoleBaseAuthorizer")

        val AuthorizationPhase = PipelinePhase("Authorization")

        override fun install(
            pipeline: ApplicationCallPipeline,
            configure: Configuration.() -> Unit
        ): RoleBasedAuthorization {
            val configuration = Configuration().apply(configure)
            return RoleBaseAuthorizer(configuration)
        }
    }
}
```

As explained earlier, in `Configuration`, we set a function to get the user's role information and save it. And in `interceptPipeline`, we will have that function to intercept `Pipeline` and perform role validation.

`interceptPipeline` also requires setting the positioning of `Pipeline` passed as an argument. In the above code, we are positioning it as "After Authentication". I think there are various ways to do the logic after that, so I will omit it here.

In addition, in the case of `Feature`, configure settings so that `RoleBaseAuthorizer` can be used as an independent module. It's a polite thing to do, just give it a name and return the instance.

Once you have completed the implementation up to this point, you will be finished creating the module related to authorization. However, there are two possible ways to handle `interceptPipeline` when the user's role does not allow access to the API.

### Return a response and exit

The first possible method is to return an appropriate response and end the process there. In this case, you can implement it as follows.

```kotlin
if (roles.none { role }) {
    // Return a response
    call.respond(
        status = HttpStatusCode.Forbidden,
        message = "permission not granted: $role"
    )

    // End the pipeline
    finish()
}
```

The thing to note here is that `Pipeline` does not end just by returning a response. If you want to return a response and stop processing, be sure to call `finish()` and terminate `Pipeline`.

### throw an exception

Another way is to throw an exception. For example:

```kotlin
// Exception for unauthorized access
class AuthorizationException(override val message: String) : Exception(message)

if (roles.none { role }) {
    throw AuthorizationException("permission not granted: $role")
}
```

If you throw an exception, `Pipeline` processing will naturally stop, but this is not a good idea as the exception will also appear in the application log. Fortunately, Ktor also has a feature similar to Spring's `ExceptionHandler`, which can be used to properly handle exceptions. For example, you can do something like the following.

```kotlin
// Response for unauthorized access
data class AuthFailedResponse(val reason: String)

// Exception handling
install(StatusPages) {
    exception<Throwable> { cause ->
        when (cause) {
            // Authorization-related handling
            is AuthorizationException -> {
                call.respond(
                    status = HttpStatusCode.Forbidden,
                    message = AuthFailedResponse(
                        reason = cause.message
                    )
                )
            }
        }
    }
}
```

This will make the application log cleaner, and you will be able to handle other exception handling simply by increasing the branches of `when`.

## lastly

I ended up dealing with a lot more content than I had originally expected, so I feel like my explanations are more general than usual, but I think I've been able to explain a lot about `Pipeline` and implementing self-made modules that apply it. So, if you apply this, I think it won't be that difficult to add other modules. I feel like if I dig deeper, I'll come up with a lot more, but I'll save that for another post when I have a chance. (Honestly, I don't know much about it...)

Personally, I learned about this while creating the Role-based Authorization function. I thought the concept of handling a series of processes in units of `Pipeline` was quite new and good. You can intercept processing in Spring, but I feel like you can do a lot more if you can handle the flow of processing itself as a unit. I've only just started using Ktor, so I think I'll need to spend more time researching the details.

What is certain is that Ktor is a pretty attractive framework. At first, the authentication lacked various functions compared to existing well-known frameworks such as Spring, but I feel like if I could easily create modules like this, there might not be any problems. Of course, there are still many certifications that need to be verified in order to create something at a production level.

See you soon!
