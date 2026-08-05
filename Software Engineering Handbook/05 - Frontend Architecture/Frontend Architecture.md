# Frontend Architecture

> **In one line —** how a frontend codebase is organised so that adding the hundredth feature is no harder than adding the tenth.

| | |
|---|---|
| **Category** | Architecture Discipline |
| **Architectural Layer** | Client |
| **Related notes** | [React](React.md) · [State Management](State%20Management.md) · [Next.js](Next.js.md) · [Clean Architecture](../07%20-%20Backend%20Design%20Patterns/Clean%20Architecture.md) · [Architecture Principles](../01%20-%20Foundation/08%20-%20Architecture%20Principles.md) |

---

## 1. Short Definition

*What is it?*

Frontend architecture is the set of decisions about folder structure, component boundaries, data flow, state ownership and rendering strategy that determine whether a UI codebase stays workable as it grows.

---

## 2. Purpose

*What is its main purpose?*

To keep the cost of change flat. A well-architected frontend lets you find any feature quickly, change it without breaking three others, and delete it cleanly when it is no longer needed.

---

## 3. Problem

*What engineering problem does it solve?*

Frontends decay in a recognisable pattern: components grow to a thousand lines, state ends up in the wrong place, everything imports everything, and nobody dares delete anything because they cannot tell what uses it.

---

## 4. Folder structure — the first real decision

```text
BY TYPE  (common, scales badly)      BY FEATURE  (scales well)

/components   ← 200 files            /features
/hooks        ← 80 files                /checkout
/utils        ← everything              /orders
/services                               /products
                                     /shared
                                        /ui
                                        /lib
```

> [!TIP]
> **Organise by feature, not by file type.** A change to checkout should touch one folder. With type-based folders, one feature is scattered across five directories and deleting it safely becomes archaeology.

---

## 5. The layers

```text
PAGES / ROUTES        composition and routing only
    ↓
FEATURES              a business capability: components + hooks + logic
    ↓
SHARED UI             design system primitives — Button, Modal, Input
    ↓
LIB / API             HTTP client, formatting, validation
    ↓
Backend
```

> [!IMPORTANT]
> **Dependencies point downward only.** A shared Button must never import from a feature. The moment that rule breaks, the folder structure is decoration rather than architecture — and it can be enforced automatically with lint rules.

---

## 6. Component design

```text
PRESENTATIONAL          takes props, renders UI, no data fetching
CONTAINER / FEATURE     fetches data, holds state, composes presentational parts
```

Practical heuristics:

- If a component exceeds ~200 lines, it is probably doing two things
- If you cannot name it clearly, its responsibility is unclear
- If it takes more than ~7 props, it likely wants splitting or a config object

---

## 7. Where data lives

Covered in depth in [State Management](State%20Management.md); the architectural summary:

```text
Server data     → a caching library (React Query), never a global store
Client state    → as local as possible; global only when genuinely shared
URL state       → filters, tabs, pagination, search
Persisted       → localStorage/IndexedDB, cleared deliberately on logout
```

---

## 8. Rendering strategy

An architectural decision, not a framework detail:

```text
Static (SSG)      content that rarely changes         fastest, cheapest
Server-rendered   fresh or personalised, needs SEO    server cost per request
Client-rendered   behind a login, no SEO needed       simplest to deploy
```

See [Next.js](Next.js.md).

---

## 9. Real World Example

- **Design systems** (Material, Carbon, in-house) exist because shared UI primitives are the only way large teams stay visually consistent.
- **Micro-frontends** let independent teams deploy parts of one page separately — genuinely useful at large organisational scale, and usually overkill below it.
- **Monorepos** (Turborepo, Nx) allow web, mobile and shared packages to live together with shared types.

---

## 10. Boundaries and dependencies

```text
✓ feature → shared        allowed
✓ page    → feature       allowed
✗ shared  → feature       forbidden — inverts the hierarchy
✗ feature → feature       usually a sign of a missing shared module
```

> [!TIP]
> Enforce this mechanically with ESLint import rules. A convention nobody can violate accidentally is worth more than a document describing the convention.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Apply real structure from the point a project has more than one developer or is expected to live longer than a few months. Retrofitting boundaries is far more expensive than starting with them.

> [!CAUTION]
> Do not build a five-layer architecture for a three-page site. Abstraction has a cost paid at every future change; a small project pays it without ever collecting the benefit.

---

## 12. Advantages and Disadvantages

**Advantages of deliberate structure**
- Features are easy to find, change and delete
- Parallel work with fewer conflicts
- Reuse happens by design rather than by accident
- Onboarding is faster because the shape is predictable

**Disadvantages**
- Upfront thinking before any visible progress
- Over-abstraction is a real and common failure
- Structure needs maintaining as the product changes
- Rules require enforcement, or they quietly stop being true

---

## 13. Performance as an architectural concern

| Decision | Effect |
|---|---|
| **Route-level code splitting** | Users download only the page they visit |
| **Server components** | Rendering code never reaches the browser |
| **Image optimisation** | Usually the largest bytes on the page |
| **Dependency discipline** | One heavy library can outweigh all your own code |
| **Bundle budgets in CI** | Prevents slow, invisible growth |

> [!IMPORTANT]
> Frontend performance is decided by architecture far more than by micro-optimisation. What you ship matters more than how tight the loops are.

---

## 14. Security Considerations

> [!CAUTION]
> **Everything in the frontend is public.** The bundle, environment variables baked into it, API endpoints, and all client state are visible to anyone with DevTools. There is no such thing as a client-side secret.

- **Client-side route guards are UX, not authorisation** — the server must enforce every rule
- **Escape all rendered user content**; avoid `dangerouslySetInnerHTML`
- **Content Security Policy** limits the damage of any XSS that does occur
- **Dependencies are your attack surface** — audit and pin them
- **Do not send data the user may not see**, then hide it with CSS

---

## 15. Mental Model

> [!NOTE]
> **Frontend architecture is town planning.**
>
> Anyone can build one house. The planning decides whether the hundredth house still has road access, whether utilities reach it, and whether demolishing one building brings down its neighbours. Nobody notices good planning; everybody notices its absence.

---

## 16. Mini Architecture Diagram

```text
        Routes / Pages
              ↓
    ┌─────────┼─────────┐
 Feature   Feature   Feature
    └─────────┼─────────┘
              ↓
        Shared UI  (design system)
              ↓
        Lib / API client
              ↓
      ━━━ network boundary ━━━
              ↓
           Backend
```

---

## 17. Complete Request Flow

Opening a page in a well-structured application:

```text
Route matched
    ↓
Route-level code chunk loaded  ← only this page's JavaScript
    ↓
Page composes feature components
    ↓
Feature triggers a query via the API layer
    ↓
Server-state cache: fresh?  → render immediately
    ↓ stale
Fetch → cache → render
    ↓
Shared UI components render the result
    ↓
User interacts → local state changes → only that subtree re-renders
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Organise by feature, let dependencies point one way only, keep state as local as possible, and choose the rendering strategy deliberately — those four decisions determine whether a frontend stays maintainable.

---

## 19. Common Mistakes

- **Organising by file type** instead of by feature
- **No dependency rules**, so everything imports everything
- **Global state as a default** rather than a last resort
- **Components that fetch, transform and render** all at once
- **A `utils` folder** that becomes a landfill
- **No bundle budget**, so size grows unnoticed until it is a crisis
- **Treating client-side guards as security**
- **Copying a large-company architecture** without their constraints

---

## 20. Open Source Technologies

- **Turborepo**, **Nx** — monorepo tooling
- **Storybook** — develop and document shared UI in isolation
- **ESLint import rules**, **dependency-cruiser** — enforce boundaries mechanically
- **Vite**, **Next.js** — build and rendering
- **Lighthouse CI**, **bundlesize** — performance budgets in CI

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Draw your current folder structure and mark every dependency that points the wrong way.
- [ ] Pick one feature and check whether deleting it entirely would be a single-folder operation.
- [ ] Measure your bundle size per route and find the largest single dependency.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Pages → Features → Shared UI → Lib/API → Backend
```

## 2. Request Flow

```text
Input       a route and user intent
    ↓
Processing  composed from features, data fetched through one API layer
    ↓
Output      a rendered page, and a codebase that survives the next feature
```

## 3. Real-World Usage

**Design systems** at large companies exist to solve exactly this problem: without a shared UI layer with enforced boundaries, twenty teams produce twenty subtly different buttons and no one can change any of them safely.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The structural decisions behind a frontend codebase |
| **Why does it exist?** | Because unstructured UI code becomes unchangeable |
| **Where does it belong?** | Above components, below the product requirements |
| **When should I use it?** | From the point a project has a second developer or a second month |
