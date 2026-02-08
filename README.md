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

## Overview

| API | Base URL | Purpose |
|-----|----------|---------|
| **Config API** | `https://config.doxx.net/v1/` | Account, tunnels, DNS, domains, firewall, proxy, certificates |
| **Stats API** | `https://secure-wss.doxx.net` | Real-time bandwidth, security events, threat monitoring |
| **Conntrack API** | `wss://conntrack.doxx.net/ws` | Real-time connection tracking across all VPN nodes |

**Config API** uses `POST` with `application/x-www-form-urlencoded`. Endpoints are selected by setting `endpoint_name=1` as a parameter.

**Regional failover endpoints:**
- `https://config-us-east.doxx.net/v1/`
- `https://config-us-west.doxx.net/v1/`
- `https://config-eu-central.doxx.net/v1/`

---

## Authentication

doxx.net uses token-based auth. No usernames, no passwords, no email.

| Token Type | What It Is | How You Get It |
|------------|-----------|----------------|
| **Auth Token** | Your account identity. ~43 char base64 string. | Human creates account at [a0x13.doxx.net](https://a0x13.doxx.net) |
| **Tunnel Token** | Identifies a specific VPN tunnel. | Returned by `list_tunnels` or `create_tunnel` |
| **POW Token** | One-time human verification. | DOXX POW challenge at account creation |

**You cannot create accounts via API.** A human must visit [a0x13.doxx.net](https://a0x13.doxx.net), complete the proof-of-work challenge, and accept the Terms of Service. The auth token from that process is then used for all API calls.

---

## Common Workflows

### Workflow 1: Set Up a VPN Tunnel

```bash
TOKEN="your_auth_token_here"
API="https://config.doxx.net/v1/"

# Step 1: List available servers
curl -s -X POST $API -d "servers=1" | jq '.servers[] | {server_name, location, description}'

# Step 2: Create a tunnel
curl -s -X POST $API -d "create_tunnel=1&token=$TOKEN&name=My+Laptop&server=wireguard.mia.us.doxx.net" | jq .

# Step 3: List your tunnels (get tunnel_token)
curl -s -X POST $API -d "list_tunnels=1&token=$TOKEN" | jq '.tunnels[] | {tunnel_token, name, assigned_ip, server}'

# Step 4: Get WireGuard config
curl -s -X POST $API -d "wireguard=1&token=$TOKEN&tunnel_token=TUNNEL_TOKEN_HERE" | jq .config
```

### Workflow 2: Register a Domain and Add DNS Records

```bash
# Step 1: Register domain
curl -s -X POST $API -d "create_domain=1&token=$TOKEN&domain=mysite.doxx" | jq .

# Step 2: Add an A record
curl -s -X POST $API -d "create_dns_record=1&token=$TOKEN&domain=mysite.doxx&name=mysite.doxx&type=A&content=1.2.3.4&ttl=300" | jq .

# Step 3: Add a wildcard
curl -s -X POST $API -d "create_dns_record=1&token=$TOKEN&domain=mysite.doxx&name=*.mysite.doxx&type=A&content=1.2.3.4&ttl=300" | jq .

# Step 4: Sign a TLS certificate
openssl ecparam -genkey -name prime256v1 -out mysite.key
openssl req -new -key mysite.key -out mysite.csr -subj "/CN=mysite.doxx"
curl -s -X POST $API -d "sign_certificate=1&token=$TOKEN&domain=mysite.doxx" --data-urlencode "csr=$(cat mysite.csr)" -o mysite.crt

# Step 5: Verify DNS is live
dig A mysite.doxx @a.root-dx.net +short
```

### Workflow 3: Configure DNS Blocking

```bash
# Step 1: See available blocklists
curl -s -X POST $API -d "dns_get_options=1" | jq '.options[] | {name, display_name, category, domain_count}'

# Step 2: Enable a blocklist on your tunnel
curl -s -X POST $API -d "dns_set_subscription=1&token=$TOKEN&tunnel_token=TUNNEL_TOKEN&subscription=ads&enabled=1" | jq .

# Step 3: Check tunnel DNS config
curl -s -X POST $API -d "dns_get_tunnel_config=1&token=$TOKEN&tunnel_token=TUNNEL_TOKEN" | jq .

# Step 4: Add a custom whitelist entry
curl -s -X POST $API -d "dns_add_whitelist=1&token=$TOKEN&tunnel_token=TUNNEL_TOKEN&domain=example.com" | jq .
```

### Workflow 4: Monitor Your Network (Stats API)

```bash
# Real-time events via WebSocket
websocat "wss://secure-wss.doxx.net/ws?token=$TOKEN"

# Historical bandwidth (last hour)
curl -s "https://secure-wss.doxx.net/api/stats/bandwidth?token=$TOKEN&start=$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ)&end=$(date -u +%Y-%m-%dT%H:%M:%SZ)" | jq .

# Security alerts (last 24h)
curl -s "https://secure-wss.doxx.net/api/stats/alerts?token=$TOKEN&last=1d" | jq '.totals'

# Real-time connections via WebSocket
websocat "wss://conntrack.doxx.net/ws?token=$TOKEN"
```

---

## Error Handling

All errors return:

```json
{
  "status": "error",
  "message": "Description of what went wrong"
}
```

| HTTP Code | Meaning | What To Do |
|-----------|---------|------------|
| 200 | Success | Parse `status` field ("success" or "error") |
| 400 | Missing/invalid parameter | Check required parameters |
| 401 | Invalid or missing token | Verify your auth token |
| 403 | Forbidden | POW required or wrong owner |
| 404 | Not found | Resource doesn't exist |
| 500 | Server error | Retry or contact support |
| 503 | Service degraded | Try a different regional endpoint |

**Important:** HTTP 200 can still contain `"status": "error"` in the JSON body. Always check the `status` field.

---

# Config API Reference

## Account

### `auth`

```bash
curl -s -X POST $API -d "auth=1&token=$TOKEN"
```

```json
{"status": "success", "message": "Authentication successful"}
```

### `tos_status`

```bash
curl -s -X POST $API -d "tos_status=1&token=$TOKEN"
```

```json
{"status": "success", "tos_accepted": true, "accepted_at": "2026-01-15 10:00:00", "version": "1.0"}
```

### `accept_tos`

```bash
curl -s -X POST $API -d "accept_tos=1&token=$TOKEN"
```

```json
{"status": "success", "message": "Terms of Service accepted"}
```

### `get_profile`

```bash
curl -s -X POST $API -d "get_profile=1&token=$TOKEN"
```

```json
{
  "status": "success",
  "profile": {
    "recovery_email": null,
    "recovery_phone": null,
    "email_notifications": 0,
    "sms_notifications": 0,
    "created_at": "2025-06-01 12:00:00",
    "updated_at": "2026-02-08 10:00:00"
  },
  "recovery_codes_count": 10
}
```

### `update_profile`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `email` | No |
| `name` | No |

### `create_account_recovery`

```bash
curl -s -X POST $API -d "create_account_recovery=1&token=$TOKEN"
```

```json
{
  "status": "success",
  "message": "Recovery codes generated successfully",
  "codes": ["abc123", "def456", "..."],
  "set_id": "set_abc",
  "created_at": "2026-02-08T19:00:00Z"
}
```

### `verify_account_recovery`

| Parameter | Required |
|-----------|----------|
| `recovery_code` | Yes |

```json
{"status": "success", "message": "Account recovery successful", "new_token": "new_token_here", "user_id": 123}
```

### `delete_account`

```bash
curl -s -X POST $API -d "delete_account=1&token=$TOKEN"
```

```json
{"status": "success", "message": "Account deleted successfully"}
```

### `merge_account`

| Parameter | Required |
|-----------|----------|
| `token` | Yes (destination) |
| `source_token` | Yes (to merge from) |

```json
{
  "status": "success",
  "new_token": "master_token",
  "master_user_id": 4,
  "merged_tunnels": 3,
  "merged_whitelist": 5,
  "merged_blacklist": 2,
  "message": "Account merged successfully. Please save the new token."
}
```

---

## Servers

### `servers`

No auth required.

```bash
curl -s -X POST $API -d "servers=1"
```

```json
{
  "status": "success",
  "servers": [
    {
      "server_name": "wireguard.mia.us.doxx.net",
      "location": "Miami, FL",
      "description": "US Southeast",
      "type": "wireguard",
      "public_key": "abc123...",
      "best_for": "US East Coast",
      "operator": "doxx.net",
      "bg_image": "miami.jpg",
      "flag_image": "us.svg",
      "continent": "NA"
    }
  ]
}
```

---

## Tunnels

### `list_tunnels`

```bash
curl -s -X POST $API -d "list_tunnels=1&token=$TOKEN"
```

```json
{
  "status": "success",
  "tunnels": [
    {
      "tunnel_token": "Eh1xwlLd...",
      "name": "My Laptop",
      "server": "wireguard.mia.us.doxx.net",
      "assigned_ip": "10.1.0.226/31",
      "assigned_v6": "2602:f5c1:1::1c0:8916/127",
      "public_key": "abc...",
      "private_key": "xyz...",
      "type": "wireguard",
      "device_hash": "",
      "device_type": "",
      "created_at": "2025-06-01T12:00:00Z",
      "block_bad_dns": 1,
      "firewall": 1,
      "ipv6_enabled": 1,
      "onion_enabled": 0,
      "proxy_enabled": 0,
      "is_connected": true,
      "connection_status": "connected"
    }
  ]
}
```

### `create_tunnel`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `name` | No | Tunnel name |
| `server` | Yes | Server hostname from `servers` endpoint |

```json
{"status": "success", "message": "Tunnel created successfully"}
```

### `create_tunnel_mobile`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `server` | Yes | Server hostname |
| `device_hash` | No | Device identifier |
| `device_type` | No | `mobile`, `desktop`, `server`, `web` |

```json
{
  "status": "success",
  "message": "Mobile tunnel created successfully",
  "tunnel_token": "new_token...",
  "server": "wireguard.mia.us.doxx.net",
  "assigned_ip": "10.1.2.3/31",
  "assigned_v6": "2602:f5c1:1::abc:1234/127",
  "public_key": "abc...",
  "private_key": "xyz..."
}
```

### `update_tunnel`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `name` | No | New name |
| `server` | No | New server |
| `firewall` | No | `1` or `0` |
| `ipv6_enabled` | No | `1` or `0` |
| `block_bad_dns` | No | `1` or `0` |

```json
{"status": "success", "message": "Tunnel updated successfully"}
```

### `delete_tunnel`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `tunnel_token` | Yes |

```json
{"status": "success", "message": "Tunnel deleted successfully"}
```

### `wireguard`

Get WireGuard configuration file data.

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `tunnel_token` | Yes |

```json
{
  "status": "success",
  "config": {
    "interface": {
      "private_key": "your_private_key",
      "address": "10.1.0.227/31, 2602:f5c1:1::1c0:8917/128",
      "dns": "10.10.10.10,fd53::"
    },
    "peer": {
      "public_key": "server_public_key",
      "allowed_ips": "0.0.0.0/0, ::/0",
      "endpoint": "wireguard.mia.us.doxx.net:51820",
      "persistent_keepalive": 25
    }
  }
}
```

### `disconnect_peer`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `tunnel_token` | Yes |

---

## DNS Blocking

### `dns_get_options`

No auth required.

```json
{
  "status": "success",
  "options": [
    {
      "name": "ads",
      "display_name": "Advertising",
      "description": "Block ad networks and trackers",
      "category": "privacy",
      "icon": "ad-icon",
      "domain_count": 150000,
      "default_enabled": true,
      "user_toggleable": true,
      "is_base_safety": false
    }
  ]
}
```

### `dns_get_tunnel_config`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `tunnel_token` | Yes |

```json
{
  "status": "success",
  "tunnel_token": "abc...",
  "dns_blocking_enabled": true,
  "base_protections": ["malware", "phishing"],
  "subscriptions": [
    {"blocklist_name": "ads", "enabled": 1}
  ],
  "whitelists": [
    {"domain": "example.com", "reason": null}
  ],
  "blacklists": [
    {"domain": "evil.com", "reason": "manual block"}
  ]
}
```

### `dns_set_subscription`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `subscription` | Yes | Blocklist name |
| `enabled` | Yes | `1` or `0` |
| `apply_to_all` | No | `1` to apply to all tunnels |

```json
{"status": "success", "message": "Subscription updated", "blocklist": "ads", "enabled": true, "tunnels_updated": 1}
```

### `dns_add_whitelist` / `dns_remove_whitelist`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `tunnel_token` | Yes |
| `domain` | Yes |
| `apply_to_all` | No |

### `dns_add_blacklist` / `dns_remove_blacklist`

Same parameters as whitelist.

### `dns_blocklist_stats`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |

```json
{
  "status": "success",
  "total_domains": 500000,
  "count": 12,
  "lists": [
    {
      "name": "ads",
      "display_name": "Advertising",
      "domain_count": 150000,
      "category": "privacy",
      "is_base_safety": false,
      "default_enabled": true,
      "enabled": true
    }
  ]
}
```

---

## Firewall

### `firewall_rule_list`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | No | Filter by tunnel |

```json
{
  "status": "success",
  "link_all_enabled": false,
  "rules": [
    {
      "tunnel_token": "abc...",
      "protocol": "TCP",
      "src_ip": "0.0.0.0/0",
      "src_port": "ALL",
      "dst_ip": "10.1.0.227",
      "dst_port": "443"
    }
  ],
  "count": 1
}
```

### `firewall_rule_add`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `protocol` | Yes | `TCP`, `UDP`, `ICMP`, `ALL` |
| `src_ip` | Yes | Source IP/CIDR |
| `src_port` | Yes | Port or `ALL` |
| `dst_ip` | Yes | Your tunnel IP |
| `dst_port` | Yes | Destination port |

```json
{
  "status": "success",
  "message": "Firewall rule created successfully",
  "rule": {"tunnel_token": "abc...", "protocol": "TCP", "src_ip": "0.0.0.0/0", "src_port": "ALL", "dst_ip": "10.1.0.227", "dst_port": "443", "enabled": 1}
}
```

### `firewall_rule_delete`

Same parameters as `firewall_rule_add`.

```json
{"status": "success", "message": "Firewall rule deleted successfully"}
```

### `firewall_link_all_toggle`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `enabled` | Yes (`1` or `0`) |

```json
{"status": "success", "message": "Link all enabled", "link_all_tunnels": 1, "rules_deleted": 0}
```

### `firewall_link_all_status`

```json
{"status": "success", "link_all_tunnels": 0}
```

---

## Domains

### `list_domains`

```json
{
  "status": "success",
  "domains": [
    {"name": "mysite.doxx", "id": 1234}
  ]
}
```

### `create_domain`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | e.g., `mysite.doxx` or `mysite` (defaults to `.doxx`) |

196 TLDs available: `.doxx`, `.crypto`, `.vpn`, `.hack`, `.dao`, `.eth`, `.dns`, `.tor`, `.onion`, `.cyber`, and more.

```json
{"status": "success", "message": "Domain registered successfully"}
```

### `delete_domain`

```json
{"status": "success", "message": "Domain deleted successfully"}
```

### `import_domain`

Import external domains (`.com`, `.net`, `.org`) via TXT record verification.

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `domain` | Yes |

```json
{
  "status": "success",
  "message": "Domain imported successfully",
  "nameservers": ["a.root-dx.net", "a.root-dx.com", "a.root-dx.org"],
  "note": "Update your domain registrar to use these nameservers"
}
```

### `get_domain_validation`

Get the TXT verification code. Set as `_doxx-verify.yourdomain.com` TXT record, then call `import_domain`.

```json
{"status": "success", "validation_code": "a1b2c3d4e5f6..."}
```

---

## DNS Records

Supported types: `A`, `AAAA`, `CNAME`, `MX`, `TXT`, `NS`, `SRV`, `PTR`

### `list_dns`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `domain` | Yes |

```json
{
  "status": "success",
  "domain": "mysite.doxx",
  "records": [
    {"name": "mysite.doxx", "type": "A", "content": "1.2.3.4", "ttl": 300, "prio": 0},
    {"name": "mysite.doxx", "type": "SOA", "content": "ns.doxx. hostmaster.doxx. 2026020801 10800 3600 604800 3600", "ttl": 3600, "prio": 0},
    {"name": "mysite.doxx", "type": "NS", "content": "ns.doxx.", "ttl": 3600, "prio": 0}
  ]
}
```

### `create_dns_record`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `domain` | Yes | Domain name |
| `name` | Yes | FQDN or `@` for apex |
| `type` | Yes | Record type |
| `content` | Yes | Record value |
| `ttl` | No | Default: 3600 |
| `prio` | No | Priority (MX) |

SRV records use: `srv_priority`, `srv_weight`, `srv_port`, `srv_target`

```json
{"status": "success", "message": "DNS record created successfully"}
```

### `update_dns_record`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `domain` | Yes |
| `old_name` | Yes |
| `old_type` | Yes |
| `old_content` | Yes |
| `name` | Yes |
| `content` | Yes |
| `ttl` | Yes |

```json
{"status": "success", "message": "DNS record updated successfully"}
```

### `delete_dns_record`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `domain` | Yes |
| `name` | Yes |
| `type` | Yes |
| `content` | Yes |

```json
{"status": "success", "message": "DNS record deleted successfully"}
```

---

## Public DNS (Secure DNS Sharing)

Create DoH/DoT endpoints that share your tunnel's DNS blocking config: `HASH.sdns.doxx.net`

### `public_dns_list_hashes`

```json
{
  "status": "success",
  "count": 1,
  "hashes": [
    {
      "host_hash": "gl6nqcbyhsau",
      "tunnel_token": "abc...",
      "label": "",
      "created_at": "2025-12-01 10:00:00",
      "tunnel_name": "My Laptop",
      "tunnel_server": "wireguard.mia.us.doxx.net",
      "doh_url": "https://gl6nqcbyhsau.sdns.doxx.net/dns-query",
      "dot_host": "gl6nqcbyhsau.sdns.doxx.net"
    }
  ]
}
```

### `public_dns_create_hash`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `tunnel_token` | Yes |

```json
{
  "status": "success",
  "host_hash": "gl6nqcbyhsau",
  "tunnel_token": "abc...",
  "doh_url": "https://gl6nqcbyhsau.sdns.doxx.net/dns-query",
  "dot_host": "gl6nqcbyhsau.sdns.doxx.net"
}
```

### `public_dns_delete_hash`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `host_hash` | Yes |

---

## Proxy

### `get_proxy_config`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `tunnel_token` | Yes |

```json
{
  "status": "success",
  "config": {
    "tunnel_token": "abc...",
    "assigned_ip": "10.1.0.226",
    "assigned_v6": "2602:f5c1:1::1c0:8916",
    "enabled": false,
    "location": "newyork-us",
    "browser": null,
    "custom_lat": null,
    "custom_lon": null
  }
}
```

### `update_proxy_config`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | Yes | Tunnel token |
| `enabled` | No | `1` to enable |
| `location` | No | Location ID |
| `browser` | No | Browser fingerprint |

```json
{"status": "success", "message": "Proxy configuration updated"}
```

---

## Certificates

### `sign_certificate`

Signs a CSR with the doxx.net root CA. Auto-upgrades to wildcard. **Returns raw PEM, not JSON.**

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `domain` | Yes (must own it) |
| `csr` | Yes (PEM-encoded) |

```bash
curl -s -X POST $API \
  -d "sign_certificate=1&token=$TOKEN&domain=mysite.doxx" \
  --data-urlencode "csr=$(cat mysite.csr)" -o mysite.crt
```

The certificate includes SAN: `DNS:*.mysite.doxx, DNS:mysite.doxx`

---

## Mobile Options

### `get_mobile_options`

```json
{
  "status": "success",
  "mobile_options": {
    "connect_on_startup": 0,
    "kill_switch": 0,
    "transport": "wireguard",
    "proxy_enabled": 0,
    "onion_enabled": 0,
    "port": null
  }
}
```

### `set_mobile_options`

| Parameter | Required |
|-----------|----------|
| `token` | Yes |
| `connect_on_startup` | No (`1`/`0`) |
| `kill_switch` | No (`1`/`0`) |
| `proxy_enabled` | No (`1`/`0`) |
| `onion_enabled` | No (`1`/`0`) |

---

## Utility

### `version_check`

No auth required.

```json
{"status": "success", "version": "2.1.0", "download_url": "https://doxx.net/download"}
```

### `generate_qr`

No auth required. **Returns binary PNG, not JSON.**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `data` | Yes | Text to encode |
| `size` | No | 100-2048 pixels (default: 512) |

```bash
curl -s -X POST $API -d "generate_qr=1&data=hello&size=256" -o qr.png
```

---

## DOXX POW

### `doxxpow_challenge`

No auth required. Returns a proof-of-work challenge.

### `doxxpow_verify`

No auth required. Submits a completed POW solution, returns a token.

### `doxxpow_validate_token`

| Parameter | Required |
|-----------|----------|
| `pow_token` | Yes |

```json
{"status": "success", "valid": true, "accuracy": 95}
```

---

# Stats API

> `https://secure-wss.doxx.net`

## WebSocket

```
wss://secure-wss.doxx.net:443/ws?token=YOUR_TOKEN
```

Optional: `&tunnel_token=X` to filter to one tunnel.

### Event Types

| Type | Description | Key Fields |
|------|-------------|------------|
| `dns_block` | Blocked DNS query | `value` (domain), `category`, `count` |
| `security_event` | Security alert | `category`, `value` (service/port info) |
| `dangerous_port` | Dangerous port connection | `value` (e.g., "SSH (Port 22)") |
| `dns_bypass` | DNS bypass attempt | `value` (provider) |
| `doh_bypass` | DoH bypass attempt | `value` (provider) |
| `bandwidth` | Bandwidth (Mbps) | `value` (format: `in=X,out=Y`) |
| `dns_nxdomain` | Non-existent domain | `value` (domain) |
| `tunnel_status` | Tunnel state change | `value` (sleeping/offline) |
| `port_scan` | Port scan detected | `value` (details) |

### Event Structure

```json
{
  "tunnel_token": "abc...",
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

## REST

### `GET /api/stats/bandwidth`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | No | Filter by tunnel |
| `start` | No | ISO 8601 (default: 1h ago) |
| `end` | No | ISO 8601 (default: now) |

```json
{
  "granularity": "1m",
  "data": [
    {"tunnel_token": "abc...", "timestamp": 1707400000, "peak_in": 125.5, "peak_out": 42.3, "samples": 60}
  ],
  "aggregate": [
    {"tunnel_token": "aggregate", "timestamp": 1707400000, "peak_in": 125.5, "peak_out": 42.3, "samples": 60}
  ]
}
```

Granularity auto-selects: `1s` (<5m), `1m` (<6h), `5m` (<48h), `1h` (<30d), `6h` (30d+).

### `GET /api/stats/alerts`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `tunnel_token` | No | Filter by tunnel |
| `last` | No | `session`, `1m`, `1h`, `1d`, `7d`, `30d` |
| `start` / `end` | No | ISO 8601 (alternative to `last`) |
| `type` | No | Filter by event type |

```json
{
  "granularity": "1m",
  "totals": {"dns_block": 1234, "security_event": 5},
  "block_count": 1234,
  "category_counts": {"ads": 800, "tracking": 300, "malware": 134},
  "data": [
    {"type": "dns_block", "value": "doubleclick.net", "count": 42, "timestamp": 1707400000, "last_seen": 1707403600}
  ]
}
```

### `GET /api/stats/summary`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Auth token |
| `days` | No | Default: 30 |

### `GET /api/stats/global`

No auth. Returns global threat counter.

```json
{"status": "success", "total": 1234567890, "ts": 1707400000}
```

### `wss://secure-wss.doxx.net/ws/global`

Public WebSocket. Streams global threat counter updates. No auth.

---

# Conntrack API

> `wss://conntrack.doxx.net/ws`

Real-time connection tracking across all VPN backbone nodes.

```
wss://conntrack.doxx.net:443/ws?token=YOUR_TOKEN
```

### Messages

First message: `type: "initial"` (all current connections). Then `type: "snapshot"` every 10 seconds.

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
      "upload_speed": 1024.5,
      "download_speed": 8192.0,
      "server": "bh1.mia1.doxx.net",
      "tunnel_name": "My Laptop"
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
{"status": "healthy", "mode": "master", "clients": 3, "worker_connections": 4, "timestamp": 1707400000}
```

---

## Support

- **Portal:** [a0x13.doxx.net](https://a0x13.doxx.net)
- **Discord:** [discord.gg/Gr9rByrEzZ](https://discord.gg/Gr9rByrEzZ)
- **Email:** support@doxx.net

<p align="center"><em>doxx.net - Freedom and Privacy by Design</em></p>
