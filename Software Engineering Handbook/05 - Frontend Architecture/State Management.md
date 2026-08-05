# State Management

> **In one line —** deciding where each piece of data lives and who is allowed to change it; most frontend complexity is this question answered badly.

| | |
|---|---|
| **Category** | Architecture Pattern |
| **Architectural Layer** | Client |
| **Related notes** | [React](React.md) · [Frontend Architecture](Frontend%20Architecture.md) · [Next.js](Next.js.md) · [Cache](../08%20-%20Databases%20and%20Data/Cache.md) |

---

## 1. Short Definition

*What is it?*

State management is how an application stores, updates and shares the data that drives its UI — and, critically, how it keeps multiple views of the same data consistent.

---

## 2. Purpose

*What is its main purpose?*

To have **one source of truth** for each piece of data, so that two parts of the interface can never disagree about it.

---

## 3. Problem

*What engineering problem does it solve?*

```text
Header shows the cart count
Sidebar shows cart items
Checkout page shows the total
    ↓
Each keeps its own copy
    ↓
An item is removed → three places must be updated
    ↓
One is forgotten → the UI contradicts itself
```

---

## 4. The distinction that matters most

> [!IMPORTANT]
> **Server state and client state are different problems, and using one tool for both is the most common mistake in this area.**

| | Server state | Client state |
|---|---|---|
| **Owner** | Your backend | The browser |
| **Examples** | Users, orders, products | Modal open, form draft, theme |
| **Can go stale** | **Yes** | No |
| **Needs caching** | Yes | No |
| **Needs loading/error states** | Yes | No |
| **Right tool** | React Query, SWR, RTK Query | useState, Zustand, Context |

```text
Putting API data in Redux means hand-writing:
    loading flags · error handling · caching · refetching · invalidation
    ↓
React Query gives you all of that as its default behaviour
```

---

## 5. The scope ladder

Always start at the top and move down only when forced.

```text
1. Local component state       useState                    ← start here, always
    ↓ needed by a sibling?
2. Lift it to a common parent  props
    ↓ prop drilling 4+ levels?
3. Context                     for stable, rarely-changing data
    ↓ complex, frequently updated, shared widely?
4. Global store                Zustand, Redux Toolkit, Jotai
    ↓ it came from the server?
5. Server state library        React Query, SWR            ← different problem entirely
```

> [!TIP]
> Most applications need far less global state than they end up with. State that lives closest to where it is used is easier to reason about, easier to delete, and re-renders less.

---

## 6. URL as state

An underused option: the URL is state that survives refresh, is shareable, and works with the back button.

```text
/products?category=books&sort=price&page=2
```

Filters, tabs, pagination and search terms usually belong here rather than in a store.

---

## 7. Real World Example

- **React Query / TanStack Query** became the default because it correctly identified that most "state management" was really caching.
- **Redux Toolkit** remains standard in large applications with genuinely complex shared client state, and its DevTools time-travel debugging is still unmatched.
- **Zustand** is popular for being a store in a few lines with no boilerplate and no provider.

---

## 8. Input, Processing, Output

**Input:** user actions, server responses, URL changes.
**Processing:** state updated in one place; subscribed components re-render.
**Output:** a consistent UI wherever that data appears.

---

## 9. Communication and Dependencies

- **[React](React.md)** — or another framework — as the consumer
- **The network layer** — for server state
- **The URL / router** — for shareable state
- **localStorage / IndexedDB** — for persistence across sessions

---

## 10. Alternatives compared

| Tool | Best for | Cost |
|---|---|---|
| **useState** | Local, simple | Nothing |
| **Context** | Theme, locale, current user | Re-renders all consumers on change |
| **Zustand** | Shared client state, minimal ceremony | Small library |
| **Redux Toolkit** | Large apps, complex flows, debugging | Boilerplate, learning curve |
| **Jotai / Recoil** | Fine-grained atomic state | Different mental model |
| **React Query / SWR** | **All server data** | Learning curve, worth it |

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use a server-state library for anything fetched from an API — this is almost always the highest-value change a React codebase can make. Use a global store only for client state genuinely shared across distant parts of the app.

> [!CAUTION]
> - **Do not reach for Redux by default.** Many applications need no global store at all.
> - **Do not put server data in a global store** and hand-roll caching.
> - **Do not put everything in Context** — every consumer re-renders on every change.

---

## 12. Advantages and Disadvantages

**Advantages of deliberate state management**
- One source of truth; no contradictory UI
- Predictable, traceable updates
- Server state libraries remove enormous amounts of boilerplate
- Debugging tools that show exactly what changed and why

**Disadvantages**
- Every layer adds indirection
- Global stores make it easy to put things where they do not belong
- Over-engineering is the norm rather than the exception
- More libraries means more to learn and more to upgrade

---

## 13. Performance Impact

| Choice | Impact |
|---|---|
| **State too high in the tree** | Large subtrees re-render unnecessarily |
| **Context for frequently changing values** | All consumers re-render on every change |
| **Selector-based stores** | Only components using the changed slice re-render |
| **Server-state caching** | Removes duplicate network requests entirely |

---

## 14. Security Considerations

> [!CAUTION]
> **Client state is fully visible and fully editable.** Anyone can open DevTools, inspect your store and modify it. A `user.isAdmin` flag in client state is a UI hint, never an authorisation decision.

- **Never store secrets** in client state — it is all in the bundle and in memory
- **Never persist tokens to localStorage** if XSS is a realistic risk; `HttpOnly` cookies are safer
- **Server must re-check everything** the client believes about permissions
- **Be careful persisting state** — cached personal data survives logout unless you clear it explicitly

---

## 15. Mental Model

> [!NOTE]
> **State management is deciding where to keep each document in an office.**
>
> Something only you use stays on your desk (local state). Something your team shares goes in the team cabinet (store). And a copy of a document that actually lives in head office (server state) needs a rule for how often you check whether it has changed — that rule is caching, and it is a different problem from storage.

---

## 16. Mini Architecture Diagram

```text
        Server
          ↓  React Query / SWR  (cache, refetch, invalidate)
    Server state
          ↓
    ┌─────────────────────────────┐
    │        Components           │
    └─────────────────────────────┘
          ↑                ↑
    Client state        URL state
    (useState,          (filters, tabs,
     Zustand)            pagination)
```

---

## 17. Complete Request Flow

Adding an item to a cart:

```text
User clicks "Add to cart"
    ↓
Mutation fires (React Query)
    ↓
Optimistic update — UI responds immediately
    ↓
POST /cart sent to the server
    ↓
Success → invalidate the "cart" query
    ↓
Cart refetched; header, sidebar and checkout all update from ONE source
    ↓
Failure → optimistic update rolled back, error shown
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Server state and client state are different problems — use a caching library for anything from an API, keep everything else as local as it can possibly be.

---

## 19. Common Mistakes

- **Putting API data in Redux** and hand-writing caching
- **Reaching for a global store** before trying local state
- **Context for rapidly changing values**, causing cascading re-renders
- **Duplicating server data** into client state, then having two versions of the truth
- **Keeping filters and tabs in state** instead of the URL
- **Trusting client state for authorisation**
- **Persisting sensitive data** and forgetting to clear it on logout

---

## 20. Open Source Technologies

- **TanStack Query**, **SWR**, **RTK Query** — server state
- **Zustand**, **Jotai**, **Redux Toolkit**, **Valtio** — client state
- **React Context** — built in, for stable shared values
- **Redux DevTools** — inspect and time-travel through state changes
- **nuqs** — typed URL state for React

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] List every piece of state in one feature and classify it as server, client or URL state.
- [ ] Find one piece of global state that could be local, and move it.
- [ ] Replace one manual fetch-plus-loading-flag with React Query and compare the amount of code.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Server → server-state cache → components
                                 ↑
                    client state · URL state
```

## 2. Request Flow

```text
Input       a user action or a server response
    ↓
Processing  a single source of truth is updated; subscribers re-render
    ↓
Output      a consistent UI everywhere that data appears
```

## 3. Real-World Usage

**TanStack Query** changed how the React community thinks about this by pointing out that most global stores were being used as bad caches. Separating server state from client state removed thousands of lines of hand-written loading and error handling from typical codebases.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Deciding where data lives and who may change it |
| **Why does it exist?** | Because duplicated data produces contradictory interfaces |
| **Where does it belong?** | Between the network layer and your components |
| **When should I use it?** | Always — but start local and escalate only when forced |
