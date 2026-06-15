# React Query Conventions

This project uses [TanStack Query (React Query)](https://tanstack.com/query) for all server state management. Every API hook follows the same layered pattern across all three sample apps (`react-vite`, `nextjs-app`, `nextjs-pages`). This guide explains the conventions so that contributors can add new queries and mutations consistently.

## Core Configuration

Each app shares an identical configuration file at `src/lib/react-query.ts` that exports three things:

### 1. `queryConfig` -- Default QueryClient Options

```ts
export const queryConfig = {
  queries: {
    refetchOnWindowFocus: false,
    retry: false,
    staleTime: 1000 * 60,
  },
} satisfies DefaultOptions;
```

This is passed to `new QueryClient({ defaultOptions: queryConfig })` in each app's `provider.tsx`. All queries inherit these defaults unless overridden.

### 2. `QueryConfig<T>` -- Type Helper for Query Hooks

```ts
export type QueryConfig<T extends (...args: any[]) => any> = Omit<
  ReturnType<T>,
  'queryKey' | 'queryFn'
>;
```

`QueryConfig` takes a **query options function** (e.g. `typeof getDiscussionsQueryOptions`) as its type parameter. It strips out `queryKey` and `queryFn` -- the fields the hook controls -- and leaves all other React Query options (`enabled`, `staleTime`, `select`, etc.) available for callers to override. This is what makes `queryConfig` on each hook type-safe: callers can tune behavior without accidentally replacing the query key or fetch function.

### 3. `MutationConfig<MutationFnType>` -- Type Helper for Mutation Hooks

```ts
export type MutationConfig<
  MutationFnType extends (...args: any) => Promise<any>,
> = UseMutationOptions<
  ApiFnReturnType<MutationFnType>,
  Error,
  Parameters<MutationFnType>[0]
>;
```

`MutationConfig` takes a **mutation function** (e.g. `typeof createDiscussion`) as its type parameter. It produces fully typed `UseMutationOptions` where `data`, `error`, and `variables` are all inferred from the function signature.

[Configuration Source Code](../apps/react-vite/src/lib/react-query.ts)

---

## The Three-Layer Pattern

Every API endpoint in this project is declared in a single file with three layers:

```
1. Fetcher function    -- calls the API, returns typed data
2. Query options       -- defines queryKey + queryFn (for queries) or is skipped (for mutations)
3. React hook          -- wraps useQuery/useMutation, accepts config overrides
```

### Why This Structure?

- **Fetcher functions** are pure data-fetching -- they can be used in loaders, server components, or tests without React.
- **Query options functions** are exported separately so that route loaders, server-side prefetching, hover prefetching, and cache invalidation can reference the same `queryKey` without importing the hook.
- **Hooks** are the consumer-facing API. They compose the query options with caller-provided overrides via `queryConfig` or `mutationConfig`.

---

## Adding a New Query

Follow these steps to add a new query hook. We'll use a hypothetical "get projects" endpoint as an example.

### Step 1: Create the API file

Create `src/features/projects/api/get-projects.ts` inside the relevant app:

```ts
import { queryOptions, useQuery } from '@tanstack/react-query';

import { api } from '@/lib/api-client';
import { QueryConfig } from '@/lib/react-query';
import { Project } from '@/types/api';

// Layer 1: Fetcher function
export const getProjects = (): Promise<{ data: Project[] }> => {
  return api.get('/projects');
};

// Layer 2: Query options (exported for prefetching and cache invalidation)
export const getProjectsQueryOptions = () => {
  return queryOptions({
    queryKey: ['projects'],
    queryFn: () => getProjects(),
  });
};

// Layer 3: Hook with queryConfig override
type UseProjectsOptions = {
  queryConfig?: QueryConfig<typeof getProjectsQueryOptions>;
};

export const useProjects = ({ queryConfig = {} }: UseProjectsOptions = {}) => {
  return useQuery({
    ...getProjectsQueryOptions(),
    ...queryConfig,
  });
};
```

**Key points:**

- The `queryKey` is always a descriptive array (e.g. `['projects']`). Add parameters to the key when the query depends on them (e.g. `['projects', { page }]`).
- The hook spreads query options first, then `queryConfig` second -- this lets callers override options like `enabled` or `staleTime` while the hook owns `queryKey` and `queryFn`.
- Export both the query options function and the hook. The options function is needed for prefetching and cache invalidation from other parts of the code.

### Step 2: Use the hook in a component

```tsx
const projectsQuery = useProjects();

// With overrides:
const projectsQuery = useProjects({
  queryConfig: {
    enabled: shouldLoadProjects,  // conditionally fetch
  },
});
```

For a real example of conditional fetching, see how `useTeams` is called with `enabled`:
[Teams Query Usage](../apps/react-vite/src/app/routes/auth/register.tsx)

### Existing query examples

| Hook | File | Notes |
|---|---|---|
| `useDiscussions` | [`get-discussions.ts`](../apps/react-vite/src/features/discussions/api/get-discussions.ts) | Paginated query with `page` parameter in query key |
| `useDiscussion` | [`get-discussion.ts`](../apps/react-vite/src/features/discussions/api/get-discussion.ts) | Single-resource query with `discussionId` parameter |
| `useUsers` | [`get-users.ts`](../apps/react-vite/src/features/users/api/get-users.ts) | Simple list query |
| `useTeams` | [`get-teams.ts`](../apps/react-vite/src/features/teams/api/get-teams.ts) | Simplest example -- no parameters |

---

## Adding a New Mutation

Mutations follow a similar pattern but additionally handle cache invalidation. We'll use a hypothetical "create project" endpoint.

### Step 1: Create the API file

Create `src/features/projects/api/create-project.ts`:

```ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { z } from 'zod';

import { api } from '@/lib/api-client';
import { MutationConfig } from '@/lib/react-query';
import { Project } from '@/types/api';

import { getProjectsQueryOptions } from './get-projects';

// Input validation schema
export const createProjectInputSchema = z.object({
  name: z.string().min(1, 'Required'),
  description: z.string().min(1, 'Required'),
});

export type CreateProjectInput = z.infer<typeof createProjectInputSchema>;

// Layer 1: Fetcher function
export const createProject = ({
  data,
}: {
  data: CreateProjectInput;
}): Promise<Project> => {
  return api.post('/projects', data);
};

// Layer 2: Hook with mutationConfig override
type UseCreateProjectOptions = {
  mutationConfig?: MutationConfig<typeof createProject>;
};

export const useCreateProject = ({
  mutationConfig,
}: UseCreateProjectOptions = {}) => {
  const queryClient = useQueryClient();

  const { onSuccess, ...restConfig } = mutationConfig || {};

  return useMutation({
    onSuccess: (...args) => {
      queryClient.invalidateQueries({
        queryKey: getProjectsQueryOptions().queryKey,
      });
      onSuccess?.(...args);
    },
    ...restConfig,
    mutationFn: createProject,
  });
};
```

**Key points:**

- `onSuccess` is **destructured** from `mutationConfig` so the hook can run cache invalidation first, then call the caller's `onSuccess` afterward. This ensures cache is always updated regardless of what the caller does.
- `...restConfig` is spread to allow callers to provide other mutation options (`onError`, `onSettled`, etc.).
- `mutationFn` is placed **after** `...restConfig` so it cannot be accidentally overridden.
- Input is validated with a Zod schema. Export the schema so that form components can use it for validation.

### Step 2: Use the hook in a component

```tsx
const createProjectMutation = useCreateProject({
  mutationConfig: {
    onSuccess: () => {
      addNotification({ type: 'success', title: 'Project Created' });
      navigate('/app/projects');
    },
  },
});

// Trigger the mutation:
createProjectMutation.mutate({ data: formValues });
```

### Cache invalidation strategies

The project uses two strategies depending on the operation:

| Strategy | When to use | Example |
|---|---|---|
| `invalidateQueries` | After create or delete -- the list needs a full refetch | [`create-discussion.ts`](../apps/react-vite/src/features/discussions/api/create-discussion.ts), [`delete-discussion.ts`](../apps/react-vite/src/features/discussions/api/delete-discussion.ts) |
| `refetchQueries` | After update -- refetch the specific item that changed | [`update-discussion.ts`](../apps/react-vite/src/features/discussions/api/update-discussion.ts) |

For update mutations, the `onSuccess` callback receives the updated data as its first argument, which you can use to target the specific query key:

```ts
onSuccess: (data, ...args) => {
  queryClient.refetchQueries({
    queryKey: getDiscussionQueryOptions(data.id).queryKey,
  });
  onSuccess?.(data, ...args);
},
```

### Existing mutation examples

| Hook | File | Cache Strategy |
|---|---|---|
| `useCreateDiscussion` | [`create-discussion.ts`](../apps/react-vite/src/features/discussions/api/create-discussion.ts) | `invalidateQueries` on discussions list |
| `useUpdateDiscussion` | [`update-discussion.ts`](../apps/react-vite/src/features/discussions/api/update-discussion.ts) | `refetchQueries` on specific discussion |
| `useDeleteDiscussion` | [`delete-discussion.ts`](../apps/react-vite/src/features/discussions/api/delete-discussion.ts) | `invalidateQueries` on discussions list |
| `useCreateComment` | [`create-comment.ts`](../apps/react-vite/src/features/comments/api/create-comment.ts) | `invalidateQueries` on comments for that discussion |
| `useDeleteComment` | [`delete-comment.ts`](../apps/react-vite/src/features/comments/api/delete-comment.ts) | `invalidateQueries` on comments for that discussion |

---

## Infinite Queries

For paginated data loaded incrementally (e.g. comments), use `infiniteQueryOptions` and `useInfiniteQuery` instead:

```ts
import { infiniteQueryOptions, useInfiniteQuery } from '@tanstack/react-query';

export const getInfiniteCommentsQueryOptions = (discussionId: string) => {
  return infiniteQueryOptions({
    queryKey: ['comments', discussionId],
    queryFn: ({ pageParam = 1 }) => {
      return getComments({ discussionId, page: pageParam as number });
    },
    getNextPageParam: (lastPage) => {
      if (lastPage?.meta?.page === lastPage?.meta?.totalPages) return undefined;
      return lastPage.meta.page + 1;
    },
    initialPageParam: 1,
  });
};
```

The options function is reused for both the hook and for prefetching in route loaders or server components.

[Infinite Query Example Code](../apps/react-vite/src/features/comments/api/get-comments.ts)

---

## Differences Across the Three Apps

All three apps (`react-vite`, `nextjs-app`, `nextjs-pages`) share the same hook interface and file structure. The differences are in how data is prefetched and how auth cookies are handled at the framework level.

### `react-vite`

- **Prefetching in route loaders**: Route files define a `clientLoader` that calls `queryClient.getQueryData() ?? queryClient.fetchQuery()` to prepopulate the cache before the component renders.
- **Hover prefetching**: Components call `queryClient.prefetchQuery()` on `onMouseEnter` to speculatively load data.
- **Auth**: Uses `react-query-auth` library via `configureAuth()`.

[Route Loader Example](../apps/react-vite/src/app/routes/app/discussions/discussions.tsx) |
[Hover Prefetch Example](../apps/react-vite/src/features/discussions/components/discussions-list.tsx)

### `nextjs-app`

- **Server-side prefetching**: Server components create a `new QueryClient()`, call `prefetchQuery`, then pass `dehydrate(queryClient)` to a `<HydrationBoundary>` that wraps the client component.
- **Auth**: Defines its own `useUser`, `useLogin`, `useRegister`, `useLogout` hooks directly (does **not** use `react-query-auth`).

[Server Prefetch Example](../apps/nextjs-app/src/app/app/discussions/page.tsx)

### `nextjs-pages`

- **SSR with cookies**: Fetcher functions accept an optional `cookie` parameter and pass it via `attachCookie(cookie).headers` for server-side rendering with authentication.
- **Auth**: Uses `react-query-auth` library, same as `react-vite`.

[SSR Cookie Example](../apps/nextjs-pages/src/features/discussions/api/get-discussions.ts)

### What stays the same

Across all three apps, the following are identical:

- The `src/lib/react-query.ts` configuration and types
- The hook interface (`queryConfig` / `mutationConfig` pattern)
- The feature folder structure (`src/features/<name>/api/`)
- Cache invalidation strategies in mutation hooks
- Zod input schemas for mutations

When adding a new feature, implement the hooks in `react-vite` first, then replicate to the other two apps, adjusting only the framework-specific prefetching and SSR patterns as needed.

---

## Quick Reference

### File placement

```
src/features/<feature-name>/api/
  get-<resource>.ts        -- useQuery hook
  get-<resources>.ts       -- useQuery hook (list)
  create-<resource>.ts     -- useMutation hook
  update-<resource>.ts     -- useMutation hook
  delete-<resource>.ts     -- useMutation hook
```

### Checklist for a new query

1. Create fetcher function that calls `api.get()`
2. Create query options function using `queryOptions()` with a descriptive `queryKey`
3. Export both the options function and the hook
4. Type the hook's options with `QueryConfig<typeof yourQueryOptions>`
5. Spread `queryConfig` after the query options in `useQuery()`

### Checklist for a new mutation

1. Define a Zod input schema and export it
2. Create fetcher function that calls `api.post()` / `api.patch()` / `api.delete()`
3. Type the hook's options with `MutationConfig<typeof yourMutationFn>`
4. Destructure `onSuccess` from `mutationConfig` to compose cache invalidation
5. Place `mutationFn` after `...restConfig` to prevent overrides
6. Use `invalidateQueries` for create/delete, `refetchQueries` for update
