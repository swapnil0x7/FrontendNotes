# State Persistence in React Apps — SDE-2 Interview Prep

**State persistence** means keeping some piece of application state alive **beyond its normal lifetime** — surviving a page refresh, a closed tab, a lost network connection, or even a full app restart. By default, React state (`useState`, `useReducer`, Redux store, Context) lives only in memory: the moment the page reloads, it's gone. Persistence is the deliberate act of writing that state somewhere durable and reading it back.

This topic tests whether you know **where** to persist data, **when** to read/write it, and the **trade-offs** (security, size limits, sync timing, staleness) of each option — not just "use localStorage."

---

## 1. Where Can You Persist State? (Know All the Options)

| Storage                                  | Survives                                   | Size limit                | Sync/Async | Typical use                                            |
| ---------------------------------------- | ------------------------------------------ | ------------------------- | ---------- | ------------------------------------------------------ |
| **`localStorage`**                       | Tab close, browser restart                 | ~5–10MB                   | Sync       | Theme, auth token (with caveats), simple preferences   |
| **`sessionStorage`**                     | Page refresh, NOT tab close                | ~5–10MB                   | Sync       | Per-tab wizard/form state, one-time flags              |
| **Cookies**                              | Configurable (expiry date)                 | ~4KB                      | Sync       | Auth tokens sent to server automatically, SSR-readable |
| **IndexedDB**                            | Tab close, browser restart                 | Much larger (100s of MB+) | Async      | Offline-first data, large datasets, structured queries |
| **URL (query params / path)**            | Tab close, refresh, **shareable via link** | Practically limited       | N/A        | Filters, pagination, selected tab — anything shareable |
| **Server (DB via API)**                  | Everything, including device change        | Unlimited                 | Async      | The actual source of truth for real user data          |
| **Service Worker Cache API**             | Offline access                             | Large                     | Async      | Caching assets/responses for offline use               |
| **In-memory only (default React state)** | Nothing                                    | N/A                       | Sync       | Anything that _shouldn't_ survive a refresh            |

**Interview signal:** naming more than just `localStorage` — especially URL state and IndexedDB — shows you understand persistence isn't one-size-fits-all. The right choice depends on size, sensitivity, and whether it needs to be shareable or queryable.

---

## 2. localStorage / sessionStorage — the Basics

```js
// Writing
localStorage.setItem("theme", JSON.stringify({ mode: "dark" }));

// Reading (always guard — value may not exist, or may be malformed)
function readTheme() {
  try {
    const raw = localStorage.getItem("theme");
    return raw ? JSON.parse(raw) : { mode: "light" };
  } catch {
    return { mode: "light" }; // corrupted JSON, or storage disabled (private browsing)
  }
}
```

**Key facts to know:**

- Both APIs only store **strings** — you must `JSON.stringify`/`JSON.parse` objects yourself.
- Both are **synchronous** — reading/writing large amounts of data can block the main thread (a real, if usually small, performance consideration).
- `localStorage` is **shared across all tabs of the same origin**; `sessionStorage` is **isolated per tab** (even if it's the same origin, opening a new tab gives it fresh `sessionStorage`).
- Both throw if disabled (e.g. some private-browsing modes, or storage quota exceeded) — always wrap access in `try/catch`.
- Neither is available during server-side rendering (`window` doesn't exist on the server) — a common bug in Next.js apps that read `localStorage` directly during render.

---

## 3. Syncing React State with localStorage — the Custom Hook Pattern

This is a very common "write it live" interview exercise. Know the two directions of sync: **read on mount** (hydration) and **write on change** (persistence).

```jsx
import { useState, useEffect } from "react";

function usePersistedState(key, defaultValue) {
  const [value, setValue] = useState(() => {
    // Lazy initializer — runs ONCE, avoids reading storage on every render
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : defaultValue;
    } catch {
      return defaultValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// Usage
function App() {
  const [theme, setTheme] = usePersistedState("theme", "light");
  return (
    <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      Toggle ({theme})
    </button>
  );
}
```

**Two details interviewers probe:**

1. **Why the lazy initializer (`() => {...}`) instead of just `useState(localStorage.getItem(key))`?** Because passing a direct expression means it's evaluated on _every_ render (even though `useState` only uses the result on the first one) — the lazy function form guarantees `localStorage` is read exactly once, on mount.
2. **Why `useEffect` for the write, not directly in the event handler?** So _any_ state change — regardless of what triggered it — stays persisted, without having to remember to call `localStorage.setItem` at every call site that updates the state. It centralizes the sync logic in one place.

---

## 4. Redux + Persistence — `redux-persist`

For Redux apps, the standard library is **`redux-persist`**. Know roughly how it works, not just that it exists:

```js
import { persistStore, persistReducer } from "redux-persist";
import storage from "redux-persist/lib/storage"; // defaults to localStorage
import { combineReducers } from "redux";
import { configureStore } from "@reduxjs/toolkit";

const persistConfig = {
  key: "root",
  storage,
  whitelist: ["user", "settings"], // ONLY persist these slices
  // blacklist: ["notifications"] // or exclude specific ones instead
};

const rootReducer = combineReducers({
  user: userReducer,
  settings: settingsReducer,
  notifications: notificationsReducer,
});
const persistedReducer = persistReducer(persistConfig, rootReducer);

export const store = configureStore({ reducer: persistedReducer });
export const persistor = persistStore(store);
```

```jsx
import { PersistGate } from "redux-persist/integration/react";

<Provider store={store}>
  <PersistGate loading={<Spinner />} persistor={persistor}>
    <App />
  </PersistGate>
</Provider>;
```

**Key concepts to be able to explain:**

- **`whitelist`/`blacklist`** — you almost never want to persist the _entire_ store. Things like loading flags, transient UI state, or sensitive tokens usually shouldn't survive a refresh.
- **Rehydration** — the process of reading persisted state back into the store on app start. This is asynchronous (reading from storage takes a tick), which is why `PersistGate` exists: it blocks rendering the real app until rehydration completes, showing a loading state in the meantime, so components don't briefly render with default/empty state before flashing to the real persisted state.
- **Migrations/versioning** — if you change your state shape (e.g. rename a field) between app versions, old persisted data can be incompatible with new reducers. `redux-persist` supports a `version` + `migrate` function to transform old persisted shapes into the new one, rather than crashing or silently discarding user data.

```js
const persistConfig = {
  key: "root",
  storage,
  version: 2,
  migrate: (state) => {
    if (state && state._persist.version < 2) {
      return Promise.resolve({
        ...state,
        settings: { ...state.settings, newField: "default" },
      });
    }
    return Promise.resolve(state);
  },
};
```

This connects directly to the Redux sync topic we covered earlier — persistence is really just another form of keeping the store in sync, except the "other side" is disk storage instead of a server or another tab.

---

## 5. URL State — the Most Underrated Persistence Mechanism

A lot of candidates forget this one, and it's often the _correct_ answer for filters, tabs, pagination, and search queries — because it's persistent **and shareable**.

```jsx
import { useSearchParams } from "react-router-dom";

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get("category") ?? "all";

  function handleCategoryChange(newCategory) {
    setSearchParams({ category: newCategory }); // updates the URL, triggers a re-render
  }

  return <CategoryFilter value={category} onChange={handleCategoryChange} />;
}
```

**Why this beats `localStorage` for this use case:**

- Refreshing the page keeps the filter (persistence ✅).
- **Sharing the link** gives someone else the exact same filtered view — `localStorage` can't do that, it's local to your machine.
- Back/forward browser buttons work naturally, because the URL is part of browser history.

**Trade-off to mention:** URL state is public and visible (bookmarks, browser history, server logs) — never put sensitive data there (tokens, PII).

---

## 6. IndexedDB — for Larger or Offline-First Data

Know _when_ to reach for this instead of `localStorage`: when data is large (beyond a few MB), needs to be queried (indexes, ranges), or needs to work fully offline (PWAs). Most people don't hand-roll raw IndexedDB — they use a wrapper like **Dexie.js** or **idb**.

```js
import { openDB } from "idb";

const dbPromise = openDB("my-app-db", 1, {
  upgrade(db) {
    db.createObjectStore("notes", { keyPath: "id" });
  },
});

async function saveNote(note) {
  const db = await dbPromise;
  await db.put("notes", note);
}

async function getAllNotes() {
  const db = await dbPromise;
  return db.getAll("notes");
}
```

**Interview talking point:** IndexedDB is asynchronous and non-blocking (unlike `localStorage`), which matters for larger datasets — you don't want to freeze the main thread reading/writing megabytes of data synchronously.

---

## 7. Persisting React Query / TanStack Query Cache

If the role touches server-state libraries, know that TanStack Query supports persisting its entire cache so the app can show data instantly on reload instead of a loading spinner, then silently refetch in the background:

```js
import { QueryClient } from "@tanstack/react-query";
import { persistQueryClient } from "@tanstack/react-query-persist-client";
import { createSyncStoragePersister } from "@tanstack/query-sync-storage-persister";

const queryClient = new QueryClient();
const persister = createSyncStoragePersister({ storage: window.localStorage });

persistQueryClient({ queryClient, persister, maxAge: 1000 * 60 * 60 * 24 }); // 24h
```

This is conceptually the same "stale-while-revalidate" idea as caching in general — persisted cache is shown immediately (possibly stale), then a background refetch brings it current. This ties back to the cache-invalidation discussion from your Redux sync notes.

---

## 8. Security Considerations (Frequently the Real Point of the Question)

This is where many candidates lose points — persistence isn't just "does it work," it's "is it safe."

- **Never store sensitive tokens in `localStorage`/`sessionStorage` if you can avoid it** — both are readable by any JS running on the page, so they're vulnerable to **XSS**: if an attacker injects a script, they can read and exfiltrate everything in storage.
- **`httpOnly` cookies** are the standard safer alternative for auth tokens — JavaScript cannot read `httpOnly` cookies at all, which closes off the XSS attack vector for token theft (though cookies bring their own concern: CSRF, mitigated with `SameSite` and CSRF tokens).
- **Don't persist PII unnecessarily.** Every piece of persisted data is something that lives on the user's disk indefinitely (until cleared) — minimize what you write.
- **Clear persisted state on logout.** A very common real bug: a user logs out, but stale persisted Redux state (or React Query cache) still contains their previous session's data, briefly visible to the next person who uses that browser/device.

```js
function logout() {
  dispatch({ type: "auth/loggedOut" });
  persistor.purge(); // redux-persist API to wipe persisted state
  localStorage.clear();
}
```

---

## 9. Server-Side Rendering (SSR) Gotchas

If the interview touches Next.js/Remix, this is a near-guaranteed follow-up:

- `localStorage`/`sessionStorage`/`window` **don't exist on the server.** Reading them directly during the render phase throws (`window is not defined`) or, in the browser, causes a **hydration mismatch** — the server-rendered HTML (with default state) doesn't match what the client immediately renders (with persisted state), and React logs a hydration warning.

```jsx
// ❌ Breaks SSR / causes hydration mismatch
function App() {
  const [theme, setTheme] = useState(localStorage.getItem("theme") || "light");
  ...
}

// ✅ Read persisted state only after mount, so server and first client render match
function App() {
  const [theme, setTheme] = useState("light"); // same on server and first client render
  useEffect(() => {
    const stored = localStorage.getItem("theme");
    if (stored) setTheme(stored);
  }, []);
  ...
}
```

The trade-off: the "real" persisted value only appears after mount, so there's a brief flash of default state (a well-known problem called **FOUC-like flash** for theme toggles specifically — often solved with a small inline `<script>` in the HTML `<head>` that sets the theme class _before_ React even loads).

---

## 10. Common Interview Questions to Rehearse

1. **"How would you persist form data so a user doesn't lose progress on refresh?"**
   → `sessionStorage` (per-tab, cleared on tab close, appropriate for "don't lose this half-filled form") synced via a `useEffect` on every field change, or a debounced write to avoid excessive writes on every keystroke.

2. **"Where would you store an auth token?"**
   → Ideally an `httpOnly` cookie set by the server (immune to XSS token theft). If forced to use client-side storage, understand and be able to state the XSS trade-off of `localStorage` vs the CSRF trade-off of cookies.

3. **"How do you avoid a flash of incorrect UI when persisted state loads asynchronously?"**
   → `PersistGate`-style loading gate (Redux), or a `useEffect`-based hydration with a loading flag, or (for SSR) an inline script that sets initial state before hydration.

4. **"What's the difference between `localStorage` and `sessionStorage`, and when would you pick one over the other?"**
   → Both are ~5–10MB, string-only, synchronous. `localStorage` persists indefinitely and is shared across tabs; `sessionStorage` is cleared when the tab closes and is isolated per tab — use it for anything that genuinely shouldn't outlive that one tab session (e.g. a multi-step signup wizard).

5. **"Your persisted Redux state has an old shape from a previous app version — what happens, and how do you handle it?"**
   → Without a migration strategy, the reducer can crash or silently misbehave against unexpected shape. `redux-persist`'s `version` + `migrate` (or a manual "shape check + transform on load" step) handles this gracefully.

6. **"Why would you put state in the URL instead of localStorage?"**
   → Shareability and back/forward navigation — URL state can be sent to someone else and reproduces the exact same view; `localStorage` is local to the browser and can't be shared via a link.

---

## Quick Mental Model to Recite

> "Persistence = picking the right storage for the data's size, sensitivity, and shareability (localStorage/sessionStorage for small client-only data, cookies for anything the server needs, IndexedDB for large/offline data, URL for shareable UI state) → syncing it with React state via a lazy initializer on read and a `useEffect` on write → gating render until rehydration completes to avoid a flash of default state → versioning/migrating the persisted shape as the app evolves → and treating what you persist as a security decision, not just a convenience one."

If you can walk through each clause with a concrete example, you're well prepared for this topic.
