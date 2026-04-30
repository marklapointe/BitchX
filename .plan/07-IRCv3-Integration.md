# IRCv3 Integration Plan

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** None

## Overview

This document describes the integration of IRCv3 specifications into the BitchX Rust rewrite. IRCv3 builds on the core IRC protocol (RFC 1459, RFC 2812) with backward-compatible extensions.

### Key Resources

- **IETF Draft:** [draft-samuelson-irc-v3-00](https://www.ietf.org/archive/id/draft-samuelson-irc-v3-00.html) — Audio/Video extensions
- **IRCv3 Specs:** [ircv3.net/irc](https://ircv3.net/irc/) — Full specification index
- **Modern IRC:** [ircv3.net/irc](https://ircv3.net/irc/) — Core protocol specification (supersedes RFC1459, RFC2812)

### Compatibility Goals

1. **Backward Compatible** — All existing BitchX commands and behavior preserved
2. **IRCv3 Feature Parity** — Support all stable IRCv3 extensions
3. **Progressive Enhancement** — Features work without extensions, enhanced with them

---

## IETF IRCv3 Draft (Audio/Video)

### Source: [draft-samuelson-irc-v3-00](https://www.ietf.org/archive/id/draft-samuelson-irc-v3-00.html)

The IETF draft extends IRC for audio/video capabilities:

### 1. DCC Extensions for A/V Streaming

Extended DCC protocol for streaming audio and video data:

```
DCC SEND <filename> <address> <port> <stream_type> <codec> [options]
DCC STREAM <type> <address> <port> <session_id>
DCC ACCEPT <session_id> <port>
DCC CLOSE <session_id>
```

| Parameter | Description |
|-----------|-------------|
| `stream_type` | `audio`, `video`, or `av` (combined) |
| `codec` | Codec identifier (e.g., `opus`, `vp8`, `h264`) |
| `session_id` | Unique session identifier |
| `options` | Optional parameters (bitrate, resolution, etc.) |

### 2. New Channel Modes

| Mode | Character | Description |
|------|-----------|-------------|
| `a` | Audio | Channel supports audio streaming |
| `v` | Video | Channel supports video streaming |
| `A` | Audio-Only | Only audio streams allowed |
| `V` | Video-Only | Only video streams allowed |

### 3. Modified JOIN Command

```
JOIN <channel>[,<channel>]* [<key>[,<key>]*] [:<media_preferences>]
```

Media preferences: `audio`, `video`, `av`, `none`

### 4. New PRIVMSG Subcommands

```
PRIVMSG <target> :!stream start <session_id>
PRIVMSG <target> :!stream stop <session_id>
PRIVMSG <target> :!stream pause <session_id>
PRIVMSG <target> :!stream resume <session_id>
```

### 5. Server-to-Server Commands

```
AVCAPS <server> :<capabilities>
AVSYNC <channel> <session_id>
AVQUIT <session_id> [<reason>]
```

### 6. Security Requirements (from IETF Draft)

| Requirement | Implementation |
|-------------|----------------|
| **DoS Prevention** | Rate limiting, stream validation, max bandwidth caps |
| **User Privacy** | TLS 1.3 encryption mandatory for all A/V streams |
| **Content Verification** | Sanitize received A/V data, reject malformed streams |
| **Encryption** | All audio/video streams MUST use TLS 1.3 or later |

---

## IRCv3 Specifications

### Source: [ircv3.net/irc](https://ircv3.net/irc/)

### Priority Implementation Order

#### P0 — Critical (Must Have)

| Spec | Capability | Description |
|------|-----------|-------------|
| Capability Negotiation | `cap` | Framework for all other extensions |
| SASL Authentication | `sasl` | Secure authentication before channel join |
| Message Tags | `message-tags` | Extended metadata on messages |
| Echo Message | `echo-message` | Confirm sent messages |
| Server Time | `server-time-3.2` | Accurate message timestamps |

#### P1 — High Priority

| Spec | Capability | Description |
|------|-----------|-------------|
| Account Tracking | `account-tag`, `account-notify` | Know logged-in users |
| Extended Join | `extended-join` | Full client info on join |
| Multi-prefix | `multi-prefix` | See all channel statuses |
| Away Notify | `away-notify` | Real-time away status |
| Monitor | `monitor` | Efficient online status tracking |

#### P2 — Medium Priority

| Spec | Capability | Description |
|------|-----------|-------------|
| STS (Strict Transport Security) | `sts` | Prevent TLS downgrades |
| Labeled Responses | `labeled-response` | Correlate sent/received messages |
| WHOX | `whoX` | Extended user queries |
| Bot Mode | `bot` | Mark client as bot |
| Invite Notify | `invite-notify` | See channel invites |

#### P3 — Low Priority (Future)

| Spec | Capability | Description |
|------|-----------|-------------|
| Chat History | `chathistory` | Request message history |
| Message IDs | `msgid` | Unique message identifiers |
| SetName | `setname` | Change realname in-use |
| CHGHOST | `chghost` | See username/host changes |
| Standard Replies | `stdcmpro` | Structured error messages |
| Multiline | `draft/multiline` | Extended message length |

#### P4 — Deprecation

| Spec | Status | Notes |
|------|--------|-------|
| STARTTLS | Deprecated | Use STS + port 6697 instead |

---

## Implementation Details

### 1. Capability Negotiation

```rust
// src/protocol/cap.rs

/// IRCv3 Capability Negotiation
#[derive(Debug, Clone)]
pub struct Cap negotiation {
    /// Available capabilities from server
    available: HashMap<String, CapabilityValue>,
    /// Enabled capabilities
    enabled: HashSet<String>,
}

#[derive(Debug, Clone)]
pub enum CapabilityValue {
    /// No value (e.g., `cap-notify`)
    None,
    /// Star value (e.g., `*`)
    Asterisk,
    /// Specific value (e.g., `sasl=PLAIN`)
    Value(String),
}

impl Protocol {
    /// Handle CAP LS - List capabilities
    pub async fn handle_cap_ls(&mut self, caps: Vec<(String, Option<String>)>) {
        for (name, value) in caps {
            self.capabilities.insert(name, value.into());
        }
    }

    /// Request specific capabilities
    pub async fn request_caps(&mut self, caps: &[&str]) -> Result<()> {
        let request = caps.join(" ");
        self.send_command(&format!("CAP REQ :{}", request)).await?;
        Ok(())
    }

    /// Handle CAP ACK/NAK responses
    pub async fn handle_cap_response(&mut self, caps: Vec<String>, ack: bool) {
        if ack {
            for cap in caps {
                self.enabled_caps.insert(cap);
            }
        }
    }
}
```

### 2. SASL Authentication

```rust
// src/protocol/sasl.rs

/// SASL Authentication Mechanism
#[derive(Debug, Clone)]
pub enum SaslMechanism {
    /// PLAIN - Simple username/password
    Plain,
    /// EXTERNAL - Certificate-based
    External,
    /// SCRAM-SHA-256 - Challenge-response
    ScramSha256,
}

/// SASL Authentication State
#[derive(Debug, Clone, PartialEq)]
pub enum SaslState {
    /// Not authenticating
    Inactive,
    /// Waiting for AUTHENTICATE command
    WaitingForAuthenticate,
    /// Send initial response
    SendInitial,
    /// Waiting for challenge
    WaitingForChallenge,
    /// Send response
    SendResponse,
    /// Success
    Success,
    /// Failed
    Failed(String),
}

impl Server {
    /// Perform SASL authentication
    pub async fn authenticate_sasl(&mut self, mechanism: SaslMechanism, credentials: &Credentials) -> Result<()> {
        // Check server supports SASL
        if !self.capabilities.has("sasl") {
            return Err(BitchXError::Protocol("Server does not support SASL".into()));
        }

        self.send_command("CAP REQ :sasl").await?;
        
        // Wait for ACK
        self.sasl_state = SaslState::WaitingForAuthenticate;
        
        match mechanism {
            SaslMechanism::Plain => {
                self.authenticate_plain(credentials).await
            }
            SaslMechanism::ScramSha256 => {
                self.authenticate_scram(credentials).await
            }
            SaslMechanism::External => {
                self.authenticate_external().await
            }
        }
    }

    /// PLAIN authentication
    async fn authenticate_plain(&mut self, creds: &Credentials) -> Result<()> {
        let auth_string = format!("{}\0{}\0{}", 
            creds.authorization_id.as_deref().unwrap_or(&creds.username),
            creds.username,
            creds.password
        );
        let encoded = base64::Engine::encode(&base64::engine::general_purpose::STANDARD, &auth_string);
        
        self.send_command(&format!("AUTHENTICATE {}", encoded)).await?;
        self.sasl_state = SaslState::WaitingForChallenge;
        
        Ok(())
    }
}
```

### 3. Message Tags

```rust
// src/protocol/tags.rs

/// IRCv3 Message Tag
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct MessageTag {
    /// Tag key (e.g., `+draft/channel-context`)
    pub key: String,
    /// Tag value (escaped)
    pub value: String,
}

/// Parse message tags from raw IRC line
pub fn parse_tags(raw: &str) -> Result<Vec<MessageTag>, ParseError> {
    if !raw.starts_with('@') {
        return Ok(Vec::new());
    }

    let tags_str = &raw[1..];
    let mut tags = Vec::new();

    for tag in tags_str.split(';') {
        let (key, value) = tag.split_once('=')
            .unwrap_or((tag, ""));
        tags.push(MessageTag {
            key: key.to_string(),
            value: unescape_tag_value(value),
        });
    }

    Ok(tags)
}

/// Unescape tag value per IRCv3 spec
fn unescape_tag_value(value: &str) -> String {
    value
        .replace("\\:", ";")
        .replace("\\s", " ")
        .replace("\\\\", "\\")
        .replace("\\r", "\r")
        .replace("\\n", "\n")
}

/// Escape tag value per IRCv3 spec
pub fn escape_tag_value(value: &str) -> String {
    value
        .replace('\\', "\\\\")
        .replace(';', "\\:")
        .replace(' ', "\\s")
        .replace('\r', "\\r")
        .replace('\n', "\\n")
}

/// Add tag to outgoing message
pub fn add_tag(message: &mut IrcMessage, key: &str, value: &str) {
    let escaped = escape_tag_value(value);
    message.tags.push(MessageTag {
        key: key.to_string(),
        value: escaped,
    });
}
```

### 4. Server Time

```rust
// src/protocol/server_time.rs

use chrono::{DateTime, Utc, TimeZone};

/// Server Time Tag
#[derive(Debug, Clone)]
pub struct ServerTime {
    /// Timestamp as ISO 8601 string
    pub timestamp: DateTime<Utc>,
    /// Optional microseconds
    pub microseconds: Option<u32>,
}

impl ServerTime {
    /// Parse server-time tag value
    pub fn parse(value: &str) -> Result<Self, ParseError> {
        // Format: YYYY-MM-DDTHH:MM:SS(.ffffff)Z
        let value = value.trim_end_matches('Z');
        
        let (timestamp_str, micros) = if let Some((ts, us)) = value.split_once('.') {
            (ts, Some(us.parse()?))
        } else {
            (value, None)
        };

        let dt = DateTime::parse_from_rfc3339(&format!("{}Z", timestamp_str))
            .map_err(|_| ParseError::InvalidServerTime)?;
        
        Ok(ServerTime {
            timestamp: dt.with_timezone(&Utc),
            microseconds: micros,
        })
    }

    /// Format for outgoing message tag
    pub fn to_tag_value(&self) -> String {
        if let Some(us) = self.microseconds {
            format!("{}.{:06}Z", self.timestamp.format("%Y-%m-%dT%H:%M:%S"), us)
        } else {
            format!("{}Z", self.timestamp.format("%Y-%m-%dT%H:%M:%S"))
        }
    }
}
```

### 5. Extended Join

```rust
// src/irc/channel.rs

/// Extended Join Account Info
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum AccountStatus {
    /// User is logged into an account
    LoggedIn(String),
    /// User is not logged in (shows as "*")
    NotLoggedIn,
    /// Account status unknown
    Unknown,
}

/// Extended Join Information
#[derive(Debug, Clone)]
pub struct ExtendedJoinInfo {
    /// Nickname
    pub nick: String,
    /// Username (ident)
    pub user: String,
    /// Hostname
    pub host: String,
    /// Real name (gecos)
    pub gecos: String,
    /// Account status
    pub account: AccountStatus,
}

impl Channel {
    /// Handle extended JOIN (with account info)
    pub fn handle_extended_join(&mut self, info: ExtendedJoinInfo) {
        let user = User {
            nick: info.nick.clone(),
            user: info.user.clone(),
            host: info.host.clone(),
            gecos: info.gecos.clone(),
            account: info.account.clone(),
            ..Default::default()
        };

        self.members.insert(info.nick, user);
    }

    /// Handle standard JOIN (when extended-join not available)
    pub fn handle_standard_join(&mut self, nick: &str, user: &str, host: &str) {
        let user = User {
            nick: nick.to_string(),
            user: user.to_string(),
            host: host.to_string(),
            account: AccountStatus::Unknown,
            ..Default::default()
        };

        self.members.insert(nick.to_string(), user);
    }
}
```

### 6. Audio/Video Extensions (IETF Draft)

```rust
// src/dcc/av_stream.rs

use tokio::net::TcpStream;
use tokio_rustls::client::TlsStream;

/// Stream type for A/V transmission
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AvStreamType {
    Audio,
    Video,
    Combined, // Audio + Video
}

/// Codec identifiers
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AvCodec {
    Opus,
    VP8,
    VP9,
    H264,
    H265,
    AV1,
}

/// A/V Stream Session
#[derive(Debug, Clone)]
pub struct AvSession {
    /// Unique session identifier
    pub id: Uuid,
    /// Stream type
    pub stream_type: AvStreamType,
    /// Primary codec
    pub codec: AvCodec,
    /// Remote address
    pub address: IpAddr,
    /// Remote port
    pub port: u16,
    /// TLS stream (encrypted per security requirement)
    pub stream: Option<TlsStream<TcpStream>>,
    /// Session state
    pub state: AvSessionState,
    /// Bandwidth limit (bytes/sec)
    pub bandwidth_limit: u64,
}

#[derive(Debug, Clone, PartialEq)]
pub enum AvSessionState {
    /// Session created, not connected
    Created,
    /// Connecting to peer
    Connecting,
    /// Waiting for acceptance
    WaitingAccept,
    /// Active streaming
    Streaming,
    /// Paused
    Paused,
    /// Closed
    Closed,
    /// Failed
    Failed(String),
}

impl AvSession {
    /// Create new A/V session
    pub fn new(stream_type: AvStreamType, codec: AvCodec) -> Self {
        Self {
            id: Uuid::new_v4(),
            stream_type,
            codec,
            address: IpAddr::V4(Ipv4Addr::UNSPECIFIED),
            port: 0,
            stream: None,
            state: AvSessionState::Created,
            bandwidth_limit: 10_000_000, // 10 MB/s default
        }
    }

    /// Initiate A/V stream (TLS encrypted)
    pub async fn start(&mut self, address: IpAddr, port: u16) -> Result<()> {
        if self.state != AvSessionState::Created {
            return Err(BitchXError::Dcc("Session already started".into()));
        }

        self.address = address;
        self.port = port;
        self.state = AvSessionState::Connecting;

        // Create TCP connection
        let addr = SocketAddr::new(address, port);
        let tcp = TcpStream::connect(addr).await?;

        // MUST use TLS 1.3 for A/V streams (per IETF draft security requirement)
        let tls_config = create_av_tls_config()?;
        let connector = TlsConnector::from(Arc::new(tls_config));
        
        let domain = webpki::DnsNameRef::try_from_ascii_str("av-stream")
            .map_err(|_| BitchXError::Tls("Invalid DNS name".into()))?;
        
        let tls_stream = connector.connect(domain, tcp).await?;
        self.stream = Some(tls_stream);
        self.state = AvSessionState::Streaming;

        Ok(())
    }

    /// Handle incoming stream control message
    pub fn handle_control(&mut self, cmd: &str, args: &[&str]) -> Result<()> {
        match cmd {
            "start" => {
                self.state = AvSessionState::Streaming;
            }
            "stop" => {
                self.state = AvSessionState::Closed;
            }
            "pause" => {
                self.state = AvSessionState::Paused;
            }
            "resume" => {
                self.state = AvSessionState::Streaming;
            }
            _ => {
                return Err(BitchXError::Dcc(format!("Unknown control command: {}", cmd)));
            }
        }
        Ok(())
    }
}

/// Create TLS configuration for A/V streams
fn create_av_tls_config() -> Result<rustls::ClientConfig> {
    let mut config = rustls::ClientConfig::builder()
        .with_safe_default_cipher_suites()
        .with_safe_default_kx_groups()
        .with_protocol_versions(&[&rustls::version::TLS13])? // TLS 1.3 ONLY per spec
        
    config.set_certificate_store(rustls::RootCertStore::empty());
    
    Ok(config)
}
```

---

## Channel Mode Handling

### Mode Registration

```rust
// src/irc/channel_modes.rs

/// Channel mode definitions
pub fn register_channel_modes(modes: &mut ModeRegistry) {
    // Standard modes (RFC 2811)
    modes.register_mode('o', ModeType::List, ModeClass::Privilege); // Channel operator
    modes.register_mode('v', ModeType::List, ModeClass::Privilege);   // Voice
    modes.register_mode('b', ModeType::List, ModeClass::Ban);         // Ban
    modes.register_mode('e', ModeType::List, ModeClass::Ban);         // Ban exception
    modes.register_mode('I', ModeType::List, ModeClass::Invite);      // Invite exception
    
    // IRCv3 Audio/Video modes (IETF draft)
    modes.register_mode('a', ModeType::Parameter, ModeClass::Media);  // Audio capability
    modes.register_mode('v', ModeType::Parameter, ModeClass::Media); // Video capability  
    modes.register_mode('A', ModeType::NoParam, ModeClass::Media);   // Audio-only
    modes.register_mode('V', ModeType::NoParam, ModeClass::Media);  // Video-only
    
    // Other standard modes
    modes.register_mode('k', ModeType::Parameter, ModeClass::Key);   // Channel key
    modes.register_mode('l', ModeType::Parameter, ModeClass::Limit); // User limit
    modes.register_mode('i', ModeType::NoParam, ModeClass::Access);  // Invite-only
    modes.register_mode('m', ModeType::NoParam, ModeClass::Moderation); // Moderated
    modes.register_mode('n', ModeType::NoParam, ModeClass::Access);  // No external messages
    modes.register_mode('p', ModeType::NoParam, ModeClass::Access);  // Private
    modes.register_mode('s', ModeType::NoParam, ModeClass::Access);   // Secret
    modes.register_mode('t', ModeType::NoParam, ModeClass::Access);  // Topic locked
}

pub fn register_user_modes(modes: &mut ModeRegistry) {
    // Standard user modes
    modes.register_mode('i', ModeType::NoParam, ModeClass::Visibility); // Invisible
    modes.register_mode('w', ModeType::NoParam, ModeClass::Visibility); // WALLOPS
    modes.register_mode('s', ModeType::NoParam, ModeClass::Visibility); // Server notices
    modes.register_mode('o', ModeType::NoParam, ModeClass::Operator);   // Operator
    
    // IRCv3 Bot mode
    modes.register_mode('B', ModeType::NoParam, ModeClass::Bot);       // Bot
}
```

---

## Command Registration Changes

### New IRCv3 Commands

```rust
// src/protocol/commands.rs

/// IRCv3 Command Registry
pub fn register_ircv3_commands(registry: &mut CommandRegistry) {
    // SASL
    registry.register("AUTHENTICATE", CommandType::Auth, AuthenticateHandler);
    
    // CAP extensions
    registry.register("CAP", CommandType::Capability, CapHandler);
    
    // Metadata
    registry.register("METADATA", CommandType::Metadata, MetadataHandler);
    
    // MONITOR
    registry.register("MONITOR", CommandType::Monitor, MonitorHandler);
    
    // User property changes
    registry.register("CHGHOST", CommandType::Property, ChghostHandler);
    registry.register("SETNAME", CommandType::Property, SetnameHandler);
    
    // Account
    registry.register("ACCOUNT", CommandType::Account, AccountHandler);
    
    // Chat history (draft)
    registry.register("CHATHISTORY", CommandType::History, ChatHistoryHandler);
    
    // Channel rename (draft)
    registry.register("RENAME", CommandType::Channel, RenameHandler);
    
    // A/V extensions (IETF draft)
    registry.register("AVCAPS", CommandType::Server, AvcapsHandler);
    registry.register("AVSYNC", CommandType::Server, AvsyncHandler);
    registry.register("AVQUIT", CommandType::Server, AvquitHandler);
}

/// Handle CAP command
pub struct CapHandler;

impl Handler for CapHandler {
    async fn handle(&self, server: &mut Server, msg: &Message) -> Result<()> {
        let subcommand = msg.params.get(0).map(|s| s.as_str()).unwrap_or("");
        
        match subcommand {
            "LS" => {
                // List capabilities
                if let Some(caps) = msg.params.get(1) {
                    server.handle_cap_ls(caps)?;
                }
            }
            "ACK" => {
                // Capabilities acknowledged
                if let Some(caps) = msg.params.get(1) {
                    server.handle_cap_ack(caps)?;
                }
            }
            "NAK" => {
                // Capabilities rejected
                if let Some(caps) = msg.params.get(1) {
                    server.handle_cap_nak(caps)?;
                }
            }
            "DEL" => {
                // Capability removed
                if let Some(cap) = msg.params.get(1) {
                    server.handle_cap_del(cap)?;
                }
            }
            "NEW" => {
                // Capability added
                if let Some(cap) = msg.params.get(1) {
                    server.handle_cap_new(cap)?;
                }
            }
            _ => {
                warn!("Unknown CAP subcommand: {}", subcommand);
            }
        }
        Ok(())
    }
}
```

---

## Feature Detection

```rust
// src/irc/capability.rs

/// Capability feature detection
#[derive(Default)]
pub struct CapabilityDetector {
    /// Detected capabilities
    caps: HashSet<String>,
    /// Capability values
    values: HashMap<String, String>,
}

impl CapabilityDetector {
    /// Check if capability is available
    pub fn has(&self, cap: &str) -> bool {
        self.caps.contains(cap)
    }

    /// Get capability value (if any)
    pub fn get(&self, cap: &str) -> Option<&str> {
        self.values.get(cap).map(|s| s.as_str())
    }

    /// Get SASL mechanisms
    pub fn sasl_mechanisms(&self) -> Vec<&str> {
        self.get("sasl")
            .map(|v| v.split(',').collect())
            .unwrap_or_default()
    }

    /// Check for multi-prefix support
    pub fn supports_multi_prefix(&self) -> bool {
        self.has("multi-prefix")
    }

    /// Check for away-notify support
    pub fn supports_away_notify(&self) -> bool {
        self.has("away-notify")
    }

    /// Check for account tracking
    pub fn supports_account_notify(&self) -> bool {
        self.has("account-notify")
    }

    /// Check for extended join
    pub fn supports_extended_join(&self) -> bool {
        self.has("extended-join")
    }

    /// Check for server time
    pub fn supports_server_time(&self) -> bool {
        self.has("server-time-3.2")
    }

    /// Check for labeled responses
    pub fn supports_labeled_response(&self) -> bool {
        self.has("labeled-response")
    }

    /// Check for STS
    pub fn supports_sts(&self) -> bool {
        self.has("sts")
    }

    /// Check for A/V capabilities (IETF draft)
    pub fn supports_av(&self) -> bool {
        self.get("av-capabilities")
            .map(|v| !v.is_empty())
            .unwrap_or(false)
    }

    /// Get A/V capability details
    pub fn av_capabilities(&self) -> AvCapabilities {
        AvCapabilities {
            audio: self.get("av-capabilities")
                .map(|v| v.contains("audio"))
                .unwrap_or(false),
            video: self.get("av-capabilities")
                .map(|v| v.contains("video"))
                .unwrap_or(false),
            codecs: self.get("av-codecs")
                .map(|v| v.split(',').map(|s| s.to_string()).collect())
                .unwrap_or_default(),
        }
    }
}

/// A/V Capability details
#[derive(Debug, Clone)]
pub struct AvCapabilities {
    pub audio: bool,
    pub video: bool,
    pub codecs: Vec<String>,
}
```

---

## Configuration

### Server Configuration with IRCv3

```rust
// src/config/server.rs

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct ServerConfig {
    /// Server hostname
    pub host: String,
    
    /// Server port
    pub port: u16,
    
    /// Use TLS (recommended)
    pub tls: bool,
    
    /// TLS skip verification (dangerous!)
    #[serde(default)]
    pub tls_skip_verify: bool,
    
    /// IRCv3: SASL authentication
    #[serde(default)]
    pub sasl: Option<SaslConfig>,
    
    /// IRCv3: Requested capabilities
    #[serde(default)]
    pub requested_caps: Vec<String>,
    
    /// IRCv3: Auto-enable STS
    #[serde(default = "default_sts")]
    pub sts_policy: StsPolicy,
    
    /// IETF IRCv3: Auto-enable A/V if available
    #[serde(default)]
    pub av_auto_enable: bool,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct SaslConfig {
    /// SASL mechanism
    pub mechanism: String,
    
    /// Username
    pub username: String,
    
    /// Password or token
    pub password: String,
    
    /// Authorization ID (optional, for SASL EXTERNAL)
    #[serde(default)]
    pub authzid: Option<String>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(tag = "mode")]
pub enum StsPolicy {
    /// Always use STS
    Force,
    /// Upgrade if server advertises
    Upgrade,
    /// Disabled
    Disabled,
}

/// Default configuration
fn default_sts() -> StsPolicy {
    StsPolicy::Upgrade
}

impl Default for ServerConfig {
    fn default() -> Self {
        Self {
            host: String::new(),
            port: 6667,
            tls: false,
            tls_skip_verify: false,
            sasl: None,
            requested_caps: vec![
                "account-tag".to_string(),
                "away-notify".to_string(),
                "echo-message".to_string(),
                "extended-join".to_string(),
                "multi-prefix".to_string(),
                "server-time-3.2".to_string(),
            ],
            sts_policy: StsPolicy::Upgrade,
            av_auto_enable: false,
        }
    }
}
```

---

## Testing

### Protocol Compliance Tests

```rust
// tests/ircv3_compliance.rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_capability_negotiation() {
        let server = MockServer::new();
        server.send(":server CAP * LS :sasl multi-prefix away-notify");
        
        let mut detector = CapabilityDetector::new();
        detector.process(":server CAP * LS :sasl multi-prefix away-notify");
        
        assert!(detector.has("sasl"));
        assert!(detector.has("multi-prefix"));
        assert!(detector.has("away-notify"));
    }

    #[test]
    fn test_sasl_plain_auth() {
        let mut ctx = SaslContext::new(SaslMechanism::Plain);
        let creds = Credentials {
            username: "testuser".to_string(),
            password: "testpass".to_string(),
            authorization_id: None,
        };
        
        let response = ctx.create_response(&creds).unwrap();
        let decoded = base64::decode(&response).unwrap();
        let parts: Vec<&str> = decoded.split('\0').collect();
        
        assert_eq!(parts.len(), 3);
        assert_eq!(parts[0], "testuser");
        assert_eq!(parts[1], "testuser");
        assert_eq!(parts[2], "testpass");
    }

    #[test]
    fn test_message_tag_escaping() {
        let input = "hello;world\\test";
        let escaped = escape_tag_value(input);
        assert_eq!(escaped, "hello\\:world\\\\test");
        
        let unescaped = unescape_tag_value(&escaped);
        assert_eq!(unescaped, input);
    }

    #[test]
    fn test_server_time_parsing() {
        let time = ServerTime::parse("2024-08-30T12:34:56.123456Z").unwrap();
        assert_eq!(time.timestamp.year(), 2024);
        assert_eq!(time.timestamp.month(), 8);
        assert_eq!(time.timestamp.day(), 30);
        assert_eq!(time.microseconds, Some(123456));
    }

    #[test]
    fn test_extended_join_parsing() {
        let raw = "nick!user@host JOIN #channel account :Real Name";
        let info = ExtendedJoinInfo::parse(raw).unwrap();
        
        assert_eq!(info.nick, "nick");
        assert_eq!(info.user, "user");
        assert_eq!(info.host, "host");
        assert_eq!(info.account, AccountStatus::LoggedIn("account".to_string()));
    }

    #[test]
    fn test_av_session_requires_tls() {
        let session = AvSession::new(AvStreamType::Audio, AvCodec::Opus);
        assert_eq!(session.state, AvSessionState::Created);
        // Note: Actual TLS enforcement tested in integration tests
    }
}
```

---

## Migration Phase Integration

Add IRCv3 work to existing migration phases:

### Phase 1: Core Protocol (Weeks 3-6) — UPDATED

| Task | Description | Priority |
|------|-------------|----------|
| 1.4 | Implement CAP negotiation framework | P0 |
| 1.5 | Implement SASL authentication (PLAIN, SCRAM-SHA-256) | P0 |
| 1.6 | Implement Message Tags parsing/generation | P0 |
| 1.7 | Implement Echo Message | P0 |
| 1.8 | Implement Server Time | P0 |

### Phase 2: Channel & User Management (Weeks 7-10) — UPDATED

| Task | Description | Priority |
|------|-------------|----------|
| 2.3 | Implement Extended Join | P1 |
| 2.4 | Implement Account Tracking (account-tag, account-notify) | P1 |
| 2.5 | Implement Away Notify | P1 |
| 2.6 | Implement Multi-prefix | P1 |
| 2.7 | Implement Monitor system | P1 |

### Phase 5: User Interface (Weeks 19-22) — UPDATED

| Task | Description | Priority |
|------|-------------|----------|
| 5.5 | Display account status badges | P1 |
| 5.6 | Display bot mode indicator | P2 |
| 5.7 | Show server timestamps | P0 |
| 5.8 | Message reply threading (client-only tags) | P3 |

### Phase X: Audio/Video Extensions (New)

| Task | Description | Priority |
|------|-------------|----------|
| X.1 | Implement AVCAPS capability | P4 |
| X.2 | Implement AVSYNC/AVQUIT commands | P4 |
| X.3 | Implement DCC STREAM/ACCEPT | P4 |
| X.4 | TLS 1.3 encryption for A/V | P4 |
| X.5 | A/V stream handling UI | P4 |

---

## Backward Compatibility

### Feature Detection Flow

```
1. Connect to server
2. Send: CAP LS 302
3. Parse: CAP * LS :<capabilities>
4. Store capabilities
5. If SASL available and credentials configured:
   a. CAP REQ :sasl
   b. AUTHENTICATE <mechanism>
   c. Complete SASL handshake
6. CAP END
7. NICK/USER registration
8. Request channels with JOIN
9. Use available capabilities:
   - If extended-join: Full user info on JOIN
   - If multi-prefix: All prefixes in NAMES
   - If away-notify: Track away status
   - etc.
```

### Graceful Degradation

| Feature | With Extension | Without Extension |
|---------|----------------|------------------|
| SASL | Authenticated before JOIN | Authenticated via NickServ |
| Extended Join | Full user info on JOIN | WHO after JOIN |
| Account Tags | See account on messages | Check /WHOIS |
| Away Notify | Real-time updates | Periodic ISON |
| Server Time | Accurate timestamps | Local receive time |
| Multi-prefix | All statuses visible | Highest status only |
| Message Tags | Metadata available | Ignored |

---

## Security Considerations

### SASL Security

| Mechanism | Security Level | Notes |
|------------|---------------|-------|
| PLAIN | Low | Transmits password (use TLS) |
| EXTERNAL | High | Certificate-based |
| SCRAM-SHA-256 | High | Challenge-response, no password transmission |

### A/V Stream Security (IETF Draft)

| Requirement | Implementation |
|-------------|----------------|
| TLS 1.3 Mandatory | `rustls` with TLS 1.3 only |
| DoS Prevention | Bandwidth limiting, rate limiting |
| Content Validation | Sanitize all received media data |
| User Consent | Require explicit user approval for A/V |

### STS Policy

- Store STS policy in configuration
- Auto-upgrade to TLS on reconnect
- Reject connections if TLS unavailable
- Support STS port mapping

---

## References

### IRCv3 Specifications

| Spec | URL |
|------|-----|
| Capability Negotiation | https://ircv3.net/specs/extensions/capability-negotiation |
| Message Tags | https://ircv3.net/specs/extensions/message-tags |
| SASL 3.1 | https://ircv3.net/specs/extensions/sasl-3.1 |
| SASL 3.2 | https://ircv3.net/specs/extensions/sasl-3.2 |
| Account Notify | https://ircv3.net/specs/extensions/account-notify |
| Account Tag | https://ircv3.net/specs/extensions/account-tag |
| Extended Join | https://ircv3.net/specs/extensions/extended-join |
| Away Notify | https://ircv3.net/specs/extensions/away-notify |
| Echo Message | https://ircv3.net/specs/extensions/echo-message |
| Server Time | https://ircv3.net/specs/extensions/server-time |
| STS | https://ircv3.net/specs/extensions/sts |
| Multi-prefix | https://ircv3.net/specs/extensions/multi-prefix |
| WHOX | https://ircv3.net/specs/extensions/whox |
| Bot Mode | https://ircv3.net/specs/extensions/bot-mode |
| Labeled Responses | https://ircv3.net/specs/extensions/labeled-response |
| Monitor | https://ircv3.net/specs/extensions/monitor |
| Chat History | https://ircv3.net/specs/extensions/chathistory |

### IETF Draft

| Draft | URL |
|-------|-----|
| IRC v3 (A/V) | https://www.ietf.org/archive/id/draft-samuelson-irc-v3-00.html |

### RFCs

| RFC | Description |
|-----|-------------|
| RFC 1459 | Original IRC protocol |
| RFC 2810-2813 | IRC architecture and commands |
| RFC 4616 | SASL PLAIN mechanism |
| RFC 4422 | SASL framework |
| RFC 7677 | SCRAM-SHA-256 |
| RFC 8314 | TLS for email |
| RFC 2119 | Requirement keywords |
