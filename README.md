# Engineering Mastery Plugin

Eighteen skills for professional software engineering, built from canonical literature. Language-agnostic principles with idiomatic reference files for multiple languages and platforms.

## Skills

| Skill | Lines | References | Sources |
|-------|-------|------------|---------|
| **clean-code** | 176 | — | Clean Code (Martin) |
| **clean-architecture** | 192 | — | Clean Architecture (Martin), A Philosophy of Software Design (Ousterhout), DDD (Evans) |
| **defensive-programming** | 154 | C++, Python, Rust, Java | Code Complete (McConnell), Effective C++ (Meyers), Pragmatic Programmer (Hunt & Thomas), Design by Contract (Meyer) |
| **testing** | 260 | C++, Python, Rust, Java | Unit Testing (Khorikov), Working with Legacy Code (Feathers), xUnit Test Patterns (Meszaros), TDD By Example (Beck) |
| **performance** | 202 | C++, Python, Rust, Java | Systems Performance (Gregg), What Every Programmer Should Know About Memory (Drepper), Art of Writing Efficient Programs (Pikus) |
| **code-review** | 219 | — | Google Engineering Practices, Best Kept Secrets of Peer Code Review (Cohen) |
| **security** | 209 | C++, Python, Rust, Java | OWASP, CERT Secure Coding, Threat Modeling (Shostack), The Tangled Web (Zalewski) |
| **observability** | 184 | C++, Python, Rust, Java | Observability Engineering (Majors et al.), Debugging (Agans), Site Reliability Engineering (Beyer et al.), Distributed Systems Observability (Sridharan), OpenTelemetry |
| **refactoring** | 200 | C++, Python, Rust, Java | Refactoring (Fowler), Working with Legacy Code (Feathers), A Philosophy of Software Design (Ousterhout) |
| **api-design** | 200 | C++, Python, Rust, Java | API Design Patterns (Geewax), Designing Web APIs (Jin et al.), RESTful Web APIs (Richardson & Amundsen), Google API Design Guide, Microsoft REST API Guidelines, RFC 7231, RFC 7807 |
| **concurrency** | 205 | C++, Python, Rust, Java | Designing Data-Intensive Applications (Kleppmann), Java Concurrency in Practice (Goetz), Release It! (Nygard), Art of Multiprocessor Programming (Herlihy & Shavit), Understanding Distributed Systems (Vitillo) |
| **documentation** | 109 | — | A Philosophy of Software Design (Ousterhout), Living Documentation (Martraire), Domain-Driven Design (Evans), Docs for Developers (Bhatti et al.) |
| **ai-native** | 180 | — | Anthropic (Context Engineering, Best Practices), Borg & Tornhill (peer-reviewed, 2026), Wang et al. (arXiv 2307.12488), Ronacher, Willison, Osmani, Fowler, Spotify Engineering |
| **ml-experimentation** | 189 | Python | Designing ML Systems (Huyen), ML Engineering (Burkov), Reliable ML (Chen et al.), Google's Rules of ML (Zinkevich), Model Cards (Mitchell et al.) |
| **ab-testing** | 180 | Python | Trustworthy Online Controlled Experiments (Kohavi et al.), Statistical Methods in Online A/B Testing (Georgiev), Causal Inference: The Mixtape (Cunningham) |
| **mobile** | 240 | iOS, Android, React Native, Flutter | Apple HIG, Material Design, Big Nerd Ranch iOS/Android, Mobile Design Pattern Gallery (Neil), WCAG 2.1 Mobile |
| **frontend** | 201 | React, Vue, Tailwind, TypeScript | Learning Patterns (Hallie & Osmani), Inclusive Design Patterns (Pickering), Refactoring UI (Wathan & Schoger), WCAG 2.1/2.2, Web Vitals |
| **algorithms** | 201 | C++, Python, Rust, Java | CLRS (Cormen et al.), Algorithm Design Manual (Skiena), Programming Pearls (Bentley), Algorithms (Sedgewick & Wayne) |

## Installation

### From GitHub (recommended)

```bash
# Add the marketplace
claude plugin marketplace add tswr/engineering-mastery-plugin

# Install the plugin
claude plugin install engineering-mastery@engineering-mastery-marketplace
```

### From a local clone

```bash
# Clone and add as local marketplace
git clone https://github.com/tswr/engineering-mastery-plugin.git
claude plugin marketplace add ./engineering-mastery-plugin
claude plugin install engineering-mastery@engineering-mastery-marketplace
```

### As standalone skills (without plugin wrapper)

```bash
# Copy all skills to your project
cp -r engineering-mastery-plugin/skills/* .claude/skills/

# Or to your personal skills directory
cp -r engineering-mastery-plugin/skills/* ~/.claude/skills/
```

## How It Works

Each skill triggers automatically based on context. The SKILL.md files load into Claude's context when relevant — you don't need to invoke them manually.

Skills with `references/` directories contain language-specific files (cpp.md, python.md, rust.md, java.md) that Claude reads only when generating code in that language, keeping context efficient.

The **code-review** skill ties all others together — it references all other skills as its review checklist.

## Sources

All skills are built from canonical software engineering literature. Source attributions are kept here (not in skill files) to minimize agent context usage.

- **Clean Code** — Robert C. Martin
- **Clean Architecture** — Robert C. Martin
- **A Philosophy of Software Design** — John Ousterhout
- **Domain-Driven Design** — Eric Evans
- **Code Complete** — Steve McConnell
- **Effective C++ / Effective Modern C++** — Scott Meyers
- **The Pragmatic Programmer** — Andrew Hunt & David Thomas
- **Object-Oriented Software Construction (Design by Contract)** — Bertrand Meyer
- **Unit Testing: Principles, Practices, and Patterns** — Vladimir Khorikov
- **Working Effectively with Legacy Code** — Michael Feathers
- **xUnit Test Patterns** — Gerard Meszaros
- **Test-Driven Development: By Example** — Kent Beck
- **Systems Performance** — Brendan Gregg
- **What Every Programmer Should Know About Memory** — Ulrich Drepper
- **The Art of Writing Efficient Programs** — Fedor Pikus
- **OWASP Top 10 & Cheat Sheet Series**
- **CERT Secure Coding Standards**
- **Threat Modeling: Designing for Security** — Adam Shostack
- **The Tangled Web** — Michal Zalewski
- **Google Engineering Practices** (code review guidelines)
- **Best Kept Secrets of Peer Code Review** — Jason Cohen
- **Observability Engineering** — Charity Majors, Liz Fong-Jones, George Miranda
- **Debugging: The 9 Indispensable Rules** — David J. Agans
- **Site Reliability Engineering** — Betsy Beyer, Chris Jones, Jennifer Petoff, Niall Murphy
- **The Site Reliability Workbook** — Betsy Beyer et al.
- **Distributed Systems Observability** — Cindy Sridharan
- **OpenTelemetry Specification** (telemetry data standard)
- **W3C Trace Context** (trace propagation standard)
- **Refactoring: Improving the Design of Existing Code** — Martin Fowler (2nd edition)
- **API Design Patterns** — JJ Geewax
- **Designing Web APIs** — Brenda Jin, Saurabh Sahni, Amir Shevat
- **RESTful Web APIs** — Leonard Richardson, Mike Amundsen
- **Google API Design Guide**
- **Microsoft REST API Guidelines**
- **RFC 7231** (HTTP Semantics) and **RFC 7807** (Problem Details for HTTP APIs)
- **Designing Data-Intensive Applications** — Martin Kleppmann
- **Java Concurrency in Practice** — Brian Goetz et al.
- **The Art of Multiprocessor Programming** — Maurice Herlihy, Nir Shavit
- **Release It!** — Michael Nygard (2nd edition)
- **Understanding Distributed Systems** — Roberto Vitillo
- **Living Documentation** — Cyrille Martraire
- **Docs for Developers** — Jared Bhatti et al. (Google)
- **Effective Context Engineering for AI Agents** — Anthropic Engineering Blog
- **Best Practices for Claude Code** — Anthropic Documentation
- **Context Engineering for Coding Agents** — Martin Fowler
- **How Does Naming Affect LLMs on Code Analysis Tasks?** — Wang et al. (arXiv 2307.12488, peer-reviewed)
- **AI Coding Assistants and Defect Risk** — Markus Borg & Adam Tornhill (peer-reviewed, 2026)
- **Agentic Coding Recommendations** — Armin Ronacher
- **Agentic Engineering Patterns** — Simon Willison
- **Beyond Vibe Coding** — Addy Osmani (Google)
- **Agentic AI Coding: Best Practice Patterns** — Adam Tornhill / CodeScene
- **Context Engineering for Background Coding Agents** — Spotify Engineering
- **Designing Machine Learning Systems** — Chip Huyen
- **Machine Learning Engineering** — Andriy Burkov
- **The Hundred-Page Machine Learning Book** — Andriy Burkov
- **Reliable Machine Learning** — Cathy Chen et al.
- **Google's Rules of ML** — Martin Zinkevich
- **Pattern Recognition and Machine Learning** — Christopher Bishop
- **Model Cards for Model Reporting** — Mitchell et al. (Google, 2019)
- **Approaching (Almost) Any Machine Learning Problem** — Abhishek Thakur
- **Trustworthy Online Controlled Experiments** — Ron Kohavi, Diane Tang, Ya Xu
- **Statistical Methods in Online A/B Testing** — Georgi Georgiev
- **Causal Inference: The Mixtape** — Scott Cunningham
- **Apple Human Interface Guidelines** (HIG)
- **Google Material Design Guidelines**
- **Android Programming: The Big Nerd Ranch Guide** — Bill Phillips et al.
- **iOS Programming: The Big Nerd Ranch Guide** — Christian Keur, Aaron Hillegass
- **Mobile Design Pattern Gallery** — Theresa Neil
- **Google Guide to App Architecture** (Android Jetpack)
- **Programming Flutter** — Carmine Zaccagnino
- **WCAG 2.1/2.2 Guidelines** (W3C)
- **Learning Patterns** — Lydia Hallie & Addy Osmani (patterns.dev)
- **Inclusive Design Patterns** — Heydon Pickering
- **Don't Make Me Think** — Steve Krug
- **Refactoring UI** — Adam Wathan & Steve Schoger
- **Designing for Performance** — Lara Hogan
- **Google Web Vitals** documentation
- **CSS: The Definitive Guide** — Eric Meyer & Estelle Weyl
- **Introduction to Algorithms (CLRS)** — Cormen, Leiserson, Rivest, Stein (4th ed.)
- **The Algorithm Design Manual** — Steven Skiena (3rd ed.)
- **Algorithms** — Robert Sedgewick & Kevin Wayne (4th ed.)
- **Programming Pearls** — Jon Bentley
- **Grokking Algorithms** — Aditya Bhargava
- **The Art of Computer Programming** — Donald Knuth

## Structure

```
engineering-mastery-plugin/
├── .claude-plugin/
│   └── plugin.json
├── README.md
└── skills/
    ├── clean-code/
    │   └── SKILL.md
    ├── clean-architecture/
    │   └── SKILL.md
    ├── defensive-programming/
    │   ├── SKILL.md
    │   └── references/
    │       ├── cpp.md
    │       ├── python.md
    │       ├── rust.md
    │       └── java.md
    ├── testing/
    │   ├── SKILL.md
    │   └── references/
    │       ├── cpp.md
    │       ├── python.md
    │       ├── rust.md
    │       └── java.md
    ├── performance/
    │   ├── SKILL.md
    │   └── references/
    │       ├── cpp.md
    │       ├── python.md
    │       ├── rust.md
    │       └── java.md
    ├── code-review/
    │   └── SKILL.md
    ├── security/
    │   ├── SKILL.md
    │   └── references/
    │       ├── cpp.md
    │       ├── python.md
    │       ├── rust.md
    │       └── java.md
    ├── observability/
    │   ├── SKILL.md
    │   └── references/
    │       ├── cpp.md
    │       ├── python.md
    │       ├── rust.md
    │       └── java.md
    ├── refactoring/
    │   ├── SKILL.md
    │   └── references/
    │       ├── cpp.md
    │       ├── python.md
    │       ├── rust.md
    │       └── java.md
    ├── api-design/
    │   ├── SKILL.md
    │   └── references/
    │       ├── cpp.md
    │       ├── python.md
    │       ├── rust.md
    │       └── java.md
    ├── concurrency/
    │   ├── SKILL.md
    │   └── references/
    │       ├── cpp.md
    │       ├── python.md
    │       ├── rust.md
    │       └── java.md
    ├── documentation/
    │   └── SKILL.md
    ├── ai-native/
    │   └── SKILL.md
    ├── ml-experimentation/
    │   ├── SKILL.md
    │   └── references/
    │       └── python.md
    ├── ab-testing/
    │   ├── SKILL.md
    │   └── references/
    │       └── python.md
    ├── mobile/
    │   ├── SKILL.md
    │   └── references/
    │       ├── ios.md
    │       ├── android.md
    │       ├── react-native.md
    │       └── flutter.md
    ├── frontend/
    │   ├── SKILL.md
    │   └── references/
    │       ├── react.md
    │       ├── vue.md
    │       ├── tailwind.md
    │       └── typescript.md
    └── algorithms/
        ├── SKILL.md
        └── references/
            ├── cpp.md
            ├── python.md
            ├── rust.md
            └── java.md
```
