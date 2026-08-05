# Prisma

> **In one line —** define your schema in one file and get a fully typed database client generated from it, so wrong queries fail at compile time.

| | |
|---|---|
| **Category** | ORM / Query Builder |
| **Architectural Layer** | Data access |
| **Language** | TypeScript / JavaScript |
| **Related notes** | [ORM](ORM.md) · [TypeScript](../05%20-%20Frontend%20Architecture/TypeScript.md) · [PostgreSQL](PostgreSQL.md) · [NestJS](../06%20-%20Backend%20Architecture/NestJS.md) · [Next.js](../05%20-%20Frontend%20Architecture/Next.js.md) |

---

## 1. Short Definition

*What is it?*

Prisma is a TypeScript ORM built around code generation. You declare models in a `schema.prisma` file, and Prisma generates a client whose types exactly match your database schema.

---

## 2. Purpose

*What is its main purpose?*

To close the gap between the database schema and the application's types. In most stacks these are two separate truths that drift; in Prisma the schema is the single source and the types are derived from it.

---

## 3. Problem

*What engineering problem does it solve?*

```text
WITHOUT                               WITH PRISMA
schema in migrations                  schema.prisma  ← one source of truth
types hand-written in TS                  ↓ generate
    ↓                                 fully typed client
someone adds a column                     ↓
types not updated                     wrong field name = compile error
runtime error in production           result type inferred from the query
```

---

## 4. Architecture Position

```text
schema.prisma          ← models, relations, indexes
    ↓  prisma generate
@prisma/client         ← typed, generated TypeScript
    ↓
Your application
    ↓
Prisma query engine
    ↓
PostgreSQL / MySQL / SQLite / MongoDB
```

---

## 5. The schema file

```prisma
model User {
  id     Int     @id @default(autoincrement())
  email  String  @unique
  orders Order[]
}

model Order {
  id     Int   @id @default(autoincrement())
  total  Decimal
  user   User  @relation(fields: [userId], references: [id])
  userId Int

  @@index([userId])
}
```

From this, Prisma generates the client, the migrations, and the TypeScript types — all consistent by construction.

---

## 6. Queries are typed end to end

```typescript
const orders = await prisma.order.findMany({
  where:   { userId: 42, total: { gt: 100 } },
  include: { user: true },          // ← eager loading, explicit
  orderBy: { createdAt: 'desc' },
  take: 20,
});

orders[0].user.email    // ✓ typed — because `include` was used
orders[0].items         // ✗ compile error — not included
```

> [!IMPORTANT]
> That last line is the real value. The **result type changes based on the query you wrote**. Forgetting to include a relation is a compile error, not a runtime `undefined` — which eliminates a whole category of bugs other ORMs leave to tests.

---

## 7. No lazy loading — deliberately

Prisma has no lazy loading at all. Every relation must be explicitly `include`d or `select`ed.

```text
Other ORMs      order.customer     → silently issues a query → N+1 risk
Prisma          must be included   → cannot happen accidentally
```

> [!TIP]
> This is a design decision worth appreciating: the most common ORM performance bug is made structurally impossible. The cost is verbosity on every query that traverses a relation.

---

## 8. Migrations

```bash
prisma migrate dev      # generate and apply a migration in development
prisma migrate deploy   # apply pending migrations in production
prisma db push          # prototype only — no migration file
```

> [!CAUTION]
> `prisma db push` skips migration history. It is fine while prototyping and dangerous on a shared or production database — it can drop columns to make the database match the schema, without a reviewable file.

---

## 9. Real World Example

- **Next.js applications** — Prisma is the most common data layer, especially with server components and server actions.
- **NestJS** projects that want typed data access without decorators.
- **Full-stack TypeScript teams** where the same types flow from the database to the frontend.

---

## 10. Communication and Dependencies

- **`schema.prisma`** — the source of truth
- **A generation step** — `prisma generate` must run after every schema change and in CI
- **The query engine** — a Rust binary that ships with the client
- **A connection pool** — configured via the connection string

---

## 11. Prisma vs the alternatives

| | Prisma | Drizzle | TypeORM |
|---|---|---|---|
| **Types** | Generated, excellent | Inferred from schema, excellent | Decorator-based, weaker |
| **SQL closeness** | Abstracted | Very close to SQL | Abstracted |
| **Complex queries** | Limited — falls back to raw | Strong | Moderate |
| **Bundle / runtime** | Rust engine binary | Pure TypeScript, tiny | Moderate |
| **Serverless** | Historically awkward | Excellent | Moderate |
| **Migrations** | Excellent | Good | Weaker |

> [!TIP]
> **Drizzle** is the strongest alternative if you want SQL-shaped queries and a small footprint, particularly on edge and serverless runtimes. Prisma's advantages are its migrations, its developer experience and Prisma Studio.

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use Prisma for TypeScript applications with a relational database where type safety and fast development matter — it is currently the most productive option in that space.

> [!CAUTION]
> - **Complex analytical SQL** — window functions, CTEs, sophisticated aggregation are limited; you will use `$queryRaw`.
> - **Serverless at scale** — connection handling has been a recurring pain point; use Prisma Accelerate, a pooler, or a driver adapter.
> - **Non-TypeScript projects** — the entire value proposition is the type generation.

---

## 13. Advantages and Disadvantages

**Advantages**
- Best-in-class type safety, derived from the schema
- N+1 structurally prevented
- Excellent migrations and tooling (Prisma Studio)
- One readable schema file as the source of truth
- Very good developer experience

**Disadvantages**
- Limited support for complex SQL; raw queries lose type safety
- A generation step that must not be forgotten
- The Rust query engine adds size and deployment considerations
- Connection management in serverless requires extra thought
- Less flexible than a query builder for read-heavy analytical work

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Query generation** | Small overhead |
| **N+1** | Prevented by design |
| **Relation loading** | `include` can over-fetch — use `select` to narrow columns |
| **Connections** | The historical weak point in serverless; plan for pooling |
| **Bulk operations** | `createMany` is fast; per-row loops are not |

---

## 15. Security Considerations

> [!IMPORTANT]
> Prisma parameterises all generated queries, so ordinary usage is not injectable. The exception is raw SQL:

```typescript
prisma.$queryRawUnsafe(`SELECT * FROM users WHERE email = '${email}'`)  // ✗
prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`            // ✓ tagged template, parameterised
```

- **`$queryRawUnsafe` is named that way for a reason** — treat every use as requiring justification
- **`select` is your response filter** — returning a whole model can leak fields such as `passwordHash`. Prisma has no automatic serialisation guard, so this is your responsibility
- **Never expose the Prisma client to client-side code** — in Next.js, keep it strictly in server components and route handlers
- **Least privilege** on the database user, separate from the migration user

---

## 16. Mental Model

> [!NOTE]
> **Prisma is a made-to-measure suit, re-cut every time your measurements change.**
>
> The schema is the measurement; the generated client is the suit. Change a field and regenerate, and anything that no longer fits fails immediately — at compile time, in your editor, rather than on the day you wear it.

---

## 17. Mini Architecture Diagram

```text
schema.prisma
    ↓ generate
Typed client + migrations
    ↓
Application code (compile-time checked)
    ↓
Query engine
    ↓
Database
```

---

## 18. Complete Request Flow

```text
Developer edits schema.prisma
    ↓
prisma migrate dev → SQL migration file + regenerated client
    ↓
TypeScript now fails anywhere the old shape was assumed
    ↓
─────────── at runtime ───────────
prisma.order.findMany({ where, include, take })
    ↓
Query engine builds parameterised SQL
    ↓
One query — no lazy loading, so no hidden N+1
    ↓
Rows mapped to objects whose TYPE reflects the include clause
    ↓
Response built with an explicit select, never the whole model
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Prisma generates types from your schema so query mistakes fail at compile time, and it removes N+1 by refusing lazy loading — at the cost of limited support for complex SQL.

---

## 20. Common Mistakes

- **Forgetting `prisma generate`** after a schema change, especially in CI
- **`prisma db push` on a production database**
- **`$queryRawUnsafe` with interpolated input**
- **Returning whole models** and leaking sensitive columns
- **Connection exhaustion in serverless** without a pooler
- **Fighting Prisma on a complex report** instead of writing SQL
- **Importing the client into client-side code** in Next.js

---

## 21. Open Source Technologies

- **Prisma** — client, migrations, Studio
- **Drizzle ORM** — the leading alternative, SQL-shaped and edge-friendly
- **Kysely** — a typed query builder, closer to SQL
- **PgBouncer**, **Prisma Accelerate** — connection pooling for serverless
- **Zod** — validating input before it reaches the database

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Check whether `prisma generate` runs in your CI pipeline.
- [ ] Find one query returning a whole model and narrow it with `select`.
- [ ] Search for `$queryRawUnsafe` and justify or replace every occurrence.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
schema.prisma → generated client → query engine → database
```

## 2. Request Flow

```text
Input       a typed query against the generated client
    ↓
Processing  parameterised SQL built by the query engine; relations explicit
    ↓
Output      results whose TypeScript type matches the query that was written
```

## 3. Real-World Usage

**Next.js plus Prisma** is currently the most common full-stack TypeScript data layer. Server components query the database directly through a typed client, and the same types flow through to the UI — one schema, one set of types, no drift.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A TypeScript ORM that generates a typed client from a schema file |
| **Why does it exist?** | Because database schema and application types otherwise drift apart |
| **Where does it belong?** | Between server-side TypeScript and a relational database |
| **When should I use it?** | TypeScript projects with relational data — with raw SQL for complex reports |
