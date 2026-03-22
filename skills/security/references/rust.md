# Rust Security Reference

Rust's ownership model eliminates most memory safety vulnerabilities at compile time. The security focus shifts to: unsafe code boundaries, dependency supply chain, cryptographic correctness, and application-level vulnerabilities (injection, auth) that no language prevents automatically.

---

## Unsafe Code

```rust
// Rust's safety guarantees only hold outside of unsafe blocks.
// Every unsafe block is a security-critical trust boundary.

// RULES FOR UNSAFE:
// 1. Minimize unsafe surface area — wrap in a safe interface
// 2. Document the safety invariant (why this is actually safe)
// 3. Prefer safe alternatives first
// 4. Review all unsafe blocks in security audits

// BAD: large unsafe block with unclear invariants
unsafe {
    let ptr = data.as_ptr();
    let len = data.len();
    // ... 50 lines of pointer arithmetic ...
}

// GOOD: minimal unsafe, documented invariant, safe wrapper
/// Returns the element at `index` without bounds checking.
///
/// # Safety
/// Caller must ensure `index < self.len()`.
pub unsafe fn get_unchecked(&self, index: usize) -> &T {
    // SAFETY: caller guarantees index is in bounds
    unsafe { &*self.ptr.add(index) }
}

// BETTER: provide a safe interface that enforces the invariant
pub fn get(&self, index: usize) -> Option<&T> {
    if index < self.len() {
        // SAFETY: we just verified index is in bounds
        Some(unsafe { self.get_unchecked(index) })
    } else {
        None
    }
}

// AUDIT TOOL: find all unsafe usage in your codebase
// cargo install cargo-geiger
// cargo geiger
// Reports unsafe usage in your code AND your dependencies
```

## FFI Boundaries

```rust
// FFI is inherently unsafe — C code has no Rust safety guarantees.
// Treat every FFI boundary as an untrusted input boundary.

// BAD: trusting C function output
extern "C" {
    fn get_data() -> *const u8;
    fn get_len() -> usize;
}

let slice = unsafe {
    std::slice::from_raw_parts(get_data(), get_len())  // trusts C completely
};

// GOOD: validate before creating safe types
let slice = unsafe {
    let ptr = get_data();
    let len = get_len();

    if ptr.is_null() {
        return Err(FfiError::NullPointer);
    }
    if len > MAX_REASONABLE_SIZE {
        return Err(FfiError::SuspiciousSize(len));
    }

    std::slice::from_raw_parts(ptr, len)
};

// Copy FFI data into owned Rust types as soon as possible
let owned: Vec<u8> = slice.to_vec();  // now fully under Rust's safety model


// String handling across FFI:
// CStr/CString for null-terminated C strings
use std::ffi::CStr;

let c_str = unsafe { CStr::from_ptr(raw_ptr) };
let rust_str = c_str.to_str().map_err(|_| FfiError::InvalidUtf8)?;
// Now rust_str is a safe &str with validated UTF-8
```

## Cryptography

```rust
// Use established crates — never implement your own

// Recommended crates:
// ring        — fast, audited, from BoringSSL author
// RustCrypto  — pure-Rust implementations (aes, sha2, chacha20poly1305, etc.)
// argon2      — password hashing
// rand        — cryptographically secure random (with OsRng)

// RANDOM NUMBER GENERATION
use rand::rngs::OsRng;
use rand::RngCore;

let mut token = [0u8; 32];
OsRng.fill_bytes(&mut token);  // cryptographically secure

// DANGEROUS: using rand::thread_rng() is fine for most purposes,
// but for key generation, always use OsRng directly.

// PASSWORD HASHING
use argon2::{self, Config, Variant, Version};

fn hash_password(password: &str) -> String {
    let salt = generate_salt();  // random 16+ bytes
    let config = Config {
        variant: Variant::Argon2id,
        version: Version::Version13,
        mem_cost: 65536,    // 64 MB
        time_cost: 3,
        lanes: 4,
        ..Config::default()
    };
    argon2::hash_encoded(password.as_bytes(), &salt, &config).unwrap()
}

fn verify_password(password: &str, hash: &str) -> bool {
    argon2::verify_encoded(hash, password.as_bytes()).unwrap_or(false)
}


// CONSTANT-TIME COMPARISON
use ring::constant_time;

fn verify_token(provided: &[u8], expected: &[u8]) -> bool {
    constant_time::verify_slices_are_equal(provided, expected).is_ok()
}
// Or with subtle crate:
use subtle::ConstantTimeEq;
provided.ct_eq(expected).into()


// ZEROING SECRETS FROM MEMORY
use zeroize::Zeroize;

let mut secret_key = load_key();
// ... use secret_key ...
secret_key.zeroize();  // guaranteed to zero memory, not optimized away

// Or use the Zeroizing wrapper for automatic cleanup:
use zeroize::Zeroizing;
let secret = Zeroizing::new(load_key());
// automatically zeroed when dropped
```

## Input Validation and Injection

```rust
// SQL INJECTION — use parameterized queries

// DANGEROUS (with raw SQL crates):
let query = format!("SELECT * FROM users WHERE name = '{}'", user_input);
conn.execute(&query)?;

// SAFE: parameterized (sqlx)
let user = sqlx::query_as!(User, "SELECT * FROM users WHERE name = $1", user_input)
    .fetch_one(&pool)
    .await?;

// SAFE: ORM (Diesel) — parameterized by default
users.filter(name.eq(&user_input)).first::<User>(&conn)?;


// COMMAND INJECTION — use std::process::Command (no shell)
use std::process::Command;

// DANGEROUS: shell invocation
Command::new("sh")
    .arg("-c")
    .arg(format!("convert {} output.png", filename))  // shell interprets filename
    .output()?;

// SAFE: direct execution with argument list
Command::new("convert")
    .arg(&filename)         // single argument, no shell interpretation
    .arg("output.png")
    .output()?;


// PATH TRAVERSAL
use std::path::{Path, PathBuf};

fn serve_file(base: &Path, user_path: &str) -> Result<Vec<u8>, Error> {
    let full = base.join(user_path).canonicalize()?;

    if !full.starts_with(base.canonicalize()?) {
        return Err(Error::PathTraversal);
    }

    std::fs::read(full).map_err(Into::into)
}
```

## Web Security (Actix-web / Axum)

```rust
// SSRF: validate user-provided URLs
fn validate_url(user_url: &str) -> Result<Url, SecurityError> {
    let parsed = Url::parse(user_url)?;

    // Scheme allowlist
    if parsed.scheme() != "https" {
        return Err(SecurityError::InsecureScheme);
    }

    // Host allowlist
    let allowed_hosts = ["api.example.com", "cdn.example.com"];
    let host = parsed.host_str().ok_or(SecurityError::NoHost)?;
    if !allowed_hosts.contains(&host) {
        return Err(SecurityError::DisallowedHost);
    }

    // Block private IP ranges (resolve hostname and check)
    let addrs: Vec<_> = (host, 443).to_socket_addrs()?.collect();
    for addr in &addrs {
        if addr.ip().is_loopback() || addr.ip().is_private() {
            return Err(SecurityError::PrivateAddress);
        }
    }

    Ok(parsed)
}


// Set security headers (tower middleware for Axum)
async fn security_headers(
    request: Request,
    next: Next,
) -> Response {
    let mut response = next.run(request).await;
    let headers = response.headers_mut();
    headers.insert("X-Content-Type-Options", "nosniff".parse().unwrap());
    headers.insert("X-Frame-Options", "DENY".parse().unwrap());
    headers.insert("Strict-Transport-Security",
        "max-age=31536000; includeSubDomains".parse().unwrap());
    response
}
```

## Dependency Auditing

```bash
# cargo-audit: check for known vulnerabilities
cargo install cargo-audit
cargo audit

# In CI:
cargo audit --deny warnings

# cargo-deny: comprehensive dependency policy
# Checks licenses, bans specific crates, detects duplicates, advisories
cargo install cargo-deny
cargo deny check

# cargo-geiger: find unsafe usage in dependency tree
cargo install cargo-geiger
cargo geiger
# Reports: functions, expressions, impls, traits with unsafe in every dependency

# Cargo.lock: always commit it for applications (ensures reproducible builds)
# cargo update only when intentional — review the diff

# Minimal dependency versions: test with -Z minimal-versions (nightly)
# Ensures your declared version bounds actually work at minimum
```
