# Design Patterns

> **In one line —** named solutions to problems that recur in object-oriented code; useful as vocabulary, harmful when applied before the problem exists.

| | |
|---|---|
| **Category** | Design Discipline |
| **Architectural Layer** | Code structure |
| **Origin** | *Design Patterns* (Gamma, Helm, Johnson, Vlissides, 1994) |
| **Related notes** | [SOLID Principles](SOLID%20Principles.md) · [Repository Pattern](Repository%20Pattern.md) · [Dependency Injection](Dependency%20Injection.md) · [Clean Architecture](Clean%20Architecture.md) |

---

## 1. Short Definition

*What is it?*

A design pattern is a documented, reusable solution to a recurring design problem, together with a name that lets engineers discuss it in one word.

---

## 2. Purpose

*What is its main purpose?*

**Shared vocabulary**, above all else. Saying "we'll use a Strategy here" conveys a structure in three words that would otherwise take five minutes and a whiteboard.

---

## 3. Problem

*What engineering problem does it solve?*

The same structural problems appear repeatedly: how do I vary one step of an algorithm, how do I add behaviour without editing a class, how do I decouple a sender from a receiver. Patterns record answers that were found many times independently.

---

## 4. The three categories

```text
CREATIONAL     how objects are made
               Factory · Builder · Singleton · Prototype

STRUCTURAL     how objects are composed
               Adapter · Decorator · Facade · Proxy · Composite

BEHAVIOURAL    how objects interact
               Strategy · Observer · Command · Template Method · State
```

---

## 5. The ones you will actually use

| Pattern | Problem it solves | Where you have seen it |
|---|---|---|
| **Strategy** | Swap an algorithm at runtime | Payment providers, sorting, pricing rules |
| **Factory** | Create objects without naming the concrete class | Parsers by file type, drivers by config |
| **Adapter** | Make an incompatible interface fit | Wrapping a third-party SDK |
| **Decorator** | Add behaviour without changing the class | [Middleware](Middleware.md), caching a repository |
| **Observer** | Notify many listeners of an event | Domain events, pub/sub, UI reactivity |
| **Facade** | One simple interface over a complex subsystem | A [service](Services.md) over several repositories |
| **Repository** | Abstract data storage | [Repository Pattern](Repository%20Pattern.md) |
| **Builder** | Construct complex objects step by step | Query builders, test fixtures |

---

## 6. Strategy — the most useful one

```python
class PaymentMethod(Protocol):
    def charge(self, amount: Money) -> Receipt: ...

class StripePayment:  ...
class PayPalPayment:  ...

class Checkout:
    def __init__(self, method: PaymentMethod):   # ← injected strategy
        self.method = method
```

Adding a provider means adding a class, not editing a growing `if/elif` chain. This is the Open/Closed principle from [SOLID](SOLID%20Principles.md) made concrete — and note it is also [dependency injection](Dependency%20Injection.md).

---

## 7. Singleton — the one to be careful with

> [!CAUTION]
> **Singleton is the most overused and most regretted pattern.** It creates global mutable state, hides dependencies from constructors, makes tests order-dependent, and behaves badly under concurrency.

```text
Need one shared instance?
    ↓
Register it as a SINGLETON in your DI container
    ↓
Same lifetime, but the dependency stays explicit and replaceable
```

---

## 8. The real risk: patterns as decoration

> [!CAUTION]
> Applying patterns before the problem exists produces an `AbstractRequestHandlerFactoryBuilder` where a function would do. Complexity is paid at every future change; the pattern's benefit is only collected if the variation it anticipated actually arrives.

```text
HEALTHY                              UNHEALTHY
"this if/elif has grown to eight     "we should use patterns here"
 branches and changes weekly"            ↓
    ↓                                a factory producing one type
extract a Strategy                   an interface with one implementation
    ↓                                indirection with no variation behind it
the pattern earned its place
```

> [!TIP]
> **Refactor into a pattern; do not design into one.** The third time you write the same conditional, the pattern is justified. The first time, it is speculation.

---

## 9. Patterns you are already using

Most engineers use these daily without naming them:

- **[Middleware](Middleware.md)** is Chain of Responsibility plus Decorator
- **[Repository](Repository%20Pattern.md)** is a data-access pattern
- **[Dependency injection](Dependency%20Injection.md)** is how Strategy is usually delivered
- **React components** are Composite
- **Event listeners** are Observer
- **ORMs** are Data Mapper or Active Record

---

## 10. Real World Example

- **Payment integrations** are Strategy in almost every system that has more than one provider.
- **Third-party SDK wrappers** are Adapter, and they are what makes replacing a vendor possible.
- **Caching a repository** is Decorator — the callers are unchanged.
- **Domain events** are Observer, and the basis of [event-driven architecture](../10%20-%20Distributed%20Systems/Event%20Driven%20Architecture.md).

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use a pattern when you have felt the pain it solves, at least twice. Use the *name* freely — vocabulary is the cheapest part and the most valuable.

> [!CAUTION]
> Do not apply patterns to demonstrate knowledge, do not use them in languages where a simpler feature exists (in Python, a function is often a better Strategy than a class), and do not treat the 1994 catalogue as a checklist. Many patterns exist to work around limitations of languages of that era.

---

## 12. Advantages and Disadvantages

**Advantages**
- Shared vocabulary across teams and languages
- Proven solutions with known trade-offs
- Make variation points explicit
- Guide code toward testable structure

**Disadvantages**
- Over-application is the norm rather than the exception
- Add indirection, which costs on every read
- Some are language-specific workarounds
- The names invite cargo-culting

---

## 13. Performance Impact

Essentially none. Patterns are structural; the cost is measured in **cognitive load and file count**, not CPU cycles. The exception is deeply nested decorators or proxies in a very hot path.

---

## 14. Security Considerations

> [!CAUTION]
> **Indirection hides where checks happen.** A five-layer chain of decorators and factories makes it genuinely hard to answer "where is authorisation enforced for this operation?" — and a question nobody can answer is a question nobody audits.

- **Singletons holding request state** leak data between users
- **Factories choosing implementations from configuration** can silently select an insecure variant
- **Keep security controls explicit and shallow**; a pattern that obscures them is a bad trade

---

## 15. Mental Model

> [!NOTE]
> **Design patterns are chess openings.**
>
> Knowing them by name lets two players discuss a position instantly. Playing an opening because you memorised it, without reading the board, loses games. They are a language for describing good moves — not a substitute for looking at the position.

---

## 16. Mini Architecture Diagram

```text
Problem appears repeatedly
    ↓
Recognise the shape
    ↓
┌────────────┬──────────────┬──────────────┐
Creational    Structural     Behavioural
Factory       Adapter        Strategy
Builder       Decorator      Observer
              Facade         Command
    ↓
Apply the minimum that solves the problem you actually have
```

---

## 17. Complete Request Flow

How several patterns compose in one real request:

```text
Request
    ↓
MIDDLEWARE          Chain of Responsibility + Decorator
    ↓
Controller
    ↓
Service             Facade over several repositories
    ↓
PaymentMethod       Strategy, chosen at runtime
    ↓
CachedOrderRepo     Decorator wrapping SqlOrderRepo
    ↓
Database
    ↓
OrderCreated event  Observer — listeners react independently
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Patterns are vocabulary for solutions you should reach by refactoring, not by design — apply one when the problem has appeared, never in anticipation.

---

## 19. Common Mistakes

- **Applying patterns preemptively**, before any variation exists
- **Singleton for everything**, creating global mutable state
- **An interface with exactly one implementation**, forever
- **Using class-based patterns in languages with simpler features**
- **Treating the catalogue as a checklist**
- **Layering patterns until authorisation is unfindable**
- **Confusing patterns with architecture** — patterns are code-level, architecture is system-level

---

## 20. Open Source Technologies

- Read real implementations: **Django middleware** (Chain), **SQLAlchemy** (Data Mapper), **React** (Composite), **Spring** (Proxy, Factory, Template Method)
- **refactoring.guru** — the clearest modern catalogue
- **Martin Fowler's *Patterns of Enterprise Application Architecture*** — closer to backend reality than the 1994 book

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Find one growing `if/elif` chain in your code and decide whether Strategy would genuinely help.
- [ ] Identify three patterns you already use without having named them.
- [ ] Find one interface in your codebase with a single implementation and ask whether it earns its place.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Recurring problem
    ↓
Named pattern (Creational / Structural / Behavioural)
    ↓
Code structure with an explicit variation point
```

## 2. Request Flow

```text
Input       a design problem that has appeared more than once
    ↓
Processing  match it to a known pattern; apply the minimum version
    ↓
Output      code that varies where it needs to and is fixed elsewhere
```

## 3. Real-World Usage

**Payment integration** is the pattern most engineers meet first. One provider is an `if`; three providers with different flows is a Strategy — and the transition point between those two states is exactly when the pattern becomes correct.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Named, reusable solutions to recurring design problems |
| **Why does it exist?** | To give engineers shared vocabulary and proven structures |
| **Where does it belong?** | At the code level, below architecture |
| **When should I use it?** | After the problem appears — reached by refactoring, not by design |
