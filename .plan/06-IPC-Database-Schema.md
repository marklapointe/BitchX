# IPC & Database Schema Design

## IPC Architecture

### Unix Domain Socket Design

```
/tmp/bitchx-XXXX.sock  (per-instance)
```

### Message Protocol

```rust
// Message envelope
#[derive(Serialize, Deserialize)]
pub struct IpcMessage {
    pub version: u8,
    pub msg_type: MessageType,
    pub payload: Vec<u8>,
}

#[derive(Serialize, Deserialize)]
pub enum MessageType {
    // Client → Server
    ExecuteCommand,
    SendMessage,
    QueryState,
    SubscribeEvents,
    
    // Server → Client
    CommandResult,
    StateUpdate,
    Event,
    Error,
}
```

### IPC Use Cases

1. **CLI control** - Control running instance via command line
2. **Script control** - Scripts communicate with IRC client
3. **Multi-instance** - Manage multiple connections
4. **Remote control** - Control from external applications

---

## SQLite Database Schema

### Database Location
```
~/.config/bitchx/bitchx.db
```

### Schema Version
```sql
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY,
    applied_at TEXT NOT NULL
);
```

### Servers Table
```sql
CREATE TABLE servers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    host TEXT NOT NULL,
    port INTEGER NOT NULL DEFAULT 6667,
    ssl BOOLEAN NOT NULL DEFAULT FALSE,
    ssl_verify BOOLEAN NOT NULL DEFAULT TRUE,
    password TEXT,  -- encrypted
    username TEXT,
    realname TEXT,
    nick TEXT,
    auto_connect BOOLEAN DEFAULT FALSE,
    auto_join TEXT,  -- comma-separated channels
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Channels Table
```sql
CREATE TABLE channels (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    server_id INTEGER NOT NULL REFERENCES servers(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    key TEXT,  -- encrypted
    auto_join BOOLEAN DEFAULT FALSE,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(server_id, name)
);
```

### Channel State Table
```sql
CREATE TABLE channel_state (
    channel_id INTEGER PRIMARY KEY REFERENCES channels(id) ON DELETE CASCADE,
    topic TEXT,
    mode TEXT,
    synced_at TEXT
);
```

### Users Table
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    server_id INTEGER NOT NULL REFERENCES servers(id) ON DELETE CASCADE,
    nick TEXT NOT NULL,
    user TEXT NOT NULL,
    host TEXT,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(server_id, nick)
);
```

### Channel Members Table
```sql
CREATE TABLE channel_members (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id INTEGER NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    mode TEXT DEFAULT '',
    joined_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(channel_id, user_id)
);
```

### Ban List Table
```sql
CREATE TABLE ban_list (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id INTEGER NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    ban_mask TEXT NOT NULL,
    set_by TEXT,
    set_at TEXT,
    UNIQUE(channel_id, ban_mask)
);
```

### Ignore List Table
```sql
CREATE TABLE ignore_list (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nick TEXT NOT NULL UNIQUE,
    ignore_time INTEGER DEFAULT 0,  -- 0 = permanent
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Notify List Table
```sql
CREATE TABLE notify_list (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nick TEXT NOT NULL UNIQUE,
    notify_on_offline BOOLEAN DEFAULT TRUE,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Aliases Table
```sql
CREATE TABLE aliases (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    replacement TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Key Bindings Table
```sql
CREATE TABLE key_bindings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    key_sequence TEXT NOT NULL UNIQUE,
    command TEXT NOT NULL,
    description TEXT
);
```

### Variables/Settings Table
```sql
CREATE TABLE variables (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    value TEXT,
    updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### History Table
```sql
CREATE TABLE history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    server_id INTEGER REFERENCES servers(id) ON DELETE CASCADE,
    channel TEXT,
    type TEXT NOT NULL,  -- 'message', 'action', 'notice', 'join', etc.
    sender TEXT,
    content TEXT NOT NULL,
    timestamp TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    indexed timestamp
);

CREATE INDEX idx_history_server ON history(server_id);
CREATE INDEX idx_history_timestamp ON history(timestamp DESC);
```

### Lastlog Table
```sql
CREATE TABLE lastlog (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    server_id INTEGER REFERENCES servers(id) ON DELETE CASCADE,
    window TEXT,
    type TEXT,
    sender TEXT,
    content TEXT,
    timestamp TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### DCC Transfers Table
```sql
CREATE TABLE dcc_transfers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type TEXT NOT NULL,  -- 'send' or 'recv'
    filename TEXT NOT NULL,
    size INTEGER,
    transferred INTEGER DEFAULT 0,
    host TEXT,
    port INTEGER,
    partner_nick TEXT,
    status TEXT DEFAULT 'pending',  -- pending, active, complete, failed, cancelled
    started_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TEXT
);
```

### DCC Chat Table
```sql
CREATE TABLE dcc_chat (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    partner_nick TEXT NOT NULL,
    host TEXT,
    port INTEGER,
    local_port INTEGER,
    status TEXT DEFAULT 'listening',
    started_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Script Events Table
```sql
CREATE TABLE script_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    description TEXT
);
```

### Script Hooks Table
```sql
CREATE TABLE script_hooks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    script_id INTEGER REFERENCES scripts(id) ON DELETE CASCADE,
    event_id INTEGER REFERENCES script_events(id),
    priority INTEGER DEFAULT 0,
    code TEXT NOT NULL,
    enabled BOOLEAN DEFAULT TRUE
);
```

### Scripts Table
```sql
CREATE TABLE scripts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    path TEXT NOT NULL,
    loaded BOOLEAN DEFAULT FALSE,
    loaded_at TEXT,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

## Encryption for Sensitive Data

### Key Derivation
```rust
// Use Argon2 for password hashing
// Use AES-GCM for field encryption

const KEY_FILE: &str = "~/.config/bitchx/keyfile";

pub fn derive_key(password: &str) -> [u8; 32] {
    // PBKDF2 with 100k iterations
    pbkdf2_hmac::<Sha256>(password.as_bytes(), b"bitchx-salt", 100_000)
}
```

### Encrypted Fields
- Server passwords
- Channel keys
- NickServ passwords
- DCC files (optional)

---

## Migration from Legacy Format

### Old Format (BitchX.rc, .bitchxrc)
```
SET SERVER_HOST irc.example.com
SET SERVER_PORT 6667
SET NICK mynick
```

### Migration Strategy
1. Parse legacy format
2. Import into SQLite
3. Validate data integrity
4. Backup original files
5. Move to new format

### Migration Script
```rust
pub fn migrate_legacy_config(path: &Path) -> Result<(), BitchXError> {
    let content = std::fs::read_to_string(path)?;
    
    for line in content.lines() {
        if line.starts_with("SET ") {
            let parts: Vec<_> = line[4..].splitn(2, ' ').collect();
            if parts.len() == 2 {
                let (key, value) = (parts[0], parts[1]);
                migrate_setting(key, value)?;
            }
        }
    }
    
    Ok(())
}
```

---

## Performance Considerations

### Indexes
- `history.timestamp` - for time-based queries
- `history.server_id` - for server filtering
- `channel_members.channel_id` - for member lookups
- `ignore_list.nick` - for ignore checks

### Query Optimization
- Use parameterized queries
- Batch inserts for bulk operations
- Vacuum periodically

### Size Limits
- History: Keep last 100,000 messages per channel
- Config: No practical limit
- Ignore list: No practical limit

---

## Backup & Restore

### Automatic Backup
```bash
# Daily backup at 3 AM
0 3 * * * /usr/bin/sqlite3 ~/.config/bitchx/bitchx.db ".backup /backup/bitchx.db.$(date +\%Y\%m\%d)"
```

### Restore Procedure
```bash
sqlite3 ~/.config/bitchx/bitchx.db ".restore /backup/bitchx.db.20260430"
```
