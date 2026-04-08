# Token Management

## Overview

doxx.net supports multiple auth tokens per account. Each token has a role, optional expiration, and can be restricted by IP address, country, or specific tunnels. Tokens are managed through the Config API using the endpoints below.

All token management endpoints require authentication with an existing token. The `user_list_tokens` endpoint is available to any token role. All other management endpoints require `admin` role.

## Roles

| Role | Permissions |
|------|------------|
| **admin** | Full access: everything net-admin can do, plus account management (update_profile, delete_account, create_account_recovery), token CRUD (create_token, revoke_token, unrevoke_token, delete_token, update_token), geo/IP fencing, tunnel scoping, device deletion, and Apple/billing operations |
| **net-admin** | Network configuration: everything read-only can do, plus create/update/delete tunnels, manage DNS records and domains, firewall rules, proxy settings, saved profiles, IP address assignment, device rename/offline, and subscription device management |
| **read-only** | View only: list tunnels, servers, domains, DNS records, firewall rules, profiles, addresses, blocklist stats, proxy config. No modifications. |

Roles are hierarchical: `admin` includes all `net-admin` permissions, which includes all `read-only` permissions. The primary token created at account signup is always `admin`. New tokens created via `create_token` default to `read-only` if no role is specified.

---

## Token CRUD

### user_list_tokens

List all tokens for the authenticated account.

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `user_list_tokens` | Yes | Set to `1` |
| `token` | Yes | Your auth token |

**Response:**

```json
{
  "status": "success",
  "tokens": [
    {
      "token": "...gtGwEnvY",
      "label": "Primary",
      "role": "admin",
      "created_at": "2026-04-08T08:17:14Z",
      "is_current": true,
      "geo_fence": null,
      "ip_fence": null,
      "tunnel_scope": null
    },
    {
      "token": "...sl83CZ0k",
      "label": "CI Pipeline",
      "role": "net-admin",
      "created_at": "2026-04-08T09:00:22Z",
      "expires_at": "2026-12-31T23:59:59Z",
      "is_current": false,
      "geo_fence": [
        {"country": "US", "label": "United States"}
      ],
      "ip_fence": [
        {"cidr": "10.0.0.0/8", "label": "Office"}
      ],
      "tunnel_scope": [
        {"tunnel_token": "abc123...", "label": "Production"}
      ]
    }
  ]
}
```

**Notes:**
- Tokens are listed in creation order (oldest first)
- `is_current: true` marks the token used in this request
- Revoked tokens include a `revoked_at` timestamp
- Expired tokens include an `expires_at` in the past
- Fence and tunnel scope data is included inline for each token

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "user_list_tokens=1&token=YOUR_TOKEN"
```

---

### create_token

Generate a new auth token for the account.

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `create_token` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `label` | No | Human-readable name (max 64 chars, alphanumeric + spaces/hyphens/apostrophes/periods) |
| `role` | No | `admin`, `net-admin`, or `read-only`. Defaults to `read-only` |
| `expires_at` | No | Expiration time in RFC3339 format (e.g. `2027-01-01T00:00:00Z`). Omit for no expiration |

**Response:**

```json
{
  "status": "success",
  "new_token": "kQamWzu97cEpufFtK9CayDa8oEc7Uy-TO9Qsl83CZ0k"
}
```

**The full token is only returned once at creation time.** Store it securely.

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "create_token=1&token=YOUR_TOKEN&label=CI+Pipeline&role=net-admin&expires_at=2027-01-01T00:00:00Z"
```

---

### revoke_token

Revoke (soft-delete) a token. The token becomes immediately unusable but remains visible in `user_list_tokens` with a `revoked_at` timestamp.

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `revoke_token` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string to revoke |

**Response:**

```json
{"status": "success", "message": "Token revoked"}
```

**Safety checks:**
- Cannot revoke your own active token (the one used in this request)
- Cannot revoke the last `admin` token on the account

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "revoke_token=1&token=YOUR_TOKEN&target_token=TOKEN_TO_REVOKE"
```

---

### unrevoke_token

Re-enable a previously revoked token. Clears the `revoked_at` timestamp and the token becomes immediately usable again with its original role, geo fences, IP fences, and tunnel scopes intact.

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `unrevoke_token` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string to unrevoke |

**Response:**

```json
{"status": "success", "message": "Token re-enabled"}
```

**Notes:**
- Only works on revoked tokens (returns error if the token is not revoked)
- All geo fences, IP fences, and tunnel scopes are preserved through revoke/unrevoke

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "unrevoke_token=1&token=YOUR_TOKEN&target_token=TOKEN_TO_UNREVOKE"
```

---

### delete_token

Permanently deletes a token and all its associated geo fences, IP fences, and tunnel scopes. Unlike `revoke_token`, the token is completely removed from the database and will no longer appear in `user_list_tokens`.

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `delete_token` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string to delete |

**Response:**

```json
{"status": "success", "message": "Token permanently deleted"}
```

**Safety checks:**
- Cannot delete your own active token (the one used in this request)
- Cannot delete the last `admin` token on the account
- Works on active, expired, and revoked tokens

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "delete_token=1&token=YOUR_TOKEN&target_token=TOKEN_TO_DELETE"
```

---

### update_token

Update a token's label, role, or expiration. Can be used to reactivate an expired token by extending its expiry.

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `update_token` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string to update |
| `label` | No | New label |
| `role` | No | New role: `admin`, `net-admin`, or `read-only` |
| `expires_at` | No | New expiration (RFC3339), or `never` or `null` to remove expiration |

**Response:**

```json
{"status": "success", "message": "Token updated"}
```

**Safety checks:**
- Cannot downgrade your own active token's role
- Works on expired tokens (allows reactivation by setting a future `expires_at`)
- Does not work on revoked tokens

**Note:** Geo fence, IP fence, and tunnel scope modifications require the target token to be active (not expired and not revoked). To modify fences on an expired token, first reactivate it with `update_token` by extending the expiry, then add/remove fences.

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "update_token=1&token=YOUR_TOKEN&target_token=TARGET&expires_at=never"
```

---

## Geo Fence (Country Allowlist)

Restrict a token to only work from specific countries. If any geo fence entries exist for a token, the client's country (resolved via GeoIP) must match one of them. No entries means unrestricted.

If the server cannot determine the client's country (GeoIP lookup failure), the request is allowed by default.

### add_geo_fence

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `add_geo_fence` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string to add the fence to |
| `country` | Yes | ISO 3166-1 alpha-2 country code (e.g. `US`, `DE`, `GB`) |
| `label` | No | Human-readable name (e.g. "United States") |

**Response:**

```json
{"status": "success", "message": "Geo fence added"}
```

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "add_geo_fence=1&token=YOUR_TOKEN&target_token=TARGET&country=US&label=United+States"
```

### remove_geo_fence

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `remove_geo_fence` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string |
| `country` | Yes | Country code to remove |

**Response:**

```json
{"status": "success", "message": "Geo fence removed"}
```

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "remove_geo_fence=1&token=YOUR_TOKEN&target_token=TARGET&country=US"
```

---

## IP Fence (CIDR Allowlist)

Restrict a token to only work from specific IP addresses or networks. If any IP fence entries exist for a token, the client IP must match at least one CIDR. No entries means unrestricted.

Bare IP addresses (without a prefix length) are automatically normalized to `/32` (IPv4) or `/128` (IPv6). CIDR inputs are normalized to the network address (e.g. `10.0.0.1/8` is stored as `10.0.0.0/8`).

### add_ip_fence

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `add_ip_fence` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string to add the fence to |
| `cidr` | Yes | IPv4 or IPv6 address or CIDR (e.g. `203.0.113.0/24`, `10.1.2.3`, `2001:db8::/32`) |
| `label` | No | Human-readable name (e.g. "Office network") |

**Response:**

```json
{"status": "success", "message": "IP fence added"}
```

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "add_ip_fence=1&token=YOUR_TOKEN&target_token=TARGET&cidr=203.0.113.0/24&label=Office"
```

### remove_ip_fence

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `remove_ip_fence` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string |
| `cidr` | Yes | CIDR to remove (must match exactly as stored) |

**Response:**

```json
{"status": "success", "message": "IP fence removed"}
```

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "remove_ip_fence=1&token=YOUR_TOKEN&target_token=TARGET&cidr=203.0.113.0/24"
```

---

## Tunnel Scope

Restrict a token to only operate on specific tunnels. If any tunnel scope entries exist for a token, the token can only view and modify those tunnels. No entries means access to all tunnels on the account.

### add_token_tunnel

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `add_token_tunnel` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string to scope |
| `tunnel_token` | Yes | Tunnel token to grant access to (must be owned by your account) |
| `label` | No | Human-readable name (e.g. "Production server") |

**Response:**

```json
{"status": "success", "message": "Tunnel scope added"}
```

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "add_token_tunnel=1&token=YOUR_TOKEN&target_token=TARGET&tunnel_token=TUNNEL_ID&label=Production"
```

### remove_token_tunnel

**Parameters:**

| Param | Required | Description |
|-------|----------|-------------|
| `remove_token_tunnel` | Yes | Set to `1` |
| `token` | Yes | Your auth token (must be `admin` role) |
| `target_token` | Yes | Full token string |
| `tunnel_token` | Yes | Tunnel token to remove from scope |

**Response:**

```json
{"status": "success", "message": "Tunnel scope removed"}
```

```bash
curl -s -X POST https://config.doxx.net/v1/ \
  -d "remove_token_tunnel=1&token=YOUR_TOKEN&target_token=TARGET&tunnel_token=TUNNEL_ID"
```

---

## Account Recovery

When recovery codes are used (`verify_account_recovery`), the following happens:

1. **Nuclear revoke**: All existing tokens on the account are revoked (`revoked_at` set to now). This assumes compromise since someone is using recovery codes.
2. **New admin token**: A fresh `admin` token with label `Primary` is created and returned.
3. **Code consumed**: The recovery code is marked as used (one-time use).

Old tokens and their geo fences, IP fences, and tunnel scopes are preserved as revoked rows. After recovery, an admin can:
- Use `unrevoke_token` to selectively restore tokens that were not compromised
- Use `delete_token` to permanently clean up old revoked tokens
- Use `user_list_tokens` to review all tokens (revoked tokens show a `revoked_at` timestamp)

Recovery codes are generated once per account via `create_account_recovery` (10 codes) and cannot be regenerated.

## Account Deletion

When an account is deleted (`delete_account`), all tokens, geo fences, IP fences, and tunnel scopes are deleted along with the account.
