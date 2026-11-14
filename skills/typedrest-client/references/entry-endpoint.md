# Entry endpoint: HTTP client, authentication, error handling

Every endpoint sends its requests through the HTTP client of the entry endpoint it descends from, and inherits its serializer, error handler and link handling. All cross-cutting configuration therefore goes in one place.

Entry endpoint constructors:

| Language   | Signatures                                                                                                           |
| ---------- | -------------------------------------------------------------------------------------------------------------------- |
| C#         | `EntryEndpoint(Uri uri, NetworkCredential? credentials, serializer, errorHandler, linkExtractor)` and `EntryEndpoint(HttpClient httpClient, Uri? uri, serializer, errorHandler, linkExtractor)`; everything after the first parameter is optional |
| Kotlin/Java | `EntryEndpoint(uri, credentials, serializer, errorHandler, linkExtractor)` and `EntryEndpoint(uri, httpClient: OkHttpClient, serializer, errorHandler, linkExtractor)`; optional from the second parameter on (Kotlin default arguments, `@JvmOverloads` for Java) |
| TypeScript | `EntryEndpoint(uri: URL \| string, serializer?, errorHandler?, linkExtractor?, httpClient?)`                         |

With a **generated** client, this configuration goes into custom code around the generated entry endpoint (extra or replaced constructors in C#, a subclass elsewhere); see [extending-generated-code.md](extending-generated-code.md). Keep the serializer the DTOs were generated for.

Design question to settle with the user for a library: should the client class take credentials in its constructor (convenient, one obvious way to use it), or take a pre-configured HTTP client (flexible, the consumer owns authentication)? Offering both constructors is common in C#.

## Default headers and timeouts

- **C#**: pass a configured `HttpClient`, or change the entry endpoint's `HttpClient` property (`client.HttpClient.DefaultRequestHeaders.Add(...)`). When the URI is omitted, the `HttpClient`'s `BaseAddress` is used.
- **Kotlin/Java**: pass an `OkHttpClient` built with timeouts and an interceptor that sets the headers.
- **TypeScript**: `client.httpClient.defaultHeaders.set(...)`.

Examples: https://typedrest.net/endpoints/entry/#http-client

## Authentication

| Scheme                          | C#                                                             | Kotlin / Java                                              | TypeScript                                              |
| ------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------- |
| HTTP Basic                      | `new EntryEndpoint(uri, new NetworkCredential(user, password))` | `EntryEndpoint(uri, HttpCredentials(user, password))`      | default `Authorization: Basic ...` header               |
| Basic via URI                   | `https://user:password@host/` is picked up                      | same                                                       | not supported; set the header                           |
| API key / static bearer token   | `HttpClient.DefaultRequestHeaders`                              | OkHttp interceptor                                         | `httpClient.defaultHeaders`                             |
| OAuth 2.0 client credentials    | `TypedRest.OAuth` package: `OAuthHandler` + `OAuthOptions`      | no built-in support: OkHttp interceptor that caches and refreshes the token | no built-in support: class derived from `FetchHttpClient` passed as fifth constructor argument |

In C#, wrap an `HttpClient` around an `OAuthHandler` and pass it to the client's `HttpClient` constructor. The handler discovers the token endpoint via OpenID Connect discovery, reuses the token until shortly before expiry and retries once on `401`.

For the JVM interceptor and the TypeScript `FetchHttpClient` subclass, start from the complete examples at https://typedrest.net/tutorials/authentication/ rather than writing them from scratch. A retry after `401` resends the request body, which does not work for bodies that can be read only once (stream uploads); skip the retry for those.

Never hard-code secrets in the client library or in tests against real services; take them as constructor parameters or from configuration.

## Dependency injection (C#)

`services.AddTypedRest<IMyClient, MyClient>(uri)` registers the entry endpoint as a typed HTTP client created by `IHttpClientFactory`. What the client library has to provide for this is a constructor taking an `HttpClient`; with a generated client, add it in a partial class.

Registration, `.AddBasicAuth()`, `.AddOAuthHandler()` and lifetime rules are the consumer's side: https://typedrest.net/tutorials/dependency-injection/

## Error handling

Non-success responses are mapped to exceptions by the entry endpoint's error handler. If the response body is JSON with a `message` or `details` field, that text becomes the exception message.

| HTTP status               | .NET                          | Java/Kotlin               | TypeScript                 |
| ------------------------- | ----------------------------- | ------------------------- | -------------------------- |
| 400 Bad Request           | `InvalidDataException`        | `BadRequestException`     | `BadRequestError`          |
| 401 Unauthorized          | `AuthenticationException`     | `AuthenticationException` | `AuthenticationError`      |
| 403 Forbidden             | `UnauthorizedAccessException` | `AuthorizationException`  | `AuthorizationError`       |
| 404 Not Found, 410 Gone   | `KeyNotFoundException`        | `NotFoundException`       | `NotFoundError`            |
| 408 Request Timeout       | `TimeoutException`            | `TimeoutException`        | `TimeoutError`             |
| 409 Conflict              | `InvalidOperationException`   | `ConflictException`       | `ConflictError`            |
| 412 Precondition Failed   | `InvalidOperationException`   | `ConcurrencyException`    | `ConcurrencyError`         |
| 416 Range Not Satisfiable | `InvalidOperationException`   | `ConflictException`       | `RangeNotSatisfiableError` |
| other                     | `HttpRequestException`        | `HttpException`           | `HttpError`                |

Document the exceptions a consumer should expect in the client library's README or doc comments; they are part of its contract.

### Custom error handlers

Write one when the API reports errors in its own body format (error codes, problem details, validation lists) and consumers should get typed exceptions or the structured details.

- **C#**: derive from `DefaultErrorHandler` and override `MapException(HttpStatusCode statusCode, string message)`, or implement `IErrorHandler.HandleAsync(HttpResponseMessage)` to read the body yourself. Pass it as `errorHandler:` to the entry endpoint's base constructor. Keep exceptions for `409`/`412` derived from `InvalidOperationException`, because `UpdateAsync` retries only on that type.
- **Kotlin/Java**: derive from `DefaultErrorHandler` and override `mapException(status, message, response)`, or implement `ErrorHandler.handle(response: okhttp3.Response)`. Pass it as the `errorHandler` constructor argument.
- **TypeScript**: implement `ErrorHandler` (`handle(response: Response): Promise<void>`), or derive from `DefaultErrorHandler`, and pass it as the third constructor argument. `HttpError` carries `status` and the parsed body in `data`.

A handler must do nothing for success responses and must throw for everything else. Keep the default mapping for statuses you do not treat specially, so that built-in behavior such as retry-on-conflict keeps working.

With a generated C# client, a custom error handler requires `GenerateEntryConstructor="false"` (or `--generate-entry-constructor false`) and a hand-written constructor.
