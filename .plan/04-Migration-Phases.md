# Migration Phases

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** None

## Phase 0: Setup & Infrastructure (Weeks 1-2)

### 0.1 Project Setup
- [ ] Create Rust project with Cargo
- [ ] Configure CI/CD (GitHub Actions)
- [ ] Set up code coverage (cargo-llvm-cov)
- [ ] Configure clippy and cargo-audit
- [ ] Create Fuzz testing harness (cargo-fuzz)

### 0.2 Build System
- [ ] Replace autoconf with Cargo build
- [ ] Create cross-platform build scripts
- [ ] Document build requirements
- [ ] Set up release automation

### 0.3 Testing Infrastructure
- [ ] Set up integration test framework
- [ ] Create mock IRC server for testing
- [ ] Write protocol conformance tests
- [ ] Set up fuzz testing targets

---

## Phase 1: Core Protocol (Weeks 3-6)

### 1.1 IRC Message Parsing
- [ ] Implement RFC 1459 message parser
- [ ] Implement RFC 2812 extensions
- [ ] Add support for modern irc-utils parsing
- [ ] Write parser fuzz tests
- [ ] Validate against 500+ IRC message samples

**Source files mapped:**
- `source/parse.c` → `src/protocol/parser.rs`

### 1.2 IRC Message Building
- [ ] Implement message serializer
- [ ] Handle CTCP quoting/dequoting
- [ ] Support all numeric reply codes
- [ ] Test with RFC test vectors

**Source files mapped:**
- `source/parse.c` (message building) → `src/protocol/serializer.rs`

### 1.3 Connection Management
- [ ] Implement TCP connection handler
- [ ] Implement TLS 1.3 support with rustls
- [ ] Add reconnection logic
- [ ] Implement connection timeouts
- [ ] Handle network errors gracefully

**Source files mapped:**
- `source/network.c` → `src/network/connection.rs`
- `source/server.c` → `src/irc/server.rs`

### 1.4 Server State Machine
- [ ] Implement IRC connection state
- [ ] Handle password authentication
- [ ] Implement nick/USER registration
- [ ] Handle MOTD display
- [ ] Track server-specific settings

**Source files mapped:**
- `source/irc.c` → `src/irc/mod.rs`
- `source/server.c` → `src/irc/server.rs`

---

## Phase 2: Channel & User Management (Weeks 7-10)

### 2.1 Channel State
- [ ] Track channel membership
- [ ] Handle channel modes
- [ ] Manage channel topic
- [ ] Track channel key (password)
- [ ] Handle channel limits

**Source files mapped:**
- `source/names.c` → `src/irc/channel.rs`
- `source/channel.c` → `src/irc/channel.rs`

### 2.2 User Tracking
- [ ] Track user modes (op, voice, etc.)
- [ ] Handle ban/invite/exception lists
- [ ] Track user realname/hostname
- [ ] Handle nick changes
- [ ] Handle user quit handling

**Source files mapped:**
- `source/user.c` → `src/irc/user.rs`
- `source/userlist.c` → `src/irc/user.rs`
- `source/banlist.c` → `src/irc/channel.rs`

### 2.3 Notify System
- [ ] Implement online/offline tracking
- [ ] Handle notify list management
- [ ] Implement notify events
- [ ] Add notify settings

**Source files mapped:**
- `source/notify.c` → `src/irc/notify.rs`

### 2.4 Ignore System
- [ ] Implement ignore list
- [ ] Handle ignore patterns (nick!user@host)
- [ ] Implement temporary ignores
- [ ] Persist ignore list

**Source files mapped:**
- `source/ignore.c` → `src/irc/ignore.rs`

---

## Phase 3: DCC File Transfers (Weeks 11-14)

### 3.1 DCC SEND (Outgoing)
- [ ] Implement DCC offer generation
- [ ] Implement file sending
- [ ] Handle bandwidth throttling
- [ ] Implement send cancellation
- [ ] Support large file handling

**Source files mapped:**
- `source/dcc.c` (SEND) → `src/dcc/sender.rs`

### 3.2 DCC RECEIVE (Incoming)
- [ ] Implement DCC offer parsing
- [ ] Implement file receiving
- [ ] Handle file overwrite prompts
- [ ] Implement receive cancellation
- [ ] Path traversal prevention (SECURITY CRITICAL)

**Source files mapped:**
- `source/dcc.c` (RECEIVE) → `src/dcc/receiver.rs`

### 3.3 DCC RESUME
- [ ] Implement RESUME support
- [ ] Implement ACCEPT handling
- [ ] Handle partial file resumes
- [ ] Validate resume requests

**Source files mapped:**
- `source/dcc.c` (RESUME) → `src/dcc/resume.rs`

### 3.4 DCC CHAT
- [ ] Implement DCC CHAT listener
- [ ] Implement bidirectional chat
- [ ] Handle DCC CHAT input/output
- [ ] Integrate with main UI

**Source files mapped:**
- `source/dcc.c` (CHAT) → `src/dcc/chat.rs`

---

## Phase 4: Command System (Weeks 15-18)

### 4.1 Command Parser
- [ ] Implement `/command` parsing
- [ ] Handle command aliases
- [ ] Support command history
- [ ] Implement tab completion
- [ ] Handle escape sequences

**Source files mapped:**
- `source/input.c` → `src/commands/parser.rs`
- `source/alias.c` → `src/commands/alias.rs`

### 4.2 IRC Commands
- [ ] Implement all JOIN variants
- [ ] Implement PART with message
- [ ] Implement QUIT handling
- [ ] Implement PRIVMSG/NOTICE
- [ ] Implement NICK command
- [ ] Implement WHO/WHOWAS
- [ ] Implement LIST
- [ ] Implement INVITE
- [ ] Implement KICK
- [ ] Implement TOPIC
- [ ] Implement MODE

**Source files mapped:**
- `source/commands.c` → `src/commands/irc_commands.rs`
- `source/commands2.c` → `src/commands/irc_commands.rs`

### 4.3 Utility Commands
- [ ] Implement ME/ACTION
- [ ] Implement AWAY
- [ ] Implement WALLOPS
- [ ] Implement MOTD
- [ ] Implement LUSERS
- [ ] Implement PING/PONG
- [ ] Implement ADMIN
- [ ] Implement INFO
- [ ] Implement VERSION
- [ ] Implement STATS
- [ ] Implement TIME
- [ ] Implement LINKS
- [ ] Implement CONNECT
- [ ] Implement TRACE

**Source files mapped:**
- `source/funny.c` → `src/commands/utility_commands.rs`

### 4.4 DCC Commands
- [ ] Implement /DCC SEND
- [ ] Implement /DCC GET
- [ ] Implement /DCC CHAT
- [ ] Implement /DCC CLOSE
- [ ] Implement /DCC LIST
- [ ] Implement /DCC RESUME

**Source files mapped:**
- `source/dcc.c` → `src/commands/dcc_commands.rs`

---

## Phase 5: User Interface (Weeks 19-22)

### 5.1 Terminal Handling
- [ ] Implement terminal detection
- [ ] Handle terminal resizing
- [ ] Implement raw mode
- [ ] Handle terminal attributes
- [ ] Implement signal handling

**Source files mapped:**
- `source/screen.c` → `src/ui/screen.rs`
- `source/term.c` → `src/ui/terminal.rs`

### 5.2 Window Management
- [ ] Implement multiple windows
- [ ] Handle window creation/destruction
- [ ] Implement window scrolling
- [ ] Handle window navigation
- [ ] Implement window status

**Source files mapped:**
- `source/window.c` → `src/ui/window.rs`

### 5.3 Output Rendering
- [ ] Implement text rendering
- [ ] Handle timestamps
- [ ] Implement nick highlighting
- [ ] Handle scrollback buffer
- [ ] Implement line wrapping

**Source files mapped:**
- `source/output.c` → `src/ui/output.rs`
- `source/status.c` → `src/ui/status.rs`

### 5.4 Input Handling
- [ ] Implement command input
- [ ] Handle input history
- [ ] Implement paste detection
- [ ] Handle escape sequences
- [ ] Implement character input

**Source files mapped:**
- `source/input.c` → `src/ui/input.rs`

### 5.5 Color Handling
- [ ] Implement ANSI color parsing
- [ ] Handle mIRC color codes
- [ ] Implement color themes
- [ ] Handle color strip option
- [ ] Implement reverse video

**Source files mapped:**
- `include/color.h` → `src/ui/colors.rs`

---

## Phase 6: Scripting Engine (Weeks 23-26)

### 6.1 Scripting Infrastructure
- [ ] Integrate Rhai scripting engine
- [ ] Implement built-in functions
- [ ] Handle script loading
- [ ] Implement script events
- [ ] Create security sandbox

**Source files mapped:**
- `source/tcl.c` → `src/scripting/engine.rs`
- `source/tcl_public.c` → `src/scripting/builtins.rs`

### 6.2 Built-in Functions
- [ ] Implement server functions
- [ ] Implement channel functions
- [ ] Implement user functions
- [ ] Implement string functions
- [ ] Implement file functions
- [ ] Implement DCC functions

**Source files mapped:**
- `source/functions.c` → `src/scripting/builtins.rs`

### 6.3 Hook System
- [ ] Implement event hooks
- [ ] Handle hook priorities
- [ ] Implement hook parameters
- [ ] Handle hook errors
- [ ] Document hook types

**Source files mapped:**
- `source/hook.c` → `src/hooks/mod.rs`

---

## Phase 7: Configuration & Persistence (Weeks 27-30)

### 7.1 Configuration System
- [ ] Implement config file parser
- [ ] Handle server configurations
- [ ] Implement channel configurations
- [ ] Handle alias configurations
- [ ] Implement key bindings

**Source files mapped:**
- `source/vars.c` → `src/storage/config.rs`
- `source/struct.c` → `src/storage/config.rs`

### 7.2 Database Schema
- [ ] Design SQLite schema
- [ ] Implement server state table
- [ ] Implement channel state table
- [ ] Implement history table
- [ ] Implement DCC state table

**Source files mapped:**
- `source/history.c` → `src/storage/db.rs`
- `source/lastlog.c` → `src/storage/db.rs`

### 7.3 State Persistence
- [ ] Implement save/restore state
- [ ] Handle reconnection state
- [ ] Implement history persistence
- [ ] Handle config backup
- [ ] Implement migration from old format

**Source files mapped:**
- `source/queue.c` → `src/storage/state.rs`

---

## Phase 8: Security Hardening (Ongoing)

### 8.1 Input Validation
- [ ] Validate all IRC messages
- [ ] Sanitize all output
- [ ] Validate filenames
- [ ] Validate nicknames
- [ ] Validate channel names

### 8.2 Cryptographic Security
- [ ] Remove SSLv2/SSLv3 support
- [ ] Remove weak ciphers
- [ ] Implement certificate pinning
- [ ] Add TLS 1.3 only mode
- [ ] Remove legacy encryption

### 8.3 Memory Safety
- [ ] Audit all unsafe blocks
- [ ] Add Miri testing
- [ ] Run AddressSanitizer on C code
- [ ] Remove dangerous patterns

### 8.4 Penetration Testing
- [ ] Fuzz IRC parser
- [ ] Test DCC path traversal
- [ ] Test command injection
- [ ] Test CTCP injection
- [ ] Test format strings

---

## Phase 9: Feature Parity Testing (Weeks 31-34)

### 9.1 Command Coverage
- [ ] Test all /commands
- [ ] Test all /set options
- [ ] Test all /alias patterns
- [ ] Test all /bind keys

### 9.2 Protocol Coverage
- [ ] Test all numeric replies
- [ ] Test all CTCP commands
- [ ] Test all DCC types
- [ ] Test extended join

### 9.3 Integration Testing
- [ ] Test multi-server connections
- [ ] Test DCC with firewalls
- [ ] Test reconnection handling
- [ ] Test ZNC/bouncer compatibility

---

## Phase 10: Release & Documentation (Weeks 35-38)

### 10.1 Release Preparation
- [ ] Create release checklist
- [ ] Write changelog
- [ ] Update README
- [ ] Create man page
- [ ] Set up version tagging

### 10.2 Documentation
- [ ] Write user guide
- [ ] Write developer guide
- [ ] Document API
- [ ] Write security policy
- [ ] Create migration guide

---

## Timeline Summary

| Phase | Duration | Focus | Deliverable |
|-------|----------|-------|-------------|
| 0 | 2 weeks | Infrastructure | Build system, CI/CD |
| 1 | 4 weeks | Protocol | IRC parsing, connections |
| 2 | 4 weeks | State | Channels, users, notifies |
| 3 | 4 weeks | DCC | File transfers, chat |
| 4 | 4 weeks | Commands | All user commands |
| 5 | 4 weeks | UI | Terminal interface |
| 6 | 4 weeks | Scripting | Rhai integration |
| 7 | 4 weeks | Persistence | Config, database |
| 8 | Ongoing | Security | Hardening, fuzzing |
| 9 | 4 weeks | Testing | Feature parity |
| 10 | 4 weeks | Release | Docs, release |

**Total Estimated: ~38 weeks (9-10 months)**
