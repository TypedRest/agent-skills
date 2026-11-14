# Testing a TypedRest client

Three things are worth testing. Use the test framework the project already has.

1. **The requests the client sends**: does `Contacts["1337"].Poke` go to `POST /contacts/1337/poke`? Use a mock HTTP server.
2. **Code that uses the client**: replace the client with a mock; no HTTP involved.
3. **Client and service together** (only if the user also owns the service): host the service in the test process and point the client at it. See https://typedrest.net/testing/integration/.

What to cover in a client library:

- Hand-written child endpoints: one request-level test each (URI and verb). Not every method of every endpoint; that behavior is TypedRest's.
- Generated endpoints: no test each. One smoke test through the generated client is enough to catch a broken setup (serializer mismatch, wrong base URI handling).
- Custom code: every custom method, the entry endpoint's authentication (is the header sent?), and custom error mapping (does this body produce that exception?).
- At least one test that round-trips a DTO with non-trivial wire names, to pin the serializer configuration.

## Mock HTTP server

| Language    | Tool                                                  | How the client is pointed at it                                   |
| ----------- | ----------------------------------------------------- | ----------------------------------------------------------------- |
| C#          | `RichardSzalay.MockHttp`                              | `new MyClient(new HttpClient(mock) { BaseAddress = ... })`; needs an `HttpClient` constructor on the client |
| Kotlin/Java | OkHttp `MockWebServer`                                | `MyClient(server.url("/").toUri())` (Java: `.uri()`)              |
| TypeScript  | Vitest with `vitest-fetch-mock` (or the `fetch` mocking of the project's test runner) | global `fetch` is replaced        |

The core of a request-level test in each language. Complete test classes with setup and teardown, request body and header checks, and a conflict (`412`) scenario are at https://typedrest.net/tutorials/testing/ and https://typedrest.net/testing/endpoints/.

```csharp
var mock = new MockHttpMessageHandler();
var client = new MyClient(new HttpClient(mock) { BaseAddress = new Uri("http://localhost/") });

mock.Expect(HttpMethod.Post, "http://localhost/contacts/1337/poke").Respond(HttpStatusCode.NoContent);
await client.Contacts["1337"].Poke.InvokeAsync();
mock.VerifyNoOutstandingExpectation();
```

```kotlin
val server = MockWebServer().apply { start() }
val client = MyClient(server.url("/").toUri())

server.enqueue(MockResponse().setResponseCode(204))
client.contacts["1337"].poke.invoke()

val request = server.takeRequest()
assertEquals("POST", request.method)
assertEquals("/contacts/1337/poke", request.path)
```

```java
server.enqueue(new MockResponse().setResponseCode(204));
client.getContacts().get("1337").getPoke().invoke();

RecordedRequest request = server.takeRequest();
assertEquals("POST", request.getMethod());
assertEquals("/contacts/1337/poke", request.getPath());
```

```typescript
fetchMock.mockResponseOnce({ status: 204 });
await client.contacts.get("1337").poke.invoke();

const [request] = fetchMock.requests();
expect(request.url).toBe("http://localhost/contacts/1337/poke");
expect(request.method).toBe("POST");
```

Notes:

- C#: with a generated client, add the `HttpClient` constructor in a partial class first; see [extending-generated-code.md](extending-generated-code.md#entry-endpoint-constructors).
- JVM: the `MockWebServer` API differs between OkHttp major versions (`MockResponse` builder, `close()` vs. `shutdown()`); match the version the project resolves.
- TypeScript: call `createFetchMock(vi).enableMocks()` in a Vitest setup file (this exposes the global `fetchMock`) and set `mockReset: true` in the Vitest config; `fetchMock.resetMocks()` alone leaves unused `mockOnceIf()` responses queued for the next test. For responses without a body pass a response object such as `{ status: 404 }`: `mockOnceIf(url, "", { status: 404 })` silently registers nothing (the request gets `200 OK`), and `Response` rejects a `204` with a `""` body. If the project already uses another test runner, use its `fetch` mocking instead of adding Vitest.
- Error responses: respond with an error status and assert the mapped exception from the table in [entry-endpoint.md](entry-endpoint.md#error-handling), e.g. `404` → `KeyNotFoundException` / `NotFoundException` / `NotFoundError`.

## Mocking the client in consumer tests

This is about making the client library **mockable** for its users, which is a design decision to take while building it.

- **C#**: Moq and similar libraries can only mock interfaces and virtual members, so the whole chain must be interfaces: `IMyClient` exposing `ICollectionEndpoint<Contact, IContactEndpoint>`, and so on. The source generator emits these by default; the CLI with `--generate-interfaces`. For hand-written clients see [endpoints.md](endpoints.md#c). Then one expression sets up the chain:

  ```csharp
  var client = new Mock<IMyClient>();
  client.Setup(x => x.Contacts["1337"].Poke.InvokeAsync(It.IsAny<CancellationToken>()))
        .Returns(Task.CompletedTask);
  ```

- **Kotlin/Java**: Mockito (or mockito-kotlin) can mock the classes; stub each level of the chain (`client.contacts` → `contacts["1337"]` → `contact.poke`). Generating interfaces is optional.
- **TypeScript**: any object with the members the code under test uses can stand in, built from `vi.fn()`s and cast with `as unknown as MyClient`.

Details: https://typedrest.net/testing/mocking/
