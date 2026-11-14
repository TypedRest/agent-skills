# Endpoint types and hand-written clients

Use this when there is no OpenAPI document, or when adding endpoints by hand to a generated client.

## Choosing an endpoint type

Map each resource of the API to the closest type. Derive from a type and add methods when a resource does a little more than the type offers.

| Resource behaves like                                              | Endpoint type        | Main operations                                         |
| ------------------------------------------------------------------ | -------------------- | ------------------------------------------------------- |
| The API root                                                       | Entry                | holds configuration and the top-level children          |
| A single resource: `GET`, `PUT`, `PATCH`, `DELETE`                 | Element              | read, set, merge, update (read-modify-write), delete, exists |
| A list with `GET` (all), `POST` (create) and `/{id}` children      | Collection           | read all, read range, create, create all, set all, get by ID |
| Only `/{id}` children, no listing                                  | Indexer              | get by ID                                               |
| `POST` without request or response body                            | Action               | invoke                                                  |
| `POST` with a request body only                                    | Consumer             | invoke(entity)                                          |
| `POST` with a response body only                                   | Producer             | invoke() returning an entity                            |
| `POST` with both                                                   | Function             | invoke(entity) returning an entity                      |
| Binary content: `GET` to download, `PUT` to upload                 | Blob                 | download, upload from                                   |
| `POST` of binary content (raw body or multipart form field)        | Upload               | upload from                                             |
| A resource whose state should be watched by polling                | Polling              | element operations plus a stream of states              |
| A long-lived response delivering entities split by a separator     | Streaming            | stream of entities                                      |
| Server-Sent Events                                                 | SSE streaming        | stream of entities, optional event type filter, reconnect |
| A collection that can also be observed as an append-only stream    | Streaming collection | collection operations plus a stream                     |

A path segment that has no operations of its own but has children (e.g. `/admin/...`) becomes a small class deriving from the plain endpoint base (`EndpointBase` in C#, `AbstractEndpoint` on the JVM, `Endpoint` in TypeScript) that only exposes child properties. The simpler alternative is to fold the segment into the children's relative URIs (`"./admin/users"`).

Details per type: https://typedrest.net/endpoints/

## Structure of a hand-written client

1. One class deriving from the entry endpoint, named `<Service>Client`.
2. One property/getter per top-level resource, returning a new endpoint with `this` as referrer and a `./`-prefixed relative URI.
3. A derived element endpoint class wherever an element has children or needs extra methods; tell the collection to use it.
4. DTO/entity classes matching the JSON, annotated for the serializer in use.

Use the built-in types directly where no children or extra methods are needed (`CollectionEndpoint<Contact>`); do not create empty subclasses.

## C#

Namespaces: `TypedRest.Endpoints`, `TypedRest.Endpoints.Generic`, `TypedRest.Endpoints.Rpc`, `TypedRest.Endpoints.Raw`, `TypedRest.Endpoints.Reactive` (package `TypedRest.Reactive`).

```csharp
public class AddressBookClient : EntryEndpoint
{
    public AddressBookClient(Uri uri) : base(uri) {}                      // direct use
    public AddressBookClient(HttpClient httpClient) : base(httpClient) {} // AddTypedRest() and tests

    public CollectionEndpoint<Contact, ContactEndpoint> Contacts => new(this, relativeUri: "./contacts");
}

public class ContactEndpoint(IEndpoint referrer, Uri relativeUri)
    : ElementEndpoint<Contact>(referrer, relativeUri)
{
    public ElementEndpoint<Note> Note => new(this, relativeUri: "./note");
    public ActionEndpoint Poke => new(this, relativeUri: "./poke");
    public BlobEndpoint Picture => new(this, relativeUri: "./picture");
}
```

- A custom element endpoint needs a public constructor taking `(IEndpoint, Uri)` (or `(IEndpoint, string)`); the collection creates elements through it.
- Constructors: `ElementEndpoint<T>(referrer, relativeUri)`, `CollectionEndpoint<T>` / `CollectionEndpoint<T, TElementEndpoint>`, `IndexerEndpoint<TElementEndpoint>`, `ActionEndpoint`, `ConsumerEndpoint<T>`, `ProducerEndpoint<TResult>`, `FunctionEndpoint<T, TResult>`, `BlobEndpoint`, `UploadEndpoint`, `PollingEndpoint<T>(referrer, relativeUri, endCondition: ...)`, `StreamingEndpoint<T>`, `SseStreamingEndpoint<T>(referrer, relativeUri, eventType: ...)`, `StreamingCollectionEndpoint<T>`.
- Methods are async with an optional `CancellationToken`: `ReadAsync`, `SetAsync`, `MergeAsync`, `UpdateAsync`, `DeleteAsync`, `ExistsAsync`, `ReadAllAsync`, `ReadRangeAsync`, `CreateAsync`, `InvokeAsync`, `DownloadAsync`, `UploadFromAsync`, `GetObservable()`. Elements are addressed with an indexer: `Contacts["1337"]`.
- **For a library, expose interfaces** so consumers can mock the client: every type has one (`IElementEndpoint<T>`, `ICollectionEndpoint<T, TElementEndpoint>`, `IActionEndpoint`, `IBlobEndpoint`, ...). Declare properties with the interface type and give your own classes interfaces too:

  ```csharp
  public interface IContactEndpoint : IElementEndpoint<Contact>
  {
      IElementEndpoint<Note> Note { get; }
      IActionEndpoint Poke { get; }
  }

  public interface IAddressBookClient : IEndpoint
  {
      ICollectionEndpoint<Contact, IContactEndpoint> Contacts { get; }
  }
  ```

  `AddressBookClient.Contacts` still creates a `CollectionEndpoint<Contact, ContactEndpoint>`; only its declared type changes.
- Default serializer is Newtonsoft.Json; annotate wire names with `[JsonProperty]`. For System.Text.Json add `TypedRest.SystemTextJson`, pass `serializer: new SystemTextJsonSerializer()` and use `[JsonPropertyName]`.
- If the server answers a create without a `Location` header, mark the entity's ID property with `[Key]` (`System.ComponentModel.DataAnnotations`) so TypedRest can find the new element's ID in the response body.

## Kotlin

Packages: `net.typedrest.endpoints`, `.endpoints.generic`, `.endpoints.rpc`, `.endpoints.raw`, `.endpoints.reactive` (artifact `typedrest-reactive`). Each type is an interface plus an `Impl` class; expose the interface, instantiate the `Impl`.

```kotlin
class AddressBookClient(uri: URI) : EntryEndpoint(uri) {
    val contacts: GenericCollectionEndpoint<Contact, ContactEndpoint>
        get() = GenericCollectionEndpointImpl(this, "./contacts", Contact::class.java, ::ContactEndpoint)
}

class ContactEndpoint(referrer: Endpoint, relativeUri: URI) :
    ElementEndpointImpl<Contact>(referrer, relativeUri, Contact::class.java) {
    val note: ElementEndpoint<Note>
        get() = ElementEndpointImpl(this, "./note", Note::class.java)

    val poke: ActionEndpoint
        get() = ActionEndpointImpl(this, "./poke")
}
```

- Entity classes are passed as `Class` objects (`Contact::class.java`).
- `CollectionEndpointImpl(referrer, uri, Contact::class.java)` for plain elements; `GenericCollectionEndpointImpl(..., ::ContactEndpoint)` for a custom element type; `IndexerEndpointImpl(referrer, uri, ::UserEndpoint)`.
- Others: `ConsumerEndpointImpl`, `ProducerEndpointImpl`, `FunctionEndpointImpl(referrer, uri, In::class.java, Out::class.java)`, `BlobEndpointImpl`, `UploadEndpointImpl`, `PollingEndpointImpl(referrer, uri, T::class.java) { endCondition }`, `StreamingEndpointImpl`, `SseStreamingEndpointImpl(referrer, uri, T::class.java, eventType = ...)`.
- Calls are blocking: `read()`, `set()`, `merge()`, `update { }`, `delete()`, `exists()`, `readAll()`, `readRange()`, `create()`, `invoke()`, `download()`, `uploadFrom()`, `getObservable()`. Elements: `contacts["1337"]`.
- Default serializer is kotlinx.serialization: mark entities `@Serializable` and add the serialization Gradle plugin plus `kotlinx-serialization-json`. Use `data class`es.

## Java

Same library and packages as Kotlin. Java must pass a serializer explicitly, because the default (kotlinx.serialization) cannot handle Java classes: add `net.typedrest:typedrest-serializers-jackson` and pass `new JacksonJsonSerializer()`.

```java
public class AddressBookClient extends EntryEndpoint {
    public AddressBookClient(URI uri) {
        super(uri, (HttpCredentials) null, new JacksonJsonSerializer());
    }

    public GenericCollectionEndpoint<Contact, ContactEndpoint> getContacts() {
        return new GenericCollectionEndpointImpl<>(this, "./contacts", Contact.class, ContactEndpoint::new);
    }
}

public class ContactEndpoint extends ElementEndpointImpl<Contact> {
    public ContactEndpoint(Endpoint referrer, URI relativeUri) {
        super(referrer, relativeUri, Contact.class);
    }

    public ElementEndpoint<Note> getNote() {
        return new ElementEndpointImpl<>(this, "./note", Note.class);
    }

    public ActionEndpoint getPoke() {
        return new ActionEndpointImpl(this, "./poke");
    }
}
```

- The `(HttpCredentials) null` cast disambiguates between the credentials and `OkHttpClient` constructor overloads.
- Kotlin default arguments are not available from Java for every parameter; trailing ones may have to be passed explicitly, e.g. `new UploadEndpointImpl(client, "./attachments", null)`, `new PollingEndpointImpl<>(client, "./status", Status.class, null)`, `new SseStreamingEndpointImpl<>(client, "./events", Event.class, null)`.
- Children are getters: `client.getContacts().get("1337").getNote().read()`.

## TypeScript

Imports: `typedrest/endpoints` (entry), `typedrest/endpoints/generic`, `typedrest/endpoints/rpc`, `typedrest/endpoints/raw`, `typedrest/endpoints/reactive`, `typedrest/errors`, `typedrest/http`.

```typescript
import { EntryEndpoint } from "typedrest/endpoints";
import { ElementEndpoint, GenericCollectionEndpoint } from "typedrest/endpoints/generic";
import { ActionEndpoint } from "typedrest/endpoints/rpc";

export class AddressBookClient extends EntryEndpoint {
    get contacts() {
        return new GenericCollectionEndpoint<Contact, ContactEndpoint>(this, "./contacts", ContactEndpoint);
    }
}

export class ContactEndpoint extends ElementEndpoint<Contact> {
    get note() {
        return new ElementEndpoint<Note>(this, "./note");
    }

    get poke() {
        return new ActionEndpoint(this, "./poke");
    }
}
```

- `CollectionEndpoint<T>(referrer, uri)` for plain elements; `GenericCollectionEndpoint<T, TElement>(referrer, uri, ElementClass)` for a custom element type; `IndexerEndpoint<TElement>`.
- Others: `ConsumerEndpoint`, `ProducerEndpoint`, `FunctionEndpoint<In, Out>`, `BlobEndpoint`, `UploadEndpoint`, `PollingEndpoint<T>(referrer, uri, endCondition?)`, `StreamingEndpoint`, `SseStreamingEndpoint<T>(referrer, uri, eventType?)`, `StreamingCollectionEndpoint`.
- Methods return promises and take an optional `AbortSignal` (`read(signal)`, `set(entity, signal)`, ...): `read()`, `set()`, `merge()` (takes `Partial<T>`), `update()`, `delete()`, `exists()`, `readAll()`, `readRange()`, `create()`, `invoke()`, `download()`, `uploadFrom()`. Elements: `contacts.get("1337")`. Reactive endpoints provide an `AsyncIterable` via `stream()` (`for await`), not observables.
- There are no endpoint interfaces; DTOs are plain `interface`s whose members must use the exact wire names, because responses are parsed with `JSON.parse()` and cast.

## Things to get right

- **Relative URIs**: always `./child`. See https://typedrest.net/link-handling/relative-uris/.
- **Create result**: nullable endpoint for the new element; see the rule in SKILL.md.
- **Optimistic concurrency**: element endpoints remember the ETag from a read and send `If-Match` on the following write. The ETag lives in the endpoint object that read, so a write through a newly created object (e.g. `client.Contacts["1337"]` accessed again) sends no `If-Match`. Keep the object for the read-modify-write cycle. The update operation wraps read-modify-write and retries on a conflict. Merge sends no `If-Match` at all. See https://typedrest.net/tutorials/concurrency/.
- **Shortcuts**: collections offer `contains`, `set`, `merge` and `delete` taking an entity or ID; indexers offer `contains(id)` and `delete(id)` (only for element children). In TypeScript they take an optional `signal` and `merge` accepts `Partial<TEntity>`.
- **Paging**: `ReadRangeAsync`/`readRange` use HTTP `Range` headers. APIs that page with query parameters need custom code; see https://typedrest.net/tutorials/paging/.
- **Polling change detection**: .NET and JVM compare states with `Equals`/`equals`, so polled entities need value equality (C# `record`, Kotlin `data class`, overridden `equals` in Java).
- **Streams and timeouts (JVM)**: avoid OkHttp `callTimeout()` with streaming/SSE endpoints; it cuts off long-lived responses. `readTimeout()` limits the gap between two reads.
- **Allowed operations**: endpoints record the `Allow` header of responses and expose it as nullable flags (`DeleteAllowed`, `isDeleteAllowed`, `deleteAllowed`, ...), which is useful for UI code; see https://typedrest.net/endpoints/#allowed-methods.
