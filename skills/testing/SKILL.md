---
name: testing
description: "Apply when writing tests, reviewing tests, deciding what to test, choosing test doubles, structuring test suites, or testing legacy code. Trigger on mentions of TDD, mocking, test coverage, test smells, integration tests, characterization tests, or getting code under test. Language-specific frameworks and idioms are in references/."
---

# Testing Skill

Write tests that are valuable — tests that protect against regressions, survive refactoring, run fast, and stay maintainable. A test suite that slows down development or breaks on every refactor is worse than no tests at all, because it creates the illusion of safety while taxing every change.

**Language-specific frameworks and examples are in the `references/` directory.** Read the relevant file when generating tests:
- `references/cpp.md` — GoogleTest, GoogleMock, dependency injection in C++
- `references/python.md` — pytest, unittest.mock, fixtures, parametrize
- `references/rust.md` — built-in test framework, mockall, test organization
- `references/java.md` — JUnit 5, Mockito, AssertJ, test organization

---

## The Four Pillars of a Valuable Test

Every test should be evaluated against four qualities. A test that scores poorly on any one of them is a candidate for rewriting or deletion.

**1. Protection against regressions.** The test should catch bugs when someone changes the code it covers. A test that passes regardless of what you do to the production code protects nothing. The more production code a test exercises — and the more complex that code is — the higher its regression protection.

**2. Resistance to refactoring.** The test should not break when you restructure code without changing its behavior. A test that fails on every internal refactor produces false positives — alarms that train the team to ignore test failures. This is the most undervalued pillar and the one most tests fail on. Tests that verify implementation details (method call order, internal state, private method invocations) have low resistance to refactoring. Tests that verify observable behavior (outputs, state changes visible to the caller, side effects on external systems) have high resistance.

**3. Fast feedback.** The test should run in milliseconds to seconds. Slow tests don't get run during development, which means they only catch regressions in CI — too late to be useful for the feedback loop. A test that takes 30 seconds is a test that developers skip locally.

**4. Maintainability.** The test should be easy to read, easy to understand, and cheap to modify. A 200-line test with intricate setup is expensive to maintain. Tests are code — they require the same care for readability as production code.

**The tradeoff:** you cannot maximize all four simultaneously. The key insight is that **resistance to refactoring is non-negotiable** — you either have it or you don't, and tests without it actively harm development velocity. The real tradeoff is between regression protection and fast feedback: unit tests optimize for speed, integration tests optimize for regression coverage. Choose based on the code's characteristics.

---

## Test Behavior, Not Implementation

The single most important testing principle for large codebases. Test what the code *does*, not *how* it does it.

**Observable behavior** is anything visible to the caller or to an external system: return values, state changes accessible through the public API, calls to external dependencies (database writes, HTTP requests, messages sent). Test these.

**Implementation details** are everything else: private method calls, internal data structures, the order of internal operations, which helper classes are used. Never test these directly. If a test breaks because you extracted a method, renamed an internal class, or changed an algorithm without changing the result, the test is coupled to implementation.

**The practical test:** after a refactoring that doesn't change any public contract, how many tests break? If the answer is more than zero, those tests are testing implementation. Fix them or delete them.

**Mocks and implementation coupling.** Mocking internal collaborators couples tests to the interaction between classes — a form of implementation detail. Reserve mocks for **unmanaged dependencies** (external systems you don't control: APIs, databases, message queues, file systems). For internal collaborators within your codebase, use real instances. This is the core difference between the classical and London schools, and the classical approach produces tests that survive refactoring.

---

## The Three Styles of Testing

**Output-based testing.** Supply inputs, verify the output. No state changes, no side effects. This is the ideal style — pure function in, value out, assert on the value. Highest resistance to refactoring, easiest to maintain. Prefer this whenever possible.

```
result = calculate_discount(price=100, tier=Gold)
assert result == 85
```

**State-based testing.** Perform an operation, then verify the resulting state through the public API. More setup than output-based, but necessary when the operation mutates observable state.

```
cart.add(item)
assert cart.item_count() == 1
assert cart.total() == item.price
```

**Communication-based testing.** Verify that the system under test interacts correctly with an external dependency. Uses mocks or spies. The most fragile style — use only for interactions with **unmanaged dependencies** (external systems), never for internal collaborations.

```
order_service.place_order(order)
verify(payment_gateway).charge(order.total)  // external system
verify(email_service).send(confirmation)      // external system
```

**Priority:** prefer output-based > state-based > communication-based. Push your design toward pure functions and value objects to maximize the proportion of output-based tests.

---

## The TDD Cycle

The fundamental rhythm of test-driven development:

**Red.** Write a test that fails. The test defines the next increment of behavior. It should be small — one assertion, one concept. Run it and watch it fail. If it passes immediately, either the behavior already exists or the test is wrong.

**Green.** Write the minimum production code to make the test pass. Don't generalize, don't optimize, don't clean up. Just make it green. The goal is to get from red to green as fast as possible.

**Refactor.** Now clean up both the production code and the test. Remove duplication, improve names, extract functions, simplify. All tests should remain green throughout. This is where design emerges — driven by real needs, not speculation.

**The discipline:** never write production code without a failing test. Never refactor with a failing test. The cycle keeps you in a state where the code always works and every behavior is tested.

**When to use TDD vs. test-after:** TDD works best for new logic with clear inputs and outputs. For exploratory work, prototyping, or integration code, it's often more practical to write the code first and add tests immediately after, before moving on. The non-negotiable part is: code does not ship without tests, regardless of which came first.

---

## Test Doubles

Five types of test doubles, each with a specific purpose. Using the wrong type leads to brittle or meaningless tests.

**Dummy.** An object passed to satisfy a parameter but never actually used. The test doesn't care about it. Example: a null logger passed to a constructor that requires one.

**Stub.** Returns predefined answers to calls made during the test. It feeds data into the system under test. Stubs replace **incoming** dependencies — things the system reads from.

**Spy.** Records calls made to it so you can verify them later. A spy is a manually written mock. Use for verifying **outgoing** interactions with external systems.

**Mock.** Like a spy but with built-in verification — pre-programmed with expectations about which calls should happen. Mocks verify **outgoing** interactions with unmanaged dependencies.

**Fake.** A working implementation with a shortcut that makes it unsuitable for production. An in-memory database, a local file system instead of cloud storage, a fake HTTP server. Fakes replace **managed dependencies** — things you control but want to simplify for testing.

**The critical distinction:**
- **Incoming interactions** (system reads data): use **stubs** or **fakes**. Never assert on them — asserting that a stub was called is testing implementation.
- **Outgoing interactions with external systems** (system writes to database, sends email, calls API): use **mocks** or **spies**. Assert on these — they're part of the observable behavior.
- **Outgoing interactions with internal collaborators** (system calls another class in your codebase): use **real instances**. Don't mock what you own.

---

## Property-Based Testing

Instead of specifying individual examples, define properties that should hold for all valid inputs and let the framework generate test cases. Property-based testing excels at finding edge cases that example-based tests miss — off-by-ones, empty inputs, Unicode, integer overflow, and combinations you didn't think of.

**When to use:** algorithms with clear invariants (sort output is ordered and has same elements), serialization round-trips (encode then decode equals original), idempotent operations, commutative operations, any function with a known relationship between input and output.

**When not to use:** UI interactions, integration tests with external systems, code where the expected output can only be determined by running the code itself.

**The process:** identify a property ("for all valid inputs, sorting then checking order should hold"), write a generator for the input space, let the framework find counterexamples, then shrink to the minimal failing case. When a property test finds a bug, add the minimal failing case as a regression example-based test.

See reference files for language-specific frameworks: Hypothesis (Python), proptest (Rust), jqwik (Java), RapidCheck (C++).

---

## Testing Legacy Code

Legacy code is code without tests. The challenge: you can't refactor safely without tests, but you can't write tests without some refactoring. Feathers provides a disciplined way to break this cycle.

### The Legacy Code Change Algorithm

1. **Identify change points.** What do you need to change?
2. **Find test points.** Where can you observe the behavior you need to preserve?
3. **Break dependencies.** Make the code testable without changing its behavior.
4. **Write characterization tests.** Tests that document what the code currently does, not what it should do.
5. **Make changes and refactor.** Now you have a safety net.

### Characterization Tests

A characterization test captures the current behavior of existing code. You're not asserting what's correct — you're asserting what's true right now, so that you'll know if you accidentally change it.

**Process:** call the code with known inputs, observe what it actually produces, and write assertions that match. If the code returns 42 for some input, assert 42 — even if you suspect 42 might be wrong. The point is to detect unintended changes, not to validate correctness. Correctness can be addressed once you have the safety net.

### Finding Seams

A seam is a place where you can alter behavior without editing the code at that point. Feathers identifies three types:

**Object seam.** Replace a dependency via polymorphism — extract an interface and inject a test implementation. The most common and cleanest seam.

**Preprocessing seam.** In languages with preprocessors or compile-time configuration, swap implementations at build time.

**Link seam.** Replace a dependency at the linking stage — provide a test implementation of a library function. Language-specific; see reference files.

**The goal of seam identification:** find the narrowest point where you can break a dependency chain to get the code under test. Don't refactor the whole class — find one seam, write one test, make one change. Expand coverage incrementally.

### Breaking Dependencies Safely

**Extract and Override.** Extract the dependency interaction into a virtual method, then override it in a test subclass. Minimally invasive — doesn't require changing the production constructor.

**Parameterize Constructor.** Add the dependency as a constructor parameter with a default that preserves current behavior. Production code keeps working; tests pass in a test double.

**Wrap Method.** When you need to add behavior around an existing method, wrap it rather than modifying it. Write the wrapper test-first, then call the original.

These are not aspirational refactoring patterns — they're survival techniques for getting a test foothold in code that wasn't designed for testability.

---

## Test Organization and Hygiene

### Arrange-Act-Assert

Every test follows a three-phase structure:

**Arrange** — set up the preconditions and inputs. Create objects, configure state, prepare test data.

**Act** — execute the behavior under test. Ideally a single method call or operation.

**Assert** — verify the outcome. Check return values, observable state, or interactions with external dependencies.

Keep these phases visually distinct. If a test is hard to classify into three phases, it's probably testing too many things.

### One Concept Per Test

Each test should verify a single behavior or scenario. This doesn't necessarily mean one assertion — multiple assertions are fine when they all verify different facets of the same outcome. But a test that arranges, acts, asserts, then arranges again and acts again is two tests crammed into one.

### Name Tests for Behavior

A test name should describe the scenario and expected outcome, not the method being tested.

```
Good:  expired_subscription_denies_access
Good:  order_with_no_items_is_rejected
Good:  discount_applies_to_gold_tier_members

Bad:   test_calculate
Bad:   test_process_order_1
Bad:   testGetUser
```

When a test fails, the name should tell you what broke without reading the test body.

### Test Smells to Watch For

**Fragile test.** Breaks when production code is refactored without behavior change. Cause: testing implementation details. Fix: test through the public API only.

**Obscure test.** Hard to understand what it's testing. Cause: too much setup, shared fixtures, unclear assertions. Fix: make each test self-contained and readable.

**Slow test.** Takes seconds or more. Cause: real I/O, unnecessary integration, heavy setup. Fix: isolate external dependencies, use fakes for managed dependencies.

**Erratic test.** Sometimes passes, sometimes fails without code changes. Cause: shared mutable state between tests, time-dependent logic, concurrency. Fix: isolate test state completely, use deterministic time sources.

**Test with no assertions.** Runs code but never verifies anything. Either a work in progress or dead weight. Fix: add assertions or delete.

**Conditional test logic.** Tests with if/else or loops. Tests should be straight-line code — branching makes them hard to reason about. Fix: split into multiple tests.

---

## Deciding What to Test

Not all code deserves the same test investment. Focus testing effort where it produces the most value.

**High value:** domain logic, business rules, algorithms, state machines, validation logic, any code where a bug means wrong data or wrong decisions. Test thoroughly with unit tests.

**Medium value:** coordination code that orchestrates calls between domain logic and external systems. Cover with integration tests that verify the wiring is correct, not every edge case.

**Low value:** trivial code with no branching (simple getters, data containers, pass-through methods), and overcomplicated code that needs refactoring more than testing. For trivial code, tests add maintenance cost without meaningful regression protection. For overcomplicated code, write characterization tests as a safety net, then refactor.

**The coverage trap:** high code coverage is not the goal. A suite of shallow tests that touch every line but verify nothing meaningful gives 100% coverage and zero confidence. A smaller suite of well-targeted tests on complex domain logic gives less coverage but far more protection. Optimize for test value, not coverage percentage.

---

## Integration Test Strategy

Integration tests verify that components work together correctly. They are slower than unit tests but catch wiring bugs, configuration errors, and contract mismatches that unit tests cannot.

**Use real infrastructure when:** the behavior depends on the infrastructure's semantics (SQL query correctness, message ordering guarantees, transaction isolation). Test containers or in-memory equivalents keep these fast and repeatable.

**Use fakes when:** you control the dependency and its behavior is simple enough to replicate faithfully (in-memory repositories, local file systems instead of cloud storage).

**Use mocks when:** the dependency is external and unmanaged (third-party APIs, payment gateways, email services). Verify the outgoing interaction, not the response processing.

**Keep integration tests focused.** Each test should verify one integration point, not an entire workflow. End-to-end tests cover full workflows; integration tests cover the seams.

---

## Applying This Skill

When writing new code:
1. Start with a failing test (red) for the next behavior increment
2. Write minimum code to pass (green)
3. Refactor both production and test code
4. Test behavior through the public API — never test implementation details
5. Use real collaborators for internal dependencies, mocks only for external systems
6. Prefer output-based tests, then state-based, then communication-based

When adding to existing untested code:
1. Identify the change point and find a test point
2. Find a seam to break the dependency
3. Write characterization tests to document current behavior
4. Make your change under the safety net
5. Add targeted tests for the new behavior

When reviewing tests:
- Does it test behavior or implementation?
- Will it survive a refactoring that preserves behavior?
- Is the test name descriptive of the scenario?
- Is the Arrange-Act-Assert structure clear?
- Are mocks used only for unmanaged external dependencies?

When the target language is known, read the corresponding file in `references/` for language-specific frameworks and idioms.
