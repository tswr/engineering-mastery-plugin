---
name: code-review
description: "Apply when reviewing a pull request, diff, or code change using a structured multi-pass methodology. Connects to clean-code, clean-architecture, defensive-programming, testing, performance, and security skills as its review checklist. Trigger on requests to review code, evaluate merge readiness, or give feedback on changes."
---

# Code Review Skill

Code review is the highest-leverage quality practice in a large codebase. A good review catches defects before they ship, spreads knowledge across the team, maintains consistency, and raises the bar over time. A bad review is either a rubber stamp (misses real issues) or a gatekeeping exercise (blocks progress on style preferences).

The goal: **improve the overall health of the codebase over time while not blocking reasonable progress.** A change doesn't need to be perfect to be approved. It needs to improve the codebase and not introduce problems that outweigh the improvement.

---

## 1. The Review Order

Review in this order. Each layer builds on the previous one — there's no point polishing naming if the design is wrong.

### Pass 1: Purpose and Scope

Before reading any code, understand what the change is trying to do.

**Read the description first.** What problem does this solve? Is there a linked issue, design doc, or spec? If the description doesn't explain the *why*, ask for it — code without context is unreviable.

**Check the scope.** Does the change do one thing, or is it several changes bundled together? A change that refactors, adds a feature, and fixes a bug should be three separate changes. Large, unfocused changes get worse reviews because reviewer attention is a finite resource.

**Right approach?** Before reading implementation details, ask: is this the right way to solve this problem? Is there a simpler approach? Does it duplicate something that already exists? This is the most valuable feedback a reviewer can give and the earliest point to give it — redesigning after implementation is expensive.

### Pass 2: Correctness and Logic

Now read the code. The primary question: **does this change do what it claims to do?**

**Trace the logic.** Walk through the main code paths mentally. Do the inputs produce the expected outputs? Are edge cases handled? What happens on empty input, null values, boundary conditions, maximum sizes?

**Error handling.** What happens when things fail? Are errors propagated correctly? Are resources cleaned up on failure paths? Is the error handling appropriate for the failure type (contract violation vs. operational failure — see defensive-programming skill)?

**Concurrency.** If shared state is involved: is access synchronized? Could there be a race condition? Is there a TOCTOU (time-of-check-to-time-of-use) vulnerability? Are locks held for the minimum duration? (See concurrency skill for detailed patterns.)

**State management.** Does the change leave the system in a consistent state? If it modifies multiple pieces of state, what happens if it fails partway through? Are invariants preserved?

### Pass 3: Design and Architecture

**Dependency direction.** Does the change respect the Dependency Rule? Do new dependencies point inward toward domain logic, or outward toward infrastructure? (See clean-architecture skill.)

**Module boundaries.** Is the new code in the right module/package/bounded context? Does it introduce coupling between modules that should be independent? Is it deep (meaningful complexity behind a simple interface) or shallow (pass-through wrappers)?

**Extensibility vs. simplicity.** Does the change over-engineer for hypothetical future requirements? Or does it paint itself into a corner that will be expensive to change? The right balance: the simplest design that handles current requirements and doesn't close obvious doors.

**Consistency with existing patterns.** Does the change follow the conventions established in this codebase? Introducing a new pattern is fine if it's better, but the reviewer should ask: is this a deliberate improvement, or did the author just not know about the existing pattern?

**API design.** If the change introduces or modifies a public API: are resources named consistently? Is it backward-compatible? Are errors structured and actionable? Is pagination cursor-based for large collections? (See api-design skill.)

### Pass 4: Defensive Quality

**Immutability.** Are variables and fields const/final/immutable where possible? Are parameters treated as read-only? (See defensive-programming skill.)

**Visibility.** Are new members as private as possible? Are classes sealed/final unless designed for extension? Does the change expose internal state that should be hidden?

**Contracts.** Are preconditions checked at boundaries? Are assertions used for internal invariants? Can illegal states be constructed through the new API?

**Resource safety.** Are resources tied to scope (RAII, try-with-resources, context managers)? Is there any path where a resource could leak?

### Pass 5: Tests

**Existence.** Does the change come with tests? If not, why not? Untested code is a liability.

**Quality.** Do the tests verify behavior or implementation? Would they survive a refactoring? (See testing skill.) Are they testing the right things — the complex logic, the edge cases, the error paths — or just the happy path?

**Coverage of the change.** Do the tests cover the new code paths introduced by this change? Are boundary conditions tested? Are error paths tested?

**Test readability.** Can you understand what each test is verifying from its name and structure? Is the Arrange-Act-Assert structure clear?

### Pass 6: Naming and Readability

**Names.** Do variable, function, and class names reveal intent? Could you understand the code without comments? (See clean-code skill.)

**Complexity.** Are functions small and single-purpose? Is there deep nesting that should be flattened? Are there long parameter lists that should be grouped into objects?

**Comments.** Are there comments that should be code (rename, extract function)? Are there missing comments where a non-obvious decision needs explanation? Do comments explain why, not what? (See documentation skill.)

### Pass 7: Performance (When Relevant)

Not every change needs a performance review. Apply this pass when the change is on a known hot path, processes large data volumes, or introduces new I/O operations.

**Algorithmic complexity.** Is the algorithm appropriate for the data size? Are there hidden quadratics (nested loops, repeated linear scans)?

**Memory and allocation.** Does the change allocate unnecessarily in a loop? Are collections pre-sized? Are there opportunities for reuse?

**I/O patterns.** Are individual I/O operations batched where possible? Are there N+1 query patterns?

(See performance skill for detailed checklist.)

### Pass 8: Security (When Relevant)

Apply this pass when the change handles user input, crosses trust boundaries, touches authentication/authorization, or manages secrets.

**Injection vectors.** Does user input flow into SQL, shell commands, templates, or log formats without parameterization or sanitization?

**Authentication and authorization.** Is authorization enforced at every entry point, including object-level? Are credentials stored securely, never hardcoded or logged?

**Secrets exposure.** Are API keys, tokens, or passwords hardcoded in source, logged, or exposed in error messages?

**Trust boundaries.** Where does untrusted data enter? Is it validated and converted to internal types at the boundary?

(See security skill for detailed checklist.)

---

## 2. How to Give Feedback

The way you give feedback determines whether it's acted on or ignored.

### Be Specific and Actionable

**Bad:** "This function is confusing."
**Good:** "This function does three things: validates input, applies the discount, and persists the result. Consider extracting the validation and persistence into separate functions so each one has a single responsibility."

**Bad:** "Use better names."
**Good:** "`d` doesn't reveal intent — since this represents elapsed days since enrollment, consider `days_since_enrollment`."

Every comment should make clear: what the issue is, why it matters, and what a fix looks like.

### Distinguish Severity

Not all feedback is equally important. Signal which comments are blocking and which are suggestions.

**Blocking (must fix before merge):**
- Bugs and logic errors
- Security vulnerabilities
- Missing error handling that could cause data corruption
- Missing tests for critical paths
- Architectural violations that will be expensive to fix later

**Non-blocking (should fix, but don't block merge):**
- Naming improvements
- Minor structural suggestions
- Performance improvements on non-hot paths
- Style preferences not covered by automated linting

**Nitpick (take it or leave it):**
- Subjective style preferences
- Alternative approaches that aren't clearly better
- "I would have done it differently" without a concrete quality argument

**Label your comments.** Use prefixes like `[blocking]`, `[suggestion]`, `[nit]`, or `[question]` so the author knows how to prioritize.

### Ask Questions Before Making Demands

If something looks wrong but you're not sure, ask rather than assert. "Why is this mutable?" is better than "Make this const." The author may have a reason you don't see. Questions also teach — the author thinks about the answer, which is more valuable than mechanically applying a fix.

### Praise What's Good

If the change introduces a clean abstraction, a well-designed test, or a clever-but-readable solution, say so. Positive feedback reinforces good practices and makes critical feedback easier to receive.

### Don't Bikeshed

If you've been debating a naming choice for three comment rounds and neither option is clearly better, let it go. The time spent arguing over marginal differences is more expensive than either option. Reserve your review capital for things that matter.

---

## 3. The Approval Bar

**Approve when:**
- The change does what it claims
- It improves (or doesn't degrade) the overall health of the codebase
- Tests cover the new behavior
- No blocking issues remain
- The design is reasonable (not necessarily what you would have done, but sound)

**Don't block on:**
- Style preferences that aren't team conventions
- Alternative designs that aren't clearly better
- Perfect code — good enough to ship and improve later is fine
- Missing tests for pre-existing untested code (the change shouldn't make coverage worse, but it doesn't have to fix all existing gaps)

**Block on:**
- Bugs that will affect production
- Security issues
- Missing tests for new behavior
- Architectural decisions that will be very expensive to reverse
- Changes that violate team conventions without justification

---

## 4. Review Efficiency

Research shows reviewer effectiveness drops sharply after 60-90 minutes and after 200-400 lines of code. Structure your reviews accordingly.

**Small changes get better reviews.** If a change is large, it's reasonable to ask the author to split it. A 2000-line change will get a worse review than four 500-line changes, no matter how careful the reviewer is.

**Review promptly.** Stale reviews block the author and create merge conflicts. Aim to respond within one business day. A fast "I'll look at this tomorrow, it's large" is better than silence.

**Use tooling.** Let linters, formatters, type checkers, and CI catch mechanical issues. Don't spend review time on things a machine can check. Your attention is for design, correctness, and clarity — things only a human can evaluate.

**One pass, not ten.** Read the change thoroughly once. Collect all your comments, then submit them together. Drip-feeding comments across multiple rounds is frustrating for the author and wastes both parties' time.

---

## 5. Reviewing Your Own Code Before Submitting

Before requesting review, self-review against the same checklist. You'll catch obvious issues and save your reviewer's attention for the things you can't see yourself.

**Checklist before submitting:**
- [ ] Description explains what and why (not just what)
- [ ] Change does one thing (not bundled with unrelated fixes)
- [ ] All new variables/fields are immutable unless justified
- [ ] All new members are as private as possible
- [ ] Error paths are handled and tested
- [ ] Tests exist for new behavior and pass
- [ ] No debug code, commented-out code, or TODOs without tracking
- [ ] No hardcoded secrets, credentials, or sensitive data
- [ ] CI passes (lint, type check, tests)
- [ ] Logging and observability are adequate for production debugging
- [ ] The diff tells a coherent story when read top-to-bottom

---

## Applying This Skill

When reviewing code:
1. Read the description and understand the purpose before reading code
2. Follow the eight-pass order: purpose → correctness → design → defensive → tests → readability → performance → security
3. Label comment severity: blocking, suggestion, nit, question
4. Approve when the change improves the codebase, even if imperfect
5. Block only on bugs, security, missing tests, or costly architecture mistakes

When the review involves specific concerns:
- Defensive quality → reference the defensive-programming skill
- Architecture and boundaries → reference the clean-architecture skill
- Test quality → reference the testing skill
- Performance on hot paths → reference the performance skill
- Naming and function structure → reference the clean-code skill
- Security and trust boundaries → reference the security skill
- API contracts and compatibility → reference the api-design skill
- Concurrency and thread safety → reference the concurrency skill
- Logging and observability → reference the observability skill
- Code restructuring → reference the refactoring skill
- Comments and documentation → reference the documentation skill
- Agent-readiness and context engineering → reference the ai-native skill
- Algorithm choice and complexity → reference the algorithms skill
- Frontend components and accessibility → reference the frontend skill
- Mobile platform patterns → reference the mobile skill
- ML model quality and experiment design → reference the ml-experimentation skill
- A/B test methodology → reference the ab-testing skill

When writing code for review:
- Self-review against the checklist before submitting
- Write a clear description that explains the why
- Keep changes small and focused
- Include tests alongside the code
