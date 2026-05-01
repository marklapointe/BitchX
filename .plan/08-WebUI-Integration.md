# BitchX WebUI Integration Plan

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** None

## 1. Overview

The WebUI provides a browser-based interface for controlling and interacting with BitchX. This is an **optional feature** that must be explicitly enabled by the user.

### 1.1 Design Principles

- **Off by default** — WebUI must never start without explicit user request
- **Secure by default** — Authentication required, TLS preferred
- **Minimal attack surface** — Only expose what is necessary
- **Graceful degradation** — Works without WebUI if disabled

### 1.2 Activation Methods

The WebUI can be activated in two ways:

1. **Startup argument** — `--webui` or `--web-ui` flag
2. **Runtime command** — `/webui start` command within the IRC session

Both methods require the same authentication credentials.

---

## 2. WebUI Architecture

### 2.1 Module Structure

```
bitchx-rs/
├── src/
│   └── webui/
│       ├── mod.rs              # WebUI module root
│       ├── server.rs           # HTTP/WebSocket server
│       ├── routes.rs           # HTTP route handlers
│       ├── auth.rs             # Authentication middleware
│       ├── tls.rs              # TLS configuration
│       ├── websocket.rs        # WebSocket handling
│       ├── api/
│       │   ├── mod.rs
│       │   ├── servers.rs      # Server management API
│       │   ├── channels.rs     # Channel management API
│       │   ├── messages.rs     # Message sending API
│       │   └── status.rs       # Status/presence API
│       ├── templates/          # HTML/CSS/JS assets
│       │   ├── index.html      # Main dashboard
│       │   ├── chat.html       # Chat view
│       │   ├── settings.html   # Settings page
│       │   └── static/         # CSS, JS, fonts
│       └── state.rs            # WebUI session state
```

### 2.2 Dependencies (Cargo.toml Additions)

```toml
[dependencies]
# Web framework
axum = "0.7"              # Async HTTP server
tower = "0.4"             # Middleware
tower-http = { version = "0.5", features = ["cors", "fs", "trace"] }

# WebSocket
tokio-tungstenite = "0.21"
futures-util = "0.3"

# TLS
rustls = { version = "0.23", features = ["quic"] }
rustls-pemfile = "2"
rcgen = "0.13"

# Password hashing
argon2 = "0.5"

# Session management
uuid = { version = "1.6", features = ["v4", "serde"] }
```

### 2.3 Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         Web Browser                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Dashboard  │  │   Chat UI   │  │     Settings Panel      │  │
│  └──────┬──────┘  └──────┬──────┘  └────────────┬────────────┘  │
│         │                 │                      │               │
│         └─────────────────┼──────────────────────┘               │
│                           │ WebSocket/HTTP                        │
└───────────────────────────┼───────────────────────────────────────┘
                            │ TLS (optional)
┌───────────────────────────┼───────────────────────────────────────┐
│                    WebUI Server (axum)                            │
│  ┌──────────────┐  ┌─────┴─────┐  ┌────────────────────────────┐ │
│  │ Auth Middleware│─▶│  Router   │──▶│  API Handlers             │ │
│  └──────────────┘  └───────────┘  └────────────────────────────┘ │
│                           │                                       │
│                           ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              WebUI State Manager                             │ │
│  │  - Active connections tracking                               │ │
│  │  - Session tokens                                            │ │
│  │  - Command forwarding                                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼ IPC (Unix socket)
┌───────────────────────────────────────────────────────────────────┐
│                      BitchX Core                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  IRC Protocol │  │   UI Layer   │  │   Scripting Engine       │ │
│  │   Handler    │  │   (crossterm)│  │     (Rhai)               │ │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

---

## 3. Configuration

### 3.1 Configuration File (config.toml additions)

```toml
[webui]
# Enable/disable WebUI (default: false)
enabled = false

# Port to listen on (default: 8080)
port = 8080

# Bind address (default: 127.0.0.1 for security)
bind_address = "127.0.0.1"

# Enable TLS (default: true if certificates configured)
tls_enabled = true

# TLS certificate path (generate with --webui-gen-cert)
tls_cert = "~/.config/bitchx/webui.crt"

# TLS private key path
tls_key = "~/.config/bitchx/webui.key"

# Require TLS even for localhost (default: false)
tls_required_local = false

# Session timeout in seconds (default: 3600 = 1 hour)
session_timeout = 3600

# Maximum concurrent sessions (default: 5)
max_sessions = 5

# CORS settings (default: deny all)
cors_allowed_origins = []

# Logging level for WebUI (default: info)
log_level = "info"
```

### 3.2 Command-Line Arguments

```bash
# Enable WebUI with defaults
bitchx --webui

# Enable WebUI with custom port
bitchx --webui --webui-port 9000

# Enable WebUI with TLS
bitchx --webui --webui-tls

# Generate self-signed certificate
bitchx --webui-gen-cert

# Show WebUI help
bitchx --webui-help
```

### 3.3 Runtime Commands

```irc
# Start WebUI (if enabled in config)
/webui start

# Stop WebUI
/webui stop

# Restart WebUI
/webui restart

# Show WebUI status
/webui status

# Generate new TLS certificate
/webui gencert

# Change WebUI password
/webui password <newpassword>

# Show connected WebUI clients
/webui clients

# Kick a WebUI client
/webui kick <session_id>

# Reload WebUI configuration
/webui reload
```

---

## 4. Authentication

### 4.1 Credential Storage

Passwords are hashed using Argon2id with secure parameters:

```rust
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};

fn hash_password(password: &str) -> Result<String, Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    let hash = argon2.hash_password(password.as_bytes(), &salt)?;
    Ok(hash.to_string())
}

fn verify_password(password: &str, hash: &str) -> Result<bool, Error> {
    let parsed_hash = PasswordHash::new(hash)?;
    Ok(Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok())
}
```

### 4.2 Initial Setup Flow

1. On first WebUI activation, prompt for password creation:
   ```
   [WebUI] First-time setup required
   [WebUI] Enter new WebUI password: ********
   [WebUI] Confirm password: ********
   [WebUI] Password set. Access at https://127.0.0.1:8080
   ```

2. Store hashed password in `~/.config/bitchx/webui_passwd`

### 4.3 Session Authentication

```rust
// Login endpoint
async fn login(
    Json(payload): Json<LoginRequest>,
    State(state): State<Arc<AppState>>,
) -> Result<Json<LoginResponse>, StatusCode> {
    // Verify password
    if !verify_password(&payload.password, &state.password_hash)? {
        return Err(StatusCode::UNAUTHORIZED);
    }

    // Check session limit
    if state.active_sessions() >= state.config.max_sessions {
        return Err(StatusCode::TOO_MANY_REQUESTS);
    }

    // Generate session token
    let token = Uuid::new_v4().to_string();
    let session = Session {
        id: token.clone(),
        created_at: Utc::now(),
        last_activity: Utc::now(),
        ip_address: payload.ip_address,
    };

    state.add_session(session).await;
    Ok(Json(LoginResponse { token }))
}
```

### 4.4 Session Token Format

- UUID v4 format: `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`
- Stored in `HttpOnly`, `Secure`, `SameSite=Strict` cookie
- Transmitted via `Authorization: Bearer <token>` header for API calls
- Session expires after `session_timeout` seconds of inactivity

### 4.5 Authentication Middleware

```rust
async fn auth_middleware(
    req: Request,
    next: Next,
    state: Arc<AppState>,
) -> Result<Response, StatusCode> {
    // Extract token from header or cookie
    let token = req
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .or_else(|| {
            req.headers()
                .get("Cookie")
                .and_then(|v| v.to_str().ok())
                .and_then(|c| c.split("session=").nth(1))
                .and_then(|c| c.split(';').next())
        });

    let token = token.ok_or(StatusCode::UNAUTHORIZED)?;

    // Validate session
    let session = state.validate_session(token).await
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // Update last activity
    state.touch_session(&session.id).await;

    // Inject session into request extensions
    let mut req = req;
    req.extensions_mut().insert(session);
    Ok(next.run(req).await)
}
```

---

## 5. TLS Configuration

### 5.1 Certificate Generation

```rust
use rcgen::{BasicConstraints, Certificate, DnType, SanType};
use rustls::{Certificate as RustlsCert, PrivateKey};
use rustls_pemfile::{read_one, Item};

fn generate_self_signed_cert(validity_days: u64) -> Result<(RustlsCert, PrivateKey), Error> {
    let mut cert_params = CertificateParams::default();
    cert_params.common_name = "BitchX WebUI";
    cert_params.is_ca = BasicConstraints::Ca(false);
    cert_params.key_usages = vec![
        KeyUsagePurpose::KeyEncipherment,
        KeyUsagePurpose::DigitalSignature,
    ];
    cert_params.subject_alt_names = vec![
        SanType::DnsName("localhost".to_string()),
        SanType::IpAddress(IpAddr::from([127, 0, 0, 1])),
        SanType::IpAddress(IpAddr::from([0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1])),
    ];

    let cert = cert_params.self_signed()?;
    let pem = cert.serialize_pem()?;
    
    let cert_data = read_one(&mut pem.as_bytes())?;
    match cert_data {
        Item::Cert(cert) => Ok((RustlsCert(cert), PrivateKey(cert.serialize_private_key_der()))),
        _ => Err(Error::InvalidCert),
    }
}
```

### 5.2 Rustls Server Config

```rust
use rustls::{ServerConfig, RootStore};
use rustls_pemfile::read_one;

fn create_tls_config(cert_path: &Path, key_path: &Path) -> Result<ServerConfig, Error> {
    let mut cert_file = File::open(cert_path)?;
    let cert_data = read_one(&mut cert_file)?.certificate()?;
    
    let mut key_file = File::open(key_path)?;
    let key_data = read_one(&mut key_file)?.pkey()?;
    
    let config = ServerConfig::builder()
        .with_safe_defaults()
        .with_no_client_auth()
        .with_single_cert(
            vec![RustlsCert(cert_data)],
            PrivateKey(key_data),
        )?;
    
    Ok(config)
}
```

### 5.3 TLS Security Requirements

- **Minimum TLS version:** 1.2 (1.3 preferred)
- **Allowed cipher suites:** TLS_AES_128_GCM_SHA256, TLS_AES_256_GCM_SHA384, TLS_CHACHA20_POLY1305_SHA256
- **Disallowed:** Any cipher suite using CBC mode, RC4, 3DES, or export-grade ciphers
- **Certificate pinning:** Support for HSTS header
- **Perfect forward secrecy:** Required (ECDHE key exchange)

---

## 6. API Design

### 6.1 REST Endpoints

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| POST | `/api/auth/login` | Authenticate user | No |
| POST | `/api/auth/logout` | End session | Yes |
| GET | `/api/status` | Get client status | Yes |
| GET | `/api/servers` | List connected servers | Yes |
| POST | `/api/servers/connect` | Connect to server | Yes |
| POST | `/api/servers/disconnect` | Disconnect from server | Yes |
| GET | `/api/channels` | List joined channels | Yes |
| POST | `/api/channels/join` | Join a channel | Yes |
| POST | `/api/channels/part` | Part from a channel | Yes |
| GET | `/api/channels/:name/messages` | Get channel history | Yes |
| POST | `/api/channels/:name/send` | Send message to channel | Yes |
| GET | `/api/users` | List known users | Yes |
| GET | `/api/config` | Get configuration | Yes |
| PUT | `/api/config` | Update configuration | Yes |
| GET | `/api/webui/status` | WebUI status | Yes |
| GET | `/api/webui/clients` | List WebUI clients | Yes |

### 6.2 WebSocket Endpoints

| Endpoint | Description |
|----------|-------------|
| `/ws/chat` | Real-time chat messages |
| `/ws/events` | Server/channel events (joins, parts, etc.) |
| `/ws/dcc` | DCC transfer progress |

### 6.3 API Examples

#### Login
```http
POST /api/auth/login
Content-Type: application/json

{
    "username": "admin",
    "password": "secretpassword"
}

Response:
{
    "token": "550e8400-e29b-41d4-a716-446655440000",
    "expires_in": 3600
}
```

#### Send Message
```http
POST /api/channels/%23bitchx/send
Authorization: Bearer 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
    "message": "Hello from WebUI!"
}
```

### 6.4 Error Responses

```json
{
    "error": {
        "code": "UNAUTHORIZED",
        "message": "Invalid or expired session"
    }
}
```

Error codes:
- `UNAUTHORIZED` (401) - Invalid credentials or session
- `FORBIDDEN` (403) - Insufficient permissions
- `NOT_FOUND` (404) - Resource not found
- `RATE_LIMITED` (429) - Too many requests
- `INTERNAL_ERROR` (500) - Server error

---

## 7. WebUI Frontend

### 7.1 Technology Stack

- **No build step required** — Vanilla HTML/CSS/JavaScript for simplicity
- **Progressive enhancement** — Works without JavaScript for basic functionality
- **Responsive design** — Mobile-friendly layout
- **No external dependencies** — All assets bundled

### 7.2 HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BitchX WebUI</title>
    <link rel="stylesheet" href="/static/styles.css">
</head>
<body>
    <div id="app">
        <header id="header">
            <h1>BitchX IRC Client</h1>
            <div id="connection-status"></div>
        </header>
        
        <nav id="sidebar">
            <div id="server-list"></div>
            <div id="channel-list"></div>
        </nav>
        
        <main id="chat-area">
            <div id="messages"></div>
            <div id="input-area">
                <input type="text" id="message-input" placeholder="Type message...">
                <button id="send-btn">Send</button>
            </div>
        </main>
        
        <aside id="user-list">
            <h3>Users</h3>
            <ul id="users"></ul>
        </aside>
    </div>
    
    <script src="/static/app.js"></script>
</body>
</html>
```

### 7.3 Security Headers

```rust
fn security_headers() -> HeaderMap {
    let mut headers = HeaderMap::new();
    headers.insert("X-Frame-Options", "SAMEORIGIN".parse().unwrap());
    headers.insert("X-Content-Type-Options", "nosniff".parse().unwrap());
    headers.insert("X-XSS-Protection", "1; mode=block".parse().unwrap());
    headers.insert("Referrer-Policy", "strict-origin-when-cross-origin".parse().unwrap());
    headers.insert(
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'".parse().unwrap(),
    );
    headers
}
```

---

## 8. Integration with Core

### 8.1 IPC Communication

WebUI communicates with the core via Unix domain sockets:

```rust
// webui/src/ipc.rs
use tokio::net::UnixStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[derive(Serialize, Deserialize)]
pub enum WebUiCommand {
    SendMessage { target: String, message: String },
    JoinChannel { channel: String, password: Option<String> },
    PartChannel { channel: String, reason: Option<String> },
    Connect { server: String, port: u16, ssl: bool },
    Disconnect { server: String },
    GetStatus,
    GetChannels,
    GetUsers { channel: String },
}

#[derive(Serialize, Deserialize)]
pub enum CoreResponse {
    Ok,
    Error { message: String },
    Status(StatusResponse),
    Channels(Vec<ChannelInfo>),
    Users(Vec<UserInfo>),
}
```

### 8.2 Event Forwarding

```rust
// Forward IRC events to WebUI clients
async fn forward_to_webui(event: IrcEvent) {
    let json = serde_json::to_string(&event).unwrap();
    for client in webui_clients.lock().await.iter() {
        if let Err(e) = client.send(Message::Text(json.clone())).await {
            // Handle disconnected client
            remove_client(client.id).await;
        }
    }
}
```

---

## 9. Security Considerations

### 9.1 Threat Model

1. **Unauthorized access** — Prevented by password authentication
2. **Session hijacking** — Prevented by secure session tokens, HTTPS
3. **Cross-site scripting (XSS)** — Prevented by CSP headers, input sanitization
4. **Cross-site request forgery (CSRF)** — Prevented by SameSite cookies, CSRF tokens
5. **Brute force attacks** — Rate limiting on login endpoint
6. **Denial of service** — Connection limits, resource quotas

### 9.2 Rate Limiting

```rust
use std::sync::Arc;
use std::time::{Duration, Instant};

struct RateLimiter {
    requests: Mutex<HashMap<String, Vec<Instant>>>,
    max_requests: usize,
    window: Duration,
}

impl RateLimiter {
    fn check(&self, key: &str) -> bool {
        let mut requests = self.requests.lock().unwrap();
        let now = Instant::now();
        
        // Remove old requests outside window
        requests.entry(key.to_string())
            .or_insert_with(Vec::new)
            .retain(|&t| now.duration_since(t) < self.window);
        
        // Check limit
        if requests[key].len() >= self.max_requests {
            return false;
        }
        
        requests[key].push(now);
        true
    }
}

// Apply to login endpoint: 5 attempts per minute
```

### 9.3 Input Validation

```rust
fn validate_nick(nick: &str) -> Result<(), ValidationError> {
    if nick.is_empty() || nick.len() > 50 {
        return Err(ValidationError::new("Nick must be 1-50 characters"));
    }
    if !nick.chars().all(|c| c.is_alphanumeric() || c == '-' || c == '_' || c == '[' || c == ']') {
        return Err(ValidationError::new("Invalid characters in nick"));
    }
    Ok(())
}

fn validate_channel(channel: &str) -> Result<(), ValidationError> {
    if !channel.starts_with('#') && !channel.starts_with('&') {
        return Err(ValidationError::new("Channel must start with # or &"));
    }
    if channel.len() > 200 {
        return Err(ValidationError::new("Channel name too long"));
    }
    // Additional IRC channel validation
    Ok(())
}
```

---

## 10. Migration Phase Integration

### 10.1 Task Dependencies

| Task ID | Task | Dependencies |
|---------|------|--------------|
| P5.1.1 | Create WebUI module structure | P0.1.1 (Project setup) |
| P5.1.2 | Implement HTTP server with axum | P5.1.1 |
| P5.1.3 | Add TLS support | P5.1.2 |
| P5.1.4 | Implement authentication | P5.1.2 |
| P5.1.5 | Create WebSocket handler | P5.1.2 |
| P5.1.6 | Implement REST API endpoints | P5.1.4, P5.1.5 |
| P5.1.7 | Create frontend HTML/CSS/JS | P5.1.6 |
| P5.1.8 | Add CLI arguments | P5.1.1 |
| P5.1.9 | Add runtime commands | P5.1.2 |
| P5.1.10 | Security hardening | P5.1.4, P5.1.7 |

### 10.2 Phase Assignment

WebUI integration is a **Phase 5** task, running concurrently with:
- Phase 4: Command System
- Phase 5: User Interface (Terminal)
- Phase 5: Database/Persistence

### 10.3 Testing Strategy

1. **Unit tests** — Test all handlers, auth, validation
2. **Integration tests** — Test WebUI ↔ Core communication
3. **Security tests** — Fuzz endpoints, test auth bypasses
4. **Load tests** — Verify concurrent session handling

---

## 11. Default Port Assignment

| Port | Service | Protocol |
|------|---------|----------|
| 8080 | WebUI HTTP | TCP |
| 8443 | WebUI HTTPS | TCP |
| 9000 | WebUI (alternative) | TCP |

The default port is **8080** for HTTP, **8443** for HTTPS.

---

## 12. Implementation Notes

### 12.1 Graceful Shutdown

```rust
async fn shutdown_signal() {
    // Handle Ctrl+C
    tokio::signal::ctrl_c().await.ok();
}

#[tokio::main]
async fn main() {
    let server = start_webui_server().await;
    
    tokio::spawn(async {
        shutdown_signal().await;
        info!("Shutting down WebUI...");
        server.shutdown().await;
    });
}
```

### 12.2 Health Check Endpoint

```rust
async fn health() -> &'static str {
    "OK"
}

let app = Router::new()
    .route("/health", get(health))
    .route("/api/*path", any(handler))
    .layer(auth_layer);
```

### 12.3 Metrics Endpoint (Optional)

```rust
#[derive(Serialize)]
struct Metrics {
    active_sessions: usize,
    total_requests: u64,
    uptime_seconds: u64,
}

async fn metrics(state: State<Arc<AppState>>) -> Json<Metrics> {
    Json(Metrics {
        active_sessions: state.active_sessions(),
        total_requests: state.total_requests(),
        uptime_seconds: state.uptime_secs(),
    })
}
```

---

## 13. References

- [Axum Web Framework](https://docs.rs/axum/latest/axum/)
- [Rustls TLS](https://docs.rs/rustls/latest/rustls/)
- [Tokio WebSocket](https://docs.rs/tokio-tungstenite/latest/tokio_tungstenite/)
- [Argon2 Password Hashing](https://docs.rs/argon2/latest/argon2/)
- [OWASP Web Security Guidelines](https://cheatsheetseries.owasp.org/)
