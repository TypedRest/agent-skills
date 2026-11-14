# TypedRest with TanStack Query (React, Solid, Vue, Svelte, Angular)

Use this when a TypeScript UI app consumes a TypedRest client and uses, or should use, TanStack Query (`@tanstack/react-query`, `@tanstack/solid-query`, `@tanstack/vue-query`, `@tanstack/svelte-query`, `@tanstack/angular-query-experimental`).

Package: npm `typedrest-tanstack-query` (peer dependencies `typedrest` and `@tanstack/query-core` ≥ 5.89). Entry points: `typedrest-tanstack-query` (all frameworks) and `typedrest-tanstack-query/react`. **Check `npm view typedrest-tanstack-query` first; until it is published, it only exists in the `query/` folder of TypedRest-TypeScript.** Tutorial: https://typedrest.net/tutorials/tanstack-query/

The helpers return plain option objects. Pass them to the adapter's hooks; in Solid, Svelte and Angular wrap them in a function.

```tsx
// React
const query = useQuery(readQuery(client.contacts.get(id)));   // data: Contact | undefined
const save = useMutation(setMutation(client.contacts.get(id))); // mutate(contact: Contact)

// Solid / Svelte / Angular
const query = useQuery(() => readQuery(client.contacts.get(props.id)));
```

## React context

```tsx
import { createClientContext } from "typedrest-tanstack-query/react";
export const { ClientProvider, useClient } = createClientContext<AddressBookClient>();
// <QueryClientProvider client={queryClient}><ClientProvider client={new AddressBookClient(uri)}>...</ClientProvider></QueryClientProvider>
```

Solid, Vue, Svelte and Angular components run setup once; a module-level client or the framework's own DI is enough.

## Queries

| Helper | `data` |
|---|---|
| `readQuery(element)` | `TEntity` |
| `existsQuery(element)` | `boolean` |
| `readAllQuery(collection)` | `TEntity[]` |
| `readRangeQuery(collection, from?, to?)` | `PartialResponse<TEntity>` |
| `readRangeInfiniteQuery(collection, pageSize)` | pages of `PartialResponse<TEntity>`, for `useInfiniteQuery` |
| `probeQuery(rpcOrBlob)` | snapshot of `*Allowed` flags |
| `readMetaQuery(entry)` | `null` |
| `downloadQuery(blob)` | `Blob` |
| `streamQuery(streaming \| sse, { reducer?, initialValue?, refetchMode? })` | `TEntity[]` or reducer result |
| `streamCollectionQuery(streamingCollection, startIndex?)` | `TEntity[]` |
| `pollQuery(polling)` | latest `TEntity` |

Override options by spreading: `useQuery({ ...readQuery(contact), staleTime: 60_000 })`. TanStack's `signal` is passed into TypedRest. Streams stop when the query is cancelled or unused, default to `staleTime: Infinity`, and are never invalidated automatically. They use TanStack's `experimental_streamedQuery`.

## Mutations

| Helper | `mutate(...)` | Default invalidation |
|---|---|---|
| `setMutation(element, { basedOn? })` | `TEntity` | `lineage` |
| `mergeMutation(element)` | `Partial<TEntity>` | `lineage` |
| `updateMutation(element, { maxRetries? })` | `(entity) => void` | `lineage` |
| `deleteMutation(element, { basedOn? })` | nothing | removes element's queries, then `ancestors` |
| `createMutation(collection)` | `TEntity` (returns the new element endpoint) | `ancestors` |
| `createAllMutation` / `setAllMutation(collection, { basedOn? })` | `TEntity[]` | `subtree` |
| `invokeMutation(action \| consumer \| function \| producer)` | input or nothing | `parent` |
| `uploadMutation(blob \| upload)` | `Blob` | `lineage` / `parent` |

Scopes are relative to the mutated endpoint's URL: `self`, `subtree` (it and below), `ancestors` (it and above), `lineage` (both, not siblings), `parent` (everything below its parent). Override with `{ invalidate: "self" | false, alsoInvalidate: [endpoint] }`. Invalidation runs inside `mutationFn` and is awaited, so spreading your own `onSuccess` is safe: `useMutation({ ...setMutation(contact), onSuccess })`.

TypedRest errors pass through unchanged: check `mutation.error instanceof ConcurrencyError` (from `typedrest/errors`).

## Edit forms

Capture the version when the form opens and save against it, so a background refetch cannot turn a conflict into a silent overwrite:

```tsx
const [basedOn, setBasedOn] = useState(() => versionOf(initial));
const save = useMutation({
    ...setMutation(contact, { basedOn }),
    onSuccess: () => setBasedOn(versionOf(queryClient.getQueryData(readQuery(contact).queryKey))),
});
```

`versionOf(data)` is `undefined` for non-object data, responses without ETag, and data from persisted caches or SSR; the save is then unconditional. `mergeMutation` never sends an ETag.

## Rules

- **Recreating endpoints is fine.** The package shares one endpoint instance per `QueryClient`, HTTP client, endpoint class and URL, so ETags survive re-renders. No `useMemo` for endpoints. The first instance's constructor settings (e.g. a polling end condition) win.
- **Read capability flags from the shared instance**: `endpointFor(queryClient, endpoint).setAllowed`. The object you just created has not seen any response.
- **Query keys** are `[origin, ...pathSegments, { typedrest: operation, ...args }]` (`endpointKey(endpoint)` for the prefix). Invalidate by hand with `invalidateEndpoint(queryClient, endpoint, scope)` rather than building keys.
- Do not wrap TypedRest calls in hand-written `queryFn`s when a helper exists; you would lose cancellation, versions and the shared ETag cache.
