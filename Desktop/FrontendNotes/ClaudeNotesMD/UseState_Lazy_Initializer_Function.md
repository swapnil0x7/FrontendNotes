# `useState` Lazy Initializer — Interview Notes

## The One-Line Definition

`useState` accepts either a **value** or a **function that returns a value**. If you pass a function, React calls it **only once, on the very first render**, and uses the return value as the initial state. On every subsequent render, that function is never called again.

```js
useState(expensiveValue());     // ❌ eager — evaluated on EVERY render
useState(() => expensiveValue()); // ✅ lazy — evaluated ONCE, on mount only
```

---

## Why This Matters: The Core Problem

JavaScript must fully evaluate an expression **before** it can be passed as an argument to a function call. So with the eager form, `expensiveValue()` runs on every re-render — even though `useState` only *uses* that value on the first render and silently discards it every time after.

```jsx
function App() {
  const [value, setValue] = useState(localStorage.getItem("theme")); // runs on EVERY render
  return <button onClick={() => setValue("x")}>Click</button>;
}
```

Click the button 100 times → `localStorage.getItem` executes 100 times, but only the very first call's result was ever actually used. The other 99 reads were pure waste.

With the lazy form, you pass React a **function reference**, not a computed value. React is specifically built to detect this: if the argument is a function, it calls it once (on mount) and then never touches it again.

```jsx
function App() {
  const [value, setValue] = useState(() => {
    console.log("computing initial value"); // logs exactly ONCE, ever
    return localStorage.getItem("theme");
  });
  return <button onClick={() => setValue("x")}>Click</button>;
}
```

---

## Simple Analogy

- `useState(cookDinner())` — you cook the full dinner and hand it to the waiter *every time* you call them, even though after the first time they already have your plate and will ignore anything new.
- `useState(() => cookDinner())` — you hand the waiter a **recipe card**. They only actually cook from it the first time. After that, they already have the plate and never look at the card again.

---

## When It Actually Matters

Use the lazy form when computing the initial value is **non-trivial** — otherwise the difference is invisible:

| Cheap (lazy form unnecessary) | Expensive (lazy form matters) |
|---|---|
| `useState(0)` | `useState(() => JSON.parse(localStorage.getItem(key)))` |
| `useState("")` | `useState(() => expensiveFilter(bigArray))` |
| `useState(false)` | `useState(() => new Map(largeInitialEntries))` |
| `useState(null)` | `useState(() => heavyComputation(props))` |

For a plain literal like `0` or `""`, evaluating it on every render costs nothing measurable — the lazy form is optional there, and the plain form is more concise.

---

## Real-World Example: Persisted State Hook

This is the pattern that shows up constantly in real apps and interview live-coding:

```jsx
function usePersistedState(key, defaultValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : defaultValue;
    } catch {
      return defaultValue; // storage disabled, or corrupted JSON
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}
```

Without the lazy initializer, every re-render of any component using this hook would re-read and re-parse `localStorage` — wasted work that only gets thrown away, since `useState` already has the real value after mount.

---

## Same Pattern Applies to `useReducer`

`useReducer` supports the identical optimization via its third argument — an initializer function applied to the second argument:

```js
function init(initialCount) {
  console.log("computing initial state"); // runs ONCE
  return { count: initialCount };
}

const [state, dispatch] = useReducer(reducer, initialCountProp, init);
```

This avoids recomputing a potentially expensive initial state object on every render, exactly like the `useState` lazy form.

---

## Common Interview Questions

1. **"What's the difference between `useState(getValue())` and `useState(() => getValue())`?"**
   → The first calls `getValue()` on every render (result discarded after mount); the second calls it exactly once, on the first render only.

2. **"When would you actually need the lazy form?"**
   → Whenever computing the initial value has a real cost: reading/parsing `localStorage`, filtering or transforming a large array, running any loop or heavy computation, or instantiating something like a `Map`/`Set` from a large source.

3. **"Does the lazy initializer re-run if the component re-renders due to a prop change?"**
   → No. It only ever runs once, on mount, for the lifetime of that component instance. It re-runs only if the component fully unmounts and remounts (e.g. a `key` change resets it).

4. **"Is there a performance cost to always using the lazy form, even for cheap values?"**
   → Negligible — you're trading a cheap expression evaluation for a cheap function call. It's not wrong to always use it, but it's unnecessary boilerplate for trivial defaults like `useState(0)`.

5. **"Does this apply anywhere else in React besides `useState`?"**
   → Yes — `useReducer`'s third-argument initializer function follows the exact same lazy-evaluation idea, for the same reason (avoid recomputing expensive initial state on every render).

---

## One-Sentence Recap to Recite

> "Pass `useState` a function instead of a value when computing that initial value is expensive, so React only pays that cost once on mount instead of on every render where the result would just be thrown away."