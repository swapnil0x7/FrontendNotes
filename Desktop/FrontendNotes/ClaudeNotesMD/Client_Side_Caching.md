# 8 Caching Strategies Every React Dev Should Know

### Interview prep guide — explanation, code, gotchas, and what to say out loud

**Mental model to carry through all 8:** client-side caching is about _server state_ — data you don't own, that goes out of date without telling you. Every strategy below is answering one of three questions: _Do I need to fetch?_ (1, 2, 6) · _What do I show while fetching?_ (3, 7) · _When do I throw it away?_ (4, 5, 8)

Code uses **TanStack Query v5** (the `useQuery` API) and **Next.js App Router**, since that's what interviewers ask about.

---

## 1. Request Deduplication

### The idea

Three components mount at the same time and all need the current user. Naively that's three identical network calls. Deduplication means: if a request for the same key is already **in flight**, don't start a new one — subscribe to the existing promise.

Think of it like a group of people ordering the same coffee. One person goes to the counter; everyone else waits for that one cup to be made and shares the result.

### With TanStack Query (automatic)

```jsx
// components/Header.jsx
function Header() {
  const { data: user } = useQuery({ queryKey: ["user"], queryFn: fetchUser });
  return <span>{user?.name}</span>;
}

// components/Sidebar.jsx
function Sidebar() {
  const { data: user } = useQuery({ queryKey: ["user"], queryFn: fetchUser });
  return <Avatar src={user?.avatar} />;
}

// components/Settings.jsx — same key again
function Settings() {
  const { data: user } = useQuery({ queryKey: ["user"], queryFn: fetchUser });
  return <Form defaults={user} />;
}
```

All three mount together → **exactly one** `fetchUser()` call. The cache is keyed by `['user']`, and the query has one "promise slot" per key.

### How it actually works under the hood

You should be able to sketch this on a whiteboard — it's a common follow-up:

```js
const inFlight = new Map();

function dedupedFetch(url) {
  if (inFlight.has(url)) {
    return inFlight.get(url); // hand back the SAME promise
  }

  const promise = fetch(url)
    .then((res) => res.json())
    .finally(() => inFlight.delete(url)); // clear once settled

  inFlight.set(url, promise);
  return promise;
}
```

The critical line is `.finally(() => inFlight.delete(url))`. Without it you've built a permanent cache, not a deduplicator — and you'd never get fresh data again.

### On the server (React Server Components)

React's `cache()` dedupes per-request, so a helper called in five different server components hits the DB once:

```js
import { cache } from "react";

export const getUser = cache(async (id) => {
  return db.user.findUnique({ where: { id } });
});
```

Scope matters: `cache()` is **per render pass**, cleared between requests. It's deduplication, not caching across users.

> **Gotcha:** deduplication only works if the key is identical. `['user']` and `['user', undefined]` are different keys, so you get two requests. This is where sloppy query keys silently cost you performance.

**Say this:** _"Deduplication collapses concurrent identical requests into one shared in-flight promise. React Query does it by query key; on the server React's `cache()` does it per request. It's about concurrency, not persistence — the entry is released as soon as the promise settles."_

---

## 2. staleTime vs gcTime

This is the single most asked React Query question. Get it crisp.

|                     | `staleTime`                                     | `gcTime` (was `cacheTime` in v4)                     |
| ------------------- | ----------------------------------------------- | ---------------------------------------------------- |
| Controls            | **Network requests**                            | **Memory**                                           |
| Question it answers | "Is this data fresh enough to skip refetching?" | "How long do I keep unused data before deleting it?" |
| Default             | `0` (stale immediately)                         | `5 * 60 * 1000` (5 min)                              |
| Timer starts        | when data arrives                               | when the **last** component using it unmounts        |
| While it's running  | no background refetch                           | data stays in memory                                 |

### The lifecycle

```
fetch succeeds
   │
   ├── FRESH ──(staleTime elapses)──> STALE
   │                                    │
   │                    still served from cache instantly,
   │                    but refetches in background on triggers
   │
   └── component unmounts → query becomes INACTIVE
                              │
                     (gcTime elapses with no observers)
                              │
                         DELETED from cache
```

### Code

```jsx
useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
  staleTime: 5 * 60 * 1000, // don't refetch for 5 minutes
  gcTime: 10 * 60 * 1000, // keep in memory 10 min after last unmount
});
```

### The rule of thumb

Keep **`gcTime >= staleTime`**. If `gcTime` is shorter, data gets deleted while still notionally fresh — so the user gets a hard loading spinner instead of an instant cached render.

### Picking staleTime by data type

```js
// Config / feature flags / country list — basically never changes
staleTime: Infinity;

// Never refetch, even on manual invalidation (v5+)
staleTime: "static";

// User profile, dashboard summary
staleTime: 5 * 60 * 1000;

// Stock price, live inventory, chat
staleTime: 0; // default — always revalidate
```

> **Gotcha #1:** the default `staleTime: 0` is why people say "React Query fetches too much." It refetches on mount, on window focus, and on reconnect. The fix is almost always raising `staleTime`, _not_ raising `gcTime`.
>
> **Gotcha #2:** stale data is still **shown**. Stale ≠ gone. This trips people up — they assume stale means the user sees a spinner. They don't; see strategy #3.

**Say this:** _"`staleTime` controls how often you hit the network; `gcTime` controls how long unused data survives in memory. The gcTime clock only starts when there are zero observers. If a screen feels chatty, raise staleTime — gcTime won't help."_

---

## 3. Stale While Revalidate (SWR)

### The idea

When cached data exists but is stale, **show it immediately** and refetch in the background. Swap in the fresh data when it arrives. The user never sees a spinner on a repeat visit.

The trade is deliberate: a few seconds of possibly-outdated data in exchange for a UI that feels instant. For a dashboard, that's a great trade. For a bank balance transfer confirmation, it isn't.

### First visit vs. repeat visit

```
First visit:   [ spinner ] ────────> [ data ]
Repeat visit:  [ cached data shown instantly ]
                      └── background refetch ──> [ updated data ]
```

### Code — the flag that matters

```jsx
function Products() {
  const { data, isPending, isFetching } = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts,
    staleTime: 30_000,
  });

  // isPending = no data at all yet → real loading state
  if (isPending) return <Skeleton />;

  return (
    <div>
      {/* subtle indicator, NOT a full-page spinner */}
      {isFetching && <RefreshDot />}
      <ProductGrid items={data} />
    </div>
  );
}
```

Know the difference cold — it's a favourite follow-up:

- **`isPending`** — there is no data in the cache. Show a skeleton.
- **`isFetching`** — a request is in flight, _including_ background revalidation. Show a subtle spinner or nothing.
- **`isLoading`** — shorthand for `isPending && isFetching` (first-ever load).

### Keeping the previous page's data during pagination

```jsx
import { keepPreviousData } from "@tanstack/react-query";

useQuery({
  queryKey: ["products", page],
  queryFn: () => fetchProducts(page),
  placeholderData: keepPreviousData, // page 2 shows page 1 until it loads
});
```

Without this, changing the page key means new key → no cached data → layout collapses to a spinner and the page jumps.

### The HTTP version

Same concept at the CDN layer — worth mentioning to show breadth:

```
Cache-Control: max-age=60, stale-while-revalidate=300
```

Serve from cache for 60s. For the next 300s, still serve the stale copy instantly _and_ revalidate in the background.

**Say this:** _"SWR trades a small window of staleness for perceived instant loads. The implementation detail people miss is distinguishing `isPending` from `isFetching` — if you gate your UI on `isFetching` you re-introduce the spinner you were trying to remove."_

---

## 4. Invalidation on Mutation

> "There are only two hard things in computer science: cache invalidation and naming things."

### The idea

After you change data on the server, the cache is lying. Invalidation marks the affected queries stale and refetches the ones currently on screen.

### The basic pattern

```jsx
function useAddTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newTodo) => api.post("/todos", newTodo),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["todos"] });
    },
  });
}
```

Because invalidation matches by **prefix**, `['todos']` invalidates `['todos']`, `['todos', 'list']`, `['todos', { status: 'done' }]` — everything beneath it. This is exactly why key structure matters (strategy #8).

### Know the four tools and when to use each

| Method              | What it does                                | Use when                           |
| ------------------- | ------------------------------------------- | ---------------------------------- |
| `invalidateQueries` | marks stale + refetches **active** queries  | default choice after a mutation    |
| `refetchQueries`    | forces a refetch regardless of active state | you need it _now_, even off-screen |
| `setQueryData`      | writes the cache directly, no network call  | server returned the updated object |
| `resetQueries`      | back to initial state, discards data        | logout, tenant switch              |

### Skip the round-trip when the server already told you the answer

```jsx
useMutation({
  mutationFn: updateTodo,
  onSuccess: (updatedTodo) => {
    // write the single item directly — no refetch needed
    queryClient.setQueryData(["todos", updatedTodo.id], updatedTodo);
    // but the list ordering might have changed, so refresh that
    queryClient.invalidateQueries({ queryKey: ["todos", "list"] });
  },
});
```

### Optimistic updates (the senior-level answer)

Update the UI _before_ the server confirms, and roll back if it fails.

```jsx
useMutation({
  mutationFn: toggleTodo,

  onMutate: async (newTodo) => {
    // 1. stop in-flight refetches from overwriting our optimistic write
    await queryClient.cancelQueries({ queryKey: ["todos"] });

    // 2. snapshot for rollback
    const previous = queryClient.getQueryData(["todos"]);

    // 3. optimistically update
    queryClient.setQueryData(["todos"], (old) =>
      old.map((t) => (t.id === newTodo.id ? { ...t, done: !t.done } : t)),
    );

    return { previous }; // becomes `context` below
  },

  onError: (err, newTodo, context) => {
    queryClient.setQueryData(["todos"], context.previous); // roll back
  },

  onSettled: () => {
    // always resync with the server, success or failure
    queryClient.invalidateQueries({ queryKey: ["todos"] });
  },
});
```

The `cancelQueries` call is the part candidates forget. Without it, a refetch that started before your mutation can land _after_ your optimistic write and silently clobber it.

> **Gotcha:** `invalidateQueries` returns a promise. If you `await` it inside `onSuccess`, the mutation stays in `isPending` until the refetch finishes — useful when you want a modal to stay open with a spinner until the list is genuinely up to date.

**Say this:** _"I invalidate by key prefix in `onSuccess`, which marks matching queries stale and refetches the mounted ones. For snappy UX I use optimistic updates with `onMutate`/`onError` rollback, and always cancel in-flight queries first so a late response can't overwrite the optimistic state."_

---

## 5. Server Component Caching

This is the section most likely to be outdated in whatever blog post the interviewer read, so being precise about **versions** scores points.

### The Next.js 15 model (implicit caching)

Four separate caches, which is why it confused everyone:

1. **Request Memoization** — dedupes identical `fetch()` calls within a single render pass.
2. **Data Cache** — persists fetch results across requests and deployments.
3. **Full Route Cache** — the rendered HTML/RSC payload for static routes.
4. **Router Cache** — client-side, caches RSC payloads for back/forward navigation.

```jsx
// Next 15 style
const res = await fetch("https://api.example.com/products", {
  next: { revalidate: 3600, tags: ["products"] },
});
```

```js
// invalidate from a Server Action
"use server";
import { revalidateTag, revalidatePath } from "next/cache";

export async function createProduct(formData) {
  await db.product.create({
    /* ... */
  });
  revalidateTag("products"); // targeted
  revalidatePath("/products"); // whole route
}
```

### The Next.js 16 model: Cache Components (`use cache`)

Next 16 flipped the default. **Everything is dynamic unless you explicitly cache it.** Opt in via config:

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheComponents: true,
};

export default nextConfig;
```

Then mark what should be cached:

```jsx
async function BlogPost({ slug }) {
  "use cache";
  const post = await db.post.findUnique({ where: { slug } });
  return <article>{post.body}</article>;
}
```

Control lifetime and invalidation:

```jsx
import { cacheLife, cacheTag } from "next/cache";

async function getProducts(category) {
  "use cache";
  cacheLife("hours");
  cacheTag(`products-${category}`);

  return db.product.findMany({ where: { category } });
}
```

There are three directive variants:

- `'use cache'` — shared, in-memory on the server. **Never** use for user-specific data.
- `'use cache: private'` — for personalized content that reads request APIs.
- `'use cache: remote'` — durable, shared across server instances; costs a network hop.

### The security point that will impress an interviewer

A cached function returning user-specific data with the plain shared directive can **leak one user's data to another**. Cookies and headers can't be read inside a shared cached scope — read them outside and pass the value in as an argument, so it becomes part of the cache key:

```jsx
// ❌ dangerous — closes over request-scoped data
async function getDashboard() {
  "use cache";
  const userId = (await cookies()).get("uid").value; // not allowed / unsafe
  return db.dashboard.findMany({ where: { userId } });
}

// ✅ pass it in — userId becomes part of the cache key
async function getDashboard(userId) {
  "use cache";
  return db.dashboard.findMany({ where: { userId } });
}

export default async function Page() {
  const userId = (await cookies()).get("uid").value; // read outside
  const data = await getDashboard(userId);
  return <Dashboard data={data} />;
}
```

**Say this:** _"Next 15 cached implicitly across four layers — request memoization, data cache, full route cache, router cache. Next 16's Cache Components inverts that: dynamic by default, opt in with `use cache`, tune with `cacheLife` and invalidate with `cacheTag`. The main hazard is caching personalized data with the shared directive — request-scoped values have to be read outside and passed in as arguments so they're part of the key."_

---

## 6. Prefetch on Intent

### The idea

Fetch data _before_ the user asks for it, based on a signal that they're about to. Hovering a link, focusing it with the keyboard, or scrolling it into view are all "intent." By the time they click, the data is already in the cache and navigation is instant.

### Hover / focus prefetch

```jsx
function ProductLink({ id, children }) {
  const queryClient = useQueryClient();

  const prefetch = () => {
    queryClient.prefetchQuery({
      queryKey: ["product", id],
      queryFn: () => fetchProduct(id),
      staleTime: 60_000, // <- essential, see gotcha
    });
  };

  return (
    <Link
      href={`/products/${id}`}
      onMouseEnter={prefetch}
      onFocus={prefetch} // keyboard users too
      onTouchStart={prefetch} // mobile has no hover
    >
      {children}
    </Link>
  );
}
```

> **Gotcha:** `prefetchQuery` respects `staleTime`. Omit it (default `0`) and every hover fires a fresh request — mousing across a list of 50 products means 50 requests. Setting `staleTime` makes repeat hovers no-ops.

### Debounce so accidental pass-overs don't fire

```jsx
function useIntentPrefetch(prefetchFn, delay = 100) {
  const timer = useRef();

  return {
    onMouseEnter: () => {
      timer.current = setTimeout(prefetchFn, delay);
    },
    onMouseLeave: () => clearTimeout(timer.current),
  };
}
```

### Prefetch on viewport entry

```jsx
const ref = useRef(null);

useEffect(() => {
  const observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting) {
      queryClient.prefetchQuery({ queryKey: ['product', id], queryFn: ... });
      observer.disconnect();
    }
  }, { rootMargin: '200px' });   // start 200px before it's visible

  if (ref.current) observer.observe(ref.current);
  return () => observer.disconnect();
}, [id]);
```

### Server-side prefetch + hydration (Next.js)

Fetch on the server, ship the cache to the client so there's no loading state at all:

```jsx
// app/products/page.jsx  (Server Component)
import {
  dehydrate,
  HydrationBoundary,
  QueryClient,
} from "@tanstack/react-query";

export default async function Page() {
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({
    queryKey: ["products"],
    queryFn: fetchProducts,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProductList /> {/* client component; useQuery resolves from cache */}
    </HydrationBoundary>
  );
}
```

### The cost side (mention this — it shows judgment)

Prefetching spends bandwidth and server capacity on navigations that may never happen. On mobile data or a large list, that's real waste. Guard it:

```js
const conn = navigator.connection;
const shouldPrefetch = !conn?.saveData && !/2g/.test(conn?.effectiveType ?? "");
```

**Say this:** _"Prefetch on hover, focus, or viewport entry so the cache is warm before the click. Two things make it work in practice: setting `staleTime` so repeat hovers don't re-request, and debouncing so a mouse crossing the list doesn't fire everything. And I'd throttle it on slow connections — prefetching isn't free."_

---

## 7. Persisted Query Cache

### The idea

By default the cache lives in memory and dies on refresh. Persisting it to `localStorage` or IndexedDB means a returning user sees content instantly on first paint — and the app can work offline.

### Setup

```jsx
import { PersistQueryClientProvider } from "@tanstack/react-query-persist-client";
import { createSyncStoragePersister } from "@tanstack/query-sync-storage-persister";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 1000 * 60 * 60 * 24, // must be >= maxAge, or entries get GC'd first
    },
  },
});

const persister = createSyncStoragePersister({
  storage: window.localStorage,
});

export function App() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister,
        maxAge: 1000 * 60 * 60 * 24, // discard anything older than 24h
        buster: "v2", // bump to nuke old caches on deploy
      }}
    >
      <Routes />
    </PersistQueryClientProvider>
  );
}
```

### Don't persist everything

This is the answer that separates a considered response from a copy-pasted one:

```jsx
persistOptions={{
  persister,
  dehydrateOptions: {
    shouldDehydrateQuery: (query) => {
      const [scope] = query.queryKey;
      // never write auth/PII to localStorage — it's readable by any script
      if (['session', 'payment', 'me'].includes(scope)) return false;
      return query.state.status === 'success';
    },
  },
}}
```

Three real constraints to name:

1. **Security** — `localStorage` is plain text, readable by any JS on the page (XSS). No tokens, no PII.
2. **Size** — `localStorage` caps around 5MB and is **synchronous**, so it blocks the main thread. For anything large, use IndexedDB via `createAsyncStoragePersister` with `idb-keyval`.
3. **Schema drift** — cached data from an old app version can crash new components. That's what `buster` is for: bump it on every deploy that changes response shapes.

### Offline mutations

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    mutations: { retry: 3, networkMode: "offlineFirst" },
  },
});

// mutations queued while offline replay on reconnect
queryClient.setMutationDefaults(["todos", "add"], {
  mutationFn: addTodo,
});
// after rehydration:
queryClient.resumePausedMutations();
```

**Say this:** _"Persisting the cache to storage gives instant first paint for returning users and enables offline mode. Three things I always configure: filter out auth and PII with `shouldDehydrateQuery`, set a `buster` so a deploy invalidates stale schemas, and use IndexedDB instead of localStorage for anything sizeable since localStorage is synchronous and capped."_

---

## 8. Query Key Structuring

### The idea

The query key is simultaneously the **cache identity**, the **dependency array**, and the **invalidation selector**. Get it right and everything else in this list gets easier.

### Rule 1: every variable used inside `queryFn` belongs in the key

```jsx
// ❌ BUG — switching filters shows the wrong cached data
useQuery({
  queryKey: ["todos"],
  queryFn: () => fetchTodos(status, page),
});

// ✅ key changes when inputs change → automatic refetch
useQuery({
  queryKey: ["todos", { status, page }],
  queryFn: () => fetchTodos(status, page),
});
```

Same discipline as a `useEffect` dependency array. Miss a variable and you serve stale-but-wrong data — worse than slow data.

### Rule 2: structure hierarchically, general → specific

```js
["todos"][("todos", "list")][("todos", "list", { status: "done" })][ // everything todo-related // all lists // one filtered list
  ("todos", "detail")
][("todos", "detail", 5)]; // all details // one todo
```

Because invalidation matches by prefix, this gives you precise blast radius control:

```js
queryClient.invalidateQueries({ queryKey: ["todos"] }); // all of it
queryClient.invalidateQueries({ queryKey: ["todos", "list"] }); // lists only
queryClient.invalidateQueries({ queryKey: ["todos", "detail", 5] }); // one item
```

### Rule 3: use a query key factory

Scattering string literals across 40 files means one typo silently breaks invalidation, with no error. Centralize:

```js
// queries/todoKeys.js
export const todoKeys = {
  all: ["todos"],
  lists: () => [...todoKeys.all, "list"],
  list: (filters) => [...todoKeys.lists(), filters],
  details: () => [...todoKeys.all, "detail"],
  detail: (id) => [...todoKeys.details(), id],
};
```

```jsx
useQuery({
  queryKey: todoKeys.list({ status, page }),
  queryFn: () => fetchTodos({ status, page }),
});

queryClient.invalidateQueries({ queryKey: todoKeys.lists() });
```

Now the structure is one file, refactorable, and typo-proof.

### Details worth knowing

- Keys are hashed **deterministically**, and object key _order doesn't matter_: `['t', { a: 1, b: 2 }]` and `['t', { b: 2, a: 1 }]` are the same key. Array order **does** matter.
- Keys must be serializable — no functions, no class instances, no `Date` objects (use `.toISOString()`).
- Watch out for accidental key churn: an inline object rebuilt every render is fine (it's hashed by value), but an unstable value like `Date.now()` inside a key means a new cache entry every render.

**Say this:** _"The query key is the cache identity, the dependency array, and the invalidation selector all at once. I structure it hierarchically from general to specific so prefix matching gives me exactly the blast radius I want, and I centralize keys in a factory object so invalidation can't silently break from a typo."_

---

## Rapid-fire round

**Q: How is caching server state different from client state?**
Client state (modal open, form input) you own — it's always correct. Server state you're holding a _copy_ of something that can change without telling you, so it needs freshness policy, revalidation, and invalidation. That's why `useState` is the wrong tool for it.

**Q: Why not just use `useEffect` + `useState`?**
You'd hand-roll deduplication, caching, background refetch, retry with backoff, race-condition handling, and garbage collection — that's what the library is. Also, `useEffect` fetching has a classic race bug: two fetches in flight can resolve out of order and the slower/older one wins unless you write cleanup logic.

**Q: `invalidateQueries` vs `refetchQueries` vs `setQueryData`?**
Invalidate = mark stale, refetch what's on screen (default choice). Refetch = force it now regardless. setQueryData = write the cache directly with no network call.

**Q: Does React Query replace Redux/Zustand?**
For server state, yes — and it should. For genuine client state (wizard step, theme, selection), keep a client store. Most Redux codebases were 80% cached API responses.

**Q: What's structural sharing?**
On refetch, React Query diffs the new response against the cached one and keeps the old object references for unchanged parts. So a re-fetch that returns identical data doesn't break memoization or trigger re-renders downstream.

**Q: How do you cache-bust after a deploy?**
Client cache: `buster` in persist options. Next.js data cache: `revalidateTag` / a build-id-derived tag. HTTP: content-hashed filenames with long `max-age, immutable`.

**Q: A user reports seeing outdated data. How do you debug?**
Reach for React Query Devtools first — check whether the query is fresh/stale/inactive and when it last fetched. Then work through: is `staleTime` too high? Does the key include every input? Does the mutation actually invalidate the right prefix? Is a CDN or `Cache-Control` header caching it upstream of the app?

---

## The 30-second summary

| #   | Strategy                 | One line                                                       |
| --- | ------------------------ | -------------------------------------------------------------- |
| 1   | Request Deduplication    | Collapse concurrent identical requests into one shared promise |
| 2   | staleTime vs gcTime      | staleTime = network frequency; gcTime = memory lifetime        |
| 3   | Stale While Revalidate   | Show cached data instantly, refresh in the background          |
| 4   | Invalidation on Mutation | After writes, mark affected keys stale by prefix               |
| 5   | Server Component Caching | Next 16 is dynamic by default; opt in with `use cache`         |
| 6   | Prefetch on Intent       | Warm the cache on hover/focus/viewport before the click        |
| 7   | Persisted Query Cache    | Cache to storage for instant returns and offline; filter PII   |
| 8   | Query Key Structuring    | Hierarchical keys + a key factory = precise invalidation       |
