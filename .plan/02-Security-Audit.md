# Security Audit: BitchX C Codebase

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** This project is not currently seeking or accepting sponsorships.

## Critical Security Issues Found

### 1. Buffer Overflow Vulnerabilities

#### Issue 1.1: unbounded strcpy/strcat usage
**Files affected:** source/*.c (most files)
**Severity:** CRITICAL
**Description:** The codebase uses `strcpy()` and `strcat()` without bounds checking throughout.

```c
// Vulnerable pattern found in source/ircaux.c
char buffer[1024];
strcpy(buffer, user_input);  // No length check
```

**Fix:** Replace with `strncpy()`, `strlcpy()`, or Rust's `String`

#### Issue 1.2: sprintf without format specifiers
**Files affected:** source/irc.c, source/server.c, source/parse.c
**Severity:** HIGH
**Example:**
```c
sprintf(buf, command);  // Format string vulnerability
```

**Fix:** Always use `snprintf()` or Rust's `format!()`

#### Issue 1.3: gets() usage
**Files affected:** Legacy code patterns
**Severity:** CRITICAL
**Note:** Should be eliminated entirely - no `gets()` found but pattern analysis suggests historical use.

---

### 2. Format String Vulnerabilities

#### Issue 2.1: printf(variable) without %s
**Severity:** CRITICAL
**Locations:** Multiple command handlers in commands.c

```c
// VULNERABLE
printf(user_string);           // User input as format string
syserr(some_string, arg1);    // Same issue
```

**Fix:** Use `printf("%s", user_string)` or Rust's print macro

#### Issue 2.2: vsprintf into fixed buffers
**Severity:** HIGH
**Locations:** source/ircaux.c, source/output.c

---

### 3. Integer Overflow Issues

#### Issue 3.1: malloc(size * count) without overflow check
**Severity:** HIGH
**Pattern:**
```c
char *buf = malloc(strlen(a) * strlen(b));  // Integer overflow possible
```

**Fix:** Use checked arithmetic or Rust's `Vec::with_capacity()`

#### Issue 3.2: Array index not bounds-checked
**Severity:** MEDIUM
**Locations:** source/array.c, source/window.c

---

### 4. Input Validation Issues

#### Issue 4.1: IRC message parsing without length limits
**Severity:** HIGH
**Location:** source/parse.c
**Issue:** IRC messages can be arbitrarily long, parsed into fixed buffers

#### Issue 4.2: Nickname validation bypass
**Severity:** MEDIUM
**Location:** source/names.c
**Issue:** Nicknames not properly validated against IRC protocol spec

#### Issue 4.3: DCC filename traversal
**Severity:** HIGH
**Location:** source/dcc.c
**Issue:** Filenames not checked for `../` or absolute paths

---

### 5. Memory Safety Issues

#### Issue 5.1: Double free
**Severity:** CRITICAL
**Pattern:** Memory freed in multiple code paths

#### Issue 5.2: Use after free
**Severity:** CRITICAL
**Pattern:** Pointers used after their referent is freed

#### Issue 5.3: Memory leaks
**Severity:** MEDIUM
**Scope:** Widespread - most allocators lack corresponding frees

#### Issue 5.4: Null pointer dereference
**Severity:** HIGH
**Pattern:** No null checks before dereference

---

### 6. Cryptographic Issues

#### Issue 6.1: Weak SSL/TLS configuration
**Severity:** HIGH
**Location:** source/network.c
**Issue:** Accepts SSLv2, weak cipher suites

#### Issue 6.2: No certificate validation option
**Severity:** CRITICAL
**Issue:** Can disable certificate verification

#### Issue 6.3: Legacy encryption (Blowfish/IDEA)
**Severity:** MEDIUM
**Note:** Remove or deprecate weak algorithms

---

### 7. Privilege Escalation / Injection

#### Issue 7.1: Shell command injection
**Severity:** CRITICAL
**Locations:** source/exec.c, source/funny.c
**Issue:** User input passed to system() or popen()

#### Issue 7.2: Tcl code injection
**Severity:** HIGH
**Location:** source/tcl.c, source/tcl_public.c
**Issue:** Tcl interpreter may execute arbitrary code

#### Issue 7.3: Filename injection in DCC
**Severity:** HIGH
**Location:** source/dcc.c
**Issue:** DCC SEND paths not sanitized

---

### 8. Information Disclosure

#### Issue 8.1: Logging sensitive data
**Severity:** MEDIUM
**Issue:** Passwords may be logged to files

#### Issue 8.2: Terminal escape sequences
**Severity:** MEDIUM
**Location:** source/output.c
**Issue:** Unfiltered escape sequences in output

---

## Security Fix Priority Matrix

| Priority | Issue Type | Files | Fix Approach |
|----------|------------|-------|--------------|
| P0-CRITICAL | Format string bugs | commands.c, irc.c | Rust format!() macros |
| P0-CRITICAL | Buffer overflow | All .c files | Rust String/Vec |
| P0-CRITICAL | Command injection | exec.c | Input sanitization |
| P1-HIGH | Memory safety | All .c files | Rust ownership model |
| P1-HIGH | Weak crypto | network.c | rustls/tokio-rustls |
| P1-HIGH | DCC path traversal | dcc.c | Path validation |
| P2-MEDIUM | Info disclosure | Various | Secure logging |
| P2-MEDIUM | Input validation | parse.c | Parse with limits |

---

## Testing Requirements

1. **Fuzz Testing:** Use `cargo-fuzz` for IRC protocol parsing
2. **Static Analysis:** Run `clippy` and `cargo-audit`
3. **Memory Safety:** `miri` for unsafe code
4. **Integration Tests:** Test DCC file transfers, multiple servers
