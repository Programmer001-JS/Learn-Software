# TypeScript

> **In one line —** JavaScript with a type checker bolted on at compile time; the types vanish before the code runs, which is both its greatest strength and its most misunderstood limitation.

| | |
|---|---|
| **Category** | Programming Language |
| **Architectural Layer** | Client and server, build time |
| **Compiles to** | JavaScript |
| **Related notes** | [JavaScript](JavaScript.md) · [React](React.md) · [Validation](../07%20-%20Backend%20Design%20Patterns/Validation.md) · [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) |

---

## 1. Short Definition

*What is it?*

TypeScript is a superset of [JavaScript](JavaScript.md) that adds static types. A compiler checks those types and then **erases them**, emitting plain JavaScript that any runtime can execute.

---

## 2. Purpose

*What is its main purpose?*

To catch a large class of errors before the code runs, and to make large codebases navigable — types are documentation the compiler enforces and the editor understands.

---

## 3. Problem

*What engineering problem does it solve?*

In JavaScript, a typo in a property name, a function called with the wrong arguments, or a value that is sometimes `undefined` all fail at runtime, often in production, far from the cause.

```text
JavaScript                          TypeScript
user.emial                          user.emial
    ↓                                   ↓
undefined at runtime                compile error, in your editor, immediately
bug reaches production              bug never leaves your machine
```

---

## 4. Architecture Position

```text
TypeScript source
    ↓  tsc / esbuild / swc  — compile time only
JavaScript
    ↓
Engine → runtime
```

> [!IMPORTANT]
> **Types exist only at compile time.** At runtime there is no TypeScript — the browser and Node see plain JavaScript with every annotation stripped out. This has a direct consequence discussed in section 7.

---

## 5. What it gives you

```typescript
interface User {
  id: number;
  email: string;
  name?: string;          // optional
}

function greet(user: User): string {
  return `Hello ${user.name ?? user.email}`;
}

greet({ id: 1 });         // ✗ compile error: 'email' is missing
```

- **Autocomplete that actually knows the shape of your data**
- **Safe refactoring** — rename a field and every usage is found
- **Self-documenting function signatures**
- **Exhaustiveness checking** on unions and switch statements

---

## 6. Structural typing

TypeScript compares **shapes**, not names.

```typescript
interface Point { x: number; y: number; }
const p = { x: 1, y: 2, z: 3 };
const q: Point = p;        // ✓ fine — it has everything Point requires
```

This suits JavaScript's object-literal style far better than the nominal typing of Java or C#.

---

## 7. The critical limitation

> [!CAUTION]
> **TypeScript gives you no runtime safety whatsoever.** An API response typed as `User` is only a *claim*. If the server returns something else, TypeScript will not notice — the type was erased before the code ran.

```typescript
const user: User = await res.json();   // ← a lie the compiler happily accepts
```

The fix is runtime [validation](../07%20-%20Backend%20Design%20Patterns/Validation.md) at every boundary:

```typescript
const user = UserSchema.parse(await res.json());   // Zod: checks at runtime AND infers the type
```

> [!TIP]
> **Validate at every boundary**: API responses, request bodies, environment variables, database rows, anything parsed from a file. Inside those boundaries, trust the types.

---

## 8. `any` and `unknown`

```text
any        turns off type checking entirely  →  a hole in the type system
unknown    must be narrowed before use       →  safe alternative
```

> [!CAUTION]
> Every `any` is a place where the compiler stops helping — and it spreads, because anything derived from `any` becomes `any` too. Enable `noImplicitAny` and treat explicit `any` as something requiring justification.

---

## 9. Real World Example

- **VS Code** is written in TypeScript, by the team that created it — a genuine large-scale proof.
- **Most major frontend libraries** now ship type definitions; the ecosystem has effectively standardised on it.
- **Angular requires it**; React, Vue and Svelte all support it first-class.
- **Backend TypeScript** with NestJS or Express is common, sharing types between client and server.

---

## 10. Communication and Dependencies

- **Compiler** — `tsc`, or faster transpilers (esbuild, swc) that strip types without checking them
- **`@types/*` packages** — type definitions for libraries written in plain JavaScript
- **`tsconfig.json`** — where strictness is configured, and where most of the value is won or lost
- **Editors** — the language server is a large part of the practical benefit

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use TypeScript for anything beyond a throwaway script — and enable `strict: true` from the start. Adding strictness later to a large codebase is far more painful than starting with it.

> [!CAUTION]
> The costs are real: a build step, longer compile times, occasional fights with complex generic types, and third-party definitions that are wrong. For a fifty-line script, plain JavaScript is fine.

---

## 12. Advantages and Disadvantages

**Advantages**
- Catches a large class of bugs before running
- Excellent editor support and refactoring
- Types serve as enforced documentation
- Gradual adoption — it works file by file
- Shared types across frontend and backend

**Disadvantages**
- A build step, and slower feedback than plain JS
- **No runtime guarantees at all**
- Complex generic types can become unreadable
- `any` quietly disables everything it touches
- Type definitions for JS libraries can be inaccurate

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Runtime speed** | Identical to JavaScript — types are erased |
| **Bundle size** | Identical — nothing is emitted for types |
| **Build time** | Type checking is slow; transpilers skip it and are fast |
| **Developer speed** | Slower initially, faster on any codebase you must maintain |

---

## 14. Security Considerations

> [!CAUTION]
> The most common security mistake with TypeScript is **believing the types are validation**. Marking a request body as `LoginRequest` does not check it. An attacker sends whatever they want; only runtime validation stops them.

- **Validate all external input at runtime** — Zod, io-ts, class-validator
- **`as` assertions bypass checking** and are frequently used to silence a warning rather than to fix it
- Types describe intent, never enforcement — every server-side security check must exist independently

---

## 15. Mental Model

> [!NOTE]
> **TypeScript is a spell-checker for your code.**
>
> It catches an enormous number of mistakes as you type. It cannot tell you whether what you wrote is *true*. And when the document is printed, the spell-checker is not attached to it — anyone can write anything into the printed copy afterwards.

---

## 16. Mini Architecture Diagram

```text
TypeScript source
    ↓
Type checker (tsc)  ────► errors in your editor
    ↓
Types ERASED
    ↓
JavaScript
    ↓
Runtime  ← where validation must happen, because types are gone
```

---

## 17. Complete Request Flow

```text
Developer writes code
    ↓
Language server checks types live in the editor
    ↓
Build: tsc type-checks, esbuild strips types
    ↓
Plain JavaScript deployed
    ↓
Runtime: API response arrives
    ↓
Declared as User — but NOT verified
    ↓
Zod schema parses it → runtime error if the shape is wrong
    ↓
Only now is the typed value actually trustworthy
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> TypeScript catches mistakes at compile time and disappears at runtime — so validate every external input, because the types were never there when the data arrived.

---

## 19. Common Mistakes

- **Treating types as runtime validation**
- **Using `any`** to make an error go away
- **`as` assertions** instead of narrowing properly
- **Not enabling `strict`**, losing most of the value
- **Trusting `@types` packages** that are out of date or wrong
- **Over-engineering generics** until nobody can read the code
- **Assuming a compiling build is a correct build**

---

## 20. Open Source Technologies

- **TypeScript** — the compiler and language
- **Zod**, **io-ts**, **Valibot** — runtime validation that infers types
- **esbuild**, **swc** — fast transpilation without type checking
- **ts-node**, **tsx** — run TypeScript directly
- **typescript-eslint** — lint rules that understand types

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Enable `strict: true` on a project and fix the first ten errors it reports.
- [ ] Find one place where an API response is typed but not validated, and add a runtime schema.
- [ ] Count the `any` and `as` occurrences in your codebase and categorise why each exists.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
TypeScript
    ↓  compile time
JavaScript
    ↓  runtime
Engine
```

## 2. Request Flow

```text
Input       annotated source code
    ↓
Processing  type-checked, then types erased
    ↓
Output      plain JavaScript with no runtime type information
```

## 3. Real-World Usage

**VS Code** — an application of over a million lines used daily by millions of developers — is written in TypeScript. Microsoft created the language specifically because that codebase had become unmanageable in plain JavaScript.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | JavaScript plus compile-time static types |
| **Why does it exist?** | Because dynamic typing does not scale to large codebases |
| **Where does it belong?** | At build time, before the JavaScript runtime |
| **When should I use it?** | Any project you will maintain — with `strict` on and runtime validation at the edges |
