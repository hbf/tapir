# Common server options

## Status codes

By default, successful responses are returned with the `200 OK` status code, and errors with `400 Bad Request`. However,
this can be customised when interpreting an endpoint as a directive/route, by providing implicit values of 
`type StatusMapper[T] = T => StatusCode`, where `type StatusCode = Int`.

This can be especially useful for error responses, in which case having an `Endpoint[I, E, O, S]`, you'd need to provide
an implicit `StatusMapper[E]`.
  
## Server options

Each interpreter accepts an implicit options value, which contains configuration values for:

* how to create a file (when receiving a response that is mapped to a file, or when reading a file-mapped multipart 
  part)
* how to handle decode failures  
  
### Handling decode failures

Quite often user input will be malformed and decoding will fail. Should the request be completed with a 
`400 Bad Request` response, or should the request be forwarded to another endpoint? By default, tapir follows OpenAPI 
conventions, that an endpoint is uniquely identified by the method and served path. That's why:

* an "endpoint doesn't match" result is returned if the request method or path doesn't match. The http library should
  attempt to serve this request with the next endpoint.
* otherwise, we assume that this is the correct endpoint to serve the request, but the parameters are somehow 
  malformed. A `400 Bad Request` response is returned if a query parameter, header or body is missing / decoding fails, 
  or if the decoding a path capture fails with an error (but not a "missing" decode result).

This can be customised by providing an implicit instance of `tapir.server.DecodeFailureHandler`, which basing on the 
request,  failing input and failure description can decide, whether to return a "no match", an endpoint-specific error 
value,  or a specific response.

Only the first failure is passed to the `DecodeFailureHandler`. Inputs are decoded in the following order: method, 
path, query, header, body.

## Extracting common route logic

Quite often, especially for [authentication](../endpoint/auth.html), some part of the route logic is shared among multiple 
endpoints. However, these functions don't compose in a straightforward way, as the results can both contain errors
(represented as `Either`s), and are wrapped in a container. Suppose you have the following methods:

```scala
type AuthToken = String

def authFn(token: AuthToken): Future[Either[ErrorInfo, User]]
def logicFn(user: User, data: String, limit: Int): Future[Either[ErrorInfo, Result]]
```

which you'd like to apply to an endpoint with type:

```scala
val myEndpoint: Endpoint[(AuthToken, String, Int), ErrorInfo, Result, Nothing] = ...
```

To avoid composing these functions by hand, tapir defines a helper extension method, `composeRight`. If the first 
function returns an error, that error is propagated to the final result; otherwise, the result is passed as input to 
the second function.

This extension method is defined in the same traits as the route interpreters, both for `Future` (in the akka-http
interpreter) and for an arbitrary monad (in the http4s interpreter), so importing the package is sufficient to use it:

```scala
import tapir.server.akkahttp._
val r: Route = myEndpoint.toRoute((authFn _).composeRight(logicFn _))
```