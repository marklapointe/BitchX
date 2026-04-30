# AGENTS START HERE — BitchX IRC Client Rust Rewrite

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** None
>
> **Purpose:** This is the primary entry point for autonomous agents working on the BitchX IRC client modernization project. Read this file **first** before consuming any other documents in the `.plan/` directory.

> **FreeBSD:** The environment in which this work is being done may have elements that state that you are in linux, that would be false. You are running in FreeBSD with sudo/su access for pure FreeBSD environment.

---

## What We're Building

A **modern IRC client** written in Rust, rewritten from the legacy C BitchX codebase with:

- **Memory safety** — Eliminate buffer overflows, use-after-free, and null pointer dereferences
- **Security hardening** — Fix format string bugs, command injection, path traversal, and weak cryptography
- **Modern TLS** — TLS 1.3 support with safe cipher suites via rustls
- **Async networking** — tokio-based async runtime for concurrent server connections
- **Feature parity** — All original BitchX functionality preserved and modernized

The project involves:
1. **Complete Rust rewrite** — Modern C++/Rust architecture replacing legacy C code
2. **Security audit & fixes** — Address all critical vulnerabilities identified in the security audit
3. **Protocol implementation** — Full IRC RFC compliance (RFC 1459, 2812, modern extensions)
4. **DCC file transfers** — Secure file transfer with path traversal prevention
5. **Terminal UI** — crossterm-based interface with full color support

---

## Document Structure

All plan documents are in the `.plan/` directory:

| # | File | What It Covers |
|---|------|----------------|
| `0.0` | [`.plan/00-MODERNIZATION-TOC.md`](.plan/00-MODERNIZATION-TOC.md) | Master table of contents |
| `1.0` | [`.plan/01-Executive-Summary.md`](.plan/01-Executive-Summary.md) | Project overview, goals, rationale |
| `2.0` | [`.plan/02-Security-Audit.md`](.plan/02-Security-Audit.md) | **Read this second** — all vulnerabilities and fixes |
| `3.0` | [`.plan/03-Rust-Architecture.md`](.plan/03-Rust-Architecture.md) | Target Rust architecture, module design |
| `4.0` | [`.plan/04-Migration-Phases.md`](.plan/04-Migration-Phases.md) | **Read this third** — phased migration strategy |
| `5.0` | [`.plan/05-Code-Analysis.md`](.plan/05-Code-Analysis.md) | Source file inventory, code style issues |
| `6.0` | [`.plan/06-IPC-Database-Schema.md`](.plan/06-IPC-Database-Schema.md) | SQLite schema, IPC design |
| `7.0` | [`.plan/07-IRCv3-Integration.md`](.plan/07-IRCv3-Integration.md) | IRCv3 protocol extensions (SASL, message tags, A/V) |

---

## Primary Directives

### 1. Security First
- **Zero memory safety violations** — Rust's ownership model eliminates entire classes of bugs
- **TLS 1.3 only by default** — No SSLv2/SSLv3, no weak ciphers
- **Input validation everywhere** — Validate all IRC messages, filenames, user input
- **Path canonicalization** — Block all path traversal attempts in DCC
- **No secrets in logs** — Passwords automatically redacted from logging
- **Sandboxed scripting** — Rhai engine with restricted permissions

### 2. Feature Parity
- **All IRC commands must work** — Match original BitchX command behavior
- **All DCC types supported** — SEND, RECEIVE, RESUME, CHAT
- **Multi-server support** — Concurrent connections to multiple servers
- **Script compatibility** — Migrate Tcl scripting to secure Rhai engine

### 3. Code Quality
- **Clippy clean** — No compiler warnings, pass all linting
- **Test coverage** — Unit tests for all public functions
- **Fuzz testing** — cargo-fuzz targets for IRC parser
- **Documentation** — All public APIs documented with doc comments

### 4. No Unsafe Code (Except FFI)
- **Minimize unsafe blocks** — Only for terminal control FFI
- **Miri testing** — Verify unsafe code has no undefined behavior
- **No raw pointers** — Use references and smart pointers exclusively

---

## Migration Phases

### Phase 0: Infrastructure (Weeks 1-2)
- Set up Cargo project with proper dependencies
- Configure CI/CD with GitHub Actions
- Set up clippy, cargo-audit, fuzz testing

### Phase 1: Core Protocol (Weeks 3-6)
- IRC message parsing with bounds checking
- TCP/TLS connections with rustls
- Server state machine

### Phase 2: Channel & User Management (Weeks 7-10)
- Channel state tracking
- User tracking and ban lists
- Notify and ignore systems

### Phase 3: DCC File Transfers (Weeks 11-14)
- DCC SEND/RECEIVE with path validation
- DCC RESUME support
- DCC CHAT implementation

### Phase 4: Command System (Weeks 15-18)
- All user commands migrated
- Alias system
- Tab completion

### Phase 5: User Interface (Weeks 19-22)
- Terminal handling with crossterm
- Window management
- Color and theme support

### Phase 6: Scripting Engine (Weeks 23-26)
- Rhai integration
- Built-in functions
- Hook system

### Phase 7: Persistence (Weeks 27-30)
- SQLite database schema
- Configuration migration
- State serialization

### Phase 8: Security Hardening (Ongoing)
- Fuzz testing
- Penetration testing
- Certificate pinning

---

## Quick Reference

### Source File Mapping

| Original C File | Target Rust Module | Priority |
|-----------------|-------------------|----------|
| `source/parse.c` | `src/protocol/parser.rs` | P0 |
| `source/server.c` | `src/irc/server.rs` | P0 |
| `source/dcc.c` | `src/dcc/*.rs` | P0 |
| `source/commands.c` | `src/commands/irc_commands.rs` | P1 |
| `source/network.c` | `src/network/connection.rs` | P1 |
| `source/screen.c` | `src/ui/screen.rs` | P2 |
| `source/tcl.c` | `src/scripting/engine.rs` | P2 |

### Key Dependencies

| Crate | Version | Purpose |
|-------|---------|---------|
| tokio | 1.35 | Async runtime |
| rustls | 0.21 | TLS implementation |
| crossterm | 0.22 | Terminal UI |
| rhai | 1.17 | Scripting engine |
| rusqlite | 0.31 | Database |
| irc-proto | 0.15 | IRC parsing |

### Critical Security Fixes

| Vulnerability | File | Fix |
|--------------|------|-----|
| Buffer overflow | All .c files | Rust String/Vec |
| Format string | commands.c, irc.c | Rust format!() |
| Command injection | exec.c | Input sanitization |
| Path traversal | dcc.c | Path canonicalization |
| Weak TLS | network.c | rustls with TLS 1.3 |

---

## Workflow Summary

### Starting a Task
1. Pull latest: `git pull --rebase`
2. Read the relevant phase in `.plan/04-Migration-Phases.md`
3. Find a task marked `TODO` with no assignee
4. Check dependencies are complete
5. Update task status to `🔄 IN PROGRESS`
6. Commit: `git add .plan && git commit -m "Claim: <task>" && git push`

### Completing a Task
1. Implement the feature following the architecture docs
2. Write unit tests
3. Run clippy and ensure clean build
4. Update task status to `✅ DONE`
5. Commit: `git add -A && git commit -m "Complete: <task>" && git push`

### Handling Issues
1. Document the problem in the task's Notes section
2. Mark as `🟡 BLOCKED` with reason
3. Commit and push so other agents know
4. Request guidance

---

## Reading Order

For a new agent, read the documents in this order:

1. **This file** (`AGENTS_START_HERE.md`) — You are here
2. **[`.plan/01-Executive-Summary.md`](.plan/01-Executive-Summary.md)** — The big picture
3. **[`.plan/02-Security-Audit.md`](.plan/02-Security-Audit.md)** — All vulnerabilities
4. **[`.plan/03-Rust-Architecture.md`](.plan/03-Rust-Architecture.md)** — Target architecture
5. **[`.plan/04-Migration-Phases.md`](.plan/04-Migration-Phases.md)** — Migration strategy
6. **[`.plan/05-Code-Analysis.md`](.plan/05-Code-Analysis.md)** — Source file mapping
7. **[`.plan/07-IRCv3-Integration.md`](.plan/07-IRCv3-Integration.md)** — IRCv3 extensions (SASL, message tags, A/V)

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | Rust | Memory safety, modern tooling, strong type system |
| Async runtime | tokio | Mature, async-ecosystem compatible |
| TLS | rustls | Memory-safe, modern TLS 1.3 support |
| Terminal UI | crossterm | Cross-platform, no runtime dependencies |
| Scripting | Rhai | Safe sandbox, Rust-native, embeddable |
| Database | SQLite | Simple, single-file, well-tested |
| Build system | Cargo | Standard Rust build, dependency management |

---

## Environment Notes

### FreeBSD-Specific
- Use `sudo` or `su` for privileged operations
- Some system calls differ from Linux — check man pages
- ncurses on FreeBSD may need `ncursesw` package

### Build Commands
```bash
# Build the project
cargo build --release

# Run tests
cargo test

# Lint
cargo clippy

# Security audit
cargo audit

# Fuzz testing
cargo fuzz run irc_parser
```

---

## Need Help?

If you encounter issues:
1. Check the relevant plan document for guidance
2. Check task Notes for known issues
3. Mark task as `🟡 BLOCKED` with reason
4. Commit and push so other agents know

> **IMPORTANT — Commit Guidelines:**
> - **NEVER** add `Co-authored-by: Junie <junie@jetbrains.com>` or any similar co-author trailers
> - **ALWAYS** use `git commit --trailer "Co-authored-by: ..."` only if explicitly requested by the user
> - The commit author **must always** be `Mark LaPointe <mark@cloudbsd.org`

> **Remember:** The goal is to build a secure, modern IRC client that eliminates the security vulnerabilities in the legacy C codebase while maintaining full feature parity. Every task should bring us closer to that goal.
