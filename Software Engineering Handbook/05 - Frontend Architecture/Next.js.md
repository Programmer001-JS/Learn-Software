# Next.js

> **In one line —** React with a server attached: it decides, per page, whether HTML should be built at build time, on the server per request, or in the browser.

| | |
|---|---|
| **Category** | Full-stack Framework |
| **Architectural Layer** | Client + Server |
| **Built on** | React |
| **Related notes** | [React](React.md) · [Frontend Architecture](Frontend%20Architecture.md) · [State Management](State%20Management.md) · [Backend Fundamentals](../06%20-%20Backend%20Architecture/Backend%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

Next.js is a framework built on [React](React.md) that adds server-side rendering, static generation, file-based routing, API endpoints and a build pipeline. It blurs the line between frontend and backend deliberately.

---

## 2. Purpose

*What is its main purpose?*

To let each page choose **where and when** its HTML is produced, so that content-heavy pages are fast and indexable while interactive pages remain fully dynamic.

---

## 3. Problem

*What engineering problem does it solve?*

A pure client-side React application sends an empty HTML shell and then downloads a large bundle before anything appears.

```text
CLIENT-SIDE ONLY                    SERVER-RENDERED

<div id="root"></div>               <h1>Full content</h1>
    ↓                                   ↓
download 300 KB of JS               visible immediately
    ↓                                   ↓
execute, fetch data                 hydrate for interactivity
    ↓
finally render
↑ slow on mobile, poor for SEO
```

---

## 4. The rendering strategies

This is the core of Next.js, and the thing worth understanding properly.

| Strategy | HTML built | Best for |
|---|---|---|
| **SSG** — Static Generation | At build time | Blogs, docs, marketing pages |
| **ISR** — Incremental Static Regeneration | At build, then refreshed periodically | Product catalogues, news |
| **SSR** — Server-Side Rendering | On every request | Personalised or always-fresh pages |
| **CSR** — Client-Side Rendering | In the browser | Dashboards behind a login |

```text
Does the content change per user?
    ↓ NO                                    ↓ YES
Does it change often?                   Is SEO needed?
    ↓ NO → SSG                              ↓ YES → SSR
    ↓ YES → ISR                             ↓ NO  → CSR
```

---

## 5. Server Components

Modern Next.js (App Router) makes components **server-side by default**.

```text
SERVER COMPONENT                     CLIENT COMPONENT  ('use client')
runs on the server only              runs in the browser
can query the database directly      can use state, effects, event handlers
ships ZERO JavaScript to the client  ships JavaScript
cannot use useState or onClick       cannot touch the database
```

> [!IMPORTANT]
> The point is bundle size. A server component that renders a product list sends **HTML only** — the rendering code never reaches the browser. This is a genuinely different model from classic React, and it is where most of the learning curve now sits.

---

## 6. Architecture Position

```text
Browser
    ↓
Next.js server / edge runtime
    ├─ static files (CDN)
    ├─ server components → HTML
    ├─ route handlers → API endpoints
    └─ server actions → mutations
    ↓
Your database / external services
```

> [!TIP]
> Next.js is a **real backend**. Route handlers and server actions can talk to a database directly, which means a small application may not need a separate API service at all.

---

## 7. Real World Example

- **Vercel, Notion, TikTok, Twitch** and a large share of modern marketing sites run on it.
- **E-commerce** is the archetypal fit: product pages statically generated or ISR-refreshed for SEO, cart and checkout fully dynamic.
- **Documentation sites** — SSG produces plain HTML that loads instantly and indexes perfectly.

---

## 8. Input, Processing, Output

**Input:** a route, plus data from your database or APIs.
**Processing:** the chosen rendering strategy produces HTML on the server or at build time; the client hydrates only the interactive parts.
**Output:** HTML that is immediately visible, plus the minimum JavaScript needed for interactivity.

---

## 9. Communication and Dependencies

- **React** — it is a framework *on* React, not an alternative to it
- **Node.js or an edge runtime** — server components need a server
- **A database or API** — reached directly from server code
- **A CDN** — static assets and cached pages
- **Vercel or self-hosting** — it runs anywhere Node runs, though some features are smoothest on Vercel

---

## 10. Alternatives

```text
Next.js       React, most features, largest ecosystem
    ↓
Remix         React, web-standards-first, excellent form handling
    ↓
Astro         content-first; ships almost no JavaScript by default
    ↓
Nuxt          the Vue equivalent
    ↓
SvelteKit     Svelte equivalent, very small bundles
    ↓
Vite + React  no server rendering; simplest for pure dashboards
```

> [!TIP]
> For a content site where interactivity is rare, **Astro** often produces a dramatically faster result than Next.js, because it ships no framework at all by default.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Next.js when you need SEO or fast first paint *and* real interactivity — marketing plus app, e-commerce, content platforms.

> [!CAUTION]
> - **A dashboard entirely behind a login** does not need SSR; plain Vite + React is simpler and cheaper to run.
> - **A separate backend team with an existing API** may find Next.js's backend features redundant and confusing.
> - **Self-hosting** is possible but noticeably more work than the Vercel path — check this before committing if you must run it yourself.

---

## 12. Advantages and Disadvantages

**Advantages**
- Per-page choice of rendering strategy
- Server components genuinely reduce shipped JavaScript
- File-based routing with no configuration
- Built-in image, font and script optimisation
- API routes remove the need for a separate service in small projects

**Disadvantages**
- Substantial complexity — App Router, server/client boundaries, caching layers
- Caching behaviour has been a recurring source of confusion
- Some coupling to Vercel's deployment model
- Frequent breaking changes between major versions
- A server to run and pay for, unless fully static

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **First paint** | Much faster with SSG/SSR than client-only React |
| **Bundle** | Smaller with server components, but the framework is still substantial |
| **Server cost** | SSR means CPU per request; SSG means none |
| **SEO** | Real HTML for crawlers rather than an empty shell |

---

## 14. Security Considerations

> [!CAUTION]
> The server/client boundary is a **security boundary**, and it is easy to cross by accident. Anything imported into a client component ships to the browser — including secrets that happened to be in that module.

- **`NEXT_PUBLIC_` environment variables are public**, baked into the bundle at build time. Everything else stays server-side — know which is which.
- **Server actions are HTTP endpoints.** They look like function calls but anyone can invoke them; authenticate and authorise inside every one.
- **Never trust client-side route protection** — enforce in middleware and in the data layer
- Validate all input in route handlers exactly as you would in any API

---

## 15. Mental Model

> [!NOTE]
> **Next.js is a restaurant that can serve you three ways.**
>
> **SSG** is a pre-made dish from the display case — instant. **SSR** is cooked to order when you arrive — fresh, but you wait. **CSR** is a kit you assemble at the table. Next.js lets you choose per dish rather than committing the whole menu to one style.

---

## 16. Mini Architecture Diagram

```text
Request
    ↓
┌──────── Next.js ────────┐
│  Middleware (auth)      │
│  Server components      │ → query the database directly
│  Route handlers (API)   │
│  Server actions         │
└───────────┬─────────────┘
            ↓
      HTML + minimal JS
            ↓
      Browser hydrates interactive parts
```

---

## 17. Complete Request Flow

A product page:

```text
Request /products/42
    ↓
Middleware runs (auth, redirects, headers)
    ↓
Cached static version available?  → serve from CDN, ~10 ms
    ↓ no
Server component runs on the server
    ↓
Queries the database directly — no API round trip
    ↓
HTML rendered and streamed to the browser
    ↓
Content visible immediately          ← crawlers see this too
    ↓
Client components hydrate: cart button becomes interactive
    ↓
User clicks "Add to cart" → server action → database → UI updates
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Next.js lets each page choose where its HTML is produced — and server components mean the code that renders your content never has to reach the browser at all.

---

## 19. Common Mistakes

- **Marking everything `'use client'`**, discarding the entire benefit
- **Leaking secrets into client components** through a shared import
- **Treating server actions as trusted** — they are public endpoints
- **Misunderstanding the caching layers** and serving stale data
- **Using SSR for a page that could be static**, paying server cost for nothing
- **Choosing Next.js for a login-only dashboard** where it adds complexity without benefit

---

## 20. Open Source Technologies

- **Next.js** — the framework
- **Remix**, **Astro**, **SvelteKit**, **Nuxt** — the main alternatives
- **Vercel**, **Netlify**, **Docker + Node** — deployment targets
- **next/image**, **next/font** — built-in asset optimisation

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Take one page of your project and decide which of SSG, ISR, SSR or CSR it should use, with a reason.
- [ ] Find every `'use client'` in a codebase and check whether each is genuinely necessary.
- [ ] Verify that one server action authenticates the caller rather than assuming the UI prevented misuse.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Browser
    ↓
CDN (static / cached)
    ↓
Next.js server (server components, route handlers)
    ↓
Database
```

## 2. Request Flow

```text
Input       a route request
    ↓
Processing  static, regenerated, server-rendered or client-rendered per page
    ↓
Output      HTML plus the minimum JavaScript needed for interactivity
```

## 3. Real-World Usage

**E-commerce sites** are the clearest case: product pages must be indexable and load instantly, while cart and checkout must be fully dynamic and personalised. Next.js allows both in one application without splitting it into two.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A React framework with server rendering, routing and backend capabilities |
| **Why does it exist?** | Because client-only React is slow to first paint and poor for SEO |
| **Where does it belong?** | Between the browser and your data, spanning both |
| **When should I use it?** | When you need SEO or fast first paint alongside real interactivity |
