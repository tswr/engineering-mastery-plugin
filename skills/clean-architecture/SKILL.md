---
name: clean-architecture
description: "Apply when designing system structure, defining module or package boundaries, choosing layering, deciding where code belongs in a large codebase, or evaluating dependency direction. Trigger on questions about bounded contexts, module depth, interface design, dependency management, or monolith decomposition. Not for implementation-level code quality — use clean-code for that."
---

# Clean Architecture Skill

Principles for structuring large, domain-heavy codebases so they remain understandable, testable, and changeable over time. Tuned for environments where the repo is big, the domain is complex, multiple teams contribute, and wrong structural decisions compound fast.

The central enemy is **complexity** — specifically, the kind that makes it hard for an engineer to understand what a change will affect, where new logic belongs, and what a module actually promises to its consumers.

---

## The Dependency Rule

The single most important structural principle. All source code dependencies must point **inward**, toward higher-level policy. Never let the inner layers know about the outer layers.

```
┌──────────────────────────────────┐
│  Frameworks & Infrastructure     │  ← outer: databases, UI, gRPC, queues
│  ┌──────────────────────────┐    │
│  │  Interface Adapters      │    │  ← controllers, presenters, gateways
│  │  ┌──────────────────┐    │    │
│  │  │  Use Cases        │    │    │  ← application-specific business rules
│  │  │  ┌────────────┐   │    │    │
│  │  │  │  Entities   │   │    │    │  ← enterprise/domain business rules
│  │  │  └────────────┘   │    │    │
│  │  └──────────────────┘    │    │
│  └──────────────────────────┘    │
└──────────────────────────────────┘
```

**Entities** encode domain rules that would exist even if there were no software system. They change only when the business fundamentally changes.

**Use Cases** encode application-specific business rules — the workflows that orchestrate entities. They change when the application's behavior changes, not when the database or transport changes.

**Interface Adapters** translate between the use cases and the outside world. Controllers, presenters, mappers, serializers. They convert data from the format most convenient for use cases into the format required by frameworks.

**Frameworks & Infrastructure** are details. The database is a detail. The web framework is a detail. The message queue is a detail. These are the outermost layer and the most replaceable.

**The test:** if swapping your database from Postgres to Spanner requires touching your domain logic, the Dependency Rule is violated. If adding a new RPC endpoint requires modifying business rules, the Dependency Rule is violated.

This principle is also known as **Hexagonal Architecture** (Ports and Adapters). Ports are the interfaces that the application defines for its needs (inbound ports for use cases, outbound ports for infrastructure). Adapters implement these ports for specific technologies. The vocabulary differs but the structural constraint is identical: dependencies point inward.

---

## Deep Modules Over Shallow Modules

From Ousterhout: the best modules provide **powerful functionality behind a simple interface**. A deep module hides significant complexity from its consumers. A shallow module exposes almost as much complexity in its interface as it contains internally.

**Deep module:** `UnixFileDescriptor` — the interface is `open()`, `read()`, `write()`, `close()`. Behind it: buffering, caching, device drivers, permissions, journaling. Enormous complexity, tiny interface.

**Shallow module:** A class with 15 methods that each delegate to one line of internal logic. The interface *is* the implementation. The abstraction adds overhead without hiding anything.

**In large codebases this matters because:**
- Shallow modules multiply the number of things an engineer must understand to make a change
- They fragment related logic across many files, increasing cognitive load
- They create "pass-through" layers that add indirection without adding abstraction

**Practical guidance:**
- When decomposing, ask: does this boundary hide a meaningful chunk of complexity, or does it just move code to a different file?
- Prefer fewer, deeper modules over many thin wrappers
- A module's interface should be much simpler than its implementation. If it isn't, reconsider the boundary
- Default to somewhat larger modules and split only when there's a clear complexity-hiding benefit

**Tension with Clean Architecture layering:** Martin's layers can produce shallow pass-through adapters if applied mechanically. Use the layers when they genuinely separate concerns (domain vs. infrastructure), but don't create an adapter layer just because the diagram says to. If the adapter does nothing meaningful, collapse it.

---

## Complexity Indicators

Ousterhout defines three symptoms that signal a system is becoming too complex:

**Change amplification** — a simple conceptual change requires editing many different places. Often caused by duplicated knowledge or missing abstractions. In a large repo, this is the most costly symptom because it multiplies review burden and merge conflicts across teams.

**Cognitive load** — an engineer must hold too much context in their head to make a safe change. Caused by non-obvious dependencies, implicit contracts, and modules that leak their internals. In domain-heavy code, this often manifests as tribal knowledge that isn't encoded in the structure.

**Unknown unknowns** — the worst kind. It's not obvious what you need to know to make a change correctly. Caused by hidden dependencies, implicit ordering requirements, and action-at-a-distance. A well-structured codebase makes dependencies and contracts explicit so that even engineers unfamiliar with a module can understand its boundaries.

When evaluating any architectural decision, ask: does this increase or decrease these three symptoms?

---

## Strategic Domain-Driven Design

In complex domains, the biggest architectural mistakes aren't at the class level — they're at the boundary level. DDD's strategic patterns address this directly.

### Bounded Contexts

A bounded context is a boundary within which a particular domain model is consistent and meaningful. Different parts of a large system will have different models of the same real-world concept, and that's correct.

**Example:** "User" means something different to the authentication service (credentials, sessions, permissions) than to the billing service (payment methods, invoices, subscription tier) than to the recommendations service (preferences, interaction history). Forcing a single User model across all three creates a god-object that every team must coordinate on.

**Each bounded context:**
- Owns its model and its data
- Defines the meaning of its terms (ubiquitous language)
- Has an explicit contract at its boundary for communicating with other contexts
- Can evolve internally without forcing changes on other contexts

**In a large repo**, bounded contexts often map to top-level directories, packages, or services. The key structural question is: where do the model boundaries fall, and how do contexts communicate across those boundaries?

### Context Mapping

When two bounded contexts must communicate, the relationship between them has a shape:

**Shared Kernel** — two teams share a small subset of the model. Changes require coordination. Use sparingly; it creates coupling.

**Customer-Supplier** — upstream context provides what downstream needs. Downstream can request features. Upstream has obligations. Common in platform/product team relationships.

**Anti-Corruption Layer (ACL)** — downstream creates a translation layer to protect its model from the upstream's model. Critical when integrating with legacy systems, external APIs, or contexts owned by teams with different conventions. Never let someone else's model leak into your domain.

**Conformist** — downstream accepts the upstream model as-is. Appropriate when the cost of translation exceeds the benefit (e.g., using a well-designed internal platform's types directly).

**In practice at scale:** always default to building an ACL when integrating across team boundaries unless the upstream model is genuinely a good fit. The cost of translation is much lower than the cost of a foreign model corrupting your domain logic.

### Ubiquitous Language

Within a bounded context, the code should use the same terms that domain experts use. If the business says "enrollment," the code should not say "registration" or "signup" in that context. Name mismatches between code and domain create a constant translation tax that slows every conversation between engineers and domain experts.

This matters most in domain-heavy codebases where getting the model wrong means building the wrong thing.

---

## Dependency Management at Scale

In large repos, uncontrolled dependencies are the primary source of architectural decay.

**Depend on abstractions at module boundaries.** Within a module, concrete types are fine. At the boundary where modules communicate, use interfaces or abstract types so that modules can be tested, replaced, and evolved independently.

**Make dependencies explicit and visible.** If module A depends on module B, that should be visible in the build graph, not hidden in a runtime call that only manifests in production. Implicit dependencies are the source of unknown unknowns.

**Avoid circular dependencies between modules.** If A depends on B and B depends on A, they are effectively one module with a confusing split. Either merge them or introduce an interface that one of them owns and the other implements (Dependency Inversion).

**Separate policy from mechanism.** Policy is the business rule ("orders over $10k require manager approval"). Mechanism is how it's executed (database query, RPC call, notification). Policy should depend on abstractions for mechanism, never on concrete infrastructure.

**Build boundaries that align with team boundaries.** Conway's Law is real. If two teams share a tightly coupled module, coordination overhead will dominate. Structure the code so that each team owns cohesive modules with clear interfaces to other teams' code.

---

## Interface Design Principles

An interface is a promise. In a large codebase, bad interfaces compound — every consumer inherits the confusion.

**Define errors out of existence.** Before adding error handling for a case, ask whether you can design the interface so the case cannot arise. An API that accepts only valid states by construction is better than one that accepts anything and validates later. This reduces both the implementor's and the consumer's burden.

**Interfaces should make the common case simple and the uncommon case possible.** If 90% of callers use a three-argument version, don't force everyone through the twelve-argument version. Provide a layered API: simple defaults with escape hatches for power users.

**General-purpose interfaces are deeper than special-purpose ones.** A general text-editing interface (insert, delete, get-range) is deeper and more reusable than a set of special-purpose methods (backspace, deleteSelection, typeCharacter). Generality often simplifies rather than complicates, because it reduces the number of concepts.

**Together or apart?** Bring things together when they share information, when they are always used together, when they overlap conceptually, or when combining them simplifies the interface. Keep them apart when they are about genuinely different things, or when combining them would create a shallow module with a large, confusing interface.

---

## Structural Patterns Worth Knowing

These recur across large codebases regardless of language. Know them by name so you can reference them in design discussions.

**Repository** — abstracts data access behind a collection-like interface. Your domain logic asks for entities; the repository handles how they're stored and retrieved. Prevents storage details from leaking into business logic.

**Gateway** — wraps communication with an external system (API, service, third-party). All translation between your model and the external model happens here. This is your ACL implementation point.

**Service Layer** — defines the application's boundary and coordinates use-case execution. Thin orchestration: receives a request, calls domain logic, manages transactions, returns a result. Should not contain business rules itself.

**Strategy** — encapsulates an algorithm behind an interface so the caller can swap implementations. Useful when behavior varies by context (different pricing rules per region, different ranking algorithms per experiment).

**Observer / Event Bus** — decouples the producer of an event from its consumers. In a large codebase, this prevents fan-out coupling where one change triggers updates in a dozen dependent modules. Be cautious: overuse makes control flow invisible.

**Facade** — provides a simplified interface to a complex subsystem. Useful when a bounded context exposes too much internal surface area. The facade becomes the official entry point and hides the internal module structure.

---

## Decision Framework

When facing a structural choice in a large codebase, evaluate in this order:

1. **Does it respect the Dependency Rule?** Dependencies point inward toward domain logic, not outward toward infrastructure or frameworks.

2. **Does it reduce or increase complexity?** Check for change amplification, cognitive load, and unknown unknowns. A "cleaner" design that makes the system harder to understand is not actually cleaner.

3. **Is the module deep?** Does the boundary hide meaningful complexity behind a simple interface, or does it just add indirection?

4. **Are the bounded contexts correct?** Is each model consistent within its boundary? Are cross-context contracts explicit? Is there an ACL where needed?

5. **Are dependencies explicit and acyclic?** Can you see the dependency graph? Does it form a DAG? Do module boundaries align with team boundaries?

6. **Is it the simplest structure that satisfies the above?** Don't add layers, abstractions, or indirection speculatively. Every structural element should earn its place by solving a concrete problem.

---

## Applying This Skill

When designing new systems, start from the domain model and work outward — decide on entities and use cases before choosing frameworks or storage. When adding to an existing system, locate the correct bounded context and layer first, write the code second. When reviewing, check dependency direction and module depth before checking implementation details. When refactoring, target the highest-impact complexity symptom first (usually unknown unknowns or change amplification).

Name architectural decisions explicitly. In a large repo, implicit decisions become tribal knowledge that doesn't scale.

### Architecture Decision Records (ADRs)

When introducing a new pattern, changing a structural convention, or choosing between significant alternatives, write an ADR. A minimal ADR contains: **Title** (the decision), **Context** (what forces led to this decision), **Decision** (what was chosen and why), **Consequences** (what changes, what trade-offs are accepted). Store ADRs alongside the code they govern. They are immutable once accepted — if a decision is superseded, write a new ADR that references the old one.
