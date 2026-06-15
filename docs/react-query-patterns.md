# React Query Patterns — Contributor Guide

This document explains the React Query conventions used across all three
applications in this repository (**react-vite**, **nextjs-app**,
**nextjs-pages**). After reading it you should be able to add new query
and mutation hooks that are consistent with the existing codebase.

> **TL;DR** — Every API file exports three things: a *fetcher function*,
> a *query-options / mutation-options factory*, and a *React hook*.
> Hooks accept an optional `queryConfig` or `mutationConfig` object so
> that consumers can override React Query behaviour (`enabled`, `onSuccess`,
> `staleTime`, …) without rewriting the base configuration.

---

## 1. Foundational Types (`src/lib/react-query.ts`)

All three apps ship an identical `src/lib/react-query.ts`. It exports:

| Export | What it does |
|--------|-------------|
| `queryConfig` | Default `QueryClient` options applied globally (staleTime 60 s, no retry, no refetchOnWindowFocus). Passed to `new QueryClient({ defaultOptions: queryConfig })` in `src/app/provider.tsx`. |
| `ApiFnReturnType<Fn>` | Utility type that extracts the resolved return type of an async function. Used to keep mutation types in sync with their fetcher automatically. |
| `QueryConfig<T>` | The type consumers use for `queryConfig`. It is `Omit<ReturnType<T>, 'queryKey' \| 'queryFn'>` — everything a `queryOptions()` call returns **except** `queryKey` and `queryFn`, which must stay under the hook's control. |
| `MutationConfig<Fn>` | A type-safe wrapper around `UseMutationOptions` that infers `data`, `error`, and `variables` from the fetcher's signature. |

### Why `QueryConfig` omits `queryKey` and `queryFn`

The hook is the single owner of the cache identity. If a consumer could
override `queryKey`, it would silently break cache sharing with other
components that use the same hook. The same logic applies to `queryFn` —
the hook knows which endpoint to call; the consumer should only control
*caching and lifecycle* options (`enabled`, `staleTime`, `refetchInterval`,
`select`, etc.).

### Why `MutationConfig` exists instead of raw `UseMutationOptions`

`UseMutationOptions<TData, TError, TVariables>` requires three generic
parameters. By wrapping it in `MutationConfig<typeof myFetcher>`, we let
TypeScript infer all three from the fetcher function. This eliminates
manual type annotations and keeps mutation hooks type-safe end-to-end.

---

## 2. The Three-Layer Pattern

Every API endpoint file in `src/features/*/api/` follows the same
three-layer structure. The pattern is identical in all three apps.

```
┌──────────────────────────────────────────────────────────┐
│  Layer 1 — Fetcher function                              │
│  Pure async function. Calls the API client.              │
│  No React dependency. Testable in isolation.             │
├──────────────────────────────────────────────────────────┤
│  Layer 2 — Options factory                               │
│  Returns queryOptions() / infiniteQueryOptions() or      │
│  nothing (mutations). Produces the reusable, cache-key-  │
│  aware configuration object.                             │
├──────────────────────────────────────────────────────────┤
│  Layer 3 — React hook                                    │
│  Thin wrapper: spreads the factory output into useQuery  │
│  / useMutation, then spreads the consumer's queryConfig  │
│  or mutationConfig on top.                               │
└──────────────────────────────────────────────────────────┘
```

---

## 3. Query Hooks — Step by Step

**Reference implementation:** `src/features/discussions/api/get-discussions.ts`

### 3.1 Anatomy of a query file

```typescript
import { queryOptions, useQuery } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { QueryConfig } from '@/lib/react-query';
import { Discussion, Meta } from '@/types/api';

// ── Layer 1: Fetcher ──────────────────────────────────────
export const getDiscussions = (page = 1): Promise<{
  data: Discussion[];
  meta: Meta;
}> => {
  return api.get(`/discussions`, { params: { page } });
};

// ── Layer 2: Options factory ──────────────────────────────
export const getDiscussionsQueryOptions = ({ page }: { page?: number } = {}) => {
  return queryOptions({
    queryKey: page ? ['discussions', { page }] : ['discussions'],
    queryFn: () => getDiscussions(page),
  });
};

// ── Layer 3: Hook ─────────────────────────────────────────
type UseDiscussionsOptions = {
  page?: number;
  queryConfig?: QueryConfig<typeof getDiscussionsQueryOptions>;
};

export const useDiscussions = ({ queryConfig, page }: UseDiscussionsOptions) => {
  return useQuery({
    ...getDiscussionsQueryOptions({ page }),  // base config
    ...queryConfig,                            // consumer overrides
  });
};
```

### 3.2 Key rules

1. **The fetcher is a plain async function** — no hooks, no React.
   This makes it reusable for server-side prefetching (`queryClient.prefetchQuery`)
   and unit testing.
2. **The options factory is also exported** — mutation hooks import it
   to derive the `queryKey` for cache invalidation without duplicating
   the key logic.
3. **`queryConfig` is always optional** — calling `useDiscussions({})`
   must work with zero configuration.
4. **Spread order matters** — factory first, then `queryConfig`. This
   lets consumers override any non-protected option while the hook
   retains control of `queryKey` and `queryFn`.

### 3.3 Single-entity query (no pagination)

**Reference:** `src/features/discussions/api/get-discussion.ts`

```typescript
export const getDiscussionQueryOptions = (discussionId: string) => {
  return queryOptions({
    queryKey: ['discussions', discussionId],
    queryFn: () => getDiscussion({ discussionId }),
  });
};

export const useDiscussion = ({ discussionId, queryConfig }: UseDiscussionOptions) => {
  return useQuery({
    ...getDiscussionQueryOptions(discussionId),
    ...queryConfig,
  });
};
```

Note: `getDiscussionQueryOptions` is also imported by
`update-discussion.ts` so the update mutation can refetch the exact
discussion after a successful PATCH.

---

## 4. Mutation Hooks — Step by Step

**Reference implementation:** `src/features/discussions/api/create-discussion.ts`

### 4.1 Anatomy of a mutation file

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { z } from 'zod';
import { api } from '@/lib/api-client';
import { MutationConfig } from '@/lib/react-query';
import { Discussion } from '@/types/api';
import { getDiscussionsQueryOptions } from './get-discussions';

// ── Validation schema + derived type ─────────────────────
export const createDiscussionInputSchema = z.object({
  title: z.string().min(1, 'Required'),
  body: z.string().min(1, 'Required'),
});
export type CreateDiscussionInput = z.infer<typeof createDiscussionInputSchema>;

// ── Layer 1: Fetcher ─────────────────────────────────────
export const createDiscussion = ({
  data,
}: { data: CreateDiscussionInput }): Promise<Discussion> => {
  return api.post(`/discussions`, data);
};

// ── Layer 3: Hook (no factory needed for mutations) ──────
type UseCreateDiscussionOptions = {
  mutationConfig?: MutationConfig<typeof createDiscussion>;
};

export const useCreateDiscussion = ({
  mutationConfig,
}: UseCreateDiscussionOptions = {}) => {
  const queryClient = useQueryClient();

  // Extract onSuccess so we can chain it AFTER cache invalidation
  const { onSuccess, ...restConfig } = mutationConfig || {};

  return useMutation({
    onSuccess: (...args) => {
      // Step A: invalidate stale queries
      queryClient.invalidateQueries({
        queryKey: getDiscussionsQueryOptions().queryKey,
      });
      // Step B: call the consumer's onSuccess if provided
      onSuccess?.(...args);
    },
    ...restConfig,
    mutationFn: createDiscussion,  // always last — prevents override
  });
};
```

### 4.2 Key rules

1. **`mutationFn` is placed last** in the `useMutation` config object.
   This guarantees the consumer's `...restConfig` spread can never
   accidentally replace the fetcher.
2. **`onSuccess` is chained, not replaced.** The hook destructures
   `onSuccess` from `mutationConfig`, runs cache invalidation first,
   then calls the consumer's callback. This ensures the cache is
   always fresh regardless of what the consumer does.
3. **The rest of `mutationConfig` is spread** (`...restConfig`) so
   consumers can still pass `onError`, `onSettled`, `retry`, etc.
4. **Zod schemas are exported** alongside the hook so form components
   can reuse the exact same validation rules.

### 4.3 Mutation without input schema (delete)

**Reference:** `src/features/discussions/api/delete-discussion.ts`

When no complex input is needed, skip the Zod schema:

```typescript
export const deleteDiscussion = ({ discussionId }: { discussionId: string }) => {
  return api.delete(`/discussions/${discussionId}`);
};

export const useDeleteDiscussion = ({ mutationConfig }: UseDeleteDiscussionOptions = {}) => {
  const queryClient = useQueryClient();
  const { onSuccess, ...restConfig } = mutationConfig || {};

  return useMutation({
    onSuccess: (...args) => {
      queryClient.invalidateQueries({
        queryKey: getDiscussionsQueryOptions().queryKey,
      });
      onSuccess?.(...args);
    },
    ...restConfig,
    mutationFn: deleteDiscussion,
  });
};
```

### 4.4 Mutation that refetches a specific query (update)

**Reference:** `src/features/discussions/api/update-discussion.ts`

For updates, `refetchQueries` on the *specific* entity's query key
rather than invalidating the entire list:

```typescript
onSuccess: (data, ...args) => {
  queryClient.refetchQueries({
    queryKey: getDiscussionQueryOptions(data.id).queryKey,
  });
  onSuccess?.(data, ...args);
},
```

---

## 5. Infinite (Paginated) Query Hooks

**Reference implementation:** `src/features/comments/api/get-comments.ts`

```typescript
import { infiniteQueryOptions, useInfiniteQuery } from '@tanstack/react-query';

// Layer 1: Fetcher
export const getComments = ({
  discussionId,
  page = 1,
}: { discussionId: string; page?: number }): Promise<{ data: Comment[]; meta: Meta }> => {
  return api.get(`/comments`, { params: { discussionId, page } });
};

// Layer 2: Infinite-query options factory
export const getInfiniteCommentsQueryOptions = (discussionId: string) => {
  return infiniteQueryOptions({
    queryKey: ['comments', discussionId],
    queryFn: ({ pageParam = 1 }) => getComments({ discussionId, page: pageParam as number }),
    getNextPageParam: (lastPage) => {
      if (lastPage?.meta?.page === lastPage?.meta?.totalPages) return undefined;
      return lastPage.meta.page + 1;
    },
    initialPageParam: 1,
  });
};

// Layer 3: Hook
export const useInfiniteComments = ({ discussionId }: UseCommentsOptions) => {
  return useInfiniteQuery({
    ...getInfiniteCommentsQueryOptions(discussionId),
  });
};
```

Key differences from regular queries:
- Uses `infiniteQueryOptions` + `useInfiniteQuery` instead of `queryOptions` + `useQuery`.
- The factory must define `getNextPageParam` and `initialPageParam`.
- The consumer reads data via `data.pages` / `data.pageParams`.

---

## 6. Consumer Usage

### 6.1 Using `queryConfig` to conditionally enable a query

```typescript
// apps/react-vite/src/app/routes/auth/register.tsx
const [chooseTeam, setChooseTeam] = useState(false);

const teamsQuery = useTeams({
  queryConfig: {
    enabled: chooseTeam, // only fetch when the user opts in
  },
});
```

Other common `queryConfig` overrides:

| Option | When to use |
|--------|------------|
| `enabled: false` | Defer the request until a condition is met |
| `staleTime` | Override the default 60 s stale time for this call |
| `refetchInterval` | Poll the endpoint on an interval |
| `select` | Transform the data before it reaches the component |
| `retry` | Override the global `retry: false` default |

### 6.2 Using `mutationConfig` to show a notification on success

```typescript
// apps/react-vite/src/features/discussions/components/create-discussion.tsx
const createDiscussionMutation = useCreateDiscussion({
  mutationConfig: {
    onSuccess: () => {
      addNotification({ type: 'success', title: 'Discussion Created' });
    },
  },
});

// Trigger the mutation from a form submit:
createDiscussionMutation.mutate({ data: formValues });
```

Other common `mutationConfig` overrides:

| Option | When to use |
|--------|------------|
| `onError` | Show a toast or redirect on failure |
| `onSettled` | Close a loading dialog regardless of outcome |
| `retry` | Retry transient network failures |

---

## 7. Cross-App Comparison

All three apps follow the **exact same pattern**. The only differences
are framework-specific details (API client implementation, cookie
handling in Next.js Pages).

| Aspect | react-vite | nextjs-app | nextjs-pages |
|--------|-----------|-----------|-------------|
| `src/lib/react-query.ts` | Identical | Identical | Identical |
| Query hook pattern | fetcher → queryOptions → hook | Same | Same |
| Mutation hook pattern | fetcher → hook with cache invalidation | Same | Same |
| Infinite query pattern | fetcher → infiniteQueryOptions → hook | Same | Same |
| API client | Axios | Native `fetch` | Axios |
| Cookie support | — | — | `attachCookie()` helper |
| `provider.tsx` | `QueryClientProvider` + `queryConfig` | Same | Same |

### File locations (same relative path in each app)

```
apps/{react-vite,nextjs-app,nextjs-pages}/src/
├── lib/
│   └── react-query.ts          ← QueryConfig, MutationConfig, queryConfig
├── app/
│   └── provider.tsx            ← QueryClient initialization
└── features/
    ├── discussions/api/
    │   ├── get-discussions.ts  ← list query (queryOptions)
    │   ├── get-discussion.ts   ← single-entity query
    │   ├── create-discussion.ts← mutation (onSuccess + invalidate)
    │   ├── update-discussion.ts← mutation (onSuccess + refetch)
    │   └── delete-discussion.ts← mutation (onSuccess + invalidate)
    └── comments/api/
        └── get-comments.ts     ← infinite query (infiniteQueryOptions)
```

---

## 8. Adding a New API Endpoint — Checklist

### New query (GET)

1. Create `src/features/<feature>/api/get-<entity>.ts`.
2. Define the **fetcher** — a plain async function returning a typed `Promise`.
3. Define a **`getXxxQueryOptions`** factory using `queryOptions()`.
4. Export the **hook** `useXxx` that spreads the factory + `queryConfig`.
5. Make sure `queryConfig` is typed as `QueryConfig<typeof getXxxQueryOptions>`.

### New mutation (POST / PATCH / DELETE)

1. Create `src/features/<feature>/api/<action>-<entity>.ts`.
2. (Optional) Define a **Zod schema** and export the inferred type.
3. Define the **fetcher** — a plain async function.
4. Export the **hook** `useXxxMutation` that:
   - Destructures `{ onSuccess, ...restConfig }` from `mutationConfig`.
   - Passes `onSuccess` that **first** invalidates/refetches related queries, **then** calls the consumer's `onSuccess`.
   - Spreads `...restConfig`.
   - Sets `mutationFn` **last**.
5. Make sure `mutationConfig` is typed as `MutationConfig<typeof fetcher>`.

### New infinite query (paginated GET)

1. Same as a regular query but use `infiniteQueryOptions` + `useInfiniteQuery`.
2. Define `getNextPageParam` and `initialPageParam` in the factory.

---

## 9. Common Pitfalls

| Pitfall | Why it's wrong | Do this instead |
|---------|---------------|-----------------|
| Putting `queryKey` in `queryConfig` | `QueryConfig` omits it; TypeScript will reject it | Let the hook own `queryKey` |
| Spreading `...queryConfig` *before* the factory | Consumer values get silently overwritten by the factory | Always spread factory first, `queryConfig` second |
| Calling `useMutation` without destructuring `onSuccess` | Consumer's `onSuccess` replaces cache invalidation | Always destructure and chain |
| Placing `mutationFn` before `...restConfig` | Consumer could override it | Place `mutationFn` last |
| Skipping the options factory | Other hooks (mutations, prefetching) can't reuse the `queryKey` | Always export a `getXxxQueryOptions` factory |
| Duplicating query keys as string literals | Keys drift out of sync across files | Import the factory: `getDiscussionsQueryOptions().queryKey` |

---

## 10. Related Documentation

- [API Layer](./api-layer.md) — high-level philosophy of the API layer
- [State Management](./state-management.md) — where React Query fits in the overall state strategy
- [Project Structure](./project-structure.md) — feature-based folder conventions
- [Testing](./testing.md) — how API hooks are tested with MSW
