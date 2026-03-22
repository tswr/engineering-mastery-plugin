---
name: clean-code
description: "Apply when writing, refactoring, or reviewing code with focus on naming, function design, readability, class structure, error handling, or code smells. Trigger on requests to 'clean up' code, improve readability, reduce complexity, or apply SOLID principles. Not for architecture, performance, security, or test strategy — those have dedicated skills."
---

# Clean Code Skill

Apply clean code principles to every piece of code you write, refactor, or review. Clean code reads like well-written prose — every name is intentional, every function does one thing, and the structure guides the reader without needing comments to explain it.

The overarching standard: **code is read far more often than it is written.** Optimize for the reader, not the writer.

---

## Naming

Names are the first thing a reader encounters. They should reveal intent, not require decoding.

**Use intention-revealing names.** A name should answer: why does this exist, what does it do, and how is it used? If a name requires a comment to explain it, it doesn't reveal its intent.

```
# Bad
d = 7           # elapsed time in days
lst = get(u)    # get user list

# Good
elapsed_days = 7
active_users = fetch_active_users(organization)
```

**Avoid disinformation.** Don't call a grouping `account_list` unless it's actually a `List`. Don't use names that differ in tiny ways (`XYZControllerForHandlingOfStrings` vs `XYZControllerForStorageOfStrings`). Avoid `l`, `O`, `0`, `1` as variable names.

**Make meaningful distinctions.** Never differentiate names with number series (`a1`, `a2`) or noise words (`ProductInfo` vs `ProductData`). If two names exist in the same scope, their names must explain how they differ.

**Use pronounceable, searchable names.** Avoid encodings and abbreviations. Single-letter names are acceptable only as loop counters in tiny scopes. The length of a name should correspond to the size of its scope — broader scope means longer, more descriptive name.

**Class names** are nouns or noun phrases: `Customer`, `WikiPage`, `AddressParser`. Never verbs. Avoid vague names like `Manager`, `Processor`, `Data`, `Info`.

**Method names** are verbs or verb phrases: `save`, `delete_page`, `calculate_pay`. Accessors use `get_`, mutators use `set_`, predicates use `is_` or `has_`.

---

## Functions

Functions are the verbs of your codebase. Each one should do exactly one thing.

**Keep functions small.** A function should rarely exceed 20 lines. If you're writing a function longer than that, it almost certainly does more than one thing. Extract until you can't extract anymore.

**Do one thing.** A function does one thing when you cannot meaningfully extract another function from it. If a function has sections (declarations, initialization, logic, output), it's doing more than one thing.

**One level of abstraction per function.** Don't mix high-level intent (`get_page_html()`) with low-level detail (`buffer.append("\n")`) in the same function. Follow the **Stepdown Rule**: code should read top-down like a narrative, with each function introducing the next level of detail.

**Function arguments — fewer is better.**
- Zero arguments (niladic): ideal
- One argument (monadic): good — asking a question about it, transforming it, or signaling an event
- Two arguments (dyadic): acceptable when there's a natural ordering (like coordinates)
- Three arguments (triadic): avoid when possible — consider wrapping into an object
- Never use flag arguments (`render(true)`). They announce the function does two things. Write two functions instead.

**No side effects.** If a function promises to check a password, it should not also initialize a session. Side effects are lies — they create temporal couplings and order dependencies that aren't visible in the function's name.

**Command-Query Separation.** A function should either do something (command) or answer something (query), never both. `set_attribute` that returns `true`/`false` for success violates this — separate into `attribute_exists()` and `set_attribute()`.

**Prefer clean error paths over inline error checking.** Deeply nested error-code checks obscure the main logic. Use whatever error-handling idiom your language favors (exceptions, result types, early returns) to keep the happy path front and center. Extract error-handling blocks into their own functions — error handling is one thing.

---

## Comments

Comments are, at best, a necessary evil. They compensate for our failure to express ourselves in code. Before writing a comment, ask: can I express this in the code itself?

**Good comments (the rare exceptions):**
- Legal/license headers required by convention
- Explanation of intent behind a non-obvious decision ("We use this algorithm because the standard approach has O(n²) in our specific data shape")
- Clarification of return values from opaque library calls
- Warnings of consequences ("This test takes 30 minutes to run")
- TODO markers for acknowledged technical debt (with a plan)
- Doc comments on public API boundaries

**Bad comments (the majority):**
- Restating what the code already says (`i += 1  # increment i`)
- Journal comments or changelogs in the file (use version control)
- Commented-out code (delete it; version control remembers)
- Position markers or banners (`// ===== GETTERS =====`)
- Closing-brace comments (`} // end of while`)
- Attribution comments (`// Added by Bob`)
- Mandated doc comments on every function regardless of need
- Comments that will become lies when the code changes (and the comment doesn't)

**The rule: if you feel the need to write a comment, first try to restructure the code so the comment is unnecessary.** Rename the variable, extract the function, introduce an explaining variable.

---

## Objects and Data Structures

There is a fundamental tension between objects and data structures. Understanding it prevents architectural mistakes.

**Objects** hide their data behind abstractions and expose functions that operate on that data. They make it easy to add new types (just add a new class) but hard to add new behaviors (you must change every class).

**Data structures** expose their data and have no meaningful behavior. They make it easy to add new behaviors (just write a new function) but hard to add new types (you must change every function).

Choose the right tool. Don't create hybrids that half-expose data and half-hide it — these get the worst of both worlds.

**The Law of Demeter:** a method should only call methods on its own object, its parameters, objects it creates, and its direct instance variables. Avoid chaining like `context.getOptions().getScratchDir().getAbsolutePath()`. This kind of "train wreck" couples you to the entire chain of implementation details.

---

## Error Handling

Error handling is important, but it must not obscure the logic of the code.

**Separate the happy path from error handling.** Whatever mechanism your language uses — exceptions, result types, error codes, monads — structure your code so the main logic is clearly visible and not buried in error-checking noise.

**Handle errors at the right level.** Deal with errors where you have enough context to do something meaningful about them. Propagating errors up blindly or swallowing them silently both erode reliability.

**Create informative error messages.** Include what operation failed, why it failed, and enough context to locate the source. Mention the failing constraint or value.

**Define error types by the caller's needs.** The most useful classification is usually: can I recover from this, or not? Wrap third-party APIs so you can translate their error reporting into your own conventions and decouple from the library's error taxonomy.

**Don't return null.** Returning null is an invitation for null-dereference bugs and forces null checks everywhere. Return an empty collection, a special-case object, use an optional/maybe type, or signal the error explicitly through your language's idiom. Similarly, don't pass null as an argument unless the API explicitly expects it.

---

## Tests

Test code follows the same readability and naming standards as production code. For testing principles, strategy, and test doubles, see the testing skill.

---

## Classes

**Keep classes small.** Smallness in classes is measured by responsibilities, not lines. A class should have one and only one reason to change (the Single Responsibility Principle).

If you can't describe what a class does without using "and" or "or," it has too many responsibilities. If you can't give it a concise name, it's probably too big.

**High cohesion.** A class where every method uses every instance variable has maximum cohesion. When cohesion drops — when clusters of variables are used by clusters of methods — that's a signal to split the class.

**Organize for change (Open-Closed Principle).** Classes should be open for extension but closed for modification. When a new feature requires modifying existing, tested classes, the design is fragile. Use abstractions and interfaces to isolate what varies from what stays the same.

**Depend on abstractions, not concretions (Dependency Inversion).** High-level policy should not depend on low-level detail. Both should depend on abstractions. This makes classes testable (you can inject test doubles) and flexible (you can swap implementations).

---

## Emergent Design

Follow Kent Beck's four rules of simple design, in priority order:

1. **Runs all the tests.** A system that cannot be verified cannot be trusted. Testability drives good design — small, single-purpose classes with decoupled dependencies.
2. **Contains no duplication.** Duplication is the primary enemy of a well-designed system. Extract common patterns into shared abstractions.
3. **Expresses the intent of the programmer.** Choose good names, keep functions and classes small, use standard nomenclature and patterns. The clearer the code, the less time others spend understanding it.
4. **Minimizes the number of classes and methods.** Don't over-engineer. Don't create a class for every tiny concept. This rule has the lowest priority — only apply it after satisfying the first three.

---

## Smells and Heuristics — Quick Reference

Use this as a checklist during code review or refactoring.

**In comments:** obsolete comment, redundant comment (repeats the code), commented-out code, comment used instead of a better name or extraction.

**In the environment:** build requires more than one step, tests require more than one step.

**In functions:** too many arguments, flag arguments, dead function (never called).

**General smells:** obvious behavior not implemented (principle of least surprise violated), duplication, code at the wrong level of abstraction, feature envy (a method that's more interested in another class's data), inappropriate intimacy between classes, dead code, vertical separation (related things are far apart), inconsistency in conventions, clutter (unused variables, unreachable code, default constructors that do nothing).

**In names:** name doesn't describe side effects, name is at the wrong level of abstraction, name uses encoding or prefix conventions that add noise.

**In tests:** insufficient tests, no tests for boundary conditions, skipping trivial tests ("it's too simple to break" — test it anyway), bugs congregate (test nearby when you find one), test execution is slow.

---

## Applying This Skill

When generating code, follow these principles by default. When refactoring, use the smells checklist to identify what to fix first. When reviewing, call out violations by name (e.g., "this function has a flag argument — split into two functions") so the feedback is specific and actionable.

Prioritize readability over cleverness, small over large, explicit over implicit, and simple over complex. If the code doesn't look clean yet, it isn't done.
