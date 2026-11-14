---
name: typedrest-client
description: Build a REST API client library with TypedRest in C#, Java, Kotlin or TypeScript. Use when the user wants to create, generate, extend or test a client for a REST/HTTP API with TypedRest, mentions TypedRest endpoints (EntryEndpoint, CollectionEndpoint, ElementEndpoint, ...), typedrest-codegen or TypedRest.SourceGenerator, or wants a typed client from an OpenAPI/Swagger document.
---

# Building client libraries with TypedRest

TypedRest (https://typedrest.net/) models a REST API as a tree of endpoint objects that mirrors the API's URL structure. You declare which pattern each resource follows (collection, element, action, blob, stream, ...) and inherit the behavior: `client.Contacts["1337"].Note.SetAsync(note)` instead of hand-built URLs and verbs.

There are three ports with the same concepts and different idioms:

| Language      | Package                                   | API docs                          |
| ------------- | ----------------------------------------- | --------------------------------- |
| C#            | NuGet `TypedRest`                         | https://dotnet.typedrest.net/     |
| Java / Kotlin | Maven `net.typedrest:typedrest`           | https://java.typedrest.net/       |
| TypeScript    | npm `typedrest`                           | https://typescript.typedrest.net/ |

## Workflow

### 1. Establish the facts

Find out from the project before asking the user:

- **Language**: C#, Java, Kotlin or TypeScript. Take it from the existing project.
- **Is there an OpenAPI/Swagger document?** Search the project for `*.yml`, `*.yaml` and `*.json` files containing `openapi:` or `swagger:`, and check whether the user mentioned one. Do not go hunting for a spec elsewhere and do not write one yourself; if none is at hand, go straight to a hand-written client.
- **Is there a project to put the client in?** If not, see [scaffolding.md](references/scaffolding.md).
- **Service name** (the entry endpoint becomes `<ServiceName>Client`) and **namespace/package**. Derive them from the spec title and the project; ask only if the result is not obvious.

### 2. Pick the path

| Situation                              | Path                                                                                   |
| -------------------------------------- | -------------------------------------------------------------------------------------- |
| Spec available, C#                     | **Roslyn source generator** (`TypedRest.SourceGenerator`).                             |
| Spec available, Java/Kotlin/TypeScript | **CLI** (`typedrest-codegen`), with regeneration wired into the build.                 |
| Spec available, C#, but the user wants the generated code checked in | CLI, as a fallback.                                      |
| No spec                                | **Hand-written client**; see [endpoints.md](references/endpoints.md).                  |

When a spec is available, lean towards code generation even if the user only asked for "a client". Say that you are doing so and why (the client stays in sync with the spec). Hand-write instead only if the user declines.

### 3. Get code generation working first

Follow [code-generation.md](references/code-generation.md). In short:

1. Run `typedrest-codegen pattern` **once** to see what the generator infers, and compare it against what the API actually does.
2. Fix wrong inferences by editing the `x-typedrest` extension in the spec. Streaming, SSE and polling endpoints are never inferred and always need this.
3. Generate: source generator for C#, CLI for the others.
4. Add the runtime dependencies the generated code needs.
5. **Build the project.** Do not move on until it compiles and you have shown the user a short usage snippet against the generated client.
6. For CLI-based projects, wire regeneration into the build so client and spec cannot drift apart.

### 4. Then build what the generator does not cover

Only after generation works, go through the gaps with the user. Typical ones:

- Convenience methods that combine or filter calls (`FindByName`, `RenameAsync`).
- Query parameters, custom headers, non-standard verbs or other operations no endpoint type models.
- Authentication, default headers, timeouts, custom error mapping on the entry endpoint.
- Value equality on DTOs used with polling endpoints.

**Never edit generated files**; they are overwritten. Each language has its own extension mechanism (partial classes in C#, subclassing in Java/TypeScript, subclassing or extension functions in Kotlin). See [extending-generated-code.md](references/extending-generated-code.md), and [entry-endpoint.md](references/entry-endpoint.md) for authentication and error handling.

Prefer fixing structural problems in `x-typedrest` (wrong endpoint kind, wrong schema, missing child) over papering over them in custom code.

### 5. Test

Add tests for the custom code and one request-level test per hand-written child endpoint; see [testing.md](references/testing.md). Generated endpoints do not need a test each: their behavior comes from TypedRest and the generator.

## Rules that hold everywhere

- **Prefix relative URIs with `./`**, e.g. `"./contacts"`. Without it, URI resolution replaces the parent's last path segment when the parent has no trailing slash.
- **Endpoints are cheap.** Expose children as properties/getters that create a new endpoint object on each access. Building endpoints sends no request. The ETag for optimistic concurrency lives in the endpoint object, though: keep the object across a read-modify-write.
- **Everything is configured on the entry endpoint**: HTTP client, serializer, error handler, link handling. Children inherit through the `referrer` constructor parameter, so always pass `this`.
- **Serializer and DTO annotations must match.** A DTO annotated for one serializer silently falls back to member names under another, changing the wire format without an error. The generated entry constructor takes care of this; a hand-written constructor must pass the same serializer.
- **Creating an element returns an endpoint, not the entity**, and it is nullable: TypedRest takes the URI from the `Location` header or the ID in the response body, and returns null/undefined if the server sends neither.
- **Look up current package versions** (NuGet, Maven Central, npm) instead of guessing them.
- **Verify names against the installed library** when something does not compile. The reference files here are condensed; the API docs linked above and the package sources are authoritative.

## Reference files

| File                                                                    | Read it when                                                             |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| [code-generation.md](references/code-generation.md)                     | A spec is available: inference, `x-typedrest`, source generator, CLI, build wiring. |
| [extending-generated-code.md](references/extending-generated-code.md)   | Adding custom code on top of a generated client.                         |
| [endpoints.md](references/endpoints.md)                                 | Hand-writing endpoints, or choosing an endpoint type, in any language.   |
| [entry-endpoint.md](references/entry-endpoint.md)                       | Authentication, headers, timeouts, error handling, dependency injection. |
| [testing.md](references/testing.md)                                     | Writing tests for the client or for code that uses it.                   |
| [scaffolding.md](references/scaffolding.md)                             | No project exists yet for the client library.                            |
| [tanstack-query.md](references/tanstack-query.md)                       | A TypeScript UI app (React, Solid, Vue, Svelte, Angular) uses the client, especially with TanStack Query, or needs edit forms. |

Online documentation for details that are not covered here:

- Tutorials: https://typedrest.net/tutorials/
- Endpoint types: https://typedrest.net/endpoints/
- Code generation: https://typedrest.net/code-generation/
- Serializers: https://typedrest.net/serializers/
- Link handling (URI templates, Link headers, HAL): https://typedrest.net/link-handling/
