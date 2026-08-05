# React

> **In one line —** a library for describing what the UI should look like for a given state, and letting the framework work out how to get there.

| | |
|---|---|
| **Category** | UI Library |
| **Architectural Layer** | Client (and server, via RSC) |
| **Created by** | Meta, 2013 |
| **Related notes** | [DOM and CSSOM](DOM%20and%20CSSOM.md) · [State Management](State%20Management.md) · [Next.js](Next.js.md) · [TypeScript](TypeScript.md) · [Frontend Architecture](Frontend%20Architecture.md) |

---

## 1. Short Definition

*What is it?*

React is a JavaScript library for building user interfaces from **components** — functions that take data and return a description of what should be on screen. React compares that description with the previous one and updates the real [DOM](DOM%20and%20CSSOM.md) accordingly.

---

## 2. Purpose

*What is its main purpose?*

To make UI a **function of state**. You stop writing instructions for how to change the page and start declaring what it should look like, which removes an entire category of synchronisation bugs.

---

## 3. Problem

*What engineering problem does it solve?*

Manual DOM manipulation means keeping the page and your data in sync by hand. Every new piece of state multiplies the number of transitions you must handle.

```text
IMPERATIVE                          DECLARATIVE (React)

if (loggedIn) {                     return loggedIn
  showProfile();                      ? <Profile />
  hideLoginButton();                  : <LoginButton />;
} else {
  hideProfile();                    describe the result;
  showLoginButton();                React works out the transition
}
↑ every state combination by hand
```

---

## 4. Architecture Position

```text
State (data)
    ↓
Components  →  JSX  →  virtual tree
    ↓
Reconciliation (diff against the previous tree)
    ↓
Minimal real DOM updates
    ↓
Browser renders
```

---

## 5. The core ideas

| Concept | Meaning |
|---|---|
| **Component** | A function that takes props and returns UI |
| **Props** | Inputs, passed down, read-only |
| **State** | Data that changes over time and triggers re-render |
| **JSX** | HTML-like syntax that compiles to `React.createElement` calls |
| **Hooks** | Functions that add state and side effects to components |
| **Reconciliation** | Diffing the new tree against the old to find minimal changes |

---

## 6. Hooks

```javascript
function Counter() {
  const [count, setCount] = useState(0);          // state

  useEffect(() => {                                // side effect
    document.title = `Count: ${count}`;
  }, [count]);                                     // ← dependency array

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

| Hook | Purpose |
|---|---|
| `useState` | Local component state |
| `useEffect` | Side effects — subscriptions, DOM APIs, non-React systems |
| `useContext` | Read shared data without prop drilling |
| `useMemo` / `useCallback` | Cache expensive values or stable function identities |
| `useRef` | Mutable value that does not trigger re-render |

> [!CAUTION]
> `useEffect` is **not** a place to fetch data in modern React. Effects run after render, cause waterfalls, and are easy to get wrong. Use a data library ([React Query](State%20Management.md), SWR) or a framework's loader ([Next.js](Next.js.md)).

---

## 7. Re-rendering — the mental model people get wrong

```text
State changes in a component
    ↓
That component re-renders
    ↓
ALL of its children re-render by default
    ↓
React diffs the output and updates only what actually changed in the DOM
```

> [!IMPORTANT]
> "Re-render" means React called your function again — it does **not** mean the browser repainted. Most re-renders are cheap. Optimise only after measuring with the React Profiler; premature `memo` everywhere adds complexity and often makes things slower.

---

## 8. Real World Example

- **Meta, Netflix, Airbnb, Discord** — large production applications.
- **React Native** — the same component model producing native mobile apps.
- **Component libraries** (MUI, shadcn/ui, Chakra) exist because the component model makes UI genuinely reusable.

---

## 9. Input, Processing, Output

**Input:** props and state.
**Processing:** components produce a virtual tree; React reconciles it against the previous one.
**Output:** a minimal set of real DOM operations.

---

## 10. Communication and Dependencies

- **[DOM](DOM%20and%20CSSOM.md)** — what it ultimately updates
- **[State management](State%20Management.md)** — Context, Zustand, Redux, React Query
- **Router** — React Router, or [Next.js](Next.js.md)'s built-in routing
- **Build tooling** — Vite or a framework; JSX needs compiling
- **[TypeScript](TypeScript.md)** — near-universal in modern React projects

---

## 11. Alternatives

```text
React        largest ecosystem, virtual DOM, most jobs
    ↓
Vue          gentler learning curve, reactive by default
    ↓
Svelte       compiles away — no runtime framework in the bundle
    ↓
Solid        React-like syntax, fine-grained reactivity, no virtual DOM
    ↓
HTMX / vanilla   for pages that never needed a framework
```

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use React when the UI has substantial interactive state, many components share data, and the application will be maintained over time. Its ecosystem and hiring pool are genuine practical advantages.

> [!CAUTION]
> Do **not** use React for a mostly static site. Shipping a framework, hydrating it, and re-implementing links and forms in JavaScript is slower and worse than HTML that already worked. A marketing page does not need a virtual DOM.

---

## 13. Advantages and Disadvantages

**Advantages**
- Declarative — UI as a function of state
- Genuinely reusable components
- Enormous ecosystem and community
- Excellent developer tooling
- React Native shares the model with mobile

**Disadvantages**
- Bundle size — you ship the framework
- A build step and real tooling complexity
- Hooks have rules that are easy to violate subtly
- Easy to cause unnecessary re-renders
- Ecosystem churn — best practices change every few years

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Bundle** | ~45 KB for React + ReactDOM, before your code |
| **Runtime** | Reconciliation costs CPU on every update |
| **Memory** | The virtual tree is held alongside the real DOM |
| **First paint** | Slower than server-rendered HTML unless you use SSR |

---

## 15. Security Considerations

> [!IMPORTANT]
> React escapes interpolated values by default, which prevents most XSS automatically. `{userInput}` is safe.

> [!CAUTION]
> **`dangerouslySetInnerHTML` removes that protection** — the name is a deliberate warning. Sanitise with DOMPurify if you must render user-supplied HTML.

- **Never put secrets in client code** — everything in the bundle is public, including environment variables baked in at build time
- **Client-side route guards are not authorisation** — anyone can call your API directly; enforce on the server
- **`href={userInput}`** allows `javascript:` URLs; validate the scheme

---

## 16. Mental Model

> [!NOTE]
> **React is a photograph of the UI, taken again whenever the data changes.**
>
> You do not describe how to move things. You describe what the picture should look like now, and React works out which pixels actually differ from the last photograph — and only changes those.

---

## 17. Mini Architecture Diagram

```text
        State
          ↓
    Components (functions)
          ↓
        JSX
          ↓
   Virtual tree (new)  ──diff──  Virtual tree (previous)
          ↓
   Minimal DOM operations
          ↓
   Layout → Paint → Screen
```

---

## 18. Complete Request Flow

A button click in a React application:

```text
User clicks
    ↓
onClick handler calls setState
    ↓
React schedules a re-render
    ↓
Component function runs again → new virtual tree
    ↓
Reconciliation: diff old vs new
    ↓
Only changed nodes patched into the real DOM
    ↓
Browser: style → layout → paint → composite
    ↓
Screen updated
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> React makes the UI a function of state and handles the transitions for you — the cost is a shipped runtime, a build step, and the discipline of understanding when components re-render.

---

## 20. Common Mistakes

- **Fetching data in `useEffect`** instead of using a data library
- **Missing or wrong dependency arrays**, causing stale values or infinite loops
- **Mutating state directly** instead of creating new objects
- **Using array index as `key`** in a reorderable list
- **`memo`/`useMemo` everywhere** without measuring
- **Client-side route guards treated as security**
- **Reaching for React on a static site**

---

## 21. Open Source Technologies

- **React**, **React DOM**, **React Native**
- **Next.js**, **Remix** — frameworks built on it
- **TanStack Query**, **SWR** — server state
- **Zustand**, **Redux Toolkit**, **Jotai** — client state
- **React Testing Library**, **Playwright** — testing
- **React DevTools Profiler** — find unnecessary re-renders

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Profile a page of yours with React DevTools and find the component that re-renders most.
- [ ] Take one `useEffect` that fetches data and replace it with React Query or SWR.
- [ ] Explain in two sentences why "re-render" does not mean "repaint".

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
State
    ↓
Components → virtual tree
    ↓
Reconciliation
    ↓
DOM
    ↓
Browser rendering
```

## 2. Request Flow

```text
Input       props and state
    ↓
Processing  render → diff → patch
    ↓
Output      a minimal set of real DOM updates
```

## 3. Real-World Usage

**Discord** runs a highly interactive real-time client in React across web and desktop. Its component model is what makes an interface with thousands of live-updating elements maintainable by a team rather than by heroic effort.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A library for building UI as a function of state |
| **Why does it exist?** | Because manual DOM synchronisation does not scale with state |
| **Where does it belong?** | Between application state and the DOM |
| **When should I use it?** | Interactive, stateful applications — not static content sites |
