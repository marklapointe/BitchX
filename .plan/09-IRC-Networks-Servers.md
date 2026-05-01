# BitchX IRC Networks & Servers Database

> **Author:** Mark LaPointe <mark@cloudbsd.org>
>
> **Sponsorship:** None

## 1. Overview

This document describes the design for an IRC network and server database, including connection properties, network definitions, and integration with common IRC services (NickServ, ChanServ).

### 1.1 Goals

1. **Comprehensive server list** — Include major networks with all connection properties
2. **Multiple protocols** — Support IPv4, IPv6, SSL/TLS connections
3. **Service integration** — Native support for NickServ and ChanServ
4. **Auto-authentication** — Store credentials for automatic identification
5. **Auto-registration** — Automatically register nicknames when available
6. **Channel management** — Support ChanServ operations (AKICK, access lists)

---

## 2. Data Models

### 2.1 Server Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Server {
    /// Server hostname (e.g., "irc.libera.chat")
    pub hostname: String,
    
    /// Server port
    pub port: u16,
    
    /// Connection type
    pub connection_type: ConnectionType,
    
    /// IP version preference
    pub ip_version: IpVersion,
    
    /// Connection password (server password, if required)
    pub password: Option<String>,
    
    ///bind_address: Option<String>,
    
    /// Connection timeout in seconds
    pub timeout: u64,
    
    /// Retry count (-1 for infinite)
    pub retries: i32,
    
    /// Retry delay in seconds
    pub retry_delay: u64,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ConnectionType {
    /// Plaintext connection (no encryption)
    Plaintext,
    /// TLS/SSL connection
    Tls,
    /// TLS with certificate verification disabled
    TlsInsecure,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum IpVersion {
    /// Prefer IPv4
    Ipv4Only,
    /// Prefer IPv6
    Ipv6Only,
    /// Prefer IPv4, fall back to IPv6
    PreferIpv4,
    /// Prefer IPv6, fall back to IPv4
    PreferIpv6,
    /// Use system default
    Auto,
}
```

### 2.2 Network Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Network {
    /// Unique network identifier
    pub id: String,
    
    /// Human-readable network name
    pub name: String,
    
    /// Network description
    pub description: String,
    
    /// Primary servers (tried first)
    pub primary_servers: Vec<Server>,
    
    /// Backup servers (used if primary fails)
    pub backup_servers: Vec<Server>,
    
    /// NickServ configuration
    pub nickserv: NickServConfig,
    
    /// ChanServ configuration
    pub chanserv: ChanServConfig,
    
    /// Network-specific settings
    pub settings: NetworkSettings,
    
    /// Associated channels (auto-join)
    pub auto_join_channels: Vec<AutoJoinChannel>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NickServConfig {
    /// NickServ nickname (usually "NickServ")
    pub nickname: String,
    
    /// Whether to identify automatically
    pub auto_identify: bool,
    
    /// NickServ command format
    pub identify_command: String,
    
    /// Delay before sending IDENTIFY (ms)
    pub identify_delay: u64,
    
    /// Stored password (encrypted)
    pub password: Option<String>,
    
    /// Alternative NickServ nicknames (fallback)
    pub fallback_nicks: Vec<String>,
    
    /// SASL support
    pub sasl_enabled: bool,
    
    /// SASL mechanism
    pub sasl_mechanism: SaslMechanism,
    
    /// Auto-register nickname
    pub auto_register: bool,
    
    /// Registration email
    pub register_email: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SaslMechanism {
    Plain,
    ScramSha256,
    EcdsaNist256pChallenge,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChanServConfig {
    /// ChanServ nickname (usually "ChanServ")
    pub nickname: String,
    
    /// Auto-identify to ChanServ
    pub auto_identify: bool,
    
    /// ChanServ password (may differ from NickServ)
    pub password: Option<String>,
    
    /// Default channel modes to set on join
    pub default_modes: Vec<ChannelMode>,
    
    /// AKICK management
    pub akick_enabled: bool,
    
    /// Alternative ChanServ nicknames
    pub fallback_nicks: Vec<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NetworkSettings {
    /// Maximum nick length
    pub max_nick_length: usize,
    
    /// Maximum channel name length
    pub max_channel_length: usize,
    
    /// Maximum message length (usually 512)
    pub max_message_length: usize,
    
    /// Maximum topic length
    pub max_topic_length: usize,
    
    /// Supports halfop (+h)
    pub supports_halfop: bool,
    
    /// Supports admin (+a)
    pub supports_admin: bool,
    
    /// Supports owner (+q)
    pub supports_owner: bool,
    
    /// Supports channel wallops (+z)
    pub supports_wallops: bool,
    
    /// Supports cap notify
    pub supports_cap_notify: bool,
    
    /// Supports echo message (IRCv3)
    pub supports_echo_message: bool,
    
    /// Supports server time (IRCv3)
    pub supports_server_time: bool,
    
    /// Supports account notify (IRCv3)
    pub supports_account_notify: bool,
    
    /// Supports away notify (IRCv3)
    pub supports_away_notify: bool,
    
    /// Supports multi-prefix (IRCv3)
    pub supports_multi_prefix: bool,
    
    /// Supports sasl (IRCv3)
    pub supports_sasl: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AutoJoinChannel {
    /// Channel name (e.g., "##linux")
    pub channel: String,
    
    /// Channel password (key), if required
    pub password: Option<String>,
    
    /// Auto-join only after identified
    pub after_identify: bool,
    
    /// Auto-join delay in seconds
    pub delay: u64,
}
```

---

## 3. Default Network Database

### 3.1 Libera.Chat

```toml
[[networks]]
id = "libera"
name = "Libera.Chat"
description = "General purpose IRC network"

[[networks.servers]]
hostname = "irc.libera.chat"
port = 6697
connection_type = "Tls"
ip_version = "PreferIpv6"

[[networks.servers]]
hostname = "irc.libera.chat"
port = 6667
connection_type = "Plaintext"
ip_version = "PreferIpv6"

[[networks.servers]]
hostname = "ipv6.libera.chat"
port = 6697
connection_type = "Tls"
ip_version = "Ipv6Only"

[[networks.nickserv]]
nickname = "NickServ"
auto_identify = true
identify_command = "IDENTIFY {nick} {password}"
identify_delay = 500
sasl_enabled = true
sasl_mechanism = "ScramSha256"
auto_register = true

[[networks.chanserv]]
nickname = "ChanServ"
auto_identify = true
default_modes = ["n", "t"]

[[networks.settings]]
max_nick_length = 30
max_channel_length = 64
max_message_length = 512
max_topic_length = 390
supports_halfop = true
supports_admin = false
supports_owner = false
supports_cap_notify = true
supports_echo_message = true
supports_server_time = true
supports_account_notify = true
supports_away_notify = true
supports_multi_prefix = true
supports_sasl = true

[[networks.auto_join_channels]]
channel = "#libera"
after_identify = true
```

### 3.2 EFnet

```toml
[[networks]]
id = "efnet"
name = "EFnet"
description = "Oldest public IRC network"

[[networks.servers]]
hostname = "irc.efnet.org"
port = 6697
connection_type = "Tls"
ip_version = "PreferIpv4"

[[networks.servers]]
hostname = "efnet.dig Administrators"
port = 6667
connection_type = "Plaintext"
ip_version = "PreferIpv4"

[[networks.servers]]
hostname = "irc2.efnet.org"
port = 6660
connection_type = "Plaintext"
ip_version = "Auto"

[[networks.nickserv]]
nickname = "NickServ"
auto_identify = false
sasl_enabled = false

[[networks.chanserv]]
nickname = "ChanServ"
auto_identify = false

[[networks.settings]]
max_nick_length = 30
max_channel_length = 64
max_message_length = 512
max_topic_length = 80
supports_halfop = true
supports_admin = false
supports_owner = false
supports_cap_notify = false
supports_echo_message = false
supports_server_time = false
supports_account_notify = false
supports_away_notify = false
supports_multi_prefix = false
supports_sasl = false

[[networks.auto_join_channels]]
channel = "#efnet"
```

### 3.3 OFTC

```toml
[[networks]]
id = "oftc"
name = "OFTC"
description = "Open and Free Tech Community"

[[networks.servers]]
hostname = "irc.oftc.net"
port = 6697
connection_type = "Tls"
ip_version = "Auto"

[[networks.servers]]
hostname = "irc.oftc.net"
port = 6667
connection_type = "Plaintext"
ip_version = "Auto"

[[networks.servers]]
hostname = "ipv6.oftc.net"
port = 6697
connection_type = "Tls"
ip_version = "Ipv6Only"

[[networks.nickserv]]
nickname = "NickServ"
auto_identify = true
identify_command = "IDENTIFY {password}"
identify_delay = 1000
sasl_enabled = true
sasl_mechanism = "Plain"
auto_register = true

[[networks.chanserv]]
nickname = "ChanServ"
auto_identify = true
default_modes = ["n", "t"]

[[networks.settings]]
max_nick_length = 30
max_channel_length = 64
max_message_length = 512
max_topic_length = 160
supports_halfop = true
supports_admin = false
supports_owner = false
supports_cap_notify = true
supports_echo_message = true
supports_server_time = true
supports_account_notify = true
supports_away_notify = true
supports_multi_prefix = true
supports_sasl = true
```

### 3.4 Rizon

```toml
[[networks]]
id = "rizon"
name = "Rizon"
description = "Gaming and anime focused network"

[[networks.servers]]
hostname = "irc.rizon.net"
port = 6697
connection_type = "Tls"
ip_version = "Auto"

[[networks.servers]]
hostname = "irc.rizon.net"
port = 6660
connection_type = "Plaintext"
ip_version = "Auto"

[[networks.servers]]
hostname = "irc.rizon.net"
port = 6669
connection_type = "TlsInsecure"
ip_version = "Auto"

[[networks.nickserv]]
nickname = "NickServ"
auto_identify = true
identify_command = "IDENTIFY {nick} {password}"
identify_delay = 500
sasl_enabled = true
sasl_mechanism = "Plain"
auto_register = true
register_email = "nick@rizon.gg"

[[networks.chanserv]]
nickname = "ChanServ"
auto_identify = true
default_modes = ["n", "t", "s"]

[[networks.settings]]
max_nick_length = 30
max_channel_length = 64
max_message_length = 512
max_topic_length = 250
supports_halfop = true
supports_admin = false
supports_owner = false
supports_cap_notify = true
supports_echo_message = true
supports_server_time = true
supports_account_notify = true
supports_away_notify = true
supports_multi_prefix = true
supports_sasl = true
```

### 3.5 DALnet

```toml
[[networks]]
id = "dalnet"
name = "DALnet"
description = "Historical network with global presence"

[[networks.servers]]
hostname = "irc.dal.net"
port = 6697
connection_type = "Tls"
ip_version = "Auto"

[[networks.servers]]
hostname = "irc.dal.net"
port = 6667
connection_type = "Plaintext"
ip_version = "Auto"

[[networks.servers]]
hostname = "us.dal.net"
port = 6667
connection_type = "Plaintext"
ip_version = "Ipv4Only"

[[networks.nickserv]]
nickname = "NickServ"
auto_identify = true
identify_command = "IDENTIFY {password}"
identify_delay = 500
sasl_enabled = false
auto_register = true

[[networks.chanserv]]
nickname = "ChanServ"
auto_identify = true
default_modes = ["n", "t"]

[[networks.settings]]
max_nick_length = 30
max_channel_length = 64
max_message_length = 512
max_topic_length = 120
supports_halfop = true
supports_admin = false
supports_owner = false
supports_cap_notify = false
supports_echo_message = false
supports_server_time = false
supports_account_notify = false
supports_away_notify = false
supports_multi_prefix = false
supports_sasl = false
```

### 3.6 UnrealIRCd (Test Network)

```toml
[[networks]]
id = "unreal"
name = "UnrealIRCd Test"
description = "UnrealIRCd development/test server"

[[networks.servers]]
hostname = "localhost"
port = 6697
connection_type = "Tls"
ip_version = "Auto"

[[networks.servers]]
hostname = "127.0.0.1"
port = 6667
connection_type = "Plaintext"
ip_version = "Ipv4Only"

[[networks.nickserv]]
nickname = "NickServ"
auto_identify = false
sasl_enabled = true
sasl_mechanism = "ScramSha256"

[[networks.chanserv]]
nickname = "ChanServ"
auto_identify = false

[[networks.settings]]
max_nick_length = 30
max_channel_length = 64
max_message_length = 512
max_topic_length = 360
supports_halfop = true
supports_admin = true
supports_owner = true
supports_cap_notify = true
supports_echo_message = true
supports_server_time = true
supports_account_notify = true
supports_away_notify = true
supports_multi_prefix = true
supports_sasl = true
```

---

## 4. NickServ Integration

### 4.1 Auto-Identify Flow

```rust
async fn handle_nickserv_events(state: &mut IrcState) {
    loop {
        let event = state.nickserv_events.recv().await;
        
        match event {
            NickServEvent::MessageReceived { sender, message } => {
                if sender == "NickServ" {
                    handle_nickserv_message(state, message).await;
                }
            }
            NickServEvent::Registered => {
                // Nick registered successfully
                log_info("Nickname registered");
            }
            NickServEvent::Identified => {
                // Successfully identified
                log_info("Identified with NickServ");
                state.nickserv_identified = true;
                
                // Trigger auto-join channels
                if let Some(network) = &state.current_network {
                    join_auto_channels(state, &network.auto_join_channels, true).await;
                }
            }
            NickServEvent::RegistrationRequired => {
                // Nick is not registered
                if let Some(config) = &state.current_network.as_ref().and_then(|n| n.nickserv.auto_register) {
                    if config {
                        send_nickserv_register(state).await;
                    }
                }
            }
            NickServEvent::GhostDetected => {
                // Our nick is in use by a ghost
                send_nickserv_ghost(state).await;
            }
        }
    }
}

async fn handle_nickserv_message(state: &mut IrcState, message: &str) {
    let lower = message.to_lowercase();
    
    if lower.contains("this nick is registered") 
        || lower.contains("if this is your nick, /msg nickserv identify")
        || lower.contains("password accepted") 
        || lower.contains("you are now identified") 
    {
        state.nickserv_identified = true;
        on_identify_success(state).await;
    }
    else if lower.contains("nickname is registered")
        || lower.contains("this nick is already registered")
    {
        // Attempt auto-identify if not already
        if !state.nickserv_identified {
            send_nickserv_identify(state).await;
        }
    }
    else if lower.contains("ghost") && lower.contains("has been killed") {
        // Ghost was killed, try to reclaim nick
        send_nickserv_ghost(state).await;
    }
    else if lower.contains(" unregistered ") || lower.contains("no such nick") {
        state.nickserv_identified = false;
    }
}
```

### 4.2 Identify Commands

```rust
async fn send_nickserv_identify(state: &IrcState) {
    let network = state.current_network.as_ref()
        .expect("Connected to a network");
    let config = &network.nickserv;
    
    let password = state.credential_store.get_nickserv_password(&network.id)
        .expect("Password should be configured");
    
    let command = config.identify_command
        .replace("{nick}", &state.current_nick)
        .replace("{password}", &password);
    
    send_privmsg(state, &config.nickname, &command).await;
    
    // Wait for identify_delay
    tokio::time::sleep(Duration::from_millis(config.identify_delay)).await;
}
```

### 4.3 Auto-Registration

```rust
async fn send_nickserv_register(state: &IrcState) {
    let network = state.current_network.as_ref()
        .expect("Connected to a network");
    let config = &network.nickserv;
    
    if !config.auto_register {
        return;
    }
    
    let password = state.credential_store.generate_registration_password();
    let email = config.register_email.as_ref()
        .expect("Registration email required for auto-register");
    
    // Store password for later identification
    state.credential_store.set_nickserv_password(&network.id, &password);
    
    let command = format!("REGISTER {password} {email}", password = password, email = email);
    send_privmsg(state, &config.nickname, &command).await;
}
```

### 4.4 SASL Authentication

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SaslCredentials {
    pub account: String,
    pub password: String,
}

async fn perform_sasl_auth(state: &mut IrcState, mechanism: &SaslMechanism) -> Result<(), Error> {
    // CAP REQ sasl
    send_raw(state, "CAP REQ :sasl").await;
    
    // Wait for ACK
    let response = wait_for_cap_ack(state, "sasl").await?;
    
    // Send AUTHENTICATE with mechanism
    match mechanism {
        SaslMechanism::Plain => {
            send_raw(state, "AUTHENTICATE PLAIN").await;
            
            // Wait for + token
            let challenge = wait_for_authenticate_challenge(state).await?;
            
            // Encode credentials as PLAIN
            let credentials = format!("{}\0{}\0{}", 
                state.nickserv_account,
                state.nickserv_account,
                state.nickserv_password
            );
            let encoded = base64::Engine::encode(&base64::engine::general_purpose::STANDARD, credentials);
            
            send_raw(state, format!("AUTHENTICATE {}", encoded)).await;
        }
        SaslMechanism::ScramSha256 => {
            send_raw(state, "AUTHENTICATE SCRAM-SHA-256").await;
            
            // SCRAM-SHA-256 implementation
            let client_first = build_scram_client_first(&state.nickserv_account);
            let encoded = base64::Engine::encode(&base64::engine::general_purpose::STANDARD, &client_first);
            
            send_raw(state, format!("AUTHENTICATE {}", encoded)).await;
            
            // Handle server challenge and client response
            let server_challenge = wait_for_authenticate_challenge(state).await?;
            let client_final = build_scram_client_final(
                &server_challenge,
                &state.nickserv_account,
                &state.nickserv_password
            );
            let encoded = base64::Engine::encode(&base64::engine::general_purpose::STANDARD, &client_final);
            
            send_raw(state, format!("AUTHENTICATE {}", encoded)).await;
        }
        SaslMechanism::EcdsaNist256pChallenge => {
            send_raw(state, "AUTHENTICATE ECDSA-NIST256P-CHALLENGE").await;
            // Complex challenge-response protocol
            unimplemented!("ECDSA-NIST256P-CHALLENGE not implemented");
        }
    }
    
    // Wait for SASL result
    let result = wait_for_sasl_result(state).await?;
    
    if result == "SASL authentication successful" {
        state.sasl_authenticated = true;
        // Send CAP END
        send_raw(state, "CAP END").await;
        Ok(())
    } else {
        Err(Error::SaslFailed(result))
    }
}
```

---

## 5. ChanServ Integration

### 5.1 Auto-Join Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AutoJoinChannel {
    pub channel: String,
    pub password: Option<String>,  // Channel key if required
    pub after_identify: bool,      // Wait for NickServ identify
    pub delay: u64,                // Delay before joining (seconds)
}

impl Network {
    pub fn get_channels_to_join(&self, nickserv_identified: bool) -> Vec<AutoJoinChannel> {
        self.auto_join_channels.iter()
            .filter(|c| !c.after_identify || nickserv_identified)
            .cloned()
            .collect()
    }
}
```

### 5.2 Channel Mode Management

```rust
async fn apply_default_channel_modes(state: &IrcState, channel: &str) {
    let network = state.current_network.as_ref()
        .expect("Connected to a network");
    let config = &network.chanserv;
    
    for mode in &config.default_modes {
        match mode {
            ChannelMode::NoExternalMessages => {
                send_raw(state, format!("MODE {} +n", channel)).await;
            }
            ChannelMode::TopicLock => {
                send_raw(state, format!("MODE {} +t", channel)).await;
            }
            ChannelMode::InviteOnly => {
                send_raw(state, format!("MODE {} +i", channel)).await;
            }
            ChannelMode::Moderated => {
                send_raw(state, format!("MODE {} +m", channel)).await;
            }
            ChannelMode::Private => {
                send_raw(state, format!("MODE {} +p", channel)).await;
            }
            ChannelMode::Secret => {
                send_raw(state, format!("MODE {} +s", channel)).await;
            }
            ChannelMode::Key(ref key) => {
                send_raw(state, format!("MODE {} +k {}", channel, key)).await;
            }
            ChannelMode::Limit(limit) => {
                send_raw(state, format!("MODE {} +l {}", channel, limit)).await;
            }
            _ => {}
        }
        
        // Small delay between mode changes
        tokio::time::sleep(Duration::from_millis(100)).await;
    }
}
```

### 5.3 ChanServ Access List

```rust
#[derive(Debug, Clone)]
pub enum ChanServAccessLevel {
    /// No access (AKICK'd)
    Akick,
    /// Voice (+v)
    Voice,
    /// Halfop (+h)
    Halfop,
    /// Operator (+o)
    Op,
    /// Admin (+a) - UnrealIRCd
    Admin,
    /// Owner (+q) - UnrealIRCd
    Owner,
}

impl ChanServAccessLevel {
    pub fn from_str(s: &str) -> Option<Self> {
        match s.to_uppercase().as_str() {
            "V" | "VOICE" => Some(Self::Voice),
            "H" | "HALFOP" => Some(Self::Halfop),
            "O" | "OP" => Some(Self::Op),
            "A" | "ADMIN" => Some(Self::Admin),
            "Q" | "OWNER" => Some(Self::Owner),
            _ => None,
        }
    }
    
    pub fn as_mode(&self) -> &str {
        match self {
            Self::Voice => "v",
            Self::Halfop => "h",
            Self::Op => "o",
            Self::Admin => "a",
            Self::Owner => "q",
        }
    }
}

async fn check_chanserv_access(state: &IrcState, channel: &str) -> Option<ChanServAccessLevel> {
    let network = state.current_network.as_ref()?;
    let chanserv_nick = &network.chanserv.nickname;
    
    // SendChanServ ACCESS command
    send_privmsg(state, chanserv_nick, format!("ACCESS {} LIST", channel)).await;
    
    // Wait for access list response
    let response = wait_for_chanserv_response(state, "ACCESS").await?;
    
    // Parse access levels
    parse_chanserv_access_list(&response, &state.current_nick)
}
```

### 5.4 AKICK Management

```rust
async fn add_to_akick(state: &IrcState, channel: &str, mask: &str, reason: &str) -> Result<(), Error> {
    let network = state.current_network.as_ref()
        .ok_or(Error::NotConnected)?;
    let chanserv_nick = &network.chanserv.nickname;
    
    send_privmsg(state, chanserv_nick, format!(
        "AKICK {} + {} {}",
        channel,
        mask,
        reason
    )).await;
    
    // Wait for confirmation
    let response = wait_for_chanserv_response(state, "AKICK").await?;
    
    if response.contains("added to the AutoKick list") {
        Ok(())
    } else {
        Err(Error::ChanServError(response))
    }
}

async fn remove_from_akick(state: &IrcState, channel: &str, mask: &str) -> Result<(), Error> {
    let network = state.current_network.as_ref()
        .ok_or(Error::NotConnected)?;
    let chanserv_nick = &network.chanserv.nickname;
    
    send_privmsg(state, chanserv_nick, format!("AKICK {} - {}", channel, mask)).await;
    
    let response = wait_for_chanserv_response(state, "AKICK").await?;
    
    if response.contains("removed from the AutoKick list") {
        Ok(())
    } else {
        Err(Error::ChanServError(response))
    }
}
```

---

## 6. Credential Storage

### 6.1 Secure Storage Design

```rust
use argon2::{Argon2, PasswordHasher};
use aes_gcm::{
    aead::{Aead, KeyInit},
    Aes256Gcm, Nonce,
};
use rand::Rng;

const STORAGE_VERSION: u32 = 1;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CredentialStore {
    version: u32,
    /// Per-network credentials, encrypted
    networks: HashMap<String, EncryptedBlob>,
    /// Master key salt
    salt: String,
    /// Argon2 hash of master password
    master_hash: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct EncryptedBlob {
    /// Encrypted JSON blob
    ciphertext: Vec<u8>,
    /// GCM nonce
    nonce: Vec<u8>,
}

impl CredentialStore {
    /// Initialize a new credential store with a master password
    pub fn create(master_password: &str) -> Result<Self, Error> {
        let salt = generate_salt();
        let master_hash = hash_password(master_password, &salt)?;
        
        Ok(Self {
            version: STORAGE_VERSION,
            networks: HashMap::new(),
            salt,
            master_hash,
        })
    }
    
    /// Unlock an existing credential store
    pub fn unlock(&self, master_password: &str) -> Result<bool, Error> {
        let hash = hash_password(master_password, &self.salt)?;
        Ok(hash == self.master_hash)
    }
    
    /// Store a network password
    pub fn set_network_password(&mut self, network_id: &str, password: &str) -> Result<(), Error> {
        let blob = self.encrypt(password)?;
        self.networks.insert(network_id.to_string(), blob);
        Ok(())
    }
    
    /// Retrieve a network password
    pub fn get_network_password(&self, master_password: &str, network_id: &str) -> Result<Option<String>, Error> {
        let blob = match self.networks.get(network_id) {
            Some(b) => b,
            None => return Ok(None),
        };
        
        self.decrypt(master_password, blob)
            .map(Some)
    }
}
```

### 6.2 File Storage

```rust
impl CredentialStore {
    pub fn save(&self, path: &Path) -> Result<(), Error> {
        let json = serde_json::to_string_pretty(self)?;
        let mut file = File::create(path)?;
        file.write_all(json.as_bytes())?;
        
        // Set restrictive permissions (owner only)
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            let mut perms = file.metadata()?.permissions();
            perms.set_mode(0o600);
            file.set_permissions(perms)?;
        }
        
        Ok(())
    }
    
    pub fn load(path: &Path) -> Result<Self, Error> {
        let contents = fs::read_to_string(path)?;
        let store: CredentialStore = serde_json::from_str(&contents)?;
        
        if store.version > STORAGE_VERSION {
            return Err(Error::UnsupportedVersion(store.version));
        }
        
        Ok(store)
    }
}
```

---

## 7. Command-Line Interface

### 7.1 Server Commands

```irc
# Connect to a network
/server irc.libera.chat -ssl

# Connect with specific port
/server irc.libera.chat:6697 -ssl

# Connect to network by name
/server libera

# Disconnect from server
/disconnect

# Reconnect to current server
/reconnect

# Reconnect to specific server
/reconnect irc.libera.chat
```

### 7.2 Network Management

```irc
# List configured networks
/network list

# Add a network
/network add mynetwork

# Set network property
/network set mynetwork nickserv.auto_identify on

# Configure NickServ credentials
/network mynetwork nickserv set password secret123

# Add auto-join channel
/network mynetwork addchannel #mychannel

# Remove network
/network del mynetwork

# Export network list
/network export > networks.toml

# Import network list
/network import networks.toml
```

### 7.3 NickServ Commands

```irc
# Manual identify
/msg NickServ identify <password>

# Register nickname
/msg NickServ register <password> <email>

# Drop (unregister) nickname
/msg NickServ DROP <password>

# Ghost your nick
/msg NickServ GHOST <nick> <password>

# Recover nick
/msg NickServ RECOVER <nick> <password>

# Set password
/msg NickServ SET PASSWORD <newpassword>

# Info about a nick
/msg NickServ INFO <nick>
```

### 7.4 ChanServ Commands

```irc
# Register a channel
/msg ChanServ REGISTER #channel <password> <description>

# Set channel options
/msg ChanServ SET #channel MLOCK +nt
/msg ChanServ SET #channel TOPICLOCK ON

# Get channel info
/msg ChanServ INFO #channel

# Access management
/msg ChanServ ACCESS #channel LIST
/msg ChanServ ACCESS #channel ADD <nick> OP
/msg ChanServ ACCESS #channel DEL <nick>

# AutoKick management
/msg ChanServ AKICK #channel ADD <mask> <reason>
/msg ChanServ AKICK #channel DEL <mask>
/msg ChanServ AKICK #channel LIST
/msg ChanServ AKICK #channel EXCEPT ADD <mask>
/msg ChanServ AKICK #channel EXCEPT DEL <mask>

# Clear channel
/msg ChanServ CLEAR #channel BANS
/msg ChanServ CLEAR #channel OPS
/msg ChanServ CLEAR #channel VOICES
```

---

## 8. Configuration File Format

### 8.1 networks.toml Structure

```toml
# BitchX IRC Networks Configuration
# ~/.config/bitchx/networks.toml

# Global default settings
[defaults]
nick = "MyNick"
username = "myuser"
realname = "My Real Name"
bind_address = ""
auto_connect = false
auto_reconnect = true
reconnect_attempts = 10
reconnect_delay = 30

# Network definitions
[[networks]]
id = "libera"
name = "Libera.Chat"
description = "General purpose IRC network"

[[networks.servers]]
hostname = "irc.libera.chat"
port = 6697
connection_type = "Tls"
ip_version = "PreferIpv6"
password = ""
timeout = 30
retries = 5
retry_delay = 10

[[networks.servers]]
hostname = "irc.libera.chat"
port = 6667
connection_type = "Plaintext"
ip_version = "PreferIpv6"

[[networks.nickserv]]
nickname = "NickServ"
auto_identify = true
identify_command = "IDENTIFY {nick} {password}"
identify_delay = 500
sasl_enabled = true
sasl_mechanism = "ScramSha256"
auto_register = false
password = ""  # Stored separately in credentials store

[[networks.chanserv]]
nickname = "ChanServ"
auto_identify = false
password = ""

[[networks.settings]]
max_nick_length = 30
max_message_length = 512

[[networks.auto_join_channels]]
channel = "#libera"
password = ""
after_identify = true
delay = 2

# Another network
[[networks]]
id = "rizon"
name = "Rizon"
# ...
```

---

## 9. Migration Phase Integration

### 9.1 Task Dependencies

| Task ID | Task | Dependencies |
|---------|------|--------------|
| P6.1.1 | Create network/server data models | P0.1.1 (Project setup) |
| P6.1.2 | Implement network configuration parser | P6.1.1 |
| P6.1.3 | Add default network database | P6.1.2 |
| P6.1.4 | Implement credential storage | P6.1.1 |
| P6.1.5 | Implement NickServ integration | P1.4 (Server state) |
| P6.1.6 | Implement ChanServ integration | P6.1.5 |
| P6.1.7 | Add SASL authentication | P6.1.5 |
| P6.1.8 | Create network management commands | P6.1.2, P6.1.4 |
| P6.1.9 | Add auto-join channel support | P2.1 (Channel state) |
| P6.1.10 | Create network import/export | P6.1.2 |

### 9.2 Phase Assignment

Network and service integration is a **Phase 6** task:
- Phase 6: Services & Networks
- Runs after Phase 1 (Core Protocol) and Phase 2 (Channel/User Management)

---

## 10. Testing Strategy

### 10.1 Unit Tests

- Network configuration parsing
- Credential encryption/decryption
- SASL mechanism implementations
- NickServ message parsing
- ChanServ command generation

### 10.2 Integration Tests

- Connect to test network (localhost UnrealIRCd)
- NickServ auto-identify flow
- SASL authentication
- Auto-join channels after identify
- Channel mode application

### 10.3 Mock Tests

- Mock NickServ responses
- Mock ChanServ responses
- Network failure simulations
- SASL challenge/response

---

## 11. Security Considerations

### 11.1 Credential Protection

1. **Never store passwords in plaintext** — Always encrypt
2. **Master password required** — Credentials not accessible without master password
3. **Secure file permissions** — Credentials file readable only by owner (chmod 600)
4. **Memory clearing** — Zero out password strings after use
5. **No passwords in logs** — Redact credentials from all logging

### 11.2 Network Security

1. **Prefer TLS** — Warn when connecting without TLS
2. **Verify certificates** — Don't allow insecure TLS by default
3. **Certificate pinning** — Support for known server certificates
4. **No cleartext passwords** — Prefer SASL over plain text IDENTIFY

---

## 12. References

- [Libera.Chat Server Information](https://libera.chat/guides/connect)
- [OFTC Server Information](https://www.oftc.net/oftc/About)
- [IRC Services Documentation](https://github.com/atheme/atheme/wiki)
- [SASL PLAIN Authentication](https://ircv3.net/irc/sasl-3.1.html)
- [SASL SCRAM-SHA-256](https://ircv3.net/irc/sasl-3.2.html)
- [Argon2 Password Hashing](https://github.com/P-H-C/phc-winner-argon2)
