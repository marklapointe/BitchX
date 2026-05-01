# BitchX Plan Review — Gap Analysis & Improvements

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** None
>
> **Purpose:** Comprehensive review of the existing modernization plan, identifying gaps, inconsistencies, and missing features discovered by analyzing the source codebase and comparing against the current plan.

---

## 1. Dependency Version Inconsistencies

### 1.1 Critical: rustls Version Mismatch

| Document | rustls Version | Issue |
|----------|----------------|-------|
| `03-Rust-Architecture.md` | `0.21` | Outdated |
| `08-WebUI-Integration.md` | `0.23` | More recent |

**Resolution:** Use `rustls = "0.23"` consistently. Note that `rustls` v0.23+ uses `aws-lc-rs` by default instead of `ring`.

### 1.2 irc crate Analysis

The `irc = "0.15"` dependency in `03-Rust-Architecture.md` needs verification. The `irc` crate on crates.io is at version 0.15.x but may need to be supplemented with `irc-proto` for lower-level parsing. Consider using:
- `irc-proto` for message parsing (lower level)
- Custom parser using `nom` for full control

**Recommendation:** Use `nom` for custom IRC parsing (as shown in the plan) rather than relying on the `irc` crate, to have full control over security and parsing behavior.

### 1.3 Deprecated Dependencies

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `ring` | `aws-lc-rs` | rustls 0.23+ uses aws-lc-rs by default |
| `termion` | `crossterm` or `ratatui` | Keep only one terminal crate |

---

## 2. Missing Features from Source Code

The following features exist in `source/*.c` but are NOT covered in the current plan:

### 2.1 Flood Protection (`source/flood.c`)

**Current plan status:** NOT MENTIONED

The flood protection system handles:
- JOIN flood detection
- PUBLIC flood detection  
- NICK flood detection
- KICK flood detection
- DEOP flood detection
- Configurable flood rates per channel (via `/CSET`)
- ON FLOOD hook for scripting responses

**Files:** `source/flood.c`, `source/flood.h`

**Required action:** Add to Phase P2 (Channel & User Management) as `P2.5 - Flood Protection`

### 2.2 Encryption (`source/encrypt.c`)

**Current plan status:** Security audit mentions "Legacy encryption (Blowfish/IDEA)" as MEDIUM severity

**Reality:** The encryption system supports:
- Per-nickname encryption keys
- DES, BF (Blowfish), and other algorithms
- CTCP encryption via `/ENCRYPT` command

**Required action:** Document decision: deprecate weak algorithms (DES, IDEA, weak Blowfish), support only AES-256-GCM in modern implementation.

### 2.3 Logging System (`source/log.c`)

**Current plan status:** Partially covered in `06-IPC-Database-Schema.md` (message history table)

**Reality:** The logging system supports:
- File-based logging with rotation
- Per-window logging
- mIRC-style logging format
- Stripping ANSI codes for log files

**Required action:** Add dedicated logging module to `03-Rust-Architecture.md`:
```
bitchx-rs/
└── src/
    └── logging/
        ├── mod.rs
        ├── file.rs          # File-based logging
        ├── rotater.rs       # Log rotation
        └── format.rs        # Log formatting
```

### 2.4 Exec/Script Execution (`source/exec.c`)

**Current plan status:** Security audit identifies as CRITICAL vulnerability

**Reality:** BitchX supports:
- Shell command execution from IRC
- `/EXEC` command for running external programs
- Capturing output to windows
- Pipe commands (`|cmd`)

**Required action:** Document security model:
- **Option A:** Remove exec functionality entirely
- **Option B:** Sandboxed execution with explicit user approval
- **Recommendation:** Option B with configurable policy

### 2.5 Timer System (`source/timer.c`)

**Current plan status:** NOT MENTIONED

**Reality:** Timers are used extensively for:
- Server pings
- Reconnection delays
- Away notifications
- Scripting timers (`/TIMER` command)

**Required action:** Add to `03-Rust-Architecture.md`:
```
bitchx-rs/
└── src/
    └── timer/
        ├── mod.rs
        ├── scheduler.rs     # Timer scheduling
        └── tasks.rs         # Timer task types
```

### 2.6 Desktop Notifications

**Current plan status:** NOT MENTIONED

**Reality:** Modern IRC clients support:
- System notifications for highlights
- Sound alerts for mentions
- Configurable notification rules
- Notification passthrough settings

**Required action:** Add to Phase P5 (User Interface):
```
bitchx-rs/
└── src/
    └── notifications/
        ├── mod.rs
        ├── desktop.rs      # Desktop notifications (libnotify)
        ├── sound.rs        # Sound alerts
        └── rules.rs        # Notification rules
```

### 2.7 Proxy/SOCKS Support

**Current plan status:** NOT MENTIONED

**Reality:** BitchX historically supported:
- HTTP proxy connections
- SOCKS 4/5 proxy support
- Bouncer connection modes

**Required action:** Add to Phase P1 (Core Protocol):
```
bitchx-rs/
└── src/
    └── proxy/
        ├── mod.rs
        ├── http.rs         # HTTP CONNECT proxy
        ├── socks4.rs       # SOCKS4
        └── socks5.rs       # SOCKS5
```

### 2.8 Bot/Automation System (`source/bot_link.c`)

**Current plan status:** NOT MENTIONED

**Reality:** BitchX has bot linking capabilities for:
- Bot-to-bot communication
- Shared channel control
- DCC bridging between servers

**Required action:** Evaluate for inclusion or removal. If included, add to Phase P3 (DCC).

---

## 3. Missing IRCv3 Specifications

### 3.1 Additional IRCv3 Capabilities Not Covered

| Capability | Description | Priority | Status |
|------------|-------------|----------|--------|
| `batch` | Batch multiple related messages | P1 | MISSING |
| `labeled-notify` | Client-side echo of sent messages | P1 | Covered in echo-message |
| `message-id` | Unique message IDs for references | P1 | MISSING |
| `chng` | Message change tracking | P2 | MISSING |
| `invitelist` | Enhanced invite list (v3.2) | P1 | MISSING |
| `banlist` | Enhanced ban list (v3.2) | P1 | MISSING |
| `setname` | Allow users to change realname | P1 | MISSING |
| `userhost-in-names` | Show userhosts in NAMES reply | P2 | MISSING |
| `account-notify` | Account status in WHOX | P1 | Covered in account-tag |
| `cap-notify` | Server advertises capability changes | P3 | MISSING |
| `echo-ts` | Echo TIMESTAMP as sent by server | P3 | MISSING |
| ` server-time` | Already planned | P0 | ✅ Covered |

### 3.2 ZNC/Bouncer Compatibility

**Current plan status:** NOT MENTIONED

**Reality:** Many users connect via bouncers like ZNC. IRCv3 specs for bouncer compatibility:
- Support for `znc.in/server-time-iso` (ZNC-specific extension)
- Support for `znc.in/playback` (ZNC playback module)
- Multi-server state handling (ZNC typically handles one connection per network)

**Required action:** Add to `07-IRCv3-Integration.md`:
- Section 6: Bouncer Compatibility
- ZNC-specific capability support
- Playback handling for log replay

---

## 4. Security Gaps

### 4.1 Rate Limiting

**Current plan status:** NOT MENTIONED

**Required:** Add rate limiting for:
- Outgoing messages (prevent flood from client)
- Connection attempts
- Nick change attempts
- DCC connections

**Implementation approach:**
```rust
pub struct RateLimiter {
    messages_per_second: u32,
    burst_size: u32,
    // ... token bucket implementation
}
```

### 4.2 Anti-Spam Measures

**Current plan status:** NOT MENTIONED

**Required:** 
- Auto-mute for repeated messages
- Word/pattern blocking
- CTCP reply rate limiting
- Automated AKICK suggestions

### 4.3 TLS Certificate Handling

**Current plan status:** Partially covered

**Missing:**
- Custom CA certificate import
- Certificate fingerprint pinning
- Certificate expiration warnings
- OCSP stapling support

### 4.4 Secure Memory Handling

**Current plan status:** NOT MENTIONED

**Required for password storage:**
```rust
use zeroize::Zeroizing;

// Passwords should use Zeroizing<Vec<u8>> to clear from memory
pub struct SecurePassword {
    data: Zeroizing<Vec<u8>>,
}
```

---

## 5. Documentation Corrections

### 5.1 AGENTS_START_HERE.md Corrections

**Line 24:** "Complete Rust rewrite — Modern C++/Rust architecture"
- **Issue:** Mentions C++ which is incorrect
- **Fix:** Change to "Complete Rust rewrite with modern Rust architecture"

### 5.2 Missing Version Information

All plan documents should include:
```markdown
> **Version:** 1.0.0
> **Created:** 2026-05-01
> **Last Updated:** 2026-05-01
```

### 5.3 Consistency Check

| Document | Line 24 Reference | Status |
|----------|-------------------|--------|
| `AGENTS_START_HERE.md` | "Modern C++/Rust" | ❌ Wrong |
| `01-Executive-Summary.md` | "Why Rust?" | ✅ Correct |
| `03-Rust-Architecture.md` | "Rust Architecture" | ✅ Correct |

---

## 6. Missing Platform-Specific Features

### 6.1 FreeBSD-Specific Considerations

**Current plan status:** Superficially mentioned

**Missing:**
- `kqueue` vs `epoll` event handling
- FreeBSD-specific socket options
- Capsicum capability sandboxing
- PF (Packet Filter) integration for logging

### 6.2 macOS Considerations

**Missing:**
- LaunchAgent plist for startup
- Notification Center integration
- Keychain for password storage

### 6.3 Windows Considerations

**Missing:**
- WinFifo (named pipes) for IPC
- Windows-specific DCC implementation
- Native TLS via SChannel (or stick with rustls)

---

## 7. Testing Gaps

### 7.1 Missing Test Types

| Test Type | Status | Implementation |
|-----------|--------|----------------|
| Unit tests | Planned | ✅ In CI/CD |
| Integration tests | Planned | ✅ Mock IRC server |
| Fuzz testing | Planned | ✅ cargo-fuzz |
| Property-based tests | MISSING | Add `proptest` |
| Mutation testing | MISSING | Add `mutagen` |
| Performance benchmarks | MISSING | Add `criterion` |
| Load testing | MISSING | Add IRC load test suite |

### 7.2 IRC Protocol Compliance Testing

**Missing:** Formal RFC compliance test suite for:
- RFC 1459 (base IRC)
- RFC 2812 (client-server protocol)
- RFC 7194 (CTCP)
- IRCv3 specifications

---

## 8. Missing Documentation

### 8.1 Required New Documents

| Document | Purpose | Priority |
|----------|---------|----------|
| `11-Performance-Tuning.md` | Optimization guidelines | P3 |
| `12-Platform-Support.md` | Platform-specific details | P3 |
| `13-Compatibility-Matrix.md` | BitchX script compatibility | P4 |
| `14-Deployment-Guide.md` | Installation and deployment | P4 |

### 8.2 Missing API Documentation

All Rust modules need:
- Public API documentation with `cargo doc`
- Examples in doc comments
- `rustdoc` generated documentation

---

## 9. WebUI Security Enhancements

### 9.1 Missing WebUI Security Features

| Feature | Status | Implementation |
|---------|--------|----------------|
| CSRF protection | MISSING | Anti-CSRF tokens |
| Clickjacking protection | MISSING | X-Frame-Options |
| HSTS support | MISSING | Strict-Transport-Security |
| Subresource integrity | MISSING | For bundled JS/CSS |
| Rate limiting | MISSING | Per-IP rate limits |
| IP allowlisting | MISSING | Configurable IP restrictions |
| Session revocation | MISSING | Admin panel for sessions |
| Audit logging | MISSING | WebUI action logging |

### 9.2 WebUI Two-Factor Authentication

**MISSING:** Consider adding TOTP support for enhanced security.

---

## 10. Action Items Summary

### 10.1 Critical Fixes (Before Implementation)

- [ ] Fix rustls version consistency (use 0.23)
- [ ] Remove C++ references from AGENTS_START_HERE.md
- [ ] Add version information to all documents
- [ ] Document `ring` → `aws-lc-rs` transition

### 10.2 Missing Features to Add

- [ ] Add `P2.5 - Flood Protection` to task list
- [ ] Add logging module to architecture
- [ ] Add timer module to architecture
- [ ] Add notifications module to architecture
- [ ] Add proxy module to architecture
- [ ] Add bouncer compatibility section to IRCv3 document

### 10.3 New Documents to Create

- [ ] `10-Plan-Review-Gaps.md` (this document)
- [ ] `11-Performance-Tuning.md`
- [ ] `12-Platform-Support.md`
- [ ] `13-Compatibility-Matrix.md`
- [ ] `14-Deployment-Guide.md`

### 10.4 Security Enhancements

- [ ] Add rate limiting module
- [ ] Add secure memory handling for passwords
- [ ] Add TLS certificate management
- [ ] Enhance WebUI security (CSRF, HSTS, etc.)

### 10.5 IRCv3 Completions

- [ ] Add `batch` capability
- [ ] Add `message-id` capability
- [ ] Add `invitelist`/`banlist` v3
- [ ] Add ZNC compatibility section

---

## Appendix A: Complete Source File Inventory

### Files Not in Current Plan Analysis

| File | Purpose | Plan Coverage |
|------|---------|---------------|
| `source/X.c` | Unknown/X Window | ❌ MISSING |
| `source/alist.c` | Access list management | ❌ MISSING |
| `source/alloca.c` | Memory allocation wrapper | ⚠️ PARTIAL |
| `source/art.c` | ASCII art | ❌ MISSING |
| `source/bot_link.c` | Bot linking | ❌ MISSING |
| `source/cdcc.c` | CTCP DCC | ❌ MISSING |
| `source/cdns.c` | DNS caching | ❌ MISSING |
| `source/cdrom.c` | CD-ROM detection | ❌ MISSING |
| `source/chelp.c` | Command help | ❌ MISSING |
| `source/compat.c` | Compatibility layer | ⚠️ PARTIAL |
| `source/debug.c` | Debugging utilities | ❌ MISSING |
| `source/files.c` | File operations | ❌ MISSING |
| `source/fset.c` | Flood settings | ❌ MISSING |
| `source/funny.c` | Funny/ easter eggs | ❌ MISSING |
| `source/hebrew.c` | Hebrew text support | ❌ MISSING |
| `source/if.c` | If/then/else scripting | ❌ MISSING |
| `source/ircsig.c` | Signal handling | ❌ MISSING |
| `source/lastlog.c` | Lastlog search | ❌ MISSING |
| `source/list.c` | List management | ❌ MISSING |
| `source/mail.c` | Mail checking | ❌ MISSING |
| `source/modules.c` | Module system | ❌ MISSING |
| `source/newio.c` | New I/O system | ❌ MISSING |
| `source/numbers.c` | Numeric handlers | ⚠️ PARTIAL |
| `source/opendir.c` | Directory operations | ❌ MISSING |
| `source/pmbitchx.c` | Private messages | ❌ MISSING |
| `source/readlog.c` | Log reading | ❌ MISSING |
| `source/scr-bx.c` | Scripting bridge | ❌ MISSING |
| `source/stack.c` | Stack operations | ❌ MISSING |
| `source/translat.c` | Translation | ❌ MISSING |
| `source/who.c` | WHO queries | ❌ MISSING |
| `source/whowas.c` | WHOWAS queries | ❌ MISSING |
| `source/winbitchx.c` | Windows GUI | ❌ MISSING |
| `source/words.c` | Word parsing | ❌ MISSING |
| `source/wserv.c` | Window service | ❌ MISSING |

### Files Partially Covered

| File | Covered In | Gaps |
|------|-----------|------|
| `source/ctcp.c` | Security audit | Full CTCP spec not detailed |
| `source/dcc.c` | Phase P3 | File transfer only, no CHAT |
| `source/encrypt.c` | Security audit | Legacy algorithms not addressed |
| `source/flood.c` | NOT COVERED | N/A |
| `source/history.c` | IPC/Database | Schema only, not history search |
| `source/log.c` | NOT COVERED | N/A |
| `source/network.c` | Phase P1 | TLS details missing |
| `source/notify.c` | Phase P2 | Notification integration missing |
| `source/parse.c` | Phase P1 | Security issues not detailed |
| `source/server.c` | Phase P1 | Reconnection logic missing |
| `source/tcl.c` | Phase P6 | Migration to Rhai not detailed |
| `source/timer.c` | NOT COVERED | N/A |
| `source/vars.c` | IPC/Database | Config system not designed |

---

## Appendix B: Recommended Dependency Versions (2026)

```toml
[dependencies]
# Async runtime
tokio = { version = "1.40", features = ["full"] }

# Networking
tokio-rustls = "0.26"
rustls = { version = "0.23", features = ["quic", "aws-lc-rs"] }
webpki-roots = "0.26"

# IRC parsing
nom = "7.1"
# Consider: irc-proto = "0.7" for base message types

# Terminal UI
crossterm = "0.28"
ratatui = "0.26"

# Database
rusqlite = { version = "0.32", features = ["bundled"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Scripting
rhai = "1.18"

# Error handling
thiserror = "2.0"
anyhow = "1.0"

# Logging
tracing = "0.1"
tracing-subscriber = "0.3"

# Utilities
chrono = { version = "0.4", features = ["serde"] }
dirs = "5.0"
glob = "0.3"
regex = "1.10"
uuid = { version = "1.8", features = ["v4", "serde"] }

# Security
argon2 = "0.5"
aes-gcm = "0.10"
rand = "0.8"
zeroize = "1.8"

# Web framework (for WebUI)
axum = "0.7"
tokio-tungstenite = "0.23"

# Notifications
libnotify = "0.1"  # Linux
# For macOS: cocoa-rs or notify-rust
```

---

## Appendix C: Cross-Reference Updates Required

When fixing documents, ensure these cross-references are updated:

| Document | Update Required |
|----------|-----------------|
| `00-MODERNIZATION-TOC.md` | Add entries for new documents 10-14 |
| `AGENTS_START_HERE.md` | Add new documents to table, reading order |
| `0.2-Modernization-TaskList.md` | Add tasks for missing features |
| `03-Rust-Architecture.md` | Add missing modules, fix versions |
| `07-IRCv3-Integration.md` | Add missing capabilities, ZNC section |
| `08-WebUI-Integration.md` | Add security enhancements |
