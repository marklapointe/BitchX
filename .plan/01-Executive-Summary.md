# BitchX Modernization: Executive Summary

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** None

## Project Overview

**BitchX** is a classic IRC (Internet Relay Chat) client originally written in C in the late 1990s. The codebase shows its age with:
- K&R-style C code
- Manual memory management
- No bounds checking
- Outdated build system (autoconf)
- No TLS 1.3 support

## Goals

1. **Rewrite in Rust** — Memory safety, thread safety, zero-cost abstractions
2. **Fix all security vulnerabilities** — Buffer overflows, format string bugs, injection attacks
3. **Modern cryptography** — TLS 1.3, modern cipher suites
4. **Maintain feature parity** — All existing functionality preserved
5. **FreeBSD-native** — Use FreeBSD APIs where appropriate

## Why Rust?

| Benefit | Impact |
|---------|--------|
| Memory safety | Eliminates 70%+ of security vulnerabilities |
| No null pointer dereferences | Crash prevention |
| Thread safety | Safe concurrent IRC connections |
| Modern build system | Cargo.toml replaces autoconf |
| Excellent networking libs | tokio, async-std for IRC protocol |
| FFI compatibility | Can call existing C libraries if needed |

## Scope

### In Scope
- IRC protocol implementation (RFC 1459, 2812, modern extensions)
- DCC file transfers (chat and resume)
- Multiple server connections
- Tcl scripting interface (modern alternative: embedded scripting)
- Screen/window management
- All existing commands and features

### Out of Scope (for initial rewrite)
- GTK/Windows GUI (use terminal UI initially)
- Legacy module system (replace with Rust plugin system)

## Success Criteria

1. ✅ All IRC commands functional
2. ✅ DCC send/receive works
3. ✅ TLS connections secure
4. ✅ No memory safety violations (proven by compiler)
5. ✅ 100% feature parity with original
6. ✅ Passes existing test scenarios

## References

- Original BitchX source: `/home/mlapointe/git/BitchX/source/`
- IRC RFCs: RFC 1459, 2812, 7194
- Rust async networking: tokio, async-std
