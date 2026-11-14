# Extending a generated client

Generated code is overwritten on regeneration. Never edit it; add to it from files of your own, kept outside the generated directory or package.

Order of preference when the generated client is not what the user needs:

1. **The structure is wrong** (wrong endpoint kind, wrong schema, missing or misplaced child): fix `x-typedrest` in the spec. See [code-generation.md](code-generation.md#3-fix-the-inference-with-x-typedrest).
2. **Something is missing that an endpoint type cannot express** (convenience methods, query parameters, special requests, entry endpoint configuration): add custom code as described here.

Before writing custom code, look at the generated code to get the exact type names, constructor signatures and whether interfaces exist. For the C# source generator, set `<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>` and read the files under `obj/`.

[endpoints.md](endpoints.md) lists the endpoint types and their constructors for code that adds new child endpoints by hand.

## C#: partial classes

All generated classes and interfaces are `partial` (generated enums are not). Add a file in the same namespace:

```csharp
namespace MyCompany.MyService;

public partial interface IContactElementEndpoint
{
    /// <summary>Changes the name of the contact, retrying if someone else modified it in the meantime.</summary>
    Task<Contact?> RenameAsync(string name, CancellationToken cancellationToken = default);
}

public partial class ContactElementEndpoint
{
    public Task<Contact?> RenameAsync(string name, CancellationToken cancellationToken = default)
        => UpdateAsync(contact => contact.Name = name, cancellationToken: cancellationToken);
}
```

**When interfaces are generated (the source generator default), add the member to the partial interface as well.** Collections hand out their elements as the interface type (`IContactElementEndpoint`), so a method that exists only on the class is not reachable from `client.Contacts["1337"]`.

Partial classes work on every generated endpoint, so custom members can go exactly where they belong in the tree, including new child endpoints the spec does not describe:

```csharp
public partial interface IContactElementEndpoint
{
    IBlobEndpoint Picture { get; }
}

public partial class ContactElementEndpoint
{
    public IBlobEndpoint Picture => new BlobEndpoint(this, relativeUri: "./picture");
}
```

### Entry endpoint constructors

The generated entry endpoint has a constructor taking the base URI that also configures the serializer. Add further constructors in a partial class, for example one taking an `HttpClient` (needed for `AddTypedRest()` dependency injection and for tests with a mock handler):

```csharp
public partial class MyServiceClient
{
    public MyServiceClient(HttpClient httpClient)
        : base(httpClient)
    {}
}
```

A partial class cannot change a constructor that was already generated. If the entry endpoint needs a custom error handler, default headers or mandatory credentials, turn the generated constructor off (`GenerateEntryConstructor="false"` on the `TypedRestOpenApi` item, or `--generate-entry-constructor false` for the CLI) and write it yourself:

```csharp
public partial class MyServiceClient
{
    public MyServiceClient(Uri uri, string token)
        : base(uri, errorHandler: new MyServiceErrorHandler())
    {
        HttpClient.DefaultRequestHeaders.Authorization = new("Bearer", token);
    }
}
```

If the serializer is `system-text-json`, every hand-written constructor must pass `serializer: new SystemTextJsonSerializer()` to `base`; otherwise the DTOs are serialized with Newtonsoft.Json and their `[JsonPropertyName]` attributes are ignored.

### Value equality for polling

Generated C# DTOs are plain classes. A polling endpoint skips states equal to the previous one using `Equals`, so without value equality every poll counts as a change. Add `Equals`/`GetHashCode` in a partial class for DTOs that are polled.

## Kotlin: extension functions and subclassing

Extension functions work on any generated endpoint, at any depth, without subclassing. Prefer them for added methods:

```kotlin
fun MyServiceClient.findByName(name: String) = contacts.readAll().filter { it.name == name }

fun ContactElementEndpoint.rename(name: String) = update { it.copy(name = name) }
```

Generated endpoints are `open class`es, so you can also derive from them (from the `Impl` class if interfaces are generated). Subclass the entry endpoint when you need state or constructor logic, such as a configured `OkHttpClient`:

```kotlin
class MyClient(uri: URI, token: String) : MyServiceClient(uri) {
    // ...
}
```

The generated entry endpoint's constructor takes only the base URI. If that is not enough (custom `OkHttpClient` for headers or tokens, error handler), generate with `--generate-entry-constructor false` and write the constructor in the subclass, passing the same serializer the DTOs were generated for.

Generated Kotlin DTOs are `data class`es, so polling works without extra code.

## Java: subclassing

Derive from the generated class (the `Impl` class if interfaces are generated):

```java
public class MyClient extends MyServiceClient {
    public MyClient(URI uri) {
        super(uri);
    }

    public List<Contact> findByName(String name) {
        return getContacts().readAll().stream()
            .filter(x -> name.equals(x.name))
            .toList();
    }
}
```

Generated Java DTOs have public fields and no value equality; for a polled resource, use a hand-written entity class with `equals`/`hashCode` instead.

## TypeScript: subclassing

```typescript
export class MyClient extends MyServiceClient {
    async findByName(name: string) {
        return (await this.contacts.readAll()).filter(x => x.name === name);
    }
}
```

Export the subclass from the package's public entry point instead of the generated class, so that consumers get the extended client.

## The subclassing limitation (Java, Kotlin, TypeScript)

The generated code creates child endpoints as the generated types. A subclass of a child endpoint is therefore never instantiated by the generated tree, so subclassing is mostly useful for the **entry endpoint**. For helpers that concern a child endpoint:

- Kotlin: use an extension function on the child type.
- Java, TypeScript: put the helper on the entry endpoint subclass and take the ID as a parameter (`renameContact(id, name)`), or override the getter that creates the child in your entry subclass to return your own subclass.
- If a child needs a different shape altogether, that is a structural problem: fix it in `x-typedrest`.

## Requests no endpoint type models

For query parameters, unusual verbs or custom headers on a single call, derive from the closest endpoint type and build the request with the protected members of the endpoint base class (its `HttpClient`, `Uri`, serializer and error handling helpers). Their names differ per port; look them up in the API docs for the language or in the base class of an endpoint in the installed package, and model the code on how the built-in endpoints (e.g. the element endpoint's read) are implemented.

Often a query parameter is better expressed as a child URI: an endpoint for `"./contacts?name=Smith"` is still a readable collection. URI templates and links from response headers or HAL bodies are covered at https://typedrest.net/link-handling/.
