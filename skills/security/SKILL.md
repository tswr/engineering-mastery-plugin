---
name: security
description: "Apply when code handles untrusted input, authentication, authorization, cryptography, secrets, or crosses a trust boundary. Trigger on mentions of security review, threat modeling, vulnerabilities, injection, XSS, CSRF, SSRF, auth, secrets management, or hardening. Extends the defensive-programming skill's boundary validation with adversary-aware techniques. Language-specific vulnerability patterns are in references/."
---

# Security Skill

Write code that is resistant to attack, not just resistant to bugs. Security defects are the most expensive to ship — they cause data breaches, compliance violations, and loss of user trust. Unlike functional bugs, security bugs are actively sought by adversaries.

The core mental model: **the attacker controls the input.** Every piece of data that crosses a trust boundary — user input, API responses, deserialized payloads, environment variables, file contents, HTTP headers — is potentially adversarial. Code must be correct even when inputs are deliberately crafted to break it.

**Language-specific vulnerability patterns and safe alternatives are in the `references/` directory.** Read the relevant file when generating or reviewing code:
- `references/cpp.md` — memory safety, integer overflow, format strings, buffer handling
- `references/python.md` — injection patterns, pickle risks, subprocess safety, path traversal
- `references/rust.md` — unsafe blocks, FFI boundaries, dependency auditing, crypto
- `references/java.md` — deserialization, JNDI, SQL injection, XML external entities

---

## 1. Threat Modeling

Before writing security-sensitive code, think about who would attack it and how. Threat modeling is not paranoia — it's systematic identification of what can go wrong.

**STRIDE** — six categories of threats:

**Spoofing.** Can an attacker pretend to be someone else? Affects authentication. Mitigations: strong authentication, mutual TLS, signed tokens.

**Tampering.** Can an attacker modify data in transit or at rest? Affects integrity. Mitigations: digital signatures, MACs, checksums, immutable audit logs.

**Repudiation.** Can an attacker deny having performed an action? Affects audit trails. Mitigations: secure logging, signed transactions, non-repudiable records.

**Information Disclosure.** Can an attacker access data they shouldn't? Affects confidentiality. Mitigations: encryption at rest and in transit, access controls, data classification, minimizing data collection.

**Denial of Service.** Can an attacker make the system unavailable? Affects availability. Mitigations: rate limiting, resource quotas, input size limits, graceful degradation.

**Elevation of Privilege.** Can an attacker gain permissions they shouldn't have? Affects authorization. Mitigations: least privilege, role-based access, defense in depth.

**When to threat model:** before designing any system that handles sensitive data, authenticates users, processes payments, or is exposed to untrusted networks. The five-minute version: draw the data flow diagram, mark the trust boundaries, and ask "what could go wrong?" at each boundary crossing using STRIDE.

---

## 2. Trust Boundaries and Input Validation

A trust boundary is any point where data passes between components with different privilege levels or trust assumptions. Every trust boundary is an attack surface.

**Common trust boundaries:**
- Between user and server (HTTP requests, form data, file uploads)
- Between services (API calls, message queues, RPC)
- Between your code and a database (query parameters)
- Between your code and the OS (system calls, file paths, environment)
- Between your code and third-party libraries (deserialized data, callback inputs)

**Validate at the boundary, reject by default.** Accept only what you expect. Define an allowlist of valid inputs (format, length, character set, range), and reject everything that doesn't match. An allowlist is always safer than a denylist — denylists miss novel attack patterns.

**Validate, then convert to internal types.** Raw input should be parsed into typed, validated domain objects at the boundary (see defensive-programming skill). From that point inward, code operates on trusted types. Never pass raw user input deep into the system.

**Canonicalize before validation.** Attackers use encoding tricks (URL encoding, Unicode normalization, double encoding, path traversal sequences) to bypass validation. Convert input to its canonical form before checking it. Validate the canonical form, not the raw bytes.

**Limit input size.** Unbounded input is a denial-of-service vector. Set maximum lengths for strings, maximum sizes for uploads, maximum depths for nested structures, and maximum counts for collection elements. Enforce these limits at the earliest possible point.

This section extends the defensive-programming skill's boundary validation with adversary-aware techniques: allowlist-only validation, canonicalization, size limits, and injection prevention.

**Race conditions (TOCTOU).** Time-of-check-to-time-of-use vulnerabilities occur when a security check and the action it guards are not atomic. Example: checking file permissions then opening the file — an attacker can swap the file between check and use. Mitigations: use atomic operations where possible, hold locks during check-and-use sequences, use file descriptors instead of paths after opening.

---

## 3. Injection

Injection occurs when untrusted data is interpreted as code or commands. It is the most consistently dangerous vulnerability class.

**The rule: never construct executable statements by concatenating user input.** This applies to SQL, shell commands, LDAP queries, XML, HTML, log formats, and any other language or format where data and instructions share the same channel.

**SQL Injection.** Use parameterized queries or prepared statements. Always. No exceptions. ORMs that generate parameterized queries are safe by default; raw query builders that concatenate strings are not.

```
DANGEROUS:  query("SELECT * FROM users WHERE id = " + userId)
SAFE:       query("SELECT * FROM users WHERE id = ?", userId)
```

**Command Injection.** Never pass user input to a shell. Use direct process execution with argument arrays — not shell interpolation. If you must invoke an external program, pass arguments as a list, not as a single string that gets shell-parsed.

```
DANGEROUS:  exec("convert " + filename + " output.png")
SAFE:       exec(["convert", filename, "output.png"])
```

**Log Injection.** If user input appears in log output, an attacker can inject fake log entries or terminal escape sequences. Sanitize or encode user data before logging. Never log raw user input in a format that could be parsed as structured data.

**Template Injection.** If user input is embedded in a template engine (HTML, email, config files), ensure it's treated as data, not template code. Use the template engine's built-in escaping mechanisms.

---

## 4. Authentication and Session Management

**Never implement your own cryptographic authentication.** Use established libraries and protocols (bcrypt/scrypt/argon2 for passwords, OAuth2/OIDC for delegation, FIDO2/WebAuthn for passwordless). Custom crypto is almost always broken.

**Password storage.** Hash with a slow, salted algorithm (argon2id preferred, bcrypt acceptable). Never store plaintext, MD5, SHA-1, or unsalted SHA-256. The hash algorithm must be deliberately slow to resist offline brute-force attacks.

**Session tokens.** Generate with a cryptographically secure random number generator. Make them long enough to resist brute-force (128+ bits of entropy). Set appropriate expiration. Invalidate on logout. Bind to the client where possible (device, IP range).

**Multi-factor.** For sensitive operations (admin access, financial transactions, credential changes), require a second factor. TOTP or hardware keys over SMS.

**Credential handling in code.** Never hardcode secrets, API keys, or passwords in source code. Use environment variables, secret management services (Vault, KMS, cloud provider secret stores), or credential files with restricted permissions. Never log credentials, even at debug level.

---

## 5. JWT Security

JSON Web Tokens are widely used for stateless authentication but have specific pitfalls that differ from session-based auth.

**Algorithm confusion.** JWTs specify their signing algorithm in the header. If the server doesn't enforce the expected algorithm, an attacker can switch from RS256 (asymmetric) to HS256 (symmetric) and sign the token with the public key — which the server already trusts. Always validate the algorithm server-side; never trust the token's `alg` header.

**The `none` algorithm.** Some libraries accept `alg: "none"`, which means the token is unsigned. An attacker removes the signature and sets `alg: none`. Explicitly reject unsigned tokens. Whitelist the algorithms your server accepts.

**Key management.** Signing keys in source code are compromised keys. Rotate keys periodically. Support multiple active keys (key ID in header) so rotation doesn't invalidate all existing tokens instantly.

**Token scope and lifetime.** Keep access tokens short-lived (minutes, not hours). Use refresh tokens for re-authentication. Don't store sensitive data in the payload — JWTs are base64-encoded, not encrypted. If you need encrypted claims, use JWE.

---

## 6. Authorization

Authentication tells you *who* the user is. Authorization tells you *what they can do*. These are separate concerns and must be implemented independently.

**Default deny.** No access unless explicitly granted. If the authorization check is missing, the answer is "no." This is the opposite of "default allow, check if restricted" — which means any forgotten check is an open door.

**Check at every entry point.** Authorization must be enforced on every API endpoint, every data access, every state transition. Client-side checks (hiding buttons, removing menu items) are for UX, not security — the server must enforce independently of what the client shows.

**Don't trust client-provided identity for authorization.** The user's role, permissions, and identity come from the server-side session, not from the request payload. An attacker can modify any client-side value.

**Object-level authorization.** It's not enough to check "can this user access orders" — you must check "can this user access *this specific order*?" IDOR (Insecure Direct Object Reference) is among the most common authorization flaws: the user changes an ID in the URL and accesses someone else's data.

**Least privilege.** Grant the minimum permissions needed for the task. Service accounts, API keys, database connections, and IAM roles should all have the narrowest scope that allows them to function. Broad permissions are a liability that compounds with every privilege escalation vulnerability.

---

## 7. Cryptography

**Never roll your own crypto.** Use well-established libraries maintained by cryptography experts. Don't implement encryption algorithms, key derivation functions, or signature schemes yourself. Don't invent a "custom encoding" that you believe provides security.

**Use current algorithms.** AES-256-GCM for symmetric encryption, X25519 or ECDH for key exchange, Ed25519 or ECDSA for signatures, SHA-256 or SHA-3 for hashing, Argon2id for password hashing. Avoid: DES, 3DES, RC4, MD5, SHA-1 for any security purpose.

**Encryption is not integrity.** AES-CBC encrypts but doesn't prevent tampering. Use authenticated encryption (AES-GCM, ChaCha20-Poly1305) that provides both confidentiality and integrity. If you're using encryption without authentication, an attacker can modify ciphertext without detection.

**Key management.** Keys in source code are compromised keys. Store keys in a secrets manager or HSM. Rotate keys on a schedule. Use different keys for different purposes (encryption vs. signing vs. derivation). Never log keys.

**Random number generation.** For any security-relevant randomness (tokens, keys, nonces, IVs), use a cryptographically secure PRNG. Standard library math.random / rand() is predictable and must never be used for security.

---

## 8. Data Exposure

**Minimize what you collect and store.** Data you don't have can't be breached. Don't store sensitive data "in case we need it later." Apply data retention policies and delete what's no longer needed.

**Encrypt sensitive data at rest.** Database fields containing PII, financial data, health records, and credentials should be encrypted. Use column-level or application-level encryption when database-level encryption isn't sufficient.

**Encrypt in transit.** All network communication should use TLS 1.2+. No exceptions for "internal" services — network segmentation fails, lateral movement is real.

**Don't expose sensitive data in errors.** Stack traces, database error messages, internal paths, and configuration details should never appear in responses to untrusted clients. Log the details server-side; return a generic error to the client.

**Scrub logs.** Audit your logging to ensure passwords, tokens, credit card numbers, SSNs, and other sensitive data are never written to log files. Use structured logging with explicit field selection rather than logging entire request/response objects.

---

## 9. Dependency and Supply Chain Security

**Every dependency is attack surface.** A compromised or vulnerable dependency gives an attacker code execution in your process. The 2024 xz/liblzma backdoor demonstrated this at the infrastructure level.

**Pin dependency versions.** Use lock files (package-lock.json, Cargo.lock, requirements.txt with hashes, gradle.lockfile). Don't allow automatic minor/patch upgrades without review.

**Audit dependencies.** Use automated tools (npm audit, cargo audit, pip-audit, OWASP Dependency-Check) in CI to catch known vulnerabilities. Review new dependencies before adding them: who maintains it, how many maintainers, how actively maintained, what's the download count, is the scope appropriate (does a color formatting library need network access?).

**Minimize dependency count.** Each dependency is a trust decision. Don't add a dependency for something you can write in 20 lines. The marginal convenience rarely justifies the marginal risk.

---

## 10. Secure Defaults

**Fail closed.** If a security check errors out, deny access. Don't fail open — an attacker who can trigger errors in your authorization check bypasses it entirely.

**Secure by default configuration.** The default state of a new deployment should be secure: authentication enabled, debug mode off, verbose errors off, CORS restricted, default passwords absent. Security that requires explicit opt-in is security that gets forgotten.

**Defense in depth.** Don't rely on a single control. Input validation at the boundary AND parameterized queries AND least-privilege database permissions together mean that a failure in any one layer doesn't result in compromise.

**Principle of least surprise.** Security mechanisms should work as users and developers expect. A permission named "read" that also grants write access is a design bug, not a feature.

**CORS (Cross-Origin Resource Sharing).** Misconfigured CORS is a common vulnerability. Never reflect the `Origin` header as `Access-Control-Allow-Origin` without validation — this effectively disables the same-origin policy. Whitelist specific trusted origins. Never use `Access-Control-Allow-Credentials: true` with a wildcard origin. Restrict allowed methods and headers to what the API actually needs.

---

## Applying This Skill

When writing code:
1. Identify trust boundaries — where does untrusted data enter?
2. Validate and sanitize all input at the boundary
3. Use parameterized queries, not string concatenation
4. Use established crypto libraries, not custom implementations
5. Store secrets in secret managers, not code
6. Apply least privilege to all credentials and service accounts
7. Log security-relevant events, never log secrets

When reviewing code for security:
1. Find the trust boundaries in the change
2. Trace untrusted input through the code — where does it end up?
3. Check for injection vectors: SQL, command, log, template
4. Check authorization: is it enforced at every entry point, including object-level?
5. Check secrets: are credentials hardcoded, logged, or over-exposed?
6. Check dependencies: are new dependencies justified and audited?

When designing systems:
1. Threat model with STRIDE before implementation
2. Draw the data flow, mark trust boundaries
3. Apply defense in depth — no single point of failure for security
4. Default deny for all access control
5. Encrypt in transit and at rest for sensitive data

When the target language is known, read the corresponding file in `references/` for language-specific vulnerability patterns and safe alternatives.
