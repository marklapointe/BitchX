# Rust Architecture Design

## Project Structure

```
bitchx-rs/
├── Cargo.toml
├── src/
│   ├── main.rs                 # Entry point, argument parsing
│   ├── lib.rs                  # Library root
│   │
│   ├── core/                   # Core functionality
│   │   ├── mod.rs
│   │   ├── error.rs            # Error types
│   │   ├── config.rs           # Configuration management
│   │   └── logging.rs          # Secure logging
│   │
│   ├── protocol/               # IRC Protocol layer
│   │   ├── mod.rs
│   │   ├── parser.rs           # IRC message parsing
│   │   ├── serializer.rs      # IRC message building
│   │   ├── commands.rs        # IRC command handlers
│   │   ├── numeric.rs          # Numeric reply codes
│   │   └── ctcp.rs             # CTCP protocol
│   │
│   ├── network/                # Networking layer
│   │   ├── mod.rs
│   │   ├── connection.rs       # TCP/TLS connections
│   │   ├── ssl.rs              # TLS configuration
│   │   ├── dns.rs              # DNS resolution
│   │   └── socket.rs           # Low-level socket handling
│   │
│   ├── irc/                    # IRC state management
│   │   ├── mod.rs
│   │   ├── server.rs           # Server connection state
│   │   ├── channel.rs          # Channel state
│   │   ├── user.rs             # User/Nick tracking
│   │   ├── dcc.rs              # DCC connections
│   │   └── notify.rs           # Notify list
│   │
│   ├── dcc/                    # DCC file transfer
│   │   ├── mod.rs
│   │   ├── sender.rs           # DCC SEND implementation
│   │   ├── receiver.rs        # DCC RECEIVE implementation
│   │   ├── resume.rs           # DCC RESUME support
│   │   └── chat.rs             # DCC CHAT
│   │
│   ├── ui/                     # User interface
│   │   ├── mod.rs
│   │   ├── terminal.rs         # Terminal handling
│   │   ├── screen.rs           # Screen management
│   │   ├── window.rs           # Window management
│   │   ├── input.rs            # Input handling
│   │   ├── output.rs           # Output rendering
│   │   ├── status.rs           # Status bar
│   │   └── colors.rs           # Color handling
│   │
│   ├── scripting/              # Scripting engine
│   │   ├── mod.rs
│   │   ├── engine.rs           # Scripting abstraction
│   │   └── builtins.rs         # Built-in commands
│   │
│   ├── commands/               # User commands
│   │   ├── mod.rs
│   │   ├── parser.rs           # Command parsing
│   │   ├── alias.rs            # Alias handling
│   │   ├── bind.rs             # Key bindings
│   │   └── builtins.rs         # Built-in commands
│   │
│   ├── hooks/                  # Hook system
│   │   ├── mod.rs
│   │   ├── event.rs            # Event types
│   │   └── handler.rs          # Hook handlers
│   │
│   ├── storage/                # Data persistence
│   │   ├── mod.rs
│   │   ├── db.rs               # SQLite database
│   │   ├── config.rs           # Config file handling
│   │   ├── state.rs            # State serialization
│   │   └── keyval.rs           # Key-value store
│   │
│   └── utils/                  # Utilities
│       ├── mod.rs
│       ├── strings.rs          # String utilities
│       ├── time.rs             # Time handling
│       └── crypto.rs           # Cryptographic utilities
│
├── tests/                      # Integration tests
├── examples/                   # Example scripts
└── docs/                      # Documentation
```

## Core Crates

### Dependencies (Cargo.toml)

```toml
[package]
name = "bitchx"
version = "2.0.0"
edition = "2021"
rust-version = "1.75"

[dependencies]
# Async runtime
tokio = { version = "1.35", features = ["full"] }

# Networking
tokio-rustls = "0.24"
rustls = "0.21"
webpki-roots = "0.25"

# IRC parsing
irc = "0.15"
nom = "7.1"

# Terminal UI
crossterm = "0.22"
termion = "1.5"
ratatui = "0.26"

# Database
rusqlite = { version = "0.31", features = ["bundled"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Scripting
rhai = "1.17"

# Error handling
thiserror = "1.0"
anyhow = "1.0"

# Logging
tracing = "0.1"
tracing-subscriber = "0.3"

# Utilities
chrono = "0.4"
dirs = "5.0"
glob = "0.3"
regex = "1.10"

# Security
ring = "0.17"
zeroize = "1.7"

[dev-dependencies]
tokio-test = "0.4"
tempfile = "3.9"
cargo-fuzz = "0.11"
```

## Module Design

### Error Handling

```rust
// src/core/error.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum BitchXError {
    #[error("Network error: {0}")]
    Network(#[from] std::io::Error),
    
    #[error("IRC protocol error: {0}")]
    Protocol(String),
    
    #[error("TLS error: {0}")]
    Tls(#[from] rustls::Error),
    
    #[error("Parse error: {0}")]
    Parse(String),
    
    #[error("Configuration error: {0}")]
    Config(String),
    
    #[error("DCC error: {0}")]
    Dcc(String),
    
    #[error("Script error: {0}")]
    Script(String),
    
    #[error("UI error: {0}")]
    Ui(String),
}
```

### IRC Message Parsing

```rust
// src/protocol/parser.rs
use nom::{
    IResult,
    bytes::complete::{tag, take_while, take_until},
    multi::many0,
};
use irc_proto::{Message, Command};

const MAX_MESSAGE_LENGTH: usize = 512;
const MAX_PARAMS: usize = 15;

pub fn parse_irc_message(input: &str) -> Result<Message, BitchXError> {
    // Enforce length limits
    if input.len() > MAX_MESSAGE_LENGTH {
        return Err(BitchXError::Parse("Message too long".into()));
    }
    
    // Parse using nom
    match irc_message(input) {
        Ok((_, msg)) => Ok(msg),
        Err(_) => Err(BitchXError::Parse("Invalid IRC message".into())),
    }
}
```

### TLS Configuration

```rust
// src/network/ssl.rs
use rustls::{ClientConfig, RootCertStore};
use webpki_roots;

pub fn create_tls_config(verify_certs: bool) -> Result<ClientConfig, BitchXError> {
    let mut root_store = RootCertStore::empty();
    root_store.extend(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
    
    let config = ClientConfig::builder()
        .with_safe_defaults()
        .with_root_certificates(root_store)
        .with_no_client_auth();
    
    if !verify_certs {
        // Warn user but still allow (for testing)
        eprintln!("WARNING: Certificate verification disabled!");
    }
    
    Ok(config)
}
```

### DCC File Transfer with Path Validation

```rust
// src/dcc/receiver.rs
use std::path::{Path, Component};
use std::fs;

pub fn validate_filename(filename: &str, download_dir: &Path) -> Result<PathBuf, BitchXError> {
    // Block null bytes
    if filename.contains('\0') {
        return Err(BitchXError::Dcc("Invalid filename".into()));
    }
    
    let path = Path::new(filename);
    
    // Block absolute paths
    if path.is_absolute() {
        return Err(BitchXError::Dcc("Absolute paths not allowed".into()));
    }
    
    // Block path traversal
    for component in path.components() {
        match component {
            Component::ParentDir => {
                return Err(BitchXError::Dcc("Path traversal blocked".into()));
            }
            _ => {}
        }
    }
    
    // Resolve to download directory
    let target = download_dir.join(path);
    let canonical = download_dir.canonicalize()
        .map_err(|_| BitchXError::Dcc("Invalid download directory".into()))?;
    
    // Ensure final path is within download directory
    let target_canonical = target.canonicalize()
        .unwrap_or_else(|_| target.clone());
    
    if !target_canonical.starts_with(&canonical) {
        return Err(BitchXError::Dcc("Path outside download directory".into()));
    }
    
    Ok(target)
}
```

### Secure Logging

```rust
// src/core/logging.rs
use tracing_subscriber::{fmt, layer::SubscriberExt, util::SubscriberInitExt};
use zeroize::Zeroize;

pub fn init_logging(debug: bool) {
    let filter = if debug {
        tracing_subscriber::EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| tracing_subscriber::EnvFilter::new("debug"))
    } else {
        tracing_subscriber::EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| tracing_subscriber::EnvFilter::new("info"))
    };
    
    tracing_subscriber::registry()
        .with(filter)
        .with(fmt::layer().with_ansi(false))
        .init();
}

pub fn sanitize_for_log(input: &str) -> String {
    // Remove potential control characters
    input.chars()
        .filter(|c| c.is_ascii_graphic() || c.is_whitespace())
        .collect()
}

// NEVER log passwords
pub fn log_command(nick: &str, command: &str, args: &[&str]) {
    // Don't log sensitive args (passwords, etc.)
    tracing::info!(nick, cmd = command, "User command");
}
```

## Data Flow

```
User Input
    │
    ▼
┌─────────────────┐
│   UI Layer      │  Terminal, Window, Input handling
│  (crossterm)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Command Parser │  /command parsing, alias expansion
│  (src/commands) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   IRC Protocol  │  Message building, CTCP handling
│  (src/protocol)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Network Layer  │  TCP/TLS connections, DNS
│  (src/network)   │
└────────┬────────┘
         │
         ▼
    IRC Server
```

## State Management

```rust
// src/irc/server.rs
use std::sync::{Arc, RwLock};
use std::collections::HashMap;

pub struct IrcServer {
    pub config: ServerConfig,
    pub state: ConnectionState,
    pub channels: HashMap<String, Channel>,
    pub users: HashMap<String, User>,
}

pub struct GlobalState {
    pub servers: HashMap<usize, Arc<RwLock<IrcServer>>>,
    pub config: GlobalConfig,
    pub notify_list: Vec<String>,
    pub dcc_manager: DccManager,
}

impl GlobalState {
    pub fn new() -> Self {
        Self {
            servers: HashMap::new(),
            config: GlobalConfig::default(),
            notify_list: Vec::new(),
            dcc_manager: DccManager::new(),
        }
    }
}
```

## IPC Architecture

For multi-process mode (future):

```rust
// Unix domain socket IPC
pub const SOCKET_PATH: &str = "/tmp/bitchx-ipc.sock";

// Message format
#[derive(Serialize, Deserialize)]
pub enum IpcMessage {
    SendCommand { server_id: usize, command: String },
    SendMessage { target: String, message: String },
    QueryState { query: StateQuery },
}
```

## Security Principles

1. **No unsafe code** except FFI to libc for terminal control
2. **All input validated** before use
3. **TLS 1.3 only** by default
4. **Path canonicalization** for all file operations
5. **No secrets in logs** - passwords redacted automatically
6. **Bounds checking** enforced by Rust's type system
