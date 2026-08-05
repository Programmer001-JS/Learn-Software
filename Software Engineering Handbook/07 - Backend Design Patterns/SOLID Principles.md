# SOLID Principles

> **In one line —** five guidelines for structuring classes so that changing one thing does not require changing five others.

| | |
|---|---|
| **Category** | Design Principles |
| **Architectural Layer** | Code structure |
| **Coined by** | Robert C. Martin |
| **Related notes** | [Design Patterns](Design%20Patterns.md) · [Dependency Injection](Dependency%20Injection.md) · [Clean Architecture](Clean%20Architecture.md) · [Architecture Principles](../01%20-%20Foundation/08%20-%20Architecture%20Principles.md) |

---

## 1. What they are for

All five address the same underlying question: **when requirements change, how much code has to change with them?** They are guidelines, not laws, and each has a cost.

```text
S  Single Responsibility    one reason to change
O  Open/Closed              open to extension, closed to modification
L  Liskov Substitution      subtypes must be usable as their base type
I  Interface Segregation    many small interfaces beat one large one
D  Dependency Inversion     depend on abstractions, not concretions
```

---

## 2. S — Single Responsibility

**A class should have one reason to change.**

```text
BAD                                   GOOD
class User:                           class User:          data + identity
    def save(self)                    class UserRepository: persistence
    def send_email(self)              class EmailSender:    messaging
    def generate_pdf(self)            class InvoicePdf:     reporting
    def validate(self)                class UserValidator:  validation

↑ changes when the database,          ↑ each changes for exactly one reason
  the mail provider, the PDF
  library or the rules change
```

> [!TIP]
> "One reason to change" means **one stakeholder**. The finance team's rules and the marketing team's email templates should not live in the same class, because they change on different schedules for different people.

---

## 3. O — Open/Closed

**Open to extension, closed to modification.**

```text
BAD                                   GOOD
def charge(method, amount):           class PaymentMethod(Protocol):
    if method == "stripe": ...            def charge(self, amount): ...
    elif method == "paypal": ...
    elif method == "klarna": ...      class StripePayment: ...
    # every new provider edits         class PayPalPayment: ...
    # this function and risks           # a new provider is a NEW file
    # breaking the others
```

Adding behaviour by adding code is safer than adding behaviour by editing code — the existing paths cannot break if they were not touched.

---

## 4. L — Liskov Substitution

**Anything using a base type must work with any subtype, without knowing.**

```text
class Rectangle:  set_width, set_height
class Square(Rectangle):  setting width also sets height

rect.set_width(5); rect.set_height(4)
assert rect.area() == 20     ← passes for Rectangle, FAILS for Square
```

> [!CAUTION]
> The violation is not the inheritance; it is that `Square` **breaks a promise** its base type made. A subtype may not strengthen preconditions, weaken postconditions, or throw exceptions the base type never threw. If callers need `isinstance` checks, Liskov is already broken.

---

## 5. I — Interface Segregation

**No client should depend on methods it does not use.**

```text
BAD                                   GOOD
interface Worker {                    interface Workable  { work() }
    work()                            interface Feedable  { eat() }
    eat()                             interface Sleepable { sleep() }
    sleep()
}                                     class Robot implements Workable {}
class Robot implements Worker {
    eat()  { throw new Error() }      ↑ no meaningless methods
    sleep(){ throw new Error() }
}
```

A repository interface with fourteen methods forces every fake in your tests to implement fourteen methods. That friction is the principle telling you something.

---

## 6. D — Dependency Inversion

**Depend on abstractions, not on concrete implementations.** High-level policy should not depend on low-level detail.

```text
WITHOUT                               WITH
OrderService ──► PostgresRepo         OrderService ──► OrderRepository (interface)
                                                              ▲
                                                       PostgresRepo implements it

business logic depends on the DB      the DB depends on the business logic's interface
                                      ← the arrow INVERTED; hence the name
```

This is the principle behind [dependency injection](Dependency%20Injection.md), the [repository pattern](Repository%20Pattern.md) and [clean architecture](Clean%20Architecture.md). Of the five, it has the largest structural consequences.

---

## 7. Real World Example

- **Payment providers** — Open/Closed and Strategy, in essentially every commerce system.
- **Testing** is where Dependency Inversion pays: swapping a repository for a fake is only possible if the service depended on an interface.
- **Framework interfaces** — ASP.NET's `ILogger`, Spring's `Repository`, Python's `Protocol` classes are all Interface Segregation applied.

---

## 8. When To Use / When NOT To Use

> [!TIP]
> Apply SOLID where the code actually changes. A module rewritten twice a year deserves the structure; a script that has not been touched in three years does not.

> [!CAUTION]
> **Over-application is the common failure.** SOLID taken to its extreme produces one method per class, interfaces with one implementation, and a codebase where following a single request means opening nine files. Every principle buys flexibility and pays with indirection — and indirection is only worth it where variation actually happens.

---

## 9. The honest criticisms

- **SRP is vague** — "one reason to change" is interpretable enough that two engineers will disagree in good faith
- **OCP assumes you can predict variation** — guess wrong and you built the wrong extension point
- **ISP and DIP are language-shaped** — in Python or Go, duck typing and structural interfaces give much of the benefit without formal declarations
- **They are class-level** — they say nothing about system architecture, data modelling or concurrency, which is where the hard problems usually live

> [!IMPORTANT]
> Treat SOLID as five useful questions to ask about a class, not as a scorecard. A codebase that is easy to change is the goal; SOLID is one route to it, not the destination.

---

## 10. Advantages and Disadvantages

**Advantages**
- Localises the impact of change
- Makes code testable, chiefly through DIP
- Provides shared vocabulary for design discussions
- Guides refactoring when a class starts to hurt

**Disadvantages**
- Over-application produces indirection with no payoff
- More files, more interfaces, more navigation
- Some principles are ambiguous enough to argue about indefinitely
- Class-level only; silent on the architectural decisions that matter more

---

## 11. Security Considerations

> [!CAUTION]
> **Single Responsibility applied to security is genuinely valuable**: authorisation logic scattered across controllers, services and templates cannot be audited. Concentrated in one policy object, it can be read, reviewed and tested.

- **Dependency Inversion helps** — an injected authoriser can be verified once and reused everywhere
- **But excessive indirection hurts** — if answering "where is this checked?" requires tracing five interfaces, nobody will trace it
- Keep security controls **explicit and shallow**, even when the rest of the code is layered

---

## 12. Mental Model

> [!NOTE]
> **SOLID is the electrical wiring code for a building.**
>
> Separate circuits mean one fault does not darken the whole house, and sockets are a standard interface so any appliance fits. Following it makes maintenance safe. Following it obsessively — a separate circuit for every socket — is expensive and helps nobody.

---

## 13. Mini Architecture Diagram

```text
        High-level policy  (services, use cases)
                 ↓ depends on
        ┌── interfaces (abstractions) ──┐
                 ▲ implemented by
        Low-level detail  (database, HTTP, email)

   Dependency Inversion: the arrows point INWARD, toward policy
```

---

## 14. Complete Request Flow

SOLID visible in one request:

```text
Controller depends on OrderService                     (D)
    ↓
OrderService depends on OrderRepository interface      (D)
    ↓
It does one thing: order creation rules                (S)
    ↓
Payment handled by an injected PaymentMethod strategy  (O)
    ↓
A new provider = a new class, no existing code edited  (O)
    ↓
Any PaymentMethod is substitutable without special cases (L)
    ↓
The repository interface has four methods, not forty   (I)
    ↓
In tests: fakes injected; no database, no HTTP
```

---

## 15. Key Takeaway

> [!IMPORTANT]
> SOLID exists to limit how much code must change when requirements do — apply it where change actually happens, and accept that everywhere else the indirection costs more than it saves.

---

## 16. Common Mistakes

- **Applying all five everywhere**, producing indirection without variation
- **Interfaces with exactly one implementation, permanently**
- **Treating SRP as "one method per class"**
- **Inheritance that violates Liskov**, then patching it with `isinstance` checks
- **Believing SOLID is architecture** — it is class design
- **Arguing about SRP boundaries** instead of shipping
- **Ignoring it entirely** and rediscovering each principle through pain

---

## 17. Open Source Technologies

- **Python `Protocol`**, **Go interfaces**, **TypeScript interfaces** — structural typing, much of ISP and DIP for free
- **Spring**, **NestJS**, **ASP.NET Core** — DI containers built around dependency inversion
- **SonarQube**, **ArchUnit** — enforce structural rules in CI
- Robert C. Martin, *Clean Code* and *Clean Architecture* — the source, worth reading critically

---

## 18. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 19. Workbook Exercise

- [ ] Find your largest class and list its reasons to change. If there is more than one, name the split.
- [ ] Find one `if/elif` chain over a type and decide whether Open/Closed genuinely applies.
- [ ] Find one interface with a single implementation and decide honestly whether it earns its place.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Policy (services)
    ↓ depends on interfaces
Details (database, HTTP, email) implement them
```

## 2. Request Flow

```text
Input       a requirement change
    ↓
Processing  well-applied SOLID confines it to one class or one new file
    ↓
Output      a change that does not ripple through unrelated code
```

## 3. Real-World Usage

**Dependency inversion** is the principle with the clearest practical payoff: every framework with a DI container — Spring, ASP.NET Core, NestJS — is built on it, and it is what makes fast tests without infrastructure possible at all.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Five class-design principles for limiting the impact of change |
| **Why does it exist?** | Because tightly coupled code makes every change expensive |
| **Where does it belong?** | At the class level, below architecture |
| **When should I use it?** | Where code actually changes — not uniformly everywhere |
