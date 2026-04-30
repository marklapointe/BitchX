# Detailed Code Analysis

## Source File Inventory

### Core Files (Critical Path)

| File | Lines | Purpose | Security Issues | Rust Module |
|------|-------|---------|-----------------|-------------|
| source/parse.c | 55,589 | IRC message parsing | Buffer overflow, format strings | protocol/parser |
| source/server.c | 94,483 | Server connection | strcpy, SSL issues | irc/server, network |
| source/irc.c | 39,152 | IRC state machine | Multiple | irc/mod |
| source/commands.c | 165,974 | User commands | Command injection | commands/irc_commands |
| source/commands2.c | 73,661 | User commands 2 | Same | commands/irc_commands |
| source/dcc.c | 119,756 | DCC transfers | Path traversal | dcc/* |
| source/functions.c | 178,098 | Built-in functions | Input validation | scripting/builtins |

### UI Files

| File | Lines | Purpose | Rust Module |
|------|-------|---------|-------------|
| source/screen.c | 74,343 | Screen management | ui/screen |
| source/window.c | 121,145 | Window management | ui/window |
| source/term.c | 83,813 | Terminal handling | ui/terminal |
| source/input.c | 66,076 | Input handling | ui/input |
| source/output.c | 7,713 | Output rendering | ui/output |
| source/status.c | 40,903 | Status bar | ui/status |

### State Management

| File | Lines | Purpose | Rust Module |
|------|-------|---------|-------------|
| source/channel.c | 44,292 | Channel state | irc/channel |
| source/names.c | 48,610 | Channel names | irc/channel |
| source/user.c | 14,963 | User tracking | irc/user |
| source/userlist.c | 44,043 | User list | irc/user |
| source/banlist.c | 35,462 | Ban list | irc/channel |
| source/notify.c | 25,149 | Notify list | irc/notify |
| source/ignore.c | 28,507 | Ignore list | irc/ignore |

### Network & Protocol

| File | Lines | Purpose | Rust Module |
|------|-------|---------|-------------|
| source/network.c | 19,595 | TCP/SSL | network |
| source/ctcp.c | 45,425 | CTCP protocol | protocol/ctcp |
| source/notice.c | 32,199 | NOTICE handling | protocol |
| source/ircaux.c | 63,931 | IRC utilities | utils |

### Configuration & Storage

| File | Lines | Purpose | Rust Module |
|------|-------|---------|-------------|
| source/vars.c | 64,433 | Variables/Settings | storage/config |
| source/struct.c | 28,327 | Struct definitions | storage |
| source/alias.c | 55,505 | Aliases | commands/alias |
| source/hook.c | 42,892 | Hooks | hooks |
| source/history.c | 8,396 | History | storage/db |

### Scripting

| File | Lines | Purpose | Rust Module |
|------|-------|---------|-------------|
| source/tcl.c | 58,966 | Tcl integration | scripting/engine |
| source/tcl_public.c | 47,311 | Tcl public API | scripting/builtins |
| source/expr.c | 43,343 | Expression eval | scripting |
| source/expr2.c | 40,958 | Expression eval 2 | scripting |

### Other

| File | Lines | Purpose | Rust Module |
|------|-------|---------|-------------|
| source/timer.c | 15,333 | Timer events | core |
| source/queue.c | 8,792 | Message queue | irc |
| source/keys.c | 40,899 | Key bindings | commands/bind |
| source/cset.c | 43,873 | Charset handling | utils |
| source/hash.c | 23,439 | Hash tables | utils |
| source/reg.c | 11,266 | Regex | utils |
| source/glob.c | 19,718 | Glob patterns | utils |

---

## Code Style Issues Found

### 1. K&R Function Style
```c
// OLD STYLE (K&R)
int function(argc, argv)
int argc;
char **argv;
{
    // code
}

// Should be modern style:
int function(int argc, char **argv) {
    // code
}
```

**Affected files:** ~60% of codebase

### 2. Global Variables
The entire codebase relies on ~500 global variables for state.

**Solution:** Wrap in Rust structs with Arc<RwLock<>>

### 3. Manual Memory Management
```c
// Allocate
char *buf = malloc(size);

// Use
strcpy(buf, data);

// Free (often forgotten)
free(buf);
```

**Solution:** Rust's ownership model

### 4. Error Handling
```c
// No error checking
strcpy(buf, input);
function_without_return_value();
```

**Solution:** Rust's Result type

### 5. String Handling
```c
char buffer[1024];
sprintf(buffer, "%s", user_input);
```

**Solution:** Rust's String type

---

## Specific Vulnerability Examples

### Format String in source/irc.c
```c
void
display_it(curr_item, line, stuff)
char *curr_item;
char *line;
char *stuff;
{
    // VULNERABLE: stuff could contain %n or other format specifiers
    sprintf(buffer, stuff);
}
```

### Buffer Overflow in source/server.c
```c
void set_server_password(char *password) {
    // No length check
    strcpy(server_password, password);
}
```

### Path Traversal in source/dcc.c
```c
void
dcc_receive(filename)
char *filename;
{
    // No path validation
    FILE *fp = fopen(filename, "wb");
}
```

### Command Injection in source/exec.c
```c
void
do_exec(cmd)
char *cmd;
{
    // User input directly to shell
    system(cmd);
}
```

---

## Data Structure Analysis

### ServerConnection
```c
// From include/server.h
struct Server {
    int descriptor;
    char *servername;
    char *nickname;
    char *username;
    char *realname;
    char *password;
    int port;
    int server_port;
    // ... 50+ more fields
};
```

**Issues:** Fixed-size arrays, no bounds checking

### Window
```c
// From include/window.h
struct Window {
    int refnum;
    char *name;
    char *server;
    char *channel;
    char *topic;
    char *status;
    char *input_line;
    char *command_chars;
    // ... 100+ more fields
};
```

**Issues:** Similar to Server, extensive global state

---

## Build System Analysis

### Current (autoconf)
```
configure.in → configure
Makefile.in → Makefile
aclocal.m4
```

### Target (Cargo)
```
Cargo.toml
src/*.rs
```

---

## Dependencies

### Current C Dependencies
- ncurses/termcap (terminal)
- OpenSSL (TLS)
- Tcl (scripting)
- GTK (GUI)
- libcurl (optional)

### Target Rust Dependencies
- tokio (async runtime)
- rustls (TLS - safer than OpenSSL)
- crossterm (terminal)
- rhai (scripting)
- rusqlite (database)
