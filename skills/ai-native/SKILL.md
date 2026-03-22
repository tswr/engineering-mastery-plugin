---
name: ai-native
description: "Apply when structuring a codebase for effective AI-assisted development: agent instruction files, context engineering, module boundaries for LLM comprehension, testing as agent feedback, and workflow patterns for human-agent collaboration. Trigger on mentions of AI-native, agentic coding, CLAUDE.md, AGENTS.md, context engineering, AI-friendly code, or LLM-oriented development."
---

# AI-Native Codebase Skill

The quality of AI-assisted output depends more on codebase structure and context than on the model itself. A well-structured codebase with clear conventions, explicit naming, and fast test suites amplifies AI coding agents. A tangled codebase with implicit conventions, deep inheritance hierarchies, and slow feedback loops degrades them. This is not speculation — Borg and Tornhill's peer-reviewed research (2026) found that AI coding assistants increase defect risk by at least 30% when applied to unhealthy code. The agent magnifies whatever it finds.

AI-native development is an emerging discipline (2024-2026) that treats the codebase itself as the primary interface between human engineers and AI agents. The principles below synthesize the best available evidence from official documentation, peer-reviewed research, and substantial engineering practice. They are language-agnostic and apply to any codebase where AI agents participate in development.

As Anthropic's own research puts it: "If I write sloppy code, it will definitely produce me more sloppy code." The inverse is equally true. A codebase designed for AI comprehension produces higher-quality generated code, fewer defects, and faster iteration cycles.

---

## 1. Context is the Scarce Resource

The goal of context engineering is to find "the smallest set of high-signal tokens that maximize the likelihood of some desired outcome" (Anthropic). Every token in an agent's context window either helps or hurts. Irrelevant context does not merely waste space — it actively degrades output quality by diluting the signal the agent uses to generate code.

Research from HumanLayer shows that frontier LLMs can follow approximately 150-200 instructions consistently, and agent instruction files should stay under 300 lines. Treat context budget the way you treat memory budget — deliberately, with awareness of limits.

Fowler distinguishes three tiers of context: always-loaded context (CLAUDE.md — repo-wide conventions loaded on every interaction), path-based rules (loaded when relevant files are touched), and lazy-loaded domain skills (detailed knowledge loaded on demand). Design your project's context architecture across these tiers deliberately. The wrong tier wastes budget — always-loading domain-specific knowledge that applies to one module fills the window with irrelevant tokens on every other task.

**Progressive disclosure.** Do not dump everything into one file. Place repo-wide conventions in the root instruction file, directory-specific rules in local overrides, and detailed domain knowledge in referenced documents that agents load when needed.

**Metadata as signals.** File names, folder structure, and naming conventions are context. A well-organized directory communicates intent without a single line of documentation. An agent navigating `features/user-registration/` understands more than one navigating `src/module7/`.

**Compaction.** In long sessions, accumulated context becomes noise. Summarize and preserve key decisions. Discard resolved details. Sub-agent architectures can isolate context-heavy subtasks so the main agent's window stays clean.

**Just-in-time context.** Provide information at the moment the agent needs it, not before. A test command is relevant when the agent is about to run tests. A deployment constraint is relevant when the agent is writing deployment code. Front-loading all possible context is wasteful — structure your documentation and instruction files so agents can pull context when they encounter a task, not carry it from the start.

---

## 2. Agent Instruction Files

CLAUDE.md, AGENTS.md, and similar files are the "README for AI agents" (Builder.io). They bridge the gap between what the codebase expresses implicitly and what an agent needs stated explicitly.

Anthropic's best practices are specific about what belongs in these files: commands the agent cannot guess, style rules that differ from defaults, architectural decisions and their rationale, testing conventions, and common gotchas.

Equally important is what to exclude. Do not include anything the agent can learn by reading the code. Do not restate standard language conventions. Do not add formatting or linting rules — enforce those with tooling, not with instructions. Do not describe every file — the agent can navigate the codebase.

From Spotify's engineering guidance: use concrete examples rather than abstract rules. Show a real function call, a real test invocation, a real directory path. An example is worth more than a paragraph of explanation because the agent can pattern-match against concrete syntax far more reliably than it can interpret abstract guidance.

Treat instruction files as code. Review them in pull requests. Prune stale entries. Version-control them alongside the codebase. An outdated instruction file is worse than no instruction file — it teaches the agent wrong conventions (HumanLayer). When an agent consistently ignores or misapplies an instruction, the instruction is likely unclear — rewrite it with a concrete example rather than adding more words.

From Tornhill's six patterns: encode principles in these files, not procedures. "Functions should have a single responsibility" is a principle. "Run eslint before committing" is a procedure — and one the agent can discover from the project's pre-commit hooks. Principles guide judgment; procedures automate. Agent instruction files should focus on the former.

---

## 3. Explicit Over Implicit

Ronacher's principle — "do the dumbest possible thing that will work" — is even more valuable when agents are your collaborators. Avoid magic, hidden configuration, deep inheritance hierarchies, monkey patching, and implicit behavior. Prefer functions over classes. Prefer plain SQL over complex ORMs. Every layer of cleverness is a comprehension step the agent must trace, and agents trace poorly through indirection.

This principle has empirical backing. Wang et al. (arXiv 2307.12488) found that anonymizing variable, function, and class names reduced LLM code analysis scores to as low as 23.73%. Names are not decoration — they are the primary signal LLMs use to understand what code does. Descriptive, self-documenting names are a prerequisite for effective AI-assisted development.

**Naming is context.** Choose names that carry maximum information. A function called `calculate_monthly_revenue_from_subscriptions` gives the agent everything it needs. A function called `calc` gives it nothing.

**Avoid indirection.** Every proxy, decorator, metaclass, and abstract factory is a comprehension barrier. Flatten call chains where possible. When abstraction is necessary, make it shallow and well-named. A three-layer call chain where each layer delegates to the next adds zero value and three comprehension steps for the agent.

**Prefer common patterns.** Agents have strong priors on standard library patterns and well-known idioms. They have weak priors on bespoke abstractions. Use the patterns the agent already knows. When you must introduce a custom pattern, document it explicitly in the agent instruction file — do not assume the agent will infer it from a single usage.

**Make dependencies visible.** Explicit imports, constructor injection, clear function signatures. Hidden dependencies — service locators, ambient state, globals — are invisible to agents and produce incorrect generated code.

**Logging is agent feedback.** Ronacher emphasizes that logging is critical for agent workflows. When an agent runs code and something goes wrong, log output is often the only diagnostic signal available. Structured, descriptive log messages help agents diagnose failures and self-correct. Silent failures — code that returns wrong results without any observable signal — are the hardest class of bugs for agents to detect.

---

## 4. Architecture for Bounded Context Windows

LLM output quality degrades past approximately 40% of the context window (DEV Community). This means architecture is no longer just a human concern — it directly affects the quality of AI-generated code.

Hightower reports that Vertical Slice Architecture reduces AI context errors by 60-80% compared to horizontal layered architectures. Dubovikov's "LLM-Oriented Programming" reaches the same conclusion: design modules to fit in context. The implication is concrete: architectural decisions made years ago — how you organize directories, how you split modules, how you define boundaries — now have a measurable effect on AI-generated code quality.

**Vertical slices over horizontal layers.** Organize by feature, not by technical layer. An agent working on "user registration" should find the handler, the business logic, the data access, and the tests in one place — not scattered across `controllers/`, `services/`, `repositories/`, and `models/`.

Vertical slicing reduces the number of files the agent must load to understand a feature. It also reduces the risk of unintended side effects — changes to one feature's slice do not touch files shared with other features.

**Small, self-contained modules.** Each module should be comprehensible without loading the entire codebase. Define clear boundaries and explicit interfaces. Minimize coupling between modules so agents can work on one without understanding all the others. Dubovikov's guidance is specific: each module should be small enough to fit entirely within a single context window, with all its dependencies resolvable from its interface declarations.

**Colocate tests with code.** Place tests next to the code they test. An agent modifying a module finds the tests in the same directory, runs them immediately, and gets feedback without searching the codebase. Colocation also makes it obvious when a module lacks tests — an empty test file next to a source file is a visible gap, while a missing file in a distant `tests/` directory is invisible.

**Keep files focused.** A 2000-line file exceeds useful context. Monolithic service classes and god objects are particularly hostile to agents. Smaller files with clear, single responsibilities are more navigable and produce better agent output.

**Minimize obscure dependencies.** Dubovikov stresses minimizing dependencies that an agent cannot easily trace. When a module depends on another, the dependency should be obvious from imports and type signatures. Cross-module dependencies that exist only at runtime — through configuration, reflection, or event buses — force the agent to guess at connections it cannot see. Where such dependencies are necessary, document them explicitly in the module's header or in the nearest agent instruction file.

---

## 5. Tests as Agent Feedback

Red/Green TDD is particularly effective with agents (Willison). The agent writes or modifies code, runs the tests, and iterates based on the result. This feedback loop is the agent's primary mechanism for self-correction. Without it, agents produce plausible-looking code with no verification.

The pattern is simple: write a failing test (red), ask the agent to make it pass (green), then refactor. The test provides the specification and the verification in one artifact.

**Fast test suites are non-negotiable.** Agents iterate by running tests. A test suite that takes minutes destroys the feedback loop. Target seconds. Ronacher's principle applies: make tools fast-responding — crashes are fine because they give immediate signal, but hangs are catastrophic because the agent cannot distinguish a hang from slow progress.

**Tests as specification.** Well-named tests with clear assertions tell the agent what the code should do. The agent reads tests to understand intent and writes code to make them pass. Invest in test readability as seriously as production code readability.

**Automated quality gates.** Linters, type checkers, and formatters should run on every change and provide immediate, unambiguous signal. From Tornhill: safeguard with automated checks at generation time, pre-commit, and PR review. Layer your defenses.

**Agent output is untrusted.** Treat AI-generated code like code from a junior developer (Osmani). It may be correct, it may be plausible but wrong, it may introduce subtle regressions. Automated checks catch the mechanical errors; human review catches the conceptual ones.

**Coverage as behavioral guardrail.** From Tornhill: use test coverage not as a target to hit but as a guardrail that defines the boundary of safe modification. If a module has 90% coverage, the agent can modify it with confidence that regressions will be caught. If a module has 20% coverage, the agent is flying blind.

Make coverage data visible to agents so they can calibrate their confidence appropriately. When coverage is low, the agent should write tests before modifying the code — establishing the safety net first, then making changes against it.

---

## 6. Code Health is a Prerequisite

Borg and Tornhill's peer-reviewed finding bears repeating: AI coding assistants increase defect risk by at least 30% when applied to unhealthy code. Code health is not a nice-to-have — it is a prerequisite for effective AI-assisted development.

Tornhill's recommendation is concrete: assess AI readiness (target code health scores of 9.5+), and refactor to expand AI-safe zones before delegating work to agents. Code that is already difficult for humans to maintain becomes actively dangerous when agents modify it.

**Refactor before delegating.** If a module is tangled, tightly coupled, or poorly tested, refactor it to health before asking an agent to modify it. The investment pays for itself in higher-quality generated code and fewer defects to debug. This is not optional cleanup — it is a prerequisite step that determines whether AI assistance helps or harms.

**Maintain high test coverage.** Coverage is the agent's safety net. Without it, the agent cannot verify that its changes preserve existing behavior.

Coverage serves as a behavioral guardrail — not a vanity metric but a functional constraint on what the agent can safely modify.

**Reduce coupling.** Tightly coupled code requires understanding distant modules to make local changes. Agents working in tightly coupled codebases pull in too much context, exceed useful window limits, and produce errors.

Loosely coupled code lets agents work on isolated units with high confidence. The practical test: can an agent modify this module by reading only this module and its immediate interface contracts? If the answer is yes, the coupling is appropriate.

**Code health is measurable.** Tornhill's approach is data-driven: use static analysis tools to identify hotspots where complexity, coupling, and change frequency intersect. These hotspots are where AI-generated defects concentrate. Prioritize refactoring them before expanding agent-assisted development to those areas.

---

## 7. Workflow: Plan, Execute, Verify

Osmani describes spec-driven development as "waterfall in 15 minutes" — create a comprehensive specification before coding, then execute in small iterative chunks. Break work into small iterative pieces, verify each piece before moving on, and commit after each success.

Willison distinguishes "agentic engineering" (an expert using agents as force multipliers) from "vibe coding" (hoping the agent produces something correct). The difference is workflow discipline — the expert plans, specifies, and verifies at each step.

**Spec before code.** Write a clear specification or plan before asking the agent to implement. Include acceptance criteria, constraints, and edge cases.

From Spotify: state preconditions, use concrete examples, and define verifiable goals. The spec is the agent's primary input — invest in its quality. A vague spec produces vague code; a precise spec produces precise code.

**One task at a time.** Small, focused prompts produce better results than large, ambiguous ones. Break work into discrete units. From Spotify: one change per prompt.

This also limits blast radius when the agent makes a mistake. A failed attempt on a small, isolated task is easy to revert. A failed attempt on a large, cross-cutting change may leave the codebase in an inconsistent state that is harder to recover from than to redo.

**Commit as checkpoints.** Commit after each successful task (Osmani). If the next step fails, revert to the last known-good state rather than debugging forward through compounding errors. Treat the commit history as a series of save points, not a diary of attempts. This is especially important with agents because the cost of retrying from a clean state is low — the agent can regenerate the work quickly — while the cost of debugging a tangled state is high.

**Review everything.** Agents are productive but fallible. Every generated change must be reviewed with the same rigor as human-authored code.

Request agent feedback — ask the agent to explain its reasoning, flag uncertainties, and identify what it is not confident about (Spotify). An agent that can articulate its uncertainty is more useful than one that presents everything with equal confidence.

**Tailor prompts to the agent.** From Spotify: different agents have different strengths and conventions. Learn what your agent handles well and where it struggles. Adapt your prompting style accordingly rather than using one-size-fits-all instructions. An expert using agents effectively understands both the domain and the tool — this is what distinguishes agentic engineering from vibe coding.

---

## 8. Minimize Unnecessary Dependencies

With cheap code generation, the calculus around dependencies has shifted (Honeycomb). Before AI agents, writing a small utility yourself was almost always wrong — the library was better tested, better maintained, and free. The cost of writing exceeded the cost of depending.

Now that cost equation has inverted for simple utilities — an agent generates a tested implementation in seconds, while the dependency carries ongoing maintenance, upgrade, and security costs indefinitely.

Every external dependency brings transitive dependencies, a security surface, an upgrade burden, and a risk of abandonment. When an agent can generate a well-tested, focused implementation in seconds, pulling in a package that merely saves typing is a net negative.

This applies to small utilities, simple parsers, basic data transformations, and similar bounded problems where the implementation is straightforward and the behavior is fully specifiable through tests.

This is not an argument against all dependencies. Complex, well-maintained libraries — cryptography, database drivers, serialization frameworks — remain essential. The principle is to reconsider the reflexive reach for a package when the alternative is a small, self-contained, tested module that you fully control.

The decision criteria are concrete: How complex is the functionality? How actively maintained is the dependency? How large is its transitive dependency tree? How critical is it to your security surface? For simple, bounded functionality, ownership is now often cheaper than dependency.

---

## Applying This Skill

When structuring a codebase for AI-assisted development, apply these defaults:
1. Keep agent instruction files under 300 lines, focused on what the agent cannot infer from code
2. Organize context in tiers — always-loaded conventions, path-based rules, and lazy-loaded domain knowledge
3. Use descriptive, self-documenting names for all identifiers — naming is the primary context mechanism for LLMs
4. Prefer vertical slice architecture over horizontal layers to minimize context loading per feature
5. Keep modules small and self-contained, with colocated tests
6. Maintain fast test suites (seconds, not minutes) as the agent's primary feedback loop
7. Assess code health before delegating to agents — refactor unhealthy modules first
8. Write a specification with acceptance criteria before asking an agent to implement
9. Commit after each successful agent task to create revertible checkpoints
10. Evaluate dependencies critically — generate and own simple utilities rather than importing packages that merely save typing
