# Scaffolding a client library project

Use this only when no project exists for the client yet. If the repository already has projects, follow their conventions (target frameworks, build tool, test framework, directory layout, central package management) instead of the defaults here.

Before creating anything, settle with the user, in one question round:

- Library name and namespace/package (e.g. `MyCompany.MyService.Client`, `com.mycompany.myservice`).
- Whether it will be **published** (NuGet, Maven repository, npm registry) or only used inside the repository. Publishing needs package metadata; do not invent license, author or repository URLs, ask for them or leave them out.
- For the JVM: Gradle or Maven, and Kotlin or Java.

Look up the current version of every dependency in its registry; do not write versions from memory.

Layout that works for every language: the spec file inside the project directory, generated code in its own directory/package (CLI-based languages), custom code next to it but separate, and a test project/source set.

## C#

```bash
dotnet new classlib -n MyCompany.MyService.Client
dotnet new xunit -n MyCompany.MyService.Client.Tests   # or the test framework the user prefers
```

With an OpenAPI document (source generator):

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework> <!-- match the consumers; check what TypedRest supports -->
    <Nullable>enable</Nullable>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="TypedRest" Version="..." />
    <PackageReference Include="TypedRest.SourceGenerator" Version="..." PrivateAssets="all" />
  </ItemGroup>

  <ItemGroup>
    <TypedRestOpenApi Include="myservice.yml" ServiceName="MyService" Namespace="MyCompany.MyService.Client" />
  </ItemGroup>

</Project>
```

- `PrivateAssets="all"` keeps the generator from flowing to consumers of the packed library; `TypedRest` itself must flow, since the public API derives from it.
- Add `TypedRest.Reactive`, `TypedRest.SystemTextJson`, `TypedRest.OAuth` only when used.
- For a published package add `PackageId`, `Version`, `Description`, `Authors`, `PackageLicenseExpression`, `RepositoryUrl` and a `PackageReadmeFile`.
- Test project: reference the library, add `RichardSzalay.MockHttp` (and `Moq` if consumer-style tests are wanted).

Without a spec, drop the generator and the `TypedRestOpenApi` item.

## Kotlin (Gradle)

```kotlin
plugins {
    kotlin("jvm") version "..."
    kotlin("plugin.serialization") version "..."   // for the default kotlinx serializer
    `java-library`
    `maven-publish`                                 // only if published
}

dependencies {
    api("net.typedrest:typedrest:...")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:...")

    testImplementation(kotlin("test"))
    testImplementation("com.squareup.okhttp3:mockwebserver:...")
}

tasks.test { useJUnitPlatform() }
```

- `api` for `typedrest`, because the client's public types derive from TypedRest types and consumers need them on their compile classpath.
- Add `net.typedrest:typedrest-reactive` (as `api`) for polling/streaming endpoints, `net.typedrest:typedrest-serializers-jackson` or `-moshi` when not using kotlinx.
- Match the `mockwebserver` version to the OkHttp version that `typedrest` brings in.
- Regeneration task; see [code-generation.md](code-generation.md#6-keep-client-and-spec-in-sync):

  ```kotlin
  tasks.register<Exec>("generateClient") {
      commandLine("dotnet", "typedrest-codegen", "generate", "-l", "kotlin",
          "-f", "myservice.yml", "-o", "src/main/kotlin/", "-s", "MyService",
          "-n", "com.mycompany.myservice", "--generate-dtos")
  }
  ```

## Java (Gradle or Maven)

Same as Kotlin without the Kotlin plugins, and with these dependencies:

- `net.typedrest:typedrest`
- `net.typedrest:typedrest-serializers-jackson`
- `org.jspecify:jspecify` (for generated DTOs)
- test: JUnit, `com.squareup.okhttp3:mockwebserver`, optionally Mockito

Maven coordinates use the same group and artifact IDs:

```xml
<dependency>
    <groupId>net.typedrest</groupId>
    <artifactId>typedrest</artifactId>
    <version>...</version>
</dependency>
```

Generate with `-l java -o src/main/java/`.

## TypeScript

```bash
npm init -y
npm install typedrest
npm install --save-dev typescript
```

- `tsconfig.json`: `strict: true`, `declaration: true`, an `outDir`, and module settings that match how the consumers load the package and how `typedrest` itself is published (check its `package.json` for `type` and `exports`; the subpath imports such as `typedrest/endpoints/generic` need a `moduleResolution` that understands `exports`).
- `package.json`: `main`/`types` or `exports` pointing at the build output, a `build` script, and a `generate` script with the `typedrest-codegen generate -l typescript ...` call. For a published package set `files` so only the build output and README ship.
- Export the client (the subclass, if there is custom code) and the DTO types from the package's entry module.
- Tests: the runner the user prefers, defaulting to Vitest with `vitest-fetch-mock`.

## The .NET tool manifest (all CLI-based projects)

```bash
dotnet new tool-manifest
dotnet tool install typedrest-codegen
```

Commit `.config/dotnet-tools.json`. A fresh checkout or CI job then needs only `dotnet tool restore`.

## Finish

- Build and run the tests once before handing over.
- Write a short README: what the client is for, how to install it, a usage snippet, how to regenerate, and the `x-typedrest` rule if the spec is annotated.
- Do not publish the package, create repositories or push; leave that to the user.
