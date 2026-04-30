# BitchX Modernization Plan — Table of Contents

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** This project is not currently seeking or accepting sponsorships.
>
> **Project:** BitchX IRC Client Rewrite in Rust
> **Goal:** Modernize a legacy C IRC client to secure, maintainable Rust code
> **Environment:** FreeBSD (Linux environment with sudo/su access)

## Document Index

| # | Document | Description | Status |
|---|----------|-------------|--------|
| 1 | [01-Executive-Summary.md](./01-Executive-Summary.md) | Overview, goals, approach | ✅ DONE |
| 2 | [02-Security-Audit.md](./02-Security-Audit.md) | Known vulnerabilities and fixes | ✅ DONE |
| 3 | [03-Rust-Architecture.md](./03-Rust-Architecture.md) | Target Rust architecture | ✅ DONE |
| 4 | [04-Migration-Phases.md](./04-Migration-Phases.md) | Incremental migration strategy | ✅ DONE |
| 5 | [05-Code-Analysis.md](./05-Code-Analysis.md) | Detailed code structure analysis | ✅ DONE |
| 6 | [06-IPC-Database-Schema.md](./06-IPC-Database-Schema.md) | IPC mechanism and data persistence | ✅ DONE |

---

## Quick Start

### Phase 1: Analysis & Planning
1. Run security audit on C codebase
2. Map all source files to Rust modules
3. Design IPC and data storage layer

### Phase 2: Core Infrastructure
1. Set up Rust project with Cargo
2. Implement network/IRC protocol layer
3. Build safe string/text handling

### Phase 3: Feature Migration
1. Migrate IRC commands (commands.c, commands2.c)
2. Migrate DCC/file transfers (dcc.c)
3. Migrate UI layer (screen.c, window.c)

### Phase 4: Security Hardening
1. Fix buffer overflow vulnerabilities
2. Implement proper input validation
3. Add encryption support (modern TLS)

### Phase 5: Testing & Deployment
1. Write integration tests
2. Fuzz testing for protocol parsing
3. Performance benchmarking
