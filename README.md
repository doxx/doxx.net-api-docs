<p align="center">
  <img src="https://raw.githubusercontent.com/doxxcorp/style/main/logo-png/imagotype-white/imagotype-white-512.png" alt="doxx.net" width="300">
</p>

<p align="center">
  <strong>Freedom and Privacy by Design</strong>
</p>

<p align="center">
  <a href="https://a0x13.doxx.net">Portal</a> &middot;
  <a href="https://discord.gg/Gr9rByrEzZ">Discord</a> &middot;
  <a href="https://doxx.net/terms">Terms</a> &middot;
  <a href="https://doxx.net/privacy">Privacy</a>
</p>

---

# doxx.net API

All doxx.net services are accessible via three APIs:

| API | Base URL | Purpose |
|-----|----------|---------|
| **Config API** | `https://config.doxx.net/v1/` | Account, tunnels, DNS, domains, firewall, proxy, certificates |
| **Stats API** | `https://secure-wss.doxx.net` | Real-time bandwidth, security events, threat monitoring |
| **Conntrack API** | `wss://conntrack.doxx.net/ws` | Real-time connection tracking across all VPN nodes |

---

## Quick Start

### 1. Create an Account

Account creation requires human verification. **You cannot create accounts via the API directly.**

Visit **[https://a0x13.doxx.net](https://a0x13.doxx.net)** to:

1. Complete the DOXX POW (proof-of-work) challenge
2. Accept the Terms of Service
3. Receive your auth token

> Calling `create_account` without a valid POW token returns:
> ```json
> {
>   "status": "error",
>   "message": "Account creation requires human verification.",
>   "help": {
>     "steps": [
>       "1. Visit https://a0x13.doxx.net to create an account",
>       "2. Accept the Terms of Service",
>       "3. Save your auth token securely",
>       "4. Use the auth token as the 'token' parameter in all API requests"
>     ]
>   }
> }
> ```

### 2. Using Your Token

Your auth token is a URL-safe base64 string. Include it as `token` in every request:

```bash
curl -X POST https://config.doxx.net/v1/ \
  -d "list_tunnels=1&token=YOUR_AUTH_TOKEN"
```

### 3. Key Concepts

| Concept | Description |
|---------|-------------|
| **Auth Token** | Your account identity. Treat it like a password. |
| **Tunnel Token** | Identifies a specific VPN tunnel. Obtained from `list_tunnels`. |
| **POW Token** | One-time human verification token from DOXX POW challenge. |

---

# Config API

> **Base URL:** `https://config.doxx.net/v1/`
> **Method:** `POST` with `application/x-www-form-urlencoded` body
> **Endpoint selection:** Set `endpoint_name=1` as a parameter (e.g., `list_tunnels=1`)

### Regional Endpoints

| Endpoint | Region |
|----------|--------|
| `https://config.doxx.net/v1/` | Primary (anycast) |
| `https://config-us-east.doxx.net/v1/` | US East (IAD) |
| `https://config-us-west.doxx.net/v1/` | US West (LAX) |
| `https://config-eu-central.doxx.net/v1/` | EU Central (ZRH) |

---

## Authentication & Account

### `auth`

Validate a token and get account info.

```bash
curl -X POST https://config.doxx.net/v1/ -d "auth=1&token=YOUR_TOKEN"
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |

### `create_account`

Create a new account. Requires a valid DOXX POW token (human verification).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `doxxpow_token` | Yes | POW token from human verification challenge |

### `get_profile` / `update_profile`

Get or update account profile.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `email` | No | Email address (update only) |
| `name` | No | Display name (update only) |

### `delete_account`

Permanently delete your account and all associated data.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |

### `tos_status` / `accept_tos`

Check or accept Terms of Service.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |

### `create_account_recovery`

Generate recovery codes.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |

### `verify_account_recovery`

Recover account using a recovery code.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `recovery_code` | Yes | Recovery code |

### `merge_account`

Merge one account into another.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Destination account token |
| `source_token` | Yes | Account to merge from |

---

## VPN Tunnels

### `list_tunnels`

List all VPN tunnels.

```bash
curl -X POST https://config.doxx.net/v1/ -d "list_tunnels=1&token=YOUR_TOKEN"
```

**Response:**
```json
{
  "status": "success",
  "tunnels": [
    {
      "tunnel_token": "abc123...",
      "name": "My Laptop",
      "assigned_ip": "10.1.0.226/31",
      "assigned_v6": "2602:f5c1:1::1c0:8916/127",
      "server": "wireguard.mia.us.doxx.net",
      "firewall": 1,
      "ipv6_enabled": 1,
      "block_bad_dns": 1
    }
  ]
}
```

### `create_tunnel`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `name` | No | Tunnel name |
| `server` | Yes | Server hostname (from `servers`) |

### `create_tunnel_mobile`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `server` | Yes | Server hostname |
| `device_hash` | No | Device identifier hash |
| `device_type` | No | `mobile`, `desktop`, `server`, `web` |

### `update_tunnel`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `name` | No | New name |
| `server` | No | New server |
| `firewall` | No | `1` enable, `0` disable |
| `ipv6_enabled` | No | `1` enable, `0` disable |
| `block_bad_dns` | No | `1` enable DNS blocking, `0` disable |

### `delete_tunnel`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |

### `wireguard`

Get WireGuard configuration (keys, IPs, endpoint).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |

### `disconnect_peer`

Force disconnect a peer from the WireGuard server.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |

---

## DNS Blocking

### `dns_get_options`

List available DNS blocking categories. No auth required.

### `dns_get_tunnel_config`

Get DNS blocking config for a tunnel.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |

### `dns_set_subscription`

Enable/disable a DNS blocking category.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `subscription` | Yes | Category name |
| `enabled` | Yes | `1` or `0` |
| `apply_to_all` | No | `1` to apply to all tunnels |

### `dns_add_whitelist` / `dns_remove_whitelist`

Manage DNS whitelist (always allow).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `domain` | Yes | Domain |
| `apply_to_all` | No | `1` to apply to all tunnels |

### `dns_add_blacklist` / `dns_remove_blacklist`

Manage DNS blacklist (always block).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `domain` | Yes | Domain |
| `apply_to_all` | No | `1` to apply to all tunnels |

### `dns_blocklist_stats`

Get DNS blocking statistics.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |

---

## Firewall

### `firewall_rule_list`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | No | Filter by tunnel |

### `firewall_rule_add`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `protocol` | Yes | `TCP`, `UDP`, `ICMP`, `ALL` |
| `src_ip` | Yes | Source IP/CIDR |
| `src_port` | Yes | Source port or `ALL` |
| `dst_ip` | Yes | Destination IP (your tunnel IP) |
| `dst_port` | Yes | Destination port |

### `firewall_rule_delete`

Same parameters as `firewall_rule_add`.

### `firewall_link_all_toggle`

Enable/disable Link All Tunnels (mesh networking between your tunnels).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `enabled` | Yes | `1` or `0` |

### `firewall_link_all_status`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |

---

## Domain Management

### `list_domains`

```bash
curl -X POST https://config.doxx.net/v1/ -d "list_domains=1&token=YOUR_TOKEN"
```

**Response:**
```json
{
  "status": "success",
  "domains": [
    {"name": "mysite.doxx", "id": 1234}
  ]
}
```

### `create_domain`

Register a doxx.net domain. 196 TLDs available (`.doxx`, `.crypto`, `.vpn`, `.hack`, `.dao`, `.eth`, and more).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | Domain name (e.g., `mysite.doxx` or `mysite` for default `.doxx`) |

### `delete_domain`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | Domain to delete |

### `import_domain`

Import an external domain (`.com`, `.net`, `.org`, etc.) via TXT record verification.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | External domain |

### `get_domain_validation`

Get a validation code for domain import. Set it as a TXT record at `_doxx-verify.yourdomain.com`.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |

---

## DNS Records

Supported types: `A`, `AAAA`, `CNAME`, `MX`, `TXT`, `NS`, `SRV`, `PTR`

### `list_dns`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | Domain name |

### `create_dns_record`

```bash
curl -X POST https://config.doxx.net/v1/ \
  -d "create_dns_record=1&token=YOUR_TOKEN&domain=mysite.doxx&name=www.mysite.doxx&type=A&content=1.2.3.4&ttl=300"
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | Domain name |
| `name` | Yes | Record name (FQDN or `@` for apex) |
| `type` | Yes | Record type |
| `content` | Yes | Record value |
| `ttl` | No | TTL in seconds (default: 3600) |
| `prio` | No | Priority (MX/SRV) |

**SRV records** use additional parameters: `srv_priority`, `srv_weight`, `srv_port`, `srv_target`.

### `update_dns_record`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | Domain name |
| `old_name` | Yes | Current record name |
| `old_type` | Yes | Current record type |
| `old_content` | Yes | Current record content |
| `name` | Yes | New name |
| `content` | Yes | New content |
| `ttl` | Yes | New TTL |

### `delete_dns_record`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | Domain name |
| `name` | Yes | Record name |
| `type` | Yes | Record type |
| `content` | Yes | Record content |

---

## Public DNS (Secure DNS Sharing)

### `public_dns_list_hashes`

List your Secure DNS hashes. Each hash creates a `HASH.sdns.doxx.net` DoH/DoT endpoint.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |

### `public_dns_create_hash`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |

### `public_dns_delete_hash`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `host_hash` | Yes | Hash to delete |

---

## Proxy Configuration

### `get_proxy_config`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |

**Response:**
```json
{
  "status": "success",
  "config": {
    "tunnel_token": "...",
    "assigned_ip": "10.x.x.x",
    "assigned_v6": "2602:...",
    "enabled": true,
    "location": "newyork-us",
    "browser": null,
    "custom_lat": null,
    "custom_lon": null,
    "custom_timezone": null,
    "custom_language": null
  }
}
```

### `update_proxy_config`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `enabled` | No | `1` to enable |
| `location` | No | Location ID (default: `newyork-us`) |
| `browser` | No | Browser fingerprint |

---

## Certificate Signing

### `sign_certificate`

Sign a CSR with the doxx.net root CA. Automatically upgrades to wildcard.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | Domain (must be owned by you) |
| `csr` | Yes | PEM-encoded CSR |

**Returns:** Raw PEM certificate (not JSON).

```bash
openssl ecparam -genkey -name prime256v1 -out mysite.key
openssl req -new -key mysite.key -out mysite.csr -subj "/CN=mysite.doxx"

curl -X POST https://config.doxx.net/v1/ \
  -d "sign_certificate=1&token=YOUR_TOKEN&domain=mysite.doxx" \
  --data-urlencode "csr=$(cat mysite.csr)" -o mysite.crt
```

The signed certificate includes SAN entries for both `*.mysite.doxx` and `mysite.doxx`.

---

## Mobile Options (iOS)

### `get_mobile_options` / `set_mobile_options`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `connect_on_startup` | No | `1` or `0` (set only) |
| `kill_switch` | No | `1` or `0` (set only) |
| `proxy_enabled` | No | `1` or `0` (set only) |
| `onion_enabled` | No | `1` or `0` (set only) |

---

## Utility

### `servers`

List available VPN servers. **No auth required.**

```bash
curl -X POST https://config.doxx.net/v1/ -d "servers=1"
```

### `version_check`

Get current app version. **No auth required.**

### `generate_qr`

Generate a QR code PNG. **No auth required.**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `data` | Yes | Text to encode |
| `size` | No | Pixels (100-2048, default: 512) |

**Returns:** Binary PNG image.

---

## DOXX POW (Human Verification)

### `doxxpow_challenge`

Request a proof-of-work challenge. **No auth required.**

### `doxxpow_verify`

Submit a completed proof-of-work solution. **No auth required.**

### `doxxpow_validate_token`

Validate a POW token (internal use).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `pow_token` | Yes | POW token |

---

# Stats API

> **Base URL:** `https://secure-wss.doxx.net`
> Real-time VPN statistics, bandwidth monitoring, and security event streaming.

## WebSocket Stream

```
wss://secure-wss.doxx.net:443/ws?token=YOUR_TOKEN
```

Optional parameter: `tunnel_token` to filter to a single tunnel (mobile mode).

### Event Types

#### Threat Events

| Type | Description | Key Fields |
|------|-------------|------------|
| `dns_block` | Blocked DNS query | `domain`, `category` (ads/tracking/malware), `source` |
| `security_event` | Security alert | `category`, `service`, `port`, `protocol`, `src`, `dst` |
| `dangerous_port` | Connection to dangerous port | `service` (SSH/Telnet), `port`, `protocol`, `src`, `dst` |
| `dns_bypass` | DNS bypass attempt | `provider`, `dst` |
| `doh_bypass` | DoH bypass attempt | `provider`, `dst` |
| `doh_blocked` | DoH request blocked | `provider` |

#### Monitoring Events

| Type | Description | Key Fields |
|------|-------------|------------|
| `bandwidth` | Bandwidth usage (Mbps) | `value` (format: `in=X,out=Y`), `prefix` |
| `dns_nxdomain` | Non-existent domain query | `domain` |
| `tunnel_status` | Tunnel state change | `value` (sleeping/offline) |
| `port_scan` | Port scanning detected | `src`, `dst`, `port` |

### Event Structure

```json
{
  "tunnel_token": "...",
  "ts": 1707400000,
  "prefix": "10.1.0.226/31",
  "type": "dns_block",
  "action": "block",
  "category": "ads",
  "value": "doubleclick.net",
  "count": 5,
  "display": {
    "domain": "doubleclick.net",
    "source": "easylist",
    "reason": "advertising tracker"
  }
}
```

## REST Endpoints

### `GET /api/stats/bandwidth`

Historical bandwidth data with automatic granularity.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | No | Filter by tunnel |
| `start` | No | ISO 8601 start time (default: 1h ago) |
| `end` | No | ISO 8601 end time (default: now) |

**Response:**
```json
{
  "granularity": "1m",
  "data": [
    {
      "tunnel_token": "...",
      "timestamp": 1707400000,
      "peak_in": 125.5,
      "peak_out": 42.3,
      "samples": 60
    }
  ],
  "aggregate": [
    {
      "tunnel_token": "aggregate",
      "timestamp": 1707400000,
      "peak_in": 125.5,
      "peak_out": 42.3,
      "samples": 60
    }
  ]
}
```

Granularity auto-selects: `1s` (<5m), `1m` (<6h), `5m` (<48h), `1h` (<30d), `6h` (30d+).

### `GET /api/stats/alerts`

Historical security events and alerts.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | No | Filter by tunnel |
| `last` | No | Shortcut: `session`, `1m`, `1h`, `1d`, `7d`, `30d` |
| `start` | No | ISO 8601 start (alternative to `last`) |
| `end` | No | ISO 8601 end |
| `type` | No | Filter by event type |

**Response:**
```json
{
  "granularity": "1m",
  "totals": {"dns_block": 1234, "security_event": 5},
  "block_count": 1234,
  "category_counts": {"ads": 800, "tracking": 300, "malware": 134},
  "data": [
    {
      "tunnel_token": "...",
      "prefix": "10.1.0.226/31",
      "type": "dns_block",
      "action": "block",
      "value": "doubleclick.net",
      "timestamp": 1707400000,
      "count": 42,
      "last_seen": 1707403600
    }
  ]
}
```

### `GET /api/stats/summary`

Summary statistics over a time period.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | No | Filter by tunnel |
| `days` | No | Number of days (default: 30) |

### `GET /api/stats/global`

Global threat counter. **No auth required.**

```json
{"status": "success", "total": 1234567890, "ts": 1707400000}
```

### `wss://secure-wss.doxx.net/ws/global`

Public WebSocket for global threat counter (landing page widget). **No auth required.**

```json
{"type": "global_stats", "total": 1234567890, "ts": 1707400000}
```

---

# Conntrack API

> **Base URL:** `wss://conntrack.doxx.net/ws`
> Real-time connection tracking across all VPN backbone nodes.

## WebSocket Stream

```
wss://conntrack.doxx.net:443/ws?token=YOUR_TOKEN
```

### Message Types

#### `initial`

Sent immediately on connection. Contains all current connections.

#### `snapshot`

Sent every 10 seconds with updated connection data.

### Message Structure

```json
{
  "type": "snapshot",
  "timestamp": 1707400000,
  "connections": [
    {
      "id": "10.1.0.227:49823-93.184.216.34:443",
      "protocol": "tcp",
      "state": "ESTABLISHED",
      "src_ip": "10.1.0.227",
      "dst_ip": "93.184.216.34",
      "src_port": 49823,
      "dst_port": 443,
      "bytes_sent": 1234,
      "bytes_recv": 56789,
      "packets_sent": 12,
      "packets_recv": 45,
      "upload_speed": 1024.5,
      "download_speed": 8192.0,
      "server": "bh1.mia1.doxx.net",
      "tunnel_name": "My Laptop",
      "is_local": false
    }
  ],
  "stats": {
    "total_connections": 67,
    "total_upload": 1048576,
    "total_download": 10485760,
    "upload_speed": 5120.0,
    "download_speed": 40960.0,
    "protocol_breakdown": {"tcp": 60, "udp": 7}
  }
}
```

### Health Check

```
GET https://conntrack.doxx.net/health
```

```json
{
  "status": "healthy",
  "mode": "master",
  "clients": 3,
  "timestamp": 1707400000,
  "worker_connections": 4
}
```

---

## Error Responses

All errors follow this format:

```json
{
  "status": "error",
  "message": "Description of what went wrong"
}
```

| HTTP Code | Meaning |
|-----------|---------|
| 200 | Success |
| 400 | Bad request (missing/invalid parameters) |
| 401 | Unauthorized (invalid token) |
| 403 | Forbidden (POW required or permission denied) |
| 404 | Not found |
| 500 | Server error |

---

## Rate Limits

No strict rate limits currently enforced. Please be reasonable with automated requests.

---

## Support

- **Portal:** [a0x13.doxx.net](https://a0x13.doxx.net)
- **Discord:** [discord.gg/Gr9rByrEzZ](https://discord.gg/Gr9rByrEzZ)
- **Email:** support@doxx.net

---

<p align="center"><em>doxx.net - Freedom and Privacy by Design</em></p>
