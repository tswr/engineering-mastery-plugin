# C++ Security Reference

C++ specific vulnerability patterns and mitigations. C++ gives you direct memory access and no runtime safety net — the majority of critical CVEs in systems software stem from memory safety issues. The patterns here prevent the most common vulnerability classes.

---

## Memory Safety

```cpp
// BUFFER OVERFLOW — the classic C/C++ vulnerability

// DANGEROUS: no bounds checking
void copy_input(const char* input) {
    char buffer[256];
    strcpy(buffer, input);  // no length check — overflow if input > 255
}

// SAFE: bounded operations
void copy_input(const char* input) {
    char buffer[256];
    strncpy(buffer, input, sizeof(buffer) - 1);
    buffer[sizeof(buffer) - 1] = '\0';
}

// BEST: use std::string — no manual buffer management
void copy_input(std::string_view input) {
    std::string buffer{input};  // handles allocation and bounds
}


// USE-AFTER-FREE — accessing memory after it's been deallocated

// DANGEROUS: raw pointer with unclear ownership
Widget* create_widget() {
    Widget* w = new Widget();
    return w;  // who deletes this?
}

// SAFE: smart pointers with clear ownership
std::unique_ptr<Widget> create_widget() {
    return std::make_unique<Widget>();  // ownership is explicit and automatic
}


// DOUBLE-FREE — freeing the same memory twice

// DANGEROUS: manual memory management
void process(Widget* w) {
    delete w;
}
void cleanup(Widget* w) {
    delete w;  // double-free if process() already deleted it
}

// SAFE: unique_ptr prevents double-free by construction
void process(std::unique_ptr<Widget> w) {
    // automatically freed when w goes out of scope — once and only once
}


// DANGLING REFERENCES — returning references to local/temporary objects

// DANGEROUS:
const std::string& get_name() {
    std::string local = compute_name();
    return local;  // dangling reference — local is destroyed
}

// SAFE: return by value (move semantics make this efficient)
std::string get_name() {
    std::string local = compute_name();
    return local;  // moved out, no dangling reference
}


// UNINITIALIZED MEMORY

// DANGEROUS: uninitialized variable
int count;        // indeterminate value in C++
if (count > 0) {  // undefined behavior
    process();
}

// SAFE: always initialize
int count = 0;
// Or use {} initialization which zero-initializes:
int values[100]{};  // all zeros
```

## Integer Safety

```cpp
// INTEGER OVERFLOW — undefined behavior in signed, wrapping in unsigned

// DANGEROUS: signed overflow is undefined behavior
int32_t compute_size(int32_t width, int32_t height) {
    return width * height;  // overflow if width * height > INT32_MAX
}

// SAFE: check before the operation
std::optional<int32_t> compute_size(int32_t width, int32_t height) {
    if (width > 0 && height > INT32_MAX / width) {
        return std::nullopt;  // would overflow
    }
    return width * height;
}

// Or use a safe integer library / compiler builtins
bool overflow = false;
int32_t result = __builtin_mul_overflow(width, height, &result);
if (overflow) { /* handle */ }


// SIGNED/UNSIGNED COMPARISON — implicit conversion can wrap

// DANGEROUS:
void process(int index, std::vector<int>& v) {
    if (index < v.size()) {  // if index is -1, converts to huge unsigned value
        v[index] = 0;        // out-of-bounds access
    }
}

// SAFE: check sign first, or use same types
void process(int index, std::vector<int>& v) {
    if (index >= 0 && static_cast<size_t>(index) < v.size()) {
        v[index] = 0;
    }
}


// TRUNCATION — narrowing conversions lose data

// DANGEROUS:
size_t length = get_length();       // could be > INT_MAX
int buf_size = static_cast<int>(length);  // truncated
char* buf = new char[buf_size];     // undersized allocation
read_data(buf, length);             // buffer overflow
```

## Format String Safety

```cpp
// DANGEROUS: user input as format string
void log_input(const char* user_input) {
    printf(user_input);          // if input contains %s, %x, %n → crash or exploit
    syslog(LOG_INFO, user_input); // same problem
}

// SAFE: user input as argument, never as format
void log_input(const char* user_input) {
    printf("%s", user_input);    // user input is data, not format
    syslog(LOG_INFO, "%s", user_input);
}

// BEST: use type-safe formatting (C++20 std::format, or fmtlib)
#include <format>
void log_input(std::string_view user_input) {
    auto msg = std::format("User said: {}", user_input);  // type-safe, no format vuln
}
```

## Command and Path Injection

```cpp
// COMMAND INJECTION

// DANGEROUS: system() passes through the shell
void process_file(const std::string& filename) {
    std::string cmd = "convert " + filename + " output.png";
    system(cmd.c_str());  // filename="; rm -rf /" → catastrophe
}

// SAFE: use exec-family with argument arrays (no shell)
void process_file(const std::string& filename) {
    pid_t pid = fork();
    if (pid == 0) {
        execlp("convert", "convert", filename.c_str(), "output.png", nullptr);
        _exit(1);
    }
    waitpid(pid, nullptr, 0);
}
// Or use a process library that takes argument vectors


// PATH TRAVERSAL

// DANGEROUS: user controls file path
void serve_file(const std::string& user_path) {
    std::ifstream f("/data/uploads/" + user_path);
    // user_path = "../../etc/passwd" → reads arbitrary files
}

// SAFE: canonicalize and verify the resolved path is within the allowed directory
void serve_file(const std::string& user_path) {
    auto base = std::filesystem::canonical("/data/uploads/");
    auto full = std::filesystem::canonical(base / user_path);

    // Verify resolved path is under the base directory
    auto [base_end, full_it] = std::mismatch(
        base.begin(), base.end(), full.begin());
    if (base_end != base.end()) {
        throw std::runtime_error("Path traversal attempt");
    }
    std::ifstream f(full);
}
```

## Cryptography

```cpp
// RANDOM NUMBER GENERATION

// DANGEROUS: predictable RNG for security purposes
#include <cstdlib>
int token = rand();  // predictable, small range, NOT secure

// SAFE: cryptographically secure RNG
#include <random>
std::random_device rd;  // OS-provided entropy source
std::array<uint8_t, 32> token;
std::generate(token.begin(), token.end(), std::ref(rd));

// For high-volume, use a CSPRNG seeded from random_device
// Or use OpenSSL: RAND_bytes(buffer, sizeof(buffer));


// CONSTANT-TIME COMPARISON — prevent timing attacks on secrets

// DANGEROUS: early-exit comparison leaks information via timing
bool check_token(const std::string& provided, const std::string& expected) {
    return provided == expected;  // returns faster if first bytes differ
}

// SAFE: constant-time comparison
bool check_token(const std::string& provided, const std::string& expected) {
    if (provided.size() != expected.size()) return false;
    volatile uint8_t result = 0;
    for (size_t i = 0; i < provided.size(); ++i) {
        result |= provided[i] ^ expected[i];
    }
    return result == 0;
}
// Or use OpenSSL: CRYPTO_memcmp()


// MEMORY SCRUBBING — clear secrets from memory after use

// DANGEROUS: compiler may optimize away the memset
char password[64];
// ... use password ...
memset(password, 0, sizeof(password));  // optimizer may remove this

// SAFE: use explicit_bzero or SecureZeroMemory
explicit_bzero(password, sizeof(password));  // guaranteed not optimized away
// Or: OPENSSL_cleanse(password, sizeof(password));
```

## Compiler and Tooling Defenses

```bash
# Enable all security-relevant compiler flags

# Stack protection
-fstack-protector-strong

# Position-independent code (enables ASLR)
-fPIE -pie

# Non-executable stack
-Wl,-z,noexecstack

# Full RELRO (prevents GOT overwrite)
-Wl,-z,relro,-z,now

# Fortify source (runtime buffer overflow detection)
-D_FORTIFY_SOURCE=2

# Address Sanitizer (development/CI — catches memory errors at runtime)
-fsanitize=address,undefined

# Thread Sanitizer (catches data races)
-fsanitize=thread

# Warnings as errors for security-relevant warnings
-Wall -Wextra -Werror -Wformat-security -Wconversion
```
